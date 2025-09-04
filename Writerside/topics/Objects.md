# Objects

Objects in Loom are **first-class, reference types** that bundle **state** (fields) with **behavior** (methods and computed properties). They give your programs structure: you name concepts (like `Human`), store data about them (like `name`, `age`), and expose well-defined operations (like `greet()`).

This page explains what an object *is* and what it’s *for*. Creation syntax, inheritance, and template contracts are covered in the next chapters:

* [](Object-Instantiation.md) — creating and initializing objects (`new`, constructors)
* [](Object-Inheritance.md) — subclassing, overrides
* [](Template-Classes.md) — template/abstract contracts

---

## What is an object?

An object is an instance of a **class**. A class defines:

* **Fields** (also called *properties*): named pieces of state with types.
* **Methods**: functions attached to the class that operate on its state.
* **Computed properties**: read-only “fields” implemented with code (getter bodies).
* **Visibility** and **immutability** rules that control how state is accessed and changed.

Objects are **reference values**: assigning one variable to another copies a reference, not the entire object. Swapping two variables swaps *which instance* each name points to.

> **Memory model (high level):** instances live on the heap; lifetimes are managed by the Loom runtime. You don’t manually free objects.

---

## Why use objects?

* **Encapsulation:** hide representation details behind a stable API.
* **Coherence:** keep data and the code that manipulates it in one place.
* **Reusability:** define once, use across modules and programs.
* **Substitutability:** (via inheritance/templates) treat different concrete types through a shared interface (covered later).

---

## Object anatomy (at a glance)

Here’s a trimmed version of your `Human` example with annotations:

```loom
# Human.lm
mod first
import first.Main::{ Animal, Type }

pub class Human: Animal {
    # 1) Fields (state)
    priv var attributes: Person        # encapsulated record
    priv fin var favoriteColor: Color  # 'fin' => write-once after construction

    # 2) Constructor (initializes fields)
    pub Human(name: string, age: i32, favoriteColor: Color) {
        attributes.name = name
        attributes.age = age
        self.favoriteColor = favoriteColor
    }

    # 3) Methods (behavior)
    pub func getName(): string { ret self.attributes.name }
    pub func getAge(): i32     { ret self.attributes.age }

    # 4) Computed property (getter-only) overriding a template slot
    pub override var getType(): Type { ret Type.HUMAN }

    # 5) Convenience computed property returning the instance
    pub var get(): self { ret self }

    pub func greet(): void {
        printf(
          "Hello, my name is {}. I am a {}! My favorite color is {}!\n",
          self.attributes.name,
          self.getType().toString(),
          self.getFavoriteColor().toString()
        )
    }

    pub func getFavoriteColor(): Color { ret self.favoriteColor }
}

# Support types used by Human
pub struct Person { name: string, age: i32 }
pub enum Color {
    RED("red"), GREEN("green"), BLUE("blue")
    priv fin var name: string
    pub Color(name: string) { self.name = name }
    pub func toString(): string { ret self.name }
}
```

### Key ideas shown above

* **Fields** use `var`; add `fin` to make them **write-once** (commonly set in the constructor).
* **Constructor** is a function named the same as the class; it establishes class invariants.
* **Methods** use `pub/priv func` and can read or mutate state through `self`.
* **Computed properties** use `var name(): Type { ... }` with a body: they *look like fields* to callers but are implemented by code. (Overriding such a slot is how abstract/template “properties” are fulfilled—details later.)
* **Encapsulation**: `attributes` is `priv`; callers use `getName()`/`getAge()` instead of reaching in.

---

## Using objects across modules (conceptual)

Your `Main.lm` imports `Human` and works with instances:

```loom
# Main.lm (excerpt)
mod first
import first.Human::{ Human, Color }

pub func Main(): i8 {
    var alice: Human = new Human("Alice", 21, Color.BLUE)
    var bob:   Human = new Human("Bob",   23, Color.RED)

    alice.greet()
    bob.greet()

    # Compare derived/computed information
    println(alice.getAge() > bob.getAge()
        ? "Alice is older than Bob."
        : alice.getAge() < bob.getAge()
            ? "Bob is older than Alice."
            : "Alice and Bob are the same age.")

    # Swap references (identity follows the variable)
    let temp = alice
    alice = bob
    bob = temp

    alice.greet()
    bob.greet()
    ret 0
}
```

This illustrates two important semantics:

* **Method dispatch:** `alice.greet()` invokes the `Human` method with `self = alice`.
* **Reference assignment:** `alice = bob` changes what `alice` refers to; it does **not** clone `bob`.

---

## Visibility & encapsulation

* `pub` members are part of the public API of the class/module.
* `priv` members are internal implementation details.
* Prefer exposing **methods** or **computed properties** over public mutable fields.
* Use **`fin`** for values that must not change after construction (e.g., IDs, creation timestamps, “favorite” selections in the example).

```loom
priv fin var id: i64              # immutable identity
pub  func id(): i64 { ret self.id }  # expose as read-only
```

---

## Methods vs. computed properties

* **Method** — action, may take parameters: `pub func greet(): void`.
* **Computed property** — value derived from state, parameterless, declared with `var name(): Type { ... }`. Reads like a field from the outside, but you can add logic without changing callers.

Use computed properties when the *concept* is a value, not an action (e.g., `getType`, `fullName`, `isAdult`).

---

## State modeling patterns

* **Value objects**: small immutable aggregates (e.g., `Person`) used as fields inside larger objects.
* **Enums with payload**: model labeled variants with associated data (your `Color`/`Type` enums use a constructor to bind a display string).
* **Records vs. classes**: prefer small `struct`/`record` types for plain data; promote to a `class` when behavior and invariants matter.

---

## Equality, identity, and printing (conceptual guidance)

* **Identity**: two variables can reference the *same* object (identity equality).
* **Value equality**: provide methods to compare by content (e.g., same `name` and `age`) if needed.
* **String form**: implement `toString()` on your types/enums to control how objects print in logs/UI.

*(Exact operators and hooks for identity/value equality are defined in the language reference; this section focuses on object concepts.)*

---

## Best practices

* **Keep fields `priv`;** publish behavior, not representation.
* **Use `fin`** to lock invariants after construction.
* **Prefer computed properties** for derived values; they make refactors safer.
* **Validate in the constructor** to keep objects always-valid.
* **Keep modules small and focused;** expose only what other modules need (`import` the minimal surface).
