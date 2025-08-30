# Bytes

`byte` is Loom’s **owned, growable, mutable** buffer of raw 8-bit values. Use it for binary I/O, network protocols, file formats, cryptography, and any data that is **not** guaranteed to be UTF-8 text.

* **A single byte:** `u8`
* **A borrowed view:** `[]u8` (slice)
* **An owned buffer:** `byte`

> If you need human text, use `string` (UTF-8). Convert between `string` and `byte` with the UTF-8 helpers below.

---

## Quick reference

* **Mutability:** mutable (in-place edits)
* **Growth:** dynamic; supports `reserve`, `shrink_to_fit`
* **Indexing:** by byte (bounds-checked)
* **Slicing:** cheap views (`[]u8`) or copied sub-buffers (`byte.slice`)
* **Equality:** value-based byte comparison
* **Ordering:** lexicographic (byte order)
* **Zeroization:** `zeroize_in_place()` for sensitive data

---

## Creating `byte`

```loom
let a: byte = byte.new()
let b: byte = byte.with_capacity(1024)
let c: byte = byte.from_slice([0xDE, 0xAD, 0xBE, 0xEF]u8)

# From hex / base64 text
let h: byte = byte.from_hex("deadbeef")?         # spaces/underscores allowed
let b64: byte = byte.from_base64("SGVsbG8=")?

# From a string's UTF-8
let s: string = "Hello, 世界"
let utf8: byte = s.to_utf8()                      # copies UTF-8 byte
```

### Byte literals

```loom
let raw: byte  = b"Hello\x00"         # byte string literal (escapes enabled)
let arr: []u8   = [0x01, 0x02, 0x03]u8 # array literal (typed as []u8)
let own: byte  = byte.from_slice(arr)
```

Supported escapes in `b"..."`: `\n \r \t \\ \" \' \xNN`

---

## Core API

```loom
var buf: byte = byte.with_capacity(16)

buf.len()                # usize
buf.is_empty()           # bool
buf.capacity()           # usize
buf.reserve(100)         # ensure additional space
buf.shrink_to_fit()

# Index & mutate
let x: u8 = buf[0]                    # get (panics if OOB)
buf[1] = 0xFF                         # set (panics if OOB)
let ok = buf.get(2)                   # Option<u8>
buf.set(3, 0x7F)?                     # throws if OOB

# Append / insert / remove
buf.push(0xAA)
let last = buf.pop()?                 # Option<u8>
buf.extend([0x01, 0x02]u8)           # from slice
buf.insert(0, 0x00)?
buf.remove(1)?                        # remove at index, returns byte

# Ranges (byte indices)
buf.replace_range(2, 5, [0x99]u8)?    # [start,end) with slice
buf.clear()
buf.fill(0x00)                        # set all byte to value
buf.zeroize_in_place()                # securely wipe contents

# Views & copies
let view: []u8   = buf.view(2, 6)?    # borrow byte [2,6) (no copy)
let sub:  byte  = buf.slice(2, 6)?   # copy byte [2,6) (owned)
let all:  []u8   = buf.as_slice()
```

Iteration:

```loom
for b in buf.iter() { print(b) }      # by value
for i in 0..buf.len() { buf[i] ^= 0xFF }  # in-place edit
```

---

## Search & split

```loom
buf.contains([0xDE, 0xAD]u8)           # bool
buf.find([0x00]u8)                     # Option<usize> (first index)
buf.rfind([0x00]u8)                    # Option<usize>

for chunk in buf.chunks(16) { /* []u8 view */ }
for win   in buf.windows(4) { /* []u8 view of rolling 4 */ }

let parts: []byte = buf.split([0x00]u8)       # copies parts
let views: [][]u8 = buf.split_view([0x00]u8)   # borrowed views
```

---

## Encoding & conversions

### UTF-8 (byte ⇄ string)

```loom
# byte → string
let text_ok: string = string.from_utf8(buf.as_slice())?      # validate
let text_lo: string = string.from_utf8_lossy(buf.as_slice()) # U+FFFD for bad

# string → byte
let data: byte = "hi".to_utf8()
```

### Hex / Base64

```loom
let hx: string = buf.to_hex(lower=true, spaced_every=2)   # "de ad be ef"
let b6: string = buf.to_base64(line_width=76)
```

---

## Numbers & endianness

Use per-type helpers to pack/unpack:

```loom
# Integers to byte
let b4: [u8;4] = 0x12345678u32.to_be_byte()
let b8: [u8;8] = (-1i64).to_le_byte()

# byte to integers (expects exact-length slices)
let n: u32 = u32.from_be_byte(b4)
let m: i64 = i64.from_le_byte(b8)

# Read from a buffer view
let v: []u8 = buf.view(0, 4)?
let x: u32  = u32.from_le_byte_exact(v)?   # throws on wrong length

# Floating point (IEEE-754)
let fb: [u8;8] = double.to_be_byte(3.14159)
let pi: double = double.from_be_byte(fb)
```

Stream-style helpers:

```loom
let rdr = ByteReader.from_slice(buf.as_slice())
let wtr = ByteWriter::new()

let u = rdr.read_u16_le()?
let f = rdr.read_f32_be()?
wtr.write_u32_be(0xDEADBEEF)
wtr.write_all([0x00, 0x01]u8)
let out: byte = wtr.finish()
```

---

## Safety notes

* Indexing and range methods are bounds-checked. Use `get`/`view` if you prefer options/errors over panics.
* Keep track of **endianness** when mapping to numeric types.
* For secrets (keys, tokens), prefer `zeroize_in_place()` and **constant-time** compare:

```loom
let eq = byte.ct_eq(secret_a.as_slice(), secret_b.as_slice())  # bool, constant-time
```

---

## Equality, ordering, hashing

* `==` / `!=` compare contents byte-for-byte.
* `<, <=, >, >=` compare lexicographically.
* `hash()` uses byte sequence; stable within a process but not guaranteed across versions/platforms.

---

## I/O

```loom
# Read entire file into byte
let data: byte = fs.read_all_byte("image.png")?

# Write byte
fs.write_all("copy.png", data.as_slice())?

# Socket I/O
sock.send(buf.as_slice())
let n = sock.recv_into(&mut buf, 4096)    # may grow as needed
```

---

## Performance tips

* **Prefer views (`[]u8`)** to avoid copies; use `slice` only when you need ownership.
* Call **`reserve`** before large appends to reduce reallocations.
* Use `extend` / `copy_from_slice` to leverage efficient memory copies.
* Avoid converting byte ↔ strings unless you need text; decoding/encoding has cost.

---

## Examples

### Build a length-prefixed packet

```loom
pub func make_packet(kind: u8, payload: []u8): byte {
    var out = byte.with_capacity(1 + 2 + payload.len())
    out.push(kind)
    let len: u16 = payload.len() as u16
    out.extend(len.to_be_byte())         # network byte order
    out.extend(payload)
    ret out
}
```

### Parse a BMP header (partial)

```loom
pub struct BmpHeader {
    size: u32,
    data_offset: u32,
}

pub func parse_bmp_header(src: []u8): BmpHeader? {
    if src.len() < 14 { throw "short file" }
    if src[0] != 'B' as u8 || src[1] != 'M' as u8 { throw "not BMP" }
    let size       = u32.from_le_byte_exact(src[2..6])?
    let data_off   = u32.from_le_byte_exact(src[10..14])?
    ret BmpHeader { size: size, data_offset: data_off }
}
```

### Hex dump utility

```loom
pub func hexdump(data: []u8): string {
    var out: string = ""
    for i in 0..data.len() {
        if i % 16 == 0 { out += string.format("\n{0:08X}: ", i) }
        out += string.format("{0:02X} ", data[i])
    }
    ret out
}
```

### XOR in place

```loom
pub func xor_in_place(dst: &mut byte, key: []u8) {
    for i in 0..dst.len() {
        dst[i] ^= key[i % key.len()]
    }
}
```

---

## FAQs

**Q: When should I use `byte` vs `[]u8`?**
A: Use `byte` when you need ownership and growth. Use `[]u8` for **borrowing**/viewing existing memory without copying.

**Q: Are `byte` null-terminated?**
A: No. They’re length-delimited; embedded `0x00` is fine.

**Q: Can I store invalid UTF-8?**
A: Yes. `byte` has no text semantics. Convert to `string` with `from_utf8` (validates) or `from_utf8_lossy` (replaces invalid sequences).

**Q: Do range methods use byte indices?**
A: Yes. All indexing/slicing on `byte` is by **byte**.

---

## See also

* `u8` (single byte), `[]u8` (byte slice)
* `string` (UTF-8 text), `char` (Unicode scalar)
* Integer/float `to_*_byte` and `from_*_byte` helpers for endianness
* File & socket I/O utilities
