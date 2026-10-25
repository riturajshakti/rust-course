[← 5.2 Methods and Associated Functions](02-methods.md) · [Course Index](../README.md) · [5.4 Enums and Variants with Data →](04-enums.md)

# 5.3 — Tuple Structs and Unit Structs

## What You'll Learn

- Tuple structs, `.0`/`.1` access, and destructuring them
- The newtype pattern: how wrapping `u64` stops you mixing two kinds of ID
- Unit structs and what they are actually for
- The size of each form

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `type UserId = number` (an alias, erased) | `struct UserId(u64)` — a *distinct* type |
| Python `NamedTuple("Point", [...])` | `struct Point(i32, i32)` |
| Go `type Meters float64` | `struct Meters(f64)` |

The key difference from a TS/Go alias: `UserId` is not interchangeable with
`u64`. Passing the wrong one is a compile error, not a bug you ship.

## Tuple structs

Fields are positional and unnamed. The declaration ends in a semicolon.

```rust
#[derive(Debug)]
struct Point(i32, i32);

fn main() {
    let p = Point(3, 4);
    println!("{} {}", p.0, p.1);
    println!("{:?}", p);
    println!("{:#?}", p);
}
```

Output:

```
3 4
Point(3, 4)
Point(
    3,
    4,
)
```

The name doubles as a constructor function: `Point(3, 4)` is a call.

## Destructuring

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Rgb(u8, u8, u8);

let c = Rgb(255, 128, 0);
let Rgb(r, g, b) = c;
println!("{r} {g} {b}");

let p = Point(3, 4);
let Point(x, y) = p;
println!("{x} {y}");
```

Output:

```
255 128 0
3 4
```

The pattern works anywhere patterns do — `let`, `match`, function parameters,
`for` bindings:

```rust
let nums = vec![Point(1, 2), Point(3, 4)];
for Point(a, b) in &nums {
    println!("{a},{b}");
}
```

Output:

```
1,2
3,4
```

## Mutating and methods

`.0` is a normal place expression, and `impl` works as usual.

```rust
#[derive(Debug)]
struct Meters(f64);

impl Meters {
    fn to_feet(&self) -> f64 { self.0 * 3.28084 }
}

let mut m = Meters(100.0);
m.0 += 1.0;
println!("{:.3} {:?}", m.to_feet(), m);
```

Output:

```
331.365 Meters(101.0)
```

## The newtype pattern

A one-field tuple struct around an existing type. It costs nothing at runtime
and buys type safety. With raw `u64`, nothing stops this:

```rust
fn fetch_user(id: u64) -> String { format!("user {id}") }

let order_id: u64 = 42;
println!("{}", fetch_user(order_id));
```

Output:

```
user 42
```

Compiles, runs, wrong. Now wrap both:

```rust
struct UserId(u64);
struct OrderId(u64);

fn fetch_user(id: UserId) -> String { format!("user {}", id.0) }

let order = OrderId(42);
println!("{}", fetch_user(order));
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:8:31
  |
8 |     println!("{}", fetch_user(order));
  |                    ---------- ^^^^^ expected `UserId`, found `OrderId`
  |                    |
  |                    arguments to this function are incorrect
  |
note: function defined here
 --> src/main.rs:4:4
  |
4 | fn fetch_user(id: UserId) -> String { format!("user {}", id.0) }
  |    ^^^^^^^^^^ ----------
```

Same for `Meters` vs `Feet`, `Celsius` vs `Fahrenheit`, `Sanitized` vs `Raw`.
The newtype constructor is also a function value, so it works in `map`:

```rust
#[derive(Debug, PartialEq)]
struct UserId(u64);

let ids: Vec<UserId> = vec![1, 2, 3].into_iter().map(UserId).collect();
println!("{:?}", ids);
println!("{}", ids[0] == UserId(1));
```

Output:

```
[UserId(1), UserId(2), UserId(3)]
true
```

The other reason to newtype: the orphan rule stops you implementing a foreign
trait on a foreign type, but `struct MyVec(Vec<i32>)` is *yours*, so you can
implement anything on it. Module 08 covers that rule.

## Newtype plus Deref

Wrapping hides the inner type's methods. Implementing `Deref` forwards them.

```rust
#[derive(Debug)]
struct Wrapper(Vec<i32>);

impl std::ops::Deref for Wrapper {
    type Target = Vec<i32>;
    fn deref(&self) -> &Vec<i32> { &self.0 }
}

let w = Wrapper(vec![1, 2, 3]);
println!("{} {}", w.len(), w.iter().sum::<i32>());
```

Output:

```
3 6
```

Convenient, but it leaks the inner API and partly defeats the point of the
newtype — use it for smart-pointer-like wrappers, not for ID types. Module 11
covers `Deref` properly.

## Unit structs

No fields, no parentheses, no braces.

```rust
#[derive(Debug, Default, Clone, Copy, PartialEq, Eq)]
struct Always;

let a = Always;
println!("{:?} {}", a, Always == Always::default());
```

Output:

```
Always true
```

The name is both the type and its single value.

## What unit structs are for

**A target for a trait impl.** The behaviour is in the impl; the type carries no
data.

```rust
trait Logger { fn log(&self, msg: &str); }

struct ConsoleLogger;
struct NullLogger;

impl Logger for ConsoleLogger {
    fn log(&self, msg: &str) { println!("[console] {msg}"); }
}
impl Logger for NullLogger {
    fn log(&self, _msg: &str) {}
}

fn run(l: &dyn Logger) { l.log("starting"); l.log("done"); }

run(&ConsoleLogger);
run(&NullLogger);
```

Output:

```
[console] starting
[console] done
```

`NullLogger` is a no-op implementation that occupies zero bytes.

**A marker or state tag.** Combined with generics, unit structs become typestate
flags — `Connection<Open>` vs `Connection<Closed>` — so the wrong call is a
compile error. Lesson 5.7 sketches this.

**A unit-conversion source.** Unit structs and newtypes pair well with `From`:

```rust
struct Grams(f64);
struct Kilograms(f64);

impl From<Kilograms> for Grams {
    fn from(k: Kilograms) -> Self { Grams(k.0 * 1000.0) }
}

let g: Grams = Kilograms(2.5).into();
println!("{}", g.0);
```

Output:

```
2500
```

## Size and layout

```rust
use std::mem::{size_of, size_of_val};

struct Marker;

println!("{}", size_of::<Point>());
println!("{}", size_of::<Rgb>());
println!("{}", size_of::<UserId>());
println!("{}", size_of::<u64>());
println!("{}", size_of::<Marker>());

let v = vec![Always; 1000];
println!("{} {}", v.len(), size_of_val(&*v));
```

Output:

```
8
3
8
8
0
1000 0
```

A newtype is exactly the size of what it wraps — the wrapper is erased at
compile time. Unit structs are zero-sized, and a thousand of them in a `Vec`
still occupy zero bytes of heap data; the `Vec` tracks only the length.

## Named fields vs positional

Use a tuple struct when the position is self-evident and there are one to three
fields. Past that, name them.

```rust
struct Rgb(u8, u8, u8);
struct Rect { width: u32, height: u32 }
```

`Rect(3, 4)` would be ambiguous at every call site; `Rgb(255, 0, 0)` is not.

## Quick Reference

| Form | Syntax |
|---|---|
| Tuple struct | `struct Point(i32, i32);` |
| Construct | `Point(3, 4)` |
| Access | `p.0`, `p.1` |
| Mutate | `let mut p = ...; p.0 = 9;` |
| Destructure | `let Point(x, y) = p;` |
| Newtype | `struct UserId(u64);` |
| Newtype as fn | `.map(UserId)` |
| Forward inner API | `impl Deref for Wrapper` |
| Unit struct | `struct Marker;` |
| Construct unit | `Marker` |
| Size | `size_of::<UserId>() == size_of::<u64>()`, `size_of::<Marker>() == 0` |

## Common Mistakes

**Writing `struct Point(i32, i32) {}`.** Tuple and unit struct declarations end
with a semicolon, not a block.

**Expecting `UserId` to accept a `u64`.** It is a distinct type; write
`UserId(n)` or implement `From<u64>`.

**Using a tuple struct for five fields.** `cfg.3` tells a reader nothing. Switch
to named fields.

**Reaching for `Deref` on an ID newtype.** It re-exposes the inner type and the
type safety erodes. Add the two or three methods you need instead.

**Thinking a unit struct is a singleton with identity.** It is zero-sized;
every `Marker` value is indistinguishable, and `&Marker` is not a meaningful
address.

## Exercises

1. Define `struct Celsius(f64)` and `struct Fahrenheit(f64)`, with `From` in
   both directions. Round-trip 37.0 and print both.
2. Define `Rgb(u8, u8, u8)` with a method `to_hex(&self) -> String`. Print
   `Rgb(255, 128, 0)` as `#ff8000`.
3. Write `fn charge(amount: Cents)` with `struct Cents(u64)`, then try calling
   it with a bare `u64`. Capture the error.
4. Make `struct Inventory(Vec<String>)` and implement `Deref` so `.len()` and
   `.iter()` work. Print both results.
5. Define unit structs `Json` and `Csv`, a trait `Format { fn ext(&self) -> &str }`,
   implement it for both, and print `size_of` of each.

---

[← 5.2 Methods and Associated Functions](02-methods.md) · [Course Index](../README.md) · [5.4 Enums and Variants with Data →](04-enums.md)
