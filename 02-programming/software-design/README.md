# Software design and code quality

[Back to Programming](../README.md) · [Back to repository index](../../README.md)

This package is a self-contained study reference for writing and evolving maintainable software. It brings together the design-pattern, clean-code, refactoring, and architecture topics from the original [`design-pattern.md`](../../design-pattern.md) outline, with the repeated material organized into focused notes.

## Start here

1. [Clean code](clean-code.md): readability, local reasoning, error handling, testing, and review.
2. [Principles](principles.md): SOLID, DRY, KISS, YAGNI, and composition with their trade-offs.
3. [Design patterns](design-patterns.md): the complete 23-pattern Gang of Four catalog, comparisons, and selection guidance.
4. [Refactoring](refactoring.md): code smells, behavior-preserving transformations, and safe workflow.
5. [Architecture patterns](architecture-patterns.md): application boundaries, enterprise patterns, and distributed-system resilience patterns.

These are not a sequence of mandatory techniques. Start from a concrete maintenance or change problem, then use the smallest design that resolves it. Patterns and principles are vocabulary for reasoning, not goals or compliance checklists.

## Suggested six-week practice path

| Week | Focus | Practice |
| --- | --- | --- |
| 1 | Clean code and tests | Improve naming and function boundaries in a small existing module; preserve behavior with tests. |
| 2 | SOLID and related principles | Find one actual change seam, then compare a direct implementation with a small abstraction. |
| 3 | Creational and structural patterns | Implement a factory, builder, adapter, and decorator only where their problems occur. |
| 4 | Behavioral patterns | Compare strategy/state, observer/mediator, and command/chain using tested examples. |
| 5 | Refactoring and architecture | Refactor a code smell in small steps; map the module's dependencies and boundaries. |
| 6 | Applied project | Build a small provider-agnostic client or notification service; record decisions, tests, and costs of abstractions. |

The schedule is a guide, not a prerequisite order. A suitable project should be small enough to test and operate locally. Do not start with microservices or a framework-heavy architecture just to demonstrate patterns.

## Practice project ideas

- **Multi-provider model client:** use an adapter for each provider and a strategy for a real routing policy; add bounded retries only after defining timeout and idempotency behavior.
- **Notification service:** compare direct calls, in-process observers, and queued events; make ordering, duplicate delivery, and subscriber failures explicit.
- **Payment/workflow simulator:** model valid state transitions and commands; add a saga only if the workflow crosses independently committed services.
- **File-processing pipeline:** separate format-specific decoding from shared stages; compare a strategy, handler chain, and explicit pipeline.
- **Cache-backed catalog:** implement cache-aside with a stated staleness policy, then measure whether the cache helps.

The original outline's LLM gateway is a useful capstone, not a requirement: provider adapters and routing can teach object design, while reliability and distributed-pattern work should be a separate increment with tests for its failure semantics.

## How to study a pattern

For each candidate, write down:

1. The observed change or design pressure (not the pattern name).
2. The current implementation and its measurable pain: duplication, coupling, failure mode, or test difficulty.
3. The pattern's participants and the direction in which dependencies move.
4. The benefit and the new costs: indirection, configuration, lifecycle, runtime behavior, or concepts future maintainers must learn.
5. A simpler alternative, including doing nothing.
6. Tests that establish behavior before and after the change.

## Related notes

- [System design](../../04-system-design/README.md) covers service-level concerns beyond this programming-focused package.
- [Projects and labs](../../13-projects-labs/README.md) is the home for runnable implementation studies.
- [Security](../../11-security/README.md) provides the security context for validation, identity, authorization, and safe error handling.
