# Refactoring and code smells

[Software design and code quality](README.md) · [Clean code](clean-code.md) · [Architecture patterns](architecture-patterns.md)

## Refactoring

Refactoring is a disciplined change to the internal structure of software that preserves its externally observable behavior. It makes later work easier; it is not a synonym for rewriting, feature work, or changing behavior while cleaning up.

Martin Fowler describes refactoring as a sequence of small, behavior-preserving transformations. Small steps keep the software working and reduce the distance between a regression and the change that caused it.

## A safe workflow

1. **Name the goal.** Identify the specific smell, change pressure, or boundary to improve. State the behavior that must remain unchanged.
2. **Establish a safety net.** Run the relevant tests. Add focused tests for missing behavior; use characterization tests when legacy behavior is poorly documented. Do not encode a suspected bug as desired behavior without deciding whether it is part of the contract.
3. **Change one structure at a time.** Use IDE-supported rename/extract/move operations where reliable. Keep functional behavior changes separate when possible.
4. **Run targeted checks after each meaningful step.** Re-run focused tests, formatters, static analysis, and broader tests as warranted. Inspect failures rather than treating a passing build as proof.
5. **Keep the diff reviewable.** Preserve useful history, avoid unrelated formatting churn, and document non-obvious decisions.
6. **Check the result against the goal.** Is the design easier to understand or change? Did coupling, state, or failure behavior become less visible? Revert the refactor if complexity increased without a compensating benefit.

When tests are absent or expensive, make the smallest safe change, gather additional evidence (logging, manual reproduction, contract checks), and state the residual risk. Do not silently claim behavior preservation.

## Common code smells and useful responses

A smell is a clue to investigate, not proof that code is wrong. First understand its context and change history; then select the smallest refactoring that addresses the underlying cause.

| Smell | Why it can hurt | Candidate response | Caution |
| --- | --- | --- | --- |
| Long Method / Function | Hides steps and decisions in one unit | Extract a cohesive helper; replace temporary variables with named values | Do not fragment a clear linear algorithm into trivial wrappers. |
| Large Class / God Object | Mixes unrelated state, policy, and change ownership | Extract a focused responsibility or collaborator | Keep cohesive workflows together; avoid arbitrary “one class per method.” |
| Duplicated Knowledge | A rule must be changed consistently in several places | Consolidate the shared rule behind one owner | Similar-looking code may encode rules that should evolve separately. |
| Long Parameter List | Call sites and function intent are hard to read | Introduce a parameter object for a real concept; group options by responsibility | A data bag can just move the long list and create weak typing. |
| Data Clumps | The same values travel together across APIs | Introduce a cohesive value type or domain object | Do not wrap unrelated values solely because they co-occur once. |
| Primitive Obsession | Strings/numbers encode domain concepts and invalid states | Add a value object/type with validation and behavior | Avoid types that provide no invariant or semantic benefit. |
| Feature Envy | A method repeatedly reaches into another object's data | Move behavior to the object that owns the data/rule, or introduce a domain service | Data locality is not absolute; domain boundaries may intentionally keep behavior elsewhere. |
| Shotgun Surgery | One rule requires edits in many locations | Move related behavior behind a clear owner; consolidate scattered policy | Multiple edits may be appropriate when components are intentionally independent. |
| Divergent Change | One module changes for unrelated causes | Separate responsibilities along real change/ownership boundaries | Avoid predicting every possible future responsibility. |
| Deep Nesting / Complex Conditionals | Control flow is hard to follow and edge cases are obscured | Guard clauses, named predicates, or a state/strategy model when variation is stable | Polymorphism is not automatically clearer than a small conditional. |
| Repeated Type Switch | New variants require edits across consumers | Strategy, State, or polymorphism when the type family and variation are stable | Closed data models and pattern matching can be simpler and safer. |
| Speculative Generality | Unused hooks/interfaces/configuration add concepts | Inline or remove unused flexibility; add extension points when needed | Preserve explicit compatibility or roadmap requirements. |
| Comments explaining tangled code | The comment must compensate for unclear structure | Clarify names/control flow; retain comments that explain rationale or constraints | Comments are not inherently a smell. |
| Hidden temporal coupling | Calls must occur in a fragile order | Make lifecycle/state explicit; provide a cohesive operation | Some ordering is inherent; document and enforce it rather than abstracting blindly. |

## Common behavior-preserving transformations

- **Extract Function / Method:** name a coherent sequence and replace it with a call. Keep inputs and outputs explicit.
- **Inline Function:** remove an unnecessary indirection when the body is clearer at the call site.
- **Rename:** make the domain meaning or side effect accurate at all call sites.
- **Move Function / Field:** place behavior or state with the data/rule it belongs to, respecting module boundaries.
- **Introduce Parameter Object:** group related values and validate their invariant in one place.
- **Replace Nested Conditional with Guard Clauses:** return or fail early for exceptional cases, preserving the original order and side effects.
- **Encapsulate Variable:** replace direct mutation with an API that can maintain an invariant.
- **Substitute Algorithm:** replace an implementation with a clearer one while retaining the same contract.
- **Replace Conditional with Polymorphism / Strategy:** isolate stable variations when the conditional is growing and policies are independently meaningful.
- **Extract Class / Module:** separate a cohesive responsibility whose changes, state, or lifecycle differ from the original.

The name of a refactoring does not make it safe. Check preconditions and side effects; use tests to confirm equivalence.

## Testing during refactoring

- Start with externally observable behavior and contracts, not the internal arrangement you intend to change.
- Include boundary, error, and state-transition cases relevant to the refactor.
- Use a characterization test when legacy behavior must be preserved but is not specified; review surprising behavior before locking it in.
- Keep integration tests around important persistence, network, and framework seams; not every refactoring can be safely verified by isolated unit tests alone.
- Make a separate bug fix when behavior should change; explicitly update tests to reflect the corrected contract.

## Sources and sublinks

Inspected on **2026-11-06**:

- [Martin Fowler, Refactoring: Improving the Design of Existing Code](https://martinfowler.com/books/refactoring.html) describes refactoring as small behavior-preserving transformations and its second edition includes a catalog of techniques, code smells, and testing context.
- [Refactoring.Guru: Refactoring](https://refactoring.guru/refactoring) links to [the refactoring process](https://refactoring.guru/refactoring/how-to), [code smells](https://refactoring.guru/refactoring/smells), and [techniques](https://refactoring.guru/refactoring/techniques). Its smell index includes groups such as [Bloaters](https://refactoring.guru/refactoring/smells/bloaters), with [Long Method](https://refactoring.guru/smells/long-method), [Large Class](https://refactoring.guru/smells/large-class), [Primitive Obsession](https://refactoring.guru/smells/primitive-obsession), and [Long Parameter List](https://refactoring.guru/smells/long-parameter-list). Its technique index includes [Extract Method](https://refactoring.guru/extract-method), [Inline Method](https://refactoring.guru/inline-method), and [Substitute Algorithm](https://refactoring.guru/substitute-algorithm).
- [Google reviewer guide](https://google.github.io/eng-practices/review/reviewer/) and [author guide](https://google.github.io/eng-practices/review/developer/) provide related review and small-change practices; see [Clean code](clean-code.md#sources-and-limitations).

