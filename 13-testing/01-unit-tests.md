[← 12.7 Publishing to crates.io](../12-modules-crates/07-publishing.md) · [Course Index](../README.md) · [13.2 Integration Tests →](02-integration-tests.md)

# 13.1 — Unit Tests and assert Macros

## What You'll Learn

- `#[test]`, `#[cfg(test)] mod tests`, and why unit tests live beside the code
- `assert!` / `assert_eq!` / `assert_ne!` and exactly what a failure prints
- `#[should_panic]`, tests returning `Result`, `#[ignore]`
- `cargo test` filtering, `--nocapture`, `--test-threads`, `debug_assert!`

## Coming From Another Language

| Other | Rust |
|---|---|
| Jest `test()` in `foo.test.ts` | `#[test] fn` in the *same* file as the code |
| pytest `test_*.py`, `assert x == y` | `#[test]`, `assert_eq!(x, y)` |
| Go `TestFoo(t *testing.T)` in `foo_test.go` | `#[test] fn foo()` inside `mod tests` |
| `npm test` / `pytest` / `go test` | `cargo test` |

The test runner is built in — no framework to install, no config file. The
surprising part for most newcomers: unit tests go **inside the file they test**,
in a child module, which is how they reach private functions.

## The anatomy

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds() {
        assert_eq!(add(2, 3), 5);
    }
}
```

Output of `cargo test`:

```
running 1 test
test tests::adds ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`#[cfg(test)]` compiles the module *only* under `cargo test`, so it is absent
from your release binary. `mod tests` is a convention — it supplies the
`tests::` prefix. `use super::*;` is needed because the test module is a child
module. A `#[test]` fn takes no arguments, returns `()` or `Result`, and fails
only by panicking.

## Testing private functions

The real reason unit tests live in the same file: a child module sees its
parent's private items.

```rust
fn normalize(s: &str) -> String {
    s.trim().to_lowercase()
}

pub fn slug(s: &str) -> String {
    normalize(s).replace(' ', "-")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn private_normalize_is_reachable() {
        assert_eq!(normalize("  Hello World  "), "hello world");
    }

    #[test]
    fn slug_works() {
        assert_eq!(slug("  Hello World  "), "hello-world");
    }
}
```

Output:

```
test tests::private_normalize_is_reachable ... ok
test tests::slug_works ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

No `pub` needed on `normalize`. Integration tests (13.2) cannot do this — they
are separate crates and see only your public API.

## assert!

Takes any `bool`.

```rust
#[test]
fn evenness() {
    assert!(is_even(4));
    assert!(!is_even(5));
}
```

Given `pub fn is_even(n: i32) -> bool { n % 2 == 0 }`, both pass. With no custom
message a failure gives you the source text and nothing else:

```rust
#[test]
fn bad_assert_no_message() {
    let s = "abc";
    assert!(s.starts_with('z'));
}
```

Output (intentional failure):

```
---- tests::bad_assert_no_message stdout ----

thread 'tests::bad_assert_no_message' (13844792) panicked at src/lib.rs:35:9:
assertion failed: s.starts_with('z')
```

You learn *what* was false, never *what the value was*. Add a format message:

```rust
#[test]
fn bad_assert() {
    let v = vec![1, 2, 3];
    assert!(v.len() > 5, "vec too short: {:?}", v);
}
```

Output (intentional failure):

```
---- tests::bad_assert stdout ----

thread 'tests::bad_assert' (13844791) panicked at src/lib.rs:29:9:
vec too short: [1, 2, 3]
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The custom message **replaces** the expression text rather than appending, so
interpolate the values yourself.

## assert_eq! and the left/right diff

Requires `PartialEq` and `Debug`. In exchange you get both values on failure.

```rust
#[test]
fn bad_eq() {
    assert_eq!(add(2, 2), 5);
}
```

Output (intentional failure):

```
---- tests::bad_eq stdout ----

thread 'tests::bad_eq' (13844793) panicked at src/lib.rs:17:9:
assertion `left == right` failed
  left: 4
 right: 5
```

`left` is the first argument. Rust has no "expected vs actual" convention —
`assert_eq!` is symmetric. Here a custom message is appended after a colon and
the values still print:

```rust
#[test]
fn bad_eq_with_message() {
    let n = 7;
    assert_eq!(add(2, 2), 5, "adding 2+2 with input n={}", n);
}
```

Output (intentional failure):

```
---- tests::bad_eq_with_message stdout ----

thread 'tests::bad_eq_with_message' (13844794) panicked at src/lib.rs:23:9:
assertion `left == right` failed: adding 2+2 with input n=7
  left: 4
 right: 5
```

That diff is why `assert_eq!` beats `assert!(a == b)`. For your own types,
derive both traits and it comes free:

```rust
#[derive(Debug, PartialEq)]
pub struct User {
    pub name: String,
    pub age: u32,
}

#[test]
fn bad_struct_eq() {
    let a = User { name: "raj".into(), age: 30 };
    let b = User { name: "raj".into(), age: 31 };
    assert_eq!(a, b);
}
```

Output (intentional failure):

```
---- tests::bad_struct_eq stdout ----

thread 'tests::bad_struct_eq' (13844796) panicked at src/lib.rs:42:9:
assertion `left == right` failed
  left: User { name: "raj", age: 30 }
 right: User { name: "raj", age: 31 }
```

There is no field-level diff in std — each whole value prints on one line. For
large structs that gets unreadable; `use pretty_assertions::assert_eq;` swaps in
a colored multi-line diff.

## assert_ne!

Asserts inequality, and still prints both values on failure.

```rust
#[test]
fn bad_ne() {
    assert_ne!(add(2, 2), 4);
}
```

Output (intentional failure):

```
---- tests::bad_ne stdout ----

thread 'tests::bad_ne' (13844795) panicked at src/lib.rs:47:9:
assertion `left != right` failed
  left: 4
 right: 4
```

## The failure summary

Each failure's stdout, then a list, then the counts:

```
running 6 tests
test tests::bad_eq_with_message ... FAILED
test tests::bad_struct_eq ... FAILED
test tests::bad_eq ... FAILED

failures:

---- tests::bad_eq stdout ----
...

failures:
    tests::bad_assert
    tests::bad_eq
    tests::bad_struct_eq

test result: FAILED. 0 passed; 6 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

The second `failures:` block is the copy-paste list. `0 measured` is always zero
unless you use unstable `#[bench]` (13.5).

## #[should_panic]

Asserts the test *does* panic. `#[test]` goes first.

```rust
pub fn take_first(v: &[i32]) -> i32 {
    v[0]
}

#[test]
#[should_panic]
fn empty_slice_panics() {
    take_first(&[]);
}
```

Output:

```
test tests::empty_slice_panics - should panic ... ok
```

Bare `#[should_panic]` is blunt — *any* panic passes, including one from a typo
in your own test. Pin it with `expected`, a **substring** match:

```rust
pub fn new_age(age: i32) -> i32 {
    if age < 0 {
        panic!("age must be non-negative, got {age}");
    }
    age
}

#[test]
#[should_panic(expected = "age must be non-negative")]
fn negative_age_panics() {
    new_age(-1);
}
```

Output:

```
test tests::negative_age_panics - should panic ... ok
```

A mismatch prints both strings:

```rust
#[test]
#[should_panic(expected = "must be positive")]
fn wrong_expected_message() {
    new_age(-1);
}
```

Output (intentional failure):

```
---- tests::wrong_expected_message stdout ----

thread 'tests::wrong_expected_message' (13846012) panicked at src/lib.rs:11:9:
age must be non-negative, got -1
note: panic did not contain expected string
      panic message: "age must be non-negative, got -1"
 expected substring: "must be positive"
```

If the code does not panic at all, the whole report is one line:

```
---- tests::does_not_panic_at_all stdout ----
note: test did not panic as expected at src/lib.rs:48:8
```

## Tests returning Result

`Ok(())` passes, `Err` fails — so you can use `?` instead of chained
`.unwrap()`.

```rust
pub fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
    s.parse::<u16>()
}

#[test]
fn parses() -> Result<(), std::num::ParseIntError> {
    let p = parse_port("8080")?;
    assert_eq!(p, 8080);
    Ok(())
}
```

Output:

```
test tests::parses ... ok
```

`Box<dyn Error>` is usual when several error types flow through one test. The
failure output is just the error's `Debug` — no line number, the tradeoff:

```rust
#[test]
fn result_test_failing() -> Result<(), String> {
    let v = parse_port("70000").map_err(|e| e.to_string())?;
    assert_eq!(v, 0);
    Ok(())
}
```

Output (intentional failure):

```
---- tests::result_test_failing stdout ----
Error: "number too large to fit in target type"
```

You cannot combine `#[should_panic]` with a `Result` test — `?` returns, it
does not panic.

## #[ignore]

Skips a test by default. The optional reason string prints in the output.

```rust
#[test]
#[ignore = "slow: builds a big vec"]
fn slow_test() {
    let v: Vec<i32> = (0..10_000_000).collect();
    assert_eq!(v.len(), 10_000_000);
}
```

Output:

```
test tests::slow_test ... ignored, slow: builds a big vec

test result: ok. 5 passed; 4 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`cargo test -- --ignored` runs *only* those
(`1 passed; ... 9 filtered out`); `--include-ignored` runs everything.

## Filtering by name

A bare argument is a **substring** match on the full test path:

```
cargo test float
```

Output:

```
test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 8 filtered out; finished in 0.00s
```

`filtered out` counts what the filter skipped. For one exact test,
`cargo test -- --exact tests::parses` gives
`1 passed; 0 failed; 0 ignored; 0 measured; 9 filtered out`.

`cargo test --lib -- --list` dumps every test name (`tests::parses: test`, one
per line) and runs nothing.

Mind the two-level argument split: flags before `--` go to **cargo** (`--lib`,
`--release`), flags after `--` go to the **test binary** (`--nocapture`,
`--ignored`, `--exact`, `--test-threads`).

## println in tests and --nocapture

Output from passing tests is swallowed; failing tests show theirs in the
`---- name stdout ----` block.

```rust
pub fn mean(v: &[f64]) -> f64 {
    v.iter().sum::<f64>() / v.len() as f64
}

#[test]
fn prints_things() {
    println!("computing mean");
    let m = mean(&[1.0, 2.0, 4.0]);
    println!("mean = {m}");
    assert!((m - 2.3333333333333335).abs() < 1e-9);
}
```

`cargo test prints_things` shows nothing but `ok`. With `-- --nocapture`:

```
running 1 test
computing mean
mean = 2.3333333333333335
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 9 filtered out; finished in 0.00s
```

`--show-output` is friendlier: capture stays on, but passing tests' output
prints in a labeled block afterward.

## --test-threads

Tests run in parallel, one thread each, in arbitrary order. Force serial
execution when tests share a resource (a file, an env var, a port):

```
cargo test -- --test-threads=1 --nocapture
```

With parallel threads, `--nocapture` lines from different tests interleave — set
`--test-threads=1` whenever you add it. Parallelism is also why no test may
depend on another having run; there is no shared setup phase (13.4).

## Floats need an epsilon

Never `assert_eq!` two computed floats.

```rust
#[test]
fn float_naive_eq_fails() {
    assert_eq!(0.1 + 0.2, 0.3);
}
```

Output (intentional failure):

```
---- tests::float_naive_eq_fails stdout ----

thread 'tests::float_naive_eq_fails' (13846007) panicked at src/lib.rs:90:9:
assertion `left == right` failed
  left: 0.30000000000000004
 right: 0.3
```

Compare the absolute difference against a tolerance instead:

```rust
#[test]
fn float_epsilon() {
    let x: f64 = 0.1 + 0.2;
    assert!((x - 0.3).abs() < f64::EPSILON);
}
```

Output:

```
test tests::float_epsilon ... ok
```

`f64::EPSILON` (~`2.2e-16`) is right only for values near 1.0. For accumulated
results pick your own (`1e-9`) or compare relatively:
`(a - b).abs() / b.abs() < 1e-9`. The `approx` crate has `assert_relative_eq!`.

The annotation `let x: f64` is required: without it the literal stays an
ambiguous `{float}` and `x.abs()` gives `error[E0689]`.

## debug_assert!

`debug_assert!`, `debug_assert_eq!` and `debug_assert_ne!` behave like their
siblings but **compile to nothing** when `debug_assertions` is off, the release
default. Use them for internal invariants too expensive to check in production.

```rust
pub fn halve(n: u32) -> u32 {
    debug_assert_eq!(n % 2, 0, "halve expects an even number");
    n / 2
}

#[test]
fn debug_assert_eq_trips() {
    halve(7);
}
```

`cargo test` (intentional failure):

```
---- tests::debug_assert_eq_trips stdout ----

thread 'tests::debug_assert_eq_trips' (13847304) panicked at src/lib.rs:7:5:
assertion `left == right` failed: halve expects an even number
  left: 1
 right: 0
```

The same test under `cargo test --release`:

```
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 4 filtered out; finished in 0.00s
```

It passes vacuously because the check no longer exists. Never put a check your
correctness depends on in a `debug_assert!`.

## Quick Reference

| Item | Meaning |
|---|---|
| `#[test]` | Marks a test function; no args, returns `()` or `Result` |
| `#[cfg(test)]` | Compile only under `cargo test` |
| `mod tests { use super::*; }` | Conventional unit-test module; sees private items |
| `assert!(cond)` | Fails printing the expression text |
| `assert!(cond, "fmt {}", x)` | Custom message **replaces** the expression text |
| `assert_eq!(a, b)` | Needs `PartialEq + Debug`; prints `left`/`right` |
| `assert_ne!(a, b)` | Asserts inequality; also prints both |
| `assert_eq!(a, b, "fmt")` | Message **appended**, values still printed |
| `#[should_panic]` | Any panic passes |
| `#[should_panic(expected = "s")]` | Panic message must *contain* `s` |
| `-> Result<(), E>` | Enables `?`; `Err` fails the test |
| `#[ignore = "why"]` | Skipped unless `--ignored`/`--include-ignored` |
| `debug_assert*!` | Removed in release builds |
| `cargo test name` | Substring filter on test path |
| `cargo test -- --exact p::n` | Exact single test |
| `cargo test -- --ignored` | Only ignored tests |
| `cargo test -- --nocapture` | Show `println!` from passing tests |
| `cargo test -- --show-output` | Same, but grouped per test |
| `cargo test -- --test-threads=1` | Serial execution |
| `cargo test --lib -- --list` | List names, run nothing |
| `cargo test --lib` | Unit tests only, skip integration and doc tests |

## Common Mistakes

**Forgetting `use super::*;`.** The test module is a child module, so every item
it tests must be imported — you get `cannot find function` otherwise.

**Using `assert!(a == b)` instead of `assert_eq!`.** You lose the `left`/`right`
values and only see `assertion failed: a == b`.

**Expecting a custom `assert!` message to append.** It replaces the expression
text; interpolate the values yourself.

**Bare `#[should_panic]`.** It passes on *any* panic, including an unwrap typo in
the test. Always add `expected = "..."`.

**`assert_eq!` on floats.** `0.1 + 0.2 != 0.3`. Compare `(a - b).abs()` against
a tolerance.

**Relying on a `debug_assert!` for a real check.** It compiles out in release,
so the test passes vacuously under `cargo test --release`.

**Assuming test order or shared state.** Tests run in parallel in arbitrary
order. Pass `--test-threads=1` or make each test self-contained.

## Exercises

1. Write `pub fn fizzbuzz(n: u32) -> String` and a `mod tests` covering 1, 3, 5,
   15, and 0. Make one assertion fail on purpose and read the `left`/`right`
   output.
2. Add a **private** `fn classify(n: u32) -> u8` helper that `fizzbuzz` calls,
   and test it directly without making it `pub`.
3. Write `pub fn parse_pair(s: &str) -> Result<(i32, i32), String>` that splits
   on `,`. Test the happy path with a `Result`-returning test using `?`, and the
   two error paths with `assert!(matches!(..))`.
4. Add a function that panics on invalid input. Write one test with
   `#[should_panic(expected = "...")]` that passes, then change the expected
   string and confirm the mismatch output.
5. Write a float test that fails with `assert_eq!`, then fix it with an epsilon
   comparison. Mark a slow test `#[ignore]` and run it with `--ignored`.

---

[← 12.7 Publishing to crates.io](../12-modules-crates/07-publishing.md) · [Course Index](../README.md) · [13.2 Integration Tests →](02-integration-tests.md)
