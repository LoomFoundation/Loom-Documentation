# Home

<img src="logo.svg" alt="Cloth Logo" width="200" style="block">

> This documentation is a work in progress, alongside Cloth itself. We need your help!
{style="warning"}

Cloth is a statically typed, object-oriented programming language for math-heavy work: models, formulas, and simulations.  
It aims to be easy to read, fast to run, and predictable in behavior.

## What Cloth is

- **Statically typed.** Types are checked at compile time, so common mistakes are caught early.
- **Object-oriented.** Classes, encapsulation, and modules help structure both small scripts and large systems.
- **Built for numbers.** Comfortable with equations and simulation code, with a focus on correctness and clarity.
- **Practical.** Syntax stays out of the way; what you write should match what you mean.

## Design goals

- **Clarity over ceremony.** Minimal surface area; explicit where it matters.
- **Performance you can rely on.** Compiled output designed for scientific workloads and long-running simulations.
- **Predictable semantics.** The same code produces the same results across runs and environments.
- **Maintainability.** Scales from quick experiments to production systems without rewriting everything.

## When to reach for Cloth

- You’re modeling systems (physics, finance, biology, etc.) and care about correctness.
- You’re building simulations that need to run efficiently.
- You want readable, strongly-typed code that’s straightforward to review and extend.
- You prefer a simple, modern OOP model over a web of language features.

## What Cloth isn’t

- Not a scripting language optimized for one-off throwaways.
- Not a “kitchen sink” language—features are added only when they pay for themselves.
- Not tied to a particular framework or runtime.

## Language at a glance

- **Types:** Statically checked; explicit where helpful.
- **Errors:** Compile-time where possible; clear runtime messages when not.
- **Modules:** Organize code into reusable units.
- **Tooling:** Aims for fast builds and a smooth edit-compile-run loop.

## Where to go next

- **Overview:** How Cloth programs are organized → `/guide/overview`
- **Type system:** Numbers, strings, collections, and custom types → `/language/types`
- **OOP model:** Classes, methods, visibility, and modules → `/language/oop`
- **Building & running:** Tooling, compiler, and workflow → `/tooling`
- **Examples & recipes:** Common patterns for modeling and simulation → `/recipes`