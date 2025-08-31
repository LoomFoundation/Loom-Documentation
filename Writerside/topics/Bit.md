# Bits

`bit` is a **1-bit signedness-free integer** with two values: **0** and **1**. It’s numeric (not `bool`) and participates in bitwise/arithmetic promotion rules. Use it for parity, bitfields, masks, and packed data.

---

## Quick reference

| Property        | Value                                                                                    |
| --------------- | ---------------------------------------------------------------------------------------- |
| Values          | `0`, `1`                                                                                 |
| Storage         | 1 bit (stand-alone variables typically occupy 1 byte; arrays/bit-collections are packed) |
| Default literal | Use `0`/`1` with type context or cast (`as bit`)                                         |
| Truthiness      | **Not** implicitly boolean (explicit cast to/from `bool`)                                |
| Promotions      | Promotes to the **widest integer** in mixed expressions                                  |

---

## Declaring & initializing

```loom
let a: bit = 1
var f: bit = 0
f = 1
f = (3 & 1) as bit          # explicit cast from int
```

You can also annotate via context:

```loom
# In a bit array/map API that expects bit:
bits.set(42, 1)             # 1 inferred as bit by parameter type
```

---

## Operators

`bit` supports the integer operator set; results follow normal integer promotion:

* **Bitwise:** `~  &  |  ^`
* **Arithmetic:** `+  -  *` (on `bit` these are equivalent to boolean algebra; in mixed types, result is promoted)
* **Comparison:** `==  !=  <  <=  >  >=`

```loom
let x: bit = 1
let y: bit = 0
let z1 = x & y               # 0  (type: bit)
let z2 = x ^ y               # 1  (type: bit)
let w  = x + 5               # 6  (type: i32, via promotion)
```

> Division (`/`) and remainder (`%`) are valid only after promotion to a wider integer.

---

## Conversions

### With integers

* `bit → iN/uN`: `0` or `1` (lossless)
* `iN/uN → bit`: `(value & 1)`; **use explicit cast**

```loom
let b: bit = (n as u32 & 1u32) as bit
let i: i32 = b as i32
```

### With `bool`

No implicit interchange (to prevent accidental logic–arithmetic mixing):

```loom
let t: bool = (b == 1)               # explicit check
let b2: bit = if t { 1 } else { 0 }  # explicit mapping
# or use helpers if available:
# b.to_bool(), bit.from_bool(t)
```

---

## Bit collections

Loom provides packed bit collections for efficient storage and random access.

### `BitVec` (growable)

* **Create:** `BitVec::new()`, `with_len(n, fill: bit)`, `from_bytes([]u8)`
* **Core ops:** `get(i) -> bit`, `set(i, bit)`, `flip(i)`, `push(bit)`, `len()`
* **Bulk:** `fill(bit)`, `clear()`, `count_ones()`, `any()`, `all()`
* **Bitwise:** `and(&rhs)`, `or(&rhs)`, `xor(&rhs)`, `not()`
* **Slicing:** `view(start, end) -> BitSlice` (borrowed, packed)

```loom
var bv = BitVec::with_len(10, 0)
bv.set(3, 1)
bv.flip(3)
let p: bit = bv.get(3)      # 0
let ones = bv.count_ones()
```

### `BitArray<N>` (fixed length)

* Compile-time sized bitset with the same API subset as `BitVec`.

```loom
var mask: BitArray<128>
mask.set(5, 1)
```

### Interop with bytes

```loom
let by: bytes  = bv.to_bytes_be()           # pack to bytes (bit 0 is MSB in each byte)
let bv2: BitVec = BitVec::from_bytes_le(by) # choose endianness for packing
```

> **Indexing unit:** indices are **bit positions** (`usize`), not bytes.

---

## Packing fields (bitfields)

Use bit operations on integers or a helper type like `BitField`:

```loom
# Pack: [flag:1][type:3][id:12] → u16
pub func pack(flag: bit, typ: u3, id: u12): u16 {
    ret ((flag as u16) << 15) | ((typ as u16) << 12) | (id as u16)
}

# Unpack flag
let flag: bit = ((word >> 15) & 1u16) as bit
```

---

## Patterns & guidance

* Prefer `bit` when the **domain is strictly {0,1}** and arithmetic/bitwise algebra is intended.
* Prefer `bool` for **logical truth values** and control flow.
* In mixed arithmetic, be explicit about promotion to avoid surprises:

  ```loom
  let sum: i32 = (b as i32) + count
  ```
* For large boolean tables or sparse flags, **`BitVec`/`BitArray`** minimize memory.

---

## Examples

### Compute parity (even = 0, odd = 1)

```loom
pub func parity(mut x: u64): bit {
    var p: bit = 0
    while x != 0 {
        p ^= (x as bit)         # low bit
        x >>= 1
    }
    ret p
}
```

### Toggle a feature flag in a mask

```loom
const FEATURE_A: u32 = 1u32 << 7

pub func toggle(mask: u32, on: bit): u32 {
    ret if on == 1 { mask | FEATURE_A } else { mask & ~FEATURE_A }
}
```

### BitVec usage: sieve of Eratosthenes (mark non-primes)

```loom
pub func sieve(n: usize): BitVec {
    var is_prime = BitVec::with_len(n + 1, 1)  # assume 1 (true) for prime
    if n >= 0 { is_prime.set(0, 0) }
    if n >= 1 { is_prime.set(1, 0) }
    var p: usize = 2
    while p * p <= n {
        if is_prime.get(p) == 1 {
            var m = p * p
            while m <= n {
                is_prime.set(m, 0)
                m += p
            }
        }
        p += 1
    }
    ret is_prime
}
```

---

## FAQs

**Q: Why not just use `bool`?**
A: `bit` is numeric and algebraic (0/1) and packs densely in arrays. `bool` represents logical truth and participates in control flow; it need not pack densely.

**Q: Is `bit` signed or unsigned?**
A: Neither—it's a 1-bit integer with values `{0,1}`. When promoted, it behaves like an **unsigned** value.

**Q: Does `bit` convert to `bool` automatically in `if`/`while`?**
A: No. Convert explicitly to avoid mixing numeric 0/1 with logical truth accidentally.

**Q: What is the memory layout of `[]bit`?**
A: `BitVec`/`BitArray` pack bits into machine words; indexing is O(1). Iteration yields `bit` values.

---

## See also

* `bool` (logical truth type)
* Unsigned integers (`u8`, `u16`, …) for masks and shifts
* `byte`, `BitVec`, `BitArray` for packed representations
