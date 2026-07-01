# PwC DevOps Engineer Interview — Complete Answers

Both rounds covered, with the depth a senior interview expects.

---

# 🎯 Round 1: Screening

## 1. Current project architecture and your role

This needs to be your own story, but here's the structure that works:

> "I'm a Senior DevOps Engineer on a platform team supporting [N] product squads. The platform runs on **AWS EKS across two regions** in active-active configuration, fronted by **Route 53 + CloudFront** for global routing. Workloads include ~[N] microservices (Java Spring Boot, Node.js, Python) and a few stateful components — RDS Postgres Multi-AZ, ElastiCache Redis, S3, SQS, MSK.
>
> **CI/CD flow:** GitHub → GitHub Actions for build/test/scan → push to **ECR** → update image tag in a GitOps repo → **ArgoCD** reconciles to EKS via Helm. Infrastructure is fully managed via **Terraform** with remote state in S3 + DynamoDB locking, per-environment state files.
>
> **Observability:** Prometheus + Grafana for metrics, Loki for logs, Tempo for traces, all instrumented via OpenTelemetry. PagerDuty for paging.
>
> **My role:** I own the platform end-to-end — designing the Terraform module library, owning the CI/CD framework as a shared library, leading incident response, and driving SLO-based reliability practices. I also mentor app teams on Kubernetes patterns and review their Helm charts."

Keep it under 90 seconds, anchor in concrete scale and tools, end on a personal-impact note.

## 2. DevOps tools used in the last 2 years

I'd group rather than list-dump:

- **CI/CD:** Jenkins, GitHub Actions, Azure DevOps
- **IaC:** Terraform (primary), Ansible (for VM config), Helm, Kustomize
- **Containers & orchestration:** Docker, Kubernetes (EKS/AKS), Argo CD, Argo Rollouts
- **Code quality & security:** SonarQube, Snyk, Trivy, Checkov, tfsec, Cosign
- **Observability:** Prometheus, Grafana, Loki, Tempo, ELK, CloudWatch, OpenTelemetry
- **Artifact management:** ECR, ACR, JFrog Artifactory
- **Source control:** Git (GitHub Enterprise, Bitbucket)
- **Secrets:** AWS Secrets Manager, HashiCorp Vault, External Secrets Operator
- **Scripting:** Bash, Python (boto3), Groovy (Jenkins shared libraries)

I'd name 6–8 I've used deepest. Padding the list invites trap follow-ups on something you've barely touched.

## 3. AWS services used in production

| Layer | Services |
|---|---|
| Compute | EC2, EKS, ECS Fargate, Lambda |
| Storage | S3, EBS, EFS |
| Database | RDS (Postgres/MySQL), Aurora, DynamoDB, ElastiCache |
| Networking | VPC, ALB, NLB, CloudFront, Route 53, Transit Gateway, VPC Endpoints, PrivateLink |
| Identity & security | IAM, IAM Identity Center, KMS, Secrets Manager, Parameter Store, ACM, WAF, GuardDuty, Security Hub |
| Containers | ECR (image storage + scanning) |
| Messaging | SQS, SNS, EventBridge, MSK |
| CI/CD adjacent | CodePipeline, CodeBuild, CodeDeploy (when not using GitHub Actions) |
| Observability | CloudWatch (Logs, Metrics, Alarms), X-Ray |
| Governance | Organizations, Control Tower, Config, CloudTrail, Cost Explorer, Budgets |

## 4. CI/CD pipeline experience

Standard structure:

> "I've designed pipelines from scratch for Java, Node.js, Python, and .NET workloads. The pattern I follow: **build once, promote the artifact**.
>
> A typical pipeline: PR triggers → checkout, lint, unit tests, SonarQube + quality gate, dependency scan (Snyk), build artifact (JAR / Docker image), Trivy CVE scan, sign with Cosign, push to ECR. Then a separate CD flow updates the image tag in the GitOps repo, and Argo CD reconciles to the target environment.
>
> Promotions are gated: dev is auto, staging requires a passing smoke test, prod needs human approval. For prod we use Argo Rollouts canary with automated SLO-based rollback — error rate or p95 latency regression aborts the rollout."

Key principles to mention: build-once-promote, fail fast, security at every stage, GitOps for the deploy half, automated rollback.

## 5. Exposing a Kubernetes application to external traffic

Options ordered from simplest to most production-grade:

1. **NodePort** — exposes a port on every node (30000–32767). Useful for dev/testing; not for prod (no LB, exposes node IPs, ephemeral).

2. **LoadBalancer Service** — provisions a cloud LB (NLB/ALB on AWS) pointing at the service. One LB per service. Fine for a small number of services; expensive at scale.

3. **Ingress Controller** (recommended for HTTP/HTTPS):
   - Install NGINX Ingress (or AWS Load Balancer Controller for native ALB).
   - One ALB/NLB shared across many services.
   - Path- and host-based routing, TLS termination, WAF integration.
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: myapp
     annotations:
       kubernetes.io/ingress.class: alb
       alb.ingress.kubernetes.io/scheme: internet-facing
       alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
       alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
   spec:
     rules:
     - host: myapp.example.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend: { service: { name: myapp, port: { number: 80 } } }
   ```

4. **Service Mesh Gateway** (Istio, Linkerd) — when a mesh is already in play; adds mTLS, traffic shifting, circuit breaking.

5. **Global pattern** — CloudFront (edge/CDN/WAF) → ALB (regional) → Ingress (in-cluster) for tier-1 production.

## 6. Deployment vs StatefulSet

| | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Interchangeable; random names (`app-7f5b-x9k2`) | Stable, ordered (`db-0`, `db-1`, `db-2`) |
| Storage | Shared or ephemeral | Per-pod PersistentVolume via `volumeClaimTemplates` |
| Startup order | Parallel | Sequential, ordered |
| Termination order | Parallel | Reverse order |
| DNS | Single service DNS | Per-pod DNS (`db-0.db-svc.namespace.svc.cluster.local`) |
| Use case | Stateless apps — web, API | Stateful — databases, Kafka, Elasticsearch, Zookeeper |
| Scaling | Trivial — add/remove anywhere | Careful — adds/removes from the end |

Rule of thumb: stateless = Deployment. Anything with persistent identity or per-replica state = StatefulSet. But in practice, **avoid running databases on Kubernetes for production** unless you have strong operational maturity — use managed RDS/Aurora/Cloud SQL and let StatefulSets handle things like Kafka, Elasticsearch, or specialty workloads with good operators.

## 7. ConfigMap vs Secret

- **ConfigMap** — non-sensitive config (log levels, feature flags, URLs). Plain text in etcd.
- **Secret** — sensitive data (passwords, tokens). Base64-encoded (which is *not* encryption — that requires `--encryption-provider-config` on the API server or external KMS).

Both can be consumed identically: as env vars, mounted as volume files, or in command args.

```yaml
envFrom:
- configMapRef: { name: app-config }
- secretRef: { name: app-secret }
```

For production secrets I prefer **External Secrets Operator** or **CSI Secrets Store driver** pulling from AWS Secrets Manager / Azure Key Vault / HashiCorp Vault — secrets stay in the dedicated KMS with rotation and audit, and never land in Git.

## 8. Purpose of a NAT Gateway

A NAT Gateway lets resources in **private subnets** make **outbound** connections to the internet (or other VPCs over public endpoints) while remaining **un-reachable from the internet**.

Used for: EC2 in private subnets pulling OS packages, calling third-party APIs, hitting public AWS service endpoints when VPC endpoints aren't configured.

Key facts:
- **Managed service** (vs the older NAT Instance — manual, single point of failure).
- **AZ-scoped** — one per AZ for high availability; a NAT Gateway in AZ-A failing takes down outbound for AZ-A only if you deployed one per AZ.
- **Costs:** hourly charge + per-GB data processed. NAT egress costs surprise everyone — for high-traffic outbound workloads, **VPC Endpoints** for AWS services often save substantial money by bypassing NAT entirely.
- **Direction:** outbound only. Cannot initiate inbound connections.

## 9. Checking network connectivity between two servers

Layered approach by OSI level:

```bash
# Layer 3 — basic IP reachability (ICMP)
ping -c 4 <host>

# Layer 4 — specific port reachability (TCP)
nc -zv <host> 443
telnet <host> 443       # legacy but universally available
nmap -p 443 <host>

# Path tracing
traceroute <host>
mtr <host>              # combines ping + traceroute, very useful

# DNS
nslookup <host>
dig <host>
host <host>

# Endpoint test (Layer 7)
curl -v https://<host>/health
curl -I --connect-timeout 5 https://<host>

# Inside the host — what's listening?
ss -tuln                # modern replacement for netstat
netstat -tuln
lsof -i :443

# In AWS — additional checks
# - Security groups (stateful, allow rules only)
# - NACLs (subnet-level, stateless — both directions matter)
# - Route tables (correct next hop?)
# - VPC Reachability Analyzer  ← my favorite for complex paths
```

Order I follow: DNS → ICMP → TCP port → application response. Skip ICMP if firewalls drop it; jump to `nc -zv` for TCP. In AWS specifically, **Reachability Analyzer** in the VPC console is criminally underused — it tells you exactly which SG/NACL/route is blocking.

## 10. Checking running processes in Linux

```bash
# Snapshot view
ps aux | head
ps -ef
ps -ef | grep nginx
ps auxf                 # tree view

# Interactive, refreshing
top
htop                    # nicer; install separately
btop                    # even nicer modern tool

# Process tree
pstree -p

# Per-user
ps -u <username>

# Sort by memory or CPU
ps aux --sort=-%mem | head
ps aux --sort=-%cpu | head

# By systemd unit
systemctl status nginx
systemctl list-units --type=service --state=running

# What's a process actually doing
strace -p <pid>          # syscalls
lsof -p <pid>            # open files/sockets
cat /proc/<pid>/status

# Find a process by port
ss -tlnp | grep :443
lsof -i :443
fuser 443/tcp
```

For ad-hoc troubleshooting `htop` is what I reach for; for scripts and CI it's `ps -ef | grep`.

## 11. Finding files larger than 100MB

```bash
find /path -type f -size +100M

# With human-readable details and sorted by size
find /path -type f -size +100M -exec ls -lh {} \; | awk '{print $5, $9}' | sort -hr

# Top 20 biggest files anywhere
find / -type f -size +100M 2>/dev/null -exec du -h {} + | sort -hr | head -20

# Directory-level — du is usually what you actually want
du -ah /path | sort -hr | head -20
du -sh /path/*

# Visual disk usage explorer
ncdu /path

# Exclude directories you don't care about
find / -type f -size +100M -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
```

In an interview, the answer they're looking for is the first line. Mentioning `ncdu` and `du` shows you've actually used these tools in anger.

---

# 🎯 Round 2: Technical

## 1. Cross-account S3 access — App in Account A, bucket in Account B

Two roles to think about: **the principal** doing the accessing (in Account A) and the **resource** being accessed (in Account B). Both sides need to allow the access.

**Pattern 1: Cross-account IAM role assumption** (cleanest, most common)

**In Account B (bucket owner)** — create an IAM role that trusts Account A:

```json
// Trust policy on role in Account B
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::ACCOUNT_A_ID:root" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "shared-secret-uuid" }
    }
  }]
}
```

```json
// Permissions policy on role in Account B
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::shared-bucket",
      "arn:aws:s3:::shared-bucket/*"
    ]
  }]
}
```

**In Account A** — the app's role can assume that role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::ACCOUNT_B_ID:role/CrossAccountS3Access"
  }]
}
```

App code uses STS to assume the role and gets temporary credentials.

**Pattern 2: Bucket policy + principal in Account A** (simpler, fewer hops)

**Bucket policy on the S3 bucket in Account B:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::ACCOUNT_A_ID:role/AppRole" },
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::shared-bucket",
      "arn:aws:s3:::shared-bucket/*"
    ]
  }]
}
```

**IAM role in Account A** also needs to allow the same actions on that bucket (S3 cross-account requires *both* identity policy and resource policy to allow).

**Watch out for:**
- **Object ownership**: by default, objects uploaded by Account A's role are owned by Account A. Set `s3:x-amz-acl: bucket-owner-full-control` on PUT, or enable **Bucket Owner Enforced** ownership (recommended) — ACLs go away and the bucket owner always owns objects.
- **KMS** — if the bucket uses SSE-KMS with a CMK, the key policy in Account B must also grant Account A access. Otherwise you'll get cryptic AccessDenied errors.
- **VPC endpoints** with bucket policy conditions can break cross-account if not configured properly.

For an EKS pod in Account A specifically: use **IRSA (IAM Roles for Service Accounts)** so the pod's service account maps to a role that can assume the cross-account role. No long-lived credentials anywhere.

## 2. EC2 in private subnet downloads packages without NAT Gateway

Several alternatives, ordered by cost-effectiveness:

**1. VPC Endpoints (Gateway type) for S3 and DynamoDB — free**
- Route table entry for the prefix list, traffic stays on the AWS network.
- For S3-based package mirrors (Amazon Linux's `amazon-linux-extras` repos are in S3), this single endpoint is enough.

**2. VPC Endpoints (Interface type, PrivateLink) — hourly + GB charges, still cheaper than NAT for AWS service traffic**
- For services like SSM, ECR, CloudWatch Logs, Systems Manager.
- For container image pulls from ECR: configure `com.amazonaws.region.ecr.api`, `com.amazonaws.region.ecr.dkr`, and `com.amazonaws.region.s3` (ECR stores layers in S3).

**3. NAT Instance instead of NAT Gateway** — cheaper but you manage it yourself (HA, patching, throughput). Reasonable for dev/non-prod, not for production.

**4. Internal package mirror in the VPC**
- Host an apt/yum/pip/npm mirror on an EC2 in a private subnet that *does* have outbound (or sync nightly from the internet).
- Other instances pull from the internal mirror over the VPC network.

**5. Pre-baked AMI with packages already installed (Packer)**
- Treat instances as immutable. Bake the AMI with everything needed; instances just boot.
- Eliminates runtime package pulls entirely — also faster and more reproducible. My preference for production.

**6. Transit Gateway / VPC peering to a shared services VPC**
- Centralized egress VPC with the NAT Gateway, accessed via TGW.
- Cost-effective at scale — one NAT shared by many spoke VPCs.

**For the interview, lead with #1 (VPC Endpoints) — it's the answer they're looking for and is genuinely the right call for AWS-service-targeted traffic.**

## 3. Geolocation-based routing with AWS

**Route 53** is the primary service. Three relevant routing policies:

- **Geolocation routing** — route based on the user's *geographic location* (continent, country, US state). "EU users → EU endpoint, US users → US endpoint, all others → default."
- **Geoproximity routing** — route based on geographic *distance* between user and resource, with optional bias to expand or shrink a region's coverage. Requires Traffic Flow.
- **Latency-based routing** — route to the AWS region with lowest network latency for that user. Different from geo — fastest, not closest.

**Geolocation setup example:**

```hcl
resource "aws_route53_record" "eu" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  set_identifier = "eu"
  geolocation_routing_policy { continent = "EU" }
  alias {
    name                   = aws_lb.eu.dns_name
    zone_id                = aws_lb.eu.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "us" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  set_identifier = "us"
  geolocation_routing_policy { continent = "NA" }
  alias { ... aws_lb.us.dns_name ... }
}

resource "aws_route53_record" "default" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  set_identifier = "default"
  geolocation_routing_policy { country = "*" }  # catch-all
  alias { ... aws_lb.us.dns_name ... }
}
```

**Always include the default record** — without it, users in unmapped regions get nothing.

**Pair with health checks** so a failed regional endpoint routes elsewhere even if it matches the geo rule.

**For richer scenarios — CloudFront with origin failover** offers edge-based routing combined with caching and WAF, and AWS Global Accelerator gives static anycast IPs with intelligent routing. The right answer depends on whether you need data residency (geolocation), performance (latency-based), or DDoS-resistant anycast (Global Accelerator).

## 4. Dockerfile for Node.js with multi-stage build

```dockerfile
# ---------- Build stage ----------
FROM node:20-alpine AS build
WORKDIR /build

# Layer cache optimization — copy package files first
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

# Copy source and build (e.g., TypeScript compile, bundling)
COPY . .
RUN npm run build

# ---------- Dependencies stage (production only) ----------
FROM node:20-alpine AS deps
WORKDIR /deps
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --no-audit --no-fund

# ---------- Runtime stage ----------
FROM node:20-alpine AS runtime

# Non-root user — node:alpine ships with the 'node' user (uid 1000)
WORKDIR /app
ENV NODE_ENV=production \
    PORT=3000

# Install tini for proper signal handling (PID 1 + zombie reaping)
RUN apk add --no-cache tini

COPY --chown=node:node --from=deps  /deps/node_modules ./node_modules
COPY --chown=node:node --from=build /build/dist        ./dist
COPY --chown=node:node package.json ./

USER node
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/server.js"]
```

What this Dockerfile gets right:
- **Three stages** — build, prod deps, runtime — so the final image has no devDependencies, no build tools, no source files.
- **Pinned base image** with explicit Node version.
- **Layer ordering** — `package*.json` copied before source so dependency installs cache.
- **`npm ci`** instead of `npm install` — strict, reproducible, faster, fails on lockfile mismatch.
- **Non-root user**.
- **`tini` as PID 1** — handles SIGTERM properly so graceful shutdown works (without it, Node ignores SIGTERM by default in some setups).
- **HEALTHCHECK** defined.
- **No secrets** in any layer.

Companion `.dockerignore`:
```
node_modules
.git
.env
.env.*
*.log
coverage
.vscode
.idea
Dockerfile
.dockerignore
```

## 5. COPY vs ADD in Dockerfile

- **COPY** — copies files/dirs from build context into the image. Simple, predictable.
- **ADD** — does everything COPY does, *plus*:
  - Can fetch a remote URL (`ADD https://example.com/file.tar.gz /tmp/`).
  - Auto-extracts local tar archives (`ADD app.tar.gz /app` extracts into `/app`).

**Rule of thumb: always use COPY** unless you specifically need ADD's tarball auto-extraction. URL fetching via ADD is discouraged — you can't verify checksums in one step, and it's better to `RUN wget && check && extract` so the steps are explicit and cacheable.

Why this matters: ADD's behavior is non-obvious. New engineers reading your Dockerfile shouldn't have to remember that ADD might silently extract a tar. COPY is boring and predictable, which is what you want in infrastructure code.

## 6. Debugging an exited container

```bash
# 1. See exit code and reason
docker ps -a | grep <container>
docker inspect <container> --format='{{.State.ExitCode}} {{.State.Error}} {{.State.Status}}'
docker inspect <container> | jq '.[0].State'

# 2. Logs — the obvious first step
docker logs <container>
docker logs --tail 200 <container>
docker logs --since 10m <container>

# 3. Check stdout/stderr from a previous restart
docker logs <container> 2>&1 | less

# 4. Look at what's in the container at exit time
#    A container that exited still has its filesystem; commit it to a new image
#    and start it with an override command:
docker commit <container> dead-container-debug
docker run -it --entrypoint /bin/sh dead-container-debug

# 5. Re-run with overridden command for live debugging
docker run -it --entrypoint /bin/sh <image>     # bypass the broken CMD/ENTRYPOINT
docker run -it --rm <image> sh                   # if you have a shell

# 6. Restart with attached terminal to see what happens
docker run --rm -it <image>

# 7. Resource issues — OOM kills, etc.
docker inspect <container> --format='{{.State.OOMKilled}}'
# Exit code 137 = SIGKILL (often OOM), 143 = SIGTERM, 139 = SIGSEGV, 1 = generic error

# 8. Examine the image itself
docker history <image>
dive <image>     # third-party tool — shows layer contents
```

**Common exit-code meanings:**
- `0` — clean exit
- `1` — generic error (check logs)
- `125` — Docker daemon error
- `126` — command not executable
- `127` — command not found (often a missing binary in the image, or wrong PATH)
- `137` — killed by SIGKILL (usually OOM)
- `139` — segfault
- `143` — SIGTERM (graceful)

In Kubernetes the same skills apply: `kubectl logs <pod> --previous` is the equivalent of inspecting an exited container's logs. `kubectl describe pod` shows the Last State and exit reason.

## 7. Secrets for a PHP app connecting to MySQL in Docker

Layered, in order of maturity:

**Bad — but worth naming so you can explain why it's bad:**
```dockerfile
ENV DB_PASSWORD=secret123    # NEVER — secret is in image layers, visible to anyone who pulls
```

**Better — runtime env injection:**
```bash
docker run -e DB_PASSWORD="$DB_PASSWORD" myphp
docker run --env-file=./.env.prod myphp     # .env.prod gitignored
```
Better, but still: secrets visible in `docker inspect`, in env vars (any process inside the container can read), and in shell history.

**Production patterns:**

**1. Docker secrets (Swarm mode only):**
```bash
echo "secretpass" | docker secret create db_password -
docker service create --secret db_password myphp
# App reads from /run/secrets/db_password (tmpfs, not env)
```

**2. Bind-mount a secrets file** (read-only):
```bash
docker run -v /etc/myapp/secrets:/run/secrets:ro myphp
```

**3. External secrets manager — the production answer:**
- **AWS Secrets Manager** — app uses IAM role (IRSA on EKS, instance profile on EC2) to fetch at startup.
- **HashiCorp Vault** — Vault Agent sidecar fetches secrets and templates them into a file.
- **Azure Key Vault** — Managed Identity + CSI driver.
- App reads from a mounted file or fetches via SDK; secret is never in env, never in image, never in Git.

**PHP-specific example using AWS Secrets Manager + IAM:**
```php
<?php
require 'vendor/autoload.php';
use Aws\SecretsManager\SecretsManagerClient;

$client = new SecretsManagerClient([
    'version' => 'latest',
    'region'  => 'us-east-1',
]);
$result = $client->getSecretValue(['SecretId' => 'prod/mysql/credentials']);
$secret = json_decode($result['SecretString'], true);

$pdo = new PDO(
    "mysql:host={$secret['host']};dbname={$secret['dbname']}",
    $secret['username'],
    $secret['password']
);
?>
```

**Even better: skip the password entirely** — IAM database authentication for RDS MySQL/PostgreSQL. The app gets short-lived tokens via IAM; no password to manage or rotate. Best secret is no secret.

**4. BuildKit secrets** (for build-time secrets like private package registry credentials):
```dockerfile
# syntax=docker/dockerfile:1.4
RUN --mount=type=secret,id=composer_auth \
    COMPOSER_AUTH=$(cat /run/secrets/composer_auth) composer install
```
```bash
DOCKER_BUILDKIT=1 docker build --secret id=composer_auth,src=./auth.json .
```
Secret isn't stored in any layer — used during the build, then gone.

## 8. Blue-green deployment in Kubernetes

**Pattern 1: Two Deployments, one Service, label-selector flip**

```yaml
# Blue deployment (currently live)
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp-blue }
spec:
  replicas: 3
  selector: { matchLabels: { app: myapp, version: blue } }
  template:
    metadata: { labels: { app: myapp, version: blue } }
    spec:
      containers:
      - { name: app, image: myapp:1.4.2 }
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp-green }
spec:
  replicas: 3
  selector: { matchLabels: { app: myapp, version: green } }
  template:
    metadata: { labels: { app: myapp, version: green } }
    spec:
      containers:
      - { name: app, image: myapp:1.5.0 }
---
# Service — selector controls which version receives traffic
apiVersion: v1
kind: Service
metadata: { name: myapp }
spec:
  selector: { app: myapp, version: blue }   # flip this to "green" to cut over
  ports: [{ port: 80, targetPort: 8080 }]
```

**Cutover:**
```bash
# Verify green is healthy via direct check
kubectl port-forward deploy/myapp-green 8080:8080 &
curl localhost:8080/health

# Flip the service to green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# If problems → rollback by flipping back
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'

# After confirming green is stable for some time → scale down blue
kubectl scale deployment myapp-blue --replicas=0
```

**Pattern 2: Argo Rollouts (production-grade)**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: myapp }
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: myapp-active       # production traffic
      previewService: myapp-preview     # for pre-cutover testing
      autoPromotionEnabled: false       # require manual promote
      scaleDownDelaySeconds: 300        # keep old version 5min for fast rollback
      prePromotionAnalysis:
        templates: [{ templateName: smoke-tests }]
      postPromotionAnalysis:
        templates: [{ templateName: success-rate }]
```

This gives you: pre-cutover analysis (run smoke tests against the preview service before traffic flip), automated promote/abort based on metrics, retain old version for fast rollback, full audit trail in Argo CD UI.

**Pattern 3: Two Ingresses + weighted routing** — for very high traffic where the L4 service flip isn't granular enough; gives you canary-within-blue-green.

**Trade-offs vs Rolling Update:**
- **Pro:** instant rollback, full validation before traffic, no mixed-version state.
- **Con:** 2× resource consumption during transition, DB migrations must be backwards-compatible (expand/contract pattern is essential), longer-lived in-flight connections need draining.

## 9. NetworkPolicies for pod-to-pod restriction

NetworkPolicies are namespaced, label-selector-based firewall rules between pods. **Requires a CNI that enforces them** — Calico, Cilium, Weave, Antrea. AWS VPC CNI on EKS requires Calico or Cilium add-on.

**Step 1: Default-deny in the namespace**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: app
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  # No ingress/egress rules = deny all
```

**Step 2: Explicit allows**

Frontend can be reached by ingress controller only:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-from-ingress
  namespace: app
spec:
  podSelector: { matchLabels: { app: frontend } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector: { matchLabels: { name: ingress-nginx } }
    ports: [{ protocol: TCP, port: 8080 }]
```

Backend can be reached only by frontend:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-from-frontend
  namespace: app
spec:
  podSelector: { matchLabels: { app: backend } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: { matchLabels: { app: frontend } }
    ports: [{ protocol: TCP, port: 8080 }]
```

DNS egress (every namespace needs this — pods can't resolve names without it):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: app
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: { matchLabels: { name: kube-system } }
      podSelector: { matchLabels: { k8s-app: kube-dns } }
    ports:
    - { protocol: UDP, port: 53 }
    - { protocol: TCP, port: 53 }
```

Backend can egress to RDS:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-rds
  namespace: app
spec:
  podSelector: { matchLabels: { app: backend } }
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock: { cidr: 10.0.10.0/24 }    # RDS subnet
    ports: [{ protocol: TCP, port: 5432 }]
```

**Production patterns I follow:**
- Default-deny in every namespace.
- Explicit allow per traffic direction (separate ingress and egress policies).
- `kube-system` and ingress-nginx namespaces labeled (`kubectl label ns kube-system name=kube-system`) so other namespaces can reference them.
- DNS allow-egress in every namespace — the most common debugging trap.
- Use **Cilium** for L7 policies (HTTP method/path-aware, not just port) and better observability via Hubble.
- Test policies in dev with `kubectl exec` from one pod attempting connection to another.

## 10–11. Production Kubernetes incident: ImagePullBackOff + Evictions + 503s

This is the kind of multi-symptom incident that's common and worth structured triage. Walk through it like an incident commander.

**Symptom triage — these three are often *related*:**

- ImagePullBackOff → pods can't start → fewer healthy backends.
- Evictions → pods being killed → fewer healthy backends.
- 503 errors → ingress can't find healthy backends.

**Most likely root cause hypothesis**: node-level resource pressure cascading. Evictions kick in due to memory/disk pressure → new pods scheduled elsewhere → cluster-wide image pulls hit registry rate limits or out-of-disk on nodes → ImagePullBackOff. Net effect: too few ready pods → ingress returns 503.

**Diagnosis sequence:**

```bash
# 1. Get the lay of the land
kubectl get pods -A -o wide | grep -v Running
kubectl get nodes
kubectl top nodes
kubectl top pods -A --sort-by=memory

# 2. ImagePullBackOff specifically
kubectl describe pod <pod>     # look at Events section
#   - "no space left on device" → node disk full
#   - "manifest unknown" → wrong tag
#   - "unauthorized" → image pull secret problem
#   - "rate limit exceeded" → DockerHub rate limiting, or ECR throttling
#   - "no such host" → DNS/network issue reaching registry
kubectl get events --sort-by='.lastTimestamp' | grep -i pull

# 3. Evictions
kubectl get pods -A | grep Evicted
kubectl describe pod <evicted-pod>   # The "Status:" field tells you which pressure caused it
#   - "The node was low on resource: memory"
#   - "The node was low on resource: ephemeral-storage"
#   - "The node had condition: [DiskPressure]"
kubectl describe node <node>    # see conditions: MemoryPressure, DiskPressure, PIDPressure

# 4. 503s — usually means ingress has no healthy upstreams
kubectl get endpoints <service>      # are there any IPs?
kubectl get endpointslices -n <ns>
kubectl logs -n ingress-nginx <controller-pod> | grep 503

# 5. Cluster-wide health
kubectl get pdb -A
kubectl get hpa -A
kubectl get deployment -A -o wide
```

**Immediate mitigation (stop the bleeding):**

1. **If node disk is full** (ImagePullBackOff with "no space left on device" or DiskPressure):
   ```bash
   # SSH to node or use a debug pod
   crictl rmi --prune          # remove unused images
   journalctl --vacuum-size=200M
   ```
   Or simpler: cordon and drain affected nodes, let autoscaler replace them with fresh nodes.

2. **If registry is rate-limited** (DockerHub on anonymous pull is the classic): switch to authenticated pulls or mirror to ECR/private registry.

3. **If memory pressure**: increase node pool size or add larger instance type. Cordon overloaded nodes.

4. **If a runaway pod is causing pressure**: identify and limit/kill:
   ```bash
   kubectl top pods -A --sort-by=memory | head
   ```

5. **For 503s**: scale up healthy deployments (`kubectl scale --replicas=10`); restart ingress-nginx if it's in a bad state; verify endpoints are populated.

**Post-incident hardening (the "how to avoid" part):**

| Risk | Fix |
|---|---|
| Node disk fills with images | Configure kubelet GC: `image-gc-high-threshold=85, image-gc-low-threshold=70`. Set `ephemeral-storage` requests on workloads. Use larger disks. |
| Registry rate limiting | Use ECR/ACR (private, no rate limits) or DockerHub auth (Pro plan). Image mirror in-cluster for hot images. |
| Pods overcommit | Set `requests` AND `limits` on every workload. Use **VPA in recommendation mode** to right-size. **LimitRanges** per namespace to set defaults. **ResourceQuotas** to cap teams. |
| Eviction during normal operations | Use **PodDisruptionBudgets** (`minAvailable`). Set **Priority Classes** so critical workloads aren't evicted first. |
| 503s during deploys/scaling | Always set readiness probes, `terminationGracePeriodSeconds`, `preStop` hooks for connection draining. Use `topologySpreadConstraints` to avoid putting all replicas on one node/zone. |
| No early warning | Alerts on: node memory > 85%, node disk > 80%, image-pull failures > 5/min, pod eviction rate > 0, ingress 5xx rate > SLO. |
| No room to scale | **Cluster Autoscaler / Karpenter** with appropriate min/max. Overprovisioning pods (pause pods) for faster scale-up response. |
| Operators surprised by cascade | Runbook documenting these specific symptoms and triage steps. Game-day exercises. |

**Real talk** to add in the interview: I'd narrate this as "first I'd stabilize, then I'd diagnose root cause, then I'd put in preventive controls — and I'd also schedule a postmortem because three concurrent symptoms is almost never bad luck; it's a system that lacks guardrails."

## 12. Terraform state file corruption — recovery

```bash
# 1. STOP — don't run any more terraform commands that might write state
#    Take a backup of whatever state exists right now
terraform state pull > backup-broken.tfstate

# 2. Recover from S3 versioning (this is why versioning is mandatory on the state bucket)
aws s3api list-object-versions \
  --bucket company-tfstate \
  --prefix prod/network.tfstate

aws s3api get-object \
  --bucket company-tfstate \
  --key prod/network.tfstate \
  --version-id <last-known-good-version-id> \
  recovered.tfstate

# 3. Validate the recovered state is parseable
cat recovered.tfstate | jq . > /dev/null
terraform show recovered.tfstate

# 4. Force-unlock if a stuck lock exists in DynamoDB
terraform force-unlock <lock-id>   # only after verifying no apply is actually running

# 5. Push the recovered state back
terraform state push recovered.tfstate

# 6. Run plan — should show no changes if state matches reality
terraform plan
```

**If versioning wasn't enabled (or recovery from S3 isn't possible):**

```bash
# 7. Rebuild state by importing each resource
terraform import aws_vpc.main vpc-0abc123
terraform import 'aws_subnet.private[0]' subnet-0def456
# ... for each resource in the config

# Or use terraformer to bulk-discover and import
terraformer import aws --resources=vpc,subnet,ec2_instance --regions=us-east-1
```

**Prevention is the actual answer in the interview:**
- **Versioning + MFA delete** on the state bucket — non-negotiable.
- **DynamoDB locking** so concurrent applies can't corrupt state.
- **Server-side encryption** with KMS CMK.
- **Bucket policy restricting access** — only the pipeline role and a small SRE break-glass role can write.
- **Separate state per environment** so corruption in dev doesn't take down prod.
- **Never edit `.tfstate` by hand** — use `terraform state mv/rm/import` subcommands which validate.
- **Automated state backup** to a separate account/region.

## 13. Importing an existing AWS VPC into Terraform

**Step 1: Inventory the existing VPC**
```bash
aws ec2 describe-vpcs --vpc-ids vpc-0abc123 --query 'Vpcs[0]'
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-0abc123
aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-0abc123
aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=vpc-0abc123
aws ec2 describe-security-groups --filters Name=vpc-id,Values=vpc-0abc123
```

**Step 2: Write minimal Terraform resource blocks**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  for_each          = toset(["10.0.1.0/24", "10.0.2.0/24"])
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.key
  availability_zone = "us-east-1a"   # placeholder; fix per real data
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
}
```

**Step 3: Use `import` blocks (Terraform 1.5+, preferred declarative way)**
```hcl
import {
  to = aws_vpc.main
  id = "vpc-0abc123"
}

import {
  to = aws_subnet.public["10.0.1.0/24"]
  id = "subnet-0def456"
}

import {
  to = aws_internet_gateway.main
  id = "igw-0ghi789"
}
```

**Step 4: Generate config and plan**
```bash
terraform init
terraform plan -generate-config-out=generated.tf
# Review generated.tf, merge into your real config
terraform plan
```

**Step 5: Iterate until plan is clean**

The first plan after import almost always shows differences because your hand-written config doesn't match every actual attribute. Update the config until `plan` says "No changes." This step is where most of the time goes.

**Step 6: Apply (no-op) to confirm**
```bash
terraform apply
# Should show: 0 to add, 0 to change, 0 to destroy
```

**For a large existing estate:** use **terraformer** to bulk-discover and generate Terraform code + state for thousands of resources at once. The output needs cleanup but it's vastly faster than hand-writing each resource.

**Common gotchas to mention:**
- Default tags, default security groups, default route table — these often exist and need handling.
- The order of imports matters when there are dependencies; do parent (VPC) before child (subnets, RTs).
- After import, plan may want to change attributes that AWS returns differently than Terraform schema (e.g., security group rule ordering). Use `lifecycle.ignore_changes` selectively or fix the config.

## 14. Managing Terraform secrets without hardcoding

**Anti-pattern (never do this):**
```hcl
resource "aws_db_instance" "this" {
  password = "Pa$$w0rd"    # NO — visible in code, Git history, state file
}
```

**Pattern 1: Reference from a secrets manager at runtime**
```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/rds/master-password"
}

resource "aws_db_instance" "this" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
  lifecycle {
    ignore_changes = [password]   # so password rotation outside TF doesn't drift
  }
}
```

**Pattern 2: Generate and store in secrets manager**
```hcl
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "db" {
  name = "prod/rds/master-password"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}

resource "aws_db_instance" "this" {
  password = random_password.db.result
}
```
Password lives in state file but never in code or Git.

**Pattern 3: Eliminate the secret entirely**
- **IAM database authentication** for RDS Postgres/MySQL — the app uses IAM tokens; no password to manage.
- **IRSA for EKS workloads** — pod has IAM identity directly, no static creds.

**Pattern 4: Vault provider for dynamic secrets**
```hcl
data "vault_generic_secret" "db" {
  path = "secret/prod/rds"
}

resource "aws_db_instance" "this" {
  password = data.vault_generic_secret.db.data["password"]
}
```

**Pattern 5: Inject via env var, never written to disk**
```hcl
variable "db_password" {
  sensitive = true
  type      = string
}

# Pipeline runs:
# TF_VAR_db_password=$(aws secretsmanager get-secret-value ...) terraform apply
```

**Hardening the state file itself** (since it still contains secrets):
- S3 backend with **KMS encryption** using a CMK.
- **Bucket policy** restricting access to only the pipeline role.
- **Versioning + soft delete + MFA delete**.
- **Mark sensitive variables and outputs** with `sensitive = true` so they're masked in logs.
- **Don't `terraform output` secrets** in CI logs.

**Hierarchy of options to mention:**
1. Use IAM auth and skip the password (best)
2. Generate in TF, store in Secrets Manager (good)
3. Reference an externally-managed secret (good)
4. `sensitive` variable injected via env from a secrets manager (acceptable)
5. `.tfvars` file gitignored (acceptable for dev; risky for prod)
6. Hardcoded (never)

## 15. Cross-account resource provisioning with Terraform

**Pattern: Provider aliases with cross-account assume-role**

```hcl
# Main account provider — the orchestration account
provider "aws" {
  alias  = "shared"
  region = "us-east-1"
}

# Provider for account A — assumes role in Account A
provider "aws" {
  alias  = "account_a"
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::ACCOUNT_A_ID:role/TerraformExecutionRole"
    session_name = "terraform-${var.environment}"
    external_id  = var.external_id
  }
}

# Provider for account B
provider "aws" {
  alias  = "account_b"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::ACCOUNT_B_ID:role/TerraformExecutionRole"
  }
}

# Resources in account A
resource "aws_s3_bucket" "logs" {
  provider = aws.account_a
  bucket   = "company-logs-bucket"
}

# Resources in account B
resource "aws_iam_role" "consumer" {
  provider = aws.account_b
  name     = "log-consumer"
  # ... policy granting access to the bucket in account A
}

# Modules can take providers explicitly
module "shared_services" {
  source = "./modules/shared"
  providers = {
    aws.primary = aws.account_a
    aws.dr      = aws.account_b
  }
}
```

**Pre-requisites:**
- An IAM role in *each* target account with a trust policy allowing the central account/identity to assume it.
- The role has whatever permissions Terraform needs (often a custom policy, not `AdministratorAccess`).
- ExternalID condition for additional security.

**Enterprise patterns:**
- **AWS Control Tower / Organizations** — centralized account management; new accounts come with a baseline Terraform execution role.
- **Account Factory for Terraform (AFT)** — Control Tower's IaC for account vending.
- **Separate state files per account/environment** — never mix multi-account state in one file.
- **Pipeline service principal** federated via OIDC (GitHub Actions / Azure DevOps OIDC to AWS) — short-lived credentials only.

**State file storage for multi-account setups:** keep state in a dedicated **shared services / management account** — never in the same account whose resources it's managing. That account's S3 bucket and DynamoDB are the source of truth across the org.

## 16. S3 bucket drift — someone manually added a policy

This is a classic drift scenario. Three responses depending on intent:

**Step 1: Identify the drift**
```bash
terraform plan
# Output:
#   ~ resource "aws_s3_bucket_policy" "this" {
#       ~ policy = jsonencode({...changes...})
#     }
```

Or proactively detect with a scheduled job:
```bash
terraform plan -detailed-exitcode
# Exit 0 = no changes, 2 = changes detected → alert
```

**Step 2: Decide what to do**

**Option A: The manual change was wrong — revert it**
```bash
terraform apply
# Pushes the Terraform-defined policy back, overwriting the manual change
```

**Option B: The manual change was correct — adopt it**
```hcl
# Read the actual policy
aws s3api get-bucket-policy --bucket my-bucket --query Policy --output text | jq

# Update the Terraform code to match
resource "aws_s3_bucket_policy" "this" {
  bucket = aws_s3_bucket.this.id
  policy = jsonencode({
    # ... include the manually-added statements ...
  })
}

# Now plan is clean
terraform plan
```

**Option C: The change is environment-specific or managed by another system — ignore it**
```hcl
resource "aws_s3_bucket_policy" "this" {
  bucket = aws_s3_bucket.this.id
  policy = jsonencode(local.base_policy)

  lifecycle {
    ignore_changes = [policy]   # Terraform sets initial policy; doesn't manage afterwards
  }
}
```
Use sparingly — overuse defeats the purpose of IaC.

**The cultural fix (the real answer):**

- **Continuous drift detection** — scheduled `terraform plan -detailed-exitcode` in CI, alert on drift.
- **IAM lockdown** — engineers can't modify Terraform-managed resources directly; only the pipeline role has the relevant `s3:PutBucketPolicy` permission.
- **AWS Config rules** to detect bucket policy changes and notify.
- **CloudTrail event-driven alerts** on `PutBucketPolicy` events outside the pipeline IAM role.
- **Postmortem** — why did someone make a manual change? Was there an emergency where the pipeline was too slow? Fix the underlying friction.

**For tools beyond Terraform itself:**
- **driftctl** — continuous drift detection across providers, reports drift even on unmanaged resources.
- **Terraform Cloud / Enterprise** — built-in drift detection.

## 17. Python script — backup files older than 30 days

```python
#!/usr/bin/env python3
"""Backup files older than N days from a source directory to a destination."""
import argparse
import logging
import shutil
import sys
import tarfile
from datetime import datetime, timedelta
from pathlib import Path


def setup_logging(verbose: bool) -> None:
    logging.basicConfig(
        level=logging.DEBUG if verbose else logging.INFO,
        format="%(asctime)s [%(levelname)s] %(message)s",
    )


def find_old_files(source: Path, age_days: int) -> list[Path]:
    """Return files in `source` (recursive) older than `age_days`."""
    cutoff = datetime.now() - timedelta(days=age_days)
    cutoff_ts = cutoff.timestamp()
    old_files: list[Path] = []

    for entry in source.rglob("*"):
        if not entry.is_file():
            continue
        try:
            if entry.stat().st_mtime < cutoff_ts:
                old_files.append(entry)
        except OSError as e:
            logging.warning("Skipping %s: %s", entry, e)

    return old_files


def backup_to_archive(files: list[Path], source: Path, dest: Path,
                      dry_run: bool, delete_after: bool) -> Path | None:
    if not files:
        logging.info("No files older than threshold; nothing to back up.")
        return None

    dest.mkdir(parents=True, exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    archive_path = dest / f"backup-{timestamp}.tar.gz"

    logging.info("Found %d files to back up.", len(files))
    if dry_run:
        for f in files:
            logging.info("[dry-run] would archive: %s", f)
        return None

    with tarfile.open(archive_path, "w:gz") as tar:
        for f in files:
            arcname = f.relative_to(source)
            tar.add(f, arcname=str(arcname))
            logging.debug("Added: %s", arcname)

    logging.info("Archive created: %s (%.2f MB)",
                 archive_path,
                 archive_path.stat().st_size / (1024 * 1024))

    if delete_after:
        for f in files:
            try:
                f.unlink()
                logging.debug("Deleted: %s", f)
            except OSError as e:
                logging.warning("Could not delete %s: %s", f, e)
        logging.info("Deleted %d source files after archiving.", len(files))

    return archive_path


def main() -> int:
    p = argparse.ArgumentParser(description="Backup files older than N days.")
    p.add_argument("source", type=Path, help="Source directory to scan")
    p.add_argument("dest", type=Path, help="Destination for backup archive")
    p.add_argument("--days", type=int, default=30, help="Age threshold in days (default: 30)")
    p.add_argument("--delete", action="store_true", help="Delete source files after successful archive")
    p.add_argument("--dry-run", action="store_true", help="Show what would happen without doing it")
    p.add_argument("-v", "--verbose", action="store_true")
    args = p.parse_args()

    setup_logging(args.verbose)

    if not args.source.is_dir():
        logging.error("Source is not a directory: %s", args.source)
        return 1

    files = find_old_files(args.source, args.days)
    backup_to_archive(files, args.source, args.dest,
                      dry_run=args.dry_run,
                      delete_after=args.delete)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Usage:
```bash
# Dry run first
python backup.py /var/log /backups --days 30 --dry-run -v

# Real run
python backup.py /var/log /backups --days 30

# With deletion after successful backup
python backup.py /var/log /backups --days 30 --delete
```

What this script does right:
- **Dry-run mode** before destructive operations — non-negotiable for backup scripts.
- **Logging** instead of `print()` — controllable verbosity.
- **Explicit cutoff calculation** — readable and testable.
- **Tar.gz archive** with timestamp — easy to retain multiple backups.
- **Optional deletion** — separate flag, default safe (no deletion).
- **Error handling** — skips unreadable files with warnings, doesn't abort.
- **Type hints** — better readability and tooling support.
- **Pathlib** — modern, cross-platform paths.

For production I'd add: S3 upload (`boto3`), checksum verification, retention policy on the destination, and probably hand this off to **AWS Backup** / **S3 Lifecycle** rules which do this declaratively without a custom script.

## 18. Cloud cost optimization without impacting performance

I'd structure this as a **FinOps program**, not a one-off cleanup.

**Phase 1: Visibility (you can't optimize what you can't see)**
- **Tagging strategy** enforced: `CostCenter`, `Owner`, `Environment`, `Application`, `Team`. Non-tagged resources get auto-flagged or quarantined via SCPs.
- **AWS Cost Explorer + Cost and Usage Reports (CUR)** to QuickSight for custom dashboards.
- **AWS Budgets** with alerts at 50/80/100% per team/env, routed to Slack.
- **CloudHealth / Vantage / Cloudability** for cross-account aggregation if you're at scale.
- **Kubecost / OpenCost** for Kubernetes workload attribution down to namespace/pod.
- Showback or chargeback reports per team monthly.

**Phase 2: Quick wins (typically 15–30% reduction)**
- **Right-size EC2** based on CloudWatch metrics — Cost Explorer's right-sizing recommendations are pretty good.
- **Delete orphans:** unattached EBS volumes, unattached EIPs (charged when not attached), unused snapshots, old AMIs, idle load balancers, abandoned NAT Gateways.
- **Stop non-prod nightly/weekends** — dev/test environments via Lambda + EventBridge schedule, often 70% savings on dev.
- **S3 lifecycle policies** — Standard → IA → Glacier → Deep Archive based on access patterns. Intelligent-Tiering for unknown access patterns.
- **EBS gp2 → gp3** — 20% cheaper for equivalent or better performance, no downtime.
- **VPC Endpoints for S3/DynamoDB** — eliminates NAT Gateway data charges for those flows. Big saver in heavy-egress workloads.
- **CloudWatch Logs retention** — most teams default to "never expire." Set per log group based on actual needs (typically 30/90/365 days).

**Phase 3: Commitments (10–40% additional)**
- **Savings Plans** (Compute Savings Plans cover EC2 + Fargate + Lambda flexibly) — 1-year, no upfront for safety. Cover your baseline (~70–80% of steady-state usage).
- **Reserved Instances** for RDS, ElastiCache, Redshift.
- **Spot instances** for fault-tolerant workloads — batch, CI runners, stateless workers, EKS data plane via Karpenter with diversification across instance types and AZs. 60–90% discount.

**Phase 4: Architecture changes (variable but often biggest impact)**
- **Right architecture for the workload** — Lambda for spiky/bursty, Fargate Spot for batch, EC2 with Savings Plans for steady state.
- **Replace expensive managed services** where overkill — e.g., RDS for a low-traffic internal tool → Aurora Serverless v2 or even DynamoDB if access patterns allow.
- **Multi-region: do you really need active-active?** — DR with warm-standby costs significantly less and meets most RTO/RPO targets.
- **Compress and cache aggressively** — CloudFront in front of S3-backed static assets, gzip/brotli on origin.
- **Right-size Kubernetes:** VPA recommendations to right-size requests; cluster autoscaler / Karpenter to avoid idle nodes; bin-packing via PodDisruptionBudgets that allow consolidation.
- **Database tier rationalization** — Aurora vs RDS, reader replicas at right scale, archive cold data to S3.

**Phase 5: Cultural and governance**
- **FinOps reviews** — monthly meeting per team with their cost dashboard. Team accountability.
- **Cost-aware CI/CD** — `infracost` in PRs showing the cost impact of each Terraform change.
- **Service Control Policies** preventing expensive resource types in dev (no `r5.24xlarge` in non-prod, etc.).
- **Auto-shutdown defaults** for dev resources via Lambda + tags.
- **Unit economics tracking** — $/transaction, $/user — to ensure cost scales sublinearly with growth.

**What I explicitly *don't* do for cost cutting:**
- Reduce redundancy/HA in production. Outage costs >> infra savings.
- Disable monitoring/logging. Blind operations cost much more in incidents.
- Aggressive limits that cause throttling. User experience degradation has revenue cost.
- Last-minute Spot for tier-1 stateful workloads. Wrong tool.

**The framing I'd use in the interview:** cost optimization is a continuous discipline, not a quarterly fire drill. The companies that do it well treat cost as a first-class non-functional requirement — same as latency, availability, security — owned by engineering, supported by finance, not the other way around.

---

That's the full set. Likely follow-ups to be ready for:

- "Walk me through the worst production incident you've led" — have a 5–7 minute structured story (situation → action → decisions → outcome → lessons).
- "How do you decide between Terraform and CloudFormation/CDK?" — multi-cloud need, team skills, ecosystem, drift detection, modularity.
- "What's your view on GitOps vs imperative CD?" — preferred GitOps, with honest trade-offs (initial complexity, debug-ability).
- "How do you onboard a new team to your platform?" — Golden Paths, self-service templates, Backstage, paved roads not paved walls.


# Senior DevOps Interview — Tailored to Your Background

I've reviewed your CV carefully. You have 13+ years of experience with strong Azure depth, Banking/Healthcare domains, and recent senior DevOps leadership at Finastra. I'll tailor these answers to leverage your actual experience (Finastra, IKS Health, Kelp) rather than generic examples. Where AWS depth is asked but your CV shows lighter AWS exposure, I'll be honest about how to position it.

---

# ✅ Round 1 — General

## 1. Current project architecture and your role (Finastra)

> "I'm currently working as a Senior DevOps Engineer at Finastra, supporting the Global Payplus and GFX platforms — wire payment systems used by banking clients in the US, UK, and Canada. Team of 28.
>
> **The platform:**
> The applications are a mix of legacy and modern stacks — C++, VB.NET, .NET Core, and C# — running on both Windows Server and Linux environments. Banking clients have strict deployment requirements: regulatory compliance, segregation of duties, on-premises Azure DevOps Server (not cloud-hosted), and high availability for wire payment processing.
>
> **Infrastructure:**
> - Azure DevOps Server deployed on Finastra's on-premises infrastructure within Azure Cloud (private hosted, not the SaaS version).
> - AKS clusters for the modernized C++ and .NET application components.
> - Azure VMs for legacy Windows-based components.
> - Azure SQL and App Services backing the platform.
> - Terraform managing all the infrastructure provisioning.
>
> **CI/CD flow:**
> Source code recently migrated from Perforce to Azure Repos. Build pipelines defined in YAML with PowerShell for Windows-specific automation. Multi-stage pipelines build C#, C++, and .NET artifacts, package via InstallShield for client-side deployments, build Docker images for containerized components, push to ACR, then deploy via release pipelines to Dev → QA → UAT → Production.
>
> **My specific role:**
> - **Owned the Perforce-to-Azure-Repos migration** — large legacy codebase, preserved history, retrained developers on Git workflow.
> - **Designed the CI/CD pipeline architecture** for the multi-language stack — same pipeline pattern across C#, C++, and .NET, parameterized via YAML templates.
> - **Built Terraform modules** for Azure VMs, AKS, Storage Accounts, SQL, and App Services. Reusable across environments.
> - **Set up the AKS clusters** for the application's web tier, including ingress, certificate management, and Ansible-based bootstrap.
> - **Integrated SonarQube** for static code analysis with quality gates blocking the pipeline on policy violations.
> - **Embedded security testing (SAST, DAST, SCA)** into the pipelines — non-negotiable for banking compliance.
> - **Automated OS hardening and patching** for Windows and RHEL servers using Ansible — meeting CIS benchmarks and banking regulatory requirements.
> - **Lead release coordination** across DEV, QA, UAT, and Production with post-release support.
>
> **What makes this work distinctive:**
> Banking compliance shapes everything. Every change is auditable, every deployment is traceable to a work item, every secret is rotated, and the entire pipeline is designed around 'four eyes' principles for production changes. The C++ and InstallShield work is unusual in modern DevOps — most engineers focus on container-native stacks — but it's critical for the legacy components that wire payment systems still rely on."

## 2. DevOps tools used in the last couple of years

Drawing directly from your CV:

| Capability | Tools |
|---|---|
| CI/CD | Azure DevOps Pipelines, Jenkins, GitHub Actions, AWS CodePipeline |
| Source Control | Azure Repos, Git, GitHub, GitLab, Perforce (migrating away) |
| Containers & Orchestration | Docker, Kubernetes, AKS, EKS |
| Infrastructure as Code | Terraform, Ansible, ARM Templates |
| Build & Packaging | Maven, NPM, MSBuild, InstallShield |
| Scripting | PowerShell, Bash, Shell, YAML, Windows Batch, Python |
| Code Quality / Security | SonarQube, SAST/DAST/SCA integrations |
| Monitoring | Grafana, Dynatrace, Zabbix, Kibana, Posthog |
| Logging | Elastic Search, Kibana, File Beat, Logstash, Kafka, Zookeeper |
| Project Management | Azure Boards, Jira, ServiceNow, Bitnami Redmine |
| Databases | PostgreSQL, MySQL, MSSQL, Redis |
| Web Servers | NGINX, Apache |
| Cloud | Azure (primary), AWS |
| AI-assist | Copilot, GitHub Copilot |

**How I'd frame this in the interview:**

> "The combination matters more than the individual tools. Azure DevOps + Terraform + Docker is my primary stack at Finastra. Before that at Kelp, it was Jenkins + Terraform + Docker + Kubernetes + Elastic stack for observability. The principle I follow: **pick tools that match the team's operational maturity, not the latest trend**. Banking clients want stable, audited tools; startups want fast iteration. The toolchain reflects that."

## 3. AWS services used in production

This is where I'd be tactically honest given your CV — your primary depth is Azure. Frame your AWS experience accurately:

> "My AWS production experience comes primarily from the Kelp project at Full Value Technologies (2018-2023), where we had infrastructure on both Azure and AWS. The AWS services I worked with in production:
>
> | Layer | AWS services |
> |---|---|
> | Compute | EC2 instances with autoscaling, EKS |
> | Storage | S3 for object storage |
> | Networking | VPC, ELB |
> | Database | RDS |
> | File system | EFS (Elastic File System) |
> | Monitoring | CloudWatch |
> | CI/CD | AWS CodePipeline |
>
> **Specific work:**
> - Set up multi-tier application using EC2, VPC, RDS, S3 with autoscaling for the Kelp analytics platform.
> - Configured ELB for load balancing across EC2 instances.
> - Implemented CloudWatch monitoring and alarms.
> - Worked with EFS for shared file systems across EC2 instances.
> - Used S3 for application data and log archival.
> - Configured VPC with public and private subnets, NAT gateways, security groups.
>
> **Honest framing:** My deepest cloud expertise is Azure — Azure DevOps, AKS, ARM templates, Azure VMs, Azure Storage. I'm AWS SysOps Administrator certified, so my fundamentals are strong, but in the last 2 years at Finastra and IKS Health, the work has been Azure-heavy. I'm fully comfortable with AWS architectural patterns and could ramp on deeper AWS-native services like Step Functions, Lambda, or AWS-specific tooling quickly if needed."

This positions you honestly without underselling. Saying you're AWS-certified plus having multi-cloud exposure is strong; pretending you have deep recent AWS production experience when your CV doesn't show it would be risky.

## 4. Exposing Kubernetes apps to external traffic

> "Multiple patterns depending on the use case:
>
> **Pattern 1: LoadBalancer Service**
> Simplest — Kubernetes provisions a cloud load balancer (Azure Load Balancer on AKS, AWS ELB/NLB on EKS) with a public IP, traffic flows directly to the service. Used this for early stages or single-service exposure. Downside: one LB per service, doesn't scale economically.
>
> **Pattern 2: Ingress Controller (most common in my work)**
> At Finastra, we use ingress controllers fronted by Azure Load Balancer. One LB serves many services with path-based and host-based routing.
>
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: payments-api
>   namespace: payments
>   annotations:
>     cert-manager.io/cluster-issuer: letsencrypt-prod
>     nginx.ingress.kubernetes.io/ssl-redirect: "true"
> spec:
>   ingressClassName: nginx
>   tls:
>   - hosts: [api.bank.example.com]
>     secretName: api-tls
>   rules:
>   - host: api.bank.example.com
>     http:
>       paths:
>       - path: /payments
>         pathType: Prefix
>         backend:
>           service: { name: payments-service, port: { number: 80 } }
>       - path: /accounts
>         pathType: Prefix
>         backend:
>           service: { name: accounts-service, port: { number: 80 } }
> ```
>
> **Pattern 3: Azure Application Gateway Ingress Controller (AGIC)**
> For Azure-specific deployments needing WAF, Entra ID integration, deeper Azure integration. Used this at Finastra for the banking compliance posture — WAF rules at the gateway protect against OWASP Top 10.
>
> **Pattern 4: API Gateway in front**
> For external partner APIs at Finastra, Azure API Management sits in front of the AKS ingress, handling rate limiting, API key management, and request transformation.
>
> **For banking specifically:** We layer these. Azure Front Door at the edge (WAF + DDoS), Application Gateway as regional ingress (WAF + Entra ID auth), AKS NGINX Ingress internally for in-cluster routing. Each layer does what it's best at."

## 5. NAT Gateway purpose

> "NAT Gateway enables outbound internet access for resources in private subnets while keeping them un-reachable from the internet inbound.
>
> **Where I've used it:** At Kelp, our application tier ran in private subnets — EC2 instances and AKS pods that needed to call external APIs (third-party data providers, OS package repositories, container registries) but should never accept inbound internet traffic. NAT Gateway was the bridge.
>
> **Key properties:**
> - **AZ-scoped on AWS** — one NAT Gateway per AZ for HA so a single-AZ failure doesn't kill outbound for all subnets.
> - **Managed by the cloud provider** — no servers to maintain, no patching, no failover to configure.
> - **Outbound only** — cannot initiate inbound connections.
> - **Costs at scale** — hourly charge plus per-GB data processing charges. The data charges add up fast for chatty workloads.
>
> **A pattern I implemented to reduce NAT costs:** VPC Endpoints (PrivateLink) for AWS services like S3 and ECR. Traffic to those services bypasses NAT entirely and stays on AWS's internal network. Massive cost reduction at Kelp's traffic volumes.
>
> **Azure equivalent:** Azure NAT Gateway with similar properties. At Finastra, our private subnets use Azure NAT Gateway for outbound; for private connectivity to Azure services we use Private Endpoints, the Azure equivalent of VPC Endpoints."

## 6. Linux basics — process monitoring and file handling

> "Drawing from 13+ years of Linux experience starting with RHEL admin work at Atos:
>
> **Process monitoring:**
>
> ```bash
> # Snapshot views
> ps aux | grep nginx
> ps -ef
> ps auxf                    # tree view with parent-child relationships
>
> # Interactive
> top                        # classic
> htop                       # modern, more usable
> btop                       # even nicer
>
> # Per-user, per-resource
> ps aux --sort=-%mem | head -20
> ps aux --sort=-%cpu | head -20
>
> # By systemd unit
> systemctl status nginx
> systemctl list-units --type=service --state=running
>
> # What is a process actually doing?
> strace -p <pid>            # syscalls
> lsof -p <pid>              # open files and sockets
> cat /proc/<pid>/status
>
> # Find process by port
> ss -tlnp | grep :443
> lsof -i :443
> fuser 443/tcp
> ```
>
> **At Atos for Standard Chartered I used these for production monitoring** — vmstat, iostat, Topas, netstat for performance analysis on AIX and RHEL servers running banking workloads.
>
> **File handling:**
>
> ```bash
> # Finding files
> find /path -type f -name "*.log"
> find /path -type f -size +100M
> find /var/log -type f -mtime +30        # older than 30 days
>
> # Disk usage
> du -sh /var/log/*
> du -ah /var/log | sort -hr | head -20
> ncdu /var/log               # interactive
> df -h                       # filesystem level
>
> # Permissions
> chmod 755 script.sh
> chmod 600 ~/.ssh/id_rsa
> chmod -R 750 /var/www
> chown -R www-data:www-data /var/www
>
> # ACLs for finer control
> getfacl /var/log/app
> setfacl -m u:appuser:r /var/log/app
>
> # File search and content
> grep -r "error" /var/log
> rg "ERROR" /var/log         # ripgrep — much faster
>
> # File handling under load
> tail -f /var/log/app.log | grep ERROR
> journalctl -u nginx -f      # follow systemd logs
> ```
>
> **At Kelp**, I wrote shell scripts for database backup, PM2 log housekeeping, and application backup automation — leveraging these primitives daily."

## 7. Deployment vs StatefulSet

> "Different primitives for different workload shapes:
>
> | | Deployment | StatefulSet |
> |---|---|---|
> | Pod identity | Interchangeable, random names | Stable, ordered (pod-0, pod-1) |
> | Storage | Shared or ephemeral | Per-pod PersistentVolume via volumeClaimTemplates |
> | Startup order | Parallel | Sequential, ordered |
> | DNS | Single service DNS | Per-pod DNS |
> | Use case | Stateless apps | Stateful — databases, message brokers |
>
> **Deployment use:** All our microservices at Finastra — the API tier, web tier, processing workers. They're stateless; any pod can serve any request. We use Deployments with rolling updates, HPA, and PDBs.
>
> **StatefulSet use cases I've worked with:**
> - **Redis cluster** at IKS Health for caching session data — needed stable identity for cluster topology.
> - **PostgreSQL StatefulSet** for non-production databases at Kelp where we needed per-pod persistent volumes.
> - **Elasticsearch cluster** at Kelp — each node has stable identity, shards assigned per node.
>
> **Rule of thumb I follow:**
> Stateless = Deployment. Stateful with persistent identity or per-replica state = StatefulSet. **But in production, prefer managed services** (Azure SQL, RDS, ElastiCache, MSK) over running stateful workloads on Kubernetes — operational complexity is significant. We use StatefulSets only where managed services aren't suitable."

## 8. ConfigMap vs Secret

> "Both project key-value data into pods; the distinction is sensitivity:
>
> **ConfigMap** — non-sensitive configuration: feature flags, log levels, URLs, application settings.
>
> **Secret** — sensitive data: passwords, tokens, certificates, API keys.
>
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata: { name: app-config }
> data:
>   LOG_LEVEL: 'info'
>   API_ENDPOINT: 'https://api.internal.bank.com'
>   FEATURE_NEW_PAYMENTS: 'true'
> ---
> apiVersion: v1
> kind: Secret
> metadata: { name: app-secret }
> type: Opaque
> stringData:
>   DB_PASSWORD: 'from-key-vault'
>   API_TOKEN: 'rotated-monthly'
> ```
>
> **Critical caveat I always emphasize:** Kubernetes Secrets are **base64-encoded, not encrypted**. They sit in etcd as plain base64. For real protection:
> - **Enable etcd encryption at rest** with KMS provider on the API server.
> - **Use External Secrets Operator** or **CSI Secrets Store Driver** to sync from Azure Key Vault / AWS Secrets Manager.
>
> **In my Finastra work specifically:** Banking compliance doesn't accept native Kubernetes Secrets for sensitive data. We use Azure Key Vault with the CSI Secrets Store driver — secrets live in Key Vault with audit logging, rotation, and access policies, and pods mount them as files at runtime. The native Secret resource is only a sync target, not the source of truth.
>
> **Consumption patterns:**
>
> ```yaml
> # As env vars
> envFrom:
> - configMapRef: { name: app-config }
> - secretRef: { name: app-secret }
>
> # Mounted as files (preferred for secrets)
> volumes:
> - name: secrets
>   secret: { secretName: app-secret }
> volumeMounts:
> - { name: secrets, mountPath: /etc/secrets, readOnly: true }
> ```
>
> **Files over env vars for secrets** — env vars are visible to any process in the container and can leak via crash dumps."

## 9. Network connectivity checks between servers

Drawing from your Linux admin background:

> "Layered approach by OSI level — what I've used daily for 13+ years across AIX, RHEL, Ubuntu, and Windows Server:
>
> **Layer 3 — basic IP reachability:**
> ```bash
> ping -c 4 <host>
> traceroute <host>
> mtr <host>                  # my favorite — combines ping + traceroute, watches over time
> ```
>
> **Layer 4 — specific port:**
> ```bash
> nc -zv <host> 443
> telnet <host> 443           # legacy but universally available
> nmap -p 443,80,8080 <host>  # range
> ```
>
> **DNS:**
> ```bash
> nslookup <host>
> dig <host>
> dig +trace <host>           # full delegation path
> host <host>
> ```
>
> **HTTP/HTTPS — endpoint level:**
> ```bash
> curl -v https://<host>/health
> curl -I --connect-timeout 5 https://<host>
> curl -w "%{time_namelookup} %{time_connect} %{time_starttransfer}\n" -o /dev/null -s https://<host>
> ```
>
> **TLS specifics:**
> ```bash
> openssl s_client -connect <host>:443 -showcerts
> openssl s_client -connect <host>:443 -servername <host>     # SNI test
> ```
>
> **What's listening on the host itself:**
> ```bash
> ss -tlnp                    # modern, faster than netstat
> netstat -tulnp
> lsof -i :443
> ```
>
> **In Azure/cloud context:**
> - **Network Watcher (Azure)** / **VPC Reachability Analyzer (AWS)** — surfaces SG/NSG/route issues in seconds.
> - **NSG flow logs** to detect blocked traffic.
> - **Application Insights** dependency tracking for distributed apps.
>
> **The order I follow:** DNS → ICMP → TCP port → application response. Skip ICMP if firewalls drop it. For complex Azure setups, Network Watcher's connectivity test tool saves enormous debugging time."

## 10. CI/CD pipeline experience

> "Designed and operated CI/CD pipelines across multiple stacks and clients. Walking through the most recent — Finastra:
>
> **Stack:** Azure DevOps Pipelines (on-prem), YAML-based, PowerShell for Windows automation.
>
> **Pipeline structure (multi-stage):**
>
> ```yaml
> trigger:
>   branches: { include: [main, develop] }
>
> variables:
>   imageRepo: finastraacr.azurecr.io/payments-api
>   tag: '$(Build.BuildId)-$(Build.SourceVersion)'
>
> stages:
> - stage: Build
>   jobs:
>   - job: BuildAndTest
>     pool: { vmImage: 'windows-2022' }
>     steps:
>     - task: UseDotNet@2
>       inputs: { version: '8.x' }
>     - script: dotnet restore
>     - script: dotnet build --configuration Release
>     - script: dotnet test --logger trx --collect "XPlat Code Coverage"
>     - task: SonarQubePrepare@5
>     - script: dotnet sonarscanner begin
>     - script: dotnet sonarscanner end
>     - task: SonarQubePublish@5
>
> - stage: Package
>   dependsOn: Build
>   jobs:
>   - job: ContainerBuild
>     steps:
>     - task: Docker@2
>       inputs:
>         containerRegistry: 'finastra-acr'
>         repository: 'payments-api'
>         command: 'buildAndPush'
>         tags: |
>           $(tag)
>           latest
>
> - stage: DeployDev
>   dependsOn: Package
>   jobs:
>   - deployment: DeployToDev
>     environment: 'dev'
>     strategy:
>       runOnce:
>         deploy:
>           steps:
>           - task: HelmDeploy@0
>
> - stage: DeployUAT
>   dependsOn: DeployDev
>   condition: succeeded()
>   jobs:
>   - deployment: DeployToUAT
>     environment: 'uat'             # has approval gate
>
> - stage: DeployProduction
>   dependsOn: DeployUAT
>   jobs:
>   - deployment: DeployToProduction
>     environment: 'production'      # required approvers + change ticket gate
> ```
>
> **Specific elements that matter for banking:**
> - **Approval gates** at UAT and Production — banking requires segregation of duties.
> - **ServiceNow change ticket integration** — pipelines validate an approved CR exists before deploying to prod.
> - **SonarQube quality gate** blocking the pipeline on policy violations.
> - **SAST + DAST + SCA scans** embedded — non-negotiable.
> - **Approved-images-only** policy — Kyverno or similar admission controller blocks unsigned images.
> - **Audit trail** — every deploy traces back to a work item, an approval, and a build artifact.
>
> **Across my career I've also worked with:**
> - **Jenkins** at Kelp — declarative pipelines with shared library for common steps.
> - **GitHub Actions** for OSS-style flows.
> - **AWS CodePipeline** for AWS-native deployments.
>
> **The principle:** build once, promote the artifact. Never rebuild for prod. The same image deploys to dev, UAT, and production. This is what makes audit traceability work in banking."

---

# ✅ Round 2 — Technical Deep Dive

## 1. Cross-account S3 access (Account A → Account B)

> "Two main patterns, depending on whether you need temporary or persistent access:
>
> **Pattern 1: Cross-account IAM role assumption (recommended)**
>
> In Account B (bucket owner), create an IAM role that trusts Account A:
>
> ```json
> // Trust policy on role in Account B
> {
>   "Version": "2012-10-17",
>   "Statement": [{
>     "Effect": "Allow",
>     "Principal": { "AWS": "arn:aws:iam::ACCOUNT_A_ID:root" },
>     "Action": "sts:AssumeRole",
>     "Condition": {
>       "StringEquals": { "sts:ExternalId": "shared-secret-uuid" }
>     }
>   }]
> }
> ```
>
> Permission policy on the role:
> ```json
> {
>   "Effect": "Allow",
>   "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
>   "Resource": [
>     "arn:aws:s3:::shared-bucket",
>     "arn:aws:s3:::shared-bucket/*"
>   ]
> }
> ```
>
> Account A's principal can then `sts:AssumeRole` and get short-lived credentials.
>
> **Pattern 2: Bucket policy + direct principal grant**
>
> Bucket policy on the S3 bucket in Account B:
> ```json
> {
>   "Effect": "Allow",
>   "Principal": { "AWS": "arn:aws:iam::ACCOUNT_A_ID:role/AppRole" },
>   "Action": ["s3:GetObject", "s3:ListBucket"],
>   "Resource": [
>     "arn:aws:s3:::shared-bucket",
>     "arn:aws:s3:::shared-bucket/*"
>   ]
> }
> ```
>
> IAM role in Account A also needs to allow the action — both sides must allow.
>
> **Critical gotchas I'd mention:**
> - **Object ownership** — objects uploaded by Account A are owned by Account A by default. Use `bucket-owner-full-control` ACL or enable **Bucket Owner Enforced** ownership (ACLs disabled — the recommended setting).
> - **KMS encryption** — if the bucket uses SSE-KMS with a CMK in Account B, the KMS key policy must also grant Account A's role access. Otherwise pulls fail with cryptic AccessDenied errors.
> - **Service-Linked patterns** — for EKS pods in Account A accessing S3 in Account B, use IRSA — the pod's service account maps to a role that can assume the cross-account role.
>
> **From my Kelp days:** I set this up for log archival — application logs in Account A's S3, with a separate Account B providing read-only access to a data analytics team. The IRSA + cross-account role pattern was the cleanest."

## 2. Multi-stage Dockerfile for Node.js

> "Walking through a production-grade multi-stage Node.js Dockerfile:
>
> ```dockerfile
> # syntax=docker/dockerfile:1.6
>
> # ─── Build stage ──────────────────────────────────────────────────────
> FROM node:20-alpine AS build
> WORKDIR /build
>
> # Copy package files first to maximize layer caching
> COPY package.json package-lock.json ./
> RUN npm ci --no-audit --no-fund
>
> # Copy source and build
> COPY . .
> RUN npm run build
>
> # ─── Production dependencies stage ────────────────────────────────────
> FROM node:20-alpine AS deps
> WORKDIR /deps
> COPY package.json package-lock.json ./
> RUN npm ci --omit=dev --no-audit --no-fund
>
> # ─── Runtime stage ────────────────────────────────────────────────────
> FROM node:20-alpine AS runtime
>
> # Install tini for proper PID 1 signal handling
> RUN apk add --no-cache tini
>
> WORKDIR /app
> ENV NODE_ENV=production \
>     PORT=3000
>
> # Copy ONLY what's needed for runtime — no source files, no build tools
> COPY --chown=node:node --from=deps  /deps/node_modules ./node_modules
> COPY --chown=node:node --from=build /build/dist        ./dist
> COPY --chown=node:node package.json ./
>
> # Run as non-root (node:alpine ships with the 'node' user)
> USER node
>
> EXPOSE 3000
>
> HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
>   CMD wget -qO- http://localhost:3000/health || exit 1
>
> ENTRYPOINT ["/sbin/tini", "--"]
> CMD ["node", "dist/server.js"]
> ```
>
> **What this Dockerfile gets right:**
> - **Three stages** — build, prod-deps, runtime. Final image has no devDependencies, no source files, no build tools.
> - **Pinned base image** with explicit Node version (not `:latest`).
> - **Layer order optimized** — package files copied before source so dependency installs cache when only code changes.
> - **`npm ci` not `npm install`** — strict, reproducible, fails on lockfile mismatch.
> - **Non-root user** — security baseline.
> - **`tini` as PID 1** — handles SIGTERM properly so graceful shutdown works during pod termination.
> - **HEALTHCHECK** defined for runtime visibility.
>
> **Companion `.dockerignore`:**
> ```
> node_modules
> .git
> .env*
> *.log
> coverage
> .vscode
> .idea
> Dockerfile
> .dockerignore
> ```
>
> Without `.dockerignore`, `node_modules` and `.git` blow up the build context to hundreds of MB unnecessarily.
>
> **Image size achievement:** Without multi-stage, this Node app would be ~1.2GB. With multi-stage + alpine, it drops to ~180MB. Massive savings on pull time and registry storage."

## 3. Handling Terraform state file corruption

> "I've handled this in production at Kelp once. The recovery sequence:
>
> **Step 1: Stop everything**
>
> ```bash
> # Disable the CI pipeline that runs terraform apply
> # Notify the team — no manual applies
> # Take a snapshot of whatever state exists now
> terraform state pull > backup-broken.tfstate
> ```
>
> **Step 2: Recovery from versioned backups (best case)**
>
> This is why versioning on the state bucket is mandatory:
>
> ```bash
> # AWS S3 — list versions
> aws s3api list-object-versions \
>   --bucket company-tfstate \
>   --prefix prod/network.tfstate
>
> # Download last known good
> aws s3api get-object \
>   --bucket company-tfstate \
>   --key prod/network.tfstate \
>   --version-id <good-version-id> \
>   recovered.tfstate
>
> # Validate parseable
> jq . recovered.tfstate > /dev/null
> terraform show recovered.tfstate
>
> # Force-unlock if stuck lock
> terraform force-unlock <lock-id>
>
> # Push recovered state back
> terraform state push recovered.tfstate
>
> # Verify with plan
> terraform plan        # should show no/minimal changes
> ```
>
> **Step 3: If no versioning (worst case)**
>
> Rebuild state by importing every resource:
>
> ```bash
> rm -rf .terraform terraform.tfstate*
> terraform init
>
> # Bulk import via terraformer
> terraformer import aws \
>   --resources=vpc,subnet,route_table,security_group,iam_role,s3,rds \
>   --regions=us-east-1 \
>   --filter="Name=tags.Environment;Value=prod"
>
> # Or modern import blocks (TF 1.5+)
> import {
>   to = aws_vpc.main
>   id = "vpc-0abc123def456"
> }
>
> terraform plan -generate-config-out=generated.tf
> ```
>
> Iterate until `terraform plan` shows no changes.
>
> **Prevention — what I'd implement (and did at Kelp):**
> - **Bucket versioning** + soft-delete on state bucket — non-negotiable.
> - **State locking** via DynamoDB (AWS) or blob lease (Azure).
> - **Encryption** with customer-managed KMS key.
> - **Restricted IAM** — only the pipeline role and a small SRE group can write to state.
> - **Separate state files per environment** — corruption in dev doesn't take down prod.
> - **Nightly state backup** to a separate account/region.
> - **`terraform plan -detailed-exitcode`** scheduled daily, alert on drift.
>
> **The architect-level point:** State corruption is rare *if* the backend is hardened properly. Most 'corruption' incidents I've seen are actually concurrent applies without locking, or hand-editing the state file. The discipline around the backend matters more than the recovery procedure."

## 4. Alternatives to NAT Gateway for private subnet access

> "Several patterns by cost-effectiveness:
>
> **1. VPC Endpoints (Gateway type) — FREE**
> For S3 and DynamoDB only. Adds a route table entry, traffic stays on AWS network. **First thing I'd enable** — eliminates the most common NAT data charge driver.
>
> **2. VPC Endpoints (Interface type, PrivateLink) — hourly + GB charges, still cheaper than NAT**
> For ECR, Secrets Manager, KMS, CloudWatch, SSM. Massive cost reduction for chatty workloads. At Kelp, ECR pulls were a major NAT data cost — switching to Interface endpoints saved significantly.
>
> **3. Pre-baked AMIs (Packer)**
> Instead of pulling packages at runtime, bake the AMI with everything pre-installed. Treat instances as immutable. **My preference for production** — faster boot, reproducible, no runtime package pulls.
>
> **4. Internal package mirror**
> Host an apt/yum/pip/npm mirror on an EC2 in a private subnet that has outbound access (or syncs nightly). Other instances pull from the mirror over internal network.
>
> **5. NAT Instance (legacy)**
> Cheaper than NAT Gateway but self-managed. Single point of failure unless deployed in HA pair. **Not recommended for production** — operational burden outweighs cost savings.
>
> **6. Transit Gateway + shared services VPC**
> Centralized egress VPC with one NAT, accessed via TGW from multiple spoke VPCs. Cost-effective at scale.
>
> **7. Azure equivalents**
> - **Private Endpoints** (PrivateLink) — Azure equivalent of VPC Interface Endpoints.
> - **Service Endpoints** — older Azure pattern, less powerful.
> - **NAT Gateway** still exists in Azure with similar trade-offs.
>
> **My typical recommendation:** Enable Gateway VPC Endpoints for S3/DynamoDB (free, immediate savings), Interface endpoints for ECR + SSM + Secrets Manager + KMS, then NAT Gateway for general internet egress. This combo dramatically reduces NAT data charges while keeping flexibility."

## 5. Debugging an exited container

> "Container exits and the application is gone — but the container object lives until removed, so we can inspect what happened.
>
> **Step 1: Confirm exit and reason**
>
> ```bash
> docker ps -a | grep <container>
> docker inspect <container> --format='{{.State.ExitCode}} {{.State.Error}} {{.State.Status}}'
> docker inspect <container> | jq '.[0].State'
> ```
>
> **Step 2: Logs — the obvious first step**
>
> ```bash
> docker logs <container>
> docker logs --tail 200 <container>
> docker logs --since 10m <container>
> ```
>
> **Step 3: For Kubernetes, the equivalent is critical**
>
> ```bash
> kubectl logs <pod> --previous       # logs from the crashed container
> kubectl describe pod <pod>          # Last State: Reason, Exit Code
> ```
>
> The `--previous` flag is the single most-missed debugging command.
>
> **Step 4: Inspect the filesystem at exit time**
>
> ```bash
> # Container that exited still has its filesystem — commit to a new image
> docker commit <container> dead-container-debug
> docker run -it --entrypoint /bin/sh dead-container-debug
> # Now I can inspect /tmp, config files, logs that weren't on stdout
> ```
>
> **Step 5: Re-run with overridden command for live debugging**
>
> ```bash
> docker run -it --entrypoint /bin/sh <image>    # bypass broken CMD/ENTRYPOINT
> docker run --rm -it <image>                    # interactive run with attached terminal
> ```
>
> **Step 6: Exit code semantics**
>
> | Exit code | Meaning |
> |---|---|
> | 0 | Clean exit |
> | 1 | Generic error (check logs) |
> | 125 | Docker daemon error |
> | 126 | Command not executable |
> | 127 | Command not found |
> | 137 | SIGKILL — usually OOM |
> | 139 | Segfault |
> | 143 | SIGTERM — graceful |
>
> **Step 7: Resource issues**
>
> ```bash
> docker inspect <container> --format='{{.State.OOMKilled}}'
> # On Kubernetes:
> kubectl describe pod <pod> | grep -A5 'Last State'
> # Reason: OOMKilled, Exit Code: 137
> ```
>
> **From production:** At IKS Health we had a healthcare app that kept exiting with code 1 — turned out to be a missing environment variable that the new version expected. `--previous` logs showed the exact missing config. The container model makes this kind of forensic debugging straightforward."

## 6. Importing existing AWS VPC into Terraform

> "Walking through it:
>
> **Step 1: Inventory the existing infrastructure**
>
> ```bash
> aws ec2 describe-vpcs --vpc-ids vpc-0abc123 --query 'Vpcs[0]'
> aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-0abc123
> aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-0abc123
> aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=vpc-0abc123
> aws ec2 describe-security-groups --filters Name=vpc-id,Values=vpc-0abc123
> ```
>
> **Step 2: Write minimal Terraform resource blocks**
>
> ```hcl
> resource 'aws_vpc' 'main' {
>   cidr_block = '10.0.0.0/16'
> }
>
> resource 'aws_subnet' 'public' {
>   for_each          = toset(['10.0.1.0/24', '10.0.2.0/24'])
>   vpc_id            = aws_vpc.main.id
>   cidr_block        = each.key
>   availability_zone = 'us-east-1a'   # placeholder; fix per real data
> }
>
> resource 'aws_internet_gateway' 'main' {
>   vpc_id = aws_vpc.main.id
> }
> ```
>
> **Step 3: Import via the modern declarative approach (TF 1.5+)**
>
> ```hcl
> import {
>   to = aws_vpc.main
>   id = 'vpc-0abc123def456'
> }
>
> import {
>   to = aws_subnet.public['10.0.1.0/24']
>   id = 'subnet-0def456...'
> }
>
> import {
>   to = aws_internet_gateway.main
>   id = 'igw-0ghi789'
> }
> ```
>
> **Step 4: Generate config and plan**
>
> ```bash
> terraform init
> terraform plan -generate-config-out=generated.tf
> # Review generated.tf, merge into your real config
> terraform plan
> ```
>
> **Step 5: Iterate until plan is clean**
>
> Almost always shows differences initially — hand-written config doesn't match every actual attribute. Update until `plan` says 'No changes.'
>
> **Step 6: For bulk imports — terraformer**
>
> For large existing estates:
> ```bash
> terraformer import aws \
>   --resources=vpc,subnet,route_table,igw,nat,sg \
>   --regions=us-east-1 \
>   --filter='Name=tag:Environment;Value=prod'
> ```
>
> **Common gotchas:**
> - **Default tags, default SGs, default route tables** — these often exist and need handling.
> - **Order matters when there are dependencies** — import VPC first, then subnets, then route tables.
> - **After import, plan may want to 'change' attributes** that AWS returns differently than expected (e.g., SG rule ordering). Use `lifecycle.ignore_changes` selectively.
> - **Don't apply destroy** during the import phase — verify plan is zero-change before any apply."

## 7. Blue-green deployment in Kubernetes

> "Two main patterns:
>
> **Pattern 1: Two Deployments + Service selector flip (simple, manual)**
>
> ```yaml
> # Blue deployment (currently live)
> apiVersion: apps/v1
> kind: Deployment
> metadata: { name: myapp-blue }
> spec:
>   replicas: 3
>   selector: { matchLabels: { app: myapp, version: blue } }
>   template:
>     metadata: { labels: { app: myapp, version: blue } }
>     spec:
>       containers:
>       - { name: app, image: 'myapp:1.4.2' }
> ---
> # Green deployment (new version)
> apiVersion: apps/v1
> kind: Deployment
> metadata: { name: myapp-green }
> spec:
>   replicas: 3
>   selector: { matchLabels: { app: myapp, version: green } }
>   template:
>     metadata: { labels: { app: myapp, version: green } }
>     spec:
>       containers:
>       - { name: app, image: 'myapp:1.5.0' }
> ---
> # Service controls which version receives traffic
> apiVersion: v1
> kind: Service
> metadata: { name: myapp }
> spec:
>   selector: { app: myapp, version: blue }   # flip to 'green' to cut over
>   ports: [{ port: 80, targetPort: 8080 }]
> ```
>
> **Cutover:**
> ```bash
> # Verify green via port-forward first
> kubectl port-forward deploy/myapp-green 8080:8080 &
> curl localhost:8080/health
>
> # Flip the service
> kubectl patch service myapp -p '{\"spec\":{\"selector\":{\"version\":\"green\"}}}'
>
> # Rollback by flipping back
> kubectl patch service myapp -p '{\"spec\":{\"selector\":{\"version\":\"blue\"}}}'
>
> # After confidence period, scale down blue
> kubectl scale deployment myapp-blue --replicas=0
> ```
>
> **Pattern 2: Argo Rollouts (production-grade)**
>
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Rollout
> metadata: { name: myapp }
> spec:
>   replicas: 3
>   strategy:
>     blueGreen:
>       activeService: myapp-active       # serves production traffic
>       previewService: myapp-preview     # for pre-cutover testing
>       autoPromotionEnabled: false       # require manual promote
>       scaleDownDelaySeconds: 300        # keep old version 5min for fast rollback
>       prePromotionAnalysis:
>         templates: [{ templateName: smoke-tests }]
>       postPromotionAnalysis:
>         templates: [{ templateName: success-rate }]
> ```
>
> **What Argo Rollouts adds:**
> - Pre-cutover smoke tests against preview service.
> - Automated promote/abort based on metrics.
> - Old version retained for fast rollback.
> - Full audit trail in ArgoCD UI.
>
> **When I'd use blue-green vs canary:**
> - **Blue-green**: major version upgrade, coordinated DB+app cutover, need instant rollback.
> - **Canary**: most ongoing deployments, risk-sensitive incremental changes.
>
> **Banking context:** At Finastra, we use blue-green for major payment system upgrades where regulatory traceability requires the new version to be fully validated against production-like traffic *before* any user exposure. Canary works for ongoing changes; blue-green works for tier-1 cuts with strict rollback requirements."

## 8. Managing secrets in Terraform without hardcoding

> "Layered approach, from worst to best:
>
> **1. Don't do this:**
> ```hcl
> resource 'aws_db_instance' 'this' {
>   password = 'Pa$$w0rd'    # NO — in code, Git history, state file
> }
> ```
>
> **2. Reference from a secrets manager at runtime:**
> ```hcl
> data 'aws_secretsmanager_secret_version' 'db' {
>   secret_id = 'prod/rds/master-password'
> }
>
> resource 'aws_db_instance' 'this' {
>   password = data.aws_secretsmanager_secret_version.db.secret_string
>   lifecycle {
>     ignore_changes = [password]   # rotation outside TF doesn't drift
>   }
> }
> ```
>
> **3. Generate and store:**
> ```hcl
> resource 'random_password' 'db' {
>   length  = 32
>   special = true
> }
>
> resource 'aws_secretsmanager_secret' 'db' {
>   name = 'prod/rds/master-password'
> }
>
> resource 'aws_secretsmanager_secret_version' 'db' {
>   secret_id     = aws_secretsmanager_secret.db.id
>   secret_string = random_password.db.result
> }
>
> resource 'aws_db_instance' 'this' {
>   password = random_password.db.result
> }
> ```
>
> Password lives in state but never in code or Git.
>
> **4. Eliminate the secret entirely:**
> - **IAM database authentication** for RDS Postgres/MySQL — app gets short-lived IAM tokens, no password.
> - **IRSA** for EKS workloads — pod has IAM identity directly, no static creds.
> - **Workload Identity** on Azure for AKS — same pattern.
>
> **5. Mark variables and outputs sensitive:**
> ```hcl
> variable 'db_password' {
>   sensitive = true
>   type      = string
> }
>
> output 'db_endpoint' {
>   value     = aws_db_instance.this.endpoint
>   sensitive = false
> }
>
> output 'db_password' {
>   value     = aws_db_instance.this.password
>   sensitive = true    # masked in plan/apply output
> }
> ```
>
> **Securing the state file itself** (since secrets still end up there):
> - **S3 backend with KMS encryption** using a customer-managed key.
> - **Bucket policy restricting access** to only the pipeline role.
> - **Versioning + MFA delete**.
> - **Restricted state access** — engineers don't need read access to prod state.
>
> **At Finastra specifically:** We use Azure Key Vault for all secrets. Terraform reads via `azurerm_key_vault_secret` data sources. The pipeline service principal has Key Vault Reader role only on specific secrets, not blanket access. Banking compliance audits this — who accessed what secret, when, why.
>
> **Hierarchy I'd recommend:**
> 1. IAM auth, no secret needed (best).
> 2. Generate in Terraform, store in secrets manager.
> 3. Reference an externally-managed secret.
> 4. Sensitive variable via env from secrets manager.
> 5. Encrypted state with restricted access (defense in depth)."

## 9. COPY vs ADD in Dockerfile

> "Both copy files from the build context into the image, but with different capabilities:
>
> **COPY** — simple, predictable. Just copies files/directories from build context to image.
>
> ```dockerfile
> COPY app.jar /app/app.jar
> COPY src/ /build/src/
> COPY --from=build /build/dist /app/dist     # multi-stage
> ```
>
> **ADD** — does everything COPY does, *plus*:
> - Can fetch a remote URL (`ADD https://example.com/file.tar.gz /tmp/`).
> - Auto-extracts local tar archives (`ADD app.tar.gz /app` extracts into /app).
>
> ```dockerfile
> ADD https://example.com/release.tar.gz /tmp/
> ADD app.tar.gz /app           # auto-extracts
> ```
>
> **Rule of thumb: always use COPY unless you specifically need ADD's tarball auto-extraction.**
>
> **Why this matters:**
> - **ADD's behavior is non-obvious** — new engineers reading your Dockerfile shouldn't have to remember that ADD might silently extract a tar.
> - **URL fetching via ADD is discouraged** — you can't verify checksums in one step. Better to `RUN wget && verify && extract` so the steps are explicit and cacheable.
> - **COPY is boring and predictable** — exactly what you want in infrastructure code.
>
> **In my Dockerfiles I almost always use COPY.** ADD only when:
> - Bootstrapping with a pre-tar'd application bundle.
> - Multi-stage build where copy-from is the better path anyway.
>
> **Production Dockerfile snippet:**
> ```dockerfile
> # GOOD — explicit and reviewable
> RUN wget https://example.com/release.tar.gz \
>     && echo 'expected-sha256-sum *release.tar.gz' | sha256sum -c \
>     && tar -xzf release.tar.gz \
>     && rm release.tar.gz
>
> # BAD — opaque, no integrity check
> ADD https://example.com/release.tar.gz /tmp/
> ```
>
> The explicit approach gives you checksum verification, error handling, and obvious behavior."

## 10. Cross-account resource provisioning using Terraform

> "Pattern: **provider aliases with assume-role**.
>
> ```hcl
> # Main provider — orchestration account
> provider 'aws' {
>   alias  = 'shared'
>   region = 'us-east-1'
> }
>
> # Account A
> provider 'aws' {
>   alias  = 'account_a'
>   region = 'us-east-1'
>   assume_role {
>     role_arn     = 'arn:aws:iam::ACCOUNT_A_ID:role/TerraformExecutionRole'
>     session_name = 'terraform-${var.environment}'
>     external_id  = var.external_id
>   }
> }
>
> # Account B
> provider 'aws' {
>   alias  = 'account_b'
>   region = 'us-east-1'
>   assume_role {
>     role_arn = 'arn:aws:iam::ACCOUNT_B_ID:role/TerraformExecutionRole'
>   }
> }
>
> # Resources in Account A
> resource 'aws_s3_bucket' 'logs' {
>   provider = aws.account_a
>   bucket   = 'company-logs-bucket'
> }
>
> # Resources in Account B
> resource 'aws_iam_role' 'consumer' {
>   provider = aws.account_b
>   name     = 'log-consumer'
> }
>
> # Modules taking explicit providers
> module 'shared_services' {
>   source = './modules/shared'
>   providers = {
>     aws.primary = aws.account_a
>     aws.dr      = aws.account_b
>   }
> }
> ```
>
> **Pre-requisites:**
> - IAM role in each target account with a trust policy allowing assumption from the central identity.
> - Each role has custom permissions (not blanket `AdministratorAccess`).
> - `ExternalId` for additional security against confused-deputy attacks.
>
> **Enterprise patterns I'd mention:**
> - **AWS Organizations / Control Tower** — centralized account management; new accounts come with a baseline Terraform execution role.
> - **Account Factory for Terraform (AFT)** — Control Tower's IaC for account vending.
> - **Separate state files per account/environment** — never mix multi-account state in one file.
> - **Pipeline service principal federated via OIDC** — short-lived credentials, no static keys.
>
> **State file storage:** State stored in a dedicated **shared services / management account**, never in the same account whose resources it's managing. That account's S3 + DynamoDB is the source of truth.
>
> **In a banking context like Finastra:** This pattern is critical for landing-zone architecture — separate AWS accounts per workload tier (dev, staging, prod), separate accounts for shared services (logging, security, networking), and Terraform crossing account boundaries via assume-role with strict IAM policies."

## 11. Secrets in Docker — PHP + MySQL use case

> "Specific to PHP + MySQL — multiple layered approaches:
>
> **Bad (worth naming to explain why):**
> ```dockerfile
> ENV DB_PASSWORD=secret123    # NEVER — secret in image layers
> ```
>
> **Better — runtime env injection:**
> ```bash
> docker run -e DB_PASSWORD=\"$DB_PASSWORD\" myphp
> docker run --env-file=./.env.prod myphp     # .env.prod gitignored
> ```
> Still leaks via `docker inspect`, env vars visible to any in-container process.
>
> **Production patterns:**
>
> **1. Docker secrets (Swarm mode only):**
> ```bash
> echo 'secretpass' | docker secret create db_password -
> docker service create --secret db_password myphp
> # App reads from /run/secrets/db_password
> ```
>
> **2. Bind-mount secrets file** (read-only):
> ```bash
> docker run -v /etc/myapp/secrets:/run/secrets:ro myphp
> ```
>
> **3. External secrets manager (the production answer):**
> ```php
> <?php
> require 'vendor/autoload.php';
> use Aws\\SecretsManager\\SecretsManagerClient;
>
> $client = new SecretsManagerClient([
>     'version' => 'latest',
>     'region'  => 'us-east-1',
> ]);
>
> $result = $client->getSecretValue(['SecretId' => 'prod/mysql/credentials']);
> $secret = json_decode($result['SecretString'], true);
>
> $pdo = new PDO(
>     \"mysql:host={$secret['host']};dbname={$secret['dbname']}\",
>     $secret['username'],
>     $secret['password']
> );
> ?>
> ```
>
> App uses IAM role (IRSA on EKS or instance profile on EC2) to fetch at startup. Secret never in env, never in image, never in Git.
>
> **4. Better: skip the password entirely**
> **IAM database authentication for RDS MySQL** — app gets short-lived tokens via IAM; no password to manage or rotate.
>
> ```php
> // Generate auth token via IAM
> $token = $client->createAuthToken([...]);
> $pdo = new PDO(\"mysql:host=...\", $username, $token);
> ```
>
> **5. BuildKit secrets** (for build-time secrets like Composer credentials):
> ```dockerfile
> # syntax=docker/dockerfile:1.4
> RUN --mount=type=secret,id=composer_auth \\
>     COMPOSER_AUTH=$(cat /run/secrets/composer_auth) composer install
> ```
> ```bash
> DOCKER_BUILDKIT=1 docker build --secret id=composer_auth,src=./auth.json .
> ```
>
> **My preference for production PHP+MySQL on Kubernetes:**
> 1. **IAM database auth** if AWS RDS — best, no secret needed.
> 2. **External Secrets Operator + AWS Secrets Manager / Azure Key Vault** — for cases needing password.
> 3. **Init container fetching secrets** into shared emptyDir volume; app reads from file.
> 4. **Never** env vars in Kubernetes manifests for sensitive data."

## 12. Terraform drift (manual changes vs IaC)

> "Walking through diagnosis and resolution:
>
> **Detection (continuous, not reactive):**
>
> 1. **Scheduled drift check in CI:**
> ```bash
> terraform plan -detailed-exitcode -refresh-only
> # Exit 0: no drift
> # Exit 2: drift detected → alert
> ```
> Runs nightly across all environments. Alerts to Slack.
>
> 2. **driftctl** for continuous detection including unmanaged resources.
>
> 3. **Terraform Cloud / Enterprise** built-in drift detection.
>
> 4. **Cloud-native config rules** (AWS Config, Azure Policy) detecting changes and routing events.
>
> **Once detected — decide intent:**
>
> **A. Manual change was wrong — revert:**
> ```bash
> terraform apply
> # Pushes Terraform's desired state back, overwriting the manual change
> ```
>
> **B. Manual change was correct — adopt:**
> ```hcl
> # Update Terraform code to match the reality
> resource 'aws_security_group' 'app' {
>   ingress {
>     description = 'SSH from internal network — added during incident #1234'
>     from_port   = 22
>     to_port     = 22
>     protocol    = 'tcp'
>     cidr_blocks = ['10.0.0.0/16']
>   }
> }
> ```
> Submit as PR, get review, merge. `terraform plan` now shows no diff.
>
> **C. Change managed externally — ignore:**
> ```hcl
> resource 'aws_autoscaling_group' 'app' {
>   lifecycle {
>     ignore_changes = [desired_capacity]   # AWS autoscaling manages this
>   }
> }
> ```
> Use sparingly — overuse defeats the purpose of IaC.
>
> **The cultural fix (the real answer):**
>
> Detection-and-repair treats the symptom. The actual fix is making manual changes hard:
> - **IAM lockdown** — engineers can't modify Terraform-managed resources directly outside the pipeline role.
> - **SCPs/Azure Policy** preventing break-glass scenarios except via formal change procedure.
> - **Resource tagging** (`ManagedBy: terraform`) with Config rules detecting changes to tagged resources.
> - **CloudTrail/Activity Log alerts** on resource changes outside the pipeline IAM role.
> - **Postmortem when drift happens repeatedly** — why is someone bypassing the pipeline?
>
> **In my Finastra banking context:** Drift on a production resource is a postmortem-worthy event. Banking compliance requires every change to be auditable through a change ticket. Console-based changes are essentially forbidden — we have IAM/AAD policies that prevent them, and Azure Policy detects exceptions for security review."

## 13. Kubernetes Network Policies

> "NetworkPolicies are namespaced, label-selector-based firewall rules between pods.
>
> **Critical: requires a CNI that enforces them.** Calico, Cilium, Weave, Antrea. AWS VPC CNI on EKS requires Calico or Cilium add-on. Azure CNI on AKS supports it natively in Azure CNI Powered by Cilium.
>
> **Step 1: Default-deny in every namespace**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: default-deny-all
>   namespace: app
> spec:
>   podSelector: {}
>   policyTypes: [Ingress, Egress]
>   # No rules = deny all
> ```
>
> **Step 2: Explicit allows**
>
> Frontend reachable only by ingress controller:
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: frontend-from-ingress
>   namespace: app
> spec:
>   podSelector: { matchLabels: { app: frontend } }
>   policyTypes: [Ingress]
>   ingress:
>   - from:
>     - namespaceSelector: { matchLabels: { name: ingress-nginx } }
>     ports: [{ protocol: TCP, port: 8080 }]
> ```
>
> Backend reachable only by frontend:
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: backend-from-frontend
>   namespace: app
> spec:
>   podSelector: { matchLabels: { app: backend } }
>   policyTypes: [Ingress]
>   ingress:
>   - from:
>     - podSelector: { matchLabels: { app: frontend } }
>     ports: [{ protocol: TCP, port: 8080 }]
> ```
>
> **DNS egress (every namespace needs this — common debugging trap):**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: allow-dns-egress
>   namespace: app
> spec:
>   podSelector: {}
>   policyTypes: [Egress]
>   egress:
>   - to:
>     - namespaceSelector: { matchLabels: { name: kube-system } }
>       podSelector: { matchLabels: { k8s-app: kube-dns } }
>     ports:
>     - { protocol: UDP, port: 53 }
>     - { protocol: TCP, port: 53 }
> ```
>
> **Backend egress to RDS:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: backend-to-rds
>   namespace: app
> spec:
>   podSelector: { matchLabels: { app: backend } }
>   policyTypes: [Egress]
>   egress:
>   - to:
>     - ipBlock: { cidr: 10.0.10.0/24 }    # RDS subnet
>     ports: [{ protocol: TCP, port: 5432 }]
> ```
>
> **Production patterns I follow:**
> - **Default-deny in every namespace.**
> - **Explicit allow per traffic direction** — separate ingress and egress policies.
> - **`kube-system` and ingress-nginx namespaces labeled** so other namespaces can reference them.
> - **DNS allow-egress in every namespace** — the most common debugging trap.
> - **Use Cilium for L7 policies** (HTTP method/path-aware, not just port) for stronger zero-trust.
> - **Test policies in dev** via `kubectl exec` from one pod attempting connection to another.
>
> **For banking at Finastra:** NetworkPolicies are non-negotiable for PCI-DSS compliance. Default-deny is enforced by Kyverno admission policy — any namespace without one is rejected. Audit logs surface policy violations to the security team."

## 14. Python script — backup files older than 30 days

> "Production-grade script with logging, error handling, and dry-run:
>
> ```python
> #!/usr/bin/env python3
> '''Backup files older than N days from a source directory.'''
> import argparse
> import logging
> import shutil
> import sys
> import tarfile
> from datetime import datetime, timedelta
> from pathlib import Path
>
>
> def setup_logging(verbose: bool) -> None:
>     logging.basicConfig(
>         level=logging.DEBUG if verbose else logging.INFO,
>         format='%(asctime)s [%(levelname)s] %(message)s',
>     )
>
>
> def find_old_files(source: Path, age_days: int) -> list[Path]:
>     '''Return files in source older than age_days.'''
>     cutoff = datetime.now() - timedelta(days=age_days)
>     cutoff_ts = cutoff.timestamp()
>     old_files: list[Path] = []
>
>     for entry in source.rglob('*'):
>         if not entry.is_file():
>             continue
>         try:
>             if entry.stat().st_mtime < cutoff_ts:
>                 old_files.append(entry)
>         except OSError as e:
>             logging.warning('Skipping %s: %s', entry, e)
>
>     return old_files
>
>
> def backup_to_archive(files: list[Path], source: Path, dest: Path,
>                       dry_run: bool, delete_after: bool) -> Path | None:
>     if not files:
>         logging.info('No files older than threshold; nothing to back up.')
>         return None
>
>     dest.mkdir(parents=True, exist_ok=True)
>     timestamp = datetime.now().strftime('%Y%m%d-%H%M%S')
>     archive_path = dest / f'backup-{timestamp}.tar.gz'
>
>     logging.info('Found %d files to back up.', len(files))
>     if dry_run:
>         for f in files:
>             logging.info('[dry-run] would archive: %s', f)
>         return None
>
>     with tarfile.open(archive_path, 'w:gz') as tar:
>         for f in files:
>             arcname = f.relative_to(source)
>             tar.add(f, arcname=str(arcname))
>             logging.debug('Added: %s', arcname)
>
>     logging.info('Archive created: %s (%.2f MB)',
>                  archive_path,
>                  archive_path.stat().st_size / (1024 * 1024))
>
>     if delete_after:
>         for f in files:
>             try:
>                 f.unlink()
>                 logging.debug('Deleted: %s', f)
>             except OSError as e:
>                 logging.warning('Could not delete %s: %s', f, e)
>         logging.info('Deleted %d source files after archiving.', len(files))
>
>     return archive_path
>
>
> def main() -> int:
>     p = argparse.ArgumentParser(description='Backup files older than N days.')
>     p.add_argument('source', type=Path, help='Source directory')
>     p.add_argument('dest', type=Path, help='Destination for backup archive')
>     p.add_argument('--days', type=int, default=30)
>     p.add_argument('--delete', action='store_true', help='Delete source after archive')
>     p.add_argument('--dry-run', action='store_true')
>     p.add_argument('-v', '--verbose', action='store_true')
>     args = p.parse_args()
>
>     setup_logging(args.verbose)
>
>     if not args.source.is_dir():
>         logging.error('Source is not a directory: %s', args.source)
>         return 1
>
>     files = find_old_files(args.source, args.days)
>     backup_to_archive(files, args.source, args.dest,
>                       dry_run=args.dry_run,
>                       delete_after=args.delete)
>     return 0
>
>
> if __name__ == '__main__':
>     sys.exit(main())
> ```
>
> **Usage:**
> ```bash
> python backup.py /var/log /backups --days 30 --dry-run -v
> python backup.py /var/log /backups --days 30                  # real run
> python backup.py /var/log /backups --days 30 --delete         # with cleanup
> ```
>
> **What this gets right:**
> - **Dry-run mode** before destructive operations — non-negotiable for backup scripts.
> - **Logging** with verbosity control.
> - **Explicit cutoff calculation** — readable and testable.
> - **Tar.gz with timestamp** — versioned archives.
> - **Optional deletion** — separate flag, default safe.
> - **Error handling** — skips unreadable files with warnings.
> - **Type hints** for tooling support.
> - **Pathlib** — modern, cross-platform.
>
> **For production:** I'd add S3 upload (boto3), checksum verification, retention policy on destination. Or — even better — use **AWS Backup** / **Azure Backup** / **S3 Lifecycle** rules that handle this declaratively without custom scripts.
>
> **At Kelp** I had similar shell scripts for PostgreSQL backup, PM2 log rotation, and application backup. The pattern is consistent: dry-run, log everything, fail-loudly, and retention-policy externalized."

## 15. Cloud cost optimization strategies

> "Comprehensive approach — I've done this several times now, most recently at Kelp:
>
> **Phase 1: Visibility (you can't optimize what you can't see)**
> - **Tagging strategy** enforced: CostCenter, Owner, Environment, Application.
> - **Cost Explorer + CUR** to QuickSight for team-level dashboards.
> - **Azure Cost Management** for Azure workloads with anomaly detection.
> - **Budgets with alerts** at 50/80/100% of forecast.
> - **Kubecost or OpenCost** for Kubernetes workload-level cost attribution.
>
> **Phase 2: Quick wins (typically 20-30% reduction)**
> - **Right-sized EC2/Azure VMs** based on metrics — Cost Explorer's right-sizing recommendations are reliable.
> - **Deleted orphans** — unattached EBS, unattached EIPs, idle ALBs, old AMIs.
> - **Stopped non-prod environments** nightly and weekends via EventBridge + Lambda. Typically 65% savings on dev.
> - **S3 lifecycle policies** — Standard → IA → Glacier → Deep Archive. Intelligent-Tiering for unpredictable patterns.
> - **EBS gp2 → gp3** — 20% cheaper, equal/better performance.
> - **VPC Endpoints for S3/DynamoDB/ECR** — eliminated significant NAT data charges at Kelp.
> - **CloudWatch Logs retention** — most teams default to never expire. Set per log group based on actual need.
>
> **Phase 3: Commitments (10-40% additional)**
> - **Compute Savings Plans** for baseline workload (covers EC2 + Fargate + Lambda). 1-year, no upfront — covered ~75% of steady-state.
> - **Reserved Instances** for RDS, ElastiCache (more predictable than EC2).
> - **Spot instances** for fault-tolerant workloads — CI runners, EKS data plane via Karpenter, batch jobs. 60-90% discount.
> - **Azure Reserved Instances** for predictable AKS node pools.
>
> **Phase 4: Architecture changes**
> - **Right-sized Kubernetes** via VPA recommendations.
> - **Consolidated low-utilization microservices.**
> - **Lambda-heavy batch workflow to Fargate Spot** — cheaper at steady throughput.
> - **Aurora Serverless v2** for bursty access patterns.
>
> **Phase 5: Specific wins I had at Kelp:**
> - **NAT Gateway data charges** were ~$8k/month. Moved AWS API traffic through Interface endpoints; cost dropped to ~$2k/month. ROI in 3 weeks.
> - **EKS overprovisioning** — VPA showed average pod CPU usage was 35% of requests. Tightened requests; node count dropped 25%.
> - **CloudWatch Logs ingest** for one chatty service was $12k/month at debug level. Moved debug logs to Loki (S3-backed), kept INFO-level in CloudWatch. Saved $9k/month.
> - **Reserved Instances for RDS** — committed 70% of production fleet. Saved 40% on those instances.
>
> **Phase 6: Cultural**
> - **Monthly FinOps reviews** per team with cost dashboards.
> - **Cost-aware CI/CD** — infracost in PRs showing cost impact of Terraform changes.
> - **Service Control Policies** preventing expensive resource types in dev.
> - **Unit economics tracking** — cost per transaction, cost per user.
>
> **What I explicitly don't do for cost cutting:**
> - Reduce production HA/redundancy.
> - Disable monitoring/logging.
> - Last-minute Spot for tier-1 stateful workloads.
>
> **The framing:** Cost optimization is a continuous discipline owned by engineering, not a quarterly fire drill. The companies that do it well treat cost as a first-class non-functional requirement, alongside latency and availability."

## 16. Geo-location based routing using AWS

> "Route 53 is the primary service. Three relevant routing policies:
>
> **1. Geolocation routing** — based on user's geographic location (continent, country, US state).
> 'EU users → EU endpoint, US users → US endpoint, all others → default.'
>
> **2. Geoproximity routing** — based on geographic distance with adjustable bias.
> Requires Traffic Flow.
>
> **3. Latency-based routing** — to the AWS region with lowest network latency for that user.
> Different from geo — fastest, not closest.
>
> **Geolocation setup:**
>
> ```hcl
> resource 'aws_route53_record' 'eu' {
>   zone_id        = aws_route53_zone.main.zone_id
>   name           = 'app.example.com'
>   type           = 'A'
>   set_identifier = 'eu'
>   geolocation_routing_policy { continent = 'EU' }
>   alias {
>     name                   = aws_lb.eu.dns_name
>     zone_id                = aws_lb.eu.zone_id
>     evaluate_target_health = true
>   }
> }
>
> resource 'aws_route53_record' 'us' {
>   zone_id        = aws_route53_zone.main.zone_id
>   name           = 'app.example.com'
>   type           = 'A'
>   set_identifier = 'us'
>   geolocation_routing_policy { continent = 'NA' }
>   alias { ... aws_lb.us.dns_name ... }
> }
>
> resource 'aws_route53_record' 'default' {
>   zone_id        = aws_route53_zone.main.zone_id
>   name           = 'app.example.com'
>   type           = 'A'
>   set_identifier = 'default'
>   geolocation_routing_policy { country = '*' }
>   alias { ... }
> }
> ```
>
> **Always include the default record** — without it, users in unmapped regions get nothing.
>
> **Pair with health checks** so a failed regional endpoint routes elsewhere even if it matches the geo rule.
>
> **For richer scenarios:**
> - **CloudFront with origin failover** for edge-based routing with caching and WAF.
> - **AWS Global Accelerator** for anycast routing with intelligent failover.
>
> **The decision depends on need:**
> - **Data residency** → geolocation.
> - **Performance** → latency-based.
> - **DDoS-resistant anycast** → Global Accelerator.
>
> **In a banking context (Finastra):** We use geolocation for regulatory data residency — EU customer data routes to EU regions for GDPR. Combined with latency-based fallback within each geographic boundary."

## 17. Troubleshooting Kubernetes production issues

> "Walking through the common patterns:
>
> **ImagePullBackOff:**
>
> ```bash
> kubectl describe pod <pod>
> # Events show specific pull error:
> #   'manifest for ...:1.4.2 not found'           → wrong tag
> #   'unauthorized'                                → missing imagePullSecrets
> #   'no such host'                                → DNS/network issue
> #   'rate limit exceeded'                         → DockerHub rate limit
> #   'no space left on device'                    → node disk full
> ```
>
> **Causes ranked by likelihood:**
> 1. Wrong image name or tag.
> 2. Missing imagePullSecrets for private registry.
> 3. Authentication expired.
> 4. Network can't reach registry (pod's network can't egress; missing VPC endpoint for ECR).
> 5. Rate limit exceeded.
> 6. Wrong architecture (ARM image on x86).
> 7. KMS issue (encrypted with CMK node can't decrypt).
>
> **Pod Evicted:**
>
> ```bash
> kubectl describe pod <evicted-pod>
> # Status: shows pressure type:
> #   'The node was low on resource: memory'
> #   'The node was low on resource: ephemeral-storage'
> #   'The node had condition: [DiskPressure]'
> ```
>
> Causes:
> 1. **Memory pressure** — node OOMing; pods with lowest QoS evicted first.
> 2. **Disk pressure** — node disk filling up.
> 3. **PriorityClass** — lower-priority pods evicted to make room for higher.
> 4. **PodDisruptionBudget** blocking voluntary evictions.
>
> **Fixes:**
> - Set proper resource requests/limits.
> - Larger nodes or more nodes.
> - Cleanup orphan images: kubelet image GC tuning.
> - PriorityClass for critical workloads.
>
> **CrashLoopBackOff:**
>
> ```bash
> kubectl logs <pod>                          # current
> kubectl logs <pod> --previous               # crashed container — critical
> kubectl describe pod <pod>                  # Last State, Reason, Exit Code
> ```
>
> Causes ranked by likelihood:
> 1. **Application error on startup** — bad config, missing env var, can't connect to DB.
> 2. **Misconfigured probes** — liveness failing because app needs more startup time. Fix: add startupProbe.
> 3. **OOM kill** — exit code 137. Memory limit too low.
> 4. **Missing config or secrets** — referenced ConfigMap/Secret doesn't exist.
> 5. **Volume mount issues** — PVC unbound, file permissions wrong (especially with non-root users).
> 6. **Wrong architecture** — ARM image on x86, missing dynamic libraries.
> 7. **PID 1 problem** — wrapper script exits because main process detaches.
>
> **Pods Pending forever:**
>
> ```bash
> kubectl describe pod <pod>
> # Events show scheduling failures:
> #   '0/8 nodes are available: 8 Insufficient cpu'   → cluster needs more capacity
> #   'pod has unbound immediate PersistentVolumeClaims' → PVC issue
> #   'no nodes match node affinity/selector'           → wrong nodeSelector
> ```
>
> Fixes:
> - **Insufficient resources:** scale up node pool, smaller requests, Karpenter for elastic scaling.
> - **PVC unbound:** check storage class, CSI driver, topology constraints.
> - **Wrong affinity:** verify labels match, adjust selectors.
>
> **Service unreachable / 502s despite healthy pods:**
>
> ```bash
> # Check endpoints — Service should have IPs
> kubectl get endpoints <service>
> # If empty, label selector doesn't match any pod
>
> # Check service mesh / sidecar issues
> kubectl get pod <pod> -o yaml | grep -A5 envoy
>
> # SNAT exhaustion (EKS)
> # NodePool's NAT Gateway port exhaustion
> ```
>
> **My systematic flow when triaging:**
> 1. **What does kubectl describe pod say?** Events at the bottom usually answer.
> 2. **What do logs from --previous say?** Most-missed debug command.
> 3. **Are dependencies healthy?** DB, downstream services, secrets in place.
> 4. **Are resource limits hit?** Top pod, top nodes.
> 5. **Is the change recent?** Rollout history, recent deploys.
> 6. **Are policies blocking?** Admission policies, NetworkPolicies, RBAC.
>
> **The principle:** for K8s troubleshooting, `kubectl describe` and `kubectl logs --previous` answer 80% of questions in 30 seconds. The systematic mistake is jumping to changing config before reading what kubelet is actually saying."

---

That covers both rounds. Based on your CV, likely follow-ups to be ready for:

- **"Tell me about the Perforce-to-Azure-Repos migration."** — Strong story of ownership; have specific challenges (history preservation, developer retraining, branching strategy translation) ready.
- **"Walk me through a banking compliance incident you handled."** — Banking context is your differentiator; have an SAST/DAST/SCA finding or production security issue story prepared.
- **"How do you balance security depth with delivery velocity?"** — DevSecOps experience; emphasize shifting left.
- **"You've worked on both Azure and AWS — how do you decide between them for a new project?"** — multi-cloud framing; honest about strengths.
- **"What's your biggest architectural improvement at Finastra?"** — Should have measurable outcome ready.

Your real strength is depth in **banking + Azure + multi-language CI/CD**. Lean into that rather than competing on AWS-specific depth where your CV is lighter.