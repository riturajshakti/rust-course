[← 4.1 Stack, Heap, and Why Ownership Exists](01-why-ownership.md) · [Course Index](../README.md) · [4.3 References and Borrowing Rules →](03-borrowing.md)

# 4.2 — Move Semantics and Copy Types

## What You'll Learn

- What `let b = a;` does when `a` owns a heap allocation
- Every shape of `error[E0382]` and how to read it
- Which types are `Copy` and why `String` can never be
- Returning ownership, partial moves, and `mem::swap` / `replace` / `take`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS/Python: `b = a` aliases the object | `b = a` **moves**; `a` is dead |
| Go: `b := a` copies the struct (shallow) | Moves, and the original is unusable |
| C++: copy constructor by default | Move by default, `.clone()` to copy |

Assignment of a non-`Copy` value transfers ownership. The bytes are copied (the
24-byte `String` handle), the heap buffer is **not**, and the source binding is
invalidated so that only one owner can free it.

## Move on assignment

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    println!("{s2}");
}
```

Output:

```
hello
```

The three words were copied into `s2`; the heap buffer was not touched. `s1` is
now invalid.

## error[E0382]: borrow of moved value

Does not compile (intentionally):

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{s1}");
```

```
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:4:16
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 |     println!("{s1}");
  |                ^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let s2 = s1.clone();
  |                ++++++++
```

Read it bottom-up: *value borrowed here after move* is the line that broke,
*value moved here* is the culprit, and the `note` names the type and the reason.
Every E0382 has this shape.

## Same error for Vec

Does not compile:

```rust
let v = vec![1, 2, 3];
let w = v;
println!("{:?}", v);
```

```
error[E0382]: borrow of moved value: `v`
 --> src/main.rs:4:22
  |
2 |     let v = vec![1, 2, 3];
  |         - move occurs because `v` has type `Vec<i32>`, which does not implement the `Copy` trait
3 |     let w = v;
  |             - value moved here
4 |     println!("{:?}", v);
  |                      ^ value borrowed here after move
```

`Vec<i32>` is not `Copy` even though `i32` is — the heap buffer is what matters.

## Same error for a struct

Does not compile:

```rust
#[derive(Debug)]
struct Config { name: String, port: u16 }

let c = Config { name: String::from("db"), port: 5432 };
let d = c;
println!("{:?}", c);
```

```
error[E0382]: borrow of moved value: `c`
 --> src/main.rs:7:22
  |
5 |     let c = Config { name: String::from("db"), port: 5432 };
  |         - move occurs because `c` has type `Config`, which does not implement the `Copy` trait
6 |     let d = c;
  |             - value moved here
7 |     println!("{:?}", c);
  |                      ^ value borrowed here after move
  |
note: if `Config` implemented `Clone`, you could clone the value
 --> src/main.rs:2:1
  |
2 | struct Config { name: String, port: u16 }
  | ^^^^^^^^^^^^^ consider implementing `Clone` for this type
...
6 |     let d = c;
  |             - you could clone this value
```

One non-`Copy` field makes the whole struct non-`Copy`.

## Move on function call

Passing by value moves the argument in.

Does not compile:

```rust
fn consume(s: String) {
    println!("got {s}");
}

fn main() {
    let s = String::from("hello");
    consume(s);
    println!("{}", s.len());
}
```

```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:8:20
  |
6 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
7 |     consume(s);
  |             - value moved here
8 |     println!("{}", s.len());
  |                    ^ value borrowed here after move
  |
note: consider changing this parameter type in function `consume` to borrow instead if owning the value isn't necessary
 --> src/main.rs:1:15
  |
1 | fn consume(s: String) {
  |    -------    ^^^^^^ this parameter takes ownership of the value
  |    |
  |    in this function
help: consider cloning the value if the performance cost is acceptable
  |
7 |     consume(s.clone());
  |              ++++++++
```

The compiler names both fixes: take `&String` instead (lesson 4.3), or clone.
Taking a reference is almost always the right one.

## Partial move of a field

Moving one field out leaves the struct *partially moved*: `Copy` fields still
read, the whole value does not.

Does not compile:

```rust
#[derive(Debug)]
struct Config { name: String, port: u16 }

let c = Config { name: String::from("db"), port: 5432 };
let n = c.name;
println!("{n} {}", c.port);
println!("{:?}", c);
```

```
error[E0382]: borrow of partially moved value: `c`
 --> src/main.rs:8:22
  |
6 |     let n = c.name;
  |             ------ value partially moved here
7 |     println!("{n} {}", c.port);
8 |     println!("{:?}", c);
  |                      ^ value borrowed here after partial move
  |
  = note: partial move occurs because `c.name` has type `String`, which does not implement the `Copy` trait
```

Note line 7 is fine — `c.port` is `u16`, which is `Copy`. Only the whole-value
use on line 8 fails.

Struct update syntax triggers the same thing:

```rust
let u2 = Config { port: 1, ..c };
println!("{:?}", c);
```

```
error[E0382]: borrow of partially moved value: `c`
 --> src/main.rs:7:22
  |
6 |     let u2 = Config { port: 1, ..c };
  |              ----------------------- value partially moved here
7 |     println!("{:?}", c);
  |                      ^ value borrowed here after partial move
  |
  = note: partial move occurs because `c.name` has type `String`, which does not implement the `Copy` trait
```

Use `..c.clone()` when you need `c` afterwards.

## Copy types: assignment duplicates

If a type is `Copy`, `let b = a;` copies and `a` stays valid.

```rust
fn main() {
    let a = 5i32;
    let b = a;
    println!("{a} {b}");

    let t = (1, 2.5, 'x', true);
    let u = t;
    println!("{:?} {:?}", t, u);

    let arr = [1, 2, 3];
    let arr2 = arr;
    println!("{:?} {:?}", arr, arr2);

    let r = "literal";
    let r2 = r;
    println!("{r} {r2}");
}
```

Output:

```
5 5
(1, 2.5, 'x', true) (1, 2.5, 'x', true)
[1, 2, 3] [1, 2, 3]
literal literal
```

## The Copy list

| `Copy` | Not `Copy` |
|---|---|
| all integers (`i8`..`i128`, `u8`..`u128`, `usize`, `isize`) | `String` |
| `f32`, `f64` | `Vec<T>` |
| `bool`, `char` | `Box<T>`, `Rc<T>`, `Arc<T>` |
| `&T` (shared references) | `&mut T` |
| `[T; N]` where `T: Copy` | `[T; N]` where `T` is not `Copy` |
| tuples of all-`Copy` types | tuples containing any non-`Copy` type |
| unit `()`, unit structs | `HashMap`, `BTreeMap`, `Option<String>` |
| structs/enums deriving `Copy` | anything implementing `Drop` |

The pattern: `Copy` means "a bitwise duplicate is a valid second value." That is
false the instant a heap pointer or a destructor is involved.

Note `&mut T` is *not* `Copy` — duplicating an exclusive reference would break
the one-writer rule (lesson 4.4).

## Tuples: Copy-ness is contagious

```rust
let t = (1, 2.5, 'x', true);
let u = t;
println!("{:?} {:?}", t, u);
```

Output:

```
(1, 2.5, 'x', true) (1, 2.5, 'x', true)
```

One `String` inside and the whole tuple moves. Does not compile:

```rust
let p = (1, String::from("owned"));
let q = p;
println!("{:?}", p);
```

```
error[E0382]: borrow of moved value: `p`
 --> src/main.rs:4:22
  |
2 |     let p = (1, String::from("owned"));
  |         - move occurs because `p` has type `(i32, String)`, which does not implement the `Copy` trait
3 |     let q = p;
  |             - value moved here
4 |     println!("{:?}", p);
  |                      ^ value borrowed here after move
```

## Why String can't be Copy

A bitwise duplicate of a `String` would give two owners of one buffer, and both
would free it at end of scope — a double free. `Copy` is therefore forbidden on
any type with a non-`Copy` field.

Does not compile:

```rust
#[derive(Copy, Clone)]
struct Wrong { name: String }
```

```
error[E0204]: the trait `Copy` cannot be implemented for this type
 --> src/main.rs:2:8
  |
1 | #[derive(Copy, Clone)]
  |          ---- in this derive macro expansion
2 | struct Wrong { name: String }
  |        ^^^^^   ------------ this field does not implement `Copy`
```

`Clone` is the opt-in, explicit version: it runs code (reallocating the buffer)
instead of copying bytes. Lesson 4.8 covers the pair.

## Returning ownership

A function can hand ownership out as easily as it takes it in.

```rust
fn make() -> String {
    String::from("built inside")
}

fn shout(mut s: String) -> String {
    s.push('!');
    s
}

fn main() {
    let s = make();
    let s = shout(s);
    let s = shout(s);
    println!("{s}");
}
```

Output:

```
built inside!!
```

`mut s: String` means the parameter binding is mutable — the caller's
`let` does not need `mut`, because ownership moved.

## Passing then returning

Before `&` existed in your toolkit, the workaround was to give the value back.
You will see this in older code and in builder APIs.

```rust
fn sum_back(v: Vec<i32>) -> (Vec<i32>, i32) {
    let t = v.iter().sum();
    (v, t)
}

fn main() {
    let v = vec![1, 2, 3];
    let (v, total) = sum_back(v);
    println!("{:?} {}", v, total);
}
```

Output:

```
[1, 2, 3] 6
```

Idiomatic Rust takes `&[i32]` and returns just the sum. Use ownership-returning
signatures only when the function genuinely transforms the value.

## mem::swap

Exchanges two values behind `&mut` without moving out of either.

```rust
use std::mem;

let mut a = String::from("alpha");
let mut b = String::from("beta");
mem::swap(&mut a, &mut b);
println!("{a} {b}");
```

Output:

```
beta alpha
```

## mem::replace

Puts a new value in and returns the old one.

```rust
let mut s = String::from("old");
let prev = mem::replace(&mut s, String::from("new"));
println!("prev={prev} now={s}");
```

Output:

```
prev=old now=new
```

## mem::take

`replace` with `T::default()` — the usual way to move a field out of a struct
you only have `&mut` access to.

```rust
let mut t = String::from("taken");
let got = mem::take(&mut t);
println!("got={got} left={t:?} len={}", t.len());

let mut v = vec![1, 2, 3];
let old = mem::take(&mut v);
println!("{:?} {:?}", old, v);
```

Output:

```
got=taken left="" len=0
[1, 2, 3] []
```

Requires `T: Default`. These three are the sanctioned escape hatch when you
need to move a value out from behind a reference — the borrow checker will not
let you do it directly.

## Quick Reference

| Operation | Effect on source |
|---|---|
| `let b = a;` (non-`Copy`) | moved, `a` unusable |
| `let b = a;` (`Copy`) | copied, `a` still usable |
| `f(a)` where `f(x: T)` | moved |
| `f(&a)` where `f(x: &T)` | borrowed, still usable |
| `let b = a.clone();` | untouched, new allocation |
| `let x = s.field;` (non-`Copy`) | `s` partially moved |
| `let x = s.field;` (`Copy`) | `s` untouched |
| `S { a: v, ..s }` | `s` partially moved |
| `mem::swap(&mut a, &mut b)` | both valid, contents exchanged |
| `mem::replace(&mut a, new)` | `a` holds `new`, old returned |
| `mem::take(&mut a)` | `a` holds `Default`, old returned |

| Error | Means |
|---|---|
| `E0382 borrow of moved value` | used after a move |
| `E0382 borrow of partially moved value` | used after one field moved out |
| `E0204 Copy cannot be implemented` | a field is not `Copy` |
| `E0505 cannot move out ... it is borrowed` | moved while a reference was live |

## Common Mistakes

**Passing by value when you only need to read.** Take `&T`; the compiler's own
`note` tells you so.

**Trying to move a field out from behind `&mut self`.** `self.data` alone is
`E0507: cannot move out of ... which is behind a mutable reference`. Use
`mem::take(&mut self.data)`.

**Assuming a struct of `Copy` fields is `Copy`.** You still have to
`#[derive(Copy, Clone)]`; it is never automatic.

**Reaching for the `(value, result)` return tuple.** That was the pre-reference
workaround. Take a reference instead.

## Exercises

1. Trigger `E0382` four ways: assignment, function call, field move, and
   struct update syntax. Read each diagnostic and name the fix.
2. Write `fn count_words(s: String) -> (String, usize)`, then rewrite it as
   `fn count_words(s: &str) -> usize`. Compare the call sites.
3. Which of these are `Copy`? `(u8, char)`, `[String; 2]`, `&Vec<i32>`,
   `&mut i32`, `Option<u8>`, `Option<String>`. Verify by assigning twice.
4. Add `#[derive(Copy, Clone)]` to a struct with a `String` field and read the
   `E0204` output. Then make it work by changing the field type.
5. Given `struct Buf { data: Vec<u8> }` and `fn drain(&mut self) -> Vec<u8>`,
   implement `drain` with `mem::take` so `self.data` ends up empty.

---

[← 4.1 Stack, Heap, and Why Ownership Exists](01-why-ownership.md) · [Course Index](../README.md) · [4.3 References and Borrowing Rules →](03-borrowing.md)
