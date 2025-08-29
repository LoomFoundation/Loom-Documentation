# Double Precision Types

`double` is Loom’s **64-bit IEEE-754 binary64** floating-point type. Use it for high-precision real-number arithmetic in science, simulation, graphics, finance (with caveats), and general numeric work.

> Note: In some older examples you may see `f64`. Treat it as synonymous with `double`.

---

## Quick reference

| Property           | Value                                       |
| ------------------ | ------------------------------------------- |
| Width              | 64 bits (binary64)                          |
| Precision          | 53 significant bits ≈ 15–16 decimal digits  |
| Finite range       | \~±2.23e−308 … ±1.79e+308                   |
| Rounding mode      | Round-to-nearest, ties-to-even              |
| Specials           | `+0.0`, `-0.0`, `+∞`, `-∞`, `NaN` (quiet)   |
| Subnormals         | Supported (gradual underflow)               |
| Default float type | Yes — bare float literals infer to `double` |

ABI: `sizeof(double) == 8`, `alignof(double) == 8` (typical).

---

## Literals

Decimal with optional exponent; underscores allowed for readability. Suffix is optional for `double` (it’s the default).

```loom
let a: double = 1.0
let b          = 3.14159          # inferred double
let c: double = 6.022_140_76e23
let z: double = -0.0               # negative zero is distinct from +0.0
```

Special constants:

```loom
let inf  = double::INFINITY
let ninf = double::NEG_INFINITY
let nan  = double::NAN
```

---

## Operators & semantics

* **Arithmetic:** `+  -  *  /`
* **Remainder:** `a % b` → IEEE remainder (`a - trunc(a/b)*b`)
* **Comparisons:** `==  !=  <  <=  >  >=`

```loom
let x: double = 5.5
let y: double = 2.0
let q = x / y      # 2.75
let r = x % y      # 1.5
```

### Exceptional results

* `(+val) / 0.0 → +∞`, `(-val) / 0.0 → -∞`
* `0.0 / 0.0`, `∞ - ∞`, `sqrt(-1.0)` → `NaN`
* Overflow → `±∞`; underflow → subnormal or signed zero (precision lost)

---

## NaN, ±0.0, ordering

* Any comparison with `NaN` is **false** except `!=`, which is **true**.
* `+0.0 == -0.0` is **true**, but the sign bit differs.
* For total ordering (sorting, maps), use `total_cmp(a, b)`.

Useful predicates & helpers:

```loom
x.is_nan()
x.is_finite()
x.is_infinite()
x.is_subnormal()
x.signum()            # +1.0, -1.0, or NaN
copysign(magnitude, sign_source)
```

---

## Conversions

### Double ↔ other floats

* Widening to a higher-precision type (if present) is exact; narrowing to a smaller one rounds to nearest.
* Casts: `as` rounds; `to_f32_checked()` and `to_f32_saturating()` provide safe variants.

### Double ↔ integers

* **Int → double:** exact if the integer fits within 53 bits of precision; otherwise rounded.
* **Double → int:** **truncates toward zero**; out-of-range is checked in Debug and undefined in Release unless using safe helpers.

```loom
let i: i32    = (3.9 as i32)         # 3
let f: double = (1_000_000 as double)
let (v, of)   = double.to_i64_checked(9.22e18)  # detect overflow
```

---

## Mixed-type rules

* Mixing integers with `double` promotes the integer operand to `double`.
* Mixing `double` with lower-precision floats promotes to `double`.
* No implicit conversion between `double` and strings—use parse/format APIs.

---

## Math library (selected)

* Magnitude & roots: `abs`, `sqrt`, `cbrt`, `hypot`
* Rounding: `floor`, `ceil`, `round`, `trunc`, `fract`
* Exp & logs: `exp`, `exp2`, `ln`, `log10`, `log2`, `powf(y)`, `powi(k)`
* Trig: `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2(y, x)`
* Hyperbolic: `sinh`, `cosh`, `tanh`, …
* **FMA:** `fma(a, b, c)` computes `a*b + c` with a single rounding
* Decompose/compose: `frexp()` → `(mantissa, exp)`, `ldexp(m, e)`
* Split: `modf()` → `(int_part, frac_part)`
* Classification: `classify()` → `{Zero, Subnormal, Normal, Inf, NaN}`

```loom
let r: double = double::hypot(3.0, 4.0)         # 5.0
let z: double = 1.0.fma(1e10, -1e10)            # avoids extra rounding
```

---

## Bit-level access & endianness

```loom
let bits: u64   = (3.5 as double).to_bits()     # raw IEEE-754
let x: double   = double.from_bits(bits)

let be = x.to_be_bytes()                        # explicit byte order for I/O
let y  = double.from_be_bytes(be)
```

---

## Formatting & parsing

```loom
let v: double = 1234.56789

print(v)                                        # 1234.56789
printf("fixed=%.2f sci=%.3e gen=%g\n", v, v, v)
# fixed=1234.57 sci=1.235e+03 gen=1234.57

let a: double = double.parse("3.14")
let b: double = double.parse("6.022e23")
let o_opt     = double.parse_opt("NaN")         # Option<double>
```

Accepted special inputs: `inf`, `+inf`, `-inf`, `nan` (case-insensitive). `nan(payload)` is permitted; payload bit semantics are platform-dependent.

---

## Performance, precision & determinism

* Prefer `double` for numerically sensitive work; use lower precision only for bandwidth/memory.
* Floating arithmetic is **not associative**: `(a+b)+c` may differ from `a+(b+c)`.
* For reproducibility across platforms/compilers:

    * Avoid “fast-math” for critical code.
    * Use **`fma`**, numerically stable algorithms (e.g., **Kahan summation**), and fixed evaluation order.
* Many decimals (e.g., `0.1`) aren’t exactly representable; compare with tolerances.

```loom
pub func approx_eq(a: double, b: double, eps: double = 1e-12): bool {
    ret (a - b).abs() <= eps * (1.0 + a.abs().max(b.abs()))
}
```

---

## Examples

### Kahan (compensated) summation

```loom
pub func sum_kahan(xs: []double): double {
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

### Robust normalization

```loom
pub func normalize(x: double, y: double): (double, double) {
    let d = double::hypot(x, y)
    if d == 0.0 { ret (0.0, 0.0) }
    ret (x / d, y / d)
}
```

### Stable linear interpolation using FMA

```loom
pub func lerp(a: double, b: double, t: double): double {
    ret (b - a).fma(t, a)
}
```

### Total ordering with NaNs (for sorting)

```loom
pub func sort_doubles(xs: []double): []double {
    xs.sort_with(|l, r| total_cmp(l, r))
    ret xs
}
```

---

## FAQs

**Q: Should I use `double` for currency?**
A: Prefer scaled integers (e.g., cents in `i64`) or decimal types to avoid rounding artifacts. Use `double` only when approximate values are acceptable.

**Q: Why does `0.1 + 0.2` print `0.30000000000000004`?**
A: `0.1` and `0.2` can’t be represented exactly in binary floating point; the nearest representable values sum to a close neighbor. Compare with tolerances.

**Q: How do I make results reproducible across platforms?**
A: Disable aggressive fast-math, use stable algorithms (Kahan, pairwise summation), rely on `fma` where available, and fix evaluation order.

---

## See also

* Integers: `i8/i16/i32/i64`, `u8/u16/u32/u64`
* Lower-precision floats (if enabled): `f16`, `f32`, `f64`
* Numerics & math utilities in the standard library
