# Application and system design patterns

[Software design and code quality](README.md) · [Design patterns](design-patterns.md) · [System design](../../04-system-design/README.md)

## Scope and vocabulary

GoF patterns organize object collaboration; architecture patterns organize application boundaries and dependencies; distributed-system patterns address multiple processes, unreliable networks, and independent data stores. The word “pattern” alone does not say which scale is meant.

Architecture is not a diagram or a folder layout. It is a set of important decisions and constraints that shape change, reliability, security, and operation. Select a pattern based on the forces in the system and evaluate its trade-offs. Many architecture patterns solve different problems and may be combined; they are not an all-or-nothing menu.

## Application architecture

### Layered architecture

Organizes responsibilities into layers, commonly presentation/API, application/domain, and infrastructure/data. A layer may call a lower layer through a defined interface. This is easy to understand and supports conventional applications, but unrestricted cross-layer calls or a “database-first” dependency direction can make business rules depend on infrastructure.

Choose and document whether layers are strict (only adjacent calls) or relaxed. A layer is useful when it provides a real boundary, not just because every project needs three folders.

### Hexagonal architecture (Ports and Adapters)

Places application/domain behavior behind ports that express what it needs or offers; adapters translate those contracts to external technologies such as HTTP, a database, a queue, or a test double. The core owns the policy-facing abstraction; infrastructure depends inward and implements it.

Useful when external systems vary or the application behavior should be testable independently. Ports should reflect meaningful application capabilities, not mirror every vendor API. Hexagonal, onion, and clean architecture share dependency-inward ideas but are not identical recipes; avoid duplicating layers merely to match terminology.

### Clean Architecture

Organizes dependencies so policy and use cases do not depend on volatile delivery or persistence details. Outer layers call inward; boundary interfaces and data-transfer shapes keep framework or database details from leaking into inner policy.

Useful for long-lived systems with substantial domain rules and multiple changing delivery/infrastructure concerns. It carries a cost in mapping, indirection, and wiring; a small CRUD service may be clearer with fewer layers. “Clean Architecture” is not a prescribed set of folder names.

### Modular monolith

A single deployable application contains modules with explicit responsibilities and controlled dependencies. Modules can own their APIs and data access while sharing an in-process runtime and often a transaction boundary.

Useful when the system needs internal separation but independent deployment or distributed operations are not yet justified. Enforce module boundaries (dependency rules, ownership, tests); a single binary with unrestricted shared tables and imports is not meaningfully modular.

### Microservices

Independent services own capabilities and can be deployed/scaled independently, usually with explicit network APIs and separately managed data. This can align deployment and team ownership with business boundaries, but introduces network failure, partial availability, observability, security, versioning, operational cost, and distributed data consistency.

Choose them when independent delivery, scale, fault isolation, or ownership outweighs these costs. Splitting a system by technical layer or anticipated traffic alone often produces a distributed monolith. Start with well-defined modules if the need for independent deployment is unproven.

### Event-driven architecture

Producers communicate that something happened through events; consumers react asynchronously, often through a broker. This reduces direct temporal coupling and helps absorb bursts, but moves complexity to delivery guarantees, ordering, duplicates, schema evolution, tracing, retries, and eventual consistency.

Distinguish a **notification** (“an order was placed”) from a **command** (“place this order”) and a **state-transfer event**. Define ownership, payload sensitivity, retention, replay, and compatibility. Use synchronous request/response where an immediate result is required and simpler.

### Domain-Driven Design

DDD is an approach for managing domain complexity, not a single deployment architecture. Strategic tools include bounded contexts, ubiquitous language, and context maps; tactical tools include entities, value objects, aggregates, domain services, and repositories.

Use it when business rules and boundaries are complex enough to justify domain modeling and collaborative discovery. It does not require microservices, event sourcing, or an ORM. Apply terms only when they help express invariants and ownership.

## Enterprise application patterns

These patterns address common application/data concerns cataloged by Martin Fowler. Select them based on the data model and transaction semantics; modern frameworks may provide a mechanism without removing the design decision.

| Pattern | What it does | Trade-off / distinction |
| --- | --- | --- |
| Transaction Script | Organizes one business operation as a procedure that coordinates application logic and data access. | Simple for straightforward workflows; repeated domain rules may drift across scripts. |
| Domain Model | Represents business rules and state in an object model. | Useful for complex domain behavior; modeling can be unnecessary for simple CRUD. |
| Table Module | Organizes domain behavior around a database table or record set. | Can fit tabular data and procedural systems; weaker fit for rich entity relationships. |
| Service Layer | Defines application operations and coordinates transactions and domain work. | Establishes an API boundary; should not become a pass-through layer with duplicated policy. |
| Repository | Presents a collection-like/domain-facing interface for retrieving and storing aggregates or entities. | Isolates persistence mechanics; avoid generic repositories that obscure query capabilities or transaction behavior. |
| Data Mapper | Moves data between objects and a relational store while keeping them independent. | Supports rich domain models, with mapping and identity complexity. |
| Active Record | Couples a domain record to persistence operations. | Convenient for simple CRUD; can mix domain behavior with storage and complicate multiple persistence strategies. |
| Unit of Work | Tracks changes and coordinates persistence in a transaction. | Clarifies atomic work; transaction boundaries and concurrency still need explicit design. |
| Identity Map | Ensures a data-loading context has one in-memory object per database identity. | Avoids inconsistent duplicate instances; scoped identity maps have lifecycle/memory costs. |
| Data Transfer Object | Carries data across a boundary without exposing internal domain objects. | Supports serialization/versioning boundaries; mapping can duplicate fields and must be maintained. |
| MVC / Front Controller | Organizes request handling and presentation responsibilities. | Framework conventions vary; avoid making controllers own domain policy. |

These concepts are related but not synonyms: a DAO generally hides storage operations; a Repository aims at a collection-like domain/application abstraction; a Unit of Work manages changes/transaction scope.

## Resilience and messaging patterns

Remote calls can fail, time out, duplicate, or complete after the caller has given up. Reliability patterns need explicit budgets and operational evidence.

| Pattern | Problem and mechanism | Required design decisions / risks |
| --- | --- | --- |
| Timeout | Bounds how long a caller waits for a dependency. | Set realistic end-to-end budgets; a timeout does not prove the remote operation stopped. |
| Retry with backoff | Repeats a plausibly transient failure after bounded delays. | Bound attempts and total duration; use jitter to avoid synchronized retry storms; retry only idempotent or deduplicated operations. |
| Circuit breaker | Temporarily rejects calls to a failing dependency, then probes for recovery. | Define failure thresholds, open/half-open behavior, fallback semantics, and monitoring. It does not repair the dependency. |
| Bulkhead | Isolates pools/limits so one slow dependency or workload cannot consume all resources. | Partitioning reduces shared capacity; set pool limits and queue behavior from workload needs. |
| Rate limiting | Caps accepted work over a time window or resource budget. | Decide key, fairness, rejection response, and whether limits apply per user, tenant, or system. |
| Cache-aside | Read from cache; on miss load from the source and populate the cache. | Define TTL, invalidation, stampede control, stale-read tolerance, and cache-key safety. A cache is not automatically the source of truth. |
| Competing consumers | Multiple workers consume from one queue to increase throughput. | Ensure message processing is safe under concurrency and duplicates; manage ordering, backpressure, visibility, and poison messages. |
| Idempotent consumer | Repeated delivery produces one logical effect, often through a deduplication key and atomic update. | Define key scope/retention and make dedupe atomic with the side effect where possible. |
| Transactional outbox | Store business changes and an event record in one local database transaction; publish the outbox asynchronously. | A publisher may deliver more than once; consumers still need idempotency and operational replay/cleanup. |
| Saga / compensating transaction | Coordinate a multi-step workflow across service-owned transactions, with compensating actions on failure. | Compensation is a new business action, not time reversal; define partial states, retries, deadlines, and manual recovery. |
| Choreography vs orchestration | Choreography lets services react to events; orchestration uses a coordinator to direct steps. | Choreography can hide workflow state; orchestration can concentrate policy. Choose based on visibility, coupling, and workflow complexity. |
| CQRS | Separates command/write responsibilities from query/read models. | Adds synchronization and operational complexity; use when read/write needs differ materially, not as a default CRUD architecture. |
| Event sourcing | Stores an append-only event history as the canonical source and derives current state by replay/projection. | Requires event/schema evolution, replay, projection repair, audit/privacy strategy, and eventual consistency. It is not simply “publish events.” |

Do not stack retry, circuit breaker, and queue patterns without considering their interaction. For example, retries at several layers multiply load, while a circuit breaker around a retry loop may have surprising timing. Set an end-to-end deadline and make overload behavior explicit.

## Choosing architecture deliberately

| Need | Consider first | Common overreach |
| --- | --- | --- |
| Separate UI, rules, and persistence responsibilities in one application | A small layered or ports-and-adapters design | Deep layers that only forward calls |
| Preserve complex business rules from framework/database changes | Domain model with inward dependencies | Applying Clean Architecture to every CRUD endpoint |
| Keep growing product modules independent without network boundaries | Modular monolith | Unrestricted shared tables and imports |
| Deploy/scale a capability independently with clear team/data ownership | A service boundary; then microservices if justified | Splitting by technical layer or speculative future scale |
| Handle work that does not need an immediate response | Queue/event-driven flow | Making all operations asynchronous |
| Improve a read-heavy or independently optimized query path | A read model; consider CQRS if divergence merits it | Event sourcing solely to enable CQRS |
| Protect a service from a flaky dependency | Timeouts, bounded retries, bulkheads, circuit breaker | Retry loops with no deadline or idempotency |
| Reliably publish after a database commit | Transactional outbox | Assuming a database commit and message-broker publish are one atomic transaction |

## References and source details

Inspected on **2026-11-06**:

- [Azure Architecture Center: Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/) says to select a pattern from a workload's problem and constraints and describes trade-offs. Its current catalog links individual entries including [Bulkhead](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead), [Cache-Aside](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside), [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker), [Compensating Transaction](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction), [Competing Consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers), [CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs), [Event Sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing), [Idempotent Consumer](https://learn.microsoft.com/en-us/azure/architecture/patterns/idempotent-consumer), [Leader Election](https://learn.microsoft.com/en-us/azure/architecture/patterns/leader-election), and [Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry). Azure examples are implementation examples, not requirements to use Azure.
- [Design Gurus pattern repository](https://github.com/design-gurus/design-patterns) separates [code patterns](https://github.com/design-gurus/design-patterns/tree/main/code-patterns) from [system patterns](https://github.com/design-gurus/design-patterns/tree/main/system-patterns) and points to [examples](https://github.com/design-gurus/design-patterns/tree/main/examples). The README claims 60+ system patterns; the exact set can evolve, so the repository is the index rather than a fixed count in this note.
- [Martin Fowler's Enterprise Application Architecture catalog](https://martinfowler.com/eaaCatalog/) groups patterns for domain logic, data sources, object-relational mapping, web presentation, distribution, offline concurrency, and more. The [book page](https://martinfowler.com/books/eaa.html) describes roughly forty field-observed patterns and notes that frameworks may implement mechanics while leaving design decisions to the developer.
- [Martin Fowler's architecture guide](https://martinfowler.com/architecture/) discusses architecture as consequential internal design and trade-offs; it is useful context rather than a prescriptive catalog.
- [Microsoft's eShop reference application](https://github.com/dotnet/eShop) is an evolving example of a services-based .NET application using Aspire. At the time inspected it targeted .NET 10; treat the repository's version and architecture as a moving example, not a timeless architecture template.
- [Clean Architecture book page](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/) was cited by the original outline but not fetched in this review; the explanation above is an original summary, not a chapter-by-chapter transcription.
- The [Game Programming Patterns contents](https://gameprogrammingpatterns.com/contents.html) also includes sequencing, decoupling, and optimization patterns such as [Game Loop](https://gameprogrammingpatterns.com/game-loop.html), [Event Queue](https://gameprogrammingpatterns.com/event-queue.html), [Dirty Flag](https://gameprogrammingpatterns.com/dirty-flag.html), and [Spatial Partition](https://gameprogrammingpatterns.com/spatial-partition.html). These are especially relevant to its game context and are not substitutes for application or distributed-system patterns.
