[← 15.1 async/await and Futures](01-async-await.md) · [Course Index](../README.md) · [15.3 Spawning Tasks and JoinSet →](03-tasks.md)

# 15.2 — The Tokio Runtime

## What You'll Learn

- `#[tokio::main]`, what it expands to, and the manual equivalent
- `multi_thread` vs `current_thread`, `worker_threads`, `Builder` options
- `block_on` from sync code, and why nesting it panics
- Which tokio feature flags to turn on, and where the runtime lives in a backend

## Coming From Another Language

| Other | Rust |
|---|---|
| Node's event loop, always there | You construct one: `Runtime::new()` |
| `asyncio.run(main())` | `rt.block_on(main())` |
| Go's scheduler, built into the binary | Tokio, a library you opt into |

Node and Go give you one event loop per process, implicitly. Tokio gives you a
value you create, configure, and drop. That is more ceremony but it means you
decide the thread count, and you can run two independent runtimes if you want.

## The manual form first

A runtime plus `block_on` is the whole idea: `block_on` parks the calling OS
thread and drives the future to completion on the runtime.

```rust
use tokio::runtime::Runtime;

async fn work() -> &'static str { "done" }

fn main() {
    let rt = Runtime::new().unwrap();
    println!("{}", rt.block_on(work()));
    println!("{}", rt.block_on(async { 1 + 1 }));
}
```

Output:

```
done
2
```

`Runtime::new()` gives a multi-threaded runtime with all drivers enabled. One
runtime serves any number of `block_on` calls.

## What #[tokio::main] expands to

```rust
#[tokio::main]
async fn main() {
    println!("hi");
}
```

is sugar for, roughly:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("hi");
        })
}
```

Note the generated `main` is **not** async — the attribute strips `async` and
wraps the body. That is why `main` is the one place a runtime appears from
nowhere, and why the error in 15.1 says "must be called from the context of a
Tokio 1.x runtime": outside `main`'s wrapper, there is none.

The attribute takes arguments.

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("current_thread flavor via attribute");
    println!("workers = {}", tokio::runtime::Handle::current().metrics().num_workers());
}
```

Output:

```
current_thread flavor via attribute
workers = 1
```

```rust
#[tokio::main(worker_threads = 2)]
async fn main() {
    println!("workers = {}", tokio::runtime::Handle::current().metrics().num_workers());
}
```

Output:

```
workers = 2
```

`#[tokio::test]` is the same thing for tests, defaulting to `current_thread`.

## multi_thread vs current_thread

`multi_thread` runs a pool of worker threads with work stealing: spawned tasks
may move between threads, so they must be `Send`. `current_thread` runs
everything on the thread that called `block_on` — no stealing, no `Send`
requirement on the task's internals beyond what you spawn.

```rust
use tokio::runtime::Builder;

fn main() {
    let rt = Builder::new_multi_thread()
        .worker_threads(2)
        .thread_name("my-worker")
        .enable_all()
        .build()
        .unwrap();

    rt.block_on(async {
        let mut set = tokio::task::JoinSet::new();
        for i in 0..4 {
            set.spawn(async move {
                std::thread::sleep(std::time::Duration::from_millis(20));
                format!("task {i} ran on thread named {:?}", std::thread::current().name())
            });
        }
        let mut out = Vec::new();
        while let Some(r) = set.join_next().await {
            out.push(r.unwrap());
        }
        out.sort();
        for line in out { println!("{line}"); }
    });

    let rt2 = Builder::new_current_thread().enable_all().build().unwrap();
    rt2.block_on(async {
        println!("current_thread runs on {:?}", std::thread::current().name());
    });
}
```

Output:

```
task 0 ran on thread named Some("my-worker")
task 1 ran on thread named Some("my-worker")
task 2 ran on thread named Some("my-worker")
task 3 ran on thread named Some("my-worker")
current_thread runs on Some("main")
```

Worker threads all share the name you set. `current_thread` reuses `main` —
it spawns no threads at all for scheduling.

Pick `multi_thread` for servers (the default, and correct). Pick
`current_thread` for CLIs, tests, single-core targets, and anywhere you want
`!Send` data in tasks (15.3's `spawn_local`).

## worker_threads actually changes parallelism

```rust
use std::collections::BTreeSet;
use tokio::runtime::Builder;

fn threads_used(workers: usize) -> usize {
    let rt = Builder::new_multi_thread().worker_threads(workers).enable_all().build().unwrap();
    rt.block_on(async {
        let mut set = tokio::task::JoinSet::new();
        for _ in 0..workers * 4 {
            set.spawn(async {
                std::thread::sleep(std::time::Duration::from_millis(30));
                format!("{:?}", std::thread::current().id())
            });
        }
        let mut seen = BTreeSet::new();
        while let Some(r) = set.join_next().await {
            seen.insert(r.unwrap());
        }
        seen.len()
    })
}

fn main() {
    println!("1 worker  -> {} distinct threads", threads_used(1));
    println!("4 workers -> {} distinct threads", threads_used(4));
    println!("available_parallelism = {}", std::thread::available_parallelism().unwrap());
}
```

Output (on a 12-core machine):

```
1 worker  -> 1 distinct threads
4 workers -> 4 distinct threads
available_parallelism = 12
```

The default `worker_threads` is `available_parallelism()`. Raising it above core
count does not help — async tasks are not blocked on CPU, so more workers just
add contention. Lower it when you are sharing a box with other processes or
running in a container with a CPU quota the runtime cannot see.

## Builder options worth knowing

```rust
use tokio::runtime::Builder;
use std::time::Duration;

let rt = Builder::new_multi_thread()
    .worker_threads(4)
    .thread_name("api")
    .thread_stack_size(2 * 1024 * 1024)
    .max_blocking_threads(64)
    .thread_keep_alive(Duration::from_secs(10))
    .enable_time()
    .enable_io()
    .build()
    .unwrap();
```

| Option | What it controls |
|---|---|
| `worker_threads(n)` | async worker pool size |
| `max_blocking_threads(n)` | cap on the `spawn_blocking` pool (default 512) |
| `thread_name(s)` / `thread_name_fn(f)` | worker thread names, for profilers |
| `thread_stack_size(n)` | per-worker stack (default 2MB) |
| `thread_keep_alive(d)` | idle timeout for blocking threads |
| `enable_time()` | turns on timers (`sleep`, `timeout`, `interval`) |
| `enable_io()` | turns on the I/O reactor (sockets, `tokio::fs`) |
| `enable_all()` | both of the above |
| `on_thread_start(f)` | hook, e.g. to install thread-local tracing state |

Forget `enable_time()` and a `sleep` panics with the same "no reactor running"
message from 15.1. `enable_all()` is the right default.

## Handle: reaching the runtime from elsewhere

`Handle` is a cheap clonable reference to a running runtime. It is how sync code
deep in a codebase gets back to the runtime that started it.

```rust
use std::time::{Duration, Instant};
use tokio::runtime::{Handle, Runtime};

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let h = Handle::current();
        println!("flavor: {:?}", h.runtime_flavor());
        println!("workers: {}", h.metrics().num_workers());
    });

    let handle = rt.handle().clone();
    std::thread::spawn(move || {
        handle.block_on(async { println!("block_on from another thread") });
    })
    .join()
    .unwrap();

    rt.spawn(async {
        tokio::time::sleep(Duration::from_secs(5)).await;
        println!("never prints");
    });

    let t = Instant::now();
    rt.shutdown_timeout(Duration::from_millis(50));
    println!("shutdown returned after {:?}", t.elapsed());
}
```

Output:

```
flavor: MultiThread
workers: 12
block_on from another thread
shutdown returned after 128.708µs
```

`Handle::current()` panics outside a runtime; `Handle::try_current()` returns a
`Result` if you need to branch.

## Shutdown

Dropping a `Runtime` waits for currently-running tasks to reach their next
`.await` point, then aborts everything pending. Any task still sleeping is
dropped, not completed — that is why `never prints` never printed above.

| Call | Behavior |
|---|---|
| `drop(rt)` | blocks until tasks yield, then aborts the rest |
| `rt.shutdown_timeout(d)` | same, but gives up waiting after `d` |
| `rt.shutdown_background()` | returns immediately, leaks the threads |

`shutdown_timeout` returning in microseconds above shows it did not wait out the
5-second sleep: it aborted it. For real graceful shutdown you need to tell tasks
to finish, not just drop them — lesson 15.4.

Calling `drop(rt)` from inside an async context panics for the same reason as
nested `block_on`. Use `shutdown_background()` there, or restructure.

## Nested block_on panics

```rust
use tokio::runtime::Runtime;

#[tokio::main]
async fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async { println!("inner") });
}
```

Output (intentional panic):

```
thread 'main' (13858396) panicked at .../tokio-1.54.1/src/runtime/scheduler/multi_thread/mod.rs:89:9:
Cannot start a runtime from within a runtime. This happens because a function (like `block_on`) attempted to block the current thread while the thread is being used to drive asynchronous tasks. If you are in an async function, use `.await` on the future instead of blocking on it.
```

The message tells you the fix. This bites most often when a sync helper function
secretly calls `block_on` and someone calls it from a handler. If you must block
from inside async, use `task::block_in_place` or `spawn_blocking` (15.8).

## Feature flags

`tokio` with no features is almost empty — the crate is aggressively
feature-gated so a small binary does not pay for the whole stack.

| Feature | Gives you |
|---|---|
| `rt` | `Runtime`, `spawn`, `current_thread` scheduler |
| `rt-multi-thread` | the multi-threaded scheduler |
| `macros` | `#[tokio::main]`, `#[tokio::test]`, `select!`, `join!` |
| `time` | `sleep`, `timeout`, `interval`, `Instant` |
| `sync` | `mpsc`, `oneshot`, `broadcast`, `watch`, `Mutex`, `Semaphore` |
| `net` | `TcpListener`, `TcpStream`, `UdpSocket` |
| `io-util` | `AsyncReadExt`, `AsyncWriteExt`, `BufReader` |
| `fs` | `tokio::fs` |
| `signal` | `tokio::signal::ctrl_c` |
| `process` | `tokio::process::Command` |
| `full` | all of the above |

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

Use `full` while learning and in applications — the compile-time cost is a few
seconds and you stop fighting missing-symbol errors. In a **library**, enable
only what you use, so your dependents are not forced into the whole stack:

```toml
tokio = { version = "1", default-features = false, features = ["rt", "sync", "time"] }
```

`enable_all()` on the Builder is a *runtime* switch for drivers; the features
are a *compile-time* switch for whether the code exists at all. You need both.

## Two runtimes in one process

Nothing stops you. This is occasionally useful to isolate a latency-sensitive
path from a noisy one.

```rust
use tokio::runtime::Builder;

fn main() {
    let api = Builder::new_multi_thread().worker_threads(4)
        .thread_name("api").enable_all().build().unwrap();
    let bg = Builder::new_multi_thread().worker_threads(1)
        .thread_name("bg").enable_all().build().unwrap();

    bg.spawn(async { /* batch jobs */ });
    api.block_on(async { /* serve */ });
}
```

A future spawned on `api` cannot be awaited on `bg` and vice versa; cross
between them with channels (15.5), never by `block_on`.

## Where the runtime lives in a real backend

In practice: exactly one place, `main`, and nothing else thinks about it.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = connect_db().await?;
    let app = build_router(pool);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

Handlers are `async fn`; they never see a `Runtime`. Library code takes `async
fn` and lets the caller supply the runtime — a library that constructs its own
runtime internally is a bug, because it will nest and panic. Module 17 builds
this server for real.

`main` returning `Result` works with `#[tokio::main]`: the attribute preserves
the return type and `?` propagates out of it.

## Quick Reference

| Item | Use |
|---|---|
| `#[tokio::main]` | wrap `main` in a multi-thread runtime |
| `#[tokio::main(flavor = "current_thread")]` | single-threaded instead |
| `#[tokio::main(worker_threads = N)]` | set pool size inline |
| `#[tokio::test]` | per-test runtime, `current_thread` by default |
| `Runtime::new()` | multi-thread runtime, all drivers on |
| `Builder::new_multi_thread()` | configurable multi-thread |
| `Builder::new_current_thread()` | configurable single-thread |
| `rt.block_on(fut)` | drive a future from sync code |
| `rt.spawn(fut)` | spawn onto that runtime from sync code |
| `Handle::current()` | handle to the ambient runtime (panics if none) |
| `Handle::try_current()` | same, as a `Result` |
| `rt.shutdown_timeout(d)` | bounded shutdown |
| `rt.shutdown_background()` | non-blocking shutdown |

## Common Mistakes

**Calling `block_on` inside async code.** `Cannot start a runtime from within a
runtime`. Use `.await`, or `spawn_blocking` if the callee is genuinely sync.

**Building a runtime inside a library function.** It panics for any caller who
already has one. Expose `async fn` and let `main` own the runtime.

**Forgetting `enable_time()` on a hand-built Builder.** Timers panic at runtime,
not compile time. Use `enable_all()`.

**Creating a `Runtime` per request.** Each one spawns a thread pool. Create one,
clone its `Handle`.

**Assuming more `worker_threads` means more throughput.** Past core count it
only adds scheduling overhead. The fix for a slow server is almost never more
workers (see 15.8).

## Exercises

1. Rewrite a `#[tokio::main]` program using `Builder` + `block_on` explicitly,
   and confirm the output is identical.
2. Build a `current_thread` runtime and print `Handle::current().metrics().num_workers()`.
   Then do the same with `worker_threads(3)`.
3. Build a runtime with only `enable_io()` and call `tokio::time::sleep`. Capture
   the panic, then fix it.
4. Spawn a task that sleeps 2 seconds, then call `shutdown_timeout` with 100ms.
   Print elapsed time and explain what happened to the task.
5. Write a sync `fn config_value() -> String` that calls `block_on`, call it from
   inside an async handler, capture the panic, then fix it with `spawn_blocking`.

---

[← 15.1 async/await and Futures](01-async-await.md) · [Course Index](../README.md) · [15.3 Spawning Tasks and JoinSet →](03-tasks.md)
