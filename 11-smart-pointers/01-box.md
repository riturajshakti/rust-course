[← 10.8 itertools and rayon](../10-closures-iterators/08-itertools-rayon.md) · [Course Index](../README.md) · [11.2 Rc and Shared Ownership →](02-rc.md)

# 11.1 — Box and Heap Allocation

## What You'll Learn

- `Box::new` to move a value to the heap, `*` to get it back
- Why recursive types need a `Box` and what `E0072` looks like
- `Box<dyn Trait>` and `Box<dyn Error>` for one-of-many types
- Sizes: a `Box` is a pointer, whatever it holds

## Coming From Another Language

| Other | Rust |
|---|---|
| JS/Python: every object is already heap-allocated | only when you ask: `Box::new(v)` |
| Go `*T` from `new(T)` | `Box<T>` — but with a single owner and automatic drop |
| C++ `std::unique_ptr<T>` | `Box<T>`, essentially the same thing |

A `Box<T>` is the simplest smart pointer: one owner, heap storage, freed when
the box drops. No reference counting, no runtime cost beyond the allocation.

## Box::new and deref

```rust
fn main() {
    let b = Box::new(5);
    println!("{}", b);
    println!("{}", *b + 1);
    let big = Box::new([0u8; 1024]);
    println!("{}", big.len());
    let s: Box<String> = Box::new(String::from("heap"));
    println!("{} {}", s, s.len());
}
```

Output:

```
5
6
1024
heap 4
```

`*b` reads through the box. You rarely write `*` — method calls and `println!`
go through the box automatically (lesson 11.5 explains that coercion).

## When you actually need a Box

Boxing an `i32` is pointless. The cases that matter:

1. A recursive type, whose size would otherwise be infinite.
2. A trait object, where the concrete type isn't known at compile time.
3. A value too large to keep moving on the stack.
4. Shrinking an enum whose one fat variant inflates every value.

Everything else: use the value directly.

## Recursive types and E0072

A cons list that stores `List` inline has no finite size.

```rust
enum List {
    Cons(i32, List),
    Nil,
}

fn main() {
    let _l = List::Cons(1, List::Cons(2, List::Nil));
}
```

Does not compile — intentional:

```
error[E0072]: recursive type `List` has infinite size
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^
2 |     Cons(i32, List),
  |               ---- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
  |
2 |     Cons(i32, Box<List>),
  |               ++++    +
```

Take the suggestion. `Box<List>` is a pointer, so `Cons` is `i32` + pointer and
the size is fixed.

```rust
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn sum(l: &List) -> i32 {
    match l {
        Cons(v, rest) => v + sum(rest),
        Nil => 0,
    }
}

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{:?}", list);
    println!("{}", sum(&list));
}
```

Output:

```
Cons(1, Cons(2, Cons(3, Nil)))
6
```

Matching on `rest` gives you a `&Box<List>`, and the recursive call takes it as
`&List` by coercion — no `*` needed.

## Expression tree

The same shape, used for something real.

```rust
enum Expr {
    Num(f64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}

fn eval(e: &Expr) -> f64 {
    match e {
        Expr::Num(n) => *n,
        Expr::Add(a, b) => eval(a) + eval(b),
        Expr::Mul(a, b) => eval(a) * eval(b),
    }
}

fn main() {
    let e = Expr::Add(
        Box::new(Expr::Num(2.0)),
        Box::new(Expr::Mul(Box::new(Expr::Num(3.0)), Box::new(Expr::Num(4.0)))),
    );
    println!("{}", eval(&e));
}
```

Output:

```
14
```

## Box\<dyn Trait\>

A `Vec` must have one element type. `Box<dyn Shape>` makes that one type cover
every `Shape` implementor.

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Circle(f64);
struct Square(f64);

impl Shape for Circle {
    fn area(&self) -> f64 { 3.14159 * self.0 * self.0 }
    fn name(&self) -> &str { "circle" }
}
impl Shape for Square {
    fn area(&self) -> f64 { self.0 * self.0 }
    fn name(&self) -> &str { "square" }
}

fn pick(big: bool) -> Box<dyn Shape> {
    if big { Box::new(Circle(10.0)) } else { Box::new(Square(2.0)) }
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Circle(1.0)), Box::new(Square(3.0))];
    for s in &shapes {
        println!("{}: {:.2}", s.name(), s.area());
    }
    let s = pick(true);
    println!("{} {:.1}", s.name(), s.area());
}
```

Output:

```
circle: 3.14
square: 9.00
circle 314.2
```

Returning two different concrete types from one `fn` is impossible without a
box (or an enum). `-> impl Shape` would also fail here, since it means *one*
hidden type, not a choice at runtime. Module 09 covers trait objects in depth.

## Box\<dyn Error\>

The pragmatic error type: `?` converts any `std::error::Error` into it.

```rust
use std::error::Error;

fn parse_and_double(s: &str) -> Result<i32, Box<dyn Error>> {
    let n: i32 = s.trim().parse()?;
    Ok(n * 2)
}

fn main() {
    println!("{:?}", parse_and_double(" 21 "));
    match parse_and_double("abc") {
        Ok(v) => println!("{}", v),
        Err(e) => println!("error: {}", e),
    }
}
```

Output:

```
Ok(42)
error: invalid digit found in string
```

Good for binaries and prototypes; libraries should define a concrete error
enum instead. Module 08 covers the trade-off.

## Sizes

```rust
fn main() {
    println!("{}", size_of::<i32>());
    println!("{}", size_of::<Box<i32>>());
    println!("{}", size_of::<[u8; 4096]>());
    println!("{}", size_of::<Box<[u8; 4096]>>());
    println!("{}", size_of::<Box<dyn std::fmt::Debug>>());
    println!("{}", size_of::<Box<[u8]>>());
    println!("{}", size_of::<Option<Box<i32>>>());
}
```

Output:

```
4
8
4096
8
16
16
8
```

A `Box<T>` for sized `T` is one pointer. A `Box<dyn Trait>` or `Box<[T]>` is
*two* — data pointer plus vtable or length. And `Option<Box<T>>` is still 8:
the null pointer encodes `None`, so `Option` is free here.

`size_of` is in the prelude, so no `std::mem::` prefix is needed.

## Shrinking an enum with Box

An enum is as large as its largest variant. One fat variant makes every value
fat.

```rust
#[derive(Debug)]
struct Big([u64; 32]);

enum Inline {
    Small(u8),
    Huge(Big),
}

enum Boxed {
    Small(u8),
    Huge(Box<Big>),
}

fn main() {
    println!("{} {}", size_of::<Inline>(), size_of::<Boxed>());
}
```

Output:

```
264 16
```

Worth doing when the fat variant is rare — every `Small` was paying 264 bytes.
Not worth it when the big variant is the common case, since you add an
allocation and an indirection to the hot path.

## Moving out of a Box

`*box` on a non-`Copy` value moves the contents out and frees the box.

```rust
fn main() {
    let b = Box::new(String::from("owned"));
    let s: String = *b;
    println!("{}", s);

    let bs = Box::new(Big([7; 32]));
    let inner: Big = *bs;
    println!("{}", inner.0[0]);
}
```

Output:

```
owned
7
```

`Box` is the only type that lets you move the value out through `*`; `Rc` and
`RefCell` cannot (they need `try_unwrap` / `into_inner`). After the move, `b` is
gone — using it is a use-after-move error.

## Box::leak

Converts a `Box<T>` into a `&'static mut T` by never freeing it.

```rust
fn main() {
    let leaked: &'static mut String = Box::leak(Box::new(String::from("static")));
    leaked.push('!');
    println!("{}", leaked);
}
```

Output:

```
static!
```

A deliberate one-time leak, not a bug — useful for config loaded once at
startup that the rest of the program wants as `&'static`. Never call it in a
loop.

## Quick Reference

| Form | Meaning |
|---|---|
| `Box::new(v)` | move `v` to the heap |
| `*b` | deref; moves out if `T` isn't `Copy` |
| `Box<T>` size | 8 bytes (one pointer) for sized `T` |
| `Box<dyn Trait>` size | 16 bytes (data + vtable) |
| `Box<Self>` in enum variant | breaks recursion, fixes `E0072` |
| `Box<dyn Error>` | catch-all error type for `?` |
| `Box::leak(b)` | `&'static mut T`, never freed |
| `Option<Box<T>>` | 8 bytes — null pointer is `None` |
| drop | frees the heap allocation automatically |

## Common Mistakes

**Boxing for no reason.** `Box<i32>` is slower than `i32` and buys nothing. Box
for recursion, trait objects, or genuine size problems.

**Expecting `E0072` to mention `Box`.** It says "recursive without indirection";
the `help` block gives you the exact fix — apply it.

**Writing `*b.method()`.** Deref coercion already calls methods through the box;
`b.method()` is correct and `*b.method()` parses as `*(b.method())`.

**Using `Box<dyn Error>` in a library's public API.** Callers can't match on it.
Define an error enum and keep the box for binaries.

**Trying to move out of an `Rc` with `*`.** Only `Box` supports that. See
lesson 11.2 for `Rc::try_unwrap`.

## Exercises

1. Define `enum Tree { Leaf(i32), Node(Box<Tree>, Box<Tree>) }`. Write
   `fn depth(t: &Tree) -> u32` and `fn sum(t: &Tree) -> i32`, and test both on a
   three-level tree.
2. Write the cons `List` from this lesson, then add
   `fn push_front(list: List, v: i32) -> List` and build a list by folding over
   `[1, 2, 3]`.
3. Define a trait `Animal` with `speak(&self)`, implement it for two structs,
   and store both in a `Vec<Box<dyn Animal>>`. Then write a function returning
   `Box<dyn Animal>` chosen from a `&str`.
4. Print `size_of` for an enum with a `[u8; 1000]` variant, then box that
   variant and print it again. Report both numbers.
5. Write a function returning `Result<u32, Box<dyn Error>>` that parses a string
   and rejects values over 100 with a custom message. Print both the success and
   failure paths.

---

[← 10.8 itertools and rayon](../10-closures-iterators/08-itertools-rayon.md) · [Course Index](../README.md) · [11.2 Rc and Shared Ownership →](02-rc.md)
