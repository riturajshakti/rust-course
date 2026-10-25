[← 5.6 Deriving Common Traits](06-derive.md) · [Course Index](../README.md) · [6.1 String vs &str →](../06-strings/01-string-vs-str.md)

# 5.7 — Builder Pattern and Default

## What You'll Learn

- `Default`: derived, manual, and `#[default]` on an enum variant
- `..Default::default()` in a struct literal
- The builder pattern both ways: owned `self` and `&mut self`
- `build()` returning `Result` for validation, plus a typestate sketch

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `function f({port = 8080} = {})` | no default args — use `Default` or a builder |
| Python `def f(port=8080)` | same; Rust has no default parameters at all |
| Go zero value, free | `#[derive(Default)]`, explicitly opted into |

Rust has **no default function arguments, no optional parameters, and no
overloading.** Those three absences are why `Default` and builders are
everywhere in Rust APIs.

## derive(Default)

Every field gets its type's default: `0`, `""`, `false`, `[]`, `None`.

```rust
#[derive(Debug, Default, Clone, PartialEq)]
struct Config {
    host: String,
    port: u16,
    tls: bool,
    retries: u8,
    tags: Vec<String>,
    timeout: Option<u32>,
}

fn main() {
    println!("{:?}", Config::default());
    println!("{:#?}", Config::default());
}
```

Output:

```
Config { host: "", port: 0, tls: false, retries: 0, tags: [], timeout: None }
Config {
    host: "",
    port: 0,
    tls: false,
    retries: 0,
    tags: [],
    timeout: None,
}
```

Every field must itself implement `Default` — same hard rule as every other
derive (lesson 5.6).

## What the standard defaults are

```rust
println!("{:?} {:?} {:?} {:?}",
    i32::default(), String::default(), bool::default(), Vec::<u8>::default());
println!("{:?} {:?} {:?}",
    Option::<u8>::default(), f64::default(), char::default());
println!("{:?}", <(i32, bool, String)>::default());
println!("{:?}", <[u8; 3]>::default());
```

Output:

```
0 "" false []
None 0.0 '\0'
(0, false, "")
[0, 0, 0]
```

Tuples and arrays are default-able element-wise. References are not — there is
no default `&T`.

## Two call forms

```rust
let a = Config::default();
let mut d: Config = Default::default();
d.port = 1;
println!("{} {}", a.port, d.port);
```

Output:

```
0 1
```

`Config::default()` when you want to be explicit, `Default::default()` when the
type is already pinned by context — which is what makes struct update syntax
read well.

## ..Default::default() in a literal

Set the fields you care about; `..` fills the rest.

```rust
let c = Config { port: 443, tls: true, ..Default::default() };
println!("{:?}", c);

let c2 = Config { host: "example.com".into(), ..Config::default() };
println!("{:?}", c2);
```

Output:

```
Config { host: "", port: 443, tls: true, retries: 0, tags: [], timeout: None }
Config { host: "example.com", port: 0, tls: false, retries: 0, tags: [], timeout: None }
```

This is the closest Rust gets to named optional arguments, and for a struct you
own it is usually enough — no builder needed. Note it does construct a whole
throwaway `Config` and then move the unset fields out of it.

## Manual impl Default

Derive gives zeros. When the sensible default is not zero, write it.

```rust
#[derive(Debug)]
struct Server { host: String, port: u16, workers: usize }

impl Default for Server {
    fn default() -> Self {
        Server { host: "127.0.0.1".into(), port: 8080, workers: 4 }
    }
}

println!("{:?}", Server::default());
```

Output:

```
Server { host: "127.0.0.1", port: 8080, workers: 4 }
```

A `port: 0` default would be a bug the compiler cannot see. Prefer a manual impl
over a derive whose zeros are meaningless.

## Default on enums

A derived `Default` needs you to say which variant.

```rust
#[derive(Debug, Default, PartialEq)]
enum Mode { #[default] Fast, Safe, Paranoid }

println!("{:?}", Mode::default());
```

Output:

```
Fast
```

Forget the attribute and it fails:

```rust
#[derive(Debug, Default)]
enum Mode { Fast, Safe }
```

Does not compile:

```
error[E0665]: `#[derive(Default)]` on enum with no `#[default]`
 --> src/main.rs:1:17
  |
1 | #[derive(Debug, Default)]
  |                 ^^^^^^^
2 | enum Mode { Fast, Safe }
  | ------------------------ this enum needs a unit variant marked with `#[default]`
  |
help: make this unit variant default by placing `#[default]` on it
  |
2 | enum Mode { #[default] Fast, Safe }
  |             ++++++++++
```

`#[default]` works on **unit variants only**:

```rust
#[derive(Debug, Default)]
enum Mode { #[default] Fast(u8), Safe }
```

```
error: the `#[default]` attribute may only be used on unit enum variants
 --> src/main.rs:2:24
  |
2 | enum Mode { #[default] Fast(u8), Safe }
  |                        ^^^^
  |
  = help: consider a manual implementation of `Default`
```

Take the suggestion: `impl Default for Mode { fn default() -> Self { Mode::Fast(0) } }`.

## Default nests

```rust
#[derive(Debug, Default)]
struct Outer { inner: Config, mode: Mode, label: Option<String> }

println!("{:?}", Outer::default());
```

Output:

```
Outer { inner: Config { host: "", port: 0, tls: false, retries: 0, tags: [], timeout: None }, mode: Fast, label: None }
```

## Where Default pays off

`Default` unlocks `unwrap_or_default`, `reduce(...).unwrap_or_default()`,
`mem::take`, and `#[serde(default)]`.

```rust
let none: Option<Config> = None;
println!("{:?}", none.unwrap_or_default());

let nums = vec![1, 2, 3];
println!("{}", nums.iter().copied().reduce(|a, b| a + b).unwrap_or_default());
```

Output:

```
Config { host: "", port: 0, tls: false, retries: 0, tags: [], timeout: None }
6
```

## The builder pattern, owned self

Each setter takes `self` by value and returns it, so the calls chain. Required
fields live in the builder as `Option`, so `build` can detect what is missing.

```rust
#[derive(Debug, PartialEq)]
struct Server { host: String, port: u16, tls: bool, workers: usize, max_body: usize }

#[derive(Debug, Default)]
struct ServerBuilder {
    host: Option<String>,
    port: Option<u16>,
    tls: bool,
    workers: Option<usize>,
    max_body: Option<usize>,
}

impl Server {
    fn builder() -> ServerBuilder { ServerBuilder::default() }
}

impl ServerBuilder {
    fn host(mut self, h: &str) -> Self { self.host = Some(h.to_string()); self }
    fn port(mut self, p: u16) -> Self { self.port = Some(p); self }
    fn tls(mut self, on: bool) -> Self { self.tls = on; self }
    fn workers(mut self, w: usize) -> Self { self.workers = Some(w); self }

    fn build(self) -> Result<Server, String> {
        let host = self.host.ok_or("host is required")?;
        let port = self.port.unwrap_or(8080);
        if port < 1024 {
            return Err(format!("port {port} is privileged"));
        }
        let workers = self.workers.unwrap_or(4);
        if workers == 0 {
            return Err("workers must be > 0".to_string());
        }
        Ok(Server {
            host,
            port,
            tls: self.tls,
            workers,
            max_body: self.max_body.unwrap_or(1 << 20),
        })
    }
}

println!("{:?}", Server::builder().host("example.com").port(8443).tls(true).build());
println!("{:?}", Server::builder().port(8443).build());
println!("{:?}", Server::builder().host("a").port(80).build());
println!("{:?}", Server::builder().host("a").workers(0).build());
```

Output:

```
Ok(Server { host: "example.com", port: 8443, tls: true, workers: 4, max_body: 1048576 })
Err("host is required")
Err("port 80 is privileged")
Err("workers must be > 0")
```

`build() -> Result` is the key move: validation happens once, at the boundary,
and a `Server` that exists is a `Server` that is valid. Required fields are not
checkable at compile time this way, but they are impossible to forget silently.

`Server::builder()` as the entry point keeps the builder type out of the caller's
imports. Note `tls` is a plain `bool`, not `Option<bool>` — an unset flag with an
unambiguous default needs no `Option`.

## Owned builders are single-use

```rust
let b = B::default();
let x = b.n(1);
let y = b.n(2);
```

Does not compile:

```
error[E0382]: use of moved value: `b`
 --> src/main.rs:9:13
  |
7 |     let b = B::default();
  |         - move occurs because `b` has type `B`, which does not implement the `Copy` trait
8 |     let x = b.n(1);
  |               ---- `b` moved due to this method call
9 |     let y = b.n(2);
  |             ^ value used here after move
  |
note: `B::n` takes ownership of the receiver `self`, which moves `b`
```

Fine for one fluent chain; a problem if you want to configure in a loop or
conditionally. For that, use the `&mut self` variant.

## The builder pattern, &mut self

Setters borrow mutably and return `&mut Self`. `build` takes `&self` and clones,
so the builder stays usable.

```rust
#[derive(Debug, Default, Clone)]
struct Query { table: String, wheres: Vec<String>, limit: Option<usize> }

#[derive(Debug, Default)]
struct QueryBuilder { q: Query }

impl QueryBuilder {
    fn new(table: &str) -> Self {
        QueryBuilder { q: Query { table: table.into(), ..Default::default() } }
    }
    fn filter(&mut self, c: &str) -> &mut Self { self.q.wheres.push(c.into()); self }
    fn limit(&mut self, n: usize) -> &mut Self { self.q.limit = Some(n); self }
    fn build(&self) -> Query { self.q.clone() }
}

let mut qb = QueryBuilder::new("users");
qb.filter("age > 18").filter("active = true").limit(10);
let q = qb.build();
println!("{:?}", q);

qb.limit(50);
println!("{:?}", qb.build());
println!("{:?}", q.limit);
```

Output:

```
Query { table: "users", wheres: ["age > 18", "active = true"], limit: Some(10) }
Query { table: "users", wheres: ["age > 18", "active = true"], limit: Some(50) }
Some(10)
```

The already-built `q` is unaffected by the later `limit(50)`.

| | owned `self` | `&mut self` |
|---|---|---|
| Chains in one expression | yes | yes |
| Reusable after `build()` | no | yes |
| Conditional/loop setting | awkward | natural |
| `build` cost | moves, no clone | clones |
| Can be the whole chain's value | `let s = X::builder()...build()?;` | needs a `let mut` first |

Owned is the common default in the ecosystem. Switch to `&mut self` when callers
need to build in steps.

## Typestate: enforcing required fields at compile time

A type parameter tracks the state, and methods only exist in the states where
they are legal. The missing method becomes a compile error rather than an
`Err`.

```rust
use std::marker::PhantomData;

struct Open;
struct Closed;

#[derive(Debug)]
struct Door<S> { _state: PhantomData<S> }

impl Door<Closed> {
    fn new() -> Self { Door { _state: PhantomData } }
    fn open(self) -> Door<Open> { println!("opening"); Door { _state: PhantomData } }
}

impl Door<Open> {
    fn close(self) -> Door<Closed> { println!("closing"); Door { _state: PhantomData } }
    fn walk_through(&self) { println!("walking through"); }
}

let d = Door::new();
let d = d.open();
d.walk_through();
let d = d.close();
```

Output:

```
opening
walking through
closing
```

Calling `d.walk_through()` on the closed door:

```
error[E0599]: no method named `walk_through` found for struct `Door<Closed>` in the current scope
  --> src/main.rs:24:7
   |
 7 | struct Door<S> { _state: PhantomData<S> }
   | -------------- method `walk_through` not found for this struct
...
24 |     d.walk_through();
   |       ^^^^^^^^^^^^ method not found in `Door<Closed>`
   |
   = note: the method was found for `Door<Open>`
```

`PhantomData<S>` is zero-sized, so the whole thing costs nothing at runtime —
`size_of::<Door<Open>>()` is 0. Applied to a builder, `ServerBuilder<HostSet>`
would make `build()` exist only once `host` is set, eliminating the
`Err("host is required")` branch. It also doubles your type count, so save it
for APIs where a wrong call sequence is expensive. Modules 09 and 16 cover
generics and `PhantomData`.

## When a builder beats a plain literal

| Situation | Use |
|---|---|
| Struct you own, all fields public, few fields | literal + `..Default::default()` |
| 6+ fields, most optional | builder |
| Fields need validation before the value exists | builder with `build() -> Result` |
| Private fields, construction crosses a module boundary | builder |
| Derived values (one field computed from others) | builder |
| Public API that must stay back-compatible when a field is added | builder |
| Field order matters to the reader | builder (setters are named) |

The last row is the quiet one: adding a field to a public struct is a breaking
change for every literal in every downstream crate; adding a setter to a builder
is not.

The `derive_builder` and `typed-builder` crates generate all of this from an
attribute. Write one by hand first so you know what they emit.

## Quick Reference

| Form | Meaning |
|---|---|
| `#[derive(Default)]` | field-wise zeros/empties |
| `T::default()` | explicit |
| `Default::default()` | inferred from context |
| `#[default]` on a unit variant | which enum variant is the default |
| `impl Default for T` | custom, non-zero defaults |
| `T { a: 1, ..Default::default() }` | set some, default the rest |
| `opt.unwrap_or_default()` | needs `T: Default` |
| `fn set(mut self, v) -> Self` | owned builder setter, chainable, single use |
| `fn set(&mut self, v) -> &mut Self` | borrowing builder setter, reusable |
| `fn build(self) -> Result<T, E>` | validate at construction |
| `T::builder()` | conventional builder entry point |
| `PhantomData<S>` | typestate tag, zero-sized |

## Common Mistakes

**Deriving `Default` when zero is not a valid value.** `port: 0`, `retries: 0`,
`timeout: 0` are silent bugs. Write `impl Default` by hand.

**`#[derive(Default)]` on an enum with no `#[default]`.** `E0665`. Mark one unit
variant, or implement `Default` manually if the default carries data.

**Reusing an owned builder after a setter.** `E0382`; the setter moved it. Use
`&mut self` setters, or clone.

**`build()` returning `T` when fields can be missing.** Then `build` has to
`unwrap` or invent a value. Return `Result` and report what is missing.

**Writing a builder for a 3-field struct you own.** `..Default::default()` on a
literal does the job with no extra type.

**Making every builder field `Option`.** A `bool` flag or a `Vec` that defaults
to empty needs no `Option` — the plain type's default already means "unset".

## Exercises

1. Give `struct Retry { attempts: u32, backoff_ms: u64, jitter: bool }` a manual
   `Default` of 3 / 250 / true. Print it, then build one with
   `Retry { attempts: 5, ..Default::default() }`.
2. Define `enum Compression { None, Gzip, Zstd }` deriving `Default` with `Gzip`
   as the default. Then try `#[default]` on a variant carrying a level and
   capture the error.
3. Write an owned-`self` builder for `Email { to: String, subject: String,
   body: String, cc: Vec<String>, html: bool }`, with `build() -> Result<Email,
   String>` requiring `to` and rejecting an empty `subject`. Print one success
   and two distinct failures.
4. Convert that builder to `&mut self` setters and prove the builder is still
   usable after `build()` by producing two different `Email`s from it.
5. Add `cc(&mut self, addr: &str)` that appends, and build an email whose `cc`
   list is assembled in a `for` loop over a `Vec<&str>`.
6. Sketch a typestate `EmailBuilder<ToSet>` where `build()` exists only after
   `to()` is called. Show the compile error from calling `build()` first.

---

[← 5.6 Deriving Common Traits](06-derive.md) · [Course Index](../README.md) · [6.1 String vs &str →](../06-strings/01-string-vs-str.md)
