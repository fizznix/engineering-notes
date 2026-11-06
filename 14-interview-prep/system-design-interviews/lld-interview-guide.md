# Low-Level Design (LLD/OOD) interview guide

[HLD/LLD interview hub](README.md) · [HLD interview guide](hld-interview-guide.md) · [Software design and code quality](../../02-programming/software-design/README.md)

## What an LLD interview evaluates

You are asked to design a bounded feature—such as a parking lot, game, scheduler, or vending machine—by identifying responsibilities, state, relationships, and contracts, then explaining or implementing the core behavior.

Interview formats vary:

- **Object-oriented design (OOD):** discuss a model and interactions, often with pseudocode or an informal diagram.
- **Machine coding:** build a runnable subset under a time limit; structure, correctness, and tests matter.
- **Language-specific design:** use the language's idioms and features rather than forcing Java-style interfaces into every language.

Ask what format is expected. “LLD” is not always synonymous with a UML-only exercise.

## A practical 40–45-minute delivery framework

| Approx. time | Activity | Deliverable |
| --- | --- | --- |
| 0–5 min | Clarify use cases, rules, and scope | Agreed requirements and explicit exclusions |
| 5–8 min | Identify core entities/values and relationships | Small domain model and ownership |
| 8–18 min | Define state, responsibilities, and public contracts | Methods that trace back to requirements |
| 18–32 min | Implement or walk through the core use cases | Happy path and important invariants |
| 32–40 min | Test edge cases and discuss concurrency/extensions | Correctness evidence and bounded trade-offs |
| 40–45 min | Summarize and answer follow-ups | Clear current design and next evolution point |

If the interviewer expects a 35-minute exercise, reduce the number of requirements and methods; do not skip clarifying scope or verifying one scenario. If the task is a machine-coding round of 90 minutes, reserve explicit time for a runnable project, tests, and a final cleanup pass.

## Step 1: Clarify behavior before naming classes

Use a small set of targeted questions:

- What operations must the system support?
- What are the important rules for success, failure, and state transitions?
- What inputs are invalid, and what should happen for them?
- Which actors, object types, or policies vary?
- What is explicitly out of scope (UI, persistence, network, concurrency, payment integration)?
- Is this an in-memory model, production service, or executable implementation?
- How much code/notation does the interviewer expect?

Write 3–6 requirements. Avoid turning every noun in the prompt into a class. A candidate class should own meaningful state, enforce an invariant, represent a domain concept, or provide a useful boundary.

## Step 2: Derive ownership, state, and behavior

For each requirement, ask:

1. Which object owns the state needed to satisfy it?
2. Which object can enforce the invariant without exposing its internals?
3. What command changes state, and what query reads it?
4. Which workflow coordinates the objects?
5. What can fail between steps?

Use this mapping:

| Requirement | Owner/state | Operation/contract | Test |
| --- | --- | --- | --- |
| A player places a move only on an empty cell | Board owns cell state and board rules | `place(player, cell)` returns a defined result | Occupied/out-of-range cell is rejected without changing state |
| A match alternates players and ends on win/draw | Game owns turn and lifecycle | `makeMove(player, cell)` validates turn and updates status | Wrong player, win, draw, and move-after-game-over |
| A spot is assigned to at most one active vehicle | Allocation boundary owns reservation | Atomic `reserveCompatible(vehicle)` | Concurrent allocation never double-books one spot |

This keeps the policy near its state while letting an application/service object coordinate the full use case. “Tell, Don't Ask” is a useful prompt, not a ban on all queries/getters.

## Step 3: Design the smallest useful model

Common building blocks:

- **Entity:** has identity and lifecycle; equality is usually by stable identity.
- **Value object:** describes a value/concept, ideally immutable and validated at construction (for example, `Money`, `EmailAddress`, or `TimeRange`).
- **Service/use case:** coordinates a workflow that does not belong to one entity or is an application boundary.
- **Repository/gateway:** isolates persistence or an external dependency when that boundary matters.
- **Policy/strategy:** isolate an independently varying rule (pricing, allocation, ranking) when the requirement supports multiple policies.
- **Event/command:** represent something that happened or an operation that needs an explicit lifecycle/queue; do not wrap every method call.

Prefer composition for combining behaviors and collaborators. Use inheritance when the subtype is genuinely substitutable under the base contract and shared implementation is stable. If each subtype overrides behavior in incompatible ways, reconsider the hierarchy.

### Interfaces: add them for a reason

Introduce an interface/abstract contract when at least one is true:

- Two or more real implementations must be selected.
- A volatile external system needs an adapter boundary.
- A contract isolates important logic or enables useful tests.
- A plugin/extension point is an explicit requirement.

Do not add an interface for every class “for testability.” A small function or concrete collaborator may be clearer and easier to change.

### UML essentials

Draw only what helps communicate:

- **Association:** one object knows/uses another; label direction/multiplicity if relevant.
- **Composition:** strong ownership/lifecycle; the part normally belongs to one whole.
- **Aggregation:** weaker whole/part relationship; use only if the distinction communicates something.
- **Dependency:** a temporary “uses” relationship (often a method parameter/call).
- **Generalization/inheritance:** subtype relationship with a behavioral contract.
- **Sequence diagram:** key interaction order for one use case.
- **State diagram:** allowed transitions and terminal/invalid states.

A compact class sketch is often enough:

```text
ParkingService
  + park(vehicle): Ticket
  + exit(ticketId): Fee
  - allocator: SpotAllocator
  - tickets: TicketRepository

SpotAllocator
  + reserveCompatible(vehicle): Spot or NoSpotAvailable

Ticket
  - id, vehicleId, spotId, enteredAt, status
  + close(exitedAt): Duration
```

UML is a modeling language, not a process or an objective. Use formal notation if the interviewer requests it; otherwise clear boxes, arrows, state, and method contracts are enough.

## Step 4: Define contracts and invariants

For each important operation, state:

- Preconditions (valid ID, correct owner, allowed lifecycle state).
- Postconditions (created object, updated state, side effects).
- Failure results (not found, conflict, unavailable resource, invalid transition).
- Whether the call is atomic or may partially succeed.
- Whether it is safe to retry.

Choose return types that distinguish success from expected failure. Do not let a method return a success-shaped empty value when it actually failed.

## Step 5: Implement the interesting path, not every getter

Start with one use case end to end. Keep method bodies readable and make state transitions explicit. Then trace edge cases. If asked to write code:

- Use the language you can implement clearly under pressure.
- Compile/run a minimal skeleton early in a machine-coding round.
- Keep setup small; do not build a generic framework.
- Include a few focused tests rather than claiming broad coverage without evidence.
- Explain any code that is pseudocode and ask whether more complete syntax is expected.

## Worked example: parking lot

This design deliberately chooses a small scope: a multi-floor lot supports motorcycles, cars, and large vehicles; the system assigns an available compatible spot and issues a ticket; exit calculates an hourly fee. UI, gates, hardware, persistence, and payment-provider integration are initially out of scope. Confirm these assumptions with the interviewer.

### Requirements and invariants

1. `park(vehicle)` assigns a compatible free spot and returns a unique ticket.
2. If no compatible spot exists, return `NoSpotAvailable` and make no partial allocation.
3. `exit(ticketId)` closes an active ticket, calculates the fee from entry/exit time, and frees the spot.
4. Unknown or already-closed tickets are rejected.
5. A spot has at most one active reservation/ticket; an active vehicle has at most one ticket.
6. Pricing rule and time rounding are explicit (for example, round each started hour up, with a stated minimum).

Do not add valet, EV charging, monthly passes, or multiple pricing zones unless asked. If the interviewer adds one, identify the change point rather than prebuilding every extension.

### Responsibilities

| Component | Owns/does | Should not do |
| --- | --- | --- |
| `ParkingService` | Coordinates park/exit use cases and result handling | Know UI or database-specific commands |
| `ParkingFloor` | Owns its spots and provides candidate availability | Decide payment policy |
| `ParkingSpot` | Knows its type and free/reserved/occupied state | Be assigned independently by arbitrary callers |
| `Vehicle` / `VehicleType` | Represents vehicle identity and category | Be a giant inheritance tree for each incidental behavior |
| `ParkingTicket` | Owns ticket identity, spot/vehicle association, entry and close state | Be reused after closure |
| `SpotAllocator` | Selects/reserves a compatible spot | Calculate fees |
| `PricingPolicy` | Calculates fee from duration and stated rules | Mutate spot/ticket lifecycle |
| `Clock` | Supplies time to the use case | Be read from global system time inside every class |

For a first version with one fixed allocation/pricing rule, those may be simple collaborators or methods, not necessarily interfaces. If multiple strategies are required, define focused contracts and implementations.

### Core operation sketch

```text
park(vehicle):
    validate vehicle
    atomically:
        spot = allocator.reserveCompatible(vehicle)
        if no spot: return NoSpotAvailable
        ticket = ticketStore.createActive(vehicle.id, spot.id, clock.now())
        return ticket
```

The word **atomically** matters. In a single in-memory program, use a lock/critical section around selecting and reserving. In a database implementation, use a transaction or conditional update/lock so two concurrent requests cannot reserve the same spot. Do not perform a slow external call while holding a global lock.

```text
exit(ticketId):
    atomically:
        ticket = ticketStore.findActive(ticketId)
        if missing: return InvalidOrClosedTicket
        exitTime = clock.now()
        fee = pricingPolicy.calculate(ticket.enteredAt, exitTime)
        ticket.close(exitTime, fee)
        allocator.release(ticket.spotId)
        return fee
```

If payment is added, do not pretend ticket closure, spot release, and an external payment charge form one database transaction. Define a payment-pending state, idempotency key, timeout/reconciliation behavior, and when the spot becomes available.

### Tests that expose design quality

- Allocate a compatible spot; reject incompatible/full lot.
- Two concurrent park operations cannot receive the same spot.
- Entry/exit use an injected clock so fee rounding is deterministic.
- An invalid/closed ticket does not release a spot a second time.
- Failure during ticket creation does not strand a reservation.
- A new allocator/pricing rule can be added without changing unrelated ticket and spot invariants—but only if that extension is actually required.

### What to say when requirements change

If asked for EV charging, ask whether charger compatibility is a hard allocation constraint; model charger availability and reservation rather than adding an `if` across every class. If asked for a new fee rule, isolate pricing if multiple rules truly vary. If asked for multiple entrances, discuss allocation concurrency and shared state. Identify which design element changes and which invariants must remain.

## Machine-coding round checklist

1. Confirm required language, build/test command, input/output format, and allowed libraries.
2. Write the core use cases and invariants before project structure.
3. Create the smallest executable skeleton; compile or run a smoke test early.
4. Implement one happy path end to end.
5. Add invalid input/state and boundary tests.
6. Add the next requirement through a focused change.
7. Leave time to run tests, explain limitations, and summarize design choices.

Avoid spending half the round creating generic layers, factories, dependency-injection frameworks, and UML before the required behavior runs.

## LLD problem practice

| Priority | Problem | Main concepts to practice |
| --- | --- | --- |
| Core | Parking lot | State, allocation, invariants, concurrency |
| Core | Elevator system | State machine, scheduling policy, concurrent requests |
| Core | Library management | Entities/relationships, availability, holds, overdue rules |
| Core | Splitwise | Money/value objects, balance invariants, split strategies |
| Core | Tic-Tac-Toe | Board ownership, turn/state transitions, validation |
| Core | Vending machine | State machine, inventory, payment/refund semantics |
| Core | Notification service | Channel adapters, preferences, retries, idempotency |
| Next | Movie-ticket booking | Seat reservation, expiry, atomic concurrency |
| Next | Chess | Board model, legal moves, game state, extensibility |
| Next | ATM | State, commands, account/transaction boundaries |
| Next | Food delivery | Order lifecycle, strategy, notifications |
| Next | Ride sharing | Matching policies, trip state, location freshness |
| Advanced | Logging framework | Levels, appenders, formatting, concurrency |
| Advanced | Rate limiter | Algorithms, clock, concurrency, keying |
| Advanced | Task scheduler | Commands/jobs, priority, retries, leases |

For every problem, do a short object-model discussion and a runnable core implementation at least once. Do not memorize one class diagram as the answer to every variant.

## Common LLD traps

- Coding before the interviewer and you agree on behavior.
- Making every noun a class or every class an interface.
- Putting all rules in a manager/controller class or all behavior in entities.
- Modeling type variation with a long switch that touches unrelated classes—or reflexively replacing every switch with polymorphism.
- Using inheritance where composition or a simple data field is clearer.
- Exposing mutable internal collections or allowing illegal state transitions.
- Ignoring atomicity/race conditions when the prompt includes concurrent actors or scarce resources.
- Adding design patterns for vocabulary rather than a real variation.
- Drawing a perfect UML diagram but never walking through a core operation.
- Writing code without testing one concrete normal and failure scenario.

## Further reading

Inspected on **2026-11-06**:

- [Hello Interview LLD introduction](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/introduction) distinguishes LLD from HLD and links to [delivery](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/delivery), [design principles](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/design-principles), [OOP concepts](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/oop-concepts), and [pattern guidance](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/patterns). Its delivery page covers requirements, entities/relationships, class state/behavior, implementation, and scenario verification. The linked parking-lot breakdown shows requirements but gates later detail; this document provides an independent complete worked example.
- [Awesome Low Level Design](https://github.com/ashishps1/awesome-low-level-design) links topic pages for [OOP fundamentals](https://github.com/ashishps1/awesome-low-level-design#-oop-fundamentals), [relationships](https://github.com/ashishps1/awesome-low-level-design#-class-relationships), [patterns](https://github.com/ashishps1/awesome-low-level-design#-design-patterns), [class/use-case/sequence/activity/state diagrams](https://github.com/ashishps1/awesome-low-level-design#-uml), concurrency, and interview problems.
- [Java Design Patterns](https://github.com/iluwatar/java-design-patterns) provides commented Java examples and links to [design principles](https://java-design-patterns.com/principles/) and [pattern groups](https://java-design-patterns.com/patterns/). Its project explicitly recommends beginning with KISS/YAGNI and adding complexity when a practical extension requires it.
- [UML Diagrams](https://www.uml-diagrams.org/) explains UML as a process-independent modeling language and provides links to class, sequence, and state diagrams. Formal notation is optional unless requested.
- [System Design Primer object-oriented design questions](https://github.com/donnemartin/system-design-primer#object-oriented-design-interview-questions-with-solutions) gives sample OOD exercises and solutions. Use it after attempting a design yourself.
- For detailed SOLID, pattern comparisons, clean-code practices, and refactoring, continue with the local [software-design package](../../02-programming/software-design/README.md).

