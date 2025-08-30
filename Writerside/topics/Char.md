# Characters

`char` represents a single **Unicode scalar value**: any code point in `U+0000 … U+10FFFF` **excluding** the surrogate range `U+D800 … U+DFFF`. It is not a “byte” and not necessarily what a user perceives as a single letter (see **grapheme clusters** note below).

* **Storage:** 32-bit value (fixed width).
* **Encoding:** UTF-8 when serialized; 1–4 bytes.
* **Domain:** All Unicode scalar values; surrogates are invalid. (Noncharacters are allowed by Unicode but typically discouraged in text meant for interchange.)

---

## Quick reference

* Construct with literals (`'A'`, `'\n'`, `'\u2603'`, `'\U0001F680'`).
* Convert to bytes: `c.to_utf8()` or `c.encode_utf8(buf)`.
* Inspect & classify: `is_ascii()`, `is_alphabetic()`, `is_numeric()`, `is_whitespace()`, …
* Case convert: `to_upper()`, `to_lower()`, `case_fold()` (may expand to multiple chars).
* Order & compare by **code point**; locale-sensitive collation requires higher-level APIs.

---

## Literals & escapes

```loom
let a: char = 'A'
let b: char = '\n'            # newline
let c: char = '\t'            # tab
let d: char = '\''            # single quote
let e: char = '\\'            # backslash
let s: char = '\u2603'        # U+2603 SNOWMAN (☃)
let r: char = '\U0001F680'    # U+1F680 ROCKET (🚀)
let z: char = '\x00'          # single byte 0x00 (NUL)
```

Supported escapes:

* Control/whitespace: `\n \r \t \0`
* Quote/backslash: `\' \" \\`
* Byte: `\xNN` (two hex digits; must map to a valid scalar)
* Unicode (BMP): `\uNNNN` (four hex)
* Unicode (full): `\UNNNNNNNN` (up to eight hex, ≤ `10FFFF`, non-surrogate)

Invalid escapes or surrogate code points raise a compile-time error.

---

## Construction & validation

```loom
let ok:  char = char.from_u32(0x03A9)?       # 'Ω' (throws if invalid)
let any: char = char.from_u32_lossy(0xD800)  # returns U+FFFD replacement
let raw: char = char.from_u32_unchecked(0x41) # unsafe: no validation
```

* `from_u32(u)`           → `char?` (throws on surrogate/out-of-range)
* `from_u32_lossy(u)`     → returns `U+FFFD` for invalid input
* `from_u32_unchecked(u)` → **unsafe**: caller guarantees validity

```loom
let u: u32 = 'Ω'.to_u32()    # 0x03A9
```

---

## Encoding & decoding UTF-8

```loom
let bytes: []u8 = 'é'.to_utf8()            # [0xC3, 0xA9]
let mut buf: [u8; 4]
let n: usize = '🚀'.encode_utf8(&mut buf)  # writes into buf; n == 4

let (ch, used) = char.decode_utf8_prefix(bytes)?  # decode first scalar
```

* `to_utf8()` allocates a tiny slice (1–4 bytes).
* `encode_utf8(&mut [u8;4]) -> usize` writes without allocating.
* `decode_utf8_prefix(bs)` decodes the first scalar from a byte slice.

---

## Classification & queries

```loom
let c = 'Ƶ'

c.is_ascii()           # false
c.is_alphabetic()      # true (Unicode Alpha property)
c.is_alphanumeric()    # Alpha or Nd
c.is_numeric()         # true only for decimal digits (General Category Nd)
c.is_whitespace()      # Unicode whitespace (space, NBSP, tabs, CR/LF, etc.)
c.is_control()         # Cc or Cf
c.is_uppercase()       # Unicode Uppercase
c.is_lowercase()       # Unicode Lowercase
c.category()           # → UnicodeCategory enum (e.g., LetterUppercase, MarkNonspacing, ...)
c.width_display()      # 0/1/2 for typical monospace terminals (combining = 0)
c.is_combining_mark()  # true for Mn/Mc
```

ASCII-focused helpers:

```loom
c.is_ascii_alphabetic()
c.is_ascii_alphanumeric()
c.is_ascii_digit()
c.to_ascii_lowercase_char()  # → char (no expansion)
c.to_ascii_uppercase_char()
```

---

## Numeric value & digit parsing

```loom
'7'.to_digit(10)   # Some(7)
'a'.to_digit(16)   # Some(10)
'Ⅷ'.to_digit(10)   # None (Roman numerals are not Nd)
```

* `to_digit(radix: u32) -> Option<u32>` supports radix `2…36` and uses Unicode Nd for 0–9 and ASCII letters for 10–35.

---

## Case conversion

Some mappings expand (e.g., `'ß'` → `"SS"`). Loom offers string and single-char variants:

```loom
'ß'.to_upper()          # "SS"        (string)
'ß'.to_upper_char()     # None        (no single-char upper)
'İ'.to_lower()          # "i̇"         (note combining dot)
'a'.to_upper_char()     # Some('A')
```

APIs:

* `to_upper() / to_lower() / case_fold()` → `string` (full Unicode)
* `to_upper_char() / to_lower_char()` → `Option<char>` (only when 1:1)

---

## Comparison, ordering, hashing

* `==` / `!=` compare code points.
* `<, <=, >, >=` order by code point value.
* `hash()` uses code point value.

For locale-aware sorting/collation, use string-level collation.

---

## Interaction with `string`

Iterating over a `string` yields `char` values:

```loom
for ch in "Hello, 世界".chars() {
    print(ch)
}
```

Counting characters:

```loom
let n = "café".chars_len()  # 4
```

**Important:** A `char` is a scalar value, not a **grapheme cluster**. Many user-visible “characters” are multiple scalars (e.g., `'e'` + COMBINING ACUTE, emoji with skin tones/ZWJ sequences). For cursoring, deletion, and UI selection, operate on grapheme clusters (via `graphemes()` in the text/icu add-on), not raw `char`s.

---

## Performance notes

* `char` operations are O(1).
* Converting `char` ⇄ UTF-8 uses small fixed code paths (1–4 bytes).
* Frequent case conversions that expand should be done at the **string** level to avoid repeated allocation.

---

## Examples

### Filter only ASCII digits

```loom
pub func only_ascii_digits(s: string): string {
    var out: string = ""
    out.reserve(s.bytes_len())
    for ch in s.chars() {
        if ch.is_ascii_digit() { out += ch.to_string() }
    }
    ret out
}
```

### Titlecase first letter of each word (simple ASCII)

```loom
pub func title_ascii_words(s: string): string {
    var out: string = ""
    var new_word = true
    for ch in s.chars() {
        if ch.is_ascii_alphanumeric() {
            out += (new_word ? ch.to_ascii_uppercase_char() : ch).to_string()
            new_word = false
        } else {
            out += ch.to_string()
            new_word = true
        }
    }
    ret out
}
```

### Count combining marks

```loom
pub func count_combining(s: string): usize {
    var n: usize = 0
    for ch in s.chars() {
        if ch.is_combining_mark() { n += 1 }
    }
    ret n
}
```

### Encode without allocation

```loom
pub func write_char_utf8(dst: &mut []u8, ch: char): usize {
    var tmp: [u8; 4]
    let n = ch.encode_utf8(&mut tmp)
    dst.write(&tmp[0..n])
    ret n
}
```

---

## FAQs

**Q: Can a `char` be a surrogate value?**
A: No. Surrogates (`U+D800 … U+DFFF`) are not Unicode scalar values and are rejected by safe constructors.

**Q: Why does uppercasing a single `char` sometimes return a `string`?**
A: Unicode rules allow expansions (e.g., `'ß'` → `"SS"`). Use `to_upper_char()` if you need a single-char mapping only when it exists.

**Q: How many bytes does a `char` take in UTF-8?**
A: 1 to 4 bytes. Use `encode_utf8` to write into a 4-byte buffer or `utf8_len()` to query the length.

**Q: Is `char` the same as a user-visible “character”?**
A: Not necessarily. Many user-visible characters are **grapheme clusters** composed of multiple `char`s. Use grapheme-aware APIs when working with UI text.

---

## See also

* `string` (mutable UTF-8 strings)
* Byte slices `[]u8`
* Unicode/ICU utilities (`graphemes()`, normalization, collation)
