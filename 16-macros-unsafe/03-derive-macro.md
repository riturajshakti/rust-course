[← Course Index](../README.md)

# 16.3 — Writing a Derive Macro

## What You'll Learn

- Parsing with `parse_macro_input!` and `DeriveInput`
- Walking `Data::Struct` fields: named, tuple, and unit
- Generating an impl with `quote!`, including generics via `split_for_impl`
- Adding a helper attribute (`#[rename = "..."]`) and spanned errors

## Coming From Another Language

| Other | Rust |
|---|---|
| `JSON.stringify` walking own properties at runtime | derive macro walking fields at **compile** time |
| Python `dataclasses.fields(obj)` | `DeriveInput` -> `Data::Struct` -> `Fields` |
| Go `reflect.TypeOf(v).Field(i).Name` | `field.ident` in the macro |

Rust has no runtime reflection. A derive macro is how you get field names at
all — it reads them from the source tokens and bakes the result into the binary.

## The target trait

We will derive `Describe`, which prints a struct's field names and values.

```rust
trait Describe {
    fn describe(&self) -> String;
}
```

In a real crate this trait lives in the normal library crate and the derive in
its companion `proc-macro = true` crate (16.2). Here the trait is in the
consumer so both files stay readable.

## Step 1: the skeleton

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    quote! {
        impl Describe for #name {
            fn describe(&self) -> String {
                stringify!(#name).to_string()
            }
        }
    }
    .into()
}
```

`DeriveInput` is `syn`'s model of anything `#[derive]` can go on:

| Field | Holds |
|---|---|
| `attrs` | outer attributes on the item |
| `vis` | `pub`, `pub(crate)`, or nothing |
| `ident` | the type's name |
| `generics` | type params, lifetimes, const params, `where` clause |
| `data` | `Data::Struct` / `Data::Enum` / `Data::Union` |

That is already a working derive. Everything below adds detail.

## Step 2: reject what you don't support

Return a spanned error rather than panicking.

```rust
let data = match &input.data {
    Data::Struct(s) => s,
    _ => {
        return syn::Error::new_spanned(&name, "Describe only supports structs")
            .to_compile_error()
            .into()
    }
};
```

## Step 3: walk the fields

`DataStruct::fields` is a three-way enum, and each arm needs different access
syntax.

| Variant | Struct shape | Access |
|---|---|---|
| `Fields::Named` | `struct S { a: u8 }` | `self.a` |
| `Fields::Unnamed` | `struct S(u8);` | `self.0` |
| `Fields::Unit` | `struct S;` | no fields |

```rust
let lines = match &data.fields {
    Fields::Named(named) => named
        .named
        .iter()
        .map(|f| {
            let ident = f.ident.as_ref().unwrap();
            let label = ident.to_string();
            quote! { out.push(format!("  {} = {:?}", #label, self.#ident)); }
        })
        .collect::<Vec<_>>(),
    Fields::Unnamed(unnamed) => unnamed
        .unnamed
        .iter()
        .enumerate()
        .map(|(i, _)| {
            let idx = syn::Index::from(i);
            let label = i.to_string();
            quote! { out.push(format!("  {} = {:?}", #label, self.#idx)); }
        })
        .collect::<Vec<_>>(),
    Fields::Unit => Vec::new(),
};
```

Two things that bite:

- `f.ident` is `Option<Ident>` — `None` for tuple fields. That is why the two
  arms are separate.
- For tuple fields you **must** use `syn::Index::from(i)`, not the `usize`.
  Interpolating a plain `usize` emits `self.0usize`, which does not compile.

Then splice the vector into the impl with `#(#lines)*`:

```rust
let header = name.to_string();
quote! {
    impl Describe for #name {
        fn describe(&self) -> String {
            let mut out = vec![format!("{} {{", #header)];
            #(#lines)*
            out.push("}".to_string());
            out.join("\n")
        }
    }
}
.into()
```

`quote!` repetition mirrors `macro_rules!`: `#(#v)*`, `#(#v),*`,
`#(#v);*` all work over anything iterable.

## Step 4: generics with split_for_impl

Naive `impl Describe for #name` breaks the moment the struct has a type
parameter, because the impl needs the params in three different positions.
`split_for_impl()` returns exactly those three pieces:

```rust
let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

quote! {
    impl #impl_generics Describe for #name #ty_generics #where_clause {
        /* ... */
    }
}
```

| Piece | Expands to |
|---|---|
| `impl_generics` | `<T: Clone, 'a>` — the declaration, with bounds |
| `ty_generics` | `<T, 'a>` — the use, no bounds |
| `where_clause` | `where T: Debug` or nothing |

Never hand-build these. It is the one `syn` helper you cannot skip.

### Adding the bound the generated code needs

`describe` formats every field with `{:?}`, so each field type must be `Debug`.
A bare generic has no such bound:

```rust
#[derive(Describe)]
struct Bare<T> { inner: T }
```

Does not compile (intentional):

```
error[E0277]: `T` doesn't implement `Debug`
 --> consumer/src/main.rs:5:10
  |
5 | #[derive(Describe)]
  |          ^^^^^^^^ `T` cannot be formatted using `{:?}` because it doesn't implement `Debug`
  |
help: consider restricting type parameter `T` with trait `Debug`
  |
6 | struct Bare<T: std::fmt::Debug> { inner: T }
  |              +++++++++++++++++
```

Pushing the users' structs around is rude. Inject the bound into the generics
*before* splitting — this is what `#[derive(Clone)]` and `serde` do:

```rust
let mut input = parse_macro_input!(input as DeriveInput);
let name = input.ident.clone();
for p in input.generics.type_params_mut() {
    p.bounds.push(syn::parse_quote!(::std::fmt::Debug));
}
let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
```

`parse_quote!` is `quote!`'s sibling: it builds tokens and immediately parses
them into the `syn` type inferred from context (here a `TypeParamBound`). The
leading `::` keeps the path absolute so it resolves in any module.

## Step 5: a helper attribute

Helper attributes must be registered, or the compiler rejects `#[rename]` as
unknown:

```rust
#[proc_macro_derive(Describe, attributes(rename))]
```

Then read it off the field. `#[rename = "..."]` is a name-value meta, so
`require_name_value()` then match the literal:

```rust
fn field_rename(f: &syn::Field) -> syn::Result<Option<String>> {
    for attr in &f.attrs {
        if attr.path().is_ident("rename") {
            let nv = attr.meta.require_name_value()?;
            if let syn::Expr::Lit(syn::ExprLit { lit: syn::Lit::Str(s), .. }) = &nv.value {
                return Ok(Some(s.value()));
            }
            return Err(syn::Error::new_spanned(
                &nv.value,
                "expected a string literal: #[rename = \"...\"]",
            ));
        }
    }
    Ok(None)
}
```

The named-fields arm now prefers the rename, and turns a parse failure into
error tokens in place of that one line:

```rust
let label = match field_rename(f) {
    Ok(Some(s)) => s,
    Ok(None) => ident.to_string(),
    Err(e) => return e.to_compile_error(),
};
```

For the `#[attr(a, b = 1)]` list form instead, use
`attr.parse_nested_meta(|m| { ... })`; for a bare `#[flag]`,
`attr.meta.require_path_only()`.

## The complete macro

`describe_derive/src/lib.rs`:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(Describe, attributes(rename))]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let mut input = parse_macro_input!(input as DeriveInput);
    let name = input.ident.clone();
    for p in input.generics.type_params_mut() {
        p.bounds.push(syn::parse_quote!(::std::fmt::Debug));
    }
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

    let data = match &input.data {
        Data::Struct(s) => s,
        _ => {
            return syn::Error::new_spanned(&name, "Describe only supports structs")
                .to_compile_error()
                .into()
        }
    };

    let lines = match &data.fields {
        Fields::Named(named) => named
            .named
            .iter()
            .map(|f| {
                let ident = f.ident.as_ref().unwrap();
                let label = match field_rename(f) {
                    Ok(Some(s)) => s,
                    Ok(None) => ident.to_string(),
                    Err(e) => return e.to_compile_error(),
                };
                quote! { out.push(format!("  {} = {:?}", #label, self.#ident)); }
            })
            .collect::<Vec<_>>(),
        Fields::Unnamed(unnamed) => unnamed
            .unnamed
            .iter()
            .enumerate()
            .map(|(i, _)| {
                let idx = syn::Index::from(i);
                let label = i.to_string();
                quote! { out.push(format!("  {} = {:?}", #label, self.#idx)); }
            })
            .collect::<Vec<_>>(),
        Fields::Unit => Vec::new(),
    };

    let header = name.to_string();
    quote! {
        impl #impl_generics Describe for #name #ty_generics #where_clause {
            fn describe(&self) -> String {
                let mut out = vec![format!("{} {{", #header)];
                #(#lines)*
                out.push("}".to_string());
                out.join("\n")
            }
        }
    }
    .into()
}

fn field_rename(f: &syn::Field) -> syn::Result<Option<String>> {
    for attr in &f.attrs {
        if attr.path().is_ident("rename") {
            let nv = attr.meta.require_name_value()?;
            if let syn::Expr::Lit(syn::ExprLit { lit: syn::Lit::Str(s), .. }) = &nv.value {
                return Ok(Some(s.value()));
            }
            return Err(syn::Error::new_spanned(
                &nv.value,
                "expected a string literal: #[rename = \"...\"]",
            ));
        }
    }
    Ok(None)
}
```

## It works

`consumer/src/main.rs`:

```rust
use describe_derive::Describe;

trait Describe {
    fn describe(&self) -> String;
}

#[derive(Describe)]
struct User {
    id: u64,
    #[rename = "e-mail"]
    email: String,
    active: bool,
}

#[derive(Describe)]
struct Pair(i32, &'static str);

#[derive(Describe)]
struct Marker;

#[derive(Describe)]
struct Wrapper<T> {
    inner: T,
}

fn main() {
    let u = User { id: 7, email: "raj@example.com".to_string(), active: true };
    println!("{}", u.describe());
    println!("{}", Pair(3, "hi").describe());
    println!("{}", Marker.describe());
    println!("{}", Wrapper { inner: vec![1, 2] }.describe());
}
```

Output:

```
User {
  id = 7
  e-mail = "raj@example.com"
  active = true
}
Pair {
  0 = 3
  1 = "hi"
}
Marker {
}
Wrapper {
  inner = [1, 2]
}
```

All four shapes work, the rename applies to exactly one field, and the generic
struct compiles without the user writing a bound.

## The errors

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

error: could not compile `consumer` (bin "consumer") due to 2 previous errors
```

Both errors in one build, each spanned at the user's token.

A field type that is not `Debug` fails inside the generated `format!`, and the
note tells the reader where it came from:

```rust
struct NoDebug;

#[derive(Describe)]
struct Holder { inner: NoDebug }
```

Does not compile (intentional):

```
error[E0277]: `NoDebug` doesn't implement `Debug`
 --> consumer/src/main.rs:7:10
  |
7 | #[derive(Describe)]
  |          ^^^^^^^^ `NoDebug` cannot be formatted using `{:?}` because it doesn't implement `Debug`
  |
  = note: this error originates in the macro `$crate::__export::format_args` which comes from the expansion of the derive macro `Describe`
help: consider annotating `NoDebug` with `#[derive(Debug)]`
```

Errors from *generated* code land on the `#[derive]`. This is why a derive
should emit as little code as possible and delegate to real functions: a bug in
a 200-line expansion is debugged by reading `cargo expand`, not the error.

## Reading what you generated

```
cargo install cargo-expand
cargo expand -p consumer
```

The first thing to check is the impl header, which confirms the injected bound:

```
struct Wrapper<T> {
    inner: T,
}
impl<T: ::std::fmt::Debug> Describe for Wrapper<T> {
    fn describe(&self) -> String {
```

`cargo expand` expands **everything**, including `std`'s own macros, so your
`vec![]` and `format!` come back unrecognizable:

```
struct Marker;
impl Describe for Marker {
    fn describe(&self) -> String {
        let mut out = ::alloc::boxed::box_assume_init_into_vec_unsafe(
            ::alloc::intrinsics::write_box_via_move(
                ::alloc::boxed::Box::new_uninit(),
                [
                    ::alloc::__export::must_use({
                        ::alloc::fmt::format(format_args!("{0} {{", "Marker"))
                    }),
                ],
            ),
        );
        out.push("}".to_string());
        out.join("\n")
    }
}
```

That noise is `vec![format!(...)]`, not something your macro emitted. Read past
it to the impl header and the field lines. Check the expansion whenever a derive
misbehaves: it is almost always a missing bound, a wrong path, or `self.0usize`.

## Quick Reference

| API | Purpose |
|---|---|
| `parse_macro_input!(input as DeriveInput)` | parse the derived item |
| `input.ident` | type name |
| `input.generics.split_for_impl()` | `(impl_generics, ty_generics, where_clause)` |
| `input.generics.type_params_mut()` | iterate type params to add bounds |
| `syn::parse_quote!(T)` | build + parse into a `syn` type |
| `input.data` | `Data::Struct` / `Enum` / `Union` |
| `data.fields` | `Fields::Named` / `Unnamed` / `Unit` |
| `f.ident` | `Option<Ident>`; `None` for tuple fields |
| `f.ty` | the field's type |
| `syn::Index::from(i)` | tuple index that interpolates as `0`, not `0usize` |
| `attributes(rename)` in the derive attr | register a helper attribute |
| `attr.path().is_ident("rename")` | match the attribute name |
| `attr.meta.require_name_value()` | read `#[a = v]` |
| `attr.parse_nested_meta(...)` | read `#[a(b, c = 1)]` |
| `quote! { #(#v)* }` | splice a `Vec<TokenStream>` |
| `syn::Error::new_spanned(t, m).to_compile_error()` | error at `t`'s span |

## Common Mistakes

**Interpolating a `usize` tuple index.** `self.#i` with `i: usize` emits
`self.0usize`. Use `syn::Index::from(i)`.

**Unwrapping `f.ident`.** It is `None` for tuple structs, so handle
`Fields::Unnamed` separately instead of panicking.

**Hand-writing the generics.** `impl<#generics>` loses the where clause and
duplicates bounds. Always `split_for_impl()`.

**Forgetting `attributes(...)`.** Without registering it, `#[rename]` is
"cannot find attribute" before your macro ever runs.

**Writing unqualified paths in `quote!`.** Generated code is resolved at the
call site, so emit `::std::fmt::Debug` and `::std::string::String`, not the
short names.

## Exercises

1. Build the workspace from this lesson and reproduce all four outputs.
2. Add `#[skip]` as a second helper attribute that omits a field from the
   output. Register it and verify the `email` field disappears.
3. Extend the macro to `Data::Enum`: generate a `describe` that returns the
   variant name via a `match self { ... }`. Handle unit variants only, and error
   on variants with fields.
4. Change `Describe` to return `Vec<(String, String)>` of name/value pairs
   instead of a formatted `String`, so the caller decides the formatting.
5. Write a `#[derive(Builder)]` that generates `User::builder()`, a
   `UserBuilder` struct with `Option` fields, one setter per field, and a
   `build() -> Result<User, String>` that errors naming the first missing field.

---

[← Course Index](../README.md)
