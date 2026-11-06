# Software design principles

[Software design and code quality](README.md) · [Clean code](clean-code.md) · [Design patterns](design-patterns.md)

Principles are prompts for evaluating a design under a particular kind of change. They are not independent laws and do not guarantee a good design when followed mechanically. A principle is useful when it clarifies responsibility, dependency direction, or a costly change; applying it without such pressure often adds indirection.

## SOLID

### S — Single Responsibility Principle

**Idea:** A module should have one coherent responsibility, often expressed as one primary reason or stakeholder/actor that drives its change.

This does not mean “one method per class” or “one thing” in the abstract. A module that handles policy, persistence, and report formatting may change for unrelated reasons; split only where ownership or change causes actually differ. Too many tiny classes can make control flow harder to follow.

**Ask:** Which business rule or actor causes this module to change? Are those changes independent enough to justify separate ownership?

### O — Open/Closed Principle

**Idea:** A stable part of the system should be open to required extension while avoiding repeated edits to code that should not change for that variation.

Interfaces, strategies, registries, plugins, and polymorphism can support extension, but each creates an extension point with a maintenance cost. The principle does not say “never modify existing code.” Prefer the direct, understandable implementation until a real axis of variation appears or repeated edits show a stable need.

**Ask:** What variation is expected, who owns it, and can it be added without making every caller branch on the new case?

### L — Liskov Substitution Principle

**Idea:** A subtype must be usable wherever its base type is expected without breaking the base type's behavioral contract.

Compatibility includes preconditions, postconditions, invariants, observable side effects, and error behavior—not just matching method signatures. A subtype that rejects inputs accepted by its base, returns weaker guarantees, or changes lifecycle semantics may violate substitutability. Prefer composition or separate interfaces if the supposed subtype cannot honor the contract.

**Ask:** Do callers relying on the base contract still work with every subtype, including edge cases and failure behavior?

### I — Interface Segregation Principle

**Idea:** Clients should not be forced to depend on operations they do not use; prefer focused, role-oriented interfaces over a “fat” interface.

Smaller contracts reduce unrelated coupling and make fakes/test doubles easier to implement. Too many fragmented interfaces can also make APIs difficult to discover. Split when different clients need different capabilities or change for different reasons, not just to reduce an interface's line count.

**Ask:** Which clients consume this contract, and do they all need the same operations and semantics?

### D — Dependency Inversion Principle

**Idea:** High-level policy should not be forced to depend directly on low-level implementation details; both should depend on an abstraction appropriate to the policy boundary.

The important part is a dependency direction that protects important rules from volatile details. An abstraction owned by the core may be implemented by an outer database, provider, or transport adapter. Do not add interfaces for every class: abstractions are useful when they create a meaningful boundary, substitution point, or test seam.

**Dependency injection** is a technique for supplying a dependency from outside a component. It can help implement dependency inversion, but the terms are not synonyms: injecting a concrete low-level detail does not automatically invert the design.

**Ask:** If an infrastructure library or vendor changed, would core policy need to change with it?

## Related principles

### DRY — Don't Repeat Yourself

Avoid duplicated knowledge that must be kept consistent. Repeated syntax is not necessarily repeated knowledge: two rules that happen to look alike today may evolve independently. Prematurely combining them creates conditional behavior and shared change risk. See [Clean code](clean-code.md#dry-kiss-and-yagni-without-cargo-culting).

### KISS — Keep It Simple

Prefer the least complex design that meets the current requirements and makes likely change safe. Include conceptual, operational, and debugging complexity in the cost; a short but clever expression may be less simple than a few explicit steps.

### YAGNI — You Aren't Gonna Need It

Do not implement speculative features, layers, or configuration knobs without a credible need. YAGNI does not mean ignoring known future requirements; it means distinguishing confirmed requirements from imagined flexibility.

### Composition over inheritance

Prefer composing collaborators when behavior must vary independently or be combined dynamically. Inheritance is still appropriate for a genuine substitutable subtype with a stable contract. Composition avoids some base-class coupling but can add wiring and forwarding; decide from the relationship and change pattern, not a blanket rule.

### Principle of least knowledge (Law of Demeter)

Avoid making a component traverse long chains of another component's object graph. Talk to direct collaborators through meaningful operations. This is not a ban on all chained syntax; the concern is coupling to internals and navigation paths that change independently.

## Applying principles together

| Situation | Principle that may help | Counter-check |
| --- | --- | --- |
| A class changes for unrelated stakeholder needs | SRP | Is the split a real ownership boundary or just a scattering of one workflow? |
| Every new provider requires edits across many callers | OCP / DIP | Is provider variation real, and where should its adapter/factory live? |
| A subtype cannot support part of its parent contract | LSP / ISP | Should the contract be narrower, or are these types not a subtype family? |
| Many clients depend on unused operations | ISP | Will separate contracts improve client clarity without fragmenting the API? |
| An abstraction exists only for a hypothetical extension | YAGNI / KISS | What evidence would justify it, and can it be introduced later safely? |
| Similar code is diverging in separate business rules | DRY | Is shared knowledge truly one rule, or merely similar syntax? |

## Practical decision test

Before introducing a new abstraction, write one sentence for each:

- **Change pressure:** what concrete change or failure is expensive today?
- **Boundary:** which module should own the rule, and which details should it not know?
- **Benefit:** what duplication, coupling, or invalid state will be removed?
- **Cost:** what new concept, indirection, configuration, or lifecycle must maintainers learn?
- **Evidence:** what test or example would show that the abstraction helps?

If these answers are unclear, a direct implementation is usually the better starting point. Refactor once requirements and variation are visible; see [safe refactoring](refactoring.md).

## References

- The original outline linked a Refactoring.Guru SOLID path, but the supplied URL (`/design-patterns/solid`) returned **404** when checked on **2026-11-06**. The accessible [Refactoring.Guru pattern catalog](https://refactoring.guru/design-patterns/catalog) remains useful for pattern concepts, but is not treated here as a verified SOLID page.
- [Microsoft's C# coding conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions) and [Google's review overview](https://google.github.io/eng-practices/review/) provide concrete coding and review guidance; neither is a language-independent specification of SOLID.
- [Ardalis Clean Architecture](https://github.com/ardalis/CleanArchitecture) is a practical .NET example referenced by the original outline. Treat any example architecture as one implementation, not as the definition of the principles.

