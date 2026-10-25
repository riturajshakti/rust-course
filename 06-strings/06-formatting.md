[← 6.5 UTF-8, Chars, Bytes, and Grapheme Safety](05-utf8-chars.md) · [Course Index](../README.md) · [6.7 Parsing and Converting Text →](07-parsing.md)

# 6.6 — Formatting and the format! Family

## What You'll Learn

- The full format spec: positional, named, inline, fill/align, sign, precision
- Radix and scientific output: `{:x}`, `{:#b}`, `{:e}`
- The whole macro family, including `write!` into a `String`
- `Display` vs `Debug`, and the error when a type has neither

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `` `${a} and ${b}` `` | `format!("{a} and {b}")` |
| Python f-string `f"{x:.2f}"` | `format!("{x:.2}")` |
| Python `"%s" % x` | `format!("{}", x)` |
| Go `fmt.Sprintf("%d", n)` | `format!("{}", n)` — no type letter |
| Go `fmt.Printf("%v")` / `%+v` | `{:?}` / `{:#?}` |

Rust's spec is closest to Python's. The big difference from Go/C: **no type
letters**. `{}` works for every type that implements `Display`, and the format
string is checked at compile time — a wrong argument count is a build error, not
a runtime surprise.

## Positional, named, and inline arguments

```rust
let name = "raj";
let age = 30;

println!("{} is {}", name, age);
println!("{0} {1} {0}", name, age);
println!("{n} is {a}", n = name, a = age);
println!("{name} is {age}");
```

Output:

```
raj is 30
raj 30 raj
raj is 30
raj is 30
```

The last form — **inline captures** — reads a variable with that exact name from
scope. It's the idiomatic choice whenever the value is already in a local.
Positional `{0}` is for repeating one argument.

Inline capture only works for a bare variable name; `{self.x}` or `{a + b}` do
not work, so compute into a local first.

## Escaping braces

```rust
println!("{{literal braces}}");
let name = "raj";
println!("{{{name}}}");
println!("{}%", 50);
```

Output:

```
{literal braces}
{raj}
50%
```

`{{` and `}}` are the only escapes. `%` needs none.

## `{:?}` and `{:#?}`

`{:?}` is `Debug` — the developer-facing form. `{:#?}` is the same, pretty.

```rust
println!("{:?}", vec![1, 2]);
println!("{:#?}", vec![1, 2]);
println!("{:?}", Some("x"));
println!("{:?}", ("a", 1, true));
println!("{:?}", "quoted\ttab");
println!("{}", "quoted\ttab");
```

Output:

```
[1, 2]
[
    1,
    2,
]
Some("x")
("a", 1, true)
"quoted\ttab"
quoted	tab
```

The last two lines are the reason to reach for `{:?}` when debugging strings: it
adds quotes and escapes, so you can see whitespace.

## Fill, alignment, and width

The spec is `{:[[fill]align][sign][#][0][width][.precision][type]}`. Fill comes
first, then an alignment arrow.

```rust
println!("[{:>8}]", "hi");
println!("[{:<8}]", "hi");
println!("[{:^8}]", "hi");
println!("[{:*^8}]", "hi");
println!("[{:.>8}]", "hi");
println!("[{:-^20}]", " mid ");
println!("[{:>3}]", "longer text");
```

Output:

```
[      hi]
[hi      ]
[   hi   ]
[***hi***]
[......hi]
[------- mid --------]
[longer text]
```

| Align | Meaning | Default for |
|---|---|---|
| `<` | left | non-numeric types |
| `^` | center | — |
| `>` | right | numeric types |

Width is a **minimum** — nothing is ever truncated by width alone. Width counts
`char`s, not terminal columns, so CJK and emoji will not align visually.

## Zero padding and sign

```rust
println!("[{:08}]", 42);
println!("[{:+}]", 42);
println!("[{:+}]", -42);
println!("[{:+}]", 0);
println!("[{:+08.2}]", 3.14159);
```

Output:

```
[00000042]
[+42]
[-42]
[+0]
[+0003.14]
```

`0` is a distinct flag from `fill`+`align`: it pads **after** the sign, which is
what you want for numbers. `+` forces a sign on positives; negatives always show
one.

## Precision

For floats, precision is decimal places. For strings, it **truncates**.

```rust
println!("[{:.2}]", 1234.5678);
println!("[{:.5}]", 1.0 / 3.0);
println!("[{:.0}]", 3.7);
println!("[{:.0}]", 2.5);
println!("[{:.3}]", "truncated");
println!("[{:10.3}]", "truncated");
println!("[{:08.3}]", 3.14159);
```

Output:

```
[1234.57]
[0.33333]
[4]
[2]
[tru]
[tru       ]
[0003.142]
```

`{:.0}` on `2.5` gives `2`, not `3` — ties round to even. Combine width and
precision freely: `{:10.3}` is "at least 10 wide, at most 3 of precision".

## Dynamic width and precision

Three syntaxes, all resolved at compile time against an argument.

```rust
let w = 12;
let p = 4;

println!("[{:.p$}]", 3.14159265);
println!("[{:.1$}]", 3.14159265, p);
println!("[{:.*}]", 3, 3.14159265);
println!("[{:>w$}]", "dyn");
println!("[{:>w$.p$}]", 3.14159265);
println!("[{:*>width$}]", "x", width = 5);
```

Output:

```
[3.1416]
[3.1416]
[3.142]
[         dyn]
[      3.1416]
[****x]
```

| Form | Takes the value from |
|---|---|
| `w$` | A variable or named argument called `w` |
| `1$` | Positional argument 1 |
| `.*` | The **next** positional argument, consumed *before* the value |

`{:.*}` is the odd one: it reads two arguments, precision first. Prefer `w$`
with named locals — it's far easier to read.

## Radix: hex, octal, binary

```rust
let n = 255;
println!("{:x} {:X} {:o} {:b}", n, n, n, n);
println!("{:#x} {:#X} {:#o} {:#b}", n, n, n, n);
```

Output:

```
ff FF 377 11111111
0xff 0xFF 0o377 0b11111111
```

`#` adds the base prefix. Combine with zero padding — note the prefix counts
toward the width:

```rust
println!("{:08b}", 5);
println!("{:#010b}", 5);
println!("{:04x}", 255);
println!("{:#06x}", 255);
println!("{:x}", -1i8);
```

Output:

```
00000101
0b00000101
00ff
0x00ff
ff
```

`{:#010b}` produces 10 characters total including `0b`. Negative integers print
their two's-complement bits, so `-1i8` is `ff`.

### Debug-hex byte dumps

`{:02x}` per byte is the standard hex-dump idiom.

```rust
for b in "Hi🦀".as_bytes() {
    print!("{:02x} ", b);
}
println!();

let hex: String = "Hi🦀".as_bytes().iter().map(|b| format!("{:02x}", b)).collect();
println!("{}", hex);
```

Output:

```
48 69 f0 9f a6 80 
4869f09fa680
```

## Scientific notation

```rust
println!("[{:e}]", 1234.5678);
println!("[{:E}]", 1234.5678);
println!("[{:e}]", 0.00012);
println!("[{:.2e}]", 1234.5678);
```

Output:

```
[1.2345678e3]
[1.2345678E3]
[1.2e-4]
[1.23e3]
```

Rust writes `e3`, not `e+03` — no sign or zero padding on the exponent.

## The macro family

| Macro | Writes to | Returns |
|---|---|---|
| `print!` | stdout, no newline | `()` |
| `println!` | stdout + `\n` | `()` |
| `eprint!` | stderr, no newline | `()` |
| `eprintln!` | stderr + `\n` | `()` |
| `format!` | a new `String` | `String` |
| `write!` | any `fmt::Write` or `io::Write` | `Result` |
| `writeln!` | same, + `\n` | `Result` |

```rust
print!("no newline");
print!(" -> still same line\n");
println!("with newline");
eprintln!("this goes to stderr");
let s = format!("{}-{}", 1, 2);
println!("{}", s);
```

Output:

```
no newline -> still same line
with newline
this goes to stderr
1-2
```

`stdout` is line-buffered, so a `print!` without a newline may not appear until
later; `println!` or an explicit `flush()` releases it. Diagnostics go to
stderr so they survive a redirected stdout.

## `write!` into a `String`

This is the no-extra-allocation way to build text. It needs the `fmt::Write`
trait in scope.

```rust
use std::fmt::Write;

let mut out = String::new();
write!(out, "{}-{}", "a", "b").unwrap();
writeln!(out, "!").unwrap();
write!(out, "{:>5}", 42).unwrap();
println!("{:?}", out);
```

Output:

```
"a-b!\n   42"
```

The `Result` exists because `write!` is generic over sinks that can fail (files,
sockets). Writing to a `String` cannot fail, so `.unwrap()` is correct there.

The same macro writes to bytes and files via `io::Write`:

```rust
use std::io::Write;

let mut buf: Vec<u8> = Vec::new();
write!(buf, "{} bytes", 5).unwrap();
println!("{:?}", String::from_utf8(buf).unwrap());

let mut stdout = std::io::stdout();
writeln!(stdout, "direct to stdout").unwrap();
stdout.flush().unwrap();
```

Output:

```
"5 bytes"
direct to stdout
```

The two `Write` traits collide if you import both, so alias one:
`use std::fmt::Write as FmtWrite;`.

## `Display` vs `Debug`

`{}` requires `Display`, which you must write by hand. `{:?}` requires `Debug`,
which you derive.

Without `Display`:

```rust
struct Point { x: i32, y: i32 }
println!("{}", Point { x: 1, y: 2 });
```

Does not compile:

```
error[E0277]: `Point` doesn't implement `std::fmt::Display`
 --> src/main.rs:3:20
  |
3 |     println!("{}", Point { x: 1, y: 2 });
  |               --   ^^^^^^^^^^^^^^^^^^^^ `Point` cannot be formatted with the default formatter
  |               |
  |               required by this formatting parameter
  |
help: the trait `std::fmt::Display` is not implemented for `Point`
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
```

Without `Debug`:

```rust
struct Point { x: i32, y: i32 }
println!("{:?}", Point { x: 1, y: 2 });
```

Does not compile:

```
error[E0277]: `Point` doesn't implement `Debug`
 --> src/main.rs:3:22
  |
3 |     println!("{:?}", Point { x: 1, y: 2 });
  |               ----   ^^^^^^^^^^^^^^^^^^^^ `Point` cannot be formatted using `{:?}` because it doesn't implement `Debug`
help: consider annotating `Point` with `#[derive(Debug)]`
  |
1 + #[derive(Debug)]
2 | struct Point { x: i32, y: i32 }
```

`Debug` has a derive; `Display` deliberately does not, because there is no
sensible default for user-facing text.

With both:

```rust
use std::fmt;

#[derive(Debug)]
struct Point { x: i32, y: i32 }

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

let p = Point { x: 1, y: 2 };
println!("{}", p);
println!("{:?}", p);
println!("{:#?}", p);
println!("{:>12}", p);
println!("{}", p.to_string());
```

Output:

```
(1, 2)
Point { x: 1, y: 2 }
Point {
    x: 1,
    y: 2,
}
(1, 2)
(1, 2)
```

Two things to note. Implementing `Display` gives you `.to_string()` for free —
it comes from the blanket `ToString` impl. And `{:>12}` was **ignored**: a hand-
written `Display` that calls `write!` directly does not apply the caller's
padding. Use `f.pad(&string)` if you want width to work. Module 09 covers
implementing these traits properly.

| Trait | Spec | Derive? | Audience |
|---|---|---|---|
| `Display` | `{}` | no | Users |
| `Debug` | `{:?}` / `{:#?}` | yes | Developers |
| `LowerHex` / `UpperHex` | `{:x}` / `{:X}` | no | — |
| `Binary` / `Octal` | `{:b}` / `{:o}` | no | — |
| `LowerExp` / `UpperExp` | `{:e}` / `{:E}` | no | — |
| `Pointer` | `{:p}` | no | — |

## Quick Reference

| Spec | Effect |
|---|---|
| `{}` | `Display` |
| `{:?}` / `{:#?}` | `Debug` / pretty `Debug` |
| `{0}` / `{name}` / `{var}` | Positional / named / inline capture |
| `{{` / `}}` | Literal brace |
| `{:>8}` `{:<8}` `{:^8}` | Right / left / center in width 8 |
| `{:*^8}` | Center, fill with `*` |
| `{:08}` | Zero-pad to 8, after the sign |
| `{:+}` | Always show sign |
| `{:.2}` | 2 decimals, or truncate a string to 2 chars |
| `{:10.3}` | Width 10, precision 3 |
| `{:w$}` / `{:.p$}` | Width/precision from variable `w`/`p` |
| `{:.1$}` | Precision from positional arg 1 |
| `{:.*}` | Precision from the next positional arg |
| `{:x}` `{:X}` `{:o}` `{:b}` | Hex, upper hex, octal, binary |
| `{:#x}` `{:#b}` `{:#o}` | Same, with `0x`/`0b`/`0o` prefix |
| `{:02x}` | Zero-padded hex — the byte-dump idiom |
| `{:e}` / `{:E}` | Scientific notation |
| `{:p}` | Pointer address |

## Common Mistakes

**Using `%d` or `%s`.** There are no type letters; `{}` handles every
`Display` type.

**Writing `{:*>8}` as `{:>*8}`.** Fill comes before the alignment arrow.

**Expecting `{}` to work on your struct.** `{}` needs a hand-written `Display`;
`{:?}` needs only `#[derive(Debug)]`.

**Forgetting `use std::fmt::Write` before `write!(my_string, ..)`.** The macro
resolves to a trait method that must be in scope.

**Interpolating an expression inline.** `{a.b}` and `{x + 1}` are not valid.
Bind a local first.

**Assuming width aligns terminal columns.** It counts `char`s, so emoji and CJK
break alignment.

## Exercises

1. Print a 3-column table of `(name, qty, price)` for five rows: name left in
   12, qty right in 5, price right in 10 with 2 decimals and a `+` sign.
2. Print `255` in decimal, hex, octal, and binary — each both with and without
   the `#` prefix, each zero-padded to 8 characters of digits.
3. Write `fn bar(pct: u32) -> String` rendering a 20-char progress bar like
   `[#####...............]` using `repeat` and `{:<20}`.
4. Build one `String` holding 10 lines of `"row N: value"` using only `writeln!`
   and a single allocation via `String::with_capacity`.
5. Give a `Temperature(f64)` tuple struct a `Display` impl printing one decimal
   and a `°C` suffix, then confirm `.to_string()` works and that `{:>10}` is
   ignored until you switch to `f.pad`.

---

[← 6.5 UTF-8, Chars, Bytes, and Grapheme Safety](05-utf8-chars.md) · [Course Index](../README.md) · [6.7 Parsing and Converting Text →](07-parsing.md)
