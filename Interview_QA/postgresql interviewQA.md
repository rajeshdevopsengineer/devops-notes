Absolutely — here’s a **Senior DevOps Engineer–level PostgreSQL interview bank of 100 questions with concise, interview-ready answers**, covering deployment, configuration, HA/DR, performance tuning, observability, security, backup/restore, Kubernetes/cloud, and troubleshooting.

## 1. PostgreSQL Fundamentals & Architecture

### 1. What is PostgreSQL?

**Answer:** PostgreSQL is an open-source object-relational database management system (ORDBMS) known for ACID compliance, MVCC, extensibility, advanced SQL features, JSON/JSONB support, and strong transactional consistency.

### 2. Explain PostgreSQL's architecture.

**Answer:** PostgreSQL uses a **process-based architecture**. A server has a postmaster/main server process and multiple backend processes, typically one per client connection. Shared memory contains structures such as `shared_buffers`, WAL buffers, and lock-related data.

### 3. What is MVCC?

**Answer:** Multi-Version Concurrency Control allows transactions to see consistent snapshots without readers blocking writers. PostgreSQL maintains row versions and uses transaction IDs to determine tuple visibility.

### 4. What is a PostgreSQL cluster?

**Answer:** A PostgreSQL cluster is a collection of databases managed by one PostgreSQL server instance and sharing the same data directory, configuration, WAL, and background processes.

### 5. What is the PostgreSQL data directory?

**Answer:** It contains database files, WAL, configuration, metadata, and transaction state. Its location can be obtained with:

```sql
SHOW data_directory;
```

### 6. What is WAL?

**Answer:** **Write-Ahead Logging** records changes to WAL before corresponding data pages are persisted. WAL provides crash recovery, durability, replication, and point-in-time recovery.

### 7. What is a checkpoint?

**Answer:** A checkpoint flushes dirty buffers to disk and records a checkpoint location in WAL. It limits crash-recovery work but excessive checkpoints can increase I/O.

### 8. What happens when PostgreSQL crashes?

**Answer:** PostgreSQL starts crash recovery using WAL. It replays WAL records from the last consistent checkpoint to restore database pages to a consistent state.

### 9. What is a transaction ID?

**Answer:** PostgreSQL assigns transaction IDs to transactions. MVCC uses transaction IDs to determine whether row versions are visible to a transaction.

### 10. What is transaction ID wraparound?

**Answer:** Transaction IDs have a finite range. If old transaction IDs aren't frozen, PostgreSQL can eventually risk data corruption or shutdown. **VACUUM/freezing** prevents this.

---

# 2. PostgreSQL Deployment

### 11. How would you deploy PostgreSQL in production?

**Answer:** I would define requirements first—version, workload, storage, HA, RPO/RTO, security, connection count, and backup requirements. Then automate installation and configuration using tools such as Ansible/Terraform, configure monitoring and backups, validate performance, and document rollback procedures.

### 12. How would you deploy PostgreSQL on Linux?

**Answer:** Typically:

1. Install the appropriate PostgreSQL packages.
2. Initialize the database cluster if required.
3. Configure networking/authentication.
4. Configure storage and PostgreSQL parameters.
5. Enable/start the service.
6. Create roles/databases.
7. Configure backups and monitoring.
8. Validate with health checks.

### 13. What should you consider when choosing a PostgreSQL version?

**Answer:** Support lifecycle, application compatibility, extension compatibility, upgrade complexity, performance improvements, security fixes, and operational tooling compatibility.

### 14. How do you automate PostgreSQL deployments?

**Answer:** I use Infrastructure as Code for infrastructure and configuration management for OS/database configuration. Database schema changes should generally be managed separately through version-controlled migration tooling.

### 15. How would you perform a zero/minimal-downtime PostgreSQL upgrade?

**Answer:** For major versions, options include logical replication, replication-based migration, or tools such as `pg_upgrade` depending on downtime requirements. I would test the upgrade, validate extensions, establish rollback, synchronize data, switch traffic, and monitor carefully.

### 16. `pg_upgrade` vs logical replication?

**Answer:** `pg_upgrade` is usually faster for in-place major-version upgrades but requires downtime. Logical replication can support a low-downtime migration because changes can be replicated while the target is running.

### 17. How do you validate a new PostgreSQL deployment?

**Answer:** I check:

* Connectivity
* Authentication
* Version
* Storage
* WAL
* Replication
* Backup/restore
* Configuration
* Extensions
* Query performance
* Monitoring
* Failover
* Security

### 18. What are common PostgreSQL deployment mistakes?

**Answer:** Overprovisioning connections, incorrect memory settings, poor storage selection, missing backups, unrestricted network access, insufficient monitoring, replication without WAL/slot planning, and testing only the happy path.

### 19. How would you deploy PostgreSQL in Kubernetes?

**Answer:** I would generally use a mature PostgreSQL operator rather than manually managing StatefulSets. The operator should handle provisioning, replication, failover, backups, upgrades, and configuration while Kubernetes manages scheduling and storage.

### 20. What are the challenges of PostgreSQL on Kubernetes?

**Answer:** Persistent storage performance, pod scheduling, failure handling, network reliability, backup strategy, fencing during failover, replication, upgrades, and ensuring database availability isn't dependent on a single Kubernetes node.

---

# 3. PostgreSQL Configuration

### 21. What is `postgresql.conf`?

**Answer:** It's the primary PostgreSQL configuration file containing parameters for memory, WAL, connections, logging, autovacuum, planner behavior, replication, and other server behavior.

### 22. What is `pg_hba.conf`?

**Answer:** It controls **client authentication and access rules**. It specifies database/user/source-address combinations and authentication methods.

Example:

```text
host    appdb    appuser    10.10.0.0/16    scram-sha-256
```

### 23. What is the difference between `listen_addresses` and `pg_hba.conf`?

**Answer:** `listen_addresses` determines which network interfaces PostgreSQL accepts connections on. `pg_hba.conf` determines which clients are allowed to authenticate and how.

### 24. How do you check PostgreSQL configuration?

**Answer:**

```sql
SHOW shared_buffers;
SHOW work_mem;
SHOW max_connections;
```

For all settings:

```sql
SELECT name, setting, unit, source
FROM pg_settings;
```

### 25. How do you reload PostgreSQL configuration?

**Answer:**

```sql
SELECT pg_reload_conf();
```

Or using the service/control tooling.

A reload applies parameters that are reloadable; parameters requiring a restart won't change until PostgreSQL restarts.

### 26. How do you determine whether a parameter requires restart?

**Answer:**

```sql
SELECT name, context
FROM pg_settings
WHERE name = 'shared_buffers';
```

A `postmaster` context generally requires a restart.

### 27. What is `shared_buffers`?

**Answer:** It defines PostgreSQL's shared memory cache for database pages. It should be sized according to workload and system memory rather than blindly applying a fixed percentage.

### 28. What is `work_mem`?

**Answer:** `work_mem` is memory available for an individual query operation such as sorting or hashing. Importantly, a single query can use `work_mem` multiple times and concurrently, so increasing it globally can cause memory exhaustion.

### 29. What is `maintenance_work_mem`?

**Answer:** It's memory available for maintenance operations such as `VACUUM`, `CREATE INDEX`, and certain `ALTER TABLE` operations. It can generally be larger than `work_mem`.

### 30. What is `effective_cache_size`?

**Answer:** It is a planner estimate of the amount of memory available for caching data, including PostgreSQL and OS cache. It doesn't allocate memory itself.

---

# 4. Connection Management

### 31. What does `max_connections` control?

**Answer:** It controls the maximum number of concurrent client connections, subject to reserved superuser connections and other constraints.

### 32. Why shouldn't you simply increase `max_connections`?

**Answer:** Each connection consumes resources. Excessive connections can cause memory pressure, context switching, lock contention, and poor performance.

### 33. How would you handle thousands of PostgreSQL connections?

**Answer:** Use a connection pooler such as **PgBouncer**, right-size PostgreSQL connections, use appropriate pooling modes, and investigate whether applications are leaking or unnecessarily holding connections.

### 34. What is PgBouncer?

**Answer:** PgBouncer is a lightweight PostgreSQL connection pooler that reduces connection overhead and allows many application clients to share a smaller number of database connections.

### 35. How do you find current connections?

```sql
SELECT usename, client_addr, state, count(*)
FROM pg_stat_activity
GROUP BY usename, client_addr, state;
```

### 36. How do you find long-running queries?

```sql
SELECT pid,
       now() - query_start AS duration,
       usename,
       state,
       query
FROM pg_stat_activity
WHERE query_start IS NOT NULL
ORDER BY duration DESC;
```

### 37. How do you terminate a problematic session?

```sql
SELECT pg_terminate_backend(pid);
```

Use carefully, particularly with production workloads.

### 38. What is the difference between `pg_cancel_backend()` and `pg_terminate_backend()`?

**Answer:** `pg_cancel_backend()` cancels the currently running query while retaining the connection. `pg_terminate_backend()` terminates the entire backend session.

---

# 5. PostgreSQL Performance Tuning

### 39. How do you approach PostgreSQL performance troubleshooting?

**Answer:** I first establish symptoms and baselines, then examine CPU, memory, disk I/O, connections, locks, replication, database statistics, slow queries, execution plans, and vacuum health. I fix the root cause rather than randomly changing configuration.

### 40. What is `EXPLAIN`?

**Answer:** `EXPLAIN` displays the query execution plan chosen by PostgreSQL.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 100;
```

### 41. What is `EXPLAIN ANALYZE`?

**Answer:** It actually executes the query and reports real execution statistics such as actual rows, timing, loops, and planning/execution time.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 100;
```

### 42. What is the danger of `EXPLAIN ANALYZE`?

**Answer:** It executes the query. On `UPDATE`, `DELETE`, or other modifying statements, this can modify data. Use appropriate transactions or safer approaches when analyzing production queries.

### 43. What is a sequential scan?

**Answer:** PostgreSQL reads the table pages sequentially. It's not automatically bad—if a large percentage of rows is required, a sequential scan may be faster than using an index.

### 44. What is an index scan?

**Answer:** PostgreSQL uses an index to locate relevant rows and then accesses table data as necessary.

### 45. What is a bitmap heap scan?

**Answer:** PostgreSQL first identifies matching row locations using one or more indexes and then accesses heap pages efficiently in batches.

### 46. Why might PostgreSQL ignore an index?

**Answer:** Possible reasons include:

* Low selectivity
* Small table
* High random I/O cost
* Stale statistics
* Function/cast preventing index use
* Query returns many rows
* Planner estimates favor another plan

### 47. How do you identify slow queries?

**Answer:** Use `pg_stat_statements`, PostgreSQL logging, monitoring systems, and `pg_stat_activity`.

### 48. What is `pg_stat_statements`?

**Answer:** It's an extension that tracks query execution statistics, including execution count, total time, average time, rows, and I/O-related statistics depending on configuration/version.

### 49. How would you optimize a slow query?

**Answer:** I'd inspect:

1. SQL
2. Execution plan
3. Cardinality estimates
4. Indexes
5. Statistics
6. Joins
7. Sort/hash operations
8. I/O
9. Lock waits
10. Application behavior

Then make the smallest validated change.

### 50. What causes bad query plans?

**Answer:** Common causes include stale statistics, data distribution changes, incorrect estimates, correlated columns, inappropriate indexes, parameter sensitivity, or configuration that poorly represents the hardware/workload.

---

# 6. Indexing

### 51. What index types does PostgreSQL support?

**Answer:** Common types include **B-tree, Hash, GiST, SP-GiST, GIN, and BRIN**. Each serves different access patterns.

### 52. When would you use a B-tree index?

**Answer:** It's the default choice for equality and range comparisons and supports ordering efficiently.

### 53. When would you use a GIN index?

**Answer:** GIN is useful for composite values such as arrays and JSONB, and for certain full-text-search workloads.

### 54. When would you use a BRIN index?

**Answer:** BRIN is useful for very large tables where column values correlate with physical row order—for example, timestamped append-heavy data.

### 55. What is a composite index?

**Answer:** An index containing multiple columns:

```sql
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date);
```

Column order matters because PostgreSQL can exploit the index differently depending on predicates and ordering.

### 56. What is a partial index?

**Answer:** An index covering only rows satisfying a condition.

```sql
CREATE INDEX idx_active_users
ON users(email)
WHERE active = true;
```

### 57. What is a covering index?

**Answer:** An index containing columns needed by a query so PostgreSQL may be able to perform an index-only scan.

```sql
CREATE INDEX idx_customer
ON orders(customer_id)
INCLUDE (order_date, amount);
```

### 58. Can too many indexes hurt performance?

**Answer:** Yes. Every insert/update/delete may require index maintenance, indexes consume storage and cache, and excessive indexes can increase vacuum and write overhead.

### 59. How do you find unused indexes?

**Answer:** `pg_stat_user_indexes` can help identify indexes with little or no scan activity, but usage statistics must be interpreted carefully because counters reset and workload patterns vary.

### 60. What is `CREATE INDEX CONCURRENTLY`?

**Answer:** It creates an index with reduced blocking of normal writes compared with a regular index build. It takes longer and has additional operational considerations.

---

# 7. VACUUM & Autovacuum

### 61. Why does PostgreSQL need VACUUM?

**Answer:** PostgreSQL's MVCC leaves dead row versions after updates/deletes. VACUUM helps reclaim/reuse space, update visibility information, and prevent transaction ID wraparound.

### 62. What is autovacuum?

**Answer:** Autovacuum automatically performs vacuuming and analysis based on table activity and configurable thresholds.

### 63. Why is autovacuum important for performance?

**Answer:** Without adequate vacuuming, dead tuples accumulate, table/index bloat increases, statistics become stale, and transaction ID wraparound risk increases.

### 64. How would you troubleshoot excessive table bloat?

**Answer:** Investigate update/delete workload, autovacuum thresholds, long-running transactions, dead tuples, indexes, and vacuum effectiveness. Depending on severity, consider more aggressive vacuuming or table/index maintenance.

### 65. What is the difference between VACUUM and VACUUM FULL?

**Answer:** `VACUUM` generally reclaims/reuses dead space without rewriting the whole table. `VACUUM FULL` rewrites the table and can return disk space to the OS but requires stronger locking and significant I/O.

### 66. What is `ANALYZE`?

**Answer:** It collects table statistics used by the query planner to estimate row counts and choose execution plans.

### 67. How can long-running transactions affect vacuum?

**Answer:** A transaction holding an old snapshot can prevent PostgreSQL from removing row versions that might still be visible to that transaction, causing bloat.

### 68. How do you find old transactions?

```sql
SELECT pid,
       usename,
       xact_start,
       now() - xact_start AS age,
       state,
       query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

---

# 8. High Availability & Replication

### 69. What is streaming replication?

**Answer:** A primary sends WAL records to a standby, which receives and replays them. It is commonly used for high availability and read scaling.

### 70. What is physical replication?

**Answer:** Physical replication replicates the database at the WAL/block level, producing a standby with essentially the same physical database contents as the primary.

### 71. What is logical replication?

**Answer:** Logical replication publishes logical changes to selected tables or datasets, allowing more flexible replication and migration scenarios.

### 72. Physical vs logical replication?

**Answer:**

| Physical                                     | Logical                            |
| -------------------------------------------- | ---------------------------------- |
| WAL/block-level                              | Logical row changes                |
| Typically whole cluster                      | Selective objects                  |
| Strong HA use case                           | Migration/integration              |
| Same major-version constraints traditionally | More flexible version combinations |
| Replicates physical state                    | Replicates logical changes         |

### 73. What is synchronous replication?

**Answer:** The primary waits for confirmation from configured synchronous standby(s) before considering a transaction committed according to the selected synchronous-commit behavior. It can improve durability but adds latency.

### 74. What is asynchronous replication?

**Answer:** The primary doesn't wait for a standby before completing the commit. It provides lower latency but introduces potential replication lag and data loss during catastrophic failure.

### 75. What is replication lag?

**Answer:** Replication lag represents how far a standby is behind the primary. It can arise from network throughput, WAL generation, standby I/O, replay speed, or queries blocking replay.

### 76. How do you check replication status?

On the primary:

```sql
SELECT *
FROM pg_stat_replication;
```

On a standby:

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

### 77. What is a replication slot?

**Answer:** A replication slot ensures required WAL isn't removed before a consumer has received it. An inactive or stalled slot can cause WAL to accumulate and fill disk.

### 78. What happens if WAL fills the disk?

**Answer:** PostgreSQL may eventually stop accepting writes or become unhealthy. I would immediately identify the WAL consumer/slot, check replication, archive status, and disk usage, and safely remediate the underlying cause.

### 79. What is failover?

**Answer:** Failover promotes a standby to primary after the current primary becomes unavailable. Automation requires careful handling to avoid split-brain.

### 80. How do you prevent split-brain?

**Answer:** Use reliable failure detection, quorum/fencing, consensus or cluster-management mechanisms, and ensure the old primary cannot continue serving writes after another node is promoted.

---

# 9. Backup & Disaster Recovery

### 81. What PostgreSQL backup strategies do you know?

**Answer:**

* `pg_dump`
* `pg_dumpall`
* Physical base backups
* WAL archiving
* Point-in-time recovery
* Storage-level snapshots, when appropriately coordinated

### 82. `pg_dump` vs physical backup?

**Answer:** `pg_dump` produces a logical backup suitable for object-level migration and selective restoration. Physical backups are better suited for full-cluster recovery and PITR.

### 83. What is PITR?

**Answer:** **Point-in-Time Recovery** restores a base backup and replays archived WAL until a specified recovery point.

### 84. How do you design PostgreSQL backups for production?

**Answer:** Follow the required RPO/RTO, keep regular physical backups plus WAL archiving where appropriate, store backups separately from the database, encrypt them, monitor backup success, and regularly perform restoration tests.

### 85. Why is a successful backup job not enough?

**Answer:** A backup isn't useful unless it can be restored. Restore testing validates backup integrity, permissions, tooling, retention, recovery procedures, and actual RTO.

### 86. How would you recover a deleted table?

**Answer:** If PITR is available, restore a separate PostgreSQL instance to a point immediately before deletion, then extract the required table/data and import it into production.

### 87. What is RPO?

**Answer:** **Recovery Point Objective** is the maximum acceptable amount of data loss measured in time.

### 88. What is RTO?

**Answer:** **Recovery Time Objective** is the maximum acceptable time required to restore service after an incident.

---

# 10. Troubleshooting

### 89. PostgreSQL is suddenly slow. What do you check?

**Answer:** I would systematically check:

* CPU
* Memory/swap
* Disk latency/IOPS
* Connections
* Locks
* Long-running transactions
* Replication lag
* Autovacuum
* Bloat
* Slow queries
* Query-plan changes
* Recent deployments/config changes

I would first establish whether the problem is **database-wide or query-specific**.

### 90. CPU is 100%. How do you troubleshoot?

**Answer:** Identify expensive PostgreSQL processes and queries, inspect `pg_stat_activity` and `pg_stat_statements`, examine execution plans, and correlate with OS metrics. I would determine whether CPU is caused by inefficient queries, excessive concurrency, bad plans, or maintenance activity.

### 91. Disk I/O is extremely high. What do you investigate?

**Answer:** Check:

* `shared_buffers`
* Cache hit ratio
* Sequential scans
* Missing/ineffective indexes
* Checkpoint activity
* WAL generation
* Autovacuum
* Temporary files
* Query plans
* Storage latency
* Bloat

### 92. PostgreSQL says "too many connections." What do you do?

**Answer:** First identify connection sources and states. Then determine whether there is a connection leak, oversized pool, traffic spike, or insufficient pooling. I would use PgBouncer where appropriate rather than blindly increasing `max_connections`.

### 93. Queries are waiting on locks. How do you troubleshoot?

**Answer:** Inspect `pg_stat_activity` and lock relationships in `pg_locks`, identify the blocking PID, determine why it holds the lock, and resolve the underlying transaction/application problem.

### 94. How do you identify blocking queries?

**Answer:** A useful approach is:

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocking.pid AS blocking_pid,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_l
  ON blocked.pid = blocked_l.pid
JOIN pg_locks blocking_l
  ON blocking_l.locktype = blocked_l.locktype
 AND blocking_l.database IS NOT DISTINCT FROM blocked_l.database
 AND blocking_l.relation IS NOT DISTINCT FROM blocked_l.relation
 AND blocking_l.granted
JOIN pg_stat_activity blocking
  ON blocking.pid = blocking_l.pid
WHERE NOT blocked_l.granted;
```

For production, I'd use a tested query appropriate to the PostgreSQL version and lock types involved.

### 95. What causes connection failures to PostgreSQL?

**Answer:** Common causes include:

* PostgreSQL service down
* Wrong host/port
* `listen_addresses`
* Firewall/security group
* `pg_hba.conf`
* DNS
* Authentication failures
* Connection exhaustion
* TLS configuration
* Network/load-balancer problems

### 96. PostgreSQL won't start. What do you do?

**Answer:** Check:

1. Service status
2. PostgreSQL logs
3. Data-directory permissions
4. Disk space
5. Configuration syntax
6. Port conflicts
7. WAL/recovery state
8. Filesystem availability
9. Recent configuration changes

I avoid deleting WAL or database files as a "fix."

---

# 11. Security

### 97. How do you secure PostgreSQL in production?

**Answer:** Use:

* TLS
* SCRAM authentication
* Least-privilege roles
* Restricted network access
* Secure `pg_hba.conf`
* Encrypted backups
* Secrets management
* Auditing/logging
* Regular patching
* Separation of administrative and application roles

### 98. What is SCRAM?

**Answer:** SCRAM is a stronger password-authentication mechanism supported by PostgreSQL. `scram-sha-256` is generally preferred over older MD5 password authentication for modern deployments.

### 99. How would you manage PostgreSQL secrets?

**Answer:** I would avoid storing passwords in source code or static configuration repositories. Use a secrets manager such as a cloud secret service, Vault, or Kubernetes Secrets with appropriate encryption/access controls, and rotate credentials periodically.

### 100. You're paged at 2 AM: PostgreSQL production is down. Walk me through your response.

**Answer:**

> "First, I establish impact and whether the database is actually unavailable or merely unhealthy. I check monitoring, PostgreSQL service status, logs, connectivity, CPU/memory/disk, and replication status.
>
> If the primary is genuinely down, I determine whether the HA system can safely promote a healthy standby. Before promotion, I verify replication health and fencing to prevent split-brain.
>
> If recovery is possible on the existing primary, I avoid unnecessary failover. If failover is required, I promote the standby, redirect application traffic, and validate writes and application health.
>
> After service restoration, I investigate the root cause, validate backups and replication, rebuild/rejoin the failed node, and document the incident. Finally, I'd conduct a post-incident review and implement preventive measures."

---

# Senior-Level Scenario Questions

For a **Senior DevOps Engineer**, interviewers will often go beyond definitions and ask scenario-based questions.

Here are several high-value scenarios to practice:

### Scenario 1: "PostgreSQL CPU suddenly jumped from 30% to 95%. Nothing was deployed."

A strong answer should investigate:

**Application → PostgreSQL → OS → infrastructure**

Check:

```sql
SELECT pid, usename, state, query,
       now() - query_start AS duration
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Then inspect `pg_stat_statements`, execution plans, locks, statistics, and OS metrics.

Possible causes include a changed query plan, traffic increase, stale statistics, missing index, batch job, or autovacuum.

---

### Scenario 2: "Replication lag is continuously increasing."

Check:

1. WAL generation rate
2. Network throughput
3. Standby disk latency
4. WAL receiver
5. WAL replay
6. Long-running queries
7. Replication slots
8. Standby resource utilization

The important senior-level point is to determine **where the bottleneck is** rather than immediately increasing replication-related parameters.

---

### Scenario 3: "Database disk is 95% full."

Don't immediately delete files.

Check:

```bash
df -h
```

Then investigate PostgreSQL:

* WAL growth
* Replication slots
* Archived WAL
* Temporary files
* Table/index growth
* Logs
* Bloat
* Backup files

A replication slot retaining WAL is a particularly important failure mode.

---

### Scenario 4: "A query that used to take 100 ms now takes 30 seconds."

I'd compare:

* Old/new execution plans
* Statistics
* Data distribution
* Index availability
* Query parameters
* PostgreSQL version/configuration
* Table growth
* Lock waits
* I/O

Then use:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

The key is to determine whether the query became **CPU-bound, I/O-bound, lock-bound, or planner-bound**.

---

### Scenario 5: "Autovacuum isn't keeping up."

I would check:

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_vacuum,
       last_autovacuum,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Then investigate:

* Table-specific thresholds
* Vacuum cost settings
* Worker availability
* Long-running transactions
* High update/delete rates
* Disk I/O
* Transaction ID age

For heavily updated tables, **per-table autovacuum settings** can be more appropriate than changing global settings.

---

# What Interviewers Expect From a Senior DevOps Engineer

For senior roles, don't answer PostgreSQL questions as a DBA who only knows SQL commands.

Frame your answers around:

**Reliability → Automation → Observability → Performance → Security → Recovery**

For example, instead of:

> "I would increase `work_mem`."

Say:

> "I'd first identify whether sorts or hash operations are spilling to disk. I'd inspect the execution plan and temporary-file metrics, estimate concurrent memory consumption, and then consider increasing `work_mem` at the session/role level rather than globally. I'd validate the change under realistic concurrency."

That distinction demonstrates **senior-level operational judgment**.

### The 15 topics I'd prioritize before a senior DevOps interview

1. PostgreSQL architecture
2. WAL and checkpoints
3. MVCC
4. `pg_hba.conf`
5. Memory configuration
6. Connection pooling/PgBouncer
7. `EXPLAIN (ANALYZE, BUFFERS)`
8. Indexing
9. VACUUM/autovacuum
10. Locks and blocking
11. Streaming replication
12. Failover/split-brain
13. Backup + PITR
14. Monitoring/`pg_stat_statements`
15. Production incident troubleshooting
