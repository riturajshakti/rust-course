[← 5.1 Defining and Instantiating Structs](01-structs.md) · [Course Index](../README.md) · [5.3 Tuple Structs and Unit Structs →](03-tuple-unit-structs.md)

# 5.2 — Methods and Associated Functions

## What You'll Learn

- `impl` blocks, and the four receiver forms: `&self`, `&mut self`, `self`, none
- The `new` convention, `Self`, and associated constants
- Method chaining, auto-ref/auto-deref, getters, consuming `into_*` methods
- Methods on enums

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `class User { greet() {} }` | `struct User {}` + `impl User { fn greet(&self) {} }` |
| Python `def greet(self)` | `fn greet(&self)` — but `&`/`&mut`/by-value are distinct |
| Go `func (u *User) Greet()` | `fn greet(&mut self)` |
| JS `static create()` | `fn create() -> Self` (no `self` parameter) |

Data and behaviour are declared separately: `struct` holds fields, `impl` holds
functions. The receiver is explicit and its *mode* is part of the signature.

## impl blocks and the four receivers

```rust
#[derive(Debug)]
struct Rect { width: u32, height: u32 }

impl Rect {
    fn area(&self) -> u32 { self.width * self.height }
    fn scale(&mut self, k: u32) { self.width *= k; self.height *= k; }
    fn into_square(self) -> Rect {
        let s = self.width.max(self.height);
        Rect { width: s, height: s }
    }
    fn describe() -> &'static str { "an axis-aligned rectangle" }
}

fn main() {
    let mut r = Rect { width: 3, height: 4 };
    println!("{}", r.area());
    r.scale(2);
    println!("{:?}", r);
    println!("{:?}", r.into_square());
    println!("{}", Rect::describe());
}
```

Output:

```
12
Rect { width: 6, height: 8 }
Rect { width: 8, height: 8 }
an axis-aligned rectangle
```

| Receiver | Means | Call as |
|---|---|---|
| `&self` | read-only borrow | `r.area()` |
| `&mut self` | exclusive borrow, can mutate | `r.scale(2)` (needs `let mut`) |
| `self` | takes ownership, consumes the value | `r.into_square()` |
| none | associated function, no instance | `Rect::describe()` |

Default to `&self`. Reach for `&mut self` only to mutate, and `self` only when
the method logically destroys or transforms the value.

## &mut self needs a mut binding

```rust
let r = Rect { width: 3, height: 4 };
r.scale(2);
```

Does not compile:

```
error[E0596]: cannot borrow `r` as mutable, as it is not declared as mutable
 --> src/main.rs:8:5
  |
8 |     r.scale(2);
  |     ^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
7 |     let mut r = Rect { width: 3, height: 4 };
  |         +++
```

## self by value consumes the receiver

```rust
struct Temp { celsius: f64 }
impl Temp {
    fn into_fahrenheit(self) -> f64 { self.celsius * 9.0 / 5.0 + 32.0 }
}

let t = Temp { celsius: 25.0 };
println!("{}", t.into_fahrenheit());
println!("{}", t.celsius);
```

Does not compile:

```
error[E0382]: borrow of moved value: `t`
 --> src/main.rs:8:20
  |
6 |     let t = Temp { celsius: 25.0 };
  |         - move occurs because `t` has type `Temp`, which does not implement the `Copy` trait
7 |     println!("{}", t.into_fahrenheit());
  |                      ----------------- `t` moved due to this method call
8 |     println!("{}", t.celsius);
  |                    ^^^^^^^^^ value borrowed here after move
  |
note: `Temp::into_fahrenheit` takes ownership of the receiver `self`, which moves `t`
 --> src/main.rs:3:24
  |
3 |     fn into_fahrenheit(self) -> f64 { self.celsius * 9.0 / 5.0 + 32.0 }
  |                        ^^^^
```

This is the point of `self`: a **conversion** should not leave the old value
lying around. `String::into_bytes`, `Vec::into_iter`, `Box::into_inner` all do
this. It also lets the method reuse the old allocation instead of cloning.

## Associated functions and the `new` convention

A function without a `self` parameter is called on the type. `new` is a
convention, not a keyword — there is no constructor syntax and no `new` operator.

```rust
#[derive(Debug)]
struct Rect { width: u32, height: u32 }

impl Rect {
    fn new(width: u32, height: u32) -> Self { Self { width, height } }
    fn square(side: u32) -> Self { Self::new(side, side) }
}

println!("{:?}", Rect::new(3, 4));
println!("{:?}", Rect::square(5));
```

Output:

```
Rect { width: 3, height: 4 }
Rect { width: 5, height: 5 }
```

`Self` is an alias for the implementing type. Prefer it over repeating `Rect` —
renaming the struct then touches one line.

## Associated constants

```rust
impl Rect {
    const MAX_SIDE: u32 = 4096;
}

println!("{}", Rect::MAX_SIDE);
```

Output:

```
4096
```

Namespaced to the type, resolved at compile time, no instance involved.

## Multiple impl blocks

Any number of `impl` blocks for the same type are allowed, in the same file or
across the module. Useful for grouping, and required once generic bounds differ.

```rust
impl Rect {
    fn area(&self) -> u32 { self.width * self.height }
}

impl Rect {
    fn perimeter(&self) -> u32 { 2 * (self.width + self.height) }
}

let r = Rect::new(3, 4);
println!("{} {}", r.area(), r.perimeter());
```

Output:

```
12 14
```

## Getters and setters

Fields are private outside the module, so accessors are how you expose them.
A getter returning a `Copy` field returns it by value; a `String` field is
returned as `&str`.

```rust
struct Temp { celsius: f64 }

impl Temp {
    fn celsius(&self) -> f64 { self.celsius }
    fn set_celsius(&mut self, c: f64) { self.celsius = c; }
}

let mut t = Temp { celsius: 25.0 };
println!("{}", t.celsius());
t.set_celsius(100.0);
println!("{}", t.celsius());
```

Output:

```
25
100
```

A getter may share the field's name — methods and fields live in different
namespaces. Rust does not prefix getters with `get_`; it does prefix setters
with `set_`.

## Method chaining with a builder

`fn method(mut self) -> Self` takes ownership, mutates the local copy, and hands
it back. Each call consumes and returns, so they chain.

```rust
#[derive(Debug)]
struct Counter { n: u32 }

impl Counter {
    fn new() -> Self { Counter { n: 0 } }
    fn bump(mut self) -> Self { self.n += 1; self }
    fn get(&self) -> u32 { self.n }
}

println!("{}", Counter::new().bump().bump().bump().get());
```

Output:

```
3
```

`mut self` is not a fifth receiver — it is `self` with a mutable binding.

A real builder with a separate output type:

```rust
#[derive(Debug)]
struct Request { url: String, method: String, timeout: u32 }

struct RequestBuilder { url: String, method: String, timeout: u32 }

impl RequestBuilder {
    fn new(url: &str) -> Self {
        RequestBuilder { url: url.to_string(), method: "GET".into(), timeout: 30 }
    }
    fn method(mut self, m: &str) -> Self { self.method = m.to_string(); self }
    fn timeout(mut self, t: u32) -> Self { self.timeout = t; self }
    fn build(self) -> Request {
        Request { url: self.url, method: self.method, timeout: self.timeout }
    }
}

let req = RequestBuilder::new("https://example.com")
    .method("POST")
    .timeout(5)
    .build();
println!("{:?}", req);
```

Output:

```
Request { url: "https://example.com", method: "POST", timeout: 5 }
```

Lesson 5.7 covers the `&mut self` variant and validating in `build`.

## Auto-ref and auto-deref

`r.area()` is sugar. The compiler inserts `&`, `&mut`, or `*` as needed to make
the receiver match the signature.

```rust
struct Rect { w: u32, h: u32 }
impl Rect { fn area(&self) -> u32 { self.w * self.h } }

let r = Rect { w: 4, h: 5 };
println!("{} {}", r.area(), Rect::area(&r));

let boxed = Box::new(Rect { w: 2, h: 3 });
println!("{}", boxed.area());
```

Output:

```
20 20
6
```

`r.area()` becomes `Rect::area(&r)`. `boxed.area()` derefs the `Box` first.
This is why Rust has no `->` operator: `.` handles both cases.

## Methods on enums

`impl` works on enums identically. The body typically `match`es on `self`.

```rust
enum Shape { Circle(f64), Square(f64) }

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => std::f64::consts::PI * r * r,
            Shape::Square(s) => s * s,
        }
    }
    fn name(&self) -> &'static str {
        match self {
            Shape::Circle(_) => "circle",
            Shape::Square(_) => "square",
        }
    }
}

for s in &[Shape::Circle(1.0), Shape::Square(2.0)] {
    println!("{} {:.3}", s.name(), s.area());
}
```

Output:

```
circle 3.142
square 4.000
```

Lesson 5.4 covers enums in full.

## Quick Reference

| Form | Meaning |
|---|---|
| `impl S { ... }` | inherent methods for `S` |
| `fn f(&self)` | immutable borrow of the instance |
| `fn f(&mut self)` | mutable borrow; caller needs `let mut` |
| `fn f(self)` | consumes the instance |
| `fn f(mut self) -> Self` | consumes, mutates, returns — chainable |
| `fn f()` | associated function, called `S::f()` |
| `Self` | alias for the implementing type |
| `const C: T = v;` inside `impl` | associated constant, `S::C` |
| `fn new(...) -> Self` | constructor convention |
| `fn into_x(self) -> X` | consuming conversion |
| `s.m()` | auto-refs/derefs to match the receiver |

## Common Mistakes

**Forgetting `&self`.** `fn area()` inside `impl` is an associated function, so
`r.area()` fails with "no method named `area`". Add the receiver.

**Using `self` where `&self` works.** It consumes the value and surprises every
caller. Reserve by-value receivers for conversions and builders.

**Calling a `&mut self` method on an immutable binding.** `E0596`; add `mut` to
the `let`.

**Expecting a constructor keyword.** `new` is just a function name. Nothing
calls it implicitly, and you can name it `from_parts`, `with_capacity`, anything.

**Writing `get_width()`.** Rust's convention is `width()` for the getter and
`set_width()` for the setter.

## Exercises

1. Define `Circle { radius: f64 }` with `new`, `area(&self)`, and an associated
   const `PI_APPROX`. Print all three.
2. Add `grow(&mut self, by: f64)` and call it on an immutable binding. Read the
   error, then fix it.
3. Write `fn into_diameter(self) -> f64` and try to use the `Circle` afterwards.
   Capture the error text.
4. Build a `Pizza` with a chaining builder: `size`, `topping`, `extra_cheese`,
   then `build()`. Print the result with `{:#?}`.
5. Define `enum Status { Active, Suspended, Closed }` with a method
   `can_login(&self) -> bool`. Print it for all three variants.

---

[← 5.1 Defining and Instantiating Structs](01-structs.md) · [Course Index](../README.md) · [5.3 Tuple Structs and Unit Structs →](03-tuple-unit-structs.md)
