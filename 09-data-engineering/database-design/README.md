# Database design and data stores

[Back to Data Engineering](../README.md) · [Foundations](../../01-foundations/README.md) · [System Design](../../04-system-design/README.md) · [Repository index](../../README.md)

Database design is the work of deciding **what facts the system owns, how they are represented, which operations must be correct, how those operations are queried, and what happens as data and traffic grow**. A good design is not the one with the most indexes, denormalization, or database products. It is the simplest model that preserves required invariants and serves measured access patterns with acceptable operational cost.

## Guide map

| Note | Read it for |
| --- | --- |
| [SQL fundamentals and practice](sql-fundamentals-and-practice.md) | SQL from filtering and joins through aggregation, windows, CTEs, transactions, interview problem solving, and practice resources. |
| [Relational database design](relational-database-design.md) | Modeling entities and relationships, normalization, constraints, transactions, indexes, query plans, PostgreSQL-specific behavior, and a worked order/inventory schema. |
| [Document database design](document-database-design.md) | MongoDB schema design, embedding versus references, atomicity, indexes, consistency, and sharding. |
| [Redis data design](redis-data-design.md) | Choosing Redis data structures and designing caches, counters, queues, TTLs, memory limits, persistence, and locks safely. |
| [Database selection and operations](database-selection-and-operations.md) | How to choose a store from requirements, scale deliberately, plan backups/migrations/security, and review a design. |
| [Database internals](database-internals.md) | Pages, buffer pools, B-tree and LSM storage, WAL, MVCC, recovery, and query execution. |
| [PostgreSQL administration](postgresql-administration.md) | Roles, configuration, capacity, backup/restore, PITR, replication, failover, monitoring, maintenance, and safe local labs. |
| [Database engine comparison](database-engine-comparison.md) | A careful, version-aware comparison of PostgreSQL, MySQL/InnoDB, SQL Server, and SQLite. |
| [Query performance labs](query-performance-labs.md) | EXPLAIN plans, index and query experiments, pgbench, measurement protocols, and a reproducible lab record. |
| [Data warehousing and OLAP](data-warehousing-and-olap.md) | OLTP/OLAP, ETL/ELT, dimensional modeling, columnar execution, Parquet, Iceberg, DuckDB, ClickHouse, and a capstone project. |

For the distributed-systems context behind replication, partitioning, transactions, and trade-offs, see [Designing Data-Intensive Applications](https://dataintensive.net/) and the [HLD interview guide](../../14-interview-prep/system-design-interviews/hld-interview-guide.md).

## Suggested learning path

1. Learn query structure and output grain with [SQL fundamentals and practice](sql-fundamentals-and-practice.md).
2. Design a relational schema and understand transaction invariants with [Relational database design](relational-database-design.md).
3. Study pages, indexes, WAL, MVCC, and the optimizer with [Database internals](database-internals.md).
4. Operate a local PostgreSQL instance and prove backup/restore behavior with [PostgreSQL administration](postgresql-administration.md).
5. Inspect and measure real plans with [Query performance labs](query-performance-labs.md).
6. Compare engines and then learn analytical modeling with [Database engine comparison](database-engine-comparison.md) and [Data warehousing and OLAP](data-warehousing-and-olap.md).

This sequence is flexible; database internals can be studied in much greater depth through the CMU course linked in that guide. The expanded source roadmap provided for this work is retained at [design-pattern.md](../../design-pattern.md).

## A repeatable design method

Use this sequence before choosing a product or writing DDL:

1. **Name the facts and owners.** Which service is authoritative for each fact? Which facts are derived, cached, copied, or retained only for history?
2. **List invariants.** Examples: an email is unique within a tenant; quantity cannot be negative; an order total is the sum of its captured line prices; a reservation cannot be active twice.
3. **Write access patterns.** For each read/write: filter, sort, join/lookup, result size, latency target, rate, and consistency/freshness requirement.
4. **Choose a logical model.** Relational entities and keys, document aggregates, or another model should express the invariants and common operations naturally.
5. **Choose the correctness boundary.** Decide which changes must be atomic, how concurrent requests conflict, and what retries mean.
6. **Design physical access paths.** Add only the indexes, partitions, caches, or replicas that support required queries or capacity.
7. **Plan operations and change.** Define migration, backup/restore, access control, monitoring, retention, and failure behavior before production.
8. **Measure and revise.** Use representative data and query plans; change the design only when evidence or a changed requirement calls for it.

This order is not a one-pass waterfall. A query may reveal that a model needs a different key; a new invariant may require a transaction or schema change. The point is to make the reasoning explicit.

## Quick store-selection guide

| Store/model | Strong fit when | Cost or question to examine |
| --- | --- | --- |
| Relational database (for example, PostgreSQL) | Relationships, constraints, transactions, joins, and flexible querying are important; business facts have clear owners and invariants. | Index/write cost, schema evolution, connection/query limits, and cross-service transaction boundaries. |
| Document database (for example, MongoDB) | The application reads/writes a bounded aggregate together; document shape and lifecycle are natural; independent data can be referenced. | Unbounded arrays, duplicated facts, cross-document invariants, query/index growth, and shard-key quality. |
| Key-value/in-memory store (for example, Redis) | Very low-latency key operations, cache/session state, counters, membership, ranking, or carefully designed ephemeral coordination are needed. | Memory budget, eviction, persistence and recovery requirements, hot keys, and whether it is truly safe for the data's source-of-truth role. |
| Wide-column / distributed key-value | Very high scale with known partition-key-centered queries and predictable access paths. | Query flexibility, partition skew, consistency semantics, and operational/repair complexity. |
| Search index | Text relevance, tokenization, facets, and search-specific ranking are core requirements. | It is normally a derived search projection, not the authoritative transaction store; indexing lag and rebuild are part of the design. |
| Object storage | Large blobs, media, backups, or immutable artifacts need durable, economical storage. | Store searchable metadata separately; define access control, lifecycle, versioning, and delivery/caching. |

These are tendencies, not laws. Products continue to add features; evaluate the exact version, configuration, workload, and operational environment. “SQL cannot scale” and “NoSQL is faster” are not useful design conclusions.

## Core concepts to keep separate

- **Logical schema** describes entities, attributes, relationships, and business constraints.
- **Physical design** describes indexes, partitioning, clustering, compression, and placement.
- **Normalization** reduces update anomalies by storing each fact in a canonical place. **Denormalization** copies or precomputes facts to improve a measured read path, at the cost of synchronization and repair logic.
- **Partitioning** divides a logical table/collection into parts, often for lifecycle management or query pruning. **Sharding** distributes partitions across independently scaled nodes; vendors sometimes use these terms differently.
- **Replication** copies current data for availability/read capacity. **Backup** preserves recoverable historical state. Replicas are not a substitute for backups.
- **Consistency** is a guarantee about which committed writes a read can observe. It is distinct from durability, availability, and application-level correctness.
- **An index is another maintained data structure.** It consumes space and adds work to writes; a query plan may correctly prefer a sequential scan.
- **A transaction has a scope.** A database transaction can make database operations atomic, but not automatically make an email, message publication, remote API, or payment provider call atomic with them.

## Source map and research notes

The following links were checked on **2026-11-06**. The notes in this package summarize concepts in original language; use the vendor manuals for exact syntax and version-specific behavior. Some image-preview URLs in the supplied roadmap were proxy thumbnails rather than canonical references; the guides link to original resource pages when available. The publisher page for *Database Internals* and MySQL documentation pages returned HTTP 403 to the research fetcher, so those references are identified as not independently inspected rather than summarized as though their full content had been read.

- [Designing Data-Intensive Applications](https://dataintensive.net/) — the official site describes the book as a comparison of approaches for reliable, scalable, maintainable data systems, emphasizing how and why designs differ rather than a manual for one product. The site is a book landing page, not the full text or a complete online chapter index.
- [PostgreSQL current documentation](https://www.postgresql.org/docs/current/) — the root page was sparse in the web fetch, but these linked chapters loaded: [data definition](https://www.postgresql.org/docs/current/ddl.html), [constraints](https://www.postgresql.org/docs/current/ddl-constraints.html), [date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html), [indexes](https://www.postgresql.org/docs/current/indexes.html), [index types](https://www.postgresql.org/docs/current/indexes-types.html), [multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html), [partial indexes](https://www.postgresql.org/docs/current/indexes-partial.html), [EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html), [transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html), [concurrency control/MVCC](https://www.postgresql.org/docs/current/mvcc.html), [partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html), [backup and restore](https://www.postgresql.org/docs/current/backup.html), and [routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html). `/current/` follows the current release; confirm syntax and behavior for the deployed major version.
- [MongoDB documentation](https://www.mongodb.com/docs/) — the inspected sublinks include [data modeling](https://www.mongodb.com/docs/manual/data-modeling/), [schema design process](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/), [best practices](https://www.mongodb.com/docs/manual/data-modeling/best-practices/), [embedding](https://www.mongodb.com/docs/manual/data-modeling/embedding/), [referencing](https://www.mongodb.com/docs/manual/data-modeling/referencing/), [schema validation](https://www.mongodb.com/docs/manual/core/schema-validation/), [indexes](https://www.mongodb.com/docs/manual/indexes/), [single-document atomicity](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/), [transactions](https://www.mongodb.com/docs/manual/core/transactions/), [read consistency](https://www.mongodb.com/docs/manual/core/read-isolation-consistency-recency/), [explain](https://www.mongodb.com/docs/manual/reference/method/cursor.explain/), and [shard-key selection](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/).
- [Redis documentation](https://redis.io/docs/latest/develop/data-types/) — the supplied `/docs/latest/` root returned 404 in the fetch, so use the valid focused pages: [data types](https://redis.io/docs/latest/develop/data-types/), [compare data types](https://redis.io/docs/latest/develop/data-types/compare-data-types), [key eviction](https://redis.io/docs/latest/develop/reference/eviction/), [persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/), [transactions](https://redis.io/docs/latest/develop/using-commands/transactions/), [Streams](https://redis.io/docs/latest/develop/data-types/streams/), [expiration](https://redis.io/docs/latest/commands/expire/), [Redis Cluster scaling](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/), and [distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/). The data-type overview links individual strings, hashes, lists, sets, sorted sets, streams, JSON, geospatial, time-series, and probabilistic structures.
- [System Design Primer](https://github.com/donnemartin/system-design-primer) is an interview-oriented index with sections on [database design](https://github.com/donnemartin/system-design-primer#database), [relational databases](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms), [replication](https://github.com/donnemartin/system-design-primer#replication), [sharding](https://github.com/donnemartin/system-design-primer#sharding), [denormalization](https://github.com/donnemartin/system-design-primer#denormalization), [SQL tuning](https://github.com/donnemartin/system-design-primer#sql-tuning), [NoSQL families](https://github.com/donnemartin/system-design-primer#nosql), and [SQL vs. NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql). Use it for breadth and interview framing; verify product-specific claims against current manuals.
