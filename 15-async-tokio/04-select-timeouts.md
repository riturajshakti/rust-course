[← Course Index](../README.md)

# 15.4 — select!, Timeouts, and Cancellation

## What You'll Learn

- `tokio::select!` with multiple branches, `biased`, and the `else` arm
- `timeout`, `sleep`, `interval` and how interval handles a missed tick
- Cancellation safety: what happens to the losing branch's half-done work
- `CancellationToken` and a real graceful-shutdown loop

## Coming From Another Language

| Other | Rust |
|---|---|
| `Promise.race([a, b])` | `tokio::select! { a = x => .., b = y => .. }` |
| `AbortController` / `signal` | `CancellationToken`, or just drop the future |
| Go `select { case <-a: ... }` | `tokio::select!` — very close |
| `setTimeout` racing a fetch | `tokio::time::timeout(dur, fut)` |

`select!` is Go's `select` generalized to any future, not just channels. The
difference from `Promise.race` is sharp: in JS the losing promise keeps running.
In Rust the losing future is **dropped**, so its work stops dead.

## Basic select!

Each branch is `pattern = future => expression`. The first future to complete
wins; all others are dropped.

```rust
use tokio::time::{sleep, Duration};

async fn fast() -> &'static str { sleep(Duration::from_millis(20)).await; "fast" }
async fn slow() -> &'static str { sleep(Duration::from_millis(200)).await; "slow" }

#[tokio::main]
async fn main() {
    tokio::select! {
        a = fast() => println!("winner: {a}"),
        b = slow() => println!("winner: {b}"),
    }

    let v = tokio::select! {
        a = fast() => format!("got {a}"),
        _ = sleep(Duration::from_millis(5)) => "timed out".to_string(),
    };
    println!("{v}");
}
```

Output:

```
winner: fast
timed out
```

`select!` is an expression, so every branch must produce the same type — the
second block returns a `String` from both arms.

## Branch order is random by default

With two futures ready at once, Tokio picks a random branch to avoid starving a
branch that is always listed last.

```rust
#[tokio::main]
async fn main() {
    for _ in 0..4 {
        tokio::select! {
            _ = async {} => println!("unbiased picked A"),
            _ = async {} => println!("unbiased picked B"),
        }
    }
    for _ in 0..4 {
        tokio::select! {
            biased;
            _ = async {} => println!("biased picked A"),
            _ = async {} => println!("biased picked B"),
        }
    }
}
```

Output (the unbiased lines differ every run):

```
unbiased picked B
unbiased picked B
unbiased picked A
unbiased picked B
biased picked A
biased picked A
biased picked A
biased picked A
```

`biased;` as the first line makes polling strictly top-to-bottom. Use it when
one branch must take priority — a shutdown signal, or an error channel you want
checked before more work is accepted. It is also slightly faster (no RNG) and
makes tests deterministic. The cost is that a perpetually-ready top branch
starves the ones below it.

## Branch preconditions and else

A branch whose pattern does not match is *disabled* for the rest of that
`select!`. When every branch is disabled, the `else` arm runs; without an `else`
the whole `select!` panics.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<u32>(4);
    tx.send(1).await.unwrap();
    tx.send(2).await.unwrap();
    drop(tx);

    loop {
        tokio::select! {
            biased;
            Some(v) = rx.recv() => println!("got {v}"),
            else => {
                println!("every branch disabled -> else");
                break;
            }
        }
    }
}
```

Output:

```
got 1
got 2
every branch disabled -> else
```

`rx.recv()` returns `None` once the channel is closed and drained; `None` fails
the `Some(v)` pattern, so the branch disables and `else` fires. You can also add
an explicit guard: `v = rx.recv(), if enabled => ...`.

## timeout

`tokio::time::timeout` wraps any future and returns `Result<T, Elapsed>`.

```rust
use tokio::time::{sleep, timeout, Duration, Instant};

async fn job(ms: u64) -> &'static str {
    sleep(Duration::from_millis(ms)).await;
    "finished"
}

#[tokio::main]
async fn main() {
    let r = timeout(Duration::from_millis(100), job(20)).await;
    println!("{r:?}");

    let t = Instant::now();
    let r = timeout(Duration::from_millis(50), job(500)).await;
    println!("{:?} after {:?}", r, t.elapsed());
    println!("err display: {}", r.unwrap_err());

    let r: Result<&str, _> = timeout(Duration::from_millis(50), job(500)).await;
    match r {
        Ok(v) => println!("ok {v}"),
        Err(_elapsed) => println!("deadline passed"),
    }
}
```

Output:

```
Ok("finished")
Err(Elapsed(())) after 52.009042ms
err display: deadline has elapsed
deadline passed
```

Roughly 50ms, not 500 — the inner future was dropped at the deadline. Note the
nesting when the inner future already returns a `Result`:
`timeout(d, f).await` is `Result<Result<T, E>, Elapsed>`, so you end up with
`.await??` or an explicit double match.

`timeout_at(Instant, fut)` takes an absolute deadline instead — better when you
have a request-wide budget to split across several calls.

## sleep and interval

`sleep` waits a duration. `interval` fires repeatedly, and its first tick
completes **immediately**.

```rust
use tokio::time::{interval, sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let t = Instant::now();
    let mut iv = interval(Duration::from_millis(30));
    for i in 0..4 {
        iv.tick().await;
        println!("tick {i} at {:>3}ms", t.elapsed().as_millis());
    }

    let t = Instant::now();
    let mut iv = interval(Duration::from_millis(30));
    iv.tick().await;
    sleep(Duration::from_millis(100)).await;
    iv.tick().await;
    iv.tick().await;
    println!("after a 100ms stall, two ticks took {:>3}ms total", t.elapsed().as_millis());
}
```

Output:

```
tick 0 at   1ms
tick 1 at  32ms
tick 2 at  62ms
tick 3 at  92ms
after a 100ms stall, two ticks took 102ms total
```

Tick 0 at ~0ms is the surprise. The second block shows the default
`MissedTickBehavior::Burst`: after the 100ms stall, three ticks were "due", so
both subsequent `tick()` calls returned instantly rather than spacing out.

| Behavior | On a missed tick |
|---|---|
| `Burst` (default) | fire back-to-back until caught up |
| `Delay` | reset the schedule from now |
| `Skip` | drop missed ticks, resync to the original grid |

```rust
use tokio::time::MissedTickBehavior;
let mut iv = interval(Duration::from_millis(30));
iv.set_missed_tick_behavior(MissedTickBehavior::Skip);
```

`Delay` is usually what you want for polling loops — `Burst` can cause a
thundering herd of catch-up requests after a slow cycle.

`interval_at(start, period)` delays the first tick. `sleep_until(Instant)` is
the absolute version of `sleep`.

## Cancellation safety — the important part

When a `select!` branch loses, its future is dropped mid-flight. Everything that
future had done so far and not yet returned is **lost**. There is no unwinding,
no cleanup callback, no `finally` — just `Drop` on the state machine.

```rust
use tokio::time::{sleep, Duration};

async fn count_to_three(label: &str) -> u32 {
    let mut n = 0;
    for i in 1..=3 {
        sleep(Duration::from_millis(20)).await;
        n = i;
        println!("  {label} reached {n}");
    }
    n
}

#[tokio::main]
async fn main() {
    let result = tokio::select! {
        v = count_to_three("A") => v,
        _ = sleep(Duration::from_millis(50)) => {
            println!("  timer won; A's future is dropped mid-flight");
            0
        }
    };
    println!("result = {result}");
}
```

Output:

```
  A reached 1
  A reached 2
  timer won; A's future is dropped mid-flight
result = 0
```

Two iterations of real progress evaporated. If those iterations had popped items
off a queue, those items are gone. That is the hazard.

A future is **cancel-safe** if dropping it mid-poll loses no observable work.
`tokio::sync::mpsc::Receiver::recv` is cancel-safe by design: a message is only
removed from the channel once `recv` is about to return it, so a cancelled
`recv` leaves the queue untouched.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    tokio::spawn(async move {
        for i in 1..=5 {
            sleep(Duration::from_millis(25)).await;
            let _ = tx.send(i).await;
        }
    });

    let mut got = Vec::new();
    for _ in 0..12 {
        tokio::select! {
            Some(v) = rx.recv() => got.push(v),
            _ = sleep(Duration::from_millis(10)) => {}
        }
    }
    println!("recv is cancel-safe: {got:?}");
}
```

Output:

```
recv is cancel-safe: [1, 2, 3, 4]
```

Twelve `select!` rounds, most of them lost by `recv`, and not one message was
dropped.

| Cancel-safe | Not cancel-safe |
|---|---|
| `mpsc::Receiver::recv` | `AsyncReadExt::read_exact` (partial read lost) |
| `broadcast::Receiver::recv` | `AsyncWriteExt::write_all` (partial write *sent*) |
| `watch::Receiver::changed` | `AsyncBufReadExt::read_line` |
| `Notify::notified` | any multi-step `async fn` you wrote yourself |
| `Mutex::lock`, `Semaphore::acquire` | a DB transaction mid-statement |
| `sleep`, `interval::tick` | |

The rule of thumb: **anything you wrote yourself with two or more `.await`s and
mutable state is not cancel-safe.** The crates document it; `select!`'s own docs
list the std/tokio cases.

Two ways out when you need a non-cancel-safe operation in a `select!`:

1. Hoist the future out and pin it, so the *same* future is re-polled each
   round instead of a fresh one (lesson 15.7 shows `pin!` in a loop).
2. `tokio::spawn` the operation and `select!` on its `JoinHandle` — then losing
   the race only stops you *waiting*, not the work.

## CancellationToken

Dropping a future is abrupt. `CancellationToken` from `tokio-util` is the
cooperative version: tasks poll it and wind down on their own terms.

```rust
use tokio::time::{sleep, Duration};
use tokio_util::sync::CancellationToken;

async fn worker(id: u32, token: CancellationToken) -> (u32, u32) {
    let mut n = 0;
    loop {
        tokio::select! {
            _ = token.cancelled() => return (id, n),
            _ = sleep(Duration::from_millis(20)) => n += 1,
        }
    }
}

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let mut set = tokio::task::JoinSet::new();
    for id in 0..3 {
        set.spawn(worker(id, token.child_token()));
    }
    sleep(Duration::from_millis(70)).await;
    println!("is_cancelled before = {}", token.is_cancelled());
    token.cancel();
    println!("is_cancelled after  = {}", token.is_cancelled());

    let mut done = Vec::new();
    while let Some(r) = set.join_next().await {
        done.push(r.unwrap());
    }
    done.sort();
    for (id, n) in done {
        println!("worker {id} stopped after {n} rounds");
    }
}
```

Output:

```
is_cancelled before = false
is_cancelled after  = true
worker 0 stopped after 3 rounds
worker 1 stopped after 3 rounds
worker 2 stopped after 3 rounds
```

Each worker returned a value — it got to finish its own bookkeeping. Compare
that to `abort()`, which gives you a `JoinError` and nothing else.

| Method | Does |
|---|---|
| `cancel()` | signal; idempotent, wakes all waiters |
| `cancelled()` | future that resolves once cancelled; cancel-safe |
| `cancelled_owned()` | same, but `'static` — for `spawn` |
| `is_cancelled()` | non-blocking check |
| `child_token()` | cancelled when the parent is, or independently |
| `run_until_cancelled(fut)` | `Option<T>`: `None` if cancelled first |
| `drop_guard()` | cancels on drop, for scope-bound lifetimes |

`child_token()` gives a cancellation tree: cancel a subtree without touching
siblings, while a root cancel still reaches everyone.

## Graceful shutdown

The production pattern: one shutdown signal fanned out to every long-lived task,
each task's main loop in a `select!` with it, then wait for all of them to
actually finish.

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

async fn server(name: &'static str, mut shutdown: broadcast::Receiver<()>) -> String {
    let mut served = 0;
    loop {
        tokio::select! {
            _ = shutdown.recv() => {
                sleep(Duration::from_millis(10)).await;
                return format!("{name} drained after {served} requests");
            }
            _ = sleep(Duration::from_millis(15)) => served += 1,
        }
    }
}

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel(1);
    let a = tokio::spawn(server("http", tx.subscribe()));
    let b = tokio::spawn(server("grpc", tx.subscribe()));

    sleep(Duration::from_millis(50)).await;
    println!("SIGTERM: broadcasting shutdown");
    tx.send(()).unwrap();

    let (ra, rb) = tokio::join!(a, b);
    println!("{}", ra.unwrap());
    println!("{}", rb.unwrap());
}
```

Output:

```
SIGTERM: broadcasting shutdown
http drained after 3 requests
grpc drained after 3 requests
```

Both servers ran their 10ms drain *after* the signal — that is the whole point.
Replacing the `sleep(50)` with `tokio::signal::ctrl_c().await` makes it a real
service. `broadcast` and `CancellationToken` are interchangeable here; the token
is nicer for trees of tasks, broadcast for carrying a reason payload.

Timebox the wait so a stuck task cannot hang the process forever:

```rust
let _ = tokio::time::timeout(Duration::from_secs(30), async { tokio::join!(a, b) }).await;
```

## abort vs cancel vs drop

| Way out | Mechanism | Task gets to clean up? |
|---|---|---|
| drop the future | losing `select!` branch, `timeout` expiry | no — `Drop` only |
| `handle.abort()` | runtime cancels at next `.await` | no — `Drop` only |
| `set.abort_all()` | same, whole set | no |
| `token.cancel()` | cooperative, task notices | yes |
| shutdown channel | cooperative, task notices | yes |

Use abort for "I no longer care about the result". Use a token for "stop, but
finish properly".

## Quick Reference

| Item | Use |
|---|---|
| `select! { p = f => e, .. }` | first to finish wins, rest dropped |
| `biased;` | poll branches in order, no RNG |
| `else => e` | all branches disabled |
| `p = f, if cond => e` | per-branch guard |
| `timeout(dur, fut)` | `Result<T, Elapsed>` |
| `timeout_at(instant, fut)` | absolute deadline |
| `sleep(dur)` / `sleep_until(instant)` | one-shot wait |
| `interval(dur)` | repeating; first tick is immediate |
| `iv.set_missed_tick_behavior(..)` | `Burst` / `Delay` / `Skip` |
| `CancellationToken::cancel()` | cooperative stop signal |
| `token.cancelled().await` | wait for it; cancel-safe |
| `token.run_until_cancelled(f)` | `Option<T>` |
| `token.child_token()` | subtree cancellation |
| `handle.abort()` | hard cancel a spawned task |

## Common Mistakes

**Assuming the losing branch keeps running.** It is dropped. This is the single
biggest `select!` footgun for anyone coming from `Promise.race`.

**Putting a non-cancel-safe operation in a `select!` loop.** `read_exact` or
your own multi-step async fn will silently lose partial work on every lost
round. Pin it outside the loop, or spawn it.

**Forgetting `else` on a `select!` in a loop.** Once every branch disables, the
`select!` panics instead of exiting.

**Expecting `interval`'s first tick to wait.** It fires immediately. Use
`interval_at(Instant::now() + period, period)` if you want the delay.

**Treating `abort()` as graceful shutdown.** It drops the task wherever it is.
In-flight requests get no response. Use a token or a shutdown channel.

## Exercises

1. Write a `select!` that races a 100ms sleep against an `mpsc::recv`, in a loop
   that exits after 5 iterations. Send one message at 250ms and confirm it is
   received, not lost.
2. Build an `async fn fetch_all(urls: Vec<&str>) -> Vec<Result<String, String>>`
   where each simulated fetch gets a 50ms `timeout` and a timeout becomes
   `Err("timeout")`.
3. Write an async fn that appends to a `Vec` across three `.await`s, put it in a
   `select!` against a short timer, and show how many elements are lost. Then fix
   it by spawning the fn and selecting on its `JoinHandle`.
4. Build a `CancellationToken` tree: one root, two children, and show that
   cancelling one child leaves the other running while a root cancel stops both.
5. Write a graceful shutdown that waits on `tokio::signal::ctrl_c()`, cancels a
   token, and joins three worker tasks with a 2-second `timeout` around the join.

---

[← Course Index](../README.md)
