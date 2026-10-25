[← 5.3 Tuple Structs and Unit Structs](03-tuple-unit-structs.md) · [Course Index](../README.md) · [5.5 Option: No More Null →](05-option.md)

# 5.4 — Enums and Variants with Data

## What You'll Learn

- Unit, tuple, and struct variants — and mixing them in one enum
- Matching each variant shape, and why the compiler forces exhaustiveness
- Enums as state machines, and `impl` blocks on enums
- Explicit discriminants, `as` casting, layout, niche optimization, and `Box`

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `type M = {k:"quit"} \| {k:"write", s:string}` | `enum M { Quit, Write(String) }` |
| Python `Enum` (names only) | unit variants — but Rust variants carry data |
| Go `const ( A = iota )` | `enum A { ... }`, a real type, not an int alias |

A Rust enum is a **tagged union**: one of N shapes, each with its own payload.
It is the TS discriminated union with the tag built in and exhaustiveness
checked by the compiler. This is the single most used Rust feature there is no
direct equivalent for in Go.

## Unit variants

```rust
#[derive(Debug)]
enum Direction { North, South, East, West }

fn main() {
    let d = Direction::East;
    println!("{:?}", d);
    println!("{:?}", [Direction::North, Direction::West]);
}
```

Output:

```
East
[North, West]
```

Variants are namespaced under the enum: `Direction::East`, never bare `East`
(unless you `use Direction::*`).

## Variants with data

Each variant chooses its own shape: unit, tuple, or struct. They can mix freely.

```rust
#[derive(Debug)]
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}

let msgs = vec![
    Message::Quit,
    Message::Move { x: 3, y: -1 },
    Message::Write("hello".to_string()),
    Message::ChangeColor(255, 0, 128),
];

println!("{:?}", msgs[1]);
println!("{:#?}", msgs[3]);
```

Output:

```
Move { x: 3, y: -1 }
ChangeColor(
    255,
    0,
    128,
)
```

All four are the same type `Message`, so they live in one `Vec` and one function
signature handles all of them.

## Matching each shape

The pattern mirrors the declaration.

```rust
impl Message {
    fn describe(&self) -> String {
        match self {
            Message::Quit => "quit".to_string(),
            Message::Move { x, y } => format!("move to {x},{y}"),
            Message::Write(s) => format!("write {s:?}"),
            Message::ChangeColor(r, g, b) => format!("color {r},{g},{b}"),
        }
    }
}

for m in &msgs { println!("{}", m.describe()); }
```

Output:

```
quit
move to 3,-1
write "hello"
color 255,0,128
```

Matching on `&self` binds the payloads by reference, so nothing is moved out.

Shorthands that cut noise:

```rust
let n = match d {
    Direction::North | Direction::South => "vertical",
    _ => "horizontal",
};
println!("{n}");
```

Output:

```
horizontal
```

`|` for or-patterns, `_` for a catch-all, `..` to ignore a struct variant's
fields (`Message::Move { .. }`). Module 06 covers `match` exhaustively.

## Exhaustiveness is enforced

```rust
enum Direction { North, South, East, West }

let s = match d {
    Direction::North => "up",
    Direction::South => "down",
};
```

Does not compile:

```
error[E0004]: non-exhaustive patterns: `Direction::East` and `Direction::West` not covered
 --> src/main.rs:4:19
  |
4 |     let s = match d {
  |                   ^ patterns `Direction::East` and `Direction::West` not covered
  |
note: `Direction` defined here
 --> src/main.rs:1:6
  |
1 | enum Direction { North, South, East, West }
  |      ^^^^^^^^^                 ----  ---- not covered
  |                                |
  |                                not covered
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern, a match arm with multiple or-patterns as shown, or multiple match arms
  |
6 ~         Direction::South => "down",
7 ~         Direction::East | Direction::West => todo!(),
  |
```

This is why enums beat strings or ints for state: add a variant, and every
`match` that forgot it fails to compile. A `_ => ` arm throws that away — only
add one when the ignored cases genuinely do not matter.

## if let, let-else, matches!

For one variant out of many, skip the full `match`.

```rust
if let Message::Write(s) = &msgs[2] { println!("got {s}"); }

let Message::Move { x, y } = &msgs[1] else { panic!("not a move") };
println!("{x} {y}");

println!("{}", matches!(msgs[0], Message::Quit));
```

Output:

```
got hello
3 -1
true
```

`matches!` returns a `bool` and is the shortest way to ask "is it this variant".

## Enums as a state machine

Give each state its own variant and its own data. A transition function consumes
the old state and returns the new one, so an invalid state is unrepresentable.

```rust
#[derive(Debug, PartialEq)]
enum State {
    Idle,
    Running { pid: u32 },
    Paused { pid: u32 },
    Finished(i32),
}

#[derive(Debug)]
enum Event { Start(u32), Pause, Resume, Exit(i32) }

impl State {
    fn next(self, ev: Event) -> State {
        match (self, ev) {
            (State::Idle, Event::Start(pid)) => State::Running { pid },
            (State::Running { pid }, Event::Pause) => State::Paused { pid },
            (State::Paused { pid }, Event::Resume) => State::Running { pid },
            (State::Running { .. }, Event::Exit(code)) => State::Finished(code),
            (s, _) => s,
        }
    }
    fn is_terminal(&self) -> bool { matches!(self, State::Finished(_)) }
}

let mut s = State::Idle;
for ev in [Event::Start(42), Event::Pause, Event::Resume, Event::Exit(0), Event::Start(9)] {
    let before = format!("{s:?}");
    s = s.next(ev);
    println!("{before} -> {s:?}");
}
println!("{}", s.is_terminal());
```

Output:

```
Idle -> Running { pid: 42 }
Running { pid: 42 } -> Paused { pid: 42 }
Paused { pid: 42 } -> Running { pid: 42 }
Running { pid: 42 } -> Finished(0)
Finished(0) -> Finished(0)
true
```

`fn next(self, ...)` by value is the trick: you cannot accidentally keep a
reference to the old state. Matching on the `(state, event)` tuple puts the whole
transition table in one place. Note `Idle` has no `pid` field — the type system,
not a runtime check, guarantees you never read a pid before one exists.

## Explicit discriminants and `as`

A fieldless enum can pin each variant to an integer, useful for protocol codes.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Status { Ok = 200, NotFound = 404, Error = 500 }

#[derive(Debug, Clone, Copy, PartialEq)]
enum Level { Low, Mid, High }

println!("{} {} {}", Status::Ok as u16, Status::NotFound as u16, Status::Error as u16);
println!("{} {} {}", Level::Low as i32, Level::Mid as i32, Level::High as i32);
```

Output:

```
200 404 500
0 1 2
```

Without explicit values, discriminants start at 0 and count up. `as` casting
works **only** for enums where no variant has data.

## C-like enums and `#[repr]`

`#[repr(u8)]` fixes the representation, which matters when the bytes cross an
FFI or wire boundary.

```rust
use std::mem::size_of;

#[derive(Debug, Clone, Copy, PartialEq)]
#[repr(u8)]
enum Opcode { Nop = 0x00, Load = 0x01, Store = 0x02, Halt = 0xFF }

println!("{}", Opcode::Halt as u8);
println!("{} {}", size_of::<Opcode>(), size_of::<Option<Opcode>>());
```

Output:

```
255
1 1
```

`as` goes one way only. `u8 as Opcode` is `error[E0605]: non-primitive cast` —
because not every byte is a valid variant. Write the reverse conversion yourself
and return `Option`:

```rust
impl Opcode {
    fn from_byte(b: u8) -> Option<Opcode> {
        match b {
            0x00 => Some(Opcode::Nop),
            0x01 => Some(Opcode::Load),
            0x02 => Some(Opcode::Store),
            0xFF => Some(Opcode::Halt),
            _ => None,
        }
    }
}

println!("{:?} {:?}", Opcode::from_byte(0x02), Opcode::from_byte(0x7A));
```

Output:

```
Some(Store) None
```

Lesson 16.5 covers `#[repr]` for FFI.

## Size and layout

An enum is sized for its largest variant plus a tag, rounded for alignment.

```rust
use std::mem::size_of;

println!("{}", size_of::<Direction>());
println!("{}", size_of::<Message>());
println!("{}", size_of::<Status>());
println!("{}", size_of::<Option<u8>>());
println!("{}", size_of::<Option<i64>>());
```

Output:

```
1
24
2
2
16
```

`Direction` has four variants and no data: one byte. `Message`'s largest variant
is `Write(String)` at 24 bytes, and the tag fits in `String`'s padding.
`Option<i64>` needs 16: 8 for the payload, 8 more once alignment is applied to
the tag.

## Niche optimization

When a type has a value it can never hold, the compiler uses it as the tag.
A `Box` is never null, so `None` becomes the null pointer:

```rust
use std::mem::size_of;

println!("{} {}", size_of::<Box<i32>>(), size_of::<Option<Box<i32>>>());
println!("{} {}", size_of::<&i32>(), size_of::<Option<&i32>>());
```

Output:

```
8 8
8 8
```

`Option<Box<T>>` is the same size as `Box<T>`. The nullable pointer you would
write by hand in C costs exactly the same here, with the null check enforced.

## Recursive enums need Box

An enum containing itself directly has no finite size.

```rust
enum Expr { Num(i64), Add(Expr, Expr) }
```

Does not compile:

```
error[E0072]: recursive type `Expr` has infinite size
 --> src/main.rs:1:1
  |
1 | enum Expr { Num(i64), Add(Expr, Expr) }
  | ^^^^^^^^^                 ---- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
  |
1 | enum Expr { Num(i64), Add(Box<Expr>, Expr) }
  |                           ++++    +
```

`Box<Expr>` is a pointer — fixed size — so the type closes.

```rust
#[derive(Debug)]
enum Expr {
    Num(i64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}

impl Expr {
    fn eval(&self) -> i64 {
        match self {
            Expr::Num(n) => *n,
            Expr::Add(a, b) => a.eval() + b.eval(),
            Expr::Mul(a, b) => a.eval() * b.eval(),
        }
    }
}

let e = Expr::Add(
    Box::new(Expr::Num(2)),
    Box::new(Expr::Mul(Box::new(Expr::Num(3)), Box::new(Expr::Num(4)))),
);
println!("{}", e.eval());
println!("{:?}", e);
```

Output:

```
14
Add(Num(2), Mul(Num(3), Num(4)))
```

This is how every AST, linked list, and tree is written in Rust. Module 11
covers `Box`, `Rc`, and the rest.

## Quick Reference

| Form | Syntax |
|---|---|
| Unit variant | `enum E { A }` |
| Tuple variant | `enum E { A(i32, i32) }` |
| Struct variant | `enum E { A { x: i32 } }` |
| Construct | `E::A`, `E::A(1, 2)`, `E::A { x: 1 }` |
| Match tuple variant | `E::A(a, b) => ...` |
| Match struct variant | `E::A { x } => ...` / `E::A { .. }` |
| Or-pattern | `E::A \| E::B => ...` |
| Single-variant test | `matches!(v, E::A(_))` |
| Single-variant bind | `if let E::A(x) = &v` / `let E::A(x) = v else {...}` |
| Methods | `impl E { fn f(&self) { match self { ... } } }` |
| Discriminant | `enum E { A = 200 }`, then `E::A as u16` |
| Fixed repr | `#[repr(u8)] enum E { ... }` |
| Recursive | `A(Box<E>)` |
| Size | largest variant + tag; `Option<Box<T>>` == `Box<T>` |

## Common Mistakes

**Writing bare `East` instead of `Direction::East`.** Variants are always
namespaced; add `use Direction::*;` if you want the short form.

**Adding `_ => {}` out of habit.** It silences the one check that makes enums
worth using. Spell out the variants and let the next `git pull` break the build.

**Casting an int to an enum with `as`.** One-way only: `E0605`. Write a
`from_*` function returning `Option`.

**Giving a data-carrying variant a discriminant.** `enum E { A(u8) = 1 }` and
`as` casting on it are not available for enums with fields.

**Recursing without indirection.** `E0072`; wrap the recursive field in `Box`.

**Matching on `self` instead of `&self` in a method.** It moves the enum out of
the receiver. Use `&self` unless the method is meant to consume.

## Exercises

1. Define `enum Shape { Circle { r: f64 }, Rect { w: f64, h: f64 }, Point }`
   with `area(&self) -> f64`. Print all three.
2. Write `enum Token { Num(f64), Plus, Minus, LParen, RParen }` and a function
   that prints each token's symbol. Delete one arm and capture the error.
3. Model a traffic light as an enum with `fn next(self) -> Self`. Run it through
   seven transitions and print each.
4. Give `enum HttpStatus` explicit discriminants for 200/301/404/500, plus
   `from_code(u16) -> Option<HttpStatus>`. Test a valid and an invalid code.
5. Build `enum Json { Null, Bool(bool), Num(f64), Str(String), Arr(Vec<Json>) }`
   and write `fn depth(&self) -> usize`. Print the depth of a nested value.
6. Print `size_of` for your `Shape`, `Token`, and `Option<Box<Json>>`. Explain
   each number from the variant list.

---

[← 5.3 Tuple Structs and Unit Structs](03-tuple-unit-structs.md) · [Course Index](../README.md) · [5.5 Option: No More Null →](05-option.md)
