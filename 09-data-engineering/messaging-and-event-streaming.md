# Messaging and event streaming

[Back to Data Engineering](README.md) · [System Design](../04-system-design/README.md) · [Database design and data stores](database-design/README.md)

Messaging and event streaming move work or facts across process boundaries. They can decouple producers from consumers, smooth bursts, fan data out to several readers, and support replay or asynchronous processing. They also add queues of work, new failure modes, delayed consistency, and operational responsibilities.

The right starting question is not “Kafka or RabbitMQ?” It is: **what contract does the producer need, what does each consumer need to observe, and what should happen when any participant is slow, unavailable, or retries?**

## Vocabulary and mental model

- **Message:** data exchanged between components. It may represent a command/request (“resize this image”) or an event/fact (“image resize completed”).
- **Producer/publisher:** writes a message/event.
- **Broker:** accepts, routes, stores, or delivers messages.
- **Queue:** buffers work, commonly distributing each message to one competing worker that successfully processes it.
- **Pub/sub:** one publication is delivered to each matching independent subscription/queue. “Topic” can mean different structures in different products.
- **Event stream/log:** retained, ordered records that consumers can read by position and often replay. Reading does not necessarily remove an event.
- **Consumer/subscriber:** reads and processes messages/events; the application must define when processing is complete.
- **Offset/cursor:** position of a consumer in a retained stream/log.
- **Partition:** ordered subset of a topic/log used for parallelism and placement; ordering is usually scoped to one partition.

These are related patterns, not synonyms. A broker can support both queues and stream-like structures. Event sourcing is a domain data model in which events are the authoritative history; it is not the same thing as sending ordinary integration events through a broker.

## Why use asynchronous messaging?

### Benefits

- A front-end request can finish before expensive independent work completes.
- A buffer absorbs short bursts and lets workers scale separately from producers.
- A publisher can emit one event to several independently evolving consumers.
- A retained log can be replayed to rebuild projections or recover a consumer.
- Producers and consumers can be deployed independently if the event contract remains compatible.

### Costs and new obligations

- The caller may see “accepted/pending” before the operation finishes.
- State becomes eventually consistent across consumers/projections.
- Messages can be duplicated, delayed, reordered across keys/partitions, or permanently fail.
- Queues can grow without bound; delay can become an outage even when the API is healthy.
- Schema evolution and replay can break old consumers or repeat side effects.
- Broker, producer, consumer, and external-store reliability are all part of end-to-end data safety.

Use a synchronous call when the caller must immediately know the result and the operation fits a request deadline. Use asynchronous work when decoupling, buffering, fan-out, or replay is a real requirement. Do not put a broker in a simple call path only to make a diagram look distributed.

## Queue or retained stream?

| Need | Queue / task broker | Retained event stream |
| --- | --- | --- |
| Typical intent | Distribute work among workers | Publish a durable sequence of facts/events for independent readers |
| Consumption | A message is commonly acknowledged and removed/settled for that queue | Consumers track position; events remain until retention/compaction policy removes them |
| Fan-out | Bind separate queues/subscriptions for independent copies | Multiple consumer groups read the same topic independently |
| Replay | Usually requires explicit retry/republish/DLQ handling | Rewind/reset a consumer position while data is retained |
| Ordering | Queue/broker semantics and parallel consumers can change observed completion order | Usually ordered within a partition/log, not globally across partitions |
| Scale knob | Consumers, queue/broker capacity, prefetch and routing | Partitions, consumer groups, retention and key distribution |
| Common fit | Background jobs, routing, request/reply, broker-managed delivery | Event backbone, CDC, analytics, replayable integration events |

This table is a conceptual comparison, not a feature guarantee for every deployment. Durability, acknowledgment, ordering, retention, replication, and delivery guarantees depend on product, protocol, topology, and configuration.

## Kafka: partitioned, retained event log

Apache Kafka describes event streaming as capturing events, storing them durably, processing them in real time or retrospectively, and routing them to destinations. Kafka's core model:

- **Topic:** named event stream.
- **Partition:** append-only ordered log segment distributed across brokers.
- **Key:** commonly used to choose a partition. Records with the same key are routed together by a partitioning strategy, enabling per-key ordering when the key remains mapped consistently.
- **Producer:** publishes records; configuration influences acknowledgment, retries, batching, compression, and idempotence.
- **Consumer group:** a set of consumers cooperating over a topic; a partition is assigned to one consumer in a group at a time, so partitions limit parallel reads within that group.
- **Offset:** consumer progress position. Commit timing affects restart/replay and duplicate/loss risk.
- **Retention:** records are retained by age/size policy or compaction rules, rather than automatically removed just because one group read them.
- **Replication:** topic partitions have replicas; availability and acknowledged-write durability depend on replication factor, in-sync replica behavior, acknowledgments, and failure configuration.

Kafka guarantees order within a partition, not a single total order across a topic's partitions. A key that is too common can create a hot partition; an unkeyed/randomized strategy may distribute load but lose per-entity ordering. Select partitioning from both order requirements and expected load.

### Consumer groups and replay

Consumer groups let separate applications independently process a topic, and multiple workers within a group divide assigned partitions. If a consumer crashes or membership changes, partitions are reassigned; processing may repeat from the last committed offset. A consumer should make its side effects idempotent and coordinate offset commits with the effect boundary.

Replay is powerful but has product consequences: an old consumer may repeat email, charge, or external API calls unless it deduplicates or rebuilds a side-effect-free projection. Keep event retention, replay permissions, and privacy/deletion requirements aligned.

Kafka's Java/Scala ecosystem includes the Kafka Streams client library for stateful transformations, joins, aggregations, and event-time windows. Kafka Connect provides reusable source/sink connectors. These are distinct from the broker itself and should be introduced when integration or stream processing requires them.

## RabbitMQ: broker, exchanges, queues, and acknowledgements

In the RabbitMQ AMQP 0-9-1 tutorial model:

- A **publisher** publishes to an **exchange**.
- An exchange routes to zero or more queues based on its type, binding, and routing key.
- A **queue** buffers messages for consumers.
- A **consumer** receives a delivery and acknowledges it when processing is complete.

Common exchange behavior from the tutorials:

- **Direct:** exact routing-key match.
- **Fanout:** broadcast to all bound queues; routing key is ignored.
- **Topic:** pattern matching over dot-separated routing-key words (`*` for one word, `#` for zero or more).
- **Headers:** route based on message headers rather than only a routing key.

The tutorial page currently includes three learning tracks: AMQP 1.0, AMQP 0-9-1, and RabbitMQ Streams. Do not mix protocol-specific APIs or semantics. The Python work-queue/pub-sub/routing/topics/RPC pages are examples using Pika and the AMQP 0-9-1 model.

### Work queues and flow control

- Use work queues to defer expensive jobs and distribute them among competing consumers.
- Manual acknowledgements let a worker acknowledge after a defined success point; if the worker dies first, the broker can redeliver.
- Set consumer prefetch to bound outstanding deliveries and memory/in-flight work; too low can reduce throughput, too high can overload a worker and increase redelivery work.
- Publisher confirms acknowledge publisher-to-broker handling; consumer acknowledgements represent consumer-to-broker processing. These are separate links in the delivery path.
- Durable queues/messages, replicated queue types, confirms, and correct recovery must be configured for the required durability. A publisher confirm does not mean the business side effect has completed.

RabbitMQ Streams provide a persistent replicated append-only structure that consumers can reread, with offsets and non-destructive consumer semantics. The RabbitMQ documentation says Streams complement queues rather than simply replace them. Choose based on queue work distribution versus replayable stream consumption.

## Design a message contract

Use a small stable envelope. For example:

```json
{
  "eventId": "unique-id",
  "eventType": "orders.order-confirmed",
  "schemaVersion": 1,
  "occurredAt": "2026-11-06T12:00:00Z",
  "aggregateId": "order-123",
  "traceparent": "00-...",
  "data": {
    "customerId": "customer-456"
  }
}
```

Decisions to make:

- **Event vs command:** an event states something that happened; a command requests an action and may be rejected.
- **Identity:** unique event/command ID for deduplication and support investigations.
- **Key:** choose by the entity whose order matters (account/order/device) while checking skew.
- **Schema compatibility:** add optional fields or version deliberately; deploy producers and consumers without requiring a lockstep rollout; preserve old events long enough for replay.
- **Payload size:** avoid large blobs; publish an authorized, expiring reference (claim-check pattern) when payloads exceed broker limits or harm throughput.
- **Privacy:** do not put secrets or unnecessary personal data in a replayable event; define deletion/retention and access controls.
- **Trace context:** propagate trace identifiers across asynchronous boundaries so producer and consumer spans can be correlated.

## Delivery semantics and correctness

### At-most-once, at-least-once, and effectively-once

- **At-most-once:** deliver/process zero or one time; a failure may lose work.
- **At-least-once:** retry until acknowledged; a failure may cause duplicate delivery.
- **Effectively-once business effect:** at-least-once transport plus idempotent processing/deduplication so repeated deliveries do not repeat the domain effect.
- **Exactly-once:** a product may provide exactly-once guarantees inside a defined transactional boundary (for example, a stream processor's Kafka input-to-output processing under configured semantics). It does not automatically make an arbitrary database write, email, payment API, or other external side effect exactly once.

### Producer and consumer safety are independent

An event can fail to reach the broker, be accepted but not confirmed to the publisher, be delivered but not processed, or be processed while the acknowledgment is lost. For every boundary, define what is retried and whether a duplicate is possible.

An idempotent consumer can store `eventId` in an inbox/deduplication table with a uniqueness constraint in the same database transaction as its business update. If a duplicate arrives, the unique key makes it a no-op. The dedupe retention must cover the maximum replay/retry window.

### Transactional outbox

When a service must update its database and publish an event, writing to the database and broker as two independent operations creates a dual-write gap:

1. Database commits, publish fails → state changed but no event.
2. Publish succeeds, database rolls back → event describes state that never committed.

An **outbox** writes the business change and an event row in one database transaction. A relay or change-data-capture process publishes outbox rows and records progress. The relay can publish a duplicate if it crashes after publish but before marking progress, so consumers remain idempotent. Define outbox retention, ordering, retry, poison-event handling, and monitoring.

This is not a distributed transaction across database and broker; it changes an unsolved dual write into a durable local write plus retryable publication.

### Retries, dead letters, and poison messages

- Retry only transient failures; validation/schema/business errors usually need correction, not infinite retry.
- Use bounded attempts, exponential backoff with jitter, and a total deadline.
- Avoid immediately requeueing a poison message in a tight loop.
- Route exhausted/unprocessable messages to a dead-letter or quarantine path with original ID, failure reason, attempt count, and safe replay tooling.
- Alert on dead-letter volume and oldest message age; a DLQ without an owner/process is just hidden data loss.
- Make replay explicit, audited, and safe to repeat.

## Ordering, backpressure, and overload

- Global ordering limits parallelism. Prefer per-entity ordering only when the domain needs it.
- A queue grows when arrival rate exceeds successful processing rate. Track depth and **oldest message age**; depth alone can hide a growing latency breach.
- Bound consumers' in-flight work (prefetch/batch size), control producer admission, and set broker quotas/retention/capacity alarms.
- Scale consumers only if the downstream database/API can handle the added concurrency.
- Choose behavior at the queue limit: reject, shed, pause, or persist elsewhere. Never assume unbounded buffering solves overload.
- Consumer lag is a service signal: if the result is user-visible, define a freshness SLO and communicate pending status.

## Worked example: order confirmation

```text
Checkout API
  -> database transaction:
       write order(status = confirmed)
       write outbox(event = OrderConfirmed, key = orderId)
  -> outbox relay publishes to "order-events"
  -> consumer group: inventory projection
  -> consumer group: notification service
  -> consumer group: analytics pipeline
```

Design decisions:

1. The order and outbox row commit together.
2. `orderId` is the stream key if per-order event order matters; monitor key skew.
3. Inventory and notifications are separate consumer groups; each receives its own logical copy/stream of events.
4. Each consumer records `eventId` and its domain effect atomically or safely deduplicates.
5. A slow notification provider does not block checkout, but the notification SLO and retry state must be visible.
6. Outbox age, broker publish failures, consumer lag, processing failures, redeliveries, and DLQ age are monitored.
7. Replaying analytics must not resend customer notifications or create duplicate charges.

If the product requires inventory reservation to succeed before confirming the order, this sequence is insufficient by itself; define a synchronous reservation or an explicit pending/saga state machine and compensation behavior.

## Pattern selection from the Microsoft catalog

Select a pattern because its problem statement fits:

- [Queue-Based Load Leveling](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling) — buffer a burst between producer and constrained worker.
- [Competing Consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers) — add parallel workers to a shared work channel; preserve idempotency and downstream capacity.
- [Publisher-Subscriber](https://learn.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber) — distribute events to independent subscribers without direct producer-to-consumer calls.
- [Claim Check](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check) — keep a large payload in external storage and send a controlled reference through the broker.
- [Asynchronous Request-Reply](https://learn.microsoft.com/en-us/azure/architecture/patterns/asynchronous-request-reply) — let a caller poll/observe a long-running operation instead of holding a synchronous connection open.
- [Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry) and [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) — address transient failure and a dependency that is persistently unhealthy; retries need deadlines and idempotency.
- [Event Sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) — persist a domain's authoritative event history; it is a separate, higher-commitment architecture, not a synonym for publishing integration events.

Patterns have trade-offs; the Azure Architecture Center explicitly recommends selecting based on workload constraints rather than desired technology.

## Questions to ask in a design review

- Is the message a command or immutable fact? Who owns its schema?
- Is the system a queue, fan-out, or retained/replayable log?
- What is the durability/ack boundary at producer, broker, and consumer?
- Which duplicates/losses/reordering can occur? How does the consumer handle them?
- What exactly does an offset/ack mean relative to the business side effect?
- What is the partition/routing key, ordering scope, and hot-key risk?
- How are schema changes rolled out and old messages replayed?
- What are retention, PII/deletion, message-size, and DLQ policies?
- What are backlog/freshness limits, backpressure behavior, and downstream capacity limits?
- How do operators trace one event across producer, broker, consumer, and storage?

## Further reading

Inspected on **2026-11-06**:

- [Apache Kafka 4.4 introduction](https://kafka.apache.org/44/getting-started/introduction/) links to its [quick start](https://kafka.apache.org/44/getting-started/quickstart/), [documentation/design section](https://kafka.apache.org/44/documentation/#design), [semantics](https://kafka.apache.org/44/documentation/#semantics), [Kafka Streams overview](https://kafka.apache.org/44/streams/), [Streams introduction](https://kafka.apache.org/44/streams/introduction/), and [Streams core concepts](https://kafka.apache.org/44/streams/core-concepts/). The introduction covers producers/consumers, topics, keys, partitions, retention, replication, APIs, Connect, and Streams. The `/44/documentation/` page exposed only a footer in this research fetch, so the core claims here rely on the accessible 4.4 introduction, quickstart, and Streams pages.
- [RabbitMQ tutorials](https://www.rabbitmq.com/tutorials) groups examples by AMQP 1.0, AMQP 0-9-1, and Streams; the current page says the tutorials target RabbitMQ 4.x. The [Python Hello World](https://www.rabbitmq.com/tutorials/tutorial-one-python), [work queue](https://www.rabbitmq.com/tutorials/tutorial-two-python), [publish/subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-python), [routing](https://www.rabbitmq.com/tutorials/tutorial-four-python), [topics](https://www.rabbitmq.com/tutorials/tutorial-five-python), and [RPC](https://www.rabbitmq.com/tutorials/tutorial-six-python) tutorials build up the AMQP 0-9-1 model.
- RabbitMQ [reliability](https://www.rabbitmq.com/docs/reliability), [publisher confirms and consumer acknowledgements](https://www.rabbitmq.com/docs/confirms), [queues](https://www.rabbitmq.com/docs/queues), [publishers](https://www.rabbitmq.com/docs/publishers), [consumers](https://www.rabbitmq.com/docs/consumers), and [Streams](https://www.rabbitmq.com/docs/streams) give the operational safety model, delivery confirmations, prefetch, replay, and queue/stream distinctions.
- The [Microsoft Cloud Design Patterns catalog](https://learn.microsoft.com/en-us/azure/architecture/patterns/) links the message and resilience patterns summarized above. Its pattern pages explain problem fit, considerations, and trade-offs, with Azure examples.
- [System Design Primer](https://github.com/donnemartin/system-design-primer) is an interview-oriented breadth reference for queues, messaging, availability, consistency, and system design.

