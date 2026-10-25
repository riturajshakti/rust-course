[← 15.7 Async Traits and Pinning](07-async-traits-pin.md) · [Course Index](../README.md) · [16.1 Declarative Macros with macro_rules! →](../16-macros-unsafe/01-macro-rules.md)

# 15.8 — Blocking Code in Async Contexts

## What You'll Learn

- Why one blocking call stalls unrelated tasks, proven by timing
- `spawn_blocking`, `block_in_place`, and when each applies
- `rayon` for CPU-bound work inside an async service
- A decision table for "where does this work belong"

## Coming From Another Language

| Other | Rust |
|---|---|
| Node: blocking the event loop stalls everything | same, per worker thread |
| Node `worker_threads` for CPU work | `spawn_blocking` / `rayon` |
| Go: the runtime detects blocking syscalls and reschedules | Tokio does **not** — you must tell it |
| Python asyncio `run_in_executor` | `tokio::task::spawn_blocking` |

Go is the dangerous comparison. Go's scheduler notices a goroutine making a
blocking syscall and spins up another OS thread. Tokio has no such mechanism: a
task that does not reach an `.await` owns its worker thread until it does.

## The demonstration

One `current_thread` runtime, two tasks. One does a `std::thread::sleep` — the
sync, thread-blocking kind.

```rust
use tokio::time::{sleep, Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let t = Instant::now();
    let a = tokio::spawn(async move {
        std::thread::sleep(Duration::from_millis(300));
        println!("blocker finished at {:>3}ms", t.elapsed().as_millis());
    });
    let b = tokio::spawn(async move {
        sleep(Duration::from_millis(10)).await;
        println!("quick task finished at {:>3}ms", t.elapsed().as_millis());
    });
    let _ = tokio::join!(a, b);
    println!("total {:>3}ms", t.elapsed().as_millis());
}
```

Output:

```
blocker finished at 305ms
quick task finished at 317ms
total 317ms
```

The quick task asked to wake at 10ms. It woke at **317ms** — a 30x latency
miss, caused entirely by an unrelated task. Its timer fired on schedule; there
was simply no thread free to run it.

Change one line — `std::thread::sleep` to `tokio::time::sleep` — and nothing
else:

```rust
use tokio::time::{sleep, Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let t = Instant::now();
    let a = tokio::spawn(async move {
        sleep(Duration::from_millis(300)).await;
        println!("slow task finished at {:>3}ms", t.elapsed().as_millis());
    });
    let b = tokio::spawn(async move {
        sleep(Duration::from_millis(10)).await;
        println!("quick task finished at {:>3}ms", t.elapsed().as_millis());
    });
    let _ = tokio::join!(a, b);
    println!("total {:>3}ms", t.elapsed().as_millis());
}
```

Output:

```
quick task finished at  12ms
slow task finished at 302ms
total 302ms
```

12ms instead of 317ms. Exact milliseconds vary per run; the gap does not.
`tokio::time::sleep` yields the thread; `std::thread::sleep` parks it.

**`std::thread::sleep` in an async fn is always a bug.** It is the easiest
version of this mistake to spot and the easiest to fix.

## The multi-thread runtime only delays the problem

"We have 12 workers, one blocking task is fine." It is, until there are 12.

```rust
use tokio::time::{sleep, Duration, Instant};

#[tokio::main(worker_threads = 2)]
async fn main() {
    let t = Instant::now();
    let mut set = tokio::task::JoinSet::new();
    for _ in 0..2 {
        set.spawn(async move {
            std::thread::sleep(Duration::from_millis(300));
            None
        });
    }
    for i in 0..3 {
        set.spawn(async move {
            sleep(Duration::from_millis(10)).await;
            Some((i, t.elapsed().as_millis()))
        });
    }
    let mut out = Vec::new();
    while let Some(r) = set.join_next().await {
        if let Some(x) = r.unwrap() { out.push(x); }
    }
    out.sort();
    for (i, ms) in out {
        println!("quick task {i} wanted 10ms, actually ran at {ms}ms");
    }
    println!("total {}ms", t.elapsed().as_millis());
}
```

Output:

```
quick task 0 wanted 10ms, actually ran at 305ms
quick task 1 wanted 10ms, actually ran at 305ms
quick task 2 wanted 10ms, actually ran at 305ms
total 317ms
```

Two blocking tasks, two workers, total starvation. Under production load this is
exactly what happens: concurrency scales up, every worker lands on the blocking
path at once, and p99 latency goes vertical while CPU sits near idle. The metric
signature — high latency, low CPU, no queue depth on the DB — is the giveaway.

## spawn_blocking

Moves a sync closure to a separate, dynamically-sized pool (default max 512
threads) so it cannot touch the async workers.

```rust
use tokio::time::{sleep, Duration, Instant};

#[tokio::main(worker_threads = 2)]
async fn main() {
    let t = Instant::now();
    let mut set = tokio::task::JoinSet::new();
    for _ in 0..2 {
        set.spawn_blocking(move || {
            std::thread::sleep(Duration::from_millis(300));
            None
        });
    }
    for i in 0..3 {
        set.spawn(async move {
            sleep(Duration::from_millis(10)).await;
            Some((i, t.elapsed().as_millis()))
        });
    }
    let mut out = Vec::new();
    while let Some(r) = set.join_next().await {
        if let Some(x) = r.unwrap() { out.push(x); }
    }
    out.sort();
    for (i, ms) in out {
        println!("quick task {i} ran at {ms}ms");
    }
    println!("total {}ms", t.elapsed().as_millis());
}
```

Output:

```
quick task 0 ran at 12ms
quick task 1 ran at 12ms
quick task 2 ran at 12ms
total 307ms
```

12ms again, and the blocking work still completed. One method call, problem
gone.

It returns a `JoinHandle`, so it composes with everything from 15.3.

```rust
use tokio::task;

fn expensive(n: u64) -> u64 {
    (1..=n).map(|x| x * x).sum()
}

#[tokio::main]
async fn main() {
    let h = task::spawn_blocking(|| expensive(2_000_000));
    println!("sum = {}", h.await.unwrap());

    let h = task::spawn_blocking(|| -> Result<String, std::io::Error> {
        std::fs::read_to_string("Cargo.toml")
    });
    let text = h.await.unwrap().unwrap();
    println!("first line: {}", text.lines().next().unwrap());

    let h = task::spawn_blocking(|| panic!("sync panic"));
    println!("panic surfaced as JoinError: {}", h.await.unwrap_err().is_panic());
}
```

Output:

```
sum = 2666668666667000000
first line: [package]

thread 'tokio-rt-worker' (13970708) panicked at src/main.rs:18:37:
sync panic
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
panic surfaced as JoinError: true
```

Note the double `unwrap` on the middle one: `JoinError` first, then the
`io::Error`.

Constraints: the closure must be `FnOnce + Send + 'static`, so it cannot borrow
locals — move clones in. It cannot `.await` anything; if you need async inside,
restructure or use a `Handle` to `block_on` (that is legal from a blocking
thread, since it is not a worker).

The pool is capped by `max_blocking_threads`. Blocking tasks beyond the cap queue
up, so `spawn_blocking` is not a licence for unbounded blocking work — put a
`Semaphore` in front if the volume is high.

## block_in_place

Tells the multi-thread runtime "I am about to block this worker; move my sibling
tasks elsewhere first." The closure runs inline, so it can borrow locals and
needs no `'static`.

```rust
use tokio::task;
use tokio::time::{Duration, Instant};

#[tokio::main]
async fn main() {
    let t = Instant::now();
    let v = task::block_in_place(|| {
        std::thread::sleep(Duration::from_millis(100));
        "done inline"
    });
    println!("{v} at {}ms, no closure move needed", t.elapsed().as_millis());

    let local = String::from("borrowed by reference");
    task::block_in_place(|| {
        std::thread::sleep(Duration::from_millis(10));
        println!("{local}");
    });
}
```

Output:

```
done inline at 105ms, no closure move needed
borrowed by reference
```

It is multi-thread only:

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    tokio::task::block_in_place(|| println!("hi"));
}
```

Output (intentional panic):

```
thread 'main' (13971189) panicked at src/main.rs:3:5:
can call blocking only when running on the multi-threaded runtime
```

| | `spawn_blocking` | `block_in_place` |
|---|---|---|
| Runs on | separate blocking pool | the current worker thread |
| Borrowing locals | no, needs `'static` | yes |
| Returns | `JoinHandle<T>` | the value directly |
| Concurrency with other work | yes, fully | the current task waits |
| Runtime flavor | any | multi-thread only |
| Cost | thread handoff | migrates the worker's other tasks |

Use `spawn_blocking` by default. Reach for `block_in_place` when the closure
must borrow, or when you are already in a function that cannot be made async.
Calling it in a tight loop is worse than useless — each call forces a worker
reshuffle.

## CPU-bound work: rayon

`spawn_blocking` keeps CPU work off the async workers but still runs it on one
thread. For genuinely parallel compute, hand it to `rayon` and bridge back with a
`oneshot`.

```rust
use rayon::prelude::*;
use tokio::sync::oneshot;
use tokio::time::Instant;

fn hash_all(data: &[u64]) -> u64 {
    data.iter().map(|&x| (x.wrapping_mul(2654435761)) % 1_000_003).sum()
}

fn hash_all_parallel(data: &[u64]) -> u64 {
    data.par_iter().map(|&x| (x.wrapping_mul(2654435761)) % 1_000_003).sum()
}

#[tokio::main]
async fn main() {
    let data: Vec<u64> = (0..8_000_000).collect();

    let t = Instant::now();
    let a = tokio::task::spawn_blocking({
        let d = data.clone();
        move || hash_all(&d)
    })
    .await
    .unwrap();
    println!("spawn_blocking serial   -> {a} in {}ms", t.elapsed().as_millis());

    let t = Instant::now();
    let (tx, rx) = oneshot::channel();
    rayon::spawn(move || {
        let _ = tx.send(hash_all_parallel(&data));
    });
    let b = rx.await.unwrap();
    println!("rayon::spawn + oneshot  -> {b} in {}ms", t.elapsed().as_millis());
}
```

Output:

```
spawn_blocking serial   -> 4000009339307 in 48ms
rayon::spawn + oneshot  -> 4000009339307 in 6ms
```

Roughly 8x on a 12-core machine; the exact numbers vary with hardware and load.
`rayon::spawn` returns immediately, the `oneshot` gives you an awaitable result,
and rayon's own pool handles the parallelism. Never call `par_iter()` directly on
a Tokio worker — rayon's work-stealing would block that worker.

## std::fs vs tokio::fs

`tokio::fs` exists, but understand what it is: every call is a `spawn_blocking`
around the std call. There is no async filesystem syscall on Linux or macOS for
Tokio to use.

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    tokio::fs::write("/tmp/demo15.txt", b"alpha\nbeta\ngamma\n").await?;
    let text = tokio::fs::read_to_string("/tmp/demo15.txt").await?;
    println!("{} bytes, {} lines", text.len(), text.lines().count());

    let f = tokio::fs::File::open("/tmp/demo15.txt").await?;
    let mut lines = BufReader::new(f).lines();
    while let Some(l) = lines.next_line().await? {
        println!("line: {l}");
    }

    let mut f = tokio::fs::File::create("/tmp/demo15b.txt").await?;
    f.write_all(b"written async").await?;
    f.flush().await?;
    println!("{:?}", tokio::fs::metadata("/tmp/demo15b.txt").await?.len());

    println!("{:?}", tokio::fs::read_to_string("/tmp/does-not-exist-15").await.map_err(|e| e.kind()));

    tokio::fs::remove_file("/tmp/demo15.txt").await?;
    tokio::fs::remove_file("/tmp/demo15b.txt").await?;
    Ok(())
}
```

Output:

```
17 bytes, 3 lines
line: alpha
line: beta
line: gamma
13
Err(NotFound)
```

Practical guidance:

- Config read once at startup, before the server accepts traffic: `std::fs` is
  fine and simpler.
- File I/O in a request handler: `tokio::fs`, or one `spawn_blocking` wrapping
  several `std::fs` calls — one handoff beats five.
- A loop over 10,000 small files: definitely one `spawn_blocking` around the
  whole loop. `tokio::fs` would pay the handoff 10,000 times.
- Always remember to `flush()` a `tokio::fs::File` before dropping it; unlike
  the std version, drop cannot flush because flushing is async.

## What counts as blocking

Anything that does not reach an `.await` for a long time. The rough threshold:
**10 to 100 microseconds** of uninterrupted work is fine; a millisecond is
suspicious; 10ms is a problem.

| Blocking | The async version |
|---|---|
| `std::thread::sleep` | `tokio::time::sleep` |
| `std::fs::*` | `tokio::fs::*`, or `spawn_blocking` |
| `std::net::TcpStream` | `tokio::net::TcpStream` |
| `reqwest::blocking` | `reqwest` async |
| `std::process::Command::output` | `tokio::process::Command` |
| `rusqlite`, `diesel` (sync by design) | `spawn_blocking`, or `sqlx` (async) |
| `std::io::stdin().read_line` | `tokio::io::stdin` + `BufReader::lines` |
| `std::sync::Mutex::lock` under contention | `tokio::sync::Mutex` (15.5) |
| image resize, compression, hashing, big sorts | `spawn_blocking` or `rayon` |
| `serde_json::from_str` on 1KB | nothing — it is microseconds |
| `serde_json::from_str` on 50MB | `spawn_blocking` |

Note the last pair: the same function is fine or not depending on input size.
"Is this blocking?" is a question about duration, not about which crate it came
from.

## Detecting it

The cheap signal: latency climbs while CPU stays low and downstream services
look healthy. The async workers are idle-but-occupied.

`tokio-console` is the proper tool. It is a terminal UI showing every task, its
poll count, total poll time, and — the key column — a **warning on tasks whose
average poll time is high**, which is precisely the definition of blocking.

```toml
[dependencies]
console-subscriber = "0.4"
tokio = { version = "1", features = ["full", "tracing"] }
```

```rust
#[tokio::main]
async fn main() {
    console_subscriber::init();
    // ... your app
}
```

Build with `RUSTFLAGS="--cfg tokio_unstable"`, run `tokio-console` in another
terminal, and sort by poll time. A task with a 300ms average poll is your
culprit.

Also available: `Handle::metrics()` for programmatic counters
(`num_workers`, `num_alive_tasks`, `global_queue_depth`), and
`Builder::on_thread_park` hooks if you want to build your own detector.

## Where does this work belong

| Work | Put it |
|---|---|
| Network I/O (HTTP, DB over async driver, sockets) | plain `async` on a worker |
| Timers, channels, locks | plain `async` on a worker |
| Sub-100µs sync compute (parse a small JSON, hash a token) | inline, do nothing |
| Sync DB driver call (`rusqlite`, `diesel`) | `spawn_blocking` |
| File I/O in a handler | `tokio::fs`, or `spawn_blocking` for a batch |
| Single-threaded CPU task, 1ms-1s (resize, compress) | `spawn_blocking` |
| Parallelizable CPU task (big map/sort/aggregate) | `rayon` + `oneshot` |
| Long CPU task that must borrow locals | `block_in_place` (multi-thread only) |
| Minutes-long batch job | a separate process or worker service |
| A `block_on` you cannot remove from sync code | `spawn_blocking`, then `Handle::block_on` inside |

Two rules of thumb worth memorizing:

1. **Between two `.await`s, do microseconds of work.** If a stretch of code can
   take milliseconds, it belongs on a blocking thread.
2. **`spawn_blocking` is cheap; being wrong about blocking is expensive.** When
   unsure, wrap it. The cost is a thread handoff of a few microseconds.

## Quick Reference

| Call | Use |
|---|---|
| `task::spawn_blocking(f)` | sync work on the blocking pool; `JoinHandle<T>` |
| `set.spawn_blocking(f)` | same, into a `JoinSet` |
| `task::block_in_place(f)` | block the current worker, borrowing allowed |
| `rayon::spawn(f)` + `oneshot` | parallel CPU work, awaitable result |
| `Handle::current().block_on(f)` | await from a blocking thread (not a worker) |
| `Builder::max_blocking_threads(n)` | cap the blocking pool (default 512) |
| `tokio::fs::*` | `spawn_blocking` wrappers around `std::fs` |
| `tokio::time::sleep` | never `std::thread::sleep` in async |
| `console_subscriber::init()` | enable `tokio-console` |
| `Handle::metrics()` | runtime counters |

## Common Mistakes

**`std::thread::sleep` inside an async fn.** The canonical version of this bug.
Always `tokio::time::sleep`.

**Assuming the multi-thread runtime makes it safe.** It raises the number of
concurrent blocking calls you survive, and production will exceed it.

**`task::yield_now()` to "fix" a CPU-heavy loop.** Yielding between iterations
helps a little but still burns a worker on compute. Use `spawn_blocking` or
`rayon`.

**`block_in_place` on `current_thread`.** It panics. The flavor matters.

**`tokio::fs` inside a loop over thousands of files.** Each call is a
`spawn_blocking`. Wrap the whole loop in one instead.

**Forgetting to `flush()` a `tokio::fs::File`.** The std version flushes on
drop; the async one cannot, so writes can be silently lost.

## Exercises

1. Reproduce the stall: a `current_thread` runtime with one task doing
   `std::thread::sleep(500ms)` and five tasks sleeping 10ms. Print each task's
   actual wake time. Then fix it two ways — `tokio::time::sleep` and
   `spawn_blocking` — and compare timings.
2. Write a CPU-bound `fn count_primes(n: u64) -> usize` and call it three ways:
   directly in an async fn, via `spawn_blocking`, and via `rayon::spawn` with a
   `oneshot`. Time all three while a 10ms heartbeat task logs.
3. Read 500 small files, once with `tokio::fs::read_to_string` in a loop and
   once with one `spawn_blocking` wrapping a `std::fs` loop. Compare.
4. Call `block_in_place` on a `current_thread` runtime, capture the panic, then
   make it work by switching flavors.
5. Add `console_subscriber` to a program with one deliberately blocking task,
   run `tokio-console`, and find the task by its poll time.

---

[← 15.7 Async Traits and Pinning](07-async-traits-pin.md) · [Course Index](../README.md) · [16.1 Declarative Macros with macro_rules! →](../16-macros-unsafe/01-macro-rules.md)
