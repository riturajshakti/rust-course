[← Course Index](../README.md)

# 3.5 — Pattern Matching in Depth

## What You'll Learn

- Destructuring structs, tuples, slices, and arbitrary nests of them
- Slice patterns: `[first, .., last]` and `rest @ ..`
- `_` vs `..` vs `_name`, and `ref`/`ref mut`
- Refutable vs irrefutable, and the six places a pattern is allowed

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `const { a, b } = obj` | `let Struct { a, b } = s;` |
| JS `const [x, ...rest] = arr` | `let [x, rest @ ..] = a;` (match only) |
| JS `const { a: renamed } = o` | `let Struct { a: renamed } = s;` |
| Python `a, *mid, b = lst` | `[a, mid @ .., b]` in a `match` |

Rust's destructuring looks like JS's but goes further: it can *test* while
binding, and the compiler rejects a pattern that might not match where one must.

## Destructuring a struct

Field names bind to variables of the same name.

```rust
struct Point { x: i32, y: i32 }

let p = Point { x: 1, y: 2 };
let Point { x, y } = p;
println!("{x} {y}");
```

Output:

```
1 2
```

## Renaming while destructuring

`field: newname`.

```rust
let p2 = Point { x: 5, y: 6 };
let Point { x: a, y: b } = p2;
println!("{a} {b}");
```

Output:

```
5 6
```

## Ignoring the rest with `..`

```rust
struct User { name: String, email: String, age: u32, active: bool }

let u = User { name: "raj".into(), email: "r@x.com".into(), age: 30, active: true };

let User { name, age, .. } = &u;
println!("{name} {age}");

let User { email: e, .. } = &u;
println!("{e}");
```

Output:

```
raj 30
r@x.com
```

Without `..`, every field must be listed — the same completeness rule as a
struct literal. `..` goes last.

## Struct patterns in match

Literals in field positions test; names bind.

```rust
let p3 = Point { x: 0, y: 7 };
match p3 {
    Point { x: 0, y: 0 } => println!("origin"),
    Point { x: 0, y } => println!("on y at {y}"),
    Point { x, y: 0 } => println!("on x at {x}"),
    Point { x, y } => println!("{x},{y}"),
}
```

Output:

```
on y at 7
```

## Tuple patterns

```rust
let t = (1, "two", 3.0);
let (a, b, c) = t;
println!("{a} {b} {c}");

let (first, .., last) = (1, 2, 3, 4, 5);
println!("{first} {last}");

let (_, second, _) = (1, 2, 3);
println!("{second}");

let ((a, b), c) = ((1, 2), 3);
println!("{a} {b} {c}");
```

Output:

```
1 two 3
1 5
2
1 2 3
```

Destructuring assignment works on existing bindings too — a swap with no
temporary:

```rust
let mut x = 5;
let mut y = 10;
(x, y) = (y, x);
println!("{x} {y}");
```

Output:

```
10 5
```

## Array and slice patterns

A fixed-length array destructures irrefutably because the length is known:

```rust
let a = [1, 2, 3];
let [p, q, r] = a;
println!("{p} {q} {r}");
```

Output:

```
1 2 3
```

A slice has unknown length, so slice patterns are a `match` tool. Matching on
length is exhaustive when you end with a variable-length pattern:

```rust
for v in [vec![], vec![1], vec![1, 2], vec![1, 2, 3, 4]] {
    let s: &[i32] = &v;
    let d = match s {
        [] => "empty".to_string(),
        [only] => format!("one: {only}"),
        [first, last] => format!("two: {first},{last}"),
        [first, .., last] => format!("many: {first}..{last}"),
    };
    println!("{d}");
}
```

Output:

```
empty
one: 1
two: 1,2
many: 1..4
```

`[first, .., last]` requires at least two elements — hence the separate `[]`
and `[only]` arms.

## Rest patterns: binding the middle

`name @ ..` binds the skipped elements as a sub-slice.

```rust
let v = vec![1, 2, 3, 4, 5];
match v.as_slice() {
    [head, tail @ ..] => println!("head {head} tail {tail:?}"),
    [] => println!("empty"),
}
match v.as_slice() {
    [init @ .., last] => println!("init {init:?} last {last}"),
    [] => println!("empty"),
}
match v.as_slice() {
    [a, b, mid @ .., z] => println!("{a} {b} mid={mid:?} {z}"),
    _ => println!("too short"),
}
```

Output:

```
head 1 tail [2, 3, 4, 5]
init [1, 2, 3, 4] last 5
1 2 mid=[3, 4] 5
```

Only one `..` per slice pattern — otherwise the split is ambiguous:

```rust
match v.as_slice() {
    [.., mid @ .., ..] => println!("{mid:?}"),
    _ => {}
}
```

Does not compile — intentional:

```
error: `..` can only be used once per slice pattern
 --> src/main.rs:4:20
  |
4 |         [.., mid @ .., ..] => println!("{mid:?}"),
  |          --        ^^ can only be used once per slice pattern
  |          |
  |          previously used here
```

Literals work in slice positions too:

```rust
match v.as_slice() {
    [1, rest @ ..] => println!("starts with 1, rest {rest:?}"),
    _ => println!("other"),
}
match v.as_slice() {
    [_, _, third, ..] => println!("third is {third}"),
    _ => println!("too short"),
}
```

Output:

```
starts with 1, rest [2, 3, 4, 5]
third is 3
```

This makes parsing split output pleasant:

```rust
let words = ["get", "/users", "HTTP/1.1"];
match &words[..] {
    [method, path, version] => println!("{method} {path} ({version})"),
    [method, path] => println!("{method} {path}"),
    _ => println!("malformed"),
}
```

Output:

```
get /users (HTTP/1.1)
```

## `_` vs `..` vs `_name`

| Form | Meaning |
|---|---|
| `_` | exactly one value, ignored, **not moved** |
| `..` | zero or more values/fields, ignored |
| `_name` | binds and **does** move; suppresses the unused warning |

The move difference is real:

```rust
let s = Some(String::from("x"));
let _ = s;
println!("{s:?}");
let t = Some(String::from("y"));
let _unused = t;
println!("{t:?}");
```

Does not compile — intentional, and only the second half is the problem:

```
error[E0382]: borrow of moved value: `t`
 --> src/main.rs:7:16
  |
5 |     let t = Some(String::from("y"));
  |         - move occurs because `t` has type `Option<String>`, which does not implement the `Copy` trait
6 |     let _unused = t;
  |                   - value moved here
7 |     println!("{t:?}");
  |                ^ value borrowed here after move
```

`let _ = s;` was fine. `let _unused = t;` moved. Use bare `_` when you mean
"discard".

## Nested destructuring

Structs inside structs inside enums inside tuples — it all composes.

```rust
struct Addr { city: String, zip: String }
struct Person { name: String, addr: Addr, tags: Vec<String> }

let p = Person {
    name: "raj".into(),
    addr: Addr { city: "Pune".into(), zip: "411001".into() },
    tags: vec!["admin".into(), "dev".into()],
};

let Person { name, addr: Addr { city, .. }, tags } = &p;
println!("{name} in {city} with {} tags", tags.len());

match &p {
    Person { addr: Addr { city, .. }, tags, .. } if tags.iter().any(|t| t == "admin") => {
        println!("admin in {city}")
    }
    Person { name, .. } => println!("plain {name}"),
}
```

Output:

```
raj in Pune with 2 tags
admin in Pune
```

Enums with mixed payload shapes:

```rust
enum Event {
    Click { at: (i32, i32) },
    Key(char),
    Resize(u32, u32),
}

let events = vec![
    Event::Click { at: (10, 20) },
    Event::Key('q'),
    Event::Resize(800, 600),
    Event::Click { at: (0, 0) },
];
for e in &events {
    match e {
        Event::Click { at: (0, 0) } => println!("click at origin"),
        Event::Click { at: (x, y) } => println!("click {x},{y}"),
        Event::Key('q') => println!("quit key"),
        Event::Key(c) => println!("key {c}"),
        Event::Resize(w, h) if w > h => println!("landscape {w}x{h}"),
        Event::Resize(w, h) => println!("portrait {w}x{h}"),
    }
}
```

Output:

```
click 10,20
quit key
landscape 800x600
click at origin
```

Patterns inside `Option` inside a tuple inside a slice:

```rust
let data = vec![Some((1, "a")), None, Some((2, "b"))];
for d in &data {
    match d {
        Some((n, s)) if *n > 1 => println!("big {n}{s}"),
        Some((n, s)) => println!("small {n}{s}"),
        None => println!("gap"),
    }
}

let v: Vec<Option<i32>> = vec![Some(1), None, Some(3)];
match v.as_slice() {
    [Some(a), None, Some(b)] => println!("{a} gap {b}"),
    _ => println!("other"),
}
```

Output:

```
small 1a
gap
big 2b
1 gap 3
```

Note `*n > 1`: matching `&Option<(i32, &str)>` makes `n` a `&i32`, so the guard
dereferences.

## ref and ref mut

Match ergonomics (lesson 3.4) means you rarely need these, but they appear in
older code and in cases where you match a value yet want a borrow of one part.

```rust
let maybe = Some(String::from("hi"));
match maybe {
    Some(ref s) => println!("borrowed {s}"),
    None => {}
}
println!("still {maybe:?}");
```

Output:

```
borrowed hi
still Some("hi")
```

Without `ref`, `Some(s)` would have moved the `String` out and the final
`println!` would be E0382. `match &maybe { Some(s) => .. }` is the modern
equivalent.

`ref mut` borrows mutably:

```rust
let mut m = Some(5);
match m {
    Some(ref mut n) => *n += 1,
    None => {}
}
println!("{m:?}");
```

Output:

```
Some(6)
```

In a `let`, it lets you move one part and borrow another: `let (n, ref s) = t;`
on a `(i32, String)` gives `n: i32` and `s: &String`.

Idiomatic: prefer `match &x` / `match &mut x`. Reach for `ref` only for partial
borrows like the one above.

## Refutable vs irrefutable

An **irrefutable** pattern always matches: `x`, `(a, b)`, `Point { x, y }`,
`[a, b, c]` on `[T; 3]`. A **refutable** pattern might fail: `Some(n)`,
`Ok(v)`, `3`, `[a, b]` on a slice, `'a'..='z'`.

`let` and function parameters require irrefutable patterns:

```rust
let o: Option<i32> = Some(3);
let Some(n) = o;
```

Does not compile — intentional:

```
error[E0005]: refutable pattern in local binding
 --> src/main.rs:3:9
  |
3 |     let Some(n) = o;
  |         ^^^^^^^ pattern `None` not covered
  |
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
help: you might want to use `let...else` to handle the variant that isn't matched
  |
3 |     let Some(n) = o else { todo!() };
  |                     ++++++++++++++++
```

The compiler points straight at `let ... else`, which is lesson 3.6.

Conversely, `if let`/`while let` expect a refutable pattern; an irrefutable one
is pointless:

```rust
let x = 5;
if let n = x {
    println!("{n}");
}
```

```
warning: irrefutable `if let` pattern
 --> src/main.rs:3:8
  |
3 |     if let n = x {
  |        ^^^^^^^^^
  |
  = note: this pattern will always match, so the `if let` is useless
  = help: consider replacing the `if let` with a `let`
```

| Position | Pattern must be | Refutable allowed? |
|---|---|---|
| `let p = v;` | irrefutable | no (E0005) |
| `fn f(p: T)` | irrefutable | no |
| closure params | irrefutable | no |
| `for p in it` | irrefutable | no |
| `match` arm | either | yes |
| `if let` / `while let` / `let else` | refutable | yes (irrefutable warns) |

## Patterns in function parameters

A parameter is a pattern, so it can destructure directly.

```rust
fn dist(Point { x, y }: &Point) -> f64 {
    ((x * x + y * y) as f64).sqrt()
}

fn sum_pair((a, b): (i32, i32)) -> i32 { a + b }

fn fst(&(a, _): &(i32, i32)) -> i32 { a }

println!("{:.3}", dist(&Point { x: 3, y: 4 }));
println!("{}", sum_pair((2, 3)));
println!("{}", fst(&(9, 10)));
```

Output:

```
5.000
5
9
```

`&(a, _)` destructures *through* the reference, so `a` is `i32` rather than
`&i32`. Use sparingly — the signature stops naming its arguments, which hurts
docs and call sites.

## Patterns in for loops

```rust
let pairs = vec![(1, 'a'), (2, 'b')];
for (n, c) in &pairs { print!("{n}{c} "); }
for &(n, c) in &pairs { print!("{n}{c} "); }
println!();

let points = vec![Point { x: 1, y: 2 }, Point { x: 3, y: 4 }];
for Point { x, y } in &points { print!("({x},{y}) "); }
println!();
```

Output:

```
1a 2b 1a 2b 
(1,2) (3,4) 
```

The first loop binds `n: &i32`, the second `n: i32`. Same printed result,
different types.

## Patterns in closures

```rust
let pts = vec![(1, 2), (3, 4)];
let sums: Vec<i32> = pts.iter().map(|(a, b)| a + b).collect();
let prods: Vec<i32> = pts.iter().map(|&(a, b)| a * b).collect();
println!("{sums:?} {prods:?}");
let f = |Point { x, y }: &Point| x + y;
println!("{}", f(&Point { x: 10, y: 20 }));
```

Output:

```
[3, 7] [2, 12]
30
```

`|(a, b)|` is why `.map` over a map or a zipped iterator reads so cleanly.
Module 10 covers closures.

## Quick Reference

| Pattern | Example | Notes |
|---|---|---|
| Struct | `Point { x, y }` | all fields, or add `..` |
| Struct rename | `Point { x: a }` | |
| Struct partial | `User { name, .. }` | `..` last |
| Struct field test | `Point { x: 0, y }` | match-only |
| Tuple | `(a, b, c)` | |
| Tuple partial | `(first, .., last)` | |
| Array | `[a, b, c]` | irrefutable on `[T; 3]` |
| Slice length | `[]`, `[x]`, `[a, b]` | refutable |
| Slice ends | `[first, .., last]` | needs len >= 2 |
| Rest binding | `[head, tail @ ..]` | one `..` per pattern |
| Ignore one | `_` | does not move |
| Ignore many | `..` | |
| Bind + silence | `_name` | **does** move |
| Borrow in pattern | `ref s` / `ref mut s` | prefer `match &x` |
| Deref in pattern | `&(a, b)` | binds by value |
| Nested | `Some((n, Point { x, .. }))` | any depth |

## Common Mistakes

**`let Some(x) = opt;`** E0005 — refutable in a `let`. Use `if let`, `match`, or
`let ... else` (lesson 3.6).

**`let _x = big;` to silence a warning.** That moves `big`. Use `let _ = big;`
or `drop(big)`.

**Listing only some struct fields without `..`.** Every field must be accounted
for. Add `..`.

**Two `..` in one slice pattern.** Ambiguous and rejected. Restructure to one.

**Assuming `[first, .., last]` matches a one-element slice.** It needs at least
two. Add `[only]`.

**Forgetting `*` in a guard.** Matching `&Option<i32>` binds `n: &i32`, so
`n > 1` fails to compile; write `*n > 1`.

## Exercises

1. Define `struct Rect { w: f64, h: f64, label: String }` and write a `let` that
   destructures only `w` and `h`.
2. Write `fn classify(v: &[i32]) -> String` using slice patterns for empty, one,
   two, and "first N last" cases, naming the middle with `mid @ ..`.
3. Take `"GET /users/42 HTTP/1.1"`, split on whitespace, collect to a `Vec`, and
   match `&v[..]` to pull out method, path, and version.
4. Write `fn midpoint(&(x1, y1): &(f64, f64), &(x2, y2): &(f64, f64)) -> (f64,
   f64)` with the destructuring entirely in the parameters.
5. Given `Vec<(String, Option<u32>)>`, use one `for` with a destructuring
   pattern plus `match` to print `"name: value"` or `"name: unknown"`.
6. Write code that triggers E0005, then fix it three ways: `match`, `if let`,
   and `let ... else`.

---

[← Course Index](../README.md)
