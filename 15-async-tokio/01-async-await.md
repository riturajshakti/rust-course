[← 14.6 Scoped Threads and Thread Pools](../14-concurrency/06-scoped-pools.md) · [Course Index](../README.md) · [15.2 The Tokio Runtime →](02-tokio-runtime.md)

# 15.1 — async/await and Futures

## What You'll Learn

- `async fn` returns a `Future`; nothing runs until you `.await` it
- The `Future` trait: `poll`, `Poll::Ready`/`Poll::Pending`, `Context`
- Async blocks, `async move`, and what "state machine" actually means
- Why calling an `async fn` from `main` needs a runtime

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `async function f()` returns a Promise | `async fn f()` returns a `Future` |
| Promise starts running immediately | Future does nothing until `.await`ed |
| Node/browser has a built-in event loop | You pick and start a runtime (Tokio) |
| Python `async def` + `asyncio.run` | `async fn` + `#[tokio::main]` |

**The one difference that matters:** a JS Promise is *eager* — the work starts
the moment you call the function. A Rust `Future` is *lazy* — it is an inert
value describing work that has not begun. `let p = fetch()` in JS has already
fired the request; `let f = fetch()` in Rust has fired nothing.

Python's `asyncio` is the closer analogue: a coroutine object also does nothing
until awaited or wrapped in a task. But Python ships one event loop; Rust ships
none, so you add Tokio yourself.

## async fn returns a Future

```rust
async fn greet(name: &str) -> String {
    format!("hello, {name}")
}

#[tokio::main]
async fn main() {
    let fut = greet("raj");
    println!("future created, nothing has run yet");
    let s = fut.await;
    println!("{s}");
}
```

Output:

```
future created, nothing has run yet
hello, raj
```

The declared return type is `String`, but the actual type is
`impl Future<Output = String>`. The compiler rewrites the signature for you.

## Futures are lazy — the warning proves it

Drop a future on the floor and the body never executes.

```rust
async fn work() {
    println!("working");
}

#[tokio::main]
async fn main() {
    work();
    println!("done");
}
```

`cargo build`:

```
warning: unused implementer of `Future` that must be used
 --> src/main.rs:7:5
  |
7 |     work();
  |     ^^^^^^
  |
  = note: futures do nothing unless you `.await` or poll them
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
```

It compiles and prints only `done`. In JS the equivalent would have printed
`working` and merely left a dangling promise.

## .await can only appear inside async

```rust
async fn work() -> i32 { 7 }

fn main() {
    let v = work().await;
    println!("{v}");
}
```

Does not compile (intentional):

```
error[E0728]: `await` is only allowed inside `async` functions and blocks
 --> src/main.rs:4:20
  |
3 | fn main() {
  | --------- this is not `async`
4 |     let v = work().await;
  |                    ^^^^^ only allowed inside `async` functions and blocks
```

`.await` is postfix, not a prefix keyword — it chains: `client.get(url).await?`.

## Awaiting needs a runtime at runtime too

Even inside an async context, Tokio's own futures need Tokio's reactor driving
them. Hand one to a different executor and it panics.

```rust
use tokio::time::{sleep, Duration};

fn main() {
    let fut = sleep(Duration::from_millis(10));
    futures::executor::block_on(fut);
    println!("unreachable");
}
```

Output (intentional panic):

```
thread 'main' (13854585) panicked at src/main.rs:4:15:
there is no reactor running, must be called from the context of a Tokio 1.x runtime
```

A future is just a value; something has to poll it. Lesson 15.2 covers how to
start that something.

## Sequential await vs concurrent

`.await` suspends the current task until that one future completes. Two awaits
in a row are sequential, exactly like JS `await a; await b;`.

```rust
use tokio::time::{sleep, Duration, Instant};

async fn fetch(id: u32) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("item-{id}")
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let a = fetch(1).await;
    let b = fetch(2).await;
    println!("{a} {b} sequential {:?}", start.elapsed());

    let start = Instant::now();
    let (a, b) = tokio::join!(fetch(3), fetch(4));
    println!("{a} {b} concurrent {:?}", start.elapsed());
}
```

Output:

```
item-1 item-2 sequential 204.484ms
item-3 item-4 concurrent 102.146042ms
```

Millisecond values vary per run; the shape does not — roughly 200ms versus
roughly 100ms. `tokio::join!` is the analogue of `Promise.all`. Lesson 15.3
covers it properly.

## The Future trait

```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`poll` is a question: "are you done?" It answers `Poll::Ready(value)` or
`Poll::Pending`. If pending, the future must have stashed the waker from `cx`
somewhere so whatever it is waiting on can wake it later. A runtime that gets
`Pending` stops polling and goes to do other work until that wake arrives.

You almost never write this by hand, but writing it once makes `.await`
concrete.

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct CountDown(u32);

impl Future for CountDown {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        println!("poll called, n = {}", self.0);
        if self.0 == 0 {
            Poll::Ready("liftoff")
        } else {
            self.0 -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let out = CountDown(3).await;
    println!("{out}");
}
```

Output:

```
poll called, n = 3
poll called, n = 2
poll called, n = 1
poll called, n = 0
liftoff
```

Four polls for one `.await`. The `wake_by_ref()` call is what schedules the next
poll — forget it and the task sleeps forever. Note `poll` takes
`Pin<&mut Self>`, not `&mut Self`; lesson 15.7 explains why.

## What "state machine" means here

The compiler turns each `async fn` into an enum: one variant per `.await` point,
plus the locals that must survive across that point. `.await` becomes "save
state, return `Pending`"; the next `poll` resumes at the saved variant.

The practical consequence is that the future's size is the size of its largest
state, not its average.

```rust
use tokio::time::{sleep, Duration};

async fn small() -> u8 { 1 }

async fn big() {
    let buf = [0u8; 1024];
    sleep(Duration::from_millis(1)).await;
    println!("{}", buf.len());
}

#[tokio::main]
async fn main() {
    println!("{}", std::mem::size_of_val(&small()));
    println!("{}", std::mem::size_of_val(&big()));
    big().await;
}
```

Output:

```
1
1144
1024
```

`big`'s 1KB array lives across the `.await`, so it lives in the future. This is
why a large async fn is best split, or its big locals boxed — a 100KB future
spawned 10,000 times is a gigabyte.

There is no heap allocation here: the future is a plain value on the stack. Only
`Box::pin` or `tokio::spawn` allocates.

## Async blocks

`async { ... }` is an expression that evaluates to a future. It is the
closure-equivalent of `async fn`.

```rust
use std::future::Future;

#[tokio::main]
async fn main() {
    let block = async {
        println!("inside async block");
        41 + 1
    };
    println!("{}", block.await);

    let name = String::from("raj");
    let owned = async move {
        format!("moved: {name}")
    };
    println!("{}", owned.await);

    let n = 5;
    let borrowed = async {
        n * 2
    };
    println!("{}", borrowed.await);

    let futs: Vec<std::pin::Pin<Box<dyn Future<Output = u32>>>> =
        vec![Box::pin(async { 1 }), Box::pin(async { 2 })];
    for f in futs {
        print!("{} ", f.await);
    }
    println!();
}
```

Output:

```
inside async block
42
moved: raj
10
1 2
```

The last block is the standard way to put differently-shaped futures in a
collection: every `async` block has its own anonymous type, so they need a
`dyn Future` behind a `Pin<Box<_>>`.

## async move

Same rule as closures: `async` borrows its captures, `async move` takes them.
The borrowing form fails the moment the future outlives the borrow, which is
always true for `tokio::spawn` — lesson 15.3 shows that error. Inside a
`.await`ed-right-here block, plain `async` is fine.

```rust
let name = String::from("raj");
let f = async move { name.len() };
```

`name` is now owned by the future and unusable outside it.

## Async fn in a struct impl

No special syntax; `async` goes on the method.

```rust
struct Client { base: String }

impl Client {
    async fn get(&self, path: &str) -> String {
        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
        format!("{}{}", self.base, path)
    }
}
```

The returned future borrows `self`, so `self` must outlive the `.await`. Lesson
15.7 covers `async fn` in *traits*, which has extra rules.

## Error handling reads the same

`?` works inside async fns and propagates through `.await`.

```rust
async fn parse(s: &str) -> Result<u32, std::num::ParseIntError> {
    let n: u32 = s.parse()?;
    Ok(n * 2)
}
```

A `Result`-returning async fn is just a future whose `Output` is a `Result`.
Nothing about Module 08 changes.

## Quick Reference

| Form | Meaning |
|---|---|
| `async fn f() -> T` | returns `impl Future<Output = T>` |
| `f()` | builds the future, runs nothing |
| `f().await` | polls it to completion, suspending the task |
| `async { .. }` | future-valued expression, borrows captures |
| `async move { .. }` | same, takes ownership of captures |
| `Pin<Box<dyn Future<Output = T>>>` | type-erased future, for collections |
| `Poll::Ready(v)` / `Poll::Pending` | poll's two answers |
| `cx.waker().wake_by_ref()` | ask the runtime to poll again |
| `#[tokio::main]` | wraps `main` in a runtime (15.2) |
| `tokio::join!(a, b)` | run futures concurrently, like `Promise.all` |

## Common Mistakes

**Expecting the work to start when you call the function.** It does not. If you
never `.await` (or spawn) the future, nothing happens — heed the
`unused_must_use` warning.

**Writing `await f()`.** It is postfix: `f().await`.

**Awaiting in a loop and calling it concurrency.** `for u in users { fetch(u).await }`
is strictly sequential. Use `join!`, `JoinSet`, or `buffer_unordered` (15.3, 15.6).

**Holding a big buffer across an `.await`.** It becomes part of the future's
size. Scope it tightly or box it.

**Calling `block_on` from a plain executor on a Tokio future.** You get
`there is no reactor running`. Tokio's timers and sockets only work on a Tokio
runtime.

## Exercises

1. Write `async fn double(n: u32) -> u32` and call it without awaiting. Confirm
   the `unused_must_use` warning, then fix it.
2. Write three async fns that each sleep 50ms and return a number. Time running
   them with three `.await`s, then with `tokio::join!`. Print both durations.
3. Implement `Future` by hand for a struct `Ready<T>(Option<T>)` that returns
   `Poll::Ready` on the first poll by `take()`ing the value. Await it.
4. Write an async fn that holds a `Vec<u8>` of 10,000 bytes across an `.await`.
   Print `size_of_val` of its future. Then box the `Vec` and print again.
5. Build a `Vec<Pin<Box<dyn Future<Output = String>>>>` with three differently
   shaped async blocks and await them in a loop.

---

[← 14.6 Scoped Threads and Thread Pools](../14-concurrency/06-scoped-pools.md) · [Course Index](../README.md) · [15.2 The Tokio Runtime →](02-tokio-runtime.md)
