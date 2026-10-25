[← 15.8 Blocking Code in Async Contexts](../15-async-tokio/08-blocking.md) · [Course Index](../README.md) · [16.2 Procedural Macros Overview →](02-proc-macros.md)

# 16.1 — Declarative Macros with macro_rules!

## What You'll Learn

- `macro_rules!` syntax: rules, matchers, and all 13 fragment specifiers
- Repetition with `$(...),*`, `$(...);*`, `$(...)+`, and nesting
- Recursion, hygiene, `#[macro_export]`, and `$crate`
- Debugging expansions and where declarative macros run out of road

## Coming From Another Language

| Other | Rust |
|---|---|
| C `#define SQ(x) ((x)*(x))` | `macro_rules! sq { ($x:expr) => { $x * $x } }` |
| JS template helper / codegen script | macro expanded by the compiler |
| Go `go:generate` | macro, no separate build step |
| Python decorator | attribute **proc** macro (16.2), not `macro_rules!` |

C macros substitute text. Rust macros match and emit **token trees**: the input
is already parsed into balanced groups, the arguments are typed fragments, and
the output must be syntactically valid. There is no `((x))` defensive
parenthesizing because `$x:expr` captures one expression as a unit.

## The smallest macro

```rust
macro_rules! say_hi {
    () => { println!("hi") };
}

fn main() {
    say_hi!();
}
```

Output:

```
hi
```

A macro is a list of `matcher => { expansion }` rules. Invoke it with `!` and
`()`, `[]`, or `{}` — the brackets are interchangeable, so `vec![1]` and
`vec!(1)` are the same call.

## Fragments capture, they don't paste

```rust
macro_rules! square {
    ($x:expr) => { $x * $x };
}

fn main() {
    println!("{}", square!(4));
    println!("{}", square!(2 + 3));
}
```

Output:

```
16
25
```

`square!(2 + 3)` is 25, not 11. `$x:expr` holds the whole expression as one
node, so no extra parens are ever needed — the classic C macro bug does not
exist here.

## Fragment specifiers

| Specifier | Matches | Example argument |
|---|---|---|
| `expr` | an expression | `2 + 3`, `f(x)` |
| `ident` | identifier or keyword | `my_fn`, `Vec` |
| `ty` | a type | `Vec<u8>`, `&'a str` |
| `tt` | one token tree | `+`, `foo`, `(a, b)` |
| `pat` | a pattern | `Some(x) \| None` |
| `literal` | a literal | `3.5`, `"s"`, `-1` |
| `block` | a braced block | `{ let x = 1; x }` |
| `stmt` | one statement (no trailing `;`) | `let x = 1` |
| `path` | a path | `std::process::id` |
| `item` | an item | `fn f() {}`, `struct S;` |
| `vis` | a visibility, possibly empty | `pub`, `pub(crate)` |
| `lifetime` | a lifetime | `'a` |
| `meta` | an attribute's contents | `derive(Debug)` |

All 13 in one program:

```rust
macro_rules! make_fn {
    ($name:ident) => {
        fn $name() { println!("called {}", stringify!($name)); }
    };
}
make_fn!(alpha);

macro_rules! default_of { ($t:ty) => { <$t as Default>::default() }; }
macro_rules! show_tt   { ($($t:tt)*) => { stringify!($($t)*) }; }
macro_rules! classify  { ($p:pat) => { stringify!($p) }; }
macro_rules! lit_name  { ($l:literal) => { concat!("literal: ", stringify!($l)) }; }
macro_rules! run_block { ($b:block) => { $b }; }
macro_rules! twice_stmt{ ($s:stmt) => { $s $s }; }
macro_rules! call_path { ($p:path) => { $p() }; }
macro_rules! show_meta { ($m:meta) => { stringify!($m) }; }

macro_rules! emit_item { ($i:item) => { $i }; }
emit_item! { const EMITTED: u32 = 99; }

macro_rules! field_struct {
    ($v:vis $name:ident) => { pub struct Holder { $v $name: u8 } };
}
field_struct!(pub count);

macro_rules! with_lt {
    ($l:lifetime) => { fn borrow<$l>(s: &$l str) -> &$l str { s } };
}
with_lt!('a);

fn main() {
    alpha();
    println!("{:?} {:?}", default_of!(i32), default_of!(Vec<u8>));
    println!("{}", show_tt!(a + b * (c)));
    println!("{}", classify!(Some(x) | None));
    println!("{}", lit_name!(3.5));
    println!("{}", run_block!({ let x = 10; x + 1 }));
    twice_stmt!(println!("stmt"));
    println!("{}", call_path!(std::process::id) > 0);
    println!("{}", EMITTED);
    println!("{}", Holder { count: 5 }.count);
    println!("{}", borrow("lifetimed"));
    println!("{}", show_meta!(derive(Debug)));
}
```

Output:

```
called alpha
0 []
a + b * (c)
Some(x) | None
literal: 3.5
11
stmt
stmt
true
99
5
lifetimed
derive(Debug)
```

`stringify!` turns tokens back into a string and is the fastest way to see what
a fragment captured.

### What can follow a fragment

The parser needs to know where a fragment ends, so only certain tokens may
follow certain specifiers. `expr` and `stmt` may be followed only by `=>`, `,`,
or `;`. `ty` and `path` by `=>`, `,`, `=`, `|`, `;`, `:`, `>`, `>>`, `[`, `{`,
`as`, `where`, or a `block` fragment. `tt`, `ident`, `lifetime`, and `literal`
may be followed by anything. This is why separator-based repetition almost
always uses `,` or `;`.

## Repetition

`$( ... ) sep rep` where `sep` is an optional separator token and `rep` is
`*` (zero or more), `+` (one or more), or `?` (zero or one, no separator
allowed).

```rust
macro_rules! my_vec {
    ($($x:expr),* $(,)?) => { vec![$($x),*] };
}

macro_rules! sum_all {
    ($($x:expr),+) => { 0 $(+ $x)+ };
}

macro_rules! semi_list {
    ($($x:expr);*) => { vec![$($x),*] };
}

fn main() {
    println!("{:?}", my_vec![1, 2, 3]);
    println!("{:?}", my_vec![1, 2, 3,]);
    println!("{:?}", my_vec![] as Vec<i32>);
    println!("{}", sum_all!(1, 2, 3, 4));
    println!("{:?}", semi_list![1; 2; 3]);
}
```

Output:

```
[1, 2, 3]
[1, 2, 3]
[]
10
[1, 2, 3]
```

Note `sum_all` expands to `0 + 1 + 2 + 3 + 4` — the repetition in the expansion
need not use the same shape as the matcher.

### Trailing comma

`$(,)?` at the end of the matcher is the idiomatic way to accept an optional
trailing comma. Without it:

```rust
macro_rules! my_vec {
    ($($x:expr),*) => { vec![$($x),*] };
}

fn main() {
    println!("{:?}", my_vec![1, 2, 3,]);
}
```

Does not compile:

```
error: unexpected end of macro invocation
  --> src/main.rs:6:38
   |
 1 | macro_rules! my_vec {
   | ------------------- when calling this macro
...
 6 |     println!("{:?}", my_vec![1, 2, 3,]);
   |                                      ^ missing tokens in macro arguments
   |
note: while trying to match meta-variable `$x:expr`
  --> src/main.rs:2:8
   |
 2 |     ($($x:expr),*) => { vec![$($x),*] };
   |        ^^^^^^^
```

Always add `$(,)?`. Users will type the trailing comma.

### Nested repetition

Inner repetitions bind per outer iteration.

```rust
macro_rules! grid {
    ($([$($c:expr),*]),*) => { vec![$(vec![$($c),*]),*] };
}

fn main() {
    println!("{:?}", grid![[1, 2], [3, 4, 5], []]);
}
```

Output:

```
[[1, 2], [3, 4, 5], []]
```

## Multiple rules and matching order

Rules are tried **top to bottom**; the first whose matcher fits wins. Put the
specific rules first.

```rust
macro_rules! count {
    () => { 0 };
    ($head:expr) => { 1 };
    ($head:expr, $($tail:expr),+) => { 1 + count!($($tail),+) };
}

fn main() {
    println!("{} {} {}", count!(), count!(9), count!(9, 8, 7, 6));
}
```

Output:

```
0 1 4
```

If the general `$($x:expr),*` rule came first it would swallow every call and
the later rules would be dead.

## Recursion

A macro may invoke itself. Peel one argument per step and recurse on the rest.

```rust
macro_rules! my_min {
    ($x:expr) => { $x };
    ($x:expr, $($rest:expr),+) => {
        { let a = $x; let b = my_min!($($rest),+); if a < b { a } else { b } }
    };
}

fn main() {
    println!("{}", my_min!(5, 3, 9, 1, 7));
}
```

Output:

```
1
```

The default recursion limit is 128 expansion steps; raise it with
`#![recursion_limit = "256"]` at the crate root if you build something deep.

## A real macro: hashmap!

`std` has `vec!` but no map literal. This is the canonical hand-rolled one.

```rust
use std::collections::HashMap;

macro_rules! hashmap {
    () => { HashMap::new() };
    ($($k:expr => $v:expr),+ $(,)?) => {{
        let mut m = HashMap::new();
        $( m.insert($k, $v); )+
        m
    }};
}

fn main() {
    let m: HashMap<&str, i32> = hashmap! { "a" => 1, "b" => 2, "c" => 3 };
    let mut pairs: Vec<_> = m.iter().collect();
    pairs.sort();
    println!("{:?}", pairs);

    let empty: HashMap<&str, i32> = hashmap!();
    println!("{:?}", empty);
}
```

Output:

```
[("a", 1), ("b", 2), ("c", 3)]
{}
```

Two details: `=>` is just a literal token in the matcher, and the double brace
`{{ ... }}` makes the expansion a block expression that evaluates to the map.

## Hygiene

Identifiers the macro *creates* live in the macro's own syntax context. They
cannot capture or collide with the caller's names.

```rust
macro_rules! make_x {
    () => { let x = 999; };
}

macro_rules! set_to_42 {
    ($v:ident) => { $v = 42; };
}

fn main() {
    let x = 1;
    make_x!();
    println!("x = {}", x);

    let mut y = 0;
    set_to_42!(y);
    println!("y = {}", y);
}
```

Output:

```
x = 1
y = 42
```

The `x` inside `make_x!` is a *different* `x`; the caller's stays 1. But `y`
was passed in as an `ident`, so it carries the caller's context and the
assignment lands. That is the rule: hygiene protects what the macro invents,
not what it is handed.

**The surprise:** hygiene applies to local variables, labels, and `$crate` — but
**not** to types, functions, modules, or constants. A macro that expands to
`Config::new()` resolves `Config` at the call site, so it breaks if the caller
has a different `Config` in scope, or none at all. Always write full paths in
expansions: `::std::collections::HashMap::new()` instead of `HashMap::new()`.

## Exporting: #[macro_export] and $crate

`macro_rules!` is textually scoped: usable only *after* its definition in the
file, and in child modules. `#[macro_export]` lifts it to the crate root so
other crates can `use` it. Inside the expansion, `$crate` expands to a path to
the **defining** crate.

In `src/lib.rs` of crate `w16`:

```rust
pub fn helper(n: i32) -> i32 { n * 10 }

#[macro_export]
macro_rules! scaled {
    ($n:expr) => { $crate::helper($n) };
}
```

In `src/main.rs`:

```rust
use w16::scaled;

fn helper(_n: i32) -> i32 { -1 }

fn main() {
    println!("{}", scaled!(7));
    println!("{}", w16::scaled!(3));
    println!("{}", helper(7));
}
```

Output:

```
70
30
-1
```

The consumer has its own `helper`, but `$crate::helper` points at the library's.
Without `$crate` this macro would silently call the wrong function. Exported
macros can be called by `use`-ing them or by full path.

To make a macro usable *before* its definition point in the same crate, put
`#[macro_use]` on the module declaration:

```rust
#[macro_use]
mod macros;
```

## Debugging expansions

**`cargo expand`** prints the whole crate after macro expansion. Install once:

```
cargo install cargo-expand
cargo expand
cargo expand --bin w16 main
```

It is the single most useful macro tool — read the expansion instead of
guessing. It shells out to a nightly-only rustc flag internally but installs
and runs on stable.

**`trace_macros!`** logs each expansion step at compile time. It is a nightly
feature, so on stable 1.99.1 you will see:

```
error[E0658]: use of unstable library feature `trace_macros`
```

On nightly, `#![feature(trace_macros)]` plus `trace_macros!(true);` before the
call prints every rule match. On stable, use `cargo expand`.

**`stringify!`** and `log_syntax!` are the lightweight tricks: wrap a fragment
in `stringify!` and print it to confirm what was captured.

## Limitations

`macro_rules!` matches and emits tokens. It cannot:

- **Compute.** No arithmetic, no string manipulation, no `if` on the input. You
  cannot turn `$n:literal` into `$n + 1` tokens, or lowercase an ident.
- **Build identifiers.** There is no `concat_ident!` on stable, so you cannot
  generate `get_foo` from `foo`. This is the most common reason to reach for a
  proc macro.
- **Inspect types.** A `$t:ty` is opaque tokens; the macro cannot know its
  fields or whether it implements a trait.
- **Report good errors.** Error spans land inside the macro body:

```rust
macro_rules! square {
    ($x:expr) => { $x * $x };
}

fn main() {
    println!("{}", square!("nope"));
}
```

Does not compile:

```
error[E0369]: cannot multiply `&str` by `&str`
 --> src/main.rs:2:23
  |
2 |     ($x:expr) => { $x * $x };
  |                       ^
...
6 |     println!("{}", square!("nope"));
  |                    --------------- in this macro invocation
```

The caret points at the macro definition, not at `"nope"`. For deliberate
errors, `compile_error!("message")` at least lets you choose the text.

When you hit these walls, write a procedural macro (16.2).

## Quick Reference

| Form | Meaning |
|---|---|
| `macro_rules! m { () => {} }` | define |
| `m!()`, `m![]`, `m!{}` | invoke (brackets interchangeable) |
| `$x:expr` | capture one expression |
| `$($x:expr),*` | zero or more, comma-separated |
| `$($x:expr),+` | one or more |
| `$($x:expr),* $(,)?` | ...with optional trailing comma |
| `$($x:expr);*` | semicolon-separated |
| `$(+ $x)+` | repetition in the expansion |
| `$( $($i:expr),* );*` | nested repetition |
| `stringify!($x)` | fragment tokens as a `&'static str` |
| `concat!(a, b)` | join literals at compile time |
| `compile_error!("msg")` | fail with a chosen message |
| `#[macro_export]` | export at crate root |
| `$crate` | path to the defining crate |
| `#[macro_use] mod m;` | use macros before their definition point |
| `#![recursion_limit = "256"]` | raise the 128-step expansion limit |
| `cargo expand` | print the expanded crate |

## Common Mistakes

**Forgetting `$(,)?`.** Callers type trailing commas; the macro fails with
"unexpected end of macro invocation". Add it to every comma-separated matcher.

**Putting the general rule first.** Rules match top to bottom, so a broad
`$($x:expr),*` makes every later rule unreachable. Specific rules first.

**Writing bare type names in the expansion.** `HashMap::new()` resolves at the
call site and breaks for callers who have not imported it. Use
`::std::collections::HashMap::new()`.

**Exporting without `$crate`.** A macro calling `helper()` will call the
*caller's* `helper`, or fail to compile. Always `$crate::helper()`.

**Expecting the macro to compute.** `macro_rules!` cannot add numbers, build
identifiers, or look at a type's fields. That is a proc macro's job.

## Exercises

1. Write `max!` taking one or more expressions, recursively, mirroring `my_min!`.
   Verify `max!(3, 11, 7)` is 11.
2. Write `count_args!` that returns the number of arguments *without* using
   `1 + recurse` — use `<[()]>::len(&[$(replace_with_unit!($x)),*])` where the
   helper expands each argument to `()`.
3. Write `btreemap!` mirroring `hashmap!`, with the `$(,)?` trailing comma and an
   empty-input rule. Use full paths so it works unimported.
4. Write a `test_table!` macro that takes `name: input => expected` rows and
   generates one `#[test] fn` per row. You will hit the "cannot build
   identifiers" wall — take the test name as an `ident` argument instead.
5. Write `debug_vars!(a, b, c)` printing `a = 1, b = 2, c = 3` using
   `stringify!` and repetition. Then run `cargo expand` and read the expansion.

---

[← 15.8 Blocking Code in Async Contexts](../15-async-tokio/08-blocking.md) · [Course Index](../README.md) · [16.2 Procedural Macros Overview →](02-proc-macros.md)
