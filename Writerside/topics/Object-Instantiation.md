# Object Instantiation

> This page shows *how to create objects* in Loom: constructors, `new`, field initialization, `fin` rules, reference semantics, and common creation patterns.

---

## Construction

A constructor has the same name as its class and runs after field storage is allocated.


```loom
pub class Human {
    priv var attributes: Person              # value field (record)
    priv fin var favoriteColor: Color        # write-once after construction

    pub Human(name: string, age: i32, favoriteColor: Color) {
        attributes.name = name               # initialize nested record fields
        attributes.age  = age
        self.favoriteColor = favoriteColor   # assign 'fin' exactly once
    }

    pub func greet(): void {
        printf("Hello, my name is {}.\n", self.attributes.name)
    }
}
```

### Access control

* `pub Human(...)` → constructible from other modules.
* `priv Human(...)` → constructible only inside the defining module (use a public factory if you want controlled creation).

### Overloading

You may define multiple constructors with different parameter lists. Choose the one you need at call-site:

```loom
pub Human(name: string) {
    self(name, 0, Color.BLUE)    # call into a canonical initializer (see below)
}

pub Human(name: string, age: i32, color: Color) {
    attributes.name = name
    attributes.age  = age
    self.favoriteColor = color
}
```

> If your codebase uses *delegating constructors*, the idiom is to route the shorter constructor’s body into the canonical one (e.g., by repeating the initialization or by factoring shared logic into a private init method). Exact delegation syntax is a style choice; the example above shows the intent.

---

## Creating objects with `new`

```loom
var h1: Human = new Human("Alice", 21, Color.BLUE)
var h2        = new Human("Bob", 23, Color.RED)   # type inferred as Human
```

* `new` returns a **reference** to a freshly allocated instance.
* The constructor runs exactly once before `new` returns.
* If any required field is left uninitialized (e.g., a `fin` field), construction fails at runtime.

---

## Field initialization strategies

You can initialize fields **inline** or **in the constructor**.

### Inline (field initializers)

```loom
pub class Counter {
    priv var value: i64 = 0
    priv fin var id: i64 = now_nanos()   # computed once at allocation

    pub Counter() { }                    # empty body is fine
}
```

* Field initializers run **before** the constructor body.
* They are a good fit for constants, defaults, and one-time computed values.
* `fin` fields may be initialized inline **or** assigned once in the constructor (not both).

### In the constructor

Use the constructor when initialization depends on parameters or requires validation:

```loom
pub class PersonCard {
    priv fin var name: string
    priv var age: i32

    pub PersonCard(name: string, age: i32) {
        if age < 0 { throw "age must be nonnegative" }
        self.name = name
        self.age  = age
    }
}
```

---

## `fin` (write-once) fields

`fin` enforces *single assignment*:

* A `fin` field must be assigned exactly once (inline **or** in the constructor).
* Reassigning a `fin` field later is a compile/runtime error.
* Expose read-only access via a getter or computed property.

```loom
priv fin var id: i64
pub func id(): i64 { ret self.id }
```

---

## Reference semantics & variable mutability

Variables can be `let` (binding is immutable) or `var` (binding is reassignable).
**This affects the *reference*, not the object’s internal mutability.**

```loom
let a = new Human("A", 20, Color.RED)
# a = new Human(...)     # ERROR: cannot rebind 'let' variable

var b = new Human("B", 25, Color.BLUE)
b = a                    # OK: 'b' now refers to the same object as 'a'
```

* Multiple variables can point to the **same instance**.
* Mutating through one reference is visible through the others.

---

## Optional construction patterns

### Factories (named constructors)

Hide complex setup behind a static factory:

```loom
pub class Human {
    # ... fields/constructors ...

    pub static func student(name: string): Human {
        ret new Human(name, 18, Color.GREEN)
    }
}

var s = Human.student("Eve")
```

### Builder methods (for readability)

When many optional parameters exist, a fluent/builder API can keep call-sites tidy. (This is style and library, not required by Loom.)

---

## Error handling during construction

* Throw inside a constructor to abort creation (e.g., invalid arguments).
* If a constructor throws, no partially-initialized object escapes.

```loom
pub Account(name: string, balance: i64) {
    if balance < 0 { throw "negative opening balance" }
    self.name    = name
    self.balance = balance
}
```

---

## Common gotchas

* **Uninitialized `fin`:** ensure every constructor path assigns each `fin` field exactly once.
* **Shadowing parameters:** prefer `self.field = param` to avoid accidental self-assignment.
* **Reference vs copy:** `alice = bob` rebinds the reference; it does not clone. Provide an explicit `clone()` if you need a deep copy.

---

## Complete example

```loom
mod first

pub enum Color { RED("red"), GREEN("green"), BLUE("blue")
    priv fin var name: string
    pub Color(name: string) { self.name = name }
    pub func toString(): string { ret self.name }
}

pub struct Person { name: string, age: i32 }

pub class Human {
    priv var attributes: Person
    priv fin var favoriteColor: Color

    # Canonical constructor
    pub Human(name: string, age: i32, favoriteColor: Color) {
        attributes.name = name
        attributes.age  = age
        self.favoriteColor = favoriteColor
    }

    # Convenience constructor
    pub Human(name: string) {
        self(name, 0, Color.BLUE)   # delegate to canonical initialization
    }

    pub func getName(): string { ret self.attributes.name }
    pub func getAge(): i32     { ret self.attributes.age }
    pub func getFavoriteColor(): Color { ret self.favoriteColor }

    pub func greet(): void {
        printf("Hello, my name is {}. I am {} years old and I like {}.\n",
               self.getName(), self.getAge(), self.getFavoriteColor().toString())
    }
}

pub func demo(): void {
    var a = new Human("Alice", 21, Color.BLUE)
    var b = new Human("Bob")                    # uses convenience constructor
    a.greet()
    b.greet()
}
```

---

## See next

* **Object-Inheritance.md** — subclassing (`Human: Animal`), overrides, dynamic dispatch
* **Templates.md** — declaring & fulfilling template members (e.g., `template var getType(): Type`)
