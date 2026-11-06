# Data warehousing and OLAP

[Database design hub](README.md) · [Relational database design](relational-database-design.md) · [Database internals](database-internals.md) · [Messaging and event streaming](../messaging-and-event-streaming.md)

An operational database and an analytical platform answer different workload shapes. A production architecture may use both and move data between them; it should make freshness, correctness, ownership, and replay behavior explicit rather than sending every large report to the transactional primary.

## OLTP and OLAP

| Workload | Typical operation shape | Design priorities |
| --- | --- | --- |
| OLTP (online transaction processing) | Many small reads/writes; point lookups; short transactions; concurrent changes to current state. | Correct constraints, transaction latency, predictable access paths, write concurrency, and recovery. |
| OLAP (online analytical processing) | Scans across many rows/columns; grouping, joins, historical comparisons, and complex aggregations. | Scan throughput, compression, column pruning, parallel execution, data freshness, and reproducible metrics. |

These are workload tendencies, not mutually exclusive product categories. Some relational databases perform analytics well; some analytical systems support transactions. The question is whether mixed workloads compete for CPU, memory, I/O, locks, or service-level objectives, and what operating model best fits.

## Warehouse, lake, and lakehouse

- A **data warehouse** is an analytical platform that integrates and organizes data for governed queries and reporting. It may be a managed service or self-operated system.
- A **data lake** commonly stores diverse raw/curated files in object storage; schema and governance can be applied by processing/metadata systems.
- A **lakehouse** is an architectural approach that combines open file/object storage with table metadata and transactional/management capabilities. The term does not by itself guarantee one feature set.
- A **data mart** is an analytical dataset organized for a business domain or consumer group.

Keep the distinction between **file format** and **table format** clear. Parquet describes columnar files. Iceberg manages a table abstraction and metadata over data files, enabling features such as schema/partition evolution and snapshot/time-travel workflows when supported by the compute engine.

## ETL and ELT pipelines

- **ETL** transforms data before loading it to the target.
- **ELT** loads data first and transforms it in the target platform.

Both need explicit contracts and failure handling. A robust pipeline defines source-of-truth ownership, extraction cursor/watermark, idempotency, late-arriving or corrected records, schema evolution, deletion propagation, data quality checks, replay/backfill, lineage, and alerting. A pipeline that “ran successfully” can still produce incorrect totals.

Separate ingestion state from business data where useful. Make incremental loads restartable: a retry should not silently duplicate facts or skip data at the boundary. Define whether a report is based on event time, processing time, or a source's last-updated timestamp.

## Dimensional modeling

Dimensional modeling structures data around measurable business events and the descriptive context needed to analyze them.

### Start with grain

The **grain** is the precise real-world event represented by one fact row. Declare it before choosing columns. For example:

> One row in `fact_order_line` represents one product line on one accepted customer order.

That grain determines keys, valid measures, joins, and duplicate handling. If one order has three lines, the fact contains three rows; it does not represent one row per order.

### Facts and dimensions

- **Fact table:** records events or periodic snapshots and numeric measures at the declared grain. Examples include quantity, net amount, discount, and transaction count.
- **Dimension table:** provides descriptive context used to group/filter facts, such as date, product, customer, location, or channel.
- **Surrogate key:** warehouse-assigned identifier used to connect facts to a dimension version; the natural/business key remains useful for matching source records.
- **Conformed dimension:** a shared, consistently defined dimension used by multiple fact tables or business processes.

Measures can be:

- **Additive:** safely summed across all relevant dimensions (for example, order-line quantity or sales amount under a fixed currency definition).
- **Semi-additive:** additive across some dimensions but not others (for example, account balance across accounts but not across time).
- **Non-additive:** should not be summed directly (for example, ratios or percentages); store/aggregate their components and calculate the ratio at query time.

Do not sum mixed currencies without a defined conversion and reporting policy. Ratios and averages are often best computed from the underlying numerator/denominator rather than summing row-level percentages.

### Star and snowflake schemas

- A **star schema** places a central fact table next to relatively denormalized dimensions. It is straightforward for common analytical joins and business use.
- A **snowflake schema** normalizes some dimension hierarchy into related tables. It can reduce repeated attributes but adds joins and model complexity.

Neither is an absolute rule. Optimize the model for clear grain, reliable definitions, manageable updates, query performance, and the capabilities of the target platform.

### Slowly changing dimensions

Dimensions change over time. Decide whether reports need the current value or the value that was true when a fact occurred:

- **Type 1:** overwrite the old attribute. The model reports the corrected/current value, not historical versions.
- **Type 2:** add a new dimension row/version with effective dates or a current flag; facts link to the version valid at the event time. This preserves history but requires careful key assignment, late-arriving data handling, and non-overlapping validity.

The policy is a business decision. Document it per attribute/dimension, and test that historical facts resolve to the intended version.

## Columnar storage and analytical execution

Row stores group values by record; columnar formats/engines organize data by column, often in row groups or blocks. When a query needs only a few columns from a wide dataset, a column-oriented layout can avoid reading unrelated columns and can compress similar values effectively.

Key techniques:

- **Projection pushdown:** read only columns required by the query.
- **Predicate pushdown:** apply filters close to the data source when supported.
- **Columnar compression/encoding:** exploit similar values within a column or block.
- **Row groups / column chunks / pages:** Parquet organizes files into row groups; each row group contains column chunks, which contain pages. Metadata and layout influence parallelism and pruning.
- **Vectorized execution:** process batches of values through operators, improving CPU efficiency versus row-at-a-time interpretation in suitable engines.
- **Partition pruning:** skip partitions whose partition values cannot match a filter.
- **Zone maps / min-max statistics / data skipping:** use block or file metadata to skip data ranges that cannot satisfy a predicate. They help only when the data distribution and layout make the metadata selective.
- **Materialized views:** persist a computed result for faster reads, with refresh cost, freshness, and correctness trade-offs.

Columnar storage is not automatically faster for point updates or single-row lookups; it may be better for scans/aggregations. Data layout, compression, file sizes, sort order, query engine, and object-store behavior matter.

## Platforms and formats to explore

| Technology | What it is useful for learning | Important boundary |
| --- | --- | --- |
| DuckDB | Embedded analytical SQL engine; convenient for local experiments and reading Parquet. Its documentation explains vectorized execution. | Do not assume a local analytical engine is a distributed warehouse or OLTP primary. Check the current version and deployment mode. |
| ClickHouse | Column-oriented analytical database; its MergeTree family writes parts and merges them, with sorting/primary-key and partition behavior documented per engine. | Choose table ordering and engine behavior for the workload; a ClickHouse “primary key” is not equivalent to a PostgreSQL row-unique constraint. |
| BigQuery | Managed analytical platform with documented separation between storage and compute. | Cost, quotas, partitioning/clustering, and service-specific semantics matter; validate current pricing and limits. |
| Apache Parquet | Columnar file format with row groups, column chunks, pages, encodings, and compression. | A file format is not a query engine, catalog, or complete table-management protocol. |
| Apache Iceberg | Open table format and metadata model for analytical datasets; supports schema/partition evolution and snapshots through compatible engines. | Feature availability and write/read behavior depend on the engine and catalog integration. |

## Worked model: order-line analytics

Use the source transactional model from [Relational database design](relational-database-design.md#example-orders-with-preserved-purchase-facts). Define the analytical fact grain as one row per order line:

```text
fact_order_line
  order_id, line_number
  order_date_key
  customer_key
  product_key
  quantity
  unit_price_at_purchase
  gross_amount
```

Related dimensions may include:

```text
dim_date       date, week, month, quarter, fiscal_period
dim_customer   customer_key, source_customer_id, segment, region, valid_from, valid_to
dim_product    product_key, source_product_id, category, brand, valid_from, valid_to
```

Reason through the pipeline:

1. Extract confirmed orders and their lines using a restartable watermark or change feed; define what happens to later corrections/cancellations.
2. Map source customer/product IDs to dimension versions. If using Type 2 history, assign the fact to the version valid for the order's event time.
3. Compute `gross_amount` from line-level quantity and purchase-time unit price under a documented currency/rounding rule.
4. Write to an intermediate Parquet dataset or an analytical table with an idempotent key such as `(order_id, line_number)`.
5. Validate source/target row counts, duplicate keys, null foreign keys, date ranges, and reconciled monetary totals.
6. Query revenue by date, region, and product category; verify aggregation grain to prevent double counting.

This is a teaching model. Real commerce systems must specify taxes, refunds, discounts, multiple currencies, status transitions, privacy deletion, late-arriving updates, and audit requirements.

## End-to-end learning project

Build a small **transactional-to-analytics platform**:

1. Model customers, products, orders, and order lines in PostgreSQL with keys, checks, and transactions.
2. Write SQL exercises from [SQL fundamentals and practice](sql-fundamentals-and-practice.md).
3. Add a tested role policy, logical backup/restore, and a documented PITR/replication lab using [PostgreSQL administration](postgresql-administration.md).
4. Benchmark representative OLTP queries and write an experiment record using [query performance labs](query-performance-labs.md).
5. Export a bounded dataset to Parquet, load/query it with DuckDB, and compare the fact/dimension model with the normalized source.
6. Optionally compare an analytical workload on ClickHouse or a managed warehouse; document semantics, cost assumptions, data freshness, and operational trade-offs.
7. Deliver a short architecture note showing data ownership, pipeline/retry boundaries, schema evolution, quality checks, recovery, and known limitations.

Do not call the project complete solely because a query returns a result. Verify totals, duplicates, idempotent retries, late updates, access control, and restore procedure.

## Suggested 10-week study plan

This is a suggested pacing of roughly 1–2 hours per day, not a fixed-time promise. Slow down for hands-on internals or if prerequisites are new; the capstone can grow alongside the phases.

| Phase | Duration | Focus | Evidence of learning |
| --- | --- | --- | --- |
| 1 | 2 weeks | SQL fundamentals and advanced queries | Solve retrieval, joins, aggregates, windows, CTEs, and null-edge exercises; explain output grain. |
| 2 | 2 weeks | Indexes and database internals | Defend an index hypothesis; trace a page read, WAL commit, B-tree/LSM trade-off, and MVCC cleanup. |
| 3 | 2 weeks | PostgreSQL administration | Restore a backup, inspect sessions/locks, reason about replication lag, and rehearse isolated recovery. |
| 4 | 1 week | EXPLAIN ANALYZE and performance tuning | Read plan estimates/actuals, run repeated controlled experiments, and report trade-offs. |
| 5 | 1 week | Database engine comparisons | Explain one meaningful difference across PostgreSQL, MySQL/InnoDB, SQL Server, and SQLite. |
| 6 | 2 weeks | Data warehousing and OLAP | State fact grain, model dimensions/history, query Parquet, and validate analytical reconciliation. |

## Sources and further reading

Checked on **2026-11-06**:

- [Kimball Group data warehouse resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/) and [dimensional modeling techniques](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) — definitions and techniques for dimensional models.
- [dbt Learn](https://learn.getdbt.com/) — optional courses for SQL transformations, testing, documentation, and analytics engineering.
- [BigQuery overview and architecture](https://cloud.google.com/bigquery/docs/introduction) — managed analytical platform and storage/compute architecture.
- [Apache Parquet documentation](https://parquet.apache.org/docs/), especially its [concepts](https://parquet.apache.org/docs/concepts/) describing row groups, column chunks, and pages.
- [Apache Iceberg documentation](https://iceberg.apache.org/docs/latest/), including [schema and partition evolution](https://iceberg.apache.org/docs/latest/evolution/).
- [DuckDB documentation](https://duckdb.org/docs/) and [execution format/vector internals](https://duckdb.org/docs/2.0/internals/vector.html) — vectorized execution; the documentation root redirected to the versioned 2.0 documentation during research.
- [ClickHouse documentation](https://clickhouse.com/docs/) and [MergeTree engine reference](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree) — data parts, merging, sorting/primary-key behavior, and partition pruning.
- [ClickHouse introductory architecture page](https://github.com/ClickHouse/clickhouse-docs/blob/main/docs/intro.md) — a source-repository introduction to column-oriented OLAP; the official documentation site is the better current reference for versioned details.
- [PostHog: ClickHouse vs Elasticsearch](https://posthog.com/blog/clickhouse-vs-elasticsearch) — a vendor-authored comparison that illustrates the difference between analytical aggregation and full-text/search workloads; use it as an example, not a neutral benchmark.
- [PostgreSQL Exercises](https://pgexercises.com/) and [PostgreSQL SQL tutorial](https://www.postgresql.org/docs/current/tutorial.html) for the transactional/query side of the capstone.

## Related notes

- [Messaging and event streaming](../messaging-and-event-streaming.md) — event contracts, replay, and delivery semantics for data movement.
- [Observability and telemetry](../../10-cloud-devops/observability-and-telemetry.md) — metrics, logs, traces, and SLOs for pipelines and data services.
- [Database selection and operations](database-selection-and-operations.md) — operational and recovery trade-offs across stores.
