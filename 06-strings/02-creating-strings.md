[← 6.1 String vs &str](01-string-vs-str.md) · [Course Index](../README.md) · [6.3 String Operations Reference →](03-string-operations.md)

# 6.2 — Creating and Growing Strings

## What You'll Learn

- Every constructor: `new`, `from`, `with_capacity`, `from_utf8`, `from_iter`
- Growing: `push`, `push_str`, `insert`, `insert_str`, `extend`, `+`, `format!`
- Joining collections: `concat`, `join`, `repeat`
- How `capacity` grows, and when to `reserve` or `shrink_to_fit`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `s += x` in a loop | `s.push_str(x)` |
| JS `arr.join("-")` | `arr.join("-")` — same name |
| Python `"".join(parts)` | `parts.concat()` |
| Python `"ab" * 3` | `"ab".repeat(3)` |
| Go `strings.Builder` | `String` itself — it *is* the builder |

There is no separate builder type. `String` already amortizes growth, so
`push_str` in a loop is the idiomatic fast path.

## `String::new` and `String::from`

```rust
let e = String::new();
println!("[{}] len={} cap={} empty={}", e, e.len(), e.capacity(), e.is_empty());

let a = String::from("hello");
println!("{} len={} cap={}", a, a.len(), a.capacity());
```

Output:

```
[] len=0 cap=0 empty=true
hello len=5 cap=5
```

`String::new()` allocates nothing — capacity 0. `String::from` allocates exactly
what it needs.

## `String::with_capacity`

Pre-allocates the buffer without changing the length.

```rust
let b = String::with_capacity(10);
println!("[{}] len={} cap={}", b, b.len(), b.capacity());
```

Output:

```
[] len=0 cap=10
```

Use it when you know roughly how many bytes you'll write. It saves reallocations,
not correctness.

## From bytes: `from_utf8` and `from_utf8_lossy`

`from_utf8` takes a `Vec<u8>` and **validates**, returning a `Result`.

```rust
let bytes = vec![104, 105, 33];
println!("{:?}", String::from_utf8(bytes));

let bad = vec![0x68, 0xFF, 0x69];
println!("{:?}", String::from_utf8(bad.clone()));
```

Output:

```
Ok("hi!")
Err(FromUtf8Error { bytes: [104, 255, 105], error: Utf8Error { valid_up_to: 1, error_len: Some(1) } })
```

The error carries the original bytes back (nothing is lost) plus `valid_up_to`,
so you know exactly where decoding failed.

`from_utf8_lossy` takes a `&[u8]`, never fails, and substitutes U+FFFD for each
bad sequence.

```rust
let bad = vec![0x68, 0xFF, 0x69];
println!("{}", String::from_utf8_lossy(&bad));
println!("{:?}", String::from_utf8_lossy(&bad));
```

Output:

```
h�i
"h�i"
```

It returns a `Cow<str>` — borrowed when the input was already valid, owned only
when a replacement happened. Call `.into_owned()` if you need a real `String`.

```rust
let crab = vec![0xF0, 0x9F, 0xA6, 0x80];
println!("{}", String::from_utf8(crab).unwrap());
```

Output:

```
🦀
```

## `from_iter` and `collect`

```rust
let fi = String::from_iter(['a', 'b', 'c']);
let fi2 = String::from_iter(["ab", "cd"]);
let fi3: String = vec!["x", "y"].into_iter().collect();
println!("{} {} {}", fi, fi2, fi3);
```

Output:

```
abc abcd xy
```

Works for iterators of `char`, `&str`, `String`, and `Box<str>`. `collect()` is
the same operation spelled as a method.

## `push` and `push_str`

`push` takes one `char`; `push_str` takes a `&str`.

```rust
let mut s = String::new();
s.push('H');
s.push_str("ello");
s.push(' ');
s.push_str("world");
println!("{}", s);
```

Output:

```
Hello world
```

Both append at the end and both amortize to O(1).

## `insert` and `insert_str`

Both take a **byte** index and shift everything after it — O(n).

```rust
let mut t = String::from("world");
t.insert(0, '!');
println!("{}", t);
t.insert_str(0, "hello ");
println!("{}", t);
t.insert_str(t.len(), " end");
println!("{}", t);
```

Output:

```
!world
hello !world
hello !world end
```

The index must land on a char boundary or it panics — see lesson 6.5.

## The `+` operator

`+` is `Add<&str> for String`: it **takes `self` by value** and returns it, so
no new allocation happens if capacity allows.

```rust
let a = String::from("Hello, ");
let b = String::from("world!");
let c = a + &b;
println!("{}", c);

let d = c + "!!" + &b;
println!("{}", d);
```

Output:

```
Hello, world!
Hello, world!!!world!
```

Left side must be an owned `String`, right side a `&str`. Chaining works because
each `+` hands the same buffer along.

Two `String`s do not add:

```rust
let c = a + b;
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:4:17
  |
4 |     let c = a + b;
  |                 ^ expected `&str`, found `String`
  |
help: consider borrowing here
  |
4 |     let c = a + &b;
  |                 +
```

And because it consumes the left operand, the left operand is gone:

```rust
let c = a + &b;
println!("{} {}", a, c);
```

Does not compile:

```
error[E0382]: borrow of moved value: `a`
 --> src/main.rs:5:23
  |
4 |     let c = a + &b;
  |             - value moved here
5 |     println!("{} {}", a, c);
  |                       ^ value borrowed here after move
```

Two literals never add — neither side is owned:

```rust
let c = "a" + "b";
```

Does not compile:

```
error[E0369]: cannot add `&str` to `&str`
 --> src/main.rs:2:17
  |
2 |     let c = "a" + "b";
  |             --- ^ --- &str
  |             |   |
  |             |   `+` cannot be used to concatenate two `&str` strings
  |             &str
  |
  = note: string concatenation requires an owned `String` on the left
help: create an owned `String` from a string reference
  |
2 |     let c = "a".to_owned() + "b";
  |                +++++++++++
```

Idiomatic choice: use `format!` for more than two pieces. `+` is only pleasant
for exactly one concatenation.

## `format!`

```rust
let n = 5;
let f = format!("{}-{}-{:?}", "a", n, vec![1, 2]);
println!("{}", f);
```

Output:

```
a-5-[1, 2]
```

Allocates a fresh `String`. Lesson 6.6 covers the whole format spec.

## `extend`

Appends anything iterable over `char`, `&str`, or `String`.

```rust
let mut ex = String::from("ab");
ex.extend(['c', 'd']);
ex.extend(["ef", "gh"]);
ex.extend("ij".chars());
println!("{}", ex);
```

Output:

```
abcdefghij
```

## `concat` and `join` on slices

These are slice methods, not `String` methods.

```rust
let parts = ["a", "b", "c"];
println!("{}", parts.concat());
println!("{}", parts.join("-"));
println!("{}", parts.join(""));

let owned = vec![String::from("x"), String::from("y")];
println!("{}", owned.join(", "));

let nested = [["a", "b"], ["c", "d"]];
println!("{:?}", nested.concat());
```

Output:

```
abc
a-b-c
abc
x, y
["a", "b", "c", "d"]
```

`concat()` on a slice of slices flattens one level — it returns a `Vec`, not a
`String`. On a slice of string-like items it returns a `String`.

## `repeat`

```rust
println!("{}", "ab".repeat(3));
println!("{}", "-".repeat(10));
println!("[{}]", "x".repeat(0));
println!("{}", String::from("hi").repeat(2));
```

Output:

```
ababab
----------
[]
hihi
```

`repeat(0)` gives an empty string. A huge count panics on capacity overflow.

## Building in a loop

Four idioms, all correct. Pick by readability.

```rust
let mut out = String::new();
for i in 1..=3 {
    out.push_str(&format!("{i}:"));
}
println!("{}", out);
```

Output:

```
1:2:3:
```

`write!` into a `String` avoids the intermediate allocation that `format!` makes.
It needs `use std::fmt::Write`.

```rust
use std::fmt::Write;

let mut w = String::new();
for i in 1..=3 {
    write!(w, "{i};").unwrap();
}
println!("{}", w);
```

Output:

```
1;2;3;
```

Iterator pipelines:

```rust
let built: String = (1..=5).map(|n| n.to_string()).collect::<Vec<_>>().join("+");
println!("{}", built);

let rev: String = "hello".chars().rev().collect();
println!("{}", rev);

let folded = (1..=4).fold(String::new(), |mut acc, n| {
    acc.push_str(&n.to_string());
    acc
});
println!("{}", folded);
```

Output:

```
1+2+3+4+5
olleh
1234
```

## Capacity growth and reallocation

Capacity roughly doubles, starting at 8.

```rust
let mut s = String::new();
let mut last = s.capacity();
println!("len={} cap={}", s.len(), s.capacity());
for i in 0..40 {
    s.push('x');
    if s.capacity() != last {
        last = s.capacity();
        println!("after push {}: len={} cap={}", i + 1, s.len(), s.capacity());
    }
}
```

Output:

```
len=0 cap=0
after push 1: len=1 cap=8
after push 9: len=9 cap=16
after push 17: len=17 cap=32
after push 33: len=33 cap=64
```

Each jump is a fresh allocation plus a byte copy. Four reallocations for 40
bytes; `with_capacity(40)` would have been zero.

## `reserve`, `reserve_exact`, `shrink_to_fit`, `shrink_to`

```rust
let mut r = String::with_capacity(4);
println!("cap={}", r.capacity());
r.reserve(100);
println!("after reserve(100): len={} cap={}", r.len(), r.capacity());
r.push_str("abc");
println!("len={} cap={}", r.len(), r.capacity());
r.shrink_to_fit();
println!("after shrink_to_fit: len={} cap={}", r.len(), r.capacity());
```

Output:

```
cap=4
after reserve(100): len=0 cap=100
len=3 cap=100
after shrink_to_fit: len=3 cap=3
```

`reserve(n)` guarantees room for `n` **more** bytes beyond the current length —
it is not "set capacity to n".

```rust
let mut e = String::with_capacity(50);
e.push_str("hello");
e.shrink_to(20);
println!("after shrink_to(20): len={} cap={}", e.len(), e.capacity());

let mut x = String::from("abc");
x.reserve_exact(10);
println!("reserve_exact: cap={}", x.capacity());
x.clear();
println!("after clear: len={} cap={}", x.len(), x.capacity());
```

Output:

```
after shrink_to(20): len=5 cap=20
reserve_exact: cap=13
after clear: len=0 cap=13
```

`reserve_exact(10)` on a 3-byte string gives 13, not 10 — again, it's "room for
10 more". `clear()` drops the length but **keeps the buffer**, which is why
reusing one `String` across loop iterations is cheap.

## Quick Reference

| Call | Returns | Note |
|---|---|---|
| `String::new()` | `String` | No allocation, cap 0 |
| `String::from(s)` | `String` | From `&str`, `char`, `Box<str>`, `Cow<str>` |
| `String::with_capacity(n)` | `String` | Empty, cap ≥ n |
| `String::from_utf8(vec)` | `Result<String, FromUtf8Error>` | Validates; returns bytes on error |
| `String::from_utf8_lossy(&[u8])` | `Cow<str>` | Replaces bad bytes with U+FFFD |
| `String::from_iter(it)` | `String` | Over `char`/`&str`/`String` |
| `s.push(c)` | `()` | One `char`, appended |
| `s.push_str(x)` | `()` | A `&str`, appended |
| `s.insert(i, c)` | `()` | Byte index; O(n); panics off-boundary |
| `s.insert_str(i, x)` | `()` | Byte index; O(n) |
| `s.extend(it)` | `()` | Append from any matching iterator |
| `a + &b` | `String` | Consumes `a`; left must be owned |
| `format!(..)` | `String` | New allocation |
| `write!(&mut s, ..)` | `fmt::Result` | Needs `use std::fmt::Write` |
| `slice.concat()` | `String`/`Vec` | No separator |
| `slice.join(sep)` | `String`/`Vec` | With separator |
| `s.repeat(n)` | `String` | `n == 0` gives `""` |
| `s.capacity()` | `usize` | Bytes allocated |
| `s.reserve(n)` | `()` | Room for n **more** bytes |
| `s.reserve_exact(n)` | `()` | Same, minimal over-allocation |
| `s.shrink_to_fit()` | `()` | Capacity down to len |
| `s.shrink_to(n)` | `()` | Capacity down to max(len, n) |

## Common Mistakes

**`a + b` with two `String`s.** Right side must be `&str`: write `a + &b`.

**Using `a` after `a + &b`.** `+` moves the left operand. Use `format!` or
clone.

**Reading `reserve(n)` as "capacity becomes n".** It reserves `n` bytes *beyond*
the current length.

**Calling `clear()` to free memory.** It only resets the length; use
`shrink_to_fit()` after, or drop the `String`.

**`insert` at a character position.** Indices are bytes. `s.insert(1, 'x')` on
`"é..."` panics; see lesson 6.5.

## Exercises

1. Build `"1,2,3,...,10"` three ways: a `push_str` loop, `write!`, and
   `map` + `join`. Confirm all three are equal with `assert_eq!`.
2. Write a loop that pushes 1000 chars onto a `String::new()` and prints every
   capacity change. Repeat with `with_capacity(1000)` and compare the counts.
3. Turn `vec![0x52, 0x75, 0x73, 0x74, 0xFF]` into a `String` twice — once with
   `from_utf8` (handle the `Err`) and once with `from_utf8_lossy`. Print both
   results and the `valid_up_to` value.
4. Write `fn banner(title: &str) -> String` producing the title with a row of
   `=` above and below, each exactly as wide as the title. Use `repeat`.
5. Write `fn csv_line(fields: &[&str]) -> String` using `join`, then a second
   version using `push_str` with no trailing comma. Verify they match.

---

[← 6.1 String vs &str](01-string-vs-str.md) · [Course Index](../README.md) · [6.3 String Operations Reference →](03-string-operations.md)
