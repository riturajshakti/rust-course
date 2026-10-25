[← 15.4 select!, Timeouts, and Cancellation](04-select-timeouts.md) · [Course Index](../README.md) · [15.6 Streams →](06-streams.md)

# 15.5 — Async Channels and Synchronization

## What You'll Learn

- `mpsc` bounded vs unbounded, and what `send().await` buys you
- `oneshot` for a single reply, `broadcast` for fan-out, `watch` for state
- `tokio::sync::Mutex` vs `std::sync::Mutex` — when you actually need the async one
- `RwLock`, `Semaphore` for concurrency limits, and `Notify`

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `make(chan T, 8)` | `tokio::sync::mpsc::channel::<T>(8)` |
| Go `make(chan T)` unbuffered | `mpsc::channel(1)` — closest, not identical |
| `asyncio.Queue` | `mpsc::Receiver` |
| `asyncio.Lock` | `tokio::sync::Mutex` |
| `asyncio.Semaphore` | `tokio::sync::Semaphore` |
| Node `EventEmitter` fan-out | `broadcast` |

These are the async twins of Module 14's `std::sync::mpsc` and `Mutex`: waiting
*yields the task* instead of parking the OS thread. A blocked
`std::sync::mpsc::recv()` in async code stalls a whole worker — see 15.8.

## mpsc: the workhorse

Multi-producer, single-consumer. Clone the sender, never the receiver.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(4);

    tokio::spawn(async move {
        for i in 1..=3 {
            tx.send(format!("msg {i}")).await.unwrap();
        }
    });

    while let Some(m) = rx.recv().await {
        println!("{m}");
    }
    println!("sender dropped -> recv returned None");

    let (tx, mut rx) = mpsc::unbounded_channel::<u32>();
    tx.send(1).unwrap();
    tx.send(2).unwrap();
    drop(tx);
    while let Some(v) = rx.recv().await {
        println!("unbounded {v}");
    }

    let (tx, rx) = mpsc::channel::<u32>(1);
    drop(rx);
    match tx.send(9).await {
        Ok(()) => println!("sent"),
        Err(e) => println!("send error, value recovered: {}", e.0),
    }
}
```

Output:

```
msg 1
msg 2
msg 3
sender dropped -> recv returned None
unbounded 1
unbounded 2
send error, value recovered: 9
```

`recv()` yields `None` once every sender is dropped, which is how `while let`
loops terminate. `unbounded_channel`'s `send` is sync — nothing to wait for. And
`SendError` hands back the value, so nothing is lost.

| Method | Blocks? | Notes |
|---|---|---|
| `send(v).await` | yes, when full | `Err(SendError(v))` if closed |
| `try_send(v)` | no | `Err(Full(v))` or `Err(Closed(v))` |
| `reserve().await` | yes | get a `Permit`, send later without failing |
| `recv().await` | yes | `None` when closed and drained |
| `try_recv()` | no | `Err(Empty)` or `Err(Disconnected)` |
| `recv_many(&mut vec, n).await` | yes | batch up to `n`, returns count |
| `close()` | no | receiver stops accepting, drains what is queued |
| `blocking_send` / `blocking_recv` | yes, the thread | from sync code only |

## Bounded channels give you backpressure

This is the reason to pick bounded. When the buffer is full, `send().await`
suspends the producer until the consumer catches up.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<u32>(2);

    let producer = tokio::spawn(async move {
        for i in 0..5 {
            let t = Instant::now();
            tx.send(i).await.unwrap();
            println!("sent {i} after waiting {:>3}ms", t.elapsed().as_millis());
        }
    });

    let consumer = tokio::spawn(async move {
        while let Some(v) = rx.recv().await {
            sleep(Duration::from_millis(50)).await;
            println!("  consumed {v}");
        }
    });

    let _ = tokio::join!(producer, consumer);
}
```

Output:

```
sent 0 after waiting   0ms
sent 1 after waiting   0ms
sent 2 after waiting   0ms
  consumed 0
sent 3 after waiting  52ms
  consumed 1
sent 4 after waiting  51ms
  consumed 2
  consumed 3
  consumed 4
```

Millisecond values vary; the pattern does not. The first three go straight
through (buffer 2 plus one in-flight), then each subsequent send waits ~50ms for
the consumer. The producer has been throttled to the consumer's rate
automatically.

`unbounded_channel` removes that throttle, so a fast producer grows the queue
until you run out of memory. **Default to bounded.** Unbounded is justified only
when the producer is itself rate-limited and you cannot `.await` at the send
site (e.g. inside a `Drop` impl or a sync callback).

## oneshot: exactly one value

```rust
use tokio::sync::oneshot;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<u32>();
    tokio::spawn(async move {
        sleep(Duration::from_millis(20)).await;
        tx.send(42).unwrap();
    });
    println!("reply = {:?}", rx.await);

    let (tx, rx) = oneshot::channel::<u32>();
    drop(tx);
    println!("sender gone: {:?}", rx.await);

    let (tx, rx) = oneshot::channel::<u32>();
    drop(rx);
    println!("receiver gone: {:?}", tx.send(1));
}
```

Output:

```
reply = Ok(42)
sender gone: Err(RecvError(()))
receiver gone: Err(1)
```

`send` is sync and consumes the sender, so the type enforces "exactly once".
The receiver *is* a future — `rx.await` directly, no `recv()`.
`tx.closed().await` lets the sender detect an abandoned receiver and skip work.

## oneshot + mpsc: the actor pattern

This combination is how you give a task an API. Commands go in over `mpsc`; each
command carries a `oneshot::Sender` for its reply.

```rust
use tokio::sync::{mpsc, oneshot};

enum Cmd {
    Get { key: String, reply: oneshot::Sender<Option<u32>> },
    Set { key: String, val: u32 },
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<Cmd>(8);

    tokio::spawn(async move {
        let mut map = std::collections::HashMap::new();
        while let Some(cmd) = rx.recv().await {
            match cmd {
                Cmd::Set { key, val } => { map.insert(key, val); }
                Cmd::Get { key, reply } => { let _ = reply.send(map.get(&key).copied()); }
            }
        }
    });

    tx.send(Cmd::Set { key: "a".into(), val: 7 }).await.unwrap();

    let (rtx, rrx) = oneshot::channel();
    tx.send(Cmd::Get { key: "a".into(), reply: rtx }).await.unwrap();
    println!("get a -> {:?}", rrx.await.unwrap());

    let (rtx, rrx) = oneshot::channel();
    tx.send(Cmd::Get { key: "zz".into(), reply: rtx }).await.unwrap();
    println!("get zz -> {:?}", rrx.await.unwrap());
}
```

Output:

```
get a -> Some(7)
get zz -> None
```

The `HashMap` is a plain local — no `Mutex`, no `Arc`, no contention, because
exactly one task owns it. This is usually a better answer than shared mutable
state, and the idiomatic Rust shape for "stateful thing many callers talk to".

## broadcast: fan-out, with a lag error

Every receiver sees every message. The buffer is a ring, so a receiver that
falls behind loses the oldest messages and is told so.

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, mut r1) = broadcast::channel::<u32>(4);
    let mut r2 = tx.subscribe();

    println!("receivers = {}", tx.receiver_count());
    tx.send(1).unwrap();
    tx.send(2).unwrap();
    println!("r1 {:?} {:?}", r1.recv().await, r1.recv().await);
    println!("r2 {:?} {:?}", r2.recv().await, r2.recv().await);

    for i in 0..6 {
        tx.send(i).unwrap();
    }
    println!("slow receiver: {:?}", r1.recv().await);
    println!("then          {:?}", r1.recv().await);

    drop(tx);
    println!("after drop: {:?}", r2.recv().await);
}
```

Output:

```
receivers = 2
r1 Ok(1) Ok(2)
r2 Ok(1) Ok(2)
slow receiver: Err(Lagged(2))
then          Ok(2)
after drop: Err(Lagged(2))
```

Six messages into a buffer of four, so the first two were overwritten:
`Err(Lagged(2))` says "you missed 2". **`Lagged` is not fatal** — the next
`recv()` succeeds and resumes from the oldest surviving message, as the `Ok(2)`
shows. Always match on it rather than `unwrap`:

```rust
loop {
    match rx.recv().await {
        Ok(v) => handle(v),
        Err(broadcast::error::RecvError::Lagged(n)) => {
            tracing::warn!("dropped {n} messages");
        }
        Err(broadcast::error::RecvError::Closed) => break,
    }
}
```

`send` returns `Err` only when there are zero receivers — and it returns the
count of receivers on success. A subscriber that joins later sees only messages
sent after it subscribed.

Use broadcast for shutdown signals, config reloads, and pub/sub to connected
clients. Do not use it as a work queue: every consumer gets every item.

## watch: current state, not a log

`watch` holds one value. Receivers do not get history, only "the value changed,
here is the latest". Perfect for config and health state.

```rust
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

#[derive(Debug, Clone, PartialEq)]
enum State { Starting, Ready, Draining }

#[tokio::main]
async fn main() {
    let (tx, rx) = watch::channel(State::Starting);
    println!("current = {:?}", *rx.borrow());

    let watcher = tokio::spawn({
        let mut rx = rx.clone();
        async move {
            while rx.changed().await.is_ok() {
                let s = rx.borrow_and_update().clone();
                println!("watcher saw {s:?}");
                if s == State::Draining { break; }
            }
        }
    });

    sleep(Duration::from_millis(10)).await;
    tx.send(State::Ready).unwrap();
    sleep(Duration::from_millis(10)).await;
    tx.send(State::Draining).unwrap();
    watcher.await.unwrap();

    println!("final = {:?}", *rx.borrow());
    println!("receiver_count = {}", tx.receiver_count());
}
```

Output:

```
current = Starting
watcher saw Ready
watcher saw Draining
final = Draining
receiver_count = 1
```

`borrow()` reads without consuming a change notification; `borrow_and_update()`
reads *and* marks it seen, which is what you want inside a `changed()` loop.
Never hold the `Ref` from `borrow()` across an `.await` — it blocks senders.

If you write the value 10 times while a receiver is busy, it wakes once and sees
the 10th. That coalescing is the feature.

| Method | Does |
|---|---|
| `tx.send(v)` | set value, wake receivers; `Err` if no receivers |
| `tx.send_replace(v)` | set and return the old value |
| `tx.send_if_modified(f)` | only notify if the closure reports a change |
| `tx.subscribe()` | new receiver, starts unmarked |
| `rx.changed().await` | wait for a new value; cancel-safe |
| `rx.borrow()` / `rx.borrow_and_update()` | read current |
| `rx.wait_for(pred).await` | wait until the value satisfies a predicate |

## tokio::sync::Mutex vs std::sync::Mutex

The async `Mutex` is **slower** than the std one and you should reach for it
less often than you think. Its only job is to let you hold a lock across an
`.await`.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0u32));
    let mut set = tokio::task::JoinSet::new();
    for _ in 0..4 {
        let c = Arc::clone(&counter);
        set.spawn(async move {
            let mut g = c.lock().await;
            sleep(Duration::from_millis(10)).await;
            *g += 1;
        });
    }
    while set.join_next().await.is_some() {}
    println!("counter = {}", *counter.lock().await);

    let m = Mutex::new(5);
    println!("try_lock while free: {:?}", m.try_lock().map(|g| *g));
    let g = m.lock().await;
    println!("try_lock while held: {:?}", m.try_lock().is_err());
    drop(g);
    println!("into_inner = {}", m.into_inner());
}
```

Output:

```
counter = 4
try_lock while free: Ok(5)
try_lock while held: true
into_inner = 5
```

`lock()` is `.await`ed and never poisons, unlike `std::sync::Mutex`. Try the
same code with the std `Mutex` and the compiler stops you:

```rust
use std::sync::{Arc, Mutex};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let c = Arc::new(Mutex::new(0u32));
    let c2 = Arc::clone(&c);
    tokio::spawn(async move {
        let mut g = c2.lock().unwrap();
        sleep(Duration::from_millis(10)).await;
        *g += 1;
    })
    .await
    .unwrap();
}
```

Does not compile (intentional):

```
error: future cannot be sent between threads safely
   --> src/main.rs:8:5
    |
  8 | /     tokio::spawn(async move {
...
    | |______^ future created by async block is not `Send`
    |
    = help: within `{async block@src/main.rs:8:18: 8:28}`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, u32>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:42
    |
  9 |         let mut g = c2.lock().unwrap();
    |             ----- has type `std::sync::MutexGuard<'_, u32>` which is not `Send`
 10 |         sleep(Duration::from_millis(10)).await;
    |                                          ^^^^^ await occurs here, with `mut g` maybe used later
```

That error is the design question: do you *need* to hold it across the await?
Usually no — scope the guard tighter and the std `Mutex` is fine in async code:

```rust
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let c = Arc::new(Mutex::new(0u32));
    let mut set = tokio::task::JoinSet::new();
    for _ in 0..4 {
        let c = Arc::clone(&c);
        set.spawn(async move {
            {
                let mut g = c.lock().unwrap();
                *g += 1;
            }
            tokio::time::sleep(tokio::time::Duration::from_millis(5)).await;
        });
    }
    while set.join_next().await.is_some() {}
    println!("std Mutex works fine: {}", *c.lock().unwrap());
}
```

Output:

```
std Mutex works fine: 4
```

**Decision rule:**

| Situation | Lock |
|---|---|
| short critical section, no `.await` inside | `std::sync::Mutex` |
| guard must live across an `.await` (DB call, I/O while holding) | `tokio::sync::Mutex` |
| a counter | `AtomicUsize` (14.5), no lock |
| per-task state, no sharing | no lock — use the actor pattern above |
| mostly reads | `RwLock` |

Even with `tokio::sync::Mutex`, holding a lock across a slow `.await`
serializes your server on that lock — the counter example above took four
sequential 10ms turns. That is a design smell, not a solved problem.

## RwLock

Many readers or one writer. Readers genuinely overlap.

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use tokio::time::{sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let lock = Arc::new(RwLock::new(vec![1, 2, 3]));

    let t = Instant::now();
    let mut set = tokio::task::JoinSet::new();
    for id in 0..3 {
        let l = Arc::clone(&lock);
        set.spawn(async move {
            let g = l.read().await;
            sleep(Duration::from_millis(40)).await;
            (id, g.len())
        });
    }
    let mut out = Vec::new();
    while let Some(r) = set.join_next().await { out.push(r.unwrap()); }
    out.sort();
    println!("{out:?} — three readers in {}ms total", t.elapsed().as_millis());

    lock.write().await.push(4);
    println!("after write: {:?}", *lock.read().await);
    println!("try_write while reading: {}", {
        let _g = lock.read().await;
        lock.try_write().is_err()
    });
}
```

Output:

```
[(0, 3), (1, 3), (2, 3)] — three readers in 42ms total
after write: [1, 2, 3, 4]
try_write while reading: true
```

~42ms, not ~120ms: the readers overlapped. Tokio's `RwLock` is write-preferring
— a waiting writer blocks new readers, so writers cannot be starved.
`read_owned()` / `write_owned()` give `'static` guards movable into a task.

## Semaphore: concurrency limiting

The tool for "at most N of these at once" — the thing you reach for when a
downstream API rate-limits you or a connection pool has 10 slots.

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;
use tokio::time::{sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let sem = Arc::new(Semaphore::new(2));
    let t = Instant::now();
    let mut set = tokio::task::JoinSet::new();

    for id in 0..6 {
        let sem = Arc::clone(&sem);
        set.spawn(async move {
            let _permit = sem.acquire().await.unwrap();
            sleep(Duration::from_millis(50)).await;
            (id, t.elapsed().as_millis() / 50)
        });
    }
    let mut out = Vec::new();
    while let Some(r) = set.join_next().await { out.push(r.unwrap()); }
    out.sort();
    for (id, wave) in &out {
        println!("request {id} finished in wave {wave}");
    }
    println!("6 requests, limit 2, total {}ms", t.elapsed().as_millis());
    println!("available_permits = {}", sem.available_permits());

    let sem = Semaphore::new(1);
    let p = sem.try_acquire().unwrap();
    println!("try_acquire when exhausted: {:?}", sem.try_acquire().is_err());
    drop(p);
    println!("after drop: {}", sem.available_permits());
}
```

Output:

```
request 0 finished in wave 1
request 1 finished in wave 1
request 2 finished in wave 2
request 3 finished in wave 2
request 4 finished in wave 3
request 5 finished in wave 3
6 requests, limit 2, total 158ms
available_permits = 2
try_acquire when exhausted: true
after drop: 1
```

Three waves of two: all six spawned instantly, only two ran at a time. The
permit releases on drop, so there is nothing to call. Bind it to `_permit`, not
`_` — plain `_` drops immediately and the limit does nothing.

| Method | Does |
|---|---|
| `acquire().await` | one permit; `Err` if the semaphore was closed |
| `acquire_many(n).await` | n permits atomically |
| `try_acquire()` | non-blocking |
| `acquire_owned().await` | `'static` permit, movable into a task |
| `add_permits(n)` | raise the limit at runtime |
| `close()` | wake all waiters with an error; no new permits |
| `available_permits()` | current count |

`Semaphore::new(1)` is a mutex guarding nothing — useful for serializing a side
effect with no shared data.

## Notify: a bare wakeup

No value, no count — just "wake up". Use it when the state lives elsewhere and
you only need the nudge.

```rust
use std::sync::Arc;
use tokio::sync::Notify;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let n = Arc::new(Notify::new());

    let n2 = Arc::clone(&n);
    let waiter = tokio::spawn(async move {
        n2.notified().await;
        println!("woken once");
    });
    sleep(Duration::from_millis(10)).await;
    n.notify_one();
    waiter.await.unwrap();

    let n = Arc::new(Notify::new());
    let mut set = tokio::task::JoinSet::new();
    for id in 0..3 {
        let n = Arc::clone(&n);
        set.spawn(async move {
            n.notified().await;
            id
        });
    }
    sleep(Duration::from_millis(10)).await;
    n.notify_waiters();
    let mut woken = Vec::new();
    while let Some(r) = set.join_next().await { woken.push(r.unwrap()); }
    woken.sort();
    println!("notify_waiters woke {woken:?}");
}
```

Output:

```
woken once
notify_waiters woke [0, 1, 2]
```

The asymmetry to remember: `notify_one()` stores a permit if nobody is waiting,
so a later `notified()` returns immediately. `notify_waiters()` stores nothing,
so a task calling `notified()` afterwards waits for the *next* notification —
a classic missed-wakeup source. Prefer `watch` when the thing is really state.

## Channel comparison

| Type | Senders | Receivers | Delivery | Buffer | Use for |
|---|---|---|---|---|---|
| `mpsc` | many | **one** | each message to the one receiver | bounded or unbounded | work queues, task pipelines |
| `oneshot` | one, once | one | exactly one value | 1 | request/reply, task result |
| `broadcast` | many | many | **every** receiver gets **every** message | bounded ring, drops oldest | shutdown, config reload, pub/sub |
| `watch` | many | many | latest value only, coalesced | 1 | current state, health, config |

`Barrier` rounds out the set: N tasks wait for each other, then all proceed.

Decision shortcut: **state** -> `watch`; **events every listener needs** ->
`broadcast`; **work exactly one worker should do** -> `mpsc`; **one reply** ->
`oneshot`; **a limit** -> `Semaphore`.

## Quick Reference

| Call | Returns |
|---|---|
| `mpsc::channel(n)` | `(Sender<T>, Receiver<T>)`, bounded |
| `mpsc::unbounded_channel()` | `(UnboundedSender<T>, UnboundedReceiver<T>)` |
| `tx.send(v).await` | `Result<(), SendError<T>>` |
| `rx.recv().await` | `Option<T>` |
| `rx.recv_many(&mut v, n).await` | `usize` |
| `oneshot::channel()` | `(Sender<T>, Receiver<T>)` |
| `rx.await` (oneshot) | `Result<T, RecvError>` |
| `broadcast::channel(n)` | `(Sender<T>, Receiver<T>)` |
| `rx.recv().await` (broadcast) | `Result<T, RecvError>` — `Lagged(n)` or `Closed` |
| `watch::channel(init)` | `(Sender<T>, Receiver<T>)` |
| `rx.changed().await` | `Result<(), RecvError>` |
| `Mutex::lock().await` | `MutexGuard<T>` |
| `RwLock::read().await` / `.write().await` | `RwLockReadGuard` / `RwLockWriteGuard` |
| `Semaphore::acquire().await` | `Result<SemaphorePermit, AcquireError>` |
| `Notify::notified().await` | `()` |

## Common Mistakes

**Reaching for `unbounded_channel` to avoid thinking about buffer size.** You
have traded a bounded queue for unbounded memory growth. Pick a bound.

**`unwrap()` on `broadcast::recv()`.** `Lagged` is normal under load and is
recoverable. Match on it.

**Using `tokio::sync::Mutex` everywhere "because async".** It is slower. If no
`.await` is inside the critical section, `std::sync::Mutex` is correct.

**Holding a `watch::Ref` or a lock guard across an `.await`.** It blocks every
other party for the duration of your network call.

**Cloning the `Receiver` of an `mpsc`.** It is single-consumer and does not
clone. For multiple workers off one queue, wrap it in `Arc<Mutex<Receiver>>`, or
restructure to one receiver that fans out to spawned tasks.

## Exercises

1. Build a bounded `mpsc(1)` pipeline with a producer sending 10 items and a
   consumer sleeping 20ms each. Print elapsed time per send to show the
   backpressure.
2. Write an actor task owning a `Vec<String>` with `Push(String)` and
   `Len(oneshot::Sender<usize>)` commands. Drive it from three concurrent tasks.
3. Make a `broadcast(2)` channel, send 5 messages before reading, then read until
   `Closed`, printing each `Lagged(n)` you hit.
4. Use a `watch` channel to hold an `Option<Config>`. Spawn two tasks that log
   every change. Send three updates and confirm both see all three.
5. Limit 20 simulated HTTP calls (50ms each) to 4 concurrent with a `Semaphore`.
   Print total elapsed and verify it is roughly 250ms, not 1000ms.

---

[← 15.4 select!, Timeouts, and Cancellation](04-select-timeouts.md) · [Course Index](../README.md) · [15.6 Streams →](06-streams.md)
