[← Course Index](../README.md)

# 11.4 — Arc and Mutex

## What You'll Learn

- `Arc<T>`: `Rc` with an atomic refcount, so it crosses threads
- `Arc<Mutex<T>>` for shared mutable state, and why `lock()` returns a `Result`
- `MutexGuard` drop timing — the deadlock you get from holding one too long
- `RwLock` for many readers / one writer

Module 14 covers the threading model itself — spawn, join, channels, scoped
threads. This lesson is about the pointer types.

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `sync.Mutex` + a shared pointer | `Arc<Mutex<T>>` — the data lives *inside* the lock |
| Python `threading.Lock` around a global | `Arc<Mutex<T>>`, no globals needed |
| JS: single-threaded, no locks | locks are required and the compiler enforces it |

The structural difference: in Go or Python a mutex sits *next to* the data and
nothing stops you reading the data without it. In Rust the data is *inside* the
mutex, so the only way to reach it is to lock.

## Arc vs Rc

Identical API, one difference: `Arc`'s refcount uses atomic instructions, so it
is `Send + Sync` and works across threads. Same size.

```rust
use std::rc::Rc;
use std::sync::Arc;

fn main() {
    println!("{} {}", size_of::<Rc<i32>>(), size_of::<Arc<i32>>());
    let a = Arc::new(String::from("shared"));
    let b = Arc::clone(&a);
    println!("{} {}", Arc::strong_count(&a), b);
}
```

Output:

```
8 8
2 shared
```

Atomic increments are not free — they're a locked instruction and a cache-line
bounce when threads contend. Prefer `Rc` inside a single thread; reach for `Arc`
when the value actually crosses one. In practice the difference rarely shows up
in profiles, so do not contort your design over it.

## Arc alone is not enough to mutate

`Arc<T>` is read-only, exactly like `Rc<T>`. For mutation you need a lock
inside it. First, what happens *without* the `Arc`:

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];
    for _ in 0..2 {
        handles.push(thread::spawn(move || {
            *counter.lock().unwrap() += 1;
        }));
    }
    for h in handles { h.join().unwrap(); }
}
```

Does not compile — intentional:

```
error[E0382]: use of moved value: `counter`
 --> src/main.rs:8:36
  |
5 |     let counter = Mutex::new(0);
  |         ------- move occurs because `counter` has type `std::sync::Mutex<i32>`, which does not implement the `Copy` trait
7 |     for _ in 0..2 {
  |     ------------- inside of this loop
8 |         handles.push(thread::spawn(move || {
  |                                    ^^^^^^^ value moved into closure here, in previous iteration of loop
```

One `Mutex` can only be moved into one closure. `Arc` is what gives each thread
its own handle to the same lock.

## Arc\<Mutex\<T\>\>: a real counter

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..8 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                *c.lock().unwrap() += 1;
            }
        }));
    }
    for h in handles {
        h.join().unwrap();
    }
    println!("final = {}", *counter.lock().unwrap());
    println!("count = {}", Arc::strong_count(&counter));
}
```

Output:

```
final = 8000
count = 1
```

8000 every run — the lock serializes the increments. The count is back to 1
because every worker's `Arc` dropped when its closure ended.

`let c = Arc::clone(&counter);` before the closure is the pattern you will write
constantly: clone outside, `move` the clone in.

## lock() returns a Result

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    {
        let mut g = m.lock().unwrap();
        *g += 1;
    }
    println!("{}", *m.lock().unwrap());
    println!("{}", m.try_lock().is_ok());
    println!("{:?}", m.into_inner().unwrap());
}
```

Output:

```
6
true
6
```

The `Result` is not about contention — `lock()` blocks until the lock is free.
It is about **poisoning**: if a thread panics while holding the guard, the data
may be half-updated, so every later `lock()` returns `Err`. `unwrap()` is the
normal choice (propagate the panic); handle the `Err` only if you can repair the
state.

`try_lock()` returns immediately with `Err` if the lock is taken.

## MutexGuard and drop timing

`lock()` returns a `MutexGuard<T>`, which unlocks on drop. Scope is everything.

```rust
let mut g = m.lock().unwrap();
*g += 1;
```

The guard lives to the end of the enclosing scope. These three forms are all
fine and all release immediately:

```rust
*m.lock().unwrap() += 1;

{
    let mut g = m.lock().unwrap();
    *g += 1;
}

let mut g = m.lock().unwrap();
*g += 1;
drop(g);
```

## The classic deadlock

Locking the same mutex twice on one thread. The second `lock()` waits for a
guard that only *this* thread could release.

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    let a = m.lock().unwrap();
    println!("got {}", *a);
    let b = m.lock().unwrap();
    println!("never prints {}", *b);
}
```

Compiles cleanly. Output — then it hangs forever:

```
got 5
```

No panic, no error, no timeout. You have to kill the process. This is the one
thing `RefCell` does better: it panics and tells you where.

In real code it looks like a helper that locks, called from a function that
already holds the lock:

```rust
fn total(m: &Mutex<Vec<i32>>) -> i32 {
    m.lock().unwrap().iter().sum()
}

let g = m.lock().unwrap();
println!("{}", total(&m));
```

Same hang. Rules that avoid it: never call anything while holding a guard, keep
guards to one statement, and if you must take two locks, always take them in
the same order everywhere.

## Poisoning in practice

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let m = Arc::new(Mutex::new(vec![1, 2, 3]));
    let m2 = Arc::clone(&m);
    let h = thread::spawn(move || {
        let _g = m2.lock().unwrap();
        panic!("worker died");
    });
    println!("{}", h.join().is_err());

    println!("is_poisoned = {}", m.is_poisoned());
    match m.lock() {
        Ok(g) => println!("ok {:?}", *g),
        Err(e) => {
            println!("err: {}", e);
            println!("recovered {:?}", *e.into_inner());
        }
    }
}
```

Output (the panic message comes from the worker thread; the id in parens will
differ):

```
thread '<unnamed>' (13887334) panicked at src/main.rs:9:9:
worker died
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
true
is_poisoned = true
err: poisoned lock: another task failed inside
recovered [1, 2, 3]
```

The data is still there — `e.into_inner()` hands you the guard anyway. Poisoning
is a warning, not a lock-out. Most code just `unwrap()`s and lets the panic
spread.

## RwLock: many readers, one writer

Use it when reads vastly outnumber writes — a config, a routing table, a cache.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let cfg = Arc::new(RwLock::new(String::from("v1")));

    let readers: Vec<_> = (0..3)
        .map(|i| {
            let c = Arc::clone(&cfg);
            thread::spawn(move || {
                let g = c.read().unwrap();
                format!("reader {} saw {}", i, *g)
            })
        })
        .collect();
    for r in readers {
        println!("{}", r.join().unwrap());
    }

    {
        let mut w = cfg.write().unwrap();
        w.push_str("-patched");
    }
    println!("{}", *cfg.read().unwrap());

    let r1 = cfg.read().unwrap();
    let r2 = cfg.read().unwrap();
    println!("two readers at once: {} {}", *r1, *r2);
    println!("try_write while reading: {}", cfg.try_write().is_err());
}
```

Output:

```
reader 0 saw v1
reader 1 saw v1
reader 2 saw v1
v1-patched
two readers at once: v1-patched v1-patched
try_write while reading: true
```

`read()` and `write()` both return `Result` and both poison, same as `Mutex`.
Two `read()` guards coexist; a `write()` excludes everything.

Caveat: `RwLock` is more expensive per-operation than `Mutex`. If writes are
frequent, or the critical section is tiny, a plain `Mutex` is usually faster.
And locking twice for read on one thread can deadlock on some platforms — the
same discipline applies.

## Which lock

| Need | Use |
|---|---|
| Shared, never mutated | `Arc<T>` — no lock at all |
| Shared, mutated, single thread | `Rc<RefCell<T>>` (lesson 11.3) |
| Shared, mutated, multi-thread | `Arc<Mutex<T>>` |
| Read-heavy, rarely written | `Arc<RwLock<T>>` |
| A single integer or flag | `Arc<AtomicUsize>` / `AtomicBool` — no lock |
| Write-once at startup | `OnceLock<T>` / `LazyLock<T>`, usable in a `static` |
| Hand off values, not share them | a channel (Module 14) |

`parking_lot` is the common third-party alternative: its `Mutex` and `RwLock` do
not poison, so `lock()` returns the guard directly rather than a `Result`, and
they are smaller and often faster. Worth it in a hot path; `std` is fine
everywhere else.

## Quick Reference

| Call | Returns |
|---|---|
| `Arc::new(v)` / `Arc::clone(&a)` | `Arc<T>` |
| `Arc::strong_count(&a)` | `usize` |
| `Arc::try_unwrap(a)` | `Result<T, Arc<T>>` |
| `Arc::get_mut(&mut a)` | `Option<&mut T>` |
| `Mutex::new(v)` | `Mutex<T>` |
| `m.lock()` | `LockResult<MutexGuard<T>>` — blocks |
| `m.try_lock()` | `TryLockResult<MutexGuard<T>>` — immediate |
| `m.is_poisoned()` | `bool` |
| `m.into_inner()` | `LockResult<T>` |
| `m.get_mut()` | `LockResult<&mut T>` — needs `&mut self`, no locking |
| `rw.read()` / `rw.write()` | guard, shared / exclusive |
| `err.into_inner()` | the guard from a poison error |

## Common Mistakes

**Holding a guard while calling a function that locks again.** Hangs with no
diagnostic. One statement per guard.

**`Arc<Mutex<T>>` where `Arc<T>` would do.** If nothing mutates, skip the lock.

**Reaching for `RwLock` by reflex.** It only pays off when reads dominate.
Measure before switching from `Mutex`.

**Cloning the `Arc` inside the closure.** `thread::spawn(move || { let c = Arc::clone(&counter); ... })`
moves `counter` itself. Clone *before* the closure.

**Treating a poison error as data loss.** `e.into_inner()` gives you the value.
The lock is flagged, not emptied.

## Exercises

1. Spawn four threads that each push 100 values into a shared
   `Arc<Mutex<Vec<u32>>>`. Join them and assert the length is 400.
2. Rewrite that counter with `Arc<AtomicUsize>` and `fetch_add(1, Ordering::Relaxed)`.
   Confirm the result matches.
3. Reproduce the double-lock deadlock, then fix it two ways: a tighter scope and
   an explicit `drop`.
4. Make a `Arc<RwLock<HashMap<String, u32>>>`, have three threads read while one
   writes, and print the final map.
5. Deliberately panic a thread while it holds a lock, then write the main thread
   so it recovers the data through `into_inner()` and continues.

---

[← Course Index](../README.md)
