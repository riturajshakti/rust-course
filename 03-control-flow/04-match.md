[← Course Index](../README.md)

# 3.4 — match: Rust's Superpower

## What You'll Learn

- `match` is exhaustive: the compiler proves you handled every case
- Every arm form: literals, ranges, `|`, tuples, guards, `@` bindings, `_`
- `match` is an expression, so every arm must produce the same type
- Matching `Option`/`Result`, by value and by reference

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `switch` with `break` | `match`, no fallthrough, no `break` |
| JS `switch` missing a case | compile error, not silent |
| Python `match` (3.10+) | very close — Rust's is exhaustive |
| Go `switch { case }` | similar, but Rust checks completeness |

Two things no `switch` gives you: **exhaustiveness checking** and
**destructuring**. `match` replaces the long `else if` chain, the visitor
pattern, and most null checks.

## Basic form

Arms are `pattern => expression,`. First match wins. No fallthrough, no `break`.

```rust
let n = 3;
match n {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("many"),
}
```

Output:

```
three
```

Trailing commas after an arm are required unless the arm is a block.

## Exhaustiveness is enforced

Miss a case and it does not compile. This is the whole point.

```rust
enum Direction { North, South, East, West }

let d = Direction::North;
let s = match d {
    Direction::North => "up",
    Direction::South => "down",
};
```

Does not compile — intentional:

```
error[E0004]: non-exhaustive patterns: `Direction::East` and `Direction::West` not covered
 --> src/main.rs:5:19
  |
5 |     let s = match d {
  |                   ^ patterns `Direction::East` and `Direction::West` not covered
  |
note: `Direction` defined here
 --> src/main.rs:1:6
  |
1 | enum Direction { North, South, East, West }
  |      ^^^^^^^^^                 ----  ---- not covered
  |                                |
  |                                not covered
  = note: the matched value is of type `Direction`
help: ensure that all possible cases are being handled by adding a match arm with
a wildcard pattern, a match arm with multiple or-patterns as shown, or multiple
match arms: `Direction::East | Direction::West => todo!(),`
```

The compiler names the missing variants and offers the patch. Add a variant to
the enum later and every `match` that needs updating lights up.

For integers it spells out the uncovered ranges:

```rust
let n = 3;
let s = match n {
    1 => "one",
    2 => "two",
};
```

Does not compile — intentional:

```
error[E0004]: non-exhaustive patterns: `i32::MIN..=0_i32` and `3_i32..=i32::MAX` not covered
 --> src/main.rs:3:19
  |
3 |     let s = match n {
  |                   ^ patterns `i32::MIN..=0_i32` and `3_i32..=i32::MAX` not covered
  |
  = note: the matched value is of type `i32`
```

An exhaustive integer match without `_` is possible for small types:

```rust
let n: u8 = 200;
match n {
    0..=127 => println!("low"),
    128..=255 => println!("high"),
}
```

Output:

```
high
```

## `_` versus a catch-all binding

`_` matches anything and binds nothing. A bare name matches anything and binds
the value.

```rust
let n = 7;
match n {
    1 => println!("one"),
    _ => println!("ignored, value not bound"),
}
match n {
    1 => println!("one"),
    other => println!("caught {other}"),
}
```

Output:

```
ignored, value not bound
caught 7
```

A catch-all must be last — anything after it is unreachable:

```rust
let s = match n {
    _ => "any",
    1 => "one",
};
```

```
warning: unreachable pattern
 --> src/main.rs:5:9
  |
4 |         _ => "any",
  |         - matches any value
5 |         1 => "one",
  |         ^ no value can reach this
```

A warning, not an error — so read your warnings.

Idiomatic: use `_` when you don't need the value, a name when you do, and avoid
`_` on an enum if you'd rather the compiler nag you when a variant is added.

## Literals

```rust
let cmd = String::from("start");
match cmd.as_str() {
    "start" | "run" => println!("go"),
    "stop" => println!("halt"),
    other => println!("unknown: {other}"),
}
```

Output:

```
go
```

Matching a `String` against string literals needs `.as_str()`:

```rust
let cmd = String::from("start");
match cmd {
    "start" => println!("go"),
    _ => println!("?"),
}
```

Does not compile — intentional:

```
error[E0308]: mismatched types
 --> src/main.rs:4:9
  |
3 |     match cmd {
  |           --- this expression has type `String`
4 |         "start" => println!("go"),
  |         ^^^^^^^ expected `String`, found `&str`
  |
help: consider converting the `String` to a `&str` using `.as_str()`
```

`bool` needs no `_` — `true` and `false` are exhaustive. So is `()`, whose only
pattern is `()`.

## Range patterns

Only `..=` (inclusive) is allowed in a pattern; `..` as an exclusive range
pattern is not stable.

```rust
let grade = match 87 {
    90..=100 => "A",
    80..=89 => "B",
    70..=79 => "C",
    0..=69 => "F",
    _ => "invalid",
};
println!("{grade}");
```

Output:

```
B
```

Works on `char` too:

```rust
for c in ['a', 'z', '5', '!'] {
    let kind = match c {
        'a'..='z' => "lowercase",
        'A'..='Z' => "uppercase",
        '0'..='9' => "digit",
        _ => "other",
    };
    print!("{c}:{kind} ");
}
println!();
```

Output:

```
a:lowercase z:lowercase 5:digit !:other 
```

For an open-ended range, name the type's bound: `i32::MIN..=-1` for "any
negative", `100..=i32::MAX` for "100 or more".

## `|` alternatives

One arm, several patterns. No fallthrough needed.

```rust
for n in [1, 3, 5, 7, 11] {
    let d = match n {
        1 | 2 | 3 => "small",
        4 | 5 | 6 => "medium",
        _ => "large",
    };
    print!("{n}:{d} ");
}
println!();
```

Output:

```
1:small 3:small 5:medium 7:large 11:large 
```

Mix with ranges freely: `1 | 5..=9 | 20`.

## `@` bindings

`name @ pattern` tests the pattern *and* binds the matched value.

```rust
let x = 42;
match x {
    n @ 1..=9 => println!("single digit {n}"),
    n @ 10..=99 => println!("double digit {n}"),
    n => println!("big {n}"),
}
```

Output:

```
double digit 42
```

Without `@`, `1..=9 => println!("{}", ???)` has no name for the value. It nests
anywhere a pattern goes, including inside a tuple:
`(level, code @ 500..=599) => ..`.

## Tuple patterns

Destructure and match positionally in one step.

```rust
let pair = (0, 7);
match pair {
    (0, 0) => println!("origin"),
    (0, y) => println!("on y axis at {y}"),
    (x, 0) => println!("on x axis at {x}"),
    (x, y) => println!("at {x},{y}"),
}
```

Output:

```
on y axis at 7
```

Mixing literals and bindings per position:

```rust
let config: (bool, u16) = (true, 443);
match config {
    (true, 443) => println!("tls default"),
    (true, p) => println!("tls on port {p}"),
    (false, p) => println!("plain on {p}"),
}
```

Output:

```
tls default
```

This one is exhaustive with no `_`: `bool` has two values, and `p` covers every
port.

## Match guards

`if cond` after the pattern. The guard can use the bound variables and any
other value in scope.

```rust
let n = 15;
let s = match n {
    x if x % 15 == 0 => "FizzBuzz".to_string(),
    x if x % 3 == 0 => "Fizz".to_string(),
    x if x % 5 == 0 => "Buzz".to_string(),
    x => x.to_string(),
};
println!("{s}");
```

Output:

```
FizzBuzz
```

Guards are invisible to the exhaustiveness checker — a guarded arm never counts
as covering anything, so you will still need a final unguarded arm.

## match is an expression

Every arm produces a value of the same type.

```rust
let msg = match 0 {
    0 => {
        let a = 1;
        let b = 2;
        format!("block arm {}", a + b)
    }
    _ => String::from("other"),
};
println!("{msg}");
```

Output:

```
block arm 3
```

Block arms need no comma after `}`. Mismatched arm types:

```rust
let s = match n {
    1 => "one",
    _ => 0,
};
```

Does not compile — intentional:

```
error[E0308]: `match` arms have incompatible types
 --> src/main.rs:5:14
  |
4 |         1 => "one",
  |              ----- this is found to be of type `&str`
5 |         _ => 0,
  |              ^ expected `&str`, found integer
```

When the arms are statements, the common type is `()`, and an arm that does
nothing is written `1 => {}`.

## Matching Option

The pattern you will write most often in Rust.

```rust
let opts = [Some(5), None];
for o in opts {
    match o {
        Some(0) => println!("zero"),
        Some(n) if n > 3 => println!("big {n}"),
        Some(n) => println!("small {n}"),
        None => println!("nothing"),
    }
}
```

Output:

```
big 5
nothing
```

Lesson 5.5 covers `Option`'s full API; lesson 3.6 shows `if let` for the
one-case version.

## Matching Result

`Ok` and `Err` are exhaustive for `Result`, so no `_` is needed.

```rust
for s in ["42", "abc"] {
    match s.parse::<i32>() {
        Ok(n) => println!("parsed {n}"),
        Err(e) => println!("failed: {e}"),
    }
}
```

Output:

```
parsed 42
failed: invalid digit found in string
```

## Enum variants with data

Each variant's payload destructures in the pattern. (Module 05 teaches enum
definitions; this is just the matching side.)

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Rect { w: f64, h: f64 },
    Point,
}

let shapes = vec![Shape::Circle(2.0), Shape::Rect { w: 3.0, h: 4.0 }, Shape::Point];
for s in &shapes {
    let area = match s {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rect { w, h } => w * h,
        Shape::Point => 0.0,
    };
    println!("{s:?} area {area:.2}");
}
```

Output:

```
Circle(2.0) area 12.57
Rect { w: 3.0, h: 4.0 } area 12.00
Point area 0.00
```

Three payload shapes, three pattern shapes: `Circle(r)` for a tuple variant,
`Rect { w, h }` for a struct variant, bare `Point` for a unit variant.

## Nested patterns

Patterns nest to any depth.

```rust
let nested: Option<Option<i32>> = Some(Some(3));
match nested {
    Some(Some(n)) => println!("inner {n}"),
    Some(None) => println!("outer only"),
    None => println!("none"),
}

```

Output:

```
inner 3
```

All three `Option<Option<T>>` cases are distinct and the compiler knows it.
Guards combine with nesting: `Some((n, label)) if n > 0 => ..`.

## Match ergonomics with references

Matching `&T` against a non-reference pattern automatically binds by
reference. You do not write `&Some(ref s)`.

```rust
let name = Some(String::from("raj"));
match &name {
    Some(s) => println!("len {} of {s}", s.len()),
    None => println!("none"),
}
println!("still have {name:?}");
```

Output:

```
len 3 of raj
still have Some("raj")
```

`s` is a `&String` here, so `name` is untouched. Trying to treat it as owned:

```rust
match &name {
    Some(s) => {
        let owned: String = s;
        println!("{owned}");
    }
    None => {}
}
```

Does not compile — intentional:

```
error[E0308]: mismatched types
 --> src/main.rs:5:33
  |
5 |             let owned: String = s;
  |                        ------   ^ expected `String`, found `&String`
  |
help: try using a conversion method: `s.to_string()`
```

Matching `&mut T` gives `&mut` bindings, so you can mutate through them:

```rust
let mut v = Some(10);
match &mut v {
    Some(n) => *n += 5,
    None => {}
}
println!("{v:?}");
```

Output:

```
Some(15)
```

Matching the value itself moves non-`Copy` payloads out — `match name { Some(s)
=> println!("owned {s}"), None => {} }` prints `owned raj`, and
`name` is unusable after that — the `String` moved into `s`. Rule: `match &x`
to inspect, `match x` to consume. Lesson 3.5 covers `ref`/`ref mut`, the
pre-ergonomics spelling you'll still see in older code.

## Quick Reference

| Pattern form | Example | Matches |
|---|---|---|
| Literal | `3`, `"go"`, `true`, `'a'` | that exact value |
| Wildcard | `_` | anything, binds nothing |
| Binding | `n` | anything, binds it |
| Range | `1..=9`, `'a'..='z'` | inclusive range |
| Open range | `i32::MIN..=-1` | use type bounds |
| Alternative | `1 \| 2 \| 3` | any listed pattern |
| Binding + test | `n @ 10..=99` | tests and binds |
| Tuple | `(0, y)` | positionally |
| Tuple variant | `Shape::Circle(r)` | payload bound |
| Struct variant | `Shape::Rect { w, h }` | fields bound |
| Unit variant | `Shape::Point` | |
| Nested | `Some(Some(n))` | any depth |
| Guard | `n if n % 2 == 0` | extra bool test |
| Reference | `match &x { Some(s) => }` | `s` is `&T` |
| Mutable reference | `match &mut x { Some(n) => *n += 1 }` | `n` is `&mut T` |
| Block arm | `p => { .. }` | no trailing comma |

## Common Mistakes

**Expecting fallthrough.** There is none, and no `break`. Use `|` to share an
arm.

**Using `_` on an enum out of habit.** It silences E0004 forever, so adding a
variant compiles silently. List the variants when you want the warning.

**`match some_string { "x" => .. }`.** E0308 — add `.as_str()`.

**Thinking a guard counts for exhaustiveness.** `x if x > 0` covers nothing as
far as the checker knows. Keep a final unguarded arm.

**Putting the catch-all first.** The arms below become unreachable — a warning,
not an error. Catch-all goes last.

**`match x` when you only wanted to look.** It moves non-`Copy` payloads.
`match &x`.

## Exercises

1. Write `fn http_status(code: u16) -> &'static str` using range patterns for
   1xx through 5xx, with a catch-all for anything else.
2. Define `enum Op { Add, Sub, Mul, Div }` and `fn apply(op: Op, a: f64, b: f64)
   -> Option<f64>`, returning `None` on division by zero. Use no `_` arm.
3. Delete one arm from exercise 2's match and read the E0004 message. Note which
   variant it names.
4. Match on `(i32, i32)` and print `"origin"`, `"x-axis"`, `"y-axis"`,
   `"diagonal"`, or `"general"`, using one guard.
5. Write a FizzBuzz for 1..=20 as a single `match` with guards, printing each
   result.
6. Given `let v: Vec<Option<i32>>`, use `match` in a `for` loop to sum only the
   `Some` values greater than 10, with an `@` binding to name the value.

---

[← Course Index](../README.md)
