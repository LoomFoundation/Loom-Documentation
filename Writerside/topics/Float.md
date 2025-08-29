# Floating-Point Types

This page documents Loom’s **binary IEEE-754** floating-point types. Use these for real-number arithmetic, scientific formulas, graphics, and simulations where fractional values are required.

---

## Quick reference

|  Type | Bits | IEEE-754 format | Sig. precision (bits / \~digits) | Min/Max finite            | Default? |
| ----: | ---: | --------------- | -------------------------------- | ------------------------- | -------- |
| `f16` |   16 | binary16        | 11 bits ≈ 3–4 dec digits         | \~±6.10e−5 … ±6.55e+4     | No       |
| `f32` |   32 | binary32        | 24 bits ≈ 6–7 dec digits         | \~±1.18e−38 … ±3.40e+38   | No       |
| `f64` |   64 | binary64        | 53 bits ≈ 15–16 dec digits       | \~±2.23e−308 … ±1.79e+308 | **Yes**  |

* **Representation:** IEEE-754 with **sign**, **exponent**, **fraction**; **round-to-nearest, ties-to-even** by default.
* **Specials:** `+0.0`, `-0.0`, `+∞`, `-∞`, `NaN` (quiet NaN used by default).
* **Subnormals:** Supported (gradual underflow).

> `f16` is space-efficient but imprecise; prefer `f32` for UI/graphics and `f64` for scientific/finance.
> Bare float literals without suffix infer to **`f64`** unless context suggests otherwise.

---

## Literals

Decimal literals with optional exponent and underscores:

```loom
let a: f64 = 1.0
let b: f32 = 3.1415927f32
let c      = 6.022_140_76e23          # inferred f64
let d: f16 = 1.0e-3f16
let n: f64 = 0.0                       # +0.0 and -0.0 both exist
```

Type suffixes: `f16`, `f32`, `f64`.

Special constants:

```loom
let inf  = f64::INFINITY
let ninf = f64::NEG_INFINITY
let nan  = f64::NAN
```

---

## Operators & semantics

* **Arithmetic:** `+  -  *  /`
* **Remainder:** `a % b` is IEEE remainder (equivalent to `a - trunc(a/b) * b`)
* **Comparisons:** `== != < <= > >=` (see NaN rules below)

```loom
let x: f32 = 5.5
let y: f32 = 2.0
let q = x / y          # 2.75
let r = x % y          # 1.5
```

### Exceptional results

* Divide by zero: `(+ value) / 0.0 → +∞`, `(- value) / 0.0 → -∞`
* `0.0 / 0.0`, `∞ - ∞`, `sqrt(-1.0)` → `NaN`
* Overflow → `±∞`; underflow → subnormal or `±0.0` with loss of precision

---

## NaN, ±0.0, and comparisons

* Any comparison with `NaN` is **false** except `!=`, which is **true**.
* `+0.0 == -0.0` is **true**, but they have different signs.
* For a **total ordering** (sorting, maps), use `total_cmp(a, b)`.

Helpers:

```loom
x.is_nan()
x.is_finite()
x.is_infinite()
x.is_subnormal()
x.signum()        # +1.0, -1.0, or NaN
copysign(mag, sign_source)
```

---

## Rounding & next-after

* Default rounding: **nearest, ties-to-even**.
* Step to adjacent representable values:

```loom
x.next_up()
x.next_down()
```

---

## Conversions

### Float ↔ Float

* Widening (e.g., `f32 → f64`) is exact.
* Narrowing (e.g., `f64 → f32`) **rounds**; use:

    * `as` cast (rounds to nearest)
    * `to_f32_checked()` → `(f32, overflowed: bool)`  (flags `∞`, `NaN`, or out-of-range)
    * `to_f32_saturating()` → clamps to finite max/min

### Integer ↔ Float

* **Int → Float:** exact if within mantissa precision; else rounded.
* **Float → Int:** **truncates toward zero**.

    * Checked variants: `to_i32_checked()`, `to_u32_checked()`, …
    * Saturating variants: `to_i32_saturating()`, …

```loom
let i: i32 = (3.9f32) as i32   # 3
let f: f64 = (1_000_000i32) as f64
```

---

## Promotions & mixed-type rules

* `f16`, `f32`, `f64` in an expression promote to the **widest** present (`f64` > `f32` > `f16`).
* Mixing ints and floats promotes the int to the float’s type.
* No implicit conversion between floats and strings; use parse/format APIs.

---

## Math library (selected)

* Roots & magnitudes: `sqrt`, `cbrt`, `hypot`
* Rounds: `floor`, `ceil`, `round`, `trunc`, `fract`
* Exponentials & logs: `exp`, `exp2`, `ln`, `log10`, `log2`, `powf(y)`, `powi(k)`
* Trig: `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2(y, x)`
* Hyperbolic: `sinh`, `cosh`, `tanh`, …
* FMA: `fma(a, b, c)`  (computes `a*b + c` with a single rounding)
* Decompose/compose: `frexp()` → `(mantissa, exp)`, `ldexp(mantissa, exp)`
* Split: `modf()` → `(int_part, frac_part)`
* Classification: `classify()` → enum of {Zero, Subnormal, Normal, Inf, NaN}

```loom
let r: f64 = f64::hypot(3.0, 4.0)      # 5.0
let z: f32 = 1.0f32.fma(1e10f32, -1e10f32)  # avoids catastrophic cancelation
```

---

## Formatting & parsing

```loom
let v: f64 = 1234.56789

print(v)                                  # 1234.56789 (default)
printf("fixed=%.2f sci=%.3e gen=%g\n", v, v, v)
# fixed=1234.57 sci=1.235e+03 gen=1234.57

let a: f32 = f32.parse("3.14")
let b: f64 = f64.parse("6.022e23")
let o_opt  = f64.parse_opt("NaN")         # → Option<f64>
```

> Parsing accepts `inf`, `+inf`, `-inf`, and `nan` (case-insensitive). `nan(payload)` is permitted but payload bits are not guaranteed to round-trip across platforms.

---

## Bit-level access & endianness

```loom
let bits: u32 = (3.5f32).to_bits()        # raw IEEE-754
let x: f32   = f32.from_bits(bits)

let be = x.to_be_bytes()                  # explicit byte order for I/O
let y  = f32.from_be_bytes(be)
```

---

## Performance, precision & determinism

* **Prefer `f64`** for numerically sensitive work; `f32` for memory/throughput.
* Floating arithmetic is **not** associative: `(a+b)+c` may differ from `a+(b+c)`.
* For reproducible results across platforms:

    * Avoid “fast-math” compilation for critical code.
    * Use **`fma`**, stable algorithms (e.g., **Kahan summation**), and fixed evaluation order.
* Binary floats cannot exactly represent many decimals (e.g., `0.1`). Compare with tolerances.

```loom
pub func approx_eq(a: f64, b: f64, eps: f64 = 1e-9): bool {
    ret (a - b).abs() <= eps * (1.0 + a.abs().max(b.abs()))
}
```

---

## Examples

### Kahan (compensated) summation

```loom
pub func sum_kahan(xs: []f64): f64 {
    var s = 0.0
    var c = 0.0
    for x in xs {
        let y = x - c
        let t = s + y
        c = (t - s) - y
        s = t
    }
    ret s
}
```

### Fast, precise linear interpolation

```loom
# lerp(a, b, t) = a + t*(b - a), but fma avoids extra rounding
pub func lerp(a: f32, b: f32, t: f32): f32 {
    ret (b - a).fma(t, a)
}
```

### Safe normalization with edge cases

```loom
pub func normalize(x: f64, y: f64): (f64, f64) {
    let d = f64::hypot(x, y)     # robust sqrt(x*x + y*y)
    if d == 0.0 { ret (0.0, 0.0) }
    ret (x / d, y / d)
}
```

### Constraining to a finite range

```loom
pub func clamp01(x: f32): f32 {
    if x.is_nan() { ret 0.0f32 }                  # define your policy
    ret x.max(0.0f32).min(1.0f32)
}
```

---

## FAQs

**Q: Why did my `0.1 + 0.2` become `0.30000000000000004`?**
A: `0.1` and `0.2` are not exactly representable in binary floating point. Compare using a tolerance (see `approx_eq`).

**Q: Should I store currency in floats?**
A: Prefer **scaled integers** (e.g., cents in `i64`) or decimal types. Use floats only for approximate calculations.

**Q: When should I use `f16`?**
A: For dense arrays where memory/bandwidth matter and precision requirements are low (e.g., ML activations, approximate textures). Convert to `f32`/`f64` for computation if needed.

**Q: How do I sort values with NaNs?**
A: Use `total_cmp(a, b)`; it defines a total order (e.g., `NaN` ordered after finite numbers).

---

## See also

* Integers: `i8/i16/i32/i64`, `u8/u16/u32/u64`
* Complex numbers (if enabled by your profile)
* Numerics & math utilities in the standard library
