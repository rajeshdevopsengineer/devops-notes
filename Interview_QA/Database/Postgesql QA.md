Below is a Senior DevOps–focused PostgreSQL interview guide. It emphasizes operational judgment, production deployment, HA/DR, configuration, tuning, observability, automation, and incident troubleshooting—not just SQL syntax.

# Top 100 PostgreSQL Interview Questions and Answers

## Senior DevOps Engineer — Deployment, Configuration, Performance Tuning & Troubleshooting

**Version baseline:** PostgreSQL 18.x, with most concepts applicable to PostgreSQL 14–18. PostgreSQL 18.6 is the current PostgreSQL 18 release as of August 13, 2026. PostgreSQL 19 is still a development/beta branch. ([PostgreSQL][1])

---

# Section 1 — PostgreSQL Architecture and Core Concepts

## 1. What is PostgreSQL and why is it commonly used for production workloads?

**Answer:** PostgreSQL is an open-source relational database management system with ACID transactions, MVCC concurrency control, WAL-based durability, sophisticated indexing, replication, partitioning, JSON support, extensions, and strong SQL standards support.

For a DevOps Engineer, PostgreSQL is important because operating it involves much more than starting a service. Production responsibilities include:

* storage and filesystem design
* backups and PITR
* replication and failover
* connection management
* monitoring
* security
* OS tuning
* capacity planning
* upgrades
* automation
* incident response

A senior engineer should understand how PostgreSQL interacts with CPU, memory, disks, networking, containers, Kubernetes, and cloud infrastructure.

---

## 2. What is the difference between a PostgreSQL cluster, database, schema, and table?

**Answer:**

A **PostgreSQL cluster** is one PostgreSQL server instance managed by one data directory.

Inside the cluster there can be multiple:

**Databases → Schemas → Tables/Indexes/Views**

Example:

```text
PostgreSQL Instance
│
├── database1
│   ├── public
│   │   ├── customers
│   │   └── orders
│   └── reporting
│
└── database2
    └── public
```

A PostgreSQL cluster also shares certain global objects, especially:

* roles
* tablespaces
* configuration
* WAL infrastructure

This distinction matters during backup and migration. For example, `pg_dump` normally backs up one database while `pg_dumpall` can also capture cluster-wide objects such as roles. ([PostgreSQL][2])

---

## 3. Explain the PostgreSQL process architecture.

**Answer:** PostgreSQL traditionally uses a **process-per-connection architecture** rather than one thread per connection.

The main server process accepts connections and creates backend processes. Other important background processes include:

* checkpointer
* background writer
* WAL writer
* autovacuum launcher/workers
* archiver
* WAL sender
* WAL receiver
* logical replication workers
* parallel query workers

You can inspect processes at OS level:

```bash
ps -ef | grep postgres
```

And sessions through:

```sql
SELECT pid,
       usename,
       datname,
       application_name,
       client_addr,
       state,
       query
FROM pg_stat_activity;
```

`pg_stat_activity` provides one row per PostgreSQL server process/session. ([PostgreSQL][3])

**Senior-level consideration:** Large `max_connections` values can therefore create significant memory and process overhead. Connection pooling is usually preferable to blindly increasing connections.

---

## 4. What is MVCC?

**Answer:** MVCC means **Multi-Version Concurrency Control**.

Instead of overwriting an existing row directly, PostgreSQL can create a new row version. Transactions see the version appropriate to their transaction snapshot.

For example:

```sql
UPDATE accounts
SET balance = 1000
WHERE id = 10;
```

PostgreSQL generally creates a new tuple version and leaves the previous tuple until it becomes reclaimable.

Benefits:

* readers usually do not block writers
* writers usually do not block readers
* transactions obtain consistent snapshots

The trade-off is **dead tuples**.

These eventually need to be cleaned up by `VACUUM`.

That is one reason autovacuum is absolutely critical in PostgreSQL. PostgreSQL documentation identifies vacuuming as necessary for reclaiming/reusing dead tuple space, planner statistics, visibility-map maintenance, and protection from transaction-ID wraparound. ([PostgreSQL][4])

---

## 5. What is WAL?

**Answer:** WAL means **Write-Ahead Logging**.

Before PostgreSQL considers persistent data changes safe, information describing those changes is written to WAL.

The key principle is:

```text
Write WAL first
      ↓
Flush WAL
      ↓
Data pages can be written later
```

If PostgreSQL crashes before dirty data pages reach disk, the database can replay WAL during recovery.

WAL is fundamental to:

* crash recovery
* streaming replication
* PITR
* WAL archiving
* logical decoding
* backups

PostgreSQL documents WAL as the mechanism where changes are logged before corresponding data-file modifications need to be persisted, enabling REDO recovery after crashes. ([PostgreSQL][5])

The WAL directory is normally:

```text
$PGDATA/pg_wal/
```

A full `pg_wal` filesystem can stop database writes or eventually bring the instance down, making WAL capacity monitoring critical.

---

## 6. What is a PostgreSQL checkpoint?

**Answer:** A checkpoint ensures dirty data buffers are synchronized to persistent data files and establishes a known recovery point.

Checkpoint activity is controlled primarily through parameters such as:

```text
checkpoint_timeout
max_wal_size
checkpoint_completion_target
```

Frequent checkpoints can cause:

* heavy write I/O
* increased WAL generation
* latency spikes

Very infrequent checkpoints can mean:

* larger WAL requirements
* longer crash recovery

PostgreSQL normally spreads checkpoint writes over time. The default `checkpoint_completion_target` is designed to avoid large bursts of writes. ([PostgreSQL][6])

Useful monitoring:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

A senior engineer should correlate application latency with checkpoint activity rather than simply tuning one parameter in isolation.

---

## 7. What transaction isolation levels does PostgreSQL support?

**Answer:** PostgreSQL exposes:

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

`READ UNCOMMITTED` is accepted syntactically but behaves as `READ COMMITTED`.

**READ COMMITTED** is commonly the default and creates a new statement-level snapshot.

**REPEATABLE READ** maintains a consistent transaction snapshot.

**SERIALIZABLE** attempts to make concurrent execution behave as though transactions executed serially and can abort transactions when serialization anomalies are detected.

Applications using SERIALIZABLE therefore need retry logic.

Example:

```text
ERROR: could not serialize access due to read/write dependencies
```

A DevOps engineer should not treat that message automatically as a database failure—it can be expected application-level concurrency behavior.

---

## 8. What types of locks exist in PostgreSQL?

**Answer:** PostgreSQL has multiple locking mechanisms.

Common table-level modes include:

```text
ACCESS SHARE
ROW SHARE
ROW EXCLUSIVE
SHARE UPDATE EXCLUSIVE
SHARE
SHARE ROW EXCLUSIVE
EXCLUSIVE
ACCESS EXCLUSIVE
```

Row-level locking includes operations generated by:

```sql
SELECT ... FOR UPDATE;
SELECT ... FOR SHARE;
UPDATE;
DELETE;
```

To inspect lock contention:

```sql
SELECT *
FROM pg_locks;
```

A practical blocking query:

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocker.pid AS blocking_pid,
    blocked.query AS blocked_query,
    blocker.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocker
  ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

For senior troubleshooting, always identify the **root blocker**, not merely all waiting sessions.

---

## 9. What is a PostgreSQL deadlock?

**Answer:** A deadlock occurs when transactions wait on each other in a circular dependency.

Example:

```text
Transaction A
locks Row 1
waits Row 2

Transaction B
locks Row 2
waits Row 1
```

PostgreSQL eventually detects the cycle and aborts one transaction.

Typical error:

```text
ERROR: deadlock detected
```

Common mitigation:

* access resources in consistent order
* keep transactions short
* reduce unnecessary locks
* create appropriate indexes
* avoid user interaction inside transactions
* add retry logic

Useful configuration:

```text
deadlock_timeout
log_lock_waits
```

A senior engineer should inspect logs to determine the statements participating in the deadlock rather than increasing `deadlock_timeout` as a supposed fix.

---

## 10. What is a HOT update?

**Answer:** HOT means **Heap-Only Tuple** update.

PostgreSQL can sometimes avoid inserting new index entries when:

1. the updated columns are not indexed, and
2. sufficient space exists on the same heap page.

This reduces:

* index writes
* WAL generation
* index bloat
* update cost

For update-heavy workloads, adjusting table `fillfactor` can create space for HOT updates.

Example:

```sql
ALTER TABLE customer
SET (fillfactor = 80);
```

A lower fillfactor intentionally leaves room in table pages for future versions.

Check HOT statistics:

```sql
SELECT relname,
       n_tup_upd,
       n_tup_hot_upd
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC;
```

---

# Section 2 — Deployment and Production Architecture

## 11. What does `initdb` do?

**Answer:** `initdb` creates a new PostgreSQL database cluster/data directory.

Example:

```bash
initdb -D /var/lib/postgresql/18/main
```

It creates components including:

```text
base/
global/
pg_wal/
postgresql.conf
pg_hba.conf
pg_ident.conf
```

Production considerations include:

* filesystem ownership
* locale
* encoding
* checksums
* authentication defaults
* WAL/data storage
* filesystem capacity

Database initialization choices should therefore be part of infrastructure automation rather than undocumented manual setup.

---

## 12. Describe your production PostgreSQL deployment checklist.

**Answer:** A senior production checklist should cover at least:

**Infrastructure**

```text
CPU sizing
RAM sizing
storage latency/IOPS
filesystem capacity
network redundancy
availability zones
```

**PostgreSQL**

```text
shared_buffers
work_mem
maintenance_work_mem
max_connections
WAL configuration
autovacuum
logging
authentication
TLS
```

**Reliability**

```text
standby replica
WAL archive
physical backups
PITR
restore testing
monitoring
alerting
```

**Operations**

```text
automated configuration
secrets management
upgrade procedure
failover procedure
runbooks
capacity forecasting
```

Deployment should not be considered complete simply because:

```bash
systemctl status postgresql
```

returns `active`.

---

## 13. How would you deploy PostgreSQL on Linux?

**Answer:** Typical steps:

```text
1. Install supported PostgreSQL packages
2. Provision dedicated storage
3. Initialize the cluster
4. Configure postgresql.conf
5. Configure pg_hba.conf
6. Configure TLS
7. Configure OS/systemd limits
8. Start PostgreSQL
9. Create roles/databases
10. Enable monitoring
11. Configure backups/WAL archive
12. Create replica
13. Test recovery/failover
```

Example service operations:

```bash
systemctl enable postgresql
systemctl start postgresql
systemctl status postgresql
```

Production configuration should normally be deployed through configuration management or IaC rather than manual edits.

Examples:

```text
Ansible
Chef
Puppet
Terraform + cloud-init
Kubernetes Operator
```

---

## 14. What is the difference between a PostgreSQL minor and major upgrade?

**Answer:** PostgreSQL versioning distinguishes major-version upgrades from maintenance/minor upgrades.

A minor update within the same major release generally keeps the on-disk data format compatible.

For example:

```text
18.5 → 18.6
```

normally means update binaries/packages and restart.

A major upgrade such as:

```text
17 → 18
```

requires an upgrade strategy such as:

* `pg_upgrade`
* logical replication
* dump/restore

`pg_upgrade` is designed to perform major upgrades without requiring traditional full dump/restore migration. ([PostgreSQL][7])

Always read release notes for extension and compatibility issues.

---

## 15. How would you perform a PostgreSQL major-version upgrade?

**Answer:** Three common strategies exist.

### Method 1 — pg_upgrade

Best when:

* maintenance downtime is acceptable
* local storage/data files are available

Process:

```text
install new version
install compatible extensions
initialize target cluster
run pg_upgrade --check
stop applications
stop old cluster
run pg_upgrade
start new cluster
ANALYZE
validate
```

Example:

```bash
pg_upgrade --check ...
```

### Method 2 — Logical replication

Best when minimal downtime is required.

```text
old primary
   ↓ logical replication
new PostgreSQL cluster
```

Synchronize the new environment and perform controlled application cutover.

### Method 3 — Dump/restore

```bash
pg_dump
pg_restore
```

Simple but slow for large databases.

Senior candidates should discuss rollback strategy, extensions, ANALYZE, replication rebuilding, application compatibility, and rehearsals—not just `pg_upgrade`.

---

## 16. How do you achieve near-zero-downtime PostgreSQL upgrades?

**Answer:** Logical replication is a common strategy.

Architecture:

```text
PostgreSQL old version
       │
       │ logical replication
       ↓
PostgreSQL new version
```

Procedure:

```text
1. Provision new cluster.
2. Configure logical replication.
3. Perform initial table synchronization.
4. Allow replication to catch up.
5. Validate data.
6. Stop application writes.
7. Wait for replication lag = 0.
8. Synchronize sequences if required.
9. Change connection endpoint.
10. Validate.
11. Keep old environment temporarily for rollback planning.
```

Important limitations must be examined carefully because not every database object or operation is replicated automatically.

---

## 17. What are the main concerns when running PostgreSQL in containers?

**Answer:** The database process itself works well inside containers, but data must outlive containers.

Never rely on the container writable filesystem for production data.

Use persistent storage:

```text
Container
   ↓
Persistent Volume
   ↓
Durable block/storage system
```

Key concerns:

* persistent `PGDATA`
* graceful shutdown
* memory limits
* CPU throttling
* disk latency
* backup access
* anti-affinity
* health checks
* secrets
* security contexts
* failover orchestration

A container restart is not equivalent to database HA.

---

## 18. What are the important considerations for PostgreSQL on Kubernetes?

**Answer:** PostgreSQL is stateful, so Kubernetes deployment requires more than a basic Deployment object.

Typical architecture:

```text
PostgreSQL Operator
      │
      ├── Primary Pod
      ├── Replica Pod
      ├── Replica Pod
      │
      └── Persistent Volumes
```

Consider:

* Stateful storage
* PodDisruptionBudgets
* topology spread
* anti-affinity
* automatic failover
* fencing
* leader election
* backups
* WAL archive
* TLS
* secret rotation
* readiness versus liveness checks
* controlled upgrades

Operators are generally preferable to hand-built Kubernetes scripts because PostgreSQL failover involves database state, replication state, and split-brain prevention.

---

## 19. What storage characteristics are most important for PostgreSQL?

**Answer:** Database performance depends heavily on latency rather than only advertised bandwidth.

Important metrics include:

```text
read latency
write latency
fsync latency
IOPS
queue depth
throughput
storage capacity
burst limits
```

WAL writes are especially sensitive to durable-write latency.

Random-access workloads commonly make database performance sensitive to storage IOPS.

Monitor OS metrics using tools such as:

```bash
iostat -x 1
vmstat 1
pidstat -d 1
```

and PostgreSQL with:

```sql
SELECT *
FROM pg_stat_io;
```

Modern PostgreSQL exposes cluster-level I/O statistics through `pg_stat_io`. ([PostgreSQL][8])

---

## 20. Why should PostgreSQL use a connection pooler?

**Answer:** PostgreSQL has per-connection overhead, so thousands of application connections can become expensive.

A pooler such as PgBouncer can transform:

```text
5000 application connections
          ↓
       PgBouncer
          ↓
200 PostgreSQL connections
```

Benefits:

* fewer PostgreSQL processes
* lower memory consumption
* protection from connection storms
* faster connection establishment

Pool modes commonly include:

```text
session
transaction
statement
```

Transaction pooling gives high efficiency but can break session-dependent features.

A senior engineer should therefore understand application behavior before switching pool modes.

---

# Section 3 — PostgreSQL Configuration

## 21. Where can PostgreSQL configuration parameters be defined?

**Answer:** PostgreSQL settings can originate from several places, including:

```text
postgresql.conf
postgresql.auto.conf
ALTER SYSTEM
ALTER DATABASE
ALTER ROLE
session SET
command-line startup parameters
```

Inspect effective settings:

```sql
SHOW shared_buffers;

SELECT name,
       setting,
       unit,
       source,
       sourcefile
FROM pg_settings
WHERE name = 'shared_buffers';
```

This is important when troubleshooting because editing `postgresql.conf` may have no effect if a higher-priority configuration source overrides the parameter.

---

## 22. What is the difference between configuration reload and restart?

**Answer:** Some parameters can be reloaded without restarting PostgreSQL.

Example:

```bash
pg_ctl reload
```

or:

```sql
SELECT pg_reload_conf();
```

Examples of commonly reloadable settings include many logging parameters.

Others require server restart.

For example:

```text
shared_buffers
max_connections
shared_preload_libraries
```

You can check:

```sql
SELECT name,
       context,
       pending_restart
FROM pg_settings
WHERE pending_restart;
```

`pg_hba.conf` is read at startup and on configuration reload/SIGHUP. ([PostgreSQL][9])

---

## 23. How do you tune `shared_buffers`?

**Answer:** `shared_buffers` controls PostgreSQL's primary shared buffer cache.

A commonly used starting point for a dedicated database server is approximately:

```text
25% of RAM
```

but this is only a starting point.

PostgreSQL documentation specifically suggests about 25% for a dedicated server with at least 1 GB RAM and notes that values much above roughly 40% are unlikely to perform better because PostgreSQL also depends on the OS page cache. ([PostgreSQL][10])

Example:

```conf
shared_buffers = '16GB'
```

Never tune it solely by formula.

Evaluate:

* database size
* active working set
* OS cache
* concurrent queries
* checkpoint behavior
* available memory

It requires restart.

---

## 24. What is `work_mem` and why can setting it too high crash a server?

**Answer:** `work_mem` is memory available to individual query operations such as:

* sort
* hash
* aggregation
* joins

Example:

```conf
work_mem = '32MB'
```

The critical point is:

**It is not 32 MB per PostgreSQL server.**

One query can use several work-memory allocations.

Example:

```text
100 sessions
× 5 memory-intensive operations
× 32 MB

≈ 16 GB potential usage
```

and parallel workers can amplify it further.

PostgreSQL documentation explicitly warns that multiple operations and sessions can each consume `work_mem`. ([PostgreSQL][10])

For special analytical jobs, prefer:

```sql
SET LOCAL work_mem = '512MB';
```

instead of globally raising it for every connection.

---

## 25. What is `maintenance_work_mem`?

**Answer:** `maintenance_work_mem` controls memory used by maintenance operations such as:

```text
VACUUM
CREATE INDEX
ALTER TABLE ADD FOREIGN KEY
```

It can often be larger than `work_mem` because fewer maintenance operations normally execute simultaneously.

Example:

```conf
maintenance_work_mem = '1GB'
```

However, autovacuum memory must also be considered when multiple workers run simultaneously.

During a major index build, you might temporarily increase it.

---

## 26. What is `effective_cache_size`?

**Answer:** `effective_cache_size` is a **planner estimate**, not allocated memory.

It tells PostgreSQL approximately how much data could be cached across:

```text
PostgreSQL shared_buffers
+
operating-system filesystem cache
```

Example:

```conf
effective_cache_size = '48GB'
```

On a 64-GB dedicated server, that might be a reasonable initial estimate depending on workload.

Higher values can make index-based plans look more attractive to the optimizer.

It does **not** reserve 48 GB.

That distinction is a very common interview question.

---

## 27. How should `max_connections` be configured?

**Answer:** Do not simply configure:

```conf
max_connections = 5000
```

because an application wants 5,000 connections.

Every active backend consumes resources.

A better architecture is commonly:

```text
Application
   ↓
Connection Pool
   ↓
Controlled PostgreSQL backend count
```

Estimate based on:

* CPU cores
* workload
* query duration
* memory
* pool configuration
* expected concurrency

Monitor:

```sql
SELECT state,
       count(*)
FROM pg_stat_activity
GROUP BY state;
```

Too many active database sessions typically increases context switching and resource contention.

---

## 28. Which PostgreSQL parameters are most important for WAL tuning?

**Answer:** Key parameters include:

```text
wal_level
max_wal_size
min_wal_size
wal_buffers
checkpoint_timeout
checkpoint_completion_target
archive_mode
archive_command/archive_library
wal_compression
max_wal_senders
```

For replication, also consider:

```text
wal_keep_size
max_replication_slots
max_slot_wal_keep_size
```

Increasing `max_wal_size` can reduce checkpoint frequency for write-intensive workloads, although capacity and recovery trade-offs must be considered. PostgreSQL's WAL configuration documentation describes the trade-off between checkpoints, I/O load, WAL generation, and recovery time. ([PostgreSQL][6])

---

## 29. How would you configure PostgreSQL logging for production?

**Answer:** Useful settings commonly include:

```conf
logging_collector = on
log_min_duration_statement = '500ms'
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = '1s'
```

Exact values depend on workload.

For production systems, logs should make it possible to answer:

```text
Who connected?
What failed?
What query was slow?
What transaction caused lock contention?
Did autovacuum struggle?
Were checkpoints abnormal?
Did archiving fail?
```

Avoid logging every statement on high-throughput production environments unless needed temporarily, because volume and sensitive-data exposure can become significant.

---

## 30. Explain `pg_hba.conf`.

**Answer:** `pg_hba.conf` controls client authentication.

Typical structure:

```text
TYPE  DATABASE  USER  ADDRESS          METHOD
```

Example:

```conf
hostssl appdb app_user 10.10.0.0/16 scram-sha-256
```

Meaning:

```text
TCP + TLS connection
database = appdb
role = app_user
source network = 10.10.0.0/16
authentication = SCRAM-SHA-256
```

Rules are evaluated in order.

Therefore:

```text
specific rules first
broad rules later
```

is generally easier to manage.

After changes:

```sql
SELECT pg_reload_conf();
```

`pg_hba.conf` is PostgreSQL's standard host-based authentication configuration. ([PostgreSQL][9])

---

# Section 4 — Security

## 31. How do you secure PostgreSQL network connections?

**Answer:** Use TLS.

Server configuration:

```conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

Then enforce encrypted connections through:

```conf
hostssl
```

in `pg_hba.conf`.

Client connections should validate the server certificate where feasible:

```text
sslmode=verify-full
```

PostgreSQL natively supports TLS-encrypted client/server connections when built appropriately and `ssl` is enabled. ([PostgreSQL][11])

Avoid treating `sslmode=require` and full identity verification as identical security guarantees.

---

## 32. What authentication mechanism would you prefer?

**Answer:** For password authentication, modern deployments should generally use:

```text
SCRAM-SHA-256
```

rather than older weak password mechanisms.

For enterprise environments, authentication might also involve:

* certificates
* LDAP
* GSSAPI/Kerberos
* OAuth in newer PostgreSQL environments
* external proxy/IAM mechanisms in managed platforms

PostgreSQL 18 includes OAuth authentication support. ([PostgreSQL][12])

Authentication design should be coupled with:

* TLS
* secret rotation
* least privilege
* network policies

---

## 33. Explain PostgreSQL role management and least privilege.

**Answer:** Applications should not normally connect as a superuser.

Example:

```sql
CREATE ROLE app_user LOGIN PASSWORD '...';

GRANT CONNECT ON DATABASE appdb TO app_user;
GRANT USAGE ON SCHEMA app TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO app_user;
```

Separate roles can be designed as:

```text
application runtime
schema migration
read-only reporting
backup
monitoring
replication
administration
```

Avoid giving:

```text
SUPERUSER
CREATEDB
CREATEROLE
```

unless actually required.

---

## 34. How should PostgreSQL credentials be managed in DevOps environments?

**Answer:** Do not hardcode credentials in:

```text
Dockerfile
Git repository
Terraform source
Helm values committed to Git
shell scripts
```

Use a secrets system such as:

```text
cloud secret manager
Vault-like secrets platform
Kubernetes Secrets with appropriate encryption controls
CI/CD secret store
```

Rotation workflow:

```text
generate new credential
→ update secret manager
→ update applications/pools
→ verify connectivity
→ revoke old credential
```

For highly available systems, plan connection-pool credential refresh during rotation.

---

## 35. How can database access be audited?

**Answer:** Audit layers can include:

* PostgreSQL connection logs
* DDL logging
* role/authentication logs
* database audit extensions where appropriate
* cloud audit logs
* operating-system security logs
* centralized SIEM

Useful settings can include:

```text
log_connections
log_disconnections
log_statement = 'ddl'
```

Audit strategy must balance:

```text
security
performance
storage
privacy
compliance
```

Database logs should be centrally shipped rather than stored only on the database host.

---

# Section 5 — Replication and High Availability

## 36. How does PostgreSQL streaming replication work?

**Answer:** A primary generates WAL.

A standby receives that WAL through a replication connection and replays it.

```text
Primary
   |
   | WAL streaming
   ↓
Standby
```

Important processes include:

```text
Primary: WAL sender
Standby: WAL receiver
```

Useful primary configuration may include:

```conf
wal_level = replica
max_wal_senders = 10
```

A standby typically needs `primary_conninfo`.

PostgreSQL supports built-in streaming replication and physical hot/warm standby operation. ([PostgreSQL][13])

---

## 37. What is the difference between synchronous and asynchronous replication?

**Answer:**

### Asynchronous

```text
Client → Primary → COMMIT success
                    ↓
                 Standby later
```

Advantages:

* lower latency
* standby/network failure does not normally stop commits

Risk:

* recent committed transactions can potentially be lost during primary failure.

### Synchronous

```text
Client
  ↓
Primary
  ↓
Standby confirmation
  ↓
COMMIT returns
```

Advantages:

* stronger durability guarantees

Trade-offs:

* increased latency
* standby problems can affect commit availability depending on configuration

PostgreSQL streaming replication is asynchronous by default. Synchronous replication can make commit wait for selected standby confirmation. ([PostgreSQL][14])

---

## 38. What are replication slots?

**Answer:** Replication slots tell the primary that certain WAL—or logical decoding information—must remain available for a consumer.

Benefits:

```text
replica disconnects
     ↓
primary keeps required WAL
     ↓
replica reconnects
```

Physical slot example:

```sql
SELECT *
FROM pg_create_physical_replication_slot('standby1');
```

Monitor:

```sql
SELECT *
FROM pg_replication_slots;
```

Major risk:

**An inactive slot can cause WAL to accumulate until storage fills.**

PostgreSQL explicitly warns that replication slots can retain enough WAL to fill `pg_wal`; `max_slot_wal_keep_size` can place a limit on retention. ([PostgreSQL][14])

This is one of the highest-value PostgreSQL DevOps interview topics.

---

## 39. How do you monitor replication lag?

**Answer:** On the primary:

```sql
SELECT application_name,
       client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

On standby:

```sql
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn();
```

Potential causes of lag:

```text
slow network
high WAL generation
slow standby storage
CPU saturation
long standby query
recovery conflict
WAL receiver issue
checkpoint pressure
```

Do not measure replication health only in seconds. Also monitor:

```text
lag bytes
network throughput
replay rate
WAL generation rate
```

---

## 40. What happens during PostgreSQL failover?

**Answer:** A standby is promoted and becomes writable.

Conceptually:

```text
Primary X
     |
     X failure

Standby
   ↓ promote
New Primary
```

Promotion can be triggered using tools such as:

```bash
pg_ctl promote
```

or equivalent mechanisms.

But production failover requires much more:

```text
failure detection
quorum/consensus
fencing
promotion
DNS/proxy/service update
application reconnect
old-primary isolation
replica reconfiguration
```

The hardest problem is often not promotion but **preventing two primaries**.

---

## 41. What is split brain and how do you prevent it?

**Answer:** Split brain happens when two nodes both believe they are primary.

Example:

```text
Primary A ← clients
Primary B ← clients
```

Both accept writes and data diverges.

Prevention requires mechanisms such as:

* fencing
* STONITH
* distributed consensus
* quorum
* reliable leader election
* preventing failed primary from rejoining as writable

A proper HA tool must coordinate more than simply executing:

```bash
pg_ctl promote
```

This is why production systems often employ HA orchestration rather than custom shell scripts.

---

## 42. What is `hot_standby`?

**Answer:** A hot standby can accept read-only connections while it is replaying WAL.

Architecture:

```text
Primary → writes
   |
   ├── Replica → reporting queries
   └── Replica → read traffic
```

Potential issue:

A long-running query on the standby may need row versions that WAL replay wants to remove.

Possible outcomes include a **recovery conflict** and cancellation of the standby query.

Related parameters include:

```text
max_standby_streaming_delay
hot_standby_feedback
```

`hot_standby_feedback` can reduce certain recovery conflicts but may increase bloat on the primary.

---

## 43. What is cascading replication?

**Answer:** Instead of every standby streaming from the primary:

```text
Primary
 ├── Standby1
 ├── Standby2
 └── Standby3
```

you can use:

```text
Primary
   ↓
Standby1
   ├── Standby2
   └── Standby3
```

Benefits:

* reduced bandwidth pressure on primary
* useful across regions

Trade-off:

* downstream replicas can inherit additional lag
* failure topology becomes more complicated

PostgreSQL supports cascading replication; current documentation notes cascading replication remains asynchronous. ([PostgreSQL][14])

---

## 44. What is logical replication?

**Answer:** Logical replication transfers row-level logical changes rather than reproducing the entire physical database cluster byte-for-byte.

Architecture:

```text
Publisher
    ↓
Publication
    ↓
Subscription
    ↓
Subscriber
```

Useful for:

* selective table replication
* migrations
* version upgrades
* integration
* partial data replication

Physical replication is usually preferred for identical HA standby replicas.

Logical replication is often preferred where:

```text
source and destination differ
selected tables are needed
minimal-downtime migration is required
```

---

## 45. How would you design a highly available PostgreSQL architecture?

**Answer:** Example:

```text
             Applications
                  |
             DB Proxy/VIP
                  |
          +-------+-------+
          |               |
       Primary         Standby-A
          |
          +----------> Standby-B

             |
          WAL archive
             |
        Object Storage

     + Physical Backups
```

Requirements:

* replicas across failure domains
* automated failure detection
* safe leader election
* fencing
* backups independent of replicas
* PITR
* monitoring
* tested disaster recovery

**Important interview answer:**

> Replication is not a backup.

Deleting a table on the primary generally replicates the deletion.

---

# Section 6 — Backup, Restore and Disaster Recovery

## 46. What PostgreSQL backup methods are available?

**Answer:** PostgreSQL documentation groups backup strategies into three broad approaches:

1. SQL dump
2. filesystem-level backup
3. continuous archiving/PITR ([PostgreSQL][15])

Operationally you may use:

```text
pg_dump
pg_dumpall
pg_basebackup
physical backup software
WAL archiving
storage snapshots designed correctly for PostgreSQL
```

Production environments often combine physical backups and continuous WAL archive.

---

## 47. What is the difference between `pg_dump` and physical backup?

**Answer:**

### pg_dump

Logical backup.

```bash
pg_dump -Fc appdb > appdb.dump
```

Advantages:

* object-level restore
* portable
* useful for migrations

Disadvantages:

* slower for very large databases
* slower recovery

### Physical backup

Copies database storage structure.

Advantages:

* faster large-system recovery
* supports PITR when combined with WAL

Disadvantages:

* less granular
* version/platform constraints matter

`pg_dump` provides consistent exports while the database remains in use but PostgreSQL documentation cautions that it is not generally the ideal regular production-backup mechanism except in suitable cases. ([PostgreSQL][2])

---

## 48. What does `pg_basebackup` do?

**Answer:** `pg_basebackup` creates a physical base backup from a running PostgreSQL cluster.

Example:

```bash
pg_basebackup \
  -h primary-db \
  -U replicator \
  -D /backup/base \
  -Fp \
  -Xs \
  -P
```

It is commonly used for:

* physical backups
* initial standby creation
* PITR foundations

Modern PostgreSQL also supports incremental base-backup workflows, with incremental data later combined using `pg_combinebackup`. ([PostgreSQL][16])

---

## 49. Explain Point-in-Time Recovery.

**Answer:** PITR allows restoration to a specific time or transaction position.

Requirements:

```text
base backup
+
continuous WAL archive
```

Example scenario:

```text
09:00 base backup
12:00 user drops production table
```

Restore:

```text
base backup
   +
WAL 09:00 → 11:59:59
```

Result:

Database state just before the destructive event.

Targets can include:

```text
timestamp
LSN
transaction ID
restore point
```

PITR is essential for recovering from logical errors that replicas faithfully reproduce.

---

## 50. What are RPO and RTO?

**Answer:**

### RPO — Recovery Point Objective

How much data can the business afford to lose?

Example:

```text
RPO = 5 minutes
```

means losing more than approximately five minutes of committed data would violate the objective.

### RTO — Recovery Time Objective

How long can the service remain unavailable?

Example:

```text
RTO = 15 minutes
```

Architecture should derive from these targets.

Example:

```text
RPO 24 hours
→ nightly backup might theoretically satisfy data-loss target

RPO near zero
→ synchronous/multi-site durability architecture may be required
```

Do not choose database technology first and invent RPO/RTO afterward.

---

## 51. Why is WAL archiving important?

**Answer:** WAL archiving continuously saves completed WAL segments to durable external storage.

Typical conceptual configuration:

```conf
archive_mode = on
archive_command = '...'
```

Architecture:

```text
PostgreSQL
    |
    ↓
pg_wal
    |
    ↓
WAL archive
    |
Object Storage
```

It enables:

* PITR
* recovery beyond a particular base backup
* disaster recovery

Monitor:

```sql
SELECT *
FROM pg_stat_archiver;
```

Repeated archive failures can eventually lead to excessive WAL accumulation.

---

## 52. How do you verify that PostgreSQL backups actually work?

**Answer:** A backup is not trustworthy merely because the backup command returned zero.

Perform:

```text
1. restore backup into isolated environment
2. replay WAL
3. start PostgreSQL
4. verify expected databases
5. run consistency/application checks
6. measure recovery duration
7. record actual RTO
```

Automate restore drills.

Example validation:

```sql
SELECT count(*) FROM critical_table;
```

plus application-level checks.

The most important principle:

**Backup success ≠ recovery success.**

---

## 53. What causes PostgreSQL backup failures in production?

**Answer:** Common causes include:

```text
expired credentials
object-storage permission change
network failure
disk full
archive_command failure
retention mistake
corrupt backup
missing WAL
encryption-key loss
wrong PostgreSQL version
extension incompatibility
```

Monitor:

* backup completion
* backup age
* WAL archive age
* archive failures
* restore verification age

An alert saying only "backup process is running" is insufficient.

---

## 54. How would you design multi-region disaster recovery?

**Answer:** Example:

```text
Region A
Primary
   |
   ↓
Standby

   ║ network
   ↓

Region B
DR Standby
   |
   ↓
Independent backups/WAL archive
```

Consider:

* replication latency
* RPO
* network partition
* failover authority
* DNS TTL
* application dependencies
* fencing
* backup-region independence
* secret availability
* restore procedure

For geographically distant regions, fully synchronous replication may impose unacceptable application latency. PostgreSQL documentation explicitly notes the performance trade-off of synchronous replication over slower networks. ([PostgreSQL][13])

---

## 55. Why is replication not a backup?

**Answer:** Suppose someone runs:

```sql
DROP TABLE customers;
```

Streaming replication faithfully transmits the WAL changes.

Now:

```text
Primary: customers deleted
Replica: customers deleted
```

Likewise:

```text
bad UPDATE
malicious DELETE
schema mistake
application bug
```

may propagate immediately.

Backup/PITR protects against historical and logical failures; replication primarily protects availability.

A production architecture usually needs **both**.

---

# Section 7 — Performance Tuning

## 56. How do you troubleshoot a slow PostgreSQL query?

**Answer:** Start methodically.

```text
1. Find the slow query.
2. Measure frequency and total impact.
3. EXPLAIN it.
4. Check actual execution.
5. Check indexes.
6. Check row-estimation accuracy.
7. Inspect cache/disk usage.
8. Check locks.
9. Check temp-file usage.
10. Check host CPU/I/O.
```

Useful:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Never run an expensive `EXPLAIN ANALYZE` blindly against a dangerous production mutation because it actually executes the statement.

PostgreSQL's planner chooses execution plans based on estimated costs and statistics, and `EXPLAIN` is the main tool for understanding those decisions. ([PostgreSQL][1])

---

## 57. What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?

**Answer:**

```sql
EXPLAIN
SELECT ...
```

shows estimated plan information.

```sql
EXPLAIN ANALYZE
SELECT ...
```

actually executes the query and shows real execution timing and rows.

Better:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Compare:

```text
estimated rows
vs
actual rows
```

Large differences often indicate:

* stale statistics
* skewed distributions
* correlated columns
* inadequate statistics target

---

## 58. Why might PostgreSQL choose a sequential scan instead of an index scan?

**Answer:** A sequential scan is not automatically bad.

PostgreSQL may prefer it when:

* large portion of table is needed
* table is small
* index selectivity is poor
* random I/O looks expensive
* statistics suggest sequential scan is cheaper

Example:

```sql
SELECT *
FROM employees
WHERE active = true;
```

If 95% of employees are active, an index on `active` may not help.

The correct question is not:

> Why isn't PostgreSQL using my index?

It is:

> Is the planner's chosen plan actually cheaper for this workload?

---

## 59. What index types does PostgreSQL provide?

**Answer:** Important types include:

### B-tree

Default.

Good for:

```text
=
<
>
BETWEEN
ORDER BY
```

### Hash

Equality operations.

### GIN

Useful for:

```text
arrays
JSONB
full-text search
```

### GiST

Useful for various specialized data and geometric/search operations.

### BRIN

Very small indexes useful for physically correlated, very large tables.

Example use case:

```text
multi-terabyte append-only table
ordered approximately by timestamp
```

Choosing index type should be based on operators and data characteristics, not just table size.

---

## 60. Why does column order matter in a multi-column B-tree index?

**Answer:** Consider:

```sql
CREATE INDEX idx_orders
ON orders(customer_id, created_at);
```

It is particularly useful for:

```sql
WHERE customer_id = 100
AND created_at > now() - interval '7 days';
```

It can also help queries filtering just:

```sql
customer_id
```

but is generally much less useful for queries only on:

```sql
created_at
```

The common guideline is to design indexes around actual query predicates and ordering requirements.

Do not create indexes solely from individual columns observed in `WHERE` clauses.

---

## 61. What is a partial index?

**Answer:** A partial index stores only rows satisfying a predicate.

Example:

```sql
CREATE INDEX idx_unprocessed_orders
ON orders(created_at)
WHERE status = 'PENDING';
```

If only 1% of orders are pending, this index can be much smaller than indexing every row.

Advantages:

* smaller storage
* faster index maintenance
* better cache efficiency

Use when queries repeatedly target a predictable subset.

---

## 62. What is an expression index?

**Answer:** An expression index indexes a computed expression.

Example:

```sql
CREATE INDEX idx_users_lower_email
ON users(lower(email));
```

Useful query:

```sql
SELECT *
FROM users
WHERE lower(email) = 'user@example.com';
```

Without the expression index, a normal index on `email` may not satisfy this predicate efficiently.

---

## 63. What is a covering index and `INCLUDE`?

**Answer:** PostgreSQL allows extra columns to be stored in an index without making them part of the search key.

Example:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id)
INCLUDE (status, total);
```

Query:

```sql
SELECT status, total
FROM orders
WHERE customer_id = 100;
```

may become eligible for an index-only scan if visibility conditions permit.

Trade-off:

* larger index
* greater write cost

Do not use `INCLUDE` indiscriminately.

---

## 64. What causes table and index bloat?

**Answer:** Common causes include:

* frequent UPDATE
* DELETE
* long transactions
* ineffective autovacuum
* low vacuum frequency
* poor HOT update rate

MVCC leaves obsolete tuple versions until vacuum can reclaim them.

Monitor:

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Standard VACUUM usually makes space reusable inside the relation.

`VACUUM FULL` can physically shrink it but takes an `ACCESS EXCLUSIVE` lock and should not be treated as normal maintenance. ([PostgreSQL][4])

---

## 65. How would you tune autovacuum for a high-write table?

**Answer:** Default thresholds may react too slowly on very large and highly active tables.

You can use per-table configuration:

```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

Potential global tuning:

```text
autovacuum_max_workers
autovacuum_naptime
autovacuum_vacuum_cost_limit
autovacuum_vacuum_cost_delay
```

Monitor:

```sql
SELECT relname,
       n_dead_tup,
       last_autovacuum,
       autovacuum_count
FROM pg_stat_user_tables;
```

PostgreSQL recommends autovacuum and enables it by default; disabling it completely is generally unwise. ([PostgreSQL][4])

---

## 66. What is the difference between VACUUM and VACUUM FULL?

**Answer:**

### VACUUM

```sql
VACUUM orders;
```

* marks dead-tuple space reusable
* normally does not return most space to OS
* permits normal concurrent DML

### VACUUM FULL

```sql
VACUUM FULL orders;
```

* rewrites the relation
* can return disk space to OS
* needs additional temporary disk space
* requires strong/exclusive table locking
* can produce major downtime

PostgreSQL recommends avoiding `VACUUM FULL` for routine maintenance. ([PostgreSQL][4])

---

## 67. What is ANALYZE?

**Answer:** `ANALYZE` collects table statistics for the query planner.

Example:

```sql
ANALYZE orders;
```

Statistics include information about:

```text
value distribution
null fraction
common values
distinct-value estimates
histograms
```

Bad statistics can produce incorrect row estimates and poor execution plans.

After large bulk loads:

```sql
ANALYZE;
```

is often important.

`VACUUM ANALYZE` combines maintenance and planner statistics collection.

---

## 68. How do you tune planner statistics for skewed data?

**Answer:** Increase the statistics target for important columns.

Example:

```sql
ALTER TABLE orders
ALTER COLUMN customer_id
SET STATISTICS 1000;

ANALYZE orders;
```

For correlated columns, consider extended statistics.

Example:

```sql
CREATE STATISTICS orders_stats
ON country, state
FROM orders;

ANALYZE orders;
```

This can help the optimizer understand relationships that single-column statistics miss.

---

## 69. What causes PostgreSQL queries to spill to disk?

**Answer:** Operations such as:

```text
SORT
HASH
aggregation
```

can exceed available memory.

Then PostgreSQL writes temporary files.

Monitor logging:

```conf
log_temp_files = 0
```

and examine execution:

```sql
EXPLAIN (ANALYZE, BUFFERS)
...
```

Possible fixes:

* improve query
* reduce rows earlier
* add index
* selectively increase `work_mem`

Do not globally increase `work_mem` to several GB merely because one analytical query spills.

---

## 70. How would you investigate high CPU on PostgreSQL?

**Answer:** First correlate OS processes with SQL.

OS:

```bash
top
pidstat -u 1
```

Database:

```sql
SELECT pid,
       usename,
       state,
       query_start,
       query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

Then use `pg_stat_statements` to find CPU-heavy query candidates.

Investigate:

```text
execution plans
missing indexes
bad statistics
excessive parallelism
high query frequency
JSON processing
sort/hash operations
connection storm
autovacuum activity
```

Do not immediately add CPU before understanding whether a regression introduced a pathological query.

---

## 71. How do you investigate high disk I/O?

**Answer:** OS:

```bash
iostat -xz 1
```

Look at:

```text
await
utilization
queue depth
read/write IOPS
throughput
```

PostgreSQL:

```sql
SELECT *
FROM pg_stat_io;
```

Also inspect:

```text
pg_stat_database
pg_stat_statements
pg_stat_bgwriter
pg_stat_checkpointer
pg_stat_wal
```

Common causes:

* large sequential scans
* checkpoint storms
* autovacuum
* low cache efficiency
* bulk writes
* sort spill
* index creation
* backup workload

`pg_stat_io` is designed to expose cluster-wide I/O statistics by backend/object/context. ([PostgreSQL][3])

---

## 72. What is cache hit ratio and should it always be 99%?

**Answer:** A common query is:

```sql
SELECT
  sum(blks_hit) /
  NULLIF(sum(blks_hit) + sum(blks_read), 0)::numeric
FROM pg_stat_database;
```

A high cache hit ratio is generally desirable for OLTP workloads.

But the number should not be treated as a universal SLA.

Examples:

```text
data warehouse scan → lower ratio may be normal
tiny OLTP working set → very high ratio expected
```

Evaluate latency, workload type, physical reads, and execution plans—not one percentage alone.

---

## 73. How do checkpoints affect performance?

**Answer:** A checkpoint writes dirty pages toward durable storage.

If checkpoints occur too aggressively, they can cause:

```text
write bursts
latency spikes
high WAL traffic
storage saturation
```

Investigate:

```sql
SELECT *
FROM pg_stat_checkpointer;
```

Configuration:

```text
checkpoint_timeout
max_wal_size
checkpoint_completion_target
```

PostgreSQL intentionally spreads checkpoint writes to reduce I/O bursts; overly frequent checkpoints are generally undesirable. ([PostgreSQL][6])

---

## 74. How does partitioning improve performance?

**Answer:** Partitioning divides a logical table into physically separate relations.

Example:

```text
orders
├── orders_2026_07
├── orders_2026_08
└── orders_2026_09
```

Benefits can include:

* partition pruning
* easier lifecycle management
* smaller per-partition indexes
* faster removal of historical data

Instead of:

```sql
DELETE FROM orders
WHERE created_at < ...;
```

you may be able to drop an obsolete partition.

But partitioning is not automatically faster.

Too many partitions can increase:

```text
planning complexity
catalog overhead
operational complexity
```

---

## 75. How would you optimize a large bulk load?

**Answer:** Prefer `COPY` over millions of individual inserts when practical.

Example:

```sql
COPY orders
FROM '/data/orders.csv'
WITH (FORMAT csv, HEADER true);
```

For controlled initial-load scenarios:

```text
increase maintenance_work_mem
increase max_wal_size
load data
build indexes appropriately
ANALYZE afterward
```

PostgreSQL's performance documentation specifically recommends `COPY` and notes tuning considerations such as `maintenance_work_mem`, `max_wal_size`, and running `ANALYZE` after population. ([PostgreSQL][17])

---

# Section 8 — Troubleshooting Production Problems

## 76. PostgreSQL is slow. What is your troubleshooting workflow?

**Answer:** Avoid random parameter changes.

Use a layered investigation.

### Layer 1 — Application

```text
What changed?
Which endpoint is slow?
When did it start?
```

### Layer 2 — Sessions

```sql
SELECT *
FROM pg_stat_activity;
```

Check:

```text
active queries
blocking
idle transactions
connection count
```

### Layer 3 — Query statistics

```text
pg_stat_statements
```

### Layer 4 — Database internals

```text
locks
autovacuum
WAL
checkpoints
replication
temp files
```

### Layer 5 — OS

```text
CPU
memory
swap
disk latency
network
filesystem capacity
```

### Layer 6 — Infrastructure

```text
storage throttling
VM noisy neighbor
cloud limits
network changes
```

This systematic approach is much stronger than immediately changing `shared_buffers`.

---

## 77. How do you identify blocking queries?

**Answer:**

```sql
SELECT
    pid,
    usename,
    query,
    pg_blocking_pids(pid) AS blocking_pids
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Then investigate blockers.

Do not immediately run:

```sql
SELECT pg_terminate_backend(...);
```

Determine:

* transaction purpose
* impact of rollback
* root blocker
* application owner
* how long it has been running

Termination can cause a large transaction rollback that itself takes time and resources.

---

## 78. How do you troubleshoot `idle in transaction` sessions?

**Answer:** Query:

```sql
SELECT pid,
       usename,
       xact_start,
       state,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

These sessions are dangerous because they may:

* hold locks
* retain old snapshots
* prevent vacuum cleanup
* contribute to bloat
* delay XID cleanup

Configure protection where appropriate:

```conf
idle_in_transaction_session_timeout = '5min'
```

The application should generally keep transactions short and commit/rollback promptly.

---

## 79. A DDL deployment is hanging. What would you check?

**Answer:** Example:

```sql
ALTER TABLE orders ADD COLUMN ...
```

may wait for a lock.

Check:

```sql
SELECT pid,
       wait_event_type,
       wait_event,
       pg_blocking_pids(pid),
       query
FROM pg_stat_activity
WHERE state <> 'idle';
```

Common root cause:

```text
long transaction
   ↓
holds conflicting lock
   ↓
migration waits
   ↓
application connections accumulate
```

Production migrations should use safeguards such as:

```sql
SET lock_timeout = '5s';
SET statement_timeout = '10min';
```

where appropriate.

It is usually better for a migration to fail cleanly than wait indefinitely and trigger an outage.

---

## 80. What would you do if the PostgreSQL filesystem is full?

**Answer:** First identify which filesystem:

```bash
df -h
du -sh $PGDATA/*
```

Potential causes:

```text
pg_wal growth
logs
table growth
temporary files
backup files
replication slots
failed archiving
```

Do **not** manually delete files inside:

```text
$PGDATA/pg_wal
```

because you can destroy recoverability or make the database unusable.

Find the cause before removing anything.

Then:

* expand storage if necessary
* fix WAL/archive problem
* remove safe external files
* manage logs
* clean obsolete slots safely
* verify database health

---

## 81. `pg_wal` is consuming all disk space. What could cause it?

**Answer:** Common causes:

1. inactive replication slot
2. failed `archive_command`
3. lagging replica
4. excessive WAL generation
5. backup-related retention

Check:

```sql
SELECT slot_name,
       active,
       restart_lsn,
       wal_status
FROM pg_replication_slots;
```

Check archive state:

```sql
SELECT *
FROM pg_stat_archiver;
```

Calculate slot retention:

```sql
SELECT slot_name,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       )
FROM pg_replication_slots;
```

This is one of the most common serious PostgreSQL operational incidents.

---

## 82. What is transaction ID wraparound?

**Answer:** PostgreSQL transaction IDs are finite and wrap around.

Old tuples must eventually be frozen so their visibility remains safe across XID wraparound.

Autovacuum performs anti-wraparound vacuum operations.

Monitor database age:

```sql
SELECT datname,
       age(datfrozenxid)
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

If wraparound risk becomes critical, PostgreSQL can become increasingly aggressive about vacuuming and ultimately protect itself from unsafe operation.

This is why autovacuum must not casually be disabled.

PostgreSQL documents prevention of transaction-ID wraparound as one of VACUUM's fundamental responsibilities. ([PostgreSQL][4])

---

## 83. How do long-running transactions affect PostgreSQL?

**Answer:** A transaction holding an old snapshot can prevent obsolete row versions from being cleaned.

Consequences:

```text
dead tuple accumulation
table bloat
index bloat
larger storage
slower scans
XID pressure
```

Find them:

```sql
SELECT pid,
       now() - xact_start AS transaction_age,
       state,
       query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Application design should avoid transactions spanning:

```text
user interaction
external API calls
long batch pauses
```

unless deliberately required.

---

## 84. A replica is suddenly hours behind. How do you troubleshoot?

**Answer:** Check the pipeline:

```text
Primary WAL generation
        ↓
Network transfer
        ↓
Standby receive
        ↓
Disk write
        ↓
WAL replay
```

Primary:

```sql
SELECT *
FROM pg_stat_replication;
```

Standby:

```sql
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp();
```

Check:

* network errors
* CPU
* disk latency
* long reporting queries
* recovery conflicts
* WAL generation spike

Compare `receive_lsn` and `replay_lsn`.

If receive is current but replay is behind, the bottleneck is likely on standby replay rather than network transmission.

---

## 85. A standby will not start. What do you examine?

**Answer:** Start with PostgreSQL logs.

Check:

```text
primary_conninfo
standby.signal
timeline history
WAL availability
replication permissions
TLS
DNS/networking
version compatibility
data-directory ownership
```

Common fatal case:

```text
requested WAL segment has already been removed
```

If the necessary WAL no longer exists in:

```text
primary pg_wal
or
WAL archive
```

the standby may need to be rebuilt from a new base backup.

Replication slots or correctly sized WAL retention can reduce that risk.

---

## 86. PostgreSQL is using 100% memory and gets OOM-killed. What do you investigate?

**Answer:** Examine:

```text
work_mem
max_connections
parallel workers
maintenance jobs
autovacuum workers
shared_buffers
OS memory
container memory limit
huge pages
other processes
```

Remember:

```text
work_mem × operations × sessions × workers
```

can be huge.

Check the kernel:

```bash
dmesg | grep -i oom
```

and container events if applicable.

A typical mistake:

```conf
work_mem = '1GB'
max_connections = 500
```

which can create enormous theoretical memory demand.

---

## 87. PostgreSQL reports "too many connections." How do you respond?

**Answer:** First determine why.

```sql
SELECT usename,
       application_name,
       state,
       count(*)
FROM pg_stat_activity
GROUP BY usename, application_name, state
ORDER BY count(*) DESC;
```

Possible causes:

```text
application connection leak
pooler failure
traffic spike
health checks
stuck application requests
incorrect pool sizing
```

Do not simply increase `max_connections`.

Immediate remediation may include:

* restore pooler
* terminate clearly abandoned sessions
* scale application carefully

Long-term:

```text
pool connections
fix leaks
set sane pool limits
protect reserved admin connectivity
```

---

## 88. Authentication suddenly fails after a deployment. What would you check?

**Answer:** Investigate:

1. `pg_hba.conf`
2. rule ordering
3. source client IP
4. database/user names
5. password/secret rotation
6. authentication method
7. TLS requirements
8. certificate validity
9. DNS
10. connection string

Check server logs for messages such as:

```text
no pg_hba.conf entry
password authentication failed
certificate verify failed
```

Use:

```sql
SELECT pg_reload_conf();
```

after reloadable authentication configuration changes.

---

## 89. What does "could not serialize access" mean?

**Answer:** This can be a normal consequence of stricter transaction isolation rather than PostgreSQL corruption.

Example:

```text
Transaction A
and
Transaction B

produce a serialization conflict.
```

PostgreSQL aborts one transaction to preserve serializable semantics.

Application response:

```text
rollback
retry transaction
```

Investigate if frequency becomes unusually high, because application access patterns may need optimization.

---

## 90. PostgreSQL crashed and repeatedly fails to restart. What is your incident process?

**Answer:** Avoid destructive improvisation.

Check:

```bash
journalctl -u postgresql
```

and PostgreSQL logs.

Verify:

```text
disk capacity
inode capacity
memory/OOM
file permissions
configuration syntax
port conflicts
shared memory
WAL state
storage availability
certificates
extension libraries
```

Check configuration where supported:

```bash
postgres -C data_directory
```

Look for the **first meaningful error**, not merely the final shutdown message.

Never delete `postmaster.pid` automatically without first proving no PostgreSQL server is actually using that data directory.

---

# Section 9 — Monitoring and Observability

## 91. Which PostgreSQL system views should a Senior DevOps Engineer know?

**Answer:** At minimum:

```text
pg_stat_activity
pg_stat_replication
pg_stat_replication_slots
pg_stat_wal_receiver
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_archiver
pg_stat_bgwriter
pg_stat_checkpointer
pg_stat_wal
pg_stat_io
pg_locks
pg_stat_progress_vacuum
pg_stat_progress_create_index
```

The PostgreSQL monitoring system provides dedicated views for sessions, replication, WAL receivers, slots, I/O, checkpointer activity and much more. ([PostgreSQL][8])

---

## 92. What is `pg_stat_statements`?

**Answer:** `pg_stat_statements` records aggregated planning and execution statistics for normalized SQL statements.

Enable:

```conf
shared_preload_libraries = 'pg_stat_statements'
```

Restart, then:

```sql
CREATE EXTENSION pg_stat_statements;
```

Find expensive statements:

```sql
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

It can answer two different questions:

```text
Which query is slowest per execution?
Which query consumes the most total database time?
```

Those are not necessarily the same query.

The extension needs shared-memory preload and tracks planning/execution statistics across statements. ([PostgreSQL][18])

---

## 93. What PostgreSQL metrics would you alert on?

**Answer:** Important categories include:

### Availability

```text
database unreachable
primary unavailable
```

### Connections

```text
connections/max_connections
connection surge
```

### Replication

```text
replica disconnected
lag bytes
replay delay
```

### Storage

```text
disk %
disk latency
pg_wal growth
```

### WAL/archive

```text
archive failures
archive lag
slot WAL retention
```

### Transactions

```text
long transactions
idle in transaction
XID age
```

### Maintenance

```text
dead tuples
autovacuum failures
```

### Performance

```text
query latency
lock waits
temp files
CPU
I/O
```

Thresholds should reflect workload baselines rather than universal numbers.

---

## 94. How do you monitor slow queries in PostgreSQL?

**Answer:** Combine multiple approaches.

### Server logging

```conf
log_min_duration_statement = '500ms'
```

### Aggregated workload

```text
pg_stat_statements
```

### Active investigation

```sql
SELECT pid,
       now() - query_start AS duration,
       query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

### Distributed/application tracing

Attach:

```text
application_name
trace ID
request ID
```

where operationally appropriate.

This helps correlate:

```text
HTTP request
→ application
→ PostgreSQL session/query
```

---

## 95. What would your PostgreSQL Grafana dashboard contain?

**Answer:** A strong overview should contain:

```text
Transactions/sec
Connections
Active sessions
Query latency
Cache activity
Locks/waits
Dead tuples
Autovacuum
Database growth
WAL generation
Checkpoint rate/time
Archive status
Replication lag
Disk latency/IOPS
CPU
RAM
Swap
Network
Filesystem %
```

Separate dashboards are often useful for:

```text
cluster overview
query performance
replication
storage
autovacuum
backup/recovery
```

Avoid dashboards containing hundreds of graphs without actionable thresholds.

---

# Section 10 — OS Tuning, Automation and Senior-Level Scenarios

## 96. What Linux tuning matters for PostgreSQL?

**Answer:** Important areas include:

```text
filesystem/storage scheduler
swappiness
huge pages
Transparent Huge Pages behavior
file descriptor limits
process limits
memory availability
disk queue/latency
kernel writeback behavior
network limits
time synchronization
```

For example, PostgreSQL can use explicit huge pages for its main shared-memory area, configured through `huge_pages`. ([PostgreSQL][10])

Senior answer:

Do not copy a generic `/etc/sysctl.conf` from a blog.

Tune based on:

```text
PostgreSQL version
Linux kernel
memory
storage type
cloud platform
workload measurements
```

---

## 97. How would you manage PostgreSQL configuration using Infrastructure as Code?

**Answer:** Treat database configuration as reviewed code.

Example workflow:

```text
Git
 ↓
Pull Request
 ↓
CI validation
 ↓
Ansible/Terraform/Operator
 ↓
Staging PostgreSQL
 ↓
Production rollout
```

Store parameters such as:

```text
shared_buffers
WAL configuration
autovacuum configuration
logging
pg_hba.conf
```

as templates or declarative configuration.

But:

```text
passwords
private keys
replication secrets
```

should come from a secret-management system, not plain Git.

Also categorize changes into:

```text
online/session change
reload-required
restart-required
```

to avoid unnecessary database restarts.

---

## 98. How should database schema migrations be handled in CI/CD?

**Answer:** Database changes should not be treated like stateless application deployments.

Safe workflow:

```text
migration validation
→ staging test
→ lock impact assessment
→ backup/recovery readiness
→ production migration
→ validation
```

Prefer backward-compatible **expand/contract** migrations.

Example:

### Release 1

```text
add new column
application writes old + new
```

### Release 2

```text
application reads new
```

### Release 3

```text
remove old column
```

Avoid dangerous changes that unexpectedly acquire long `ACCESS EXCLUSIVE` locks.

Configure migration safeguards where appropriate:

```sql
SET lock_timeout = '5s';
SET statement_timeout = '30min';
```

---

## 99. How would you capacity-plan PostgreSQL?

**Answer:** Track growth trends rather than only current usage.

Forecast:

```text
database size
table/index size
WAL generated/day
backup size
archive size
transactions/sec
connections
CPU
memory
disk latency
IOPS
network
replication bandwidth
```

Example:

```text
Current DB = 3 TB
Growth = 150 GB/month

12-month raw DB estimate
≈ 4.8 TB
```

But capacity must additionally include:

```text
indexes
temporary space
WAL
maintenance operations
backup staging
VACUUM FULL/reindex space if used
filesystem safety margin
```

You generally should not wait for a filesystem to reach 95% before planning expansion.

---

## 100. Production PostgreSQL latency jumped from 20 ms to 2 seconds. Walk through your incident response.

**Answer:** This is an excellent Senior DevOps interview scenario.

### Step 1 — Establish scope

Ask:

```text
When did it begin?
All applications or one?
Reads or writes?
Primary or replicas?
Was there a deployment?
```

### Step 2 — Check database availability and sessions

```sql
SELECT state,
       wait_event_type,
       wait_event,
       count(*)
FROM pg_stat_activity
GROUP BY state, wait_event_type, wait_event;
```

### Step 3 — Check blocking

```sql
SELECT pid,
       pg_blocking_pids(pid),
       query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Step 4 — Find long-running queries

```sql
SELECT pid,
       now() - query_start AS duration,
       query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

### Step 5 — Inspect workload changes

Use:

```text
pg_stat_statements
```

Compare:

```text
calls
execution time
rows
temp I/O
```

against baseline.

### Step 6 — Check PostgreSQL internals

Investigate:

```text
checkpoint activity
WAL rate
autovacuum
dead tuples
temporary files
replication
connection count
```

Useful views:

```sql
SELECT * FROM pg_stat_checkpointer;
SELECT * FROM pg_stat_wal;
SELECT * FROM pg_stat_io;
```

### Step 7 — Check infrastructure

```bash
top
vmstat 1
iostat -xz 1
df -h
```

Look for:

```text
CPU saturation
memory pressure
swap
disk latency
queueing
filesystem exhaustion
```

### Step 8 — Correlate with changes

Check:

```text
application deployment
schema migration
PostgreSQL configuration change
OS update
storage event
traffic spike
cloud infrastructure event
```

### Step 9 — Mitigate safely

Depending on evidence:

```text
terminate pathological query
stop bad deployment
remove blocking transaction
restore pool limits
increase storage
fix archive failure
disable problematic batch process
fail over if the primary infrastructure is unhealthy
```

Do not make five PostgreSQL configuration changes simultaneously because you lose the ability to identify which action solved—or worsened—the problem.

### Step 10 — Root-cause analysis

Document:

```text
timeline
trigger
technical root cause
business impact
detection gap
mitigation
corrective action
preventive action
```

### Example senior-level conclusion

> "I troubleshoot PostgreSQL from evidence rather than configuration folklore. I first determine whether latency originates from query execution, lock contention, connection saturation, WAL/checkpoint activity, vacuum behavior, CPU, memory, storage, networking, replication, or an external infrastructure change. I mitigate the immediate customer impact, preserve evidence, and only then implement and validate the permanent fix."

---

# Final Senior DevOps PostgreSQL Revision Checklist

Before an interview, be comfortable explaining these areas without notes:

### Architecture

```text
MVCC
WAL
checkpoints
background processes
transactions
locks
deadlocks
HOT updates
```

### Deployment

```text
Linux deployment
containers
Kubernetes
storage
connection pooling
minor upgrades
major upgrades
zero-downtime upgrades
```

### Configuration

```text
shared_buffers
work_mem
maintenance_work_mem
effective_cache_size
max_connections
WAL parameters
checkpoint settings
logging
pg_hba.conf
```

### HA

```text
physical replication
logical replication
sync vs async
replication slots
replication lag
hot standby
cascading replication
failover
split brain
fencing
```

### Backup / DR

```text
pg_dump
pg_basebackup
WAL archiving
PITR
RPO
RTO
restore testing
multi-region DR
```

### Performance

```text
EXPLAIN
EXPLAIN ANALYZE
BUFFERS
indexes
statistics
VACUUM
autovacuum
bloat
partitioning
COPY
cache
checkpoint tuning
```

### Troubleshooting

```text
high CPU
high I/O
OOM
disk full
pg_wal full
too many connections
locks
idle transactions
XID wraparound
replication lag
failed standby
authentication
restart failures
```

### Observability

```text
pg_stat_activity
pg_stat_statements
pg_stat_replication
pg_stat_io
pg_stat_wal
pg_stat_checkpointer
pg_stat_archiver
pg_locks
pg_stat_user_tables
```

---

# 20 Commands Worth Memorizing

```sql
-- 1. Current activity
SELECT * FROM pg_stat_activity;

-- 2. Blocking sessions
SELECT pid, pg_blocking_pids(pid), query
FROM pg_stat_activity;

-- 3. Locks
SELECT * FROM pg_locks;

-- 4. Replication
SELECT * FROM pg_stat_replication;

-- 5. Replication slots
SELECT * FROM pg_replication_slots;

-- 6. Database statistics
SELECT * FROM pg_stat_database;

-- 7. Table statistics
SELECT * FROM pg_stat_user_tables;

-- 8. Index statistics
SELECT * FROM pg_stat_user_indexes;

-- 9. WAL statistics
SELECT * FROM pg_stat_wal;

-- 10. I/O statistics
SELECT * FROM pg_stat_io;

-- 11. Archive status
SELECT * FROM pg_stat_archiver;

-- 12. Checkpoint statistics
SELECT * FROM pg_stat_checkpointer;

-- 13. Long transactions
SELECT pid, now() - xact_start, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- 14. XID age
SELECT datname, age(datfrozenxid)
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- 15. Dead tuples
SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 16. Query plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;

-- 17. Reload configuration
SELECT pg_reload_conf();

-- 18. Inspect parameter
SHOW shared_buffers;

-- 19. Configuration source
SELECT name, setting, source, sourcefile
FROM pg_settings
WHERE name = 'shared_buffers';

-- 20. Terminate backend — use carefully
SELECT pg_terminate_backend(<pid>);
```

# Five Principles That Make an Answer Sound Senior-Level

**1. Measure before tuning.**

Do not recommend arbitrary PostgreSQL parameter values without workload, RAM, CPU, storage and concurrency information.

**2. Replication is not backup.**

Use physical/logical replicas for availability and backups + WAL/PITR for recovery.

**3. Failover is not simply promotion.**

Safe failover requires failure detection, fencing, leader election, routing and old-primary isolation.

**4. Autovacuum is essential infrastructure.**

Disabling it to "improve performance" can ultimately create bloat and transaction-ID problems.

**5. Database incidents are systems incidents.**

Always correlate PostgreSQL behavior with:

```text
Application
+
PostgreSQL
+
Linux
+
Storage
+
Network
+
Infrastructure
```

That systems-level approach is what differentiates a Senior DevOps Engineer from someone who only knows PostgreSQL commands.

This set is deliberately weighted toward the questions that tend to distinguish a **Senior DevOps/SRE candidate**: failure modes, operational trade-offs, HA, backups, production tuning and incident diagnosis, rather than ordinary SQL-development questions. PostgreSQL's current administration documentation covers these same core domains—configuration, maintenance, backup/recovery, HA/replication and monitoring. ([PostgreSQL][19])

[1]: https://www.postgresql.org/docs/18/using-explain.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 14.1. Using EXPLAIN"
[2]: https://www.postgresql.org/docs/18/app-pgdump.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_dump"
[3]: https://www.postgresql.org/docs/current/monitoring-stats.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 27.2. The Cumulative Statistics System"
[4]: https://www.postgresql.org/docs/18/routine-vacuuming.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 24.1. Routine Vacuuming"
[5]: https://www.postgresql.org/docs/18/wal-intro.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 28.3. Write-Ahead Logging (WAL)"
[6]: https://www.postgresql.org/docs/18/wal-configuration.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 28.5. WAL Configuration"
[7]: https://www.postgresql.org/docs/18/pgupgrade.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_upgrade"
[8]: https://www.postgresql.org/docs/18/monitoring.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 27. Monitoring Database Activity"
[9]: https://www.postgresql.org/docs/18/auth-pg-hba-conf.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 20.1. The pg_hba.conf File"
[10]: https://www.postgresql.org/docs/18/runtime-config-resource.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 19.4. Resource Consumption"
[11]: https://www.postgresql.org/docs/current/ssl-tcp.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 18.9. Secure TCP/IP Connections with SSL"
[12]: https://www.postgresql.org/docs/18/release-18.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: E.6. Release 18"
[13]: https://www.postgresql.org/docs/18/high-availability.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 26. High Availability, Load Balancing, and Replication"
[14]: https://www.postgresql.org/docs/current/warm-standby.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 26.2. Log-Shipping Standby Servers"
[15]: https://www.postgresql.org/docs/18/backup.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 25. Backup and Restore"
[16]: https://www.postgresql.org/docs/current/app-pgbasebackup.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: pg_basebackup"
[17]: https://www.postgresql.org/docs/18/performance-tips.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 14. Performance Tips"
[18]: https://www.postgresql.org/docs/18/pgstatstatements.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: F.32. pg_stat_statements — track statistics of SQL planning and execution"
[19]: https://www.postgresql.org/docs/18/admin.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Part III. Server Administration"
