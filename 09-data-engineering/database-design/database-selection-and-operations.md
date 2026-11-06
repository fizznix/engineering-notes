# Database selection and operations

[Database design hub](README.md) · [Relational database design](relational-database-design.md) · [Document database design](document-database-design.md) · [Redis data design](redis-data-design.md)

Choosing a database is choosing a data model, query model, consistency boundary, failure mode, and operating burden. The best fit is the one that satisfies the real workload with the fewest unsupported assumptions—not the newest database or the one with the most attractive benchmark.

## A database selection interview

Answer these questions in order:

1. **What are the entities and facts?** Which component owns each fact? Which fields are immutable snapshots, derived projections, or transient state?
2. **What must always be true?** Uniqueness, relationships, inventory, balances, lifecycle, access-control and audit rules.
3. **What are the reads and writes?** Exact filters, sorts, joins/aggregations, result sizes, updates, rate, burst, growth, and hot-key shape.
4. **What are the quality requirements?** Latency percentile, availability, freshness, durability, recovery time/data loss, region, cost, privacy, retention.
5. **Which data model makes the common operation natural?** Relational rows/joins, document aggregate, key lookup/data structures, search index, time series, graph, or blob.
6. **Where is the atomic boundary?** One row/document, multi-row transaction, idempotent event workflow, or eventual projection?
7. **What is the smallest credible deployment?** Start with one primary technology where possible; justify added stores/replicas/partitions by a requirement or measured bottleneck.

Do not translate one prompt keyword directly into a product: “large” does not automatically mean NoSQL, “JSON” does not automatically mean MongoDB, and “fast” does not automatically mean Redis.

## Compare by requirements, not slogans

| Requirement | Questions to ask |
| --- | --- |
| Transactions and invariants | Can the store enforce uniqueness/references? What is the transaction scope and isolation? Are retries safe? |
| Query shape | Point reads, range scans, joins, ad hoc filters, full text, graph traversal, or aggregations? |
| Data shape | Stable relational entities, bounded aggregates, sparse evolving documents, time series, large blobs? |
| Scale | Which resource saturates: CPU, memory, I/O, connections, write contention, network, or one hot partition? |
| Consistency | Must writes be immediately visible to reads? Can reads be stale? Can work be asynchronous? |
| Availability/recovery | What happens when a node/region/network fails? What are RPO/RTO and restore steps? |
| Operations | Who provisions, upgrades, backs up, tunes, monitors, secures, and responds to incidents? |
| Cost | What are compute, storage, index, replication, egress, licensing, and engineering/operational costs? |

## Scale in stages

1. **Make the query correct.** Reduce unnecessary round trips/columns/rows; batch where useful; remove N+1 calls; paginate.
2. **Measure with realistic data.** Inspect query plans, latency percentiles, CPU/I/O, lock waits, connection saturation, memory, and growth.
3. **Improve schema and indexes.** Add constraints, rewrite expensive predicates, and create indexes matched to actual query shapes.
4. **Right-size and isolate.** Tune resource sizing, pools, workload classes, and maintenance; separate analytics or background work if it competes with OLTP.
5. **Add replicas/caches deliberately.** State replication lag, cache staleness, invalidation, and behavior during primary/cache failure.
6. **Partition or shard when evidence justifies it.** Plan key distribution, hot partitions, rebalancing, query routing, backup, migrations, and cross-partition operations.

This is not a required universal order. For example, a global-read requirement may justify regional replicas early. The key is to name the constraint that makes added complexity worthwhile.

## Replication, partitioning, sharding, and cache are different tools

- **Replication:** copies a dataset. It can improve read capacity or failover; asynchronous replication can be stale and may lose acknowledged-but-unreplicated writes under some failures/configurations.
- **Partitioning:** splits a logical dataset into pieces, often by time/range/list/hash for pruning, lifecycle, or management. It may be within one server/cluster.
- **Sharding:** places partitions on independent nodes to distribute storage/write capacity. It adds routing, rebalancing, hotspot, cross-shard query/transaction, and operational complexity.
- **Caching:** keeps a copy to reduce repeated origin work/latency. It introduces freshness/invalidation and fallback concerns; it does not replace a durable source of truth unless designed and operated as one.
- **Backup:** preserves recoverable state. A replica usually mirrors deletion/corruption; test a separate backup restore path and point-in-time recovery where required.

### Shard/partition key checklist

- Does it appear in the dominant queries so requests can target a limited partition set?
- Does it distribute writes and storage evenly at current and expected future scale?
- Are values high-cardinality, or are a few tenants/users responsible for most traffic?
- Is the key monotonic, creating a write hotspot?
- Will common transactions require multiple shards?
- Can a tenant or hot key exceed one partition's capacity?
- Can data be moved/rebalanced without unacceptable downtime?
- How does a key change, deletion, retention, or resharding work?

No key maximizes all properties. Choose from measured access distribution and state the trade-off.

## Read consistency and user-visible behavior

“Strong” and “eventual” are incomplete descriptions. Specify:

- Which write is acknowledged (primary only, quorum/majority, durable log, etc.)?
- Which reads may hit a replica or cache, and how stale could they be?
- Does a user require read-your-writes after an update?
- Can two concurrent updates conflict, and how does the caller resolve it?
- Is an eventual projection acceptable, and how is its lag exposed?
- During network partition/failover, do requests wait, fail, or return possibly stale data?

Use the least costly guarantee that satisfies the product invariant. A profile view may tolerate short staleness; withdrawing funds or allocating the last seat usually requires stronger coordination. The domain rule determines the design, not an abstract “consistency preference.”

## Backups, recovery, and data lifecycle

- Define **RPO**: the maximum acceptable data loss after a failure.
- Define **RTO**: the target time to restore service.
- Select backup type/frequency and retention for those targets; include encryption, access control, and off-site/account separation where appropriate.
- Test restore and point-in-time recovery on a schedule. Verify data correctness and application compatibility after restore.
- Rehearse failover, promote/fail back behavior, DNS/connection changes, and replica lag impact.
- Define archival, deletion, legal hold, privacy erasure, and retention for backups/caches/search projections—not just the primary table.
- Treat schema/data migrations as production operations: estimate lock/write impact, backfill in batches, monitor progress, and make the process resumable.

## Safe schema change: expand, migrate, contract

For a change that must coexist with old and new application versions:

1. **Expand:** add a nullable/default-compatible field or new table/index without breaking old code.
2. **Deploy compatible code:** readers accept old/new shapes; writers populate the new representation where appropriate.
3. **Backfill:** process bounded batches with checkpoints, idempotency, rate control, and validation.
4. **Switch:** read from the new representation; compare/monitor results before removing the old path.
5. **Contract:** after all versions/jobs are compatible, remove obsolete fields/tables and old code.

For a tiny table or a maintenance window, a simpler migration may be safer. Measure locks and rewrite behavior for the specific DB/version; do not assume every `ALTER TABLE` is instant.

## Security and operations are part of schema design

- Use least-privilege identities per application/job; separate migration/admin credentials from runtime access.
- Use parameterized statements and safe driver APIs; never build SQL from untrusted input.
- Classify PII/secrets, encrypt data in transit and at rest as required, rotate credentials, and restrict operational access.
- Enforce tenant boundaries in queries/constraints/policies; test cross-tenant access explicitly.
- Define who may read/change data, what is logged/audited, and how deleted data disappears from replicas, caches, exports, and backups.
- Do not log full connection strings, credentials, or sensitive query parameters.

Useful operational indicators:

- Query latency percentiles and slow-query distribution.
- Throughput, errors, timeouts, deadlocks, lock waits, and transaction age.
- CPU, I/O, disk space, memory, cache hit/miss, connections, and pool wait.
- Replica lag, replication slot/WAL growth (PostgreSQL), shard skew, and failover state.
- Index/table growth, vacuum/autovacuum health, backup age, restore tests, and migration progress.
- Cache eviction and origin fallback load; queue depth/oldest-message age for async work.

Alert on user impact and actionable capacity/recovery risk, not every internal metric deviation.

## Common database-design anti-patterns

- Selecting a database from team fashion or an unverified benchmark.
- Premature sharding and multi-region writes before defining consistency and routing.
- Using one database for every access pattern despite very different latency/retention/query needs.
- Denormalizing with no source of truth or repair/reconciliation method.
- Adding indexes as a substitute for reading query plans.
- Treating a replica, cache, or Redis persistence option as a tested backup.
- Assuming database transactions include a message broker, remote API, or payment provider.
- Using read replicas without informing the application about stale reads.
- Overusing JSON/EAV to avoid modeling stable fields, constraints, and relationships.
- Treating a schema migration as a one-line deploy rather than a rollout with mixed application versions.

## Database design review

Use this checklist for a design doc or interview:

- Are requirements, data ownership, invariants, and access patterns stated?
- Is the proposed model understandable and aligned with common operations?
- Are keys, uniqueness, nullability, relationships, deletion, and historical snapshots explicit?
- Are constraints enforced at the authoritative store?
- Are transaction and concurrency behaviors correct under simultaneous requests and retries?
- Are indexes linked to queries and verified with realistic plans/measurements?
- Are staleness, cache invalidation, replication, and failure behavior described?
- Are scale choices justified by a bottleneck rather than assumed?
- Are backup/restore, migration, retention, security, and observability included?
- Is there a simpler design that meets the same requirements?

## Sources

Inspected on **2026-11-06**:

- [Designing Data-Intensive Applications](https://dataintensive.net/) offers a broad conceptual comparison of reliable, scalable, maintainable data systems; its website does not expose the book as a complete free online reference.
- [System Design Primer database section](https://github.com/donnemartin/system-design-primer#database) and its sections on [replication](https://github.com/donnemartin/system-design-primer#replication), [sharding](https://github.com/donnemartin/system-design-primer#sharding), [denormalization](https://github.com/donnemartin/system-design-primer#denormalization), and [SQL vs. NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql) are interview-oriented overviews.
- See the focused [PostgreSQL](relational-database-design.md), [MongoDB](document-database-design.md), and [Redis](redis-data-design.md) notes for official manual links and vendor-specific caveats.

