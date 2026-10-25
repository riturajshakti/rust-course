[← Course Index](../README.md)

# 8.1 — panic! and When to Use It

## What You'll Learn

- What a panic prints, and how to read it
- Every operation in the standard library that panics
- `unwrap`/`expect` and the exact messages they produce
- `catch_unwind`, `panic = "abort"`, and why neither makes panics into exceptions

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `throw` | `panic!` — but there is no `catch` in normal code |
| Python `raise ValueError` | `return Err(...)` (lesson 8.2), not `panic!` |
| Go `panic` | `panic!` — same intent, same "this is a bug" meaning |
| Go `err != nil` | `Result<T, E>` (lesson 8.2) |

Rust splits failure into two mechanisms that do not overlap. **`Result` is for
expected failure** — a file is missing, input is malformed, a user does not
exist. **`panic!` is for bugs** — an invariant you promised is broken. There is
no `try`/`catch`; a panic unwinds the stack and kills the thread.

## panic! and what it prints

```rust
fn main() {
    println!("before");
    panic!("something went terribly wrong");
}
```

Output:

```
before

thread 'main' (13780953) panicked at src/main.rs:3:5:
something went terribly wrong
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Four parts: the thread name and its id, the exact source location, your
message, and the backtrace hint. Everything after the panic is not executed.
The process exits with code **101**. The number in parentheses is the thread
id — yours will differ.

`panic!` takes format arguments exactly like `println!`:

```rust
let id = 42;
panic!("bad config for user {id}: missing field `name`");
```

Output:

```
thread 'main' (13781041) panicked at src/main.rs:3:5:
bad config for user 42: missing field `name`
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

## unwrap on None vs on Err

`unwrap` is `panic!` with a message the standard library chose for you. The two
cases print differently:

```rust
let v: Option<i32> = None;
let x = v.unwrap();
```

Output:

```
thread 'main' (13781132) panicked at src/main.rs:3:15:
called `Option::unwrap()` on a `None` value
```

```rust
let r: Result<i32, String> = Err("file not found".to_string());
let x = r.unwrap();
```

Output:

```
thread 'main' (13781405) panicked at src/main.rs:3:15:
called `Result::unwrap()` on an `Err` value: "file not found"
```

The `Result` version appends the `Debug` of the error; the `Option` version has
nothing to show. That is the single best argument for `Result` over `Option` in
a fallible API.

## expect replaces the message

```rust
let v: Option<i32> = None;
let x = v.expect("config value must be present");
```

Output:

```
thread 'main' (13781305) panicked at src/main.rs:3:15:
config value must be present
```

On a `Result`, `expect` keeps your message **and** appends the error:

```rust
let r = "abc".parse::<i32>();
let x = r.expect("input should be a number");
```

Output:

```
thread 'main' (13781552) panicked at src/main.rs:3:15:
input should be a number: ParseIntError { kind: InvalidDigit }
```

Idiomatic: never use `unwrap` where `expect` fits. Write the expectation as a
statement of the invariant — "literal 8080 is a valid u16" — not as "failed to
parse", which the message already tells you.

## Index out of bounds

```rust
let v = vec![1, 2, 3];
let i = 10;
println!("{}", v[i]);
```

Output:

```
thread 'main' (13782078) panicked at src/main.rs:4:21:
index out of bounds: the len is 3 but the index is 10
```

`get(i)` returns `Option<&T>` and never panics. String slicing has its own
message:

```rust
let s = String::from("hello");
println!("{}", &s[0..10]);
```

Output:

```
thread 'main' (13782180) panicked at src/main.rs:3:22:
end byte index 10 is out of bounds for string of length 5
```

## Integer overflow panics in debug, wraps in release

```rust
let mut x: u8 = 250;
for _ in 0..10 {
    x += 1;
    println!("{x}");
}
```

Output (`cargo run`):

```
251
252
253
254
255

thread 'main' (13782278) panicked at src/main.rs:4:9:
attempt to add with overflow
```

The same binary built with `cargo run --release`:

```
0
```

Release builds have `overflow-checks = false` by default, so the add wraps
silently. This is the one place where debug and release *behaviour* differ.
Use `checked_add`, `saturating_add`, or `wrapping_add` when overflow is
possible rather than relying on the profile.

## Divide by zero

With a divisor the compiler can see, this is a compile error, not a panic:

```rust
fn main() {
    println!("{}", 10 / 0);
}
```

Does not compile (intentional):

```
error: this operation will panic at runtime
 --> src/main.rs:2:20
  |
2 |     println!("{}", 10 / 0);
  |                    ^^^^^^ attempt to divide `10_i32` by zero
  |
  = note: `#[deny(unconditional_panic)]` on by default
```

Hide the divisor behind a function and it becomes a runtime panic:

```rust
fn divide(a: i32, b: i32) -> i32 {
    a / b
}

fn main() {
    println!("{}", divide(10, 2));
    println!("{}", divide(10, 0));
}
```

Output:

```
5

thread 'main' (13783027) panicked at src/main.rs:2:5:
attempt to divide by zero
```

`%` panics the same way, with "attempt to calculate the remainder of `10_i32`
with a divisor of zero". `checked_div` returns `Option`. Floats do **not**
panic:

```rust
println!("{}", 10.0 / 0.0);
println!("{}", 0.0 / 0.0);
println!("{}", (-10.0) / 0.0);
```

Output:

```
inf
NaN
-inf
```

## todo!, unimplemented!, unreachable!

All three are `panic!` with a fixed intent.

```rust
fn parse_config(_s: &str) -> u32 {
    todo!()
}
```

Output:

```
thread 'main' (13784398) panicked at src/main.rs:2:5:
not yet implemented
```

```rust
fn export_pdf() {
    unimplemented!("PDF export")
}
```

Output:

```
thread 'main' (13784482) panicked at src/main.rs:2:5:
not implemented: PDF export
```

`todo!()` means "I will write this"; `unimplemented!()` means "this variant
will never be supported". Both type-check as any return type, so they let a
skeleton compile. `unreachable!()` documents a branch the type system cannot
rule out:

```rust
fn classify(n: i32) -> &'static str {
    match n.signum() {
        -1 => "negative",
        0 => "zero",
        1 => "positive",
        _ => unreachable!("signum returned {n}"),
    }
}
```

Output:

```
negative
zero
positive
```

## Assertions

```rust
let total = 7;
assert_eq!(total, 10, "total should balance");
```

Output:

```
thread 'main' (13784568) panicked at src/main.rs:3:5:
assertion `left == right` failed: total should balance
  left: 7
 right: 10
```

`assert!`, `assert_eq!`, `assert_ne!` run in every profile. The `debug_assert*`
family compiles away in release — use it for expensive invariant checks.

## RUST_BACKTRACE=1

```rust
fn level_three() {
    panic!("deep failure");
}

fn level_two() {
    level_three();
}

fn main() {
    level_two();
}
```

Output with `RUST_BACKTRACE=1 cargo run`:

```
thread 'main' (13785138) panicked at src/main.rs:2:5:
deep failure
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: w08::level_three
             at ./src/main.rs:2:5
   3: w08::level_two
             at ./src/main.rs:6:5
   4: w08::main
             at ./src/main.rs:10:5
   5: <fn() as core::ops::function::FnOnce<()>>::call_once
             at /opt/homebrew/Cellar/rust/1.99.1/lib/rustlib/src/rust/library/core/src/ops/function.rs:250:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

Read frames 2 upward — frames 0 and 1 are the panic machinery. `RUST_BACKTRACE=full`
adds runtime startup frames and is usually noise.

## Panics in tests are how tests fail

```rust
fn withdraw(balance: u64, amount: u64) -> u64 {
    if amount > balance {
        panic!("overdraft: balance {balance}, requested {amount}");
    }
    balance - amount
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ok_withdrawal() {
        assert_eq!(withdraw(100, 30), 70);
    }

    #[test]
    #[should_panic(expected = "overdraft")]
    fn rejects_overdraft() {
        withdraw(10, 50);
    }

    #[test]
    fn wrong_expectation() {
        assert_eq!(withdraw(100, 30), 71);
    }
}
```

Output of `cargo test`:

```
running 3 tests
.. 2/3
tests::wrong_expectation --- FAILED

failures:

---- tests::wrong_expectation stdout ----

thread 'tests::wrong_expectation' (13787042) panicked at src/main.rs:29:9:
assertion `left == right` failed
  left: 70
 right: 71

failures:
    tests::wrong_expectation

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`#[should_panic(expected = "...")]` passes when the panic message *contains*
the substring. Inside tests, `unwrap` is fine — a failing `unwrap` is a failing
test, which is exactly what you want.

## catch_unwind is not exception handling

```rust
use std::panic;

fn main() {
    let result = panic::catch_unwind(|| {
        println!("inside");
        panic!("boom");
    });

    match result {
        Ok(()) => println!("no panic"),
        Err(payload) => {
            if let Some(s) = payload.downcast_ref::<&str>() {
                println!("caught panic: {s}");
            }
        }
    }
    println!("still running");
}
```

Output:

```
inside

thread 'main' (13785952) panicked at src/main.rs:6:9:
boom
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
caught panic: boom
still running
```

Note the panic message **still printed** — `catch_unwind` does not suppress it.
It exists for FFI boundaries (a panic must not unwind into C) and for thread
pools and test harnesses that need to survive a worker dying. It is not for
control flow: the error payload is `Box<dyn Any>`, there is no type
information, and nothing guarantees your data structures are consistent
afterwards. The payload is `&str` for a literal message and `String` for a
formatted one, so you end up downcasting twice.

## panic = "abort"

```toml
[profile.release]
panic = "abort"
```

With that set, `cargo run --release`:

```
start

thread 'main' (13786364) panicked at src/main.rs:3:5:
fatal
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Exit code is **134** (SIGABRT) instead of 101. No unwinding, so no `Drop`
impls run and `catch_unwind` cannot catch anything. In exchange the binary is
smaller and has no landing pads. Standard for embedded and for CLI tools;
wrong for a server that runs request handlers on a pool.

## When panicking is correct

| Situation | Panic? |
|---|---|
| An invariant your own code guarantees is broken | Yes |
| Index/slice bounds you already validated | Yes (`unwrap` after a `len` check) |
| A `const`/literal you know parses | Yes, with `expect` and a reason |
| Inside a test | Yes |
| Prototype, not-yet-written code path | Yes (`todo!`) |
| Unrecoverable startup failure in a binary | Acceptable; a `Result` from `main` is nicer |
| Malformed user input | **No** — `Result` |
| Missing file, network failure, bad env var | **No** — `Result` |
| Anything in a library reachable from caller input | **No** — `Result` |

The rule: panic when continuing would be *wrong*, return `Err` when continuing
is the caller's decision. A library that panics on bad input takes that
decision away from everyone who depends on it.

## Quick Reference

| Form | Panic message |
|---|---|
| `panic!("msg {x}")` | `msg <x>` |
| `Option::unwrap()` on `None` | `called \`Option::unwrap()\` on a \`None\` value` |
| `Result::unwrap()` on `Err(e)` | `called \`Result::unwrap()\` on an \`Err\` value: <e:?>` |
| `expect("why")` on `None` | `why` |
| `expect("why")` on `Err(e)` | `why: <e:?>` |
| `v[i]` out of range | `index out of bounds: the len is N but the index is M` |
| `&s[a..b]` past end | `end byte index B is out of bounds for string of length N` |
| `x += 1` overflowing (debug) | `attempt to add with overflow` |
| `a / b`, `b == 0` | `attempt to divide by zero` |
| `todo!()` | `not yet implemented` |
| `unimplemented!("x")` | `not implemented: x` |
| `assert_eq!(a, b)` | `assertion \`left == right\` failed` + both values |
| Exit code, unwind | 101 |
| Exit code, `panic = "abort"` | 134 |

| Non-panicking alternative | Returns |
|---|---|
| `v.get(i)` | `Option<&T>` |
| `s.get(a..b)` | `Option<&str>` |
| `a.checked_add(b)` | `Option<T>` |
| `a.saturating_add(b)` | `T`, clamped |
| `a.wrapping_add(b)` | `T`, wrapped |
| `a.checked_div(b)` | `Option<T>` |

## Common Mistakes

**Using `panic!` for expected failure.** Missing files and bad input are not
bugs. Return `Result` (lesson 8.2) and let the caller decide.

**Reaching for `catch_unwind` as `try`/`catch`.** It still prints the panic,
gives you an untyped payload, and does nothing under `panic = "abort"`.

**Relying on overflow panics to find bugs.** They are off in release. Use
`checked_*` when the value can genuinely overflow.

**Bare `unwrap()` in non-test code.** Use `expect("<the invariant>")` so the
panic tells the next reader what was assumed, not just that something failed.

**Writing `expect("failed to parse port")`.** The message already says it
failed. Say *why it should have worked*: `expect("PORT validated at startup")`.

## Exercises

1. Write `fn nth(v: &[i32], i: usize) -> i32` that indexes directly, and call
   it out of range. Then rewrite it to return `Option<i32>` using `get`.
2. Trigger `attempt to multiply with overflow` on an `i8`, then make the same
   code print `127` using a `saturating_*` method.
3. Write a function that panics with a formatted message including two
   variables, and capture its output with `RUST_BACKTRACE=1`.
4. Add a `#[should_panic(expected = "...")]` test for exercise 3. Then change
   the `expected` substring so the test fails, and read the failure message.
5. Set `panic = "abort"` in a `[profile.release]` section, add a type with a
   `Drop` impl that prints, and confirm the `Drop` does not run on panic in
   release but does in debug.

---

[← Course Index](../README.md)
