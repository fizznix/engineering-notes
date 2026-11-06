# Query performance and database labs

[Database design hub](README.md) · [SQL fundamentals and practice](sql-fundamentals-and-practice.md) · [Relational database design](relational-database-design.md) · [Database internals](database-internals.md)

Query tuning is a measurement discipline. First make the result correct, then identify the measured bottleneck, form one hypothesis, change one thing, and compare under controlled conditions. An index is a trade-off, not an automatic speed switch.

## Read a PostgreSQL plan

Start with:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT order_id, created_at, status
FROM sales_order
WHERE customer_id = 42
ORDER BY created_at DESC, order_id DESC
LIMIT 50;
```

`EXPLAIN` displays the planner's chosen plan. `EXPLAIN ANALYZE` executes the statement and reports observed timing/row information, so it can have side effects and can be expensive. Use a disposable dataset or wrap a test write in a transaction that is rolled back; understand that external effects are not rolled back.

Read a plan from the leaf scans upward:

| Plan item | Questions to ask |
| --- | --- |
| Scan: sequential, index, index-only, bitmap | How many rows/pages are visited and returned? Is the predicate selective? Is the choice reasonable for the fraction of the table requested? |
| Join: nested loop, hash, merge | What are the input sizes? How many times does a node loop? Is a nested loop repeatedly probing a large relation? Is a hash spilling? |
| Sort / aggregate | How many rows are sorted/grouped? Is the sort in memory or spilling? Could an index provide useful ordering? |
| Estimated vs actual rows | Where does cardinality estimation diverge? Is analysis/statistics current? Is data skew or correlation relevant? |
| Buffers | How many shared blocks were hits versus reads? A buffer read is not necessarily a physical disk read because the OS can cache files. |
| `loops`, filters, rows removed | Is repeated work or an unexpectedly broad filter dominating? |

Costs shown by the planner are estimates in planner units, not milliseconds. Actual time depends on hardware, cache state, contention, and measurement conditions. A sequential scan can be the best plan for a large fraction of a table; an index can add random access and maintenance cost.

## Index experiments

Use a representative data distribution and write down a hypothesis before adding an index.

| Experiment | Learn |
| --- | --- |
| Run a selective filter with no supporting index | Baseline scan and returned-row count. |
| Add a B-tree index on the filter key | Whether the plan changes and whether actual time/buffers improve for this selectivity. |
| Filter on a low-cardinality/skewed value | Why the optimizer may correctly choose a sequential scan. |
| Add a composite index for equality + range/order | How key order and the actual predicate shape affect use. |
| Add included columns for a covering hypothesis | Whether an index-only scan is possible; visibility state and selectivity still matter. |
| Join two increasing-size tables | How cardinality changes join algorithm and memory needs. |
| Add an extra index, then benchmark writes | The impact of maintaining additional index structures. |

For a composite B-tree, think about the leading columns and the query's equality/range/order conditions, then verify with the target engine's rules and plan. Do not mechanically put “most selective first” in every index: index order must match the filter/order patterns and can affect multiple queries. Partial and expression indexes can help a specific predicate, but must be maintained and their predicates must match what the planner can prove.

### Optional reproducible PostgreSQL dataset

Run only in a local/disposable database. Scale the row count down for a constrained machine.

```sql
CREATE TABLE event_sample (
    event_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id integer NOT NULL,
    occurred_at timestamptz NOT NULL,
    payload text NOT NULL
);

INSERT INTO event_sample (tenant_id, occurred_at, payload)
SELECT
    (g % 1000)::integer,
    timestamptz '2024-01-01 00:00:00+00' + g * interval '1 second',
    repeat('x', 100)
FROM generate_series(1, 100000) AS g;

ANALYZE event_sample;
```

Compare the query before and after creating a composite index:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT event_id, occurred_at
FROM event_sample
WHERE tenant_id = 42
  AND occurred_at >= timestamptz '2024-01-01 00:10:00+00'
ORDER BY occurred_at DESC
LIMIT 100;

CREATE INDEX event_sample_tenant_time_idx
    ON event_sample (tenant_id, occurred_at DESC);

ANALYZE event_sample;
```

Do not assume which plan or speedup will result. The point is to observe scan type, actual rows, buffers, and timing for this dataset and environment.

## Query-shape experiments

- **Join fan-out:** add a one-to-many child relation and compare row counts before/after each join. Check whether an aggregate must happen before the join.
- **Pagination:** compare a large `OFFSET` with a stable keyset cursor; use a deterministic sort and ensure the cursor columns form an unambiguous order. See [relational database design](relational-database-design.md#indexes-derive-them-from-queries).
- **Statistics:** compare plans before and after `ANALYZE` following a significant data load or distribution change. Statistics reduce estimation error but are not a perfect description of every correlation.
- **Partitioning:** partition only when a pruning, retention, or maintenance need is demonstrated. Compare scans with and without a predicate that enables pruning; partitioning adds planning and operational complexity.
- **Memory settings:** test sort/hash behavior with a controlled workload and bounded concurrency. `work_mem` can be used by multiple operations in multiple sessions, so raising it globally can cause memory pressure.
- **Concurrency:** run multiple clients and inspect throughput, p95/p99 latency, lock waits, deadlocks, and failures—not just single-query elapsed time.
- **Vacuum and visibility:** compare plans and visibility-related behavior on a test table before/after maintenance; do not treat manual vacuuming as an index-tuning substitute.

## A controlled measurement protocol

1. Define the question and success measure (for example, p95 latency or throughput at a fixed concurrency).
2. Record database version, hardware/container limits, settings, schema, row count, data distribution, and indexes.
3. Run a baseline with representative parameters. Separate connection setup from query time if that matters to the application.
4. Warm up if measuring steady state; if cold-start behavior matters, measure it separately. Do not call a run “cold” without controlling the relevant caches.
5. Run multiple repetitions; compare distributions, not one lucky timing. Record errors/timeouts as results.
6. Change one variable at a time and rerun the same workload.
7. Check result correctness and write cost, storage cost, plan changes, and operational impact along with latency.
8. Preserve scripts and results so another person can reproduce the experiment.

`pgbench` can run a default TPC-B-like workload or custom SQL scripts with concurrent sessions. It is a tool for controlled load testing, not a universal benchmark or proof that an application workload is represented.

## Datasets and a reusable lab repository

Start with synthetic data when you need exact control over cardinality and skew. Then use a public dataset to practice import, realistic joins, date filters, and larger scans. Begin with a bounded sample; public datasets can be large, change layout, and have their own licensing/usage terms.

| Dataset/resource | Useful exercises | Caveat |
| --- | --- | --- |
| [NYC TLC trip-record data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Date filtering, time-series aggregation, window functions, partitioning, and repeated scans of a large fact-like table. | The official page returned HTTP 403 to the research fetcher; verify the current data dictionaries, file format, and terms on the source site. |
| [Olist Brazilian e-commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) | Multi-table joins, order lifecycle analysis, customer segmentation, and revenue aggregation. The Kaggle page describes a public dataset with about 100,000 orders and product/customer/review information. | Review the dataset license and data dictionary; do not assume columns or meanings from a third-party notebook. |
| [TPC-H benchmark specification](https://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp?mode=TPC-MEMBER) | Study a defined analytical workload with complex joins and aggregates. The TPC page listed TPC-H specification version 3.0.1 when checked. | Use it as a learning workload; do not claim a compliant TPC result unless all applicable rules are followed. |

A small repository makes experiments reviewable:

```text
postgres-performance-lab/
  datasets/       source, license, import notes, and generated-data parameters
  schema/         table and index definitions
  queries/        joins, aggregates, windows, and application-shaped queries
  plans/before/   baseline plans, with sensitive values removed
  plans/after/    plans after one controlled change
  experiments/    setup and one-variable-at-a-time procedures
  reports/        measurements, environment, findings, and limitations
```

Keep the data source and its license notes with the project. Store only a manageable sample when possible, and never publish personal or sensitive data or unredacted production query plans.

### Experiment record template

```text
Question / hypothesis:
Database and version:
Machine / container limits:
Schema and data volume/distribution:
Query and representative parameters:
Concurrency / duration / repetitions:
Settings and indexes:
Plan notes (estimated vs actual rows, loops, buffers, spills):
Latency distribution / throughput / failures:
Correctness check:
Before vs after:
Limitations and next experiment:
```

## Safety and privacy

- Run experiments against a local or approved non-production database.
- `EXPLAIN ANALYZE` executes the target statement; do not run it blindly on mutating or expensive statements.
- Do not paste production query plans into third-party visualizers without reviewing literals, identifiers, schema names, and other sensitive information. Redact first or keep the plan local.
- Avoid presenting one run as a performance guarantee. Results are specific to data shape, parameters, cache, hardware, concurrency, and database version.

## Sources and tools

Checked on **2026-11-06**:

- [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) — official explanation of plan nodes and estimates/actuals.
- [PostgreSQL index documentation](https://www.postgresql.org/docs/current/indexes.html), [multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html), [partial indexes](https://www.postgresql.org/docs/current/indexes-partial.html), and [index-only scans](https://www.postgresql.org/docs/current/indexes-index-only-scans.html).
- [PostgreSQL monitoring](https://www.postgresql.org/docs/current/monitoring.html) and [routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html).
- [PostgreSQL pgbench](https://www.postgresql.org/docs/current/pgbench.html) — repeatable concurrent workloads and custom scripts.
- [PostgreSQL Wiki: Using EXPLAIN](https://wiki.postgresql.org/wiki/Using_EXPLAIN) and [performance analysis tools](https://wiki.postgresql.org/wiki/Performance_Analysis_Tools).
- [pgMustard EXPLAIN guide](https://www.pgmustard.com/docs/explain/analyze) — third-party plan-reading explanations.
- [Dalibo Explain visualizer](https://explain.dalibo.com/) — optional plan visualization; review and redact sensitive plan text before uploading.
- [NYC TLC trip-record data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page), [Olist Brazilian e-commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), and [TPC-H specifications](https://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp?mode=TPC-MEMBER) — optional real/benchmark data sources. The NYC page was blocked to the research fetcher; consult its current dictionary and terms directly.

## Related notes

- [Relational database design](relational-database-design.md) — query-to-index workflow and schema choices.
- [Database internals](database-internals.md) — pages, buffers, index structures, and optimizer basics.
- [PostgreSQL administration](postgresql-administration.md) — monitoring, configuration, vacuum, and pgbench context.
