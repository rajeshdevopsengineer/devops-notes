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

# Top 100 Senior DevOps Engineer Interview Questions & Answers — PostgreSQL

A reference covering **Deployment**, **Configuration**, **Performance Tuning**, and **Troubleshooting**.

---

## Section 1: Deployment & Architecture (1–25)

**1. What are the main ways to deploy PostgreSQL in production?**
Package manager (apt/yum) install on VMs, containerized (Docker/Kubernetes with operators like Zalando's Postgres Operator, CloudNativePG, or Crunchy PGO), managed cloud services (RDS, Cloud SQL, Azure Database for PostgreSQL), or building from source for custom compile flags. Choice depends on control needs, scale, and ops maturity.

**2. How do you set up streaming replication?**
Enable `wal_level = replica` on the primary, create a replication user, configure `pg_hba.conf` to allow replication connections, take a base backup (`pg_basebackup`) on the replica, and configure `primary_conninfo` in the replica's configuration (via `postgresql.auto.conf` or a `standby.signal` file in PG12+).

**3. What is the difference between physical and logical replication?**
Physical replication (streaming/WAL-based) replicates the entire cluster byte-for-byte and is used for HA/failover. Logical replication replicates at the row/table level via `pgoutput`, allowing selective replication, cross-version upgrades, and different schemas between publisher and subscriber.

**4. What tools are commonly used for PostgreSQL HA?**
Patroni (with etcd/Consul/ZooKeeper for consensus), repmgr, pg_auto_failover, and cloud-native operators. Patroni is the most widely adopted for self-managed HA due to its robust automated failover and health-check API.

**5. How does Patroni achieve automatic failover?**
Patroni runs as an agent on each node, storing cluster state in a distributed configuration store (DCS). It uses leader-election locks with TTLs; if the primary's lock expires (health check failure), Patroni promotes the most caught-up replica and reconfigures the rest to follow it.

**6. What is a base backup and how do you take one?**
A base backup is a full binary copy of the data directory taken while the server is running, using `pg_basebackup`, which coordinates with WAL to ensure consistency. It's the foundation for both replicas and PITR (point-in-time recovery).

**7. Explain Point-in-Time Recovery (PITR).**
PITR restores a base backup and replays WAL archives up to a specific point (timestamp, transaction ID, or named restore point), enabling recovery to just before a bad transaction (e.g., accidental DELETE) rather than only to backup time.

**8. What's the difference between `pg_dump` and physical backups?**
`pg_dump` is a logical backup (SQL statements or custom-format archive of objects/data), portable across versions and platforms but slower to restore for large databases. Physical backups (`pg_basebackup`, filesystem snapshots) are faster for large clusters but tied to the same major version and architecture.

**9. How would you architect zero-downtime major version upgrades?**
Use logical replication: set up the new version as a subscriber to the old primary, let it catch up, run application-level validation, then cut over traffic (DNS/connection pooler switch) once lag is near zero. Alternatively use `pg_upgrade --link` for faster in-place upgrades with brief downtime.

**10. What is `pg_upgrade` and when would you avoid `--link` mode?**
`pg_upgrade` migrates data files between major versions without a full dump/reload. `--link` mode hard-links files instead of copying, making it fast, but you must avoid it if you need to keep the old cluster as a rollback fallback, since both clusters share the same data files.

**11. How do you deploy PostgreSQL on Kubernetes reliably?**
Use a purpose-built operator (CloudNativePG, Zalando, or Crunchy PGO) rather than raw StatefulSets — operators handle failover, backups, connection routing, and rolling upgrades. Use persistent volumes with appropriate storage classes (low-latency SSD), pod anti-affinity across nodes/zones, and PodDisruptionBudgets.

**12. What is connection pooling and why is it critical in production?**
PostgreSQL uses one OS process per connection, which is expensive at scale. Poolers like PgBouncer or Pgpool-II multiplex many client connections onto fewer backend connections, reducing memory/CPU overhead and preventing connection exhaustion.

**13. Compare PgBouncer's session, transaction, and statement pooling modes.**
Session mode ties a client to a backend for the whole session (safest, least efficient). Transaction mode releases the backend after each transaction (most common in production, but breaks session-level features like `SET`, prepared statements without care). Statement mode releases after each statement (rarely used, breaks multi-statement transactions).

**14. How do you plan storage/disk layout for a PostgreSQL server?**
Separate WAL (`pg_wal`) onto its own fast disk/volume from data (`PGDATA`) to avoid I/O contention, use separate volumes for large tablespaces if needed, and ensure enough headroom for WAL archiving delays and autovacuum bloat.

**15. What is a tablespace and when would you use one?**
A tablespace maps a location on disk to store specific database objects. Used to place hot tables/indexes on faster storage, isolate large archival tables on cheaper storage, or balance I/O across multiple disks.

**16. How do you automate PostgreSQL provisioning (IaC)?**
Use Terraform/Ansible/Pulumi for infrastructure, combined with configuration management (Ansible playbooks, or Helm charts/operator CRDs in Kubernetes) to template `postgresql.conf`, `pg_hba.conf`, users, extensions, and initial schema.

**17. What are the trade-offs of running PostgreSQL as a managed service vs. self-hosted?**
Managed services (RDS/Cloud SQL) offload patching, backups, and failover but limit access to superuser features, custom extensions, and OS-level tuning. Self-hosted gives full control (custom extensions, kernel tuning, exact version control) at the cost of operational burden.

**18. How do you handle blue-green deployments for schema changes?**
Use expand-contract migrations: add new columns/tables (expand) compatible with both old and new app versions, deploy the new app version, then remove deprecated columns (contract) only after all consumers have migrated — avoiding locking incompatible changes during cutover.

**19. What is a WAL archive and why is it needed even with replication?**
WAL archiving (`archive_mode = on`, `archive_command`) copies completed WAL segments to durable storage (S3, etc.), independent of streaming replication. It's required for PITR and for restoring from a backup taken at any earlier point, not just current replica state.

**20. How do you design a DR (disaster recovery) strategy for PostgreSQL?**
Combine cross-region physical replicas (async, given latency) with WAL archiving to object storage in a separate region, define RPO/RTO targets, regularly test restore/failover drills, and automate DNS/connection-string cutover.

**21. What's the role of `synchronous_commit` and `synchronous_standby_names` in deployment design?**
They control whether a transaction commit waits for confirmation from one or more standbys before returning success to the client, trading latency for durability guarantees (zero data loss on failover vs. lower write latency).

**22. How do you containerize PostgreSQL correctly (common pitfalls)?**
Use official images with a named volume for `PGDATA` (never store data in the container layer), set proper `shm_size` (shared memory) for larger workloads, pass tuned config via `postgresql.conf` mount or command args, and never run multiple Postgres instances against the same data directory.

**23. What is `pg_rewind` used for?**
It resynchronizes a former primary (or diverged replica) with a new primary after failover, by replaying only the changed data blocks since divergence, avoiding a full re-clone from scratch.

**24. How do you handle schema migrations safely at scale (large tables)?**
Avoid long-locking operations: use `CREATE INDEX CONCURRENTLY`, add columns with defaults using fast-path default (PG11+, no full rewrite for constant defaults), batch data backfills, and use tools like `pg_repack` or online-migration frameworks to avoid blocking writes.

**25. What should a PostgreSQL deployment runbook include?**
Backup/restore procedures with tested RTO/RPO, failover steps (manual and automated), scaling procedures (read replica addition), upgrade procedures, credential rotation, monitoring/alerting thresholds, and escalation contacts.

---

## Section 2: Configuration (26–50)

**26. What are the most important `postgresql.conf` parameters to tune first?**
`shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size`, `max_connections`, `wal_buffers`, `checkpoint_completion_target`, and `random_page_cost`.

**27. How do you size `shared_buffers`?**
Typically 25% of system RAM as a starting point (not more than ~40% in most cases), since the OS page cache also caches file data and excessive shared_buffers can cause double-buffering and slower checkpointing.

**28. What does `work_mem` control and what's the risk of setting it too high?**
`work_mem` sets the memory available per sort/hash operation *per query node* (not per query or connection). Setting it too high risks memory exhaustion under concurrent load, since many operations can each allocate up to that amount simultaneously.

**29. What is `maintenance_work_mem` used for?**
Memory allocated for maintenance operations like `VACUUM`, `CREATE INDEX`, and `ALTER TABLE ADD FOREIGN KEY`. Setting it higher (e.g., 512MB–2GB) speeds up index builds and vacuum on large tables.

**30. Explain `effective_cache_size` and its impact on the query planner.**
It doesn't allocate memory — it tells the planner an estimate of how much memory is available for disk caching (shared_buffers + OS cache), influencing whether it prefers index scans over sequential scans.

**31. How do you configure `max_connections` properly?**
Set it based on actual concurrency needs (not application thread count) since each connection consumes ~5–10MB of memory; typically kept in the low hundreds, with a connection pooler in front for higher apparent concurrency.

**32. What's the purpose of `wal_level` and its settings?**
Controls how much information is written to WAL. `minimal` (least, no replication support), `replica` (default, supports streaming replication and PITR), `logical` (adds data needed for logical decoding/replication).

**33. How do checkpoint-related parameters affect performance?**
`checkpoint_timeout` and `max_wal_size` control how often checkpoints occur; `checkpoint_completion_target` (e.g., 0.9) spreads checkpoint I/O over the interval to avoid I/O spikes. Too-frequent checkpoints cause excess I/O; too infrequent increases recovery time after a crash.

**34. What is `random_page_cost` and how should it be tuned for SSDs?**
It represents the planner's cost estimate for a non-sequential disk read relative to `seq_page_cost` (default 1.0). Default is 4.0 (tuned for spinning disks); on SSDs, lowering it to 1.1–1.5 better reflects real costs and encourages index usage.

**35. How do you configure `pg_hba.conf` securely?**
Use the most restrictive matching rule first, prefer `scram-sha-256` over `md5` for password auth, restrict by specific IP/CIDR rather than `0.0.0.0/0`, and require SSL (`hostssl`) for remote connections.

**36. What authentication methods does PostgreSQL support and when would you use each?**
`trust` (local dev only), `password`/`scram-sha-256` (standard credential auth), `cert` (mutual TLS, high security), `peer`/`ident` (OS-user mapping, local/trusted networks), and `ldap`/`gssapi` for enterprise directory integration.

**37. How do you configure SSL/TLS for PostgreSQL connections?**
Set `ssl = on`, provide `ssl_cert_file`/`ssl_key_file` (and CA file for client cert verification), then enforce via `pg_hba.conf` using `hostssl` entries, and optionally require `sslmode=verify-full` on clients.

**38. What is `autovacuum` and what parameters control its aggressiveness?**
A background process reclaiming dead tuples and updating statistics. Key parameters: `autovacuum_vacuum_scale_factor`/`autovacuum_vacuum_threshold` (trigger conditions), `autovacuum_max_workers`, `autovacuum_vacuum_cost_limit` (throttling), and `autovacuum_naptime`.

**39. How do you tune autovacuum for a high-write table?**
Lower the scale factor (e.g., from 0.2 to 0.05) and/or set a table-specific `autovacuum_vacuum_scale_factor`/`threshold` via `ALTER TABLE ... SET`, increase `autovacuum_vacuum_cost_limit`, and consider dedicating more `autovacuum_max_workers`.

**40. What is the purpose of `wal_buffers` and how is it typically set?**
Memory for WAL data not yet written to disk. Usually auto-tuned (`-1`, ~3% of shared_buffers, capped at 16MB) which is sufficient for most workloads; manual increase rarely needed except very high write throughput.

**41. How do you configure logging for effective troubleshooting?**
Enable `log_min_duration_statement` (to log slow queries), `log_line_prefix` with timestamp/PID/user/db, `log_checkpoints`, `log_lock_waits`, `log_temp_files`, and `log_autovacuum_min_duration`, shipped to a centralized log system.

**42. What is `log_min_duration_statement` and how do you choose its threshold?**
Logs any statement taking longer than the specified milliseconds. Threshold should be set based on your SLA (e.g., 200–500ms for OLTP) — too low floods logs, too high misses meaningful slow queries.

**43. How do you configure parameters differently per role or database?**
Use `ALTER ROLE ... SET parameter = value` or `ALTER DATABASE ... SET parameter = value` for overrides like `statement_timeout`, `search_path`, or `work_mem` without changing global config.

**44. What's the difference between `postgresql.conf`, `postgresql.auto.conf`, and `ALTER SYSTEM`?**
`postgresql.conf` is the main file, hand-edited. `ALTER SYSTEM SET` writes to `postgresql.auto.conf`, which overrides the main file and persists across restarts without editing files directly — useful for automation but requires a reload/restart depending on the parameter.

**45. How do you determine if a config parameter requires a restart vs. reload?**
Query `pg_settings.context` — values like `postmaster` require a full restart, `sighup` only requires `pg_reload_conf()` or `SIGHUP`, and `user`/`session` can be changed per-session.

**46. What is `idle_in_transaction_session_timeout` and why configure it?**
Terminates sessions that stay idle inside an open transaction beyond the threshold, preventing long-held locks and bloat caused by forgotten/hung transactions holding back autovacuum's xmin horizon.

**47. How do you configure `statement_timeout` safely?**
Set a reasonable global default (e.g., 30s–60s) to prevent runaway queries, but override it per-role/session for legitimate long-running jobs (reporting, batch ETL) to avoid killing valid work.

**48. What extensions are commonly enabled in production PostgreSQL and why?**
`pg_stat_statements` (query performance tracking), `pg_repack` (bloat removal without long locks), `pg_cron` (in-database scheduling), `postgis` (geospatial), and `pgaudit` (compliance logging).

**49. How do you configure `pg_stat_statements` and what does it provide?**
Add it to `shared_preload_libraries`, restart, then `CREATE EXTENSION pg_stat_statements`. It aggregates execution statistics (calls, total/mean time, rows, I/O) per normalized query, essential for identifying top resource-consuming queries.

**50. What's the significance of `shared_preload_libraries`?**
Lists extensions/modules that must be loaded at server startup (not dynamically), such as `pg_stat_statements`, `pg_cron`, `auto_explain`. Requires a full restart to change.

---

## Section 3: Performance Tuning (51–75)

**51. How do you identify slow queries in production?**
Use `pg_stat_statements` ordered by `total_exec_time` or `mean_exec_time`, enable `log_min_duration_statement`, and use `auto_explain` to automatically log execution plans for slow queries.

**52. Walk through how you'd read an `EXPLAIN ANALYZE` output.**
Read bottom-up (innermost nodes execute first). Compare estimated vs. actual rows (large discrepancies indicate stale statistics), check for sequential scans on large tables, note buffer hits/reads (`BUFFERS` option), and identify the most expensive node by actual time.

**53. What's the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?**
`EXPLAIN` shows the planner's *estimated* execution plan without running the query. `EXPLAIN ANALYZE` actually executes the query and reports real timing/row counts — useful but be cautious with write queries (wrap in a transaction and rollback).

**54. When does PostgreSQL choose a sequential scan over an index scan, and is that always wrong?**
When the table/result set is small, when a large fraction of rows are being returned (index scan overhead exceeds sequential scan benefit), or when statistics are stale. It's not always wrong — for very small tables, sequential scans can be faster than index overhead.

**55. What types of indexes does PostgreSQL support and when do you use each?**
B-tree (default, equality/range), Hash (equality only, rarely used), GiST (geometric/full-text, nearest-neighbor), GIN (arrays, JSONB, full-text search), BRIN (large sequentially-correlated data like time-series), and SP-GiST (specialized partitioned data).

**56. When would you use a partial index?**
When queries consistently filter on a condition covering a small subset of rows (e.g., `WHERE status = 'active'` when 95% of rows are `'inactive'`), keeping the index small and fast: `CREATE INDEX ... WHERE status = 'active'`.

**57. What is index bloat and how do you detect/fix it?**
Dead tuple space in indexes from updates/deletes not yet reclaimed. Detect via `pgstattuple` or bloat-estimation queries; fix with `REINDEX CONCURRENTLY` (no long lock) or `pg_repack`.

**58. How do you decide between adding an index vs. rewriting a query?**
Profile first: if the plan shows a scan on a filter/join column with high selectivity and no suitable index, add one. If the query itself does unnecessary work (e.g., `SELECT *`, unneeded joins, non-sargable predicates like functions on indexed columns), rewrite it first.

**59. What is a covering index and how does it enable index-only scans?**
An index that includes all columns needed to satisfy a query (via `INCLUDE` clause or composite key), letting PostgreSQL answer the query from the index alone without visiting the heap — provided the visibility map confirms pages are all-visible.

**60. Explain table partitioning and its performance benefits.**
Splitting a large table into smaller physical pieces (by range, list, or hash) transparently accessed via a parent table. Benefits: partition pruning (skip irrelevant partitions in queries), faster maintenance (drop old partitions instead of DELETE), and parallelizable operations.

**61. What is partition pruning and when does it fail to trigger?**
The planner's ability to skip scanning partitions that can't contain matching rows based on the WHERE clause. It fails when predicates use non-constant values the planner can't evaluate at plan time (mitigated by runtime pruning in newer versions) or when partition keys aren't referenced directly.

**62. How do you tune vacuum for a table with heavy bloat?**
Increase `autovacuum_vacuum_cost_limit` and decrease `autovacuum_vacuum_cost_delay` for that table to let vacuum work faster, lower the scale factor to trigger vacuum sooner, and consider `VACUUM FULL` or `pg_repack` for existing bloat (the former locks the table; the latter doesn't).

**63. What is transaction ID wraparound and how do you prevent it?**
PostgreSQL's XID is a 32-bit counter; if it wraps around without vacuum freezing old tuples, data can appear to vanish. Prevented by regular autovacuum freezing (monitor `age(datfrozenxid)`), and alerting well before `autovacuum_freeze_max_age` is reached.

**64. How does `VACUUM FREEZE` relate to wraparound prevention?**
It marks old tuples as "frozen" so their XID no longer needs comparison against the wraparound horizon, effectively resetting their age. Autovacuum does this automatically as tables approach `autovacuum_freeze_max_age`.

**65. What causes lock contention and how do you diagnose it?**
Concurrent DDL, long transactions holding row/table locks, or explicit `LOCK` statements. Diagnose via `pg_locks` joined with `pg_stat_activity`, looking for blocked/blocking PID chains.

**66. How do you find and resolve blocking queries?**
Query `pg_stat_activity` joined to `pg_locks` (or use the standard blocking-queries recipe) to find the blocking PID, inspect its query, and either wait, `pg_cancel_backend()` (graceful), or `pg_terminate_backend()` (forceful) if it's safe to do so.

**67. What are the main causes of high CPU utilization on a PostgreSQL server?**
Poorly optimized queries doing large sorts/hashes, missing indexes causing sequential scans, excessive connection churn without pooling, statistics-stale plans choosing bad join strategies, or runaway autovacuum on many tables simultaneously.

**68. How does `work_mem` misconfiguration cause disk I/O spikes?**
If `work_mem` is too low, sorts/hashes spill to disk (temp files), causing I/O overhead and slower queries; check `log_temp_files` and `pg_stat_statements`/`EXPLAIN` for "Sort Method: external merge" to confirm.

**69. What is parallel query execution and when does PostgreSQL use it?**
For large scans/aggregates/joins, PostgreSQL can split work across background workers (`max_parallel_workers_per_gather`), governed by cost thresholds (`parallel_setup_cost`, `min_parallel_table_scan_size`). It's disabled for small tables or queries with side effects.

**70. How do you tune for a read-heavy vs. write-heavy workload differently?**
Read-heavy: maximize cache hit ratio (shared_buffers, effective_cache_size), add read replicas, tune indexes aggressively. Write-heavy: tune checkpoint/WAL settings for throughput, consider `synchronous_commit = off` where durability trade-off is acceptable, batch writes, and watch for index write amplification.

**71. What's the impact of too many indexes on write performance?**
Every INSERT/UPDATE/DELETE must maintain all indexes on the row, increasing write latency and WAL volume. Regularly audit unused indexes via `pg_stat_user_indexes` (idx_scan = 0) and drop them.

**72. How do you use `pg_stat_activity` to monitor real-time load?**
It shows all current backend processes, their state (`active`, `idle`, `idle in transaction`), current query, wait events, and duration — critical for spotting long-running or stuck queries live.

**73. What is a "wait event" and how does it help diagnose bottlenecks?**
`pg_stat_activity.wait_event_type`/`wait_event` shows what a backend is blocked on (e.g., `Lock`, `IO`, `Client`), letting you distinguish between lock contention, disk I/O bottlenecks, or the client being slow to consume results.

**74. How would you approach query optimization for a slow reporting query joining 6+ tables?**
Check `EXPLAIN ANALYZE` for row-estimate mismatches (run `ANALYZE` if stale), verify join order and indexes on join/filter columns, consider materialized views for repeated heavy aggregations, and evaluate whether CTEs are inlined or optimization-fenced (pre-PG12 CTEs were always fenced).

**75. What is a materialized view and how does it differ from a regular view for performance?**
A materialized view stores the query result physically on disk and must be explicitly refreshed (`REFRESH MATERIALIZED VIEW`), trading data freshness for read speed — ideal for expensive aggregations queried often but not needing real-time data.

---

## Section 4: Troubleshooting (76–100)

**76. A production database suddenly has 100% CPU. What's your triage process?**
Check `pg_stat_activity` for currently running queries and their state/duration, identify top consumers, check for a sudden spike in connections or a bad deploy introducing an unindexed query, and check OS-level tools (`top`, `pidstat`) to correlate with specific backend PIDs.

**77. The database is running out of disk space rapidly. What do you check?**
Check for WAL accumulation (a stuck replication slot or failed archive_command preventing WAL recycling), runaway temp files from bad queries, table/index bloat, or unexpectedly large log files.

**78. How do you diagnose "too many connections" errors?**
Check `pg_stat_activity` for connection count and source (which app/host is opening most connections), verify pooler configuration, check for a connection leak in the application, and consider raising `max_connections` only as a last resort (with adequate memory).

**79. Replication lag has grown significantly. How do you investigate?**
Check `pg_stat_replication` on the primary for `write_lag`/`flush_lag`/`replay_lag`, verify network bandwidth/latency between nodes, check if the replica is CPU/I/O-bound from replaying large operations (e.g., bulk load, index build), and check for long-running queries on the replica blocking WAL replay (`hot_standby_feedback`/`max_standby_streaming_delay`).

**80. A replica is not catching up and `pg_stat_replication` shows increasing lag. What are common causes?**
Insufficient I/O or CPU capacity on the replica, network throughput limits, a single large transaction (e.g., bulk update) on the primary, or long-running read queries on the replica delaying WAL apply due to conflict resolution settings.

**81. What is a "replication slot" and how can it cause disk space issues?**
A replication slot ensures WAL isn't removed until a specific replica/subscriber has consumed it. If the consumer disconnects and the slot isn't dropped, WAL accumulates indefinitely on the primary — a very common cause of unexpected disk-full incidents.

**82. How do you troubleshoot "database is not accepting connections" errors?**
Check the PostgreSQL log for a startup failure reason (crash recovery in progress, corrupted WAL, out of shared memory), verify the process is actually running, check `pg_hba.conf` for auth misconfiguration, and confirm the port/firewall/network path.

**83. A query that used to be fast is now slow after a deploy. What do you check first?**
Whether an index was dropped/renamed inadvertently, whether table statistics are stale (`ANALYZE` needed after bulk data changes), whether the query plan changed (compare `EXPLAIN` before/after), or whether data volume/distribution has changed significantly (parameter sniffing-like behavior via bad cached plan in some frameworks).

**84. How do you investigate sudden autovacuum activity causing performance degradation?**
Check `pg_stat_progress_vacuum` for in-progress vacuum details, check if it's a wraparound-prevention (anti-wraparound) vacuum which can't be cancelled easily, and review `autovacuum_vacuum_cost_delay`/`cost_limit` to see if throttling is too aggressive or not aggressive enough.

**85. The application reports "deadlock detected" errors. How do you resolve this?**
Examine the deadlock log entries (PostgreSQL logs both queries and lock info), identify the conflicting access order (e.g., two transactions locking rows A→B and B→A), and fix by enforcing a consistent lock acquisition order in application code or using `SELECT ... FOR UPDATE` more surgically.

**86. How do you recover from accidental data deletion in production?**
If PITR is configured, restore to a point just before the deletion using WAL archives, either into a separate instance (safest, to extract and re-insert the lost rows) or by full restore + failover. Without PITR, options are limited to point-in-time snapshots or logical backups if available.

**87. What steps do you take when a `VACUUM FULL` or `REINDEX` accidentally locks a production table?**
Cancel it (`pg_cancel_backend`) if safe, since it's a fully blocking operation; reattempt with `pg_repack` or `REINDEX CONCURRENTLY`/`CREATE INDEX CONCURRENTLY` instead, which avoid an exclusive lock for the full duration.

**88. How do you troubleshoot "out of shared memory" or "out of memory" errors?**
Check OS-level `dmesg` for OOM-killer activity, review whether `work_mem` × concurrent connections × operations-per-query exceeds available RAM, check `shared_buffers` vs. total system memory, and consider `vm.overcommit_memory` kernel settings.

**89. A failover happened unexpectedly. How do you investigate the root cause?**
Check the HA tool's logs (Patroni/repmgr) for the health-check failure reason (timeout, connection refused), correlate with system metrics (was the primary under heavy load, swapping, or network-partitioned), and check DCS (etcd/Consul) logs for split-brain or leader-election issues.

**90. What is split-brain in a PostgreSQL HA cluster and how do you prevent it?**
When two nodes simultaneously believe they are primary (e.g., after a network partition), causing divergent writes. Prevented via fencing/STONITH mechanisms, quorum-based consensus (odd-numbered DCS cluster), and tools like Patroni that require DCS lock renewal to remain primary.

**91. How do you diagnose a sudden increase in "idle in transaction" sessions?**
Query `pg_stat_activity` filtered on `state = 'idle in transaction'`, identify the offending application/connection pattern (often a bug not committing/rolling back), and mitigate immediately with `idle_in_transaction_session_timeout` while fixing the root cause in app code.

**92. A backup restore test fails. What's your checklist?**
Verify the backup file/archive isn't corrupted (checksum), confirm WAL archive completeness for the target recovery point, check PostgreSQL version compatibility, verify sufficient disk space on the restore target, and check restore logs for the specific failure point.

**93. How do you investigate "could not extend file" or WAL-related I/O errors?**
This typically indicates a disk-full or filesystem-permission condition; check available disk space immediately, check filesystem quotas, and verify the WAL/data directory ownership and permissions haven't changed.

**94. What causes "canceling statement due to statement timeout" and how do you handle it appropriately?**
The query exceeded `statement_timeout`. Determine if it's a legitimate slow query needing optimization/indexing, or a batch job that needs a per-session timeout override rather than lowering the global timeout.

**95. How do you troubleshoot connection pooler (PgBouncer) issues like "no more connections allowed"?**
Check PgBouncer's `pool_size`/`max_client_conn` settings vs. actual demand, review PgBouncer logs for backend connection errors, and verify PostgreSQL's `max_connections` is sufficient for the pooler's configured pool size plus admin/monitoring connections.

**96. A table's query performance degrades over time despite no schema changes. What's the likely cause and fix?**
Table/index bloat from accumulated dead tuples not being vacuumed fast enough for the write rate, or stale statistics causing suboptimal plans. Fix: tune autovacuum aggressiveness for that table, run manual `ANALYZE`, and consider `pg_repack` if bloat is already significant.

**97. How do you detect and handle corrupted data pages?**
Enable `data_checksums` (set at initdb time) so PostgreSQL detects corruption on read and logs it; use `pg_checksums` to verify, and recover the affected relation from a known-good backup or replica since corrupted pages generally can't be repaired in place.

**98. What is the process for safely terminating a long-running problematic query in production?**
First attempt `pg_cancel_backend(pid)` (sends a cancel request, allows clean rollback); if unresponsive, escalate to `pg_terminate_backend(pid)` (kills the backend process, triggering a full connection reset) — reserve termination for cases where cancellation doesn't work.

**99. How do you investigate high `idx_scan`/heap fetches suggesting an index isn't being used efficiently?**
Compare `pg_stat_user_indexes` (scan counts) against expected access patterns, check for index bloat, verify statistics are current, and confirm the index actually matches query predicates (e.g., a composite index's column order must align with query filters for full use).

**100. Describe your overall incident-response approach for a production PostgreSQL outage.**
Triage impact and scope first (is it down, degraded, or partial), stabilize (failover if HA is available, kill offending queries, free disk space), communicate status to stakeholders, gather diagnostic data before restarting anything (logs, `pg_stat_activity` snapshots), perform root-cause analysis post-incident, and update runbooks/monitoring to catch the issue earlier next time.

---

Absolutely. Below are **50 additional PostgreSQL Administration interview questions and answers**, focused specifically on what a **Senior DevOps Engineer / PostgreSQL Administrator** is expected to know: operations, configuration, HA, security, maintenance, monitoring, upgrades, automation, and incident response.

## PostgreSQL Administration — 50 More Interview Questions & Answers

### 101. How do you check the PostgreSQL server version?

**Answer:**

From the command line:

```bash
psql --version
```

From SQL:

```sql
SELECT version();
```

Or:

```sql
SHOW server_version;
```

For automation, I generally prefer `SHOW server_version` or querying `pg_settings`.

---

### 102. How do you check whether PostgreSQL is running?

**Answer:**

On Linux with systemd:

```bash
systemctl status postgresql
```

I also verify database-level availability:

```bash
pg_isready -h localhost -p 5432
```

A service being "running" doesn't necessarily mean the database is healthy, so I prefer an actual connectivity/health check.

---

### 103. How do you check which port PostgreSQL is using?

**Answer:**

```sql
SHOW port;
```

Or:

```bash
ss -lntp | grep postgres
```

The default PostgreSQL port is `5432`.

---

### 104. How do you check PostgreSQL's current configuration file locations?

**Answer:**

```sql
SHOW config_file;
SHOW hba_file;
SHOW data_directory;
```

This is preferable to assuming a fixed filesystem path because installations can use different layouts.

---

### 105. How do you change a PostgreSQL parameter without editing `postgresql.conf`?

**Answer:**

For a session:

```sql
SET work_mem = '128MB';
```

For a role:

```sql
ALTER ROLE appuser SET work_mem = '64MB';
```

For a database:

```sql
ALTER DATABASE appdb SET work_mem = '64MB';
```

This allows targeted configuration instead of globally increasing resource consumption.

---

### 106. What is the difference between `SET`, `ALTER ROLE`, and `ALTER SYSTEM`?

**Answer:**

* `SET` — affects the current session.
* `ALTER ROLE` — applies defaults to sessions belonging to a role.
* `ALTER DATABASE` — applies defaults to connections to a database.
* `ALTER SYSTEM` — writes a parameter to `postgresql.auto.conf`.

Example:

```sql
ALTER SYSTEM SET log_min_duration_statement = '1000';
```

I use `ALTER SYSTEM` carefully because configuration management should ideally have a single source of truth.

---

### 107. What is `postgresql.auto.conf`?

**Answer:**

It is a configuration file automatically maintained by PostgreSQL when using `ALTER SYSTEM`.

For example:

```sql
ALTER SYSTEM SET shared_buffers = '4GB';
```

writes the setting to `postgresql.auto.conf`.

In infrastructure-as-code environments, I generally avoid having configuration split unpredictably between automation and `ALTER SYSTEM`.

---

### 108. How do you reload PostgreSQL configuration?

**Answer:**

```sql
SELECT pg_reload_conf();
```

Or:

```bash
pg_ctl reload
```

A reload doesn't restart the server. Parameters requiring a restart remain pending.

---

### 109. How do you identify parameters requiring a restart?

**Answer:**

Query `pg_settings`:

```sql
SELECT name, setting, context
FROM pg_settings
WHERE context = 'postmaster';
```

`postmaster` parameters generally require a server restart.

---

### 110. How do you find configuration changes that haven't taken effect?

**Answer:**

`pg_settings` contains useful information such as the current setting, source, and whether a restart is pending.

For example:

```sql
SELECT name, setting, source, pending_restart
FROM pg_settings
WHERE pending_restart;
```

This is particularly useful after automated configuration changes.

---

## Database and Role Administration

### 111. How do you list PostgreSQL databases?

**Answer:**

Using `psql`:

```text
\l
```

Or SQL:

```sql
SELECT datname
FROM pg_database;
```

---

### 112. How do you create a database?

**Answer:**

```sql
CREATE DATABASE appdb;
```

You can also specify an owner:

```sql
CREATE DATABASE appdb
OWNER appuser;
```

---

### 113. How do you create a PostgreSQL user?

**Answer:**

Modern PostgreSQL terminology uses roles:

```sql
CREATE ROLE appuser LOGIN PASSWORD 'strong-password';
```

I would normally manage credentials through a secrets-management system rather than putting passwords directly into deployment scripts.

---

### 114. What is the difference between a role and a user?

**Answer:**

PostgreSQL has a unified role system. A "user" is essentially a role with the `LOGIN` attribute.

For example:

```sql
CREATE ROLE reporting;
CREATE ROLE analyst LOGIN;
```

A role can be used as a group to grant privileges.

---

### 115. How do you implement least privilege?

**Answer:**

Create separate roles for applications, read-only users, reporting, migrations, and administrators.

For example:

```sql
CREATE ROLE app_readonly;
GRANT CONNECT ON DATABASE appdb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
```

I avoid giving applications superuser privileges.

---

### 116. How do you list PostgreSQL roles?

**Answer:**

In `psql`:

```text
\du
```

Or:

```sql
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin
FROM pg_roles;
```

---

### 117. How do you revoke privileges?

**Answer:**

```sql
REVOKE INSERT, UPDATE, DELETE
ON orders
FROM appuser;
```

For larger environments, I prefer role-based privilege management rather than granting permissions directly to individual users.

---

### 118. How do you prevent a user from creating databases?

**Answer:**

```sql
ALTER ROLE appuser NOCREATEDB;
```

Similarly:

```sql
ALTER ROLE appuser NOSUPERUSER;
ALTER ROLE appuser NOCREATEROLE;
```

---

### 119. How do you terminate all connections to a database?

**Answer:**

For example:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'appdb'
  AND pid <> pg_backend_pid();
```

I'd use this carefully because it terminates active application sessions.

---

### 120. How do you prevent new connections while performing maintenance?

**Answer:**

One approach is:

```sql
ALTER DATABASE appdb ALLOW_CONNECTIONS false;
```

Then terminate existing connections if necessary.

After maintenance:

```sql
ALTER DATABASE appdb ALLOW_CONNECTIONS true;
```

For production maintenance, I would coordinate this with application traffic management.

---

# PostgreSQL Logging & Monitoring

### 121. How do you enable slow-query logging?

**Answer:**

For example:

```sql
ALTER SYSTEM SET log_min_duration_statement = '1000';
```

This logs statements taking at least one second.

I'd normally combine this with centralized log collection rather than enabling extremely verbose logging indefinitely.

---

### 122. What PostgreSQL logs should a DevOps engineer monitor?

**Answer:**

Important events include:

* Startup/shutdown
* Authentication failures
* Connection errors
* Long-running queries
* Deadlocks
* Checkpoints
* Replication errors
* WAL/archive failures
* Autovacuum problems
* Out-of-memory conditions
* Disk-space problems
* Recovery/failover events

---

### 123. How do you detect deadlocks?

**Answer:**

PostgreSQL reports deadlocks in the server logs and terminates one of the conflicting transactions.

I'd investigate:

* Transaction ordering
* Lock acquisition order
* Long transactions
* Application concurrency
* Missing indexes causing excessive locking

The permanent fix is usually at the transaction/application design level.

---

### 124. What is `pg_stat_activity`?

**Answer:**

It provides information about current server processes and sessions.

Useful columns include:

```text
pid
usename
datname
client_addr
state
query
query_start
xact_start
wait_event
wait_event_type
```

It's one of my first tools during a production database incident.

---

### 125. What is `pg_stat_database`?

**Answer:**

It provides database-level statistics such as:

* Transactions
* Commits/rollbacks
* Blocks read
* Blocks hit
* Tuples returned
* Tuples inserted/updated/deleted
* Temporary files

It is useful for workload and health analysis.

---

### 126. How do you monitor PostgreSQL in production?

**Answer:**

I'd monitor four layers:

**Infrastructure**

* CPU
* Memory
* Disk
* IOPS
* Disk latency
* Network

**PostgreSQL**

* Connections
* Transactions
* Locks
* WAL
* Checkpoints
* Cache behavior
* Autovacuum

**Replication**

* Lag
* WAL retention
* Replay status
* Replication slots

**Application**

* Query latency
* Errors
* Throughput
* Connection-pool utilization

---

### 127. What PostgreSQL metrics are important for alerting?

**Answer:**

I would consider alerts for:

* Database unavailable
* Connection saturation
* Replication lag
* Disk usage
* WAL growth
* Replication-slot retention
* Long-running transactions
* Deadlocks
* Failed backups
* Autovacuum problems
* Transaction ID age
* Query latency

Alerts should be based on impact and trends rather than every unusual metric value.

---

### 128. How do you monitor transaction ID age?

**Answer:**

For example:

```sql
SELECT datname,
       age(datfrozenxid)
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

High transaction ID age indicates potential vacuum/freeze problems and should be treated seriously.

---

### 129. What is a "database is up" health check?

**Answer:**

A basic health check could be:

```bash
pg_isready -h db.example.com -p 5432
```

A stronger application health check should actually establish a connection and execute a lightweight query such as:

```sql
SELECT 1;
```

For HA environments, I'd distinguish **connectivity** from **read/write readiness**.

---

### 130. How would you build PostgreSQL observability?

**Answer:**

I'd combine:

**Metrics → Prometheus/exporter → Grafana**

**Logs → Fluent Bit/Vector → centralized logging**

**Queries → pg_stat_statements**

**Traces → application/OpenTelemetry**

The objective is correlation: an application latency spike should be traceable to database query latency, locks, CPU, I/O, or another database condition.

---

# Backup, Restore & Disaster Recovery

### 131. How do you verify that PostgreSQL backups are working?

**Answer:**

I don't consider a backup successful simply because the backup command returned exit code 0.

I'd:

1. Verify backup completion.
2. Validate backup size/integrity.
3. Verify WAL archiving.
4. Restore periodically into an isolated environment.
5. Run consistency/application checks.
6. Measure actual recovery time.

---

### 132. How do you test PITR?

**Answer:**

I would:

1. Take a base backup.
2. Enable and validate WAL archiving.
3. Generate known test data.
4. Record a recovery target.
5. Restore to a separate environment.
6. Replay WAL.
7. Validate the recovered state.

The goal is to prove the documented RPO/RTO rather than merely proving that backups exist.

---

### 133. What can cause WAL archive failures?

**Answer:**

Common causes include:

* Destination unavailable
* Network failure
* Credentials/permissions
* Storage full
* Incorrect archive command
* Object-storage outage
* Incorrect paths
* Slow archive destination

A critical point is that persistent archive failure can eventually cause disk pressure.

---

### 134. What happens if a backup is deleted accidentally?

**Answer:**

I'd determine whether another backup exists and whether WAL required for PITR is still available. Then I'd restore from the most recent valid backup and replay WAL as far as possible.

This is why backup redundancy and retention policies matter.

---

### 135. Where should production PostgreSQL backups be stored?

**Answer:**

Preferably in storage independent from the primary database infrastructure, with:

* Encryption
* Restricted access
* Retention policies
* Versioning/immutability where appropriate
* Cross-region copies for critical systems

The backup should survive failure of the production database environment.

---

# Replication & HA Administration

### 136. How do you promote a PostgreSQL standby?

**Answer:**

The exact command depends on PostgreSQL version and HA tooling, but PostgreSQL provides standby promotion mechanisms such as:

```bash
pg_ctl promote -D /path/to/data
```

In production, I'd normally let an HA manager/operator perform promotion because it must coordinate fencing, DNS/load balancing, replication state, and application traffic.

---

### 137. What should you check before promoting a standby?

**Answer:**

I'd verify:

* Primary is genuinely unavailable
* Standby is healthy
* Replication is functioning
* WAL position/lag
* Standby has sufficient data
* Fencing is in place
* No possibility of split-brain
* Application routing can switch

---

### 138. How do you rebuild a failed PostgreSQL replica?

**Answer:**

Typical process:

1. Stop the old replica.
2. Remove/reinitialize its data directory as appropriate.
3. Take a fresh base backup from the primary or another valid source.
4. Configure standby/replication settings.
5. Start PostgreSQL.
6. Verify WAL receiver/replay.
7. Confirm replication lag returns to normal.

For large databases, I would consider efficient backup/restore or replica cloning mechanisms.

---

### 139. What happens when a replication slot is inactive?

**Answer:**

The primary retains WAL required by that slot. If the consumer remains inactive, WAL can grow substantially and eventually consume disk space.

This is why replication-slot monitoring is critical.

---

### 140. How would you troubleshoot a standby that isn't receiving WAL?

**Answer:**

I'd check:

* `pg_stat_wal_receiver`
* Primary `pg_stat_replication`
* Network connectivity
* Authentication
* `pg_hba.conf`
* Replication user permissions
* Replication slot
* WAL retention
* Primary/standby configuration
* PostgreSQL logs

---

# PostgreSQL Maintenance

### 141. How often should VACUUM run?

**Answer:**

There isn't a universal schedule. PostgreSQL's autovacuum should normally handle routine vacuuming. High-churn tables may require table-specific tuning.

The important metric is whether dead tuples, bloat, and transaction age remain under control.

---

### 142. How do you manually vacuum a table?

**Answer:**

```sql
VACUUM ANALYZE orders;
```

This both cleans up dead tuples and refreshes planner statistics.

---

### 143. When would you use `VACUUM FULL`?

**Answer:**

Only when reclaiming physical disk space is necessary and the operational impact is acceptable.

`VACUUM FULL` rewrites the table and requires a strong lock, so it isn't something I'd casually run on a busy production table.

---

### 144. What is table bloat?

**Answer:**

Bloat is wasted/reusable space caused by obsolete row versions and index entries accumulating faster than maintenance can clean them.

It can increase disk usage and degrade I/O efficiency.

---

### 145. How do you reduce index bloat?

**Answer:**

Options include:

* `REINDEX`
* `REINDEX CONCURRENTLY` where supported/appropriate
* Better autovacuum behavior
* Addressing excessive update/delete patterns
* Redesigning problematic indexes

The choice depends on locking, disk space, and availability requirements.

---

### 146. Why is `ANALYZE` important after bulk loading?

**Answer:**

Bulk loading can significantly change data distribution. Without updated statistics, the optimizer may make poor cardinality estimates and select inefficient query plans.

---

### 147. What is `maintenance_work_mem` important for?

**Answer:**

It affects memory available for operations such as vacuuming and index creation. Increasing it can accelerate maintenance operations, but it must be sized with concurrency in mind.

---

### 148. How do you safely perform database maintenance in production?

**Answer:**

I first determine:

* Lock requirements
* Expected duration
* I/O impact
* Replication impact
* Application impact
* Rollback/recovery plan

Then schedule during a suitable window, test beforehand, monitor during execution, and validate afterward.

---

# Upgrades & DevOps Automation

### 149. How would you automate PostgreSQL administration?

**Answer:**

I would automate:

* Installation
* Configuration
* Role/database creation
* TLS
* Backup configuration
* Monitoring
* Replication
* Health checks
* Upgrades
* Disaster recovery testing

Infrastructure should be represented as code where practical, using tools such as Terraform and Ansible, while schema migrations remain under database migration tooling.

---

### 150. How would you safely manage PostgreSQL configuration through CI/CD?

**Answer:**

I would:

1. Store configuration in version control.
2. Review changes through pull requests.
3. Validate syntax before deployment.
4. Apply configuration automatically.
5. Determine whether reload or restart is required.
6. Perform health checks.
7. Monitor metrics after deployment.
8. Maintain rollback capability.

The key principle is **configuration drift prevention**.

---

## 10 Particularly Difficult Questions to Practice

If you're interviewing for a **Senior DevOps / SRE position**, I'd pay special attention to these:

| #  | Question                                                                | What interviewer is testing |
| -- | ----------------------------------------------------------------------- | --------------------------- |
| 1  | PostgreSQL is consuming all disk space—what do you do?                  | Incident response           |
| 2  | Replication lag keeps increasing—why?                                   | HA troubleshooting          |
| 3  | PostgreSQL is CPU-bound—how do you investigate?                         | Performance                 |
| 4  | Database suddenly has 10,000 connections—what happened?                 | Connection management       |
| 5  | A query changed from 100 ms to 30 seconds—why?                          | Query optimization          |
| 6  | How would you perform a major-version upgrade with minimal downtime?    | Upgrade strategy            |
| 7  | How do you guarantee backups are actually recoverable?                  | DR                          |
| 8  | How do you prevent PostgreSQL split-brain?                              | HA architecture             |
| 9  | Autovacuum isn't keeping up—what do you change?                         | MVCC/maintenance            |
| 10 | PostgreSQL won't start after a configuration change—how do you recover? | Production troubleshooting  |

### A strong senior-level troubleshooting framework

When answering almost any PostgreSQL incident question, use this structure:

**1. Establish impact**

> Is the entire database unavailable, or are only certain queries affected?

**2. Check recent changes**

> Deployment, configuration, traffic, schema, infrastructure, or failover?

**3. Check PostgreSQL**

> `pg_stat_activity`, `pg_locks`, `pg_stat_database`, `pg_stat_replication`, `pg_stat_statements`.

**4. Check infrastructure**

> CPU, memory, disk capacity, IOPS, latency, network.

**5. Identify the bottleneck**

> CPU-bound, I/O-bound, lock-bound, connection-bound, replication-bound, or planner/query-bound.

**6. Mitigate safely**

> Restore service first; avoid risky changes without understanding their impact.

**7. Find root cause**

> Don't stop after making the symptom disappear.

**8. Prevent recurrence**

> Automation, monitoring, alerting, capacity planning, testing, and runbooks.

That framework is often more impressive in a senior interview than simply memorizing PostgreSQL commands.


Absolutely. These 10 are the **high-value senior-level scenario questions** where the interviewer is looking for your troubleshooting methodology, not just PostgreSQL commands.

---

## 1. PostgreSQL is consuming all disk space — what do you do?

### Interview-ready answer

> "First, I would determine what is consuming the disk rather than immediately deleting files. I would check filesystem usage, PostgreSQL data directories, WAL, logs, temporary files, table/index growth, and replication slots.
>
> My first checks would be:
>
> ```bash
> df -h
> du -xh /var/lib/postgresql | sort -h | tail
> ```
>
> Then I'd investigate PostgreSQL itself:
>
> ```sql
> SELECT pg_size_pretty(pg_database_size(current_database()));
> ```
>
> I'd check WAL and replication slots, because an inactive replication slot can prevent WAL from being recycled and cause rapid disk growth.
>
> I'd also check table and index sizes, temporary files, long-running transactions, autovacuum, and database logs.
>
> If the disk is critically full, I'd first create breathing room using a safe operational action—for example, extending the volume or removing obsolete external logs/backups—rather than manually deleting PostgreSQL files.
>
> If WAL is accumulating because of a replication slot, I'd identify the consumer and fix or safely remove the slot only after confirming it is no longer required.
>
> Finally, I'd determine why the growth occurred and add monitoring and alerting to prevent recurrence."

### Commands worth knowing

```sql
-- Largest databases
SELECT datname,
       pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

Largest tables:

```sql
SELECT schemaname,
       relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

Replication slots:

```sql
SELECT slot_name,
       slot_type,
       active,
       restart_lsn,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS retained_wal
FROM pg_replication_slots;
```

### 🚨 Senior-level point

**Never manually delete files from `PGDATA`, `pg_wal`, or database relation directories to free disk space.** That's a potential data-corruption scenario.

---

# 2. Replication lag keeps increasing — why?

### Interview-ready answer

> "I'd first determine whether the lag is caused by network transfer, WAL generation on the primary, WAL receiving, or WAL replay on the standby.
>
> On the primary I'd check `pg_stat_replication`, and on the standby I'd check `pg_stat_wal_receiver`. I'd also check CPU, disk latency, network throughput, and WAL generation rate.
>
> If the primary is generating WAL faster than the standby can replay it, the standby may be CPU- or I/O-bound. A long-running query or recovery conflict on the standby can also prevent replay.
>
> I'd additionally check replication slots because a stalled slot can cause WAL retention on the primary.
>
> I would not simply increase replication-related parameters without identifying the bottleneck."

### Primary

```sql
SELECT pid,
       application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn
FROM pg_stat_replication;
```

### Standby

```sql
SELECT *
FROM pg_stat_wal_receiver;
```

### Check replay delay

Depending on PostgreSQL version/setup:

```sql
SELECT now() - pg_last_xact_replay_timestamp();
```

### Typical causes

| Cause            | Example                  |
| ---------------- | ------------------------ |
| Network          | Insufficient bandwidth   |
| Primary workload | Huge WAL generation      |
| Standby CPU      | WAL replay can't keep up |
| Standby disk     | High write latency       |
| Long query       | Recovery conflicts       |
| Replication slot | WAL retention            |
| Configuration    | Poorly sized resources   |
| Storage          | IOPS limitation          |

### 🚨 Senior-level answer

I'd identify **where the lag is occurring**:

**Primary WAL generation → network → WAL receive → standby flush → WAL replay**

That demonstrates much stronger troubleshooting than simply saying "the replica is slow."

---

# 3. PostgreSQL is CPU-bound — how do you investigate?

### Interview-ready answer

> "First I'd confirm whether PostgreSQL is actually consuming the CPU and whether the workload is database-wide or isolated to a few queries.
>
> At the OS level I'd check CPU utilization, load average, process-level usage, and whether the system is under memory pressure.
>
> Inside PostgreSQL I'd inspect `pg_stat_activity` and `pg_stat_statements` to identify expensive queries. Then I'd use `EXPLAIN (ANALYZE, BUFFERS)` on representative queries.
>
> I'd look for sequential scans, inefficient joins, excessive sorting or hashing, bad cardinality estimates, changed query plans, and excessive concurrency.
>
> I would also check whether autovacuum, index creation, or another maintenance operation is consuming CPU."

### First check

```sql
SELECT pid,
       usename,
       state,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE state <> 'idle';
```

### Find expensive queries

If `pg_stat_statements` is enabled:

```sql
SELECT calls,
       total_exec_time,
       mean_exec_time,
       rows,
       query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Then examine the plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

### Potential causes

* Bad execution plan
* Missing index
* Stale statistics
* Large sequential scan
* Expensive joins
* Excessive concurrency
* CPU-heavy aggregation
* Sort/hash operations
* Autovacuum
* Data growth

### 🚨 Senior-level point

Don't automatically add CPU or increase PostgreSQL parameters.

**First identify what is consuming the CPU.**

---

# 4. Database suddenly has 10,000 connections — what happened?

### Interview-ready answer

> "I'd first determine where those connections came from and what state they're in. I'd check `pg_stat_activity` grouped by user, application, client address, and state.
>
> Then I'd investigate the application connection pool configuration. A sudden connection spike can result from a connection leak, a deployment creating too many application instances, a pool misconfiguration, a traffic spike, or an unavailable PgBouncer layer.
>
> I would not immediately increase `max_connections`, because that can make the problem worse by consuming more memory and increasing contention.
>
> I'd stabilize the system by controlling incoming traffic, fixing or reducing the application pool size, terminating clearly idle/problematic sessions when safe, and introducing PgBouncer if appropriate."

### Find connection sources

```sql
SELECT usename,
       application_name,
       client_addr,
       state,
       count(*)
FROM pg_stat_activity
GROUP BY usename, application_name, client_addr, state
ORDER BY count(*) DESC;
```

### Check total connections

```sql
SELECT count(*)
FROM pg_stat_activity;
```

### Compare with limit

```sql
SHOW max_connections;
```

### What I'd investigate

**Application → connection pool → PgBouncer → PostgreSQL**

Potential causes:

* Connection leak
* Pool size multiplied by many application pods
* Traffic spike
* Deployment scaling
* Health checks opening connections
* PgBouncer failure
* Application retry storm

### 🚨 Senior-level point

If you have:

**200 application pods × 50 connections each = 10,000 connections**

the problem may not be PostgreSQL at all—it may be **poor connection-pool architecture**.

---

# 5. A query changed from 100 ms to 30 seconds — why?

### Interview-ready answer

> "I'd compare the old and current execution plans first. I'd determine whether the query is actually executing for 30 seconds or waiting on a lock.
>
> I'd check `pg_stat_activity`, locks, `pg_stat_statements`, and then run `EXPLAIN (ANALYZE, BUFFERS)` in a safe environment or against a carefully selected production query.
>
> I'd compare row estimates versus actual rows, indexes, statistics, table growth, query parameters, and execution-plan changes.
>
> Possible causes include stale statistics, data distribution changes, table growth, a missing or ineffective index, a changed query plan, parameter-related plan differences, or lock contention."

### Check whether it's blocked

```sql
SELECT pid,
       state,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE pid = <PID>;
```

If:

```text
wait_event_type = Lock
```

then optimizing the SQL may not solve the immediate problem.

### Check the plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Look for:

```text
Seq Scan
Nested Loop
Sort
Hash Join
Rows Removed by Filter
actual rows vs estimated rows
```

### 🚨 Senior-level answer

I'd separate:

**Query execution problem**

from:

**Query waiting problem**

A 30-second query could actually execute in 100 ms but spend 29.9 seconds waiting for a lock.

---

# 6. How would you perform a major-version upgrade with minimal downtime?

### Interview-ready answer

> "First I'd determine the required RTO/RPO and acceptable downtime, then validate application and extension compatibility with the target PostgreSQL version.
>
> For a conventional upgrade, `pg_upgrade` is often the fastest option, but it requires a maintenance window.
>
> If downtime must be minimized, I'd consider logical replication or another migration architecture. I'd provision the target cluster, configure replication, synchronize existing data, allow changes to catch up, validate the target, and then perform a controlled cutover.
>
> Before production, I'd perform the complete procedure in a staging environment using production-like data and measure the actual cutover time.
>
> I'd also have a rollback plan, monitoring, backup, and application compatibility testing."

### Upgrade strategy

```text
Current PostgreSQL
       |
       | logical replication
       v
New PostgreSQL
       |
       | synchronize
       v
Validation
       |
       | short cutover
       v
Application → New PostgreSQL
```

### Important checks

* Extensions
* Extensions' target-version compatibility
* Application drivers
* SQL behavior changes
* Authentication
* Configuration differences
* Replication
* Backup
* Monitoring
* Performance
* Rollback

### 🚨 Senior-level point

Don't say:

> "I'll upgrade PostgreSQL and test afterward."

Say:

> **"I'll rehearse the entire upgrade and rollback procedure before production."**

---

# 7. How do you guarantee backups are actually recoverable?

### Interview-ready answer

> "I don't consider a backup reliable merely because the backup job succeeded. I need to periodically perform restoration tests.
>
> I'd restore backups into an isolated environment, replay WAL when PITR is required, validate database consistency, verify critical tables and application functionality, and measure recovery time.
>
> I'd also monitor backup completion, WAL archiving, backup age, backup size, retention, and available storage.
>
> Finally, I'd document the recovery procedure and periodically perform disaster-recovery exercises."

### Backup lifecycle

```text
Backup
  ↓
Upload/store
  ↓
Integrity validation
  ↓
Restore test
  ↓
Application validation
  ↓
Measure RTO
  ↓
Document results
```

### Important distinction

**Backup success ≠ Recovery success**

The real test is:

> "Can I restore the database within the required RTO with the required RPO?"

---

# 8. How do you prevent PostgreSQL split-brain?

### Interview-ready answer

> "Split-brain occurs when two PostgreSQL nodes believe they are primary and both accept writes.
>
> Prevention requires reliable failure detection, controlled promotion, fencing, and a mechanism to ensure the old primary cannot continue serving writes after a new primary has been promoted.
>
> I would use a mature HA architecture rather than writing an ad-hoc shell script that promotes nodes based only on connectivity.
>
> The HA system should provide leader election or quorum, health checks, fencing where required, and controlled client routing."

### Dangerous scenario

```text
             Network partition
                    |
        +-----------+-----------+
        |                       |
     Node A                  Node B
   "I'm primary"           "I'm primary"
        |                       |
      WRITE                   WRITE
```

Now both nodes accept writes.

### Safer architecture

```text
             HA / Consensus
                  |
          +-------+-------+
          |               |
       Primary         Standby
          |               |
        WRITE           READ
```

If primary fails:

```text
Primary
   X
   |
Fencing
   |
Standby → promoted primary
```

### 🚨 Senior-level point

**Replication alone does not solve split-brain.**

You need **failure detection + fencing/coordination + client routing**.

---

# 9. Autovacuum isn't keeping up — what do you change?

### Interview-ready answer

> "First I would establish why autovacuum isn't keeping up instead of simply increasing the number of workers.
>
> I'd identify the tables with the highest dead-tuple counts and check their update/delete rate, autovacuum timestamps, transaction age, and table-specific settings.
>
> I'd investigate long-running transactions because they can prevent dead tuples from being removed.
>
> I'd also check I/O capacity and whether autovacuum is being throttled.
>
> For heavily updated tables, I'd consider table-specific autovacuum settings, such as lower scale factors and appropriate thresholds. I'd also consider increasing available maintenance resources if the infrastructure can support it.
>
> Finally, I'd monitor whether the changes actually reduce dead tuples and transaction age."

### Find problematic tables

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_autovacuum,
       autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Check transaction age

```sql
SELECT pid,
       usename,
       xact_start,
       now() - xact_start AS transaction_age,
       state,
       query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

### Example table-specific tuning

```sql
ALTER TABLE orders
SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

The exact values should be derived from table size and workload—not copied blindly.

### 🚨 Senior-level point

A common mistake is:

> "Increase `autovacuum_max_workers`."

A better answer is:

> **"Find out why vacuum is falling behind first."**

---

# 10. PostgreSQL won't start after a configuration change — how do you recover?

### Interview-ready answer

> "First I'd check the service status and PostgreSQL logs because they usually identify the configuration or startup problem.
>
> I'd determine exactly what changed and whether the parameter requires a restart. If the configuration is invalid, I'd revert the bad change using the configuration-management system or manually restore the previous known-good configuration.
>
> I'd verify file ownership and permissions, disk space, port conflicts, data-directory availability, and WAL/recovery state.
>
> I would avoid deleting PostgreSQL files or attempting destructive recovery without understanding the cause.
>
> Once PostgreSQL starts, I'd verify database connectivity, replication, application connectivity, and monitoring before declaring the incident resolved."

### First checks

```bash
systemctl status postgresql
```

Then:

```bash
journalctl -u postgresql
```

And PostgreSQL logs.

### Check configuration

```sql
SHOW config_file;
```

If PostgreSQL cannot start, locate the configured file from the service/environment/package layout.

### Common causes

| Problem               | Example                             |
| --------------------- | ----------------------------------- |
| Invalid configuration | Typo in parameter                   |
| Permission            | Wrong ownership                     |
| Port conflict         | Another process on 5432             |
| Disk full             | No space for startup/WAL            |
| Memory                | Impossible memory configuration     |
| Data directory        | Missing/unavailable filesystem      |
| WAL/recovery          | Recovery-related failure            |
| TLS                   | Invalid certificate/key permissions |

### 🚨 Senior-level recovery principle

**Preserve evidence first.**

Don't immediately delete files, initialize a new database, remove WAL, or recreate the cluster.

---

# Bonus: How I Would Answer These in a Real Interview

A strong senior answer follows this pattern:

### **1. Assess**

> "First I'd establish the scope and customer impact."

### **2. Observe**

> "I'd check PostgreSQL metrics, logs, OS metrics, and recent changes."

### **3. Isolate**

> "I'd determine whether the bottleneck is CPU, memory, I/O, locks, connections, WAL, replication, or the application."

### **4. Mitigate**

> "I'd restore service with the lowest-risk mitigation available."

### **5. Diagnose**

> "Then I'd identify the root cause rather than stopping at symptom resolution."

### **6. Prevent**

> "Finally I'd add monitoring, automation, capacity planning, or configuration changes to prevent recurrence."

For example, if asked **"PostgreSQL is slow"**, don't jump directly to `shared_buffers`.

A senior response is:

```text
PostgreSQL slow
      |
      +-- Is DB reachable?
      |
      +-- CPU?
      |
      +-- Memory?
      |
      +-- Disk latency?
      |
      +-- Connections?
      |
      +-- Locks?
      |
      +-- Replication?
      |
      +-- Autovacuum?
      |
      +-- Slow queries?
              |
              +-- EXPLAIN
              +-- indexes
              +-- statistics
              +-- joins
              +-- I/O
```

**That's the mindset interviewers usually want from a Senior DevOps Engineer:** don't blindly tune PostgreSQL—**measure → isolate → mitigate → fix → prevent**.


