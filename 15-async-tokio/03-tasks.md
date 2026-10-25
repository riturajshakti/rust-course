[← Course Index](../README.md)

# 15.3 — Spawning Tasks and JoinSet

## What You'll Learn

- `tokio::spawn`, `JoinHandle`, `JoinError`, and panics inside tasks
- `join!` vs `try_join!` vs `spawn` — when each is right
- `JoinSet` for a dynamic set of tasks, with `join_next` and `abort_all`
- Why tasks need `Send + 'static`, and what `Arc` fixes

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `promise` (already running) | `tokio::spawn(fut)` -> `JoinHandle` |
| `Promise.all([a, b])` | `tokio::join!(a, b)` or a `JoinSet` |
| `Promise.allSettled` | `JoinSet` draining `Result<T, JoinError>` |
| Go `go f()` | `tokio::spawn(async { f().await })` |
| Go `WaitGroup` | awaiting handles, or `JoinSet` |

A task is the unit the scheduler sees — Tokio's green thread. Unlike `go f()`, a
spawned task gives you back a handle you can await, abort, or drop.

## tokio::spawn and JoinHandle

`spawn` hands the future to the runtime *immediately* — this is where laziness
ends. The `JoinHandle` is itself a future resolving to `Result<T, JoinError>`.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let h = tokio::spawn(async {
        sleep(Duration::from_millis(30)).await;
        "task result"
    });
    println!("spawned; main keeps going");
    let v = h.await.unwrap();
    println!("{v}");

    let h2 = tokio::spawn(async { panic!("task blew up") });
    match h2.await {
        Ok(v) => println!("ok {v:?}"),
        Err(e) => println!("JoinError: panicked = {}, cancelled = {}", e.is_panic(), e.is_cancelled()),
    }
}
```

Output:

```
spawned; main keeps going
task result

thread 'tokio-rt-worker' (13861603) panicked at src/main.rs:13:35:
task blew up
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
JoinError: panicked = true, cancelled = false
```

A panic in a task does **not** kill the process or the runtime. It is caught and
delivered as `JoinError`. This is a real difference from a panic on a plain
thread in Module 14 — and from Go, where an unrecovered panic in a goroutine
takes the program down.

`e.into_panic()` gives you the `Box<dyn Any>` payload if you want to re-panic or
inspect it.

## Spawning many in a loop

```rust
use tokio::time::{sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let mut handles = Vec::new();
    for i in 0..5 {
        handles.push(tokio::spawn(async move {
            sleep(Duration::from_millis(50)).await;
            i * i
        }));
    }
    let mut out = Vec::new();
    for h in handles {
        out.push(h.await.unwrap());
    }
    println!("{out:?} in {:?}", start.elapsed());
}
```

Output:

```
[0, 1, 4, 9, 16] in 52.313292ms
```

Roughly 50ms, not 250ms: all five ran concurrently. Collecting handles first and
awaiting second is the key — awaiting inside the spawn loop would serialize it.
Results come back in spawn order because we await in spawn order.

## join! vs try_join! vs spawn

```rust
use tokio::time::{sleep, Duration};

async fn ok(n: u32) -> Result<u32, String> {
    sleep(Duration::from_millis(10)).await;
    Ok(n)
}
async fn bad() -> Result<u32, String> {
    sleep(Duration::from_millis(5)).await;
    Err("boom".to_string())
}

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(ok(1), ok(2));
    println!("join!     {a:?} {b:?}");

    let r = tokio::try_join!(ok(1), ok(2));
    println!("try_join! {r:?}");

    let r = tokio::try_join!(ok(1), bad());
    println!("try_join! {r:?}");
}
```

Output:

```
join!     Ok(1) Ok(2)
try_join! Ok((1, 2))
try_join! Err("boom")
```

| | Runs on | Count | On error | Result |
|---|---|---|---|---|
| `join!` | the current task | fixed, known at compile time | waits for all | tuple of every output |
| `try_join!` | the current task | fixed | returns at first `Err`, dropping the rest | `Result<tuple, E>` |
| `spawn` | a new task | any number | nothing special | `JoinHandle` per task |

The crucial distinction: `join!` and `try_join!` poll their futures on **one
task**, so they give concurrency but no parallelism — two CPU-heavy futures in a
`join!` still share one thread. `spawn` creates independent tasks the scheduler
can place on different workers.

Also: `join!` takes a fixed list. For "N of these, N known at runtime", use a
`JoinSet`.

## JoinSet

`JoinSet` owns a growable group of tasks and yields results **as they finish**,
not in spawn order.

```rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let mut set = JoinSet::new();
    for i in 1..=5u64 {
        set.spawn(async move {
            sleep(Duration::from_millis(i * 20)).await;
            format!("job {i}")
        });
    }
    println!("len = {}", set.len());
    while let Some(res) = set.join_next().await {
        println!("{:>8} at {:>3}ms", res.unwrap(), start.elapsed().as_millis());
    }
    println!("empty = {}", set.is_empty());

    let mut set = JoinSet::new();
    for i in 0..3 {
        set.spawn(async move {
            sleep(Duration::from_secs(10)).await;
            i
        });
    }
    set.abort_all();
    while let Some(res) = set.join_next().await {
        println!("after abort_all: cancelled = {}", res.unwrap_err().is_cancelled());
    }
}
```

Output:

```
len = 5
   job 1 at  21ms
   job 2 at  42ms
   job 3 at  62ms
   job 4 at  82ms
   job 5 at 101ms
empty = true
after abort_all: cancelled = true
after abort_all: cancelled = true
after abort_all: cancelled = true
```

Completion-ordered, so you start processing the fastest result immediately
instead of waiting on the slowest. `abort_all` cancels every task; you still
drain with `join_next` to collect the `JoinError`s.

| Method | Does |
|---|---|
| `spawn(fut)` | add a task, returns an `AbortHandle` |
| `spawn_blocking(f)` | add a blocking closure (15.8) |
| `spawn_on(fut, &handle)` | add onto a specific runtime |
| `join_next()` | await the next finisher, `None` when empty |
| `join_next_with_id()` | same, plus the task's `Id` |
| `try_join_next()` | non-blocking poll for a finished task |
| `len()` / `is_empty()` | tasks still outstanding |
| `abort_all()` | cancel all; results still need draining |
| `detach_all()` | let them run, forget them |
| `shutdown()` | abort all *and* await them in one call |

Dropping a `JoinSet` aborts everything in it — a useful structured-concurrency
property: a scope cannot leak its tasks.

## Tasks must be 'static

Spawned tasks outlive the scope that created them, so they cannot borrow local
data.

```rust
#[tokio::main]
async fn main() {
    let data = vec![1, 2, 3];
    let h = tokio::spawn(async {
        println!("{:?}", data);
    });
    h.await.unwrap();
}
```

Does not compile (intentional):

```
error[E0373]: async block may outlive the current function, but it borrows `data`, which is owned by the current function
 --> src/main.rs:4:26
  |
4 |     let h = tokio::spawn(async {
  |                          ^^^^^ may outlive borrowed value `data`
5 |         println!("{:?}", data);
  |                          ---- `data` is borrowed here
  |
  = note: async blocks are not executed immediately and must either take a reference or ownership of outside variables they use
help: to force the async block to take ownership of `data` (and any other referenced variables), use the `move` keyword
  |
4 |     let h = tokio::spawn(async move {
  |                                ++++
```

Adding `move` fixes it. There is no `std::thread::scope` equivalent for tasks —
`JoinSet` is as close as it gets, and it still requires `'static`.

## Tasks must be Send

On the multi-thread runtime a task can migrate between workers, so everything
held across an `.await` must be `Send`.

```rust
use std::rc::Rc;

#[tokio::main]
async fn main() {
    let rc = Rc::new(5);
    tokio::spawn(async move {
        println!("{rc}");
    })
    .await
    .unwrap();
}
```

Does not compile (intentional):

```
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
  6 | /     tokio::spawn(async move {
  7 | |         println!("{rc}");
  8 | |     })
    | |______^ future created by async block is not `Send`
    |
    = help: within `{async block@src/main.rs:6:18: 6:28}`, the trait `Send` is not implemented for `Rc<i32>`
note: captured value is not `Send`
   --> src/main.rs:7:20
    |
  7 |         println!("{rc}");
    |                    ^^ has type `Rc<i32>` which is not `Send`
note: required by a bound in `tokio::spawn`
    |
176 |         F: Future + Send + 'static,
    |                     ^^^^ required by this bound in `spawn`
```

Swap `Rc` for `Arc`. The same error appears for `RefCell` guards and
`std::sync::MutexGuard` held across an `.await` (15.5).

## Arc for sharing into tasks

```rust
use std::sync::Arc;
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let shared = Arc::new(vec![10, 20, 30]);
    let mut set = JoinSet::new();
    for i in 0..3 {
        let shared = Arc::clone(&shared);
        set.spawn(async move { shared[i] * 2 });
    }
    let mut out = Vec::new();
    while let Some(r) = set.join_next().await {
        out.push(r.unwrap());
    }
    out.sort();
    println!("{out:?}");
    println!("strong_count after join = {}", Arc::strong_count(&shared));
}
```

Output:

```
[20, 40, 60]
strong_count after join = 1
```

Clone the `Arc` *outside* the async block, move the clone in. Shadowing the name
(`let shared = Arc::clone(&shared);`) is the idiomatic shape. For mutable shared
state, wrap in a `Mutex` — 15.5.

## Task IDs

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let h = tokio::spawn(async {
        println!("inside: id = {}", tokio::task::id());
        sleep(Duration::from_millis(10)).await;
        1
    });
    println!("outside: id = {}", h.id());
    h.await.unwrap();

    let h = tokio::spawn(async {
        sleep(Duration::from_secs(10)).await;
        "never"
    });
    sleep(Duration::from_millis(10)).await;
    h.abort();
    println!("is_finished = {}", h.is_finished());
    match h.await {
        Ok(_) => println!("completed"),
        Err(e) => println!("cancelled = {}", e.is_cancelled()),
    }
}
```

Output:

```
outside: id = 13
inside: id = 13
is_finished = false
cancelled = true
```

The number is runtime-internal and arbitrary — useful in log lines, not as a
stable key. `tokio::task::try_id()` returns `Option` when you might not be in a
task.

`abort()` requests cancellation; `is_finished()` can still be `false` right
after, because the task only actually stops at its next `.await`. Awaiting the
handle gives the definitive answer.

## yield_now

`yield_now().await` hands control back to the scheduler voluntarily. Mostly
needed inside a long loop with no other `.await` in it.

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    let a = async {
        for i in 0..3 {
            println!("A{i}");
            tokio::task::yield_now().await;
        }
    };
    let b = async {
        for i in 0..3 {
            println!("B{i}");
            tokio::task::yield_now().await;
        }
    };
    tokio::join!(a, b);
}
```

Output:

```
A0
B0
B1
A1
A2
B2
```

Interleaved, not `A0 A1 A2 B0 B1 B2` — each yield lets the other future run.
Remove the yields and `A` runs to completion first, because nothing suspends it.

Yielding is not a fix for CPU-bound work; see 15.8.

## Detached tasks and shutdown

Drop the `JoinHandle` and the task keeps running — it is detached, with no way
to await it. When the runtime shuts down, detached tasks are aborted wherever
they are.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        sleep(Duration::from_millis(5)).await;
        println!("detached: made it");
    });
    tokio::spawn(async {
        sleep(Duration::from_millis(500)).await;
        println!("detached: never printed");
    });
    sleep(Duration::from_millis(50)).await;
    println!("main returns; runtime drops");
}
```

Output:

```
detached: made it
main returns; runtime drops
```

The 500ms task was silently killed. This is the single most common cause of
"my background job sometimes doesn't finish": `main` returned first. Keep the
handle and await it, put it in a `JoinSet`, or use an explicit shutdown
handshake (15.4).

Note the `JoinHandle` is `#[must_use]`-free: dropping it is legal and silent.
`let _ = tokio::spawn(...)` is a deliberate detach; make it visible.

## spawn_local for !Send futures

On a `current_thread` runtime inside a `LocalSet`, tasks need not be `Send`.

```rust
use std::rc::Rc;
use tokio::task::LocalSet;

#[tokio::main]
async fn main() {
    let local = LocalSet::new();
    local
        .run_until(async {
            let rc = Rc::new(String::from("not Send"));
            let rc2 = Rc::clone(&rc);
            tokio::task::spawn_local(async move {
                println!("spawn_local sees: {rc2}");
            })
            .await
            .unwrap();
            println!("strong_count = {}", Rc::strong_count(&rc));
        })
        .await;
}
```

Output:

```
spawn_local sees: not Send
strong_count = 1
```

`spawn_local` panics outside a `LocalSet`. You will need this rarely — mostly
for FFI handles and single-threaded client libraries. Prefer `Arc` and ordinary
`spawn`.

## Quick Reference

| Call | Returns | Notes |
|---|---|---|
| `tokio::spawn(fut)` | `JoinHandle<T>` | needs `Send + 'static` |
| `handle.await` | `Result<T, JoinError>` | `Err` on panic or abort |
| `handle.abort()` | `()` | stops at next `.await` |
| `handle.is_finished()` | `bool` | non-blocking check |
| `handle.id()` | `task::Id` | for logs |
| `tokio::task::id()` | `task::Id` | from inside a task |
| `tokio::join!(a, b)` | `(A, B)` | one task, fixed arity |
| `tokio::try_join!(a, b)` | `Result<(A, B), E>` | short-circuits |
| `JoinSet::new()` | — | dynamic group, completion order |
| `set.join_next().await` | `Option<Result<T, JoinError>>` | |
| `set.abort_all()` | `()` | then drain |
| `set.shutdown().await` | `()` | abort + await all |
| `task::yield_now().await` | `()` | cooperative yield |
| `task::spawn_local(fut)` | `JoinHandle<T>` | `LocalSet` only, `!Send` ok |
| `e.is_panic()` / `e.is_cancelled()` | `bool` | classify a `JoinError` |

## Common Mistakes

**Awaiting inside the spawn loop.** `for x in xs { tokio::spawn(f(x)).await; }`
is sequential. Collect handles, then await — or use a `JoinSet`.

**Dropping the handle and expecting the task to finish.** When `main` returns,
the runtime drops and pending tasks are aborted. Await it or track it.

**`.unwrap()` on every `handle.await`.** That turns a task panic into a panic in
your supervisor. Match on `JoinError` and log.

**Reaching for `spawn` where `join!` is enough.** `spawn` allocates and adds
scheduling; for two I/O futures you await right here, `join!` is cheaper.

**Cloning the `Arc` inside the async block.** `async move { Arc::clone(&a) }`
moves `a` itself, so the loop's second iteration fails. Clone before the block.

## Exercises

1. Spawn 10 tasks, each sleeping `i * 10` ms and returning `i`. Collect results
   with a `Vec<JoinHandle<_>>` and print them. Then redo it with a `JoinSet` and
   note how the order changes.
2. Spawn a task that panics. Await the handle and print `is_panic()`,
   `is_cancelled()`, and the payload via `into_panic()` downcast to `&str`.
3. Write `async fn race(a: u64, b: u64) -> u64` that spawns two sleeping tasks
   and returns whichever finishes first, aborting the other.
4. Share a `Arc<Mutex<Vec<u32>>>` across 5 spawned tasks that each push their
   index. Print the sorted result.
5. Spawn a task that loops forever incrementing a counter with no `.await`
   inside, on a `current_thread` runtime, plus a second task that prints. Explain
   the output, then fix it with `yield_now`.

---

[← Course Index](../README.md)
