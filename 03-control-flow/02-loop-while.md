[← 3.1 if, else if, and if as an Expression](01-if-expressions.md) · [Course Index](../README.md) · [3.3 for Loops and Ranges →](03-for-ranges.md)

# 3.2 — loop, while, and Loop Labels

## What You'll Learn

- `loop` is an infinite loop that can **return a value** via `break expr`
- `while` with any bool condition, including compound ones
- `'label:` on any loop, and `break`/`continue` targeting an outer loop
- When to pick `loop` over `while` over `for`

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `while (true) { }` | `loop { }` |
| JS `while (c) { }` | `while c { }` — no parens |
| JS `do { } while (c)` | no equivalent; `loop { .. if !c { break } }` |
| Go `outer: for { break outer }` | `'outer: loop { break 'outer }` |

Go has labeled break/continue too. What's new: `break` can carry a value out of
a `loop`, so a retry loop is an expression.

## loop and break

`loop` runs forever until something breaks out.

```rust
let mut i = 0;
loop {
    println!("tick {i}");
    i += 1;
    if i == 3 {
        break;
    }
}
```

Output:

```
tick 0
tick 1
tick 2
```

`while true` compiles but Clippy rejects it:

```
warning: denote infinite loops with `loop { ... }`
 --> src/main.rs:3:5
  |
3 |     while true {
  |     ^^^^^^^^^^ help: use `loop`
  |
  = note: `#[warn(while_true)]` on by default
```

## break with a value

`break expr` makes the whole `loop` evaluate to `expr`. This is the feature
that has no equivalent in JS/Python/Go.

```rust
let mut n = 0;
let x = loop {
    n += 1;
    if n * n > 50 {
        break n;
    }
};
println!("first n with n^2>50: {x}");
```

Output:

```
first n with n^2>50: 8
```

The degenerate case works and is legal:

```rust
let five = loop { break 5 };
println!("{five}");
```

Output:

```
5
```

A bare `break` is `break ()`, so a loop with only plain breaks has type `()`:

```rust
let w = loop {
    break;
};
println!("{w:?}");
```

Output:

```
()
```

A `loop` with **no** `break` at all has type `!` (never), so it coerces to
anything — and everything after it is dead code:

```rust
let x: i32 = loop {};
println!("{x}");
```

```
warning: unreachable statement
 --> src/main.rs:3:5
  |
2 |     let x: i32 = loop {};
  |                  ------- any code following this expression is unreachable
3 |     println!("{x}");
  |     ^^^^^^^^^^^^^^^ unreachable statement
```

## break with a value only works on loop

```rust
let mut n = 0;
let x = while n < 5 {
    n += 1;
    break 99;
};
```

Does not compile — intentional:

```
error[E0571]: `break` with value from a `while` loop
 --> src/main.rs:5:9
  |
3 |     let x = while n < 5 {
  |             ----------- you can't `break` with a value in a `while` loop
4 |         n += 1;
5 |         break 99;
  |         ^^^^^^^^ can only break with a value inside `loop` or breakable block
  |
help: use `break` on its own without a value inside this `while` loop
```

`while` and `for` always evaluate to `()`. Only `loop` and a labeled block can
carry a value out.

## continue

Skips to the next iteration.

```rust
let mut c = 0;
loop {
    c += 1;
    if c % 2 == 0 {
        continue;
    }
    if c > 7 {
        break;
    }
    println!("odd {c}");
}
```

Output:

```
odd 1
odd 3
odd 5
odd 7
```

`continue` never takes a value — `continue 5` is a syntax error.

## while

Condition is checked before each iteration. No parens, braces required.

```rust
let mut k = 3;
while k > 0 {
    println!("count {k}");
    k -= 1;
}
println!("liftoff");
```

Output:

```
count 3
count 2
count 1
liftoff
```

## while with compound conditions

Any `bool` expression works, including short-circuiting chains and method calls.

```rust
let mut a = 0;
let mut b = 20;
while a < b && b - a > 5 {
    a += 2;
    b -= 3;
}
println!("{a} {b}");
```

Output:

```
6 11
```

Method-call conditions are extremely common:

```rust
let mut v = vec![1, 2, 3];
while !v.is_empty() {
    println!("popped {:?}", v.pop());
}
```

Output:

```
popped Some(3)
popped Some(2)
popped Some(1)
```

That exact shape is better written `while let Some(x) = v.pop()` — lesson 3.6.

Reassigning the loop variable from an `if` expression inside the body is
idiomatic:

```rust
let mut n: u32 = 27;
let mut steps = 0;
while n != 1 {
    n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
    steps += 1;
}
println!("collatz steps: {steps}");
```

Output:

```
collatz steps: 111
```

## No do-while

There is no `do { } while (c)`. Write the loop with the test at the bottom:

```rust
let mut i = 0;
loop {
    i += 1;
    if i > 3 { break }
    print!("{i} ");
}
println!();
```

Output:

```
1 2 3 
```

Note `if i > 3 { break }` with no semicolon — `break` has type `!`, so the
block is fine either way.

## Labels

A label is `'name:` before `loop`, `while`, or `for`. It names the loop so
`break`/`continue` can target it explicitly. Labels use the lifetime syntax
(leading `'`) but are unrelated to lifetimes.

```rust
'outer: for i in 1..=3 {
    for j in 1..=3 {
        if i * j > 4 {
            println!("breaking outer at {i},{j}");
            break 'outer;
        }
        println!("{i}x{j}={}", i * j);
    }
}
```

Output:

```
1x1=1
1x2=2
1x3=3
2x1=2
2x2=4
breaking outer at 2,3
```

Without the label, `break` would only exit the inner `for`.

## continue with a label

```rust
'rows: for i in 0..4 {
    for j in 0..4 {
        if j == 2 {
            continue 'rows;
        }
        print!("{i}{j} ");
    }
}
println!();
```

Output:

```
00 01 10 11 20 21 30 31 
```

`continue 'rows` jumps to the next `i`, abandoning the rest of the `j` loop.

## break 'label with a value

Combine both: break out of several levels *and* produce a value.

```rust
let found = 'search: loop {
    for i in 0..100 {
        if i * i == 144 {
            break 'search i;
        }
    }
    break 0;
};
println!("sqrt(144)={found}");
```

Output:

```
sqrt(144)=12
```

The labeled target must be a `loop` (or labeled block) for the value form —
labeling a `while` doesn't change E0571.

```rust
let mut n = 0;
let r = 'w: while n < 10 {
    n += 1;
    if n == 4 {
        break 'w;
    }
};
println!("{n} {r:?}");
```

Output:

```
4 ()
```

## Multiple labels

Any number of nested labels; `break`/`continue` picks one by name.

```rust
let mut total = 0;
'a: loop {
    'b: loop {
        total += 1;
        if total % 3 == 0 {
            continue 'a;
        }
        if total > 7 {
            break 'a;
        }
        if total % 2 == 0 {
            continue 'b;
        }
        println!("t={total}");
    }
}
println!("final {total}");
```

Output:

```
t=1
t=5
t=7
final 8
```

## Labeled blocks

A label on a bare block gives you a one-shot forward jump — no looping.

```rust
let total = 8;
let outer = 'blk: {
    if total > 5 {
        break 'blk "big";
    }
    "small"
};
println!("{outer}");
```

Output:

```
big
```

Useful for early exit out of a chunk of computation without a helper function.

## break does not cross a closure

```rust
loop {
    let f = || break;
    f();
}
```

Does not compile — intentional:

```
error[E0267]: `break` inside of a closure
 --> src/main.rs:3:20
  |
3 |         let f = || break;
  |                 -- ^^^^^ cannot `break` inside of a closure
  |                 |
  |                 enclosing closure
```

The same applies to `continue` and `return`-into-a-loop. A closure is a
function boundary.

## Infinite loop plus match: the state machine

`loop { state = match state { .. } }` is the canonical Rust shape for a state
machine. Enums come in Module 05; here is the pattern.

```rust
enum State {
    Start,
    Middle(u32),
    Done,
}

let mut state = State::Start;
let mut log = Vec::new();
loop {
    state = match state {
        State::Start => {
            log.push("start");
            State::Middle(0)
        }
        State::Middle(n) if n < 3 => {
            log.push("step");
            State::Middle(n + 1)
        }
        State::Middle(_) => {
            log.push("exit middle");
            State::Done
        }
        State::Done => break,
    };
}
println!("{log:?}");
```

Output:

```
["start", "step", "step", "step", "exit middle"]
```

`State::Done => break` works as a match arm because `break` has type `!`, which
unifies with the `State` the other arms produce.

## Retry loop returning a Result

The classic use of `break` with a value.

```rust
fn try_connect(attempt: u32) -> Result<&'static str, &'static str> {
    if attempt >= 3 { Ok("connected") } else { Err("timeout") }
}

let mut retries = 0;
let result = loop {
    retries += 1;
    match try_connect(retries) {
        Ok(v) => break Ok(v),
        Err(_) if retries >= 3 => break Err("gave up"),
        Err(e) => println!("attempt {retries} failed: {e}"),
    }
};
println!("{result:?}");
```

Output:

```
attempt 1 failed: timeout
attempt 2 failed: timeout
Ok("connected")
```

## loop vs while vs for

| Situation | Use |
|---|---|
| Iterating a collection or range | `for` (lesson 3.3) — always first choice |
| Condition known before the body runs | `while` |
| Exit condition only discoverable mid-body | `loop` + `break` |
| The loop should produce a value | `loop` + `break expr` |
| Draining via a method returning `Option` | `while let` (lesson 3.6) |

Rule of thumb: if you are writing `let mut i = 0; while i < n { .. i += 1 }`,
you wanted `for i in 0..n`. Manual counters are where off-by-one bugs live.

## Quick Reference

| Form | Syntax | Value |
|---|---|---|
| Infinite | `loop { .. }` | `!` if no break |
| Exit | `break;` | `()` |
| Exit with value | `break expr;` | `expr`'s type |
| Skip iteration | `continue;` | — |
| Conditional | `while c { .. }` | always `()` |
| Labeled loop | `'l: loop { .. }` | |
| Labeled while/for | `'l: while c { }`, `'l: for x in it { }` | `()` |
| Break outer | `break 'l;` | |
| Break outer with value | `break 'l expr;` | loop-only |
| Continue outer | `continue 'l;` | |
| Labeled block | `'l: { .. break 'l v; .. }` | `v`'s type |
| do-while | none — `loop { .. if !c { break } }` | |

## Common Mistakes

**`break value` inside `while` or `for`.** E0571. Restructure as `loop`, or set
a variable before breaking.

**Forgetting to mutate the condition variable.** `while i < 3 { }` with no
`i += 1` hangs with no warning. Prefer `for` over manual counters.

**Unlabeled `break` in a nested loop.** It exits only the innermost loop. Add
`'outer:` and `break 'outer`.

**`break` or `continue` inside a closure.** E0267 — closures are function
boundaries. Return a value from the closure and branch outside it.

**Using `loop` where `while let` fits.** `loop { match it.next() { Some(x) =>
.., None => break } }` is the long way round. See lesson 3.6.

## Exercises

1. Find the smallest `n` where the sum `1 + 2 + .. + n` exceeds 1000, using
   `loop` and `break n`.
2. Rewrite `let mut i = 0; while i < 5 { println!("{i}"); i += 1; }` as a `for`
   loop, then explain in one line why the `for` version can't be off by one.
3. Write nested loops over a 5x5 grid that stop entirely (both loops) at the
   first cell where `i * j` is divisible by 6, printing that cell. Then change
   it so only the inner loop restarts instead.
4. Build a guessing-game skeleton: `loop` forever, `break` out when a counter
   reaches a target, and make the loop evaluate to the number of attempts.
5. Write a labeled block `'calc: { .. }` that returns `"error"` early if a
   divisor is zero, and the quotient as a string otherwise.

---

[← 3.1 if, else if, and if as an Expression](01-if-expressions.md) · [Course Index](../README.md) · [3.3 for Loops and Ranges →](03-for-ranges.md)
