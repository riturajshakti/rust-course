[← Course Index](../README.md)

# 8.6 — thiserror and anyhow

## What You'll Learn

- `thiserror`: deriving `Display`, `Error`, `source()`, and `From` from attributes
- `#[from]`, `#[source]`, `#[error(transparent)]`
- `anyhow`: `Result`, `anyhow!`, `bail!`, `ensure!`, `.context()`
- The `Caused by:` output, `downcast_ref`, and which crate to pick

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `fmt.Errorf("doing x: %w", err)` | `.context("doing x")?` (anyhow) |
| Python `raise X from e` | `#[from]` or `#[source]` (thiserror) |
| TS `new AppError(msg, { cause })` | a `thiserror` variant wrapping the cause |

Two crates, one job each. **`thiserror` writes the boilerplate from lesson 8.5
for you** — you still get a concrete, matchable type. **`anyhow` gives up the
concrete type** in exchange for a one-line error you can attach context to.
Libraries take the first, applications the second.

```bash
cargo add thiserror anyhow
```

```toml
[dependencies]
anyhow = "1.0.104"
thiserror = "2.0.21"
```

## thiserror: the whole of lesson 8.5 in attributes

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("missing key `{0}`")]
    Missing(String),

    #[error("key `{key}` has invalid value `{value}`")]
    Invalid { key: String, value: String },

    #[error("config file is empty")]
    Empty,

    #[error("too many entries: {count} (max {max})")]
    TooMany { count: usize, max: usize },
}

fn main() {
    let errs = [
        ConfigError::Missing("port".into()),
        ConfigError::Invalid { key: "port".into(), value: "abc".into() },
        ConfigError::Empty,
        ConfigError::TooMany { count: 500, max: 100 },
    ];
    for e in &errs {
        println!("{e}");
    }
    println!("{:?}", errs[0]);
    let d: &dyn std::error::Error = &errs[1];
    println!("as dyn: {d}");
}
```

Output:

```
missing key `port`
key `port` has invalid value `abc`
config file is empty
too many entries: 500 (max 100)
Missing("port")
as dyn: key `port` has invalid value `abc`
```

`#[derive(Error)]` generates `impl Display` from the `#[error("...")]` strings
and `impl std::error::Error`. The format strings use `{0}`, `{1}` for tuple
fields and `{name}` for named fields — no `self.` prefix. You still derive
`Debug` yourself.

`thiserror` is a proc macro with **no runtime component**. The generated code
is exactly what you would have written, so there is no dependency in your
public API and nothing for callers to install.

## #[from] generates the From impl and source()

The 8.5 example, now in a third of the lines:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum LoadError {
    #[error("could not read settings file")]
    Io(#[from] std::io::Error),

    #[error("settings file does not contain a number")]
    Parse(#[from] std::num::ParseIntError),

    #[error("{0} is outside 1..=65535")]
    OutOfRange(u32),
}

fn load_port(path: &str) -> Result<u16, LoadError> {
    let text = std::fs::read_to_string(path)?;
    let n: u32 = text.trim().parse()?;
    if n == 0 || n > 65535 {
        return Err(LoadError::OutOfRange(n));
    }
    Ok(n as u16)
}

fn report(e: &dyn std::error::Error) {
    println!("error: {e}");
    let mut s = e.source();
    while let Some(cur) = s {
        println!("  caused by: {cur}");
        s = cur.source();
    }
}

fn main() {
    std::fs::write("port.txt", "8080").unwrap();
    println!("{:?}", load_port("port.txt"));
    std::fs::write("port.txt", "eighty").unwrap();
    if let Err(e) = load_port("port.txt") { report(&e); }
    if let Err(e) = load_port("gone.txt") { report(&e); }
    std::fs::write("port.txt", "70000").unwrap();
    if let Err(e) = load_port("port.txt") { report(&e); }
    let _ = std::fs::remove_file("port.txt");
}
```

Output:

```
Ok(8080)
error: settings file does not contain a number
  caused by: invalid digit found in string
error: could not read settings file
  caused by: No such file or directory (os error 2)
error: 70000 is outside 1..=65535
```

Identical behaviour to the hand-written version. `#[from]` does **two** things:
generates `impl From<std::io::Error> for LoadError` (so `?` converts) and marks
that field as the `source()`. A variant may have at most one `#[from]` field,
and no other fields.

## #[source] when you do not want a From impl

Use `#[source]` alone when the variant has more fields, or when an automatic
`From` would be wrong (two variants wrapping the same error type cannot both
have `#[from]`).

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum StoreError {
    #[error("could not read {path}")]
    Read {
        path: String,
        #[source]
        cause: std::io::Error,
    },

    #[error("could not write {path}")]
    Write {
        path: String,
        #[source]
        cause: std::io::Error,
    },
}

fn read(path: &str) -> Result<String, StoreError> {
    std::fs::read_to_string(path).map_err(|cause| StoreError::Read {
        path: path.to_string(),
        cause,
    })
}

fn main() {
    if let Err(e) = read("absent.txt") {
        println!("{e}");
        println!("  caused by: {}", std::error::Error::source(&e).unwrap());
    }
}
```

Output:

```
could not read absent.txt
  caused by: No such file or directory (os error 2)
```

A field literally named `source` is treated as the source without the
attribute. Being explicit is clearer.

## #[error(transparent)]

`transparent` forwards both `Display` and `source()` to the inner error,
adding no message of its own. Use it for a pass-through variant.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
#[error("validation failed on `{field}`: {reason}")]
struct ValidationError {
    field: &'static str,
    reason: String,
}

#[derive(Debug, Error)]
enum AppError {
    #[error(transparent)]
    Validation(#[from] ValidationError),

    #[error("database unavailable")]
    Database {
        #[source]
        cause: std::io::Error,
    },

    #[error(transparent)]
    Other(#[from] Box<dyn std::error::Error + Send + Sync>),
}

fn main() {
    let v: AppError = ValidationError { field: "email", reason: "no @".into() }.into();
    println!("{v}");
    println!("{v:?}");

    let d = AppError::Database {
        cause: std::io::Error::new(std::io::ErrorKind::ConnectionRefused, "port 5432 closed"),
    };
    println!("{d}");
    println!("  source: {:?}", std::error::Error::source(&d).map(|s| s.to_string()));

    let o: AppError = AppError::Other("plain message".into());
    println!("{o}");
}
```

Output:

```
validation failed on `email`: no @
Validation(ValidationError { field: "email", reason: "no @" })
database unavailable
  source: Some("port 5432 closed")
plain message
```

Note `#[error("...")]` works on a struct too, and the `Other` variant makes the
enum accept any boxed error without naming it.

## anyhow: one error type for an application

`anyhow::Error` wraps any `E: Error + Send + Sync + 'static`. `anyhow::Result<T>`
is `Result<T, anyhow::Error>`.

```rust
use anyhow::{anyhow, bail, ensure, Result};

fn find_user(id: u32) -> Result<String> {
    if id == 0 {
        bail!("user id 0 is reserved");
    }
    ensure!(id < 1000, "user id {} out of range", id);
    if id == 7 {
        return Err(anyhow!("user {id} was deleted"));
    }
    Ok(format!("user-{id}"))
}

fn main() {
    for id in [42, 0, 5000, 7] {
        match find_user(id) {
            Ok(u) => println!("ok: {u}"),
            Err(e) => println!("err: {e}"),
        }
    }
}
```

Output:

```
ok: user-42
err: user id 0 is reserved
err: user id 5000 out of range
err: user 7 was deleted
```

- `anyhow!("...")` — build an `anyhow::Error`, format args like `println!`.
- `bail!("...")` — `return Err(anyhow!(...))`.
- `ensure!(cond, "...")` — `if !cond { bail!(...) }`. The message is optional;
  without one you get the stringified condition.

Note `Result<String>` has one type parameter — that is the `anyhow` alias, and
it is why `use anyhow::Result` must come after any `std` `Result` import you
still need.

## .context() and the Caused by: chain

This is the reason to use `anyhow` at all.

```rust
use anyhow::{Context, Result};

fn load_port(path: &str) -> Result<u16> {
    let text = std::fs::read_to_string(path)
        .with_context(|| format!("reading config from {path}"))?;
    let n: u16 = text
        .trim()
        .parse()
        .with_context(|| format!("parsing port from {text:?}"))?;
    Ok(n)
}

fn start_server() -> Result<()> {
    let port = load_port("conf.txt").context("server startup failed")?;
    println!("listening on {port}");
    Ok(())
}

fn main() {
    std::fs::write("conf.txt", "8080").unwrap();
    println!("{:?}", start_server());

    std::fs::write("conf.txt", "http").unwrap();
    if let Err(e) = start_server() {
        println!("--- display ---");
        println!("{e}");
        println!("--- debug ---");
        println!("{e:?}");
        println!("--- chain ---");
        for (i, c) in e.chain().enumerate() {
            println!("{i}: {c}");
        }
        println!("root: {}", e.root_cause());
    }
    let _ = std::fs::remove_file("conf.txt");
}
```

Output:

```
listening on 8080
Ok(())
--- display ---
server startup failed
--- debug ---
server startup failed

Caused by:
    0: parsing port from "http"
    1: invalid digit found in string
--- chain ---
0: server startup failed
1: parsing port from "http"
2: invalid digit found in string
root: invalid digit found in string
```

The important line: **`{}` prints only the outermost context, `{:?}` prints the
whole chain.** That is the opposite of normal `Debug`/`Display` convention, and
it is deliberate — `main` prints `Debug`, so a returned `anyhow::Error` shows
the full chain automatically.

`.context(v)` takes a value (eager); `.with_context(|| v)` takes a closure
(lazy). Use `with_context` whenever the message formats or allocates — which is
most of the time, because a context message without the path or id in it is
rarely worth adding.

`Context` is also implemented for `Option`, so `.context("not found")?` turns a
`None` into an error.

## anyhow in main

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let text = std::fs::read_to_string("absent.toml")
        .context("loading application config")?;
    println!("{text}");
    Ok(())
}
```

Output:

```
Error: loading application config

Caused by:
    No such file or directory (os error 2)
```

Exit code 1. Compare lesson 8.2's bare
`Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }`
— that difference, for free, is why CLI tools use `anyhow`.

## downcast_ref recovers the concrete type

`anyhow` erases the type but not the value. You can get it back:

```rust
use anyhow::{Context, Result};
use thiserror::Error;

#[derive(Debug, Error)]
#[error("rate limited, retry after {retry_after}s")]
struct RateLimited { retry_after: u32 }

fn call_api(fail: bool) -> Result<String> {
    if fail {
        return Err(RateLimited { retry_after: 30 }).context("calling /users");
    }
    Ok("payload".into())
}

fn main() {
    match call_api(true) {
        Ok(p) => println!("{p}"),
        Err(e) => {
            println!("{e:?}");
            if let Some(rl) = e.downcast_ref::<RateLimited>() {
                println!("retry_after = {}", rl.retry_after);
            }
            match e.downcast::<RateLimited>() {
                Ok(rl) => println!("owned: {rl}"),
                Err(other) => println!("not rate limited: {other}"),
            }
        }
    }
    println!("{:?}", call_api(false));
}
```

Output:

```
calling /users

Caused by:
    rate limited, retry after 30s
retry_after = 30
owned: rate limited, retry after 30s
Ok("payload")
```

`downcast_ref::<T>()` borrows, `downcast::<T>()` consumes and gives the error
back on failure, and `e.is::<T>()` just tests. It searches the whole chain, so
the context layer does not hide the inner type. This is the escape hatch that
makes `anyhow` usable in a handler that must map one specific error to a
retry — but if you find yourself downcasting in several places, that error
belongs in a `thiserror` enum.

## anyhow requires Send + Sync

```rust
use anyhow::Result;
use thiserror::Error;

#[derive(Debug, Error)]
#[error("not a library error")]
struct NotSend(std::rc::Rc<u8>);

fn f() -> Result<()> {
    Err(NotSend(std::rc::Rc::new(1)).into())
}
```

Does not compile (intentional):

```
error[E0277]: `Rc<u8>` cannot be sent between threads safely
 --> src/main.rs:9:38
  |
9 |     Err(NotSend(std::rc::Rc::new(1)).into())
  |                                      ^^^^ `Rc<u8>` cannot be sent between threads safely
  |
  = help: within `NotSend`, the trait `Send` is not implemented for `Rc<u8>`
note: required because it appears within the type `NotSend`
 --> src/main.rs:6:8
  |
6 | struct NotSend(std::rc::Rc<u8>);
  |        ^^^^^^^
  = note: required for `anyhow::Error` to implement `From<NotSend>`
  = note: required for `NotSend` to implement `Into<anyhow::Error>`
```

Errors travel across threads and `.await` points, so `anyhow::Error` demands
`Send + Sync`. Use owned data (`String`, `Arc`) in error types, never `Rc` or
`RefCell`.

## Using both together

The normal arrangement in a real workspace: each library crate defines a
`thiserror` enum; the binary uses `anyhow` and absorbs them all.

```rust
use anyhow::{Context, Result};
use thiserror::Error;

#[derive(Debug, Error)]
#[non_exhaustive]
pub enum ParseError {
    #[error("line {line}: expected `key=value`")]
    Malformed { line: usize },

    #[error("line {line}: duplicate key `{key}`")]
    Duplicate { line: usize, key: String },
}

fn parse(src: &str) -> Result<Vec<(String, String)>, ParseError> {
    let mut out: Vec<(String, String)> = Vec::new();
    for (i, line) in src.lines().enumerate() {
        let (k, v) = line
            .split_once('=')
            .ok_or(ParseError::Malformed { line: i + 1 })?;
        if out.iter().any(|(existing, _)| existing == k) {
            return Err(ParseError::Duplicate { line: i + 1, key: k.to_string() });
        }
        out.push((k.to_string(), v.to_string()));
    }
    Ok(out)
}

fn run(path: &str) -> Result<()> {
    let src = std::fs::read_to_string(path).with_context(|| format!("opening {path}"))?;
    let pairs = parse(&src).with_context(|| format!("parsing {path}"))?;
    println!("{} pairs", pairs.len());
    Ok(())
}

fn main() {
    std::fs::write("app.conf", "a=1\nb=2\n").unwrap();
    println!("{:?}", run("app.conf"));

    std::fs::write("app.conf", "a=1\nbroken\n").unwrap();
    if let Err(e) = run("app.conf") { println!("{e:?}"); }

    std::fs::write("app.conf", "a=1\na=2\n").unwrap();
    if let Err(e) = run("app.conf") {
        println!("{e:?}");
        if let Some(pe) = e.downcast_ref::<ParseError>() {
            println!("recovered variant: {pe:?}");
        }
    }
    let _ = std::fs::remove_file("app.conf");
}
```

Output:

```
2 pairs
Ok(())
parsing app.conf

Caused by:
    line 2: expected `key=value`
parsing app.conf

Caused by:
    line 2: duplicate key `a`
recovered variant: Duplicate { line: 2, key: "a" }
```

`parse` keeps its precise type for anyone who wants to branch; `run` adds
context and throws away what it does not need.

## Which to pick

| | thiserror | anyhow |
|---|---|---|
| Produces | A concrete type you define | One opaque `anyhow::Error` |
| Caller can `match` | Yes | Only via `downcast` |
| Appears in your public API | Yes | Avoid |
| Context messages | Manual, in `Display` | `.context()` per call site |
| Runtime cost | None (proc macro only) | One boxed allocation per error |
| Backtraces | No | Yes, with the `backtrace` feature |
| Use in | Libraries, domain layers | Binaries, `main`, tests, scripts |

Both are tiny, both are maintained by the same author, and depending on both is
normal. A library that exposes `anyhow::Error` in its signatures forces every
caller to downcast — that is the one combination to avoid.

## Quick Reference

### thiserror

| Attribute | Effect |
|---|---|
| `#[derive(Error)]` | Generates `Display` + `impl Error` |
| `#[error("text")]` | The `Display` string for that variant |
| `#[error("{0} and {1}")]` | Tuple fields by index |
| `#[error("{key}")]` | Named fields by name |
| `#[error("{}", self.0.len())]` | Arbitrary expression form |
| `#[error(transparent)]` | Forward `Display` and `source()` to the inner error |
| `#[from]` | Generate `From`, and mark as `source()` |
| `#[source]` | Mark as `source()` only |
| field named `source` | Implicit `#[source]` |
| `#[error("...")]` on a struct | Same, for a struct error |

### anyhow

| Item | Meaning |
|---|---|
| `anyhow::Result<T>` | `Result<T, anyhow::Error>` |
| `anyhow!("msg {x}")` | Build an error |
| `bail!("msg")` | `return Err(anyhow!(...))` |
| `ensure!(cond, "msg")` | Bail unless `cond` |
| `.context("msg")` | Add a layer, eager message |
| `.with_context(\|\| msg)` | Add a layer, lazy message |
| `{}` on the error | Outermost context only |
| `{:?}` on the error | Message + `Caused by:` chain |
| `.chain()` | Iterator over every layer |
| `.root_cause()` | The innermost error |
| `.downcast_ref::<T>()` | Borrow the concrete type if present |
| `.downcast::<T>()` | Take it, or get the error back |
| `.is::<T>()` | Test the chain for a type |

## Common Mistakes

**`anyhow::Error` in a library's public signature.** Callers lose the ability
to branch. Define a `thiserror` enum; let the binary wrap it.

**Writing `{}` to log an `anyhow::Error`.** You lose the chain. Log `{:?}`.

**Two `#[from]` variants for the same error type.** The generated `From` impls
conflict. Use `#[source]` and `map_err` for one of them.

**Context without identifying data.** `"failed to read file"` adds nothing over
the inner error. `with_context(|| format!("reading {path}"))` does.

**`Rc` or a non-`Send` field in an error type.** `anyhow::Error` requires
`Send + Sync`; store owned data.

## Exercises

1. Rewrite your lesson 8.5 `CalcError` using `thiserror`, keeping the messages
   identical. Count the lines removed.
2. Add a variant wrapping `std::io::Error` with `#[from]` and confirm `source()`
   works without writing it.
3. Add a second variant that also wraps `std::io::Error`. Observe the conflict,
   then fix it with `#[source]` plus `map_err`.
4. Write a three-level `anyhow` call chain, each level adding `.with_context`.
   Print it with `{}` and `{:?}` and compare.
5. Define a `thiserror` struct `Retryable { after: u32 }`, return it through
   two `anyhow` context layers, and recover `after` with `downcast_ref`.

---

[← Course Index](../README.md)
