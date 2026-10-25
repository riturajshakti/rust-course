[← Course Index](../README.md)

# 4.7 — Lifetime Elision and Struct Lifetimes

## What You'll Learn

- The three elision rules, stated, with the example each one covers
- Structs that hold references: the `E0106` fix and `impl` blocks
- `'static` in both of its meanings, and the error each produces
- When to stop annotating and store owned data instead

## Coming From Another Language

Nothing to map. This lesson is about why you rarely *write* lifetimes even
though every reference has one. The compiler applies three mechanical rules to
fill them in; when the rules produce an unambiguous answer, you write nothing.
`E0106` means the rules came up short.

## The three elision rules

Applied in order to any function signature:

1. Every elided lifetime in an **input** position gets its own distinct
   lifetime parameter.
2. If there is **exactly one** input lifetime, it is assigned to every elided
   **output** lifetime.
3. If one of the inputs is `&self` or `&mut self`, the lifetime of `self` is
   assigned to every elided output lifetime.

If output lifetimes remain unassigned after all three, you get `E0106`.

## Rule 1: each input gets its own

```rust
fn report(a: &str, b: &str) {
    println!("{} and {}", a, b);
}

fn main() {
    report("x", "yy");
}
```

Output:

```
x and yy
```

The compiler reads this as `fn report<'a, 'b>(a: &'a str, b: &'b str)`. Nothing
is returned, so rule 1 alone finishes the job. Independent lifetimes are the
right default — forcing them equal would reject valid callers.

## Rule 2: one input, so the output borrows from it

```rust
fn trim_left(s: &str) -> &str {
    s.trim_start()
}

fn main() {
    println!("{:?}", trim_left("   hi"));
}
```

Output:

```
"hi"
```

Expanded: `fn trim_left<'a>(s: &'a str) -> &'a str`. This is why `first_word`
(lesson 4.5) needed no annotation.

Two inputs and rule 2 does not apply. Does not compile:

```rust
fn two_in(a: &str, b: &str) -> &str { a }
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:32
  |
1 | fn two_in(a: &str, b: &str) -> &str { a }
  |              ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn two_in<'a>(a: &'a str, b: &'a str) -> &'a str { a }
  |          ++++     ++          ++          ++
```

Zero input references also fails, with a different `help`:

```rust
fn get(n: usize) -> &str { "x" }
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:21
  |
1 | fn get(n: usize) -> &str { "x" }
  |                     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime, but this is uncommon unless you're returning a borrowed value from a `const` or a `static`
  |
1 | fn get(n: usize) -> &'static str { "x" }
  |                      +++++++
```

Here `'static` genuinely is correct — the literal lives in the binary.

## Rule 3: &self wins

In an `impl` block, the receiver's lifetime is handed to the output regardless
of how many other reference parameters there are.

```rust
struct Parser<'a> { input: &'a str, pos: usize }

impl<'a> Parser<'a> {
    fn rest(&self) -> &str { &self.input[self.pos..] }
    fn advance(&mut self, n: usize) -> &str {
        self.pos += n;
        &self.input[..self.pos]
    }
}

fn main() {
    let mut p = Parser { input: "hello world", pos: 0 };
    println!("{:?}", p.rest());
    println!("{:?}", p.advance(5));
    println!("{:?}", p.rest());
}
```

Output:

```
"hello world"
"hello"
" world"
```

`rest` expands to `fn rest<'s>(&'s self) -> &'s str` — the result borrows
`self`, not `'a`. That is almost always what you want, and it is also why a
method returning a reference keeps `self` borrowed afterwards.

## When rule 3's answer is too short

`rest` ties the output to `&self`, so its result can't outlive the `Parser`.
If the data genuinely lives as long as `'a`, say `'a` explicitly.

```rust
struct Parser<'a> { input: &'a str, pos: usize }

impl<'a> Parser<'a> {
    fn borrowed(&self) -> &str { self.input }
    fn full(&self) -> &'a str { self.input }
}

fn main() {
    let text = String::from("hello world");
    let out;
    {
        let p = Parser { input: &text, pos: 0 };
        out = p.full();
    }
    println!("{out}");
}
```

Output:

```
hello world
```

Swapping `p.full()` for `p.borrowed()` here gives `error[E0597]: p does not
live long enough` — the `Parser` is gone but `borrowed`'s result was tied to
it. The `input` data is owned by `text`,
which is still alive, so `-> &'a str` is the honest signature.

## Structs holding references need a parameter

Does not compile:

```rust
struct Excerpt { part: &str }
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:24
  |
1 | struct Excerpt { part: &str }
  |                        ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 | struct Excerpt<'a> { part: &'a str }
  |               ++++          ++
```

The fix, with methods:

```rust
#[derive(Debug)]
struct Excerpt<'a> { part: &'a str }

impl<'a> Excerpt<'a> {
    fn new(text: &'a str) -> Excerpt<'a> {
        let end = text.find('.').unwrap_or(text.len());
        Excerpt { part: &text[..end] }
    }
    fn part(&self) -> &str { self.part }
    fn announce(&self, note: &str) -> &str {
        println!("attention: {note}");
        self.part
    }
    fn len(&self) -> usize { self.part.len() }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let e = Excerpt::new(&novel);
    println!("{:?}", e);
    println!("{}", e.part());
    println!("{}", e.announce("read this"));
    println!("{}", e.len());
}
```

Output:

```
Excerpt { part: "Call me Ishmael" }
Call me Ishmael
attention: read this
Call me Ishmael
15
```

`announce` has two reference inputs plus `&self`; rule 3 picks `self`, so no
annotation is needed.

## The struct lifetime constrains the instance

`Excerpt<'a>` cannot outlive what it borrows. Does not compile:

```rust
struct Excerpt<'a> { part: &'a str }

fn main() {
    let e;
    {
        let novel = String::from("some text here");
        e = Excerpt { part: &novel };
    }
    println!("{}", e.part);
}
```

```
error[E0597]: `novel` does not live long enough
 --> src/main.rs:7:29
  |
6 |         let novel = String::from("some text here");
  |             ----- binding `novel` declared here
7 |         e = Excerpt { part: &novel };
  |                             ^^^^^^ borrowed value does not live long enough
8 |     }
  |     - `novel` dropped here while still borrowed
9 |     println!("{}", e.part);
  |                    ------ borrow later used here
```

The struct is 16 bytes regardless — the lifetime is not stored anywhere:

```rust
println!("{}", std::mem::size_of::<Excerpt>());
```

Output:

```
16
```

## impl block syntax

```rust
impl<'a> Excerpt<'a> { }
```

The `<'a>` after `impl` **declares** the parameter; the `<'a>` after the type
name **uses** it — exactly like `impl<T> Wrapper<T>`. Omitting either is an
error. With `'static` you can write `impl Excerpt<'static>` for a specialized
block.

## 'static, meaning 1: a reference valid forever

```rust
fn main() {
    let lit: &'static str = "lives for the whole program";
    println!("{lit}");
    const NAME: &str = "also 'static";
    println!("{NAME}");
    println!("{}", std::mem::size_of::<&'static str>());
}
```

Output:

```
lives for the whole program
also 'static
16
```

String literals and `const`/`static` data are baked into the binary, so a
reference to them is valid for the program's entire run.

## 'static, meaning 2: a bound saying "contains no short-lived refs"

`T: 'static` does **not** mean "lives forever". It means `T` holds no reference
that could expire — so every owned type qualifies.

```rust
use std::fmt::Debug;

fn spawnish<T: Debug + 'static>(v: T) {
    println!("{:?}", v);
}

fn main() {
    spawnish(String::from("owned is 'static"));
    spawnish(5);
    spawnish("literal");
    spawnish(vec![1, 2, 3]);
}
```

Output:

```
"owned is 'static"
5
"literal"
[1, 2, 3]
```

A `String` satisfies `T: 'static` because it owns its data. A borrow of a local
does not. Does not compile:

```rust
let local = String::from("local");
spawnish(&local);
```

```
error[E0597]: `local` does not live long enough
  --> src/main.rs:13:14
   |
12 |     let local = String::from("local");
   |         ----- binding `local` declared here
13 |     spawnish(&local);
   |     ---------^^^^^^-
   |     |        |
   |     |        borrowed value does not live long enough
   |     argument requires that `local` is borrowed for `'static`
14 | }
   | - `local` dropped here while still borrowed
   |
note: requirement that the value outlives `'static` introduced here
  --> src/main.rs:3:24
   |
 3 | fn spawnish<T: Debug + 'static>(v: T) {
   |                        ^^^^^^^
```

You will hit this bound constantly with `thread::spawn` and
`tokio::spawn` (modules 14 and 15). The fix is never to add `'static` — it is
to pass owned data, usually with `.clone()` or `Arc`.

## When to store owned data instead

Borrowing in a struct is a real optimization, and it has a real cost: the
lifetime parameter propagates into every signature that mentions the type.

```rust
struct Config { name: String }

fn load() -> Config { Config { name: String::from("db") } }
fn show(c: &Config) { println!("{}", c.name); }

fn main() {
    let c = load();
    show(&c);
}
```

Output:

```
db
```

With `name: &'a str` instead, `load` cannot exist at all (nothing to borrow
from), `Config<'a>` appears in every signature, and storing it in a struct
drags `'a` there too.

| Store | When |
|---|---|
| `String` / `Vec<T>` | the default — always start here |
| `&'a str` / `&'a [T]` | short-lived view over data a caller owns, e.g. a parser or zero-copy decoder |
| `Rc<str>` / `Arc<str>` | shared ownership without a lifetime (module 11) |
| `Cow<'a, str>` | sometimes borrowed, sometimes owned (lesson 11.6) |

Rule of thumb: if the type is returned from a function, outlives one call, or is
stored in another struct, make it owned.

## Quick Reference

| Rule | Condition | Effect |
|---|---|---|
| 1 | always | each elided input lifetime is distinct |
| 2 | exactly one input lifetime | it becomes every elided output lifetime |
| 3 | `&self` / `&mut self` present | `self`'s lifetime becomes every elided output |

| Signature | Expands to |
|---|---|
| `fn f(a: &str, b: &str)` | `fn f<'a,'b>(a: &'a str, b: &'b str)` |
| `fn f(s: &str) -> &str` | `fn f<'a>(s: &'a str) -> &'a str` |
| `fn f(&self) -> &str` | `fn f<'s>(&'s self) -> &'s str` |
| `fn f(&self, x: &str) -> &str` | output borrows `self`, not `x` |
| `fn f(a: &str, b: &str) -> &str` | `E0106` |
| `fn f(n: usize) -> &str` | `E0106` |

| Form | Meaning |
|---|---|
| `struct S<'a> { r: &'a T }` | `S` cannot outlive what it borrows |
| `impl<'a> S<'a>` | declare then use |
| `&'static T` | reference valid for the whole program |
| `T: 'static` | `T` contains no borrowed data |

## Common Mistakes

**Adding `'static` to silence a lifetime error.** It is a stronger claim, not a
looser one. Pass owned data instead.

**Reading `T: 'static` as "lives forever".** It means "no short-lived
references inside"; every `String` and `Vec` satisfies it.

**Writing `impl Excerpt<'a>` without `impl<'a>`.** You must declare the
parameter before using it.

**Expecting a method's returned reference to outlive `self`.** Rule 3 ties it
to `self`. Write `-> &'a str` if the data really lives longer.

**Putting `&'a str` in a struct to avoid one allocation.** The lifetime spreads
through your whole API. Use `String` unless you are writing a parser.

## Exercises

1. For each of these, state the expansion or `E0106`:
   `fn a(x: &i32) -> &i32`, `fn b(x: &i32, y: &i32) -> &i32`,
   `fn c(&self) -> &i32`, `fn d(&self, x: &i32) -> &i32`, `fn e() -> &i32`.
2. Write `struct Tokenizer<'a> { src: &'a str, pos: usize }` with
   `fn next_token(&mut self) -> Option<&str>`. Which elision rule applies?
3. Change `next_token` to return `Option<&'a str>` and explain what callers can
   now do that they could not before.
4. Write a function generic over `T: 'static` and try calling it with a
   reference to a local. Read the full error.
5. Rewrite `Tokenizer` to own a `String` instead. Compare the two versions'
   signatures and say which you would ship.

---

[← Course Index](../README.md)
