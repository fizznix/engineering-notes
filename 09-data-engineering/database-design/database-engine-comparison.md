# Database engine comparison: PostgreSQL, MySQL/InnoDB, SQL Server, and SQLite

[Database design hub](README.md) · [Relational database design](relational-database-design.md) · [PostgreSQL administration](postgresql-administration.md) · [Database internals](database-internals.md)

Compare database engines by their guarantees and operational behavior, not by slogans. “Fast,” “scalable,” and “SQL-compliant” are incomplete without a workload, dataset, version, configuration, and failure model. The same logical schema can have different physical behavior across engines.

The comparison below is a learning map, not an exhaustive product matrix. Defaults, optional features, licensing, cloud services, and engine capabilities change; verify the exact deployed version and edition.

## High-level orientation

| Engine | Architecture and storage orientation | Concurrency/recovery concepts to study | Common learning fit |
| --- | --- | --- | --- |
| PostgreSQL | Client/server relational engine with integrated table storage and multiple index access methods. | MVCC, WAL, vacuum, planner statistics, physical streaming and logical replication. | General-purpose application database and a strong primary engine for hands-on SQL/operations study. |
| MySQL with InnoDB | Client/server SQL engine; InnoDB is the commonly used transactional storage engine and organizes table rows around its clustered primary-key index. | InnoDB MVCC, redo/undo logging, buffer pool, locking, replication modes, and engine-specific configuration. | Existing MySQL application ecosystems and workloads built around its operational model. |
| Microsoft SQL Server | Client/server relational platform with T-SQL and integrated storage, transaction logging, optimization, and management features. | Transaction log, locking and row-versioning options, Query Store, recovery model, and availability features. | Microsoft/.NET environments and organizations standardized on SQL Server tooling and operations. |
| SQLite | Embedded library/database file; there is no separate database server process in the usual deployment model. | Locking and journaling; WAL mode allows reader/writer overlap under its constraints, while writes still serialize. | Local, embedded, mobile, desktop, test, and small-service workloads where deployment simplicity is valuable. |

This table intentionally avoids ranking performance. A typical use case is not a guarantee: benchmark the real schema, queries, concurrency, durability settings, and hardware.

## What differs in practice

### Transaction and concurrency behavior

All four support transactions, but their isolation levels, locking behavior, snapshot semantics, retry conditions, and defaults differ. A transaction pattern that works in one engine may need a different isolation setting or retry strategy in another. Read each engine's isolation documentation and test concurrent cases such as duplicate insert, lost update, write skew, and deadlock.

SQLite is not a networked server with many concurrent writer processes. In WAL mode, readers and a writer can often proceed concurrently, but there is still only one writer at a time for a database, and its WAL shared-memory requirements constrain use to processes on the same host. This makes SQLite excellent for many local workloads, but not a drop-in replacement for a client/server cluster.

### Storage, logging, and indexes

- PostgreSQL commonly stores table rows in heap pages and maintains separate index structures; its WAL supports crash recovery, archiving/PITR, and physical replication.
- InnoDB's clustered-index organization means the primary key is part of the table's physical organization; secondary index design and primary-key size therefore have storage and access consequences.
- SQL Server's transaction log is central to commit/recovery and participates in backup/recovery models; row-versioning behavior is a configurable feature, not a safe blanket description of every workload.
- SQLite stores a database in a file and supports rollback-journal and WAL modes. WAL has operational constraints and is not simply “PostgreSQL WAL in a file.”

PostgreSQL and SQLite documentation are directly accessible in the cited references below. The automated research fetch for the MySQL manual returned HTTP 403, so the MySQL entries above are conservative orientation notes and should be checked against the exact [InnoDB documentation](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html). The previously supplied SQL Server transaction-log URL was stale; the corrected Microsoft Learn URL is linked below.

### SQL dialect and application portability

Core `SELECT`, joins, grouping, and transactions are broadly familiar, but application code can depend on dialect details:

- Pagination: `LIMIT`/`OFFSET`, `TOP`, or `FETCH` syntax and tie semantics.
- Upsert/merge: supported forms and conflict/concurrency behavior differ.
- Changed rows: `RETURNING`, `OUTPUT`, or another mechanism.
- Type systems: date/time, booleans, JSON, arrays, identity columns, and collation behavior differ.
- DDL and migrations: transactional DDL and online index/schema-change behavior differ.
- Query plans and hints: explain tools, optimizer controls, and plan stability differ.
- Connection model, replication, backup, and failover are not portable SQL concerns.

Use a query builder or ORM only with a clear view of generated SQL and engine-specific semantics. Avoid claiming portability based on syntax alone: constraints, null behavior, transaction isolation, and collation can change results.

## A fair comparison exercise

Use the same small application model (for example, customers, orders, and order items) and define one representative workload:

1. Create equivalent constraints and seed the same row counts/distribution.
2. Implement point lookup, recent-orders pagination, an aggregate report, and one multi-table transaction.
3. Confirm outputs match on empty sets, nulls, duplicate keys, and concurrent updates.
4. Add the same logical index hypothesis, adapting syntax as needed.
5. Record the plan, latency distribution, throughput, storage size, write cost, and settings.
6. Test backup/restore and replication/failover behavior separately; a query benchmark does not establish operational readiness.

Do not compare a default SQLite file to a tuned server cluster and conclude one product “wins.” The result only describes those particular setups and workloads.

## Recommended learning order

1. Learn SQL and relational modeling using [SQL fundamentals and practice](sql-fundamentals-and-practice.md) and [Relational database design](relational-database-design.md).
2. Use PostgreSQL as the first hands-on engine; learn [administration](postgresql-administration.md) and [query plans](query-performance-labs.md).
3. Compare one feature at a time: transaction behavior, index/storage organization, logging/recovery, query plans, then operations.
4. Learn another engine when a real project or interview target requires it; read its official versioned documentation and validate behavior with a small experiment.

## Official documentation and references

Inspected on **2026-11-06**:

- [PostgreSQL documentation](https://www.postgresql.org/docs/current/) — [indexes](https://www.postgresql.org/docs/current/indexes.html), [MVCC](https://www.postgresql.org/docs/current/mvcc.html), [WAL](https://www.postgresql.org/docs/current/wal-intro.html), [backup/restore](https://www.postgresql.org/docs/current/backup.html), and [replication](https://www.postgresql.org/docs/current/warm-standby.html).
- [MySQL Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/) — [InnoDB storage engine](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html) and [InnoDB introduction](https://dev.mysql.com/doc/refman/8.4/en/innodb-introduction.html). The manual's pages returned HTTP 403 to the research fetcher; the official browser links remain the canonical reference.
- [Microsoft SQL documentation](https://learn.microsoft.com/en-us/sql/?view=sql-server-ver17) — [SQL Server transaction-log architecture and management](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver17).
- [SQLite documentation](https://www.sqlite.org/docs.html) — [architecture](https://www.sqlite.org/arch.html), [WAL](https://www.sqlite.org/wal.html), and [isolation](https://www.sqlite.org/isolation.html).

## Related notes

- [Database internals](database-internals.md) — storage and recovery concepts behind these products.
- [PostgreSQL administration](postgresql-administration.md) — the recommended first engine for hands-on operational practice.
- [Data warehousing and OLAP](data-warehousing-and-olap.md) — analytical engines and formats beyond transactional databases.
