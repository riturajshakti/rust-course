[← Course Index](../README.md)

# 4.3 — References and Borrowing Rules

## What You'll Learn

- `&T` as a read-only borrow, and how many you may hold
- `&T` parameters: the fix for every move error in lesson 4.2
- Returning references, and the two errors that stop you (`E0106`, `E0597`)
- References to fields, references to references, and deref with `*`

## Coming From Another Language

| Other | Rust |
|---|---|
| Go `*T` pointer, can be nil | `&T`, never null, never dangling |
| JS: every object access is a reference | Explicit `&` at the borrow and the call |
| C `const T*` | `&T` — but the compiler proves the pointee outlives it |

A reference is a non-owning, non-null pointer whose validity the compiler
verifies. `&T` grants read access only. Creating one is called **borrowing**:
the owner keeps the value, the borrow ends, nothing is dropped.

## Borrowing instead of moving

```rust
fn len_of(s: &String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("hello");
    println!("{}", len_of(&s));
    println!("{s}");
}
```

Output:

```
5
hello
```

`&s` at the call site, `&String` in the signature. `s` is still the owner, and
nothing was dropped when `len_of` returned.

## Many readers at once

Any number of `&T` may coexist, and the owner can still read through its own
name.

```rust
fn main() {
    let s = String::from("hello");
    let r1 = &s;
    let r2 = &s;
    let r3 = &s;
    println!("{r1} {r2} {r3} {s}");
}
```

Output:

```
hello hello hello hello
```

This is the first half of the borrowing rules: **many `&T`, or one `&mut T`,
never both.** Lesson 4.4 covers the `&mut` half.

## Dereferencing with *

`*r` gets the value behind the reference. Most of the time you don't need it —
method calls and operators auto-deref.

```rust
fn main() {
    let n = 42;
    let rn = &n;
    println!("{}", *rn + 1);
    println!("{}", rn + 1);
    println!("{}", *rn == 42);
}
```

Output:

```
43
43
true
```

`rn + 1` works because `Add` is implemented for `&i32`. `println!("{rn}")`
works because `Display` forwards through the reference. You need the explicit
`*` mainly when assigning through `&mut` (lesson 4.4).

## References to references

```rust
let n = 42;
let rn = &n;
let rr = &rn;
println!("{}", **rr);
```

Output:

```
42
```

`&&i32` is a real type; `**` peels both layers. You see this in closure
arguments — `filter` on an iterator of `&T` hands the predicate `&&T`.

## &T parameters replace every move

Each of these reads a value without taking it.

```rust
fn describe(s: &str) -> String { format!("{} chars", s.len()) }
fn total(v: &Vec<i32>) -> i32 { v.iter().sum() }
fn biggest(xs: &[i32]) -> i32 { *xs.iter().max().unwrap() }

fn main() {
    let s = String::from("hello");
    let v = vec![3, 9, 2];
    println!("{}", describe(&s));
    println!("{}", total(&v));
    println!("{}", biggest(&v));
    println!("{s} {:?}", v);
}
```

Output:

```
5 chars
14
9
hello [3, 9, 2]
```

Note `describe(&s)` passes a `String` to a `&str` parameter — deref coercion
handles it. Lesson 4.5 covers why `&str` and `&[T]` are the better parameter
types.

## A reference to a field

```rust
#[derive(Debug)]
struct User { name: String, age: u32 }

fn main() {
    let u = User { name: String::from("raj"), age: 30 };
    let nref = &u.name;
    let aref = &u.age;
    println!("{nref} {aref}");
    println!("{:?}", u);
}
```

Output:

```
raj 30
User { name: "raj", age: 30 }
```

Borrowing a field does not partially move anything — unlike `let n = u.name;`
from lesson 4.2.

## A reference into a collection

```rust
let v = vec![10, 20, 30];
let e = &v[1];
println!("{e} {}", v.len());
```

Output:

```
20 3
```

`v[1]` would move the element for a non-`Copy` type; `&v[1]` never does.

## Returning a reference tied to a parameter

Fine, as long as the returned reference borrows from an input.

```rust
#[derive(Debug)]
struct User { name: String, age: u32 }

fn first_char(s: &str) -> &str {
    &s[0..1]
}

fn name_of(u: &User) -> &String {
    &u.name
}

fn main() {
    let s = String::from("rust");
    println!("{}", first_char(&s));

    let u = User { name: String::from("raj"), age: 30 };
    let n = name_of(&u);
    println!("{n} {}", u.age);
}
```

Output:

```
r
raj 30
```

Both signatures work with no lifetime annotation because there is exactly one
input reference, so the compiler knows where the output borrows from. Lessons
4.6 and 4.7 make that rule explicit.

## error[E0106]: nothing to borrow from

Does not compile (intentionally):

```rust
fn dangle() -> &String {
    let s = String::from("gone");
    &s
}
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:16
  |
1 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime, but this is uncommon unless you're returning a borrowed value from a `const` or a `static`
  |
1 | fn dangle() -> &'static String {
  |                 +++++++
help: instead, you are more likely to want to return an owned value
  |
1 - fn dangle() -> &String {
1 + fn dangle() -> String {
  |
```

The second `help` is the real fix: return `String`, not `&String`. In another
language this would compile and hand you a dangling pointer.

## error[E0515]: returning a reference to a local

Same mistake with the lifetime present. Does not compile:

```rust
fn bad<'a>(a: &'a str, b: &'a str) -> &'a str {
    let joined = format!("{a}{b}");
    joined.as_str()
}
```

```
error[E0515]: cannot return value referencing local variable `joined`
 --> src/main.rs:3:5
  |
3 |     joined.as_str()
  |     ------^^^^^^^^^
  |     |
  |     returns a value referencing data owned by the current function
  |     `joined` is borrowed here
```

If a function creates the data, it must return it owned. Annotations cannot
extend a value's life.

## error[E0597]: does not live long enough

The reference outlives the value it points at. Does not compile:

```rust
fn main() {
    let r;
    {
        let s = String::from("short-lived");
        r = &s;
    }
    println!("{r}");
}
```

```
error[E0597]: `s` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let s = String::from("short-lived");
  |             - binding `s` declared here
5 |         r = &s;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `s` dropped here while still borrowed
7 |     println!("{r}");
  |                - borrow later used here
```

Three annotations to read: where it was borrowed, where the owner died, and
where you used the borrow anyway. The fix is to declare `s` in the outer scope,
or to store an owned `String`.

## error[E0505]: moving a borrowed value

A borrow also blocks a move. Does not compile:

```rust
let s = String::from("hello");
let r = &s;
let t = s;
println!("{r} {t}");
```

```
error[E0505]: cannot move out of `s` because it is borrowed
 --> src/main.rs:4:13
  |
2 |     let s = String::from("hello");
  |         - binding `s` declared here
3 |     let r = &s;
  |             -- borrow of `s` occurs here
4 |     let t = s;
  |             ^ move out of `s` occurs here
5 |     println!("{r} {t}");
  |                - borrow later used here
```

Moving would invalidate `r`. Drop the use of `r` after the move and it compiles
— borrows end at their last use (lesson 4.4).

## &T vs T as a parameter type

```rust
fn by_value(s: String) -> usize { s.len() }
fn by_ref(s: &String) -> usize { s.len() }

fn main() {
    let a = String::from("one");
    let b = String::from("two");
    println!("{}", by_value(a));
    println!("{}", by_ref(&b));
    println!("{b}");
}
```

Output:

```
3
3
two
```

| | `fn f(x: T)` | `fn f(x: &T)` |
|---|---|---|
| Caller after the call | value gone | still usable |
| Call site | `f(x)` | `f(&x)` |
| Copies the heap data | no (moves the handle) | no |
| Function may mutate | yes, with `mut x: T` | no |
| Function may store it | yes | only if lifetimes allow |
| Default choice | when you consume or transform | **everything else** |

Take `&T` unless the function needs to own the value.

## Methods take references too

`&self` is just `self: &Self`.

```rust
struct Rect { w: f64, h: f64 }

impl Rect {
    fn area(&self) -> f64 { self.w * self.h }
    fn scaled(&self, k: f64) -> Rect { Rect { w: self.w * k, h: self.h * k } }
    fn consume(self) -> f64 { self.w }
}

fn main() {
    let r = Rect { w: 2.0, h: 3.0 };
    println!("{}", r.area());
    println!("{}", r.scaled(2.0).area());
    println!("{}", r.area());
    println!("{}", r.consume());
}
```

Output:

```
6
24
6
2
```

`consume(self)` moves — `r` is unusable after that line. Lesson 5.2 covers
method receivers properly.

## Quick Reference

| Form | Meaning |
|---|---|
| `&x` | borrow `x` immutably |
| `&T` | type: shared reference to `T` |
| `*r` | the value behind `r` |
| `**rr` | through two layers |
| `&x.field` | borrow one field |
| `&v[i]` | borrow one element |
| `fn f(x: &T)` | read-only parameter |
| `fn f(x: &T) -> &U` | output borrows from `x` |
| `&self` | method borrows the receiver |

| Error | Cause | Fix |
|---|---|---|
| `E0106 missing lifetime specifier` | returned `&` with no input `&` | return owned |
| `E0515 cannot return value referencing local` | returned `&` to a local | return owned |
| `E0597 does not live long enough` | ref outlives the owner | widen the owner's scope |
| `E0505 cannot move out ... borrowed` | moved while borrowed | finish using the ref first |

## Common Mistakes

**Writing `&` only in the signature or only at the call site.** Both are
required; `f(x)` with `fn f(x: &T)` is `E0308 mismatched types`.

**Returning a reference to something built inside the function.** Return the
owned value instead; no annotation can rescue it.

**Using `&String` or `&Vec<T>` as a parameter type.** Prefer `&str` and `&[T]`
— they accept strictly more callers (lesson 4.5).

**Sprinkling `*` to silence errors.** Auto-deref handles method calls and most
operators; the explicit `*` is for assigning through `&mut`.

**Declaring the reference before the value it will point at.** Reorder so the
owner outlives the borrow.

## Exercises

1. Write `fn vowels(s: &str) -> usize` and call it with a `String`, a literal,
   and a slice of a `String`.
2. Take `fn longest(a: String, b: String) -> String` and convert it to take
   references. Read the error you get before adding the lifetime.
3. Hold three `&T` to the same `Vec`, print all of them, then push to the
   `Vec` after the last use. Explain why it compiles.
4. Reproduce `E0597` and fix it twice: once by moving the `let`, once by
   cloning.
5. Write a struct with two `String` fields and a method returning a reference
   to the longer one. No annotation should be needed — explain why.

---

[← Course Index](../README.md)
