[← 9.5 Trait Objects and Dynamic Dispatch](05-trait-objects.md) · [Course Index](../README.md) · [9.7 From, Into, TryFrom, and Conversions →](07-conversions.md)

# 9.6 — Operator Overloading and Standard Traits

## What You'll Learn

- `std::ops`: `Add`/`Sub`/`Mul`/`Div`/`Rem`/`Neg`/`Not` and the `*Assign` family
- The `Rhs` generic parameter and the `Output` associated type
- `Index`/`IndexMut`, and `PartialEq`/`Eq`/`PartialOrd`/`Ord` by hand vs derive
- `Display` vs `Debug`, `Default`, `Iterator`/`IntoIterator`, `Deref`

## Coming From Another Language

| Other | Rust |
|---|---|
| Python `__add__`, `__getitem__` | `impl Add`, `impl Index` |
| TS — no operator overloading | `impl Add for T` |
| Go — none at all | `impl Add for T` |

Rust has no operator syntax of its own: `a + b` *is* `Add::add(a, b)`, `a[i]`
*is* `*Index::index(&a, i)`. You cannot invent new operators, only implement the
existing traits. Overloaded operators are still statically dispatched and
inlined.

## Add, Sub, Neg

Each has one `Output` associated type and takes `self` by value.

```rust
use std::ops::{Add, Sub, Neg};

#[derive(Debug, Clone, Copy, PartialEq)]
struct V2 { x: f64, y: f64 }

impl Add for V2 {
    type Output = V2;
    fn add(self, o: V2) -> V2 { V2 { x: self.x + o.x, y: self.y + o.y } }
}
impl Sub for V2 {
    type Output = V2;
    fn sub(self, o: V2) -> V2 { V2 { x: self.x - o.x, y: self.y - o.y } }
}
impl Neg for V2 {
    type Output = V2;
    fn neg(self) -> V2 { V2 { x: -self.x, y: -self.y } }
}

fn main() {
    let a = V2 { x: 1.0, y: 2.0 };
    let b = V2 { x: 3.0, y: 4.0 };
    println!("{:?}", a + b);
    println!("{:?}", b - a);
    println!("{:?}", -a);
    println!("{}", a == V2 { x: 1.0, y: 2.0 });
}
```

Output:

```
V2 { x: 4.0, y: 6.0 }
V2 { x: 2.0, y: 2.0 }
V2 { x: -1.0, y: -2.0 }
true
```

`V2` is `Copy`, so `a + b` does not move `a`. For a non-`Copy` type, `a + b`
consumes both operands — implement `Add for &T` as well if that is a problem.

## `Output` need not be `Self`

A dot product returns a scalar.

```rust
use std::ops::Mul;

impl Mul for V2 {
    type Output = f64;
    fn mul(self, o: V2) -> f64 { self.x * o.x + self.y * o.y }
}

println!("{}", a * b);
```

Output:

```
11
```

## The `Rhs` parameter: mixed-type operands

`Add` is really `Add<Rhs = Self>`. Name a different `Rhs` to mix types.

```rust
use std::ops::{Div, Mul};

impl Mul<f64> for V2 {
    type Output = V2;
    fn mul(self, s: f64) -> V2 { V2 { x: self.x * s, y: self.y * s } }
}

impl Mul<V2> for f64 {
    type Output = V2;
    fn mul(self, v: V2) -> V2 { v * self }
}

impl Div<f64> for V2 {
    type Output = V2;
    fn div(self, s: f64) -> V2 { V2 { x: self.x / s, y: self.y / s } }
}

println!("{:?}", a * 3.0);
println!("{:?}", 3.0 * a);
println!("{:?}", b / 2.0);
```

Output:

```
V2 { x: 3.0, y: 6.0 }
V2 { x: 3.0, y: 6.0 }
V2 { x: 1.5, y: 2.0 }
```

`impl Mul<V2> for f64` works because `V2` is local — the orphan rule (lesson
9.2) allows a foreign type when a local type appears in the trait's parameters.
Operators are not commutative automatically; `3.0 * a` needs its own impl.

## `Not` and `Rem`

```rust
use std::ops::{Not, Rem};

#[derive(Debug, Clone, Copy, PartialEq)]
struct Flags(u8);

impl Not for Flags {
    type Output = Flags;
    fn not(self) -> Flags { Flags(!self.0) }
}

#[derive(Debug)]
struct Mins(i64);

impl Rem<i64> for Mins {
    type Output = i64;
    fn rem(self, d: i64) -> i64 { self.0 % d }
}

println!("{:?}", !Flags(0b0000_1111));
println!("{}", Mins(125) % 60);
```

Output:

```
Flags(240)
5
```

`Not` is both `!` on bools and bitwise `!` on integers. `BitAnd`, `BitOr`,
`BitXor`, `Shl`, `Shr` follow the same pattern.

## The `*Assign` family

`AddAssign` takes `&mut self` and returns `()`. These are separate traits —
implementing `Add` does not give you `+=`.

```rust
use std::ops::{AddAssign, SubAssign, MulAssign};

impl AddAssign for V2 { fn add_assign(&mut self, o: V2) { self.x += o.x; self.y += o.y; } }
impl SubAssign for V2 { fn sub_assign(&mut self, o: V2) { self.x -= o.x; self.y -= o.y; } }
impl MulAssign<f64> for V2 { fn mul_assign(&mut self, s: f64) { self.x *= s; self.y *= s; } }

let mut c = a;
c += b;
println!("{:?}", c);
c -= a;
println!("{:?}", c);
c *= 2.0;
println!("{:?}", c);
```

Output:

```
V2 { x: 4.0, y: 6.0 }
V2 { x: 3.0, y: 4.0 }
V2 { x: 6.0, y: 8.0 }
```

Full set: `AddAssign`, `SubAssign`, `MulAssign`, `DivAssign`, `RemAssign`,
`BitAndAssign`, `BitOrAssign`, `BitXorAssign`, `ShlAssign`, `ShrAssign`. There
is no `NegAssign` or `NotAssign`.

## Index and IndexMut

`Index` returns `&Self::Output`; the `*` is inserted for you. The index type is
whatever you want.

```rust
use std::ops::{Index, IndexMut};

struct Grid { w: usize, cells: Vec<i32> }

impl Index<(usize, usize)> for Grid {
    type Output = i32;
    fn index(&self, (r, c): (usize, usize)) -> &i32 { &self.cells[r * self.w + c] }
}

impl IndexMut<(usize, usize)> for Grid {
    fn index_mut(&mut self, (r, c): (usize, usize)) -> &mut i32 { &mut self.cells[r * self.w + c] }
}

let mut g = Grid { w: 3, cells: vec![0; 9] };
g[(1, 1)] = 5;
g[(0, 2)] += 7;
println!("{} {} {}", g[(1, 1)], g[(0, 2)], g[(2, 2)]);
```

Output:

```
5 7 0
```

Assignment and `+=` through `[]` need `IndexMut`. Multiple index types are fine:

```rust
impl Index<&str> for Grid {
    type Output = i32;
    fn index(&self, k: &str) -> &i32 {
        match k { "first" => &self.cells[0], _ => &self.cells[self.cells.len() - 1] }
    }
}

println!("{} {}", g["first"], g["last"]);
```

Output:

```
0 0
```

There is no fallible `Index` — it must return a reference, so out of range means
panic. Provide a `get` method returning `Option` alongside it.

## PartialEq and Eq

Derive when field-by-field equality is right.

```rust
#[derive(Debug, PartialEq, Eq)]
struct Version { major: u32, minor: u32 }
```

Write it by hand when equality is custom. Here, case-insensitive:

```rust
#[derive(Debug)]
struct Ci(String);

impl PartialEq for Ci {
    fn eq(&self, o: &Self) -> bool { self.0.to_lowercase() == o.0.to_lowercase() }
}

println!("{}", Ci("Hello".into()) == Ci("hELLO".into()));
```

Output:

```
true
```

`!=` comes free from `eq`. `Eq` is a marker meaning the relation is reflexive —
`a == a` always. `f64` is `PartialEq` but not `Eq` because `NaN != NaN`:

```rust
println!("{:?}", 1.0_f64.partial_cmp(&f64::NAN));
```

Output:

```
None
```

Only `Eq + Hash` types can be a `HashMap` key, which is why `f64` cannot.

## PartialOrd and Ord

Derive compares fields in declaration order. Enums compare by variant order.

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
enum Level { Low, Mid, High }

#[derive(Debug, PartialEq, PartialOrd)]
struct Version(u32, u32);

println!("{}", Level::Low < Level::High);
let mut v = vec![Level::High, Level::Low, Level::Mid];
v.sort();
println!("{:?}", v);
println!("{}", Version(1, 9) < Version(1, 10));
```

Output:

```
true
[Low, Mid, High]
true
```

By hand: implement `Ord::cmp`, then make `partial_cmp` delegate to it.

```rust
use std::cmp::Ordering;

#[derive(Debug, Eq, PartialEq)]
struct Version { major: u32, minor: u32 }

impl Ord for Version {
    fn cmp(&self, o: &Self) -> Ordering {
        self.major.cmp(&o.major).then(self.minor.cmp(&o.minor))
    }
}

impl PartialOrd for Version {
    fn partial_cmp(&self, o: &Self) -> Option<Ordering> { Some(self.cmp(o)) }
}

let v1 = Version { major: 1, minor: 9 };
let v2 = Version { major: 1, minor: 10 };
println!("{} {} {}", v1 < v2, v1 == v2, v1.max(v2) == Version { major: 1, minor: 10 });

let mut vs = vec![Version { major: 2, minor: 0 }, Version { major: 1, minor: 5 }];
vs.sort();
println!("{:?}", vs);
```

Output:

```
true false true
[Version { major: 1, minor: 5 }, Version { major: 2, minor: 0 }]
```

`.then` chains comparisons: fall through to the next field only on `Equal`.
`Ord` gives you `.sort()`, `.max()`, `.min()`, `.clamp()`, and `BTreeMap` keys.
`PartialOrd` alone gives `<`/`>` and `.sort_by(|a, b| a.partial_cmp(b).unwrap())`.

## Display vs Debug

Both from `std::fmt`, both one `fmt` method, different audiences.

```rust
use std::fmt;

struct Celsius(f64);

impl fmt::Display for Celsius {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{:.1}C", self.0) }
}

impl fmt::Debug for Celsius {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "Celsius({})", self.0) }
}

let t = Celsius(21.46);
println!("{} / {:?}", t, t);
```

Output:

```
21.5C / Celsius(21.46)
```

`Display` → `{}`, for end users, cannot be derived, gives you `.to_string()`.
`Debug` → `{:?}` and `{:#?}`, for developers, almost always derived.

## Default

Derive gives `0`, `false`, `""`, empty collections, `None`. Write it when the
sensible default is not the zero value.

```rust
#[derive(Debug)]
struct Config { host: String, port: u16, retries: u8 }

impl Default for Config {
    fn default() -> Self { Config { host: "localhost".into(), port: 8080, retries: 3 } }
}

println!("{:?}", Config::default());
println!("{:?}", Config { port: 443, ..Default::default() });
```

Output:

```
Config { host: "localhost", port: 8080, retries: 3 }
Config { host: "localhost", port: 443, retries: 3 }
```

`..Default::default()` in a struct literal is the lightweight builder pattern.
`Default` also unlocks `unwrap_or_default()` and `T::default()` in generic code.

## Iterator and IntoIterator

Implement `Iterator` and every adaptor in std applies to your type.

```rust
struct Countdown(u32);

impl Iterator for Countdown {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.0 == 0 { None } else { self.0 -= 1; Some(self.0 + 1) }
    }
}

println!("{:?}", Countdown(4).collect::<Vec<_>>());
println!("{}", Countdown(4).sum::<u32>());
```

Output:

```
[4, 3, 2, 1]
10
```

`IntoIterator` is what `for` desugars to. The convention is three impls — for
`T`, `&T`, and `&mut T`:

```rust
struct Bag { items: Vec<String> }

impl IntoIterator for Bag {
    type Item = String;
    type IntoIter = std::vec::IntoIter<String>;
    fn into_iter(self) -> Self::IntoIter { self.items.into_iter() }
}

impl<'a> IntoIterator for &'a Bag {
    type Item = &'a String;
    type IntoIter = std::slice::Iter<'a, String>;
    fn into_iter(self) -> Self::IntoIter { self.items.iter() }
}

let bag = Bag { items: vec!["a".into(), "b".into()] };
for s in &bag { println!("ref {}", s); }
for s in bag { println!("own {}", s); }
```

Output:

```
ref a
ref b
own a
own b
```

Module 10 covers iterators in depth; lesson 9.8 explains `type Item`.

## Deref

`Deref` makes your wrapper forward method calls to the inner type.

```rust
struct Token(String);

impl std::ops::Deref for Token {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}

let tok = Token("abc123".into());
println!("{} {} {}", tok.len(), tok.to_uppercase(), tok.starts_with("abc"));
```

Output:

```
6 ABC123 true
```

None of those methods exist on `Token`; deref coercion found them on `str`.
This is how `String` gets `&str`'s methods and `Box<T>` gets `T`'s. Module 11
covers `Deref`, `DerefMut`, and `Drop`. Use it for smart pointers and newtypes
only — not as a substitute for inheritance.

## The common std traits

| Trait | Gives you | Derivable |
|---|---|---|
| `Debug` | `{:?}`, `{:#?}` | yes |
| `Display` | `{}`, `.to_string()` | no |
| `Clone` | `.clone()` | yes |
| `Copy` | implicit bitwise copy instead of move | yes (needs `Clone`) |
| `Default` | `T::default()`, `..Default::default()`, `unwrap_or_default()` | yes |
| `PartialEq` | `==`, `!=`, `.contains()` | yes |
| `Eq` | `HashMap`/`HashSet` key (with `Hash`) | yes |
| `PartialOrd` | `<`, `>`, `<=`, `>=` | yes |
| `Ord` | `.sort()`, `.max()`, `BTreeMap` key | yes |
| `Hash` | `HashMap`/`HashSet` key | yes |
| `Add`..`Rem`, `Neg`, `Not` | the arithmetic operators | no |
| `AddAssign` family | `+=`, `-=`, ... | no |
| `Index` / `IndexMut` | `a[i]`, `a[i] = v` | no |
| `Iterator` | every adaptor and consumer in std | no |
| `IntoIterator` | `for x in value` | no |
| `From<T>` | `.into()` for free, `?` conversions | no |
| `FromStr` | `.parse::<T>()` | no |
| `Deref` | inner type's methods on the wrapper | no |
| `Drop` | custom cleanup | no |
| `Error` | `Box<dyn Error>`, `?` | no |

Lesson 9.7 covers `From`/`Into`/`TryFrom`/`FromStr`; lesson 5.6 covers the
derives in detail.

## Quick Reference

| Expression | Trait and method |
|---|---|
| `a + b` | `Add::add(a, b)` |
| `a - b` | `Sub::sub` |
| `a * b` | `Mul::mul` |
| `a / b` | `Div::div` |
| `a % b` | `Rem::rem` |
| `-a` | `Neg::neg` |
| `!a` | `Not::not` |
| `a += b` | `AddAssign::add_assign(&mut a, b)` |
| `a[i]` | `*Index::index(&a, i)` |
| `a[i] = v` | `*IndexMut::index_mut(&mut a, i) = v` |
| `a == b` | `PartialEq::eq(&a, &b)` |
| `a < b` | `PartialOrd::partial_cmp` |
| `a.cmp(&b)` | `Ord::cmp` |
| `for x in c` | `IntoIterator::into_iter(c)` |
| `*p` | `Deref::deref(&p)` |
| Mixed operands | `impl Mul<f64> for V2` |
| Non-`Self` result | `type Output = f64;` |

## Common Mistakes

**Expecting `impl Add` to give you `+=`.** `AddAssign` is a separate trait.

**Forgetting that `+` consumes its operands.** `fn add(self, ...)` takes by
value. Derive `Copy` for small types or add `impl Add for &T`.

**Trying `&a + &b` with only `impl Add for V2`.** `E0369: cannot add &V2 to
&V2`. Implement `Add for &V2` too.

**Deriving `PartialOrd` and expecting a custom order.** Derive compares fields
in declaration order. Reorder the fields or write `Ord::cmp`.

**Implementing `PartialOrd` by hand and `Ord` separately with different logic.**
They must agree. Write `cmp`, then `partial_cmp` as `Some(self.cmp(o))`.

**Using `Deref` to fake inheritance.** It is for smart pointers and newtypes;
abusing it makes method resolution unreadable.

## Exercises

1. Implement `Add`, `Sub`, `Neg`, and `AddAssign` for
   `struct Money { cents: i64 }`. Print each result.
2. Add `impl Mul<i64> for Money` and `impl Mul<Money> for i64` so both
   `m * 3` and `3 * m` work.
3. Implement `Index<usize>` and `IndexMut<usize>` for
   `struct Ring { buf: Vec<i32> }` that wraps the index modulo the length.
   Add a `get` returning `Option<&i32>`.
4. Write `Ord` for `struct Task { priority: u8, name: String }` so higher
   priority sorts first and ties break by name. Sort a `Vec<Task>`.
5. Implement `Iterator` for `struct Fib { a: u64, b: u64 }`, then print the first
   ten values and the sum of the even ones using `.filter()` and `.take()`.

---

[← 9.5 Trait Objects and Dynamic Dispatch](05-trait-objects.md) · [Course Index](../README.md) · [9.7 From, Into, TryFrom, and Conversions →](07-conversions.md)
