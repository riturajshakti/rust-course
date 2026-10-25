[← Course Index](../README.md)

# 6.1 — String vs &str

## What You'll Learn

- Why Rust has two string types and what each one owns
- The memory layout of `String`, `&str`, `Box<str>`, and `Rc<str>`
- Deref coercion, and why `&String` parameters are a mistake
- `String::from` vs `.to_string()` vs `.to_owned()` vs `.into()`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `string` (immutable, GC'd) | `String` (owned, growable) **or** `&str` (borrowed view) |
| Python `str` | `String` for owned, `&str` for a slice |
| Go `string` (immutable header) | `&str` — same shape: pointer + length |
| Go `[]byte` built with `append` | `String` |

Rust splits what every other language merges: **who owns the bytes** is part of
the type. `String` owns a heap buffer it can grow. `&str` is a borrowed window
onto bytes someone else owns. Both are guaranteed valid UTF-8.

## The two types side by side

```rust
let lit: &'static str = "hello";
let owned: String = String::from("hello");

println!("{} {}", lit, owned);
println!("{} {}", lit.len(), owned.len());
```

Output:

```
hello hello
5 5
```

They print the same and measure the same. The difference is what happens at the
end of scope: `owned` frees a heap allocation, `lit` frees nothing.

## Memory layout

```rust
println!("{}", std::mem::size_of::<&str>());
println!("{}", std::mem::size_of::<String>());
println!("{}", std::mem::size_of::<Box<str>>());
println!("{}", std::mem::size_of::<std::rc::Rc<str>>());
println!("{}", std::mem::size_of::<&String>());
```

Output:

```
16
24
16
16
8
```

- `&str` is a **fat pointer**: `(ptr, len)` — 16 bytes, no allocation of its own.
- `String` is `(ptr, len, capacity)` — 24 bytes, pointing at a heap buffer.
- `&String` is a thin pointer to that 24-byte struct — one extra hop.

That extra field is the whole story: `capacity` is what lets `String` grow.
`&str` has nowhere to record a new capacity, so it can never grow.

## String literals are `&'static str`

A literal lives in the compiled binary's read-only data, so a reference to it is
valid for the entire program — that's what `'static` means here.

```rust
let s = "abcdef";
println!("{}", std::mem::size_of_val(s));
println!("{:?}", "abc".as_ptr() == "abc".as_ptr());
```

Output:

```
6
true
```

Identical literals are typically deduplicated into one address. Nothing is
allocated at runtime; `"abcdef"` costs zero instructions to "create".

## `&str` cannot be mutated or grown

```rust
let s: &str = "hello";
s.push_str(" world");
```

Does not compile:

```
error[E0599]: no method named `push_str` found for reference `&str` in the current scope
 --> src/main.rs:3:7
  |
3 |     s.push_str(" world");
  |       ^^^^^^^^ method not found in `&str`
```

`push_str` exists only on `String`. Any method that changes length lives on the
owned type.

## Deref coercion: `&String` becomes `&str` automatically

`String` implements `Deref<Target = str>`, so the compiler inserts the
conversion at call sites.

```rust
fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}

let s = String::from("world");
println!("{}", greet(&s));
println!("{}", greet("literal"));
println!("{}", greet(&s[..3]));
println!("{}", greet(s.as_str()));
```

Output:

```
Hello, world!
Hello, literal!
Hello, wor!
Hello, world!
```

One signature, four kinds of argument. `&s` and `s.as_str()` are the same thing;
`&s` is what people write.

## Why `&String` parameters are wrong

Coercion only runs in that direction. A `&String` parameter rejects everything
that isn't already a `String`.

```rust
fn greet(name: &String) -> String {
    format!("Hello, {name}!")
}

println!("{}", greet("literal"));
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:4:26
  |
4 |     println!("{}", greet("literal"));
  |                    ----- ^^^^^^^^^ expected `&String`, found `&str`
  |                    |
  |                    arguments to this function are incorrect
  |
  = note: expected reference `&String`
             found reference `&'static str`
```

You have thrown away literals, slices, and `Box<str>` in exchange for nothing.
Use `&str`. The only reason to take `&String` is if you need its `capacity`,
which is essentially never.

## Function signature guidance

| You want to | Take | Why |
|---|---|---|
| Read the text | `&str` | Accepts `String`, literals, slices |
| Mutate the caller's string | `&mut String` | Needs capacity to grow |
| Store/consume the text | `String` | Caller gives up ownership, no clone |
| Accept either, clone only if needed | `impl Into<String>` | Caller passes `&str` or `String` |
| Return borrowed text | `&str` tied to an input lifetime | No allocation |
| Return new text | `String` | Nothing to borrow from |

```rust
fn shout(s: &mut String) { s.push('!'); }
fn count(s: &str) -> usize { s.len() }

let mut s = String::from("hi");
shout(&mut s);
println!("{}", s);
println!("{}", count(&s));
```

Output:

```
hi!
3
```

## Returning a `&str` to a local fails

```rust
fn make() -> &str {
    let s = String::from("temp");
    &s
}
```

Does not compile:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:14
  |
1 | fn make() -> &str {
  |              ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: instead, you are more likely to want to return an owned value
  |
1 - fn make() -> &str {
1 + fn make() -> String {
  |
```

Take the second suggestion. Return `String` when you built the text yourself.

## Five ways to get a `String` from a `&str`

```rust
let a = String::from("x");
let b = "x".to_string();
let c = "x".to_owned();
let d: String = "x".into();
let e = String::from_iter(["x"]);

println!("{} {} {} {} {}", a, b, c, d, e);
println!("{}", a == b && b == c && c == d && d == e);
```

Output:

```
x x x x x
true
```

All five allocate and copy the same bytes. They compile to identical code.

| Form | Comes from | Use when |
|---|---|---|
| `String::from(s)` | `From<&str>` | Explicit and obvious; fine anywhere |
| `s.to_string()` | `Display` | Most common; works on *any* `Display` type, e.g. `42.to_string()` |
| `s.to_owned()` | `ToOwned` | Signals "borrowed → owned"; preferred by some lints |
| `s.into()` | `Into<String>` | When the target type is already inferred from context |
| `format!("{s}")` | macro | Only when you are also interpolating |

Pick one and be consistent. `.to_string()` is the default in most codebases;
`.into()` reads best in struct literals where the field type is known.

## `&str` in a struct drags a lifetime

```rust
struct Config { name: &str }
```

Does not compile:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:23
  |
1 | struct Config { name: &str }
  |                       ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 | struct Config<'a> { name: &'a str }
  |              ++++          ++
```

Both forms work, but they cost differently.

```rust
#[derive(Debug)]
struct Owned { name: String }

#[derive(Debug)]
struct Borrowed<'a> { name: &'a str }

let o = Owned { name: "raj".to_string() };
let b = Borrowed { name: "raj" };
println!("{:?} {:?}", o, b);
```

Output:

```
Owned { name: "raj" } Borrowed { name: "raj" }
```

`Borrowed<'a>` now infects every function, trait impl, and collection that
touches it, and can never outlive its source. Store `String` by default; reach
for `&'a str` only for short-lived parser/view types. Lesson 4.7 covers the
lifetime rules.

## `Box<str>` and `Rc<str>`

`Box<str>` is an owned string with **no spare capacity** — 16 bytes instead of
24, at the cost of never growing. Useful for strings you build once and keep by
the million.

```rust
let boxed: Box<str> = "boxed".into();
println!("{} {}", boxed, boxed.len());

let back: String = boxed.into();
println!("{}", back);
```

Output:

```
boxed 5
boxed
```

`Rc<str>` is a cheaply clonable shared string — cloning bumps a counter instead
of copying bytes.

```rust
use std::rc::Rc;

let shared: Rc<str> = Rc::from("shared");
let clone = Rc::clone(&shared);
println!("{} {} {}", shared, clone, Rc::strong_count(&shared));
```

Output:

```
shared shared 2
```

Both deref to `str`, so every read-only `&str` method works on them. Module 11
covers `Box` and `Rc` properly.

## Quick Reference

| Type | Size | Owns? | Growable | Typical use |
|---|---|---|---|---|
| `&str` | 16 | no | no | Function parameters, slices, literals |
| `&'static str` | 16 | no | no | Literals, compile-time constants |
| `String` | 24 | yes | yes | Fields, return values, building text |
| `&mut String` | 8 | no | yes | Mutating a caller's string |
| `Box<str>` | 16 | yes | no | Many immutable owned strings |
| `Rc<str>` | 16 | shared | no | Cheap clones of shared text |

| Conversion | Call |
|---|---|
| `&str` → `String` | `String::from(s)`, `s.to_string()`, `s.to_owned()`, `s.into()` |
| `String` → `&str` | `&s`, `s.as_str()`, `&s[..]` |
| `String` → `Box<str>` | `s.into_boxed_str()` |
| `Box<str>` → `String` | `b.into_string()` or `b.into()` |
| `&str` → `Rc<str>` | `Rc::from(s)` |
| `String` → bytes | `s.into_bytes()`, `s.as_bytes()` |

## Common Mistakes

**Taking `&String` in a function.** It rejects literals and slices for no
benefit. Take `&str`.

**Calling `.clone()` to pass a string to a reader.** Pass `&s`; deref coercion
handles the rest.

**Putting `&str` in a struct to avoid allocating.** It adds a lifetime parameter
to every signature that mentions the type. Use `String` until profiling says
otherwise.

**Expecting `&str` to grow.** `push_str`, `insert`, and `clear` exist only on
`String`. Convert first with `.to_string()`.

**Thinking `"literal"` allocates.** It lives in the binary. `"literal".to_string()`
is what allocates.

## Exercises

1. Write `fn initials(full: &str) -> String` returning the uppercase first
   letter of each whitespace-separated word. Call it with a literal and with a
   `String`.
2. Change its parameter to `&String` and record the exact error you get when
   passing a literal. Change it back.
3. Write `fn first_word(s: &str) -> &str` that returns the text up to the first
   space. Explain in one line why the return type needs no explicit lifetime.
4. Print `size_of` for `&str`, `String`, `Box<str>`, `Rc<str>`, and
   `Option<String>`. Account for the last one's result.
5. Build a `Vec<Box<str>>` of five names from literals, then convert the second
   element back into a `String` and push `"!"` onto it.

---

[← Course Index](../README.md)
