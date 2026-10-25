[← Course Index](../README.md)

# 15.7 — Async Traits and Pinning

## What You'll Learn

- `async fn` in traits works — and where it stops working (`dyn`)
- `#[async_trait]`, `Box<dyn Future>`, and `impl Future` in return position
- What `Pin` protects and why self-referential futures need it
- `Box::pin`, `pin!`, `Unpin`, and the exact moments a learner hits them

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `interface R { get(): Promise<T> }` | `trait R { async fn get(&self) -> T; }` |
| Any interface is trivially dynamic | `async fn` in a trait is not `dyn`-compatible |
| GC means objects never move | values move in Rust; `Pin` is the opt-out |

Two separate hard things. The trait part is just "which of four spellings do I
want". `Pin` is genuinely new but narrow: it exists because `async` blocks can
contain references into themselves.

## async fn in traits works

```rust
trait Store {
    async fn get(&self, key: &str) -> Option<String>;
    async fn put(&mut self, key: &str, val: String);
}

struct Memory(std::collections::HashMap<String, String>);

impl Store for Memory {
    async fn get(&self, key: &str) -> Option<String> {
        self.0.get(key).cloned()
    }
    async fn put(&mut self, key: &str, val: String) {
        self.0.insert(key.to_string(), val);
    }
}

async fn use_store<S: Store>(s: &mut S) {
    s.put("a", "1".into()).await;
    println!("a = {:?}", s.get("a").await);
    println!("z = {:?}", s.get("z").await);
}

#[tokio::main]
async fn main() {
    let mut m = Memory(Default::default());
    use_store(&mut m).await;
}
```

Output:

```
a = Some("1")
z = None
```

No macro, no `Box`. With a generic parameter each call site monomorphizes to the
concrete future — zero overhead, same as a sync trait method.

## The limitation: not dyn-compatible

```rust
trait Store {
    async fn get(&self, key: &str) -> Option<String>;
}

struct Memory;
impl Store for Memory {
    async fn get(&self, _key: &str) -> Option<String> { None }
}

#[tokio::main]
async fn main() {
    let b: Box<dyn Store> = Box::new(Memory);
    println!("{:?}", b.get("a").await);
}
```

Does not compile (intentional):

```
error[E0038]: the trait `Store` is not dyn compatible
  --> src/main.rs:12:20
   |
12 |     let b: Box<dyn Store> = Box::new(Memory);
   |                    ^^^^^ `Store` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
  --> src/main.rs:2:14
   |
 1 | trait Store {
   |       ----- this trait is not dyn compatible...
 2 |     async fn get(&self, key: &str) -> Option<String>;
   |              ^^^ ...because method `get` is `async`
```

Every impl returns a different anonymous future of a different size, and a
vtable needs one known return layout. A `Box<dyn Future>` would supply that, but
the language will not insert a hidden allocation for you.

Second limitation: the returned future is **not** `Send`, so you cannot
`tokio::spawn` work that awaits it through a generic bound.

```rust
trait Fetch {
    async fn fetch(&self) -> String;
}

struct Api;
impl Fetch for Api {
    async fn fetch(&self) -> String { "data".into() }
}

fn spawn_fetch<F: Fetch + Send + Sync + 'static>(f: F) {
    tokio::spawn(async move {
        println!("{}", f.fetch().await);
    });
}
```

Does not compile (intentional):

```
error: future cannot be sent between threads safely
   --> src/main.rs:11:5
    |
 11 | /     tokio::spawn(async move {
 12 | |         println!("{}", f.fetch().await);
 13 | |     });
    | |______^ future created by async block is not `Send`
    |
    = help: within `{async block@src/main.rs:11:18: 11:28}`, the trait `Send` is not implemented for `impl Future<Output = String>`
note: future is not `Send` as it awaits another future which is not `Send`
help: `Send` can be made part of the associated future's guarantees for all implementations of `Fetch::fetch`
```

Note the `F: Send + Sync` bound did not help — that is about `F`, not about the
future `F::fetch` returns. The fix is spelling the return type explicitly with
the bound, which is the next section.

## Four ways to write an async trait method

### 1. `async fn` — generics only

```rust
trait Store { async fn get(&self, k: &str) -> Option<String>; }
```

Simplest. Use it when callers are generic. No `dyn`, no `Send` guarantee.

### 2. `impl Future` in return position — adds the Send bound

```rust
use std::future::Future;

fn delayed(ms: u64) -> impl Future<Output = u64> {
    async move {
        tokio::time::sleep(tokio::time::Duration::from_millis(ms)).await;
        ms
    }
}

trait Nameable {
    fn name(&self) -> impl Future<Output = String> + Send;
}

struct Thing;
impl Nameable for Thing {
    fn name(&self) -> impl Future<Output = String> + Send {
        async { "thing".to_string() }
    }
}

#[tokio::main]
async fn main() {
    println!("{}", delayed(20).await);
    println!("{}", Thing.name().await);
}
```

Output:

```
20
thing
```

This is `async fn` plus the `Send` guarantee, and it fixes the spawn error
above. Still not `dyn`-compatible. On a free function it also lets you do sync
setup before the async part: code outside the `async move { }` runs at call
time, not at await time.

### 3. `#[async_trait]` — the dyn answer

The macro rewrites each method to return `Pin<Box<dyn Future + Send + 'a>>`,
which *is* `dyn`-compatible.

```rust
use async_trait::async_trait;

#[async_trait]
trait Store: Send + Sync {
    async fn get(&self, key: &str) -> Option<String>;
}

struct Memory;
struct Null;

#[async_trait]
impl Store for Memory {
    async fn get(&self, key: &str) -> Option<String> {
        Some(format!("mem:{key}"))
    }
}

#[async_trait]
impl Store for Null {
    async fn get(&self, _key: &str) -> Option<String> { None }
}

#[tokio::main]
async fn main() {
    let stores: Vec<Box<dyn Store>> = vec![Box::new(Memory), Box::new(Null)];
    for s in &stores {
        println!("{:?}", s.get("k").await);
    }
}
```

Output:

```
Some("mem:k")
None
```

The attribute goes on both the trait and every impl; `#[async_trait(?Send)]`
drops the `Send` bound. Cost: one heap allocation and one dynamic dispatch per
call — irrelevant next to a network round trip, measurable in a hot loop.

**You still need it whenever you want `dyn`**: plugin registries,
`Box<dyn Repository>` test doubles, heterogeneous handler lists. That is the
whole remaining use case; for generics, use form 1 or 2.

### 4. Manual `Pin<Box<dyn Future>>` — what the macro generates

```rust
use std::future::Future;
use std::pin::Pin;

type Fut<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

trait Store: Send + Sync {
    fn get<'a>(&'a self, key: &'a str) -> Fut<'a, Option<String>>;
}

struct Memory;
impl Store for Memory {
    fn get<'a>(&'a self, key: &'a str) -> Fut<'a, Option<String>> {
        Box::pin(async move { Some(format!("mem:{key}")) })
    }
}

#[tokio::main]
async fn main() {
    let b: Box<dyn Store> = Box::new(Memory);
    println!("{:?}", b.get("k").await);
}
```

Output:

```
Some("mem:k")
```

Use this to avoid the macro dependency or to be explicit about lifetimes. Note
`fn`, not `async fn`, and the `'a` tying the future to the borrows.

### Choosing

| Need | Use |
|---|---|
| generic callers, same thread | `async fn` in the trait |
| generic callers, want to `spawn` | `-> impl Future<Output = T> + Send` |
| `Box<dyn Trait>`, `Vec<Box<dyn Trait>>` | `#[async_trait]` |
| `dyn` with no extra dependency | manual `Pin<Box<dyn Future>>` |

## Pin: why it exists

Here is the whole problem in six lines.

```rust
use tokio::time::{sleep, Duration};

async fn self_referential() -> usize {
    let buf = String::from("hello world");
    let word = &buf[0..5];
    sleep(Duration::from_millis(10)).await;
    println!("{word}");
    word.len()
}

#[tokio::main]
async fn main() {
    println!("{}", self_referential().await);
    let f = self_referential();
    println!("future size = {}", std::mem::size_of_val(&f));
    println!("{}", f.await);
}
```

Output:

```
hello
5
future size = 160
hello
5
```

`word` is a reference into `buf`, and both must survive the `.await`. So the
generated state machine holds `buf` *and* a pointer into `buf` — a pointer into
its own memory. That is a self-referential struct.

Move that struct to a new address and `buf` moves with it, but the stored
pointer still aims at the old location. Dangling pointer, undefined behavior, no
borrow-checker error — the aliasing lives inside compiler-generated code.

`Pin<P>` is the type-level fix: **once pinned, a value never moves again until
it drops.** Hence `Future::poll` takes `Pin<&mut Self>`; with plain `&mut Self`
you could `std::mem::swap` two futures mid-poll and corrupt both.

`Pin` is not a runtime mechanism — it allocates nothing and checks nothing. It
is a wrapper that refuses to hand out `&mut T` unless `T: Unpin`, and `&mut T`
is the only safe way to move out of a place.

## Unpin

`Unpin` is the auto-trait for "moving me is harmless, pin me or don't, I don't
care". Almost everything is `Unpin`: all primitives, `String`, `Vec`,
`Box<T>` — and `Pin<Box<T>>` itself, because moving the *box* does not move its
contents.

The exceptions you will meet: compiler-generated `async` blocks and futures, and
the stream/future adapters built on `pin_project`.

```rust
fn needs_unpin<T: Unpin>(_: T) {}

#[tokio::main]
async fn main() {
    needs_unpin(5u32);
    needs_unpin(String::from("hi"));
    needs_unpin(Box::pin(async { 1 }));
    needs_unpin(async { 1 });
}
```

The first three compile; the fourth does not (intentional):

```
error[E0277]: `{async block@src/main.rs:8:17: 8:22}` cannot be unpinned
 --> src/main.rs:8:17
  |
8 |     needs_unpin(async { 1 });
  |     ----------- ^^^^^^^^^^^ the trait `Unpin` is not implemented for `{async block@src/main.rs:8:17: 8:22}`
  |     |
  |     required by a bound introduced by this call
  |
  = note: consider using the `pin!` macro
          consider using `Box::pin` if you need to access the pinned value outside of the current scope
```

That note is the whole cheat sheet, and it appears at every site where this
matters. `pin!` for a local; `Box::pin` to move it elsewhere.

## Box::pin vs pin!

| | `Box::pin(fut)` | `pin!(fut)` |
|---|---|---|
| Allocates | yes, heap | no, stack |
| Type | `Pin<Box<F>>` | `Pin<&mut F>` |
| Lifetime | `'static` if `F: 'static` | borrows the enclosing scope |
| Store in a struct / `Vec` | yes | no |
| Return from a function | yes | no |
| Is `Unpin` itself | yes | yes |

Rule: `pin!` when the pinned value stays in this function. `Box::pin` when it
has to go somewhere else.

## Where you will actually hit this

### Storing futures in a struct or a Vec

Every `async` block has its own anonymous type, so a heterogeneous collection
needs type erasure, and type erasure needs the box.

```rust
use std::future::Future;
use std::pin::Pin;

struct Job {
    name: String,
    task: Pin<Box<dyn Future<Output = u32> + Send>>,
}

#[tokio::main]
async fn main() {
    let jobs = vec![
        Job { name: "a".into(), task: Box::pin(async { 1 }) },
        Job { name: "b".into(), task: Box::pin(async {
            tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
            2
        }) },
    ];
    for j in jobs {
        println!("{} -> {}", j.name, j.task.await);
    }

    let futs: Vec<Pin<Box<dyn Future<Output = String> + Send>>> = vec![
        Box::pin(async { "one".to_string() }),
        Box::pin(async { "two".to_string() }),
    ];
    for f in futs {
        print!("{} ", f.await);
    }
    println!();
}
```

Output:

```
a -> 1
b -> 2
one two
```

`Pin<Box<dyn Future<Output = T> + Send>>` is common enough that `futures`
aliases it as `BoxFuture<'static, T>`, and `.boxed()` builds one.

### Re-polling the same future in a select! loop

This is the cancellation-safety fix from 15.4. Awaiting a pinned future by
`&mut` means the *same* future is re-polled each round, keeping its progress.

```rust
use std::pin::pin;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut long = pin!(async {
        sleep(Duration::from_millis(90)).await;
        "long job done"
    });

    let mut heartbeats = 0;
    loop {
        tokio::select! {
            v = &mut long => {
                println!("{v} after {heartbeats} heartbeats");
                break;
            }
            _ = sleep(Duration::from_millis(25)) => {
                heartbeats += 1;
                println!("heartbeat {heartbeats}");
            }
        }
    }
}
```

Output:

```
heartbeat 1
heartbeat 2
heartbeat 3
long job done after 3 heartbeats
```

The long job kept its progress across three lost rounds. Written as
`v = long_job() => ...` inside the loop it would restart from zero every 25ms
and never finish. `&mut F` is `Future` when `F: Future + Unpin`, and
`Pin<&mut F>` is `Unpin` — that is the whole trick.

### Non-Unpin stream adapters

The `pin!` calls in 15.6 are this same situation: `StreamExt::next` requires
`Unpin`, and adapters holding a timer are not.

### Polling by hand

Occasionally you want to know whether a future is ready without awaiting it.

```rust
use std::future::Future;
use std::task::{Context, Poll, Waker};

#[tokio::main]
async fn main() {
    let mut f = Box::pin(async { 7 });
    let mut cx = Context::from_waker(Waker::noop());
    match f.as_mut().poll(&mut cx) {
        Poll::Ready(v) => println!("Ready({v})"),
        Poll::Pending => println!("Pending"),
    }

    let mut g = Box::pin(async {
        tokio::task::yield_now().await;
        9
    });
    println!("first poll pending: {}", g.as_mut().poll(&mut cx).is_pending());
    println!("awaited: {}", g.await);
}
```

Output:

```
Ready(7)
first poll pending: true
awaited: 9
```

`as_mut()` reborrows a `Pin<Box<F>>` as `Pin<&mut F>`, which is what `poll`
wants. `Waker::noop()` is a waker that does nothing — fine for a one-shot check,
useless for anything that needs waking. Prefer `futures::FutureExt::now_or_never()`
for this in real code.

## Implementing a future that holds another future

The one place you must think about pinning in your own types. Projecting
`Pin<&mut Self>` down to `Pin<&mut Field>` is unsafe to do by hand, so use the
`pin-project-lite` macro.

```rust
use pin_project_lite::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

pin_project! {
    struct Logged<F> {
        #[pin]
        inner: F,
        label: &'static str,
    }
}

impl<F: Future> Future for Logged<F> {
    type Output = F::Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<F::Output> {
        let this = self.project();
        match this.inner.poll(cx) {
            Poll::Ready(v) => {
                println!("{} ready", this.label);
                Poll::Ready(v)
            }
            Poll::Pending => {
                println!("{} pending", this.label);
                Poll::Pending
            }
        }
    }
}
```

`#[pin]` makes `project()` hand back `Pin<&mut F>` for that field and plain
`&mut` for the rest. Most tokio/tower middleware is built this way — but you can
write async application code for years without needing it.

## Quick Reference

| Item | Means |
|---|---|
| `async fn` in a trait | works; generics only, future is not `Send` |
| `-> impl Future<Output = T> + Send` | `async fn` plus a `Send` guarantee |
| `#[async_trait]` | boxes the future so the trait is `dyn`-compatible |
| `#[async_trait(?Send)]` | same, single-threaded |
| `Pin<Box<dyn Future<Output = T> + Send>>` | the type-erased future |
| `BoxFuture<'static, T>` | `futures` alias for that |
| `fut.boxed()` | build one, from `FutureExt` |
| `Box::pin(fut)` | heap-pin; movable, storable, `'static` |
| `pin!(fut)` | stack-pin; local only, no allocation |
| `Unpin` | "moving me is safe"; everything but async blocks and adapters |
| `pinned.as_mut()` | `Pin<Box<F>>` -> `Pin<&mut F>` |
| `&mut pinned_fut` in `select!` | re-poll the same future, keep its progress |
| `pin_project!` | safely project `Pin<&mut Self>` into fields |

## Common Mistakes

**Reaching for `#[async_trait]` reflexively.** Plain `async fn` in traits works
and costs nothing. Add the macro only when you need `dyn`.

**Adding `F: Send` and expecting the returned future to be `Send`.** Those are
different types. You need `-> impl Future<Output = T> + Send` on the method.

**Calling a fresh future each round of a `select!` loop.** It restarts from
scratch every time. `pin!` it outside the loop and await `&mut` it.

**Trying to return a `pin!` value or store it in a struct.** It borrows the
current stack frame. Use `Box::pin`.

**Writing `unsafe` to project `Pin<&mut Self>` into a field.** Almost always
wrong. Use `pin-project-lite`.

## Exercises

1. Define `trait Cache { async fn get(&self, k: &str) -> Option<String>; }` with
   two impls, write a generic `async fn report<C: Cache>(c: &C)`, then try
   `Vec<Box<dyn Cache>>`. Capture the error and fix it with `#[async_trait]`.
2. Take a trait method whose returned future fails to `spawn` with a `Send`
   error. Fix it by changing the signature to `-> impl Future<..> + Send`.
3. Build a `Vec<Pin<Box<dyn Future<Output = Result<u32, String>> + Send>>>` with
   three differently shaped async blocks, await them in order, and collect the
   results.
4. Write a loop that `select!`s a 200ms job against a 50ms ticker. Do it once
   without `pin!` (observe it never completes) and once with.
5. Write an async fn that creates a `String`, takes a `&str` slice of it, awaits
   a sleep, then prints the slice. Print `size_of_val` of its future and explain
   why the number is larger than the `String` alone.

---

[← Course Index](../README.md)
