# Unsigned Integer Types 

This page documents Loom’s **unsigned, fixed-width, two’s-complement** integer types. Use these for whole-number arithmetic where values are **never negative** (e.g., sizes, counts, IDs, bitmasks, binary protocols).

---

## Quick reference

|  Type | Bits | Range (inclusive)              | Alignment\* | Default? |
| ----: | ---: | ------------------------------ | ----------- | -------- |
|  `u8` |    8 | 0 … 255                        | 1           | No       |
| `u16` |   16 | 0 … 65,535                     | 2           | No       |
| `u32` |   32 | 0 … 4,294,967,295              | 4           | **Yes**† |
| `u64` |   64 | 0 … 18,446,744,073,709,551,615 | 8           | No       |

\* Alignment is target-dependent; values shown follow typical ABIs.
† When the context requires an **unsigned** default, Loom infers `u32`. Otherwise, bare integer literals without context default to `i32`.

* **Representation:** Two’s complement bit layout (no sign bit semantics).
* **Right shifts:** **Logical** (zero-fill) for unsigned types.

---

## Literals

Loom supports decimal, binary, octal, and hexadecimal literals. Underscores are for readability.

```loom
let a: u32 = 42
let b: u32 = 0b1010_0110
let c: u32 = 0o755
let d: u32 = 0xDEAD_BEEF
let e      = 1_000_000u32   # force unsigned type
```

### Type suffixes (optional)

```loom
let small = 255u8
let wide  = 60_000u16
let big   = 0xFFFF_FFFFu64
```

If no suffix is provided, the type is inferred from context (target variable type, operation, etc.). When a literal must be **unsigned** and context is ambiguous, suffix it.

---

## Declarations & initialization

```loom
let count: u64 = 10
var total: u32 = 0      # mutable
let mask         = 0xFFu16
```

---

## Operators & semantics

* **Arithmetic:** `+  -  *  /  %`
* **Bitwise:**   `&  |  ^  ~  <<  >>`  (note: `>>` is logical on unsigned)
* **Comparison:** `==  !=  <  <=  >  >=`

```loom
let x: u16 = 300
let y: u16 = 12
let q = x / y         # 25 (integer division, trunc toward 0)
let r = x % y         # 0
let m = x << 2        # 1200
let n = ~y            # bitwise NOT
```

> **Divide by zero** is a runtime error.

---

## Overflow & underflow

By default, Loom uses **checked math in Debug** and **wrapping math in Release**. The standard library exposes explicit modes:

* `checked_add/sub/mul` → `(value, overflowed: bool)`
* `wrapping_add/sub/mul` → wrap mod 2ⁿ
* `saturating_add/sub/mul` → clamp at `MIN`/`MAX` (for unsigned, `MIN` is 0)

```loom
let (v, of) = 65_000u16.checked_add(1000u16)  # of == true
let wrap    = 255u8.wrapping_add(1u8)         # 0
let sat     = 0u8.saturating_sub(5u8)         # 0
```

**Shifts:** The shift amount is masked by the type’s bit-width (`k mod BITS`). Oversized shifts are not UB; they follow the mask rule.

---

## Type conversion

All **narrowing** conversions are **explicit** via `as`. Behavior matches signed integers:

```loom
let a: u32 = 100_000
let b: u16 = a as u16     # truncates high bits in Release; checked in Debug
let c: u64 = a as u64     # lossless widening
```

### Signed ↔ Unsigned

* `iN as uN`: reinterprets the lower `N` bits (two’s-complement) with truncation rules; checked in Debug if value is negative.
* `uN as iN`: reinterprets the lower `N` bits; checked in Debug if the unsigned value exceeds `iN::MAX`.

Prefer **checked** helpers when converting across sign:

```loom
let (ok, overflow) = u16.to_i16_checked(50_000u16)   # overflow = true
let clamped        = u32.to_i32_saturating(3_000_000_000u32)
```

### Integer ↔ Float

* Unsigned → Float: exact if the float’s mantissa can represent the value; otherwise rounds to nearest.
* Float → Unsigned: **truncates toward zero**; negative or out-of-range inputs are checked in Debug and undefined in Release unless using safe helpers.

---

## Promotions & mixed-type rules

* **Within unsigned ints:** expressions promote to the **widest unsigned** participating type.

    * `u16 + u32 → u32`, `u32 * u64 → u64`
* **Unsigned + float:** the unsigned operand promotes to the float type.
* **Signed + unsigned:** **no implicit mixing.** Cast explicitly to avoid surprises.

```loom
let a: u16 = 100
let b: u32 = 2
let c = a * b      # c: u32

let x: u32 = 5
let y: f64 = 0.5
let z = x + y      # z: f64

# Explicitly mix signed/unsigned
let s: i32 = -1
let u: u32 = (s as u32)   # reinterpret/truncate rules
```

---

## Constants & intrinsics

Each unsigned type exposes bounds and metadata:

```loom
println(u32::MIN)   # 0
println(u32::MAX)   # 4294967295
println(u32::BITS)  # 32
println(u64::BYTES) # 8
```

Bit-level helpers:

* `count_ones()`, `count_zeros()`
* `leading_zeros()`, `trailing_zeros()`
* `rotate_left(k)`, `rotate_right(k)`
* `is_power_of_two()`
* `next_power_of_two_checked()` / `next_power_of_two_saturating()`

```loom
let m: u32 = 0xFF00
println(m.count_ones())  # 8
```

---

## Formatting & parsing

```loom
let v: u32 = 48879

print(v)                          # 48879
printf("hex=%x, bin=%b\n", v, v)  # hex=beef, bin=1011111011101111
printf("padded=%08u\n", v)        # padded=00048879
```

Parsing (throws on error unless using `*_opt` variants):

```loom
let n: u32 = u32.parse("12345")
let m: u32 = u32.parse_radix("DEAD", 16)
let o_opt  = u32.parse_opt("not_a_number")  # -> Option<u32>
```

---

## Interop & memory

* **ABI:** Fixed widths with `sizeof(u32) == 4`, `alignof(u64) == 8`.
* **Byte order:** Use `to_le_bytes() / to_be_bytes()` and `from_le_bytes()/from_be_bytes()` to control endianness.

```loom
let bytes: [u8; 4] = 0x12345678u32.to_be_bytes()
let val: u32       = u32.from_be_bytes(bytes)
```

Unsigned types are ideal for **bitfields**, **protocol fields**, and **FFI** with C/C++ `uint*_t`.

---

## Common patterns & guidance

* Prefer **`u32`** for general non-negative counts, sizes, and IDs.
* Use **`u64`** for large counters (file sizes, ticks, nanoseconds).
* Use **`u8`** for raw bytes and buffers.
* Choose **`unsigned`** types for **bitmask/flag** operations and binary I/O.
* When dealing with untrusted inputs, use **checked** or **saturating** ops to avoid wraparounds.

---

## Examples

### Bitmask flags

```loom
const READ:  u32 = 1u32 << 0
const WRITE: u32 = 1u32 << 1
const EXEC:  u32 = 1u32 << 2

let perms = READ | WRITE
if (perms & EXEC) != 0u32 {
    println("exec enabled")
}
```

### Packing ARGB into `u32`

```loom
# aa rr gg bb → 0xAARRGGBB
pub func pack_argb(a: u8, r: u8, g: u8, b: u8): u32 {
    ret (a as u32) << 24 |
        (r as u32) << 16 |
        (g as u32) << 8  |
        (b as u32)
}
```

### Safe accumulation without wrap

```loom
pub func sum_checked(nums: []u32): (u32, bool) {
    var total: u32 = 0
    var overflowed = false
    for n in nums {
        let (t, of) = total.checked_add(n)
        total = t
        if of { overflowed = true }
    }
    ret (total, overflowed)
}
```

### Logical right shift behavior

```loom
let x: u8 = 0b1000_0000u8
let ar = (x as i8) >> 1      # arithmetic shift on signed → 0b1100_0000 (impl-defined for i8)
let lr = x >> 1              # logical shift on unsigned → 0b0100_0000
```

---

## FAQs

**Q: Why unsigned at all if signed exists?**
A: Unsigned types model non-negative domains, map directly onto protocol/FFI fields, and make bit-twiddling clearer.

**Q: Is `%` always non-negative?**
A: Yes. For unsigned types, remainder is in `0..=uN::MAX`.

**Q: What happens on `0u32 - 1u32`?**
A: Underflow: checked error in Debug; wraps to `u32::MAX` in Release unless you use `checked_sub`/`saturating_sub`.

**Q: Can I mix `u32` and `i32` in one expression?**
A: Not implicitly. Cast explicitly to the intended domain to avoid accidental wrap.

---

## See also

* Signed integers: `i8`, `i16`, `i32`, `i64`
* Floating-point types: `f16`, `f32`, `f64`
* Byte/bit utilities in the standard library
