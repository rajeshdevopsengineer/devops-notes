# DevOps Interview — Complete Answers

Quick, accurate, with the depth a senior engineer would bring to each.

---

## 🐧 Linux

**Find a file by size**

```bash
# Exactly 100 MB
find /path -type f -size 100M

# Larger than 100 MB
find /path -type f -size +100M

# Smaller than 50 MB
find /path -type f -size -50M

# Between 100 MB and 1 GB
find /path -type f -size +100M -size -1G

# Print human-readable size with results
find /path -type f -size +100M -exec ls -lh {} \;

# Top 20 biggest files anywhere
find / -type f -size +100M 2>/dev/null -exec du -h {} + | sort -hr | head -20
```

Size suffixes: `c` (bytes), `k` (KB), `M` (MB), `G` (GB). `find` rounds *up*, so `-size 1M` actually means files in the range (0, 1MB].

For directory-level disk usage, `du -ah /path | sort -hr | head -20` or interactive `ncdu /path` is usually what you actually want.

**Search by name**

```bash
# Exact name (recursive)
find /path -name "config.yaml"

# Case-insensitive
find /path -iname "*.LOG"

# Wildcards (quote them so the shell doesn't expand)
find / -name "*.conf" 2>/dev/null

# Faster — uses a prebuilt index, but may be slightly stale
locate config.yaml
sudo updatedb     # refresh the index

# Inside files (grep)
grep -r "DATABASE_URL" /etc

# Faster modern alternatives
fd config.yaml          # fd-find
rg "DATABASE_URL" /etc  # ripgrep
```

For interactive fuzzy search: `fzf` piped from `find` or `fd` is what I reach for daily.

**Change permissions**

```bash
# Symbolic
chmod u+x script.sh           # add execute for owner
chmod g-w file.txt            # remove write for group
chmod o=r file.txt            # set "others" to read-only
chmod a+r,a-w file.txt        # multiple changes

# Octal (the form I use most)
chmod 755 script.sh           # rwxr-xr-x — common for executables
chmod 644 file.txt            # rw-r--r-- — common for config files
chmod 600 ~/.ssh/id_rsa       # rw------- — required for SSH keys
chmod 700 ~/.ssh              # rwx------ — directory must be 700

# Recursive
chmod -R 750 /var/www

# Recursive but only directories or only files
find /var/www -type d -exec chmod 750 {} \;
find /var/www -type f -exec chmod 640 {} \;

# Ownership (separate command)
chown user:group file.txt
chown -R www-data:www-data /var/www
```

Octal mapping: `r=4, w=2, x=1`. `755` = `7 (owner: rwx) 5 (group: r-x) 5 (others: r-x)`. SetUID/SetGID/sticky bits use a fourth digit (`4755`, `2755`, `1777` respectively).

---

## 🔧 Terraform

**`terraform taint` vs `terraform untaint`**

`taint` marks a resource for forced recreation on the next apply (destroy + create). `untaint` removes that mark.

**Important context for an interview:** as of Terraform 0.15.2, `taint` and `untaint` are deprecated in favor of `-replace`:

```bash
# Old (still works, deprecated)
terraform taint aws_instance.web
terraform apply

# Modern equivalent
terraform apply -replace=aws_instance.web

# Or in code (Terraform 1.5+) — declarative
terraform {
  # ...
}
# Then run with -replace as a CLI flag
```

When you'd use it: a resource exists but is in a bad state — corrupted EBS volume, misconfigured EC2 that needs a clean rebuild, a Kubernetes resource that drifted in a way refresh can't fix. Force replacement without changing config.

**`terraform refresh`**

Updates Terraform's state file to match the real-world state of resources — reads each tracked resource from the cloud API and writes current attributes back to state. Useful when:
- Resources changed outside Terraform (drift) and you want state to reflect reality before planning.
- Cloud API returned different attributes than expected.
- After an `import` to ensure all attributes are current.

```bash
terraform refresh
terraform plan    # plan now uses refreshed state
```

**Modern note:** `terraform plan` and `terraform apply` automatically refresh before running. Standalone `terraform refresh` was deprecated in 1.0; use `terraform apply -refresh-only` instead, which produces a reviewable plan of state-only changes.

```bash
terraform apply -refresh-only
```

**`init` vs `plan` vs `apply`**

- **`terraform init`** — initializes a working directory: downloads providers, initializes the backend (where state lives), downloads modules, writes `.terraform.lock.hcl`. Run after cloning, after changing backend/providers/modules.
- **`terraform plan`** — computes the diff between desired state (code) and actual state (refreshed). Outputs what would be added (+), changed (~), or destroyed (-). Does not change anything.
- **`terraform apply`** — executes the plan. Walks the resource DAG in dependency order, calling provider CRUD operations, updates state incrementally.

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan      # apply the exact plan you reviewed
```

Applying a saved plan file is the production pattern — what you reviewed is what gets applied.

**Storing Terraform state remotely**

Remote backends are mandatory for any team setup. Common options:

**AWS — S3 + DynamoDB:**
```hcl
terraform {
  backend "s3" {
    bucket         = "company-tfstate"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tfstate-locks"   # state locking
    encrypt        = true
    kms_key_id     = "alias/tfstate"
  }
}
```

**Azure — Storage Account:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "companytfstate"
    container_name       = "tfstate"
    key                  = "prod/network.tfstate"
    use_azuread_auth     = true
  }
}
```

**GCS, Terraform Cloud, Consul** are other options.

Hardening any backend: enable **versioning + soft delete** on the bucket, enable **locking** (DynamoDB for S3, native blob lease for Azure), encrypt with a customer-managed KMS key, restrict access to only the pipeline IAM role and a small SRE group.

**Viewing the state file**

```bash
# List all tracked resources
terraform state list

# Detailed attributes of a specific resource
terraform state show aws_instance.web

# Dump the raw state (be careful — contains secrets)
terraform state pull > current.tfstate
jq . current.tfstate | less

# Show outputs only
terraform output
terraform output -raw load_balancer_dns
terraform output -json | jq .
```

Never edit the state file by hand. Use `terraform state mv`, `rm`, `replace-provider`, or `import` subcommands — they validate.

**Avoiding sensitive value exposure during apply**

```hcl
variable "db_password" {
  type      = string
  sensitive = true     # masks the value in plan/apply output and outputs
}

output "db_connection_string" {
  value     = "postgres://admin:${var.db_password}@${aws_db_instance.this.endpoint}/mydb"
  sensitive = true     # output won't be printed; must use -raw to retrieve
}
```

What `sensitive = true` does: masks the value in CLI output during plan/apply, but the value is **still in the state file in plain text**. So:
- Encrypt state at rest (KMS-encrypted S3 backend).
- Restrict state access via IAM.
- Never `terraform output` secrets in CI logs.
- For provider-level secrets (e.g., a DB resource's `password`), most providers auto-mark them sensitive.

In CI, never `echo` Terraform variables that contain secrets.

**Handling secrets in Terraform**

Layered, from worst to best:

1. **Hardcoded in `.tf` files** — never.
2. **`.tfvars` files gitignored** — acceptable for local dev, risky for prod.
3. **Environment variables** (`TF_VAR_db_password`) — better, but still requires the value to live somewhere.
4. **Reference from a secrets manager at runtime**:
   ```hcl
   data "aws_secretsmanager_secret_version" "db" {
     secret_id = "prod/rds/master-password"
   }
   resource "aws_db_instance" "this" {
     password = data.aws_secretsmanager_secret_version.db.secret_string
     lifecycle {
       ignore_changes = [password]   # rotation outside TF doesn't cause drift
     }
   }
   ```
5. **Generate in Terraform, store in secrets manager**:
   ```hcl
   resource "random_password" "db" { length = 32 }
   resource "aws_secretsmanager_secret_version" "db" {
     secret_id     = aws_secretsmanager_secret.db.id
     secret_string = random_password.db.result
   }
   ```
6. **Eliminate the secret entirely** — IAM database authentication for RDS, IRSA for EKS workloads, workload identity for GCP/Azure. The best secret is no secret.

**Avoiding hardcoded values**

- **Variables with sensible defaults**: `variable "instance_type" { default = "t3.medium" }`.
- **Per-environment `.tfvars`** files: `terraform apply -var-file=prod.tfvars`.
- **Data sources** for values that exist elsewhere: `data "aws_ami" "amazon_linux" { ... }` instead of pinning an AMI ID.
- **`locals`** for computed values and DRY:
  ```hcl
  locals {
    common_tags = {
      Environment = var.env
      Project     = var.project
      ManagedBy   = "terraform"
    }
  }
  ```
- **Modules** for repeated patterns; pass values as inputs.
- **Workspace-aware lookups**: `lookup({ dev = "small", prod = "large" }, terraform.workspace)`.

The principle: anything that varies between environments goes into a variable; anything reused goes into a local or module.

**Importing existing cloud infrastructure**

**Step 1 — Write the resource block matching the existing resource:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

**Step 2 — Import via CLI (classic):**
```bash
terraform import aws_vpc.main vpc-0abc123def456
```

**Step 2 (modern, Terraform 1.5+) — declarative `import` block:**
```hcl
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456"
}
```

**Step 3 — Generate the config (Terraform 1.5+):**
```bash
terraform plan -generate-config-out=generated.tf
```

**Step 4 — Iterate**: run `terraform plan`. It almost always shows changes initially because your hand-written config doesn't match every actual attribute. Update the config until plan shows zero changes.

**Step 5 — Verify** with `terraform apply` (should be a no-op).

For bulk imports (existing brownfield estate), **terraformer** can scan a cloud and generate Terraform + state for hundreds of resources at once. Output requires cleanup but vastly faster than hand-writing each block.

---

## ☁️ AWS

**EKS vs ECS**

| | EKS | ECS |
|---|---|---|
| Orchestrator | Kubernetes (upstream) | AWS-native proprietary |
| Portability | Kubernetes API — works elsewhere | AWS-only |
| Complexity | Higher | Lower |
| Ecosystem | Massive (Helm, Operators, CNCF) | Smaller, AWS-aligned |
| Compute | EC2, Fargate, hybrid | EC2, Fargate |
| Networking | VPC CNI, Calico, Cilium | AWS VPC mode (per-task ENI) |
| Pricing | $0.10/hour per cluster + nodes | Free control plane + nodes |
| Best for | Multi-cloud orgs, K8s expertise, complex workloads | AWS-only shops, simpler workloads, smaller teams |

**Quick decision rule:** if you need Kubernetes (Helm, custom operators, multi-cloud portability), use EKS. If you're AWS-only and want the simplest path to running containers, ECS Fargate. For purely serverless containers with simple needs, ECS Fargate is genuinely operationally lighter than EKS.

**S3 lifecycle policy**

Rules that automatically transition objects between storage classes or expire them after defined ages. Saves money by moving cold data to cheaper tiers.

```json
{
  "Rules": [{
    "Id": "archive-and-expire",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER_IR" },
      { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "Expiration": { "Days": 2555 },  // 7 years
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
  }]
}
```

Common use cases: log archival, backup retention, compliance-driven retention, cleaning up incomplete multipart uploads, expiring old versions on versioned buckets.

**Maximum S3 object size**

5 TB per object. Practical constraints:
- Single PUT: max 5 GB.
- Multipart upload: required for >100 MB, supports up to 5 TB total (10,000 parts × 5 GB per part).

**S3 bucket size limit**

No limit. Buckets scale to virtually unlimited storage and object counts. Practical limits:
- 100 buckets per account by default (can be raised to 1,000 via support).
- Performance: 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second per prefix. Beyond that, distribute keys across more prefixes for higher throughput.

**Lambda maximum runtime**

15 minutes (900 seconds). If you need longer execution, alternatives:
- **AWS Step Functions** for orchestration of multiple Lambda invocations.
- **ECS/Fargate task** for long-running workloads.
- **AWS Batch** for compute jobs.
- **EC2** or **EKS** for indefinite runtime.

Other Lambda limits worth knowing: 10 GB memory, 10 GB ephemeral `/tmp` (was 512 MB by default until 2022), 10,240 MB max memory, container images up to 10 GB.

**Common Lambda issues**

**Cold start** — when a new execution environment must be initialized (container spin-up, runtime init, code load). Typical impact: 100ms–10s depending on runtime, package size, VPC config.

Mitigations:
- **Provisioned Concurrency** — pre-warmed instances always ready. Cost: pay for the reserved capacity.
- **Smaller deployment packages** — fewer dependencies, lazy imports.
- **Lambda SnapStart** (Java, .NET, Python, Ruby) — snapshot init state for sub-second cold starts.
- **Arm64 (Graviton)** — faster cold starts in many runtimes, also cheaper.
- **Avoid VPC unless required** — VPC-attached Lambdas had massive cold starts historically (now much improved with Hyperplane ENIs).
- **Keep functions warm** with scheduled invocations (less needed with provisioned concurrency).

**Other common Lambda issues:**
- **Concurrency limits** — default 1,000 concurrent executions per region. Hitting this throttles invocations.
- **Timeout misconfiguration** — function timeout shorter than downstream call, leading to retries.
- **Memory undersizing** — Lambda CPU scales with memory; an undersized function is slow *and* expensive.
- **State management** — Lambda is stateless; reusing `/tmp` between invocations is a common foot-gun.
- **Async retry behavior** — failed async invocations retry twice; without a DLQ, errors are lost.
- **Dependency bloat** — unzipped deployment > 250 MB doesn't fit; use Lambda Layers or container images.
- **Logging cost** — verbose logs to CloudWatch can dominate Lambda billing.

**RDS high CPU troubleshooting**

Methodical approach:

**1. Confirm and characterize the spike**
- CloudWatch `CPUUtilization` metric — duration and shape (sudden spike vs gradual climb).
- Performance Insights — top SQL by load, top wait events.
- Connection count (`DatabaseConnections`) — sometimes high CPU is caused by connection storm.

**2. Identify the culprit**

```sql
-- PostgreSQL: top queries by total time
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Currently running queries
SELECT pid, usename, state, wait_event, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Long-running queries
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '1 minute';

-- For MySQL
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 20;
SHOW PROCESSLIST;
```

**3. Common root causes ranked by likelihood**

- **Missing index** — sequential scan on a large table. Fix: add index after verifying via EXPLAIN ANALYZE.
- **Bad query plan after stats drift** — `ANALYZE` the table; consider `VACUUM ANALYZE` on Postgres.
- **N+1 query pattern** — app issuing many small queries instead of one batched. Fix at app layer.
- **Connection storm** — too many short-lived connections; each connect/disconnect is expensive. Fix with PgBouncer/RDS Proxy.
- **Lock contention** — long transactions blocking others. Identify via `pg_locks` / `INNODB_LOCKS`.
- **Inefficient ORM queries** — generated SQL without proper joins/indexes.
- **VACUUM/maintenance overhead** — autovacuum running on a large table during peak. Tune `autovacuum_vacuum_cost_limit`.
- **Workload genuinely outgrew instance** — needs vertical scale or read replica offload.

**4. Mitigate and fix**

Immediate (if it's an emergency):
- Kill long-running offending queries (`pg_terminate_backend(pid)`).
- Failover to a replica if available.
- Vertical scale (db.r6g.large → db.r6g.xlarge) with minimal downtime via modify.

Permanent:
- Add the missing index.
- Refactor the bad query.
- Add a read replica and route read traffic.
- Add caching (ElastiCache).
- Move to Aurora if scaling characteristics warrant.

**5. Prevent recurrence**
- Performance Insights enabled (with retention >7 days for trend analysis).
- Slow query log alerts.
- Query review in PR for any new migrations.
- Load testing before deploys.
- Alert on CPU > 70% for 10 minutes (proactive, not reactive).

---

## 🌐 Networking

**CIDR block**

**Classless Inter-Domain Routing** — a notation for IP address ranges in the form `<network>/<prefix-length>`. The prefix length specifies how many leading bits identify the network; the rest are host bits.

`10.0.0.0/16` means:
- 16 bits identify the network (`10.0.x.x`).
- 16 bits available for hosts → 2^16 = 65,536 IP addresses.
- Range: `10.0.0.0` to `10.0.255.255`.

Common sizes:
- `/8` = 16,777,216 IPs (`10.0.0.0/8`)
- `/16` = 65,536 IPs (typical VPC size)
- `/24` = 256 IPs (typical subnet)
- `/28` = 16 IPs (smallest AWS allows)
- `/32` = a single IP (used in route entries)

AWS reserves 5 IPs per subnet (`.0`, `.1`, `.2`, `.3`, `.255`), so a `/24` gives 251 usable IPs.

**Public vs private subnet**

The distinction is **whether the route table has a default route to an Internet Gateway**:

- **Public subnet**: route table has `0.0.0.0/0 → IGW`. Resources can have public IPs and be reachable from the internet. Used for ALBs, NAT Gateways, bastions.
- **Private subnet**: no IGW route. Outbound internet via a NAT Gateway/Instance in a public subnet. Resources have private IPs only, not directly reachable from the internet. Used for application servers, databases, internal services.

The "subnet" itself isn't intrinsically public or private — it's the routing that makes it so. You can have multiple subnets sharing the same route table.

**Designing CIDR blocks for VPC + subnets**

**Principles:**
1. **Start big** — `/16` for the VPC unless you have a strong reason not to. Re-IPing later is painful.
2. **Plan for the future** — leave room for additional subnet tiers, AZs, peered VPCs.
3. **No overlap with other VPCs** you might peer with (corporate network, other AWS accounts).
4. **Three AZs minimum** for production HA.
5. **Tier per subnet** — public, private-app, private-data — each with its own NACL.

**Example design for a production VPC:**

```
VPC: 10.0.0.0/16 (65,536 IPs)

Public subnets (small — only LBs and NAT GWs):
  10.0.0.0/24    — us-east-1a (256 IPs)
  10.0.1.0/24    — us-east-1b
  10.0.2.0/24    — us-east-1c

Private app subnets (large — most workloads, EKS pods if using VPC CNI):
  10.0.16.0/20   — us-east-1a (4,096 IPs)
  10.0.32.0/20   — us-east-1b
  10.0.48.0/20   — us-east-1c

Private data subnets (medium — RDS, ElastiCache):
  10.0.64.0/22   — us-east-1a (1,024 IPs)
  10.0.68.0/22   — us-east-1b
  10.0.72.0/22   — us-east-1c

Reserved for future expansion:
  10.0.96.0/19 onwards
```

**For EKS specifically:** pod density per node × max nodes × number of AZs gives your subnet sizing requirement. With AWS VPC CNI and 30 pods per node, a `/24` (256 IPs) supports ~8 nodes per AZ. For high-density clusters, use **prefix delegation** (16 IPs per ENI prefix) or a **secondary CIDR** like `100.64.0.0/16` (CGN range) for pod IPs.

**Common mistakes to avoid:**
- VPC CIDR that overlaps with corporate network (no peering possible later).
- `/24` subnets in production — too small for growth.
- Public subnets sized large — they should be small; NAT Gateways and LBs don't need many IPs.
- Forgetting AWS reserves 5 IPs per subnet.

---

## 🚀 CI/CD & DevOps

**Pipeline failures encountered**

Categorized by stage and cause:

**Build stage:**
- **Compile failures** — code syntax errors, missing dependencies, version conflicts.
- **Test failures** — real regressions, flaky tests (timing/race conditions, leaky test state), environment differences.
- **Dependency resolution** — registry timeouts, breaking changes in transitive deps, license violations.
- **Resource exhaustion** — agent ran out of disk (huge `node_modules`), OOM during build.
- **Network blips** — registry unreachable, intermittent DNS.

**Quality/security gates:**
- **SonarQube quality gate** failing on new code coverage or duplication.
- **Trivy/Snyk** blocking on CRITICAL/HIGH CVEs.
- **License scan** finding GPL deps in proprietary code.
- **IaC scan** (tfsec, Checkov) flagging policy violations.

**Image/artifact stage:**
- **Docker build cache miss** causing 30-minute builds.
- **Registry push failures** — auth expired, rate limit, throughput cap.
- **Image scan failures** on newly-disclosed CVEs.

**Deploy stage:**
- **kubeconfig/credentials expired**.
- **Helm chart rendering errors** — missing values, schema validation.
- **Resource conflicts** — immutable field changes, name conflicts.
- **Probes failing** — new image's `/health` endpoint moved.
- **ImagePullBackOff** — wrong tag, missing pull secret, ECR auth.
- **Rollout timeout** — new pods not becoming Ready in time.
- **Cluster autoscaler lag** — pods stuck Pending waiting for nodes.

**Post-deploy:**
- **Smoke tests failing** — environment-specific config issue.
- **SLO regression** — canary detected error rate or latency increase.
- **ArgoCD sync stuck** — webhook failure, manifest validation error.

**Troubleshooting CI/CD failures**

Structured approach:

**1. Reproduce locally if possible**
- Many "CI-only" failures reproduce when you run the exact same Docker container the agent uses.
- `docker run -v $PWD:/work -w /work <ci-image> <command>` often reveals the issue in seconds.

**2. Read the logs top to bottom**
- The first failure cascades; don't fix the last error before understanding the first.
- For multi-stage pipelines, find the *first* failed step.

**3. Differentiate transient vs deterministic**
- Run the same pipeline twice. Same failure → deterministic. Different failure → flaky test or transient (network, agent).
- Deterministic: code or config fix.
- Transient: retry logic in the pipeline (with alerting on retry usage so root cause gets fixed).

**4. Bisect with git**
- `git bisect` to find the commit that introduced a regression.
- Compare the last green pipeline with the first red one — what changed?

**5. Common diagnostic commands**

```bash
# In pipeline shell
env | sort                       # what env vars are set?
which python java node           # path issues
ls -la /workspace                # working dir state
df -h                            # agent disk
free -m                          # agent memory
ps auxf                          # what's running

# Docker-specific
docker ps -a                     # exited containers
docker logs <container>
docker images                    # disk space
docker system df

# Kubernetes deploy failures
kubectl describe pod <pod>       # events at the bottom — gold
kubectl logs <pod> --previous    # previous restart's logs
kubectl get events --sort-by='.lastTimestamp' -n <ns>
```

**6. Pipeline-side fixes**

- **Pin versions** — Docker base image, Node version, Maven version, Terraform version. "Worked yesterday, broken today" almost always traces to an unpinned version.
- **Cache dependencies** intelligently — invalidate when lock file changes, not on every build.
- **Increase retries** for known-transient operations (`retry: 3` on registry pushes).
- **Increase timeouts** when the issue is genuinely slow steps, not hung steps.
- **Split slow stages** so failures fail fast.
- **Add structured logging** in CI scripts so failures are diagnosable from logs alone.

**7. Postmortem habits**

- Track flaky test rate as a metric; aim for <0.5%.
- Treat repeated pipeline failures as bugs — file tickets, fix the underlying cause.
- "Fix the build" culture — a broken `main` is top priority; nobody merges over a red build.

**The principle:** the goal isn't a green pipeline at all costs — it's a pipeline that tells you the truth quickly. Retries that mask flakiness create slower truth, not faster builds.

---

## 🐳 Docker

**Purpose of `docker tag`**

`docker tag` assigns an additional name (and optionally a different repository) to an existing image. The image data isn't duplicated; both tags point to the same image ID.

```bash
# After building
docker build -t myapp:1.4.2 .

# Add a tag for ECR
docker tag myapp:1.4.2 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Add a "latest" tag pointing at the same image
docker tag myapp:1.4.2 myapp:latest

# Now push to a registry
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

Use cases:
- **Re-tag for a registry** — local builds tagged `myapp:1.4.2` need re-tagging with the registry URL before push.
- **Mark a build with multiple tags** — same image as `1.4.2`, `1.4`, `1`, `latest`.
- **Promote between environments** — re-tag a staging image with a prod tag after validation (or use `docker buildx imagetools create` for manifest-level re-tagging in modern setups).

Tagging principles for production:
- **Immutable build tags** — `1.4.2-a1b2c3d-build2087` (semver + git SHA + build number). Never overwritten.
- **Floating tags** (`latest`, `1.4`, `1`) — for dev/local convenience only. Never referenced from prod manifests.

**Building an image from a Dockerfile**

```bash
docker build -t myapp:1.4.2 .
```

Breakdown:
- `docker build` — invokes the build.
- `-t myapp:1.4.2` — tag the resulting image (`-t` can be specified multiple times for multiple tags).
- `.` — the **build context**. Docker daemon receives all files in this directory (minus `.dockerignore` excludes); the `Dockerfile` is expected here unless `-f` specifies otherwise.

Useful variations:

```bash
# Different Dockerfile location
docker build -f docker/prod.Dockerfile -t myapp:prod .

# Multiple tags in one build
docker build -t myapp:1.4.2 -t myapp:latest .

# Build args
docker build --build-arg VERSION=1.4.2 --build-arg NODE_ENV=production -t myapp:1.4.2 .

# Target a specific stage (multi-stage builds)
docker build --target build -t myapp:build-stage .

# Use BuildKit for cache mounts and secrets
DOCKER_BUILDKIT=1 docker build -t myapp:1.4.2 .

# Or the modern unified frontend
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.4.2 --push .

# Pass secrets at build time without storing in layers (BuildKit)
docker build --secret id=npmtoken,src=$HOME/.npmrc -t myapp:1.4.2 .

# No cache (force fresh layers)
docker build --no-cache -t myapp:1.4.2 .

# Pull base images even if cached locally
docker build --pull -t myapp:1.4.2 .
```

Always pair with a `.dockerignore` to exclude `.git`, `node_modules`, secrets, build artifacts — keeps the build context small and prevents secrets from leaking into image layers.

---

That's the full set. Common follow-ups to be ready for:

- **"Show me the difference between `docker tag` and `docker push`."** — `tag` is local-only metadata; `push` actually transfers layers to a registry.
- **"How would you write a more efficient subnet design for a 50-account AWS Organization?"** — IPAM, non-overlapping CIDR ranges per account, central network account peering hub.
- **"How do you decide between Provisioned Concurrency and SnapStart for Lambda?"** — cost vs cold-start tolerance, runtime support, traffic shape.

# Senior DevOps Interview — Complete Answers

A mixed bag covering architecture, Kubernetes depth, Helm, Docker internals, and CI/CD. Going through each with the depth a senior engineer should bring.

---

## 1. Two-tier vs three-tier architecture — use cases and AWS resources

**Two-tier (client → database):**

Use case: a simple internal admin tool, a small data-entry app, a CRUD dashboard used by 20 internal users. The "client" (could be a thick desktop app or a web app) talks directly to the database — no separate application server.

AWS resources:
- **EC2** (or **Elastic Beanstalk** for a managed flavor) running the application that includes both UI and DB access logic.
- **RDS** as the database tier.
- **ALB** in front of the EC2 if web-accessed.
- **Route 53** for DNS.
- **VPC** with public subnet (LB/EC2) and private subnet (RDS).
- **ACM** for TLS.

When two-tier makes sense: small scale, simple workloads, low traffic, internal tools, MVPs. It's operationally cheaper but doesn't scale and mixes presentation with data logic.

**Three-tier (presentation → application → data):**

Use case: a customer-facing e-commerce platform, a SaaS product, a banking portal — anything customer-facing with non-trivial scale, separation of concerns, and security requirements.

AWS resources by tier:

**Presentation tier:**
- **CloudFront** for global edge caching, WAF integration, TLS termination at edge.
- **S3** for static frontend assets (React/Angular/Vue bundles).
- **Route 53** for DNS with health-check-based failover.
- **ACM** for certificates.
- **AWS WAF** + **Shield** for DDoS protection.

**Application tier:**
- **ALB** in public subnets fronting the app servers.
- **EKS** or **ECS Fargate** for containerized microservices, or **EC2 Auto Scaling Group** for traditional apps.
- **Lambda** for event-driven workloads.
- **API Gateway** for REST/HTTP/WebSocket APIs.
- **SQS / SNS / EventBridge** for async messaging.
- **ElastiCache (Redis)** for caching and sessions.

**Data tier:**
- **RDS Aurora** (Postgres or MySQL) Multi-AZ for transactional data.
- **DynamoDB** for high-write, low-latency NoSQL needs.
- **S3** for object storage with lifecycle policies.
- **OpenSearch** for search/analytics.
- All in **private data subnets**, no internet route.

**Cross-cutting:**
- **IAM** with IRSA for pod-level identity.
- **KMS** for encryption.
- **Secrets Manager** for credentials.
- **CloudWatch + Prometheus/Grafana** for observability.
- **VPC** with three subnet tiers (public, private-app, private-data) across three AZs.

**Architect's framing:**
> "Three-tier is the default for any customer-facing production system. Two-tier survives in legacy and small internal tools. The real value of three-tier isn't just scalability — it's that each tier can be secured, scaled, and replaced independently. I can swap the DB engine without touching the presentation layer; I can scale app servers without scaling the DB."

---

## 2. Major drawbacks of ECS and EKS

**ECS drawbacks:**

- **AWS-only lock-in** — proprietary orchestration; skills and IaC don't transfer to other clouds. Migrating off ECS to Kubernetes later requires rewriting task definitions, service definitions, deployment patterns.
- **Smaller ecosystem** — no equivalent to Helm charts, Operators, or the CNCF tool universe. You build things ECS doesn't ship.
- **Limited GitOps story** — ArgoCD, Flux don't natively manage ECS. You're back to imperative deploys via CodePipeline or custom tooling.
- **Less expressive scheduling** — no equivalent to topology spread constraints, complex affinity rules, or PriorityClasses. ECS placement constraints are simpler but less flexible.
- **Service discovery limitations** — relies on Cloud Map (DNS-based) or ALB target groups. No equivalent to Kubernetes Services + Endpoints with mesh integration.
- **Stateful workload story is weak** — no StatefulSets equivalent; stateful workloads typically push you to RDS/managed services or DIY on EC2.
- **Multi-region orchestration is manual** — no equivalent to Cluster API or fleet management.

**EKS drawbacks:**

- **Operational complexity** — Kubernetes itself is a distributed system. Even managed control plane doesn't free you from understanding etcd, CRDs, networking, RBAC, admission controllers.
- **Cost overhead** — $0.10/hour per cluster (~$73/month) per cluster. At 20 clusters, that's $1,460/month before any compute. ECS control plane is free.
- **VPC CNI IP exhaustion** — the AWS VPC CNI assigns one VPC IP per pod, so subnet sizing must accommodate pod count. Subnet exhaustion is a common incident.
- **EKS version lifecycle** — every K8s minor version is supported only ~14 months; forced upgrades on a treadmill. Each upgrade requires testing because of breaking API changes (`PodSecurityPolicy`, `Ingress` API moves, etc.).
- **IRSA and cluster auth setup** is non-trivial — OIDC issuer config, IAM role trust policies, aws-auth ConfigMap drift is a classic gotcha.
- **Bigger blast radius for control-plane issues** — control plane managed by AWS, but workloads still feel API server throttling, etcd slowdowns.
- **Observability requires more setup** — CloudWatch Container Insights works but is limited; production setups bolt on Prometheus, Grafana, Loki, all of which you operate.
- **Tooling churn** — Kubernetes ecosystem moves fast; what was best-practice 2 years ago (Tiller-based Helm 2, PSPs) is now deprecated.

**Architect's framing:**
> "ECS is the simpler answer if you're committed to AWS and your workload fits the model — fewer moving parts, lower operational toil, lower cost. EKS is the right answer when you need Kubernetes-specific capabilities, multi-cloud portability, or the ecosystem (Helm, operators, mesh). The biggest practical drawback of EKS is that you're operating a distributed system on top of another distributed system, and the abstractions leak. For ECS, the biggest practical drawback is that you're locked into AWS's roadmap for orchestration features."

---

## 3. RDS volume over-provisioned (100GB, low usage) — how to decrease size

**Critical answer up front: RDS does not support shrinking storage in place.** You can only increase storage; decreasing requires a different approach.

**The ideal way — logical migration to a smaller instance:**

**Approach 1: Snapshot, restore to smaller storage, cut over** (DOESN'T WORK — RDS restores use the original snapshot's size minimum). Worth mentioning as a common misconception so the interviewer knows you know.

**Approach 2: Logical dump and restore (the standard approach)**

1. **Provision a new RDS instance with the desired smaller storage** (say 20GB).
   ```bash
   aws rds create-db-instance \
     --db-instance-identifier myapp-rds-small \
     --db-instance-class db.t3.medium \
     --engine postgres \
     --allocated-storage 20 \
     --storage-type gp3 \
     --master-username admin \
     --master-user-password <from-secrets-manager>
   ```

2. **Logical dump from the source:**
   ```bash
   pg_dump -h source-rds.amazonaws.com -U admin -F c -d mydb -f mydb.dump
   ```

3. **Restore to the target:**
   ```bash
   pg_restore -h target-rds-small.amazonaws.com -U admin -d mydb -j 4 mydb.dump
   ```

4. **Cut over with minimal downtime:**
   - Best: use **AWS DMS** (Database Migration Service) for continuous replication. Replicates ongoing changes from old to new while the app keeps writing to old.
   - Once caught up, brief downtime: stop writes on old, drain final changes, point app to new (via DNS / Secrets Manager update).
   - For PostgreSQL: **logical replication** (native) can do the same without DMS.

5. **Verify and retire the old instance** after a buffer period.

**Approach 3: For Aurora — different mechanics**

Aurora storage is decoupled from compute. Aurora storage **auto-grows in 10GB increments and bills only for what you use**. There's no "over-provisioned" Aurora storage problem — it scales down automatically too (in recent versions). If on Aurora, the answer is "no action needed" — verify by checking actual used storage vs allocated.

**Approach 4: Blue-green deployment (RDS native, 2022+)**

```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name shrink-storage \
  --source <source-arn> \
  --target-engine-version <version> \
  ...
```

Creates a replica with different config (including smaller storage if data fits), then atomic switchover with minimal downtime. Cleaner than manual migration for supported scenarios.

**The architect's answer in the interview:**
> "RDS storage can't be shrunk in place — Amazon allows only increases. To genuinely reduce the storage, I'd provision a new smaller instance and migrate the data. For minimal downtime, I'd use AWS DMS or PostgreSQL native logical replication to keep the new instance in sync while the app continues writing to the old one. After a brief cutover window — typically 30 seconds to a few minutes — I'd update the application's connection string (ideally via Secrets Manager so no app redeploy is needed) and retire the old instance after a verification period. Worth noting: if this is Aurora rather than RDS, storage scales automatically and there's nothing to fix. Also worth asking: is the 100GB actually a cost problem? At gp3 storage rates, 100GB is ~$10/month — it might be cheaper to leave it and apply this engineering effort elsewhere."

That last point — questioning whether the optimization is worth the effort — shows seniority.

---

## 4. Domain-based applications in Kubernetes

This is the interviewer asking what *kind* of business application I run in K8s, with the context that team sizes and pod counts vary (10 pods vs 100+ pods).

A senior-level answer with concrete domains:

> "Across teams I've worked with, Kubernetes runs a range of business domains:
>
> **For the team running ~10 pods:**
> - A loan-origination microservice domain — about 8 services covering application intake, credit decisioning, document verification, payment scheduling. Stateless Java Spring Boot services, fronted by an API Gateway, backed by RDS Aurora and DynamoDB. The pod count stays small because each service handles modest transactional volume — peak ~500 TPS — and we sized for predictable workloads with HPA min: 2, max: 6.
>
> **For the team running ~100+ pods:**
> - A consumer-facing e-commerce platform — order management, inventory, pricing, cart, checkout, recommendation, search, notification, payment, shipping, fulfillment. ~30 microservices, each with 4-10 replicas across three AZs, totalling ~150-200 pods at baseline and scaling to ~400 during peak events (Black Friday, sale launches). Polyglot: Java, Node.js, Python, Go. Backed by Aurora, DynamoDB, Kafka (MSK), Elasticsearch, Redis.
>
> The difference in pod count maps to traffic shape and decomposition strategy. The smaller domain is fewer services with higher per-service replica counts; the e-commerce platform has many narrowly-scoped services with moderate replication. Both run on EKS with the same platform conventions: ArgoCD for GitOps, Prometheus/Grafana/Loki, Istio for mesh, Karpenter for node scaling.
>
> A few other domains I've supported in K8s:
> - **Fraud detection** — event-driven Python services consuming from Kafka, scaled via KEDA on consumer lag.
> - **Internal developer platform** — Backstage, ArgoCD, Vault, image registries — the meta-platform that supports product teams.
> - **Data pipelines** — Airflow on K8s with KubernetesPodOperator for ETL jobs."

The strength of this answer is concrete domain detail, honest scale numbers, and the meta-observation that pod count varies by decomposition strategy, not just business size.

---

## 5. End-to-end application management in Kubernetes

> "Yes — end-to-end ownership is the model I prefer. For one platform specifically, I led the journey from initial design to steady-state operation:
>
> **Design phase:** Translated the application architecture into Kubernetes primitives — Deployments for stateless services, StatefulSets for the Kafka cluster, HPA + KEDA for scaling, NetworkPolicies for zero-trust, NGINX Ingress + cert-manager for TLS. Chose EKS over self-managed for the control-plane SLA. Designed the cluster topology: three AZs, separate node pools by workload tier (system, prod, batch), Karpenter for elastic scaling.
>
> **Infrastructure provisioning:** Terraform modules for VPC, EKS cluster, IAM (including IRSA for each service), supporting AWS services (RDS, MSK, ElastiCache, S3). Per-environment state files in S3 with DynamoDB locking.
>
> **Platform layer:** Installed and configured the in-cluster tooling — ArgoCD for GitOps, Prometheus/Alertmanager/Grafana via kube-prometheus-stack, Loki + Promtail for logs, Tempo for traces, External Secrets Operator for syncing from AWS Secrets Manager, Kyverno for policy enforcement.
>
> **CI/CD:** GitHub Actions pipelines per service — build, test, SonarQube, Trivy, Cosign sign, push to ECR, bump tag in GitOps repo. Argo Rollouts for canary deployments on tier-1 services.
>
> **Observability:** RED metrics on every service via Micrometer/OpenTelemetry auto-instrumentation. Dashboards templated per service. SLO definitions in code, alerts mapped to runbooks.
>
> **Day-2 operations:** On-call rotation, runbooks for common incidents, blameless postmortems, quarterly DR game days, monthly capacity reviews. Cost visibility via Kubecost.
>
> **Continuous improvement:** Cluster upgrades (one minor version per quarter), node OS image refresh monthly, dependency upgrades, periodic chaos experiments via Chaos Mesh to validate failure handling.
>
> The thing that makes 'end-to-end' real isn't just having touched each piece — it's owning the outcomes. When the platform has a Sev-1, my team is the one explaining what happened, why, and what we're doing about it. That accountability shapes how I design things from the start."

---

## 6. Init Container vs StatefulSet

Different concepts at different layers — important to clarify the framing of the question.

**Init Container** — a *container within a pod* that runs to completion before the main containers start. Used for setup tasks: schema migrations, waiting for dependencies, fetching config, file permissions, prep work.

**StatefulSet** — a *workload controller* (peer to Deployment) that manages a set of pods with stable identity, ordered startup, and per-pod storage. Used for stateful applications like databases, message brokers, distributed coordination services.

**Side-by-side:**

| | Init Container | StatefulSet |
|---|---|---|
| Layer | Container-level (inside a pod) | Workload-level (manages pods) |
| Purpose | Prep work before main containers | Orchestrate stateful applications |
| Lifecycle | Runs once per pod, to completion | Long-running, manages many pods |
| Storage | Shares pod's volumes | Per-pod PersistentVolumeClaims |
| Identity | None | Stable, ordered (pod-0, pod-1, ...) |
| Ordering | Sequential within a pod | Sequential across pods at startup |
| Failure handling | Pod doesn't start until init succeeds | Failed pod rescheduled with same identity |

**They're orthogonal.** A StatefulSet's pods can have init containers; a Deployment's pods can also have init containers. The init container is part of the pod spec; the controller (Deployment, StatefulSet, DaemonSet, Job) is what manages the pods.

**Example combining both** — a Kafka StatefulSet with an init container that waits for Zookeeper:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka
  replicas: 3
  selector: { matchLabels: { app: kafka } }
  template:
    metadata: { labels: { app: kafka } }
    spec:
      initContainers:
      - name: wait-for-zookeeper
        image: busybox
        command: ['sh', '-c', 'until nc -z zookeeper 2181; do sleep 2; done']
      containers:
      - name: kafka
        image: kafka:3.6
        # ...
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      resources: { requests: { storage: 100Gi } }
```

---

## 7. Liveness and readiness probes on init containers — should we?

**No, you shouldn't, and Kubernetes doesn't allow it as of v1.28+ for traditional init containers.**

**The reasons it doesn't make sense:**

1. **Init containers run to completion** — they're not long-lived. Liveness probes are designed for processes that keep running; readiness probes are designed to gate traffic to pods that are accepting requests. Init containers do neither.

2. **The semantic of "ready"** doesn't apply — init containers never receive traffic. They run, finish (successfully or not), and exit.

3. **The semantic of "alive"** is implicit — if an init container hangs, the pod startup hangs. The pod's `startupProbe` on the main container effectively serves the same role at the pod level.

4. **Lifecycle ordering** — Kubernetes runs init containers serially before the main containers start. Probe loops don't fit this serial execution model.

**The exception worth knowing about — Sidecar Containers (Kubernetes 1.28+ as alpha, 1.29+ as beta):**

Kubernetes introduced "native sidecar containers," which are technically a special kind of init container with `restartPolicy: Always`. These DO support probes because they run for the lifetime of the pod alongside the main containers.

```yaml
spec:
  initContainers:
  - name: logging-sidecar
    image: fluent-bit
    restartPolicy: Always         # makes this a sidecar, not a classic init
    livenessProbe:                # ALLOWED for sidecars
      httpGet: { path: /, port: 2020 }
    readinessProbe:               # ALLOWED for sidecars
      httpGet: { path: /, port: 2020 }
  containers:
  - name: app
    image: myapp:1.0
```

So the precise answer to "shall I attach probes to init containers":
- **Classic init containers (run to completion):** no, not advisable and not supported.
- **Sidecar containers (restartPolicy: Always):** yes, probes are supported and useful.

**Architect's framing:**
> "Init containers are short-lived prep work — probes don't fit their lifecycle. If you want long-running auxiliary containers with their own health checks, the right primitive is a sidecar — either a traditional sidecar in the `containers` array, or the new native sidecar via init container with `restartPolicy: Always`. The fact that someone wants to put a probe on an init container usually means they've actually got a sidecar workload, just modeled wrong."

---

## 8. Why StatefulSet over Deployment+ReplicaSet?

The interviewer is testing whether I understand when StatefulSet's specific guarantees are required versus when they're overkill.

**StatefulSet provides three things Deployments don't:**

1. **Stable pod identity** — pods named with ordinal suffix (`kafka-0`, `kafka-1`, `kafka-2`), preserved across restarts, rescheduling, and node migrations. Pod `kafka-0` is *always* `kafka-0`.

2. **Stable DNS** — each pod gets its own DNS record (`kafka-0.kafka.namespace.svc.cluster.local`) so other clients can address a specific pod, not just "any pod behind the service."

3. **Per-pod PersistentVolumeClaim** — each pod has its own PV, created from `volumeClaimTemplates`. Pod `kafka-0`'s PV stays attached to `kafka-0` even if it's rescheduled to a different node. Deployments share PVs (ReadWriteMany) or use ephemeral storage.

Plus ordered, sequential startup and termination (configurable).

**When these matter — when you'd choose StatefulSet:**

- **Kafka brokers** — each broker has a unique ID; topic partitions are assigned to specific brokers. Re-IDing a broker requires data resync.
- **Elasticsearch / OpenSearch nodes** — shards are placed on specific nodes; nodes have stable identities.
- **Cassandra** — distributed ring; each node owns specific token ranges.
- **MongoDB replica sets** — primary/secondary roles; each member is uniquely identified.
- **Zookeeper / etcd** — consensus protocols depend on stable member identity.
- **Stateful databases on K8s** (Postgres, MySQL) — leader/follower roles, persistent data.
- **Distributed coordination services** (Consul, HashiCorp Vault).

**When you'd use Deployment + ReplicaSet (and StatefulSet would be overkill):**

- Any stateless service — REST APIs, web frontends, workers.
- Apps that share state via an external DB rather than per-pod storage.
- Apps where pods are interchangeable; load balancer doesn't need to know which is which.

**Answering "why I chose StatefulSet" specifically:**

> "In the case I'm thinking of, we ran Kafka on Kubernetes via Strimzi operator, which uses StatefulSets. The reason it had to be StatefulSet rather than a Deployment:
>
> 1. **Each Kafka broker has a unique ID** (broker.id) — that ID is in the metadata. If broker-2 is rescheduled to a new node and comes up as a different identity, the cluster sees it as a new broker, not the original one. The replicas wouldn't sync; data would be re-replicated. StatefulSet preserves the identity.
>
> 2. **Each broker has its own data directory** with topic partition data — 200GB of data per broker. We needed each pod to keep its own PV across restarts. StatefulSet's `volumeClaimTemplates` give us per-pod PVs that follow the pod identity.
>
> 3. **Ordered startup matters** — Zookeeper-coordinated bootstrapping needs broker-0 up before broker-1 joins. StatefulSet handles this naturally.
>
> 4. **Other services connect to specific brokers** — the Kafka client protocol involves direct broker connections after initial discovery. Stable DNS names are required.
>
> Using a Deployment would have produced random pod names that change on each restart, lost the per-pod PV affinity (or required all pods to share one PV via ReadWriteMany, which Kafka doesn't support), and broken the ordered-restart guarantee. StatefulSet was the only correct primitive."

---

## 9. Deploying a new image version with Helm — what changes before deployment

The expected changes before a deploy:

**1. Image tag bump** — the actual change driving the deploy
```yaml
# values-prod.yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
  tag: 1.4.3-a1b2c3d-build2087     # bumped from 1.4.2-...
  pullPolicy: IfNotPresent
```

**2. Chart version bump** — if the chart itself changed
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.4.3        # chart semver — bumped for chart changes
appVersion: 1.4.3     # tracks app version
```

**3. ConfigMap changes** — if the new image needs new config
```yaml
config:
  newFeatureFlag: true
  databasePoolSize: 50    # changed for v1.4.3
```

**4. Resource adjustments** — if profiling showed v1.4.3 needs different resources
```yaml
resources:
  requests: { cpu: 300m, memory: 768Mi }    # was 250m / 512Mi
  limits:   { cpu: 1500m, memory: 1.5Gi }
```

**5. Health check path changes** — if the new image moved `/health`
```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: http }
```

**6. Database migrations** — if v1.4.3 includes schema changes
```yaml
migrationJob:
  enabled: true
  image:
    tag: 1.4.3-a1b2c3d-build2087
  # Configured as a pre-install / pre-upgrade hook
```

**7. Feature flags / rollout controls** — for gradual rollout
```yaml
rollout:
  strategy: canary
  steps:
    - setWeight: 10
    - pause: { duration: 5m }
    - setWeight: 50
    - pause: { duration: 10m }
    - setWeight: 100
```

**Pre-deploy verification steps:**

```bash
# 1. Validate the chart renders correctly
helm template myapp ./chart -f values-prod.yaml | kubeconform -strict

# 2. Diff against currently deployed
helm diff upgrade myapp ./chart -f values-prod.yaml

# 3. Dry-run
helm upgrade --install myapp ./chart -f values-prod.yaml --dry-run

# 4. Linter
helm lint ./chart

# 5. Final apply
helm upgrade --install myapp ./chart \
  -f values-prod.yaml \
  --atomic --wait --timeout 10m
```

**In a GitOps model:** I don't run helm directly. The change is committed to the GitOps repo (the values file) and ArgoCD reconciles. The "change before deployment" is a PR with the values diff, reviewed and merged.

**Architect's framing:**
> "The actual change is usually a single line — `tag: 1.4.3-...`. The rigor around it is what makes it production-safe: chart version bump for traceability, `helm diff` for review, atomic apply for automatic rollback on failure, and migrations gated through pre-install hooks. In a GitOps model, this all becomes a PR — no human runs `helm upgrade` against prod directly."

---

## 10. Where is Helm used in the pipeline, what code, what's its contribution at build time?

**Where Helm sits in the pipeline:**

In a typical pipeline:

```
Build (CI)                          → Helm not involved
  Compile, test, SAST, image scan
  Build Docker image
  Push to ECR

Package (CI)                        → Helm packaging happens here
  Lint Helm chart
  Package Helm chart (helm package)
  Push chart to Helm registry (ChartMuseum, ECR OCI, Artifactory)

Deploy (CD)                         → Helm rendering and applying
  Helm upgrade --install (or ArgoCD does it via Helm)
```

**What code/commands are used:**

**At build/package time (CI):**
```yaml
# .github/workflows/ci.yml
- name: Helm lint
  run: helm lint ./chart

- name: Helm template (verify renders)
  run: |
    helm template myapp ./chart \
      -f chart/values-prod.yaml \
      --set image.tag=${{ env.IMAGE_TAG }} \
      | kubeconform -strict

- name: Package chart
  run: |
    helm package ./chart \
      --version ${{ env.CHART_VERSION }} \
      --app-version ${{ env.APP_VERSION }} \
      --destination ./packaged

- name: Push chart to OCI registry
  run: |
    helm push ./packaged/myapp-${{ env.CHART_VERSION }}.tgz \
      oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts
```

**At deploy time (CD):**

Either Helm CLI directly:
```bash
helm upgrade --install myapp ./chart \
  -f values-prod.yaml \
  --set image.tag=$IMAGE_TAG \
  --atomic --wait --timeout 10m
```

Or via ArgoCD's Helm support in an Application resource:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts
    chart: myapp
    targetRevision: 1.4.3
    helm:
      valueFiles: [values-prod.yaml]
      parameters:
      - name: image.tag
        value: 1.4.3-a1b2c3d
```

**Helm's contribution at build time:**

- **Validation** — `helm lint` catches syntax errors, missing required fields, schema violations before they reach the cluster.
- **Template rendering** — `helm template` produces final manifests that can be validated with `kubeconform`, scanned by `checkov`/`kubesec`, and diffed against the deployed state.
- **Packaging** — turns the chart directory into a versioned `.tgz` artifact, stamped with metadata (version, dependencies, signing). This artifact is the immutable, versioned deployment unit.
- **Pushing to a registry** — promotes the chart to the place CD will pull from. Same pattern as Docker images.
- **Dependency resolution** — `helm dependency update` pulls subchart versions referenced in `Chart.yaml`.

**At deploy time, Helm:**

- Renders templates with environment-specific values.
- Computes the diff against the cluster's existing release.
- Applies the diff in order (CRDs first, then namespaces, then everything else).
- Tracks the release in `secrets/configmaps` for history and rollback.
- Runs lifecycle hooks (`pre-install`, `pre-upgrade`, `post-install`) — useful for DB migrations.

**Architect's framing:**
> "Helm is doing two distinct jobs: it's a *template engine* for Kubernetes YAML and a *package manager* for versioned releases. At build time we use the package manager role — lint, render, package, push. At deploy time we use both roles — Helm renders with environment values and tracks the release for rollback. The build-time contribution is making the chart itself a reviewable, versioned artifact, not just templates evaluated at deploy time against an arbitrary state."

---

## 11. What happens behind the scenes during rollback?

**`helm rollback myapp 1`** triggers this sequence:

**1. Look up the target revision**

Helm stores every release as a Secret (or ConfigMap, depending on storage backend) in the namespace:
```bash
kubectl get secrets -n production | grep sh.helm.release
# sh.helm.release.v1.myapp.v1
# sh.helm.release.v1.myapp.v2
# sh.helm.release.v1.myapp.v3       ← current
```

Each revision's Secret contains the rendered manifests, values, chart, and metadata from when that revision was applied.

**2. Read the target revision's manifests**

Helm decompresses the Secret and reads the manifests as they existed at revision 1.

**3. Compute the diff between current cluster state and target manifests**

Helm three-way merges:
- Current live state (from the cluster).
- Previous revision's last-applied state.
- Target revision's manifests.

This produces the set of operations to bring the cluster from current to target state.

**4. Apply the operations**

In dependency order:
- CRDs (if changed).
- Namespaces.
- ConfigMaps, Secrets.
- PersistentVolumeClaims.
- Deployments, StatefulSets, DaemonSets — *these are updated, not destroyed and recreated*.
- Services, Ingresses.
- HPAs, PDBs.

**5. For Deployments specifically:**

Helm patches the Deployment manifest to match revision 1's spec. The Deployment controller sees the change and:
- Computes a new pod template hash.
- Creates a new ReplicaSet (or selects an existing one — see Q12).
- Begins rolling update: scales up the rolled-back ReplicaSet, scales down the current one, per `maxSurge`/`maxUnavailable`.
- Each new pod starts, passes readiness probe, joins the Service endpoints.
- Old pods get SIGTERM, run preStop hooks, drain, exit.

**6. Run rollback hooks**

If the chart defines `pre-rollback` or `post-rollback` hooks, those run at the appropriate time:
```yaml
# templates/migration-rollback-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    "helm.sh/hook": pre-rollback
    "helm.sh/hook-weight": "-1"
spec:
  # ... reverse migration
```

**7. Update Helm release history**

A new revision is recorded (revision N+1, not "revert to 1"). Helm history shows:
```
REVISION  UPDATED                   STATUS      CHART       APP VERSION  DESCRIPTION
1         2025-10-15 09:00:00       superseded  myapp-1.4.1 1.4.1        Install complete
2         2025-10-20 14:30:00       superseded  myapp-1.4.2 1.4.2        Upgrade complete
3         2025-10-22 11:15:00       superseded  myapp-1.4.3 1.4.3        Upgrade complete
4         2025-10-22 11:45:00       deployed    myapp-1.4.1 1.4.1        Rollback to 1
```

The previous "current" revision (3) becomes `superseded`. Revision 4 is the rollback itself.

**8. What's NOT rolled back automatically**

- **PVC data** — Helm doesn't restore data; the volume content is whatever it currently is.
- **Database schema changes** applied by migrations — Helm doesn't reverse SQL migrations. The pre-rollback hook would handle this if defined; otherwise schema is whatever the current app version expects.
- **External state** — anything in S3, Kafka, RDS that the app produced under v3.

**Architect's framing:**
> "Behind the scenes, rollback is just another `apply` with the manifests from a previous revision. It's not magic — Helm stored what each revision looked like, so it can reconstruct that state. The two things people miss: rollback creates a *new* revision in Helm history rather than deleting the current one (so you can roll forward again easily), and rollback doesn't touch data — if your v1.4.3 wrote bad data, rolling the app back to v1.4.1 doesn't fix the data. That's why expand-contract migrations and backwards-compatible API changes matter more than rollback mechanics."

---

## 12. During rollback, is a new ReplicaSet created, and does it also roll back?

This is a precise Kubernetes mechanics question.

**The short answer:** It depends on whether the rolled-back pod template has been seen before.

**The Deployment controller's logic:**

When a Deployment's pod template (`spec.template`) changes, the controller:

1. Computes the hash of the pod template (`pod-template-hash`).
2. Looks for an existing ReplicaSet with that hash.
3. If found: scales the existing ReplicaSet back up.
4. If not found: creates a new ReplicaSet with that hash and scales it up.

**Scenario A: Rolling back to a recent version (within revision history)**

```bash
kubectl get rs -l app=myapp
# myapp-7d4f8b9c8c   3   3   3   2h   (current)
# myapp-5b8d6c4f5d   0   0   0   1d   (previous, scaled to 0)
# myapp-3a2e1d7b6c   0   0   0   3d   (older, scaled to 0)
```

The ReplicaSets for previous revisions are *not deleted* — they're scaled to 0 and retained based on `spec.revisionHistoryLimit` (default 10).

**Rolling back via `kubectl rollout undo`:**
- The Deployment controller finds the previous ReplicaSet (already exists with the right pod template hash).
- Scales it up.
- Scales the current ReplicaSet down.
- **No new ReplicaSet is created.**

```bash
kubectl rollout undo deployment/myapp
kubectl get rs -l app=myapp
# myapp-7d4f8b9c8c   0   0   0   2h   (scaled down)
# myapp-5b8d6c4f5d   3   3   3   1d   (scaled back up — same RS, now active)
```

**Scenario B: Rolling back via Helm**

Helm's rollback re-applies the manifests as they were in the target revision. The Deployment manifest gets patched. The Deployment controller then:
- Computes the hash of the rolled-back pod template.
- If it matches an existing ReplicaSet (the original one for that revision): scales that RS up. **No new RS.**
- If it doesn't match (because, e.g., that RS was garbage-collected per `revisionHistoryLimit`): creates a new RS with the same pod template.

In practice, recent rollbacks (within `revisionHistoryLimit`) reuse the existing ReplicaSet. Older rollbacks create new ones.

**What happens to pods during the rollback rollout:**

Standard rolling update mechanics:
- New pods (from the rolled-back-to RS) come up one by one.
- Each waits for readiness probe to pass.
- Old pods (current RS) get SIGTERM in order.
- Service endpoints update as pods become Ready / NotReady.
- Per `maxSurge`/`maxUnavailable` in the Deployment's update strategy.

So yes — the *pods* roll over via rolling update during a rollback, but the *ReplicaSets* are typically reused (not new ones).

**`revisionHistoryLimit` matters:**
```yaml
spec:
  revisionHistoryLimit: 10    # keep last 10 RSes for rollback
```

Setting this to 0 means no rollback history; old RSes are deleted immediately. Setting it too high uses extra cluster resources tracking ancient ReplicaSets.

**Architect's framing:**
> "The Deployment controller uses the pod template hash as the ReplicaSet identity. Rolling back to a recent revision usually reuses the existing ReplicaSet — Kubernetes just scales it back up while scaling the current one down, doing a rolling update of pods between them. A new ReplicaSet is created only if the rolled-back-to template doesn't match any retained RS, which happens when the rollback target is beyond `revisionHistoryLimit`. The pods always do a rolling transition; the ReplicaSets are usually reused. This is why `kubectl rollout undo` is so fast — there's no new image pull, no new RS creation, just a flip in scale numbers."

---

## 13. Effect of too many lines in a Dockerfile

**Each instruction creates a layer.** More instructions → more layers → larger image, slower builds, more cache complexity, more attack surface.

**Specific impacts:**

**1. Larger image size**
Every layer adds metadata overhead and stored deltas. Even when individual instructions don't write much, the layer manifest grows. A 100-instruction Dockerfile can produce an image with significant overhead from layer metadata alone.

More critically: layers are *additive*. Removing a file in a later layer doesn't shrink the image — the file is still in the earlier layer's filesystem snapshot.

```dockerfile
# BAD — secret remains in image even though it's deleted
RUN curl -o secret.tar https://example.com/secret -H "Auth: $TOKEN"
RUN tar -xf secret.tar
RUN rm secret.tar    # ← secret.tar is still in the earlier layer
```

**2. Slower builds**
- Each layer has setup/teardown overhead.
- More layers = more cache lookups.
- A 50-step Dockerfile rebuilds slower than a 10-step equivalent doing the same work.

**3. Worse cache utilization**
- More layers = more cache invalidation surface.
- A change to step 20 invalidates steps 21-50.
- Fewer well-organized layers = better cache hits.

**4. Push/pull performance**
- Each layer is fetched/pushed separately.
- Many small layers have more HTTP overhead than fewer large layers.
- Registry costs grow with layer count and total size.

**5. Increased attack surface**
- Each layer can contain forgotten artifacts.
- Build secrets accidentally written to layers stay there even if "removed" later.
- Vulnerability scanners report findings per layer; more layers = more noise.

**6. Layer count limit**
- Older Docker (overlay2 driver): 125 layer limit.
- Modern Docker: no hard limit but performance degrades.

**How to reduce:**

**Combine related RUN commands:**
```dockerfile
# BAD — three layers, apt cache left in image
RUN apt-get update
RUN apt-get install -y curl jq
RUN rm -rf /var/lib/apt/lists/*

# GOOD — one layer, no leftover cache
RUN apt-get update && \
    apt-get install -y curl jq && \
    rm -rf /var/lib/apt/lists/*
```

**Use multi-stage builds** — discard everything you don't need in the final stage:
```dockerfile
FROM maven:3.9 AS build
# ... 30 lines of build setup, doesn't matter — discarded
RUN mvn package

FROM eclipse-temurin:17-jre-alpine
COPY --from=build /build/target/app.jar /app/app.jar
ENTRYPOINT ["java","-jar","/app/app.jar"]
# Final image has only what it needs
```

**Use `.dockerignore`** to keep the build context small.

**Architect's framing:**
> "Layer count is a proxy for discipline. A 50-line Dockerfile usually means someone added instructions over time without consolidating, leaving artifacts in earlier layers that bloat the image and create security risk. The fix is multi-stage builds and combining related RUN commands. The mental model: each RUN should be one atomic unit of work that cleans up after itself within the same layer."

---

## 14. Copy-on-write layer

**Copy-on-write (CoW)** is the foundational mechanism of container filesystems and how Docker images work efficiently.

**The model:**

Container images are stacks of immutable read-only layers. When a container runs:
- All image layers are mounted read-only.
- A new **writable layer** is added on top — the "container layer" or "copy-on-write layer."
- The union of these layers is what the container sees as its filesystem.

When the container modifies a file:
1. The file is searched through the layer stack.
2. If found in a read-only layer, it's **copied up** to the writable layer.
3. The modification is made to the copy.
4. Subsequent reads return the writable layer's copy.

This is the "copy on write" part — files are only copied when written to, not when read.

**Implications:**

**1. Image storage efficiency**
- Multiple containers from the same image share read-only layers.
- 100 containers from `nginx:latest` consume only one copy of the image layers + 100 small writable layers (only modifications).

**2. Container startup speed**
- No copying of image content at start time — just mount the existing read-only layers and add a writable layer.
- Starting a container is essentially "create an empty writable layer and start the process."

**3. Layer reuse across images**
- If two images share base layers (`ubuntu:22.04`), those layers are stored once.
- Pulling an image only downloads layers not already cached.

**4. Write performance**
- The first write to a file in a read-only layer is slower (copy-up).
- Subsequent writes are normal speed.
- For write-heavy containers, mount volumes (bypass CoW) for the busy paths.

**5. The writable layer dies with the container**
- Modifications in the CoW layer are *not* persisted.
- Removing a container removes the CoW layer.
- For persistence, use volumes — they bypass the CoW filesystem.

**Storage drivers that implement CoW:**

| Driver | Approach |
|---|---|
| **overlay2** | Current default. Union mount of lower (read-only) and upper (writable) directories. File-level CoW. |
| **devicemapper** | Block-level CoW (deprecated). |
| **btrfs / zfs** | Filesystem-native CoW snapshots. |
| **aufs** | Older union filesystem (deprecated). |

**Practical visualization:**
```
Container's view (what the process sees):
+--------------------------------------+
| Writable layer (CoW, ephemeral)      |
+--------------------------------------+
| Image layer: app source              |  ←  read-only
+--------------------------------------+
| Image layer: dependencies            |  ←  read-only
+--------------------------------------+
| Image layer: OS packages             |  ←  read-only
+--------------------------------------+
| Image layer: base OS (alpine, etc.)  |  ←  read-only
+--------------------------------------+
```

**Architect's framing:**
> "CoW is why containers are operationally cheap — fast startup, low storage overhead, efficient image distribution. It's also why you should never store important data in the container filesystem — it lives in the ephemeral CoW layer and disappears with the container. The CoW model also explains why a `rm secret.tar` after the file was added in an earlier layer doesn't reduce image size: the earlier layer is read-only and immutable; the `rm` just adds a 'whiteout' marker in a later layer hiding the file from view, but the underlying layer still contains it."

---

## 15. Docker networks I've worked with

Docker provides several network drivers. The ones I've used in production and lab:

**1. Bridge** — Docker's default network driver.
- Creates a private internal network on the host.
- Containers get IPs in the bridge subnet (default 172.17.0.0/16).
- Containers communicate via the bridge; outbound to internet via host NAT.
- Containers can resolve each other by container name (DNS) when on a user-defined bridge.

```bash
docker network create my-bridge
docker run -d --network my-bridge --name app myapp
docker run -d --network my-bridge --name db postgres
# app can reach db via the hostname "db"
```

Used for: local dev, Docker Compose, single-host multi-container setups.

**2. Host** — container shares the host's network namespace.
- No network isolation.
- Container uses host IP directly; binding to port 80 binds to host's port 80.
- Best performance (no NAT, no virtual bridge).

```bash
docker run --network host nginx
```

Used for: performance-sensitive workloads where network isolation isn't needed; system services. Caution: port conflicts with the host, no namespace isolation.

**3. None** — no network at all.
- Container has only a loopback interface.
- Useful for batch jobs that don't need network.

```bash
docker run --network none my-batch-job
```

**4. Overlay** — multi-host network for Docker Swarm or third-party orchestrators.
- Spans multiple Docker hosts.
- VXLAN encapsulation for inter-host pod-like communication.
- Mostly used with Swarm; for Kubernetes, the CNI plugins serve this role.

**5. Macvlan** — container gets its own MAC address, appears as a physical device on the network.
- Container has a real IP on the parent network.
- Bypasses Docker's NAT; container is directly addressable from the LAN.

Used for: legacy apps that need a real IP, network monitoring tools, special networking requirements.

**6. ipvlan** — similar to macvlan but containers share the parent's MAC, differentiated by IP.
- Better for environments with MAC address limits (some switches).

**In Kubernetes context** — the Docker network model is mostly replaced by **CNI (Container Network Interface)** plugins:

- **AWS VPC CNI** — each pod gets a VPC IP from the same subnet as the node. Pods are first-class VPC citizens.
- **Calico** — overlay (VXLAN/IPIP) or BGP-routed networking, strong NetworkPolicy enforcement.
- **Cilium** — eBPF-based, includes L7 policy, observability via Hubble.
- **Flannel** — simple overlay, less common now.
- **Weave Net** — overlay, declining usage.

**Docker Compose internal patterns I've used:**

```yaml
# docker-compose.yml
networks:
  frontend:    # public-facing
  backend:     # internal services
  database:    # data tier only

services:
  app:
    networks: [frontend, backend]
  db:
    networks: [database, backend]      # only app + db can talk
  cache:
    networks: [backend]
```

This three-tier-on-one-host pattern is common for local dev that mirrors production architecture.

**Architect's framing:**
> "In production on Kubernetes, the relevant question is CNI, not Docker network drivers — AWS VPC CNI for tight VPC integration, Cilium for eBPF-based L7 policies. For local dev and CI, user-defined bridge networks via Docker Compose serve the purpose. I've used host network for diagnostic tools, macvlan rarely for legacy apps that need a real LAN IP. The decision is: how much isolation do I need, what address space do I want pods/containers on, and what policy enforcement do I need at L3/L4/L7."

---

## 16. How to decrease Docker image size

Layered approach, biggest wins first:

**1. Multi-stage builds — by far the biggest single win**

```dockerfile
# Build stage — has compilers, build tools, source
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app

# Final stage — only the binary
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

A Go app: build image ~900 MB, final image ~20 MB. Massive reduction.

**2. Use minimal base images**

| Base | Size | Trade-off |
|---|---|---|
| `ubuntu:22.04` | ~77 MB | Familiar but bloated |
| `debian:bookworm-slim` | ~30 MB | Reasonable middle ground |
| `alpine:3.20` | ~5 MB | musl libc (gotchas with Python, glibc-linked binaries) |
| `gcr.io/distroless/static` | ~2 MB | No shell, no package manager — best for static binaries |
| `scratch` | 0 bytes | For statically-linked binaries only |

**3. Combine RUN commands, clean up in the same layer**

```dockerfile
# BAD
RUN apt-get update
RUN apt-get install -y python3 python3-pip
RUN pip install requests
# apt cache is in a layer; can't be removed by later RM

# GOOD
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip && \
    pip install --no-cache-dir requests && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**4. Don't install build tools in the runtime image**

Common offenders to remove from runtime images: `git`, `gcc`, `make`, `python3-dev`, `build-essential`. These belong in the build stage only.

**5. Use `.dockerignore`**

```
.git
.github
node_modules
target
dist
*.log
.env*
Dockerfile
README.md
*.md
.vscode
.idea
```

Without this, the entire `.git` directory (potentially hundreds of MB) becomes part of the build context.

**6. Choose package install flags wisely**

```dockerfile
# Don't install recommended/suggested packages
RUN apt-get install -y --no-install-recommends ...

# pip: don't keep wheel cache
RUN pip install --no-cache-dir ...

# npm: production deps only, clean cache
RUN npm ci --omit=dev && npm cache clean --force
```

**7. Use BuildKit cache mounts for package managers**

```dockerfile
# syntax=docker/dockerfile:1.6
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

Cache is stored separately, not baked into the image layer.

**8. Squash layers if all else fails** (use sparingly)

```bash
docker build --squash -t myapp:1.0 .
```

Squashes the final image into one layer. Trade-off: loses layer caching benefits. Only useful for final distribution image, not intermediate work.

**9. Profile what's in the image**

```bash
# Show layer sizes
docker history myapp:1.0

# Interactive layer-by-layer browser
dive myapp:1.0

# Detailed breakdown
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive myapp:1.0
```

Dive shows which files are in which layer and the wasted-space score — invaluable for finding what's bloating an image.

**10. Compress assets**

```dockerfile
# Pre-gzip static assets so nginx can serve them directly
RUN find /usr/share/nginx/html -type f -regex ".*\.\(html\|css\|js\)$" -exec gzip -9 -k {} \;
```

**Realistic before/after I've seen:**

| Workload | Before | After | Approach |
|---|---|---|---|
| Java Spring Boot | 850 MB | 240 MB | Multi-stage, JRE-alpine, dependency layer separation |
| Node.js API | 1.2 GB | 180 MB | Multi-stage, alpine, deps cleanup, no devDependencies |
| Python Flask | 1 GB | 110 MB | Multi-stage with venv copy, slim base, no caches |
| Go service | 900 MB | 18 MB | Multi-stage, distroless static |

**Architect's framing:**
> "Image size optimization is essentially: only ship what's needed to run. Multi-stage builds enforce this by construction — you literally can't accidentally include build tools because they're in a discarded stage. After that, base image choice and combined RUN commands handle most of the remaining bloat. I aim for under 250MB for JVM apps, under 100MB for Node/Python, and under 50MB for Go — and I use `dive` to verify there's no surprise waste."

---

## 17. Benefits of storing Docker images in ECR

**Where I store images depends on the cloud and workload:**

**Primary registries I've used:**

| Registry | When |
|---|---|
| **AWS ECR** | AWS workloads (EKS, ECS, Fargate, Lambda containers) |
| **Azure Container Registry (ACR)** | Azure workloads (AKS, App Service, Container Apps) |
| **Google Artifact Registry (formerly GCR)** | GCP workloads |
| **JFrog Artifactory** | Multi-cloud, enterprise with unified artifact strategy |
| **Docker Hub** | Public OSS images we publish |
| **Harbor** | On-prem or self-hosted enterprise |

**ECR benefits specifically:**

**1. Tight IAM integration**
- Pull permissions via IAM roles, not username/password.
- EKS pods authenticate via IRSA — no `imagePullSecrets` to manage.
- ECR repo policies for cross-account access without static credentials.

**2. Private by default**
- All ECR repos start private; no anonymous pull.
- ECR Public is a separate service for public OSS images.

**3. Built-in vulnerability scanning**
- **Basic scanning** (free): scans on push, uses Clair CVE database.
- **Enhanced scanning** (paid, via Amazon Inspector): continuous scanning of stored images, OS + language packages, deeper CVE detection.

**4. Encryption at rest**
- AES-256 by default; customer-managed KMS keys supported.
- Encryption in transit via TLS.

**5. Replication and lifecycle**
- **Cross-region replication** — push to one region, image auto-replicated to others. Critical for multi-region DR.
- **Lifecycle policies** — auto-delete old/untagged images by age or count.

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Delete untagged images after 14 days",
    "selection": { "tagStatus": "untagged", "countType": "sinceImagePushed", "countUnit": "days", "countNumber": 14 },
    "action": { "type": "expire" }
  }]
}
```

**6. Tag immutability**
- Per-repo setting: when enabled, tags can't be overwritten. Production tags are protected from `latest`-style accidental updates.

**7. Pull-through cache**
- ECR can proxy and cache images from Docker Hub, ECR Public, Quay, GitHub Container Registry. Reduces dependency on external registries and avoids rate limits.

**8. Performance from AWS workloads**
- Pulls from EC2/EKS/ECS in the same region are over AWS's internal network — fast, no internet egress charges.
- VPC endpoints (`com.amazonaws.region.ecr.api` and `com.amazonaws.region.ecr.dkr`) for fully private pulls.

**9. OCI artifact support**
- ECR stores not just container images but also Helm charts (as OCI artifacts), allowing one registry for all artifacts.

**10. Cost predictability**
- Pricing: ~$0.10 per GB-month storage, free data transfer to AWS workloads in the same region. Predictable; no per-pull charges within AWS.

**11. Audit and governance**
- CloudTrail logs every push/pull/delete.
- Repository policies grant fine-grained access.
- Resource-level IAM for which repos which users/services can read or write.

**12. Multi-architecture image support**
- Manifest lists for amd64 + arm64 images under a single tag (good for Graviton workloads).

**Decision framework: where would I store?**

- **AWS-primary, EKS workloads**: ECR. Tight integration is worth it.
- **Multi-cloud**: Artifactory or Harbor as a unified registry, replicating to cloud-native registries (ECR, ACR) for performance.
- **OSS we publish**: Docker Hub + GitHub Container Registry for community reach.
- **On-prem regulated**: Harbor with image signing (Cosign), behind the corporate firewall.

**Architect's framing:**
> "For an AWS workload, ECR is the obvious answer — IAM-native auth, private by default, replication for DR, scanning built in, and best performance from AWS compute. The decision becomes interesting in multi-cloud setups where unified governance matters more than per-cloud optimization. In those cases, Artifactory or Harbor as the source of truth, with replication to ECR/ACR/GCR for runtime performance, is the pattern I'd use."

---

## 18. How many stages in Jenkins?

This question is asking me to describe a real pipeline structure, not a number. The honest answer:

> "It varies by project, but a representative production pipeline has 8-12 stages. Here's a typical Jenkinsfile structure:"

```groovy
@Library('platform-shared-lib@v2.0.0') _

pipeline {
  agent { label 'docker-builder' }

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '50'))
  }

  environment {
    REGISTRY = "123456789012.dkr.ecr.us-east-1.amazonaws.com"
    APP      = "myapp"
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.GIT_COMMIT.take(7)}-${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Validate') {
      parallel {
        stage('Lint') {
          steps { sh 'mvn -B checkstyle:check' }
        }
        stage('Dependency Audit') {
          steps { sh 'mvn -B org.owasp:dependency-check-maven:check' }
        }
      }
    }

    stage('Build & Unit Test') {
      steps {
        sh 'mvn -B clean verify'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
          publishCoverage adapters: [jacocoAdapter('target/site/jacoco/jacoco.xml')]
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar-prod') { sh 'mvn sonar:sonar' }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Image') {
      steps {
        sh "docker build -t ${REGISTRY}/${APP}:${IMAGE_TAG} ."
      }
    }

    stage('Image Scan') {
      steps {
        sh "trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed ${REGISTRY}/${APP}:${IMAGE_TAG}"
      }
    }

    stage('Sign Image') {
      steps {
        sh "cosign sign --yes ${REGISTRY}/${APP}:${IMAGE_TAG}"
      }
    }

    stage('Push to ECR') {
      steps {
        withAWS(role: 'arn:aws:iam::123:role/jenkins-ecr-push', region: 'us-east-1') {
          sh "aws ecr get-login-password | docker login --username AWS --password-stdin ${REGISTRY}"
          sh "docker push ${REGISTRY}/${APP}:${IMAGE_TAG}"
        }
      }
    }

    stage('Update GitOps Repo') {
      when { branch 'develop' }
      steps {
        bumpGitOpsImageTag(env: 'dev', app: env.APP, tag: env.IMAGE_TAG)
      }
    }

    stage('Smoke Test') {
      when { branch 'develop' }
      steps {
        runSmokeTests(env: 'dev')
      }
    }

    stage('Approval — Production') {
      when { branch 'main' }
      steps {
        timeout(time: 24, unit: 'HOURS') {
          input message: 'Promote to production?', submitter: 'release-managers'
        }
      }
    }

    stage('Deploy Production') {
      when { branch 'main' }
      steps {
        bumpGitOpsImageTag(env: 'prod', app: env.APP, tag: env.IMAGE_TAG)
      }
    }
  }

  post {
    always { cleanWs() }
    success { notifySlack(channel: '#deploys', status: 'success') }
    failure { notifySlack(channel: '#deploys', status: 'failure') }
  }
}
```

**The 12 stages above, summarized:**

1. **Checkout** — clone the repo.
2. **Validate (parallel)** — lint + dependency audit.
3. **Build & Unit Test** — compile, run tests, publish results.
4. **SonarQube Analysis** — code quality + SAST.
5. **Quality Gate** — block on Sonar gate failure.
6. **Build Image** — docker build.
7. **Image Scan** — Trivy CVE check.
8. **Sign Image** — Cosign signature.
9. **Push to ECR** — ship the artifact.
10. **Update GitOps Repo** — bump dev tag (auto on develop).
11. **Smoke Test** — verify dev deploy works.
12. **Approval + Deploy Prod** — gated production promotion.

**Principles I'd articulate:**

- **Parallel where possible** — lint + dependency audit don't need to be sequential.
- **Fail fast** — Quality Gate before image build; no point building a Docker image if SonarQube already said the code's broken.
- **Build once, deploy many** — the same image artifact deploys to dev, staging, prod. No rebuilds.
- **GitOps separation** — Jenkins commits to a GitOps repo; ArgoCD does the actual cluster deploy. Jenkins doesn't have prod cluster credentials.
- **Approval gate for prod** — `input` step with submitter restriction.
- **Shared library** (`@Library`) — common steps (`bumpGitOpsImageTag`, `runSmokeTests`, `notifySlack`) live in a versioned library, not copy-pasted across 50 Jenkinsfiles.

**The honest interview answer:**
> "How many stages depends on what's needed for the project — I've seen everything from 3 stages (build, test, deploy) for an internal tool to 20+ stages for a regulated workload with multiple security gates, compliance checks, and multi-region promotion. The number isn't the goal; the right granularity is the goal. Each stage should represent a logical unit that can fail independently with a clear failure reason. If a stage does too much, failures are hard to diagnose; if you have too many stages, you're paying setup overhead repeatedly. For a typical microservice production pipeline, 8-12 stages is the right neighborhood."

---

That covers all 18 questions. Likely follow-ups based on these:

- **"You mentioned Strimzi for Kafka — how do you handle a Strimzi upgrade?"** — operator upgrade first, then Kafka upgrade via the CR, rolling restart with monitoring.
- **"Show me a `Chart.yaml` and how chart version differs from app version."** — chart version bumps on chart changes; appVersion tracks the actual application version.
- **"How do you decide between Helm and Kustomize?"** — Helm for templated packages with values, Kustomize for declarative overlays. Often both — Helm-render the chart, Kustomize-overlay environment specifics.
- **"What's in your shared Jenkins library?"** — common deployment steps, notification handlers, secret-fetching wrappers, standard build-test-package flow for each language stack.


# DevOps Interview — Complete Answers

Going through each with the depth a senior engineer should bring.

---

## 🐧 Linux

### Finding a file by size

```bash
# Exactly 100 MB
find /path -type f -size 100M

# Larger than 100 MB
find /path -type f -size +100M

# Smaller than 50 MB
find /path -type f -size -50M

# Between 100 MB and 1 GB
find /path -type f -size +100M -size -1G

# Print human-readable size alongside
find /path -type f -size +100M -exec ls -lh {} \;

# Top 20 biggest files anywhere
find / -type f -size +100M 2>/dev/null -exec du -h {} + | sort -hr | head -20

# Modern alternative — fd (much faster, respects .gitignore)
fd --type f --size +100m
```

Size suffixes: `c` (bytes), `k` (KB), `M` (MB), `G` (GB). `find` rounds up, so `-size 1M` actually means files in the range (0, 1MB].

For directory-level usage, `du -ah /path | sort -hr | head -20` or interactive `ncdu /path` is usually what you actually want.

### Searching by name

```bash
# Exact name, recursive
find /path -name "config.yaml"

# Case-insensitive
find /path -iname "*.LOG"

# Wildcards (quote them so the shell doesn't expand them first)
find / -name "*.conf" 2>/dev/null

# Faster — uses a prebuilt index (may be slightly stale)
locate config.yaml
sudo updatedb     # refresh the index

# Modern alternatives
fd config.yaml              # respects .gitignore, fast, smart defaults
rg --files | rg config       # ripgrep listing files

# Grep inside files (different question, but often what people actually want)
grep -r "DATABASE_URL" /etc
rg "DATABASE_URL" /etc       # much faster than grep -r
```

For interactive fuzzy finding, `fzf` piped from `find` or `fd` is unbeatable for daily work.

### Changing file permissions

```bash
# Symbolic notation
chmod u+x script.sh           # add execute for owner
chmod g-w file.txt            # remove write for group
chmod o=r file.txt            # set "others" to read-only
chmod a+r,a-w file.txt        # multiple changes at once

# Octal notation (what I use most)
chmod 755 script.sh           # rwxr-xr-x — common for executables
chmod 644 file.txt            # rw-r--r-- — common for config files
chmod 600 ~/.ssh/id_rsa       # rw------- — required for SSH private keys
chmod 700 ~/.ssh              # rwx------ — required for the SSH directory

# Recursive
chmod -R 750 /var/www

# Recursive but separate for files vs directories (often correct approach)
find /var/www -type d -exec chmod 750 {} \;
find /var/www -type f -exec chmod 640 {} \;

# Ownership is a separate command
chown user:group file.txt
chown -R www-data:www-data /var/www
```

Octal mapping: `r=4, w=2, x=1`. `755` = `7(owner: rwx) 5(group: r-x) 5(others: r-x)`. SetUID/SetGID/sticky bits use a fourth digit (`4755`, `2755`, `1777` respectively).

Special bits I see most in production:
- `1777` on `/tmp` — sticky bit, only owner can delete their files.
- `2775` on shared directories — SetGID so new files inherit the group.
- `4755` on binaries needing root for specific operations (extremely rare; usually you want capabilities instead).

---

## 🔧 Terraform

### `terraform taint` vs `terraform untaint`

`terraform taint` marks a resource for forced recreation on the next apply — Terraform destroys and recreates it. `terraform untaint` removes that mark.

```bash
# Old syntax (deprecated since Terraform 0.15.2)
terraform taint aws_instance.web
terraform apply

# Modern equivalent
terraform apply -replace=aws_instance.web

# Or for many resources
terraform apply -replace=aws_instance.web -replace=aws_instance.db
```

When you'd use it:
- Resource exists but is in a bad state — corrupted EBS volume, drifted EC2 that refresh can't reconcile.
- A Kubernetes resource needs a clean recreate.
- Forcing replacement for testing purposes.

Important interview-level context: **`taint` and `untaint` are deprecated**. New code should use the `-replace` CLI flag or the `replace_triggered_by` lifecycle meta-argument. Mentioning the deprecation shows current awareness.

### `terraform refresh`

Updates Terraform's state file to match real-world cloud state by querying each tracked resource via the provider API and writing current attributes back.

```bash
terraform refresh    # standalone (deprecated)
```

**Modern usage:** `terraform plan` and `terraform apply` automatically refresh before running. The standalone `terraform refresh` command is deprecated since 1.0 in favor of:

```bash
terraform apply -refresh-only
```

This produces a reviewable plan of *only* the state-only changes (no resource modifications) so you can see what drifted before deciding what to do.

Used when:
- Resources changed outside Terraform and you want state to reflect reality before planning normal changes.
- After `terraform import` to ensure all attributes are fully reconciled.
- Sanity-checking state before a major operation.

### `init` vs `plan` vs `apply`

- **`terraform init`** — initializes the working directory: downloads providers, initializes the backend (where state lives), downloads modules, writes `.terraform.lock.hcl`. Run after cloning, after changing backend/providers/modules.
- **`terraform plan`** — computes the diff between desired state (code) and actual state (refreshed). Outputs additions (+), changes (~), and destroys (-). Does not modify anything.
- **`terraform apply`** — executes the plan. Walks the resource DAG in dependency order, calling provider CRUD operations, updates state incrementally as it goes.

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan      # apply the exact plan you reviewed
```

Applying a saved plan file is the production pattern — what you reviewed is what gets applied, no ambiguity.

A subtle but important detail: state is updated *incrementally* during apply, so a crash mid-apply leaves state reflecting what actually got created. This is why state corruption is rare in practice — Terraform is robust to interruptions.

### Storing state remotely

Remote backends are mandatory for any team setup. Common options:

**AWS — S3 + DynamoDB:**
```hcl
terraform {
  backend "s3" {
    bucket         = "company-tfstate"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tfstate-locks"   # state locking
    encrypt        = true
    kms_key_id     = "alias/tfstate"
  }
}
```

**Azure — Storage Account:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "companytfstate"
    container_name       = "tfstate"
    key                  = "prod/network.tfstate"
    use_azuread_auth     = true
  }
}
```

**GCS, Terraform Cloud, Consul** are other supported backends.

Hardening any backend:
- **Versioning** on the bucket so state is recoverable from corruption.
- **State locking** (DynamoDB for S3, blob lease for Azure, native for TF Cloud).
- **Encryption** at rest with a customer-managed KMS key.
- **Access restricted** to only the pipeline IAM role and a small SRE break-glass group.
- **Separate state per environment** — dev/staging/prod each in their own state file.

### Viewing state

```bash
# List all tracked resources
terraform state list

# Detailed attributes of a specific resource
terraform state show aws_instance.web

# Dump raw state (be careful — contains secrets in plaintext)
terraform state pull > current.tfstate
jq . current.tfstate | less

# Show declared outputs
terraform output
terraform output -raw load_balancer_dns
terraform output -json | jq .
```

Never edit the state file by hand. Use `terraform state mv`, `rm`, `replace-provider`, or `import` subcommands — they validate before writing.

### Avoiding sensitive value exposure during apply

```hcl
variable "db_password" {
  type      = string
  sensitive = true    # masks the value in plan/apply CLI output
}

output "db_connection_string" {
  value     = "postgres://admin:${var.db_password}@${aws_db_instance.this.endpoint}/mydb"
  sensitive = true    # output won't be printed; must use -raw to retrieve explicitly
}
```

What `sensitive = true` does: masks the value in CLI output during plan/apply. The value is **still stored in the state file in plaintext**. So:
- Encrypt state at rest (KMS-encrypted backend bucket).
- Restrict state access via IAM.
- Never `terraform output` secrets in CI logs.
- For provider-level secrets (e.g., a DB resource's `password`), most providers auto-mark them sensitive.

In CI, never `echo $TF_VAR_db_password` or similar — the variable's value will appear in logs.

### Handling secrets in Terraform

Layered approach, from worst to best:

1. **Hardcoded in `.tf` files** — never.
2. **`.tfvars` files gitignored** — acceptable for local dev only.
3. **Environment variables** (`TF_VAR_db_password`) — better, but the value still has to live somewhere.
4. **Reference from a secrets manager at runtime:**

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/rds/master-password"
}

resource "aws_db_instance" "this" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
  lifecycle {
    ignore_changes = [password]   # rotation outside TF doesn't trigger drift
  }
}
```

5. **Generate in Terraform, store in secrets manager:**

```hcl
resource "random_password" "db" { length = 32 }

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}
```

The password still ends up in state, but never in code or Git.

6. **Eliminate the secret entirely** — IAM database authentication for RDS, IRSA for EKS workloads, workload identity for GCP/Azure. The best secret is no secret.

State hardening matters regardless: encrypted backend with customer-managed KMS, restricted access, versioning enabled.

### Avoiding hardcoded values

- **Variables with sensible defaults**:
  ```hcl
  variable "instance_type" {
    type    = string
    default = "t3.medium"
  }
  ```

- **Per-environment `.tfvars` files**:
  ```bash
  terraform apply -var-file=prod.tfvars
  ```

- **Data sources** for values that exist elsewhere:
  ```hcl
  data "aws_ami" "amazon_linux" {
    most_recent = true
    owners      = ["amazon"]
    filter {
      name   = "name"
      values = ["amzn2-ami-hvm-*-x86_64-gp2"]
    }
  }
  # Use data.aws_ami.amazon_linux.id instead of pinning ami-0abc123
  ```

- **`locals`** for computed values and DRY:
  ```hcl
  locals {
    common_tags = {
      Environment = var.env
      Project     = var.project
      ManagedBy   = "terraform"
    }
  }
  ```

- **Modules** for repeated patterns; pass values as inputs.

- **Workspace-aware lookups**:
  ```hcl
  locals {
    instance_size = lookup({
      dev  = "t3.small"
      prod = "m5.large"
    }, terraform.workspace, "t3.small")
  }
  ```

Principle: anything that varies between environments goes into a variable; anything reused goes into a local or module.

### Importing existing cloud infrastructure

**Step 1 — Write the resource block matching the existing resource:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

**Step 2 — Import via CLI (classic):**
```bash
terraform import aws_vpc.main vpc-0abc123def456
```

**Step 2 (modern, Terraform 1.5+) — declarative `import` block:**
```hcl
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456"
}
```

**Step 3 — Generate config (Terraform 1.5+):**
```bash
terraform plan -generate-config-out=generated.tf
```

**Step 4 — Iterate:** run `terraform plan`. It almost always shows changes initially because your hand-written config doesn't match every actual attribute. Update the config until plan shows zero changes.

**Step 5 — Verify** with `terraform apply` (should be a no-op).

For bulk imports of brownfield estates, **terraformer** can scan a cloud and generate Terraform + state for hundreds of resources at once. Output requires cleanup but vastly faster than hand-writing each block.

Common gotchas:
- Default tags, default security groups, default route tables — these often exist and need handling.
- Order matters when there are dependencies; import VPC before subnets before instances.
- After import, plan may want to "change" attributes that AWS returns differently than Terraform expects (e.g., security group rule ordering). Use `lifecycle.ignore_changes` selectively.

---

## ☁️ AWS

### EKS vs ECS

| | EKS | ECS |
|---|---|---|
| Orchestrator | Kubernetes (upstream) | AWS-native proprietary |
| Portability | Kubernetes API — works elsewhere | AWS-only |
| Complexity | Higher | Lower |
| Ecosystem | Massive (Helm, Operators, CNCF) | Smaller, AWS-aligned |
| Compute | EC2, Fargate, hybrid | EC2, Fargate |
| Networking | VPC CNI, Calico, Cilium | AWS VPC mode (per-task ENI) |
| Pricing | $0.10/hour per cluster + nodes | Free control plane + nodes |
| Best for | Multi-cloud orgs, K8s expertise, complex workloads | AWS-only shops, simpler workloads, smaller teams |

**Quick decision rule:**
- If you need Kubernetes — Helm charts, custom operators, multi-cloud portability — use **EKS**.
- If you're AWS-only and want the simplest path to running containers, use **ECS Fargate**.
- For purely serverless containers with simple needs, ECS Fargate is genuinely operationally lighter than EKS.

The trade-off I'd articulate: **ECS is simpler at the cost of lock-in; EKS is portable at the cost of operational complexity**. The Kubernetes control plane charge ($73/month per cluster) and the operator burden of running Kubernetes are real costs that ECS doesn't have. EKS earns its keep when the ecosystem (operators, mesh, GitOps) provides value beyond what ECS-native tooling can.

### S3 lifecycle policy

Rules that automatically transition objects between storage classes or expire them after defined ages. Saves money by moving cold data to cheaper tiers.

```json
{
  "Rules": [{
    "Id": "archive-and-expire",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER_IR" },
      { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "Expiration": { "Days": 2555 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
  }]
}
```

Common use cases:
- Log archival with tiered transitions over time.
- Backup retention with eventual expiration.
- Compliance-driven retention (7 years, 10 years).
- Cleaning up incomplete multipart uploads (often forgotten — these accumulate silently and cost money).
- Expiring noncurrent versions on versioned buckets.

Storage classes and their break-even points (rough rule of thumb):
- **Standard**: hot data, frequent access.
- **Standard-IA**: accessed <1x/month, savings after ~30 days vs Standard.
- **Glacier Instant Retrieval**: rare access but millisecond retrieval.
- **Glacier Flexible Retrieval**: minutes-to-hours retrieval, archive use.
- **Glacier Deep Archive**: 12+ hours retrieval, long-term archive.
- **Intelligent-Tiering**: AWS auto-tiers based on access patterns; good default for unpredictable workloads.

### Maximum object size in S3

**5 TB per object.**

Practical constraints:
- **Single PUT**: max 5 GB.
- **Multipart upload**: required for >100 MB recommended, supports up to 5 TB total (up to 10,000 parts × 5 GB per part).

For very large objects, multipart upload also gives you resumability — if the upload fails partway, you only retry the failed parts.

### S3 bucket size limit

**No limit.** Buckets scale to virtually unlimited storage and object counts.

Practical limits to know:
- **100 buckets per account** by default (soft, raisable to 1,000 via support).
- **Performance**: 3,500 PUT/COPY/POST/DELETE requests/sec and 5,500 GET/HEAD requests/sec **per prefix**. Beyond that, distribute keys across more prefixes for higher aggregate throughput.
- **Object metadata** is in S3's index; massive buckets with billions of objects can have slower LIST operations.
- **Pricing scales linearly** with storage — there's no cliff, but cost discipline matters.

### Lambda maximum runtime

**15 minutes (900 seconds).**

If you need longer:
- **AWS Step Functions** for orchestration of multiple Lambda invocations.
- **ECS/Fargate task** for long-running workloads.
- **AWS Batch** for compute-heavy jobs.
- **EC2 or EKS** for indefinite runtime.

Other Lambda limits worth knowing:
- **10 GB memory** (max).
- **10 GB ephemeral `/tmp`** (was 512 MB default until 2022).
- **6 MB synchronous payload** for request/response.
- **256 KB async invocation payload**.
- **Container images up to 10 GB**.
- **1,000 concurrent executions** per region (soft, raisable).

### Common Lambda issues

**Cold start** — initialization delay when a new execution environment must be created (container spin-up, runtime init, code load, init code execution). Typical impact: 100ms–10s depending on runtime, package size, and VPC config.

Mitigations:
- **Provisioned Concurrency** — pre-warmed instances always ready. Cost: pay for reserved capacity.
- **Smaller deployment packages** — fewer dependencies, lazy imports, tree-shaking for Node.js.
- **Lambda SnapStart** (Java, .NET, Python, Ruby) — snapshot init state for sub-second cold starts.
- **Arm64 (Graviton)** — faster cold starts in many runtimes, plus 20% cheaper.
- **Avoid VPC unless required** — historically massive cold starts; much better now with Hyperplane ENIs but still adds overhead.
- **Connection reuse** — initialize SDK clients outside the handler so subsequent invocations reuse them.

**Other common Lambda issues:**

- **Concurrency throttling** — hitting account limits causes `Rate Exceeded` errors. Reserved concurrency to guarantee capacity for critical functions.
- **Timeout misconfiguration** — function timeout shorter than downstream call, leading to retries and wasted compute.
- **Memory undersizing** — Lambda CPU scales with memory. An undersized function is slow *and* expensive (longer runtime = more billed time). Use Lambda Power Tuning to find the optimal memory.
- **Async retry behavior** — failed async invocations retry twice; without a DLQ, errors are silently lost.
- **Dependency bloat** — unzipped deployment >250 MB doesn't fit; use Lambda Layers or container images.
- **`/tmp` reuse between invocations** — Lambda is stateless, but `/tmp` persists across invocations on the same execution environment. Both a foot-gun (assuming clean state) and an optimization (caching).
- **Logging cost** — verbose CloudWatch Logs can dominate Lambda billing. Tune log levels.
- **Init code that fails** — the init phase outside the handler runs once per cold start. If it throws, the function fails before the handler even runs.

### RDS high CPU troubleshooting

Methodical approach:

**1. Confirm and characterize the spike**

- CloudWatch `CPUUtilization` — duration and shape (sudden spike vs gradual climb).
- **Performance Insights** — top SQL by load, top wait events. This is the most useful single tool.
- `DatabaseConnections` metric — connection storms cause CPU spikes too.

**2. Identify the culprit query**

PostgreSQL:
```sql
-- Top queries by total time
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Currently active queries
SELECT pid, usename, state, wait_event, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Long-running queries
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '1 minute';
```

MySQL:
```sql
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 20;
SHOW PROCESSLIST;
```

**3. Common root causes ranked by likelihood**

- **Missing index** — sequential scan on a large table. Verify with `EXPLAIN ANALYZE`. Add index.
- **Stale statistics** — query planner picking bad plans. Run `ANALYZE` (PG) or `ANALYZE TABLE` (MySQL).
- **N+1 query pattern** — application issuing many small queries instead of one batched. App-side fix.
- **Connection storm** — too many short-lived connections; each connect/disconnect is expensive. Fix with PgBouncer or RDS Proxy.
- **Lock contention** — long transactions blocking others. Check `pg_locks` or `INNODB_LOCKS`.
- **Inefficient ORM queries** — generated SQL without proper joins, fetching too much data.
- **VACUUM / maintenance overhead** — autovacuum running on a large table during peak. Tune `autovacuum_vacuum_cost_limit`.
- **Workload outgrew the instance** — needs vertical scale or read replica offload.

**4. Mitigate**

Immediate (during incident):
- **Kill the offending query**: `SELECT pg_terminate_backend(pid)`.
- **Failover to a replica** if available — promotes the read replica and bypasses the loaded primary briefly.
- **Vertical scale** the instance class — minimal downtime via `modify-db-instance`.

Permanent:
- Add the missing index.
- Refactor the query / fix the N+1.
- Add a read replica and route read traffic.
- Add caching (ElastiCache).
- Move to Aurora if scaling characteristics warrant.

**5. Prevent recurrence**

- **Performance Insights** enabled with extended retention (>7 days).
- **Slow query log** to CloudWatch with alarms on slow-query rate.
- **Query review in PR** for any new migrations or ORM queries.
- **Load testing** before deploys.
- **Alert on CPU > 70% for 10 minutes** — proactive, not reactive.

---

## 🌐 Networking

### CIDR block

**Classless Inter-Domain Routing** — a notation for IP address ranges in the form `<network>/<prefix-length>`. The prefix length specifies how many leading bits identify the network; the rest are host bits.

`10.0.0.0/16` means:
- 16 bits identify the network (`10.0.x.x`).
- 16 bits available for hosts → 2^16 = 65,536 IP addresses.
- Range: `10.0.0.0` to `10.0.255.255`.

Common sizes and what they give you:
- `/8` = 16,777,216 IPs (`10.0.0.0/8` is the entire private 10-block)
- `/16` = 65,536 IPs (typical VPC size)
- `/20` = 4,096 IPs (large subnet for high-density workloads)
- `/24` = 256 IPs (typical small subnet)
- `/28` = 16 IPs (smallest AWS subnet allowed)
- `/32` = a single IP (used in route entries and SG rules)

AWS reserves 5 IPs per subnet (`.0`, `.1`, `.2`, `.3`, and the broadcast `.255`), so a `/24` actually gives 251 usable IPs.

### Public vs private subnet

The distinction is **whether the route table has a default route to an Internet Gateway**:

- **Public subnet**: route table has `0.0.0.0/0 → IGW`. Resources can have public IPs and be reachable from the internet. Used for ALBs, NAT Gateways, bastions.
- **Private subnet**: no IGW route. Outbound internet via a NAT Gateway in a public subnet. Resources have private IPs only, not directly reachable from the internet. Used for application servers, databases, internal services.

The "subnet" itself isn't intrinsically public or private — it's the routing that makes it so. Multiple subnets can share a route table.

### Designing CIDR blocks

**Principles:**

1. **Start big** — `/16` for the VPC unless you have a strong reason not to. Re-IPing later is painful.
2. **Plan for the future** — leave room for additional subnet tiers, more AZs, peered VPCs.
3. **No overlap** with other VPCs you might peer with (corporate network, other AWS accounts, other regions).
4. **Three AZs minimum** for production HA.
5. **Tier per subnet** — public, private-app, private-data — each with its own NACL.

**Example design for a production VPC:**

```
VPC: 10.0.0.0/16 (65,536 IPs)

Public subnets (small — only LBs and NAT GWs):
  10.0.0.0/24    us-east-1a    (256 IPs)
  10.0.1.0/24    us-east-1b
  10.0.2.0/24    us-east-1c

Private app subnets (large — EKS pods if using AWS VPC CNI):
  10.0.16.0/20   us-east-1a    (4,096 IPs)
  10.0.32.0/20   us-east-1b
  10.0.48.0/20   us-east-1c

Private data subnets (medium — RDS, ElastiCache):
  10.0.64.0/22   us-east-1a    (1,024 IPs)
  10.0.68.0/22   us-east-1b
  10.0.72.0/22   us-east-1c

Reserved for future growth: 10.0.96.0/19 onwards
```

**For EKS specifically:** pod density × max nodes × number of AZs gives your subnet sizing requirement. With AWS VPC CNI and ~30 pods per node, a `/24` (256 IPs) supports only ~8 nodes per AZ. For high-density clusters:
- **Prefix delegation** — VPC CNI allocates /28 prefixes (16 IPs) per ENI, dramatically increasing pod density.
- **Secondary CIDR** — add `100.64.0.0/16` (CGN range) for pod IPs only; keeps node/service IPs on the original VPC CIDR.

**Common mistakes to avoid:**
- VPC CIDR overlapping with corporate network (no peering possible later).
- `/24` subnets in production — too small for growth.
- Public subnets sized too large — they should be small; NAT Gateways and LBs don't need many IPs.
- Forgetting AWS reserves 5 IPs per subnet.
- Identical CIDRs across regions in case of future Transit Gateway peering.

---

## 🚀 CI/CD & DevOps

### Types of pipeline failures encountered

Categorized by stage and cause:

**Build stage:**
- **Compile failures** — code syntax errors, missing dependencies, version conflicts.
- **Test failures** — real regressions, flaky tests (timing/race conditions, leaky test state), environment differences (test machine has different timezone, locale, file system).
- **Dependency resolution** — registry timeouts, breaking changes in transitive deps, license-blocked dependencies.
- **Resource exhaustion** — agent ran out of disk (huge `node_modules` directories are common), OOM during build.
- **Network blips** — registry unreachable, intermittent DNS.

**Quality and security gates:**
- **SonarQube quality gate failing** on new code coverage or duplication.
- **Trivy/Snyk blocking** on CRITICAL/HIGH CVEs.
- **License scan** finding GPL dependencies in proprietary code.
- **IaC scan** (tfsec, Checkov) flagging policy violations.

**Image and artifact stage:**
- **Docker build cache miss** causing 30-minute builds.
- **Registry push failures** — auth expired, rate limit, throughput cap.
- **Image scan failures** on newly-disclosed CVEs.

**Deploy stage:**
- **Kubeconfig or credentials expired**.
- **Helm chart rendering errors** — missing values, schema validation failure.
- **Resource conflicts** — immutable field changes, name conflicts.
- **Probes failing** — new image's `/health` endpoint moved or returns different schema.
- **ImagePullBackOff** — wrong tag, missing pull secret, ECR auth misconfigured.
- **Rollout timeout** — new pods not becoming Ready in time.
- **Cluster autoscaler lag** — pods stuck Pending waiting for nodes.

**Post-deploy:**
- **Smoke tests failing** — environment-specific config issue.
- **SLO regression** — canary detected error rate or latency increase, automatic abort.
- **ArgoCD sync stuck** — webhook failure, manifest validation error.

### Troubleshooting CI/CD failures

Structured approach:

**1. Reproduce locally if possible**

Many "CI-only" failures reproduce when you run the exact Docker container the agent uses:
```bash
docker run -v $PWD:/work -w /work <ci-image> <command>
```

Often reveals the issue in seconds.

**2. Read logs top to bottom**

The first failure cascades — don't fix the last error before understanding the first. For multi-stage pipelines, find the *first* failed step.

**3. Differentiate transient vs deterministic**

Run the same pipeline twice. Same failure → deterministic, code or config fix. Different failure → transient (network, agent capacity, flaky test).

- **Deterministic**: real fix in code/config.
- **Transient**: retry logic in the pipeline, with alerting on retry usage so root cause gets fixed.

**4. Bisect with git**

`git bisect` to find the commit that introduced a regression. Compare the last green pipeline with the first red one — what changed?

**5. Common diagnostic commands**

```bash
# In pipeline shell
env | sort                       # what env vars are set?
which python java node           # path issues
ls -la /workspace                # working dir state
df -h                            # agent disk
free -m                          # agent memory
ps auxf                          # what's running

# Docker
docker ps -a                     # exited containers
docker logs <container>
docker images                    # disk usage
docker system df

# Kubernetes deploy failures
kubectl describe pod <pod>       # events at the bottom — gold
kubectl logs <pod> --previous    # previous restart's logs
kubectl get events --sort-by='.lastTimestamp' -n <ns>
```

**6. Pipeline-side fixes**

- **Pin versions** — Docker base image, Node version, Maven version, Terraform version. "Worked yesterday, broken today" almost always traces to an unpinned version.
- **Cache dependencies** intelligently — invalidate when lock file changes, not on every build.
- **Increase retries** for known-transient operations (`retry: 3` on registry pushes).
- **Increase timeouts** when the issue is genuinely slow steps.
- **Split slow stages** so failures fail fast.
- **Add structured logging** in CI scripts so failures are diagnosable from logs alone.

**7. Postmortem habits**

- Track flaky test rate as a metric; aim for <0.5%.
- Treat repeated pipeline failures as bugs — file tickets, fix the underlying cause.
- "Fix the build" culture — a broken `main` is top priority; nobody merges over a red build.

The principle: the goal isn't a green pipeline at all costs — it's a pipeline that **tells you the truth quickly**. Retries that mask flakiness create slower truth, not faster builds.

---

## 🐳 Docker

### Purpose of `docker tag`

`docker tag` assigns an additional name (and optionally a different repository) to an existing image. The image data isn't duplicated — both tags point to the same image ID.

```bash
# After building
docker build -t myapp:1.4.2 .

# Re-tag for ECR
docker tag myapp:1.4.2 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Add a "latest" tag pointing at the same image
docker tag myapp:1.4.2 myapp:latest

# Now push to a registry
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

Use cases:
- **Re-tag for a registry** — local builds tagged `myapp:1.4.2` need re-tagging with the registry URL before push.
- **Mark a build with multiple tags** — same image as `1.4.2`, `1.4`, `1`, `latest`.
- **Promote between environments** — re-tag a staging image with a prod tag after validation. Or, for modern setups, use `docker buildx imagetools create` for manifest-level re-tagging without pulling locally.

Tagging principles for production:
- **Immutable build tags** — `1.4.2-a1b2c3d-build2087` (semver + git SHA + build number). Never overwritten.
- **Floating tags** (`latest`, `1.4`, `1`) — for dev/local convenience only. Never referenced from prod manifests.
- **Tag immutability enabled** on prod ECR repos so tags can't be overwritten accidentally.

### Building an image from a Dockerfile

```bash
docker build -t myapp:1.4.2 .
```

Breakdown:
- `docker build` — invokes the build.
- `-t myapp:1.4.2` — tag the resulting image (`-t` can be specified multiple times for multiple tags).
- `.` — the **build context**. Docker daemon receives all files in this directory (minus `.dockerignore` excludes); the `Dockerfile` is expected here unless `-f` specifies otherwise.

Useful variations:

```bash
# Different Dockerfile location
docker build -f docker/prod.Dockerfile -t myapp:prod .

# Multiple tags in one build
docker build -t myapp:1.4.2 -t myapp:latest .

# Build args
docker build --build-arg VERSION=1.4.2 --build-arg NODE_ENV=production -t myapp:1.4.2 .

# Target a specific stage (multi-stage builds)
docker build --target build -t myapp:build-stage .

# Use BuildKit for cache mounts and secrets
DOCKER_BUILDKIT=1 docker build -t myapp:1.4.2 .

# Modern unified frontend with multi-platform support
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.4.2 --push .

# Pass secrets at build time without storing in layers (BuildKit)
docker build --secret id=npmtoken,src=$HOME/.npmrc -t myapp:1.4.2 .

# No cache (force fresh layers)
docker build --no-cache -t myapp:1.4.2 .

# Pull base images even if cached locally
docker build --pull -t myapp:1.4.2 .
```

Always pair with a `.dockerignore` to exclude `.git`, `node_modules`, secrets, build artifacts — keeps the build context small and prevents secrets from leaking into image layers.

```
# Example .dockerignore
.git
.gitignore
node_modules
target
dist
*.log
.env*
Dockerfile
.dockerignore
*.md
.vscode
.idea
```

Without this, the entire `.git` directory (potentially hundreds of MB) becomes part of the build context, transferred to the daemon on every build.

---

That covers all the questions. Common follow-ups to be ready for:

- **"How would you decide between `count` and `for_each` in Terraform?"** — `for_each` for stable named sets, `count` for simple replication or boolean toggles.
- **"What's the difference between `docker tag` and `docker push`?"** — `tag` is local metadata; `push` actually transfers layers to a registry.
- **"How do you handle a Lambda that's hitting the 15-minute timeout?"** — refactor to Step Functions, or move to ECS/Fargate if it's genuinely long-running compute.
- **"Show me a NACL rule that allows web traffic but is also stateless-correct."** — inbound 443 + outbound 1024-65535 (ephemeral response ports).

# Senior DevOps Interview — Complete Answers

A wide-ranging set covering architecture, Kubernetes depth, Helm mechanics, Docker internals, and CI/CD. Going through each with the depth a senior engineer should bring.

---

## 1. Two-tier vs three-tier architecture — use cases and AWS resources

**Two-tier (client → database):**

Use case: a small internal admin tool, data-entry app, or CRUD dashboard for ~20 internal users. The "client" (thick desktop app or web app) talks directly to the database — no separate application server.

AWS resources:
- **EC2** (or Elastic Beanstalk for a managed wrapper) running the application that includes both UI and DB access logic.
- **RDS** as the database tier.
- **ALB** in front of the EC2 if web-accessed.
- **Route 53** for DNS.
- **VPC** with public subnet (LB/EC2) and private subnet (RDS).
- **ACM** for TLS.

When two-tier makes sense: small scale, simple workloads, internal tools, MVPs. Operationally cheaper but doesn't scale and mixes presentation with data logic.

**Three-tier (presentation → application → data):**

Use case: customer-facing e-commerce, SaaS, banking portal — anything customer-facing with non-trivial scale and separation-of-concerns requirements.

AWS resources by tier:

**Presentation:**
- **CloudFront** for global edge caching, WAF integration, TLS termination at edge.
- **S3** for static frontend assets (React/Angular bundles).
- **Route 53** for DNS with health-check-based failover.
- **ACM** for certificates.
- **AWS WAF** + **Shield** for DDoS protection.

**Application:**
- **ALB** in public subnets fronting app servers.
- **EKS** or **ECS Fargate** for containerized microservices, or **EC2 Auto Scaling Group** for traditional apps.
- **Lambda** for event-driven workloads.
- **API Gateway** for REST/HTTP/WebSocket APIs.
- **SQS / SNS / EventBridge** for async messaging.
- **ElastiCache Redis** for caching and sessions.

**Data:**
- **RDS Aurora** Multi-AZ for transactional data.
- **DynamoDB** for high-write low-latency NoSQL.
- **S3** for object storage with lifecycle policies.
- **OpenSearch** for search/analytics.
- All in **private data subnets**, no internet route.

**Cross-cutting:**
- **IAM** with IRSA for pod-level identity.
- **KMS** for encryption.
- **Secrets Manager** for credentials.
- **CloudWatch + Prometheus/Grafana** for observability.
- **VPC** with three subnet tiers across three AZs.

**Architect's framing:**
> "Three-tier is the default for customer-facing production systems. Two-tier survives in legacy and small internal tools. The real value of three-tier isn't just scalability — it's that each tier can be secured, scaled, and replaced independently. I can swap the DB engine without touching presentation; I can scale app servers without scaling the DB."

---

## 2. Major drawbacks of ECS and EKS

**ECS drawbacks:**

- **AWS-only lock-in** — proprietary orchestration; skills and IaC don't transfer. Migrating to Kubernetes later means rewriting task definitions, service definitions, deployment patterns.
- **Smaller ecosystem** — no equivalent to Helm charts, Operators, or the CNCF tool universe. You build things ECS doesn't ship.
- **Limited GitOps story** — ArgoCD/Flux don't natively manage ECS. Back to imperative deploys via CodePipeline or custom tooling.
- **Less expressive scheduling** — no topology spread constraints, complex affinity rules, or PriorityClasses. ECS placement constraints are simpler but less flexible.
- **Service discovery limitations** — relies on Cloud Map (DNS) or ALB target groups. No equivalent to Kubernetes Services + Endpoints with mesh integration.
- **Stateful workload story is weak** — no StatefulSets equivalent.
- **Multi-region orchestration is manual** — no equivalent to Cluster API or fleet management.

**EKS drawbacks:**

- **Operational complexity** — Kubernetes itself is a distributed system. Even managed control plane doesn't free you from understanding etcd, CRDs, networking, RBAC, admission controllers.
- **Cost overhead** — $0.10/hour per cluster (~$73/month). At 20 clusters that's $1,460/month before compute. ECS control plane is free.
- **VPC CNI IP exhaustion** — AWS VPC CNI assigns one VPC IP per pod, so subnet sizing must accommodate pod count. Subnet exhaustion is a common incident.
- **EKS version lifecycle** — every K8s minor version supported ~14 months; forced upgrade treadmill. Each upgrade requires testing because of breaking API changes (PodSecurityPolicy removal, Ingress API moves, etc.).
- **IRSA and cluster auth setup** is non-trivial — OIDC issuer config, IAM role trust policies, aws-auth ConfigMap drift is a classic gotcha.
- **Bigger blast radius for control-plane issues** — control plane managed by AWS, but workloads still feel API server throttling, etcd slowdowns.
- **Observability requires more setup** — CloudWatch Container Insights works but is limited; production setups bolt on Prometheus, Grafana, Loki, all of which you operate.
- **Tooling churn** — Kubernetes ecosystem moves fast; what was best practice 2 years ago (Tiller-Helm 2, PSPs) is now deprecated.

**Architect's framing:**
> "ECS is the simpler answer if you're committed to AWS and your workload fits the model — fewer moving parts, lower operational toil, lower cost. EKS is the right answer when you need Kubernetes-specific capabilities, multi-cloud portability, or the ecosystem. The biggest practical drawback of EKS is that you're operating a distributed system on top of another distributed system, and the abstractions leak. For ECS, the biggest drawback is being locked into AWS's roadmap for orchestration features."

---

## 3. RDS volume over-provisioned (100GB, low usage) — how to decrease size

**Critical answer up front: RDS does not support shrinking storage in place.** You can only increase storage; decreasing requires a different approach. State this clearly — knowing the constraint is the answer.

**Approach 1: Snapshot, restore to smaller storage** — DOES NOT WORK. RDS restores use the original snapshot's allocated size minimum. Worth mentioning so the interviewer knows you know.

**Approach 2: Logical dump and restore (the standard approach)**

1. **Provision a new RDS instance with the desired smaller storage** (say 20GB):
   ```bash
   aws rds create-db-instance \
     --db-instance-identifier myapp-rds-small \
     --db-instance-class db.t3.medium \
     --engine postgres \
     --allocated-storage 20 \
     --storage-type gp3 \
     --master-username admin \
     --master-user-password "<from-secrets-manager>"
   ```

2. **Logical dump from source:**
   ```bash
   pg_dump -h source-rds.amazonaws.com -U admin -F c -d mydb -f mydb.dump
   ```

3. **Restore to target:**
   ```bash
   pg_restore -h target-rds-small.amazonaws.com -U admin -d mydb -j 4 mydb.dump
   ```

4. **Cut over with minimal downtime:**
   - Best: **AWS DMS** (Database Migration Service) for continuous replication. Replicates ongoing changes from old to new while the app keeps writing to old.
   - Once caught up, brief downtime: stop writes on old, drain final changes, point app to new (via DNS or Secrets Manager update — no app redeploy needed).
   - For PostgreSQL specifically: **native logical replication** can do the same without DMS.

5. **Verify and retire the old instance** after a buffer period.

**Approach 3: For Aurora — different mechanics**

Aurora storage is decoupled from compute. Aurora storage **auto-grows in 10GB increments and bills only for what you use**. There's no "over-provisioned" Aurora storage problem — it scales down automatically too in recent versions. If on Aurora, the answer is "no action needed" — verify by checking actual used vs allocated.

**Approach 4: Blue-green deployment (RDS native, 2022+)**

```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name shrink-storage \
  --source <source-arn> \
  ...
```

Creates a replica with different config (including smaller storage if data fits), then atomic switchover with minimal downtime. Cleaner than manual migration for supported scenarios.

**The architect's answer:**
> "RDS storage can't be shrunk in place — Amazon allows only increases. To genuinely reduce storage, I'd provision a new smaller instance and migrate the data. For minimal downtime, I'd use AWS DMS or PostgreSQL native logical replication to keep the new instance in sync while the app continues writing to the old one. After a brief cutover window — typically 30 seconds to a few minutes — I'd update the connection string via Secrets Manager and retire the old instance after verification. Worth noting: if this is Aurora rather than RDS, storage scales automatically and there's nothing to fix. Also worth asking the business question: at gp3 rates, 100GB is ~$10/month. The engineering effort to shrink may exceed the savings — sometimes the right call is to leave it and apply effort elsewhere."

That last point — questioning whether the optimization is worth the effort — shows seniority.

---

## 4. Domain-based applications in Kubernetes

The interviewer is asking what *kind* of business application I run in K8s, with the context that team sizes and pod counts vary.

> "Across teams I've worked with, Kubernetes runs a range of business domains:
>
> **For the team running ~10 pods:**
> A loan-origination microservice domain — about 8 services covering application intake, credit decisioning, document verification, payment scheduling. Stateless Java Spring Boot services, fronted by API Gateway, backed by Aurora and DynamoDB. Pod count stays small because each service handles modest transactional volume — peak ~500 TPS — and we sized for predictable workloads with HPA min: 2, max: 6.
>
> **For the team running ~100+ pods:**
> A consumer-facing e-commerce platform — order management, inventory, pricing, cart, checkout, recommendation, search, notification, payment, shipping, fulfillment. ~30 microservices, each with 4-10 replicas across three AZs, totalling ~150-200 pods baseline and scaling to ~400 during peak events (Black Friday, sale launches). Polyglot: Java, Node.js, Python, Go. Backed by Aurora, DynamoDB, Kafka (MSK), Elasticsearch, Redis.
>
> The difference in pod count maps to traffic shape and decomposition strategy. The smaller domain is fewer services with higher per-service replica counts; the e-commerce platform has many narrowly-scoped services with moderate replication. Both run on EKS with the same platform conventions: ArgoCD for GitOps, Prometheus/Grafana/Loki, Istio for mesh, Karpenter for node scaling.
>
> Other domains I've supported in K8s:
> - **Fraud detection** — event-driven Python services consuming from Kafka, scaled via KEDA on consumer lag.
> - **Internal developer platform** — Backstage, ArgoCD, Vault, image registries — the meta-platform that supports product teams.
> - **Data pipelines** — Airflow on K8s with KubernetesPodOperator for ETL jobs."

The strength of this answer is concrete domain detail, honest scale numbers, and the meta-observation that pod count varies by decomposition strategy, not just business size.

---

## 5. End-to-end application management in Kubernetes

> "Yes — end-to-end ownership is the model I prefer. For one platform specifically, I led the journey from initial design to steady-state operation:
>
> **Design phase:** Translated the application architecture into Kubernetes primitives — Deployments for stateless services, StatefulSets for the Kafka cluster, HPA + KEDA for scaling, NetworkPolicies for zero-trust, NGINX Ingress + cert-manager for TLS. Chose EKS over self-managed for the control-plane SLA. Designed cluster topology: three AZs, separate node pools by workload tier (system, prod, batch), Karpenter for elastic scaling.
>
> **Infrastructure provisioning:** Terraform modules for VPC, EKS, IAM (including IRSA for each service), and supporting AWS services (RDS, MSK, ElastiCache, S3). Per-environment state files in S3 with DynamoDB locking.
>
> **Platform layer:** Installed and configured in-cluster tooling — ArgoCD for GitOps, kube-prometheus-stack for metrics, Loki + Promtail for logs, Tempo for traces, External Secrets Operator for syncing from AWS Secrets Manager, Kyverno for policy enforcement.
>
> **CI/CD:** GitHub Actions pipelines per service — build, test, SonarQube, Trivy, Cosign sign, push to ECR, bump tag in GitOps repo. Argo Rollouts for canary deployments on tier-1 services.
>
> **Observability:** RED metrics on every service via Micrometer/OpenTelemetry auto-instrumentation. Dashboards templated per service. SLO definitions in code, alerts mapped to runbooks.
>
> **Day-2 operations:** On-call rotation, runbooks for common incidents, blameless postmortems, quarterly DR game days, monthly capacity reviews. Cost visibility via Kubecost.
>
> **Continuous improvement:** Cluster upgrades (one minor version per quarter), node OS image refresh monthly, dependency upgrades, periodic chaos experiments via Chaos Mesh to validate failure handling.
>
> The thing that makes 'end-to-end' real isn't just having touched each piece — it's owning the outcomes. When the platform has a Sev-1, my team is the one explaining what happened, why, and what we're doing about it. That accountability shapes how I design things from the start."

---

## 6. Init Container vs StatefulSet

Different concepts at different layers — important to clarify the framing of the question.

**Init Container** — a *container within a pod* that runs to completion before the main containers start. Used for setup tasks: schema migrations, waiting for dependencies, fetching config, file permissions, prep work.

**StatefulSet** — a *workload controller* (peer to Deployment) that manages a set of pods with stable identity, ordered startup, and per-pod storage. Used for stateful applications like databases, message brokers, distributed coordination services.

**Side-by-side:**

| | Init Container | StatefulSet |
|---|---|---|
| Layer | Container-level (inside a pod) | Workload-level (manages pods) |
| Purpose | Prep work before main containers | Orchestrate stateful applications |
| Lifecycle | Runs once per pod, to completion | Long-running, manages many pods |
| Storage | Shares pod's volumes | Per-pod PersistentVolumeClaims |
| Identity | None | Stable, ordered (pod-0, pod-1, ...) |
| Ordering | Sequential within a pod | Sequential across pods at startup |
| Failure handling | Pod doesn't start until init succeeds | Failed pod rescheduled with same identity |

**They're orthogonal.** A StatefulSet's pods can have init containers; a Deployment's pods can also have init containers. The init container is part of the pod spec; the controller (Deployment, StatefulSet, DaemonSet, Job) is what manages the pods.

**Example combining both** — a Kafka StatefulSet with an init container that waits for Zookeeper:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka
  replicas: 3
  selector: { matchLabels: { app: kafka } }
  template:
    metadata: { labels: { app: kafka } }
    spec:
      initContainers:
      - name: wait-for-zookeeper
        image: busybox
        command: ['sh', '-c', 'until nc -z zookeeper 2181; do sleep 2; done']
      containers:
      - name: kafka
        image: kafka:3.6
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      resources: { requests: { storage: 100Gi } }
```

---

## 7. Liveness and readiness probes on init containers

**No, you shouldn't, and Kubernetes doesn't allow it for traditional init containers.**

**The reasons it doesn't make sense:**

1. **Init containers run to completion** — they're not long-lived. Liveness probes are designed for processes that keep running; readiness probes are designed to gate traffic to pods that are accepting requests. Init containers do neither.

2. **The semantic of "ready"** doesn't apply — init containers never receive traffic. They run, finish (successfully or not), and exit.

3. **The semantic of "alive"** is implicit — if an init container hangs, the pod startup hangs. The pod's `startupProbe` on the main container serves the equivalent role at the pod level.

4. **Lifecycle ordering** — Kubernetes runs init containers serially before main containers start. Probe loops don't fit this serial execution model.

**The exception worth knowing about — Sidecar Containers (Kubernetes 1.28+ alpha, 1.29+ beta, 1.33 stable):**

Kubernetes introduced "native sidecar containers," which are technically a special kind of init container with `restartPolicy: Always`. These DO support probes because they run for the lifetime of the pod alongside the main containers.

```yaml
spec:
  initContainers:
  - name: logging-sidecar
    image: fluent-bit
    restartPolicy: Always         # makes this a sidecar, not a classic init
    livenessProbe:                # ALLOWED for sidecars
      httpGet: { path: /, port: 2020 }
    readinessProbe:               # ALLOWED for sidecars
      httpGet: { path: /, port: 2020 }
  containers:
  - name: app
    image: myapp:1.0
```

So the precise answer:
- **Classic init containers (run to completion):** no, not advisable and not supported.
- **Sidecar containers (`restartPolicy: Always`):** yes, probes are supported and useful.

**Architect's framing:**
> "Init containers are short-lived prep work — probes don't fit their lifecycle. If you want long-running auxiliary containers with their own health checks, the right primitive is a sidecar — either traditional in the `containers` array, or the new native sidecar via init container with `restartPolicy: Always`. When someone wants to put a probe on an init container, they've usually actually got a sidecar workload modeled wrong."

---

## 8. Why StatefulSet over Deployment+ReplicaSet?

The interviewer is testing whether I understand when StatefulSet's specific guarantees are required versus when they're overkill.

**StatefulSet provides three things Deployments don't:**

1. **Stable pod identity** — pods named with ordinal suffix (`kafka-0`, `kafka-1`, `kafka-2`), preserved across restarts, rescheduling, and node migrations. Pod `kafka-0` is *always* `kafka-0`.

2. **Stable DNS** — each pod gets its own DNS record (`kafka-0.kafka.namespace.svc.cluster.local`) so other clients can address a specific pod, not just "any pod behind the service."

3. **Per-pod PersistentVolumeClaim** — each pod has its own PV, created from `volumeClaimTemplates`. Pod `kafka-0`'s PV stays attached to `kafka-0` even if it's rescheduled to a different node.

Plus ordered, sequential startup and termination (configurable).

**When these matter — when you'd choose StatefulSet:**

- **Kafka brokers** — each broker has a unique ID; topic partitions are assigned to specific brokers. Re-IDing a broker requires data resync.
- **Elasticsearch / OpenSearch nodes** — shards placed on specific nodes; nodes have stable identities.
- **Cassandra** — distributed ring; each node owns specific token ranges.
- **MongoDB replica sets** — primary/secondary roles; each member uniquely identified.
- **Zookeeper / etcd** — consensus protocols depend on stable member identity.
- **Stateful databases on K8s** (Postgres, MySQL) — leader/follower roles, persistent data.
- **Distributed coordination services** (Consul, HashiCorp Vault).

**When you'd use Deployment+ReplicaSet (StatefulSet would be overkill):**

- Any stateless service — REST APIs, web frontends, workers.
- Apps that share state via an external DB rather than per-pod storage.
- Apps where pods are interchangeable; load balancer doesn't need to know which is which.

**Answering "why I chose StatefulSet" specifically:**

> "In the case I'm thinking of, we ran Kafka on Kubernetes via Strimzi operator, which uses StatefulSets. The reason it had to be StatefulSet rather than a Deployment:
>
> 1. **Each Kafka broker has a unique ID** (`broker.id`) — that ID is in the metadata. If broker-2 is rescheduled to a new node and comes up as a different identity, the cluster sees it as a new broker, not the original. Replicas wouldn't sync; data would be re-replicated. StatefulSet preserves identity.
>
> 2. **Each broker has its own data directory** with topic partition data — 200GB per broker. We needed each pod to keep its own PV across restarts. StatefulSet's `volumeClaimTemplates` gives us per-pod PVs that follow pod identity.
>
> 3. **Ordered startup matters** — Zookeeper-coordinated bootstrapping needs broker-0 up before broker-1 joins. StatefulSet handles this naturally.
>
> 4. **Other services connect to specific brokers** — the Kafka client protocol involves direct broker connections after initial discovery. Stable DNS names are required.
>
> Using a Deployment would have produced random pod names changing on each restart, lost per-pod PV affinity (or required all pods to share one PV via ReadWriteMany, which Kafka doesn't support), and broken the ordered-restart guarantee. StatefulSet was the only correct primitive."

---

## 9. Deploying a new image version with Helm — what changes before deployment

The expected changes before a deploy:

**1. Image tag bump** — the actual change driving the deploy
```yaml
# values-prod.yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
  tag: 1.4.3-a1b2c3d-build2087     # bumped from 1.4.2-...
  pullPolicy: IfNotPresent
```

**2. Chart version bump** — if the chart itself changed
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.4.3        # chart semver — bumped for chart changes
appVersion: 1.4.3     # tracks app version
```

**3. ConfigMap changes** — if the new image needs new config
```yaml
config:
  newFeatureFlag: true
  databasePoolSize: 50    # changed for v1.4.3
```

**4. Resource adjustments** — if profiling showed v1.4.3 needs different resources
```yaml
resources:
  requests: { cpu: 300m, memory: 768Mi }    # was 250m / 512Mi
  limits:   { cpu: 1500m, memory: 1.5Gi }
```

**5. Health check path changes** — if the new image moved `/health`
```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: http }
```

**6. Database migrations** — if v1.4.3 includes schema changes
```yaml
migrationJob:
  enabled: true
  image:
    tag: 1.4.3-a1b2c3d-build2087
  # Configured as a pre-install / pre-upgrade hook
```

**7. Feature flags / rollout controls** — for gradual rollout
```yaml
rollout:
  strategy: canary
  steps:
    - setWeight: 10
    - pause: { duration: 5m }
    - setWeight: 50
    - pause: { duration: 10m }
    - setWeight: 100
```

**Pre-deploy verification steps:**

```bash
# 1. Validate the chart renders correctly
helm template myapp ./chart -f values-prod.yaml | kubeconform -strict

# 2. Diff against currently deployed
helm diff upgrade myapp ./chart -f values-prod.yaml

# 3. Dry-run
helm upgrade --install myapp ./chart -f values-prod.yaml --dry-run

# 4. Lint
helm lint ./chart

# 5. Final apply
helm upgrade --install myapp ./chart \
  -f values-prod.yaml \
  --atomic --wait --timeout 10m
```

**In a GitOps model:** I don't run helm directly. The change is committed to the GitOps repo (the values file) and ArgoCD reconciles. The "change before deployment" is a PR with the values diff, reviewed and merged.

**Architect's framing:**
> "The actual change is usually a single line — `tag: 1.4.3-...`. The rigor around it is what makes it production-safe: chart version bump for traceability, `helm diff` for review, atomic apply for automatic rollback on failure, migrations gated through pre-install hooks. In GitOps, this all becomes a PR — no human runs `helm upgrade` against prod directly."

---

## 10. Where is Helm used in the pipeline, what code, what's its contribution at build time?

**Where Helm sits in the pipeline:**

```
Build (CI)                          → Helm not involved
  Compile, test, SAST, image scan
  Build Docker image
  Push to ECR

Package (CI)                        → Helm packaging happens here
  Lint Helm chart
  Package Helm chart (helm package)
  Push chart to Helm registry (ChartMuseum, ECR OCI, Artifactory)

Deploy (CD)                         → Helm rendering and applying
  Helm upgrade --install (or ArgoCD does it via Helm)
```

**At build/package time (CI):**
```yaml
# .github/workflows/ci.yml
- name: Helm lint
  run: helm lint ./chart

- name: Helm template (verify renders correctly)
  run: |
    helm template myapp ./chart \
      -f chart/values-prod.yaml \
      --set image.tag=${{ env.IMAGE_TAG }} \
      | kubeconform -strict

- name: Package chart
  run: |
    helm package ./chart \
      --version ${{ env.CHART_VERSION }} \
      --app-version ${{ env.APP_VERSION }} \
      --destination ./packaged

- name: Push chart to OCI registry
  run: |
    helm push ./packaged/myapp-${{ env.CHART_VERSION }}.tgz \
      oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts
```

**At deploy time (CD):**

Either Helm CLI directly:
```bash
helm upgrade --install myapp ./chart \
  -f values-prod.yaml \
  --set image.tag=$IMAGE_TAG \
  --atomic --wait --timeout 10m
```

Or via ArgoCD's Helm support:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/charts
    chart: myapp
    targetRevision: 1.4.3
    helm:
      valueFiles: [values-prod.yaml]
      parameters:
      - name: image.tag
        value: 1.4.3-a1b2c3d
```

**Helm's contribution at build time:**

- **Validation** — `helm lint` catches syntax errors, missing required fields, schema violations before they reach the cluster.
- **Template rendering** — `helm template` produces final manifests that can be validated with `kubeconform`, scanned by `checkov`/`kubesec`, and diffed against deployed state.
- **Packaging** — turns the chart directory into a versioned `.tgz` artifact, stamped with metadata (version, dependencies, signing). This artifact is the immutable, versioned deployment unit.
- **Pushing to a registry** — promotes the chart to where CD pulls from. Same pattern as Docker images.
- **Dependency resolution** — `helm dependency update` pulls subchart versions referenced in `Chart.yaml`.

**At deploy time, Helm:**

- Renders templates with environment-specific values.
- Computes diff against the cluster's existing release.
- Applies the diff in dependency order (CRDs first, then namespaces, then everything else).
- Tracks the release in `secrets/configmaps` for history and rollback.
- Runs lifecycle hooks (`pre-install`, `pre-upgrade`, `post-install`).

**Architect's framing:**
> "Helm is doing two distinct jobs: it's a *template engine* for Kubernetes YAML and a *package manager* for versioned releases. At build time we use the package manager role — lint, render, package, push. At deploy time we use both roles — Helm renders with environment values and tracks the release for rollback. The build-time contribution is making the chart itself a reviewable, versioned artifact, not just templates evaluated at deploy time against an arbitrary state."

---

## 11. What happens behind the scenes during rollback?

**`helm rollback myapp 1`** triggers this sequence:

**1. Look up the target revision**

Helm stores every release as a Secret (or ConfigMap, depending on storage backend) in the namespace:
```bash
kubectl get secrets -n production | grep sh.helm.release
# sh.helm.release.v1.myapp.v1
# sh.helm.release.v1.myapp.v2
# sh.helm.release.v1.myapp.v3       ← current
```

Each revision's Secret contains the rendered manifests, values, chart, and metadata from when that revision was applied.

**2. Read the target revision's manifests**

Helm decompresses the Secret and reads the manifests as they existed at revision 1.

**3. Compute the diff between current cluster state and target manifests**

Helm does a three-way merge:
- Current live state (from the cluster).
- Previous revision's last-applied state.
- Target revision's manifests.

This produces the set of operations to bring the cluster from current to target.

**4. Apply the operations**

In dependency order:
- CRDs (if changed).
- Namespaces.
- ConfigMaps, Secrets.
- PersistentVolumeClaims.
- Deployments, StatefulSets, DaemonSets — *these are updated, not destroyed and recreated*.
- Services, Ingresses.
- HPAs, PDBs.

**5. For Deployments specifically:**

Helm patches the Deployment manifest to match revision 1's spec. The Deployment controller sees the change and:
- Computes a new pod template hash.
- Creates a new ReplicaSet (or selects an existing one — see Q12).
- Begins rolling update: scales up the rolled-back ReplicaSet, scales down the current one, per `maxSurge`/`maxUnavailable`.
- Each new pod starts, passes readiness probe, joins Service endpoints.
- Old pods get SIGTERM, run preStop hooks, drain, exit.

**6. Run rollback hooks**

If the chart defines `pre-rollback` or `post-rollback` hooks, those run at the appropriate time.

**7. Update Helm release history**

A new revision is recorded (revision N+1, not "revert to 1"). Helm history shows:
```
REVISION  UPDATED              STATUS      CHART       DESCRIPTION
1         2025-10-15 09:00:00  superseded  myapp-1.4.1 Install complete
2         2025-10-20 14:30:00  superseded  myapp-1.4.2 Upgrade complete
3         2025-10-22 11:15:00  superseded  myapp-1.4.3 Upgrade complete
4         2025-10-22 11:45:00  deployed    myapp-1.4.1 Rollback to 1
```

**8. What's NOT rolled back automatically**

- **PVC data** — Helm doesn't restore data; volume content is whatever it currently is.
- **Database schema changes** applied by migrations — Helm doesn't reverse SQL migrations. A pre-rollback hook would handle this if defined; otherwise schema is whatever the current app version expects.
- **External state** — anything in S3, Kafka, RDS that the app produced under v3.

**Architect's framing:**
> "Behind the scenes, rollback is just another `apply` with manifests from a previous revision. It's not magic — Helm stored what each revision looked like, so it can reconstruct that state. The two things people miss: rollback creates a *new* revision in Helm history rather than deleting the current one (so you can roll forward again easily), and rollback doesn't touch data — if your v1.4.3 wrote bad data, rolling the app back to v1.4.1 doesn't fix the data. That's why expand-contract migrations and backwards-compatible API changes matter more than rollback mechanics."

---

## 12. During rollback, is a new ReplicaSet created, and does it also roll back?

This is a precise Kubernetes mechanics question.

**The short answer:** It depends on whether the rolled-back pod template has been seen before.

**The Deployment controller's logic:**

When a Deployment's pod template (`spec.template`) changes, the controller:

1. Computes the hash of the pod template (`pod-template-hash`).
2. Looks for an existing ReplicaSet with that hash.
3. If found: scales the existing ReplicaSet back up.
4. If not found: creates a new ReplicaSet with that hash and scales it up.

**Scenario A: Rolling back to a recent version (within revision history)**

```bash
kubectl get rs -l app=myapp
# myapp-7d4f8b9c8c   3   3   3   2h   (current)
# myapp-5b8d6c4f5d   0   0   0   1d   (previous, scaled to 0)
# myapp-3a2e1d7b6c   0   0   0   3d   (older, scaled to 0)
```

The ReplicaSets for previous revisions are *not deleted* — they're scaled to 0 and retained based on `spec.revisionHistoryLimit` (default 10).

**Rolling back via `kubectl rollout undo`:**
- The Deployment controller finds the previous ReplicaSet (already exists with the right hash).
- Scales it up.
- Scales the current ReplicaSet down.
- **No new ReplicaSet is created.**

```bash
kubectl rollout undo deployment/myapp
kubectl get rs -l app=myapp
# myapp-7d4f8b9c8c   0   0   0   2h   (scaled down)
# myapp-5b8d6c4f5d   3   3   3   1d   (scaled back up — same RS, now active)
```

**Scenario B: Rolling back via Helm**

Helm's rollback re-applies manifests as they were in the target revision. The Deployment manifest gets patched. The Deployment controller then:
- Computes the hash of the rolled-back pod template.
- If it matches an existing ReplicaSet (the original for that revision): scales that RS up. **No new RS.**
- If it doesn't match (because that RS was garbage-collected per `revisionHistoryLimit`): creates a new RS with the same pod template.

In practice, recent rollbacks (within `revisionHistoryLimit`) reuse the existing ReplicaSet. Older rollbacks create new ones.

**What happens to pods during the rollback rollout:**

Standard rolling update mechanics:
- New pods (from rolled-back-to RS) come up one by one.
- Each waits for readiness probe to pass.
- Old pods (current RS) get SIGTERM in order.
- Service endpoints update as pods become Ready / NotReady.
- Per `maxSurge`/`maxUnavailable` in the Deployment's update strategy.

So yes — the *pods* roll over via rolling update during a rollback, but the *ReplicaSets* are typically reused.

**`revisionHistoryLimit` matters:**
```yaml
spec:
  revisionHistoryLimit: 10    # keep last 10 RSes for rollback
```

Setting this to 0 means no rollback history; old RSes are deleted immediately. Setting it too high uses extra cluster resources tracking ancient ReplicaSets.

**Architect's framing:**
> "The Deployment controller uses the pod template hash as the ReplicaSet identity. Rolling back to a recent revision usually reuses the existing ReplicaSet — Kubernetes just scales it back up while scaling the current one down, doing a rolling update of pods between them. A new ReplicaSet is created only if the rolled-back-to template doesn't match any retained RS, which happens when the rollback target is beyond `revisionHistoryLimit`. The pods always do a rolling transition; the ReplicaSets are usually reused. This is why `kubectl rollout undo` is so fast — there's no new image pull, no new RS creation, just a flip in scale numbers."

---

## 13. Effect of too many lines in a Dockerfile

**Each instruction creates a layer.** More instructions → more layers → larger image, slower builds, more cache complexity, more attack surface.

**Specific impacts:**

**1. Larger image size**

Every layer adds metadata overhead and stored deltas. Even when individual instructions don't write much, the layer manifest grows. A 100-instruction Dockerfile can produce an image with significant overhead from layer metadata alone.

More critically: layers are *additive*. Removing a file in a later layer doesn't shrink the image — the file is still in the earlier layer's filesystem snapshot.

```dockerfile
# BAD — secret remains in image even though it's deleted
RUN curl -o secret.tar https://example.com/secret -H "Auth: $TOKEN"
RUN tar -xf secret.tar
RUN rm secret.tar    # ← secret.tar is still in the earlier layer
```

**2. Slower builds**
- Each layer has setup/teardown overhead.
- More layers = more cache lookups.
- A 50-step Dockerfile rebuilds slower than a 10-step equivalent doing the same work.

**3. Worse cache utilization**
- More layers = more cache invalidation surface.
- A change to step 20 invalidates steps 21-50.
- Fewer well-organized layers = better cache hits.

**4. Push/pull performance**
- Each layer fetched/pushed separately.
- Many small layers have more HTTP overhead than fewer large layers.
- Registry costs grow with layer count and total size.

**5. Increased attack surface**
- Each layer can contain forgotten artifacts.
- Build secrets accidentally written to layers stay there even if "removed" later.
- Vulnerability scanners report findings per layer; more layers = more noise.

**6. Layer count practical limit**
- Older Docker (overlay2 driver): 125 layer limit.
- Modern Docker: no hard limit but performance degrades.

**How to reduce:**

**Combine related RUN commands:**
```dockerfile
# BAD — three layers, apt cache left in image
RUN apt-get update
RUN apt-get install -y curl jq
RUN rm -rf /var/lib/apt/lists/*

# GOOD — one layer, no leftover cache
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl jq && \
    rm -rf /var/lib/apt/lists/*
```

**Use multi-stage builds** — discard everything you don't need:
```dockerfile
FROM maven:3.9 AS build
# ... 30 lines of build setup — discarded
RUN mvn package

FROM eclipse-temurin:17-jre-alpine
COPY --from=build /build/target/app.jar /app/app.jar
ENTRYPOINT ["java","-jar","/app/app.jar"]
# Final image has only what it needs
```

**Use `.dockerignore`** to keep build context small.

**Architect's framing:**
> "Layer count is a proxy for discipline. A 50-line Dockerfile usually means someone added instructions over time without consolidating, leaving artifacts in earlier layers that bloat the image and create security risk. The fix is multi-stage builds and combining related RUN commands. Mental model: each RUN should be one atomic unit of work that cleans up after itself within the same layer."

---

## 14. Copy-on-write layer

**Copy-on-write (CoW)** is the foundational mechanism of container filesystems and how Docker images work efficiently.

**The model:**

Container images are stacks of immutable read-only layers. When a container runs:
- All image layers are mounted read-only.
- A new **writable layer** is added on top — the "container layer" or "copy-on-write layer."
- The union of these layers is what the container sees as its filesystem.

When the container modifies a file:
1. The file is searched through the layer stack.
2. If found in a read-only layer, it's **copied up** to the writable layer.
3. The modification is made to the copy.
4. Subsequent reads return the writable layer's copy.

This is the "copy on write" part — files are only copied when written to, not when read.

**Implications:**

**1. Image storage efficiency**
Multiple containers from the same image share read-only layers. 100 containers from `nginx:latest` consume only one copy of the image layers + 100 small writable layers (only modifications).

**2. Container startup speed**
No copying of image content at start time — just mount existing read-only layers and add a writable layer. Starting a container is essentially "create an empty writable layer and start the process."

**3. Layer reuse across images**
If two images share base layers (`ubuntu:22.04`), those layers are stored once. Pulling an image only downloads layers not already cached.

**4. Write performance**
- First write to a file in a read-only layer is slower (copy-up).
- Subsequent writes are normal speed.
- For write-heavy containers, mount volumes (bypass CoW) for busy paths.

**5. Writable layer dies with the container**
- Modifications in the CoW layer are *not* persisted.
- Removing a container removes the CoW layer.
- For persistence, use volumes — they bypass the CoW filesystem.

**Storage drivers implementing CoW:**

| Driver | Approach |
|---|---|
| **overlay2** | Current default. Union mount of lower (read-only) and upper (writable) directories. File-level CoW. |
| **devicemapper** | Block-level CoW (deprecated). |
| **btrfs / zfs** | Filesystem-native CoW snapshots. |
| **aufs** | Older union filesystem (deprecated). |

**Practical visualization:**
```
Container's view (what the process sees):
+--------------------------------------+
| Writable layer (CoW, ephemeral)      |
+--------------------------------------+
| Image layer: app source              |  ←  read-only
+--------------------------------------+
| Image layer: dependencies            |  ←  read-only
+--------------------------------------+
| Image layer: OS packages             |  ←  read-only
+--------------------------------------+
| Image layer: base OS (alpine, etc.)  |  ←  read-only
+--------------------------------------+
```

**Architect's framing:**
> "CoW is why containers are operationally cheap — fast startup, low storage overhead, efficient image distribution. It's also why you should never store important data in the container filesystem — it lives in the ephemeral CoW layer and disappears with the container. The CoW model also explains why `rm secret.tar` after the file was added in an earlier layer doesn't reduce image size: the earlier layer is read-only and immutable; the `rm` just adds a 'whiteout' marker in a later layer hiding the file from view, but the underlying layer still contains it."

---

## 15. Docker networks I've worked with

Docker provides several network drivers. The ones I've used in production and lab:

**1. Bridge** — Docker's default network driver.
- Creates a private internal network on the host (default `172.17.0.0/16`).
- Containers communicate via the bridge; outbound to internet via host NAT.
- On user-defined bridges, containers can resolve each other by name via embedded DNS.

```bash
docker network create app-net
docker run -d --network app-net --name api myapi
docker run -d --network app-net --name db postgres
# api can reach db via the hostname "db"
```

Used for: local dev, Docker Compose, single-host multi-container setups.

**2. Host** — container shares the host's network namespace.
- No network isolation.
- Container uses host IP directly; binding to port 80 binds to host's port 80.
- Best raw performance.

```bash
docker run --network host nginx
```

Used for: performance-sensitive workloads where isolation isn't needed, system services. Caution: port conflicts with host, no namespace isolation.

**3. None** — no network at all, only loopback.
- Useful for batch jobs that don't need network.

```bash
docker run --network none my-batch-job
```

**4. Overlay** — multi-host network for Docker Swarm or third-party orchestrators.
- VXLAN encapsulation for inter-host communication.
- Mostly used with Swarm; for Kubernetes, the CNI plugins serve this role.

**5. Macvlan** — container gets its own MAC address, appears as a physical device on the network.
- Container has a real IP on the parent network.
- Bypasses Docker's NAT; container directly addressable from LAN.

Used for: legacy apps that need a real IP, network monitoring tools.

**6. ipvlan** — similar to macvlan but containers share the parent's MAC, differentiated by IP.
- Better for environments with switch-level MAC limits.

**In Kubernetes context** — the Docker network model is mostly replaced by **CNI (Container Network Interface)** plugins:

- **AWS VPC CNI** — each pod gets a VPC IP from the node's subnet. Pods are first-class VPC citizens.
- **Calico** — overlay (VXLAN/IPIP) or BGP-routed networking, strong NetworkPolicy enforcement.
- **Cilium** — eBPF-based, includes L7 policy, observability via Hubble.
- **Flannel** — simple overlay, less common now.

**Docker Compose internal patterns I've used:**

```yaml
networks:
  frontend:    # public-facing
  backend:     # internal services
  database:    # data tier only

services:
  app:
    networks: [frontend, backend]
  db:
    networks: [database, backend]      # only app + db can talk
  cache:
    networks: [backend]
```

This three-tier-on-one-host pattern is common for local dev mirroring production architecture.

**Architect's framing:**
> "In production on Kubernetes, the relevant question is CNI, not Docker network drivers — AWS VPC CNI for tight VPC integration, Cilium for eBPF-based L7 policies. For local dev and CI, user-defined bridge networks via Docker Compose serve the purpose. I've used host network for diagnostic tools, macvlan rarely for legacy apps needing real LAN IPs. The decision is: how much isolation do I need, what address space do I want containers on, what policy enforcement at L3/L4/L7."

---

## 16. How to decrease Docker image size

Layered approach, biggest wins first:

**1. Multi-stage builds — by far the biggest single win**

```dockerfile
# Build stage — has compilers, build tools, source
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app

# Final stage — only the binary
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

A Go app: build image ~900 MB, final image ~20 MB. Massive reduction.

**2. Use minimal base images**

| Base | Size | Trade-off |
|---|---|---|
| `ubuntu:22.04` | ~77 MB | Familiar but bloated |
| `debian:bookworm-slim` | ~30 MB | Reasonable middle ground |
| `alpine:3.20` | ~5 MB | musl libc (gotchas with Python, glibc-linked binaries) |
| `gcr.io/distroless/static` | ~2 MB | No shell, no package manager — best for static binaries |
| `scratch` | 0 bytes | For statically-linked binaries only |

**3. Combine RUN commands, clean up in the same layer**

```dockerfile
# BAD
RUN apt-get update
RUN apt-get install -y python3 python3-pip
RUN pip install requests
# apt cache is in a layer; can't be removed by later RM

# GOOD
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 python3-pip && \
    pip install --no-cache-dir requests && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**4. Don't install build tools in the runtime image**

Common offenders to remove from runtime images: `git`, `gcc`, `make`, `python3-dev`, `build-essential`. These belong in the build stage only.

**5. Use `.dockerignore`**

```
.git
.github
node_modules
target
dist
*.log
.env*
Dockerfile
README.md
*.md
.vscode
.idea
```

Without this, the entire `.git` directory (potentially hundreds of MB) becomes part of the build context.

**6. Choose package install flags wisely**

```dockerfile
RUN apt-get install -y --no-install-recommends ...   # no recommended packages
RUN pip install --no-cache-dir ...                    # no wheel cache
RUN npm ci --omit=dev && npm cache clean --force     # prod deps only, clean cache
```

**7. Use BuildKit cache mounts for package managers**

```dockerfile
# syntax=docker/dockerfile:1.6
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

Cache stored separately, not baked into the image layer.

**8. Squash layers if all else fails** (use sparingly)

```bash
docker build --squash -t myapp:1.0 .
```

Squashes the final image into one layer. Trade-off: loses layer caching benefits. Only useful for final distribution image.

**9. Profile what's in the image**

```bash
docker history myapp:1.0           # layer sizes
dive myapp:1.0                     # interactive layer-by-layer browser
```

Dive shows which files are in which layer and a wasted-space score — invaluable for finding bloat.

**10. Compress assets**

```dockerfile
# Pre-gzip static assets so nginx can serve them directly
RUN find /usr/share/nginx/html -type f -regex ".*\.\(html\|css\|js\)$" \
    -exec gzip -9 -k {} \;
```

**Realistic before/after I've seen:**

| Workload | Before | After | Approach |
|---|---|---|---|
| Java Spring Boot | 850 MB | 240 MB | Multi-stage, JRE-alpine, dep layer separation |
| Node.js API | 1.2 GB | 180 MB | Multi-stage, alpine, deps cleanup, no devDependencies |
| Python Flask | 1 GB | 110 MB | Multi-stage with venv copy, slim base, no caches |
| Go service | 900 MB | 18 MB | Multi-stage, distroless static |

**Architect's framing:**
> "Image size optimization is essentially: only ship what's needed to run. Multi-stage builds enforce this by construction — you literally can't accidentally include build tools because they're in a discarded stage. After that, base image choice and combined RUN commands handle most of the remaining bloat. I aim for under 250MB for JVM apps, under 100MB for Node/Python, under 50MB for Go — and use `dive` to verify there's no surprise waste."

---

## 17. Benefits of storing Docker images in ECR — and where I store them

**Primary registries I've used:**

| Registry | When |
|---|---|
| **AWS ECR** | AWS workloads (EKS, ECS, Fargate, Lambda containers) |
| **Azure Container Registry (ACR)** | Azure workloads (AKS, App Service, Container Apps) |
| **Google Artifact Registry** | GCP workloads |
| **JFrog Artifactory** | Multi-cloud, enterprise with unified artifact strategy |
| **Docker Hub** | Public OSS images we publish |
| **Harbor** | On-prem or self-hosted enterprise |

**ECR benefits specifically:**

**1. Tight IAM integration**
- Pull permissions via IAM roles, not username/password.
- EKS pods authenticate via IRSA — no `imagePullSecrets` to manage.
- ECR repo policies for cross-account access without static credentials.

**2. Private by default**
- All ECR repos start private; no anonymous pull.
- ECR Public is a separate service for public OSS images.

**3. Built-in vulnerability scanning**
- **Basic scanning** (free): scans on push, uses Clair CVE database.
- **Enhanced scanning** (paid, via Amazon Inspector): continuous scanning of stored images, OS + language packages, deeper CVE detection.

**4. Encryption at rest**
- AES-256 by default; customer-managed KMS keys supported.
- Encryption in transit via TLS.

**5. Replication and lifecycle**
- **Cross-region replication** — push to one region, image auto-replicated to others. Critical for multi-region DR.
- **Lifecycle policies** — auto-delete old/untagged images by age or count.

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Delete untagged images after 14 days",
    "selection": { "tagStatus": "untagged", "countType": "sinceImagePushed", "countUnit": "days", "countNumber": 14 },
    "action": { "type": "expire" }
  }]
}
```

**6. Tag immutability**
- Per-repo setting: when enabled, tags can't be overwritten. Production tags protected from accidental updates.

**7. Pull-through cache**
- ECR can proxy and cache images from Docker Hub, ECR Public, Quay, GitHub Container Registry. Reduces dependency on external registries, avoids rate limits.

**8. Performance from AWS workloads**
- Pulls from EC2/EKS/ECS in the same region are over AWS's internal network — fast, no internet egress charges.
- VPC endpoints (`com.amazonaws.region.ecr.api` and `com.amazonaws.region.ecr.dkr`) for fully private pulls.

**9. OCI artifact support**
- ECR stores not just container images but also Helm charts (as OCI artifacts), allowing one registry for all artifacts.

**10. Cost predictability**
- Pricing: ~$0.10 per GB-month storage, free data transfer to AWS workloads in the same region. Predictable; no per-pull charges within AWS.

**11. Audit and governance**
- CloudTrail logs every push/pull/delete.
- Repository policies grant fine-grained access.
- Resource-level IAM for which repos which users/services can read or write.

**12. Multi-architecture image support**
- Manifest lists for amd64 + arm64 images under a single tag (good for Graviton workloads).

**Decision framework: where I'd store**

- **AWS-primary, EKS workloads**: ECR. Tight integration is worth it.
- **Multi-cloud**: Artifactory or Harbor as a unified registry, replicating to cloud-native registries (ECR, ACR) for performance.
- **OSS we publish**: Docker Hub + GitHub Container Registry for community reach.
- **On-prem regulated**: Harbor with image signing (Cosign), behind the corporate firewall.

**Architect's framing:**
> "For AWS workloads, ECR is the obvious answer — IAM-native auth, private by default, replication for DR, scanning built in, best performance from AWS compute. The decision becomes interesting in multi-cloud setups where unified governance matters more than per-cloud optimization. In those cases, Artifactory or Harbor as the source of truth, with replication to ECR/ACR/GCR for runtime performance, is the pattern."

---

## 18. How many stages in Jenkins?

This question is asking me to describe a real pipeline structure, not just a number. The honest answer:

> "It varies by project, but a representative production pipeline has 8-12 stages. Here's a typical Jenkinsfile structure:"

```groovy
@Library('platform-shared-lib@v2.0.0') _

pipeline {
  agent { label 'docker-builder' }

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '50'))
  }

  environment {
    REGISTRY  = "123456789012.dkr.ecr.us-east-1.amazonaws.com"
    APP       = "myapp"
    IMAGE_TAG = "${env.BRANCH_NAME}-${env.GIT_COMMIT.take(7)}-${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Validate') {
      parallel {
        stage('Lint') {
          steps { sh 'mvn -B checkstyle:check' }
        }
        stage('Dependency Audit') {
          steps { sh 'mvn -B org.owasp:dependency-check-maven:check' }
        }
      }
    }

    stage('Build & Unit Test') {
      steps {
        sh 'mvn -B clean verify'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
          publishCoverage adapters: [jacocoAdapter('target/site/jacoco/jacoco.xml')]
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar-prod') { sh 'mvn sonar:sonar' }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Image') {
      steps {
        sh "docker build -t ${REGISTRY}/${APP}:${IMAGE_TAG} ."
      }
    }

    stage('Image Scan') {
      steps {
        sh "trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed ${REGISTRY}/${APP}:${IMAGE_TAG}"
      }
    }

    stage('Sign Image') {
      steps {
        sh "cosign sign --yes ${REGISTRY}/${APP}:${IMAGE_TAG}"
      }
    }

    stage('Push to ECR') {
      steps {
        withAWS(role: 'arn:aws:iam::123:role/jenkins-ecr-push', region: 'us-east-1') {
          sh "aws ecr get-login-password | docker login --username AWS --password-stdin ${REGISTRY}"
          sh "docker push ${REGISTRY}/${APP}:${IMAGE_TAG}"
        }
      }
    }

    stage('Update GitOps Repo') {
      when { branch 'develop' }
      steps {
        bumpGitOpsImageTag(env: 'dev', app: env.APP, tag: env.IMAGE_TAG)
      }
    }

    stage('Smoke Test') {
      when { branch 'develop' }
      steps {
        runSmokeTests(env: 'dev')
      }
    }

    stage('Approval — Production') {
      when { branch 'main' }
      steps {
        timeout(time: 24, unit: 'HOURS') {
          input message: 'Promote to production?', submitter: 'release-managers'
        }
      }
    }

    stage('Deploy Production') {
      when { branch 'main' }
      steps {
        bumpGitOpsImageTag(env: 'prod', app: env.APP, tag: env.IMAGE_TAG)
      }
    }
  }

  post {
    always { cleanWs() }
    success { notifySlack(channel: '#deploys', status: 'success') }
    failure { notifySlack(channel: '#deploys', status: 'failure') }
  }
}
```

**The 12 stages above, summarized:**

1. **Checkout** — clone the repo.
2. **Validate (parallel)** — lint + dependency audit.
3. **Build & Unit Test** — compile, run tests, publish results.
4. **SonarQube Analysis** — code quality + SAST.
5. **Quality Gate** — block on Sonar gate failure.
6. **Build Image** — docker build.
7. **Image Scan** — Trivy CVE check.
8. **Sign Image** — Cosign signature.
9. **Push to ECR** — ship the artifact.
10. **Update GitOps Repo** — bump dev tag (auto on develop).
11. **Smoke Test** — verify dev deploy works.
12. **Approval + Deploy Prod** — gated production promotion.

**Principles I'd articulate:**

- **Parallel where possible** — lint + dependency audit don't need to be sequential.
- **Fail fast** — Quality Gate before image build; no point building a Docker image if SonarQube already said the code's broken.
- **Build once, deploy many** — same image artifact deploys to dev, staging, prod. No rebuilds.
- **GitOps separation** — Jenkins commits to a GitOps repo; ArgoCD does the actual cluster deploy. Jenkins doesn't have prod cluster credentials.
- **Approval gate for prod** — `input` step with submitter restriction.
- **Shared library** (`@Library`) — common steps (`bumpGitOpsImageTag`, `runSmokeTests`, `notifySlack`) live in a versioned library, not copy-pasted across 50 Jenkinsfiles.

**The honest interview answer:**
> "How many stages depends on what's needed for the project — I've seen everything from 3 stages (build, test, deploy) for an internal tool to 20+ stages for a regulated workload with multiple security gates, compliance checks, and multi-region promotion. The number isn't the goal; the right granularity is the goal. Each stage should represent a logical unit that can fail independently with a clear failure reason. If a stage does too much, failures are hard to diagnose; if you have too many stages, you're paying setup overhead repeatedly. For a typical microservice production pipeline, 8-12 stages is the right neighborhood."

---

That covers all 18 questions. Likely follow-ups based on these:

- **"You mentioned Strimzi for Kafka — how do you handle a Strimzi upgrade?"** — operator upgrade first, then Kafka upgrade via the CR, rolling restart with monitoring.
- **"Show me a `Chart.yaml` and how chart version differs from app version."** — chart version bumps on chart changes; appVersion tracks the actual application version.
- **"How do you decide between Helm and Kustomize?"** — Helm for templated packages with values, Kustomize for declarative overlays. Often both — Helm-render the chart, Kustomize-overlay environment specifics.
- **"What's in your shared Jenkins library?"** — common deployment steps, notification handlers, secret-fetching wrappers, standard build-test-package flow per language stack.