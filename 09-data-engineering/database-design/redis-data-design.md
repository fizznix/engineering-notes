# Redis data design

[Database design hub](README.md) · [Database selection and operations](database-selection-and-operations.md) · [HLD interview guide](../../14-interview-prep/system-design-interviews/hld-interview-guide.md)

Redis is a data-structure server. Its useful design question is not simply “what should I cache?” but **which operations and data shape benefit from Redis semantics, and what durability, memory, eviction, and failure behavior does the application require?**

## Choose a data structure from operations

| Redis type | Natural operations/use | Example |
| --- | --- | --- |
| String | Read/write one value; atomic counters; bit operations | `cache:v2:product:123`, `session:<opaque-id>`, counter |
| Hash | Read/update named fields under one key | Small profile or object cache |
| Set | Membership, uniqueness, set operations | Feature membership, online-user set |
| Sorted set | Unique members ordered by score; rank/range | Leaderboard, scheduled timestamps, bounded time-window index |
| List | Ordered sequence and simple push/pop/blocking queue patterns | Small transient work queue when its semantics are sufficient |
| Stream | Append-only entries, range reads, consumer groups, pending/recovery tracking | Redis-native event/work stream |
| Bitmap/bitfield | Compact per-entity boolean/counter facts | Daily activity flags or bounded counters |
| Geospatial | Coordinates and radius/nearby lookup | Nearby-device candidate search |
| Probabilistic structures | Approximate membership/count/top-K | Large-scale approximate analytics |
| JSON / time series / vector sets | Structured JSON, timestamped series, or vector similarity on supported Redis distributions/versions | Specialized application stores |

Data types are not interchangeable. Use the official [comparison guide](https://redis.io/docs/latest/develop/data-types/compare-data-types/) to narrow choices, then evaluate cardinality, operation complexity, memory use, and compatibility with the deployed Redis version.

## Key design and data lifecycle

- Use predictable, namespaced keys such as `cache:v2:product:{productId}` or `rate:{tenantId}:{window}`. Include a version where data shape changes.
- Avoid personally identifying or secret values in key names; keys can appear in diagnostics and operational tooling.
- Set a memory budget and explicit lifetime/eviction policy for each class of key. TTL is a data-lifecycle choice, not a substitute for access-control or expiry validation.
- Use bounded collections. A key whose list/set/stream grows forever is a memory leak in data-model form; trim, expire, archive, or partition it.
- Consider key cardinality and size; many tiny keys and a few very large keys have different memory/latency effects.
- Decide what happens on miss, expiration, Redis restart, failover, memory pressure, and network partition.

## Common patterns

### Cache-aside

```text
read(key):
    value = redis.get(key)
    if value exists:
        return value

    value = authoritativeStore.load(key)
    if value exists:
        redis.set(key, value, ttl = baseTtl + smallRandomJitter)
    return value
```

Writes update the source of truth first, then invalidate or refresh the cache according to the freshness requirement. TTL jitter spreads synchronized expirations; it does not replace a cache stampede strategy for an exceptionally hot key.

Discuss:

- Cache key includes all data-affecting dimensions (tenant, locale, permission scope, version).
- TTL and acceptable staleness, including deletion/revocation.
- Negative caching of “not found,” if useful, with short lifetime.
- Stampede control (request coalescing, stale-while-revalidate, bounded refresh lock).
- Fallback behavior if Redis is unavailable; ensure database fallback does not overload the database during a cache outage.

### Counters and rate limits

Redis atomic increments can implement counters and fixed windows. Sliding windows can use timestamped members in a sorted set; token-bucket variants need state and atomic refill/consume logic. Define:

- The identity key (user, API key, tenant, IP, route) and privacy policy.
- Window/bucket semantics, burst allowance, and boundary behavior.
- Atomic update/check operation (script/transaction or supported server-side operation).
- Memory bounding/expiration and behavior if the store is unavailable.
- Whether a limit is approximate or must be globally exact across regions.

Do not implement “GET count, then SET count + 1” as two uncoordinated commands; concurrent requests can lose updates.

### Queue and event use

Lists can implement simple producer/consumer work queues, but a pop can lose work if the consumer crashes after removal. Streams provide entry IDs, range reads, consumer groups, acknowledgments/pending entries, and recovery flows; retention and redelivery policy still need design. Redis Streams are not automatically equivalent to Kafka or a dedicated durable broker: compare ordering, replay, retention, throughput, durability, and ecosystem needs.

For each asynchronous pattern, state what an acknowledgment means, how duplicates are handled, how abandoned work is reclaimed, and how the stream/queue is trimmed without deleting unprocessed work.

### Sessions and ephemeral state

Opaque random session IDs can map to a hash/string with TTL. Store only what the serving path needs; revoke/rotate on logout or credential changes; treat expiry as an application security rule as well as key cleanup. If Redis loss would log users out, that may be acceptable; if it would lose business-critical workflow state, define persistence and recovery or use a more suitable source of truth.

## Atomicity and Redis transactions

Redis `MULTI`/`EXEC` serializes a queued command group so another client does not run commands in the middle; `WATCH` supports optimistic check-and-set flows. This is not the same programming model as a relational transaction:

- Runtime command errors do not provide general rollback of prior successful commands.
- A transaction cannot wrap an arbitrary external database/API call.
- Scripts/functions can make a server-side sequence atomic, but long-running scripts block other work and must be bounded.
- Atomicity of execution does not imply persistence through every crash/failover configuration.

Use a single atomic command where possible (`INCR`, set-if-absent, conditional update). Use `WATCH` or a bounded script for a small read-check-write operation. Make retries safe and understand the durability/recovery configuration separately.

## Memory limits and eviction

Redis is memory-oriented; estimate data plus key/object/index overhead, replication buffers, persistence copy-on-write headroom, and operational safety margin. Test using representative cardinality and value sizes rather than payload bytes alone.

At `maxmemory`, the configured eviction policy determines which keys may be removed (or whether writes are rejected). Decide based on data role:

- **Rebuildable cache:** an all-keys or TTL-oriented eviction policy may fit; the origin must sustain misses.
- **State that must not be silently evicted:** use a policy/configuration that rejects writes rather than evicting arbitrary live state, and alert before memory exhaustion.
- **Mixed key classes:** separate them into instances/deployments when eviction/durability requirements differ; do not assume one policy safely serves unrelated workloads.

Monitor memory, evictions, keyspace hit/miss, command latency, hot keys, connection count, replication lag, and persistence health. Large keys/commands can cause latency spikes even if average request volume looks safe.

## Persistence, replication, and cluster behavior

Redis persistence options include:

- **RDB snapshots:** point-in-time snapshots; lower write-log volume but recovery may omit writes since the last snapshot.
- **AOF:** append-only write history that is replayed; durability depends on fsync policy and implementation/configuration.
- **Both or neither:** select based on recovery point, restart time, write overhead, and whether data is reconstructable.

Replication and cluster provide availability/scaling behavior, but they do not mean every acknowledged write is synchronously present on all nodes. Define acceptable data loss, failover behavior, read routing, and restore procedure. Redis Cluster shards data by hash slots; it adds topology and multi-key placement considerations. A cluster does not make an unbounded or hot-key data model safe.

Practice restore/failover, not just configuration. If Redis is the only copy of business-critical data, validate its recovery point/time and failure semantics as rigorously as any primary database.

## Distributed locks: use caution

Distributed locks are easy to misuse. A lease can expire while a client is paused; that old client may resume and still act on the protected resource after a new client acquired the lock. A lock token alone does not stop stale work.

- Prefer database constraints/transactions or an ownership model when they can enforce correctness directly.
- If a lease lock is necessary, make acquisition unique and bounded, release only the matching token, and design recovery for crashes.
- For a correctness-critical external resource, use fencing tokens or another mechanism that lets the resource reject stale lock holders.
- Understand topology/failover guarantees; do not assume a Redis lock is a universal consensus primitive.

The Redis lock documentation presents the Redlock algorithm and also links analysis/disclaimer material; apply it to a specific failure model rather than treating one lock recipe as a proof of safety.

## Redis design review checklist

- Which exact operations require Redis, and can another store already meet them?
- Is Redis a cache, source of truth, session store, coordination system, or event store—and is that role explicit?
- Is the structure selected for required operations and bounded growth?
- Are key namespace, versioning, tenant isolation, TTL, and privacy handled?
- What happens on miss, eviction, restart, failover, partition, and persistence recovery?
- Are check-and-update operations atomic and retry-safe?
- Is memory sized with overhead/headroom and monitored before the limit?
- Can a cache outage overload the authoritative database?
- Are lock/queue semantics strong enough for the business invariant?

## Sources

Inspected on **2026-11-06**:

- [Redis data types](https://redis.io/docs/latest/develop/data-types/) and [comparison guide](https://redis.io/docs/latest/develop/data-types/compare-data-types/); the page links tutorials and command references for strings, hashes, lists, sets, sorted sets, streams, geospatial, probabilistic, time-series, and other types.
- [Key eviction](https://redis.io/docs/latest/develop/reference/eviction/) explains `maxmemory` and eviction policies, including approximated LRU/LFU.
- [Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) describes RDB snapshots, AOF, combined configurations, backups, and disaster recovery.
- [Transactions](https://redis.io/docs/latest/develop/using-commands/transactions/) explains `MULTI`, `EXEC`, `DISCARD`, `WATCH`, command errors, rollback expectations, and scripting.
- [Streams](https://redis.io/docs/latest/develop/data-types/streams/) covers consumer groups, pending entries, recovery, trimming, persistence/replication, and differences from Kafka partitions.
- [Redis Cluster scaling](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/) and the [cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) describe sharding and availability behavior.
- [Distributed locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) discusses implementations, safety/liveness, failover assumptions, and the Redlock analysis/disclaimer.
- [EXPIRE command](https://redis.io/docs/latest/commands/expire/) documents key expiration and its options.

