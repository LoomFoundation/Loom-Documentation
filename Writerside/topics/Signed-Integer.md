# Signed Integer Types

This page documents Loom’s **signed, fixed-width, two’s-complement** integer types. Use these when you need exact whole-number arithmetic with well-defined overflow and bitwise behavior.

---

## Quick reference

|  Type | Bits | Range (inclusive)                                      | Alignment\* | Default? |
|------:|-----:|--------------------------------------------------------|-------------|----------|
|  `i8` |    8 | −128 … 127                                             | 1           | No       |
| `i16` |   16 | −32,768 … 32,767                                       | 2           | No       |
| `i32` |   32 | −2,147,483,648 … 2,147,483,647                         | 4           | **Yes**  |
| `i64` |   64 | −9,223,372,036,854,775,808 … 9,223,372,036,854,775,807 | 8           | No       |

> Alignment is target-dependent but listed here by typical ABI convention.

* **Representation:** Two’s complement.
* **Endianness:** Platform-dependent; use std helpers for explicit byte order.

---

## Literals

Loom supports decimal, binary, octal, and hexadecimal integer literals. Underscores are allowed for readability and don’t affect the value.

```loom
let a: i32 = 42              # decimal
let b: i32 = 0b1010_0110     # binary
let c: i32 = 0o755           # octal
let d: i32 = 0xDEAD_BEEF     # hexadecimal
let e      = 1_000_000       # type-inferred (i32 by default)
```

### Type suffixes (optional)

You may suffix a literal to force its type:

```loom
let small = 127i8
let wide  = 9000i16
let big   = 0xFF_FF_FF_FFi64
```

If no suffix is provided and no contextual type is required, **`i32`** is inferred, unless the literal is too large to fit in **`i32`**.

---

## Declarations & initialization

```loom
let count: i64 = 10
let delta      = -3         # inferred i32
var total: i32 = 0          # mutable variable (use `var`), signed 32-bit
```

---

## Arithmetic & operators

All standard integer operators are available:

* **Arithmetic:** `+  -  *  /  %`
* **Bitwise:** `&  |  ^  ~  <<  >>`
* **Comparison:** `==  !=  <  <=  >  >=`

```loom
let x: i16 = 300
let y: i16 = 12
let q = x / y       # 25 (integer division)
let r = x % y       # 0  (remainder)
let m = x << 2      # 1200
let n = ~y          # bitwise NOT
```

> **Note:** Division by zero returns a _NaN_.
---

## Overflow behavior

Loom provides **checked** behavior in Debug and **wrapping** behavior in Release by default, plus explicit helpers for all modes:

* `checked_add/sub/mul` → returns `(value, overflowed: bool)`
* `wrapping_add/sub/mul` → wraps mod 2ⁿ
* `saturating_add/sub/mul` → clamps at `MIN`/`MAX`

```loom
let (v, of) = 32_000i16.checkedAdd(10_000i16)
if of {
    println("overflow detected")
}

let wrap = 127i8.wrappingAdd(1i8)    # -128
let sat  = (-120i8).saturatingSub(20i8) # -128
```

---

## Type conversion

Conversions between integer widths are **explicit**. Use the `as` cast operator.

```loom
let a: i32 = 1_000
let b: i16 = a as i16     # truncates on overflow in Release; checked in Debug
let c: i64 = a as i64     # always lossless here
```

### Between integers and floats

* Integer → Float: exact for all values that the float can represent precisely; otherwise rounded to nearest.
* Float → Integer: **truncates toward zero**; out-of-range is checked in Debug and undefined in Release unless using safe helpers.

```loom
let f: f32 = (123i32) as f32
let i: i32 = (3.9f32) as i32   # 3
```

---

## Promotions & mixed-type rules

* **Within signed ints:** expressions promote to the **widest** participating signed type.

    * `i16 + i32 → i32`, `i32 * i64 → i64`
* **Int + float:** integers are promoted to the float type of the expression.
* **Signed + unsigned:** disallowed implicitly (requires an explicit cast).

Examples:

```loom
let a: i16 = 100
let b: i32 = 2
let c = a * b      # c: i32

let x: i32 = 5
let y: f64 = 0.5
let z = x + y      # z: f64
```

---

## Constants & intrinsics

Each signed type exposes min/max and bit-width metadata:

```loom
println(i32::MIN)  # -2147483648
println(i32::MAX)  #  2147483647
println(i32::BITS) # 32
println(i64::BYTES)# 8
```

Bit-twiddling intrinsics:

* `countOnes()`, `countZeros()`
* `leadingZeros()`, `trailingZeros()`
* `rotateLeft(k)`, `rotateRight(k)`
* `abs()` (note: `abs(i64::MIN)` overflows; see `absChecked()`)

```loom
let mask = 0xFF00i32
println(mask.countOnes())  # 8
```

---

## Formatting & parsing

Use Loom’s built-ins for output; formatting supports bases.

```loom
let v: i32 = 48879

print(v)                        # 48879
printf("hex=%x, bin=%b\n", v, v)   # hex=beef, bin=1011111011101111
printf("padded=%08d\n", v)         # padded=00048879
```

Parsing helpers (throw on error unless using `*_opt` variants):

```loom
let n: i32 = i32.parse("12345")
let m: i32 = i32.parseRadix("DEAD", 16)
let o_opt = i32.parseOpt("not_a_number")   # -> Option<i32>
```

---

## Interop & memory

* **ABI:** Two’s-complement, fixed width. Use `sizeof(i32) == 4`, `alignof(i64) == 8`.
* **Byte order:** Use `toBytes()` and the inverse `fromBytes()`.

```loom
let bytes: []byte = 0x12345678i32.toBytes()
let val: i32 = i32.fromBytes(bytes)
```

---

## Common patterns & guidance

* Prefer **`i32`** unless you have a strong reason (protocol layout, memory, range).
* Use **`i64`** for counters/timestamps that can exceed 32-bit limits.
* Use **checked/saturating** operations when input bounds are external/untrusted.
* When crossing FFI boundaries, **match the exact width** expected by the other side.

---

## Examples

### Accumulating with safe math

```loom
pub func sum_checked(nums: []i32): (i32, bool) {
    var total: i32 = 0
    var overflowed = false
    for n in nums {
        let (t, of) = total.checked_add(n)
        total = t
        if of { overflowed = true }
    }
    ret (total, overflowed)
}
```

### Bitmask flags

```loom
const READ:  i32 = 1 << 0
const WRITE: i32 = 1 << 1
const EXEC:  i32 = 1 << 2

let perms = READ | WRITE
if (perms & EXEC) != 0 {
    println("exec enabled")
}
```

### Clamping (saturating) arithmetic

```loom
# Add user input to a budget without ever exceeding i32::MAX
var budget: i32 = 2_000_000_000
budget = budget.saturatingAdd(user_delta)
```

---

## FAQs

**Q: Why two’s complement?**
A: It’s the de-facto hardware representation; shifts and wraps are well-defined and fast.

**Q: Does `i8` promote to `i32` in expressions?**
A: Yes—mixed signed int expressions promote to the widest signed type present.

**Q: How do I avoid wrap in Release?**
A: Use `checked*` (detect), `saturating*` (clamp), or compile with overflow checks enabled for your target profile.

---

## See also

* Unsigned integers: `u8`, `u16`, `u32`, `u64`
* Floating-point types: `f16`, `f32`, `f64`
* Numeric conversions and formatting helpers in the standard library
