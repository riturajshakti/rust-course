[← 14.2 Channels and Message Passing](02-channels.md) · [Course Index](../README.md) · [14.4 Send, Sync, and Thread Safety →](04-send-sync.md)

# 14.3 — Shared State with Mutex and RwLock

## What You'll Learn

- `Arc<Mutex<T>>`: why both wrappers, and what each one buys
- `lock()` returning a `Result`, poisoning, and the `unwrap` idiom
- `MutexGuard` scope — the source of most accidental serialization and deadlock
- `RwLock` for many readers, and when an atomic is the better tool

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `var mu sync.Mutex; mu.Lock()` | `let m = Mutex::new(data); m.lock()` |
| Python `with lock:` | the `MutexGuard`'s scope |
| Java `synchronized(obj)` | `Mutex<T>` wraps the data, not a code block |

The structural difference: in Go, Java and Python the lock and the data it
protects are separate, and nothing stops you reading the data without locking.
In Rust the data lives **inside** the `Mutex`, so the only way to reach it is to
lock. Forgetting to lock is not a bug you can write.

Lesson 11.4 introduced `Arc` and `Mutex` as smart pointers; this lesson is the
concurrency use. Everything you need is here.

## The counter across threads

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                *counter.lock().unwrap() += 1;
            }
        }));
    }
    for h in handles {
        h.join().unwrap();
    }

    println!("count = {}", *counter.lock().unwrap());
}
```

Output:

```
count = 10000
```

Exactly 10000, every run. Two wrappers, two jobs:

- **`Mutex<T>`** gives mutation through a shared reference (`&Mutex<T>` →
  `&mut T` while locked) and makes the type `Sync`.
- **`Arc<T>`** gives shared *ownership* so each thread can hold a handle that
  lives as long as it needs; `spawn` requires `'static`, and `Arc::clone` is how
  you satisfy it.

Neither alone works: `Mutex<T>` by itself cannot be moved into ten closures,
and `Arc<T>` by itself gives you only `&T`, which is immutable.

`Arc::clone(&counter)` and `counter.clone()` are identical; the explicit form is
conventional because it makes the cheap refcount bump obvious.

## lock() returns a Result

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    {
        let mut g = m.lock().unwrap();
        *g += 1;
        println!("inside guard: {}", *g);
    }
    println!("after guard: {:?}", m);
    println!("try_lock: {:?}", m.try_lock());
    println!("into_inner: {:?}", m.into_inner());
}
```

Output:

```
inside guard: 6
after guard: Mutex { data: 6, poisoned: false, .. }
try_lock: Ok(6)
into_inner: Ok(6)
```

`lock()` is `Result<MutexGuard<T>, PoisonError<MutexGuard<T>>>`. The `Err` has
nothing to do with contention — it means the mutex is **poisoned**. `.unwrap()`
is the standard idiom and the right default: if another thread died mid-mutation,
you probably should not carry on with the wreckage.

`try_lock()` returns `Err(TryLockError::WouldBlock)` when the lock is held.
`into_inner()` consumes the `Mutex` and needs no lock at all, since owning it
proves nobody else has a reference. `get_mut()` does the same through `&mut`.

## Poisoning

A thread that panics while holding the lock poisons it. Every later `lock()`
fails — but the data is still reachable.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let d = Arc::clone(&data);
    let h = thread::spawn(move || {
        let mut v = d.lock().unwrap();
        v.push(4);
        panic!("died holding the lock");
    });
    let _ = h.join();

    println!("poisoned: {}", data.is_poisoned());
    match data.lock() {
        Ok(v) => println!("ok {:?}", v),
        Err(e) => {
            println!("error: {e}");
            println!("recovered: {:?}", e.into_inner());
        }
    }
}
```

Output (the thread id varies):

```
thread '<unnamed>' (13893289) panicked at src/main.rs:11:9:
died holding the lock
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
poisoned: true
error: poisoned lock: another task failed inside
recovered: [1, 2, 3, 4]
```

The `4` was pushed before the panic, so the `Vec` may be in a half-updated
state — that is exactly what poisoning is warning you about.
`PoisonError::into_inner()` hands you the guard anyway, for the cases where you
know the invariant survived. `clear_poison()` resets the flag.

## MutexGuard scope

The guard unlocks on drop. Hold it longer than the critical section and you
serialize work that did not need serializing.

```rust
use std::sync::Mutex;

fn expensive(n: i32) -> i32 { n * 2 }

fn main() {
    let m = Mutex::new(21);

    let value = *m.lock().unwrap();
    let doubled = expensive(value);
    println!("{doubled}");

    let mut g = m.lock().unwrap();
    *g += 1;
    drop(g);
    println!("{}", *m.lock().unwrap());
}
```

Output:

```
42
22
```

Two techniques in there. `let value = *m.lock().unwrap();` copies the value out
and the guard — a temporary — drops at the end of that statement, so
`expensive()` runs unlocked. And `drop(g)` releases early when you cannot
restructure. A `{ }` block around the critical section does the same thing.

The trap: `if let` and `match` keep their scrutinee's temporaries alive for the
whole body, so `while let Some(x) = queue.lock().unwrap().pop() { slow(x) }`
holds the lock during `slow`. Bind first, drop, then work.

## Deadlock from lock ordering

Two mutexes taken in opposite orders by two threads. This is a real program that
hangs; there is no error, no timeout, and no compiler diagnostic.

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let a = Arc::new(Mutex::new("A"));
    let b = Arc::new(Mutex::new("B"));

    let (a1, b1) = (Arc::clone(&a), Arc::clone(&b));
    let t1 = thread::spawn(move || {
        let _ga = a1.lock().unwrap();
        println!("t1 holds A");
        thread::sleep(Duration::from_millis(100));
        println!("t1 wants B");
        let _gb = b1.lock().unwrap();
        println!("t1 holds both");
    });

    let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
    let t2 = thread::spawn(move || {
        let _gb = b2.lock().unwrap();
        println!("t2 holds B");
        thread::sleep(Duration::from_millis(100));
        println!("t2 wants A");
        let _ga = a2.lock().unwrap();
        println!("t2 holds both");
    });

    t1.join().unwrap();
    t2.join().unwrap();
    println!("never printed");
}
```

Output (then it hangs until killed; run it under `timeout 5` or
<kbd>Ctrl-C</kbd>):

```
t1 holds A
t2 holds B
t1 wants B
t2 wants A
```

`holds both` and `never printed` never appear. The fix is a global lock order —
every thread takes `a` before `b` — or one mutex covering both pieces of state.
`try_lock` with a backoff is the escape hatch when ordering is impossible.

Rust's type system prevents data races. It does **not** prevent deadlocks: lock
ordering is a design obligation that stays yours.

## RwLock: many readers or one writer

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

fn main() {
    let cache = Arc::new(RwLock::new(HashMap::new()));

    cache.write().unwrap().insert("a", 1);
    cache.write().unwrap().insert("b", 2);

    let mut readers = Vec::new();
    for i in 0..3 {
        let cache = Arc::clone(&cache);
        readers.push(thread::spawn(move || {
            let g = cache.read().unwrap();
            thread::sleep(Duration::from_millis(50));
            format!("reader {i} sees {} keys", g.len())
        }));
    }
    for r in readers {
        println!("{}", r.join().unwrap());
    }

    {
        let mut w = cache.write().unwrap();
        w.insert("c", 3);
    }
    let mut keys: Vec<_> = cache.read().unwrap().keys().copied().collect();
    keys.sort();
    println!("{:?}", keys);
}
```

Output:

```
reader 0 sees 2 keys
reader 1 sees 2 keys
reader 2 sees 2 keys
["a", "b", "c"]
```

`read()` yields `RwLockReadGuard` (`Deref` to `&T`), `write()` yields
`RwLockWriteGuard` (`DerefMut` to `&mut T`). Both return `LockResult` and poison
the same way.

Readers really do run concurrently:

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::thread;
use std::time::{Duration, Instant};

fn main() {
    let rw = Arc::new(RwLock::new(0));
    let t = Instant::now();
    let hs: Vec<_> = (0..4).map(|_| {
        let rw = Arc::clone(&rw);
        thread::spawn(move || {
            let _g = rw.read().unwrap();
            thread::sleep(Duration::from_millis(100));
        })
    }).collect();
    for h in hs { h.join().unwrap(); }
    println!("4 readers took ~{} ms", t.elapsed().as_millis());

    let m = Arc::new(Mutex::new(0));
    let t = Instant::now();
    let hs: Vec<_> = (0..4).map(|_| {
        let m = Arc::clone(&m);
        thread::spawn(move || {
            let _g = m.lock().unwrap();
            thread::sleep(Duration::from_millis(100));
        })
    }).collect();
    for h in hs { h.join().unwrap(); }
    println!("4 mutex holders took ~{} ms", t.elapsed().as_millis());
}
```

Output (numbers vary a little; the ratio does not):

```
4 readers took ~105 ms
4 mutex holders took ~420 ms
```

Four 100 ms read sections overlap into ~100 ms; four mutex sections queue into
~400 ms.

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(10);
    let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();
    println!("two readers at once: {} {}", *r1, *r2);
    println!("write blocked while reading: {}", lock.try_write().is_err());
    drop((r1, r2));
    println!("write now available: {}", lock.try_write().is_ok());
}
```

Output:

```
two readers at once: 10 10
write blocked while reading: true
write now available: true
```

Taking `read()` twice **on the same thread** is allowed here but is not
guaranteed by the API and will deadlock under some platform implementations —
do not rely on it. `RwLock` is also not reentrant: `read()` then `write()` on
one thread deadlocks.

### Writer starvation

Std's `RwLock` delegates to the OS primitive, so fairness is platform-dependent.
Under a continuous stream of readers a writer can wait a long time (on some
platforms, indefinitely). An `RwLock` only pays for itself when reads
substantially outnumber writes *and* the critical section is long enough for the
extra bookkeeping to matter. Under mixed load a plain `Mutex` is often faster
and always more predictable — measure before switching.

## Mutex vs atomic vs channel

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let a = Arc::new(AtomicUsize::new(0));
    let m = Arc::new(Mutex::new(0usize));

    let hs: Vec<_> = (0..4).map(|_| {
        let (a, m) = (Arc::clone(&a), Arc::clone(&m));
        thread::spawn(move || {
            for _ in 0..1000 {
                a.fetch_add(1, Ordering::Relaxed);
                *m.lock().unwrap() += 1;
            }
        })
    }).collect();
    for h in hs { h.join().unwrap(); }

    println!("atomic {} mutex {}", a.load(Ordering::Relaxed), *m.lock().unwrap());
}
```

Output:

```
atomic 4000 mutex 4000
```

Same answer. Pick by shape, not by taste:

| Need | Use |
|---|---|
| one integer or flag, updated independently | atomic (14.5) |
| two or more fields that must stay consistent | `Mutex` |
| a map/vec read far more than written | `RwLock` |
| ownership handed between stages | channel (14.2) |

If a counter is the *only* shared state, a `Mutex` is pure overhead. If the
update spans several fields, an atomic cannot express it and a `Mutex` is the
only correct answer.

## parking_lot

The `parking_lot` crate offers drop-in `Mutex` and `RwLock` with no poisoning
(so `lock()` returns the guard directly, no `.unwrap()`), smaller structs, and a
fair, well-defined reader/writer policy instead of the platform's. It also has
`ReentrantMutex` and timed lock methods. Swapping it in is a one-line import
change plus deleting `.unwrap()` calls. Module 15 covers `tokio::sync::Mutex`,
which you need only when a lock is held across an `.await`.

## Quick Reference

| Item | Meaning |
|---|---|
| `Mutex::new(v)` | wraps `v`; `Sync` if `T: Send` |
| `m.lock()` | `LockResult<MutexGuard<T>>`; blocks |
| `m.try_lock()` | `Err(TryLockError::WouldBlock)` if held |
| `m.is_poisoned()` / `m.clear_poison()` | check / reset the poison flag |
| `m.get_mut()` / `m.into_inner()` | no lock needed; needs `&mut` / ownership |
| `RwLock::new(v)` | `Sync` if `T: Send + Sync` |
| `rw.read()` / `rw.write()` | shared / exclusive guard |
| `rw.try_read()` / `rw.try_write()` | non-blocking |
| `PoisonError::into_inner()` | take the guard despite poisoning |
| `Arc::clone(&x)` | +1 refcount, shared ownership for `'static` |
| guard drop | releases the lock; `drop(g)` to do it early |

## Common Mistakes

**Using `Rc<RefCell<T>>` across threads.** Neither is thread-safe; you get a
`Send`/`Sync` error (14.4). `Arc<Mutex<T>>` is the threaded equivalent.

**`Arc<Mutex<T>>` cloned *inside* the closure.** `Arc::clone` must happen in the
spawning thread, before `move` takes the original. Clone into a shadowing
`let counter = Arc::clone(&counter);` just above `thread::spawn`.

**Holding a guard across slow work.** `while let Some(x) = q.lock().unwrap().pop()`
keeps the lock for the whole body. Pop into a local, let the guard drop, then
work.

**Assuming no deadlocks because it compiles.** Rust prevents data races, not
deadlocks. Fix a global lock order.

**Reaching for `RwLock` by default.** It is slower than `Mutex` for writes and
can starve writers. Use it when reads dominate and you have measured.

## Exercises

1. Build an `Arc<Mutex<HashMap<String, u32>>>`, have four threads each increment
   the same three keys 100 times, and print the sorted map (every value 400).
2. Convert exercise 1 to `RwLock` and add two reader threads that print the
   length while the writers run.
3. Take the deadlock example and fix it by imposing a lock order. Confirm
   `holds both` prints twice.
4. Write `fn safe_lock<T>(m: &Mutex<T>) -> MutexGuard<'_, T>` that recovers from
   poisoning via `into_inner()` instead of panicking.
5. Measure it: increment a shared counter 100_000 times across four threads with
   `Mutex<u64>`, then with `AtomicU64`, and print both elapsed times.

---

[← 14.2 Channels and Message Passing](02-channels.md) · [Course Index](../README.md) · [14.4 Send, Sync, and Thread Safety →](04-send-sync.md)
