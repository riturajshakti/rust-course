[← 2.3 Tuples and Arrays](03-tuples-arrays.md) · [Course Index](../README.md) · [2.5 Numeric Operators and Type Casting →](05-operators-casting.md)

# 2.4 — Functions, Parameters, and Return Values

## What You'll Learn

- Signatures are always fully annotated; bodies almost never are
- The statement/expression rule, and how one semicolon changes what you return
- `return` for early exit, implicit tail expression for the normal case
- Nested functions, `fn` pointers, diverging `!`, and `const fn`
- No overloading, no default arguments — and what to write instead

## Coming From Another Language

| Other | Rust |
|---|---|
| TS `function f(a: number): number` | `fn f(a: i32) -> i32` |
| Python `def f(a, b=1)` | no default args — see below |
| Go `func f(a int) (int, error)` | `fn f(a: i32) -> Result<i32, E>` |
| JS hoisting | also hoisted: order of `fn` items doesn't matter |

Two real differences: every parameter and return type is mandatory, and the
last expression in the body *is* the return value — no `return` keyword needed.

## Basic forms

```rust
fn main() {
    greet();
    println!("{}", add(2, 3));
    println!("{}", five());
    println!("{:?}", nothing());
}

fn greet() {
    println!("hi");
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn five() -> i32 {
    5
}

fn nothing() {}
```

Output:

```
hi
5
5
()
```

`snake_case` by convention; clippy warns otherwise. Definition order is
irrelevant — `main` can call something defined below it.

## Signatures are never inferred

```rust
fn double(x) { x * 2 }
```

Does not compile:

```
error: expected one of `:`, `@`, or `|`, found `)`
 --> src/main.rs:1:12
  |
1 | fn double(x) { x * 2 }
  |            ^ expected one of `:`, `@`, or `|`
  |
help: if this is a parameter name, give it a type
  |
1 | fn double(x: TypeName) { x * 2 }
  |            ++++++++++
```

Inference stops at the `fn` boundary in both directions: an argument is never
adjusted to fit, which is why `dist((0, 0), ...)` below needs `0.0`.

## Statements vs expressions

A **statement** performs an action and yields nothing. An **expression**
evaluates to a value. `let x = 5;` is a statement; `5`, `x + 1`, `if c {a} else {b}`,
and `{ ... }` are expressions. A semicolon turns an expression into a
statement, discarding its value.

```rust
let x = (let y = 6);
```

```
error: expected expression, found `let` statement
 --> src/main.rs:2:14
  |
2 |     let x = (let y = 6);
  |              ^^^
  |
  = note: only supported directly in conditions of `if` and `while` expressions
```

So no `a = b = c` chains.

## One semicolon changes the return value

```rust
fn five() -> i32 {
    5
}
```

Returns `5`. Add a semicolon:

```rust
fn five() -> i32 {
    5;
}
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:1:14
  |
1 | fn five() -> i32 {
  |    ----      ^^^ expected `i32`, found `()`
  |    |
  |    implicitly returns `()` as its body has no tail or `return` expression
2 |     5;
  |      - help: remove this semicolon to return this value
```

This is the single most common beginner error in Rust, and the message always
points at it. Read "found `()`" as "you put a semicolon somewhere".

The missing-semicolon direction is a parse error instead:

```rust
let x = 5
println!("{x}");
```

```
error: expected `;`, found `println`
 --> src/main.rs:2:14
  |
2 |     let x = 5
  |              ^ help: add `;` here
```

## return for early exit

```rust
fn classify(n: i32) -> &'static str {
    if n < 0 {
        return "negative";
    }
    if n == 0 { "zero" } else { "positive" }
}

println!("{}", classify(5));
println!("{}", classify(-5));
```

Output:

```
positive
negative
```

`return` is for leaving early — guard clauses, error paths. For the final value
use the tail expression; `return x;` on the last line is valid but clippy flags
it as `needless_return`.

## Unit return

These three are identical:

```rust
fn a() {}
fn b() -> () {}
fn c() -> () { () }
```

```rust
fn explicit_unit() -> () { }
println!("{:?}", explicit_unit());
```

Output:

```
()
```

Nobody writes `-> ()`. Note `()` has no `Display`, so it needs `{:?}`.

## Parameter patterns

A parameter is a pattern, so it can destructure.

```rust
fn dist((x1, y1): (f64, f64), (x2, y2): (f64, f64)) -> f64 {
    ((x2 - x1).powi(2) + (y2 - y1).powi(2)).sqrt()
}

println!("{}", dist((0.0, 0.0), (3.0, 4.0)));
```

Output:

```
5
```

`_name` marks a parameter you must accept but do not use, silencing the warning:

```rust
fn ignore(_n: i32) {}
```

## Mutable and borrowed parameters

```rust
fn takes_mut(v: &mut i32) { *v = 7; }

let mut t = 0;
takes_mut(&mut t);
println!("{t}");
```

Output:

```
7
```

Parameters are immutable bindings unless written `mut x: T` (a local copy you
may reassign) or taken as `&mut T` (the caller's value). Module 04 covers
borrowing in full; prefer `&[T]` and `&str` over `&Vec<T>` and `&String` in
parameters.

## Nested functions

```rust
fn outer(n: i32) -> i32 {
    fn double(x: i32) -> i32 { x * 2 }
    double(n) + 1
}

println!("{}", outer(4));
```

Output:

```
9
```

A nested `fn` is just a private item scoped to that body. It **cannot** capture
locals:

```rust
let n = 5;
fn show() { println!("{n}"); }
```

```
error[E0434]: can't capture dynamic environment in a fn item
 --> src/main.rs:3:28
  |
3 |     fn show() { println!("{n}"); }
  |                            ^
  |
  = help: use the `|| { ... }` closure form instead
```

Capturing is what closures are for — module 10.

## Functions as values

A function item coerces to a `fn` pointer, which can be stored and passed.

```rust
fn add_one(x: i32) -> i32 { x + 1 }
fn add(a: i32, b: i32) -> i32 { a + b }

fn apply(f: fn(i32) -> i32, v: i32) -> i32 { f(v) }

println!("{}", apply(add_one, 10));

let f: fn(i32, i32) -> i32 = add;
println!("{}", f(1, 1));
```

Output:

```
11
2
```

A generic bound accepts closures too, which `fn` pointers alone do not:

```rust
fn apply2<F: Fn(i32) -> i32>(f: F, v: i32) -> i32 { f(v) }
println!("{}", apply2(|x| x * 3, 10));
```

Output:

```
30
```

Prefer `impl Fn(..)` or a generic bound in APIs; `fn(..)` only when you
specifically need a plain function pointer (FFI, a jump table).

## Diverging functions: `!`

`-> !` means "never returns". The never type unifies with any type, so a
diverging call can sit where a value is expected.

```rust
fn give_up() -> ! {
    panic!("no label")
}

fn pick(n: i32) -> &'static str {
    if n > 0 { "pos" } else { give_up() }
}

println!("{}", pick(1));
```

Output:

```
pos
```

With `n = -1`:

```
thread 'main' (13798467) panicked at src/main.rs:8:5:
no label
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`panic!`, `std::process::exit`, `unreachable!`, `todo!`, `loop {}` without a
`break`, and `continue`/`return` are all `!`. That is why a `match` arm can
`panic!` while its siblings produce an `i32`:

```rust
fn forever_guard(n: i32) -> i32 {
    match n {
        0 => 100,
        _ => loop { break 7 },
    }
}
println!("{}", forever_guard(0));
```

Output:

```
100
```

A `loop` *with* a `break` has the break value's type, not `!`.

## const fn

A `const fn` can be evaluated at compile time, so it can initialize a `const`.

```rust
const fn square(x: i32) -> i32 { x * x }
const SQUARE_OF_4: i32 = square(4);

println!("{}", SQUARE_OF_4);
```

Output:

```
16
```

It is callable at runtime too, with no penalty. The body is restricted to
operations the const evaluator supports. Lesson 2.6 covers `const`.

## No overloading

```rust
fn f(a: i32) -> i32 { a }
fn f(a: f64) -> f64 { a }
```

Does not compile:

```
error[E0428]: the name `f` is defined multiple times
 --> src/main.rs:2:1
  |
1 | fn f(a: i32) -> i32 { a }
  | ------------------- previous definition of the value `f` here
2 | fn f(a: f64) -> f64 { a }
  | ^^^^^^^^^^^^^^^^^^^ `f` redefined here
```

Use a generic with a trait bound instead:

```rust
fn show<T: std::fmt::Debug>(v: T) {
    println!("{v:?}");
}

show(1);
show("x");
show(vec![1, 2]);
```

Output:

```
1
"x"
[1, 2]
```

Or give the variants distinct names (`new`, `with_capacity`,
`from_str_radix`) — which is what the standard library does.

## No default arguments

Three idiomatic workarounds, in order of how often you will see them.

An options struct with `Default`:

```rust
#[derive(Debug, Default)]
struct Opts { retries: u32, verbose: bool }

fn connect(host: &str, opts: Opts) -> String {
    format!("{host} retries={} verbose={}", opts.retries, opts.verbose)
}

fn connect_default(host: &str) -> String {
    connect(host, Opts::default())
}

println!("{}", connect_default("a.com"));
println!("{}", connect("b.com", Opts { retries: 3, verbose: true }));
```

Output:

```
a.com retries=0 verbose=false
b.com retries=3 verbose=true
```

An `Option` parameter, when there is only one optional value:

```rust
fn describe(v: Option<&str>) -> &str {
    match v {
        Some(s) => s,
        None => "<none>",
    }
}

println!("{} {}", describe(Some("hi")), describe(None));
```

Output:

```
hi <none>
```

Or a builder (lesson 5.7) once there are several optional fields.

## Quick Reference

| Form | Meaning |
|---|---|
| `fn f() {}` | No params, returns `()` |
| `fn f(a: T) -> U {}` | Params and return type are mandatory |
| `a + b` (no semicolon) | Tail expression — the return value |
| `a + b;` | Statement — returns `()` instead |
| `return v;` | Early exit |
| `fn f(_x: T)` | Accept and ignore a parameter |
| `fn f((a, b): (T, U))` | Destructuring parameter |
| `fn f(v: &mut T)` | Mutate the caller's value |
| `fn f() -> !` | Never returns |
| `const fn f() -> T` | Usable in a `const` initializer |
| `f: fn(T) -> U` | Function pointer parameter |
| `F: Fn(T) -> U` | Accepts closures too |
| `fn f<T: Trait>(v: T)` | The stand-in for overloading |
| `Opts::default()` | The stand-in for default arguments |

## Common Mistakes

**Trailing semicolon on the return expression.** `5;` returns `()`; the
`expected i32, found ()` error always means this.

**`return` on the last line.** Valid, but clippy flags it. Drop the keyword and
the semicolon.

**Expecting a nested `fn` to see locals.** It cannot capture; use a closure.

**Looking for overloads or default args.** Use generics plus trait bounds, and
an options struct with `Default`.

**Taking `&String` or `&Vec<T>`.** Take `&str` and `&[T]` — strictly more
callers, same cost.

## Exercises

1. Write `fn area(w: f64, h: f64) -> f64` with no `return`. Add a semicolon to
   the last line and read the error.
2. Write `fn fizz(n: u32) -> &'static str` returning `"fizz"`, `"buzz"`,
   `"fizzbuzz"`, or `"none"`, using an early `return` for one case and a tail
   expression for the rest.
3. Write `fn pipeline(v: i32, fs: &[fn(i32) -> i32]) -> i32` that applies every
   function in order; call it with three named functions.
4. Write `fn must(v: Option<i32>) -> i32` that returns the value or calls a
   `-> !` helper that panics with a message. Run both paths.
5. Add `Opts { width: usize, pad: char }` with `Default`, and a
   `fn render(s: &str, opts: Opts) -> String`. Call it once with defaults and
   once with both fields set.

---

[← 2.3 Tuples and Arrays](03-tuples-arrays.md) · [Course Index](../README.md) · [2.5 Numeric Operators and Type Casting →](05-operators-casting.md)
