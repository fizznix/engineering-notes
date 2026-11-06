# SQL fundamentals and practice

[Database design hub](README.md) · [Relational database design](relational-database-design.md) · [Query performance labs](query-performance-labs.md)

SQL is a declarative language for describing the rows and columns a question needs. The database chooses a physical execution strategy; the person writing the query is responsible for making the result's meaning precise: its grain, filters, relationships, null behavior, ordering, and consistency boundary.

This guide is a practical path from basic retrieval to interview-level SQL. Examples use PostgreSQL syntax and the simplified order schema in [Relational database design](relational-database-design.md). Other engines share much of the SQL language but differ in syntax and features; see [Database engine comparison](database-engine-comparison.md).

## A useful mental model

SQL query results are bags (multisets), not automatically sets. A row can appear more than once unless the query or data constraints prevent duplicates. Joins can multiply rows, `UNION` removes duplicates, and `UNION ALL` preserves them. Always state the grain you intend to return, such as “one row per customer” or “one row per order line.”

The logical processing model for a typical `SELECT` is approximately:

1. `FROM` and joins construct the input relation.
2. `WHERE` filters input rows.
3. `GROUP BY` forms groups and aggregates reduce them.
4. `HAVING` filters groups.
5. `SELECT` computes output expressions.
6. `DISTINCT` removes duplicate output rows if requested.
7. `ORDER BY` orders the result.
8. `LIMIT`/`OFFSET` or a dialect equivalent restricts returned rows.

This is a reasoning model, not a promise about physical execution order. The optimizer may reorder operations as long as it preserves the query's semantics.

```sql
SELECT customer_id, created_at, status
FROM sales_order
WHERE status = 'confirmed'
ORDER BY created_at DESC, order_id DESC
LIMIT 20;
```

List projected columns explicitly in application queries. `SELECT *` is useful for exploration, but it couples callers to schema changes and may read or transfer data they do not need.

## Filtering, ordering, and `NULL`

- Use `WHERE` for row-level predicates and `HAVING` for predicates over grouped results.
- Use explicit parentheses when mixing `AND` and `OR`; do not rely on a reader remembering precedence.
- Use a stable tie-breaker in `ORDER BY` when paginating or returning top-N rows. Ordering on a non-unique timestamp alone is not deterministic.
- `NULL` means missing or unknown, not zero, an empty string, or a value that compares equal to itself. `column = NULL` is not the null test; use `IS NULL` or `IS NOT NULL`.
- Comparisons involving `NULL` can evaluate to `UNKNOWN`. A `WHERE` clause retains only rows for which its predicate is `TRUE`.
- `COUNT(*)` counts rows; `COUNT(expression)` counts non-null expression values. Most aggregates ignore null inputs, and `SUM` over no non-null values returns `NULL`, so use `COALESCE` only when zero is the intended domain result.
- `NOT IN (subquery)` can produce surprising results if the subquery contains a null. For an anti-join, prefer a correlated `NOT EXISTS` when that matches the question.

```sql
SELECT c.customer_id
FROM customer AS c
WHERE NOT EXISTS (
    SELECT 1
    FROM sales_order AS o
    WHERE o.customer_id = c.customer_id
);
```

Avoid applying a function or cast to an indexed column in a filter unless the intended expression index or plan is understood. Prefer range predicates for time intervals, and use typed, parameterized values rather than constructing SQL with string concatenation.

## Joins: match the relationship and its cardinality

| Join | Meaning |
| --- | --- |
| `INNER JOIN` | Keep only rows with a match on both sides. |
| `LEFT JOIN` | Keep every left row; unmatched right-side columns are null. |
| `RIGHT JOIN` | Keep every right row; often rewritten as a left join by reversing table order. |
| `FULL OUTER JOIN` | Keep matched rows and unmatched rows from both sides. |
| `CROSS JOIN` | Produce every pair; row count is the product of input row counts. |
| Self join | Join a table to another logical instance of itself, commonly for parent/child relations. |
| Semi/anti join | Test existence or non-existence, usually with `EXISTS` / `NOT EXISTS`, without returning matching right-side rows. |

Join keys express relationships; they do not guarantee one-to-one cardinality unless a constraint does. Joining two one-to-many relations at the same time can multiply rows and inflate sums. Check the key and expected rows per key before trusting an aggregate.

For a left join, a right-side condition in `WHERE` often removes the null-extended rows and changes the result toward an inner join. Put the condition in `ON` when unmatched left rows must remain:

```sql
SELECT c.customer_id, o.order_id
FROM customer AS c
LEFT JOIN sales_order AS o
  ON o.customer_id = c.customer_id
 AND o.status = 'confirmed';
```

To count confirmed orders for every customer, including customers with none:

```sql
SELECT
    c.customer_id,
    COUNT(DISTINCT o.order_id) AS confirmed_order_count,
    COALESCE(SUM(i.quantity * i.unit_price_at_purchase), 0) AS confirmed_sales
FROM customer AS c
LEFT JOIN sales_order AS o
  ON o.customer_id = c.customer_id
 AND o.status = 'confirmed'
LEFT JOIN sales_order_item AS i
  ON i.order_id = o.order_id
GROUP BY c.customer_id;
```

The `DISTINCT` protects the order count from line-item fan-out. The sum is at line-item grain in this query. If another one-to-many relation is added (for example, shipment events), aggregate each child relation to the intended grain before joining it, or the sum can be multiplied too.

## Aggregation versus window functions

`GROUP BY` collapses each group to one output row. A window function computes over a related set of rows while retaining each input row's identity. This distinction is the key to choosing between them.

```sql
-- One row per customer: GROUP BY collapses order rows.
SELECT customer_id, COUNT(*) AS order_count
FROM sales_order
GROUP BY customer_id;

-- One row per order: the partition count is repeated on each order.
SELECT
    order_id,
    customer_id,
    COUNT(*) OVER (PARTITION BY customer_id) AS customer_order_count
FROM sales_order;
```

Common aggregate functions include `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`. `GROUP BY` all non-aggregated selected expressions (subject to documented functional-dependency behavior in the target database). `HAVING` filters aggregate groups; it is not a substitute for an early row filter in `WHERE`.

Window patterns:

- `ROW_NUMBER()` selects a deterministic latest or highest-ranked row per group.
- `RANK()` and `DENSE_RANK()` preserve ties with different gap behavior.
- `LAG()` / `LEAD()` compare the current row with a neighboring row.
- A running total or moving average uses an ordered window frame.
- `PARTITION BY` restarts the calculation for each group; it does not sort the final result.

```sql
WITH order_totals AS (
    SELECT
        o.order_id,
        o.customer_id,
        o.created_at,
        COALESCE(SUM(i.quantity * i.unit_price_at_purchase), 0) AS total
    FROM sales_order AS o
    LEFT JOIN sales_order_item AS i USING (order_id)
    GROUP BY o.order_id, o.customer_id, o.created_at
),
ranked AS (
    SELECT
        order_totals.*,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at DESC, order_id DESC
        ) AS recency_rank
    FROM order_totals
)
SELECT customer_id, order_id, created_at, total
FROM ranked
WHERE recency_rank = 1;
```

Window calculations are logically evaluated after `WHERE` and grouping, so filtering on a window result usually requires an outer query or CTE. Specify a deterministic `ORDER BY` and an intentional frame for running aggregates; peer rows and default frames can otherwise change results in ways that are easy to miss.

## Subqueries, CTEs, and recursive queries

- A **scalar subquery** returns one value; it must not unexpectedly return multiple rows.
- A **derived table** is a subquery in `FROM`, useful for shaping an intermediate relation.
- A **correlated subquery** refers to an outer row. `EXISTS` is a clear expression of an existence test; the optimizer may transform it, but check plans for expensive workloads.
- A **CTE** (`WITH`) names an intermediate query and can make a multi-step transformation easier to read, test, and explain. It is not inherently faster than an equivalent subquery; materialization/inlining behavior is database- and version-dependent.
- A **recursive CTE** has an anchor term and a recursive term, combined with `UNION` or `UNION ALL`. It is useful for hierarchies and graph-like traversal. Define a termination condition and guard against cycles.

PostgreSQL-specific recursive example; `path` is used here to avoid following a cycle indefinitely:

```sql
WITH RECURSIVE reporting_tree AS (
    SELECT employee_id, manager_id, 0 AS depth, ARRAY[employee_id] AS path
    FROM employee
    WHERE employee_id = $1

    UNION ALL

    SELECT e.employee_id, e.manager_id, t.depth + 1, t.path || e.employee_id
    FROM employee AS e
    JOIN reporting_tree AS t ON e.manager_id = t.employee_id
    WHERE NOT e.employee_id = ANY (t.path)
)
SELECT employee_id, manager_id, depth
FROM reporting_tree
ORDER BY depth, employee_id;
```

The array syntax above is PostgreSQL-specific. Other engines provide different array, path, or cycle-handling syntax.

## Set operations and conditional expressions

- `UNION ALL` concatenates compatible result sets and preserves duplicates.
- `UNION` concatenates and removes duplicates, which requires additional work and can hide unexpected duplicate rows.
- `INTERSECT` returns rows shared by both inputs.
- `EXCEPT` returns rows in the first input but not the second.

Set-operation inputs need compatible column counts and types. Use parentheses when combining several operators so the intended grouping is clear.

`CASE` expresses conditional values inside a query; `COALESCE` chooses the first non-null value; `NULLIF(a, b)` returns null when the values are equal. Use explicit casts when a type is ambiguous. Date/time arithmetic and string functions often have dialect-specific details, so verify the target engine's manual for time zones, interval boundaries, regular expressions, and collation.

## Data changes and transactions

SQL also defines schema and data changes:

- DDL: `CREATE`, `ALTER`, and `DROP` tables, constraints, indexes, and views.
- DML: `INSERT`, `UPDATE`, `DELETE`, and, where supported, `MERGE` or an upsert form.
- `RETURNING` / equivalent: inspect rows changed by a write without issuing a separate query, where supported.

Use bind parameters for values. Parameters protect values from being interpreted as SQL and allow the database/driver to handle types correctly; they do not parameterize identifiers or SQL fragments. For dynamic identifiers, use the driver's safe identifier API or a strict allowlist.

A transaction groups statements into one all-or-nothing unit:

```sql
BEGIN;
-- Read and validate the state required by the business operation.
-- Apply related writes; check affected-row counts and constraints.
COMMIT;
-- On an error or failed invariant, ROLLBACK instead.
```

Transactions do not automatically make external API calls, messages, or files atomic with database writes. Keep transactions bounded, handle serialization/deadlock retry rules deliberately, and make retryable business operations idempotent. For isolation levels and concurrent correctness, continue with [Relational database design](relational-database-design.md#transactions-and-concurrent-correctness).

## A reliable SQL problem-solving method

For an interview question or production report:

1. Restate the output grain in one sentence. Ask what one output row represents.
2. Identify the source tables and relationship keys; estimate one-to-one versus one-to-many joins.
3. Decide which filters apply before aggregation and which apply to grouped/windowed results.
4. Choose join semantics, especially how unmatched rows and nulls should behave.
5. Choose `GROUP BY` if rows should collapse; choose a window if rows must remain visible.
6. Resolve ties, time boundaries, duplicates, empty sets, and `NULL` behavior explicitly.
7. Walk through a tiny example by hand. Check row counts and a boundary case.
8. Explain time/space implications only after the result is correct; inspect an execution plan for a performance claim.

Good interview communication is part of the solution: state assumptions, show the query in readable stages, mention a likely edge case, and validate the output grain. If stuck, test a smaller relation and write down the expected rows before changing syntax.

## Practice sequence

| Stage | Practice |
| --- | --- |
| 1. Retrieval | Projection, `WHERE`, null checks, deterministic ordering, `LIMIT`, and basic expressions. |
| 2. Relationships | Inner/outer/self joins, `EXISTS`, anti-joins, many-to-many joins, and join fan-out. |
| 3. Summaries | `GROUP BY`, aggregate filters, `HAVING`, conditional aggregation, and empty groups. |
| 4. Analysis | Ranking, top-N per group, running totals, `LAG`/`LEAD`, frames, deduplication, and gaps-and-islands. |
| 5. Composition | Scalar/correlated subqueries, CTEs, recursive CTEs, and set operations. |
| 6. Production habits | Transactions, constraints, safe parameterization, explain plans, and dialect differences. |

Use one practice schema long enough to learn its relationships rather than memorizing isolated answers. For every exercise, write down the expected grain and test at least one duplicate, missing match, tie, or null edge case.

## Sources and further practice

Checked on **2026-11-06**. These sources provide exercises and dialect references; this guide is an original synthesis, not a substitute for version-specific syntax documentation.

- [PostgreSQL tutorial](https://www.postgresql.org/docs/current/tutorial.html) — guided SQL and relational concepts; continue to the [SQL language reference](https://www.postgresql.org/docs/current/sql.html), [joins](https://www.postgresql.org/docs/current/tutorial-join.html), [aggregates](https://www.postgresql.org/docs/current/tutorial-agg.html), [window functions](https://www.postgresql.org/docs/current/tutorial-window.html), [CTEs and recursive queries](https://www.postgresql.org/docs/current/queries-with.html), [comparison and null-related operators](https://www.postgresql.org/docs/current/functions-comparison.html), and [transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html).
- [PostgreSQL Exercises](https://pgexercises.com/) — practice against one coherent dataset. Its [join exercises](https://pgexercises.com/questions/joins/) include inner/outer/self joins and subqueries; its [aggregation exercises](https://pgexercises.com/questions/aggregates/) include grouping and windows; its [string exercises](https://pgexercises.com/questions/string/) cover matching and manipulation.
- [SQLBolt](https://sqlbolt.com/) — interactive introductory lessons, starting with [SELECT](https://sqlbolt.com/lesson/select_queries_introduction).
- [SQLZoo tutorial](https://sqlzoo.net/wiki/SQL_Tutorial) — lesson/reference pages for selection, joins, nested queries, aggregates, date/time, functions, data changes, and transactions; see [SELECT from WORLD](https://sqlzoo.net/wiki/SELECT_from_WORLD_Tutorial) and [SELECT within SELECT](https://sqlzoo.net/wiki/SELECT_within_SELECT_Tutorial).
- [Mode SQL tutorial](https://mode.com/sql-tutorial/) — the supplied URL now redirects to ThoughtSpot's [SQL tutorial](https://www.thoughtspot.com/sql-tutorial); its [joins lesson](https://www.thoughtspot.com/sql-tutorial/sql-joins) explains relational matching and join structure. Treat it as a learning resource, not the authority for a particular engine's dialect.
- [SQL Practice Online scenarios](https://www.sql-practice.online/scenario) — topic collections for filtering, joins, subqueries, CTEs/windows, aggregation, and other practical patterns; its [joins collection](https://www.sql-practice.online/scenario/joins) explicitly exercises null preservation, `ON` versus `WHERE`, cardinality, and fan-out.
- [DataLemur SQL interview questions](https://datalemur.com/questions) — interview-style SQL practice; use it after learning the concepts, not instead of understanding them.

The DataCamp practice URL in the supplied roadmap returned HTTP 404 during research, so it is omitted rather than replaced with an unverified page.

## Related notes

- [Relational database design](relational-database-design.md) — schema, constraints, transactions, indexes, and the sample order model used here.
- [Query performance labs](query-performance-labs.md) — read plans and measure query behavior rather than guessing.
- [Database engine comparison](database-engine-comparison.md) — portability and vendor-specific differences.
- [HLD interview guide](../../14-interview-prep/system-design-interviews/hld-interview-guide.md) — connect query-level decisions to service and system requirements.
