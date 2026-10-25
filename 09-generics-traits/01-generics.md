[← 8.7 Error Handling Strategy for Libraries vs Apps](../08-error-handling/07-strategy.md) · [Course Index](../README.md) · [9.2 Defining and Implementing Traits →](02-traits.md)

# 9.1 — Generic Functions and Structs

## What You'll Learn

- Generic functions, structs, enums, and `impl` blocks
- Monomorphization: why Rust generics cost nothing at runtime
- Turbofish `::<T>` when inference gives up
- `PhantomData` and const generics `<const N: usize>`

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `function f<T>(x: T): T` | `fn f<T>(x: T) -> T` |
| Go `func F[T any](x T) T` | `fn f<T>(x: T) -> T` |
| Python `def f(x: T)` (hint only) | enforced at compile time |

Closest to Go 1.18+ generics or C++ templates. Two differences: Rust generics
are **fully checked at the definition site** — you cannot use `+` or `>` on a
`T` without declaring a bound — and they compile to separate specialized copies,
so there is zero dispatch cost.

## A generic function

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest { largest = item; }
    }
    largest
}

fn main() {
    println!("{}", largest(&[10, 25, 3]));
    println!("{}", largest(&["pear", "apple", "zed"]));
    println!("{}", largest(&[1.5, 0.2]));
}
```

Output:

```
25
zed
1.5
```

`<T: PartialOrd>` is a **trait bound** — lesson 9.3 covers them properly. For
now: it is the promise that lets you write `>`.

## No bound needed when you only move values

```rust
fn first<T>(list: &[T]) -> Option<&T> { list.first() }

fn swap<A, B>(t: (A, B)) -> (B, A) { (t.1, t.0) }

println!("{:?}", first(&[1, 2, 3]));
println!("{:?}", first::<i32>(&[]));
println!("{:?}", swap((1, "a")));
```

Output:

```
Some(1)
None
("a", 1)
```

## Multiple type parameters

Each one is independent and inferred separately.

```rust
use std::fmt::Debug;

fn pair<A: Debug, B: Debug>(a: A, b: B) -> String {
    format!("{:?} + {:?}", a, b)
}

println!("{}", pair(1, "two"));
```

Output:

```
1 + "two"
```

## The error when you use an operator without a bound

This is the single most common generics error. Intentionally broken:

```rust
fn add<T>(a: T, b: T) -> T {
    a + b
}
```

```
error[E0369]: cannot add `T` to `T`
 --> src/main.rs:2:7
  |
2 |     a + b
  |     - ^ - T
  |     |
  |     T
  |
help: consider restricting type parameter `T` with trait `Add`
  |
1 | fn add<T: std::ops::Add<Output = T>>(a: T, b: T) -> T {
  |         +++++++++++++++++++++++++++
```

Same error shape for `>` without `PartialOrd`:

```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest { largest = item; }
    }
    largest
}
```

```
error[E0369]: binary operation `>` cannot be applied to type `&T`
 --> src/main.rs:4:17
  |
4 |         if item > largest {
  |            ---- ^ ------- &T
  |            |
  |            &T
  |
help: consider restricting type parameter `T` with trait `PartialOrd`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
  |             ++++++++++++++++++++++
```

The compiler always names the missing trait. Take the suggestion.

## Generic structs

```rust
#[derive(Debug)]
struct Point<T> { x: T, y: T }

#[derive(Debug)]
struct Mixed<T, U> { left: T, right: U }

println!("{:?}", Point { x: 3, y: 7 });
println!("{:?}", Point { x: 1.5, y: 2.5 });
println!("{:?}", Mixed { left: 1, right: "one" });
```

Output:

```
Point { x: 3, y: 7 }
Point { x: 1.5, y: 2.5 }
Mixed { left: 1, right: "one" }
```

`Point<T>` uses one parameter for both fields, so both must be the same type:

```rust
let _p = Point { x: 1, y: 2.0 };
```

```
error[E0308]: mismatched types
 --> src/main.rs:3:31
  |
3 |     let _p = Point { x: 1, y: 2.0 };
  |                               ^^^ expected integer, found floating-point number
```

Use `Mixed<T, U>` when the two can differ.

## Generic enums

`Option<T>` and `Result<T, E>` are just this.

```rust
#[derive(Debug)]
enum Tree<T> { Leaf(T), Node(Box<Tree<T>>, Box<Tree<T>>) }

fn depth<T>(t: &Tree<T>) -> usize {
    match t {
        Tree::Leaf(_) => 1,
        Tree::Node(l, r) => 1 + depth(l).max(depth(r)),
    }
}

let t = Tree::Node(
    Box::new(Tree::Leaf(1)),
    Box::new(Tree::Node(Box::new(Tree::Leaf(2)), Box::new(Tree::Leaf(3)))),
);
println!("{:?}", t);
println!("{}", depth(&t));
```

Output:

```
Node(Leaf(1), Node(Leaf(2), Leaf(3)))
3
```

`Box` is needed because a recursive type otherwise has infinite size (Module 11).

## Generic impl blocks

Declare the parameter after `impl`, then use it in the type.

```rust
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self { Point { x, y } }
    fn x(&self) -> &T { &self.x }
}

let p = Point::new(3, 7);
println!("{}", p.x());
```

Output:

```
3
```

The `impl<T>` on the left is the declaration; the `Point<T>` on the right is the
use. Forgetting the left one gives `error[E0412]: cannot find type T in this
scope`.

## Bounded impl blocks

Methods inside only exist for `T` meeting the bound.

```rust
impl<T: std::ops::Add<Output = T> + Copy> Point<T> {
    fn sum(&self) -> T { self.x + self.y }
}

println!("{}", Point { x: 3, y: 7 }.sum());
```

Output:

```
10
```

`Point<String>` still exists, it just has no `.sum()`. This is how `Option` gets
methods like `.unwrap()` only on some variants of itself.

## impl blocks for one concrete type

Drop the parameter entirely and name a type.

```rust
impl Point<f64> {
    fn dist(&self) -> f64 { (self.x * self.x + self.y * self.y).sqrt() }
}

let f = Point::<f64>::new(3.0, 4.0);
println!("{}", f.dist());
```

Output:

```
5
```

`Point<i32>` has no `.dist()`. You can have many such blocks — one per concrete
type — alongside the generic one.

## Monomorphization

Rust does not box or type-erase generics. At compile time it generates one
concrete copy of the code per type you actually instantiate: `show(1u8)` and
`show("hi")` compile to two separate functions, each hard-coded for its type.

Consequences worth internalizing:

- Calls are direct and inlinable — same speed as hand-written code.
- Each instantiation is a real, differently sized type.
- Binary size grows with the number of instantiations.

```rust
#[derive(Debug)]
struct Point<T> { x: T, y: T }

println!("{}", std::mem::size_of::<Point<i32>>());
println!("{}", std::mem::size_of::<Point<u8>>());
println!("{}", std::mem::size_of::<Point<String>>());
println!("{}", std::any::type_name::<Point<i32>>());
```

Output:

```
8
2
48
w09::Point<i32>
```

Three different sizes means three different types. Lesson 9.5 covers
`dyn Trait`, the opposite trade: one copy of the code, a runtime vtable lookup.

## Turbofish `::<T>`

When nothing in the expression pins the type down, inference fails:

```rust
let v = "42".parse().unwrap();
```

```
error[E0284]: type annotations needed
 --> src/main.rs:2:9
  |
2 |     let v = "42".parse().unwrap();
  |         ^        ----- type must be known at this point
  |
  = note: the type must also implement `FromStr`
help: consider giving `v` an explicit type
  |
2 |     let v: /* Type */ = "42".parse().unwrap();
  |          ++++++++++++
```

Fix with an annotation or the turbofish:

```rust
println!("{}", "42".parse::<i32>().unwrap());
let v = Vec::<String>::new();
println!("{:?}", v);
println!("{:?}", (1..4).collect::<Vec<i32>>());
println!("{:?}", first::<i32>(&[]));
```

Output:

```
42
[]
[1, 2, 3]
None
```

`let v: Vec<String> = Vec::new();` is equivalent. Prefer the annotation on
bindings and the turbofish mid-chain.

## PhantomData

To use a type parameter you never store, hold a zero-sized `PhantomData<T>`.

```rust
use std::marker::PhantomData;

#[derive(Debug)]
struct Meters(f64);

#[derive(Debug)]
struct Tagged<T> { value: f64, _unit: PhantomData<T> }

impl<T> Tagged<T> {
    fn new(value: f64) -> Self { Tagged { value, _unit: PhantomData } }
}

let m: Tagged<Meters> = Tagged::new(5.0);
println!("{} {}", m.value, std::mem::size_of::<Tagged<Meters>>());
```

Output:

```
5 8
```

The struct is still 8 bytes — `PhantomData` costs nothing. Without it you get
`error[E0392]: type parameter T is never used`. Typical use: units,
compile-time state machines, and marking ownership for the borrow checker.

## Const generics

Parameters can be values, not just types. `<const N: usize>` makes array length
part of the type.

```rust
#[derive(Debug)]
struct Buffer<const N: usize> { data: [u8; N] }

impl<const N: usize> Buffer<N> {
    fn new() -> Self { Buffer { data: [0; N] } }
    fn len(&self) -> usize { N }
}

fn sum_array<const N: usize>(a: [i32; N]) -> i32 { a.iter().sum() }

let b: Buffer<4> = Buffer::new();
println!("{:?} {}", b, b.len());
let b2 = Buffer::<2>::new();
println!("{:?}", b2);
println!("{}", sum_array([1, 2, 3, 4, 5]));
```

Output:

```
Buffer { data: [0, 0, 0, 0] } 4
Buffer { data: [0, 0] }
15
```

`N` is inferred from the argument in `sum_array`. This is how you write code
over arrays of any length without going through slices.

## Quick Reference

| Form | Syntax |
|---|---|
| Generic fn | `fn f<T>(x: T) -> T` |
| Bounded fn | `fn f<T: PartialOrd>(x: T)` |
| Several params | `fn f<A, B>(a: A, b: B)` |
| Generic struct | `struct S<T> { v: T }` |
| Two params | `struct S<T, U> { a: T, b: U }` |
| Generic enum | `enum E<T> { A(T), B }` |
| Generic impl | `impl<T> S<T> { ... }` |
| Bounded impl | `impl<T: Copy> S<T> { ... }` |
| Concrete impl | `impl S<f64> { ... }` |
| Turbofish | `parse::<i32>()`, `Vec::<T>::new()` |
| Unused param | `PhantomData<T>` |
| Const generic | `struct B<const N: usize>` |
| Inspect type | `std::any::type_name::<T>()` |

## Common Mistakes

**Using an operator without a bound.** `a + b` on a bare `T` is `E0369`; the
compiler names the trait to add. Add it.

**Writing `impl Point<T>` instead of `impl<T> Point<T>`.** The parameter must be
declared after `impl` before it can be used.

**Expecting `S<T>` to allow two different types.** One parameter means one type
for every field using it. Add a second parameter.

**Reaching for generics when one type is enough.** A generic signature is harder
to read and infects callers. Write the concrete version first.

**Holding an unused type parameter without `PhantomData`.** `E0392`. Add a
`PhantomData<T>` field.

## Exercises

1. Write `fn min_max<T: PartialOrd + Copy>(list: &[T]) -> (T, T)` and test it on
   integers and on `&str`.
2. Define `struct Stack<T> { items: Vec<T> }` with `new`, `push`, `pop`
   (returning `Option<T>`), and `len`. Use it with `i32` and `String`.
3. Add `impl Stack<i32> { fn sum(&self) -> i32 }` so only integer stacks get it.
   Confirm calling `.sum()` on a `Stack<String>` fails, and read the error.
4. Write `fn describe<T>(x: T)` that prints `std::any::type_name::<T>()`, and
   call it with `i32`, `Vec<String>`, and `Option<&str>`.
5. Write `fn transpose<const R: usize, const C: usize>(m: [[i32; C]; R]) -> [[i32; R]; C]`.
   Test it on a 2x3 array.

---

[← 8.7 Error Handling Strategy for Libraries vs Apps](../08-error-handling/07-strategy.md) · [Course Index](../README.md) · [9.2 Defining and Implementing Traits →](02-traits.md)
