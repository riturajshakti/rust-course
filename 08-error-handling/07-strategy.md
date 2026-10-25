[← Course Index](../README.md)

# 8.7 — Error Handling Strategy for Libraries vs Apps

## What You'll Learn

- The decision framework: typed errors down, `anyhow` up
- Validating at the boundary so inner code cannot fail
- Writing messages and variants a caller can act on
- Logging vs returning, `expect` with a reason, and HTTP status mapping

## Coming From Another Language

| Other | Rust |
|---|---|
| Go: every package defines sentinel `error` values | Each crate defines one `thiserror` enum |
| TS: throw and let the top-level handler sort it out | Return `Result`; the binary decides the exit |
| Python: catch broadly at the entry point | One `anyhow` boundary in `main` |

The architectural rule is one line: **libraries return concrete typed errors,
applications collapse them into `anyhow`.** Everything below is a consequence.

## The two halves of a program

```
   library / domain crates        binary / handlers
   ───────────────────────        ─────────────────
   thiserror enums                anyhow::Result
   #[non_exhaustive]              .context() per call
   no panics on input             prints {:?} at the top
   caller can match               nobody matches above here
```

A worked version of exactly that split:

```rust
mod parser {
    use thiserror::Error;

    #[derive(Debug, Error)]
    #[non_exhaustive]
    pub enum ParseError {
        #[error("empty input")]
        Empty,
        #[error("expected a digit at byte {offset}, found {found:?}")]
        NotADigit { offset: usize, found: char },
        #[error("value {value} exceeds the maximum of {max}")]
        TooLarge { value: u64, max: u64 },
    }

    pub fn parse_quantity(s: &str) -> Result<u32, ParseError> {
        if s.is_empty() {
            return Err(ParseError::Empty);
        }
        let mut acc: u64 = 0;
        for (offset, c) in s.char_indices() {
            let d = c.to_digit(10).ok_or(ParseError::NotADigit { offset, found: c })?;
            acc = acc * 10 + d as u64;
            if acc > u32::MAX as u64 {
                return Err(ParseError::TooLarge { value: acc, max: u32::MAX as u64 });
            }
        }
        Ok(acc as u32)
    }
}

use anyhow::{Context, Result};

fn load(raw: &str) -> Result<u32> {
    parser::parse_quantity(raw).with_context(|| format!("reading quantity {raw:?}"))
}

fn main() {
    for raw in ["42", "", "4a2", "99999999999"] {
        match load(raw) {
            Ok(n) => println!("{raw:?} -> {n}"),
            Err(e) => println!("{raw:?} -> {e:?}"),
        }
    }
    if let Err(e) = load("") {
        if let Some(parser::ParseError::Empty) = e.downcast_ref::<parser::ParseError>() {
            println!("caller branched on Empty");
        }
    }
}
```

Output:

```
"42" -> 42
"" -> reading quantity ""

Caused by:
    empty input
"4a2" -> reading quantity "4a2"

Caused by:
    expected a digit at byte 1, found 'a'
"99999999999" -> reading quantity "99999999999"

Caused by:
    value 9999999999 exceeds the maximum of 4294967295
caller branched on Empty
```

Nothing is lost by collapsing into `anyhow`: the typed variant is still in
there if someone needs it.

## Never panic in a library on caller input

Any value that reaches your function from outside is input, and bad input is
not a bug in your code. The function that panics is unusable in a server, in a
test, and in any caller that wants to retry.

```rust
pub fn truncate_bad(s: &str, n: usize) -> &str {
    &s[..n]
}

pub fn truncate_good(s: &str, n: usize) -> Option<&str> {
    s.get(..n)
}

fn main() {
    println!("{:?}", truncate_good("héllo", 3));
    println!("{:?}", truncate_good("héllo", 20));
    println!("{:?}", truncate_good("héllo", 2));
    println!("{:?}", truncate_bad("héllo", 2));
}
```

Output:

```
Some("hé")
None
None

thread 'main' (13845699) panicked at src/main.rs:2:7:
end byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)
```

`get` returns `None` for both the out-of-range case and the char-boundary case
(byte 2 lands inside the two-byte `é`). Direct slicing panics on both.

Documented exceptions where a library may panic: a broken internal invariant,
an out-of-range index on an infallible indexing API (`Vec`'s own `Index` impl is
the standard library doing exactly this, and it is documented as panicking), and
`todo!()` in unreleased code. Everything else returns `Result` or `Option`.

## Validate at the boundary, then make failure impossible

Push validation to the edge and encode the result in a type. Inner functions
then take the validated type and cannot fail at all.

```rust
use thiserror::Error;

#[derive(Debug, Error, PartialEq)]
#[error("{0:?} is not a valid email: {1}")]
pub struct InvalidEmail(String, &'static str);

pub struct Email(String);

impl Email {
    pub fn parse(raw: &str) -> Result<Self, InvalidEmail> {
        let raw = raw.trim();
        if raw.is_empty() {
            return Err(InvalidEmail(raw.into(), "empty"));
        }
        match raw.split_once('@') {
            None => Err(InvalidEmail(raw.into(), "no @")),
            Some((_, domain)) if !domain.contains('.') => {
                Err(InvalidEmail(raw.into(), "domain has no dot"))
            }
            Some((local, _)) if local.is_empty() => Err(InvalidEmail(raw.into(), "no local part")),
            Some(_) => Ok(Email(raw.to_string())),
        }
    }

    pub fn as_str(&self) -> &str { &self.0 }
}

fn send(to: &Email) {
    println!("sending to {}", to.as_str());
}

fn main() {
    for raw in [" a@b.com ", "", "nope", "a@b", "@b.com"] {
        match Email::parse(raw) {
            Ok(e) => send(&e),
            Err(e) => println!("{e}"),
        }
    }
}
```

Output:

```
sending to a@b.com
"" is not a valid email: empty
"nope" is not a valid email: no @
"a@b" is not a valid email: domain has no dot
"@b.com" is not a valid email: no local part
```

`send` takes `&Email`, not `&str`, so it returns nothing and cannot fail. The
type is the proof that validation happened. One fallible constructor replaces a
`Result` on every function that touches an address — this is the single
highest-leverage move in Rust error handling.

Keep the field private; a `pub` field would let callers bypass `parse`.

## Error messages that help the caller

Lowercase, no trailing period, no `error:` prefix, and name the input. The
caller supplies the framing.

| Bad | Why | Good |
|---|---|---|
| `"Error: invalid input!"` | Says nothing; duplicates the framing | `"expected a digit at byte 3, found 'x'"` |
| `"failed"` | Unactionable | `"connection to 10.0.0.4:5432 refused"` |
| `"Could not parse the file."` | Which file? what was wrong? | `"line 7: expected `key=value`"` |
| `"invalid id: 42"` + no variant | Caller must parse the string | variant `NotFound { id: 42 }` |

Two more rules:

- **Include the offending value**, bounded. `found: 'x'` and `line: 7` are
  cheap; a 4 MB request body is not — truncate or omit.
- **Never put secrets in a message.** Error strings end up in logs, bug
  trackers, and HTTP bodies. `"auth failed for user alice"`, not
  `"auth failed: password hunter2 != ..."`.

## Variants a caller can branch on

Design the variant list around the decisions the caller has to make, not the
places your code happens to fail. Retryability is the usual one:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum FetchError {
    #[error("timed out after {0}ms")]
    Timeout(u64),
    #[error("server returned {0}")]
    Status(u16),
    #[error("malformed response body")]
    Body,
}

impl FetchError {
    fn is_retryable(&self) -> bool {
        match self {
            FetchError::Timeout(_) => true,
            FetchError::Status(code) => *code >= 500 || *code == 429,
            FetchError::Body => false,
        }
    }
}

fn main() {
    let errs = [
        FetchError::Timeout(5000),
        FetchError::Status(503),
        FetchError::Status(404),
        FetchError::Status(429),
        FetchError::Body,
    ];
    for e in &errs {
        println!("{:<28} retry={}", e.to_string(), e.is_retryable());
    }
}
```

Output:

```
timed out after 5000ms       retry=true
server returned 503          retry=true
server returned 404          retry=false
server returned 429          retry=true
malformed response body      retry=false
```

Shipping the `is_retryable` method alongside the enum is better than making
every caller re-derive the rule.

## HTTP status mapping

The same idea at the web boundary: one `match` from your domain error to a
status code. Lesson 17.6 wires this into an Axum `IntoResponse`; the shape is
already decidable here.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("{0} not found")]
    NotFound(&'static str),
    #[error("invalid input: {0}")]
    Validation(String),
    #[error("not authorized")]
    Unauthorized,
    #[error("internal failure")]
    Internal(#[from] std::io::Error),
}

fn status(e: &AppError) -> u16 {
    match e {
        AppError::NotFound(_) => 404,
        AppError::Validation(_) => 422,
        AppError::Unauthorized => 401,
        AppError::Internal(_) => 500,
    }
}

fn main() {
    let errs = [
        AppError::NotFound("user"),
        AppError::Validation("email missing @".into()),
        AppError::Unauthorized,
        AppError::Internal(std::io::Error::other("disk full")),
    ];
    for e in &errs {
        println!("{} {}", status(e), e);
    }
}
```

Output:

```
404 user not found
422 invalid input: email missing @
401 not authorized
500 internal failure
```

Rule for the 5xx arm: **log the detail, return a generic body.** `"disk full"`
goes to your logs; the client gets `"internal failure"`. 4xx messages are safe
to return verbatim, because the client caused them and needs them to fix the
request.

## Logging vs returning

Do exactly one, at exactly one place.

| Situation | Do |
|---|---|
| A function can fail and the caller might care | Return `Err`. Do not log. |
| You are the top-level handler | Log once (`{:?}` for the chain), then convert to a response or exit code |
| You recovered and continued | Log at `warn`, return `Ok` |
| You are ignoring a failure deliberately | Log at `debug`, `let _ = ...` |
| Background task with no caller | Log; there is nobody to return to |

Logging *and* returning produces the same failure in your logs once per stack
frame. If you want the trail, use `anyhow`'s `.context()` — that is a log line
per layer, assembled into one message.

## expect with a reason

`expect` in application code is fine when the condition is genuinely
impossible, and the string is a claim about *why*, not a restatement of the
failure.

```rust
fn main() {
    let port: u16 = "8080".parse().expect("literal 8080 fits in u16");
    println!("{port}");

    let home = std::env::var("HOME").expect("HOME is always set on unix");
    println!("{}", !home.is_empty());

    let missing = std::env::var("DEFINITELY_NOT_SET_12345");
    println!("{missing:?}");
}
```

Output:

```
8080
true
Err(NotPresent)
```

| Acceptable `expect` | Not acceptable |
|---|---|
| Parsing a literal in your own source | Parsing user input |
| A mutex you know is not poisoned | A network call |
| An index you just bounds-checked | An index from a request |
| Anything in a test | Anything in a library |

Write the invariant: `expect("HOME is always set on unix")`. Not
`expect("failed to get HOME")` — the panic message already says it failed.

## Situation -> approach

| Situation | Approach |
|---|---|
| Library function, caller-supplied input may be bad | `Result<T, YourError>` with a `thiserror` enum |
| Library function, one failure mode | `Result<T, YourStructError>` |
| Library function, failure needs no explanation | `Option<T>` |
| Library internal invariant broken | `panic!` / `unreachable!`, documented |
| Binary, any step of a pipeline | `anyhow::Result` + `.with_context()` |
| Binary entry point | `fn main() -> anyhow::Result<()>` |
| CLI needing a custom exit code | `fn main()` calling `run() -> Result<..>`, then `process::exit` |
| HTTP handler | Domain enum + `match` to a status (17.6) |
| Input crossing a trust boundary | Validate once into a newtype |
| Startup config missing | `Result` from `main`, or `expect` with the invariant |
| Retry logic needed | Variants plus an `is_retryable()` method |
| Test | `unwrap` / `expect` freely, or `-> Result<()>` and `?` |
| Prototype | `anyhow` everywhere; type it later |
| FFI boundary | `catch_unwind` so a panic does not unwind into C |

## Migration path

You do not have to design the final error type on day one, and the refactor is
mechanical:

1. `anyhow::Result` everywhere; get it working.
2. When a caller needs to branch on a specific failure, promote that one case
   to a `thiserror` variant in the crate that produces it.
3. Flip that crate's public signatures to the concrete type;
   `anyhow::Result` above still compiles unchanged, because
   `anyhow::Error: From<E>` for any `E: Error + Send + Sync`.

Step 3 is the payoff: the `.context()` calls and the `?` in the binary need no
edits. Going the other way — starting with a hand-rolled enum per function —
costs much more.

## Quick Reference

| Layer | Error type | Context strategy |
|---|---|---|
| Domain / library crate | `thiserror` enum, `#[non_exhaustive]` | Variant data; `source()` for causes |
| Adapter (db, http client) | `thiserror` enum with `#[from]` | Wrap the dependency's error |
| Application service | `anyhow::Result` | `.with_context()` per call |
| HTTP handler | Domain enum -> status (17.6) | Log 5xx detail, return generic |
| `main` | `anyhow::Result<()>` | Prints `{:?}`, exit 1 |

| Anti-pattern | Instead |
|---|---|
| `anyhow::Error` in a library signature | A `thiserror` enum |
| `panic!` on bad caller input | `Result` |
| `unwrap()` with no comment | `expect("<the invariant>")` |
| Log and return the same error | Return; log once at the top |
| `Failed(String)` catch-all variant | One variant per failure mode |
| Secrets or unbounded data in messages | Identify the input, bound the size |
| `Result<T, ()>` | `Option<T>` |

## Common Mistakes

**Exposing `anyhow::Error` from a library.** Every caller is forced to
downcast. Use `thiserror` for anything public.

**Panicking on input in a library.** It takes the recovery decision away from
every downstream user. Slicing, `unwrap`, and direct indexing are the usual
culprits.

**Logging at every level.** One failure, ten log lines. Return the error and
log once at the boundary, or use `.context()`.

**Returning a 500 body containing the internal error.** 4xx messages are for
the client; 5xx detail belongs in your logs only.

**Designing variants around your code's structure.** Design them around the
decisions the caller must make — retry, re-prompt, give up.

## Exercises

1. Take a function returning `Result<T, Box<dyn Error>>` and convert it to a
   `thiserror` enum with three variants. Note what the caller can now do.
2. Write a `Username` newtype with a fallible `parse` (3..=20 chars,
   alphanumeric or `_`) and a function taking `&Username` that cannot fail.
3. Add an `is_retryable(&self) -> bool` to an error enum of at least four
   variants and print a table of variant vs retryability.
4. Write `fn status(&self) -> u16` mapping your enum to HTTP codes, and make
   the 500 arm print the detail to stderr while returning a generic message.
5. Build a two-module program: a `store` module with a `thiserror` enum and a
   `main` using `anyhow` with two `.context()` layers. Then recover the
   concrete variant in `main` with `downcast_ref`.

---

[← Course Index](../README.md)
