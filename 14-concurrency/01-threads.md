[← 13.6 Property Testing and Fuzzing](../13-testing/06-property-testing.md) · [Course Index](../README.md) · [14.2 Channels and Message Passing →](02-channels.md)

# 14.1 — Threads and join

## What You'll Learn

- `thread::spawn`, `JoinHandle`, and what `join()` actually returns
- Why thread closures almost always need `move`
- Spawning in a loop, collecting handles, returning values
- Named threads, stack sizes, `available_parallelism()`, panics in threads

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `go f()` | `thread::spawn(f)` — a real OS thread, not a goroutine |
| JS `new Worker(...)` | `thread::spawn` — shares memory, no serialization |
| Python `threading.Thread` | `thread::spawn` — no GIL, threads actually run in parallel |

Rust threads are 1:1 OS threads. The difference from every language above is
that the compiler checks what you capture: data races are rejected at compile
time, not discovered at 3 AM. Module 15 covers async tasks, which are cheap and
many; threads are expensive and few.

**Thread output order is not reproducible.** Every example below either joins in
order or sorts before printing. Where output order genuinely varies, it says so.

## Spawn and join

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("hello from the spawned thread");
        42
    });

    println!("main keeps running");

    let value = handle.join().unwrap();
    println!("thread returned {value}");
}
```

Output (one run — the first two lines can swap):

```
main keeps running
hello from the spawned thread
thread returned 42
```

`spawn` starts the thread immediately and returns a `JoinHandle<T>`, where `T`
is the closure's return type. `join()` blocks until the thread finishes and
hands you that value.

## join returns a Result

```rust
use std::thread;

fn main() {
    let h = thread::spawn(|| 7 * 6);
    let r: std::thread::Result<i32> = h.join();
    println!("{:?}", r);
}
```

Output:

```
Ok(42)
```

`thread::Result<T>` is `Result<T, Box<dyn Any + Send>>`. The `Err` case means
the thread panicked, not that it failed in a business sense. `unwrap()` on it is
the normal idiom: it propagates the child's panic into the parent.

## The move closure

Borrowing a local in a thread closure does not compile.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let handle = thread::spawn(|| {
        println!("{:?}", data);
    });
    handle.join().unwrap();
}
```

Intentionally broken:

```
error[E0373]: closure may outlive the current function, but it borrows `data`, which is owned by the current function
 --> src/main.rs:5:32
  |
5 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `data`
6 |         println!("{:?}", data);
  |                          ---- `data` is borrowed here
  |
note: function requires argument type to outlive `'static`
help: to force the closure to take ownership of `data` (and any other referenced variables), use the `move` keyword
  |
5 |     let handle = thread::spawn(move || {
  |                                ++++
```

`spawn` requires `F: Send + 'static`. The thread may outlive the function that
started it, so it cannot hold a reference to that function's locals. `move`
transfers ownership in.

```rust
use std::thread;

fn main() {
    let s = String::from("owned by the thread");
    let h = thread::spawn(move || {
        println!("{s}");
        s.len()
    });
    println!("len = {}", h.join().unwrap());

    let nums = vec![10, 20, 30];
    let h = thread::spawn(move || nums.iter().sum::<i32>());
    println!("sum = {}", h.join().unwrap());
}
```

Output:

```
owned by the thread
len = 19
sum = 60
```

After `move`, `s` and `nums` are gone from `main` — touching them again is a
use-after-move error. Lesson 14.6 shows `thread::scope`, which lets you borrow
locals instead.

## Spawning in a loop

Push the handles into a `Vec`, then join them all.

```rust
use std::thread;

fn main() {
    let mut handles = Vec::new();
    for i in 0..5 {
        handles.push(thread::spawn(move || i * i));
    }

    let results: Vec<i32> = handles.into_iter().map(|h| h.join().unwrap()).collect();
    println!("{:?}", results);
}
```

Output:

```
[0, 1, 4, 9, 16]
```

Joining in handle order makes the *results* deterministic even though the
threads ran in an arbitrary order. The `move` captures `i` by value because
`i32` is `Copy`, so each thread gets its own copy.

Do not join inside the spawn loop — that runs them one at a time.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handles: Vec<_> = (0..4)
        .map(|i| {
            thread::spawn(move || {
                thread::sleep(Duration::from_millis(20 * (4 - i) as u64));
                format!("worker {i} finished")
            })
        })
        .collect();

    for h in handles {
        println!("{}", h.join().unwrap());
    }
}
```

Output:

```
worker 0 finished
worker 1 finished
worker 2 finished
worker 3 finished
```

Worker 0 slept the longest yet prints first — because the loop joins in order,
not in completion order.

## thread::sleep

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let h = thread::spawn(|| {
        thread::sleep(Duration::from_millis(50));
        "done sleeping"
    });
    println!("{}", h.join().unwrap());
}
```

Output:

```
done sleeping
```

`sleep` parks the current thread; it never blocks others. There is no
`Thread::sleep(handle)` — you can only sleep yourself.

## Detached threads die when main exits

Dropping a `JoinHandle` without joining detaches the thread. The process exits
when **main** returns, killing everything still running.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..=5 {
            println!("detached tick {i}");
            thread::sleep(Duration::from_millis(100));
        }
    });

    thread::sleep(Duration::from_millis(250));
    println!("main exits");
}
```

Output:

```
detached tick 1
detached tick 2
detached tick 3
main exits
```

Ticks 4 and 5 never happen. There is no `wait for all threads` at exit, unlike
the JVM's non-daemon threads. If you need the work finished, join.

## Thread identity

```rust
use std::thread;

fn main() {
    let t = thread::current();
    println!("main id={:?} name={:?}", t.id(), t.name());

    let h = thread::spawn(|| {
        println!("spawned name={:?}", thread::current().name());
    });
    h.join().unwrap();
}
```

Output:

```
main id=ThreadId(1) name=Some("main")
spawned name=None
```

`ThreadId` is opaque and comparable; the numbers are not stable across runs.
Threads from plain `spawn` are unnamed.

## Builder: names and stack size

```rust
use std::thread;

fn main() {
    let b = thread::Builder::new()
        .name("worker-1".to_string())
        .stack_size(32 * 1024);

    let h = b.spawn(|| {
        println!("inside {:?}", thread::current().name().unwrap());
    }).unwrap();

    h.join().unwrap();
}
```

Output:

```
inside "worker-1"
```

`Builder::spawn` returns `io::Result<JoinHandle<T>>` — spawning can fail when
the OS is out of thread resources, which plain `thread::spawn` papers over with
a panic. Names show up in panic messages and debuggers, which is reason enough
to use `Builder` for long-lived threads.

## available_parallelism

```rust
use std::thread;

fn main() {
    println!("{:?}", thread::available_parallelism());

    let n = thread::available_parallelism().map(|n| n.get()).unwrap_or(1);
    println!("spawning {n} workers");
}
```

Output on a 12-core machine:

```
Ok(12)
```
```
spawning 12 workers
```

It returns `io::Result<NonZeroUsize>` and respects cgroup/affinity limits, so it
is the right answer in a container — unlike reading the CPU count directly.

## Panics in threads

A panicking thread does not take the process down. `join()` returns `Err` with
the panic payload.

```rust
use std::thread;

fn main() {
    let h = thread::spawn(|| -> i32 {
        panic!("worker exploded");
    });

    match h.join() {
        Ok(v) => println!("ok: {v}"),
        Err(e) => {
            let msg = e.downcast_ref::<&str>().copied().unwrap_or("unknown");
            println!("thread panicked: {msg}");
        }
    }
    println!("main survived");
}
```

Output (the thread id in the panic line varies per run):

```
thread '<unnamed>' (13847662) panicked at src/main.rs:5:9:
worker exploded
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread panicked: worker exploded
main survived
```

The panic message is printed by the default hook as soon as it happens; the
payload is also handed to `join`. `downcast_ref::<&str>()` catches
`panic!("literal")`; use `downcast_ref::<String>()` for `panic!("{x}")`-style
formatted messages. `h.join().unwrap()` simply re-panics in the parent, which is
usually what you want.

A panic while holding a `Mutex` poisons it — lesson 14.3.

## Quick Reference

| Item | Signature / meaning |
|---|---|
| `thread::spawn(f)` | `F: FnOnce() -> T + Send + 'static` → `JoinHandle<T>` |
| `handle.join()` | `thread::Result<T>`; `Err` = the thread panicked |
| `handle.is_finished()` | `bool`, non-blocking check |
| drop a handle | detaches; thread dies at process exit |
| `thread::sleep(dur)` | parks the current thread |
| `thread::yield_now()` | hints the scheduler to run someone else |
| `thread::current()` | `Thread`: `.id()`, `.name()`, `.unpark()` |
| `thread::park()` | blocks until `unpark()` |
| `Builder::new()` | `.name(String)`, `.stack_size(usize)`, `.spawn(f) -> io::Result<_>` |
| `thread::available_parallelism()` | `io::Result<NonZeroUsize>` |
| `thread::scope(f)` | scoped threads that may borrow locals (14.6) |

## Common Mistakes

**Forgetting `move`.** `E0373`; the closure borrows a local that may not outlive
the thread. Add `move`, or use `thread::scope` (14.6) when you must borrow.

**Joining inside the spawn loop.** `for i in 0..n { spawn(...).join() }` is
sequential code with extra steps. Collect handles first, join second.

**Assuming interleaved print order is stable.** It is not, on any platform. Join
in order or sort the results before printing.

**Expecting detached threads to finish.** The process exits with `main`. Join
anything whose output you depend on.

**Spawning a thread per item.** Each thread costs an OS stack (megabytes of
virtual address space) and a context switch. For a thousand jobs, use a pool or
rayon (14.6); for a thousand IO waits, use async (Module 15).

## Exercises

1. Spawn three threads that each return the length of a `String` moved into
   them. Join in order and print the total.
2. Write `fn parallel_sum(v: Vec<u64>) -> u64` that splits the vec into four
   owned chunks, sums each in its own thread, and adds the results.
3. Spawn a thread that panics with a formatted message (`panic!("bad id {id}")`)
   and recover the text in `main` via `downcast_ref::<String>()`.
4. Use `Builder` to spawn `available_parallelism()` named threads
   (`"w-0"`, `"w-1"`, …), each printing its own name, and join them all.
5. Spawn a thread that loops printing forever, do not join it, and confirm the
   process still exits when `main` returns. Then make it finish by joining.

---

[← 13.6 Property Testing and Fuzzing](../13-testing/06-property-testing.md) · [Course Index](../README.md) · [14.2 Channels and Message Passing →](02-channels.md)
