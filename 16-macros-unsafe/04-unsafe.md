[← 16.3 Writing a Derive Macro](03-derive-macro.md) · [Course Index](../README.md) · [16.5 FFI: Calling C and Being Called →](05-ffi.md)

# 16.4 — unsafe Rust and Raw Pointers

## What You'll Learn

- The exactly five things `unsafe` unlocks — and the many it does not
- Raw pointers: `*const T`, `*mut T`, `&raw`, null, dangling, arithmetic
- `unsafe` block vs `unsafe fn` vs `unsafe trait`, unions, `static mut`
- `MaybeUninit`, `std::ptr`, `from_raw_parts`, `transmute`, Miri, and the SAFETY discipline

## Coming From Another Language

| Other | Rust |
|---|---|
| C: every pointer, always | `unsafe { *p }`, opt-in and localized |
| Go `unsafe.Pointer` | `*mut T` plus an `unsafe` block |
| C# `unsafe` + `fixed` | `unsafe` block; no pinning needed |
| Python `ctypes` | `extern "C"` + raw pointers (16.5) |

`unsafe` does **not** turn off the borrow checker. Lifetimes, ownership, moves,
and type checking all still apply inside an `unsafe` block. It only permits five
extra operations, and it shifts responsibility for their invariants from the
compiler to you.

## The five things unsafe unlocks

1. **Dereference a raw pointer** (`*p`)
2. **Call an `unsafe fn`**, including `extern` functions
3. **Access or modify a mutable static** (`static mut`)
4. **Access a union field**
5. **Implement an `unsafe trait`**

That is the complete list. A move, a borrow, an overflow, an index all behave
identically inside and outside `unsafe`.

```rust
let v = vec![1, 2, 3];
unsafe {
    let moved = v;
    println!("{:?}", v);
}
```

Does not compile (intentional):

```
error[E0382]: borrow of moved value: `v`
 --> src/main.rs:5:26
  |
2 |     let v = vec![1, 2, 3];
  |         - move occurs because `v` has type `Vec<i32>`, which does not implement the `Copy` trait
4 |         let moved = v;
  |                     - value moved here
5 |         println!("{:?}", v);
  |                          ^ value borrowed here after move
```

It also warns `unnecessary unsafe block` — none of the five operations is used.

`unsafe` bought nothing. It is a permission slip for five operations, not an
escape hatch from Rust.

## Raw pointers

Two types, differing only in what you may do through them: `*const T` and
`*mut T`. Unlike references they may be null, may dangle, may be unaligned, may
alias freely, and carry no lifetime.

```rust
fn main() {
    let mut n = 42;
    let p: *const i32 = &n;
    let q: *mut i32 = &mut n;

    println!("null? {} size {}", p.is_null(), size_of::<*const i32>());

    unsafe {
        println!("deref p = {}", *p);
        *q = 100;
        println!("after write = {}", *p);
    }
}
```

Output:

```
null? false size 8
deref p = 42
after write = 100
```

`p` and `q` coexist as a shared and a mutable pointer to one place, which the
borrow checker would never allow for `&`/`&mut`. **Creating a raw pointer is
safe; dereferencing it is unsafe** — only the read or write can cause harm.

### &raw const / &raw mut

Coercing from `&n` goes through a real reference, which is a problem when the
place is packed, uninitialized, or a `static mut`. `&raw const` and `&raw mut`
take the address without ever forming a reference.

```rust
fn main() {
    let mut n = 42;
    let p = &raw const n;
    let q = &raw mut n;
    unsafe { *q += 1; println!("{}", *p); }

    let boxed = Box::new(7);
    let dangling = &raw const *boxed;
    drop(boxed);
    println!("dangling is non-null: {}", !dangling.is_null());

    let null: *const i32 = std::ptr::null();
    println!("null? {}", null.is_null());
    unsafe { println!("as_ref: {:?}", null.as_ref()); }
}
```

Output:

```
43
dangling is non-null: true
null? true
as_ref: None
```

A dangling pointer still reports non-null: `is_null()` is not a validity check,
and nothing in the language can tell you a pointer is still good. `as_ref()` is
the one null-aware accessor, giving `Option<&T>`. Any integer also casts to a
pointer (`1usize as *const i32`) — how you address memory-mapped hardware, and
how you create UB by typo.

### Pointer arithmetic and slices

```rust
fn main() {
    let v = vec![10, 20, 30];
    let base = v.as_ptr();
    unsafe {
        for i in 0..3 {
            println!("v[{}] = {}", i, *base.add(i));
        }
    }

    let mut arr = [1, 2, 3, 4, 5];
    let ptr = arr.as_mut_ptr();
    unsafe {
        println!("slice {:?}", std::slice::from_raw_parts(ptr, 3));
        let sm = std::slice::from_raw_parts_mut(ptr, 5);
        sm[4] = 99;
    }
    println!("{:?}", arr);
}
```

Output:

```
v[0] = 10
v[1] = 20
v[2] = 30
slice [1, 2, 3]
[1, 2, 3, 4, 99]
```

`add`/`offset` count in **elements**, not bytes (`byte_add` for bytes). Both are
unsafe without any dereference, because computing an out-of-bounds address is
itself UB.

`slice::from_raw_parts(ptr, len)` manufactures a `&[T]` out of thin air:
requires `ptr` non-null and aligned, `len` elements initialized and in one
allocation, total size at most `isize::MAX`, and no `&mut` to that data for the
slice's lifetime. The returned lifetime is **inferred**, so it will outlive the
data if you let it — the most common real unsafe bug.

## unsafe block vs unsafe fn vs unsafe trait

| Form | Means |
|---|---|
| `unsafe { ... }` | "I have checked the invariants for these operations" |
| `unsafe fn f()` | "Callers must uphold a contract; read the docs" |
| `unsafe trait T` | "Implementors must uphold a contract" |
| `unsafe impl T for X` | "I promise `X` upholds it" |
| `#[unsafe(no_mangle)]` | an attribute whose misuse is unsound (16.5) |

```rust
unsafe fn read_at(p: *const i32, i: usize) -> i32 {
    unsafe { *p.add(i) }
}

unsafe trait Zeroable { fn zeroed() -> Self; }
unsafe impl Zeroable for u32 { fn zeroed() -> u32 { 0 } }

fn main() {
    let v = vec![7, 8, 9];
    unsafe { println!("read_at = {}", read_at(v.as_ptr(), 2)); }
    println!("zeroed = {}", u32::zeroed());
}
```

Output:

```
read_at = 9
zeroed = 0
```

`Send`/`Sync` are the `unsafe trait`s you will actually meet (Module 14): the
compiler cannot prove thread-safety, so `unsafe impl` is you promising it.

Note the `unsafe` block **inside** `read_at`. In edition 2024 an `unsafe fn`
body is no longer implicitly unsafe, so dropping it gives:

```
warning[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:2:5
  |
2 |     *p.add(i)
  |     ^^^^^^^^^ dereference of raw pointer
  |
note: an unsafe function restricts its caller, but its body is safe by default
  = note: `#[warn(unsafe_op_in_unsafe_fn)]` (part of `#[warn(rust_2024_compatibility)]`) on by default
```

This splits the two meanings `unsafe fn` used to conflate: the contract on
callers, and the operations in the body. Write the inner block.

From safe code the same operation is an error, not a warning:

```
error[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:4:20
```

`E0133` also covers `get_unchecked`, `from_utf8_unchecked`, `unwrap_unchecked`,
and every other `_unchecked` in `std`.

## Unions

A union stores all fields at the same address; reading one interprets whatever
bytes are there.

```rust
union IntOrFloat { i: u32, f: f32 }

fn main() {
    let u = IntOrFloat { f: 1.0 };
    unsafe { println!("1.0f32 bits = 0x{:08x}", u.i); }
    println!("union size = {}", size_of::<IntOrFloat>());
}
```

Output:

```
1.0f32 bits = 0x3f800000
union size = 4
```

Writing a field is safe; reading one is unsafe because nothing records which
field is live. Unions are for C interop — in Rust-only code use an `enum`.

## static mut

A `static mut` is shared, mutable, global state with no synchronization — a data
race by construction. Edition 2024 makes it nearly unusable, deliberately:
**taking any reference to a `static mut` is a hard error, including a shared
reference for a read.**

```rust
static mut COUNTER: u32 = 0;
static mut NAMES: Vec<String> = Vec::new();

fn main() {
    unsafe {
        COUNTER += 1;
        println!("{}", COUNTER);
    }
    unsafe {
        NAMES.push("a".to_string());
    }
}
```

Does not compile (intentional):

```
error: creating a shared reference to mutable static
 --> src/main.rs:7:24
  |
7 |         println!("{}", COUNTER);
  |                        ^^^^^^^ shared reference to mutable static
  |
  = note: shared references to mutable statics are dangerous; it's undefined behavior if the static is mutated or if a mutable reference is created for it while the shared reference lives
  = note: `#[deny(static_mut_refs)]` (part of `#[deny(rust_2024_compatibility)]`) on by default

error: creating a mutable reference to mutable static
  --> src/main.rs:10:9
   |
10 |         NAMES.push("a".to_string());
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

`println!("{}", COUNTER)` takes `&COUNTER` to format, so even reading fails;
`NAMES.push(...)` autorefs to `&mut NAMES`. This was a warn-level lint in
edition 2021 and is `deny` in 2024, so ported code hits it immediately.

What still works, and what to do instead:

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static mut COUNTER: u32 = 0;
static GOOD: AtomicU32 = AtomicU32::new(0);

fn main() {
    unsafe {
        COUNTER += 1;
        COUNTER += 1;
        let v = COUNTER;
        println!("direct read: {}", v);
    }
    unsafe {
        let p = &raw mut COUNTER;
        *p += 10;
        println!("via raw pointer: {}", *p);
    }
    GOOD.fetch_add(5, Ordering::Relaxed);
    println!("atomic: {}", GOOD.load(Ordering::Relaxed));
}
```

Output:

```
direct read: 2
via raw pointer: 12
atomic: 5
```


Direct place access (`COUNTER += 1`, `let v = COUNTER`) forms no reference, so
it is allowed; `&raw mut` is the escape hatch for everything else. The real
answer is to not use `static mut`: an atomic, `static X: Mutex<T>`, `OnceLock`,
or `LazyLock` (Module 14) cover every sound use case.

## MaybeUninit

Rust has no uninitialized variable. Reading uninit memory is UB even for `u8`
and even if the value is unused, so `mem::uninitialized()` is gone.
`MaybeUninit<T>` is a `T`-sized, `T`-aligned slot the compiler knows may hold
garbage.

```rust
use std::mem::MaybeUninit;

fn main() {
    let mut slot: MaybeUninit<String> = MaybeUninit::uninit();
    let s = unsafe {
        slot.write("built later".to_string());
        slot.assume_init()
    };
    println!("{}", s);

    let mut buf: [MaybeUninit<u8>; 4] = [const { MaybeUninit::uninit() }; 4];
    for (i, slot) in buf.iter_mut().enumerate() {
        slot.write(i as u8 * 2);
    }
    let init: [u8; 4] = unsafe { std::mem::transmute(buf) };
    println!("{:?}", init);
}
```

Output:

```
built later
[0, 2, 4, 6]
```

`write` is safe (it initializes); `assume_init` is unsafe (it asserts
initialization). Never `assume_init` a partly filled buffer, and note
`MaybeUninit` does **not** drop its contents, so abandoning an initialized one
leaks.

## std::ptr operations

```rust
fn main() {
    let mut a = 5i32;
    let mut b = 9i32;
    unsafe {
        println!("read = {}", std::ptr::read(&a));
        std::ptr::write(&mut a, 77);
        std::ptr::swap(&mut a, &mut b);
        println!("a={} b={}", a, b);
    }

    let src = [1u8, 2, 3, 4];
    let mut dst = [0u8; 4];
    unsafe { std::ptr::copy_nonoverlapping(src.as_ptr(), dst.as_mut_ptr(), 4); }
    println!("{:?}", dst);
}
```

Output:

```
read = 5
a=9 b=77
[1, 2, 3, 4]
```

| Function | Does | Watch out |
|---|---|---|
| `ptr::read(p)` | moves the value out, bitwise | the original is now a duplicate; drop runs twice |
| `ptr::write(p, v)` | writes, **without** dropping the old value | leaks the old value if it needed dropping |
| `ptr::replace(p, v)` | write + return old | the safe pairing of the two above |
| `ptr::copy(s, d, n)` | `memmove`, may overlap | both ranges must be valid |
| `ptr::copy_nonoverlapping(s, d, n)` | `memcpy`, must not overlap | overlapping is UB, not just wrong |
| `ptr::write_bytes(p, b, n)` | `memset` | `n` is in elements of `T` |
| `ptr::drop_in_place(p)` | runs `Drop` without moving | required for `?Sized` |

## transmute, and why not to

`transmute::<A, B>` reinterprets bits. It requires only that `A` and `B` have
the same size, and it will cheerfully fabricate an invalid value.

```rust
fn main() {
    let bits: u32 = unsafe { std::mem::transmute(1.5f32) };
    println!("transmute = 0x{:08x}", bits);
    println!("safe way  = 0x{:08x}", 1.5f32.to_bits());
}
```

The compiler argues against it for you:

```
warning: unnecessary transmute
  --> src/main.rs:2:30
   |
 2 |     let bits: u32 = unsafe { std::mem::transmute(1.5f32) };
   |                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[warn(unnecessary_transmutes)]` on by default
help: replace this with
   |
 2 -     let bits: u32 = unsafe { std::mem::transmute(1.5f32) };
 2 +     let bits: u32 = unsafe { f32::to_bits(1.5f32) };
```

Output:

```
transmute = 0x3fc00000
safe way  = 0x3fc00000
```

Avoid it because it infers both types, so a refactor silently changes what you
transmute; it checks size but not layout, so `transmute::<&[T], &[U]>` or
anything `#[repr(Rust)]` is unsound; and nearly every real use has a safe
equivalent — `to_bits`/`from_bits`, `as` casts, `to_ne_bytes`,
`bytemuck`/`zerocopy` for byte slices, `char::from_u32`. The legitimate uses are
narrow: `[MaybeUninit<T>; N] -> [T; N]`, and lifetime-only changes inside a
library.

## Undefined behavior categories

UB is not a wrong answer. It is the compiler assuming the situation cannot
happen and optimizing on that — so the symptom shows up in an unrelated
function, only in release, only on one platform.

| Category | Example |
|---|---|
| Dereferencing an invalid pointer | null, dangling, out-of-bounds, misaligned |
| Reading uninitialized memory | `assume_init()` too early; any read of uninit, even `u8` |
| Breaking aliasing | a `&mut` while any other live reference to the same place exists |
| Producing an invalid value | `bool` that is not 0/1, a `char` above 0x10FFFF, a null `&T`, an invalid enum discriminant |
| Data race | two threads, same place, at least one write, no synchronization |
| Unwinding across a boundary where it is forbidden | a panic escaping `extern "C"` (16.5) |
| Violating a `std` function's contract | overlapping `copy_nonoverlapping`, bad `from_raw_parts` length |
| Mutating through a shared reference | writing via `&T` without `UnsafeCell` |
| Calling a function with the wrong signature | a mismatched `extern` declaration |

Inside `unsafe`, the compiler stops verifying pointer validity, alignment,
initialization, aliasing, the lifetimes of pointer-derived references, union
field liveness, and the contracts of the unsafe functions you call. The
asymmetry that matters: a bug in an `unsafe` block can break the guarantees of
**safe** code elsewhere, so unsafe code is judged on *soundness* — whether any
possible safe use of its public API can trigger UB — not on whether it works
today.

## Miri

Miri interprets your code and detects most UB: invalid pointers, uninit reads,
aliasing violations, misalignment, leaks. Nothing else finds these reliably,
because they do not reproduce. It needs nightly:

```
rustup +nightly component add miri
cargo +nightly miri test
cargo +nightly miri run
```

The workflow: put every invariant under a `#[test]`, then run
`cargo +nightly miri test` in CI beside the normal run.
`MIRIFLAGS="-Zmiri-strict-provenance"` tightens it further. Miri is 50-500x
slower than native, so it runs the test suite, not the workload. It cannot
execute real FFI calls into C, and it only sees code paths your tests reach —
unsafe code without tests gets no benefit.

## The discipline

Five rules that experienced Rust programmers follow without exception.

**1. Keep the `unsafe` block minimal.** Wrap the single operation, not the
function.

**2. Write a SAFETY comment on every `unsafe` block**, naming the invariant and
why it holds. This is the one place the usual "no comments in code" rule is
suspended: the invariant is not in the types, not checked by the compiler, and
not inferable from the code, so without it nobody can verify the block without
re-deriving the proof. Enforce it with
`#![warn(clippy::undocumented_unsafe_blocks)]`.

**3. Wrap unsafe in a safe API** whose contract cannot be violated.
`split_at_mut` is the canonical example — the borrow checker cannot see that two
halves of one slice are disjoint:

```rust
fn split_at_mut(s: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = s.len();
    let ptr = s.as_mut_ptr();
    assert!(mid <= len);
    unsafe {
        // SAFETY: mid <= len was asserted, so both ranges are in bounds of the
        // same allocation and they do not overlap, so the two &mut are disjoint.
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    let (a, b) = split_at_mut(&mut v, 2);
    a[0] = 100;
    b[0] = 200;
    println!("{:?}", v);
}
```

Output:

```
[100, 2, 200, 4, 5, 6]
```

The `assert!` makes it safe to call with any arguments; the `&mut [i32]` input
guarantees exclusivity and the returned lifetimes are tied to it. No caller can
cause UB.

**4. Test it under Miri.** See above.

**5. Check whether you need it at all.** The honest list: FFI (16.5), a data
structure the borrow checker cannot express, a measured win with a benchmark to
prove it, and hardware/OS interfaces. "The borrow checker is annoying" is not on
it — that is what `Rc`, `RefCell`, indices, and arenas are for (Module 11).

## Quick Reference

| Item | Meaning |
|---|---|
| `unsafe { }` | permits the five operations |
| `unsafe fn` | contract on callers; body still needs its own block |
| `unsafe trait` / `unsafe impl` | contract on implementors |
| `*const T` / `*mut T` | raw pointers; creating is safe, deref is not |
| `&raw const x` / `&raw mut x` | address without forming a reference |
| `ptr::null()` / `null_mut()` | the null pointer |
| `p.is_null()` | null check; **not** a validity check |
| `p.as_ref()` / `as_mut()` | `Option<&T>`, null-aware |
| `p.add(n)` / `offset(n)` / `byte_add(n)` | pointer arithmetic in elements/bytes |
| `slice::from_raw_parts(p, n)` | build a `&[T]`; lifetime is inferred |
| `MaybeUninit::uninit()` / `write` / `assume_init` | staged initialization |
| `ptr::read` / `write` / `copy` / `copy_nonoverlapping` | raw moves and memcpy |
| `ptr::drop_in_place(p)` | run `Drop` in place |
| `mem::transmute` | reinterpret bits; prefer a safe alternative |
| `static mut` | reference to it is a hard error in edition 2024 |
| `#[repr(C)]` | stable layout (16.5) |
| `cargo +nightly miri test` | detect UB |
| `clippy::undocumented_unsafe_blocks` | enforce SAFETY comments |

## Common Mistakes

**Thinking `unsafe` disables the borrow checker.** It permits five operations
and nothing else; moves and borrows are checked as usual.

**Treating `is_null()` as a validity check.** A dangling pointer is non-null.
Nothing in the language can tell you a pointer is still valid.

**Letting `from_raw_parts` infer a too-long lifetime.** The returned slice has
no connection to the source data; tie it to an input reference by returning it
from a function that takes one.

**Reading a `static mut` in edition 2024.** Even `println!("{}", S)` is a hard
error because formatting borrows. Use an atomic, `Mutex`, or `LazyLock`.

**Reaching for `transmute`.** It infers both types and checks only size. Use
`to_bits`, `as`, `to_ne_bytes`, or `bytemuck`.

**Writing one big `unsafe fn` with no SAFETY comments.** Minimal blocks, each
documented, wrapped in a safe API — otherwise nobody including you can audit it.

## Exercises

1. Make two raw pointers to one `i32`, write through one, read through the
   other. Then do the same with `&mut`/`&` and capture the borrow-check error.
2. Write `fn first_two(s: &[u8]) -> Option<(&u8, &u8)>` using
   `get_unchecked`, guarded by a length check, with a SAFETY comment.
3. Build a `[MaybeUninit<u64>; 8]`, fill it with squares, convert to `[u64; 8]`,
   print it. Then leave index 3 uninitialized and say why that is UB even though
   you never read index 3.
4. Convert a `static mut COUNTER: u64` program to compile under edition 2024
   three ways: direct place access, `&raw mut`, and `AtomicU64`. Compare.
5. Write a `fn as_bytes<T>(v: &T) -> &[u8]` with `from_raw_parts` and
   `size_of::<T>()`. Then explain why it is unsound for an arbitrary `T`
   (hint: padding bytes) and what bound would fix it.

---

[← 16.3 Writing a Derive Macro](03-derive-macro.md) · [Course Index](../README.md) · [16.5 FFI: Calling C and Being Called →](05-ffi.md)
