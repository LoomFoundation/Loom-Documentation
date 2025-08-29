# Strings

`string` is Loom’s **mutable, first-class, UTF-8** text type. It stores valid Unicode by default, supports efficient in-place edits, and offers character-aware operations (so you don’t accidentally split code points).

---

## Quick reference

* **Encoding:** UTF-8 (always)
* **Mutability:** **Mutable** (edit in place)
* **Indexing:** Prefer character-aware APIs; byte indexing is allowed with care
* **Lengths:** `bytes_len()` vs `chars_len()`
* **Default literal type:** `string`

---

## Literals

### Basic strings

```loom
let a: string = "Hello, Loom!"
let b          = "Line 1\nLine 2\tTabbed"
let c          = "Snowman: \u2603"        # Unicode escape
let d          = "Rocket: \U0001F680"     # 🚀
```

### Escapes

* `\n \r \t \\ \" \'`
* `\xNN` (single byte, hex)
* `\uNNNN` (BMP, 16-bit hex)
* `\UNNNNNNNN` (full 21-bit hex)

### Raw strings (no escaping, multi-line OK)

```loom
let path = r"""C:\Users\loom\docs\readme.txt"""
let json = r'''{"quote": "He said: "ok"" }'''
```

* Use `r""" ... """` **or** `r''' ... '''` (pick the other delimiter if your text contains `"""` or `'''`).
* Contents are taken verbatim (no escape processing).

---

## Length, indexing, and slicing (Unicode-aware)

UTF-8 characters vary in byte length. Prefer character-aware helpers unless you explicitly need bytes.

```loom
let s = "café"                 # bytes_len=5, chars_len=4

s.bytes_len()                  # 5 (O(1))
s.chars_len()                  # 4 (O(n))

let c0: char = s.char_at(0)    # 'c'
let b0: u8   = s.byte_at(0)    # 0x63

let head   = s.slice_chars(0, 3)   # "caf"          (new string)
let suffix = s.slice_bytes(3, 5)   # "é"            (must fall on boundaries)
```

* `char_at(i)` indexes by **character** (Unicode scalar).
* `byte_at(i)` reads the i-th **byte**.
* `slice_chars(start, end)` / `slice_bytes(start, end)` create new strings.
* Byte-slicing that lands **inside** a code point raises a runtime error.

---

## In-place editing APIs

Because `string` is mutable, you can modify text without allocating new strings:

```loom
var s: string = "Hello"
s.push_char('!')                      # "Hello!"
s.append(" world")                    # "Hello! world"
s.insert_chars(0, 1, "Hey, ")         # "Hey, Hello! world"   (pos by char)
s.replace_in_place("world", "Loom")   # "Hey, Hello! Loom"
s.remove_range_chars(5, 7)            # remove chars [5,7) by char index
s.trim_in_place()                     # trims ASCII/Unicode whitespace in place
```

Selected mutators:

* `append(str)`, `push_char(ch)`
* `insert_chars(pos, count, str)` / `insert_bytes(pos, bytes)`
* `remove_range_chars(start, end)` / `remove_range_bytes(start, end)`
* `clear()`, `reserve(cap)`, `shrink_to_fit()`
* `to_upper_in_place()`, `to_lower_in_place()`, `replace_in_place(from, to)`
* `trim_in_place()`, `trim_start_in_place()`, `trim_end_in_place()`

> Pure (non-mutating) counterparts also exist, e.g., `to_upper()` returns a new string.

---

## Concatenation & formatting

```loom
var s = "Hello"
s += ", "                   # in-place
s += "Loom"
s = s + "!"                 # builds a new string; use `+=` to mutate

printf("name=%s id=%u\n", name, id)

let msg  = string.format("({0}, {1})", x, y)                # positional
let msg2 = string.format("{x} × {y} = {p}", {x:6, y:7, p:42}) # named
```

---

## Inspection & search

```loom
let t = "Hello, 世界"

t.is_empty()                 # bool
t.starts_with("Hell")        # bool
t.ends_with("界")            # bool
t.contains("lo, ")           # bool

t.find("lo")                 # Option<usize> (byte offset)
t.rfind("l")                 # Option<usize>
t.find_char('界')            # Option<usize> (char index)
```

Iteration:

```loom
for ch in t.chars() { print(ch) }    # Unicode scalars
for b  in t.bytes()  { print(b) }    # raw UTF-8
```

---

## Conversions

### Bytes ↔ string

```loom
let bytes: []u8 = t.to_utf8()                         # copy out UTF-8
let ok:  string = string.from_utf8(bytes)?            # validate (throws on invalid)
let los: string = string.from_utf8_lossy(bytes)       # U+FFFD for invalid
```

### Characters / arrays

```loom
let chars: []char = t.to_chars()
let u: string     = string.from_chars(chars)
```

### Numbers ↔ string

```loom
let n: i32 = i32.parse("123")
let o: Option<i32> = i32.parse_opt("x")               # None

let hex = string.format("0x{0:X}", 48879)             # "0xBEEF"
```

---

## Equality, ordering, hashing

* `==` / `!=` compare by **value** (byte sequence of valid UTF-8).
* `<, <=, >, >=` use lexicographic byte order.
* For locale-aware collation, use a collation library (future std extension).
* `hash()` is stable within a process; not guaranteed across versions/platforms.

---

## I/O and encodings

* File APIs read/write `string` as **UTF-8** by default.
* Other encodings via codecs:

```loom
let sjis = codecs.encode("こんにちは", "shift_jis")   # []u8
let back = codecs.decode(sjis, "shift_jis")?         # string
```

---

## Performance tips

* Use **in-place** APIs (`+=`, `append`, `insert_*`, `replace_in_place`) to avoid temporary allocations.
* Call `reserve(cap)` before large concatenations to reduce re-allocations.
* `bytes_len()` is O(1); `chars_len()` may be O(n).
* When comparing human text in security-sensitive contexts, consider **case-folding** and **normalization** (`nfc()`, `nfd()`) to avoid deceptive mismatches.

---

## Examples

### In-place sanitization

```loom
pub func sanitize_line(s: string): string {
    s.replace_in_place("\r\n", "\n")
    s.replace_in_place("\r", "\n")
    s.trim_in_place()
    ret s
}
```

### Safe prefix (by characters)

```loom
pub func take_prefix(s: string, k: usize): string {
    let n = s.chars_len()
    if k >= n { ret s }
    ret s.slice_chars(0, k)
}
```

### Join with separator (pre-reserve for speed)

```loom
pub func join(parts: []string, sep: string): string {
    if parts.len() == 0 { ret "" }
    var out: string = ""
    # Reserve approximate capacity
    var cap: usize = 0
    for p in parts { cap += p.bytes_len() }
    cap += sep.bytes_len() * (parts.len() - 1)
    out.reserve(cap)

    for i in 0..parts.len() {
        if i > 0 { out += sep }
        out += parts[i]
    }
    ret out
}
```

### Case-insensitive (ASCII) compare without allocating

```loom
pub func eq_ignore_ascii_case(a: string, b: string): bool {
    if a.bytes_len() != b.bytes_len() { ret false }
    for i in 0..a.bytes_len() {
        let x = a.byte_at(i)
        let y = b.byte_at(i)
        let xl = if x >= 'A' && x <= 'Z' { x + 32u8 } else { x }
        let yl = if y >= 'A' && y <= 'Z' { y + 32u8 } else { y }
        if xl != yl { ret false }
    }
    ret true
}
```

---

## FAQs

**Q: Do strings guarantee valid UTF-8?**
A: Yes for constructors that validate; `from_utf8_lossy` permits replacement of invalid bytes. Low-level APIs may create unchecked strings for performance—use with care.

**Q: Are strings null-terminated?**
A: No. Strings store a length and bytes; embedded `\0` is allowed.

**Q: Is character indexing O(1)?**
A: Not necessarily. Characters are variable width in UTF-8; use iterators or cache offsets if you need repeated access.

**Q: How do raw strings handle embedded quotes?**
A: Use the **other** triple-quote flavor (`r"""..."""` vs `r'''...'''`) so you don’t need escaping.

---

## See also

* `char` (Unicode scalar values)
* Arrays & slices (`[]u8`, `[]char`)
* Formatting & I/O (`print`, `printf`, `string.format`)
