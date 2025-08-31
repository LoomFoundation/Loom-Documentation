# Arrays & Array Literals

Arrays in Loom are **homogeneous, mutable sequences**. You can build them with **literals** or with the builtin **`array(type, size[, init])`** constructor.

---

## Array literals

**Syntax (no trailing comma):**

```loom
let a: []i32 = [1, 2, 3]
let s: []string = ["alpha", "beta"]
let m: [][]i32 = [[1, 2, 3], [10, 20, 30]]
```

### Interpreter semantics (`[]any` temporary)

In the interpreter/REPL, an array literal evaluates to a **runtime `[]any`**:

```loom
let z: []i32 = [10, 12, 14]   # literal is []any at runtime with elements 10,12,14
```

* When you assign a literal to a **typed** variable (e.g., `[]i32`), the interpreter **validates/casts each element** to the target element type.
* If any element can’t convert, you’ll get a runtime type error at assignment.

### Indexing & mutation

```loom
let v = z[1]       # → 12   (read)
z[1] = 99          # mutates z → [10, 99, 14]
```

* Indexing is **0-based** and **bounds-checked** (out of bounds throws).
* Nested arrays chain indexing: `m[1][2]`.

---

## Builtin: `array(type, size[, init])`

Create arrays programmatically, with optional initializer:

```loom
let a = array(i32, 3)        # → [null, null, null]
let b = array(i32, 3, 0)     # → [0, 0, 0]

let c = array(string, 2, "x")  # → ["x", "x"]
let d = array([]i32, 2, [])    # → [[], []]
```

Notes:

* `type` is the **element type**.
* `size` is the **length** (must be ≥ 0).
* `init` is **optional**. If omitted, elements start as **`null`** (useful when you intend to fill later).
  Assigning a non-nullable type and leaving `null` elements in place may error when read—fill them before use.

---

## Mutability & core ops (on `[]T`)

```loom
var a: []i32 = [1, 2, 3]
a.len()                 # → 3
a.is_empty()            # → false

a.push(4)               # [1, 2, 3, 4]
let last = a.pop()?     # → Some(4), a = [1, 2, 3]

a.insert(1, 99)?        # [1, 99, 2, 3]
let x = a.remove(2)?    # removes element at index 2 → 2, a = [1, 99, 3]

a.extend([7, 8])        # [1, 99, 3, 7, 8]
a.clear()               # []
```

Slicing (views, no copy):

```loom
var b: []i32 = [10, 20, 30, 40, 50]
let mid: []i32 = b[1..4]     # [20, 30, 40] (view)
mid[0] = 21                  # mutates b too → b = [10, 21, 30, 40, 50]
```

---

## Types, inference, and nesting

* Untyped integer literals default to **`i32`**; floats to **`double`**.
* Mixed numeric widths promote to the **widest compatible** signedness; mixing signed/unsigned or ints/floats requires a target element type.

Examples:

```loom
let xs = [1, 2, 3]                 # []i32
let ys: []i64 = [1, 2, 3]          # forced to i64
let zs: []double = [1, 2.5, 3.0]   # annotate to promote to double

let grid: [][]i32 = [[1, 2], [3, 4]]   # nested arrays
```

---

## Common patterns

**Matrix (row-major) initialization:**

```loom
let rows = 3
let cols = 2
var M: [][]i32 = []
for r in 0..rows {
    var row: []i32 = array(i32, cols, 0)   # [0, 0]
    M.push(row)
}
# M = [[0,0], [0,0], [0,0]]
```

**Prealloc then fill:**

```loom
var a: []string = array(string, 3)  # [null, null, null]
a[0] = "A"
a[1] = "B"
a[2] = "C"
```

**Build and mutate in place:**

```loom
var primes: []i32 = [2, 3, 5, 7]
primes.push(11)
primes[1] = 13
```

---

## FAQs

**Q: Do array literals allow trailing commas?**
A: No. `[` *elements separated by commas* `]` only—**no trailing comma**.

**Q: Why does the interpreter say my literal is `[]any`?**
A: Array literals are created as `[]any` at runtime. Assigning to a typed variable (e.g., `[]i32`) validates/casts the elements to the target type.

**Q: What’s in `array(T, n)` when `init` is omitted?**
A: `null` placeholders. Assign before reading to avoid runtime errors for non-nullable element types.

**Q: Are slices views or copies?**
A: Views. Mutating a slice view mutates the original array.

---

## See also

* Numeric types (`i*`, `u*`, `double`)
* `byte` (`[]u8`) for raw binary data
* Strings & chars (`string`, `char`)
