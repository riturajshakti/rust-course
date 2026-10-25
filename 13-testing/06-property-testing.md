[← 13.5 Benchmarking and Criterion](05-benchmarks.md) · [Course Index](../README.md) · [14.1 Threads and join →](../14-concurrency/01-threads.md)

# 13.6 — Property Testing and Fuzzing

## What You'll Learn

- Writing properties with `proptest!` instead of hand-picked examples
- Strategies, ranges, `prop_assume!`, and `ProptestConfig`
- Shrinking: how a random failure becomes a minimal counterexample
- Fuzzing with `cargo-fuzz`/libFuzzer, and when each tool is the right one

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `fast-check` | `proptest` |
| Python `hypothesis` | `proptest` (same shrinking idea) |
| Go `testing/quick`, Go 1.18 fuzzing | `quickcheck`, `cargo-fuzz` |

Example-based tests check the cases *you thought of*. Property tests state a rule
that must hold for **all** inputs and let the library hunt for a counterexample,
then shrink it to the smallest input that still fails. That shrinking is what
makes the approach practical.

## Setup

```
cargo add --dev proptest
```

```toml
[dev-dependencies]
proptest = "1.12.0"
```

## A first property: round-trips

The easiest property to find in any codebase is a round-trip: encode then decode
must give back the original.

```rust
pub fn encode(input: &[u8]) -> String {
    input.iter().map(|b| format!("{b:02x}")).collect()
}

pub fn decode(s: &str) -> Result<Vec<u8>, String> {
    if s.len() % 2 != 0 {
        return Err("odd length".to_string());
    }
    (0..s.len())
        .step_by(2)
        .map(|i| u8::from_str_radix(&s[i..i + 2], 16).map_err(|e| e.to_string()))
        .collect()
}
```

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn encode_decode_roundtrip(input: Vec<u8>) {
            let encoded = encode(&input);
            prop_assert_eq!(decode(&encoded).unwrap(), input);
        }

        #[test]
        fn encode_length_is_double(input: Vec<u8>) {
            prop_assert_eq!(encode(&input).len(), input.len() * 2);
        }
    }
}
```

Output:

```
test tests::encode_decode_roundtrip ... ok
test tests::encode_length_is_double ... ok
```

Each ran **256 times** by default with different random inputs, including the
empty vector. Three things about `proptest!`:

- It is a macro wrapping one or more `#[test]` functions. The attribute goes
  inside, and the functions take **parameters** — unlike a normal `#[test]`.
- A parameter's type suffices when a default strategy exists, which it does for
  primitives, `String`, `Vec<T>`, `Option<T>` and tuples.
- Use `prop_assert!`, `prop_assert_eq!`, `prop_assert_ne!`, not the std macros:
  they return `Err` rather than panicking, which lets proptest shrink cleanly.

## Invariants

A property need not involve a round-trip. Any rule that must always hold works.

```rust
pub fn my_sort(v: &mut Vec<i32>) {
    v.sort();
}

proptest! {
    #[test]
    fn sort_is_idempotent_and_ordered(mut v: Vec<i32>) {
        my_sort(&mut v);
        let once = v.clone();
        my_sort(&mut v);
        prop_assert_eq!(&v, &once);
        prop_assert!(v.windows(2).all(|w| w[0] <= w[1]));
    }
}
```

Output:

```
test tests::sort_is_idempotent_and_ordered ... ok
```

Two properties at once: sorting twice equals sorting once, and the result is
ordered. Note "ordered" alone is not a sufficient sort test — `v.clear()` passes
it too. A complete spec adds "is a permutation of the input".

Another pattern — the result's relationship to the input:

```rust
pub fn max_of(v: &[i32]) -> Option<i32> {
    v.iter().copied().max()
}

proptest! {
    #[test]
    fn max_is_in_the_vec(v: Vec<i32>) {
        match max_of(&v) {
            None => prop_assert!(v.is_empty()),
            Some(m) => {
                prop_assert!(v.contains(&m));
                prop_assert!(v.iter().all(|&x| x <= m));
            }
        }
    }
}
```

Output:

```
test tests::max_is_in_the_vec ... ok
```

That is the full specification of `max`: `None` exactly when empty, otherwise an
element of the input that no element exceeds. Property tests push you to *state*
what a function means, which is often where the bug turns up.

## Strategies and ranges

`x in <strategy>` replaces the type-driven default. A range is the most common
strategy, and it lets you exclude inputs a function does not accept:

```rust
pub fn gcd(a: u64, b: u64) -> u64 {
    if b == 0 { a } else { gcd(b, a % b) }
}

proptest! {
    #[test]
    fn gcd_is_commutative(a in 1u64..10_000, b in 1u64..10_000) {
        prop_assert_eq!(gcd(a, b), gcd(b, a));
    }

    #[test]
    fn gcd_divides_both(a in 1u64..10_000, b in 1u64..10_000) {
        let g = gcd(a, b);
        prop_assert_eq!(a % g, 0);
        prop_assert_eq!(b % g, 0);
    }
}
```

Output:

```
test tests::gcd_is_commutative ... ok
test tests::gcd_divides_both ... ok
```

Commutativity plus the defining property of a divisor pin down `gcd` far better
than `assert_eq!(gcd(12, 18), 6)`. Typed and strategy parameters mix freely:

```rust
pub fn clamp_to(v: i32, lo: i32, hi: i32) -> i32 {
    v.max(lo).min(hi)
}

proptest! {
    #[test]
    fn clamp_stays_in_range(v: i32, lo in -100i32..0, hi in 1i32..100) {
        let r = clamp_to(v, lo, hi);
        prop_assert!(r >= lo && r <= hi);
    }
}
```

Output:

```
test tests::clamp_stays_in_range ... ok
```

The strategies worth knowing:

| Strategy | Generates |
|---|---|
| `0u32..100` | A value in the range |
| `0u8..=255` | Inclusive range |
| `any::<i32>()` | The type's default strategy, written explicitly |
| `"[a-z]{1,8}"` | A `String` matching the regex |
| `prop::collection::vec(s, 1..5)` | A `Vec` of 1-4 values from strategy `s` |
| `prop::option::of(s)` | `Option<T>`, sometimes `None` |
| `prop::sample::select(vec![..])` | One of a fixed list |
| `(s1, s2)` | A tuple, each element from its strategy |
| `s.prop_map(f)` | Transform generated values |
| `s.prop_filter("why", pred)` | Discard values failing `pred` |

A regex string strategy is often the clearest way to generate structured text:

```rust
proptest! {
    #[test]
    fn slug_of_ascii_words_has_no_double_dash(
        words in proptest::collection::vec("[a-z]{1,8}", 1..5)
    ) {
        let joined = words.join(" ");
        let out = slugify(&joined);
        prop_assert!(!out.contains("--"), "got {}", out);
    }
}
```

Output:

```
test tests::slug_of_ascii_words_has_no_double_dash ... ok
```

Generating valid-shaped input rather than arbitrary `String`s is how you reach
deeper logic. For your own types, implement `Arbitrary` or `prop_map` over a
tuple of field strategies.

## prop_assume!

When some inputs are simply out of contract, discard them instead of
constraining the strategy:

```rust
proptest! {
    #[test]
    fn division_with_assume(a: i32, b: i32) {
        prop_assume!(b != 0);
        prop_assume!(!(a == i32::MIN && b == -1));
        let q = a / b;
        prop_assert_eq!(q * b + a % b, a);
    }
}
```

Output:

```
test tests::division_with_assume ... ok
```

A failed assumption rejects that case and generates a new one; it does not fail
the test. The second assume excludes the single pair where `i32` division
overflows — an edge case a property test finds immediately if you forget it.

Rejections are capped: reject too many and proptest gives up with "Too many
global rejects", counted by the `local rejects` / `global rejects` lines in
failure output. Rejecting most inputs means you should narrow the strategy.

## Shrinking

The payoff. A random failure is usually a huge opaque input; proptest
repeatedly simplifies it while the test still fails.

```rust
proptest! {
    #[test]
    fn no_vec_sums_past_100(v: Vec<u8>) {
        let total: u32 = v.iter().map(|&x| x as u32).sum();
        prop_assert!(total <= 100, "sum was {}", total);
    }
}
```

Output (intentional failure):

```
---- tests::no_vec_sums_past_100 stdout ----
proptest: Saving this and future failures in .../w13/proptest-regressions/lib.txt
proptest: If this test was run on a CI system, you may wish to add the following line to your copy of the file.
cc 04ab02eb92fdea1110040c5e56c1e799e3fb2ca89d0627b3f3066476efe646f0

thread 'tests::no_vec_sums_past_100' (14079964) panicked at src/lib.rs:54:5:
Test failed: sum was 101 at src/lib.rs:115.
minimal failing input: v = [
    101,
]
	successes: 0
	local rejects: 0
	global rejects: 0
```

The original failure was a random `Vec<u8>` of arbitrary length; the reported
counterexample is `[101]` — one element, the smallest value breaking a `<= 100`
bound. Shrinking dropped every unnecessary element, then reduced the survivor to
the boundary. With a scalar it goes straight there:

```rust
proptest! {
    #[test]
    fn no_u8_above_200(x in 0u8..=255) {
        prop_assert!(x <= 200);
    }
}
```

Output (intentional failure):

```
thread 'tests::no_u8_above_200' (14080071) panicked at src/lib.rs:54:5:
Test failed: assertion failed: x <= 200 at src/lib.rs:128.
minimal failing input: x = 201
```

`x = 201`, not some random number above 200 — the difference between a usable
bug report and a puzzle. Reading the block:

- **`minimal failing input:`** the shrunk counterexample, `Debug`-formatted.
- **`successes:`** cases that passed *during shrinking*. `0` means the very
  first generated case failed.
- **`local rejects` / `global rejects`** `prop_assume!` rejections.
- **`cc <hash>`** the seed, for reproducing exactly.

## The regressions file

Every failure is appended to `proptest-regressions/<module>.txt`, and those
seeds are **re-run first** on every subsequent run:

```
# Seeds for failure cases proptest has generated in the past. It is
# automatically read and these particular cases re-run before any
# novel cases are generated.
#
# It is recommended to check this file in to source control so that
# everyone who runs the test benefits from these saved cases.
cc 68c3d00f81877fe56da8906b675fdff4aaa762e99b7c816dfb6bd73e0e9bd55e # shrinks to v = [101]
```

Commit this file. It turns every bug the property test ever found into a
permanent regression test, so a later break fails immediately instead of waiting
for the generator to rediscover the input.

## Configuration

`#![proptest_config(..)]` inside a `proptest!` block tunes the run:

```rust
proptest! {
    #![proptest_config(ProptestConfig {
        cases: 20,
        max_shrink_iters: 500,
        .. ProptestConfig::default()
    })]

    #[test]
    fn runs_only_twenty_cases(x: u16) {
        prop_assert!(x <= u16::MAX);
    }
}
```

Output:

```
test tests::runs_only_twenty_cases ... ok
```

| Field | Default | Use |
|---|---|---|
| `cases` | 256 | Lower for slow properties, raise for cheap ones |
| `max_shrink_iters` | 1024 | Raise when shrinking gives up early |
| `max_global_rejects` | 1024 | Raise if `prop_assume!` rejects a lot |
| `failure_persistence` | file | `None` disables the regressions file |

`PROPTEST_CASES=10000 cargo test` overrides `cases` from the environment —
useful for a long nightly run against the same tests CI runs quickly.

## A real bug found by a property

Property tests find bugs you would not write a test for. This looks fine:

```rust
pub fn truncate_chars(s: &str, n: usize) -> String {
    s.chars().take(n).collect()
}
```

Two properties, one correct and one subtly wrong:

```rust
proptest! {
    #[test]
    fn truncate_never_longer(s: String, n in 0usize..20) {
        prop_assert!(truncate_chars(&s, n).chars().count() <= n);
    }

    #[test]
    fn truncate_byte_len_bounded(s: String, n in 0usize..20) {
        prop_assert!(truncate_chars(&s, n).len() <= n);
    }
}
```

Output (the second fails intentionally):

```
test tests::truncate_never_longer ... ok

---- tests::truncate_byte_len_bounded stdout ----
thread 'tests::truncate_byte_len_bounded' (14079969) panicked at src/lib.rs:141:5:
Test failed: assertion failed: truncate_chars(&s, n).len() <= n at src/lib.rs:155.
minimal failing input: s = "🬀ወ\u{c4a}", n = 1
```

`len()` is **bytes** while `n` counts **chars**, so one emoji blows a byte budget
of 1. Nobody writing example tests reaches for `"🬀ወ"`, but conflating bytes and
chars is exactly the bug class that panics in production on `&s[..n]`. Note the
shrink precision: `n = 1`, the smallest `n` that can fail.

## When property tests beat example tests

| Use a property test | Use an example test |
|---|---|
| A round-trip (parse/serialize, encode/decode, compress) | A specific known-good input/output pair |
| An algebraic law (commutativity, associativity, idempotence) | A regression for one reported bug |
| An invariant on every output (in range, sorted, non-empty) | Exact error messages and formatting |
| Comparing against a slow reference implementation | Behaviour of one documented edge case |
| Anything touching string indices, arithmetic bounds, or parsing | Integration flows with fixed fixtures |

Complements, not alternatives: a property test states the contract, an example
test pins the behaviour a user reported. When a property test finds a bug, the
committed `cc` seed becomes the example test for free.

They do cost more — 256 runs is slower, a subtly wrong property is flaky, and a
badly chosen strategy tests nothing. Start with round-trips and output
invariants, which are nearly always correct to state.

## quickcheck

The older alternative, modelled directly on Haskell's QuickCheck. The property
is a function returning `bool` or `TestResult`:

```
cargo add --dev quickcheck
```

```rust
#[cfg(test)]
mod tests {
    use quickcheck::quickcheck;

    quickcheck! {
        fn roundtrip(input: Vec<u8>) -> bool {
            crate::decode(&crate::encode(&input)).unwrap() == input
        }
    }
}
```

Terser for simple cases, but proptest is the better default: composable strategy
values rather than a trait impl per type, shrinking integrated with those
strategies, and persisted failures. Pick quickcheck only if you prefer the
trait-based style.

## Fuzzing with cargo-fuzz

Property testing generates *structured* input against a *stated* property.
Fuzzing throws **arbitrary bytes** at a target in a coverage-guided loop and asks
only "did it crash" — finding panics, overflows, out-of-bounds indexing and hangs
in code that takes untrusted input.

`cargo-fuzz` drives libFuzzer, which requires **nightly** (it needs sanitizer
and coverage instrumentation that is not on stable). The workflow:

```
cargo install cargo-fuzz
rustup toolchain install nightly
cargo fuzz init
```

`cargo fuzz init` creates a `fuzz/` subcrate — its own `Cargo.toml` depending on
`libfuzzer-sys` and your crate, plus `fuzz/fuzz_targets/fuzz_target_1.rs`.
Add a target with `cargo fuzz add decode_target`, which has this shape:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        let _ = w13::decode(s);
    }
});
```

The harness itself must **not panic on invalid input** — a crash is what gets
reported, so the target only calls your code and discards the result. For typed
input instead of raw bytes, add `arbitrary` (currently 1.5) with its `derive`
feature:

```rust
#[derive(arbitrary::Arbitrary, Debug)]
struct Input {
    n: u8,
    text: String,
}

fuzz_target!(|input: Input| {
    let _ = w13::truncate_chars(&input.text, input.n as usize);
});
```

Then:

```
cargo +nightly fuzz run decode_target
cargo +nightly fuzz run decode_target -- -max_total_time=60
```

It runs until it crashes or you stop it, printing libFuzzer progress (executions
per second, coverage counters, corpus size). On a crash it writes the input to
`fuzz/artifacts/<target>/crash-<hash>` and prints the panic. Then:

```
cargo +nightly fuzz run decode_target fuzz/artifacts/decode_target/crash-<hash>
cargo +nightly fuzz tmin decode_target fuzz/artifacts/decode_target/crash-<hash>
```

The first replays one input; `tmin` minimizes it — fuzzing's equivalent of
shrinking. Inputs accumulate in `fuzz/corpus/<target>/`; commit that corpus so
later runs start from known-interesting inputs.

Fuzzing is a long-running job measured in CPU-hours, not a `cargo test` step.
Seed the corpus with real valid inputs to get past input validation, and fuzz
the right thing — a target rejecting 99.9% of inputs at a length check explores
nothing.

| | proptest | cargo-fuzz |
|---|---|---|
| Input | Typed, from strategies | Arbitrary bytes, coverage-guided |
| Checks | Your stated property | Crashes, panics, overflow, UB |
| Toolchain | Stable | Nightly |
| Runtime | Seconds, in `cargo test` | Hours, run separately |
| Minimization | Shrinking | `cargo fuzz tmin` |
| Best for | Logic, laws, round-trips | Parsers, decoders, `unsafe`, untrusted input |

AFL++ via the `afl` crate is the other mature option, with a different
exploration strategy.

## Quick Reference

| Item | Purpose |
|---|---|
| `proptest! { #[test] fn p(x: T) {..} }` | Property test block |
| `prop_assert!` / `prop_assert_eq!` | Assertions that shrink cleanly |
| `x in 0u32..100` | Explicit strategy |
| `any::<T>()` | Default strategy, written out |
| `"[a-z]{1,8}"` | Regex string strategy |
| `prop::collection::vec(s, 1..5)` | Vec strategy |
| `s.prop_map(f)` / `s.prop_filter(..)` | Transform / filter a strategy |
| `prop_assume!(cond)` | Discard out-of-contract inputs |
| `#![proptest_config(ProptestConfig { cases: 20, .. })]` | Tune the run |
| `PROPTEST_CASES=10000 cargo test` | Override case count |
| `proptest-regressions/*.txt` | Saved failing seeds — commit it |
| `minimal failing input:` | The shrunk counterexample |
| `cargo fuzz init` / `add <t>` / `run <t>` | libFuzzer workflow (nightly) |
| `fuzz_target!(\|data: &[u8]\| {..})` | Fuzz harness |
| `#[derive(arbitrary::Arbitrary)]` | Typed fuzz input |
| `cargo fuzz tmin <t> <artifact>` | Minimize a crashing input |

## Common Mistakes

**Using `assert_eq!` instead of `prop_assert_eq!`.** It panics rather than
returning an error, degrading shrink quality.

**A property that is true but vacuous.** `prop_assert!(x <= u16::MAX)` for a
`u16` tests nothing. State a rule that could actually fail.

**Testing "is sorted" as a complete sort test.** `v.clear()` passes it. Add a
permutation check.

**Not committing `proptest-regressions/`.** You throw away every bug the suite
has ever found.

**Rejecting most inputs with `prop_assume!`.** You hit the reject cap and test
almost nothing. Narrow the strategy instead.

**Expecting `cargo-fuzz` to work on stable.** It needs nightly. Use
`cargo +nightly fuzz run`.

**Fuzzing in CI as a test step.** It is a long-running job; run it separately
and commit the corpus.

## Exercises

1. Write `pub fn rle_encode(s: &str) -> Vec<(char, usize)>` and `rle_decode`.
   Add a round-trip property and run it.
2. Add a property that `rle_encode` never emits a zero count and never two
   consecutive entries with the same char. Break the function and read the
   shrunk counterexample.
3. Write a property comparing your own `fn my_max(v: &[i32]) -> Option<i32>`
   against `v.iter().max()` for all inputs, including empty.
4. Use `prop::collection::vec` and a regex strategy to generate valid CSV-ish
   lines, then property-test a parser on them. Add `prop_assume!` to skip lines
   containing your delimiter and observe the reject counters.
5. Write a property on a function taking a byte count and a `&str` that fails
   only for multi-byte characters. Confirm the shrunk input is a single
   non-ASCII char, then commit the regressions file and re-run.

---

[← 13.5 Benchmarking and Criterion](05-benchmarks.md) · [Course Index](../README.md) · [14.1 Threads and join →](../14-concurrency/01-threads.md)
