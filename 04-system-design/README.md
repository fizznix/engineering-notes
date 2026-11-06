# System Design

[Back to the repository index](../README.md)

Understand how to design systems that meet functional and operational requirements, and explain their trade-offs.

## Learning sequence

Start with requirements and capacity estimation; work through APIs and data modeling, then caching, queues, scaling, consistency, reliability, and observability.

## Topics

- Requirements and capacity estimation
- APIs, data modeling, and architecture decision records
- Caching, queues, load balancing, partitioning, and replication
- Consistency, reliability, observability, and security
- High-level and low-level design

## Related domains

[Foundations](../01-foundations/README.md) · [Data engineering](../09-data-engineering/README.md) · [Cloud and DevOps](../10-cloud-devops/README.md) · [Security](../11-security/README.md)

## Notes

- [System design and architecture: the evolutionary approach](evolutionary-system-scaling.md) — a video-based progression from a single server through horizontal scaling, shared state, database scaling, caching, asynchronous processing, and sharding.
- [Database design and data stores](../09-data-engineering/database-design/README.md) — modeling, indexes, transactions, NoSQL, Redis, and database operations.
- [Messaging and event streaming](../09-data-engineering/messaging-and-event-streaming.md) — queues versus retained logs, Kafka/RabbitMQ, delivery semantics, retries, and overload behavior.
- [Observability and telemetry](../10-cloud-devops/observability-and-telemetry.md) — OpenTelemetry, Prometheus, SRE signals/SLOs, and actionable alerting.
- [HLD interview guide](../14-interview-prep/system-design-interviews/hld-interview-guide.md) — requirements, estimation, interfaces, architecture, trade-offs, and a worked URL-shortener design.
- [HLD and LLD interview preparation hub](../14-interview-prep/system-design-interviews/README.md) — a complete study sequence and links to both interview guides.
