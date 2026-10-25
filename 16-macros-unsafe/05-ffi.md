[← 16.4 unsafe Rust and Raw Pointers](04-unsafe.md) · [Course Index](../README.md) · [17.1 Serde: Serialization and Deserialization →](../17-backend-axum/01-serde.md)

# 16.5 — FFI: Calling C and Being Called

## What You'll Learn

- `unsafe extern "C"` blocks and calling real libc functions
- `CString`/`CStr`, the NUL trap, and the `std::ffi` type table
- `#[repr(C)]` structs, callbacks, and who frees what
- Exposing Rust with `#[unsafe(no_mangle)]`, `catch_unwind`, bindgen/cbindgen, `cc`

## Coming From Another Language

| Other | Rust |
|---|---|
| Python `ctypes` / `cffi` | `unsafe extern "C"` block, resolved at link time |
| Go `cgo` | `extern "C"` + the `cc` crate, no special comments |
| Node `node-gyp` / N-API | `#[unsafe(no_mangle)] extern "C"` + `cdylib` |
| C `#include <string.h>` | a hand-written or `bindgen`-generated declaration |

Rust's FFI is the C ABI and nothing more: no marshalling, no runtime, no
glue. A declaration is a promise to the linker, checked by nobody — a wrong
signature compiles fine and corrupts the stack at runtime.

All examples below ran on macOS arm64 against the system libc.

## Calling libc

```rust
use std::ffi::{c_char, c_int, CString};

unsafe extern "C" {
    fn abs(input: c_int) -> c_int;
    fn strlen(s: *const c_char) -> usize;
    fn getpid() -> c_int;
    fn sqrt(x: f64) -> f64;
}

fn main() {
    unsafe {
        println!("abs(-7) = {}", abs(-7));
        println!("sqrt(2.0) = {}", sqrt(2.0));
        println!("getpid() > 0: {}", getpid() > 0);
    }

    let s = CString::new("hello ffi").unwrap();
    unsafe { println!("strlen = {}", strlen(s.as_ptr())); }
    println!("rust len = {}", "hello ffi".len());
}
```

Output:

```
abs(-7) = 7
sqrt(2.0) = 1.4142135623730951
getpid() > 0: true
strlen = 9
rust len = 9
```

Two edition-2024 requirements. The block must be `unsafe extern "C"`, not bare
`extern "C"`:

```
error: extern blocks must be unsafe
  --> src/main.rs:11:1
   |
11 |   extern "C" {
   |   ^
   |   |
   |  _help: needs `unsafe` before the extern keyword: `unsafe`
```

And every *call* still needs its own `unsafe` block, because the declaration is
unverified. `"C"` is the default ABI and may be omitted, but write it. Other
ABIs: `"system"` (`stdcall` on 32-bit Windows, `C` elsewhere), `"C-unwind"` for
functions that may throw a C++ exception through you.

libc needs no `#[link]` — it is always linked. For anything else, name the
library:

```rust
#[link(name = "z")]
unsafe extern "C" {
    fn compressBound(sourceLen: u64) -> u64;
}
```

| Form | Meaning |
|---|---|
| `#[link(name = "z")]` | link `libz` dynamically (`-lz`) |
| `#[link(name = "foo", kind = "static")]` | link `libfoo.a` |
| `#[link(name = "CoreFoundation", kind = "framework")]` | macOS framework |
| `println!("cargo:rustc-link-lib=z")` in `build.rs` | same, decided at build time |
| `println!("cargo:rustc-link-search=native=/path")` | add a library search path |

## Strings: CString, CStr, and the NUL trap

C strings are NUL-terminated byte pointers. Rust strings are length-prefixed
and may contain interior NUL bytes. The two types that bridge this:

| Type | Owns? | Role |
|---|---|---|
| `CString` | yes | Rust-built, NUL-terminated buffer to hand to C |
| `&CStr` | no | borrowed view of a C string (like `&str` to `String`) |
| `OsString`/`OsStr` | | platform strings for paths, not FFI |

```rust
use std::ffi::{c_char, CStr, CString};

unsafe extern "C" { fn strlen(s: *const c_char) -> usize; }

fn main() {
    let non_ascii = CString::new("héllo").unwrap();
    unsafe { println!("strlen(héllo) = {}", strlen(non_ascii.as_ptr())); }
    println!("chars = {}", "héllo".chars().count());

    println!("{:?}", CString::new("bad\0string"));

    let c = CStr::from_bytes_with_nul(b"from bytes\0").unwrap();
    println!("{:?} -> {}", c, c.to_str().unwrap());
    println!("{:?}", CStr::from_bytes_with_nul(b"no nul"));
}
```

Output:

```
strlen(héllo) = 6
chars = 5
Err(NulError(3, [98, 97, 100, 0, 115, 116, 114, 105, 110, 103]))
"from bytes" -> from bytes
Err(NotNulTerminated)
```

Three traps in that output:

- **`strlen` counts bytes, not characters.** `"héllo"` is 5 chars and 6 UTF-8
  bytes. C has no idea about encoding.
- **`CString::new` fails on an interior NUL**, returning `NulError` with the
  byte offset. It is fallible for exactly this reason — never `unwrap()` on
  untrusted input.
- **`CStr::from_bytes_with_nul` requires the terminator.** Use
  `from_bytes_with_nul_unchecked` only when you produced the bytes yourself.

Converting back: `CStr::to_str()` returns `Result<&str, Utf8Error>` because C
strings are arbitrary bytes; `to_string_lossy()` substitutes U+FFFD and never
fails.

### The dangling-CString trap

The single most common FFI bug in Rust:

```rust
let p = CString::new("temp").unwrap().as_ptr();
unsafe { println!("{}", strlen(p)); }
```

```
warning: this creates a dangling pointer because temporary `CString` is dropped at end of statement
 --> src/main.rs:6:43
  |
6 |     let p = CString::new("temp").unwrap().as_ptr();
  |             ----------------------------- ^^^^^^ pointer created here
  |             |
  |             this `CString` is dropped at end of statement
  |
  = help: bind the `CString` to a variable such that it outlives the pointer returned by `as_ptr`
  = note: a dangling pointer is safe, but dereferencing one is undefined behavior
  = note: `#[warn(dangling_pointers_from_temporaries)]` on by default
```

It is only a warning, and it compiles and often appears to work. Bind the
`CString` to a named variable that outlives every use of the pointer.

## The std::ffi type table

Use the `c_*` aliases, never `i32` directly — `c_long` is 8 bytes on 64-bit
Unix and 4 on Windows, and `c_char` is unsigned on ARM Linux.

```rust
use std::ffi::{c_char, c_double, c_int, c_long, c_longlong, c_short, c_uchar, c_uint, c_void};

fn main() {
    println!("c_char      {} signed={}", size_of::<c_char>(), (-1 as c_char) < 0);
    println!("c_short     {}", size_of::<c_short>());
    println!("c_int       {}", size_of::<c_int>());
    println!("c_uint      {}", size_of::<c_uint>());
    println!("c_long      {}", size_of::<c_long>());
    println!("c_longlong  {}", size_of::<c_longlong>());
    println!("c_double    {}", size_of::<c_double>());
    println!("*mut c_void {}", size_of::<*mut c_void>());
}
```

Output (macOS arm64):

```
c_char      1 signed=true
c_short     2
c_int       4
c_uint      4
c_long      8
c_longlong  8
c_double    8
*mut c_void 8
```

| C | Rust |
|---|---|
| `char` | `c_char` (signedness is platform-dependent) |
| `unsigned char` | `c_uchar` / `u8` |
| `short` / `int` / `long` / `long long` | `c_short` / `c_int` / `c_long` / `c_longlong` |
| `float` / `double` | `f32` / `f64` |
| `size_t` / `ssize_t` | `usize` / `isize` |
| `void *` | `*mut c_void` |
| `const char *` | `*const c_char` |
| `T *` (nullable) | `*mut T`, or `Option<&mut T>` if guaranteed non-null-or-null |
| `int (*)(int)` | `extern "C" fn(c_int) -> c_int` |
| `bool` (C99 `_Bool`) | `bool` — same ABI, both one byte |
| enum | `#[repr(C)] enum` or the integer type |

`Option<&T>`, `Option<Box<T>>`, and `Option<extern "C" fn()>` are
null-pointer-optimized and ABI-compatible with a nullable C pointer. Prefer them
over raw pointers when C documents "NULL or valid".

## Structs across the boundary: #[repr(C)]

Rust reorders fields to minimize padding. `#[repr(C)]` pins the C layout:
declaration order, C padding and alignment rules.

```rust
#[repr(C)]
#[derive(Debug, Clone, Copy)]
struct CPoint { x: f64, y: f64 }
```

```c
typedef struct { double x; double y; } CPoint;

double c_dist2(CPoint p) { return p.x * p.x + p.y * p.y; }
```

| Repr | Use |
|---|---|
| `#[repr(C)]` | anything crossing the boundary |
| `#[repr(transparent)]` | newtype with the exact ABI of its single field |
| `#[repr(u8)]` / `#[repr(i32)]` | fix an enum's discriminant type |
| `#[repr(packed)]` | no padding; creates unaligned fields, use `read_unaligned` |
| `#[repr(align(N))]` | raise alignment |
| `#[repr(Rust)]` (default) | **never** send across FFI |

Passing a non-`repr(C)` struct to C is UB even if it happens to work. A struct
containing `String`, `Vec`, `&str`, or a trait object cannot cross at all — the
layouts are not specified and C cannot free them. Pass `*const c_char` plus a
length, or an opaque pointer.

## Exposing Rust to C

Two attributes: `extern "C"` for the ABI, `#[unsafe(no_mangle)]` to keep the
symbol name. Edition 2024 requires the `unsafe(...)` wrapper:

```rust
#[no_mangle]
pub extern "C" fn rust_double(v: c_int) -> c_int { v * 2 }
```

```
error: unsafe attribute used without unsafe
  --> src/main.rs:19:3
   |
19 | #[no_mangle]
   |   ^^^^^^^^^ usage of unsafe attribute
   |
help: wrap the attribute in `unsafe(...)`
   |
19 | #[unsafe(no_mangle)]
   |   +++++++         +
```

The reasoning: `no_mangle` can silently collide with another symbol, including
one in libc, which is unsound — so it joins `link_section`, `export_name`, and
`link_name` in the unsafe-attribute family. The correct form:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_double(v: c_int) -> c_int { v * 2 }
```

To ship a library for C, set the crate type:

```toml
[lib]
crate-type = ["cdylib", "staticlib"]
```

`cdylib` gives `.so`/`.dylib`/`.dll`; `staticlib` gives `.a`. Use
`#[unsafe(export_name = "mylib_double")]` to namespace symbols — C has no
modules, so unprefixed names will collide.

## Building C with the cc crate

`build.rs`:

```rust
fn main() {
    cc::Build::new().file("csrc/helpers.c").compile("helpers");
    println!("cargo:rerun-if-changed=csrc/helpers.c");
}
```

```toml
[build-dependencies]
cc = "1"
```

`cc` finds the platform compiler, applies the right flags, and emits the
link directives for you. `.compile("helpers")` produces `libhelpers.a` and links
it. Add `.flag("-O2")`, `.define("NDEBUG", None)`, `.include("csrc")` as needed.
The `rerun-if-changed` line is required or edits to the C file are ignored.

## Callbacks, both directions

`csrc/helpers.c`:

```c
typedef struct { double x; double y; } CPoint;

double c_dist2(CPoint p) { return p.x * p.x + p.y * p.y; }

int c_apply(int (*f)(int), int v) { return f(v); }

int rust_double(int);

int c_calls_rust(int v) { return rust_double(v); }
```

```rust
use std::ffi::c_int;

#[repr(C)]
#[derive(Debug, Clone, Copy)]
struct CPoint { x: f64, y: f64 }

unsafe extern "C" {
    fn c_dist2(p: CPoint) -> f64;
    fn c_apply(f: extern "C" fn(c_int) -> c_int, v: c_int) -> c_int;
    fn c_calls_rust(v: c_int) -> c_int;
}

extern "C" fn triple(v: c_int) -> c_int { v * 3 }

#[unsafe(no_mangle)]
pub extern "C" fn rust_double(v: c_int) -> c_int { v * 2 }

fn main() {
    let p = CPoint { x: 3.0, y: 4.0 };
    unsafe {
        println!("c_dist2 = {}", c_dist2(p));
        println!("c_apply(triple, 14) = {}", c_apply(triple, 14));
        println!("c_calls_rust(21) = {}", c_calls_rust(21));
    }
}
```

Output:

```
c_dist2 = 25
c_apply(triple, 14) = 42
c_calls_rust(21) = 42
```

A callback must be `extern "C" fn`, which is a plain function pointer: **it
cannot capture.** To carry state, use the C convention of a `void *userdata`
parameter, pass a `Box::into_raw(Box::new(state)) as *mut c_void`, and cast it
back inside the callback. A capturing Rust closure cannot be a C callback.

## Ownership across FFI

Nothing is automatic. The rule that works: **whoever allocated it frees it, with
the matching allocator.** Rust's allocator is not `malloc`, so C must never
`free()` a Rust pointer and Rust must never `drop` a `malloc`ed one. Export a
free function alongside every constructor:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn make_greeting(name: *const c_char) -> *mut c_char {
    if name.is_null() { return std::ptr::null_mut(); }
    let name = unsafe { CStr::from_ptr(name) };
    let s = format!("hello, {}", name.to_string_lossy());
    CString::new(s).unwrap().into_raw()
}

#[unsafe(no_mangle)]
pub extern "C" fn free_greeting(p: *mut c_char) {
    if p.is_null() { return; }
    unsafe { drop(CString::from_raw(p)); }
}

fn main() {
    let input = CString::new("raj").unwrap();
    let out = make_greeting(input.as_ptr());
    unsafe { println!("{}", CStr::from_ptr(out).to_str().unwrap()); }
    free_greeting(out);
    println!("freed");
}
```

Output:

```
hello, raj
freed
```

| Operation | Effect |
|---|---|
| `CString::into_raw()` | gives up ownership; **must** come back via `from_raw` |
| `CString::from_raw(p)` | takes ownership back; drop frees it |
| `CStr::from_ptr(p)` | **borrows**; C still owns the memory |
| `Box::into_raw(b)` / `Box::from_raw(p)` | the same pair for any `T` |
| `std::mem::forget(x)` | leak deliberately, when C took ownership |

`CStr::from_ptr` is the dangerous one: its lifetime is inferred and unconnected
to anything, so copy the data out (`.to_owned()`, `.to_str()?.to_string()`)
rather than storing the `&CStr`.

For opaque handles, hand C a `*mut MyThing` from `Box::into_raw` and never let
it see the fields. C's header declares `typedef struct MyThing MyThing;`.

## Panics at the boundary

A panic unwinding out of an `extern "C"` function is undefined behavior. Since
Rust 1.81 the runtime aborts the process instead, which is safe but still
terminates your caller. Catch it:

```rust
use std::panic::catch_unwind;

#[unsafe(no_mangle)]
pub extern "C" fn rust_risky(v: c_int) -> c_int {
    match catch_unwind(|| {
        if v < 0 { panic!("negative input"); }
        v + 1
    }) {
        Ok(n) => n,
        Err(_) => -1,
    }
}

fn main() {
    println!("rust_risky(5) = {}", rust_risky(5));
    println!("rust_risky(-5) = {}", rust_risky(-5));
}
```

Output:

```
rust_risky(5) = 6

thread 'main' (13995204) panicked at src/main.rs:22:20:
negative input
rust_risky(-5) = -1
```

The panic message still goes to stderr — `catch_unwind` stops the unwind, it
does not silence the hook. Install `std::panic::set_hook` to suppress it.

Caveats: `catch_unwind` cannot catch an abort, so it does nothing under
`panic = "abort"` in your release profile. The closure must be
`UnwindSafe`, so `&mut` captures need `AssertUnwindSafe`. And every panic
caught must map to an error code or out-parameter, since you cannot return a
`Result` to C.

## bindgen and cbindgen

Hand-writing declarations does not scale past a few functions, and a typo is
UB.

**bindgen** (C header -> Rust declarations). `build.rs`:

```rust
fn main() {
    let out = std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap());
    bindgen::Builder::default()
        .header("csrc/helpers.h")
        .allowlist_function("c_.*")
        .generate()
        .unwrap()
        .write_to_file(out.join("bindings.rs"))
        .unwrap();
}
```

```rust
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

It needs libclang installed. Generating at build time tracks header changes;
`bindgen --output src/sys.rs header.h` once and committing the result removes
the libclang dependency for your users. Wrap the generated `unsafe` API in a
safe one by hand — bindgen output is raw pointers and `c_int`, never idiomatic.

**cbindgen** (Rust -> C header) for the other direction:

```
cbindgen --crate mylib --output include/mylib.h
```

It reads your `#[unsafe(no_mangle)] extern "C"` items and `#[repr(C)]` types
and emits a header, so the two sides cannot drift. Run it in `build.rs` or in
CI with a check that the committed header is up to date.

## Quick Reference

| Item | Purpose |
|---|---|
| `unsafe extern "C" { ... }` | declare foreign functions (required in edition 2024) |
| `extern "C" fn` | Rust fn with the C ABI |
| `#[unsafe(no_mangle)]` | keep the symbol name (required in edition 2024) |
| `#[unsafe(export_name = "x")]` | choose the symbol name |
| `#[link(name = "z")]` | link a library |
| `#[repr(C)]` | C-compatible layout |
| `#[repr(transparent)]` | newtype with its field's exact ABI |
| `CString::new(s)` | owned C string; `Err` on interior NUL |
| `cs.as_ptr()` | `*const c_char`; the `CString` must outlive it |
| `CString::into_raw` / `from_raw` | transfer ownership out / back |
| `CStr::from_ptr(p)` | borrow a C string; inferred lifetime |
| `cstr.to_str()` / `to_string_lossy()` | to `&str`, fallibly / lossily |
| `c_char`, `c_int`, `c_long`, `c_void` | platform-correct C types |
| `Option<&T>`, `Option<extern "C" fn()>` | nullable pointer, no `*mut` needed |
| `catch_unwind` | stop a panic before it crosses out |
| `crate-type = ["cdylib", "staticlib"]` | build for C consumers |
| `cc` crate in `build.rs` | compile and link C sources |
| `bindgen` / `cbindgen` | generate declarations / generate a header |

## Common Mistakes

**Calling `as_ptr()` on a temporary `CString`.** The buffer is freed at the end
of the statement. Bind it to a named variable; the compiler warns, but only
warns.

**Letting C `free()` a Rust pointer.** Different allocators. Export a
`free_*` function for everything you hand out.

**Passing a `#[repr(Rust)]` struct.** Field order is unspecified. Add
`#[repr(C)]`, and never send `String`, `Vec`, or `&str` across.

**Writing bare `extern "C" {}` or `#[no_mangle]`.** Both are hard errors in
edition 2024. Use `unsafe extern "C"` and `#[unsafe(no_mangle)]`.

**Letting a panic unwind out of `extern "C"`.** It aborts the process. Wrap the
body in `catch_unwind` and return an error code.

**Using `i32` where C says `long`.** `c_long` is 8 bytes on 64-bit Unix, 4 on
Windows. Always use the `c_*` aliases.

## Exercises

1. Declare and call `getenv`, `strcmp`, and `atoi` from libc. Handle `getenv`
   returning NULL without dereferencing it.
2. Write a C function `int sum_array(const int *p, size_t n)` and call it from
   Rust with a `Vec<i32>`, using the `cc` crate. Then call it with a deliberately
   wrong `n` and describe what the program does.
3. Export `#[unsafe(no_mangle)] extern "C" fn rust_reverse(*const c_char) ->
   *mut c_char` plus its free function. Verify with a C `main` that the round
   trip has no leaks.
4. Define a `#[repr(C)] struct Rect { int w; int h; }` on both sides, pass it by
   value and by pointer, and have C mutate it through the pointer. Print the
   sizes on both sides and confirm they agree.
5. Add a `void *userdata` callback: C calls an `extern "C" fn(c_int, *mut
   c_void)` and your Rust callback casts `userdata` back to a
   `&mut Vec<i32>` and pushes to it. Explain why a capturing closure cannot be
   used instead.

---

[← 16.4 unsafe Rust and Raw Pointers](04-unsafe.md) · [Course Index](../README.md) · [17.1 Serde: Serialization and Deserialization →](../17-backend-axum/01-serde.md)
