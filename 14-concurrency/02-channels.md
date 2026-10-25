[← Course Index](../README.md)

# 14.2 — Channels and Message Passing

## What You'll Learn

- `mpsc::channel`, `send`/`recv`, and the `Result` on both ends
- Why disconnection is a normal event, not an error to panic on
- `try_recv`, iterating a receiver, cloning senders, bounded `sync_channel`
- Ownership transfer through a channel — the idea that makes this safe

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `ch := make(chan T)` | `mpsc::sync_channel(0)` (rendezvous) |
| Go `make(chan T, 8)` | `mpsc::sync_channel(8)` |
| Go `ch := make(chan T)` unbounded | `mpsc::channel()` — unbounded is the default here |
| JS `postMessage` (structured clone) | `tx.send(v)` — **moves**, no copy |

Two differences from Go that matter. First, the channel is split into two
non-clonable-together halves: `Sender` (clonable, many producers) and `Receiver`
(exactly one consumer — "mpsc" = multi-producer, single-consumer). Second,
closing is implicit: a channel disconnects when all senders or the receiver are
dropped. There is no `close(ch)`.

## Basic send and recv

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send("hello").unwrap();
    });

    println!("{}", rx.recv().unwrap());
}
```

Output:

```
hello
```

`recv()` blocks until a value arrives. `send` returns `Result` because the
receiver might already be gone.

## Ownership moves through the channel

This is the whole reason channels are safe here: `send` takes the value by
value. After sending, the producer cannot touch it.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let s = String::from("big payload");

    thread::spawn(move || {
        tx.send(s).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("{received} ({} bytes)", received.len());
}
```

Output:

```
big payload (11 bytes)
```

Nothing was copied — the `String`'s heap buffer was handed over. Try to use it
after sending:

```rust
use std::sync::mpsc;

fn main() {
    let (tx, _rx) = mpsc::channel();
    let s = String::from("hi");
    tx.send(s).unwrap();
    println!("{s}");
}
```

Intentionally broken:

```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:7:16
  |
5 |     let s = String::from("hi");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
6 |     tx.send(s).unwrap();
  |             - value moved here
7 |     println!("{s}");
  |                ^ value borrowed here after move
```

Exactly one thread owns the value at any moment, so there is nothing to race on.

## Iterating the receiver

A `Receiver` is an iterator that yields values until the channel disconnects.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        for i in 1..=4 {
            tx.send(i * 10).unwrap();
        }
    });

    for v in rx {
        println!("got {v}");
    }
}
```

Output:

```
got 10
got 20
got 30
got 40
```

The loop ends because the spawned thread's `tx` is dropped when the closure
returns. If you keep a `tx` alive in `main`, this loop **hangs forever** — the
most common channel bug in Rust.

## Disconnection: recv and try_recv

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel::<i32>();
    drop(tx);
    println!("{:?}", rx.recv());
    println!("{:?}", rx.try_recv());

    let (tx, rx) = mpsc::channel::<i32>();
    println!("{:?}", rx.try_recv());
    tx.send(7).unwrap();
    println!("{:?}", rx.try_recv());
}
```

Output:

```
Err(RecvError)
Err(Disconnected)
Err(Empty)
Ok(7)
```

`recv()` has one failure mode (`RecvError` = disconnected), so it carries no
extra information. `try_recv()` never blocks and distinguishes `Empty` (nothing
yet, senders alive) from `Disconnected` (never anything again) — only the second
should end your loop.

`recv_timeout(Duration)` sits between them, returning
`RecvTimeoutError::{Timeout, Disconnected}`.

## SendError carries the value back

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();
    drop(rx);

    match tx.send("nobody listening".to_string()) {
        Ok(()) => println!("sent"),
        Err(e) => {
            println!("{e}");
            println!("recovered value: {}", e.0);
        }
    }
}
```

Output:

```
sending on a closed channel
recovered value: nobody listening
```

`SendError<T>` is a newtype around the value that could not be delivered, so a
failed send does not lose data. `send` fails *only* on disconnect — an unbounded
channel never blocks.

## Multiple producers: clone the sender

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    for id in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            for n in 0..2 {
                tx.send(format!("producer {id} msg {n}")).unwrap();
            }
        });
    }
    drop(tx);

    let mut msgs: Vec<String> = rx.iter().collect();
    msgs.sort();
    for m in &msgs {
        println!("{m}");
    }
    println!("total {}", msgs.len());
}
```

Output (sorted — the arrival order varies every run):

```
producer 0 msg 0
producer 0 msg 1
producer 1 msg 0
producer 1 msg 1
producer 2 msg 0
producer 2 msg 1
total 6
```

The `drop(tx)` is load-bearing. The original sender is still alive in `main`
after the loop; without dropping it, the channel never disconnects and
`rx.iter()` blocks forever. The clones inside the threads drop themselves.

The `Receiver` cannot be cloned — `mpsc` means one consumer. To fan work out to
several consumers, share one receiver behind a `Mutex` (see the dispatch example
below) or use `crossbeam-channel`.

## sync_channel: bounded, blocking sends

```rust
use std::sync::mpsc;
use std::thread;
use std::time::{Duration, Instant};

fn main() {
    let (tx, rx) = mpsc::sync_channel(2);

    let producer = thread::spawn(move || {
        for i in 0..5 {
            let t = Instant::now();
            tx.send(i).unwrap();
            println!("sent {i} (blocked {} ms)", t.elapsed().as_millis());
        }
    });

    thread::sleep(Duration::from_millis(100));
    for v in rx {
        println!("received {v}");
        thread::sleep(Duration::from_millis(50));
    }
    producer.join().unwrap();
}
```

Output (the millisecond numbers wobble run to run):

```
sent 0 (blocked 0 ms)
sent 1 (blocked 0 ms)
received 0
sent 2 (blocked 104 ms)
received 1
sent 3 (blocked 54 ms)
received 2
sent 4 (blocked 51 ms)
received 3
received 4
```

The first two sends fill the buffer and return instantly; after that the
producer runs at the consumer's pace. That is backpressure, and it is the reason
to prefer `sync_channel` in real systems — an unbounded channel turns a slow
consumer into unbounded memory growth.

`sync_channel(0)` is a rendezvous channel: every `send` blocks until a receiver
takes the value. Bounded senders also offer `try_send`, which returns
`TrySendError::{Full, Disconnected}` instead of blocking.

## Commands as an enum

The idiomatic shape for a worker thread: one message type, a `match` on the
other side.

```rust
use std::sync::mpsc;
use std::thread;

enum Command {
    Add(i64),
    Multiply(i64),
    Report,
    Shutdown,
}

fn main() {
    let (tx, rx) = mpsc::channel();

    let worker = thread::spawn(move || {
        let mut acc: i64 = 1;
        for cmd in rx {
            match cmd {
                Command::Add(n) => acc += n,
                Command::Multiply(n) => acc *= n,
                Command::Report => println!("acc = {acc}"),
                Command::Shutdown => {
                    println!("shutting down with acc = {acc}");
                    break;
                }
            }
        }
        acc
    });

    tx.send(Command::Add(4)).unwrap();
    tx.send(Command::Report).unwrap();
    tx.send(Command::Multiply(10)).unwrap();
    tx.send(Command::Report).unwrap();
    tx.send(Command::Shutdown).unwrap();

    println!("final = {}", worker.join().unwrap());
}
```

Output:

```
acc = 5
acc = 50
shutting down with acc = 50
final = 50
```

An explicit `Shutdown` variant is clearer than relying on sender drop when the
worker holds state you want back. To get a reply per command, put a
`Sender<Reply>` inside the variant — the "actor with a reply channel" pattern.

## Worker dispatch: jobs in, results out

Two channels: one shared job queue (receiver behind a `Mutex`), one result
channel.

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;

fn main() {
    let (job_tx, job_rx) = mpsc::channel::<u64>();
    let job_rx = Arc::new(Mutex::new(job_rx));
    let (res_tx, res_rx) = mpsc::channel::<(u64, u64)>();

    let mut workers = Vec::new();
    for _ in 0..3 {
        let job_rx = Arc::clone(&job_rx);
        let res_tx = res_tx.clone();
        workers.push(thread::spawn(move || loop {
            let job = job_rx.lock().unwrap().recv();
            match job {
                Ok(n) => res_tx.send((n, n * n)).unwrap(),
                Err(_) => break,
            }
        }));
    }
    drop(res_tx);

    for n in 1..=6 {
        job_tx.send(n).unwrap();
    }
    drop(job_tx);

    let mut out: Vec<(u64, u64)> = res_rx.iter().collect();
    out.sort();
    println!("{:?}", out);

    for w in workers {
        w.join().unwrap();
    }
}
```

Output:

```
[(1, 1), (2, 4), (3, 9), (4, 16), (5, 25), (6, 36)]
```

Note `job_rx.lock().unwrap().recv()` on one line: the `MutexGuard` is a
temporary that drops at the end of the statement, so a worker does not hold the
queue lock while processing its job. Writing
`let g = job_rx.lock().unwrap(); let job = g.recv();` would serialize all the
workers. Lesson 14.3 covers guard lifetimes; 14.6 wraps this pattern into a
reusable `ThreadPool`.

Which job each worker gets is nondeterministic, which is why the results are
sorted before printing.

## crossbeam-channel

`std::sync::mpsc` has no `select` and no multi-consumer receiver. The
`crossbeam-channel` crate provides both — clonable `Receiver`s and a `select!`
macro to wait on several channels at once — with the same `Sender`/`Receiver`
shape. Module 15 covers `tokio::sync::mpsc` for async code, where `recv` is
`.await`ed instead of blocking.

## Quick Reference

| Item | Meaning |
|---|---|
| `mpsc::channel()` | `(Sender<T>, Receiver<T>)`, unbounded |
| `mpsc::sync_channel(n)` | `(SyncSender<T>, Receiver<T>)`, bounded; `n = 0` is rendezvous |
| `tx.send(v)` | `Result<(), SendError<T>>`; `Err` only on disconnect |
| `tx.try_send(v)` | `SyncSender` only; `Err(TrySendError::Full \| Disconnected)` |
| `tx.clone()` | more producers |
| `rx.recv()` | blocks; `Result<T, RecvError>` |
| `rx.try_recv()` | `Result<T, TryRecvError::Empty \| Disconnected>` |
| `rx.recv_timeout(d)` | `Result<T, RecvTimeoutError::Timeout \| Disconnected>` |
| `for v in rx` / `rx.iter()` | yields until disconnect |
| `rx.try_iter()` | drains what is buffered, then stops |
| disconnect | all senders dropped, or the receiver dropped |

## Common Mistakes

**Keeping the original `tx` alive.** `for v in rx` blocks forever because one
sender still exists in `main`. `drop(tx)` after cloning it to the producers.

**Treating `recv()`'s `Err` as a failure.** It means "all senders are gone",
which is the normal end of a stream. Break the loop; do not panic.

**Ending a loop on `TryRecvError::Empty`.** That just means nothing has arrived
yet. Only `Disconnected` is terminal.

**Using an unbounded channel for a hot producer.** A slow consumer becomes a
memory leak. Use `sync_channel(n)` so sends apply backpressure.

**Trying to clone the `Receiver`.** `mpsc` is single-consumer. Share it in an
`Arc<Mutex<_>>` or switch to `crossbeam-channel`.

## Exercises

1. Spawn two producers that each send five `u32`s, collect all ten in `main`,
   and print them sorted. Make sure the loop terminates.
2. Write a receiver loop using `try_recv` that prints `"idle"` every 10 ms while
   no message is waiting and exits cleanly on `Disconnected`.
3. Replace `mpsc::channel()` with `sync_channel(1)` in exercise 1 and print how
   long each send blocked.
4. Define `enum Req { Get(String, mpsc::Sender<Option<i32>>), Set(String, i32) }`
   and build a thread that owns a `HashMap` and answers `Get` on the reply
   channel.
5. Build a two-stage pipeline: thread A sends `1..=20`, thread B forwards only
   the even numbers, `main` sums what arrives. Print the sum (it is 110).

---

[← Course Index](../README.md)
