[← 2.5 Numeric Operators and Type Casting](05-operators-casting.md) · [Course Index](../README.md) · [2.7 Comments and Documentation Comments →](07-comments-docs.md)

# 2.6 — Constants, Statics, and Type Aliases

## What You'll Learn

- `const` is inlined at every use; `static` is one fixed memory location
- Both require an explicit type and a compile-time-evaluable initializer
- `static mut` needs `unsafe` — and what to use instead
- Type aliases rename, they do not create new types
- `const fn` and const generics, briefly

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `const X = 5` at module scope | `const X: i32 = 5;` |
| Python module-level `X = 5` | `const X: i32 = 5;` |
| Go `const X = 5` / `var X = 5` | `const` / `static` |
| TS `type Id = string` | `type Id = String;` |

`let` is for function bodies; `const` and `static` are items and can sit at
module scope. Unlike `let`, both always need a type annotation.

## const

```rust
const MAX_POINTS: u32 = 100_000;
const GREETING: &str = "hello";
const PI_2: f64 = std::f64::consts::PI * 2.0;
const BUFFER: [u8; 4] = [0; 4];
const SECONDS_PER_DAY: u32 = 60 * 60 * 24;

fn main() {
    const LOCAL: i32 = 7;
    println!("{MAX_POINTS} {GREETING} {PI_2} {:?} {SECONDS_PER_DAY} {LOCAL}", BUFFER);
}
```

Output:

```
100000 hello 6.283185307179586 [0, 0, 0, 0] 86400 7
```

The initializer is computed at compile time — `60 * 60 * 24` never runs at
runtime. A `const` can be declared anywhere an item can, including inside a
function body (scoped to it) and inside an `impl`.

`SCREAMING_SNAKE_CASE` is enforced by a lint:

```rust
const max_points: u32 = 10;
```

```
warning: constant `max_points` should have an upper case name
 --> src/main.rs:1:7
  |
1 | const max_points: u32 = 10;
  |       ^^^^^^^^^^
  |
  = note: `#[warn(non_upper_case_globals)]` (part of `#[warn(nonstandard_style)]`) on by default
```

## const: the three rules

**The type is mandatory.**

```rust
const MAX = 10;
```

```
error: missing type for `const` item
 --> src/main.rs:1:10
  |
1 | const MAX = 10;
  |          ^ help: provide a type for the constant: `: i32`
```

**It can never be mutable.**

```rust
const mut MAX: i32 = 10;
```

```
error: const globals cannot be mutable
 --> src/main.rs:1:7
  |
1 | const mut MAX: i32 = 10;
  |       ^^^ cannot be mutable
  |
help: you might want to declare a static instead
```

**The initializer must be const-evaluable.**

```rust
fn get() -> i32 { 5 }
const MAX: i32 = get();
```

```
error[E0015]: cannot call non-const function `get` in constants
 --> src/main.rs:2:18
  |
2 | const MAX: i32 = get();
  |                  ^^^^^
  |
note: function `get` is not const
```

## static

```rust
static APP_NAME: &str = "w02";
static VERSION: u32 = 1;
static NAMES: [&str; 2] = ["a", "b"];
static TABLE: [(u8, &str); 2] = [(1, "one"), (2, "two")];

fn main() {
    println!("{APP_NAME} {VERSION}");
    println!("{:?} {:?}", NAMES, TABLE);
    let r: &'static str = NAMES[0];
    println!("{r}");
}
```

Output:

```
w02 1
["a", "b"] [(1, "one"), (2, "two")]
a
```

A `static` has a single address that lives for the whole program, so references
to it are `&'static`. A `const` has no address of its own — each use site gets
its own inlined copy. That is the entire difference, and it decides which to
pick.

## const vs static vs let

| | `const` | `static` | `let` |
|---|---|---|---|
| Scope | item (module, fn, impl) | item | function body only |
| Type annotation | required | required | optional |
| Initializer | const-evaluable | const-evaluable | any expression |
| Memory | inlined per use, no address | one fixed address | stack slot |
| Reference lifetime | n/a | `'static` | scoped |
| Mutable form | never | `static mut` (unsafe) | `let mut` |
| Interior mutability | not allowed | allowed (`Atomic*`, `Mutex`) | allowed |
| Shadowing | no | no | yes |

**Which to use:** `const` for almost everything — thresholds, conversion
factors, config defaults, array sizes. `static` only when you need one shared
address (a lazily built table, an atomic counter, a `Mutex`-guarded global) or
a large value you do not want copied into every use site. `let` for everything
inside a function.

## const fn

```rust
const fn kib(n: usize) -> usize { n * 1024 }
const BUF_SIZE: usize = kib(8);

fn main() {
    println!("{BUF_SIZE}");
    let arr = [0u8; BUF_SIZE];
    println!("{}", arr.len());
}
```

Output:

```
8192
8192
```

A `const fn` runs at compile time when the context demands it and at runtime
otherwise, with no penalty either way. It is what lets a `const` be computed
rather than written as a literal — including array lengths, which must be
constants.

## Associated constants

```rust
struct Circle { r: f64 }

impl Circle {
    const UNIT: Circle = Circle { r: 1.0 };
    fn area(&self) -> f64 { std::f64::consts::PI * self.r * self.r }
}

println!("{:.4}", Circle::UNIT.area());
println!("{} {}", u32::MAX, i8::MIN);
```

Output:

```
3.1416
4294967295 -128
```

`u32::MAX` and `f64::EPSILON` are exactly this — associated constants, not
magic. Define your own on types and traits to keep names out of the module
namespace.

## static mut requires unsafe

```rust
static mut COUNTER: u32 = 0;

fn main() {
    COUNTER += 1;
    println!("{COUNTER}");
}
```

Does not compile:

```
error[E0133]: use of mutable static is unsafe and requires unsafe block
 --> src/main.rs:3:5
  |
3 |     COUNTER += 1;
  |     ^^^^^^^ use of mutable static
  |
  = note: mutable statics can be mutated by multiple threads: aliasing violations or data races will cause undefined behavior
```

Reading it directly inside `println!` is *also* rejected in edition 2024, even
inside `unsafe`, because the formatting macro takes a reference:

```
error: creating a shared reference to mutable static
 --> src/main.rs:9:20
  |
9 |         println!("{COUNTER}");
  |                    ^^^^^^^ shared reference to mutable static
  |
  = note: `#[deny(static_mut_refs)]` (part of `#[deny(rust_2024_compatibility)]`) on by default
```

You have to copy the value out first:

```rust
static mut COUNTER: u32 = 0;

unsafe {
    COUNTER += 1;
    let v = COUNTER;
    println!("{v}");
}
```

Output:

```
1
```

## Use an atomic instead

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static SAFE_COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    SAFE_COUNTER.fetch_add(1, Ordering::Relaxed);
    SAFE_COUNTER.fetch_add(1, Ordering::Relaxed);
    println!("{}", SAFE_COUNTER.load(Ordering::Relaxed));
}
```

Output:

```
2
```

No `unsafe`, no data race, and it works from multiple threads. This is the
right answer for a mutable global in essentially every case: `AtomicU32` /
`AtomicBool` for counters and flags, `Mutex`/`RwLock` for anything larger,
`OnceLock`/`LazyLock` for a value built once on first use. Note the `static`
itself is not `mut` — the mutation goes through interior mutability. Module 14
covers atomics and locks; module 11 covers interior mutability.

## Type aliases

```rust
type Pair = (i32, i32);
type Table = std::collections::HashMap<String, Vec<u32>>;
type Res<T> = Result<T, String>;
type Callback = fn(i32) -> i32;

fn parse(s: &str) -> Res<i32> {
    s.parse().map_err(|_| format!("bad: {s}"))
}

fn main() {
    let p: Pair = (1, 2);
    let mut t: Table = Table::new();
    t.insert("a".into(), vec![1, 2]);
    println!("{:?} {:?}", p, t);
    println!("{:?} {:?}", parse("5"), parse("x"));
    let cb: Callback = |x| x + 1;
    println!("{}", cb(1));
}
```

Output:

```
(1, 2) {"a": [1, 2]}
Ok(5) Err("bad: x")
2
```

Aliases can be generic (`Res<T>`), and the alias works as a constructor too
(`Table::new()`). The main payoff is a project-wide `Result` alias — module 08
makes heavy use of this.

## An alias is not a new type

```rust
type Meters = f64;
type Feet = f64;

fn add(m: Meters, f: Feet) -> f64 { m + f }

let m: Meters = 5.0;
let f: Feet = 3.0;
println!("{}", add(f, m));
```

Output:

```
8
```

Arguments swapped, no complaint: `Meters` and `Feet` are both just `f64`.
Aliases shorten names, they do not add type safety. For that you need a tuple
struct — `struct Meters(f64)` — which lesson 5.3 covers.

## Const generics

A type can be parameterized by a value, not just a type.

```rust
#[derive(Debug)]
struct Buffer<const N: usize> {
    data: [u8; N],
}

impl<const N: usize> Buffer<N> {
    const CAPACITY: usize = N;
    fn new() -> Self { Buffer { data: [0; N] } }
    fn len(&self) -> usize { self.data.len() }
}

fn first<const N: usize>(a: [i32; N]) -> Option<i32> {
    a.first().copied()
}

fn main() {
    let b: Buffer<4> = Buffer::new();
    let c = Buffer::<8>::new();
    println!("{:?} {} {}", b, b.len(), c.len());
    println!("{} {}", Buffer::<4>::CAPACITY, Buffer::<8>::CAPACITY);
    println!("{:?} {:?}", first([1, 2, 3]), first::<0>([]));
}
```

Output:

```
Buffer { data: [0, 0, 0, 0] } 4 8
4 8
Some(1) None
```

`Buffer<4>` and `Buffer<8>` are distinct types with their sizes known at
compile time. This is how `[T; N]` itself gets its standard-library impls for
any `N`. You will meet const generics mainly when writing code generic over
array length; module 09 covers generics in general.

## Where each belongs

```rust
const DEFAULT_TIMEOUT_SECS: u64 = 30;
const MAX_RETRIES: u8 = 3;
const BUF_SIZE: usize = 8 * 1024;

static APP_NAME: &str = env!("CARGO_PKG_NAME");
static REQUESTS: std::sync::atomic::AtomicU64 = std::sync::atomic::AtomicU64::new(0);

type ParseResult<T> = Result<T, std::num::ParseIntError>;

fn to_num(s: &str) -> ParseResult<u32> { s.parse() }

fn main() {
    println!("{DEFAULT_TIMEOUT_SECS} {MAX_RETRIES} {BUF_SIZE} {APP_NAME}");
    REQUESTS.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    println!("{:?} {:?}", REQUESTS, to_num("9"));
}
```

Output:

```
30 3 8192 w02
1 Ok(9)
```

Thresholds and sizes: `const`. One shared mutable counter or a lazily built
table: `static` with interior mutability. A long generic type you write more
than twice: `type`. (`env!` is a compile-time macro, so it is a valid `static`
initializer.)

## Quick Reference

| Form | Meaning |
|---|---|
| `const NAME: T = v;` | Inlined compile-time constant |
| `static NAME: T = v;` | One `'static` memory location |
| `static mut NAME: T = v;` | Mutable global, needs `unsafe` — avoid |
| `const fn f() -> T` | Callable in a const context |
| `impl S { const C: T = v; }` | Associated constant, `S::C` |
| `type Alias = Long<Type>;` | Rename, not a new type |
| `type Res<T> = Result<T, E>;` | Generic alias |
| `struct S<const N: usize>` | Const generic parameter |
| `AtomicU32`, `Mutex`, `OnceLock` in a `static` | Safe mutable global |

## Common Mistakes

**Leaving off the type.** Both `const` and `static` require an annotation;
inference does not apply to items.

**Reaching for `static mut`.** It is `unsafe` and edition 2024 blocks even
reading it through a reference. Use an atomic, a `Mutex`, or `OnceLock`.

**Expecting a type alias to prevent mix-ups.** `type Meters = f64` is `f64`.
Use a tuple struct for real distinctness.

**Calling a normal function in a `const` initializer.** Mark it `const fn`.

**Using `static` for plain values.** `const` is the default; `static` only
earns its keep when you need the single address.

## Exercises

1. Define `const MAX_USERS: u32`, `const APP: &str`, and a local `const` inside
   `main`. Then try each with a lowercase name and read the warning.
2. Write `const fn secs_in(days: u64) -> u64` and use it to initialize
   `const WEEK: u64`. Print it.
3. Build a `static` lookup table `[(&str, u8); 3]` of month abbreviations, and
   a function that returns `Option<u8>` for a given name.
4. Make a global request counter with `AtomicUsize`, bump it from three
   function calls, and print the total.
5. Define `type Grid = Vec<Vec<u8>>` and `struct Rows(usize)`. Show that two
   `Grid` values are interchangeable while `Rows(3)` cannot be passed where a
   `usize` is expected.

---

[← 2.5 Numeric Operators and Type Casting](05-operators-casting.md) · [Course Index](../README.md) · [2.7 Comments and Documentation Comments →](07-comments-docs.md)
