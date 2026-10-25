[← Course Index](../README.md)

# 3.1 — if, else if, and if as an Expression

## What You'll Learn

- `if` takes no parens, requires braces, and demands a real `bool`
- `if` is an expression: it produces a value you can bind or print
- Both branches must have the same type, and a missing `else` means `()`
- There is no ternary operator, and why you don't miss it

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `if (x > 3) { }` | `if x > 3 { }` |
| JS `if (name)` (truthy) | `if !name.is_empty()` — no truthiness |
| JS `cond ? a : b` | `if cond { a } else { b }` |
| Go `if x > 3 { }` | identical shape, but `if` returns a value |

Go's parenthesis-free, brace-mandatory style, plus one thing neither JS nor Go
has: `if` is an **expression**, so `let x = if c { 1 } else { 2 };` works.

## Basic form

No parentheses around the condition. Braces are never optional.

```rust
let n = 7;

if n % 2 == 0 {
    println!("{n} is even");
} else {
    println!("{n} is odd");
}
```

Output:

```
7 is odd
```

Dropping the braces is a parse error, not a style warning:

```rust
if n > 3 println!("big");
```

Does not compile — intentional:

```
error: expected `{`, found `println`
 --> src/main.rs:3:14
  |
3 |     if n > 3 println!("big");
  |              ^^^^^^^ expected `{`
  |
note: the `if` expression is missing a block after this condition
help: you might have meant to write this as part of a block
  |
3 |     if n > 3 { println!("big") };
  |              +                 +
```

Parens around the condition compile fine but Clippy flags them
(`unused_parens`). Don't write them.

## The condition must be a bool

No truthiness. Not zero, not empty string, not null — only `bool`.

```rust
if 1 {
    println!("yes");
}
```

Does not compile — intentional:

```
error[E0308]: mismatched types
 --> src/main.rs:2:8
  |
2 |     if 1 {
  |        ^ expected `bool`, found integer
```

Write the comparison you actually mean.

```rust
let opt: Option<i32> = Some(3);
if opt.is_some() {
    println!("got {}", opt.unwrap());
}

let v = vec![1, 2, 3];
if !v.is_empty() {
    println!("len {}", v.len());
}
```

Output:

```
got 3
len 3
```

`if opt.is_some()` then `unwrap()` is a beginner smell; lesson 3.6 replaces it
with `if let`.

## else if chains

Any number of `else if`, at most one `else`, and they are checked top to bottom.

```rust
let score = 85;
if score >= 90 {
    println!("A");
} else if score >= 80 {
    println!("B");
} else if score >= 70 {
    println!("C");
} else {
    println!("F");
}
```

Output:

```
B
```

`else` is optional even with `else if` present.

```rust
let temp = 15;
if temp < 0 {
    println!("freezing");
} else if temp < 20 {
    println!("cool");
}
```

Output:

```
cool
```

Three or more `else if` arms on the same value usually want `match` (lesson
3.4) — it checks exhaustiveness for you.

## if as an expression

The last expression of each block is the value of the `if`.

```rust
let count = 3;
let label = if count == 1 { "item" } else { "items" };
println!("{count} {label}");
```

Output:

```
3 items
```

Note the `;` at the end — that terminates the `let` statement, not the `if`.
The block arms have no semicolons inside; `{ "item" }` evaluates to `"item"`,
while `{ "item"; }` would evaluate to `()`.

Multi-arm works the same:

```rust
let n = -4;
let sign = if n > 0 {
    1
} else if n < 0 {
    -1
} else {
    0
};
println!("{sign}");
```

Output:

```
-1
```

Blocks can do real work; only the trailing expression is the value.

```rust
let x = 10;
let msg = if x > 5 {
    let doubled = x * 2;
    format!("big: {doubled}")
} else {
    String::from("small")
};
println!("{msg}");
```

Output:

```
big: 20
```

## Both arms must share a type

```rust
let n = 5;
let x = if n > 3 { "big" } else { 0 };
```

Does not compile — intentional:

```
error[E0308]: `if` and `else` have incompatible types
 --> src/main.rs:3:39
  |
3 |     let x = if n > 3 { "big" } else { 0 };
  |                        -----          ^ expected `&str`, found integer
  |                        |
  |                        expected because of this
```

## A missing else is a unit value

An `if` with no `else` evaluates to `()`. That's fine as a statement, fatal as
a value.

```rust
let n = 5;
let x = if n > 3 { "big" };
```

Does not compile — intentional:

```
error[E0317]: `if` may be missing an `else` clause
 --> src/main.rs:3:13
  |
3 |     let x = if n > 3 { "big" };
  |             ^^^^^^^^^^^-----^^
  |             |          |
  |             |          found here
  |             expected `&str`, found `()`
  |
  = note: `if` expressions without `else` evaluate to `()`
  = help: consider adding an `else` block that evaluates to the expected type
```

When both arms genuinely are `()`, the value is `()` and binding it is legal
(if pointless):

```rust
let t = 5;
let r = if t > 3 { () } else { () };
println!("{r:?}");
```

Output:

```
()
```

## if as a function's return value

Drop the `return` and the semicolon; the `if` is the tail expression.

```rust
fn classify(n: i32) -> &'static str {
    if n < 0 {
        "negative"
    } else if n == 0 {
        "zero"
    } else {
        "positive"
    }
}

println!("{} {} {}", classify(-5), classify(0), classify(5));
```

Output:

```
negative zero positive
```

`return` still exists for early exits, where there is no `else`:

```rust
fn abs_early(n: i32) -> i32 {
    if n < 0 {
        return -n;
    }
    n
}

println!("{} {}", abs_early(-7), abs_early(7));
```

Output:

```
7 7
```

## if inside an argument or format string

Because it's an expression, an `if` goes anywhere a value goes.

```rust
let a = 3;
let b = 9;
println!("max is {}", if a > b { a } else { b });

let n = 4;
println!("{n} is {}", if n % 2 == 0 { "even" } else { "odd" });
```

Output:

```
max is 9
4 is even
```

The `if` must be an argument, not inlined into the literal: `println!("{if
..}")` is not a thing. Inline `{name}` capture only takes identifiers.

## Boolean operators

`&&` and `||` short-circuit; `!` negates. No `and`/`or` keywords.

```rust
let logged_in = true;
let admin = false;

if logged_in && !admin {
    println!("user");
}
if logged_in || admin {
    println!("has access");
}
```

Output:

```
user
has access
```

Combine freely, with parens for grouping, not for the condition itself:

```rust
let p = (1, -1);
if p.0 > 0 && p.1 > 0 {
    println!("Q1");
} else if p.0 > 0 && p.1 < 0 {
    println!("Q4");
} else {
    println!("elsewhere");
}
```

Output:

```
Q4
```

## Nesting

Nests arbitrarily, same as anywhere.

```rust
let outer = 5;
if outer > 0 {
    if outer % 5 == 0 {
        println!("positive multiple of 5");
    }
}
```

Output:

```
positive multiple of 5
```

Idiomatic Rust flattens this to `if outer > 0 && outer % 5 == 0`. Deeply nested
`if` on one value is the signal to reach for `match`.

## No ternary operator

`cond ? a : b` does not exist and never will — `if cond { a } else { b }` is
the replacement, and it's barely longer.

```rust
let flag = true;
let v = if flag { 1 } else { 2 };
println!("{v}");
```

Output:

```
1
```

For a `bool` specifically there are a few shorter spellings. `then_some` takes
a value, `then` takes a closure (so the value is computed only if true), and
`u8::from(bool)` gives 1 or 0.

```rust
let v2: i32 = flag.then_some(1).unwrap_or(2);
let v3 = u8::from(flag);
println!("{v2} {v3}");
```

Output:

```
1 1
```

The laziness difference is real:

```rust
fn expensive() -> i32 {
    println!("computing...");
    99
}

let cond = false;
let lazy = if cond { expensive() } else { 0 };
let lazy2 = cond.then(expensive).unwrap_or(0);
println!("{lazy} {lazy2}");
```

Output:

```
0 0
```

Neither printed `computing...` — `if` arms and `then`'s closure are both lazy.
`then_some(expensive())` *would* have printed it.

Idiomatic: plain `if`/`else`. Reach for `then`/`then_some` only when chaining
into other `Option` combinators (lesson 8.4).

## if vs match on a value

When you're branching on *what a value is* rather than an arbitrary boolean,
`match` is shorter and checked.

```rust
let x = 5;
let parity = match x % 2 {
    0 => "even",
    _ => "odd",
};
println!("{parity}");
```

Output:

```
odd
```

## Quick Reference

| Form | Syntax | Notes |
|---|---|---|
| Statement | `if c { .. }` | no parens, braces required |
| With else | `if c { .. } else { .. }` | |
| Chain | `if c1 { } else if c2 { } else { }` | top-to-bottom |
| Expression | `let x = if c { a } else { b };` | arms must share a type |
| Tail return | `fn f() -> T { if c { a } else { b } }` | no `return`, no `;` |
| Inline argument | `println!("{}", if c { a } else { b })` | |
| No else | `if c { .. }` as a value is `()` | E0317 if bound |
| Early return | `if c { return v; }` | |
| Ternary stand-in | `if c { a } else { b }` | no `?:` in Rust |
| bool to Option | `c.then_some(v)` / `c.then(\|\| v)` | `then` is lazy |
| bool to int | `u8::from(c)` | 1 or 0 |

## Common Mistakes

**Writing `if (x > 3)`.** Legal, but Clippy warns. Drop the parens.

**Expecting truthiness.** `if 1 {}`, `if some_string {}`, and `if vec {}` are
all E0308. Write `!= 0`, `!s.is_empty()`, `!v.is_empty()`.

**Semicolon inside an arm.** `let x = if c { 1; } else { 2; };` makes both arms
`()`. Remove the inner semicolons.

**Mismatched arm types.** `if c { "a" } else { 0 }` is E0308. Both arms need
one type; use `match` or convert explicitly.

**Omitting `else` when the `if` is a value.** E0317 tells you `()` was found.
Add the `else`.

## Exercises

1. Write `fn fizzbuzz_word(n: u32) -> String` using an `else if` chain that
   returns `"Fizz"`, `"Buzz"`, `"FizzBuzz"`, or the number as a string.
2. Rewrite `let x; if c { x = 1 } else { x = 2 }` as a single `let` binding
   with an `if` expression.
3. Write code that fails with E0317, then with E0308 on the arms, and read both
   errors.
4. Given `let age: u32`, print `"minor"`, `"adult"`, or `"senior"` using exactly
   one `println!` with the `if` inline in the argument list.
5. Given `let name: String`, print a greeting only if the name is non-empty —
   without using truthiness. Then do the same with `then_some` and
   `unwrap_or`.

---

[← Course Index](../README.md)
