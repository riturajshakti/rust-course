[← Course Index](../README.md)

# 9.2 — Defining and Implementing Traits

## What You'll Learn

- `trait` definitions and `impl Trait for Type`
- Why a trait must be imported before its methods are visible
- The orphan rule and the newtype workaround
- Hand-writing `Display` and `Debug`, static trait methods, marker traits

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `interface` + `class implements` | `trait` + `impl Trait for Type` |
| Go `interface` (implicit) | explicit `impl` block, required |
| Python ABC / duck typing | `trait`, checked at compile time |

Closest to a Go interface, but **nominal**: having the right methods is not
enough, you must write the `impl`. Upside — you can implement your trait for
types you did not define, including `i32` and `Vec<T>`.

## Defining and implementing

```rust
trait Summary {
    fn summarize(&self) -> String;
    fn word_count(&self) -> usize;
}

struct Article { title: String, body: String }
struct Tweet { user: String, text: String }

impl Summary for Article {
    fn summarize(&self) -> String { format!("{}: {}", self.title, self.body) }
    fn word_count(&self) -> usize { self.body.split_whitespace().count() }
}

impl Summary for Tweet {
    fn summarize(&self) -> String { format!("@{}: {}", self.user, self.text) }
    fn word_count(&self) -> usize { self.text.split_whitespace().count() }
}

fn main() {
    let a = Article { title: "Rust 1.99".into(), body: "traits are everywhere now".into() };
    let t = Tweet { user: "raj".into(), text: "shipping it".into() };
    println!("{}", a.summarize());
    println!("{}", t.summarize());
    println!("{} {}", a.word_count(), t.word_count());
}
```

Output:

```
Rust 1.99: traits are everywhere now
@raj: shipping it
4 2
```

Signatures in the trait end with `;`. Every required method must appear in every
`impl`, or you get `error[E0046]: not all trait items implemented`.

## Implementing for types you don't own

Allowed as long as the **trait** is yours.

```rust
impl Summary for i32 {
    fn summarize(&self) -> String { format!("the number {}", self) }
    fn word_count(&self) -> usize { 3 }
}

impl Summary for Vec<String> {
    fn summarize(&self) -> String { format!("{} strings", self.len()) }
    fn word_count(&self) -> usize { self.len() }
}

println!("{}", 42.summarize());
println!("{}", vec!["a".to_string(), "b".to_string()].summarize());
```

Output:

```
the number 42
2 strings
```

## Calling trait methods three ways

```rust
println!("{}", a.summarize());
println!("{}", Summary::summarize(&a));
println!("{}", <Tweet as Summary>::summarize(&t));
```

Output:

```
Rust 1.99: traits are everywhere now
Rust 1.99: traits are everywhere now
@raj: shipping it
```

Method syntax is the normal choice. The other two matter when two traits give
the same method name — lesson 9.4 covers disambiguation.

## Trait methods need the trait in scope

A method only resolves if the trait is imported. Intentionally broken:

```rust
mod shapes {
    pub trait Area {
        fn area(&self) -> f64;
    }
    pub struct Square(pub f64);
    impl Area for Square {
        fn area(&self) -> f64 { self.0 * self.0 }
    }
}

fn main() {
    let s = shapes::Square(3.0);
    println!("{}", s.area());
}
```

```
error[E0599]: no method named `area` found for struct `Square` in the current scope
  --> src/main.rs:13:22
   |
 3 |         fn area(&self) -> f64;
   |            ---- the method is available for `Square` here
...
13 |     println!("{}", s.area());
   |                      ^^^^ method not found in `Square`
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `Area` which provides `area` is implemented but not in scope; perhaps you want to import it
   |
 1 + use crate::shapes::Area;
   |
```

The fix is the `use`:

```rust
use shapes::Area;

println!("{}", shapes::Square(3.0).area());
println!("{}", <shapes::Square as Area>::area(&shapes::Square(2.0)));
```

Output:

```
9
4
```

This is why real crates ship a `prelude` module. `std` auto-imports its own
prelude, which is why `.to_string()` and `.into()` just work.

## The orphan rule

You may implement a trait for a type only if **the trait or the type is local to
your crate**. Both foreign is forbidden. Intentionally broken:

```rust
impl std::fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "vec")
    }
}
```

```
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/main.rs:1:1
  |
1 | impl std::fmt::Display for Vec<i32> {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^--------
  |                            |
  |                            `Vec` is not defined in the current crate
  |
  = note: impl doesn't have any local type before any uncovered type parameters
  = note: define and implement a trait or new type instead
```

Without this rule, two dependencies could both implement `Display for Vec<i32>`
and your build would be ambiguous.

## Newtype workaround

Wrap the foreign type in a local tuple struct; now the type is yours.

```rust
use std::fmt;
use std::ops::Deref;

struct Wrapper(Vec<i32>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0.iter().map(|v| v.to_string()).collect::<Vec<_>>().join(", "))
    }
}

impl Deref for Wrapper {
    type Target = Vec<i32>;
    fn deref(&self) -> &Vec<i32> { &self.0 }
}

let w = Wrapper(vec![1, 2, 3]);
println!("{}", w);
println!("{} {:?}", w.len(), w.first());
```

Output:

```
1, 2, 3
3 Some(1)
```

The `Deref` impl is the trick that makes the wrapper feel transparent —
`w.len()` reaches the inner `Vec`. Module 11 covers `Deref`. Newtypes are free
at runtime: a one-field tuple struct has the same layout as its field.

Other direction: your own trait on a foreign type needs no wrapper.

```rust
trait Loud { fn loud(&self) -> String; }

impl Loud for String {
    fn loud(&self) -> String { self.to_uppercase() }
}

println!("{}", "hello".to_string().loud());
```

Output:

```
HELLO
```

## Implementing Display by hand

One required method. `write!` targets the formatter and returns `fmt::Result`.

```rust
use std::fmt;

struct Temp(f64);

impl fmt::Display for Temp {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.1}°C", self.0)
    }
}

let temp = Temp(21.456);
println!("{}", temp);
println!("{}", temp.to_string());
```

Output:

```
21.5°C
21.5°C
```

`Display` gives you `.to_string()` for free (lesson 9.7 explains why).

Multi-part output: `?` after each `write!` to propagate.

```rust
struct MyVec(Vec<i32>);

impl fmt::Display for MyVec {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[")?;
        for (i, v) in self.0.iter().enumerate() {
            if i > 0 { write!(f, " | ")?; }
            write!(f, "{}", v)?;
        }
        write!(f, "]")
    }
}

println!("{}", MyVec(vec![1, 2, 3]));
println!("{}", MyVec(vec![]));
```

Output:

```
[1 | 2 | 3]
[]
```

Hex and other formats are just `write!` format specs:

```rust
struct Rgb(u8, u8, u8);

impl fmt::Display for Rgb {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "#{:02X}{:02X}{:02X}", self.0, self.1, self.2)
    }
}

println!("{}", Rgb(255, 128, 0));
```

Output:

```
#FF8000
```

## Width and alignment need f.pad

A plain `write!` ignores the caller's `{:>10}`. Use `f.pad` with a pre-built
string to honour it.

```rust
impl fmt::Display for Temp {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let s = format!("{:.1}C", self.0);
        f.pad(&s)
    }
}

println!("[{:>10}]", Temp(21.456));
println!("[{:<10}]", Temp(21.456));
```

Output:

```
[     21.5C]
[21.5C     ]
```

## Implementing Debug by hand

Usually you derive it. When you write it, the `debug_*` builders keep `{:#?}`
working.

```rust
impl fmt::Debug for Temp {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Temp").field("celsius", &self.0).finish()
    }
}

println!("{:?}", Temp(21.456));
```

Output:

```
Temp { celsius: 21.456 }
```

`debug_tuple`, `debug_list`, `debug_map`, and `debug_set` cover the other
shapes. `Display` is for users; `Debug` is for you.

## Static trait methods

A method without `self` is an associated function. Call it on the type.

```rust
trait Shape {
    fn area(&self) -> f64;
    fn unit() -> Self;
    fn name() -> &'static str where Self: Sized { "shape" }
}

struct Circle { r: f64 }
struct Square { s: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.r * self.r }
    fn unit() -> Self { Circle { r: 1.0 } }
    fn name() -> &'static str { "circle" }
}

impl Shape for Square {
    fn area(&self) -> f64 { self.s * self.s }
    fn unit() -> Self { Square { s: 1.0 } }
}

let c = Circle::unit();
let s = <Square as Shape>::unit();
println!("{:.4} {}", c.area(), Circle::name());
println!("{} {}", s.area(), Square::name());
```

Output:

```
3.1416 circle
1 shape
```

`Square` took the default `name()`; `Circle` overrode it. Lesson 9.4 covers
defaults. A trait with a `self`-less method cannot be used as `dyn Trait`
(lesson 9.5) — `unit()` has no receiver to dispatch on.

## Marker traits

A trait with no methods. It carries no behaviour, only a compile-time claim you
can require in a bound.

```rust
trait Serialize {}
trait Sanitized {}

#[derive(Debug)]
struct Payload(String);

impl Serialize for Payload {}
impl Sanitized for Payload {}

fn ship<T: Serialize + Sanitized + std::fmt::Debug>(p: T) {
    println!("shipping {:?}", p);
}

ship(Payload("hi".into()));
```

Output:

```
shipping Payload("hi")
```

`Copy`, `Send`, `Sync`, and `Sized` are the std markers. A type without
`Sanitized` cannot reach `ship` — the check is free and total.

## Quick Reference

| Form | Syntax |
|---|---|
| Define | `trait T { fn m(&self) -> R; }` |
| Implement | `impl T for S { fn m(&self) -> R { ... } }` |
| Static method | `fn make() -> Self;` called as `S::make()` |
| Call, method form | `value.m()` |
| Call, path form | `T::m(&value)` |
| Call, qualified | `<S as T>::m(&value)` |
| Bring into scope | `use crate::module::T;` |
| Marker trait | `trait Marker {}` + `impl Marker for S {}` |
| `Display` | `fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result` |
| Honour width | `f.pad(&string)` instead of `write!` |
| `Debug` builder | `f.debug_struct("S").field("a", &self.a).finish()` |
| Orphan workaround | `struct Wrapper(Foreign);` + `impl Deref` |

## Common Mistakes

**Forgetting `use` for the trait.** `E0599` "method not found", even though the
impl exists. Import the trait.

**Implementing a foreign trait for a foreign type.** `E0117`. Wrap the type in a
local newtype.

**Omitting a required method.** `E0046` lists exactly what is missing; there is
no implicit stub.

**Writing `impl Trait { ... }` without `for Type`.** That is an inherent impl on
a trait object type and will not compile as intended; the form is
`impl Trait for Type`.

**Implementing `Display` via `write!` and expecting `{:>10}` to work.** Build a
string and call `f.pad`.

## Exercises

1. Define `trait Describe { fn describe(&self) -> String; }` and implement it
   for `i32`, `bool`, and `Vec<i32>`.
2. Hand-write `Display` for `struct Duration { secs: u64 }` printing `1h 02m 03s`.
   Test `0`, `59`, `3723`.
3. Put a trait in a `mod`, implement it for a struct in the same mod, then call
   the method from `main` without importing the trait. Read the error, then fix
   it.
4. Try `impl std::fmt::Display for Option<i32>`. Capture the error, then make it
   work with a newtype.
5. Add `fn zero() -> Self` to a `trait Numeric`, implement it for `i32` and
   `f64`, and write `fn sum_all<T: Numeric + std::ops::Add<Output = T> + Copy>(v: &[T]) -> T`
   that starts from `T::zero()`.

---

[← Course Index](../README.md)
