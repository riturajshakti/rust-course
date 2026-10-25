[← 13.3 Doc Tests](03-doc-tests.md) · [Course Index](../README.md) · [13.5 Benchmarking and Criterion →](05-benchmarks.md)

# 13.4 — Test Organization and Fixtures

## What You'll Learn

- Where a test belongs: unit vs integration, and the rule for deciding
- Fixture functions and test builders for setup data
- Setup/teardown without framework support: `Drop` guards and closure wrappers
- Table-driven tests, trait fakes, `tempfile`, and injecting time

## Coming From Another Language

| Other | Rust |
|---|---|
| pytest `@fixture`, `conftest.py` | A plain `fn` returning the value |
| Jest `beforeEach` / `afterEach` | Closure wrapper, or a struct with `Drop` |
| Go `TestMain`, table tests with `t.Run` | A `for` loop over a slice of cases |
| `unittest.mock.patch` | Generic over a trait, pass a fake impl |

Rust has **no** `beforeEach`, no `@fixture`, no subtests, and no monkey-patch
mocking. Everything here is built from ordinary functions, traits and `Drop` —
less magic to learn, but you write the plumbing yourself.

## Where a test belongs

| Put it in | When |
|---|---|
| `#[cfg(test)] mod tests` in the source file | It touches a private function, or tests one unit in isolation |
| `tests/*.rs` | It drives the public API the way a user would |
| `///` doc comment | It doubles as documentation for a `pub` item |

The deciding question is not size but **what the test needs to see**. A test
needing a private item has only one option (13.2). A test that could go either
way should be an integration test: it then constrains only your public contract
and survives refactors of your internals.

A practical split as a library grows: internal invariants unit-tested in-file;
one integration file per public feature area (`tests/auth.rs`,
`tests/pricing.rs`); a doc test on every `pub` function. If a `mod tests` grows
past a few hundred lines, the source file probably wants splitting too.

## Fixture functions

The simplest fixture is a function returning a known-good value. Call it from
every test that needs one.

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct User {
    pub name: String,
    pub email: String,
    pub age: u32,
    pub active: bool,
}

#[derive(Debug, PartialEq)]
pub enum ValidationError {
    EmptyName,
    BadEmail(String),
    TooYoung(u32),
}

pub fn validate(u: &User) -> Result<(), ValidationError> {
    if u.name.is_empty() {
        return Err(ValidationError::EmptyName);
    }
    if !u.email.contains('@') {
        return Err(ValidationError::BadEmail(u.email.clone()));
    }
    if u.age < 18 {
        return Err(ValidationError::TooYoung(u.age));
    }
    Ok(())
}
```


```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn valid_user() -> User {
        User {
            name: "raj".to_string(),
            email: "raj@example.com".to_string(),
            age: 30,
            active: true,
        }
    }

    #[test]
    fn fixture_is_valid() {
        assert_eq!(validate(&valid_user()), Ok(()));
    }
}
```

Output:

```
test tests::fixture_is_valid ... ok
```

Each test gets a **fresh** value, since the function constructs a new one every
call — no shared mutable fixture to leak between parallel tests. An unused
fixture gives a `dead_code` warning; `#[allow(dead_code)]` silences it.

## Test builders

A fixture breaks down when each test needs a different small deviation —
`valid_user_but_young()`, `valid_user_bad_email()` does not scale. Use a builder
that starts from the fixture and overrides one field:

```rust
struct UserBuilder {
    user: User,
}

impl UserBuilder {
    fn new() -> Self { UserBuilder { user: valid_user() } }
    fn name(mut self, n: &str) -> Self { self.user.name = n.to_string(); self }
    fn email(mut self, e: &str) -> Self { self.user.email = e.to_string(); self }
    fn age(mut self, a: u32) -> Self { self.user.age = a; self }
    fn build(self) -> User { self.user }
}

#[test]
fn builder_overrides_one_field() {
    let u = UserBuilder::new().age(12).build();
    assert_eq!(validate(&u), Err(ValidationError::TooYoung(12)));
}
```

Output:

```
test tests::builder_overrides_one_field ... ok
```

Each setter takes `mut self` and returns `Self`, so calls chain. The test states
exactly one thing — "age 12" — while the other fields stay valid unmentioned. A
test should name only what it varies.

For simple cases struct update syntax needs no builder at all:
`User { age: 12, ..valid_user() }`.

## Table-driven tests

One `#[test]` looping over a slice of cases, instead of one test per case. Put a
label in each row and pass it as the assert message so a failure identifies the
row.

```rust
pub fn slugify(s: &str) -> String {
    s.trim()
        .to_lowercase()
        .chars()
        .map(|c| if c.is_alphanumeric() { c } else { '-' })
        .collect::<String>()
        .trim_matches('-')
        .to_string()
}

#[test]
fn slugify_table() {
    let cases = [
        ("Hello World", "hello-world"),
        ("  Trim Me  ", "trim-me"),
        ("a!!!b", "a---b"),
        ("", ""),
        ("---", ""),
        ("Ünïcode OK", "ünïcode-ok"),
    ];
    for (input, expected) in cases {
        assert_eq!(slugify(input), expected, "input: {input:?}");
    }
}
```

Output:

```
test tests::slugify_table ... ok
```

A table makes edge cases cheap — empty string, all-separator and non-ASCII are
one line each, and they document that `slugify` collapses nothing (`a!!!b` to
`a---b`, not `a-b`). The failure message is why labels matter:

```rust
#[test]
fn slugify_table_with_a_failure() {
    let cases = [("Hello World", "hello-world"), ("a!!!b", "a-b")];
    for (input, expected) in cases {
        assert_eq!(slugify(input), expected, "input: {input:?}");
    }
}
```

Output (intentional failure):

```
---- tests::slugify_table_with_a_failure stdout ----

thread 'tests::slugify_table_with_a_failure' (13960836) panicked at src/lib.rs:185:13:
assertion `left == right` failed: input: "a!!!b"
  left: "a---b"
 right: "a-b"
```

Without the message you would see only the `assert`'s line number inside the
loop and have to count rows. The tradeoff versus one test per case: a table
**stops at the first failing row** and reports one pass/fail. Rust has no `t.Run`
subtests, so write separate tests if each case must report independently; for
exhaustive input coverage use property testing (13.6).

Tables pair well with the builder — a row of
`("too young", UserBuilder::new().age(17).build(), ValidationError::TooYoung(17))`
varies exactly one field, and the loop asserts
`assert_eq!(validate(&user), Err(expected), "case: {label}")`.

## Testing error paths

Three ways to assert on an `Err`, in order of strictness:

```rust
assert_eq!(validate(&u), Err(ValidationError::TooYoung(12)));
assert!(matches!(validate(&u), Err(ValidationError::TooYoung(_))));
assert!(validate(&u).is_err());
```

Prefer the first — it pins the payload, and needing `PartialEq` on your error
type is a good reason to derive it. Use `matches!` when the payload is not worth
pinning, `is_err()` only when any error will do.

Some error types deliberately lack `PartialEq`; `std::io::Error` is the common
one, giving `error[E0369]: binary operation == cannot be applied to type
Result<usize, std::io::Error>`. For those, match on `.kind()`:

```rust
assert_eq!(err.kind(), std::io::ErrorKind::NotFound);
```

A suite with no failing-input tests covers half your code. For every happy path,
ask what happens with empty, zero, too large, malformed and missing.

## Setup and teardown with Drop

There is no `afterEach`. The replacement is a guard struct whose `Drop` does the
cleanup — strictly better, because `Drop` also runs during unwind, so cleanup
happens even when the test panics.

```rust
struct TempDirGuard {
    path: std::path::PathBuf,
}

impl TempDirGuard {
    fn new(name: &str) -> Self {
        let path = std::env::temp_dir().join(name);
        std::fs::create_dir_all(&path).unwrap();
        println!("setup: created {}", path.display());
        TempDirGuard { path }
    }
}

impl Drop for TempDirGuard {
    fn drop(&mut self) {
        std::fs::remove_dir_all(&self.path).ok();
        println!("teardown: removed {}", self.path.display());
    }
}


#[test]
fn drop_guard_cleans_up() {
    let g = TempDirGuard::new("w13-drop-guard");
    let f = g.path.join("a.txt");
    std::fs::write(&f, "one\ntwo\n").unwrap();
    assert_eq!(count_lines(&f).unwrap(), 2);
}
```

Output of `cargo test drop_guard_cleans -- --nocapture`:

```
running 1 test
setup: created /var/folders/3l/.../T/w13-drop-guard
teardown: removed /var/folders/3l/.../T/w13-drop-guard
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 10 filtered out
```

And the important property — cleanup on panic:

```rust
#[test]
fn drop_guard_runs_even_on_panic() {
    let g = TempDirGuard::new("w13-panic-guard");
    assert!(g.path.exists());
    panic!("boom");
}
```

Output (intentional failure):

```
---- tests::drop_guard_runs_even_on_panic stdout ----
setup: created /var/folders/3l/.../T/w13-panic-guard

thread 'tests::drop_guard_runs_even_on_panic' (13960831) panicked at src/lib.rs:221:9:
boom
teardown: removed /var/folders/3l/.../T/w13-panic-guard
```

The `teardown:` line prints **after** the panic message. A bare cleanup
statement at the end of a test body would not have run at all.

Bind the guard to a name. `let _ = TempDirGuard::new(..)` drops it immediately;
`let _g = ...` holds it for the test's duration.

## Setup and teardown with a closure wrapper

A function that builds the fixture, hands it to a closure, then cleans up.

```rust
fn with_store<F: FnOnce(&mut MemStore)>(f: F) {
    let mut s = MemStore::new();
    s.put("seed", "value");
    println!("setup: seeded store");
    f(&mut s);
    println!("teardown: dropping store");
}

#[test]
fn closure_wrapper_fixture() {
    with_store(|s| {
        assert_eq!(s.get("seed"), Some("value".to_string()));
        assert_eq!(upsert_name(s, "u1", "raj"), None);
        assert_eq!(upsert_name(s, "u1", "sam"), Some("raj".to_string()));
    });
}
```

Output of `cargo test closure_wrapper -- --nocapture`:

```
running 1 test
setup: seeded store
teardown: dropping store
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 10 filtered out
```

The caveat: if the closure panics, the lines after `f(&mut s)` **do not run**.
Use a `Drop` guard when cleanup must be guaranteed, this when it is merely
convenient. (`std::panic::catch_unwind` fixes it, at the cost of `UnwindSafe`
bounds.)

## Mocking with traits

Rust's answer to `mock.patch` is to make the function generic over a trait and
pass a different implementation in tests. No framework required.

```rust
pub trait Store {
    fn get(&self, k: &str) -> Option<String>;
    fn put(&mut self, k: &str, v: &str);
}

pub fn upsert_name<S: Store>(store: &mut S, id: &str, name: &str) -> Option<String> {
    let old = store.get(id);
    store.put(id, name);
    old
}
```


Production passes the real store; the test passes a fake exercising a path the
real one makes awkward — here a store that never retains anything, so the
"previous value" return is always `None`:

```rust
struct FailingStore;

impl Store for FailingStore {
    fn get(&self, _k: &str) -> Option<String> { None }
    fn put(&mut self, _k: &str, _v: &str) {}
}

#[test]
fn trait_fake_for_error_path() {
    let mut s = FailingStore;
    assert_eq!(upsert_name(&mut s, "u1", "raj"), None);
    assert_eq!(upsert_name(&mut s, "u1", "sam"), None);
}
```

Output:

```
test tests::trait_fake_for_error_path ... ok
```

The dominant Rust style: depend on a trait rather than a concrete type. Define
the fake inside `mod tests` and it never ships.

`mockall` (`cargo add --dev mockall`, currently 0.15) generates mocks when you
need call-count and argument assertions. `#[cfg_attr(test, mockall::automock)]`
on the trait yields a `MockStore` with an expectation API:

```rust
let mut m = MockStore::new();
m.expect_get().with(eq("u1")).times(1).returning(|_| None);
```

Reach for it only when you must assert *how* a dependency was called. For
"return this value", a five-line fake is clearer and compiles faster.

## Temp files with tempfile

Never write test files to a fixed path: parallel tests collide and failed runs
leave litter. `tempfile` gives a unique directory that deletes itself on drop.

```rust
pub fn count_lines(path: &std::path::Path) -> std::io::Result<usize> {
    Ok(std::fs::read_to_string(path)?.lines().count())
}

#[test]
fn tempfile_crate() -> std::io::Result<()> {
    use std::io::Write;
    let dir = tempfile::tempdir()?;
    let path = dir.path().join("data.txt");
    let mut f = std::fs::File::create(&path)?;
    writeln!(f, "alpha")?;
    writeln!(f, "beta")?;
    f.sync_all()?;
    assert_eq!(count_lines(&path)?, 2);
    Ok(())
}
```

Output:

```
test tests::tempfile_crate ... ok
```

`tempdir()` returns a `TempDir` whose `Drop` removes the tree — the guard
pattern, pre-written. `NamedTempFile::new()` does the same for one file, and
`tempfile()` gives an unnamed handle with no path. Hold the `TempDir` for the
whole test; dropping it early deletes the directory under you.

## Injecting time and randomness

A function calling `SystemTime::now()` or a global RNG is untestable — you
cannot assert on output that changes every run. Take the source as a parameter.

```rust
pub trait Clock {
    fn now(&self) -> u64;
}

pub struct SystemClock;

impl Clock for SystemClock {
    fn now(&self) -> u64 {
        std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs()
    }
}

pub fn make_session<C: Clock>(clock: &C) -> Session {
    Session { created: clock.now() }
}

pub fn is_expired<C: Clock>(s: &Session, clock: &C, ttl: u64) -> bool {
    clock.now() - s.created >= ttl
}
```


`Cell<u64>` lets the fake advance behind `&self`, so it satisfies `Clock`
without `&mut`:

```rust
struct FakeClock {
    t: std::cell::Cell<u64>,
}

impl FakeClock {
    fn at(t: u64) -> Self { FakeClock { t: std::cell::Cell::new(t) } }
    fn advance(&self, by: u64) { self.t.set(self.t.get() + by); }
}

impl Clock for FakeClock {
    fn now(&self) -> u64 { self.t.get() }
}

#[test]
fn expiry_with_injected_clock() {
    let clock = FakeClock::at(1_000);
    let s = make_session(&clock);
    assert_eq!(s.created, 1_000);
    assert!(!is_expired(&s, &clock, 60));
    clock.advance(59);
    assert!(!is_expired(&s, &clock, 60));
    clock.advance(1);
    assert!(is_expired(&s, &clock, 60));
}
```

Output:

```
test tests::expiry_with_injected_clock ... ok
```

The boundary at exactly `ttl` is now testable deterministically with no
sleeping. A test that `thread::sleep`s to wait out a timeout is slow and flaky.

Randomness works the same way: take an `impl Rng` and pass
`StdRng::seed_from_u64(42)` for a fixed sequence. The rule — **push
nondeterminism to the edges and inject it** — also covers environment variables,
the current directory and network clients.

## Naming conventions

Nothing is enforced, but the name is all you see in the output:

- Describe behaviour, not the function: `rejects_empty_name`, not `test_validate_2`.
- Do not prefix `test_` — the attribute says so and the path prints `tests::`.
- Name condition and expectation: `expired_session_is_rejected`,
  `missing_file_returns_not_found`.
- Fixtures read as nouns (`valid_user`), wrappers as `with_*` (`with_store`).

## Test-only helpers behind #[cfg(test)]

For a test-only accessor that exposes an internal without widening the real
API, gate it:

```rust
impl MemStore {
    #[cfg(test)]
    pub fn raw_len(&self) -> usize {
        self.data.len()
    }
}
```

It exists only under `cargo test`, so consumers cannot call it and it stays out
of your docs. The same attribute works on whole `impl` blocks and modules.

The limit: `#[cfg(test)]` is **not** set when an *integration* test compiles your
library, because the library is built as an ordinary dependency. From `tests/`:

```
error[E0599]: no method named `raw_len` found for struct `MemStore` in the current scope
 --> tests/probe.rs:4:15
  |
4 |     let _ = s.raw_len();
  |               ^^^^^^^ method not found in `MemStore`
```

So such a helper is visible to unit tests only. For something integration tests
need, gate it behind a Cargo feature:

```toml
[features]
test-util = []
```

```rust
#[cfg(feature = "test-util")]
pub fn raw_len(&self) -> usize { self.data.len() }
```

Then `cargo test --features test-util`.

## Quick Reference

| Pattern | Form |
|---|---|
| Fixture | `fn valid_user() -> User { .. }`, called per test |
| Override one field | `User { age: 12, ..valid_user() }` |
| Builder | `UserBuilder::new().age(12).build()` |
| Table test | `for (label, input, expected) in cases` + label as assert message |
| Teardown, panic-safe | guard struct with `impl Drop` |
| Teardown, convenient | `fn with_x<F: FnOnce(&mut X)>(f: F)` |
| Temp dir / file | `tempfile::tempdir()?` / `NamedTempFile::new()?`, bound to a name |
| Fake dependency | generic over a trait, test impl in `mod tests` |
| Generated mock | `#[cfg_attr(test, mockall::automock)]` |
| Injected clock | `trait Clock { fn now(&self) -> u64; }` + `Cell<u64>` fake |
| Seeded RNG | `StdRng::seed_from_u64(42)` |
| Error assert, strict to loose | `assert_eq!(r, Err(E::V(x)))`, `matches!`, `is_err()` |
| `io::Error` assert | `assert_eq!(e.kind(), ErrorKind::NotFound)` |
| Unit-test-only helper | `#[cfg(test)]` on the item |
| Integration-visible helper | `#[cfg(feature = "test-util")]` |

## Common Mistakes

**Sharing mutable state between tests.** Tests run in parallel; a `static mut` or
a fixed temp path causes flakes. Construct fresh per test.

**Cleanup at the end of the test body.** It is skipped on panic. Use a `Drop`
guard.

**Dropping a `TempDir` or guard early.** `let _ = tempfile::tempdir()?` deletes
immediately. Bind it to a name.

**A table test with no row label.** You get a line number inside the loop and
have to count rows. Pass the label as the assert message.

**Calling `SystemTime::now()` deep in business logic.** Inject a `Clock`; do not
`thread::sleep` in tests to wait out a timeout.

**Expecting `#[cfg(test)]` helpers in integration tests.** The flag is not set
when your library is compiled as a dependency. Use a feature.

**Reaching for `mockall` before a hand-written fake.** A five-line test impl is
usually clearer and compiles faster.

## Exercises

1. Write `pub struct Order { id: u32, items: Vec<String>, total: u32, paid: bool }`
   with a `fn valid_order()` fixture and an `OrderBuilder`. Test three
   validation failures, each varying one field.
2. Convert those three tests into one table-driven test with labels. Break one
   expected value and confirm the label appears in the failure.
3. Write a `Drop` guard that creates and removes a temp directory, print from
   both `new` and `drop`, and confirm with `--nocapture` that teardown runs when
   the test panics. Then rewrite it using `tempfile::tempdir()`.
4. Define `trait Mailer { fn send(&mut self, to: &str, body: &str) -> Result<(), String>; }`
   and a function that uses it. Write a `RecordingMailer` fake that collects sent
   messages, and a `FailingMailer` that always errors. Test both paths.
5. Add a `Clock` parameter to a function that expires something after 30
   seconds. Test the boundary at 29, 30 and 31 seconds with a fake clock, with
   no `sleep`.

---

[← 13.3 Doc Tests](03-doc-tests.md) · [Course Index](../README.md) · [13.5 Benchmarking and Criterion →](05-benchmarks.md)
