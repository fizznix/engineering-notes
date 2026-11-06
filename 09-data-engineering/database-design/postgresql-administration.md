# PostgreSQL administration and operations

[Database design hub](README.md) · [Database internals](database-internals.md) · [Database engine comparison](database-engine-comparison.md) · [Query performance labs](query-performance-labs.md)

SQL fluency is not the same as operating a database. Production database engineering includes secure access, configuration, capacity, backup and restore, recovery, replication, maintenance, monitoring, and tested failure procedures. Use an isolated local instance for the labs below; never experiment with destructive recovery or failover commands on a production cluster.

The PostgreSQL `current` documentation URL changes as releases advance. Record the server version with every operational procedure and use documentation for that version when a setting or command is version-sensitive.

## First identify the objects

- A **server instance/cluster** is a running PostgreSQL installation backed by a data directory and configuration.
- A **database** is a namespace within a cluster; it has schemas and database-level settings/privileges.
- A **schema** groups objects such as tables, functions, and views.
- A **role** represents an identity and/or privilege group. PostgreSQL uses roles for both users and groups; object ownership and granted privileges are distinct concepts.
- A **connection** is a server backend/session. Connection count is a capacity dimension, not just an application configuration value.

The official [installation tutorial](https://www.postgresql.org/docs/current/tutorial-install.html) and [administration documentation](https://www.postgresql.org/docs/current/admin.html) are the entry points for setting up and managing a cluster.

## Roles, ownership, and least privilege

Separate object ownership from the application identity. A migration/deployment role can own schema objects; the normal runtime role should receive only the privileges required by the application. Avoid running applications as a superuser.

```sql
CREATE ROLE app_owner NOLOGIN;
CREATE ROLE app_runtime LOGIN;

GRANT CONNECT ON DATABASE appdb TO app_runtime;
GRANT USAGE ON SCHEMA app TO app_runtime;
GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA app
    TO app_runtime;
```

This is a teaching sketch, not a complete production role policy. Table privileges do not automatically define privileges for every future object. Configure default privileges for the role that creates those objects; account separately for sequences, functions, schema creation, migrations, and any read-only/reporting role. Test access using the actual runtime identity. See [PostgreSQL role management](https://www.postgresql.org/docs/current/user-manag.html) and [object privileges](https://www.postgresql.org/docs/current/ddl-priv.html).

Operational habits:

- Prefer separate login and owner roles; grant membership only where needed.
- Grant access to required schemas/objects, not `ALL` across the cluster by default.
- Review ownership and default privileges after migrations.
- Use a managed secrets mechanism and rotate credentials; do not put passwords in source control or command history.
- Revoke access deliberately during offboarding and test both allowed and denied operations.

## Configuration and capacity

Configuration changes have different contexts: some require a reload, while others require a server restart. Consult each parameter's documentation and validate the effective value after change.

| Setting / concern | What it means | Why a universal value is unsafe |
| --- | --- | --- |
| `shared_buffers` | PostgreSQL server shared page buffers. | PostgreSQL also relies on the OS page cache; total memory includes much more than this area. |
| `work_mem` | Memory limit per sort/hash operation in a query plan, not a single global pool. | One query can use multiple such operations and many sessions can run concurrently. |
| `effective_cache_size` | Planner estimate of cache likely available to a query. | It is an estimate, not memory allocated by PostgreSQL. |
| `max_connections` | Upper bound on server sessions. | More backend processes consume resources; connection pooling may be better than raising the cap. |
| `maintenance_work_mem` | Memory available to some maintenance operations. | Concurrent maintenance and the machine's total memory budget matter. |
| WAL/checkpoint settings | Influence log retention and checkpoint/write behavior. | Write rate, recovery objectives, storage, and replication determine safe settings. |

Start with workload and measurements: active/idle sessions, query memory, cache behavior, storage latency, WAL generation, and connection-pool wait time. Change one parameter at a time, record the old value and expected effect, and have a rollback plan. Use the [resource configuration reference](https://www.postgresql.org/docs/current/runtime-config-resource.html) and [WAL configuration reference](https://www.postgresql.org/docs/current/runtime-config-wal.html); do not copy a tuning recipe without accounting for total concurrent memory use.

## Backup, restore, and point-in-time recovery

Choose a recovery design from explicit **RPO** (acceptable data loss) and **RTO** (time to recover), database size, retention, and operator capability.

| Mechanism | Typical purpose | Important verification |
| --- | --- | --- |
| `pg_dump` / `pg_dumpall` | Logical export of database objects/data; useful for portability and selected-object recovery. | Restore with `pg_restore`/`psql`, check extensions/roles/ownership, and measure restore duration. |
| `pg_basebackup` or a supported physical backup tool | Base backup of a cluster for physical recovery/standby seeding. | Ensure the backup is consistent and compatible with the intended recovery process/version. |
| WAL archiving plus a base backup | Continuous archiving and point-in-time recovery to a target time/transaction. | Validate that archived segments are complete and retained for the full recovery window. |
| Replica | Availability/read scale, depending on topology and consistency. | It is not a historical backup; destructive logical changes can replicate. |

PostgreSQL's [backup and restore chapter](https://www.postgresql.org/docs/current/backup.html) describes logical and physical approaches. [Continuous archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html) combines a base backup with archived WAL so recovery can stop at an earlier point.

A backup is not proven until a restore succeeds. Keep backup data outside the failure domain of the primary, restrict access to it, monitor archive failures and storage growth, and periodically rehearse a restore into an isolated environment. Document who can initiate recovery and how applications will reconnect.

## Replication and failover

- **Physical streaming replication** replays WAL changes on a standby. It is well-suited to a PostgreSQL-compatible physical copy and can be asynchronous or synchronous.
- **Logical replication** streams logical row/table changes and supports different selection and integration use cases; it has separate limitations and DDL/data lifecycle considerations.
- **Replication lag** means the standby has not yet applied all changes. A promotion can lose acknowledged transactions under asynchronous replication if they were not received/applied.
- **Failover** is a role transition, not just “turn on the replica.” Detection, fencing the old primary, promotion, routing, client reconnect, and avoiding split brain are separate responsibilities.
- **Replication slots** can retain WAL for a consumer that is behind or disconnected; monitor their effects on disk usage.

Define desired RPO/RTO and the behavior of reads during failover. Test planned switchover and unplanned failover using a disposable topology. PostgreSQL's [warm standby/streaming replication documentation](https://www.postgresql.org/docs/current/warm-standby.html) documents the replication mechanisms; an external orchestration/consensus layer may be required for automated failover. Do not assume the database server by itself provides safe automatic leader election.

## Monitoring and maintenance

Ask questions in layers:

1. **Is the host healthy?** CPU, memory pressure, filesystem capacity, disk latency, network, and process state.
2. **Is PostgreSQL healthy?** connections, transaction/commit rates, checkpoints, WAL generation, replication lag, lock waits, deadlocks, and vacuum progress.
3. **Which query or workload is responsible?** query count/total time, rows, planning/execution behavior, and application pool wait.
4. **Is the data maintained?** statistics freshness, autovacuum progress, dead tuples, index/table growth, and transaction age.

Useful PostgreSQL views/tools:

- `pg_stat_activity`: current sessions, state, query, and wait information. Treat query text as potentially sensitive.
- `pg_locks`: lock state and wait relationships.
- `pg_stat_database`: cumulative database-level activity.
- `pg_stat_statements`: aggregated query statistics through an extension that must be installed/configured.
- `pg_stat_progress_vacuum` and related progress views: track supported long-running maintenance work.
- `VACUUM` and `ANALYZE`: routine cleanup/statistics maintenance; autovacuum usually handles ordinary operation.
- `REINDEX`: a targeted maintenance/recovery tool, not a substitute for investigating why an index is unhealthy or bloated.

Use PostgreSQL's [monitoring chapter](https://www.postgresql.org/docs/current/monitoring.html), [routine vacuuming guide](https://www.postgresql.org/docs/current/routine-vacuuming.html), and [EXPLAIN documentation](https://www.postgresql.org/docs/current/using-explain.html). Correlate database data with host-level tools; one layer alone can mislead.

## Safe local administration labs

Use a disposable local cluster/container and keep its data directory separate from valuable data. Record commands, version, settings, expected result, and cleanup steps. Docker Compose with isolated volumes is optional; the same conceptual exercises can be done with a local installation.

1. **Install and initialize:** connect with `psql`; record server version, cluster data directory, and active configuration files.
2. **Create roles:** create owner, runtime, and read-only roles; test `GRANT` and `REVOKE` as each identity, including future-object privileges.
3. **Explore capacity:** inspect active and idle sessions; use a small bounded connection pool; observe what happens when the connection limit is approached.
4. **Take a logical backup:** run `pg_dump`, restore into a fresh database, and compare schema, row counts, constraints, and ownership.
5. **Take a physical backup:** create a base backup in a separate location; follow the matching version's recovery procedure in an isolated cluster.
6. **Practice PITR conceptually or hands-on:** configure WAL archiving with protected storage, take a base backup, create a known change, and recover a copy to a time before it. Do not use a primary production data directory.
7. **Observe replication:** configure a primary/standby pair, inspect replication state and lag, stop/restart the standby, and understand retained WAL behavior.
8. **Simulate failover:** in the disposable topology, fence/stop the old primary, promote the standby as documented, validate data and routing, and define how the old node rejoins safely.
9. **Inspect activity and locks:** run a long transaction and a conflicting write in the lab; inspect `pg_stat_activity` and `pg_locks`, then end the test.
10. **Observe vacuum:** create/update/delete test rows, inspect statistics, allow autovacuum or run a targeted `VACUUM (ANALYZE)`, and compare state.
11. **Tune with evidence:** benchmark a fixed workload before and after a single change to `work_mem` or another setting; record memory, latency distribution, and plan changes.

`EXPLAIN ANALYZE` executes the statement. For a test `UPDATE`/`DELETE`, use a disposable table or a transaction that you deliberately roll back, and understand that external effects are not rolled back.

## Operational readiness checklist

- Roles, ownership, secrets, and network access follow least privilege.
- Connection limits and pool sizes are bounded and coordinated across all application instances.
- Backups are protected, monitored, retained, and restored regularly in a test.
- RPO/RTO, PITR, replication lag, and failover behavior are documented and exercised.
- Disk growth, WAL/archive retention, vacuum health, query latency, lock waits, and host saturation are monitored with actionable alerts.
- Configuration changes are versioned, measured, reviewed, and reversible.
- Destructive operations are rehearsed on disposable data and have explicit confirmation steps.

## Sources

Checked on **2026-11-06**:

- [PostgreSQL tutorial and installation](https://www.postgresql.org/docs/current/tutorial.html), [server administration](https://www.postgresql.org/docs/current/admin.html), [role management](https://www.postgresql.org/docs/current/user-manag.html), [privileges](https://www.postgresql.org/docs/current/ddl-priv.html), and [runtime configuration](https://www.postgresql.org/docs/current/runtime-config.html).
- [PostgreSQL backup and restore](https://www.postgresql.org/docs/current/backup.html), [continuous archiving/PITR](https://www.postgresql.org/docs/current/continuous-archiving.html), [warm standby and streaming replication](https://www.postgresql.org/docs/current/warm-standby.html), and [WAL introduction](https://www.postgresql.org/docs/current/wal-intro.html).
- [PostgreSQL monitoring](https://www.postgresql.org/docs/current/monitoring.html), [routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html), [EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html), and [pgbench](https://www.postgresql.org/docs/current/pgbench.html).
- [pgAdmin](https://www.pgadmin.org/) is an optional UI for administration; use it as a complement to understanding the server and its documented behavior.

## Related notes

- [Database internals](database-internals.md) explains WAL, buffers, MVCC, and query execution.
- [Query performance labs](query-performance-labs.md) provides a repeatable measurement workflow.
- [Database selection and operations](database-selection-and-operations.md) covers store choice, migrations, backup concepts, and design review.
