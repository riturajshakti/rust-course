[← Course Index](../README.md)

# 2.2 — Scalar Types: Integers, Floats, Bool, Char

## What You'll Learn

- Every integer type, its range, and when to pick `usize`
- What overflow does in debug vs release, and the four explicit alternatives
- Literal forms: separators, radixes, suffixes, byte and scientific notation
- Floats, NaN, and why `==` on floats is a bug
- `char` is 4 bytes and holds any Unicode scalar value

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `number` (one type) | `i8`…`i128`, `u8`…`u128`, `f32`, `f64` |
| Python arbitrary-precision `int` | fixed-width; overflow panics in debug |
| Go `int` (platform-sized) | `i32` default, `isize` for platform-sized |
| JS/Python implicit numeric coercion | none — every mix needs an explicit cast |

Four scalar families: integers, floats, `bool`, `char`. No `null`, no `NaN`
hiding in the integer types, and no silent widening anywhere.

## Integer types

| Length | Signed | Unsigned |
|---|---|---|
| 8-bit | `i8` | `u8` |
| 16-bit | `i16` | `u16` |
| 32-bit | `i32` | `u32` |
| 64-bit | `i64` | `u64` |
| 128-bit | `i128` | `u128` |
| pointer-sized | `isize` | `usize` |

`i32` is the default for an unconstrained integer literal. It is the right
choice until profiling or a protocol says otherwise.

## MIN and MAX

```rust
fn main() {
    println!("{} {}", i8::MIN, i8::MAX);
    println!("{} {}", i16::MIN, i16::MAX);
    println!("{} {}", i32::MIN, i32::MAX);
    println!("{} {}", i64::MIN, i64::MAX);
    println!("{} {}", i128::MIN, i128::MAX);
    println!("{} {}", u8::MIN, u8::MAX);
    println!("{} {}", u16::MIN, u16::MAX);
    println!("{} {}", u32::MIN, u32::MAX);
    println!("{} {}", u64::MIN, u64::MAX);
    println!("{} {}", u128::MIN, u128::MAX);
    println!("{} {}", isize::MIN, isize::MAX);
    println!("{} {}", usize::MIN, usize::MAX);
}
```

Output:

```
-128 127
-32768 32767
-2147483648 2147483647
-9223372036854775808 9223372036854775807
-170141183460469231731687303715884105728 170141183460469231731687303715884105727
0 255
0 65535
0 4294967295
0 18446744073709551615
0 340282366920938463463374607431768211455
-9223372036854775808 9223372036854775807
0 18446744073709551615
```

## usize and isize

Pointer-sized: 8 bytes on a 64-bit target, 4 on a 32-bit one.

```rust
println!("{} {}", size_of::<usize>(), size_of::<isize>());
```

Output:

```
8 8
```

`usize` is not "a big unsigned int" — it is the index/length type. Every
`.len()`, every `[i]`, and every capacity in the standard library is `usize`.
Use it for sizes and indices, and a fixed width (`u32`, `u64`) for data you
serialize or send over a wire.

## Literal forms

```rust
let big = 1_000_000;
let hex = 0xff;
let oct = 0o77;
let bin = 0b1010_0001;
let byte = b'A';
let suffixed = 42u8;
let f = 2.5f32;
let sci = 1.5e3;
println!("{big} {hex} {oct} {bin} {byte} {suffixed} {f} {sci}");
println!("{} {} {}", 0xDEAD_BEEFu32, 1_0_0, 1e-3);
```

Output:

```
1000000 255 63 161 65 42 2.5 1500
3735928559 100 0.001
```

`_` is a visual separator anywhere in a literal and is ignored. A type suffix
(`42u8`, `2.5f32`, `1_000i64`) pins the type inline, which is handy in
expressions where there is no binding to annotate. `b'A'` is a `u8`.

Printing in other radixes:

```rust
println!("{:b} {:o} {:x} {:X} {:#b} {:#x}", 161, 63, 255, 255, 5, 255);
```

Output:

```
10100001 77 ff FF 0b101 0xff
```

Parsing from a radix:

```rust
println!("{} {}", i32::from_str_radix("ff", 16).unwrap(), u8::from_str_radix("1010", 2).unwrap());
```

Output:

```
255 10
```

## A literal that doesn't fit is a compile error

```rust
let x: u8 = 256;
```

Does not compile:

```
error: literal out of range for `u8`
 --> src/main.rs:2:17
  |
2 |     let x: u8 = 256;
  |                 ^^^
  |
  = note: the literal `256` does not fit into the type `u8` whose range is `0..=255`
  = note: `#[deny(overflowing_literals)]` on by default
```

## Overflow: debug panics, release wraps

```rust
fn main() {
    let mut x: u8 = 254;
    x += 1;
    println!("{x}");
    x += 1;
    println!("{x}");
    x += 1;
    println!("{x}");
}
```

`cargo run` (debug):

```
255

thread 'main' (13777566) panicked at src/main.rs:5:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`cargo run --release`:

```
255
0
1
```

Release builds wrap by default (`overflow-checks = false`), because the check
costs a branch per operation. **Never rely on either behaviour.** If wrapping
is intended, say so with `wrapping_*`; if it is a bug, use `checked_*`. You can
force the check on in release with `overflow-checks = true` under
`[profile.release]` in `Cargo.toml`.

## The four explicit overflow families

```rust
let m: u8 = 255;
println!("{}", m.wrapping_add(1));
println!("{:?}", m.checked_add(1));
println!("{:?}", m.checked_add(0));
println!("{}", m.saturating_add(1));
println!("{:?}", m.overflowing_add(1));
```

Output:

```
0
None
Some(255)
255
(0, true)
```

| Family | Returns | On overflow |
|---|---|---|
| `wrapping_*` | `T` | wraps modulo 2^n |
| `checked_*` | `Option<T>` | `None` |
| `saturating_*` | `T` | clamps to `MIN`/`MAX` |
| `overflowing_*` | `(T, bool)` | wrapped value + `true` |

All four exist for `add`, `sub`, `mul`, `div`, `rem`, `neg`, `pow`, `shl`,
`shr`, and `abs`.

```rust
let z: u8 = 0;
println!("{} {:?} {} {:?}", z.wrapping_sub(1), z.checked_sub(1), z.saturating_sub(1), z.overflowing_sub(1));
println!("{} {:?}", 200u8.wrapping_mul(2), 200u8.checked_mul(2));
println!("{:?} {:?}", 10i32.checked_div(0), (-128i8).checked_neg());
println!("{} {}", i32::MAX.wrapping_add(1), i32::MIN.saturating_sub(1));
println!("{:?}", 1u8.checked_shl(9));
```

Output:

```
255 None 0 (255, true)
144 None
None None
-2147483648 -2147483648
None
```

`checked_div(0)` is `None` rather than a panic — the only way to divide without
risking a crash. `(-128i8).checked_neg()` is `None` because `128` is not an
`i8`. `saturating_sub` on unsigned types is the common "clamp at zero" idiom.

## Other useful integer methods

```rust
println!("{} {} {}", 2i32.pow(10), (-5i32).abs(), 17i32.signum());
println!("{} {}", 255u8.count_ones(), 255u8.leading_zeros());
println!("{:?} {:?}", "42".parse::<u8>(), "300".parse::<u8>());
```

Output:

```
1024 5 1
8 0
Ok(42) Err(ParseIntError { kind: PosOverflow })
```

`parse` returns `Result`, and out-of-range input is an error rather than a
truncation. Module 08 covers handling it.

## Mixing integer types is an error

```rust
let a: u8 = 5;
let b: u32 = 10;
println!("{}", a + b);
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:4:24
  |
4 |     println!("{}", a + b);
  |                        ^ expected `u8`, found `u32`

error[E0277]: cannot add `u32` to `u8`
   --> src/main.rs:4:22
    |
  4 |     println!("{}", a + b);
    |                      ^ no implementation for `u8 + u32`
    |
    = help: the trait `Add<u32>` is not implemented for `u8`
```

There is no implicit widening, even where it would be lossless. Lesson 2.5
covers `as` and the safe conversions.

## Floats

`f32` and `f64`, both IEEE 754. `f64` is the default and the right choice
unless you have a memory reason.

```rust
println!("{} {}", size_of::<f32>(), size_of::<f64>());
println!("{:e} {:e}", f32::MAX, f64::MAX);
println!("{:e} {:e}", f32::MIN_POSITIVE, f64::MIN_POSITIVE);
println!("{} {}", f32::EPSILON, f64::EPSILON);
```

Output:

```
4 8
3.4028235e38 1.7976931348623157e308
1.1754944e-38 2.2250738585072014e-308
0.00000011920929 0.0000000000000002220446049250313
```

There is no integer/float mixing either: `1 + 2.0` does not compile. Write
`1.0 + 2.0`.

## Float precision

```rust
println!("{}", 0.1f64 + 0.2);
println!("{}", 0.1f32 + 0.2f32);
println!("{:.20}", 0.1f64);
```

Output:

```
0.30000000000000004
0.3
0.10000000000000000555
```

`f32` prints `0.3` only because the shortest round-tripping decimal happens to
be `0.3` at that precision — the stored value is still inexact.

## Comparing floats

```rust
println!("{}", 0.1 + 0.2 == 0.3);
println!("{}", (0.1f64 + 0.2 - 0.3).abs() < f64::EPSILON);
```

Output:

```
false
true
```

Compare with a tolerance, never with `==`. For values of very different
magnitude, a relative tolerance (`(a - b).abs() / b.abs() < eps`) is needed —
a fixed `EPSILON` only works near 1.0.

## NaN and infinity

```rust
let nan = f64::NAN;
println!("{} {} {}", nan, nan == nan, nan.is_nan());
println!("{} {}", f64::INFINITY, f64::NEG_INFINITY);
println!("{} {}", 1.0f64 / 0.0, -1.0f64 / 0.0);
println!("{:?}", 0.0f64 / 0.0);
println!("{:?} {:?}", 1.0f64.partial_cmp(&2.0), nan.partial_cmp(&1.0));
```

Output:

```
NaN false true
inf -inf
inf -inf
NaN
Some(Less) None
```

`NaN != NaN`, which is exactly why floats implement `PartialOrd` but not `Ord`
and cannot be sorted or used as `HashMap` keys without a wrapper. Note float
division by zero gives infinity rather than panicking — unlike integer
division, which panics.

```rust
println!("{} {}", 2.0f64.sqrt(), (7.5f64).round());
println!("{} {}", 2.0f64.powi(10), 2.0f64.powf(0.5));
```

Output:

```
1.4142135623730951 8
1024 1.4142135623730951
```

A float literal out of range for its suffix is also rejected at compile time:
`1.0e40f32` gives `error: literal out of range for f32`.

## bool

```rust
let t = true;
let f: bool = false;
println!("{t} {f} {} {} {}", t && f, t || f, !t);
println!("{} {}", t as u8, f as u8);
println!("{:?} {:?}", "true".parse::<bool>(), "yes".parse::<bool>());
println!("{}", size_of::<bool>());
```

Output:

```
true false false true false
1 0
Ok(true) Err(ParseBoolError)
1
```

One byte, and only ever `true` or `false`. There is no truthiness: `if 1 { }`
and `if Some(x) { }` do not compile. Conditions must be `bool`.

## char

A `char` is one Unicode scalar value, always 4 bytes.

```rust
let c = 'z';
let heart: char = '♥';
let crab = '🦀';
let dev = 'द';
println!("{c} {heart} {crab} {dev}");
println!("{} {} {} {}", size_of::<char>(), size_of::<u8>(), "🦀".len(), crab.len_utf8());
```

Output:

```
z ♥ 🦀 द
4 1 4 4
```

`size_of::<char>()` is 4 because the type is a code point. `"🦀".len()` is also
4, but for a different reason: `str::len` counts **UTF-8 bytes**, and that crab
needs 4 of them. Single quotes are `char`, double quotes are `&str` — they are
unrelated types.

```rust
println!("{} {}", 'é'.len_utf8(), "héllo".len());
println!("{}", "héllo".chars().count());
```

Output:

```
2 6
5
```

## char conversions and queries

```rust
println!("{} {}", 'z' as u32, '🦀' as u32);
println!("{:?} {:?}", char::from_u32(0x1F980), char::from_u32(0x110000));
println!("{} {} {}", 'z'.is_alphabetic(), 'z'.is_numeric(), '7'.to_digit(10).unwrap());
println!("{} {}", 'A'.to_ascii_lowercase(), 'ß'.to_uppercase());
```

Output:

```
122 129408
Some('🦀') None
true false 7
a SS
```

`char as u32` always works; `u32 as char` does not exist, because not every
`u32` is a valid scalar value — hence `char::from_u32` returning `Option`.
`to_uppercase` returns an iterator, not a `char`, because one character can
uppercase to several. Module 06 covers text properly.

## Quick Reference

| Need | Use |
|---|---|
| Default integer | `i32` |
| Index / length / capacity | `usize` |
| Byte | `u8` |
| Default float | `f64` |
| Range of a type | `T::MIN`, `T::MAX` |
| Size in bytes | `size_of::<T>()` |
| Wrap intentionally | `a.wrapping_add(b)` |
| Detect overflow | `a.checked_add(b)` → `Option` |
| Clamp at the limit | `a.saturating_add(b)` |
| Value plus overflow flag | `a.overflowing_add(b)` → `(T, bool)` |
| Digit separators | `1_000_000` |
| Radix literals | `0xff`, `0o77`, `0b1010` |
| Byte literal | `b'A'` |
| Type suffix | `42u8`, `2.5f32` |
| Float tolerance compare | `(a - b).abs() < eps` |
| Is it NaN | `x.is_nan()` |
| Codepoint from number | `char::from_u32(n)` → `Option<char>` |

## Common Mistakes

**Assuming release wraps safely.** Debug panics and release wraps; both are
bugs waiting to happen. Pick `checked_*`, `saturating_*`, or `wrapping_*`
explicitly.

**Comparing floats with `==`.** `0.1 + 0.2 != 0.3`. Compare against a
tolerance, and never use a float as a `HashMap` key.

**Mixing widths and expecting coercion.** `u8 + u32` is a type error; cast one
side with `as` (lesson 2.5).

**Thinking `char` is a byte.** It is 4 bytes and one code point; a byte is
`u8`, and `str::len` counts bytes, not characters.

**Using `usize` for serialized data.** Its width changes with the target. Use
`u32`/`u64` on the wire.

## Exercises

1. Print `MIN` and `MAX` for `i16`, `u16`, and `i128`. Then make a literal that
   is one past `i16::MAX` and read the compile error.
2. Start from `u8::MAX` and show all four overflow families on `+ 1`, printing
   each result with its type's shape (`Option`, tuple, plain value).
3. Write a loop that sums `0.1` ten times and prints the result; then print
   whether it equals `1.0` with `==` and with an epsilon comparison.
4. Print the byte length, `char` count, and `size_of::<char>()` for the string
   `"naïve 🦀"`. Explain the three numbers.
5. Build a `u32` from the hex literal `0xCAFE_BABE`, print it in decimal,
   binary, and `{:#x}`, then convert `0x1F600` to a `char` and print it.

---

[← Course Index](../README.md)
