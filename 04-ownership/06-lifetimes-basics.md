[← Course Index](../README.md)

# 4.6 — Lifetimes: The Basics

## What You'll Learn

- What a lifetime annotation does, and what it cannot do
- `longest()` built up from the actual compiler error
- Multiple lifetime parameters, and tying a return to one input
- Lifetimes are erased at compile time — zero runtime cost

## Coming From Another Language

There is no equivalent. A lifetime is a compile-time label naming a region of
code over which a reference is valid. You are not creating or extending
anything; you are **describing a relationship the compiler cannot infer** —
usually "the returned reference borrows from this parameter."

Every reference has a lifetime. You almost never write them, because elision
covers the common cases (lesson 4.7). You write one when the compiler says it
cannot tell which input an output borrows from.

## Why they exist

You already saw the failure in lesson 4.3: a reference outliving its owner.

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
```

Inside one function the compiler sees both scopes and figures it out. Across a
function boundary it only has the signature to work with — so the signature has
to carry the information.

## The error that forces the annotation

Does not compile (intentionally):

```rust
fn longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() { a } else { b }
}
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(a: &str, b: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

Read the `help` line: *the signature does not say whether it is borrowed from
`a` or `b`*. The body is irrelevant — the compiler checks callers against the
signature alone.

## The fix

Take the compiler's suggestion verbatim.

```rust
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}

fn main() {
    println!("{}", longest("abcd", "xy"));
    let s1 = String::from("long string is long");
    {
        let s2 = String::from("xyz");
        println!("{}", longest(s1.as_str(), s2.as_str()));
    }
}
```

Output:

```
abcd
long string is long
```

## Reading the syntax

| Piece | Meaning |
|---|---|
| `<'a>` | declares a lifetime parameter, like `<T>` declares a type |
| `&'a str` | a `&str` valid for at least `'a` |
| `-> &'a str` | the result is valid for `'a` |
| same `'a` on both inputs | `'a` is the **shorter** of the two actual lifetimes |

`'a` is not a duration. At each call site the compiler picks the largest region
satisfying every constraint — here, the overlap of `a` and `b`.

## What the annotation enforces at the call site

The annotation says the output may not outlive *either* input. Does not
compile:

```rust
fn main() {
    let s1 = String::from("long string is long");
    let result;
    {
        let s2 = String::from("xyz");
        result = longest(s1.as_str(), s2.as_str());
    }
    println!("{result}");
}
```

```
error[E0597]: `s2` does not live long enough
  --> src/main.rs:10:39
   |
 9 |         let s2 = String::from("xyz");
   |             -- binding `s2` declared here
10 |         result = longest(s1.as_str(), s2.as_str());
   |                                       ^^ borrowed value does not live long enough
11 |     }
   |     - `s2` dropped here while still borrowed
12 |     println!("{result}");
   |                ------ borrow later used here
```

The compiler is conservative — it does not know `s1` is longer and will be the
one returned. If you need the value past `s2`'s scope, produce an owned value:

```rust
fn main() {
    let s1 = String::from("abc");
    let r;
    {
        let s2 = String::from("xy");
        r = longest(&s1, &s2).to_string();
    }
    println!("{r}");
}
```

Output:

```
abc
```

## Tying the output to one specific input

If the result only ever comes from `a`, say so. `b` then gets its own
(elided) lifetime and may be shorter-lived.

```rust
fn first<'a>(a: &'a str, _b: &str) -> &'a str { a }

fn main() {
    let keep = String::from("keep");
    let r;
    {
        let temp = String::from("temp");
        r = first(&keep, &temp);
    }
    println!("{r}");
}
```

Output:

```
keep
```

The same program with `longest`'s signature would be `E0597`. Writing the
narrowest honest signature is what makes callers work.

## Multiple lifetime parameters

Declare as many as the relationships require.

```rust
fn announce<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    println!("comparing {x} and {y}");
    x
}

fn main() {
    println!("{}", announce("a", "bb"));
}
```

Output:

```
comparing a and bb
a
```

`'a` and `'b` are unrelated. `-> &'b str` here would be a compile error,
because the body returns `x`.

## A longer lifetime satisfies a shorter one

`'static` outlives every `'a`, so returning a literal from a function promising
`&'a str` is fine — you are giving the caller something *more* durable than it
asked for.

```rust
fn literal_out<'a>(_a: &'a str) -> &'a str { "neither" }

fn pick<'a>(flag: bool, a: &'a str, b: &'a str) -> &'a str {
    if flag { a } else { b }
}

fn main() {
    println!("{}", literal_out("x"));
    println!("{}", pick(true, "yes", "no"));
    println!("{}", pick(false, "yes", "no"));
}
```

Output:

```
neither
yes
no
```

Lifetimes are subtyping-aware: where `'a` is required, anything living at least
as long is accepted. The reverse is `E0597`.

## Returning a slice of an input

The most common reason you ever write a lifetime by hand is because the
function has two reference-shaped inputs and one of them is irrelevant.

```rust
fn prefix<'a>(s: &'a str, n: usize) -> &'a str {
    &s[..n.min(s.len())]
}

fn find_in<'a>(haystack: &'a [i32], target: i32) -> Option<&'a i32> {
    haystack.iter().find(|x| **x == target)
}

fn main() {
    println!("{:?}", prefix("hello", 3));
    println!("{:?}", prefix("hello", 0));
    println!("{:?}", prefix("hello", 999));
    let v = vec![1, 2, 3];
    println!("{:?}", find_in(&v, 2));
    println!("{:?}", find_in(&v, 9));
}
```

Output:

```
"hel"
""
"hello"
Some(2)
None
```

Both are fully elidable — `usize` and `i32` are not references, so there is
exactly one input lifetime and rule 2 applies (lesson 4.7). The annotations
above are what the compiler infers anyway. A lifetime inside `Option<&'a i32>`
works like any other.

## Lifetimes cannot extend anything

An annotation is a claim the compiler verifies, never a request. Does not
compile:

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

No `'a` makes this work. The function creates the data, so it must return
`String`.

## Lifetimes are compile-time only

They have no runtime representation. A `&'a str` and a `&'static str` are the
same 16 bytes, and `'a` appears nowhere in the generated code.

```rust
fn main() {
    println!("{}", std::mem::size_of::<&str>());
    println!("{}", std::mem::size_of::<&'static str>());
    println!("{}", std::mem::size_of::<&i32>());
}
```

Output:

```
16
16
8
```

Borrow checking happens entirely at build time and is then discarded. There is
no cost to annotating heavily.

## Generics and lifetimes together

Lifetimes come first in the parameter list.

```rust
use std::fmt::Display;

fn longest_with<'a, T: Display>(a: &'a str, b: &'a str, note: T) -> &'a str {
    println!("note: {note}");
    if a.len() > b.len() { a } else { b }
}

fn main() {
    println!("{}", longest_with("abcd", "xy", 42));
    println!("{}", longest_with("a", "bbb", "tag"));
}
```

Output:

```
note: 42
abcd
note: tag
bbb
```

A generic function returning a reference into a generic slice:

```rust
fn largest<'a, T: PartialOrd>(items: &'a [T]) -> &'a T {
    let mut best = &items[0];
    for i in items {
        if i > best { best = i; }
    }
    best
}

fn main() {
    println!("{}", largest(&[3, 9, 2]));
    println!("{}", largest(&["a", "zz", "m"]));
}
```

Output:

```
9
zz
```

That signature is fully elidable — `fn largest<T: PartialOrd>(items: &[T]) -> &T`
compiles identically. Lesson 4.7 explains which annotations you can drop.

## Lifetime bounds on generics

`T: 'a` means "every reference inside `T` lives at least as long as `'a`".

```rust
#[derive(Debug)]
struct Wrapper<'a, T: 'a> { value: &'a T }

fn main() {
    let n = 5;
    let w = Wrapper { value: &n };
    println!("{:?}", w);
    println!("{}", *w.value + 1);
}
```

Output:

```
Wrapper { value: 5 }
6
```

The `T: 'a` bound is inferred here, so you can write
`struct Wrapper<'a, T> { value: &'a T }`. You need it explicitly only in more
tangled generic code.

## Quick Reference

| Syntax | Meaning |
|---|---|
| `fn f<'a>(x: &'a T)` | declare and use a lifetime |
| `fn f<'a>(x: &'a T) -> &'a T` | output borrows from `x` |
| `fn f<'a>(a: &'a T, b: &'a T) -> &'a T` | output tied to the shorter of both |
| `fn f<'a>(a: &'a T, b: &T) -> &'a T` | output tied to `a` only |
| `fn f<'a, 'b>(a: &'a T, b: &'b T)` | two independent lifetimes |
| `fn f<'a, T: Display>(...)` | lifetimes before type params |
| `T: 'a` | all references in `T` outlive `'a` |
| `'static` | the whole program (lesson 4.7) |

| Error | Means |
|---|---|
| `E0106 missing lifetime specifier` | ambiguous which input the output borrows from |
| `E0597 does not live long enough` | a caller violated the signature's constraint |
| `E0515 cannot return value referencing local` | the data is owned by the function |

## Common Mistakes

**Reading `'a` as "this lives longer".** It names a region; it changes nothing
about when values drop.

**Annotating to silence `E0515`.** If the function created the data, return it
owned. No lifetime fixes that.

**Putting the same `'a` on every reference by reflex.** That over-constrains
callers. Give the output the lifetime of only the input it actually borrows
from.

**Thinking annotations cost performance.** They are erased; a `&'a T` is the
same machine pointer.

**Adding lifetimes before reading the error.** Most signatures need none.
Write it without, compile, and let `E0106` tell you which one is missing.

## Exercises

1. Write `longest` without annotations, read `E0106`, then apply the fix the
   compiler prints.
2. Write `fn pick<'a>(flag: bool, a: &'a str, b: &'a str) -> &'a str`. Then try
   returning a literal `"neither"` from it and explain why that compiles.
3. Write `fn prefix<'a>(s: &'a str, n: usize) -> &'a str` returning the first
   `n` bytes, clamped to the length. Verify with `n = 0` and `n = 999`.
4. Write a two-lifetime function returning a reference from the second
   parameter, then try changing the return to the first and read the error.
5. Write `fn min_max<'a, T: PartialOrd>(xs: &'a [T]) -> (&'a T, &'a T)`. Then
   remove every lifetime annotation and confirm it still compiles.

---

[← Course Index](../README.md)
