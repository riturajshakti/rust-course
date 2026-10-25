[← Course Index](../README.md)

# 16.2 — Procedural Macros Overview

## What You'll Learn

- The three kinds: derive, attribute, function-like
- Why a proc macro needs its own crate with `proc-macro = true`
- `TokenStream` in, `TokenStream` out, and the `syn` + `quote` + `proc-macro2` trio
- Error reporting that points at the user's code, and when not to write one

## Coming From Another Language

| Other | Rust |
|---|---|
| Babel / TS transformer plugin | proc macro crate |
| Python decorator `@timed` | `#[timed]` attribute macro |
| Java annotation processor | `#[derive(Foo)]` derive macro |
| Go `go:generate` + a codegen binary | proc macro, run by the compiler |

Closest to a Java annotation processor or a Babel plugin: your code is a
compiler plugin that receives tokens and returns tokens. Unlike `macro_rules!`,
it is arbitrary Rust — it can compute, build identifiers, read a struct's
fields, and emit whatever it likes.

## The three kinds

| Kind | Attribute | Invoked as | Input | Can it modify the item? |
|---|---|---|---|---|
| Derive | `#[proc_macro_derive(Name)]` | `#[derive(Name)]` on a struct/enum | the item | No — it only *adds* |
| Attribute | `#[proc_macro_attribute]` | `#[name]` or `#[name(args)]` | attr args + the item | Yes — it replaces it |
| Function-like | `#[proc_macro]` | `name!(...)` | the tokens inside | n/a |

A derive macro's output is appended next to the item, so the original struct
always survives. An attribute macro's output *replaces* the item entirely — if
you forget to re-emit the function, it vanishes.

## Why a separate crate

Proc macros are compiled and **run** by the compiler while it compiles your
crate. That means they must be built first, for the host machine, as a compiler
plugin — which cannot happen inside the same crate that uses them. So a proc
macro lives in its own crate marked:

```toml
[package]
name = "describe_derive"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro = true

[dependencies]
proc-macro2 = "1.0.107"
quote = "1.0.47"
syn = { version = "3.0.9", features = ["full"] }
```

A `proc-macro = true` crate may export **only** macros — no helper types, no
traits. This is why the ecosystem ships pairs: `serde` + `serde_derive`,
`thiserror` + `thiserror-impl`. The trait lives in the normal crate, the derive
in the proc-macro crate, and the normal crate re-exports the derive behind a
feature so users see one dependency.

A two-crate workspace is the practical layout:

```toml
[workspace]
resolver = "3"
members = ["describe_derive", "consumer"]
```

```toml
[package]
name = "consumer"
version = "0.1.0"
edition = "2024"

[dependencies]
describe_derive = { path = "../describe_derive" }
```

## TokenStream in, TokenStream out

Every proc macro has one of three signatures:

```rust
#[proc_macro]
pub fn name(input: TokenStream) -> TokenStream { input }

#[proc_macro_attribute]
pub fn name(attr: TokenStream, item: TokenStream) -> TokenStream { item }

#[proc_macro_derive(Name)]
pub fn name(input: TokenStream) -> TokenStream { TokenStream::new() }
```

`proc_macro::TokenStream` is an iterable sequence of token trees provided by the
compiler. It carries **spans** — the source locations that make error messages
point at the right place. Returning garbage tokens is a compile error in the
user's crate, not yours.

## The trio

| Crate | Job |
|---|---|
| `proc-macro2` | A `TokenStream` that works outside the compiler, so macro logic is unit-testable |
| `syn` | Parses a `TokenStream` into a typed AST (`DeriveInput`, `ItemFn`, `Expr`, ...) |
| `quote` | `quote! { ... }` builds a `TokenStream` from a template, interpolating with `#var` |

`proc_macro::TokenStream` only exists while the compiler is running, so `syn`
and `quote` both work in terms of `proc_macro2::TokenStream`. The `.into()` at
the end of every macro converts back. `syn`'s `full` feature is needed to parse
function bodies and statements; without it you only get items and types.

## A function-like macro

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, LitStr};

#[proc_macro]
pub fn shout(input: TokenStream) -> TokenStream {
    let lit = parse_macro_input!(input as LitStr);
    let upper = lit.value().to_uppercase();
    quote! { #upper }.into()
}
```

```rust
use describe_derive::shout;

fn main() {
    println!("{}", shout!("hello macros"));
}
```

Output:

```
HELLO MACROS
```

`parse_macro_input!` parses or returns a compile error automatically.
`to_uppercase()` at compile time is exactly what `macro_rules!` cannot do.

## An attribute macro

`#[timed]` wraps a function body in timing code.

```rust
use syn::ItemFn;

#[proc_macro_attribute]
pub fn timed(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let func = parse_macro_input!(item as ItemFn);
    let vis = &func.vis;
    let sig = &func.sig;
    let body = &func.block;
    let name = sig.ident.to_string();
    quote! {
        #vis #sig {
            let __start = std::time::Instant::now();
            let __out = (|| #body)();
            println!("[timed] {} took {:?}", #name, __start.elapsed());
            __out
        }
    }
    .into()
}
```

```rust
use describe_derive::timed;

#[timed]
fn sum_to(n: u64) -> u64 {
    (1..=n).sum()
}

fn main() {
    println!("sum = {}", sum_to(1_000));
}
```

Output:

```
[timed] sum_to took 7.042µs
sum = 500500
```

Three things to copy from this:

- Destructure the `ItemFn` and re-emit `#vis #sig` verbatim. Reconstructing a
  signature by hand loses generics, `async`, and lifetimes.
- The body is wrapped in `(|| #body)()` so that `return` inside the user's body
  returns from the closure, not past the timing `println!`.
- `__start` and `__out` are **not** hygienic. Unlike `macro_rules!`, proc-macro
  identifiers made with `quote!` share the call site's context, so they *can*
  collide with the user's locals. Prefix generated names with `__`.

## Error reporting

The difference between a tolerable macro and a good one is where the error
points. Use `syn::Error::new_spanned(tokens, msg).to_compile_error()` — the span
of `tokens` becomes the span of the error.

```rust
match &input.data {
    Data::Struct(s) => s,
    _ => {
        return syn::Error::new_spanned(&input.ident, "Describe only supports structs")
            .to_compile_error()
            .into()
    }
}
```

For a bad helper attribute, span the offending value:

```rust
return Err(syn::Error::new_spanned(
    &nv.value,
    "expected a string literal: #[rename = \"...\"]",
));
```

Both paths produce errors in the *consumer's* file:

```rust
#[derive(Describe)]
enum Color { Red, Green }

#[derive(Describe)]
struct Bad {
    #[rename = 42]
    id: u64,
}
```

Does not compile (intentional):

```
error: Describe only supports structs
 --> consumer/src/main.rs:8:6
  |
8 | enum Color { Red, Green }
  |      ^^^^^

error: expected a string literal: #[rename = "..."]
  --> consumer/src/main.rs:12:16
   |
12 |     #[rename = 42]
   |                ^^
```

That is what a good macro error looks like: the caret under the user's token,
not inside your macro.

Three rules:

- **Return the error, don't panic.** A `panic!` in a proc macro produces
  `proc macro panicked` with a useless span. `to_compile_error()` keeps the span.
- **Emit as many errors as you can.** The struct example above reports both
  problems in one build because each field produces its own
  `to_compile_error()` tokens rather than bailing on the first.
- **`compile_error!("msg")`** is the blunt version, usable from `quote!` and from
  `macro_rules!` too. It fails with your text but spans the macro call, not a
  specific token. Prefer `syn::Error` when you have tokens to point at.

A useful trick for partial failures: emit the error tokens *and* a stub impl, so
the user gets your message instead of a cascade of "trait not implemented"
errors from code that depended on the derive.

## Compile-time cost

Proc macros are not free:

- `syn` with `full` is a large dependency; it and `quote`/`proc-macro2` must
  compile before your crate's build can start. On this machine that first build
  took around 1.7s for the whole chain.
- The macro **runs** once per invocation, every compile, with no caching between
  items.
- Expansion blocks parallelism: dependents cannot start until the macro crate
  is built.
- `cargo check` pays the cost too, so editor feedback slows down.

Practical effects: a `#[derive]` applied to 500 structs is noticeable; `serde`'s
derive is the single biggest compile-time line item in many projects. Keep the
generated code small, avoid `syn`'s `full` feature if you only parse items, and
do not reach for a proc macro where a function or a generic would do.

## When a proc macro is the wrong tool

- **A plain function works.** If the logic does not need to see types or
  generate items, write a function. Macros defeat type inference, IDE
  navigation, and `?` propagation.
- **A generic or a blanket impl works.** `impl<T: Display> MyTrait for T` beats a
  derive that writes the same impl N times.
- **`macro_rules!` works.** If you only need repetition and substitution, a
  declarative macro costs no dependencies and no compile time (16.1).
- **A build script works.** Generating code from an external schema (OpenAPI, SQL,
  protobuf) belongs in `build.rs` writing a `.rs` file, where you can cache and
  inspect the output.
- **You want runtime reflection.** A proc macro sees only the tokens of the one
  item it is applied to. It cannot see other modules, resolve a type, know
  whether `T: Clone`, or read the whole crate. Design around that, or it will
  surprise you.

Use a proc macro when you must derive behavior from a type's *structure* —
field names, types, variants — which nothing else in Rust can do.

## Quick Reference

| Item | Purpose |
|---|---|
| `[lib] proc-macro = true` | marks the crate; may export only macros |
| `#[proc_macro]` | function-like, `name!(...)` |
| `#[proc_macro_attribute]` | `#[name]`, replaces the item |
| `#[proc_macro_derive(N)]` | `#[derive(N)]`, appends to the item |
| `#[proc_macro_derive(N, attributes(a))]` | registers helper attribute `#[a]` |
| `parse_macro_input!(x as T)` | parse or emit a compile error |
| `syn::parse2::<T>(ts)` | parse a `proc_macro2::TokenStream`, returns `Result` |
| `quote! { ... }` | build tokens; `#var` interpolates, `#(#v),*` repeats |
| `quote_spanned! { span => ... }` | build tokens with a chosen span |
| `syn::Error::new_spanned(t, m)` | error pointing at `t` |
| `.to_compile_error()` | `syn::Error` into emittable tokens |
| `compile_error!("msg")` | unconditional failure, span at the call |
| `.into()` | `proc_macro2::TokenStream` -> `proc_macro::TokenStream` |
| `cargo expand` | see what the macro produced |

## Common Mistakes

**Putting the macro in the same crate as its users.** A `proc-macro = true`
crate cannot be used by itself, and cannot export anything but macros. Split it
and re-export.

**Panicking instead of returning a `syn::Error`.** `panic!` gives
"proc macro panicked" with no useful location; `to_compile_error()` points at
the user's token.

**Forgetting to re-emit the item in an attribute macro.** The return value
*replaces* the input, so dropping `#sig #body` deletes the user's function.

**Assuming proc-macro identifiers are hygienic.** They are not. A generated
`let start = ...` can shadow the caller's `start`. Prefix with `__`.

**Reconstructing a function signature by hand.** Emit `#vis #sig` from the
parsed `ItemFn` so generics, lifetimes, and `async` survive.

## Exercises

1. Set up the two-crate workspace from this lesson and get `shout!` compiling
   and printing.
2. Add a `#[proc_macro]` called `const_len!("hello")` that expands to the literal
   `5usize` computed at compile time. Confirm with `cargo expand`.
3. Extend `#[timed]` to accept a label: `#[timed("hot path")]`, parsing `attr`
   as a `LitStr` and falling back to the function name when it is empty.
4. Make `#[timed]` reject being applied to a `struct` with an error that spans
   the struct's name. Capture the exact compiler output.
5. Write a `#[proc_macro_attribute]` called `#[deprecated_soon]` that keeps the
   function and prepends `#[deprecated(note = "...")]` to it. Verify the warning
   appears at the call site, not the definition.

---

[← Course Index](../README.md)
