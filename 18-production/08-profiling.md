[← 18.7 CI with GitHub Actions](07-ci.md) · [Course Index](../README.md) · [18.9 The Essential Crate Directory →](09-crate-directory.md)

# 18.8 — Performance Profiling

## What You'll Learn

- Measure first: the debug-vs-release gap, measured, before anything else
- `criterion` for micro, flamegraphs for macro, `samply` on macOS, `perf` on Linux
- `tokio-console` for stalled tasks, `dhat` for allocations
- A worked optimization with real numbers, and when to stop

## Coming From Another Language

| Other | Rust |
|---|---|
| Chrome DevTools profiler | `samply` (macOS/Linux) or `cargo flamegraph` |
| `console.time` | `Instant::now()` / `criterion` |
| `cProfile` | `perf` + flamegraph; there is no built-in profiler |
| async hangs via event-loop lag | `tokio-console` |

One Rust-specific trap dominates everything: **profiling a debug build**. There
is no equivalent in JS or Python, where the "unoptimized" build is the only
build. Here it can be 10x off and it reorders your priorities.

## Rule zero: --release

Same binary, same workload, both profiles. Six implementations of the same
aggregation, best of 7 runs over 200,000 lines:

| Implementation | Debug | Release | Ratio |
|---|---|---|---|
| v0 baseline | 183.35 ms | 46.02 ms | 4.0x |
| v1 no input clone | 178.19 ms | 45.66 ms | 3.9x |
| v2 no temp Vecs | 122.26 ms | 11.47 ms | 10.7x |
| v3 entry API | 85.09 ms | 9.98 ms | 8.5x |
| v4 `&str` keys | 80.10 ms | 6.25 ms | 12.8x |
| v5 + `FxHashMap` | 69.57 ms | 5.32 ms | 13.1x |

Read the two numbers that matter: the **fully hand-optimized debug build
(69.57 ms) is still slower than the completely naive release build (46.02 ms)**.
Every optimization in this lesson, applied to a debug build, loses to typing
`--release`.

The ratio also *varies* between implementations — 3.9x to 13.1x — so a debug
profile does not even rank your hot spots correctly. It will point you at code
that the optimizer eliminates entirely.

```bash
cargo build --release
cargo run --release
cargo bench            # already uses an optimized profile
```

For profiling you need symbols too, so use the custom profile from 18.5:

```toml
[profile.release-with-debug]
inherits = "release"
debug = 1
strip = "none"
```

```bash
cargo build --profile release-with-debug
```

Identical optimization to release, symbols retained. Never profile with
`[profile.release] opt-level = 0` "to make it readable" — you are profiling a
different program.

## The discipline

1. **Measure.** Get a number for the thing the user complains about.
2. **Profile.** Find where that number goes. Do not guess.
3. **Change one thing.**
4. **Measure again.** Keep it only if the number moved.
5. **Stop** when you hit the target.

The step people skip is 2. Rust makes guessing especially unreliable because the
optimizer removes a great deal of apparent cost, and because for a web service
the answer is usually "waiting on Postgres," which no amount of `clone()` removal
will fix.

## criterion for micro-benchmarks

Lesson 13.5 covers criterion properly; here is the profiling use.

```toml
[dev-dependencies]
criterion = { version = "0.8.2", features = ["html_reports"] }

[[bench]]
name = "parse"
harness = false
```

```rust
use criterion::{criterion_group, criterion_main, Criterion, black_box};

fn bench(c: &mut Criterion) {
    let data = input(20_000);
    let mut g = c.benchmark_group("aggregate");
    g.bench_function("baseline", |b| b.iter(|| baseline(black_box(&data))));
    g.bench_function("optimized", |b| b.iter(|| optimized(black_box(&data))));
    g.finish();
}

criterion_group!(benches, bench);
criterion_main!(benches);
```

```bash
cargo bench --bench parse
```

Real output:

```
Benchmarking aggregate/baseline: Warming up for 3.0000 s
Benchmarking aggregate/baseline: Collecting 100 samples in estimated 5.1770 s (1500 iterations)
aggregate/baseline      time:   [3.4548 ms 3.4794 ms 3.5081 ms]
Found 6 outliers among 100 measurements (6.00%)
  1 (1.00%) high mild
  5 (5.00%) high severe

aggregate/optimized     time:   [573.19 µs 575.80 µs 578.45 µs]
```

The three numbers are a confidence interval: lower bound, estimate, upper bound.
3.48 ms against 576 µs is a 6.0x improvement, and the intervals do not overlap,
so it is real rather than noise.

`black_box` is mandatory. Without it LLVM can see the result is unused and
delete the entire computation, and you benchmark an empty loop.

Run it again and criterion compares against the stored baseline:

```
aggregate/baseline      time:   [3.4430 ms 3.4563 ms 3.4700 ms]
                        change: [−1.5850% −0.6640% +0.1322%] (p = 0.14 > 0.05)
                        No change in performance detected.

aggregate/optimized     time:   [604.75 µs 621.74 µs 642.18 µs]
                        change: [+1.6276% +3.5187% +6.1855%] (p = 0.00 < 0.05)
                        Performance has regressed.
```

That second verdict is instructive: nothing changed in the code between runs.
A 3.5% "regression" with p < 0.05 was machine noise — another process, thermal
state, cache residency. Treat sub-10% criterion results on a laptop with
suspicion, and re-run before believing them. This is also why the measurements
in this lesson quote best-of-N rather than a single run.

## Flamegraphs

A flamegraph is a sampling profiler's output folded into a picture: each box is
a function, its **width** is the share of samples in which it was on the stack,
and boxes stack upward as callers below callees.

```bash
cargo install flamegraph
cargo flamegraph --profile release-with-debug --bin myapi
cargo flamegraph --profile release-with-debug --bench parse -- --bench
```

How to read one:

- **Width is everything. Height is not.** A tall narrow tower is deep recursion
  costing nothing. A wide flat box is where your time goes.
- Look for a **wide box near the top** — a leaf doing the actual work.
- **The x-axis is not time.** It is alphabetically sorted samples; adjacent
  boxes are not sequential events.
- A wide `memcpy`, `malloc`, `__rust_alloc` or `realloc` means allocation
  pressure — see the `dhat` section.
- A wide `core::fmt` means `format!` in a hot path.
- Wide `HashMap`/`SipHash` hashing means try `FxHashMap`.
- A wide box you do not recognise: search the function name, it is usually a
  dependency doing something you can avoid.

`--inverted` and `--reverse` aggregate by leaf rather than by root, which is
better for "what is allocating" than "who called what".

## perf on Linux, samply on macOS

**This machine is macOS on arm64 (Apple M4), so let me be precise about what
actually runs here.**

| Tool | macOS arm64 | Linux | Notes |
|---|---|---|---|
| `perf` | **no** — Linux-only | yes | `cargo flamegraph` uses it on Linux |
| `valgrind` / `callgrind` | **no** (no modern arm64 macOS support) | yes | |
| `dtrace` | binary present, but needs SIP disabled | — | `cargo flamegraph` can use it |
| Xcode Instruments | yes, with Xcode installed | no | GUI, Time Profiler template |
| **`samply`** | **yes** | yes | cross-platform, no sudo, no SIP changes |
| `criterion` | yes | yes | pure Rust |
| `dhat` | yes | yes | pure Rust |
| `tokio-console` | yes | yes | pure Rust |

Confirmed on this machine: `which perf valgrind` → both not found, and
`xcrun -f xctrace` → "unable to find utility". `samply` installed and ran.

On Linux:

```bash
sudo sysctl -w kernel.perf_event_paranoid=1
perf record --call-graph dwarf ./target/release-with-debug/myapi
perf report
```

On macOS — and on Linux too, it is the better default either way:

```bash
cargo install samply
samply record ./target/release-with-debug/myapi 400000
```

Real run on this machine:

```
400000 lines, best of 7

v0 baseline                 88.96 ms   keys=40 total=1199997
v1 no input clone           78.73 ms   keys=40 total=1199997
v2 no temp Vecs             21.97 ms   keys=40 total=1199997
v3 entry API                19.05 ms   keys=40 total=1199997
v4 &str keys                11.09 ms   keys=40 total=1199997
v5 + FxHashMap               9.31 ms   keys=40 total=1199997
Stopping profile.
```

`samply` then opens the Firefox Profiler UI in your browser with the recorded
samples. That UI is where **symbolication** happens — it resolves addresses to
function names using the debug info in your binary, which is why
`release-with-debug` is required. (The raw saved JSON from `--save-only` contains
unsymbolized addresses, so inspect profiles in the UI, not by grepping the file.)

The UI gives you a call tree, an inverted call tree, a flame graph and a
timeline you can select a range in. For an async service, select just the slow
request's window and read the inverted tree.

`samply` needs no `sudo`, no SIP changes and no Xcode. It is the right first
profiler on macOS.

## tokio-console for async stalls

A sampling profiler shows CPU. It is blind to a task that is *waiting* — which
is almost every async bug. `tokio-console` shows task state instead.

```toml
console-subscriber = "0.5.0"
tokio = { version = "1.54.1", features = ["full", "tracing"] }
```

```rust
console_subscriber::init();
```

Requires the `tokio_unstable` cfg:

```bash
RUSTFLAGS="--cfg tokio_unstable" cargo run --release
```

Or in `.cargo/config.toml`:

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

Then, in another terminal:

```bash
cargo install tokio-console
tokio-console
```

What to look for:

- **Busy time** high on one task: it is CPU-bound and starving the worker. It
  belongs in `spawn_blocking` (15.8).
- A **"task has lost its waker"** warning: it will never be polled again. A
  hang.
- A **poll time** warning: a single `poll` took too long, i.e. blocking code in
  an async function — the classic `std::fs` or `std::thread::sleep` in a handler.
- **Idle** tasks growing without bound: a leak, usually a spawn in a loop with
  nothing joining them.
- High **scheduled** time: tasks are ready but no worker is free. You are
  CPU-saturated.

This is the tool for "the service is slow but CPU is at 10%", which no
flamegraph will explain.

## Heap profiling with dhat

```toml
dhat = "0.3.3"

[features]
dhat-heap = []

[profile.release]
debug = 1
```

```rust
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();
    // ...
}
```

Behind a feature flag because `dhat::Alloc` is much slower than the system
allocator — never ship it.

```bash
cargo run --release --features dhat-heap
```

Two implementations of the same line-splitting work, measured:

```
=== wasteful ===
wasteful: 326670
dhat: Total:     3,327,639 bytes in 119,996 blocks
dhat: At t-gmax: 880,996 bytes in 20,004 blocks
dhat: At t-end:  1,088 bytes in 2 blocks

=== lean ===
lean: 326670
dhat: Total:     1,080,965 bytes in 39,996 blocks
dhat: At t-gmax: 880,992 bytes in 20,004 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

The difference is `Vec<String>` per line versus iterating `&str`: **3.33 MB in
119,996 allocations down to 1.08 MB in 39,996** — a third of the bytes and a
third of the calls, from deleting one `.map(|s| s.to_string()).collect()`.

Three figures to read:

| Line | Meaning |
|---|---|
| `Total` | every byte ever allocated — allocator *throughput* |
| `At t-gmax` | peak live heap — your actual memory requirement |
| `At t-end` | still live at exit; large means a leak |

Note `t-gmax` is identical (880 KB) in both: the peak is the input `Vec` either
way. What changed is churn, and churn is CPU time in `malloc`. A high `Total`
with a low `t-gmax` is the signature of a hot loop allocating and freeing — the
most common fixable Rust performance problem.

`dhat-heap.json` opens in dhat's `dh_view.html` for per-call-site attribution.

On Linux, `valgrind --tool=massif` and `heaptrack` are alternatives;
`valgrind` does not run on modern arm64 macOS.

## Common wins, in priority order

1. **Build with `--release`.** 4-13x on the table above. Everything else is a
   rounding error next to this.
2. **Remove allocation from hot loops.** Clones of inputs, `format!` for keys,
   `.collect()` into a temporary you then iterate once. Measured below: 4x.
3. **`&str` over `String`, `&[T]` over `Vec<T>` in signatures.** A function
   taking `&str` can be called with either and allocates nothing. Measured
   below: 1.6x on top of everything else.
4. **`with_capacity` when the size is known.**

   ```
   Vec::new()           2.42 ms
   Vec::with_capacity() 0.97 ms
   ratio 2.5x
   ```

   2.5x for pushing 2,000,000 `u64`s — it avoids ~21 reallocations and copies.
   But honestly, on 200,000 small string pushes:

   ```
   String::new()           0.68 ms
   String::with_capacity() 0.73 ms
   ```

   No benefit, slightly worse. Growth is amortized and doubling is cheap. Use
   `with_capacity` when you know the size, do not go hunting for it.
5. **Buffer your IO.** The largest single factor measured in this lesson:

   ```
   write  unbuffered: 389.886875ms   buffered: 3.604ms  -> 108x
   read   unbuffered: 999.128541ms   buffered: 7.243667ms  -> 138x
   ```

   200,000 lines. `File` is an unbuffered syscall per write; `BufWriter`/
   `BufReader` batch into 8 KB chunks. **108x and 138x.** Unlike every other
   item on this list, forgetting this is catastrophic rather than marginal. (The
   unbuffered read here was byte-at-a-time, the worst case — but that is exactly
   what naive code does.) Note `TcpStream` is also unbuffered; so is stdout when
   not a terminal.
6. **`FxHashMap` for hot maps with cheap keys.** Measured on the aggregation:
   6.25 ms → 5.32 ms, ~15%. `std`'s `HashMap` uses SipHash, which is
   DoS-resistant and slower. `rustc-hash` is faster and **not** DoS-resistant —
   never use it for a map keyed by user input.
7. **Generics over `Box<dyn Trait>` in hot paths.** `impl Trait`/`<T: Trait>`
   static-dispatches and inlines; `Box<dyn Trait>` is a vtable call that cannot
   inline. The cost per call is small, so this matters only in a tight loop —
   and `dyn` reduces code size and compile time, which is often the better
   trade. Do not rewrite an architecture for this without a measurement.
8. **`rayon` for CPU-bound parallel work.** Measured:

   ```
   serial:   33.348333ms
   parallel: 3.085708ms
   speedup:  10.8x on 12 threads
   ```

   `.iter()` → `.par_iter()`, one line. **Not for IO-bound async work** — rayon
   is a CPU thread pool and will fight your Tokio runtime. Use it inside
   `spawn_blocking` if you need it from a handler.

Below all of these: the real answer for a web service is usually a missing
database index or an N+1 query. Check the database before you optimize Rust.

## A worked optimization

Aggregating scores by email domain from 200,000 CSV lines. Release build, best
of 7, real numbers.

v0, the way it gets written first:

```rust
fn v0_baseline(lines: &Vec<String>) -> HashMap<String, u64> {
    let mut counts: HashMap<String, u64> = HashMap::new();
    for line in lines.clone() {
        let parts: Vec<String> = line.split(',').map(|s| s.to_string()).collect();
        let domain = parts[0].split('@').collect::<Vec<&str>>()[1].to_string();
        let score: u64 = parts[2].parse().unwrap_or(0);
        let key = format!("{}", domain);
        let entry = counts.get(&key).cloned().unwrap_or(0);
        counts.insert(key, entry + score);
    }
    counts
}
```

v4, after four changes:

```rust
fn v4_borrowed_keys(lines: &[String]) -> HashMap<&str, u64> {
    let mut counts: HashMap<&str, u64> = HashMap::with_capacity(64);
    for line in lines {
        let mut it = line.split(',');
        let Some(email) = it.next() else { continue };
        let Some(domain) = email.split('@').nth(1) else { continue };
        let Some(score) = it.nth(1).and_then(|s| s.parse::<u64>().ok()) else { continue };
        *counts.entry(domain).or_insert(0) += score;
    }
    counts
}
```

Step by step:

```
200000 lines, best of 7

v0 baseline                 46.02 ms   keys=40 total=599994
v1 no input clone           45.66 ms   keys=40 total=599994
v2 no temp Vecs             11.47 ms   keys=40 total=599994
v3 entry API                 9.98 ms   keys=40 total=599994
v4 &str keys                 6.25 ms   keys=40 total=599994
v5 + FxHashMap               5.32 ms   keys=40 total=599994
```

| Step | Change | Time | Gain |
|---|---|---|---|
| v0 | baseline | 46.02 ms | — |
| v1 | `&[String]`, iterate by reference | 45.66 ms | **1.01x** |
| v2 | no intermediate `Vec<String>`/`Vec<&str>` | 11.47 ms | **4.0x** |
| v3 | `entry().or_insert()` instead of get-then-insert | 9.98 ms | 1.15x |
| v4 | `&str` keys, no `format!`, `with_capacity` | 6.25 ms | 1.6x |
| v5 | `FxHashMap` | 5.32 ms | 1.17x |
| | **total** | | **8.6x** |

The lesson is in v1 and v2. Removing `lines.clone()` — cloning a
200,000-element `Vec<String>`, the thing that *looks* most expensive — bought
**1%**. Removing the two intermediate `Vec`s inside the loop bought **4x**.

Why: the clone is 200,000 allocations once. The per-line `Vec<String>` is 3-4
allocations *per line*, 600,000+ total, in the hot path. Cost is
allocations-per-iteration, not the size of any single one — and this is the kind
of thing a flamegraph tells you immediately (a wide `__rust_alloc`) and intuition
gets backwards.

Note the `total=599994` column throughout: every version produces identical
output. An optimization you did not verify for correctness is not an
optimization.

## When to stop

Stop when:

- **You hit the target.** Have one before you start: "p99 under 200 ms". Without
  it there is no stopping condition and you will optimize forever.
- **The profile is flat.** No single item is over ~5%. Further work means
  architectural change, not local edits.
- **The remaining cost is not yours.** If 90% is in `sqlx` waiting on Postgres,
  the next fix is an index or a query change.
- **The win is under 10% and the code got worse.** `unsafe`, a hand-rolled data
  structure, or an unreadable loop for 5% is a bad trade. You will pay for it in
  bugs.
- **It is not on the hot path.** 50x on startup code that runs once is worth
  nothing.

Amdahl's law, concretely: making a function that is 5% of runtime infinitely
fast gets you 5%. Always ask what share the thing you are about to optimize
actually has — which is what a profile is for.

And for a web service, the honest ordering is usually: database indexes, then
N+1 queries, then caching, then connection pool sizing, *then* Rust-level
optimization. The language is rarely the bottleneck.

## Quick Reference

| Need | Command |
|---|---|
| Optimized build | `cargo build --release` |
| Profile-ready build | `cargo build --profile release-with-debug` |
| Micro-benchmark | `cargo bench` with `criterion` |
| Prevent dead-code elimination | `criterion::black_box(x)` |
| Flamegraph | `cargo flamegraph --profile release-with-debug` |
| Profile on macOS | `samply record ./target/release-with-debug/bin` |
| Profile on Linux | `perf record --call-graph dwarf ./bin` then `perf report` |
| Async task state | `RUSTFLAGS="--cfg tokio_unstable"` + `tokio-console` |
| Heap profile | `dhat::Profiler::new_heap()` behind a feature |
| Faster hashing | `rustc_hash::FxHashMap` (not for untrusted keys) |
| CPU parallelism | `rayon`: `.iter()` → `.par_iter()` |
| Buffered IO | `BufReader::new(f)` / `BufWriter::new(f)` |
| Preallocate | `Vec::with_capacity(n)` |
| Build-time breakdown | `cargo build --timings` |

## Common Mistakes

**Profiling a debug build.** Measured above: a fully optimized debug build loses
to a naive release build. It also mis-ranks hot spots.

**Benchmarking without `black_box`.** LLVM deletes the computation and you
measure an empty loop — a suspiciously fast result is this.

**Guessing from the source.** The `lines.clone()` above looked like the problem
and was 1%. Profile.

**Chasing `clone()` calls.** One clone of a big thing is cheap; three small
allocations per iteration are not. Count allocations per iteration.

**Trusting a single criterion run.** A 3.5% "regression" with p < 0.05 appeared
above with no code change at all. Re-run.

**Reaching for `rayon` in an async handler.** It is a CPU thread pool and will
contend with your Tokio workers. Wrap it in `spawn_blocking` or don't.

**Optimizing Rust when the database is the bottleneck.** Check the query plan
first.

## Exercises

1. Write a loop that allocates a `Vec<String>` per iteration. Time it in debug
   and release, and report the ratio for your machine.
2. Add a `release-with-debug` profile and run `samply record` (or
   `cargo flamegraph` on Linux) on the same binary. Name the widest box.
3. Benchmark the same function with and without `black_box`. Explain the
   difference in the reported time.
4. Profile with `dhat` before and after removing one `.collect()` from a hot
   loop. Report `Total` and `At t-gmax` for both and say which one moved.
5. Convert a file-reading loop from `File` to `BufReader` and measure both.
   Then take the function with the largest share of your profile and optimize
   it until the profile is flat — and write down the target you stopped at.

---

[← 18.7 CI with GitHub Actions](07-ci.md) · [Course Index](../README.md) · [18.9 The Essential Crate Directory →](09-crate-directory.md)
