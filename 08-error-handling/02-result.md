[← Course Index](../README.md)

# 8.2 — Result and Propagation

## What You'll Learn

- `Result<T, E>` and every way to inspect one
- Returning `Result` from functions and from `main`
- The `#[must_use]` warning you get for ignoring one
- `std::io::Result`, nested `Result`s, and the `unwrap_or` family

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `(T, error)` | `Result<T, E>` — one value, not two |
| TS `T \| Error` | `Result<T, E>`, but you cannot use the `T` without checking |
| Python `try/except` | `match` on `Result`, or `?` (lesson 8.3) |

`Result` is a plain enum with two variants, no magic. The difference from Go is
that the compiler will not let you touch the value without handling the error
case, and it warns if you throw the whole thing away.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Both variants are in the prelude: write `Ok(x)` and `Err(e)`, never
`Result::Ok(x)`.

## Returning Result

```rust
fn parse_port(s: &str) -> Result<u16, String> {
    match s.parse::<u16>() {
        Ok(p) if p > 0 => Ok(p),
        Ok(_) => Err("port 0 is not allowed".to_string()),
        Err(e) => Err(format!("not a number: {e}")),
    }
}

fn main() {
    println!("{:?}", parse_port("8080"));
    println!("{:?}", parse_port("0"));
    println!("{:?}", parse_port("abc"));
    println!("{:?}", parse_port("99999"));
}
```

Output:

```
Ok(8080)
Err("port 0 is not allowed")
Err("not a number: invalid digit found in string")
Err("not a number: number too large to fit in target type")
```

`"99999"` fails inside `parse::<u16>` with a different `ParseIntError` kind than
`"abc"` — the error type carries which.

## match on Result

```rust
let r: Result<i32, std::num::ParseIntError> = "42".parse();
match r {
    Ok(n) => println!("got {n}"),
    Err(e) => println!("failed: {e}"),
}

match "4x2".parse::<i32>() {
    Ok(n) => println!("got {n}"),
    Err(e) => println!("failed: {e}"),
}

match "4x2".parse::<i32>() {
    Err(e) => println!("debug: {e:?}"),
    Ok(_) => {}
}
```

Output:

```
got 42
failed: invalid digit found in string
debug: ParseIntError { kind: InvalidDigit }
```

`{}` on an error gives the human-readable `Display`; `{:?}` gives the
structured `Debug`. Log with `Display`, debug with `Debug`.

## The #[must_use] warning

`Result` is annotated `#[must_use]`, so ignoring one is a warning:

```rust
fn save(name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err("empty name".to_string());
    }
    Ok(())
}

fn main() {
    save("");
    println!("done");
}
```

Compiles, but warns:

```
warning: unused `Result` that must be used
 --> src/main.rs:9:5
  |
9 |     save("");
  |     ^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
help: use `let _ = ...` to ignore the resulting value
  |
9 |     let _ = save("");
  |     +++++++

done
```

`let _ = save("");` silences it and documents that you meant to discard it.
Add `#![deny(unused_must_use)]` to a crate when you want this to be an error.

## Predicates and conversions

```rust
let ok: Result<i32, String> = Ok(5);
let er: Result<i32, String> = Err("nope".into());

println!("{} {}", ok.is_ok(), ok.is_err());
println!("{} {}", er.is_ok(), er.is_err());
println!("{:?} {:?}", ok.clone().ok(), ok.clone().err());
println!("{:?} {:?}", er.clone().ok(), er.clone().err());
println!("{:?}", ok.as_ref());
println!("{}", ok.is_ok_and(|n| n > 3));
println!("{}", er.is_err_and(|e| e.contains("nope")));
```

Output:

```
true false
false true
Some(5) None
None Some("nope")
Ok(5)
true
true
```

`ok()` and `err()` both **consume** the `Result` and throw the other half away.
`as_ref()` turns `&Result<T, E>` into `Result<&T, &E>` so you can inspect
without moving.

## The unwrap_or family

```rust
let ok: Result<i32, String> = Ok(5);
let er: Result<i32, String> = Err("boom".into());

println!("{}", ok.clone().unwrap_or(0));
println!("{}", er.clone().unwrap_or(0));
println!("{}", er.clone().unwrap_or_else(|e| e.len() as i32));
println!("{}", er.clone().unwrap_or_default());
println!("{}", er.clone().unwrap_err());
let d: Result<i32, String> = Ok(9);
println!("{}", d.unwrap_or_default());
```

Output:

```
5
0
4
0
boom
9
```

- `unwrap_or(v)` — `v` is evaluated eagerly, even on `Ok`. Fine for literals.
- `unwrap_or_else(f)` — lazy, and `f` receives the error. Use this when the
  fallback allocates or is expensive.
- `unwrap_or_default()` — needs `T: Default`.
- `unwrap_err()` — panics on `Ok`; mostly a test helper.

## Result from main

`main` may return `Result<(), E>` for any `E: Debug`.

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let n: i32 = "17".parse()?;
    println!("parsed {n}");
    Ok(())
}
```

Output:

```
parsed 17
```

Exit code 0. Return an `Err` and the runtime prints it and exits **1**:

```rust
use std::error::Error;
use std::fs;

fn main() -> Result<(), Box<dyn Error>> {
    let text = fs::read_to_string("/nonexistent/path.txt")?;
    println!("{text}");
    Ok(())
}
```

Output:

```
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

**The runtime prints `Debug`, not `Display`.** That is why a returned
`io::Error` looks like a struct dump instead of "No such file or directory
(os error 2)". Same for other error types:

```rust
fn main() -> Result<(), String> {
    Err("plain string failure".to_string())
}
```

Output:

```
Error: "plain string failure"
```

Note the quotes — that is `Debug` on a `String`. And:

```rust
fn main() -> Result<(), std::num::ParseIntError> {
    let n: i32 = "nope".parse()?;
    println!("{n}");
    Ok(())
}
```

Output:

```
Error: ParseIntError { kind: InvalidDigit }
```

For a CLI you actually ship, either use `anyhow` (lesson 8.6), whose `Debug` is
written to look like a human message, or keep a thin `main` that calls a
`run() -> Result<...>` and formats the error itself:

```rust
fn run() -> Result<(), String> {
    Err("could not reach the database".to_string())
}

fn main() {
    if let Err(e) = run() {
        eprintln!("error: {e}");
        std::process::exit(1);
    }
}
```

Output (on stderr):

```
error: could not reach the database
```

`Ok(())` is mandatory as the last expression — the unit value must be wrapped.

## std::io::Result

Most of `std::io` returns `io::Result<T>`, an alias for
`Result<T, std::io::Error>`:

```rust
use std::fs;
use std::io;

fn read(path: &str) -> io::Result<String> {
    fs::read_to_string(path)
}

fn main() {
    match read("Cargo.toml") {
        Ok(s) => println!("{} bytes", s.len()),
        Err(e) => println!("error: {e}"),
    }
    match read("missing.txt") {
        Ok(s) => println!("{} bytes", s.len()),
        Err(e) => {
            println!("error: {e}");
            println!("kind: {:?}", e.kind());
            println!("debug: {e:?}");
        }
    }
}
```

Output:

```
114 bytes
error: No such file or directory (os error 2)
kind: NotFound
debug: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

`e.kind()` gives an `ErrorKind` enum you can match on — `NotFound`,
`PermissionDenied`, `AlreadyExists`, `WouldBlock`, and so on. Match on `kind()`
rather than parsing the message. Many crates define the same alias pattern:
`serde_json::Result<T>`, `fmt::Result`.

## Nested Results

```rust
fn outer(s: &str) -> Result<Result<i32, String>, String> {
    if s.is_empty() {
        return Err("no input".to_string());
    }
    Ok(s.parse::<i32>().map_err(|e| e.to_string()))
}

fn main() {
    println!("{:?}", outer(""));
    println!("{:?}", outer("5"));
    println!("{:?}", outer("x"));
    println!("{:?}", outer("5").and_then(|inner| inner));
    println!("{:?}", outer("x").flatten());
}
```

Output:

```
Err("no input")
Ok(Ok(5))
Ok(Err("invalid digit found in string"))
Ok(5)
Err("invalid digit found in string")
```

`flatten()` collapses `Result<Result<T, E>, E>` into `Result<T, E>` when both
error types match. Nested `Result` in a public signature is almost always a
design mistake — merge the two failure modes into one enum (lesson 8.5).

## Result in collections

```rust
let results: Vec<Result<i32, String>> = vec![Ok(1), Err("bad".into()), Ok(3)];
println!("{}", results.iter().filter(|r| r.is_ok()).count());
println!("{:?}", results.iter().filter_map(|r| r.as_ref().ok()).collect::<Vec<_>>());

let all: Result<Vec<i32>, _> = ["1", "2", "3"]
    .iter()
    .map(|s| s.parse::<i32>())
    .collect();
println!("{all:?}");
```

Output:

```
2
[1, 3]
Ok([1, 2, 3])
```

Collecting an iterator of `Result`s straight into `Result<Vec<_>, E>` is one of
the most useful tricks in the language; lesson 8.4 covers it and its
short-circuiting behaviour.

## Quick Reference

| Method | Signature sketch | Notes |
|---|---|---|
| `is_ok()` / `is_err()` | `-> bool` | Borrows |
| `is_ok_and(f)` / `is_err_and(f)` | `-> bool` | Consumes |
| `ok()` | `-> Option<T>` | Discards the error |
| `err()` | `-> Option<E>` | Discards the value |
| `as_ref()` | `&Result<T,E> -> Result<&T,&E>` | Inspect without moving |
| `as_mut()` | `-> Result<&mut T, &mut E>` | |
| `unwrap()` | `-> T` | Panics on `Err` |
| `expect("why")` | `-> T` | Panics with your message + `{e:?}` |
| `unwrap_err()` | `-> E` | Panics on `Ok` |
| `unwrap_or(v)` | `-> T` | Eager fallback |
| `unwrap_or_else(f)` | `-> T` | Lazy; `f(E) -> T` |
| `unwrap_or_default()` | `-> T` | Needs `T: Default` |
| `flatten()` | `Result<Result<T,E>,E> -> Result<T,E>` | |
| `iter()` | 0 or 1 item | |

| Return shape | What a returned `Err` prints |
|---|---|
| `fn main() -> Result<(), Box<dyn Error>>` | `Error: ` + `{e:?}`, exit 1 |
| `fn main() -> Result<(), String>` | `Error: "the message"`, exit 1 |
| `fn main() -> anyhow::Result<()>` | `Error: msg` + a `Caused by:` chain, exit 1 |
| `fn main()` + manual `eprintln!` | Whatever you format |

## Common Mistakes

**Calling `.unwrap()` in a function that already returns `Result`.** Use `?`
(lesson 8.3) — the error propagates for free.

**Expecting `main` to print `Display`.** It prints `Debug`. `io::Error` and
friends look ugly; format the error yourself or use `anyhow`.

**Using `Result<T, ()>`.** You have made a worse `Option`. Use `Option<T>` if
there is genuinely nothing to say about the failure.

**Ignoring a `Result` with a bare statement.** The `#[must_use]` warning is
there to be read. If you really mean it, write `let _ = ...`.

**Returning nested `Result<Result<T, E1>, E2>`.** Flatten the failure modes into
one error enum (lesson 8.5); callers should not unwrap twice.

## Exercises

1. Write `fn safe_div(a: i32, b: i32) -> Result<i32, String>` returning an
   `Err` when `b == 0`. Call it both ways and print with `{:?}`.
2. Call a function returning `Result` and ignore the value. Read the warning,
   then silence it correctly.
3. Write `fn read_number(path: &str) -> Result<i32, String>` using
   `fs::read_to_string` and `map_err` so both failure modes become `String`.
4. Make `main` return `Result<(), Box<dyn std::error::Error>>` and have it fail.
   Record the exact output and the exit code (`echo $?`).
5. Rewrite exercise 4 as a thin `main` wrapping `run()` that prints
   `error: <display>` to stderr and exits 1.

---

[← Course Index](../README.md)
