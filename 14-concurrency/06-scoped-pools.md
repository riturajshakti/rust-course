[← 14.5 Atomics](05-atomics.md) · [Course Index](../README.md) · [15.1 async/await and Futures →](../15-async-tokio/01-async-await.md)

# 14.6 — Scoped Threads and Thread Pools

## What You'll Learn

- `thread::scope`: threads that may borrow locals, and why `spawn` cannot
- Splitting a slice across threads with `chunks_mut`
- rayon: `par_iter`, `join`, `scope`, and real speedup numbers
- Building a small thread pool with channels, and when to use which tool

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `var wg sync.WaitGroup; wg.Wait()` | `thread::scope` — the join is structural, not manual |
| Python `ThreadPoolExecutor` | `rayon::ThreadPoolBuilder` / the hand-rolled pool below |
| JS `Promise.all(workers)` | `scope` returning a `Vec` of results |
| OpenMP `#pragma omp parallel for` | `v.par_iter_mut().for_each(...)` |

Scoped threads remove the `'static` bound by guaranteeing the threads finish
before the scope does. rayon removes the thread management entirely for
data-parallel work.

## Why spawn needs 'static

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let h = thread::spawn(|| println!("{:?}", data));
    h.join().unwrap();
}
```

Intentionally broken:

```
error[E0373]: closure may outlive the current function, but it borrows `data`, which is owned by the current function
 --> src/main.rs:5:27
  |
5 |     let h = thread::spawn(|| println!("{:?}", data));
  |                           ^^                  ---- `data` is borrowed here
  |                           |
  |                           may outlive borrowed value `data`
  |
note: function requires argument type to outlive `'static`
```

You *know* the join is on the next line, but `spawn`'s signature does not: it
returns a handle you are allowed to drop, which detaches the thread. So it must
assume the thread outlives everything.

## thread::scope

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    let label = String::from("sum");

    thread::scope(|s| {
        s.spawn(|| {
            println!("{label} = {}", data.iter().sum::<i32>());
        });
        s.spawn(|| {
            println!("max = {}", data.iter().max().unwrap());
        });
    });

    println!("data still usable: {:?}", data);
}
```

Output (the first two lines can appear in either order):

```
sum = 15
max = 5
data still usable: [1, 2, 3, 4, 5]
```

No `move`, no `Arc`, no clone — both closures borrow `data` and `label`
immutably, and `data` is still owned by `main` afterwards. `scope` joins every
thread that has not been joined before it returns, so the borrows provably end
first. `s.spawn` requires only `Send`, not `'static`.

A panic in a scoped thread propagates: `scope` re-panics in the parent once all
the threads have been joined.

## Scope with results

`thread::scope` returns whatever its closure returns, so you can collect out of
it.

```rust
use std::thread;

fn main() {
    let text = String::from("the quick brown fox jumps over the lazy dog");
    let words: Vec<&str> = text.split_whitespace().collect();

    let counts: Vec<(usize, usize)> = thread::scope(|s| {
        let handles: Vec<_> = words
            .chunks(3)
            .enumerate()
            .map(|(i, chunk)| s.spawn(move || (i, chunk.iter().map(|w| w.len()).sum())))
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).collect()
    });

    println!("{:?}", counts);
    println!("total {}", counts.iter().map(|(_, n)| n).sum::<usize>());
}
```

Output:

```
[(0, 13), (1, 12), (2, 10)]
total 35
```

The `move` on the inner closure moves the `chunk` *reference* (a `&[&str]`,
`Copy`) into the thread — that is not the same as moving `words`. Collecting the
handles into a `Vec` first and joining second keeps the threads concurrent, and
joining in order makes the output deterministic.

## Splitting a slice with chunks_mut

The payoff case: four threads writing into disjoint pieces of one `Vec`.

```rust
use std::thread;

fn main() {
    let mut data: Vec<u64> = (1..=12).collect();

    thread::scope(|s| {
        for chunk in data.chunks_mut(4) {
            s.spawn(|| {
                for v in chunk.iter_mut() {
                    *v *= *v;
                }
            });
        }
    });

    println!("{:?}", data);
}
```

Output:

```
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100, 121, 144]
```

`chunks_mut` yields non-overlapping `&mut [u64]`s, so the borrow checker accepts
three simultaneous mutable borrows of pieces of one `Vec`. This is impossible
with `spawn` without `Arc<Mutex<Vec<_>>>` and the serialization that implies.
`split_at_mut` is the two-way version.

## rayon: the data-parallel answer

Add `rayon` and most iterator chains become parallel by changing `iter` to
`par_iter`.

```rust
use rayon::prelude::*;

fn main() {
    let v: Vec<u64> = (1..=10_000).collect();

    let sum: u64 = v.par_iter().sum();
    println!("sum {sum}");

    let squares: Vec<u64> = v.par_iter().map(|x| x * x).collect();
    println!("first 5 {:?}", &squares[..5]);
    println!("len {}", squares.len());

    let evens: Vec<&u64> = v.par_iter().filter(|x| **x % 2 == 0).collect();
    println!("evens {}", evens.len());

    println!("max {}", v.par_iter().max().unwrap());
    println!("divisible by 7: {}", v.par_iter().filter(|x| **x % 7 == 0).count());

    let mut sorted: Vec<u64> = v.iter().rev().copied().collect();
    sorted.par_sort();
    println!("sorted head {:?}", &sorted[..3]);

    let (a, b) = rayon::join(
        || (1..=1000u64).sum::<u64>(),
        || (1..=1000u64).map(|x| x * x).sum::<u64>(),
    );
    println!("join {a} {b}");

    println!("threads {}", rayon::current_num_threads());
}
```

Output (thread count is machine-dependent):

```
sum 50005000
first 5 [1, 4, 9, 16, 25]
len 10000
evens 5000
max 10000
divisible by 7: 1428
sorted head [1, 2, 3]
join 500500 333833500
threads 12
```

`collect` preserves order even though the work happened out of order — rayon
splits the range, not the output. `sum`, `max`, `count` and `collect` are all
order-independent by construction.

### Real speedup

```rust
use rayon::prelude::*;

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    let mut d = 2;
    while d * d <= n {
        if n % d == 0 { return false; }
        d += 1;
    }
    true
}

fn main() {
    let n = 300_000u64;

    let t = std::time::Instant::now();
    let serial = (2..n).filter(|&x| is_prime(x)).count();
    let serial_ms = t.elapsed().as_millis();

    let t = std::time::Instant::now();
    let parallel = (2..n).into_par_iter().filter(|&x| is_prime(x)).count();
    let parallel_ms = t.elapsed().as_millis();

    println!("serial   {serial} primes in {serial_ms} ms");
    println!("parallel {parallel} primes in {parallel_ms} ms");
    println!("threads  {}", rayon::current_num_threads());
}
```

Output (`--release`, 12 logical cores; timings vary):

```
serial   25997 primes in 20 ms
parallel 25997 primes in 2 ms
threads  12
```

`into_par_iter` on a `Range` is the parallel `into_iter`. rayon's work-stealing
scheduler handles the uneven cost per item automatically — small `n` finish fast
and those threads steal the large ones.

In debug builds rayon often looks *slower* than serial code. Always benchmark
with `--release`.

### rayon::scope and a custom pool

```rust
use rayon::prelude::*;

fn main() {
    let mut a = vec![0u32; 4];
    let mut b = vec![0u32; 4];

    rayon::scope(|s| {
        s.spawn(|_| a.iter_mut().enumerate().for_each(|(i, v)| *v = i as u32 + 1));
        s.spawn(|_| b.iter_mut().enumerate().for_each(|(i, v)| *v = (i as u32 + 1) * 10));
    });
    println!("{:?} {:?}", a, b);

    let pool = rayon::ThreadPoolBuilder::new().num_threads(2).build().unwrap();
    let sum = pool.install(|| (1..=100u64).into_par_iter().sum::<u64>());
    println!("pool of {} computed {}", pool.current_num_threads(), sum);

    let words = vec!["alpha", "beta", "gamma", "delta"];
    let upper: Vec<String> = words.par_iter().map(|w| w.to_uppercase()).collect();
    println!("{:?}", upper);

    let total: usize = words.par_iter().map(|w| w.len()).reduce(|| 0, |a, b| a + b);
    println!("total chars {total}");
}
```

Output:

```
[1, 2, 3, 4] [10, 20, 30, 40]
pool of 2 computed 5050
["ALPHA", "BETA", "GAMMA", "DELTA"]
total chars 19
```

`rayon::scope` is the task-parallel escape hatch when your work is not an
iterator — same borrowing rules as `thread::scope`, but running on rayon's
existing pool instead of spawning OS threads. `rayon::join(a, b)` is the
two-task version and the primitive behind divide-and-conquer recursion.
`reduce` needs an identity closure because rayon combines partial results in an
arbitrary tree shape.

By default rayon uses one global pool sized to `available_parallelism()`.
`ThreadPoolBuilder` lets you size, name, or isolate one — useful to keep a
latency-sensitive path off the pool a batch job is saturating.

## Building a thread pool

Worth doing once to see that there is no magic: a channel of boxed closures, N
workers taking from it, and a `Drop` that shuts down cleanly.

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;

type Job = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    workers: Vec<thread::JoinHandle<()>>,
    tx: Option<mpsc::Sender<Job>>,
}

impl ThreadPool {
    fn new(size: usize) -> Self {
        let (tx, rx) = mpsc::channel::<Job>();
        let rx = Arc::new(Mutex::new(rx));
        let workers = (0..size)
            .map(|id| {
                let rx = Arc::clone(&rx);
                thread::spawn(move || loop {
                    let job = rx.lock().unwrap().recv();
                    match job {
                        Ok(job) => job(),
                        Err(_) => {
                            println!("worker {id} stopping");
                            break;
                        }
                    }
                })
            })
            .collect();
        ThreadPool { workers, tx: Some(tx) }
    }

    fn execute<F: FnOnce() + Send + 'static>(&self, f: F) {
        self.tx.as_ref().unwrap().send(Box::new(f)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.tx.take());
        for w in self.workers.drain(..) {
            w.join().unwrap();
        }
    }
}

fn main() {
    let results = Arc::new(Mutex::new(Vec::new()));
    {
        let pool = ThreadPool::new(3);
        for i in 1..=6u64 {
            let results = Arc::clone(&results);
            pool.execute(move || {
                results.lock().unwrap().push((i, i * i * i));
            });
        }
    }
    let mut out = results.lock().unwrap().clone();
    out.sort();
    println!("{:?}", out);
}
```

Output (the three `worker N stopping` lines appear in a random order):

```
worker 0 stopping
worker 1 stopping
worker 2 stopping
[(1, 1), (2, 8), (3, 27), (4, 64), (5, 125), (6, 216)]
```

Four details carry the whole design:

- `Box<dyn FnOnce() + Send + 'static>` is the job type. `FnOnce` because a job
  runs once; `Box<dyn>` because every closure has a different concrete type.
- `Arc<Mutex<Receiver>>` shares the single consumer. `rx.lock().unwrap().recv()`
  on one line matters: the guard is a temporary that drops at the end of the
  statement, so a worker does not hold the queue lock while running `job()`.
- `tx: Option<Sender>` exists so `Drop` can `take()` it. Dropping the sender
  disconnects the channel, every blocked `recv()` returns `Err`, and the workers
  break.
- The `Drop` impl joins, so leaving the `{ }` block waits for all six jobs. That
  is why the results are complete before they are printed.

This is the right exercise and the wrong production choice. Use the
`threadpool` crate for exactly this shape, or rayon when the work is
data-parallel — rayon's work-stealing beats a shared `Mutex` queue under load,
since this pool funnels every job through one lock.

## Choosing a tool

| Situation | Use |
|---|---|
| A few fixed tasks over borrowed locals | `thread::scope` |
| `for` loop over a collection, CPU-bound | `rayon` `par_iter` |
| Divide and conquer (sort, tree walk) | `rayon::join` |
| Non-iterator task parallelism | `rayon::scope` |
| Queue of heterogeneous jobs | `threadpool` crate, or the pool above |
| Thousands of concurrent network/file waits | async (Module 15) |
| One long-lived background worker | `thread::spawn` + a channel (14.2) |

### CPU-bound vs IO-bound

**CPU-bound** work — parsing, hashing, compression, image processing, numeric
loops — scales with cores. Use about `available_parallelism()` threads; more
just adds context switches. rayon's default pool is already sized this way.
Oversubscribing hurts.

**IO-bound** work — HTTP requests, database queries, file reads — spends its
time waiting. A thread blocked in `read()` is an OS stack doing nothing.
Ten thousand threads will not work; ten thousand async tasks will, because each
costs a few hundred bytes instead of a megabyte of stack. That is Module 15.

The mixed case is common: an async server that needs CPU-heavy work does it on a
blocking pool (`tokio::task::spawn_blocking`) or hands it to rayon, keeping the
async threads free to poll. Lesson 15.8 covers that boundary in detail.

## Quick Reference

| Item | Meaning |
|---|---|
| `thread::scope(\|s\| { ... })` | all scoped threads joined before it returns |
| `s.spawn(f)` | `F: Send` only — **no `'static`**; may borrow locals |
| scope return value | whatever the closure returns |
| panic in a scoped thread | re-panics in the parent after joining |
| `slice.chunks_mut(n)` / `split_at_mut(i)` | disjoint `&mut` pieces for parallel writes |
| `v.par_iter()` / `par_iter_mut()` / `into_par_iter()` | rayon's parallel iterators |
| `par_sort()` / `par_sort_unstable()` | parallel sorts |
| `rayon::join(a, b)` | run two closures, maybe in parallel |
| `rayon::scope(\|s\| s.spawn(\|_\| ...))` | task parallelism on rayon's pool |
| `par_iter().reduce(id, op)` | fold with an identity, arbitrary tree order |
| `rayon::current_num_threads()` | size of the active pool |
| `ThreadPoolBuilder::new().num_threads(n).build()` | an isolated pool; `pool.install(f)` |

## Common Mistakes

**Reaching for `Arc<Mutex<_>>` when `thread::scope` would do.** If the threads
finish inside the function, borrow instead of sharing ownership.

**Benchmarking rayon in debug mode.** Unoptimized closures swamp the
parallelism. Measure with `--release`.

**`par_iter` on tiny or cheap work.** Splitting and joining has a cost; a
parallel `map(|x| x + 1)` over 100 items is slower than serial. Parallelize when
the per-item work is real.

**Holding a lock inside a parallel loop.** `par_iter().for_each(|x| v.lock()...)`
serializes everything you just parallelized. `map` into a `Vec` and merge, or
use `reduce`.

**One thread per IO wait.** Threads are for CPU work. Use async for IO
concurrency (Module 15).

## Exercises

1. Use `thread::scope` to compute the min, max, and mean of a borrowed
   `&[f64]` in three threads, printing all three from `main`.
2. Parallelize an in-place `Vec<u8>` XOR-with-key using `chunks_mut` and
   `thread::scope`. Verify against a serial version.
3. Count how many lines in a `Vec<String>` contain `"error"` with `par_iter`,
   and time it against `iter` on 1,000,000 lines in `--release`.
4. Implement a parallel merge sort with `rayon::join`, recursing serially below
   a length of 1000.
5. Extend the `ThreadPool` above with `execute_with_result<T: Send + 'static>`
   that returns a `mpsc::Receiver<T>` for the job's value.

---

[← 14.5 Atomics](05-atomics.md) · [Course Index](../README.md) · [15.1 async/await and Futures →](../15-async-tokio/01-async-await.md)
