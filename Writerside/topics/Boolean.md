# Boolean

`bool` is Loom’s **Boolean** type with two values: **`true`** and **`false`**.
It is a **shorthand for `bit`** (0/1) with **logical semantics** and is used in control flow, predicates, and boolean algebra.

* **Values:** `true`, `false` (alias of `1` and `0` at the storage level)
* **Storage:** 1 bit (stand-alone variables typically occupy 1 byte; arrays are packed)
* **Interchange:** Zero-cost aliasing with `bit`; explicit casts for integers

---

## Literals & basics

```loom
let a: bool = true
let b: bool = false
var ok      = (3 < 5)        # true
```

**Printing:** `print(true)` → `true`

---

## Control flow

`bool` guards `if`, `while`, `match` guards, etc.

```loom
if ok { println("it passed") }

while cond() {
    step()
}
```

---

## Operators

### Logical (short-circuit)

* `!p` (NOT)
* `p and q` (AND) — evaluates `q` only if `p` is `true`
* `p or q` (OR)  — evaluates `q` only if `p` is `false`

### Bitwise (non–short-circuit, defined for `bool`)

* `p & q`, `p | q`, `p ^ q`  (evaluate both sides, return `bool`)
* Equality: `p == q`, `p != q`
* **Ordering** (`< <= > >=`) is **not defined** for `bool`; cast to `bit`/int if needed.

```loom
let p = true
let q = false
let x = p and expensive()     # short-circuits
let y = p &  expensive()     # always evaluates
let z = p ^ q                # true
```

---

## Conversions

### `bool` ⇄ `bit`

* `bool → bit`: `true → 1`, `false → 0`
* `bit  → bool`: `1 → true`, `0 → false`
* Both are **zero-cost** (same underlying bit), but casts make intent clear.

```loom
let b: bit  = true as bit
let t: bool = (1 as bit) as bool
```

### `bool` ⇄ integers

* **Explicit** casts only (avoid accidental logic–arithmetic mixing):

    * `bool → iN/uN`: `true → 1`, `false → 0`
    * `iN/uN → bool`: `(value != 0)` or `(value & 1) != 0`, via cast/helpers

```loom
let n: u32  = (flag as u32)             # 0 or 1
let ok: bool = (count as i32) != 0
# Helpers if provided:
# let ok = bool.from_int(count)   # true if nonzero
# let v  = flag.to_int()          # 0 or 1
```

### `bool` ⇄ `string`

```loom
let s = bool.to_string(true)            # "true"
let v = bool.parse("false")             # false (throws on invalid)
let o = bool.parse_opt("yes")           # None
```

---

## Collections & packing

* `[]bool` (slices) and arrays are **bit-packed**; indexing is O(1).
* Iteration yields `bool`:

```loom
var marks: [bool; 8]
for i in 0..marks.len() { marks[i] = (i % 2 == 0) }
for m in marks { if m { /* ... */ } }
```

For large bitsets plus bitwise bulk ops, prefer `BitVec`/`BitArray` (see `bit`).

---

## Standard predicates & helpers

```loom
is_ascii_alnum(ch) -> bool     # example: library predicates return bool
all(xs: []bool) -> bool        # true if all true
any(xs: []bool) -> bool        # true if any true
```

---

## Patterns & guidance

* Use **`bool`** for logic and control flow; use **`bit`** for numeric bit math and compact masks.
* Prefer **`and`/`or`** in conditionals for short-circuiting; use **`&`/`|`/`^`** when you **require** both sides to evaluate (e.g., flag combinators).
* When mixing with integers, **cast explicitly** to document intent.

---

## Examples

### Guarded initialization

```loom
var initialized: bool = false
var handle: i32 = -1

if !initialized {
    handle = open()
    initialized = (handle >= 0)
}
```

### Combining feature flags

```loom
let has_gpu:  bool = probe_gpu()
let want_gpu: bool = cfg.use_gpu
let use_gpu  = has_gpu and want_gpu
```

### Filtering with predicates

```loom
pub func count_true(xs: []bool): usize {
    var n: usize = 0
    for v in xs { if v { n += 1 } }
    ret n
}
```

---

## FAQs

**Is `bool` just a typedef of `bit`?**
Yes—same underlying representation, but `bool` carries **logical** intent, has **`true`/`false` literals**, and is accepted directly by control flow.

**Can I rely on `sizeof(bool) == 1`?**
Standalone values typically occupy 1 byte; arrays/slices are bit-packed. For FFI, match the foreign ABI explicitly (e.g., map to C `_Bool` or `uint8_t`).

**Why no `<`/`>` on `bool`?**
To avoid accidental ordinal comparisons; cast to `bit`/int if you need 0/1 ordering.

---

## See also

* `bit` (numeric single-bit type, `BitVec`, `BitArray`)
* Integers (`i*`, `u*`) for arithmetic
* Control flow (`if`, `while`, `match` guards)
