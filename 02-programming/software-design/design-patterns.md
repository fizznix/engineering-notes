# Design patterns

[Software design and code quality](README.md) · [Back to Programming](../README.md)

## What a pattern is—and is not

A design pattern is a named, reusable way to organize responsibilities for a recurring design problem. A useful pattern description includes its context, forces, participants, collaboration, consequences, and situations where it is a poor fit. It is not a snippet to paste into every project, a framework requirement, or proof that a design is good.

The Gang of Four (GoF) book presents **23 object-oriented patterns** in three intent-based families. The current Refactoring.Guru and Design Gurus code-pattern catalogs each describe **22**; those catalogs omit **Interpreter**. The local catalog below retains all 23 GoF patterns. Different modern-language references sometimes combine, rename, or reinterpret patterns; compare intent and trade-offs, not just class diagrams.

Patterns work at different scales:

- **Object/class patterns (GoF):** collaboration and object construction inside a program.
- **Enterprise/application patterns:** data access, domain logic, transactions, and web presentation.
- **Distributed/system patterns:** service failures, messaging, load, and data consistency.

These scales overlap in vocabulary but are not interchangeable. The application and distributed patterns are summarized in [Architecture patterns](architecture-patterns.md).

## Choosing a pattern from a problem

Use this sequence rather than looking for a place to use a named pattern:

1. State the problem as a change or constraint: for example, “adding a provider changes several call sites” or “a transient dependency failure exhausts all request workers.”
2. Find the variation point and identify which component should own it.
3. Check whether a simpler language feature, function, module boundary, or configuration is enough.
4. Select the pattern whose collaboration addresses that pressure; describe the new indirection it introduces.
5. Test the behavior and important edge cases before widening the abstraction.
6. Revisit the choice if variation never materializes or the abstraction becomes harder to understand than the original code.

### Quick problem-to-pattern guide

| Pressure in the code | Candidate | First question |
| --- | --- | --- |
| Creation logic varies by type or environment | Factory Method or Abstract Factory | Is there genuine variation, or is a direct constructor sufficient? |
| Many construction options or invalid partial objects | Builder | Can an immutable value object or named arguments make construction clear? |
| A dependency has an incompatible API | Adapter | Should the translation live at the integration boundary? |
| Two independent dimensions of change are tangled | Bridge | Are both dimensions expected to vary independently? |
| An algorithm varies while its caller remains stable | Strategy | Can a function value or simple conditional express it more clearly? |
| A state-dependent switch grows and transitions matter | State | Is this a real lifecycle with guarded transitions? |
| A request must be queued, audited, retried, or undone | Command | What persistence, identity, idempotency, and undo semantics are required? |
| Many listeners react to a change | Observer | Are event ordering, delivery guarantees, and unsubscribe behavior clear? |
| A complex subsystem has too many entry points | Facade | Does the facade simplify a real client workflow without becoming a god object? |
| Requests pass through optional processing stages | Decorator or Chain of Responsibility | Must every stage run, or can one handler stop/forward the request? |
| A remote dependency fails intermittently | Retry, timeout, circuit breaker, or bulkhead | Is the failure transient, and are retries safe and bounded? See [distributed patterns](architecture-patterns.md#resilience-and-messaging-patterns). |

## Creational patterns

These separate the decision or process of construction from the code that uses the created object. They are useful when creation is genuinely variable, coordinated, or complex; otherwise they can be ceremony around a constructor.

### Factory Method

**Intent:** define a creation operation while letting a specialized creator choose the concrete product. The creator can rely on a product interface rather than naming a concrete class.

Use it when subclasses or a plugin boundary need to vary the product while keeping a workflow stable. It is not merely any static function called `create`; a simple factory function is often enough. Avoid a hierarchy whose only purpose is hiding a one-line constructor.

### Abstract Factory

**Intent:** create a compatible family of related products without exposing their concrete classes to clients.

Use it when a runtime environment, platform, or theme selects several objects that must work together. Adding a new family is straightforward; adding a new product kind may require extending every factory. Prefer a simpler factory when there is only one product or no meaningful family invariant.

### Builder

**Intent:** construct a complex object in explicit steps, often separating construction from the final representation.

Use it when construction has meaningful phases, many optional choices, or invariants that must be validated before an object is usable. A builder can improve readability and prevent invalid partial state. It is overkill for a small value object with a clear constructor.

### Prototype

**Intent:** create an object by copying a configured exemplar instead of rebuilding it from scratch.

Use it when copying a configured template is simpler or cheaper than reconstructing one. Define whether copies are shallow or deep, which state is shared, and how identity/resources are handled. Avoid it when copying hides expensive work or aliases mutable state unexpectedly.

### Singleton

**Intent:** ensure a class has one instance and provide a global access point to it.

The pattern couples instance count, lifecycle, and access. It can be appropriate for a truly process-wide resource with an explicitly controlled lifecycle, but global mutable singletons hide dependencies, complicate isolation and tests, and can introduce concurrency problems. Prefer an application composition root or dependency injection when the goal is sharing one configured service.

## Structural patterns

These explain how objects or classes compose. Consider whether the composition boundary makes responsibilities easier to see, test, or replace.

### Adapter

**Intent:** translate one interface into the interface a client expects.

Use it at a vendor, legacy, or protocol boundary so domain code depends on a stable local contract. Keep translation and provider-specific error mapping in the adapter. Do not create an adapter when the two interfaces already align or when a direct rename is enough.

### Bridge

**Intent:** separate an abstraction from its implementation so both can vary independently.

Use it when two axes of change would otherwise produce a subclass cross-product (for example, report type × rendering backend). The abstraction delegates to an implementation interface. It adds indirection and is useful only when both axes really vary.

### Composite

**Intent:** let clients treat a leaf and a tree of objects uniformly through a shared component contract.

Use it for recursive structures such as syntax trees, UI trees, and nested orders. Decide which operations make sense for leaves and composites; a broad interface can otherwise force meaningless methods or runtime checks.

### Decorator

**Intent:** wrap an object with another object that implements the same interface, adding behavior before/after delegating.

Use it to compose optional behavior such as metrics, compression, or caching without multiplying subclasses. Wrapping order matters, and many wrappers can obscure runtime behavior. Avoid hidden side effects or a decorator stack that is harder to inspect than the original component.

### Facade

**Intent:** provide a simpler, cohesive entry point to a subsystem.

Use it to guide common client workflows and hide coordination among subsystem components. A facade does not have to eliminate lower-level access. Keep it focused; if unrelated workflows accumulate in it, split the boundary rather than creating a new god object.

### Flyweight

**Intent:** share intrinsic, reusable state among many fine-grained objects while keeping context-specific extrinsic state outside the shared object.

Use it only when profiling shows that duplicated object state is a material memory cost. The split between shared and per-instance data adds complexity, and shared state must be immutable or safely synchronized.

### Proxy

**Intent:** provide a substitute that controls access to another object while preserving its interface.

Common uses include lazy loading, remote access, access checks, and lifecycle management. A proxy may delay or mediate a call but should preserve the contract clients rely on. Be explicit about network latency, failures, retries, and security when the proxy is remote; it is not a substitute for documenting those costs.

## Behavioral patterns

These describe how algorithms and responsibilities are assigned and how objects communicate. Make ordering, ownership, and failure semantics explicit.

### Chain of Responsibility

**Intent:** pass a request through a sequence of handlers; each handler can process it, stop it, or forward it.

Useful for middleware, validation, and routing pipelines where handlers are independently composed. Define order, short-circuit behavior, and what happens when no handler accepts the request. If every stage must run in a fixed sequence, an explicit pipeline may be clearer.

### Command

**Intent:** encapsulate a request as an object containing the operation and required data, separating invocation from execution.

Useful for queues, scheduling, audit trails, retries, and undo/redo. Reliable use requires decisions about command identity, serialization/versioning, idempotency, authorization, and compensation. “Undo” is not automatic: the operation must preserve enough information and define a valid inverse.

### Interpreter

**Intent:** represent a small language's grammar and interpret sentences in that language.

Useful for a compact, stable grammar such as a small rule or expression language. A class per grammar rule can become unwieldy as grammar complexity grows; use a parser or compiler tool when the language is substantial.

### Iterator

**Intent:** traverse a collection without exposing its internal representation.

Most languages provide iterators as a core abstraction. Implement a custom one when traversal state or order is meaningful, such as a tree walk; do not rebuild language-provided collection iteration without a need.

### Mediator

**Intent:** centralize coordination among collaborating objects so they do not depend directly on each other.

Useful when peer-to-peer references form a complex interaction web. A mediator can reduce coupling among participants but can itself become a central god object; keep its policy bounded and observable.

### Memento

**Intent:** capture and restore an object's state without exposing its internal representation.

Useful for undo, checkpoints, and snapshots. Choose snapshot boundaries, storage cost, versioning, and privacy carefully. For large state, incremental history or event-based reconstruction may fit better.

### Observer

**Intent:** notify registered dependents when a subject changes.

Useful when one-to-many notification should not make the subject know every consumer. Specify subscription lifecycle, delivery ordering, error isolation, threading, and whether notifications are synchronous. For durable or cross-process delivery, use a messaging design rather than assuming an in-memory observer is a message broker.

### State

**Intent:** represent state-specific behavior in separate state objects and delegate behavior according to the current state.

Useful when a lifecycle has meaningful transitions and the same operation has different valid behavior in each state. Centralize transition rules and invariant checks. Avoid dozens of state classes for a tiny stable conditional.

### Strategy

**Intent:** encapsulate a family of interchangeable algorithms behind a common contract and let a context delegate to a selected algorithm.

Useful when algorithms vary independently of the caller, such as pricing, routing, or retry policy. Selection usually belongs to the caller/composition layer, not the strategy context. A function parameter or map of functions may be simpler in languages with first-class functions.

### Template Method

**Intent:** define an algorithm's fixed skeleton in a base class and let subclasses override selected steps.

Useful when a workflow's order is invariant but a few steps vary. It relies on inheritance and can create fragile base-class coupling. Prefer composition/Strategy when variants need to be combined or chosen at runtime.

### Visitor

**Intent:** add operations over a stable set of element types without placing each operation's implementation on the elements.

Useful for compiler trees and other object structures where element kinds are stable but operations are added often. Adding a new element type then changes all visitors; a simple method or pattern matching may be more appropriate when the type set is small.

## Frequently confused patterns

| Pair | Difference |
| --- | --- |
| Strategy vs State | Strategy selects an interchangeable algorithm; State models a lifecycle in which valid behavior and transitions depend on current state. |
| Factory Method vs Abstract Factory | Factory Method delegates a product decision, commonly through specialization; Abstract Factory creates a compatible family of products. |
| Factory vs Builder | A factory selects/returns a product; a builder represents a multi-step construction process. |
| Adapter vs Facade | Adapter changes an interface to match a client contract; Facade offers a simpler interface to a subsystem. |
| Decorator vs Proxy | Decorator composes additional behavior; Proxy controls access, lifecycle, or location while preserving the apparent subject contract. The structure can look similar, so intent and guarantees matter. |
| Observer vs Mediator | Observer broadcasts change notifications; Mediator coordinates interactions among peers. |
| Template Method vs Strategy | Template Method varies steps through inheritance; Strategy varies a whole algorithm through composition. |
| Composite vs Decorator | Composite represents a part-whole tree; Decorator wraps one object to add behavior. |
| Command vs Strategy | Command represents a request/invocation; Strategy represents the algorithm used to perform a task. A command may use a strategy. |

## Beyond GoF

These are useful names, but they are not additional GoF object patterns:

| Name | Primary concern |
| --- | --- |
| Dependency injection | Supplying dependencies from a composition boundary rather than constructing them invisibly inside a consumer. |
| Repository / Data Access Object | Hiding data-access mechanics behind an application- or domain-facing contract; the two names imply different abstraction goals. |
| Unit of Work | Tracking and coordinating a set of changes that should be committed together. |
| Service Layer | Providing an application boundary for operations and coordinating domain/infrastructure work. |
| Model–View–Controller | Separating input/control flow, presentation, and model responsibilities in an interface application. |
| Retry, circuit breaker, bulkhead, outbox, saga | Distributed-systems mechanisms with failure, consistency, and operational trade-offs; see [Architecture patterns](architecture-patterns.md). |

## Game-oriented, sequencing, and optimization patterns

[Game Programming Patterns](https://gameprogrammingpatterns.com/contents.html) includes GoF patterns in game contexts plus patterns for sequencing, decoupling, and performance. These chapter summaries add the non-GoF topics to the reference; prefer the book's examples for details of implementation.

| Chapter | Core idea | When it helps / main caution |
| --- | --- | --- |
| [Command](https://gameprogrammingpatterns.com/command.html) | Represent an action as a value that can be invoked later. | Useful for input remapping, queues, and undo; state capture and undo semantics are not automatic. See [Command](#command). |
| [Flyweight](https://gameprogrammingpatterns.com/flyweight.html) | Share common immutable data across many logical objects. | Useful when many instances repeat expensive data; profile first and keep per-instance state separate. See [Flyweight](#flyweight). |
| [Observer](https://gameprogrammingpatterns.com/observer.html) | Notify interested listeners when a subject changes. | Useful for one-to-many notification; lifecycle, ordering, and error propagation must be defined. See [Observer](#observer). |
| [Prototype](https://gameprogrammingpatterns.com/prototype.html) | Create an instance from a configured exemplar. | Useful for data-driven entities and avoiding repetitive setup; define copy depth and ownership. See [Prototype](#prototype). |
| [Singleton](https://gameprogrammingpatterns.com/singleton.html) | Provide one shared service instance. | A possible lifecycle mechanism, but global access hides dependencies and makes tests/state harder to isolate. See [Singleton](#singleton). |
| [State](https://gameprogrammingpatterns.com/state.html) | Represent behavior that changes with an object's current state. | Useful for entities with explicit lifecycle transitions; a few stable branches do not need a class hierarchy. See [State](#state). |
| [Double Buffer](https://gameprogrammingpatterns.com/double-buffer.html) | Write a complete next result separately, then swap which result is visible. | Prevents readers from observing a partially updated frame/snapshot; costs extra storage and requires a clear swap/synchronization point. |
| [Game Loop](https://gameprogrammingpatterns.com/game-loop.html) | Repeatedly process input, update simulation, and render, decoupling progress from one user action. | Useful for interactive real-time programs; timestep, frame pacing, responsiveness, and determinism are distinct requirements. |
| [Update Method](https://gameprogrammingpatterns.com/update-method.html) | Advance each active object's behavior by one step per outer-loop iteration. | Allows many entities to make progress while input/rendering continue; update order and shared-state interactions become observable. |
| [Bytecode](https://gameprogrammingpatterns.com/bytecode.html) | Encode behavior as instructions interpreted by a small virtual machine. | Useful when behavior must be data-driven, edited independently, or sandboxed; language tooling, debugging, versioning, and VM complexity are substantial costs. |
| [Subclass Sandbox](https://gameprogrammingpatterns.com/subclass-sandbox.html) | Let subclasses define behavior using a controlled set of operations supplied by a base class. | Can keep extension code within a supported API; base-class changes and inheritance coupling can affect many subclasses. |
| [Type Object](https://gameprogrammingpatterns.com/type-object.html) | Represent a kind/type as data so many instances can share configurable type properties. | Useful for runtime-created or data-driven variants; distinguish shared definition from per-instance mutable state and validate loaded definitions. |
| [Component](https://gameprogrammingpatterns.com/component.html) | Split one entity's responsibilities into components so domains can vary independently. | Reduces a giant entity's coupling; introduces composition, lifecycle, and inter-component coordination that must be designed. |
| [Event Queue](https://gameprogrammingpatterns.com/event-queue.html) | Queue a message so sending and processing happen at different times. | Useful for decoupled/asynchronous work and bursts; delayed/stale events, ordering, backpressure, duplicates, and delivery guarantees matter. |
| [Service Locator](https://gameprogrammingpatterns.com/service-locator.html) | Look up a shared service through a central registry rather than naming its concrete implementation at each call site. | Can centralize service replacement, but makes dependencies implicit/global; explicit constructor injection is often easier to test and understand. |
| [Data Locality](https://gameprogrammingpatterns.com/data-locality.html) | Arrange frequently accessed data so processing benefits from CPU caches and fewer pointer-chasing loads. | Useful after profiling shows memory access limits throughput; data layout can constrain flexibility and should be benchmarked on target hardware. |
| [Dirty Flag](https://gameprogrammingpatterns.com/dirty-flag.html) | Mark derived data stale when inputs change, then recompute only when the result is next needed. | Avoids repeated work when inputs change more often than results are read; invalidation must cover every dependency or stale data leaks. |
| [Object Pool](https://gameprogrammingpatterns.com/object-pool.html) | Reuse objects from a bounded pool instead of repeatedly allocating and freeing them. | Useful for measured allocation/fragmentation pressure; reset every reusable field, bound retained capacity, and compare against modern allocator/GC behavior. |
| [Spatial Partition](https://gameprogrammingpatterns.com/spatial-partition.html) | Index objects by position so nearby-object queries avoid scanning every object. | Useful for repeated neighborhood queries at scale; movement/update costs and index maintenance can outweigh query savings for small or highly dynamic sets. |

## Sources and sublinks

The notes above are an original synthesis. These sources were inspected on **2026-11-06**:

- [Refactoring.Guru: design-pattern overview](https://refactoring.guru/design-patterns) links to [what a pattern is](https://refactoring.guru/design-patterns/what-is-pattern), [catalog](https://refactoring.guru/design-patterns/catalog), [classification](https://refactoring.guru/design-patterns/classification), [benefits](https://refactoring.guru/design-patterns/why-learn-patterns), [criticism](https://refactoring.guru/design-patterns/criticism), and the [creational](https://refactoring.guru/design-patterns/creational-patterns), [structural](https://refactoring.guru/design-patterns/structural-patterns), and [behavioral](https://refactoring.guru/design-patterns/behavioral-patterns) groups. Its current catalog says 22 patterns. The [Strategy detail page](https://refactoring.guru/design-patterns/strategy) demonstrates the problem → solution → structure → pseudocode → applicability format.
- [Patterns Guru: Software patterns](https://patterns.guru/) presents all 23 original GoF patterns and discusses their translation into modern languages.
- [Refactoring.Guru's Python examples](https://github.com/RefactoringGuru/design-patterns-python) contains conceptual and real-world examples for the classic GoF patterns. The repository states that examples require Python 3.7 or later and is licensed CC BY-NC-ND 4.0; use it as an implementation reference rather than copying its examples into this knowledge base.
- [Design Gurus pattern repository](https://github.com/design-gurus/design-patterns) describes a 22-entry code-pattern set and 60+ system patterns. Its [code-pattern index](https://github.com/design-gurus/design-patterns/tree/main/code-patterns), [examples](https://github.com/design-gurus/design-patterns/tree/main/examples), and [system-pattern index](https://github.com/design-gurus/design-patterns/tree/main/system-patterns) are navigable subcollections; the repo's catalog count differs from the original GoF count.
- [Game Programming Patterns](https://gameprogrammingpatterns.com/) makes its book available in full on the web. Its [contents](https://gameprogrammingpatterns.com/contents.html) groups 19 named pattern chapters into design-pattern revisits, sequencing, behavioral, decoupling, and optimization patterns. This note summarizes the chapter intents and decision trade-offs rather than reproducing the book's examples. The game-oriented examples are useful beyond games, but performance-oriented patterns should be justified by workload evidence.
- [Head First Design Patterns, 2nd edition](https://www.oreilly.com/library/view/head-first-design/9781492077992/) is a book-level learning resource; its publisher page was not readable during this review, so no book text is summarized here.

The original outline also linked a Refactoring.Guru SOLID URL that returned **404** when checked. Use the currently accessible overview/catalog links above rather than assuming that old route is valid.
