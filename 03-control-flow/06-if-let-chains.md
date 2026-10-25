[← Course Index](../README.md)

# 3.6 — if let, while let, and let else

## What You'll Learn

- `if let` for the one-case match, with `else` and let-chains
- `let ... else` for early return — the shape that keeps functions flat
- `while let` for draining a stack, queue, or iterator
- `matches!` for a pattern test that yields a `bool`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `if (x != null) { use(x) }` | `if let Some(x) = opt { use(x) }` |
| Go `if v, ok := m[k]; ok { }` | `if let Some(v) = m.get(k) { }` |
| Go `v, err := f(); if err != nil { return }` | `let Ok(v) = f() else { return };` |
| JS `while ((m = re.exec(s)))` | `while let Some(m) = it.next() { }` |

All four are `match` with the boring arm removed. `let ... else` is the one
worth internalising: it is Rust's answer to Go's early-return style.

## if let

`match` when you only care about one arm.

```rust
let config: Option<&str> = Some("debug");
if let Some(mode) = config {
    println!("mode {mode}");
}
```

Output:

```
mode debug
```

That is exactly `match config { Some(mode) => println!("mode {mode}"), None =>
{} }`. `if let` loses exhaustiveness checking — which is the point when there's
nothing to do in the other case.

The binding is scoped to the block:

```rust
let o = Some(5);
if let Some(n) = o {
    println!("{n}");
}
println!("{n}");
```

Does not compile — intentional:

```
error[E0425]: cannot find value `n` in this scope
 --> src/main.rs:6:16
  |
6 |     println!("{n}");
  |                ^
```

## if let with else

```rust
let none: Option<i32> = None;
if let Some(n) = none {
    println!("got {n}");
} else {
    println!("nothing");
}
```

Output:

```
nothing
```

`else if let` chains too: `if let Some(a) = x { } else if let Ok(b) = y { }
else { }`. Past two or three, use `match`.

## if let on Result and other patterns

Any refutable pattern works, not just `Option`.

```rust
let r: Result<i32, String> = "7".parse::<i32>().map_err(|e| e.to_string());
if let Ok(n) = &r {
    println!("ok {n}");
}
if let Err(e) = &r {
    println!("err {e}");
}
```

Output:

```
ok 7
```

Literals, ranges, tuples, and slices are all fair game:

```rust
let n = 5;
if let 1..=9 = n {
    println!("single digit");
}
let p = (0, 3);
if let (0, y) = p {
    println!("on y at {y}");
}
let v = vec![1, 2, 3];
if let [first, ..] = v.as_slice() {
    println!("first {first}");
}
```

Output:

```
single digit
on y at 3
first 1
```

`if let 1..=9 = n` reads oddly — the pattern is on the left of `=`. Prefer
`matches!` (below) for a pure test.

## if let as an expression

Both branches must share a type, same rule as `if`.

```rust
let config = Some("debug");
let msg = if let Some(m) = config { m.to_uppercase() } else { "NONE".to_string() };
println!("{msg}");
```

Output:

```
DEBUG
```

## References and mutation

`&opt` borrows; `&mut opt` lets you write through the binding.

```rust
let name = Some(String::from("raj"));
if let Some(s) = &name {
    println!("len {}", s.len());
}
println!("still {name:?}");

let mut cnt = Some(0);
if let Some(c) = &mut cnt {
    *c += 10;
}
println!("{cnt:?}");
```

Output:

```
len 3
still Some("raj")
Some(10)
```

Without the `&`, `if let Some(s) = name` moves the `String` out and `name` is
dead afterwards.

## Nested if let, and let-chains

Nesting works but reads badly:

```rust
let nested: Option<Option<i32>> = Some(Some(3));
if let Some(inner) = nested {
    if let Some(n) = inner {
        println!("deep {n}");
    }
}
if let Some(Some(n)) = nested {
    println!("flat {n}");
}
```

Output:

```
deep 3
flat 3
```

When the values are *independent*, a nested pattern can't help — chain the
`let`s with `&&` instead. This is a let-chain, and it is stable in edition
2024.

```rust
let a = Some(1);
let b = Some(2);
if let Some(x) = a && let Some(y) = b {
    println!("{x} {y}");
}
```

Output:

```
1 2
```

Bindings from earlier links are visible in later ones, so you get guards for
free:

```rust
let c = Some(5);
if let Some(n) = c && n > 3 {
    println!("big {n}");
}
```

Output:

```
big 5
```

Let-chains require edition 2024. In an older edition the same code is a parse
error, and you nest instead.

## let ... else

This is the one to learn properly. `let PATTERN = expr else { ... };` binds on
success, and the `else` block runs on failure — but the `else` block **must
diverge**: `return`, `break`, `continue`, `panic!`, or `std::process::exit`.

```rust
let data = Some(42);
let Some(n) = data else {
    panic!("required");
};
println!("n is in scope: {n}");
```

Output:

```
n is in scope: 42
```

The payoff: `n` is in scope for the **rest of the function**, at the outer
indentation level. No block, no nesting.

```rust
fn greet(name: Option<&str>) -> String {
    let Some(n) = name else {
        return "hello, stranger".to_string();
    };
    format!("hello, {n}")
}

println!("{}", greet(Some("raj")));
println!("{}", greet(None));
```

Output:

```
hello, raj
hello, stranger
```

The `match` version puts the happy path inside an arm, indented. With one check
that's a wash; with three or four in a row, `let ... else` is dramatically
flatter — the whole reason it exists.

```rust
fn sum_two(input: &str) -> Result<i32, String> {
    let parts: Vec<&str> = input.split(',').collect();
    let [a, b] = parts.as_slice() else {
        return Err(format!("expected 2 parts, got {}", parts.len()));
    };
    let Ok(x) = a.trim().parse::<i32>() else {
        return Err(format!("bad number: {a}"));
    };
    let Ok(y) = b.trim().parse::<i32>() else {
        return Err(format!("bad number: {b}"));
    };
    Ok(x + y)
}

println!("{:?}", sum_two("1, 2"));
println!("{:?}", sum_two("1"));
println!("{:?}", sum_two("1, x"));
```

Output:

```
Ok(3)
Err("expected 2 parts, got 1")
Err("bad number:  x")
```

Three guards, zero nesting, and every binding available to the line below it.
Note the slice pattern works here too — `let ... else` takes any refutable
pattern.

Returning `None` is the same shape: `let Ok(n) = s.parse::<u32>() else { return
None; };`.

(For the `Option`/`Result` version specifically, the `?` operator is shorter
still — lesson 8.3. `let ... else` is for when you want a *different* return
value, a log line, or a `continue`.)

## The else block must diverge

```rust
let o: Option<i32> = None;
let Some(n) = o else {
    println!("missing");
};
println!("{n}");
```

Does not compile — intentional:

```
error[E0308]: `else` clause of `let...else` does not diverge
 --> src/main.rs:3:26
  |
3 |       let Some(n) = o else {
  |  __________________________^
4 | |         println!("missing");
5 | |     };
  | |_____^ expected `!`, found `()`
  |
  = note:   expected type `!`
          found unit type `()`
  = help: try adding a diverging expression, such as `return` or `panic!(..)`
  = help: ...or use `match` instead of `let...else`
```

This is enforced because `n` must be bound after the statement; if the `else`
could fall through, it wouldn't be.

## let ... else with continue

Inside a loop, `continue` is the natural divergence — a filter that also binds.

```rust
let inputs = ["5", "oops"];
for s in inputs {
    let Ok(n) = s.parse::<i32>() else {
        println!("skipping {s}");
        continue;
    };
    println!("doubled {}", n * 2);
}
```

Output:

```
doubled 10
skipping oops
```

## while let

Loops as long as the pattern matches. The canonical use is draining a stack.

```rust
let mut stack = vec![1, 2, 3];
while let Some(top) = stack.pop() {
    println!("popped {top}");
}
println!("stack now {stack:?}");
```

Output:

```
popped 3
popped 2
popped 1
stack now []
```

`pop()` returns `Option<T>`, so `None` ends the loop. Compare to lesson 3.2's
`while !v.is_empty() { v.pop() }`, which needs two checks and an `unwrap`.

A `VecDeque`'s `pop_front()` drains the same way.

## while let on an iterator

`while let Some(x) = it.next()` is what `for` desugars to. Write it by hand when
you need to advance the iterator inside the body.

```rust
let mut chars = "abc".chars();
while let Some(c) = chars.next() {
    print!("[{c}]");
}
println!();
```

Output:

```
[a][b][c]
```

For plain iteration `for n in 1..=3` is better — Clippy's
`while_let_on_iterator` will say so. The hand-written form earns its place with
`peekable`:

```rust
let mut parts = "a,b,c".split(',').peekable();
while let Some(p) = parts.next() {
    if parts.peek().is_some() { print!("{p}-"); } else { println!("{p}"); }
}
```

Output:

```
a-b-c
```

## while let with a break and a reference

Mixing a pattern, a peek, and a conditional exit:

```rust
let mut nums = vec![5, 3, 9, 1];
nums.sort();
while let Some(&smallest) = nums.first() {
    if smallest > 4 { break; }
    nums.remove(0);
    println!("dropped {smallest}");
}
println!("{nums:?}");
```

Output:

```
dropped 1
dropped 3
[5, 9]
```

The `&smallest` pattern destructures through the `&i32` that `first()` returns,
so `smallest` is a plain `i32` and the borrow ends before `remove`.

The condition is re-evaluated each pass, so reassigning the scrutinee in the
body works too.

`while let` takes let-chains too:

```rust
let mut stack = vec![Some(1), None, Some(3)];
while let Some(item) = stack.pop() && let Some(n) = item {
    println!("n {n}");
}
println!("left {stack:?}");
```

Output:

```
n 3
left [Some(1)]
```

The `None` element failed the second link and stopped the loop, leaving
`Some(1)` behind — chains stop at the first failure, they don't skip.

## matches!

When you want a `bool`, not a branch. It takes a value and a pattern, guards
included.

```rust
let o = Some(5);
println!("{}", matches!(o, Some(_)));
println!("{}", matches!(o, None));
println!("{}", matches!(o, Some(n) if n > 3));
println!("{}", matches!(o, Some(1..=4)));
```

Output:

```
true
false
true
false
```

```rust
let c = 'x';
println!("{}", matches!(c, 'a'..='z'));
println!("{}", matches!(c, 'a' | 'e' | 'i' | 'o' | 'u'));

let v: Vec<i32> = vec![1, 2];
println!("{}", matches!(v.as_slice(), [_, _]));
```

Output:

```
true
false
true
```

Its real home is a closure passed to `filter` or `any`, where writing a `match`
would be noise:

```rust
#[derive(Debug)]
enum Status { Active, Suspended(String), Closed }

let statuses = vec![Status::Active, Status::Suspended("spam".into()), Status::Closed];
let n = statuses.iter().filter(|s| matches!(s, Status::Active)).count();
println!("{n} active");
for s in &statuses {
    println!("{s:?} suspended? {}", matches!(s, Status::Suspended(_)));
}
```

Output:

```
1 active
Active suspended? false
Suspended("spam") suspended? true
Closed suspended? false
```

For `Option` and `Result` specifically, `matches!(r, Err(_))` and `r.is_err()`
agree, and the method reads better.

Use `is_some()`/`is_none()`/`is_ok()`/`is_err()` there, and `matches!` for your
own enums and for patterns with payload tests.

## Choosing between them

| Situation | Use |
|---|---|
| One pattern matters, nothing to do otherwise | `if let` |
| One pattern matters, something small otherwise | `if let .. else` |
| Guard clause: bail out, then continue flat | `let .. else` |
| Guard clause returning the same `Option`/`Result` | `?` (lesson 8.3) |
| Two or more cases each with real work | `match` |
| Several independent optional values together | `if let .. && let ..` |
| Repeat until a pattern stops matching | `while let` |
| Need a `bool`, not a branch | `matches!` |
| `Option`/`Result` presence test | `.is_some()` / `.is_ok()` |

Rule of thumb: the moment you write `if let Some(x) = ... { <most of the
function> }`, invert it into `let Some(x) = ... else { return ... };`.

## Quick Reference

| Form | Syntax | Notes |
|---|---|---|
| if let | `if let Some(x) = o { }` | binding scoped to block |
| if let else | `if let P = e { } else { }` | arms share a type |
| else if let | `if let P = a { } else if let Q = b { }` | |
| Let-chain | `if let P = a && let Q = b { }` | edition 2024 |
| Chain with guard | `if let Some(n) = o && n > 3 { }` | |
| By reference | `if let Some(x) = &o { }` | `x: &T` |
| By mutable ref | `if let Some(x) = &mut o { *x += 1 }` | |
| let else | `let Some(x) = o else { return; };` | `else` must diverge |
| let else continue | `let Ok(n) = p else { continue; };` | in a loop |
| while let | `while let Some(x) = v.pop() { }` | ends on `None` |
| while let chain | `while let Some(a) = f() && let Some(b) = g(a) { }` | |
| matches! | `matches!(v, Some(n) if n > 3)` | yields `bool` |

## Common Mistakes

**`if let` wrapping the whole function body.** Invert it to `let ... else` and
flatten.

**An `else` block that doesn't diverge.** E0308 expecting `!`. Add `return`,
`continue`, `break`, or `panic!`.

**`if let Some(x) = opt` when you still need `opt`.** That moves non-`Copy`
payloads. Use `&opt`.

**Using the binding after the `if let` block.** It's out of scope (E0425). Use
`let ... else` if you need it later.

**`if opt.is_some() { opt.unwrap() }`.** Two operations and a panic risk where
`if let` does it once, safely.

**`while let Some(x) = it.next()` for plain iteration.** That's just `for x in
it`; Clippy warns. Keep `while let` for `pop`-style draining and `peekable`.

## Exercises

1. Rewrite `match cfg { Some(v) => println!("{v}"), None => {} }` as `if let`.
2. Write `fn initials(full: Option<&str>) -> String` using `let ... else` to
   return `"??"` early, then build the initials from the words.
3. Implement a bracket-balance checker: push openers onto a `Vec`, and use
   `while let Some(c) = stack.pop()` at the end to report any unclosed ones.
4. Write code that triggers "`else` clause of `let...else` does not diverge",
   then fix it two ways.
5. Given `let a: Option<i32>` and `let b: Option<i32>`, print their sum only if
   both are `Some` and the sum exceeds 10 — using one let-chain.
6. Given `Vec<Result<i32, String>>`, count the `Err` values with `matches!`,
   then do the same with `.iter().filter(|r| r.is_err())`.

---

[← Course Index](../README.md)
