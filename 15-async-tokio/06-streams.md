[← 15.5 Async Channels and Synchronization](05-async-channels.md) · [Course Index](../README.md) · [15.7 Async Traits and Pinning →](07-async-traits-pin.md)

# 15.6 — Streams

## What You'll Learn

- The `Stream` trait: an `Iterator` whose `next` is a future
- `StreamExt` adapters, and the `while let Some(x) = s.next().await` loop
- `buffered` vs `buffer_unordered` for concurrent requests
- Why there are no async generators yet, and what to write instead

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `for await (const x of asyncIterable)` | `while let Some(x) = s.next().await` |
| JS `async function*` | no equivalent — use a channel or `unfold` |
| Python `async for` / `async def ... yield` | `StreamExt::next`; no `yield` yet |
| Go `for x := range ch` | `while let Some(x) = rx.recv().await` |

A `Stream` is to `Iterator` what `Future` is to a plain value: zero-or-more
items, each arriving later. Module 10's iterator chains all have analogues.

## The Stream trait

```rust
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

The return type tells the whole story: `Pending` (nothing yet),
`Ready(Some(item))` (here's one), `Ready(None)` (finished). Compare
`Iterator::next -> Option<Item>` and `Future::poll -> Poll<Output>`.

`Stream` lives in `futures` and is re-exported by `tokio_stream`; it is not in
`std` yet. The two crates ship separate `StreamExt` traits with overlapping but
not identical methods — import one, and qualify the other's unique adapters.

```rust
use std::pin::Pin;
use std::task::{Context, Poll};
use tokio_stream::{Stream, StreamExt};

struct Countdown(u32);

impl Stream for Countdown {
    type Item = u32;
    fn poll_next(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Option<u32>> {
        if self.0 == 0 {
            Poll::Ready(None)
        } else {
            self.0 -= 1;
            Poll::Ready(Some(self.0 + 1))
        }
    }
}

#[tokio::main]
async fn main() {
    let mut s = Countdown(4);
    while let Some(v) = s.next().await {
        print!("{v} ");
    }
    println!();
    println!("size_hint = {:?}", Countdown(3).size_hint());
}
```

Output:

```
4 3 2 1 
size_hint = (0, None)
```

This one never returns `Pending`, so it is an iterator in stream's clothing. A
real stream returns `Pending` while waiting on a socket.

## Adapters

`stream::iter` lifts any `Iterator` into a `Stream`, which is the easiest way to
play with adapters.

```rust
use tokio_stream::{self as stream, StreamExt};

#[tokio::main]
async fn main() {
    let mut s = stream::iter(vec![1, 2, 3]);
    while let Some(v) = s.next().await {
        println!("{v}");
    }

    let mut s = stream::iter(1..=6)
        .map(|n| n * 10)
        .filter(|n| n % 20 == 0)
        .take(2);
    while let Some(v) = s.next().await {
        println!("adapted {v}");
    }

    let total: i32 = stream::iter(1..=5).fold(0, |a, b| a + b).await;
    println!("fold = {total}");

    let v: Vec<i32> = stream::iter(1..=4).map(|x| x * x).collect().await;
    println!("collect = {v:?}");

    println!("count = {}", stream::iter(0..7).fold(0, |a, _| a + 1).await);
}
```

Output:

```
1
2
3
adapted 20
adapted 40
fold = 15
collect = [1, 4, 9, 16]
count = 7
```

Adapters are lazy like iterators — the chain does nothing until you `next()`,
`collect()`, or `fold()` it. `fold` and `collect` are `.await`ed because
consuming a stream takes time.

| Adapter | Does | Note |
|---|---|---|
| `next()` | `Option<Item>` | the workhorse; needs `Unpin` |
| `map(f)` | sync transform | |
| `then(f)` | **async** transform, one at a time | `f` returns a future |
| `filter(p)` / `filter_map(f)` | sync predicate | `futures` has async variants |
| `take(n)` / `take_while(p)` | truncate | |
| `skip(n)` / `skip_while(p)` | drop a prefix | |
| `chain(other)` | concatenate | |
| `zip(other)` | pair up | |
| `enumerate()` | add indices | |
| `fuse()` | guarantee `None` stays `None` | |
| `merge(other)` | interleave two streams as items arrive | tokio_stream |
| `chunks_timeout(n, d)` | batch by size **or** time | tokio_stream |
| `timeout(d)` | per-item deadline, yields `Result` | tokio_stream |
| `throttle(d)` | rate-limit items | tokio_stream |
| `collect()` | into `Vec`, `String`, `Result<Vec<_>, E>`, ... | |
| `fold(init, f)` | reduce | |
| `for_each(f)` | run for side effects | `futures` |
| `buffered(n)` / `buffer_unordered(n)` | **concurrent** | `futures` |

## then: async per item, but serial

```rust
use std::pin::pin;
use tokio::time::{sleep, Duration};
use tokio_stream::{self as stream, StreamExt};

#[tokio::main]
async fn main() {
    let mut s = pin!(stream::iter(1..=7).chunks_timeout(3, Duration::from_millis(50)));
    while let Some(c) = s.next().await {
        println!("chunk {c:?}");
    }

    let slow = stream::iter(1..=3).then(|v| async move {
        sleep(Duration::from_millis(if v == 2 { 80 } else { 10 })).await;
        v
    });
    let mut s = pin!(slow.timeout(Duration::from_millis(40)));
    while let Some(r) = s.next().await {
        match r {
            Ok(v) => println!("item {v}"),
            Err(_) => println!("item timed out"),
        }
    }
}
```

Output:

```
chunk [1, 2, 3]
chunk [4, 5, 6]
chunk [7]
item 1
item timed out
item 2
item 3
```

Two things worth pausing on.

First, the `pin!`. `next()` requires `Self: Unpin`, and adapters holding a
future or timer (`chunks_timeout`, `timeout`, `then`) are not. Without it you
get a wall of `PhantomPinned cannot be unpinned` errors ending in `consider
using the pin! macro`. Lesson 15.7 explains why; for now, wrap the stream in
`std::pin::pin!(...)` whenever `next()` refuses to compile.

Second, `timeout` does not drop the slow item — item 2 timed out *and then
arrived*, as a separate yield. Add `take_while(|r| r.is_ok())` to stop at the
first timeout instead.

## ReceiverStream: an mpsc as a stream

This is the bridge you will use most: it turns a channel into something you can
run adapters over.

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};
use tokio_stream::wrappers::ReceiverStream;
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel::<u32>(8);
    tokio::spawn(async move {
        for i in 1..=5 {
            sleep(Duration::from_millis(10)).await;
            tx.send(i).await.unwrap();
        }
    });

    let mut s = ReceiverStream::new(rx).map(|v| v * 100);
    while let Some(v) = s.next().await {
        println!("{v}");
    }
    println!("channel closed -> stream ended");
}
```

Output:

```
100
200
300
400
500
channel closed -> stream ended
```

`tokio_stream::wrappers` has one for most tokio types:

| Wrapper | Wraps |
|---|---|
| `ReceiverStream` | `mpsc::Receiver` |
| `UnboundedReceiverStream` | `mpsc::UnboundedReceiver` |
| `BroadcastStream` | `broadcast::Receiver` (items are `Result`) |
| `WatchStream` | `watch::Receiver` |
| `IntervalStream` | `time::Interval` |
| `TcpListenerStream` | `TcpListener`, yielding accepted sockets |
| `LinesStream` | `AsyncBufReadExt::lines` |

`IntervalStream` plus `merge` adds a periodic tick to an event loop without a
`select!`.

## buffered vs buffer_unordered — the concurrency win

A stream of futures can be polled concurrently. This is the stream equivalent of
`JoinSet` and the main reason to use streams at all.

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration, Instant};

async fn fetch(id: u32) -> u32 {
    sleep(Duration::from_millis(100)).await;
    id * 10
}

#[tokio::main]
async fn main() {
    let t = Instant::now();
    let v: Vec<u32> = stream::iter(1..=6).then(fetch).collect().await;
    println!("then (serial)      {v:?} in {}ms", t.elapsed().as_millis());

    let t = Instant::now();
    let v: Vec<u32> = stream::iter(1..=6).map(fetch).buffered(3).collect().await;
    println!("buffered(3)        {v:?} in {}ms", t.elapsed().as_millis());

    let t = Instant::now();
    let v: Vec<u32> = stream::iter(1..=6).map(fetch).buffer_unordered(6).collect().await;
    println!("buffer_unordered(6) {v:?} in {}ms", t.elapsed().as_millis());
}
```

Output:

```
then (serial)      [10, 20, 30, 40, 50, 60] in 612ms
buffered(3)        [10, 20, 30, 40, 50, 60] in 205ms
buffer_unordered(6) [10, 20, 30, 40, 50, 60] in 102ms
```

Roughly 600ms, 200ms, 100ms. Six 100ms requests: serial costs 600, three at a
time costs 200, all six at once costs 100. The `n` is a built-in concurrency
limit — no `Semaphore` needed.

Note `map(fetch)`, not `then(fetch)`: `map` produces a stream *of futures* that
`buffered` polls `n` at a time, while `then` awaits each before producing the
next. That is the serialization.

The two buffering adapters differ in output order.

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

async fn fetch(id: u64) -> u64 {
    sleep(Duration::from_millis((7 - id) * 30)).await;
    id
}

#[tokio::main]
async fn main() {
    let v: Vec<u64> = stream::iter(1..=6).map(fetch).buffered(6).collect().await;
    println!("buffered          {v:?}");
    let v: Vec<u64> = stream::iter(1..=6).map(fetch).buffer_unordered(6).collect().await;
    println!("buffer_unordered  {v:?}");
}
```

Output:

```
buffered          [1, 2, 3, 4, 5, 6]
buffer_unordered  [6, 5, 4, 3, 2, 1]
```

Item 1 is slowest here, item 6 fastest. `buffered` preserves input order, so it
holds finished results until their turn — a slow item at the front stalls
everything behind it. `buffer_unordered` yields whatever is ready.

**Prefer `buffer_unordered` unless you need order:** lower latency to first
result, no head-of-line blocking.

## FuturesUnordered

If you have a set of futures rather than a stream of them, `FuturesUnordered`
is the direct tool — it *is* a stream of their completions.

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut fu = FuturesUnordered::new();
    for i in 1..=4u64 {
        fu.push(async move {
            sleep(Duration::from_millis((5 - i) * 20)).await;
            i
        });
    }
    while let Some(v) = fu.next().await {
        print!("{v} ");
    }
    println!();
}
```

Output:

```
4 3 2 1 
```

Completion order. Unlike `JoinSet` these are not spawned tasks — they run on
the current task, so no `Send + 'static` requirement but no cross-thread
parallelism either. `JoinSet` to spread CPU across workers;
`FuturesUnordered` for concurrent I/O in one task.

## try_stream: streams of Results

When items are `Result<T, E>`, `TryStreamExt` gives you short-circuiting
adapters.

```rust
use futures::stream::{self, StreamExt, TryStreamExt};

async fn parse(s: &str) -> Result<u32, String> {
    s.parse::<u32>().map_err(|e| format!("{s:?}: {e}"))
}

#[tokio::main]
async fn main() {
    let ok: Result<Vec<u32>, String> = stream::iter(["1", "2", "3"])
        .then(parse)
        .try_collect()
        .await;
    println!("{ok:?}");

    let bad: Result<Vec<u32>, String> = stream::iter(["1", "x", "3"])
        .then(parse)
        .try_collect()
        .await;
    println!("{bad:?}");

    let doubled: Vec<Result<u32, String>> = stream::iter(["4", "y"])
        .then(parse)
        .map_ok(|v| v * 2)
        .collect()
        .await;
    println!("{doubled:?}");
}
```

Output:

```
Ok([1, 2, 3])
Err("\"x\": invalid digit found in string")
[Ok(8), Err("\"y\": invalid digit found in string")]
```

| Adapter | Does |
|---|---|
| `try_collect()` | `Result<Collection, E>`, stops at first `Err` |
| `try_next()` | `Result<Option<T>, E>` |
| `map_ok(f)` / `map_err(f)` | transform one side |
| `and_then(f)` | async, only on `Ok` |
| `try_filter(p)` / `try_fold(..)` | `Result`-aware versions |
| `try_for_each_concurrent(n, f)` | concurrent, stops at first `Err` |
| `into_stream()` | back to a plain stream of `Result` |

`collect()` into `Result<Vec<_>, E>` is identical to `try_collect()` — the same
trick as `Iterator::collect` in Module 08.

## There are no async generators

You cannot write this:

```rust
async fn* pages() { ... yield x ... }   // not Rust
```

`gen` blocks exist for *sync* iterators; the async version is not stable. Three
workarounds, in order of preference.

**1. Spawn a task writing into a channel.** Reads like a generator, and the
producer runs concurrently with the consumer.

```rust
use std::pin::pin;
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;
use tokio_stream::{Stream, StreamExt};

fn pages() -> impl Stream<Item = String> {
    let (tx, rx) = mpsc::channel(4);
    tokio::spawn(async move {
        for p in 1..=3 {
            if tx.send(format!("page {p}")).await.is_err() {
                return;
            }
        }
    });
    ReceiverStream::new(rx)
}

#[tokio::main]
async fn main() {
    let mut s = pages();
    while let Some(p) = s.next().await {
        println!("{p}");
    }

    let mut s = pin!(futures::stream::unfold(0u32, |n| async move {
        if n >= 3 { None } else { Some((n * n, n + 1)) }
    }));
    while let Some(v) = s.next().await {
        print!("{v} ");
    }
    println!();
}
```

Output:

```
page 1
page 2
page 3
0 1 4
```

The `is_err()` check matters: if the consumer drops the stream, `send` fails and
the task must exit instead of spinning forever.

**2. `stream::unfold`,** shown above: a seed plus an async closure returning
`Option<(item, next_seed)>`. Good for pagination where the next cursor comes
from the previous response. Not `Unpin`, hence the `pin!`.

**3. The `async-stream` crate,** which gives you macro-based `yield`:

```rust
use async_stream::stream;

fn pages() -> impl Stream<Item = String> {
    stream! {
        for p in 1..=3 {
            let body = fetch_page(p).await;
            yield body;
        }
    }
}
```

Closest to a real generator, at the cost of one dependency. The right pick when
the producer logic is branchy enough that `unfold`'s state-threading gets ugly.

## A realistic pipeline

Everything together: a channel source, bounded concurrency, batching, and a
terminal fold.

```rust
use futures::stream::StreamExt;
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;

async fn enrich(id: u32) -> (u32, String) {
    tokio::time::sleep(tokio::time::Duration::from_millis(20)).await;
    (id, format!("name-{id}"))
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(32);
    tokio::spawn(async move {
        for id in 1..=10 {
            let _ = tx.send(id).await;
        }
    });

    let total = ReceiverStream::new(rx)
        .map(enrich)
        .buffer_unordered(4)
        .fold(0usize, |acc, (_, name)| async move { acc + name.len() })
        .await;
    println!("total name bytes = {total}");
}
```

Output:

```
total name bytes = 61
```

Ten items, four at a time: roughly 65ms instead of the ~200ms a serial loop would cost. Swap `enrich` for a real
HTTP or DB call and this is a production ingest loop.

## Quick Reference

| Item | Use |
|---|---|
| `stream::iter(iter)` | `Iterator` -> `Stream` |
| `s.next().await` | `Option<Item>`; needs `Unpin` |
| `while let Some(x) = s.next().await` | the iteration loop |
| `pin!(s)` | make a non-`Unpin` stream usable with `next()` |
| `.map(f)` | sync transform |
| `.then(f)` | async transform, serial |
| `.map(f).buffered(n)` | async transform, n concurrent, in order |
| `.map(f).buffer_unordered(n)` | async transform, n concurrent, completion order |
| `.collect().await` | into a collection |
| `.try_collect().await` | `Result<Collection, E>` |
| `.fold(init, f).await` | reduce |
| `.chunks_timeout(n, d)` | batch by count or time |
| `.timeout(d)` | per-item deadline |
| `.merge(other)` | interleave |
| `ReceiverStream::new(rx)` | `mpsc` -> `Stream` |
| `FuturesUnordered` | set of futures as a completion stream |
| `stream::unfold(seed, f)` | generator-ish from a seed |

## Common Mistakes

**Using `then` and expecting concurrency.** It is strictly one at a time. Use
`map` + `buffer_unordered(n)`.

**Hitting `cannot be unpinned` and giving up.** Wrap it: `let mut s = pin!(expr);`.
Any adapter holding a timer or future is not `Unpin`.

**`buffer_unordered(usize::MAX)` on an unbounded source.** You just launched
every request at once. The `n` is your rate limit; choose it deliberately.

**Mixing the two `StreamExt` traits in one scope.** `futures::StreamExt` and
`tokio_stream::StreamExt` both define `next`, producing ambiguity errors. Import
one; qualify the other's unique methods.

**Forgetting the `is_err()` check in a channel-backed generator.** When the
consumer drops the stream, the producer task leaks and spins.

## Exercises

1. Turn `vec!["3", "7", "x", "9"]` into a stream, parse each with `then`, and
   collect into `Vec<Result<u32, _>>`. Then `try_collect()` it and compare.
2. Build a stream from an `mpsc` channel, apply `chunks_timeout(5, 100ms)`, and
   send 12 items with varying delays. Show both size-triggered and
   time-triggered batches.
3. Write an `async fn fetch(i: u32) -> u32` that sleeps `i * 10` ms. Run 10 of
   them with `buffered(3)` and `buffer_unordered(3)`, printing both the order
   and the elapsed time.
4. Implement `Stream` by hand for a type that yields the Fibonacci sequence up
   to a cap, then consume it with `collect()`.
5. Write `fn countdown(from: u32) -> impl Stream<Item = u32>` using
   `stream::unfold` where each step sleeps 10ms. Consume it with `while let`.

---

[← 15.5 Async Channels and Synchronization](05-async-channels.md) · [Course Index](../README.md) · [15.7 Async Traits and Pinning →](07-async-traits-pin.md)
