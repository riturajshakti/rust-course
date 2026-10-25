[← 13.4 Test Organization and Fixtures](04-organization.md) · [Course Index](../README.md) · [13.6 Property Testing and Fuzzing →](06-property-testing.md)

# 13.5 — Benchmarking and Criterion

## What You'll Learn

- Why a debug build tells you nothing about performance
- Setting up criterion: `[[bench]]`, `criterion_group!`, `criterion_main!`
- Reading real criterion output: time intervals, outliers, change detection
- `black_box` and why a benchmark without it measures nothing

## Coming From Another Language

| Other | Rust |
|---|---|
| `console.time()`, benchmark.js | criterion, in `benches/` |
| Python `timeit` | criterion (statistical, not a single timing) |
| Go `func BenchmarkX(b *testing.B)` | `c.bench_function("x", |b| b.iter(..))` |

Go's built-in benchmark harness is the closest analogue, and the Rust equivalent
is a crate rather than std. The part with no analogue in most languages:
LLVM optimizes so aggressively that a naive benchmark often measures **nothing
at all**, because the compiler deleted the work. `black_box` is the fix.

## Never benchmark a debug build

`cargo build` and `cargo run` default to the `dev` profile: no inlining, no
optimization, debug assertions on, overflow checks on. Numbers from it are not
merely scaled down, they are unrelated to release behaviour.

```rust
fn main() {
    let v: Vec<u64> = (0..5_000_000).collect();
    let t = std::time::Instant::now();
    let s = w13::sum_iter(std::hint::black_box(&v));
    println!("sum_iter = {s} in {:?}", t.elapsed());
    let t = std::time::Instant::now();
    let s = w13::sum_loop(std::hint::black_box(&v));
    println!("sum_loop = {s} in {:?}", t.elapsed());
}
```

`cargo run`:

```
sum_iter = 12499997500000 in 20.596959ms
sum_loop = 12499997500000 in 14.01575ms
```

`cargo run --release`:

```
sum_iter = 12499997500000 in 3.31975ms
sum_loop = 12499997500000 in 2.32125ms
```

About 6x apart. Worse than the magnitude is that the **ranking can invert**:
iterator chains rely entirely on inlining, so in debug they look slow next to a
hand-written loop and in release they usually tie. Optimizing based on debug
numbers leads you to rewrite code that was already fine.

`cargo bench` uses the `bench` profile (optimized, like `release`) automatically.
That is reason enough to use it instead of hand-rolled `Instant::now()` timing.

## std #[bench] is unstable

Rust has a built-in benchmark harness, and you cannot use it:

```rust
#![feature(test)]
extern crate test;

#[bench]
fn bench_add(b: &mut test::Bencher) {
    b.iter(|| 1 + 1);
}
```

```
error[E0554]: `#![feature]` may not be used on the stable release channel
 --> benches/stdbench.rs:1:1
  |
1 | #![feature(test)]
  | ^^^^^^^^^^^^^^^^^
```

It has been unstable for years with no stabilization planned, which is why the
`0 measured` column in every `cargo test` summary is always zero — that counter
exists for `#[bench]`. The ecosystem answer is criterion.

## Setting up criterion

Add it as a dev-dependency and declare a bench target with the default harness
disabled, so criterion can supply its own `main`:

```
cargo add --dev criterion
```

```toml
[dev-dependencies]
criterion = "0.8.2"

[[bench]]
name = "fib"
harness = false
```

`name` must match `benches/fib.rs`. Forgetting `harness = false` is the classic
first mistake: the built-in harness takes over, finds no `#[test]` functions,
and your benchmarks never run.

Unlike `tests/`, bench files are **not** auto-discovered in a way that works —
each one needs its own `[[bench]]` section.

## A first benchmark

```rust
pub fn fib_recursive(n: u64) -> u64 {
    if n < 2 { n } else { fib_recursive(n - 1) + fib_recursive(n - 2) }
}

pub fn fib_iterative(n: u64) -> u64 {
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 0..n {
        let t = a + b;
        a = b;
        b = t;
    }
    a
}
```

`benches/fib.rs`:

```rust
use criterion::{criterion_group, criterion_main, Criterion};
use std::hint::black_box;
use w13::{fib_iterative, fib_recursive};

fn bench_fib(c: &mut Criterion) {
    c.bench_function("fib_recursive 20", |b| {
        b.iter(|| fib_recursive(black_box(20)))
    });
    c.bench_function("fib_iterative 20", |b| {
        b.iter(|| fib_iterative(black_box(20)))
    });
}

criterion_group!(benches, bench_fib);
criterion_main!(benches);
```

Three layers: `criterion_main!` generates `fn main`, `criterion_group!` collects
functions into a named group, and each `bench_function` registers one benchmark.
The closure passed to `bench_function` receives a `Bencher`; `b.iter(..)` is the
part criterion times, and it runs the inner closure as many times as it needs.

Note the import is `std::hint::black_box` — criterion re-exported its own for
years, but the std one is stable and is what you should use now.

`cargo bench`:

```
     Running benches/fib.rs (target/release/deps/fib-65699f10df1b6d9d)
Gnuplot not found, using plotters backend
Benchmarking fib_recursive 20
Benchmarking fib_recursive 20: Warming up for 3.0000 s
Benchmarking fib_recursive 20: Collecting 100 samples in estimated 5.0333 s (404k iterations)
Benchmarking fib_recursive 20: Analyzing
fib_recursive 20        time:   [12.412 µs 12.465 µs 12.515 µs]
Found 2 outliers among 100 measurements (2.00%)
  2 (2.00%) low mild

Benchmarking fib_iterative 20
Benchmarking fib_iterative 20: Warming up for 3.0000 s
Benchmarking fib_iterative 20: Collecting 100 samples in estimated 5.0000 s (956M iterations)
Benchmarking fib_iterative 20: Analyzing
fib_iterative 20        time:   [5.1846 ns 5.2228 ns 5.2666 ns]
```

Reading it:

- **Warming up for 3 s** — runs the code untimed to let CPU frequency scaling,
  branch predictors and caches settle.
- **Collecting 100 samples in estimated 5 s (404k iterations)** — 100 samples,
  each averaging many iterations. Criterion picks the iteration count so a
  sample is long enough to measure; here 404k total for the µs-scale function
  and 956M for the ns-scale one.
- **`time: [12.412 µs 12.465 µs 12.515 µs]`** — the three numbers are the
  **lower bound, best estimate, and upper bound** of a 95% confidence interval,
  *not* min/mean/max. A tight interval means a reliable measurement.
- **Outliers** — samples far from the rest, classified `low`/`high` and
  `mild`/`severe`. A few percent is normal background noise. Many severe high
  outliers mean something else was running on the machine.

The comparison here is real: 12.5 µs versus 5.2 ns, about 2400x, because
recursive fib is exponential. Criterion also wrote an HTML report to
`target/criterion/report/index.html` with distribution plots.

## Change detection

Criterion saves each run's statistics and compares the next run against them.
Running the same benchmark again:

```
fib_recursive 20        time:   [12.760 µs 12.832 µs 12.901 µs]
                        change: [+2.1417% +2.6070% +3.0661%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 2 outliers among 100 measurements (2.00%)
  2 (2.00%) high mild

fib_iterative 20        time:   [5.1772 ns 5.1964 ns 5.2186 ns]
                        change: [−3.7707% −2.8737% −1.9384%] (p = 0.00 < 0.05)
                        Performance has improved.
```

This is the feature that makes criterion worth using: benchmark, change code,
benchmark again, and get a statistically justified verdict instead of eyeballing
two numbers.

But read that example carefully — **nothing changed between the two runs**. The
source was identical. Criterion still reported a 2.6% regression and a 2.9%
improvement, both with `p = 0.00`. The `p`-value says the difference in
measurements is real; it does not say your code caused it.

Six consecutive runs of one unchanged benchmark on this machine:

```
change: [−3.0368% +0.2070% +3.3193%] (p = 0.90 > 0.05)  No change in performance detected.
change: [−4.6659% −2.9122% −1.3757%] (p = 0.00 < 0.05)  Performance has improved.
change: [−1.0207% +0.2817% +2.2236%] (p = 0.78 > 0.05)  No change in performance detected.
change: [+11.407% +22.043% +36.710%] (p = 0.00 < 0.05)  Performance has regressed.
change: [−23.723% −15.201% −7.4385%] (p = 0.00 < 0.05)  Performance has improved.
change: [+40.926% +66.386% +95.053%] (p = 0.00 < 0.05)  Performance has regressed.
```

From -15% to +66% with no code change. The high-variance runs came with 11-13%
outliers while the quiet ones had 1-3% — so the **outlier count is your signal
for whether to trust a verdict at all**. Re-run a benchmark several times on an
idle machine before believing any change, and discount a run whose outliers are
in double digits.

`p > 0.05` is the honest verdict when a difference is within noise:

```
                        change: [−3.0368% +0.2070% +3.3193%] (p = 0.90 > 0.05)
                        No change in performance detected.
```

The baseline lives in `target/criterion/`. `cargo bench -- --save-baseline main`
names one, and `--baseline main` compares against it later — the right way to
benchmark a branch against `main` rather than against whatever ran last.

## black_box

This is the single most important thing in this lesson. The compiler can see
that your benchmark's result is unused and its input is a constant, so it is
entitled to delete the whole computation.

```rust
fn bench(c: &mut Criterion) {
    c.bench_function("no black_box", |b| b.iter(|| fib_iterative(20)));
    c.bench_function("with black_box", |b| b.iter(|| fib_iterative(black_box(20))));
    c.bench_function("result discarded", |b| {
        b.iter(|| {
            fib_iterative(black_box(20));
        })
    });
}
```

Output:

```
no black_box            time:   [223.66 ps 223.82 ps 224.00 ps]
with black_box          time:   [5.0725 ns 5.1025 ns 5.1342 ns]
result discarded        time:   [223.85 ps 224.22 ps 224.63 ps]
```

223 **picoseconds** versus 5.1 nanoseconds — a 23x phantom speedup. 223 ps is
below the cost of a single cache miss; it is the measurement overhead of an
empty loop. In the first case LLVM constant-folded `fib_iterative(20)` to the
literal `6765` at compile time. In the third case `black_box` protected the
input, but the trailing semicolon discards the result, so the computation was
dead code and got deleted anyway.

`std::hint::black_box(x)` is an opaque identity function: it returns `x`, but
the optimizer must assume it could do anything with it, so it cannot propagate
the value as a constant or prove the result unused. Two rules:

- Wrap every **input** that is a literal or a loop-invariant: `black_box(20)`.
- Make sure the **output** is returned from the `iter` closure, not discarded.
  `b.iter(|| f(black_box(x)))` — no semicolon.

A benchmark reporting picoseconds for real work is always this bug. Sanity-check
your numbers against physics: a cache miss is roughly 100 ns, an L1 hit about
1 ns, and a single instruction well under 1 ns.

## Benchmarking with inputs

`bench_with_input` plus a `BenchmarkGroup` compares implementations across
several input sizes. `BenchmarkId::new(name, size)` produces the
`group/name/size` labels.

```rust
use criterion::{BenchmarkId, Criterion};

fn bench_sum(c: &mut Criterion) {
    let mut group = c.benchmark_group("sum");
    for size in [100u64, 10_000] {
        let data: Vec<u64> = (0..size).collect();
        group.bench_with_input(BenchmarkId::new("loop", size), &data, |b, d| {
            b.iter(|| sum_loop(black_box(d)))
        });
        group.bench_with_input(BenchmarkId::new("iter", size), &data, |b, d| {
            b.iter(|| sum_iter(black_box(d)))
        });
    }
    group.finish();
}
```

Output:

```
sum/loop/100            time:   [4.9047 ns 4.9169 ns 4.9301 ns]
sum/iter/100            time:   [5.1363 ns 5.1575 ns 5.1812 ns]
sum/loop/10000          time:   [591.84 ns 595.42 ns 600.11 ns]
sum/iter/10000          time:   [589.72 ns 592.84 ns 596.55 ns]
```

Two findings. The hand-written loop and `v.iter().sum()` are **the same speed**
at both sizes — iterator chains compile to the same machine code, so the
idiomatic version costs nothing. And look at `sum/loop/100`: 4.9 ns to add 100
`u64`s is about 0.05 ns per element, far faster than one addition per cycle.
LLVM vectorized it. Scaling is also sub-linear (100x the data for 120x the
time), which is a hint to check whether you are measuring what you think.

Building `data` **outside** `b.iter` is essential — otherwise you measure the
allocation too. `group.finish()` is required to emit the comparison.

Two more group settings worth knowing: `group.throughput(Throughput::Bytes(n))`
makes criterion report GiB/s alongside time, and `group.sample_size(20)` reduces
samples for a benchmark too slow for the default 100.

## Common benchmarking mistakes

**Constant folding.** Covered above: the single most common error, and it
announces itself with picosecond timings.

**Including setup in the measurement.** Allocating a `Vec` inside `b.iter`
measures the allocator. Build inputs outside. When each iteration genuinely must
start from a fresh value, use `b.iter_batched(setup, routine, BatchSize::SmallInput)`,
which runs `setup` untimed.

**Cold cache versus warm cache.** Criterion's loop runs your function
back-to-back, so after the first iteration the data is in L1. That is realistic
for a hot inner loop and wildly unrealistic for code that runs once per request
on cold data. Benchmarks reliably overstate cache-friendly code.

**Allocation noise.** A function that allocates is partly measuring your
allocator's current free-list state, which varies run to run. Prefer comparing
two implementations in the same run over comparing one against yesterday.

**Benchmarking on a busy machine.** Close everything, disable turbo if you can,
and never trust a benchmark from CI — shared runners have the worst variance of
any environment. Watch the outlier count: many `high severe` outliers means the
run is contaminated.

**Micro-optimizing the wrong thing.** A 2x win on 1% of runtime is a 0.5%
win. Profile first to find where time goes (`cargo flamegraph`, `perf`,
Instruments), then benchmark that function.

**Trusting one run.** The identical-code example above swung from -15% to +66%
across six runs. Re-run before acting, and check the outlier percentage.

## hyperfine for whole-binary timing

Criterion measures a function. For a whole process — CLI startup, a build step,
a script — use `hyperfine`, which does the same statistical treatment for
commands:

```
cargo install hyperfine
cargo build --release
hyperfine --warmup 3 -N './target/release/w13' './target/debug/w13'
```

Output:

```
Benchmark 1: ./target/release/w13
  Time (mean ± σ):       5.0 ms ±   0.2 ms    [User: 2.3 ms, System: 1.9 ms]
  Range (min … max):     4.4 ms …   6.4 ms    468 runs

Benchmark 2: ./target/debug/w13
  Time (mean ± σ):      65.8 ms ±   1.0 ms    [User: 62.0 ms, System: 2.8 ms]
  Range (min … max):    64.1 ms …  69.0 ms    46 runs

Summary
  ./target/release/w13 ran
   13.28 ± 0.66 times faster than ./target/debug/w13
```

13x, and this measures what the user actually waits for: process startup,
dynamic linking and I/O included. `--warmup 3` discards initial runs, `-N`
skips the intermediate shell, and `-p 'cmd'` runs an untimed prepare step before
each timing — necessary when the command would otherwise hit a cache.

Use criterion for a function, hyperfine for a command. Neither replaces a
profiler for finding *where* the time goes.

## Running benchmarks

| Command | Effect |
|---|---|
| `cargo bench` | All bench targets, `bench` profile |
| `cargo bench --bench fib` | One bench file |
| `cargo bench -- fib_iter` | Substring filter on benchmark name |
| `cargo bench -- --save-baseline main` | Save results under a name |
| `cargo bench -- --baseline main` | Compare against that baseline |
| `cargo bench --no-run` | Compile only |
| `cargo test --benches` | Compile and run benches as tests (one iteration) |

`cargo test --benches` is the CI trick: it runs each benchmark once to prove it
still compiles and does not panic, without spending five seconds per benchmark.
Full benchmarks in CI are usually a waste because runner variance exceeds the
effects you are looking for.

## Quick Reference

| Item | Purpose |
|---|---|
| `[[bench]]` + `harness = false` | Declare a criterion bench target |
| `criterion_main!(benches)` | Generates `fn main` |
| `criterion_group!(benches, f1, f2)` | Collects bench functions |
| `c.bench_function("name", \|b\| b.iter(..))` | One benchmark |
| `b.iter(\|\| f(black_box(x)))` | The timed loop; no trailing semicolon |
| `b.iter_batched(setup, f, BatchSize::SmallInput)` | Per-iteration setup, untimed |
| `std::hint::black_box(x)` | Hide a value from the optimizer |
| `group.bench_with_input(BenchmarkId::new(n, s), &d, ..)` | Parameterized benchmark |
| `group.throughput(Throughput::Bytes(n))` | Report GiB/s |
| `group.sample_size(20)` | Fewer samples for slow benchmarks |
| `group.finish()` | Required to emit the comparison |
| `time: [lo est hi]` | 95% confidence interval, not min/mean/max |
| `change: [..] (p = ..)` | Comparison against the saved baseline |
| `target/criterion/report/index.html` | Generated HTML report |
| `hyperfine 'cmd'` | Whole-process timing |

## Common Mistakes

**Benchmarking a debug build.** `cargo run` is unoptimized; numbers are
unrelated to release, and rankings can invert. Use `cargo bench`.

**Forgetting `harness = false`.** The default harness takes over and your
benchmarks silently never run.

**Omitting `black_box` on inputs.** LLVM constant-folds the call away and you
measure an empty loop — the giveaway is a picosecond timing.

**Discarding the result.** A trailing semicolon inside `b.iter` makes the
computation dead code, deleted even with `black_box` on the input.

**Allocating inside `b.iter`.** You measure the allocator. Build inputs outside,
or use `iter_batched`.

**Reading `[lo est hi]` as min/mean/max.** They are confidence-interval bounds.

**Believing one `change:` line.** Identical code reported anything from -15% to
+66% here. Re-run, and distrust any run with double-digit outliers.

## Exercises

1. Set up criterion in a scratch project and benchmark two implementations of
   string reversal (`chars().rev().collect()` versus a byte loop). Compare.
2. Write a benchmark without `black_box` and one with it on the same function.
   Confirm the picosecond timing, then explain the gap.
3. Use `bench_with_input` to measure a linear search versus `binary_search` over
   sorted `Vec`s of 10, 1_000 and 100_000 elements. At which size does binary
   search win, and why not at 10?
4. Save a baseline, make the function 2x slower on purpose, re-run and read the
   `change:` line. Then re-run with no changes six times and record the spread
   and outlier counts.
5. Benchmark your binary's startup with hyperfine, add a large static table to
   it, and measure again. Use `-p` to clear any cache between runs.

---

[← 13.4 Test Organization and Fixtures](04-organization.md) · [Course Index](../README.md) · [13.6 Property Testing and Fuzzing →](06-property-testing.md)
