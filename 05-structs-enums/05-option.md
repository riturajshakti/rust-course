[← Course Index](../README.md)

# 5.5 — Option: No More Null

## What You'll Learn

- Why Rust has no `null` and what `Option<T>` replaces it with
- `match`, `if let`, `let else`, and `?` on an `Option`
- The full combinator reference: `map`, `and_then`, `filter`, `ok_or`, `take`, …
- Where `Option` costs nothing, and what `unwrap` on `None` actually prints

## Coming From Another Language

| Other | Rust |
|---|---|
| `null` / `undefined` / `nil` / `None` | no such value exists |
| TS `string \| null` | `Option<String>` |
| TS `x?.len() ?? 0` | `x.map(\|s\| s.len()).unwrap_or(0)` |
| Go `v, ok := m[k]` | `let v = m.get(k);` returning `Option<&V>` |

Every reference in Rust is valid. "Might be absent" is encoded in the type, so
the compiler makes you handle it — there is no null-pointer dereference to find
at 3am.

## Option is just an enum

```rust
enum Option<T> {
    None,
    Some(T),
}
```

That is the whole definition. `Some` and `None` are in the prelude, so no import
and no `Option::` prefix is needed.

```rust
let some: Option<i32> = Some(5);
let none: Option<i32> = None;
println!("{:?} {:?}", some, none);
```

Output:

```
Some(5) None
```

`None` on its own has no `T` to infer, so it often needs an annotation:
`None::<i32>` or a typed binding.

## You cannot use it as the value

```rust
let x: Option<i32> = Some(5);
let y = x + 1;
```

Does not compile:

```
error[E0369]: cannot add `{integer}` to `Option<i32>`
 --> src/main.rs:3:15
  |
3 |     let y = x + 1;
  |             - ^ - {integer}
  |             |
  |             Option<i32>
  |
note: `Option<i32>` does not implement `Add<{integer}>`
```

The wrapper must come off explicitly. That is the point.

## Unwrapping with match

```rust
let c: Option<u16> = Some(8080);
match c {
    Some(p) if p < 1024 => println!("privileged {p}"),
    Some(p) => println!("port {p}"),
    None => println!("no port"),
}
```

Output:

```
port 8080
```

## if let and let else

`if let` for "do something only if present":

```rust
let name: Option<&str> = Some("raj");
if let Some(n) = name {
    println!("hello {n}");
}
```

Output:

```
hello raj
```

`let else` for "bail out if absent", keeping the happy path unindented:

```rust
fn greet(name: Option<&str>) {
    let Some(n) = name else {
        println!("anonymous");
        return;
    };
    println!("hello {n}");
}

greet(Some("raj"));
greet(None);
```

Output:

```
hello raj
anonymous
```

The `else` block must diverge — `return`, `break`, `continue`, or `panic!`.

## unwrap on None panics

```rust
let none: Option<i32> = None;
println!("{}", none.unwrap());
```

Panics:

```
thread 'main' (13801521) panicked at src/main.rs:3:25:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`expect` lets you say what was missing, which is the only reason to prefer it:

```rust
let cfg: Option<&str> = None;
println!("{}", cfg.expect("CONFIG_PATH must be set"));
```

Panics:

```
thread 'main' (13801610) panicked at src/main.rs:3:24:
CONFIG_PATH must be set
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Use `expect` over `unwrap` in anything you will read a stack trace from. Use
neither when a combinator will do.

## Checking without unwrapping

```rust
let some: Option<i32> = Some(5);
let none: Option<i32> = None;

println!("{} {} {} {}", some.is_some(), some.is_none(), none.is_some(), none.is_none());
println!("{} {}", some.is_some_and(|n| n > 3), some.is_some_and(|n| n > 10));
println!("{} {}", none.is_none_or(|n| n > 3), some.is_none_or(|n| n > 10));
```

Output:

```
true false false true
true false
true false
```

`is_some_and` replaces `opt.is_some() && opt.unwrap() > 3`.

## Defaults

```rust
println!("{} {}", some.unwrap_or(0), none.unwrap_or(0));
println!("{} {}", some.unwrap_or_else(|| 99), none.unwrap_or_else(|| 99));
println!("{} {}", some.unwrap_or_default(), none.unwrap_or_default());
```

Output:

```
5 0
5 99
5 0
```

`unwrap_or` evaluates its argument eagerly; `unwrap_or_else` only on `None`.
Use `_else` when the fallback is expensive.

## Transforming

```rust
println!("{:?} {:?}", some.map(|n| n * 2), none.map(|n| n * 2));
println!("{} {}", some.map_or(0, |n| n * 2), none.map_or(0, |n| n * 2));
println!("{} {}", some.map_or_else(|| -1, |n| n * 2), none.map_or_else(|| -1, |n| n * 2));
println!("{:?} {:?}",
    some.and_then(|n| if n > 3 { Some(n) } else { None }),
    none.and_then(Some));
println!("{:?} {:?}", some.filter(|n| *n % 2 == 1), some.filter(|n| *n % 2 == 0));
```

Output:

```
Some(10) None
10 0
10 -1
Some(5) None
Some(5) None
```

`map` wraps the result; `and_then` expects the closure to return an `Option`
already, so it flattens — it is the chainable `?.` of other languages.

## Combining

```rust
println!("{:?} {:?}", some.and(Some("x")), none.and(Some("x")));
println!("{:?} {:?}", some.or(Some(0)), none.or(Some(0)));
println!("{:?} {:?}", some.or_else(|| Some(0)), none.or_else(|| Some(0)));
println!("{:?} {:?} {:?}", some.xor(None), none.xor(Some(1)), some.xor(Some(1)));
println!("{:?} {:?}", some.zip(Some("a")), some.zip(None::<&str>));
```

Output:

```
Some("x") None
Some(5) Some(0)
Some(5) Some(0)
Some(5) Some(1) None
Some((5, "a")) None
```

`or` is the `??` operator. `xor` gives `Some` only if exactly one side is.
`zip` pairs two `Option`s into `Option<(A, B)>`.

## Converting to Result

```rust
println!("{:?} {:?}", some.ok_or("missing"), none.ok_or("missing"));
println!("{:?} {:?}", some.ok_or_else(|| "missing"), none.ok_or_else(|| "missing"));
```

Output:

```
Ok(5) Err("missing")
Ok(5) Err("missing")
```

This is how an absent value becomes a reportable error. Module 08 covers
`Result`; lesson 8.4 covers its own combinators.

## Borrowing the inside

`map` consumes the `Option`. When it owns a `String` and you only want to look,
borrow first.

```rust
let text: Option<String> = Some("hi".to_string());
println!("{:?}", text.as_ref().map(|s| s.len()));
println!("{:?}", text.as_deref());
println!("{:?}", text);
```

Output:

```
Some(2)
Some("hi")
Some("hi")
```

`as_ref` gives `Option<&String>`, `as_deref` gives `Option<&str>`, and `text` is
still usable. `as_deref().unwrap_or("default")` is the standard way to read an
optional string field.

```rust
let mut maybe: Option<i32> = Some(1);
if let Some(v) = maybe.as_mut() { *v += 10; }
println!("{:?}", maybe);
```

Output:

```
Some(11)
```

## Mutating in place

```rust
let mut t = Some(3);
println!("{:?} {:?}", t.take(), t);

let mut r = Some(3);
println!("{:?} {:?}", r.replace(9), r);

let mut i: Option<i32> = None;
let v = *i.insert(7);
println!("{} {:?}", v, i);

let mut g: Option<i32> = None;
let v = *g.get_or_insert(4);
println!("{} {:?}", v, g);
let v = *g.get_or_insert(100);
println!("{} {:?}", v, g);
```

Output:

```
Some(3) None
Some(3) Some(9)
7 Some(7)
4 Some(4)
4 Some(4)
```

`take` leaves `None` behind and hands you the old value — the standard way to
move a field out of a `&mut self`. `insert` and `get_or_insert` return `&mut T`,
so they must not be used inside the same expression that also reads the
`Option`; that is a borrow conflict (`E0502`).

## Unwrapping a layer

```rust
let refd: Option<&i32> = Some(&5);
println!("{:?} {:?}", refd.cloned(), refd.copied());

let nested: Option<Option<i32>> = Some(Some(1));
println!("{:?} {:?}", nested.flatten(), Some(None::<i32>).flatten());
```

Output:

```
Some(5) Some(5)
Some(1) None
```

`copied` for `Copy` types, `cloned` for the rest; both turn `Option<&T>` into
`Option<T>`.

## Option as an iterator

An `Option` iterates over zero or one item, which makes it compose with any
iterator chain.

```rust
println!("{:?}", some.iter().collect::<Vec<_>>());
println!("{:?}", none.iter().collect::<Vec<_>>());

let all = vec![Some(1), None, Some(3)];
println!("{:?}", all.iter().flatten().collect::<Vec<_>>());
println!("{:?}", all.iter().copied().collect::<Option<Vec<i32>>>());
```

Output:

```
[5]
[]
[1, 3]
None
```

`.flatten()` drops the `None`s. Collecting into `Option<Vec<T>>` is all-or-
nothing: one `None` and the whole thing is `None`.

## The ? operator on Option

Inside a function returning `Option`, `?` returns `None` early.

```rust
fn domain(email: &str) -> Option<&str> {
    let at = email.find('@')?;
    let rest = email.get(at + 1..)?;
    if rest.is_empty() { None } else { Some(rest) }
}

println!("{:?} {:?} {:?}", domain("a@b.com"), domain("nope"), domain("a@"));
```

Output:

```
Some("b.com") None None
```

Flatter than nested `match`, and it is the same `?` used for `Result` — but the
two do not mix in one function without a conversion.

## Where Option shows up

Almost every fallible lookup in the standard library.

```rust
use std::collections::HashMap;

let v = vec![10, 20, 30];
println!("{:?} {:?}", v.first(), v.get(9));
println!("{:?}", v.iter().max());
println!("{:?}", Vec::<i32>::new().iter().max());

let mut m = HashMap::new();
m.insert("a", 1);
println!("{:?} {:?}", m.get("a"), m.get("z"));

println!("{:?} {:?}", "42".parse::<i32>().ok(), "x".parse::<i32>().ok());

let c: Option<String> = None;
println!("{}", c.as_deref().unwrap_or("localhost"));
```

Output:

```
Some(10) None
Some(30)
None
Some(1) None
Some(42) None
localhost
```

`v.get(9)` returning `None` is why indexing out of bounds with `v[9]` is the
unsafe-feeling choice — it panics, `get` does not.

## Size and derives

```rust
use std::mem::size_of;

println!("{} {}", size_of::<i32>(), size_of::<Option<i32>>());
println!("{} {}", size_of::<String>(), size_of::<Option<String>>());
println!("{} {}", size_of::<&str>(), size_of::<Option<&str>>());
println!("{:?}", Option::<i32>::default());
println!("{}", Some(3) > None);

let mut v = vec![Some(3), None, Some(1)];
v.sort();
println!("{:?}", v);
```

Output:

```
4 8
24 24
16 16
None
true
[None, Some(1), Some(3)]
```

For a pointer-like `T`, `Option<T>` is **free** — niche optimization puts the
tag in the unused null value (lesson 5.4). `Option::default()` is `None`, and
`None` sorts before every `Some`.

## Quick Reference

| Method | Signature sketch | On `Some(5)` / `None` |
|---|---|---|
| `is_some` | `-> bool` | `true` / `false` |
| `is_none` | `-> bool` | `false` / `true` |
| `is_some_and(f)` | `-> bool` | `f(5)` / `false` |
| `is_none_or(f)` | `-> bool` | `f(5)` / `true` |
| `unwrap` | `-> T` | `5` / **panics** |
| `expect(msg)` | `-> T` | `5` / **panics with msg** |
| `unwrap_or(d)` | `-> T` | `5` / `d` |
| `unwrap_or_else(f)` | `-> T` | `5` / `f()` |
| `unwrap_or_default()` | `-> T where T: Default` | `5` / `T::default()` |
| `map(f)` | `-> Option<U>` | `Some(f(5))` / `None` |
| `map_or(d, f)` | `-> U` | `f(5)` / `d` |
| `map_or_else(d, f)` | `-> U` | `f(5)` / `d()` |
| `and_then(f)` | `-> Option<U>`, `f: T -> Option<U>` | `f(5)` / `None` |
| `filter(p)` | `-> Option<T>` | `Some(5)` if `p(&5)` / `None` |
| `and(o)` | `-> Option<U>` | `o` / `None` |
| `or(o)` | `-> Option<T>` | `Some(5)` / `o` |
| `or_else(f)` | `-> Option<T>` | `Some(5)` / `f()` |
| `xor(o)` | `-> Option<T>` | `Some` iff exactly one is `Some` |
| `zip(o)` | `-> Option<(T, U)>` | `Some((5, u))` / `None` |
| `ok_or(e)` | `-> Result<T, E>` | `Ok(5)` / `Err(e)` |
| `ok_or_else(f)` | `-> Result<T, E>` | `Ok(5)` / `Err(f())` |
| `as_ref` | `&Option<T> -> Option<&T>` | borrows, does not consume |
| `as_mut` | `&mut Option<T> -> Option<&mut T>` | mutate in place |
| `as_deref` | `-> Option<&T::Target>` | `Option<String>` -> `Option<&str>` |
| `take` | `&mut self -> Option<T>` | returns value, leaves `None` |
| `replace(v)` | `&mut self -> Option<T>` | returns old, stores `v` |
| `insert(v)` | `&mut self -> &mut T` | stores `v`, returns `&mut` to it |
| `get_or_insert(v)` | `&mut self -> &mut T` | fills only if `None` |
| `copied` / `cloned` | `Option<&T> -> Option<T>` | unwraps the reference |
| `flatten` | `Option<Option<T>> -> Option<T>` | one layer off |
| `iter` | `-> impl Iterator<Item = &T>` | one item / zero items |
| `?` | early-returns `None` | only in a fn returning `Option` |

## Common Mistakes

**Reaching for `unwrap` by default.** It is a `panic!` waiting for the input you
did not test. Use `unwrap_or`, `?`, or `match`; keep `expect` for genuine
invariants and give it a message.

**Writing `opt.is_some() && opt.unwrap() > 3`.** Use `is_some_and`.

**Using `map` when the closure returns an `Option`.** You get
`Option<Option<T>>`. Use `and_then`, or `.flatten()` after.

**Calling `.map()` on an `Option<String>` you still need.** `map` consumes it.
Call `.as_ref()` or `.as_deref()` first.

**Treating `Option` as an error type.** `None` carries no reason. If the caller
needs to know *why*, return `Result` (Module 08), or `ok_or` your way there.

**Expecting `Option<T>` to cost a pointer.** For references, `Box`, and `NonZero`
types it is free; for a plain `i32` it adds a word of tag.

## Exercises

1. Write `fn middle(v: &[i32]) -> Option<&i32>` returning the middle element,
   `None` for an empty slice. Test with 0, 1, and 5 elements.
2. Given `Option<String>`, print its length or `0`, without consuming it and
   without `unwrap`.
3. Chain `find_user(id) -> Option<User>` and `User::email -> Option<String>`
   into one function returning `Option<String>`. Use `and_then`, then rewrite
   it with `?`.
4. Write `fn parse_pair(s: &str) -> Option<(i32, i32)>` for `"3,4"`, using `?`
   and `zip`. Test `"3,4"`, `"3"`, and `"a,b"`.
5. Given `Vec<Option<i32>>`, produce the sum of the present values, and
   separately an `Option<Vec<i32>>` that is `None` if any entry is missing.
6. Take an `Option<String>` field out of a struct behind `&mut self` using
   `take`, and print the struct before and after.

---

[← Course Index](../README.md)
