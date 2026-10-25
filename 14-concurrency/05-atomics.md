[← 14.4 Send, Sync, and Thread Safety](04-send-sync.md) · [Course Index](../README.md) · [14.6 Scoped Threads and Thread Pools →](06-scoped-pools.md)

# 14.5 — Atomics

## What You'll Learn

- The atomic types and every operation they offer
- `Ordering` in five variants, with a rule of thumb instead of a memory model
- A lock-free counter and an `AtomicBool` shutdown flag
- When an atomic beats a `Mutex`, and when `Relaxed` is enough

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `atomic.AddInt64(&n, 1)` | `n.fetch_add(1, Ordering::Relaxed)` |
| Java `AtomicInteger.incrementAndGet()` | `n.fetch_add(1, ord) + 1` |
| Python — none (GIL) | `AtomicUsize` |
| C++ `std::atomic<T>` | same model, same orderings, explicit ordering required |

The difference from Go and Java: Rust makes you name the memory ordering on
every operation. Go has one (sequentially consistent); Java has volatile
semantics baked in. The upside is you can ask for the cheap one when the cheap
one is correct, which is most of the time.

An atomic gives you mutation through `&self` — so `AtomicUsize` is `Sync` and
needs no `Mutex`. Operations happen in one indivisible step at the hardware
level.

## The types

`AtomicBool`, `AtomicI8/16/32/64/size`, `AtomicU8/16/32/64/size`, and
`AtomicPtr<T>`. There is no `AtomicF64` and no `Atomic<T>` for arbitrary `T` —
atomics map onto single machine words.

```rust
use std::sync::atomic::{AtomicBool, AtomicI64, AtomicUsize, Ordering};

fn main() {
    let a = AtomicUsize::new(10);
    println!("load         {}", a.load(Ordering::Relaxed));
    a.store(20, Ordering::Relaxed);
    println!("after store  {}", a.load(Ordering::Relaxed));
    println!("fetch_add    {}", a.fetch_add(5, Ordering::Relaxed));
    println!("fetch_sub    {}", a.fetch_sub(3, Ordering::Relaxed));
    println!("swap         {}", a.swap(100, Ordering::Relaxed));
    println!("now          {}", a.load(Ordering::Relaxed));

    let b = AtomicBool::new(false);
    println!("bool swap    {}", b.swap(true, Ordering::Relaxed));
    println!("bool now     {}", b.load(Ordering::Relaxed));

    let i = AtomicI64::new(-5);
    println!("i64 fetch_add {}", i.fetch_add(10, Ordering::Relaxed));
    println!("i64 now       {}", i.load(Ordering::Relaxed));
    println!("max {}  min {}",
        i.fetch_max(100, Ordering::Relaxed),
        i.fetch_min(1, Ordering::Relaxed));
    println!("i64 final     {}", i.load(Ordering::Relaxed));
}
```

Output:

```
load         10
after store  20
fetch_add    20
fetch_sub    25
swap         22
now          100
bool swap    false
bool now     true
i64 fetch_add -5
i64 now       5
max 5  min 100
i64 final     1
```

Every `fetch_*` and `swap` returns the **previous** value. That is why
`fetch_add` prints 20 when the new value is 25, and why `fetch_min(1)` prints
100: the `fetch_max(100)` ran first and already stored 100.

`fetch_add` wraps on overflow rather than panicking, even in debug builds — the
hardware instruction has no overflow trap. Also note `load`/`store` need no
`mut`: that is the point of an atomic.

## A lock-free counter

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));

    let hs: Vec<_> = (0..8).map(|_| {
        let c = Arc::clone(&counter);
        thread::spawn(move || {
            for _ in 0..10_000 {
                c.fetch_add(1, Ordering::Relaxed);
            }
        })
    }).collect();
    for h in hs { h.join().unwrap(); }

    println!("count = {}", counter.load(Ordering::Relaxed));
}
```

Output:

```
count = 80000
```

`Arc<AtomicUsize>` — not `Arc<Mutex<usize>>`, and not a bare `AtomicUsize`,
because `spawn` still needs `'static` ownership. If the counter is a `static`
you can skip the `Arc` entirely: `static HITS: AtomicUsize = AtomicUsize::new(0)`
is a `const` constructor, so it works at file scope and every thread just
references it.

## compare_exchange: the read-modify-write primitive

Everything else is built on this. "If the value is still `current`, set it to
`new`; otherwise tell me what it actually is."

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn main() {
    let a = AtomicUsize::new(5);

    println!("{:?}", a.compare_exchange(5, 10, Ordering::AcqRel, Ordering::Acquire));
    println!("{:?}", a.compare_exchange(5, 20, Ordering::AcqRel, Ordering::Acquire));
    println!("now {}", a.load(Ordering::Relaxed));

    let mut cur = a.load(Ordering::Relaxed);
    loop {
        let next = cur * 3;
        match a.compare_exchange_weak(cur, next, Ordering::AcqRel, Ordering::Relaxed) {
            Ok(_) => break,
            Err(actual) => cur = actual,
        }
    }
    println!("after CAS loop {}", a.load(Ordering::Relaxed));
}
```

Output:

```
Ok(5)
Err(10)
now 10
after CAS loop 30
```

`Ok(prev)` means it swapped; `Err(actual)` means it did not, and hands you the
real current value so you can retry. Two orderings: the first applies on
success, the second on failure (failure cannot be `Release` — nothing was
written).

`compare_exchange_weak` may fail *spuriously* even when the value matched, which
lets it compile to a single LL/SC instruction pair on ARM. Use `_weak` inside a
retry loop — which you need anyway — and plain `compare_exchange` for a one-shot
attempt.

## update and try_update

The CAS loop above, done for you.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn main() {
    let a = AtomicUsize::new(30);
    println!("{:?}", a.try_update(Ordering::AcqRel, Ordering::Acquire, |v| Some(v + 1)));
    println!("{:?}", a.try_update(Ordering::AcqRel, Ordering::Acquire,
        |v| if v > 100 { Some(0) } else { None }));
    println!("{}", a.update(Ordering::AcqRel, Ordering::Acquire, |v| v * 2));
    println!("final {}", a.load(Ordering::Relaxed));
}
```

Output:

```
Ok(30)
Err(31)
31
final 62
```

`try_update` takes `FnMut(T) -> Option<T>`: returning `None` aborts and gives
`Err(current)`. `update` takes `FnMut(T) -> T` and always succeeds. Both return
the **previous** value, and both may call your closure several times, so keep it
pure.

`fetch_update` is the old name for `try_update` and is deprecated as of this
toolchain — using it compiles with a warning suggesting the rename.

## Bitwise operations

```rust
use std::sync::atomic::{AtomicU8, Ordering};

const READ: u8 = 0b0001;
const WRITE: u8 = 0b0010;
const EXEC: u8 = 0b0100;

fn main() {
    let flags = AtomicU8::new(0);

    flags.fetch_or(READ | WRITE, Ordering::Relaxed);
    println!("{:04b}", flags.load(Ordering::Relaxed));

    flags.fetch_and(!WRITE, Ordering::Relaxed);
    println!("{:04b}", flags.load(Ordering::Relaxed));

    flags.fetch_xor(EXEC, Ordering::Relaxed);
    println!("{:04b}", flags.load(Ordering::Relaxed));

    println!("has READ: {}", flags.load(Ordering::Relaxed) & READ != 0);
}
```

Output:

```
0011
0001
0101
has READ: true
```

`fetch_nand` also exists. These are how you build a lock-free bitset of state
flags without a `Mutex`.

## Ordering, practically

The five variants, weakest to strongest:

| Ordering | Guarantees | Use for |
|---|---|---|
| `Relaxed` | the operation is atomic; **no** ordering with other accesses | counters, statistics, flags nobody waits on |
| `Acquire` | on a load: later reads see everything the matching `Release` store published | taking a lock, reading a "ready" flag |
| `Release` | on a store: everything written before it becomes visible to a matching `Acquire` | releasing a lock, setting a "ready" flag |
| `AcqRel` | both, for read-modify-write ops | `compare_exchange`, `swap` in a lock |
| `SeqCst` | `AcqRel` plus a single global total order across all `SeqCst` ops | when you are not sure, or the algorithm needs a global order |

The rule of thumb that covers almost every real case:

1. **A counter or a statistic nobody synchronizes on → `Relaxed`.** The final
   total is still exact; you just cannot infer anything about *other* memory from
   reading it.
2. **Publishing data behind a flag → `Release` on the store, `Acquire` on the
   load.** This is the only pattern where you actually need the stronger
   orderings, and it is the one below.
3. **Unsure, or writing a lock-free algorithm → `SeqCst`.** It is always correct,
   and on x86 it costs nothing extra for loads and a fence for stores.

Getting an ordering *too strong* costs performance. Too weak is undefined
behaviour that works on x86 and breaks on ARM, which is the worst possible bug.
If you are not sure, `SeqCst`.

## Release/Acquire: publishing data

```rust
use std::hint;
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::thread;

static DATA: AtomicU64 = AtomicU64::new(0);
static READY: AtomicBool = AtomicBool::new(false);

fn main() {
    thread::spawn(|| {
        DATA.store(99, Ordering::Relaxed);
        READY.store(true, Ordering::Release);
    });

    let consumer = thread::spawn(|| {
        while !READY.load(Ordering::Acquire) {
            hint::spin_loop();
        }
        DATA.load(Ordering::Relaxed)
    });

    println!("consumer read {}", consumer.join().unwrap());
}
```

Output:

```
consumer read 99
```

The `DATA` store is `Relaxed`, yet the consumer is guaranteed to see 99. The
`Release` store to `READY` acts as a barrier: everything the producer wrote
before it is visible to anyone who reads `READY` with `Acquire`. Make those two
`Relaxed` and the guarantee evaporates — on ARM the consumer can legitimately
read 0 after seeing `true`.

`hint::spin_loop()` emits a CPU pause instruction (`PAUSE` on x86, `YIELD` on
ARM). It is a hint only: it saves power and helps hyperthread siblings, and does
not yield to the scheduler. Spin only for waits measured in nanoseconds; for
anything longer use `thread::yield_now()`, or a channel, or a `Condvar`.

## A shutdown flag

The single most common real use of `AtomicBool`.

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;
use std::time::Duration;

fn main() {
    let running = Arc::new(AtomicBool::new(true));
    let ticks = Arc::new(AtomicUsize::new(0));

    let r = Arc::clone(&running);
    let t = Arc::clone(&ticks);
    let worker = thread::spawn(move || {
        while r.load(Ordering::Relaxed) {
            t.fetch_add(1, Ordering::Relaxed);
            thread::sleep(Duration::from_millis(10));
        }
        println!("worker saw the shutdown flag");
    });

    thread::sleep(Duration::from_millis(100));
    running.store(false, Ordering::Relaxed);
    worker.join().unwrap();
    println!("ticks > 0: {}", ticks.load(Ordering::Relaxed) > 0);
}
```

Output:

```
worker saw the shutdown flag
ticks > 0: true
```

`Relaxed` is correct here: the flag carries no data, and the worker only needs
to see it *eventually*. The exact tick count is nondeterministic, which is why
the assertion is `> 0`.

If the shutdown also publishes state the worker must read, make the store
`Release` and the load `Acquire`.

## Atomics vs Mutex: the performance intuition

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Instant;

const N: u64 = 200_000;

fn main() {
    let a = Arc::new(AtomicU64::new(0));
    let t = Instant::now();
    let hs: Vec<_> = (0..4).map(|_| {
        let a = Arc::clone(&a);
        thread::spawn(move || for _ in 0..N { a.fetch_add(1, Ordering::Relaxed); })
    }).collect();
    for h in hs { h.join().unwrap(); }
    let atomic_ms = t.elapsed().as_millis();

    let m = Arc::new(Mutex::new(0u64));
    let t = Instant::now();
    let hs: Vec<_> = (0..4).map(|_| {
        let m = Arc::clone(&m);
        thread::spawn(move || for _ in 0..N { *m.lock().unwrap() += 1; })
    }).collect();
    for h in hs { h.join().unwrap(); }
    let mutex_ms = t.elapsed().as_millis();

    println!("atomic: {} -> {} ms", a.load(Ordering::Relaxed), atomic_ms);
    println!("mutex:  {} -> {} ms", *m.lock().unwrap(), mutex_ms);
}
```

Output (`--release`, 4 threads, 800k increments; absolute numbers vary by
machine):

```
atomic: 800000 -> 8 ms
mutex:  800000 -> 23 ms
```

Roughly 3x here. The intuition:

- **Uncontended**, both are cheap — a `Mutex` lock/unlock is itself two atomic
  operations.
- **Contended**, the atomic stays in userspace spinning on a cache line, while
  the `Mutex` may park the thread in the kernel and wake it again.
- A contended atomic is **not free**: the cache line ping-pongs between cores.
  Eight threads hammering one `AtomicUsize` can be slower than eight threads
  each updating their own counter and summing at the end.

And the hard limit: an atomic protects **one word**. The moment two values must
change together, no number of atomics will do, and a `Mutex` is the only correct
answer. Reach for an atomic when the state is a single counter, flag, or
index — otherwise use 14.3.

## Quick Reference

| Item | Meaning |
|---|---|
| `AtomicBool/I8..I64/U8..U64/Isize/Usize/Ptr<T>` | the available types; no float, no generic `T` |
| `AtomicUsize::new(v)` | `const fn` — works in a `static` |
| `load(ord)` / `store(v, ord)` | read / write, no `mut` needed |
| `swap(v, ord)` | store, return previous |
| `fetch_add` / `fetch_sub` | wrapping arithmetic, return previous |
| `fetch_and` / `_or` / `_xor` / `_nand` | bitwise, return previous |
| `fetch_max` / `fetch_min` | return previous |
| `compare_exchange(cur, new, succ, fail)` | `Ok(prev)` or `Err(actual)` |
| `compare_exchange_weak(...)` | may fail spuriously; for retry loops |
| `try_update(succ, fail, f)` | `f: FnMut(T) -> Option<T>`; `None` aborts |
| `update(succ, fail, f)` | `f: FnMut(T) -> T`; returns previous |
| `get_mut()` / `into_inner()` | non-atomic access; needs `&mut` / ownership |
| `hint::spin_loop()` | CPU pause hint, not a yield |
| `atomic::fence(ord)` | standalone barrier, rarely needed |

## Common Mistakes

**Expecting `fetch_add` to return the new value.** Every `fetch_*` returns the
old one. Add the delta yourself: `n.fetch_add(1, ord) + 1`.

**Using `Relaxed` to publish data.** A `Relaxed` flag says nothing about other
memory. `Release` on the store, `Acquire` on the load.

**A load-then-store "increment".** `a.store(a.load(o) + 1, o)` is two atomic
operations with a race between them. Use `fetch_add`.

**Trying to make two atomics consistent.** Two atomic writes are not one atomic
transaction. Use a `Mutex` (14.3).

**Spinning without a bound.** `while !flag.load(..) {}` burns a core. Add
`hint::spin_loop()` for nanosecond waits, and switch to a channel or `Condvar`
for anything longer.

## Exercises

1. Build a `static REQUESTS: AtomicUsize` and have eight threads bump it 1000
   times each with `Relaxed`. Print 8000 with no `Arc` anywhere.
2. Implement `fn next_id(c: &AtomicU64) -> u64` returning 1, 2, 3, … with one
   atomic operation, and prove uniqueness by collecting 4000 ids from four
   threads into a `HashSet`.
3. Write a `compare_exchange_weak` loop that stores the maximum value seen, then
   replace it with `fetch_max` and confirm the same answer.
4. Use `try_update` to build a saturating counter that refuses to exceed 100,
   and print the `Err` you get at the ceiling.
5. Build a two-thread spin-wait handoff with `Release`/`Acquire` passing a
   `u64` through an `AtomicU64`, then explain in your own words what breaks if
   both orderings become `Relaxed`.

---

[← 14.4 Send, Sync, and Thread Safety](04-send-sync.md) · [Course Index](../README.md) · [14.6 Scoped Threads and Thread Pools →](06-scoped-pools.md)
