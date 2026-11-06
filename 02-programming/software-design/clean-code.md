# Clean code and maintainable practices

[Software design and code quality](README.md) · [Principles](principles.md) · [Refactoring](refactoring.md)

## What “clean” means in practice

Clean code is code whose behavior and intent are inexpensive for another maintainer to understand and change. This is an engineering outcome, not a universal aesthetic or a fixed line-count rule. Readability matters because code is read, reviewed, debugged, tested, and changed repeatedly; consistency with the local project also matters.

The target is not the shortest code, the most abstractions, or the largest test count. Aim for clear behavior, appropriate boundaries, explicit failure modes, and enough tests to make change safe. Balance local clarity against duplication and system-wide consistency.

## Practical rules

### Names communicate role and intent

- Name values, functions, types, and modules for the domain concept or action they represent. Prefer `retry_policy`, `invoice_total`, or `load_customer` over vague names such as `data`, `value`, or `process`.
- Use domain terminology consistently. Avoid abbreviations unless they are established in the team and domain.
- Distinguish concepts that differ; do not invent multiple names for the same concept without a reason.
- Boolean names should read as predicates (`is_expired`, `can_retry`). A function name should make side effects or resource lifecycle apparent when it is not obvious.
- Do not compensate for a vague name with a comment that repeats the implementation.

### Functions and modules expose one coherent idea

- Give a function a single cohesive purpose and a name that describes its effect. Short functions can still be confusing if they hide control flow or split one idea arbitrarily.
- Keep decisions near the data and rules they govern. Extract a helper when it clarifies a meaningful step, creates a reusable policy, or isolates a boundary—not merely to meet an arbitrary line count.
- Prefer a small number of meaningful parameters. Use a typed parameter object when several values form one concept or must be validated together; do not create a wrapper for every pair of arguments.
- Keep side effects visible. Separate querying from commands where it makes behavior easier to reason about; document unavoidable transactional or external effects in the contract.
- Make invariants explicit and establish them at a clear boundary rather than scattering defensive checks through every caller.

### State and dependencies remain understandable

- Keep mutable state limited and owned. Prefer immutable values when they clarify identity and lifecycle.
- Pass required dependencies explicitly at a stable composition boundary. Avoid ambient global state and service locators that hide what a component needs.
- Make resource ownership, cleanup, timeout, cancellation, and thread-safety expectations part of the API contract.
- Keep domain rules independent of transport, persistence, clock, and vendor details when those details change independently. Do not create layers where the dependency is stable and local.

### Errors are explicit and useful

- Distinguish invalid input, expected business outcomes, transient infrastructure failures, and programmer errors.
- Validate untrusted input at the boundary, then use types and invariants to avoid repeated defensive checks deeper in the application.
- Propagate, translate, or handle failures intentionally. A catch-all that logs and continues can turn failure into corrupted success-shaped output.
- Include useful context in errors while avoiding secrets, tokens, sensitive personal data, and unnecessarily verbose internals.
- Retry only failures that may resolve, with bounded attempts, timeouts, backoff, jitter where appropriate, and idempotent semantics. See [resilience patterns](architecture-patterns.md#resilience-and-messaging-patterns).

### Tests describe observable behavior

- Test contracts and decisions rather than private implementation details.
- Use unit tests for deterministic local rules, integration tests for important boundaries, and end-to-end tests for representative user-critical paths.
- Include failure paths, boundary values, and state transitions—not only the happy path.
- Keep tests deterministic and isolated; control time, randomness, network, and shared state where they affect results.
- Test names and fixtures should explain why a behavior matters. Avoid brittle tests that only confirm the current internal call sequence.
- A test suite is a safety mechanism, not a proof of correctness. Combine tests with review, static analysis, and production observability as appropriate.

### Comments and documentation explain the missing context

- Explain *why* a non-obvious choice exists, what invariant must hold, or what external contract constrains the implementation.
- Keep comments current; stale comments are worse than no comments.
- Document public contracts, operational requirements, compatibility constraints, security assumptions, and failure behavior.
- Prefer expressive names and simpler control flow over comments that narrate each line.
- Put durable architectural rationale in an Architecture Decision Record or design note when the decision is cross-cutting and likely to be revisited.

### Formatting and conventions are shared infrastructure

- Follow the formatter, linter, naming rules, and language conventions already adopted by the project.
- Automate mechanical style checks; use review for behavior, design, tests, risk, and clarity.
- Keep changes cohesive and reviewable. Separate unrelated cleanup from a functional change unless the cleanup is required to make the change safe.

Language-specific rules are not universal design principles. For C#, see [Microsoft's .NET coding conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions); apply the equivalent official/project guidance for the language in use.

## DRY, KISS, and YAGNI without cargo-culting

- **DRY — Don't Repeat Yourself:** reduce duplicated *knowledge* that must change together. Two similar code blocks are not necessarily duplicate knowledge; unifying them can create a brittle abstraction if their reasons to change differ.
- **KISS — Keep It Simple:** prefer the design with the least accidental complexity that still meets the real requirements. Simplicity is about concepts and failure modes, not minimizing characters.
- **YAGNI — You Aren't Gonna Need It:** avoid speculative features and extension points without a present requirement or credible near-term pressure. Preserve a clear route to future change without implementing imaginary behavior.

These rules complement [SOLID and other principles](principles.md); they can conflict in a particular situation. Use evidence from current change patterns and operating constraints to decide.

## Engineering judgment beyond style rules

The ideas highlighted by *The Pragmatic Programmer* can be used as practical checks:

- **Orthogonality:** keep components independent enough that a change in one area does not cause unrelated changes elsewhere. Shared state and hidden cross-module assumptions reduce this independence.
- **Reversibility:** when information is incomplete, avoid locking the design into a costly-to-change decision earlier than necessary. This does not mean making every decision configurable; it means keeping a reasonable exit where uncertainty and change cost are high.
- **Tracer bullets:** build a thin, working path through important layers early to test assumptions about integration and requirements. A tracer slice is not a throwaway prototype; it should reveal what the real system will need to connect.
- **Avoid programming by coincidence:** understand why a behavior works and make its dependencies explicit. A test that passes accidentally, undocumented ordering, or unexplained environment assumption is a warning to investigate.
- **Own the outcome:** validate the result and communicate known constraints. Passing a compiler or formatter is evidence about only the properties that tool checks.

## A compact code-review checklist

1. Can a reader identify what this change does and why from the names, boundaries, and tests?
2. Is the behavior correct for normal, invalid, boundary, and failure cases?
3. Is the simplest appropriate design used, with dependencies and side effects visible?
4. Are errors handled intentionally, and are retries/timeouts safe?
5. Do tests check useful behavior without overfitting implementation details?
6. Are security, privacy, compatibility, and performance implications addressed where relevant?
7. Is relevant documentation updated, and is the change small enough to review?

Google's review guides provide a useful concrete counterpart to this checklist. Their [overview](https://google.github.io/eng-practices/review/) says reviewers consider design, functionality, complexity, tests, naming, comments, style, and documentation. The [reviewer guide](https://google.github.io/eng-practices/review/reviewer/) links to [the review standard](https://google.github.io/eng-practices/review/reviewer/standard.html), [what to look for](https://google.github.io/eng-practices/review/reviewer/looking-for.html), [navigating a change](https://google.github.io/eng-practices/review/reviewer/navigate.html), [review speed](https://google.github.io/eng-practices/review/reviewer/speed.html), [writing comments](https://google.github.io/eng-practices/review/reviewer/comments.html), and [handling pushback](https://google.github.io/eng-practices/review/reviewer/pushback.html). The [change-author guide](https://google.github.io/eng-practices/review/developer/) covers [change descriptions](https://google.github.io/eng-practices/review/developer/cl-descriptions.html), [small changes](https://google.github.io/eng-practices/review/developer/small-cls.html), and [handling review comments](https://google.github.io/eng-practices/review/developer/handling-comments.html). These are Google's practices, not a universal policy every team must copy.

## Sources and limitations

Inspected on **2026-11-06**:

- [Google Engineering Practices](https://google.github.io/eng-practices/) publishes a code-review overview with subguides for reviewers and change authors. The notes here summarize the overview's review dimensions; the guides provide concrete organizational process and examples.
- [Microsoft .NET Coding Conventions for C#](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions) states that its conventions are examples rather than an authoritative universal list; it points to the [.NET runtime style guide](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md) and [Roslyn contributor guide](https://github.com/dotnet/roslyn/blob/main/CONTRIBUTING.md#csharp).
- [The Pragmatic Programmer, 20th Anniversary Edition](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) describes a language-independent treatment of maintainability, avoiding duplicated knowledge, testing, and adapting software. Its online book page was inspected.
- [Clean Code](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) remains a book reference from the original outline; the publisher page returned **403** during this review, so this note does not claim to summarize its chapters.
- For behavior-preserving changes, see Martin Fowler's [Refactoring book page](https://martinfowler.com/books/refactoring.html) and the local [refactoring guide](refactoring.md).
