# High-Level Design (HLD) interview guide

[HLD/LLD interview hub](README.md) · [System Design](../../04-system-design/README.md) · [AI/ML specialty track](ai-ml-system-design-interviews.md)

## What an HLD interview evaluates

An HLD interview asks you to turn an under-specified product or infrastructure prompt into a coherent system and explain why it fits. The interviewer is usually looking for:

- Problem navigation: clarify, prioritize, and keep moving.
- A complete baseline design that satisfies the selected requirements.
- Sound knowledge of APIs, data, networks, storage, and distributed failures.
- Trade-offs grounded in workload and explicit assumptions.
- Clear, collaborative communication and the ability to respond to new constraints.

The goal is not to draw the largest architecture. A small architecture that handles the stated workload correctly is better than a distributed diagram whose components do not correspond to requirements.

## A 45-minute delivery framework

Adapt this to the interview length and to where the interviewer wants depth. Tell the interviewer your plan and check in before moving on.

| Approx. time | Activity | Deliverable |
| --- | --- | --- |
| 0–5 min | Clarify and prioritize requirements | 3–5 functional requirements, top non-functional constraints, explicit exclusions |
| 5–8 min | Identify entities and usage shape | Core resources/actors; read/write pattern and important hot paths |
| 8–12 min | Define interfaces and data model | A few key APIs/events and the data/invariants behind them |
| 12–22 min | Draw the baseline architecture | Complete request/data flow that satisfies the chosen requirements |
| 22–38 min | Deep dive into the highest-risk areas | One or two bottlenecks, consistency decisions, failure handling, scaling |
| 38–45 min | Summarize, test failure cases, discuss trade-offs | What works, what breaks next, alternatives and remaining assumptions |

Some interviewers expect explicit capacity estimates early; others prefer estimates only when they influence a decision. Ask, or say you will estimate the quantities that determine a choice. Do not spend five minutes multiplying numbers only to reach “the system is large.”

For a 60-minute interview, take more time to explore the key deep dive and follow-ups, not to add extra requirements or services. For a shorter screen, keep the same sequence and reduce the detail.

## Step 1: Clarify the problem

Start with the user-visible actions, then agree on scope. Example prompts:

- Who are the users/clients, and what are the 2–4 operations they must perform?
- What is in scope for the first version? What can be deferred?
- What are the most important latency, availability, durability, and consistency expectations?
- Are traffic patterns steady, bursty, globally distributed, read-heavy, or write-heavy?
- What data is private or regulated? Which clients/tenants must be isolated?
- What are expected retention, deletion, and recovery requirements?

Prioritize requirements explicitly. “Support a social network” is not a specification. For a feed, maybe the core scope is follow, publish, and fetch a recent feed; ranking, ads, recommendation, and full-text search may be out of scope unless the interviewer requests them.

Turn vague quality words into contextual targets:

- Instead of “low latency,” identify the operation and a target such as “p99 feed read under 300 ms” (an interview assumption, not a universal value).
- Instead of “high availability,” ask whether reads/writes may be degraded during a dependency outage.
- Instead of “consistent,” ask which writes must be immediately visible to which reads.
- Instead of “scalable,” estimate the workload dimension that is expected to grow.

If requirements conflict, surface the trade-off rather than pretending every quality can be maximized at once.

## Step 2: Estimate only what changes the design

State assumptions and use round numbers; accuracy is less important than revealing which order of magnitude matters.

### Useful formulas

| Quantity | Estimate |
| --- | --- |
| Average requests/second | requests per day ÷ 86,400 |
| Peak requests/second | average requests/second × an explicitly stated peak factor |
| Daily storage | records created per day × average stored bytes per record |
| Retained storage | daily storage × retention days × overhead for indexes/replicas |
| Network bandwidth | requests/second × average payload bytes × 8, then convert bits/second to a useful unit |
| Read/write ratio | read operations ÷ write operations, with the time window stated |

For an illustrative system creating **10 million links/day**, writes average about **116/s** (`10,000,000 ÷ 86,400`). If reads are 100× writes, redirects average about **11,600/s**. Assuming a 5× peak gives about **580 writes/s** and **58,000 reads/s** at peak. If a link record averages 500 bytes, raw creation storage is about **5 GB/day** before indexes, replicas, and backups. These numbers are not a benchmark; they help justify that redirects, not link creation, dominate this example's hot path.

### When to calculate

Estimate when it decides:

- Whether one node or a shared database can handle the workload.
- Whether a cache, queue, index, partition, or dedicated storage system is justified.
- Memory needed for a working set or rate-limit state.
- Bandwidth/egress for images, video, or large payloads.
- A partitioning or retention strategy.

Do not invent precision. State what could change the choice and keep moving if the design is insensitive to the estimate.

## Step 3: Define APIs, events, and core data

For each core use case, identify the caller, request, response, and side effects. Keep the interface small; it is a contract, not a full implementation exercise.

Example:

```text
POST /v1/links
  request: originalUrl, optional customCode, optional expiresAt
  response: code, shortUrl, expiresAt

GET /{code}
  response: redirect to originalUrl or a defined not-found/expired result
```

Decisions to state when relevant:

- Authentication/authorization and tenant identity come from a trusted credential, not an untrusted request body.
- Use idempotency keys for retryable create/payment operations where duplicate side effects are harmful.
- Use pagination for large collections; cursor pagination is often more stable for rapidly changing feeds.
- Choose REST/HTTP as a reasonable external default; choose gRPC or another RPC mechanism for a concrete internal need, not for prestige.
- Use WebSockets for bidirectional real-time sessions, server-sent events for server-to-client updates, or polling when its simplicity is acceptable.

List only the data fields needed to explain the design. Show primary keys, important indexes, ownership, retention, and invariants. Example for the URL shortener: `Link(code PK, ownerId, destination, createdAt, expiresAt, status)` and an appendable click-event stream for asynchronous analytics.

## Step 4: Draw a complete baseline

Start with the simplest architecture that meets the selected requirements:

```text
Client -> DNS/edge -> load balancer or gateway -> stateless application
                                           |-> primary database
                                           |-> cache (only for a justified hot path)
                                           `-> queue -> background workers -> external systems
```

Do not add every box by default. Label responsibilities and arrows:

- Is the arrow a synchronous request/response, an asynchronous message, or a replication stream?
- Which component owns the source of truth?
- What data crosses the boundary?
- What happens if the call times out, is duplicated, or fails?
- Where are identity, authorization, validation, and rate limits enforced?
- Which component can scale independently, and what shared resource can still saturate?

The diagram must satisfy the API and data model. Narrate one read path and one write path end to end; this exposes missing components and hidden dependencies.

## Step 5: Deep dive by bottleneck, not by component list

Pick the highest-risk decision based on requirements. Common candidates:

- A hot read path: indexes, cache hit rate, TTL, invalidation, hot keys, replica lag.
- A high write rate: partitioning, append-only log, write contention, batching.
- A feed or notification fan-out: fan-out-on-write vs fan-out-on-read, celebrity/hot-key behavior, async queues, backpressure.
- A multi-step transaction: database transaction boundary, idempotency, outbox, saga/compensation.
- A real-time connection: connection ownership, reconnect, ordering, delivery acknowledgment, presence state.
- A global service: regional routing, data locality, replication lag, conflict resolution, disaster recovery.

For each deep dive, state: **constraint → proposed mechanism → expected benefit → new failure/cost → test or metric that would validate it**.

## HLD fundamentals, with interview intuition

### Networking and communication

- **DNS** maps names to addresses; it is a lookup/routing dependency, not a request load balancer by itself.
- **TCP** provides an ordered reliable byte stream; **UDP** is datagram transport without built-in delivery/order guarantees. **QUIC/HTTP/3** uses UDP while implementing transport behavior appropriate to modern HTTP.
- **HTTP** defines request/response semantics. HTTP/1.1, HTTP/2, and HTTP/3 differ in connection and transport behavior; do not infer application-level reliability or exactly-once effects from the protocol version.
- **TLS** protects confidentiality/integrity and authenticates endpoints when configured correctly.
- **Reverse proxy** receives requests on behalf of servers and can route, terminate TLS, or enforce policy. A **load balancer** distributes work; it may be a reverse proxy. An **API gateway** commonly adds API-specific routing, authentication, quotas, or transformation; these roles can overlap in products.
- **CDN** serves cacheable content nearer users. Dynamic content needs explicit cache keys, freshness, privacy, and invalidation rules.
- **REST** models resources and uses HTTP methods/status codes; **gRPC** offers typed RPC over HTTP/2 with protobuf in common deployments; **WebSocket** is bidirectional; **SSE** is server-to-client streaming over HTTP.

**Intuition:** select the simplest protocol whose semantics fit the use case. “Real-time” does not always mean WebSockets; server push or a short polling interval may be sufficient. Long-lived connections require reconnect, ownership, and failure plans.

### Data and storage

- **Relational databases** support constraints, transactions, joins, and flexible querying; they are often a sound default for product data. Model access patterns and invariants before switching.
- **NoSQL** is a family (key-value, document, wide-column, graph, etc.), not a single scaling property. Choose a data model and partition/query pattern that fit the workload.
- **Indexes** trade extra storage and write work for faster lookup. B-trees support ordered/range access; LSM-based structures often favor write paths with compaction/read-amplification trade-offs. Use actual query patterns.
- **Transactions/ACID** provide a scoped correctness boundary. Isolation levels determine which concurrent anomalies are possible; “a transaction” does not automatically make a distributed workflow atomic.
- **Replication** copies data for availability/read scaling; replicas can lag. **Sharding/partitioning** splits data to distribute load and capacity but introduces routing, rebalancing, hot partitions, and cross-shard operations.
- **Object storage** is suited to large blobs and immutable media; keep metadata/queryable entities elsewhere where that improves access.
- **Consistent hashing** can reduce remapping when nodes change in some distributed-partitioning designs; it does not remove hot-key skew, replication, or rebalancing concerns.
- **CAP** is about behavior during a network partition: a distributed system cannot guarantee both a linearizable/strongly consistent response and availability for every request in that partition. Partition tolerance is not an optional feature in a real distributed network; choose which requests may fail, wait, or return stale/conflicting data under the relevant failure.

**Intuition:** begin with read/write queries and invariants. Use the database that makes them correct and understandable; add partitioning only when capacity or availability requirements justify its coordination cost.

For a deeper, worked treatment of schema design, query-driven indexes, relational/document trade-offs, Redis data structures, and operations, see the [database design and data stores guide](../../09-data-engineering/database-design/README.md).

### Caching

Common patterns:

- **Cache-aside:** application reads cache, loads on miss, and populates it.
- **Write-through:** write cache and durable store in a coordinated path; define failure ordering.
- **Write-behind/write-back:** acknowledge cache before durable write; improves throughput but risks loss and requires replay/recovery.
- **Write-around:** bypass cache on writes; useful when data is unlikely to be read soon.
- **TTL/refresh-ahead:** limit freshness lifetime or proactively refresh frequently used values.

Always discuss cache key, value, freshness, invalidation, stampede protection, negative caching, privacy/tenant boundaries, and source of truth. A cache can increase latency/cost and make stale-data bugs harder; it is not automatically a database replacement.

### Queues, event streams, and asynchronous work

For a deeper comparison of queue brokers, retained streams, Kafka/RabbitMQ, delivery guarantees, retries, and backpressure, see [Messaging and event streaming](../../09-data-engineering/messaging-and-event-streaming.md).

- A **queue/work broker** distributes work to consumers; acknowledge only after a defined processing point.
- A **pub/sub** channel lets multiple subscribers receive an event; delivery, replay, and ordering depend on the technology and configuration.
- **Kafka** is a durable partitioned event-stream platform with consumer offsets/replay and consumer groups; partitioning affects order and parallelism.
- **RabbitMQ** centers on exchanges, routing, queues, acknowledgments, and work distribution; its tutorials cover publish/subscribe, routing, topics, RPC, and streams.
- **At-most-once** can lose work; **at-least-once** can repeat work. Broker “exactly once” features have specific scope; an external payment/database side effect still needs idempotency or an atomic transaction boundary.
- A **dead-letter queue** isolates repeatedly failing messages; define replay, alerting, retention, and operator ownership.
- **Backpressure** limits work admission when consumers are slower than producers; unbounded queues only move the overload into memory and latency.
- Use an **outbox** to atomically store state and an event record in one local DB transaction, then publish asynchronously. Consumers should still be idempotent.

**Intuition:** async work decouples latency and absorbs bursts, but changes the user-visible consistency and failure model. Name what the caller sees while work is pending and how failures become visible.

### Scalability and reliability

- **Vertical scale** increases one machine's resources; **horizontal scale** adds instances and requires attention to shared state and bottlenecks.
- Stateless application instances ease horizontal scaling, but databases, connection pools, caches, queues, and external services remain constrained shared dependencies.
- A **rate limiter** protects a service and enforces a quota; choose algorithm, key, storage, fairness, and degraded-mode behavior.
- A **timeout** bounds waiting. A **retry** is appropriate only for plausibly transient errors, with an end-to-end deadline, bounded attempts, backoff/jitter, and idempotent operation.
- A **circuit breaker** temporarily fails fast against an unhealthy dependency; a **bulkhead** reserves separate resource pools; neither fixes the underlying dependency.
- **Replication/failover** improve recovery but require health detection, consistency expectations, and testing. Multi-region may add latency, data-conflict, and operational complexity.
- Define backups, restore tests, **RPO** (acceptable data loss) and **RTO** (time to recover). A backup that has never been restored is an assumption, not a recovery plan.

**Intuition:** design failure behavior alongside the happy path. A system is not highly available merely because it has multiple servers if every instance depends on one overloaded database or one unprotected network call.

### Observability, operations, and security

For detailed telemetry, SLI/SLO, OpenTelemetry, Prometheus, and alerting guidance, see [Observability and telemetry](../../10-cloud-devops/observability-and-telemetry.md).

- **Logs** preserve event context; **metrics** summarize numeric behavior over time; **traces** connect a request across components. Use correlation/trace IDs carefully and avoid secrets/PII.
- Define user-facing **SLIs** (measured service indicators) and **SLOs** (target objectives over a window); an **SLA** is a customer/business commitment and may carry contractual consequences.
- Monitor latency percentiles, errors, traffic, saturation, queue age/depth, cache hit/miss, replication lag, and dependency health. Alert on user impact/actionable symptoms, not every internal fluctuation.
- Include authentication, authorization, tenant isolation, encryption, key/secret management, input validation, abuse/rate controls, audit needs, data retention, and privacy in the design.
- Deployment, autoscaling, and containers do not repair a poor data model or missing failure semantics. State whether infrastructure is in scope and keep vendor-specific details proportional to the prompt.

## Worked example: URL shortener

This is one possible design, not a memorized answer. Start by stating assumptions and inviting correction.

### 1. Scope and requirements

Core:

1. Create a short link for a valid destination.
2. Resolve a code to the destination and redirect.
3. Reject unknown, disabled, or expired codes.

Assume custom aliases and deletion are in scope only if the interviewer requests them. Analytics can be asynchronous and eventually consistent. Choose an illustrative requirement such as redirect p99 below 100 ms in the primary region; ask whether availability or immediate revocation is more important.

### 2. Workload estimate

Assume 10M link creations/day and a 100:1 redirect/create ratio. The example produces about 116 create requests/s and 11.6K redirects/s average; a stated 5× peak assumption gives ~580 creates/s and ~58K redirects/s. A 500-byte raw record gives ~5 GB/day before index/replica/backup overhead. This suggests read scaling/cache and retention matter more than write compute at the assumed scale.

If the interviewer changes the ratio, retention, or peak pattern, recalculate the affected decision instead of defending the old estimate.

### 3. API and data

- `POST /v1/links`: authenticated create; accept destination, optional expiry, and optional custom code; use an idempotency key if clients may retry creation.
- `GET /{code}`: resolve and redirect; define behavior for expiry, disabled links, and missing codes.
- `DELETE /v1/links/{code}`: optional owner-authorized disable operation.

Store `Link(code PK, destination, ownerId, createdAt, expiresAt, status)`. Enforce unique codes in the durable store. Store click analytics separately from the redirect-critical mapping.

### 4. Baseline architecture

```text
Clients -> DNS/CDN/edge -> load balancer -> stateless Link API
                                             |-> cache: code -> mapping/status/expiry
                                             |-> relational or key-value mapping store
                                             `-> click-event queue -> analytics workers/store
```

**Create path:** authenticate and validate; generate/claim a code; write the mapping with a unique constraint; return the short URL. If insertion collides, generate another code or allocate another ID. A monotonic ID encoded in Base62 avoids random collisions but needs a safe ID-allocation strategy and can reveal creation order; random codes avoid a central counter but require collision detection/retry and enough entropy.

**Redirect path:** read cache; on miss fetch the authoritative mapping; validate status/expiry; return redirect. Emit click analytics asynchronously so analytics does not block the redirect. Cache TTL must not extend past expiry; deletion/revocation needs invalidation or a bounded stale window consistent with the requirements.

### 5. Deep dives

- **Hot reads:** measure hit ratio and hot-code skew; cache only if useful; consider regional cache/CDN for popular links if stale revocation is acceptable.
- **Code uniqueness:** enforce uniqueness durably; estimate collision probability for random codes; retry boundedly; plan for abuse of custom aliases.
- **Analytics:** queues introduce duplicate and delayed events; aggregate idempotently; keep event volume and retention separate from transactional mapping.
- **Abuse/security:** validate destination schemes, rate-limit creation, prevent unauthorized edits, protect against malicious redirects, and limit enumeration where required.
- **Failure:** if cache is down, can reads fall back to DB within capacity? If the DB is down, is redirect an error or can cached entries be served? State the consistency and availability trade-off.
- **Growth:** read replicas, partitioning, or multi-region come only after bottlenecks and consistency needs are identified. Do not shard in the baseline without a requirement.

### 6. Close

Restate how the design meets the core requirements, name the main bottleneck (redirect read path under the assumptions), state the chosen freshness/availability trade-off, and identify what you would measure before scaling further.

## Common HLD interview traps

- Starting with “use Kafka/Redis/microservices” before stating the problem.
- Listing dozens of features instead of agreeing on a small in-scope set.
- Drawing components without request flow, ownership, or data contracts.
- Treating SQL vs NoSQL as a scale slogan.
- Saying “CAP means pick any two” without naming a partition scenario and the behavior of requests.
- Adding a cache without a stale-data/invalidation plan.
- Saying “exactly once” without defining the boundary and side effects.
- Assuming retries are safe or that a timeout cancels the remote operation.
- Spending all the time on estimation or low-value details and failing to present a working baseline.
- Naming a technology without explaining its role, data model, or failure mode.

## HLD practice set

Practice from a blank page, not by reciting a reference design. For each, change at least one assumption and explain how the design changes.

| Priority | Prompt | Core reasoning to practice |
| --- | --- | --- |
| Core | URL shortener | Unique IDs/codes, read-heavy path, caching, expiry, abuse |
| Core | Rate limiter | Algorithm, distributed counters, keying, atomicity, degradation |
| Core | Notification system | Fan-out, queues, retries, preferences, deduplication |
| Core | Chat application | WebSockets, message ordering/storage, presence, delivery |
| Core | News/feed system | Fan-out on write/read, hot users, ranking, cache |
| Core | File storage/Drive | Blob/object storage, metadata, chunked uploads, sync/versioning |
| Next | Video streaming | Upload/transcoding, object storage, CDN, adaptive delivery |
| Next | Ride sharing | Geospatial search, matching, location freshness, state transitions |
| Next | E-commerce | Inventory, idempotent checkout, payment, search, events |
| Next | Search engine | Crawl/index/query pipeline, inverted indexes, freshness |
| Next | Distributed cache | Partitioning, replication, eviction, consistency |
| Advanced | Job scheduler | Leases, retries, fairness, duplicate execution |
| Advanced | Payment system | Idempotency, ledger invariants, reconciliation, audit |
| Advanced | Video conferencing | Signaling vs media, latency, regional routing, packet loss |

## Further reading: verified source map

Inspected on **2026-11-06**. The locally written material above is the primary study path; these links provide protocol and implementation detail.

- [Hello Interview delivery framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery), [core concepts](https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts), [key technologies](https://www.hellointerview.com/learn/system-design/in-a-hurry/key-technologies), and [preparation plan](https://www.hellointerview.com/learn/system-design/in-a-hurry/how-to-prepare). It recommends a focused baseline and emphasizes requirements, entities, interface, architecture, and deep dives. Its advice is interview-specific, not a universal architecture rule.
- [System Design Primer](https://github.com/donnemartin/system-design-primer) sublinks cover [scalability and core system topics](https://github.com/donnemartin/system-design-primer#system-design-topics-start-here), [how to approach a question](https://github.com/donnemartin/system-design-primer#how-to-approach-a-system-design-interview-question), [sample system designs](https://github.com/donnemartin/system-design-primer#system-design-interview-questions-with-solutions), and [object-oriented designs](https://github.com/donnemartin/system-design-primer#object-oriented-design-interview-questions-with-solutions). Its open URL-shortening/Pastebin solution is [here](https://github.com/donnemartin/system-design-primer/tree/master/solutions/system_design/pastebin).
- [ByteByteGo System Design 101](https://github.com/ByteByteGoHq/system-design-101) organizes an illustrated catalog across API/web, real-world case studies, storage, messaging, and AI/ML; examples include [load balancer use cases](https://bytebytego.com/guides/key-use-cases-for-load-balancers), [API gateway concepts](https://bytebytego.com/guides/api-gateway-101), [database selection](https://bytebytego.com/guides/how-to-choose-the-right-database), [delivery semantics](https://bytebytego.com/guides/delivery-semantics), and [CAP misunderstandings](https://bytebytego.com/guides/cap-theorem-one-of-the-most-misunderstood-terms). Treat diagrams as entry points, not proof.
- [MDN HTTP guides](https://developer.mozilla.org/en-US/docs/Web/HTTP) cover [HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview), [evolution through HTTP/3](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP), [HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching), [authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication), and [proxies/tunneling](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Proxy_servers_and_tunneling).
- [Cloudflare Learning Center](https://www.cloudflare.com/learning/) links primers on DNS, CDNs, TLS, network layers, cloud, and streaming.
- [Designing Data-Intensive Applications](https://dataintensive.net/) is a book-level reference for storage and distributed-data fundamentals.
- [PostgreSQL documentation](https://www.postgresql.org/docs/current/) is the primary reference for [indexes](https://www.postgresql.org/docs/current/indexes.html), transactions, and concurrency control. The index/MVCC pages did not load in the research fetch, so this guide does not make PostgreSQL-version-specific claims from them.
- [MongoDB transactions](https://www.mongodb.com/docs/manual/core/transactions/) explains multi-document ACID transactions and cautions that effective schema design can avoid many distributed transactions.
- [Redis data types](https://redis.io/docs/latest/develop/data-types/) links to strings, hashes, lists, sets, sorted sets, streams, geospatial indexes, and other structures; choose from the data/operation contract, not the product name.
- [Apache Kafka: introduction](https://kafka.apache.org/intro) explains event streams, durable storage, processing, brokers, and clients. The [documentation index](https://kafka.apache.org/documentation/) is the deeper reference.
- [RabbitMQ tutorials](https://www.rabbitmq.com/tutorials) includes groups for AMQP 1.0, AMQP 0-9-1, and Streams, with examples for work queues, pub/sub, routing, topics, RPC, and stream offsets. The page currently targets RabbitMQ 4.x.
- [Microsoft Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/) catalogs technology-agnostic patterns including [Bulkhead](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead), [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker), [Cache-Aside](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside), [Competing Consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers), and [Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry); each has constraints and trade-offs.
- [Google SRE Book contents](https://sre.google/sre-book/table-of-contents/) links to [SLOs](https://sre.google/sre-book/service-level-objectives/), [monitoring distributed systems](https://sre.google/sre-book/monitoring-distributed-systems/), [handling overload](https://sre.google/sre-book/handling-overload/), [cascading failures](https://sre.google/sre-book/addressing-cascading-failures/), and [testing for reliability](https://sre.google/sre-book/testing-reliability/).
- [OpenTelemetry documentation](https://opentelemetry.io/docs/) links to [observability concepts](https://opentelemetry.io/docs/concepts/), [signals](https://opentelemetry.io/docs/concepts/signals/), and the [Collector](https://opentelemetry.io/docs/collector/). OpenTelemetry is instrumentation/collection/export, not itself a complete storage/alerting backend.
- [Prometheus overview](https://prometheus.io/docs/introduction/overview/) describes its time-series model, pull-based scraping, PromQL, service discovery, and alerting ecosystem; the overview also states where it is not a good fit (for example, exact per-request billing).
