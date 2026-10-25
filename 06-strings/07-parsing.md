[← 6.6 Formatting and the format! Family](06-formatting.md) · [Course Index](../README.md) · [7.1 Vec: The Workhorse →](../07-collections/01-vec.md)

# 6.7 — Parsing and Converting Text

## What You'll Learn

- `.parse::<T>()`, the turbofish, and annotation-driven inference
- `ParseIntError`/`ParseFloatError` kinds and every way to handle them
- `FromStr` — implementing `.parse()` for your own type
- `String` ↔ `Vec<u8>` ↔ `char` conversions, including lossy decoding

## Coming From Another Language

| Other | Rust |
|---|---|
| JS `parseInt(s)` → `NaN` | `s.parse::<i32>()` → `Err` |
| JS `Number(s)` | `s.parse::<f64>()` |
| Python `int(s)` (raises) | `s.parse::<i32>()` (returns `Result`) |
| Go `strconv.Atoi(s)` | `s.parse::<i32>()` — same two-value shape |
| JS `String(n)` | `n.to_string()` |

One parse method covers every type, chosen by the type you ask for. Failure is a
`Result` you must handle — there is no `NaN` sentinel and no exception.

## `.parse()` and the turbofish

`parse` is generic over the return type, so you name it one of two ways.

```rust
println!("{:?}", "42".parse::<i32>());

let n: i32 = "42".parse().unwrap();
println!("{}", n);

println!("{:?}", "true".parse::<bool>());
println!("{:?}", "x".parse::<char>());
println!("{:?}", "-7".parse::<i64>());
println!("{:?}", "+7".parse::<i32>());
println!("{:?}", "1e3".parse::<f64>());
println!("{:?}", "inf".parse::<f64>());
```

Output:

```
Ok(42)
42
Ok(true)
Ok('x')
Ok(-7)
Ok(7)
Ok(1000.0)
Ok(inf)
```

The turbofish `::<i32>` states the type at the call; the annotation `let n: i32`
lets inference work backwards. Use the annotation when you already need one, the
turbofish when you don't — e.g. inside a `map`.

Every type implementing `FromStr` works — integers, floats, `bool`, `char`,
`IpAddr`, `SocketAddr`, `Duration` (via crates), and your own types.

## Parse failures and error kinds

`parse` is strict. Surrounding whitespace is **not** accepted.

```rust
println!("{:?}", "  7".parse::<i32>());
println!("{:?}", "7 ".parse::<i32>());
println!("{:?}", "".parse::<i32>());
println!("{:?}", "abc".parse::<i32>());
println!("{:?}", "3.5".parse::<i32>());
println!("{:?}", "999999999999".parse::<i32>());
println!("{:?}", "-1".parse::<u32>());
```

Output:

```
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: Empty })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: PosOverflow })
Err(ParseIntError { kind: InvalidDigit })
```

`"3.5".parse::<i32>()` fails rather than truncating, and `"-1"` into a `u32` is
an `InvalidDigit` (the `-` is not a legal digit), not an underflow.

| Error type | Kinds |
|---|---|
| `ParseIntError` | `Empty`, `InvalidDigit`, `PosOverflow`, `NegOverflow` |
| `ParseFloatError` | `Empty`, `Invalid` |
| `ParseBoolError` | — (only `"true"`/`"false"` parse) |
| `CharTryFromError` / `ParseCharError` | `EmptyString`, `TooManyChars` |

```rust
println!("{:?}", "abc".parse::<f64>());
println!("{:?}", "True".parse::<bool>());
println!("{:?}", "ab".parse::<char>());
println!("{}", "x".parse::<i32>().unwrap_err());
println!("{}", "9999999999".parse::<i32>().unwrap_err());
println!("{}", "x".parse::<f64>().unwrap_err());
```

Output:

```
Err(ParseFloatError { kind: Invalid })
Err(ParseBoolError)
Err(ParseCharError { kind: TooManyChars })
invalid digit found in string
number too large to fit in target type
invalid float literal
```

`bool` parsing is case-sensitive, so `"True"` fails. Each error's `Display` is a
readable sentence, and `.kind()` gives you the variant to branch on:

```rust
for s in ["42", "abc", "", "99999999999"] {
    match s.parse::<i32>() {
        Ok(n) => println!("{:?} -> {}", s, n),
        Err(e) => println!("{:?} -> error: {} (kind {:?})", s, e, e.kind()),
    }
}
```

Output:

```
"42" -> 42
"abc" -> error: invalid digit found in string (kind InvalidDigit)
"" -> error: cannot parse integer from empty string (kind Empty)
"99999999999" -> error: number too large to fit in target type (kind PosOverflow)
```

## Handling the `Result`

```rust
println!("{}", "x".parse::<i32>().unwrap_or(-1));
println!("{}", "x".parse::<i32>().unwrap_or_default());
println!("{}", "x".parse::<i32>().unwrap_or_else(|_| 0));
println!("{:?}", "x".parse::<i32>().ok());
println!("{:?}", "7".parse::<i32>().ok());
println!("{:?}", "x".parse::<i32>().map(|n| n * 2));
println!("{:?}", "x".parse::<i32>().map_err(|e| e.to_string()));
```

Output:

```
-1
0
0
None
Some(7)
Err(ParseIntError { kind: InvalidDigit })
Err("invalid digit found in string")
```

| Call | On `Err` | Use when |
|---|---|---|
| `unwrap()` | panics | Tests, literals you control |
| `expect("msg")` | panics with your message | Same, with context |
| `unwrap_or(v)` | gives `v` | You have a sensible fallback |
| `unwrap_or_default()` | gives `0`/`0.0`/`false` | Zero is the right default |
| `unwrap_or_else(f)` | calls `f` | Fallback is expensive to compute |
| `ok()` | `None` | You only care whether it worked |
| `map_err(f)` | wraps the error | Converting to your error type |
| `?` | returns early | Inside a function returning `Result` |

Control flow forms — `if let Ok(n) = "5".parse::<i32>()` and
`let Ok(m) = "6".parse::<i32>() else { return };` — both bind the parsed value
without a `match`. Module 08 covers `?` and error design in full.

## Parsing a collection

Skip the failures with `filter_map`:

```rust
let total: i32 = "1 2 3".split_whitespace().filter_map(|t| t.parse::<i32>().ok()).sum();
println!("{}", total);
```

Output:

```
6
```

Or fail the whole batch — `collect` into `Result<Vec<_>, _>` short-circuits on
the first error:

```rust
let all: Result<Vec<i32>, _> = "1,2,x".split(',').map(|t| t.parse::<i32>()).collect();
let good: Result<Vec<i32>, _> = "1,2,3".split(',').map(|t| t.parse::<i32>()).collect();
println!("{:?}", all);
println!("{:?}", good);
```

Output:

```
Err(ParseIntError { kind: InvalidDigit })
Ok([1, 2, 3])
```

That `Result<Vec<T>, E>` collect is one of Rust's best tricks: all-or-nothing
parsing in one line.

## Trim, then parse

Because `parse` rejects whitespace, every real-world parse starts with `trim`.

```rust
let nums = "1, 2,3 , 4";
let parsed: Vec<i32> = nums.split(',').map(|t| t.trim().parse().unwrap()).collect();
println!("{:?}", parsed);
```

Output:

```
[1, 2, 3, 4]
```

Make it a habit: `s.trim().parse()`, always.

## `from_str_radix` for other bases

`parse` is decimal-only. For hex, binary, or anything else:

```rust
println!("{:?}", i32::from_str_radix("ff", 16));
println!("{:?}", i32::from_str_radix("1010", 2));
println!("{:?}", u8::from_str_radix("zz", 36));
```

Output:

```
Ok(255)
Ok(10)
Err(ParseIntError { kind: PosOverflow })
```

Radix 2–36. The input must have no `0x` prefix — strip it with
`strip_prefix("0x")` first.

## `FromStr`: making your own type parseable

Implementing `FromStr` is what gives a type a `.parse()`.

```rust
use std::fmt;
use std::str::FromStr;

#[derive(Debug, PartialEq)]
struct Point { x: i32, y: i32 }

#[derive(Debug)]
enum PointError {
    MissingComma,
    BadNumber(std::num::ParseIntError),
}

impl fmt::Display for PointError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            PointError::MissingComma => write!(f, "expected 'x,y'"),
            PointError::BadNumber(e) => write!(f, "bad number: {e}"),
        }
    }
}

impl FromStr for Point {
    type Err = PointError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let (a, b) = s.trim().split_once(',').ok_or(PointError::MissingComma)?;
        Ok(Point {
            x: a.trim().parse().map_err(PointError::BadNumber)?,
            y: b.trim().parse().map_err(PointError::BadNumber)?,
        })
    }
}
```

The associated type `Err` is the only required choice. Now `.parse()` works:

```rust
println!("{:?}", "3,4".parse::<Point>());

let p: Point = " 1 , 2 ".parse().unwrap();
println!("{:?}", p);

println!("{:?}", "34".parse::<Point>());
println!("{:?}", "a,4".parse::<Point>());

match "a,4".parse::<Point>() {
    Ok(p) => println!("{:?}", p),
    Err(e) => println!("error: {}", e),
}

println!("{}", "5,6".parse::<Point>().unwrap() == Point { x: 5, y: 6 });
```

Output:

```
Ok(Point { x: 3, y: 4 })
Point { x: 1, y: 2 }
Err(MissingComma)
Err(BadNumber(ParseIntError { kind: InvalidDigit }))
error: bad number: invalid digit found in string
true
```

`Point::from_str("9,10")` is the same call spelled explicitly.

`PointError::BadNumber` used bare as a function in `map_err` — enum tuple
variants *are* functions, so `map_err(PointError::BadNumber)` needs no closure.

And the collection trick works on custom types for free:

```rust
let pts: Result<Vec<Point>, _> =
    "1,2 3,4".split_whitespace().map(|t| t.parse::<Point>()).collect();
println!("{:?}", pts);
```

Output:

```
Ok([Point { x: 1, y: 2 }, Point { x: 3, y: 4 }])
```

## The other direction: `to_string` via `Display`

`ToString` is blanket-implemented for every `Display` type, so one impl gives
you both.

```rust
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{},{}", self.x, self.y)
    }
}

println!("{}", Point { x: 7, y: 8 }.to_string());
println!("{}", 42.to_string());
println!("{}", true.to_string());
```

Output:

```
7,8
42
true
```

Never implement `ToString` directly — implement `Display`. Pairing it with
`FromStr` that accepts the same shape gives you a clean round trip.

## `String` ↔ `Vec<u8>`

```rust
let s = String::from("Hi🦀");
println!("{:?}", s.as_bytes());
let bytes = s.into_bytes();
println!("{:?}", bytes);
println!("{}", String::from_utf8(bytes).unwrap());
```

Output:

```
[72, 105, 240, 159, 166, 128]
[72, 105, 240, 159, 166, 128]
Hi🦀
```

| Direction | Call | Allocates? |
|---|---|---|
| `String` → `&[u8]` | `s.as_bytes()` | no — borrow |
| `String` → `Vec<u8>` | `s.into_bytes()` | no — reuses the buffer |
| `&str` → `Vec<u8>` | `s.as_bytes().to_vec()` | yes |
| `Vec<u8>` → `String` | `String::from_utf8(v)` → `Result` | no on success |
| `&[u8]` → `&str` | `std::str::from_utf8(b)` → `Result` | no |
| `&[u8]` → `String` | `String::from_utf8_lossy(b)` → `Cow` | only if invalid |

Byte-string literals (`b"..."`) are `&[u8; N]`, ASCII only — `b"ascii"` is
`[97, 115, 99, 105, 105]` and `std::str::from_utf8(b"ascii")` gives
`Ok("ascii")`.

## `from_utf8` vs `from_utf8_lossy`

`from_utf8` validates and refuses; `from_utf8_lossy` substitutes U+FFFD.

```rust
let bad = vec![72, 0xFF, 105];
println!("{:?}", String::from_utf8(bad.clone()));

match String::from_utf8(bad.clone()) {
    Ok(v) => println!("{}", v),
    Err(e) => println!(
        "failed at byte {}, recovered {:?}",
        e.utf8_error().valid_up_to(),
        e.into_bytes()
    ),
}
```

Output:

```
Err(FromUtf8Error { bytes: [72, 255, 105], error: Utf8Error { valid_up_to: 1, error_len: Some(1) } })
failed at byte 1, recovered [72, 255, 105]
```

The error hands your bytes back, so nothing is lost — you can retry with a
different decoder.

```rust
let bad = vec![72, 0xFF, 105];
let lossy = String::from_utf8_lossy(&bad);
println!("{} / {:?} / len={}", lossy, lossy, lossy.len());
println!("{:?}", lossy.chars().collect::<Vec<char>>());
println!("{}", char::REPLACEMENT_CHARACTER);
```

Output:

```
H�i / "H�i" / len=5
['H', '�', 'i']
�
```

Three input bytes became **five**: U+FFFD is itself 3 bytes in UTF-8. Lossy
decoding changes lengths, and it is irreversible — use it for display, never for
data you will store or compare.

Choose `from_utf8` for anything you'll act on; `from_utf8_lossy` only when
garbled output beats an error (log lines, terminal echo).

## `char` conversions

| Call | Returns | Note |
|---|---|---|
| `c.to_digit(radix)` | `Option<u32>` | `None` if not a digit in that radix |
| `char::from_digit(n, radix)` | `Option<char>` | `None` if `n >= radix` |
| `char::from_u32(n)` | `Option<char>` | `None` for surrogates / > U+10FFFF |
| `char::try_from(n: u32)` | `Result<char, _>` | Same check, as a `Result` |
| `c as u32` / `u32::from(c)` | `u32` | Always valid |
| `c as u8` | `u8` | **Truncates silently** |
| `b as char` (from `u8`) | `char` | Always valid — Latin-1 |
| `c.len_utf8()` | `usize` | Bytes this char needs |

```rust
println!("{:?}", '7'.to_digit(10));
println!("{:?}", 'f'.to_digit(16));
println!("{:?}", 'f'.to_digit(10));
println!("{:?}", 'z'.to_digit(36));
println!("{:?}", char::from_digit(7, 10));
println!("{:?}", char::from_digit(15, 16));
println!("{:?}", char::from_digit(15, 10));
println!("{:?}", char::from_u32(0x1F980));
println!("{:?}", char::from_u32(0xD800));
println!("{:?}", char::from_u32(0x110000));
println!("{:?}", char::try_from(65u32));
```

Output:

```
Some(7)
Some(15)
None
Some(35)
Some('7')
Some('f')
None
Some('🦀')
None
None
Ok('A')
```

`0xD800` is a UTF-16 surrogate and `0x110000` is past the top of Unicode —
neither is a valid `char`, which is why these return `Option`/`Result`.

Casts go one way safely:

```rust
println!("{}", 'A' as u32);
println!("{}", '🦀' as u32);
println!("{}", 'A' as u8);
println!("{}", '🦀' as u8);
println!("{}", 65u8 as char);
println!("{}", (b'a' + 2) as char);
println!("{}", '🦀'.len_utf8());
```

Output:

```
65
129408
65
128
A
c
4
```

`'🦀' as u8` gives **128** — a silent truncation of 129408 to its low byte, with
no warning. Only cast a `char` to `u8` after checking `is_ascii()`; use
`as u32` or `u32::from(c)` otherwise.

Digit parsing with `to_digit`:

```rust
println!("{:?}", "12a".chars().map(|c| c.to_digit(10)).collect::<Vec<_>>());

let digits: Option<Vec<u32>> = "123".chars().map(|c| c.to_digit(10)).collect();
println!("{:?}", digits);

let sum: u32 = "12345".chars().filter_map(|c| c.to_digit(10)).sum();
println!("{}", sum);
```

Output:

```
[Some(1), Some(2), None]
Some([1, 2, 3])
15
```

`collect` into `Option<Vec<_>>` short-circuits the same way `Result` does.

## Quick Reference

| Task | Call |
|---|---|
| Text → number | `s.trim().parse::<i32>()` |
| Text → number, other base | `i32::from_str_radix(s, 16)` |
| Text → number with default | `s.parse().unwrap_or(0)` |
| Text → `Option` | `s.parse::<i32>().ok()` |
| Many → all-or-nothing | `iter.map(\|t\| t.parse()).collect::<Result<Vec<_>, _>>()` |
| Many → skip failures | `iter.filter_map(\|t\| t.parse().ok())` |
| Number → text | `n.to_string()` or `format!("{n}")` |
| Own type → parseable | `impl FromStr for T` |
| Own type → printable | `impl Display for T` (gives `to_string`) |
| Digit char → value | `c.to_digit(10)` |
| Value → digit char | `char::from_digit(n, 10)` |
| Code point → char | `char::from_u32(n)` |
| Char → code point | `c as u32` |
| `String` → bytes | `s.into_bytes()` / `s.as_bytes()` |
| Bytes → `String`, strict | `String::from_utf8(v)` |
| Bytes → `String`, lossy | `String::from_utf8_lossy(&v)` |

## Common Mistakes

**Parsing without trimming.** `" 7".parse::<i32>()` is `Err`. Always
`s.trim().parse()`.

**Expecting `"3.5".parse::<i32>()` to truncate.** It errors. Parse as `f64`
then cast with `as i32`.

**`.unwrap()` on user input.** It panics on bad data. Use `unwrap_or`, `ok()`,
or `?`.

**Reaching for `parse` on hex.** It is decimal-only. Use `from_str_radix`.

**Implementing `ToString` by hand.** Implement `Display`; `to_string` comes
free.

**`char as u8` on non-ASCII.** It truncates silently — `'🦀' as u8` is 128. Check
`is_ascii()` first.

**Using `from_utf8_lossy` for stored data.** It is irreversible and changes
lengths. Use `from_utf8` and handle the error.

## Exercises

1. Write `fn sum_csv(s: &str) -> Result<i32, std::num::ParseIntError>` summing
   comma-separated integers, trimming each, failing on the first bad field.
   Test on `"1, 2, 3"` and `"1,x,3"`.
2. Write `fn to_int(s: &str) -> i32` returning 0 for any unparseable input, with
   no `match`. Use one combinator.
3. Implement `FromStr` for `struct Rgb(u8, u8, u8)` accepting `"#rrggbb"` with a
   custom error enum. Use `from_str_radix` and reject wrong lengths.
4. Add a `Display` impl to `Rgb` that prints `"#rrggbb"` with `{:02x}`, then
   assert that `"#1a2b3c".parse::<Rgb>().unwrap().to_string() == "#1a2b3c"`.
5. Take `vec![0x52, 0x75, 0xFF, 0x74]`, decode it both strictly and lossily.
   Print the `valid_up_to` offset, the lossy result, and its `len()` — explain
   why the lossy length is not 4.

---

[← 6.6 Formatting and the format! Family](06-formatting.md) · [Course Index](../README.md) · [7.1 Vec: The Workhorse →](../07-collections/01-vec.md)
