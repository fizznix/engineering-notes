# Document database design (MongoDB)

[Database design hub](README.md) · [Relational database design](relational-database-design.md) · [Database selection and operations](database-selection-and-operations.md)

MongoDB stores BSON documents in collections. Its flexible schema is useful, but it does not remove the need for a schema: application code still depends on fields, relationships, state transitions, and query paths. The most important design question is **which data is commonly read, changed, and owned together?**

## Work from operations and aggregate boundaries

Before writing document examples, list the operations and the expected document set:

- Which queries are required? Which fields filter/sort/project? How many documents/array elements can they return?
- Which data is read together on the hot path?
- Which data changes together and must be atomic?
- Which related values change independently or are queried independently?
- Is any one-to-many relationship high-cardinality or unbounded?
- What data duplication is acceptable, and how is it refreshed?
- What are the write/read rates, largest document shape, retention, and expected growth?

MongoDB's modeling guidance emphasizes keeping data accessed together together, then choosing embedding or referencing based on cardinality, lifecycle, and update/query patterns. Do not turn that into “always embed.”

## Embed or reference?

| Prefer embedding when | Prefer references when |
| --- | --- |
| The child is small, bounded, and normally viewed with its parent. | The child collection can grow without a practical bound. |
| Parent and child are created/updated together. | Child data changes independently or has a different owner/lifecycle. |
| A single-document atomic update is valuable. | The child is frequently queried on its own or shared among parents. |
| A single read should return the aggregate without an application-side join. | Duplication would be expensive or correctness-sensitive. |

These are heuristics. One-to-many can be embedded when the “many” side is bounded and owned; it should be referenced when it is unbounded, independently queried, or large. Many-to-many and mutable shared data often need references. Embedding duplicates data by design, so decide which copy is authoritative.

### Avoid unbounded arrays

An ever-growing array causes document growth, larger rewrites/reads, and eventually a document-size limit. MongoDB's manual states documents must remain below **16 MiB**. Model unbounded events, comments, transactions, or messages as separate documents/collections with indexed parent IDs and pagination; keep a bounded recent subset or summary in the parent only if it serves a measured access path.

### Embed snapshots, reference current shared facts

For an order, embedding a bounded array of purchased line items and a snapshot of the address/price can preserve what the customer agreed to at checkout. The current product inventory or description has a separate lifecycle and is usually stored in a product collection. Avoid fetching current price to reconstruct historical order value.

Example:

```javascript
{
  _id: ObjectId("64c13ab08edf48a008793cac"),
  customerId: ObjectId("64c13ab08edf48a008793cad"),
  status: "confirmed",
  createdAt: ISODate("2026-11-06T12:00:00Z"),
  shippingAddressAtPurchase: {
    city: "Example City",
    postalCode: "A1B 2C3"
  },
  items: [
    {
      productId: ObjectId("64c13ab08edf48a008793cae"),
      skuAtPurchase: "SKU-123",
      titleAtPurchase: "Example product",
      quantity: 2,
      unitPriceAtPurchase: Decimal128("19.95")
    }
  ]
}
```

This assumes an order's line count is bounded and items are read with the order. Payment attempts, status history, or very large fulfillment events may be separate collections because they have independent lifecycles and growth.

## Flexible schema still needs discipline

- Define document shapes and required fields before production.
- Use MongoDB schema validation for important type/range/shape rules; flexibility should be an intentional migration capability, not a way for clients to write arbitrary types.
- Version documents when a shape migration requires it. Make readers tolerate old/new versions during rollout; backfill incrementally and make it restartable.
- Avoid a generic untyped key/value or deeply nested structure when fields need independent constraints, indexing, permissions, or query patterns.
- Use explicit field naming and BSON types; do not mix dates as strings and date values or quantities as numbers and strings.
- Keep sensitive data and tenant ownership explicit. Query filters and authorization must prevent cross-tenant reads/writes.

## Atomicity and transactions

MongoDB guarantees atomicity for a write to one document, including a change to multiple fields in that document. This makes the document boundary an important correctness boundary. For concurrent conditional updates, include the expected current state/version in the filter so an operation fails to match if another writer changed it; avoid unconditional overwrites for state transitions.

Multi-document ACID transactions are available, including across shards, but they add coordination and cost. Use them when a real invariant spans documents and cannot be modeled safely as one aggregate or an asynchronous workflow. First ask:

- Can the invariant be placed in one bounded document?
- Can a unique constraint, conditional update, or idempotent state machine enforce it?
- Can eventual completion with an outbox/saga/reconciliation process meet the product requirement?
- If a transaction is needed, what is its duration, retry behavior, read/write concern, and contention profile?

Do not assume “NoSQL means no transactions”; also do not use multi-document transactions to avoid making an intentional data model.

## Indexes and query plans

MongoDB indexes reduce the documents scanned for matching queries, but each index adds memory/disk use and write maintenance. Create indexes from real query patterns:

- A **single-field** index supports frequent lookups/sorts on one field.
- A **compound** index supports combined predicates/sorts; key order affects which prefixes and sort orders it can serve.
- **Multikey** indexes index array values and have specific behavior/limits.
- **Unique** indexes enforce uniqueness, including concurrent writes.
- **Text/geospatial/TTL/partial** indexes serve specialized operations; validate the feature's exact semantics and version.

Use `explain()` to compare the winning plan, keys/documents examined, returned documents, sort behavior, and rejected plans where available. Do not add a compound index just because fields appear together somewhere; match the index prefix and sort to the query. Avoid large indexes that no production query uses.

A TTL index is a cleanup mechanism for data with an expiration policy; do not treat background expiration as an exact-time authorization or billing boundary. Enforce access/validity in the request path even when cleanup is configured.

## Replication, reads, and sharding

- Replica-set secondaries replicate asynchronously. Reading from a secondary can return stale data. Choose read preference/concern to match the required freshness and durability; “read from replica” is a correctness change as well as a load choice.
- A write concern controls what acknowledgment means; a read concern controls which data a read may observe. State the desired behavior rather than saying only “strong consistency.”
- A shard key should distribute writes and data while allowing important queries to target a small number of shards. Consider cardinality, frequency/skew, monotonic growth, locality, and how the key appears in queries.
- Low-cardinality keys limit distribution; a monotonically increasing key can direct new writes to a hot range; a high-cardinality key may still be skewed if a few values dominate.
- Sharding changes query routing, unique-index constraints, balancing, and transaction behavior. Do not shard until single-cluster capacity or availability requirements justify the operational cost.

## Common modeling failures

- Embedding unbounded arrays because “document databases should denormalize.”
- Duplicating mutable shared data without an owner, refresh mechanism, or repair path.
- Splitting every nested object into its own collection and rebuilding a relational join model with many round trips.
- Assuming flexible schema means no validation or migration plan.
- Adding indexes without measuring query/write effects.
- Assuming a single-document atomic write protects a cross-document invariant.
- Assuming a replica read is fresh or that a successful write is durable under every configuration.
- Choosing a shard key from an entity name rather than query routing and data distribution.

## MongoDB design review checklist

- Is the aggregate boundary bounded and aligned with common reads/writes?
- Are frequently co-read fields embedded only when growth and update rules permit it?
- Are independently queried/high-cardinality children referenced and paginated?
- Is every duplicated value a deliberate snapshot or derived copy with a source of truth?
- Are document validation, type consistency, and compatibility/migration rules clear?
- Do conditional writes preserve concurrent invariants?
- Do indexes match filters, sorts, uniqueness, and projections without excessive write cost?
- Are read/write concerns aligned with freshness/durability needs?
- Does the shard key distribute actual traffic and target actual queries?
- Are transactions limited to boundaries that require cross-document atomicity?

## Sources

Inspected on **2026-11-06**:

- [MongoDB data modeling](https://www.mongodb.com/docs/manual/data-modeling/), [schema design process](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/), and [best practices](https://www.mongodb.com/docs/manual/data-modeling/best-practices/).
- [Embedding](https://www.mongodb.com/docs/manual/data-modeling/embedding/) and [referencing](https://www.mongodb.com/docs/manual/data-modeling/referencing/) describe the access-pattern, cardinality, duplication, and lifecycle trade-offs.
- [Schema validation](https://www.mongodb.com/docs/manual/core/schema-validation/), [indexes](https://www.mongodb.com/docs/manual/indexes/), [TTL indexes](https://www.mongodb.com/docs/manual/core/index-ttl/), and [`explain()`](https://www.mongodb.com/docs/manual/reference/method/cursor.explain/) cover shape enforcement and query performance.
- [Single-document write atomicity](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/), [transactions](https://www.mongodb.com/docs/manual/core/transactions/), and [read isolation/consistency](https://www.mongodb.com/docs/manual/core/read-isolation-consistency-recency/) clarify operation boundaries and read/write guarantees.
- [Shard-key selection](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/) covers cardinality, frequency, monotonic keys, and distribution.
- [System Design Primer: NoSQL](https://github.com/donnemartin/system-design-primer#nosql) and [SQL or NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql) offer interview-oriented trade-off summaries; check current vendor documentation for product behavior.
