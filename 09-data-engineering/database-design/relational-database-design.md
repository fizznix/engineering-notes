# Relational database design

[Database design hub](README.md) · [Database selection and operations](database-selection-and-operations.md) · [HLD interview guide](../../14-interview-prep/system-design-interviews/hld-interview-guide.md)

Relational design is not merely choosing columns. It is the discipline of stating which facts exist, which relationships and constraints must always hold, how concurrent changes are serialized, and which queries the database must support efficiently.

## Start with the workload and invariants

Before creating tables, write down:

- **Entities and ownership:** customer, order, product, inventory, payment; which service owns each?
- **Lifecycle:** created, pending, confirmed, cancelled, archived; which transitions are valid?
- **Invariants:** unique customer email within a tenant, positive quantity, one active reservation, captured unit price preserved for an order.
- **Access paths:** point lookup, list by owner and time, join, aggregate, range query, report.
- **Rate and shape:** reads/writes per second, typical rows returned, row size, burst, retention, growth, and latency target.
- **Correctness requirements:** which operations must be atomic and what stale data is acceptable?

A schema optimized for a query that is not important is still a poor schema. Include the hot paths and transaction boundaries in design notes.

## Model facts and relationships

### Tables, keys, and relationships

- Give each table a primary key that is stable for the entity's lifetime. A surrogate key is often practical; a natural key can still have a `UNIQUE` constraint.
- Do not use a mutable attribute (email, display name, phone number) as a long-lived foreign key.
- Use a foreign key to encode required referential integrity; decide deliberately whether deletion should be restricted, cascaded, or represented as a lifecycle/state change.
- Represent many-to-many relationships with a join table. Put attributes of the relationship (such as enrollment date or role) on that table.
- Use `NOT NULL`, `UNIQUE`, `CHECK`, primary-key, and foreign-key constraints to enforce invariants at the shared data boundary, not just in one application code path.
- Name important constraints so errors and migrations are understandable.

### Normalization: why and how far

Normalization reduces **update anomalies** by storing each fact in one canonical place:

- **Insertion anomaly:** a fact cannot be stored without inventing an unrelated row.
- **Update anomaly:** the same fact is copied into several rows and only some copies are changed.
- **Deletion anomaly:** deleting one fact accidentally removes another fact that should remain.

In ordinary transactional applications, begin with a normalized model:

- One row represents one entity or relationship.
- Columns contain values for that row's key, not repeating groups.
- Non-key facts depend on the key, the whole key, and nothing but the key (a useful shorthand for the goals of 2NF/3NF).
- Separate independently changing facts and relate them with keys.

You do not need to chase formal normal forms mechanically. The practical questions are: what does this value describe, which key determines it, can it change independently, and what must be updated atomically?

### Intentional denormalization

Denormalize only to serve a real read path or historical invariant:

- **Snapshot facts:** an order item can preserve the product name/SKU/price as it was at purchase, even though the current product row changes.
- **Precomputed aggregate:** a count or summary can avoid an expensive repeated computation if its update/reconciliation rules are explicit.
- **Read model:** a search/feed projection can be copied from authoritative records and rebuilt from an event log or source tables.

For every copied value, record the source of truth, update mechanism, maximum staleness, repair/rebuild path, and behavior when the update fails. “Avoid joins” alone is not a reason to duplicate every column.

## Types, nullability, and time

- Choose types that represent the domain and fit operations. Avoid storing dates, amounts, booleans, or numbers as arbitrary strings.
- Use `NOT NULL` when absence is not a valid domain state. SQL `NULL` means unknown/absent and changes comparison and uniqueness behavior; it is not the same as an empty string or zero.
- Store event instants with an explicit time-zone strategy (commonly `timestamptz` in PostgreSQL); keep a separate IANA time-zone identifier when future local-calendar interpretation matters.
- Avoid floating-point values for currency. Use a fixed-scale decimal for an explicitly bounded currency set or integer minor units with currency and exponent handled correctly.
- Use JSON/JSONB for genuinely variable or sparse attributes when its query/update behavior is understood—not to bypass all schema design.
- Keep large media/blob payloads in object storage when their size/lifecycle/access patterns make relational rows a poor fit; store durable metadata and an object reference in the database.

## Example: orders with preserved purchase facts

This simplified PostgreSQL schema assumes a single currency convention for prices. Real multi-currency systems must define currency, decimal exponent, rounding, tax, and exchange-rate rules explicitly.

```sql
CREATE TABLE customer (
    customer_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT customer_email_uq UNIQUE (email)
);

CREATE TABLE product (
    product_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku text NOT NULL,
    name text NOT NULL,
    current_price numeric(12, 2) NOT NULL CHECK (current_price >= 0),
    CONSTRAINT product_sku_uq UNIQUE (sku)
);

CREATE TABLE sales_order (
    order_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL
        REFERENCES customer(customer_id) ON DELETE RESTRICT,
    status text NOT NULL
        CHECK (status IN ('pending', 'confirmed', 'cancelled', 'fulfilled')),
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sales_order_item (
    order_id bigint NOT NULL
        REFERENCES sales_order(order_id) ON DELETE CASCADE,
    line_number integer NOT NULL CHECK (line_number > 0),
    product_id bigint NOT NULL
        REFERENCES product(product_id) ON DELETE RESTRICT,
    sku_at_purchase text NOT NULL,
    name_at_purchase text NOT NULL,
    quantity integer NOT NULL CHECK (quantity > 0),
    unit_price_at_purchase numeric(12, 2) NOT NULL
        CHECK (unit_price_at_purchase >= 0),
    PRIMARY KEY (order_id, line_number)
);

CREATE INDEX sales_order_customer_recent_idx
    ON sales_order (customer_id, created_at DESC, order_id DESC);
```

Why this shape:

- An order belongs to one customer; the foreign key prevents an order from pointing to a nonexistent customer.
- Order items are a separate relation because an order has multiple lines; their composite key makes line identity explicit.
- The item stores purchase-time descriptive and price facts. Changing the current product price does not rewrite a historical order.
- The order list index matches the query that filters by customer and sorts newest first. The primary key also breaks timestamp ties for deterministic pagination.
- `ON DELETE` expresses lifecycle policy. Customer deletion is restricted; deleting an order removes its owned lines. Production retention, audit, and legal requirements may instead require immutable records or soft deletion.

For stock reservation, a conditional update can make the check-and-decrement atomic:

```sql
UPDATE inventory
SET available_quantity = available_quantity - $2
WHERE product_id = $1
  AND $2 > 0
  AND available_quantity >= $2
RETURNING available_quantity;
```

Assume `available_quantity` also has a database constraint preventing negative values. If no row is returned, the product is absent, the requested quantity was invalid, or stock was insufficient. This avoids the unsafe application sequence “read quantity, then write a lower value” in which two requests can both see the same stock. The larger order/inventory workflow still needs a transaction boundary and an explicit reservation/compensation policy.

## Constraints are executable business rules

Use database constraints for rules that must hold regardless of which application, job, or migration writes the row:

- `NOT NULL` for required values.
- `UNIQUE` for identifiers that must not collide (include tenant columns when uniqueness is tenant-scoped).
- `CHECK` for row-local bounds and enumerated states.
- `FOREIGN KEY` for relationships that must refer to existing rows.
- Exclusion/other specialized constraints when a domain rule requires them and the database supports them.

Application validation is still useful for friendly errors and policy, but it does not prevent concurrent writers or alternate code paths from violating an invariant. A database constraint is a final guardrail. Avoid encoding workflow rules in a `CHECK` that reads other rows: constraints are not a substitute for a transaction, unique index, or explicit locking strategy.

## Transactions and concurrent correctness

### ACID at its actual boundary

- **Atomicity:** a transaction's database changes commit together or roll back together.
- **Consistency:** committed data satisfies declared constraints and application invariants; ACID does not invent the invariants for you.
- **Isolation:** concurrent transactions observe and modify data according to a defined concurrency model.
- **Durability:** acknowledged commits survive failures according to the database's durability and replication configuration.

Transactions should group operations that must succeed or fail together. Keep them short, avoid remote API calls inside them, and plan for serialization/deadlock errors or client retries where relevant.

### Isolation and anomalies

Be able to explain these cases:

- **Dirty read:** observe another transaction's uncommitted value.
- **Non-repeatable read:** re-read one row and see a concurrent committed update.
- **Phantom:** repeat a predicate query and see a changed qualifying row set.
- **Lost update:** two read-modify-write operations overwrite each other's result.
- **Write skew/serialization anomaly:** individually valid concurrent transactions combine into a state that could not result from any serial ordering.

Use constraints and atomic updates first for simple invariants. For more complex multi-row decisions, choose an isolation level/locking approach that actually protects the predicate. A common optimistic pattern is `UPDATE ... WHERE version = expected_version RETURNING ...`; if no row matches, reload/retry or return a conflict. Row locks can serialize a critical section but reduce concurrency and can deadlock if different code paths acquire locks in different orders.

In PostgreSQL's current documentation, `READ UNCOMMITTED` behaves like `READ COMMITTED`; `REPEATABLE READ` prevents phantom reads in PostgreSQL but serialization anomalies remain possible; `SERIALIZABLE` aims to make committed transactions equivalent to some serial order and may abort a transaction that must be retried. Consult the manual for the exact release and behavior.

### MVCC and maintenance (PostgreSQL)

PostgreSQL uses multiversion concurrency control (MVCC): readers and writers can often proceed without blocking each other by observing appropriate row versions. Updates/deletes leave versions that require cleanup. Vacuuming reclaims/reuses space, updates planner statistics/visibility information, and helps prevent transaction-ID wraparound. Autovacuum is important routine maintenance, not an optional optimization to disable casually.

Long-running transactions can retain old row versions and interfere with cleanup. Monitor transaction age, vacuum progress, dead tuples/bloat indicators, locks, and query duration. Do not prescribe `VACUUM FULL` as routine cleanup; it takes a stronger lock and has different operational consequences.

## Indexes: derive them from queries

An index is a second structure maintained on writes. It can reduce rows/pages examined, support ordering, or enforce uniqueness; it can also consume storage/cache, slow inserts/updates/deletes, and make planning more complex.

### A query-to-index workflow

1. Capture the actual query and representative parameter values.
2. State the expected rows scanned and returned, including selectivity and sort order.
3. Identify equality filters, range filters, joins, and ordering.
4. Propose the smallest useful index and check whether an existing primary/unique index already serves the query.
5. Run `EXPLAIN`; where safe, use `EXPLAIN (ANALYZE, BUFFERS)` on representative data.
6. Check estimates vs actual rows, scan/join/sort nodes, buffer reads, and total workload impact.
7. Measure the write/storage cost and re-check after data distribution changes.

A sequential scan is not automatically a problem. It can be cheaper for a small table, a query returning a large fraction of rows, or an index whose random lookups cost more than scanning.

### Common PostgreSQL index choices

| Type | Typical fit | Caution |
| --- | --- | --- |
| B-tree | Equality/range comparisons and ordered retrieval; general starting point | Multi-column order matters; index may not help a low-selectivity predicate. |
| GIN | Inverted lookup such as array, JSONB, or text-search operators | Larger/write-heavier; operator class and exact query operators matter. |
| GiST / SP-GiST | Extensible search structures such as ranges, geometry, and specialized operators | Behavior depends on operator class and data type. |
| BRIN | Very large tables whose values correlate with physical row order, such as append-time data | Lossy page-range filtering; not a general replacement for B-tree. |
| Hash | Equality-only lookups in supported cases | B-tree is usually the default; benchmark before choosing a specialized type. |

### Composite, partial, expression, and covering indexes

- **Composite B-tree:** for `(tenant_id, status, created_at)`, leading equality columns followed by a range/order column often match a query like `WHERE tenant_id = ? AND status = ? ORDER BY created_at`. A query constraining only a later column may not use the index as efficiently. Validate against real predicates and distributions.
- **Partial:** index only rows that satisfy a stable predicate (for example, active/pending records) when that reduces size and matches important queries. The planner must be able to prove the query predicate implies the index predicate.
- **Expression:** index a transformed value only when queries use the corresponding expression, such as a normalized/lowercase lookup.
- **Covering/`INCLUDE`:** store extra columns in the index to support index-only scans in eligible cases; it widens the index and does not guarantee a heap-free query.
- **Unique:** use a unique index/constraint as the concurrency-safe uniqueness authority rather than “check first, insert later.”

Do not index every column, every foreign key, or every JSON field without a query need. Index a foreign-key column when joins/deletes/updates make it a frequent access path; PostgreSQL automatically indexes the referenced primary/unique key but does not automatically create every useful index on the referencing side.

### Read query plans safely

`EXPLAIN` displays the planner's estimated plan. `EXPLAIN ANALYZE` executes the statement and reports actual measurements, so be careful with side effects; wrap mutating experiments in a transaction and roll back if appropriate. Estimates depend on statistics, data volume, and platform-specific costs; compare realistic environments, not just a local empty table.

For lists, deterministic keyset pagination often scales more predictably than deep `OFFSET` pagination:

```sql
SELECT order_id, created_at, status
FROM sales_order
WHERE customer_id = $1
  AND (created_at, order_id) < ($2, $3)
ORDER BY created_at DESC, order_id DESC
LIMIT 50;
```

Use the last row's `(created_at, order_id)` as the next-page cursor; define behavior if rows are inserted while paging.

## Scale, migrations, and production practices

- Start with a clear relational model and a single primary database. Fix slow queries, missing/poor indexes, inefficient result sizes, and connection handling before adding sharding.
- Use read replicas only when stale-read behavior is acceptable and the primary is not the actual bottleneck for writes.
- Partition for a demonstrated query-pruning, retention, or maintenance need; partitioning is not automatically horizontal scale.
- Separate analytical scans from latency-sensitive OLTP when they contend for CPU, memory, or I/O; choose a warehouse/replica/event-derived model as appropriate.
- Use parameterized queries; keep credentials out of code; apply least-privilege roles; classify sensitive data and define retention/deletion.
- Use a connection pool with bounded capacity. Excessive concurrent database connections can increase memory/context switching and reduce throughput.
- Migrate with expand → backfill/dual-read or dual-write where necessary → switch → contract. Make backfills restartable and bounded; avoid a deploy that requires every old and new application instance to change schema atomically.
- Back up and test restore. Replication helps failover/read capacity but can replicate accidental deletion or corruption; it is not historical recovery.

## Relational design checklist

- Are each entity and authoritative fact clear?
- Are keys stable, relationships explicit, and uniqueness/validity enforced?
- Does the schema preserve historical facts that must not change with current entities?
- Are nullability, deletion, currency, time-zone, and retention semantics explicit?
- Are transaction boundaries and concurrent updates safe under retries?
- Does each important query have an index hypothesis supported by an execution plan?
- Have write amplification, index storage, connection limits, vacuum, migration, backup, restore, and access control been considered?
- Can the design be kept simple until measured workload requires a more distributed shape?

## Sources

Inspected on **2026-11-06**:

- [PostgreSQL data definition](https://www.postgresql.org/docs/current/ddl.html), [constraints](https://www.postgresql.org/docs/current/ddl-constraints.html), [date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html), [index chapter](https://www.postgresql.org/docs/current/indexes.html), [index types](https://www.postgresql.org/docs/current/indexes-types.html), [multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html), [partial indexes](https://www.postgresql.org/docs/current/indexes-partial.html), [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html), [EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html), [isolation](https://www.postgresql.org/docs/current/transaction-iso.html), [MVCC/concurrency](https://www.postgresql.org/docs/current/mvcc.html), [partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html), [vacuum](https://www.postgresql.org/docs/current/routine-vacuuming.html), and [backup/restore](https://www.postgresql.org/docs/current/backup.html).
- [System Design Primer: relational databases](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms), [replication](https://github.com/donnemartin/system-design-primer#replication), [sharding](https://github.com/donnemartin/system-design-primer#sharding), [denormalization](https://github.com/donnemartin/system-design-primer#denormalization), and [SQL tuning](https://github.com/donnemartin/system-design-primer#sql-tuning).
- [Designing Data-Intensive Applications](https://dataintensive.net/) is an optional deeper conceptual reference for comparing storage, replication, partitioning, transactions, and processing approaches; its official site is a book overview rather than a free chapter repository.
