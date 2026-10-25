[← Course Index](../README.md)

# 8.3 — The ? Operator

## What You'll Learn

- `?` on `Result` and on `Option`, and exactly what it desugars to
- The `From` conversion `?` performs, and the error when it cannot
- `?` in `main`, in closures, and where it is rejected
- `map_err` before `?`, and `Box<dyn Error>` as the catch-all

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `if err != nil { return nil, err }` | `?` |
| TS `await` on a rejecting promise | `?` — but explicit, at every call |
| Python bare `raise` propagating up | `?`, with the type converted |

`?` is the Go error check collapsed into one character, plus an automatic type
conversion on the way out.

## ? on Result

```rust
fn double(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n: i32 = s.parse()?;
    Ok(n * 2)
}

fn main() {
    println!("{:?}", double("21"));
    println!("{:?}", double("x"));
}
```

Output:

```
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

## How ? desugars

`let n = expr?;` is equivalent to:

```rust
let n = match expr {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
};
```

Two things happen: `Ok` is unwrapped in place, and `Err` causes an **early
return** with `From::from(e)` applied. That `From` call is the whole reason `?`
is more than syntax sugar — it lets a function with one error type absorb calls
that fail with several.

On `Option`, the desugaring is the same shape with no conversion:

```rust
let v = match expr {
    Some(v) => v,
    None => return None,
};
```

## ? on Option

```rust
fn first_char_upper(s: &str) -> Option<char> {
    let c = s.chars().next()?;
    Some(c.to_ascii_uppercase())
}

fn second_word_len(s: &str) -> Option<usize> {
    let w = s.split_whitespace().nth(1)?;
    Some(w.len())
}

fn main() {
    println!("{:?}", first_char_upper("rust"));
    println!("{:?}", first_char_upper(""));
    println!("{:?}", second_word_len("hello world"));
    println!("{:?}", second_word_len("hello"));
}
```

Output:

```
Some('R')
None
Some(5)
None
```

## Chaining many ?

Each `?` is an independent exit point. The first failure wins.

```rust
fn sum_three(a: &str, b: &str, c: &str) -> Result<i32, std::num::ParseIntError> {
    Ok(a.parse::<i32>()? + b.parse::<i32>()? + c.parse::<i32>()?)
}

fn main() {
    println!("{:?}", sum_three("1", "2", "3"));
    println!("{:?}", sum_three("1", "x", "y"));
}
```

Output:

```
Ok(6)
Err(ParseIntError { kind: InvalidDigit })
```

`"y"` is never parsed — the function returned at `"x"`.

`?` also works mid-expression, on method chains:

```rust
fn trimmed_len(path: &str) -> std::io::Result<usize> {
    Ok(std::fs::read_to_string(path)?.trim().len())
}

fn main() {
    println!("{:?}", trimmed_len("Cargo.toml"));
    println!("{:?}", trimmed_len("nope").is_err());
}
```

Output:

```
Ok(113)
true
```

## ? requires the error types to convert

This is the error you will hit most often:

```rust
fn parse_and_read(s: &str) -> Result<i32, std::num::ParseIntError> {
    let text = std::fs::read_to_string(s)?;
    let n: i32 = text.trim().parse()?;
    Ok(n)
}
```

Does not compile (intentional):

```
error[E0277]: `?` couldn't convert the error to `ParseIntError`
 --> src/main.rs:2:42
  |
1 | fn parse_and_read(s: &str) -> Result<i32, std::num::ParseIntError> {
  |                               ------------------------------------ expected `ParseIntError` because of this
2 |     let text = std::fs::read_to_string(s)?;
  |                --------------------------^ the trait `From<std::io::Error>` is not implemented for `ParseIntError`
  |                |
  |                this can't be annotated with `?` because it has type `Result<_, std::io::Error>`
  |
  = note: the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait
```

Three fixes, in increasing order of effort: widen the return type to
`Box<dyn Error>`, insert `map_err`, or define your own error enum with `From`
impls (lesson 8.5).

## Converting with map_err before ?

`map_err` changes the error type so `?` has something to convert *from*.

```rust
fn load(path: &str) -> Result<i32, String> {
    let text = std::fs::read_to_string(path).map_err(|e| format!("read {path}: {e}"))?;
    let n: i32 = text.trim().parse().map_err(|e| format!("parse {path}: {e}"))?;
    Ok(n)
}

fn main() {
    println!("{:?}", load("nope.txt"));
    println!("{:?}", load("Cargo.toml"));
}
```

Output:

```
Err("read nope.txt: No such file or directory (os error 2)")
Err("parse Cargo.toml: invalid digit found in string")
```

`map_err` runs before `?`, so the `?` sees `Result<_, String>` and
`From<String> for String` is the identity. This is the right tool when you want
per-call-site context in the message.

## ? with Box<dyn Error>

`Box<dyn Error>` has a blanket `From<E>` for every `E: Error`, so `?` accepts
any standard error.

```rust
use std::error::Error;

fn load(path: &str) -> Result<i32, Box<dyn Error>> {
    let text = std::fs::read_to_string(path)?;
    let n: i32 = text.trim().parse()?;
    Ok(n)
}

fn main() {
    match load("nope.txt") {
        Ok(n) => println!("{n}"),
        Err(e) => println!("error: {e}"),
    }
    std::fs::write("num.txt", "  7 \n").unwrap();
    println!("{:?}", load("num.txt"));
    std::fs::write("num.txt", "seven").unwrap();
    match load("num.txt") {
        Ok(n) => println!("{n}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Output:

```
error: No such file or directory (os error 2)
Ok(7)
error: invalid digit found in string
```

It also accepts a bare `&str` or `String`, via `From<&str> for Box<dyn Error>`:

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

The cost: the caller can no longer `match` on which failure occurred. Fine in a
binary, wrong in a library (lesson 8.7).

## ? cannot cross Result/Option

```rust
fn f(s: &str) -> Result<char, String> {
    let c = s.chars().next()?;
    Ok(c)
}
```

Does not compile (intentional):

```
error[E0277]: the `?` operator can only be used on `Result`s, not `Option`s, in a function that returns `Result`
 --> src/main.rs:2:29
  |
1 | fn f(s: &str) -> Result<char, String> {
  | ------------------------------------- this function returns a `Result`
2 |     let c = s.chars().next()?;
  |                             ^ use `.ok_or(...)?` to provide an error compatible with `Result<char, String>`
```

Take the suggestion — `ok_or` / `ok_or_else` bridge `Option` to `Result`
(lesson 8.4). Going the other way, use `.ok()?`.

## ? cannot be used in a fn returning ()

```rust
fn show(s: &str) {
    let n: i32 = s.parse()?;
    println!("{n}");
}
```

Does not compile (intentional):

```
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:2:27
  |
1 | fn show(s: &str) {
  | ---------------- this function should return `Result` or `Option` to accept `?`
2 |     let n: i32 = s.parse()?;
  |                           ^ cannot use the `?` operator in a function that returns `()`
  |
help: consider adding return type
  |
1 ~ fn show(s: &str) -> Result<(), Box<dyn std::error::Error>> {
2 |     let n: i32 = s.parse()?;
3 |     println!("{n}");
4 +     Ok(())
  |
```

The compiler writes the fix for you, including the `Ok(())`.

## The closure gotcha

`?` returns from the **closure**, not the enclosing function. A closure whose
body is a bare expression has that expression's type, so `?` has nowhere to go:

```rust
fn main() -> Result<(), std::num::ParseIntError> {
    let nums: Vec<i32> = ["1", "2", "3"]
        .iter()
        .map(|s| s.parse::<i32>()?)
        .collect();
    println!("{nums:?}");
    Ok(())
}
```

Does not compile (intentional):

```
error[E0277]: the `?` operator can only be used in a closure that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:4:34
  |
4 |         .map(|s| s.parse::<i32>()?)
  |              ---                 ^ cannot use the `?` operator in a closure that returns `i32`
  |              |
  |              this function should return `Result` or `Option` to accept `?`
```

Two fixes. Drop the `?` and let `collect` gather into a `Result`:

```rust
fn main() -> Result<(), std::num::ParseIntError> {
    let nums: Vec<i32> = ["1", "2", "3"]
        .iter()
        .map(|s| s.parse::<i32>())
        .collect::<Result<Vec<_>, _>>()?;
    println!("{nums:?}");
    Ok(())
}
```

Output:

```
[1, 2, 3]
```

Or annotate the closure's return type so it is itself fallible:

```rust
fn main() {
    let parse = |s: &str| -> Result<i32, std::num::ParseIntError> {
        let n: i32 = s.parse()?;
        Ok(n * 10)
    };
    println!("{:?}", parse("4"));
    println!("{:?}", parse("q"));
}
```

Output:

```
Ok(40)
Err(ParseIntError { kind: InvalidDigit })
```

An explicit `-> Result<...>` on a closure forces braces around the body; that
is the shape to recognise.

## ? in main

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

Failing instead:

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let text = std::fs::read_to_string("/nonexistent/path.txt")?;
    println!("{text}");
    Ok(())
}
```

Output:

```
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

Exit code 1, and remember the runtime prints `Debug` (lesson 8.2).

## try blocks are not stable

A `try { ... }` block would let you scope `?` to a sub-expression without
writing a function or closure. It is still unstable:

```rust
fn main() {
    let r: Result<i32, std::num::ParseIntError> = try {
        let a: i32 = "1".parse()?;
        let b: i32 = "2".parse()?;
        a + b
    };
    println!("{r:?}");
}
```

Does not compile on stable (intentional):

```
error[E0658]: `try` expression is experimental
 --> src/main.rs:2:51
  |
2 |       let r: Result<i32, std::num::ParseIntError> = try {
  |  ___________________________________________________^
3 | |         let a: i32 = "1".parse()?;
4 | |         let b: i32 = "2".parse()?;
5 | |         a + b
6 | |     };
  | |_____^
```

Until it stabilises, use an immediately-invoked closure:

```rust
fn main() {
    let r = (|| -> Result<i32, std::num::ParseIntError> {
        let a: i32 = "1".parse()?;
        let b: i32 = "2".parse()?;
        Ok(a + b)
    })();
    println!("{r:?}");
}
```

Output:

```
Ok(3)
```

## Quick Reference

| Context | `?` allowed? |
|---|---|
| `fn f() -> Result<T, E>`, on a `Result<_, E2>` where `E: From<E2>` | Yes |
| `fn f() -> Result<T, E>`, on a `Result<_, E2>` without that `From` | No — E0277 |
| `fn f() -> Result<T, E>`, on an `Option` | No — use `.ok_or(...)?` |
| `fn f() -> Option<T>`, on an `Option` | Yes |
| `fn f() -> Option<T>`, on a `Result` | No — use `.ok()?` |
| `fn f()` returning `()` | No — E0277 |
| `fn main() -> Result<(), E>` where `E: Debug` | Yes |
| Closure with a bare-expression body | No — annotate `-> Result<...>` |
| `try { ... }` block | Not stable |

| Need | Write |
|---|---|
| Absorb any std error | `-> Result<T, Box<dyn Error>>` |
| Add context to one call | `.map_err(\|e\| format!("...: {e}"))?` |
| `Option` -> `Result` | `.ok_or(e)?` / `.ok_or_else(\|\| e)?` |
| `Result` -> `Option` | `.ok()?` |
| `?` inside `map` | `collect::<Result<Vec<_>, _>>()?` instead |
| Scope `?` to a block | `(\|\| -> Result<_, _> { ... })()` |

## Common Mistakes

**Using `?` inside a `map`/`for_each` closure.** It returns from the closure.
Collect into a `Result`, or annotate the closure's return type.

**Hitting E0277 and reaching for `unwrap`.** The conversion error means your
error type is too narrow, not that propagation was a bad idea. Widen it, or add
a `From` impl.

**Using `Box<dyn Error>` in a library's public signature.** Callers lose the
ability to match on the failure. Keep it for binaries (lesson 8.7).

**Forgetting `Ok(())` at the end.** `?` unwraps but does not wrap; the final
expression still has to be a `Result`.

**Assuming `?` adds context.** It propagates the error unchanged. Use `map_err`
or `anyhow`'s `.context()` (lesson 8.6) if you want to know where it came from.

## Exercises

1. Write `fn add_strs(a: &str, b: &str) -> Result<i32, std::num::ParseIntError>`
   using two `?`. Pass two bad inputs and confirm only the first is reported.
2. Write a function that reads a file and parses it as `i32`, returning
   `Result<i32, std::num::ParseIntError>`. Read the E0277 error, then fix it
   three ways: `Box<dyn Error>`, `map_err`, and a custom enum.
3. Write `fn initials(name: &str) -> Option<String>` using `?` on `Option`
   twice (first char of first word, first char of second word).
4. Take `["1", "2", "x"]` and produce `Result<Vec<i32>, _>` with `collect`,
   then produce `Vec<i32>` of only the successes with `filter_map`.
5. Write a closure that uses `?` internally and returns
   `Result<u16, std::num::ParseIntError>`, then call it with valid and invalid
   input.

---

[← Course Index](../README.md)
