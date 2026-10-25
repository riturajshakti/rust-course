[← Course Index](../README.md)

# 2.5 — Numeric Operators and Type Casting

## What You'll Learn

- Every operator Rust has, and the two it does not (`++`, `--`)
- Integer division truncates toward zero, and `%` keeps the dividend's sign
- Short-circuit `&&`/`||` versus non-short-circuit `&`/`|`
- `as` casting rules: truncation, sign reinterpretation, float saturation
- Why there is no implicit coercion, and the safe alternatives

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `7 / 2` → `3.5` | `7 / 2` → `3` (both `i32`) |
| Python `//` for floor div | `/` already truncates on integers |
| Python `-7 % 2` → `1` | `-7 % 2` → `-1` (use `rem_euclid`) |
| JS/Go `i++` | `i += 1` |
| C implicit widening | none — `u8 + u32` is a type error |

## Arithmetic

```rust
fn main() {
    println!("{} {} {} {} {}", 5 + 3, 5 - 3, 5 * 3, 7 / 2, 7 % 2);
    println!("{} {}", 7.0 / 2.0, 7.0 % 2.0);
    println!("{}", -5i32);
}
```

Output:

```
8 2 15 3 1
3.5 1
-5
```

| Op | Name |
|---|---|
| `+` | add |
| `-` | subtract / negate |
| `*` | multiply |
| `/` | divide (truncating on integers) |
| `%` | remainder |

There is no `**`; use `x.pow(n)` for integers, `x.powi(n)` / `x.powf(y)` for
floats.

## Integer division truncates toward zero

```rust
println!("{} {}", -7 / 2, -7 % 2);
println!("{} {}", 7 / -2, 7 % -2);
```

Output:

```
-3 -1
-3 1
```

`/` rounds toward zero (not toward negative infinity), and `%` takes the sign
of the **dividend** — so `a % b` can be negative. This matches C and Go, not
Python. For the always-non-negative version:

```rust
println!("{} {} {}", (-7i32).div_euclid(2), (-7i32).rem_euclid(2), (-7.0f64).rem_euclid(2.0));
```

Output:

```
-4 1 1
```

`rem_euclid` is what you want for wrapping an index or a clock value.

## Division by zero

Integer division by zero panics. With literals it is caught at compile time:

```rust
println!("{}", 10 / 0);
```

```
error: this operation will panic at runtime
 --> src/main.rs:2:20
  |
2 |     println!("{}", 10 / 0);
  |                    ^^^^^^ attempt to divide `10_i32` by zero
  |
  = note: `#[deny(unconditional_panic)]` on by default
```

With a runtime zero:

```
thread 'main' (13806992) panicked at src/main.rs:3:20:
attempt to divide by zero
```

Use `checked_div` to get `None` instead (lesson 2.2). Float division by zero
gives `inf`, not a panic.

## No ++ or --

```rust
let mut n = 10;
n += 5; n -= 3; n *= 2; n /= 4; n %= 4;
println!("{n}");
```

Output:

```
2
```

Increment and decrement operators do not exist — too easy to write
`i++ + ++i`. Also note compound assignment is a statement, not an expression:
`let y = (x += 1);` does not produce a value.

## Comparison

```rust
println!("{} {} {} {} {} {}", 1 == 1, 1 != 1, 1 < 2, 1 <= 1, 2 > 1, 2 >= 3);
```

Output:

```
true false true true true false
```

Both operands must be the same type. Comparisons are not chainable:
`a < b < c` fails, since `a < b` is a `bool`.

## Logical operators and short-circuiting

```rust
fn side(label: &str, v: bool) -> bool {
    println!("evaluated {label}");
    v
}

fn main() {
    println!("{}", side("a", false) && side("b", true));
    println!("---");
    println!("{}", side("a", true) || side("b", true));
    println!("---");
    println!("{}", side("a", false) & side("b", true));
}
```

Output:

```
evaluated a
false
---
evaluated a
true
---
evaluated a
evaluated b
false
```

`&&` and `||` short-circuit; `&` and `|` on `bool` do not. Operands must be
`bool` — there is no truthiness, and the result is `bool`, not one of the
operands (no JS-style `a || default`; use `unwrap_or` for that).

Short-circuiting is how you guard an index:

```rust
let xs = [1, 2, 3];
let i = idx();
if i < xs.len() && xs[i] > 0 { println!("yes"); } else { println!("guarded"); }
```

Output:

```
guarded
```

## Bitwise

```rust
println!("{} {} {} {} {}", 0b1100 & 0b1010, 0b1100 | 0b1010, 0b1100 ^ 0b1010, !0u8, !0i8);
```

Output:

```
8 14 6 255 -1
```

| Op | Meaning |
|---|---|
| `&` | AND |
| `\|` | OR |
| `^` | XOR |
| `!` | NOT (bitwise on integers, logical on `bool`) |

`!` is overloaded: `!true` is `false`, `!0u8` is `255`. There is no separate
`~`.

## Shifts

```rust
println!("{} {} {}", 1u8 << 3, 16u8 >> 2, -16i32 >> 2);
println!("{}", (-16i32 as u32) >> 2);
```

Output:

```
8 4 -4
1073741820
```

`>>` is arithmetic (sign-extending) on signed types and logical on unsigned —
the type decides, so there is no `>>>`. Shifting by at least the bit width
panics in debug and is UB-free but unspecified elsewhere; `checked_shl` /
`wrapping_shl` make the intent explicit.

## Compound assignment

```rust
let mut b = 0b1100u8;
b &= 0b1010; b |= 0b0001; b ^= 0b1111; b <<= 1; b >>= 2;
println!("{:04b}", b);
```

Output:

```
0011
```

All of `+= -= *= /= %= &= |= ^= <<= >>=` exist, and all require `mut`.

## Precedence

Standard C-like precedence. The one that surprises people is that `as` binds
**tighter** than arithmetic:

```rust
println!("{}", 1 + 2 as u8);
println!("{}", -1i32 as u8 as u32);
println!("{}", (1 + 2) as u8);
```

Output:

```
3
255
3
```

`1 + 2 as u8` is `1 + (2 as u8)`. Casts chain left to right. When in doubt,
parenthesize — clippy will not complain.

## No implicit coercion

```rust
let a: u8 = 5;
let b: u32 = a;
```

Does not compile:

```
error[E0308]: mismatched types
 --> src/main.rs:3:18
  |
3 |     let b: u32 = a;
  |            ---   ^ expected `u32`, found `u8`
  |            |
  |            expected due to this
  |
help: you can convert a `u8` to a `u32`
  |
3 |     let b: u32 = a.into();
  |                   +++++++
```

Even a lossless widening needs to be written out. That is deliberate: it makes
every lossy conversion visible at the call site.

## `as` casts

`as` always succeeds and never panics — which is exactly why it can lose data.
Widening within the same signedness is lossless:

```rust
println!("{} {}", 5i32 as i64, 5u8 as u32);
```

Output:

```
5 5
```

**Narrowing truncates** — it keeps the low bits:

```rust
println!("{} {} {}", 300i32 as u8, 1000i32 as u8, 65535u32 as u16);
```

Output:

```
44 232 65535
```

`300 as u8` is `300 - 256 = 44`; `1000 as u8` is `1000 - 768 = 232`.

**Signed/unsigned casts reinterpret the bits** (two's complement):

```rust
println!("{}", -1i32 as u8);
println!("{}", -1i32 as u32);
println!("{}", 255u8 as i8);
println!("{}", 200u8 as i8);
println!("{}", -128i8 as u8);
println!("{}", 127i8 as u8);
```

Output:

```
255
4294967295
-1
-56
128
127
```

One rule covers all six: truncate to the target width, then reinterpret the
bits under the target's signedness.

**Float to int truncates toward zero and saturates** at the target's range:

```rust
println!("{} {} {}", 3.9f64 as i32, -3.9f64 as i32, 0.9f64 as i32);
println!("{} {}", 1e10f64 as i32, -1e10f64 as i32);
println!("{} {}", 300.0f64 as u8, -5.0f64 as u8);
println!("{}", f64::NAN as i32);
println!("{} {}", f64::INFINITY as i32, f64::NEG_INFINITY as i64);
```

Output:

```
3 -3 0
2147483647 -2147483648
255 0
0
2147483647 -9223372036854775808
```

Saturating (not wrapping) is the defined behaviour: out of range clamps to
`MIN`/`MAX`, and `NaN` becomes `0`. This is the one place `as` is *safer* than
you might expect from C.

**Int to float may lose precision**, and a too-large `f64` becomes `inf`:

```rust
println!("{} {}", 5i32 as f64, 16_777_217i32 as f32);
println!("{} {}", 1.0e40f64 as f32, 0.1f64 as f32);
```

Output:

```
5 16777216
inf 0.1
```

`f32` has 24 bits of mantissa, so `16_777_217` is not representable.

**bool and char casts** go one way only:

```rust
println!("{} {} {}", true as i32, false as u8, 'A' as u8);
println!("{} {}", 'A' as u32, '🦀' as u8);
println!("{}", 97u8 as char);
```

Output:

```
1 0 65
65 128
a
```

`'🦀' as u8` truncating to `128` is a silent bug waiting to happen — prefer
`u32::from(c)`. Only `u8` may be cast to `char`:

```rust
let n: u32 = 65;
println!("{}", n as char);
```

```
error[E0604]: only `u8` can be cast as `char`, not `u32`
 --> src/main.rs:3:20
  |
3 |     println!("{}", n as char);
  |                    ^^^^^^^^^ invalid cast
  |
help: consider using `char::from_u32` instead
  |
3 -     println!("{}", n as char);
3 +     println!("{}", char::from_u32(n));
  |
```

And `as` is not a parser:

```rust
let s = "42";
let n = s as i32;
```

```
error[E0606]: casting `&str` as `i32` is invalid
 --> src/main.rs:3:13
  |
3 |     let n = s as i32;
  |             ^^^^^^^^
  |
  = help: cast through a raw pointer first
```

Use `s.parse::<i32>()`.

## usize casts in practice

```rust
let v = vec![1, 2, 3];
let n: u32 = v.len() as u32;
let total: u64 = v.len() as u64 * 8;
let f = v.len() as f64 / 2.0;
println!("{n} {total} {f}");
```

Output:

```
3 24 1.5
```

`.len()` is `usize`, so any arithmetic mixing it with a sized integer or a
float needs a cast. The trap is casting a signed value that might be negative:

```rust
let i: i32 = -1;
println!("{}", (i + 1) as usize);
println!("{}", i as usize);
```

Output:

```
0
18446744073709551615
```

`-1 as usize` is a plausible-looking index that will panic far from the actual
bug. Validate first, cast after.

## The safe alternatives

```rust
let small: u8 = 7;
let wide: u64 = u64::from(small);
println!("{wide}");
println!("{:?}", u8::try_from(300u32));
println!("{:?}", u8::try_from(200u32));
```

Output:

```
7
Err(TryFromIntError(PosOverflow))
Ok(200)
```

| Conversion | Use |
|---|---|
| Always lossless (`u8` → `u64`) | `u64::from(x)` or `x.into()` |
| Might not fit (`u32` → `u8`) | `u8::try_from(x)` → `Result` |
| Deliberate truncation | `x as u8` |
| Text to number | `s.parse::<T>()` → `Result` |

Rule of thumb: reach for `From`/`TryFrom` first and `as` only when truncation
or saturation is the behaviour you actually want. Lesson 9.7 covers `From`,
`Into`, and `TryFrom` in full.

## Quick Reference

| Category | Operators |
|---|---|
| Arithmetic | `+ - * / %`, unary `-` |
| Comparison | `== != < <= > >=` |
| Logical (short-circuit) | `&& \|\|`, `!` |
| Bitwise | `& \| ^ !` |
| Shift | `<< >>` |
| Compound | `+= -= *= /= %= &= \|= ^= <<= >>=` |
| Cast | `as` |
| Missing | `++`, `--`, `**`, `~`, `>>>`, `?:` |

| Need | Form |
|---|---|
| Truncating integer divide | `a / b` |
| Non-negative remainder | `a.rem_euclid(b)` |
| Power | `a.pow(n)`, `a.powi(n)`, `a.powf(x)` |
| Divide without panicking | `a.checked_div(b)` |
| Lossless widen | `u64::from(x)` |
| Checked narrow | `u8::try_from(x)` |
| Deliberate truncate | `x as u8` |
| Length into arithmetic | `v.len() as u64` |

## Common Mistakes

**Expecting `7 / 2` to be `3.5`.** Integer operands give integer division; cast
one side to `f64` first.

**Using `%` for a wrapped index.** `-1 % 5` is `-1`. Use `rem_euclid`.

**Casting a negative number to `usize`.** Produces a huge index. Validate
first, cast after.

**Using `as` to "convert" between numeric types routinely.** It silently
truncates; use `From`/`TryFrom` unless truncation is the goal.

**Writing `i++`.** Not valid Rust. `i += 1`.

## Exercises

1. Print `a / b` and `a % b` for all four sign combinations of `7` and `2`, then
   the same with `div_euclid` / `rem_euclid`, and describe the difference.
2. Write `fn pct(part: usize, whole: usize) -> f64` returning a percentage, and
   handle `whole == 0` without panicking.
3. Cast `-1i64` to `u8`, `u16`, `u32`, and `u64`. Predict each result before
   running it.
4. Cast `f64::NAN`, `1e20`, `-1e20`, and `2.9` to `i32`. Explain all four.
5. Take a `u32` from a function and produce a `u8` two ways: with `as`, and
   with `u8::try_from`. Print both for input `200` and `300`.

---

[← Course Index](../README.md)
