[← 9.6 Operator Overloading and Standard Traits](06-std-traits.md) · [Course Index](../README.md) · [9.8 Associated Types and Generic Traits →](08-associated-types.md)

# 9.7 — From, Into, TryFrom, and Conversions

## What You'll Learn

- `From` and the free `Into` you get from it
- `TryFrom`/`TryInto` with an error type, for validation
- `AsRef`/`AsMut`, `Borrow`/`ToOwned`, `FromStr` + `.parse()`, `ToString`
- When `as` is wrong and `TryFrom` is right

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `Number(x)`, `String(x)` | `i32::from`, `.to_string()` |
| Python `int(s)` raising | `s.parse::<i32>()` returning `Result` |
| Go `strconv.Atoi` | `.parse()` |

Rust splits conversion into **infallible** (`From`/`Into`) and **fallible**
(`TryFrom`/`TryInto`, `FromStr`). Nothing ever throws; a fallible conversion
returns `Result`. The rule to internalize: **implement `From`, call `.into()`**.

## From

One method. Implementing it on the destination type.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Celsius(f64);
#[derive(Debug, Clone, Copy, PartialEq)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self { Fahrenheit(c.0 * 9.0 / 5.0 + 32.0) }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self { Celsius((f.0 - 32.0) * 5.0 / 9.0) }
}

fn main() {
    let c = Celsius(100.0);
    println!("{:?}", Fahrenheit::from(c));
    let f: Fahrenheit = c.into();
    println!("{:?}", f);
    println!("{:?}", Celsius::from(f));
}
```

Output:

```
Fahrenheit(212.0)
Fahrenheit(212.0)
Celsius(100.0)
```

Read `impl From<Celsius> for Fahrenheit` as "build a `Fahrenheit` from a
`Celsius`". Only implement it when the conversion cannot fail.

## Why you get `Into` for free

std has a blanket impl: `impl<T, U> Into<U> for T where U: From<T>`. One `From`
impl gives you `.into()`, both directions of the API, and `?` conversions.

Trying to write `Into` by hand collides with it. Intentionally broken:

```rust
struct A(i32);
struct B(i32);

impl From<A> for B { fn from(a: A) -> B { B(a.0) } }
impl Into<B> for A { fn into(self) -> B { B(self.0) } }
```

```
error[E0119]: conflicting implementations of trait `Into<B>` for type `A`
 --> src/main.rs:4:1
  |
4 | impl Into<B> for A { fn into(self) -> B { B(self.0) } }
  | ^^^^^^^^^^^^^^^^^^
  |
  = note: conflicting implementation in crate `core`:
          - impl<T, U> Into<U> for T
            where U: From<T>;
```

Never implement `Into`. Use it only as a bound: `fn f<T: Into<String>>(x: T)`
accepts `&str`, `String`, and anything convertible.

## Many `From` impls on one type

`From<T>` is a generic trait, so one type can implement it once per source type
(lesson 9.8 explains the mechanism).

```rust
#[derive(Debug)]
struct Id(u64);

impl From<u32> for Id { fn from(v: u32) -> Self { Id(v as u64) } }
impl From<u64> for Id { fn from(v: u64) -> Self { Id(v) } }
impl From<&str> for Id { fn from(v: &str) -> Self { Id(v.len() as u64) } }

println!("{:?}", Id::from(5u32));
println!("{:?}", Id::from(5u64));
println!("{:?}", Id::from("hello"));
let i: Id = 9u32.into();
println!("{:?}", i);
```

Output:

```
Id(5)
Id(5)
Id(5)
Id(9)
```

## `.into()` needs an inferable target

`.into()` has to know what to produce. A `let` annotation, a function parameter,
or a return type all work.

```rust
let s: String = "hi".into();
let s2 = String::from("hi");
let n: i64 = i32::MAX.into();
println!("{} {} {}", s, s2, n);
```

Output:

```
hi hi 2147483647
```

Without an annotation you get `E0283: type annotations needed`. Idiomatic: use
`.into()` where the target is obvious from context (struct literal field,
argument, `return`), and `T::from(x)` when it isn't.

Std ships `From` for every widening numeric conversion — `u8 -> u32`,
`i32 -> i64`, `u32 -> f64` — but **not** narrowing ones; those are `TryFrom`.

## TryFrom with an error type

An associated `Error` type and a `Result` return.

```rust
use std::fmt;

#[derive(Debug, PartialEq)]
struct Even(i64);

#[derive(Debug, PartialEq)]
struct NotEven(i64);

impl fmt::Display for NotEven {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} is not even", self.0)
    }
}
impl std::error::Error for NotEven {}

impl TryFrom<i64> for Even {
    type Error = NotEven;
    fn try_from(v: i64) -> Result<Self, Self::Error> {
        if v % 2 == 0 { Ok(Even(v)) } else { Err(NotEven(v)) }
    }
}

println!("{:?}", Even::try_from(4));
println!("{:?}", Even::try_from(5));
let r: Result<Even, _> = 8i64.try_into();
println!("{:?}", r);
match Even::try_from(7) {
    Ok(e) => println!("ok {:?}", e),
    Err(e) => println!("err {}", e),
}
```

Output:

```
Ok(Even(4))
Err(NotEven(5))
Ok(Even(8))
err 7 is not even
```

`TryInto` is the mirror blanket impl — same rule, never implement it yourself.
Implementing `Display + Error` on the error type makes it usable with `?` and
`Box<dyn Error>` (Module 08).

## TryFrom for validation

The real payoff: a newtype that **cannot** hold invalid data. Once you have a
`Username`, every function downstream can skip its checks.

```rust
#[derive(Debug, PartialEq)]
struct Username(String);

#[derive(Debug, PartialEq)]
enum BadName { Empty, TooLong(usize), BadChar(char) }

impl fmt::Display for BadName {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            BadName::Empty => write!(f, "username is empty"),
            BadName::TooLong(n) => write!(f, "username too long: {} chars", n),
            BadName::BadChar(c) => write!(f, "illegal character {:?}", c),
        }
    }
}

impl TryFrom<&str> for Username {
    type Error = BadName;
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        if s.is_empty() { return Err(BadName::Empty); }
        if s.len() > 12 { return Err(BadName::TooLong(s.len())); }
        if let Some(c) = s.chars().find(|c| !c.is_ascii_alphanumeric()) {
            return Err(BadName::BadChar(c));
        }
        Ok(Username(s.to_string()))
    }
}

for n in ["raj", "", "verylongusername", "raj-99"] {
    match Username::try_from(n) {
        Ok(u) => println!("ok {:?}", u),
        Err(e) => println!("rejected: {}", e),
    }
}
```

Output:

```
ok Username("raj")
rejected: username is empty
rejected: username too long: 16 chars
rejected: illegal character '-'
```

Keep the inner field private so the only way in is `try_from`.

## FromStr and `.parse()`

`.parse::<T>()` is `T::from_str` with a turbofish. Implement `FromStr` and your
type joins in.

```rust
use std::str::FromStr;

#[derive(Debug, PartialEq)]
struct Point { x: i32, y: i32 }

impl FromStr for Point {
    type Err = String;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let (a, b) = s.split_once(',').ok_or("expected 'x,y'")?;
        let x = a.trim().parse::<i32>().map_err(|e| e.to_string())?;
        let y = b.trim().parse::<i32>().map_err(|e| e.to_string())?;
        Ok(Point { x, y })
    }
}

println!("{:?}", "3,4".parse::<Point>());
println!("{:?}", " 10 , -2 ".parse::<Point>());
println!("{:?}", "nope".parse::<Point>());
println!("{:?}", "a,4".parse::<Point>());
println!("{:?}", Point::from_str("1,1"));
```

Output:

```
Ok(Point { x: 3, y: 4 })
Ok(Point { x: 10, y: -2 })
Err("expected 'x,y'")
Err("invalid digit found in string")
Ok(Point { x: 1, y: 1 })
```

Note the associated type is spelled `Err`, not `Error` — `TryFrom` uses `Error`.
Easy typo.

## ToString via Display

Never implement `ToString`. Implement `Display` and the blanket
`impl<T: Display> ToString for T` does the rest.

```rust
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

println!("{}", Point { x: 1, y: 2 }.to_string());
```

Output:

```
(1, 2)
```

## AsRef and AsMut

Cheap reference-to-reference conversion. Use it to make a function accept any
string-ish or path-ish argument.

```rust
fn print_len(s: impl AsRef<str>) { println!("{} {}", s.as_ref().len(), s.as_ref()); }
fn open(p: impl AsRef<std::path::Path>) { println!("{:?}", p.as_ref().extension()); }
fn clear(mut v: impl AsMut<Vec<i32>>) { v.as_mut().clear(); }

print_len("str");
print_len(String::from("string"));
print_len(&String::from("ref"));
open("a.txt");
open(String::from("b.rs"));

let mut v = vec![1, 2, 3];
clear(&mut v);
println!("{:?}", v);
```

Output:

```
3 str
6 string
3 ref
Some("txt")
Some("rs")
[]
```

`AsRef<Path>` is the std convention for every file API. Unlike `Into<String>`,
`AsRef<str>` does not allocate.

## Borrow and ToOwned

`Borrow<T>` is `AsRef<T>` with an extra promise: the borrowed form hashes and
compares identically to the owner. That is what lets `HashMap<String, V>` be
looked up with a `&str`.

```rust
use std::borrow::Borrow;
use std::collections::HashMap;

#[derive(Debug, PartialEq, Eq, Hash)]
struct Key(String);

impl Borrow<str> for Key {
    fn borrow(&self) -> &str { &self.0 }
}

let mut m: HashMap<Key, i32> = HashMap::new();
m.insert(Key("a".into()), 1);
println!("{:?} {:?}", m.get("a"), m.get("z"));
```

Output:

```
Some(1) None
```

This only works because `Key`'s `Hash` is derived from the same `String` the
`borrow` returns. Breaking that invariant silently breaks lookups.

`ToOwned` is the reverse: borrowed to owned, generalizing `Clone`.

```rust
let owned: String = "slice".to_owned();
let back: &str = owned.as_str();
println!("{} {}", owned, back);
println!("{:?}", [1, 2, 3].to_owned());
```

Output:

```
slice slice
[1, 2, 3]
```

`str::to_owned() -> String` and `[T]::to_owned() -> Vec<T>` change the type;
`Clone` cannot. `Cow<str>` builds on it to defer allocation:

```rust
use std::borrow::Cow;

fn shout(s: &str) -> Cow<'_, str> {
    if s.contains('!') { Cow::Borrowed(s) } else { Cow::Owned(format!("{}!", s)) }
}

for s in ["hey", "hey!"] {
    let c = shout(s);
    println!("{} borrowed={}", c, matches!(c, Cow::Borrowed(_)));
}
```

Output:

```
hey! borrowed=false
hey! borrowed=true
```

## Numeric conversions: `as` vs `TryFrom`

`as` never fails — it **silently truncates or saturates**.

```rust
println!("{} {}", 300i32 as u8, -1i32 as u8);
println!("{} {}", 3.9f64 as i32, 1e20f64 as i32);
```

Output:

```
44 255
3 2147483647
```

`300 as u8` wrapped to 44, `-1 as u8` wrapped to 255, `3.9` truncated toward
zero, and `1e20` saturated at `i32::MAX`. None of that raised anything.

`TryFrom` tells you instead:

```rust
println!("{:?}", u8::try_from(300i32));
println!("{:?}", u8::try_from(200i32));
println!("{:?}", u8::try_from(-1i32));
println!("{:?}", i32::try_from(9_000_000_000i64));
println!("{}", u8::try_from(300i32).unwrap_or(255));
```

Output:

```
Err(TryFromIntError(PosOverflow))
Ok(200)
Err(TryFromIntError(NegOverflow))
Err(TryFromIntError(PosOverflow))
255
```

The rule: `as` only when you can prove the value fits, or you *want* truncation
(hashing, bit manipulation, `usize -> u64`). Any value from input, arithmetic,
or a config file goes through `TryFrom`. `i32 -> i64` and other widening
conversions can use `From`/`.into()`, which cannot fail at all.

## `From` powers the `?` operator

`?` calls `From::from` on the error. One `From` impl per source error and `?`
works everywhere.

```rust
#[derive(Debug)]
struct AppError(String);

impl From<std::num::ParseIntError> for AppError {
    fn from(e: std::num::ParseIntError) -> Self { AppError(format!("parse: {}", e)) }
}

fn double(s: &str) -> Result<i32, AppError> { Ok(s.parse::<i32>()? * 2) }

println!("{:?}", double("21"));
println!("{:?}", double("x"));
```

Output:

```
Ok(42)
Err(AppError("parse: invalid digit found in string"))
```

Module 08 covers this pattern in full.

## Which conversion trait to use

| Need | Trait | Signature |
|---|---|---|
| Infallible, owned to owned | `From<T>` | `fn from(T) -> Self` |
| Call site of the above | `Into<U>` (free) | `x.into()` |
| Fallible, owned to owned | `TryFrom<T>` | `fn try_from(T) -> Result<Self, Self::Error>` |
| Call site of the above | `TryInto<U>` (free) | `x.try_into()` |
| Parse from a string | `FromStr` | `fn from_str(&str) -> Result<Self, Self::Err>` |
| Render to a string | `Display` (gives `ToString`) | `fn fmt(...)` |
| Cheap reference to reference | `AsRef<T>` | `fn as_ref(&self) -> &T` |
| Same, mutable | `AsMut<T>` | `fn as_mut(&mut self) -> &mut T` |
| Reference usable as a map key | `Borrow<T>` | `fn borrow(&self) -> &T` |
| Borrowed to owned, type changes | `ToOwned` | `fn to_owned(&self) -> Self::Owned` |
| Narrowing a number | `TryFrom` | `u8::try_from(n)` |
| Widening a number | `From` | `n.into()` |
| Deliberate truncation | `as` | `n as u8` |
| Accept anything string-like, no alloc | `AsRef<str>` bound | `fn f(s: impl AsRef<str>)` |
| Accept anything string-like, will own | `Into<String>` bound | `fn f(s: impl Into<String>)` |

Rule of thumb: implement `From`, `TryFrom`, `FromStr`, `Display`, `AsRef`.
Never implement `Into`, `TryInto`, or `ToString`.

## Quick Reference

| Form | Syntax |
|---|---|
| Define infallible | `impl From<Src> for Dst { fn from(s: Src) -> Self }` |
| Use it | `Dst::from(s)` or `let d: Dst = s.into();` |
| As a bound | `fn f<T: Into<Dst>>(x: T)` |
| Define fallible | `impl TryFrom<Src> for Dst { type Error = E; fn try_from(...) }` |
| Use it | `Dst::try_from(s)?` or `let d: Dst = s.try_into()?;` |
| Define parsing | `impl FromStr for T { type Err = E; fn from_str(...) }` |
| Use it | `"...".parse::<T>()` |
| Get `.to_string()` | `impl Display for T` |
| Flexible argument | `fn f(s: impl AsRef<str>)` |
| Narrow safely | `u8::try_from(n)` |
| Widen | `let n: i64 = x.into();` |
| Truncate on purpose | `n as u8` |

## Common Mistakes

**Implementing `Into` instead of `From`.** `E0119`, conflicts with the std
blanket impl. Implement `From`.

**Using `.into()` with no inferable target.** `E0283`. Annotate, or use
`T::from(x)`.

**Reaching for `as` on untrusted numbers.** It truncates and saturates
silently. Use `TryFrom` and handle the `Err`.

**Writing `type Error` in `FromStr`.** It is `type Err`. `TryFrom` uses
`type Error`.

**Implementing `From` for a conversion that can fail.** There is no way to
report the failure — you end up panicking. Use `TryFrom`.

**Implementing `Borrow<T>` where the borrowed form hashes differently.** Map
lookups silently miss. Use `AsRef<T>` unless the hash/eq invariant holds.

## Exercises

1. Define `struct Meters(f64)` and `struct Feet(f64)` with `From` in both
   directions. Convert back and forth and check you land where you started.
2. Write `impl TryFrom<i32> for Port` rejecting anything outside `1..=65535`,
   with a `Display` error. Test `0`, `80`, `70000`.
3. Implement `FromStr` for `struct Rgb(u8, u8, u8)` parsing `#RRGGBB`, with
   `Display` printing it back. Round-trip `#FF8000`, and show what `#ZZ` gives.
4. Write `fn greet(name: impl AsRef<str>) -> String` and call it with `&str`,
   `String`, and `&String`. Then rewrite it with `impl Into<String>` and note the
   difference.
5. Write `fn sum_u8(v: &[i32]) -> Result<u8, std::num::TryFromIntError>` that
   sums as `i32` and converts the result with `TryFrom`. Test a vec that
   overflows.

---

[← 9.6 Operator Overloading and Standard Traits](06-std-traits.md) · [Course Index](../README.md) · [9.8 Associated Types and Generic Traits →](08-associated-types.md)
