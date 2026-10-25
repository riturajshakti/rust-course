[← Course Index](../README.md)

# 8.5 — Custom Error Types

## What You'll Learn

- Writing an error enum by hand: `Display`, `Error`, `source()`
- `From` impls so `?` converts your dependencies' errors automatically
- A worked example merging `io::Error` and `ParseIntError` into one type
- Enums vs structs, `Box<dyn Error>`, and `#[non_exhaustive]`

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `class MyError extends Error` | `enum MyError` + `impl Display` + `impl Error` |
| Python exception hierarchy | One enum per layer; variants, not subclasses |
| Go `errors.Is` / `errors.As` | `match` on your enum; `source()` for the chain |
| Go `fmt.Errorf("%w", err)` | A variant holding the cause + `source()` |

Rust has no inheritance, so there is no error hierarchy. One enum per module
boundary, with a variant per way that boundary can fail, is the whole pattern.
`std::error::Error` is a trait, not a base class — implementing it buys you
`Box<dyn Error>` compatibility and the `source()` chain.

Lesson 8.6 shows how `thiserror` generates everything below from attributes.
Write it by hand once so the macro is not magic.

## The three pieces

An error type needs `Debug` (derive it), `Display` (write it), and
`std::error::Error` (usually empty).

```rust
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    Missing(String),
    Invalid { key: String, value: String },
    Empty,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::Missing(k) => write!(f, "missing key `{k}`"),
            ConfigError::Invalid { key, value } => {
                write!(f, "key `{key}` has invalid value `{value}`")
            }
            ConfigError::Empty => write!(f, "config file is empty"),
        }
    }
}

impl std::error::Error for ConfigError {}

fn main() {
    let errs = [
        ConfigError::Missing("port".into()),
        ConfigError::Invalid { key: "port".into(), value: "abc".into() },
        ConfigError::Empty,
    ];
    for e in &errs {
        println!("display: {e}");
        println!("debug:   {e:?}");
    }
    let boxed: Box<dyn std::error::Error> = Box::new(ConfigError::Empty);
    println!("as dyn Error: {boxed}");
}
```

Output:

```
display: missing key `port`
debug:   Missing("port")
display: key `port` has invalid value `abc`
debug:   Invalid { key: "port", value: "abc" }
display: config file is empty
debug:   Empty
as dyn Error: config file is empty
```

`impl Error for ConfigError {}` with an empty body is enough — every method has
a default. `Error` requires `Debug + Display`, which is why both are mandatory.

Message style: lowercase, no trailing period, no "error:" prefix (the caller
adds that), and name the thing that failed. `"missing key `port`"` composes
into `"error: loading config: missing key `port`"`; `"Error: Could not find the
key."` does not.

## Variants should be matchable, not just printable

A variant exists so the caller can *react*, not only log. Compare:

```rust
#[derive(Debug)]
enum Bad {
    Failed(String),
}

#[derive(Debug)]
enum Good {
    NotFound { id: u64 },
    RateLimited { retry_after_secs: u32 },
    Unauthorized,
}

fn main() {
    let e = Good::RateLimited { retry_after_secs: 30 };
    match e {
        Good::NotFound { id } => println!("no such record {id}"),
        Good::RateLimited { retry_after_secs } => println!("sleep {retry_after_secs}s and retry"),
        Good::Unauthorized => println!("refresh the token"),
    }
    println!("{:?}", Bad::Failed("something".into()));
}
```

Output:

```
sleep 30s and retry
Failed("something")
```

Hold the *data* the caller needs, not a pre-formatted string.

## From impls make ? convert automatically

`?` calls `From::from` on the error. Implement `From<TheirError> for YourError`
once and every `?` in your module converts for free.

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum LoadError {
    Io(std::io::Error),
    Parse(ParseIntError),
    OutOfRange(u32),
}

impl fmt::Display for LoadError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            LoadError::Io(_) => write!(f, "could not read settings file"),
            LoadError::Parse(_) => write!(f, "settings file does not contain a number"),
            LoadError::OutOfRange(n) => write!(f, "{n} is outside 1..=65535"),
        }
    }
}

impl Error for LoadError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            LoadError::Io(e) => Some(e),
            LoadError::Parse(e) => Some(e),
            LoadError::OutOfRange(_) => None,
        }
    }
}

impl From<std::io::Error> for LoadError {
    fn from(e: std::io::Error) -> Self { LoadError::Io(e) }
}

impl From<ParseIntError> for LoadError {
    fn from(e: ParseIntError) -> Self { LoadError::Parse(e) }
}

fn load_port(path: &str) -> Result<u16, LoadError> {
    let text = std::fs::read_to_string(path)?;
    let n: u32 = text.trim().parse()?;
    if n == 0 || n > 65535 {
        return Err(LoadError::OutOfRange(n));
    }
    Ok(n as u16)
}

fn report(e: &dyn Error) {
    println!("error: {e}");
    let mut src = e.source();
    while let Some(s) = src {
        println!("  caused by: {s}");
        src = s.source();
    }
}

fn main() {
    std::fs::write("port.txt", "8080").unwrap();
    println!("{:?}", load_port("port.txt"));

    std::fs::write("port.txt", "99999").unwrap();
    if let Err(e) = load_port("port.txt") { report(&e); }

    std::fs::write("port.txt", "eighty").unwrap();
    if let Err(e) = load_port("port.txt") { report(&e); }

    if let Err(e) = load_port("gone.txt") { report(&e); }
    let _ = std::fs::remove_file("port.txt");
}
```

Output:

```
Ok(8080)
error: 99999 is outside 1..=65535
error: settings file does not contain a number
  caused by: invalid digit found in string
error: could not read settings file
  caused by: No such file or directory (os error 2)
```

This is the whole pattern. Two unrelated errors from two unrelated crates, two
bare `?` in the body, one error type out. No `map_err` anywhere.

## source() is what builds the chain

Note the `Display` impls above print **nothing** about the cause — `LoadError::Io`
says only "could not read settings file". The specific OS message comes from
`source()`. That separation is deliberate:

- `Display` says what *your layer* was doing.
- `source()` says what the layer below reported.

The `report` loop walks the chain. Duplicating the cause into your own
`Display` gives you "could not read settings file: No such file or directory
(os error 2): No such file or directory (os error 2)" once a caller also walks
the chain. Say your part and only your part.

`source()` returns `Option<&(dyn Error + 'static)>` — that exact signature,
including the `'static`, which exists so callers can downcast.

## Enum vs struct

Use a struct when there is exactly one way to fail and the variation is in the
data:

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct ValidationError {
    field: &'static str,
    reason: String,
}

impl fmt::Display for ValidationError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "field `{}`: {}", self.field, self.reason)
    }
}

impl Error for ValidationError {}

fn check_age(n: i64) -> Result<u8, ValidationError> {
    if !(0..=130).contains(&n) {
        return Err(ValidationError {
            field: "age",
            reason: format!("{n} not in 0..=130"),
        });
    }
    Ok(n as u8)
}

fn main() {
    println!("{:?}", check_age(42));
    match check_age(900) {
        Ok(a) => println!("{a}"),
        Err(e) => println!("{e} / {} / {}", e.field, e.reason),
    }
}
```

Output:

```
Ok(42)
field `age`: 900 not in 0..=130 / age / 900 not in 0..=130
```

| Shape | Use when |
|---|---|
| Enum | Several distinct failure modes the caller handles differently |
| Struct | One failure mode, parameterised (validation, a single parser) |
| Struct with an enum `kind` field | You want to add context fields without touching every variant |

The `kind`-field shape is what `std::io::Error` does, and it is how you keep the
public type stable while the variant list grows.

## Box<dyn Error> for quick cases

In a binary, or a prototype, skip the type entirely:

```rust
use std::error::Error;

fn parse_pair(s: &str) -> Result<(i32, i32), Box<dyn Error>> {
    let (a, b) = s.split_once(',').ok_or("expected `a,b`")?;
    Ok((a.trim().parse()?, b.trim().parse()?))
}

fn main() {
    println!("{:?}", parse_pair("3, 4"));
    for bad in ["34", "a,4", "3,b"] {
        match parse_pair(bad) {
            Ok(p) => println!("{p:?}"),
            Err(e) => println!("{bad:?} -> {e}"),
        }
    }
}
```

Output:

```
Ok((3, 4))
"34" -> expected `a,b`
"a,4" -> invalid digit found in string
"3,b" -> invalid digit found in string
```

It accepts anything `Error`, plus bare `&str` and `String`. The trade-off:
callers can only print it or `downcast_ref`, never `match`. Across threads you
need `Box<dyn Error + Send + Sync>`.

## Downcasting a Box<dyn Error>

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct Timeout { secs: u32 }

impl fmt::Display for Timeout {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "timed out after {}s", self.secs)
    }
}

impl Error for Timeout {}

fn call() -> Result<(), Box<dyn Error>> {
    Err(Box::new(Timeout { secs: 30 }))
}

fn main() {
    if let Err(e) = call() {
        println!("{e}");
        if let Some(t) = e.downcast_ref::<Timeout>() {
            println!("retryable, waited {}s", t.secs);
        }
        println!("{}", e.is::<Timeout>());
    }
}
```

Output:

```
timed out after 30s
retryable, waited 30s
true
```

Downcasting works, but it is a runtime string-free type test the compiler cannot
check for exhaustiveness. An enum is better wherever the caller must branch.

## #[non_exhaustive]

Adding a variant to a public enum is a breaking change — downstream `match`es
stop compiling. `#[non_exhaustive]` forces callers to write a `_` arm from day
one, so adding variants later is semver-compatible.

```rust
#[derive(Debug)]
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    RateLimited { retry_after: u32 },
}
```

Inside the defining crate nothing changes — you can still match exhaustively.
From another crate, a complete-looking match is rejected:

```rust
fn describe(e: &api::ApiError) -> &'static str {
    match e {
        api::ApiError::NotFound => "404",
        api::ApiError::RateLimited { .. } => "429",
    }
}
```

Does not compile from a downstream crate (intentional):

```
error[E0004]: non-exhaustive patterns: `&_` not covered
 --> user.rs:2:11
  |
2 |     match e {
  |           ^ pattern `&_` not covered
  |
note: `ApiError` defined here
 --> lib.rs:3:1
  |
3 | pub enum ApiError {
  | ^^^^^^^^^^^^^^^^^
  = note: the matched value is of type `&ApiError`
  = note: `ApiError` is marked as non-exhaustive, so a wildcard `_` is necessary to match exhaustively
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
  |
4 ~         ne::ApiError::RateLimited { .. } => "429",
5 ~         &_ => todo!(),
  |
```

Adding `_ => "500"` fixes it. The same attribute works on structs (callers
cannot use a struct literal or exhaustive destructure) and on individual
variants (callers cannot construct them).

Put it on every public error enum in a library. The cost is one `_` arm for
callers; the benefit is that you are not stuck with your first guess at the
variant list forever.

## Checklist for a public error type

```rust
#[derive(Debug)]
#[non_exhaustive]
pub enum MyError { /* ... */ }
```

- `#[derive(Debug)]` — required by the `Error` trait and by `unwrap`'s message.
- `impl Display` — lowercase, no period, describe *your* layer only.
- `impl Error` — plus `source()` when a variant wraps a cause.
- `impl From<Dep>` for each dependency error you propagate with `?`.
- `#[non_exhaustive]` if it is public.
- Do **not** derive `Clone`/`PartialEq` reflexively: `io::Error` is neither, and
  deriving them pins you.
- Prefer owned data (`String`, not `&'a str`) so the type has no lifetime.

## Quick Reference

| Need | Write |
|---|---|
| Minimal error type | `#[derive(Debug)] enum E {}` + `impl Display` + `impl Error for E {}` |
| Human message | `impl fmt::Display` with `write!(f, "...")` |
| Expose the cause | `fn source(&self) -> Option<&(dyn Error + 'static)>` |
| Let `?` convert | `impl From<Other> for E` |
| Walk the chain | `let mut s = e.source(); while let Some(c) = s { ...; s = c.source(); }` |
| Catch-all in a binary | `Box<dyn Error>` |
| Catch-all across threads | `Box<dyn Error + Send + Sync>` |
| Recover a concrete type | `e.downcast_ref::<T>()`, `e.is::<T>()` |
| Keep adding variants semver-safely | `#[non_exhaustive]` |
| One failure mode | A struct, not an enum |

## Common Mistakes

**Duplicating the cause in `Display`.** If a variant has a `source()`, your
message should describe your layer only, or the chain prints the same text
twice.

**A single `Failed(String)` variant.** That is `Box<dyn Error>` with extra
steps. Give each failure mode a variant with the data a caller needs.

**Forgetting `source()`.** Without it, `#[from]`-style wrapping loses the
original error and your "could not read settings file" has no OS message behind
it.

**Deriving `PartialEq` on an error wrapping `io::Error`.** It does not compile;
`io::Error` is not `PartialEq`. Compare on a `kind`, not the whole error.

**Omitting `#[non_exhaustive]` on a public enum.** Your next variant is then a
breaking release.

## Exercises

1. Define `enum CalcError { DivByZero, Overflow, BadInput(String) }` with
   `Display` and `Error`. Print each with `{}` and `{:?}`.
2. Write `fn calc(a: &str, b: &str, op: char) -> Result<i64, CalcError>` with a
   `From<ParseIntError>` impl so `?` works on both parses.
3. Add a variant wrapping `std::io::Error`, implement `source()`, and write a
   loop that prints the full chain for a missing file.
4. Convert `CalcError` from an enum to a struct holding a `kind: CalcErrorKind`
   plus an `input: String` context field. Note which call sites changed.
5. Make the enum `pub` and `#[non_exhaustive]`, then build a second crate that
   matches on it. Confirm the E0004 error, then fix it.

---

[← Course Index](../README.md)
