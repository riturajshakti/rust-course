[← 8.3 The ? Operator](03-question-operator.md) · [Course Index](../README.md) · [8.5 Custom Error Types →](05-custom-errors.md)

# 8.4 — Combinators on Option and Result

## What You'll Learn

- Every `Result` combinator, with what it does to `Ok` and to `Err`
- Every `Option` combinator, and the `Option` <-> `Result` bridges
- Collecting an iterator of `Result`s both ways, and partitioning
- When a chain reads better than `match`, and when it reads worse

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `.then()` / `.catch()` | `.map()` / `.map_err()` |
| JS `?.` optional chaining | `.and_then()` on `Option`, or `?` |
| JS `??` | `.unwrap_or()` |
| Python `dict.get(k, default)` | `.get(k).unwrap_or(default)` |

`map` transforms the success side and leaves the failure side alone. Every
combinator below is some variation on which side it touches and whether the
transform itself can fail.

## Result: map, map_err, map_or

```rust
let ok: Result<i32, String> = Ok(4);
let er: Result<i32, String> = Err("bad".into());

println!("{:?}", ok.clone().map(|n| n * 2));
println!("{:?}", er.clone().map(|n| n * 2));
println!("{:?}", ok.clone().map_err(|e| e.len()));
println!("{:?}", er.clone().map_err(|e| e.len()));
println!("{:?}", ok.clone().map(|n| n.to_string()));
println!("{:?}", ok.clone().map_or(0, |n| n * 10));
println!("{:?}", er.clone().map_or(0, |n| n * 10));
println!("{:?}", er.clone().map_or_else(|e| e.len() as i32, |n| n * 10));
```

Output:

```
Ok(8)
Err("bad")
Ok(4)
Err(3)
Ok("4")
40
0
3
```

`map` and `map_err` keep you inside `Result`. `map_or` and `map_or_else`
**leave** `Result` and hand you a plain value — they are `map` followed by
`unwrap_or`. Note the argument order in `map_or_else(err_fn, ok_fn)`: the error
handler comes first.

## Result: and_then, or_else, and, or

`and_then` is `map` for a transform that can itself fail — it flattens, so you
never build `Result<Result<T, E>, E>`.

```rust
fn half(n: i32) -> Result<i32, String> {
    if n % 2 == 0 { Ok(n / 2) } else { Err(format!("{n} is odd")) }
}

let ok: Result<i32, String> = Ok(8);
let er: Result<i32, String> = Err("bad".into());

println!("{:?}", ok.clone().and_then(half));
println!("{:?}", ok.clone().and_then(half).and_then(half));
println!("{:?}", ok.clone().and_then(half).and_then(half).and_then(half));
println!("{:?}", er.clone().and_then(half));
println!("{:?}", er.clone().or_else(|_| Ok::<i32, String>(0)));
println!("{:?}", ok.clone().or_else(|_| Ok::<i32, String>(0)));
println!("{:?}", ok.clone().and(Ok::<&str, String>("next")));
println!("{:?}", er.clone().and(Ok::<&str, String>("next")));
println!("{:?}", er.clone().or(Ok::<i32, String>(99)));
println!("{:?}", ok.clone().or(Ok::<i32, String>(99)));
```

Output:

```
Ok(4)
Ok(2)
Ok(1)
Err("bad")
Ok(0)
Ok(8)
Ok("next")
Err("bad")
Ok(99)
Ok(8)
```

`and`/`or` take a *value*; `and_then`/`or_else` take a *closure* and are lazy.
Prefer the closure versions unless the value is already computed.

## Result: predicates, ok, err, as_ref, iter

```rust
let ok: Result<i32, String> = Ok(4);
let er: Result<i32, String> = Err("bad".into());

println!("{:?}", ok.iter().collect::<Vec<_>>());
println!("{:?}", er.iter().collect::<Vec<_>>());
println!("{}", ok.iter().count());
println!("{:?}", ok.clone().into_iter().sum::<i32>());
let mut m: Result<i32, String> = Ok(1);
if let Some(v) = m.iter_mut().next() { *v = 50; }
println!("{m:?}");
println!("{:?}", ok.as_ref());
println!("{:?}", er.as_ref());
let s: Result<String, String> = Ok("hi".into());
println!("{:?}", s.as_deref());
```

Output:

```
[4]
[]
1
4
Ok(50)
Ok(4)
Err("bad")
Ok("hi")
```

A `Result` is an iterator of 0 or 1 items. That is how `flat_map` over results
drops the errors, and how `.extend(result)` works.

`is_ok_and` and `is_err_and` test the contained value:

```rust
let ok: Result<i32, String> = Ok(4);
let er: Result<i32, String> = Err("timeout".into());
println!("{}", ok.is_ok_and(|n| n > 3));
println!("{}", er.is_err_and(|e| e.contains("timeout")));
```

Output:

```
true
true
```

## Option -> Result bridges

```rust
let some = Some(5);
let none: Option<i32> = None;

println!("{:?}", some.ok_or("missing"));
println!("{:?}", none.ok_or("missing"));
println!("{:?}", none.ok_or_else(|| format!("missing at {}", 1 + 1)));

let ok: Result<i32, String> = Ok(5);
let er: Result<i32, String> = Err("bad".into());
println!("{:?} {:?}", ok.clone().ok(), er.clone().ok());
println!("{:?} {:?}", ok.clone().err(), er.clone().err());

let a: Option<Result<i32, String>> = Some(Ok(1));
let b: Option<Result<i32, String>> = Some(Err("e".into()));
let c: Option<Result<i32, String>> = None;
println!("{:?}", a.transpose());
println!("{:?}", b.transpose());
println!("{:?}", c.transpose());
let d: Result<Option<i32>, String> = Ok(Some(7));
println!("{:?}", d.transpose());
```

Output:

```
Ok(5)
Err("missing")
Err("missing at 2")
Some(5) None
None Some("bad")
Ok(Some(1))
Err("e")
Ok(None)
Some(Ok(7))
```

`transpose` swaps the two layers: `Option<Result<T, E>>` <-> `Result<Option<T>, E>`.
`None` becomes `Ok(None)` — "nothing to do, and that is fine". It is exactly
what you want after `.map(...)` over an optional field that needs parsing.

Use `ok_or_else` whenever the error allocates. `ok_or(format!(...))` builds the
string even on the `Some` path.

## Option combinators

```rust
let s = Some(4);
let n: Option<i32> = None;

println!("{:?} {:?}", s.map(|x| x * 2), n.map(|x| x * 2));
println!("{:?}", s.and_then(|x| if x > 2 { Some(x) } else { None }));
println!("{:?} {:?}", s.filter(|x| x % 2 == 0), s.filter(|x| x % 2 == 1));
println!("{:?} {:?}", s.or(Some(9)), n.or(Some(9)));
println!("{:?}", n.or_else(|| Some(10)));
println!("{} {}", s.unwrap_or(0), n.unwrap_or(0));
println!("{} {}", s.unwrap_or_default(), n.unwrap_or_default());
println!("{}", n.unwrap_or_else(|| 7 * 7));
println!("{:?} {:?}", s.zip(Some("x")), s.zip(None::<&str>));
println!("{:?} {:?}", Some(Some(3)).flatten(), Some(None::<i32>).flatten());
println!("{:?} {:?}", s.xor(None), s.xor(Some(1)));
println!("{}", s.is_some_and(|x| x > 3));
println!("{:?}", s.map_or("none".to_string(), |x| x.to_string()));
println!("{:?}", n.map_or("none".to_string(), |x| x.to_string()));
```

Output:

```
Some(8) None
Some(4)
Some(4) None
Some(4) Some(9)
Some(10)
4 0
4 0
49
Some((4, "x")) None
Some(3) None
Some(4) None
true
"4"
"none"
```

`filter` is the one with no `Result` counterpart — it demotes `Some` to `None`
when a predicate fails. `xor` yields `Some` only when exactly one side is
`Some`.

## Chaining through nested Options

```rust
#[derive(Debug)]
struct Config { server: Option<Server> }
#[derive(Debug)]
struct Server { tls: Option<Tls> }
#[derive(Debug)]
struct Tls { cert: String }

fn cert(c: &Config) -> Option<&str> {
    c.server.as_ref()?.tls.as_ref().map(|t| t.cert.as_str())
}

fn cert_combinator(c: &Config) -> Option<&str> {
    c.server
        .as_ref()
        .and_then(|s| s.tls.as_ref())
        .map(|t| t.cert.as_str())
}

fn main() {
    let full = Config { server: Some(Server { tls: Some(Tls { cert: "a.pem".into() }) }) };
    let no_tls = Config { server: Some(Server { tls: None }) };
    let empty = Config { server: None };

    for c in [&full, &no_tls, &empty] {
        println!("{:?} {:?}", cert(c), cert_combinator(c));
    }
}
```

Output:

```
Some("a.pem") Some("a.pem")
None None
None None
```

`as_ref()` on `&Option<T>` gives `Option<&T>`, which is what lets you chain
without moving out of a borrowed struct. Both spellings are idiomatic; `?` wins
when the chain is long.

## Collecting an iterator of Results

`Result` implements `FromIterator`, so `collect` can target either shape. This
is the single most useful thing in the lesson.

```rust
let good = ["1", "2", "3"];
let bad = ["1", "x", "3", "y"];

let a: Result<Vec<i32>, _> = good.iter().map(|s| s.parse::<i32>()).collect();
println!("{a:?}");

let b: Result<Vec<i32>, _> = bad.iter().map(|s| s.parse::<i32>()).collect();
println!("{b:?}");

let c: Vec<Result<i32, _>> = bad.iter().map(|s| s.parse::<i32>()).collect();
println!("{c:?}");

let d: Result<i32, _> = good.iter().map(|s| s.parse::<i32>()).sum();
println!("{d:?}");

let e: Option<Vec<i32>> = ["1", "2"].iter().map(|s| s.parse::<i32>().ok()).collect();
println!("{e:?}");
let f: Option<Vec<i32>> = ["1", "z"].iter().map(|s| s.parse::<i32>().ok()).collect();
println!("{f:?}");
```

Output:

```
Ok([1, 2, 3])
Err(ParseIntError { kind: InvalidDigit })
[Ok(1), Err(ParseIntError { kind: InvalidDigit }), Ok(3), Err(ParseIntError { kind: InvalidDigit })]
Ok(6)
Some([1, 2])
None
```

The direction is chosen entirely by the type annotation:

- `Result<Vec<T>, E>` — **short-circuits**. Iteration stops at the first `Err`
  and that error is returned. `"y"` was never parsed.
- `Vec<Result<T, E>>` — collects everything, keeps every error.

`sum`, `product`, and `Option<Vec<_>>` work the same way. Pick the first when
one bad record invalidates the batch; pick the second when you need to report
all the bad rows.

## Partitioning successes from failures

```rust
let input = ["1", "x", "3", "y", "5"];

let (ok, err): (Vec<_>, Vec<_>) = input
    .iter()
    .map(|s| s.parse::<i32>())
    .partition(|r| r.is_ok());
println!("{}", ok.len());
println!("{}", err.len());

let values: Vec<i32> = ok.into_iter().map(Result::unwrap).collect();
println!("{values:?}");
```

Output:

```
3
2
[1, 3, 5]
```

`partition` leaves both halves as `Vec<Result<...>>`, so you still have to
unwrap. The `unwrap` is safe here but the compiler cannot see that. A `fold`
gives you unwrapped halves directly and avoids the issue:

```rust
let input = ["1", "x", "3", "y", "5"];

let (vals, errs): (Vec<i32>, Vec<String>) = input.iter().fold(
    (Vec::new(), Vec::new()),
    |(mut v, mut e), s| {
        match s.parse::<i32>() {
            Ok(n) => v.push(n),
            Err(err) => e.push(format!("{s}: {err}")),
        }
        (v, e)
    },
);
println!("{vals:?} {}", errs.len());
```

Output:

```
[1, 3, 5] 2
```

## filter_map with ok()

To keep the good values and silently drop the rest:

```rust
let input = ["1", "x", "3", "y", "5"];
let skipped: Vec<i32> = input.iter().filter_map(|s| s.parse::<i32>().ok()).collect();
println!("{skipped:?}");
```

Output:

```
[1, 3, 5]
```

Or keep only the failures, for a validation report:

```rust
let input = ["1", "x", "3", "y", "5"];
let msgs: Vec<String> = input
    .iter()
    .filter_map(|s| s.parse::<i32>().err().map(|e| format!("{s}: {e}")))
    .collect();
for m in &msgs { println!("{m}"); }
```

Output:

```
x: invalid digit found in string
y: invalid digit found in string
```

## When a chain beats match

A chain wins when every step has one obvious failure and the error type is
uniform:

```rust
use std::collections::HashMap;

fn lookup_port(cfg: &HashMap<String, String>, key: &str) -> Result<u16, String> {
    cfg.get(key)
        .ok_or_else(|| format!("key `{key}` not set"))
        .and_then(|v| v.parse::<u16>().map_err(|e| format!("`{key}` = `{v}`: {e}")))
        .and_then(|p| if p == 0 { Err("port 0 invalid".into()) } else { Ok(p) })
}

fn main() {
    let mut cfg = HashMap::new();
    cfg.insert("port".to_string(), "8080".to_string());
    cfg.insert("admin_port".to_string(), "zero".to_string());
    cfg.insert("dead".to_string(), "0".to_string());

    println!("{:?}", lookup_port(&cfg, "port"));
    println!("{:?}", lookup_port(&cfg, "admin_port"));
    println!("{:?}", lookup_port(&cfg, "dead"));
    println!("{:?}", lookup_port(&cfg, "missing"));
}
```

Output:

```
Ok(8080)
Err("`admin_port` = `zero`: invalid digit found in string")
Err("port 0 invalid")
Err("key `missing` not set")
```

## When a chain is worse

The same logic with `?` and `match`, which handles three distinct error
*variants* instead of three strings:

```rust
use std::collections::HashMap;

#[derive(Debug)]
enum PortError {
    NotSet(String),
    NotANumber { key: String, value: String },
    Zero,
}

fn lookup_port(cfg: &HashMap<String, String>, key: &str) -> Result<u16, PortError> {
    let raw = match cfg.get(key) {
        Some(v) => v,
        None => return Err(PortError::NotSet(key.to_string())),
    };
    let port: u16 = match raw.parse() {
        Ok(p) => p,
        Err(_) => {
            return Err(PortError::NotANumber {
                key: key.to_string(),
                value: raw.clone(),
            })
        }
    };
    if port == 0 {
        return Err(PortError::Zero);
    }
    Ok(port)
}

fn main() {
    let mut cfg = HashMap::new();
    cfg.insert("admin_port".to_string(), "zero".to_string());
    println!("{:?}", lookup_port(&cfg, "admin_port"));
    println!("{:?}", lookup_port(&cfg, "missing"));
}
```

Output:

```
Err(NotANumber { key: "admin_port", value: "zero" })
Err(NotSet("missing"))
```

(dead-code warnings about the never-read fields are expected; only `Debug`
reads them.)

Rules of thumb:

- Up to three steps, one error type, no intermediate names needed -> chain.
- Different error variants per step, or you need a `let` binding mid-way, or
  the closures start needing their own `match` -> `?` plus early returns.
- Never nest a closure containing a `match` inside `and_then`. At that point
  the chain is costing you more than it saves.

## Quick Reference

### Result<T, E>

| Method | Result | Notes |
|---|---|---|
| `map(f)` | `Result<U, E>` | Transforms `Ok` |
| `map_err(f)` | `Result<T, F>` | Transforms `Err` |
| `map_or(d, f)` | `U` | Leaves `Result`; eager default |
| `map_or_else(ef, of)` | `U` | Error handler is the **first** arg |
| `and_then(f)` | `Result<U, E>` | `f` returns `Result`; flattens |
| `or_else(f)` | `Result<T, F>` | Recover from `Err`, lazily |
| `and(res)` | `Result<U, E>` | Eager `and_then` |
| `or(res)` | `Result<T, F>` | Eager `or_else` |
| `unwrap_or(v)` | `T` | Eager default |
| `unwrap_or_else(f)` | `T` | `f(E) -> T` |
| `unwrap_or_default()` | `T` | Needs `T: Default` |
| `ok()` | `Option<T>` | Discards the error |
| `err()` | `Option<E>` | Discards the value |
| `as_ref()` | `Result<&T, &E>` | |
| `as_deref()` | `Result<&T::Target, &E>` | `Ok(String)` -> `Ok(&str)` |
| `iter()` | 0 or 1 item | |
| `is_ok_and(f)` / `is_err_and(f)` | `bool` | |
| `flatten()` | `Result<T, E>` | From `Result<Result<T,E>,E>` |
| `transpose()` | `Option<Result<T,E>>` | From `Result<Option<T>,E>` |

### Option<T> -> Result<T, E>

| Method | Result |
|---|---|
| `ok_or(e)` | `Result<T, E>`, eager error |
| `ok_or_else(f)` | `Result<T, E>`, lazy error |
| `transpose()` | `Result<Option<T>, E>` from `Option<Result<T, E>>` |

### Option<T>

| Method | Result |
|---|---|
| `map(f)` | `Option<U>` |
| `and_then(f)` | `Option<U>`, flattens |
| `filter(p)` | `Option<T>`, `Some` -> `None` when `p` fails |
| `or(opt)` / `or_else(f)` | `Option<T>` |
| `xor(opt)` | `Some` only if exactly one is `Some` |
| `zip(opt)` | `Option<(T, U)>` |
| `flatten()` | `Option<T>` from `Option<Option<T>>` |
| `unwrap_or(v)` / `unwrap_or_else(f)` / `unwrap_or_default()` | `T` |
| `map_or(d, f)` / `map_or_else(df, f)` | `U` |
| `is_some_and(p)` / `is_none_or(p)` | `bool` |
| `as_ref()` / `as_mut()` / `as_deref()` | Borrowed view |
| `take()` / `replace(v)` / `get_or_insert(v)` | Needs `&mut` |

### Iterators of Result

| Target annotation | Behaviour |
|---|---|
| `Result<Vec<T>, E>` | Short-circuits at the first `Err` |
| `Vec<Result<T, E>>` | Collects every outcome |
| `Result<i32, E>` from `.sum()` | Short-circuits |
| `Option<Vec<T>>` | Short-circuits at the first `None` |
| `.filter_map(\|r\| r.ok())` | Drops failures silently |
| `.filter_map(\|r\| r.err())` | Keeps only failures |
| `.partition(\|r\| r.is_ok())` | Two `Vec<Result<..>>` |

## Common Mistakes

**Using `map` where the closure returns a `Result`.** You get
`Result<Result<T, E>, E>`. Use `and_then`.

**`ok_or(format!(...))` on a hot path.** The string is built even on `Some`.
Use `ok_or_else`.

**Forgetting `collect` short-circuits.** `Result<Vec<_>, E>` stops at the first
error, so you cannot report all bad rows that way. Collect `Vec<Result<_, _>>`
or `fold`.

**Mixing up `map_or_else` argument order.** The error handler is first:
`map_or_else(|e| ..., |v| ...)`.

**Building a six-link chain with closures containing `match`.** Rewrite with
`?`. Combinators are for short, uniform steps.

## Exercises

1. Given `Result<i32, String>`, produce `Result<String, usize>` in one chain
   (map the value to a string, the error to its length).
2. Write `fn pick(v: &[i32], i: usize) -> Result<i32, String>` using `get`,
   `ok_or_else`, and `copied`.
3. Parse `["10", "20", "x", "40"]` three ways: short-circuiting to
   `Result<Vec<i32>, _>`, collecting all outcomes, and dropping failures.
4. Write a validator that returns `(Vec<i32>, Vec<String>)` — parsed values and
   one message per bad input — using `fold`.
5. Rewrite this chain as `?` plus early returns and decide which you prefer:
   `s.split_once('=').ok_or("no =").and_then(|(k, v)| v.parse::<i32>().map_err(|_| "bad int").map(|n| (k, n)))`

---

[← 8.3 The ? Operator](03-question-operator.md) · [Course Index](../README.md) · [8.5 Custom Error Types →](05-custom-errors.md)
