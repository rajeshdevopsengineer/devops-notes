# Senior DevOps Interview — Complete Walkthrough

Going through each with the depth a senior engineer brings.

---

## 1. 3-Tier Architecture — Design and Workflow

> "Walking through the design step by step, then the request workflow.

**The three tiers:**

```
┌─────────────────────────────────────────────────────────────┐
│  TIER 1 — PRESENTATION (web/UI)                              │
│  CloudFront + S3 + ALB                                       │
│  WAF, TLS termination, edge caching                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 2 — APPLICATION (business logic)                       │
│  EKS pods / EC2 ASG in private subnets across 3 AZs          │
│  Stateless microservices, HPA + Cluster Autoscaler           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 3 — DATA (persistent state)                            │
│  RDS Aurora Multi-AZ, ElastiCache Redis, S3                  │
│  Private data subnets, no internet route                     │
└─────────────────────────────────────────────────────────────┘
```

**Networking foundation underneath all three:**

```
VPC: 10.0.0.0/16, 3 AZs
├── Public subnets (10.0.0.0/24, .1.0/24, .2.0/24)
│     ALB, NAT Gateways, bastion
├── Private app subnets (10.0.16.0/20, .32.0/20, .48.0/20)
│     EKS nodes, application pods
└── Private data subnets (10.0.64.0/22, .68.0/22, .72.0/22)
      RDS, ElastiCache, no internet route
```

**Step-by-step request workflow:**

**Step 1: DNS resolution**
- User types `app.example.com` in browser.
- Browser checks local DNS cache, then OS resolver, then ISP DNS.
- Eventually reaches Route 53 (the authoritative DNS).
- Route 53 returns an alias to the CloudFront distribution or directly to ALB.

**Step 2: Edge tier (CloudFront)**
- Request hits the nearest CloudFront POP via anycast.
- TLS handshake at the edge (faster than at origin).
- WAF rules evaluated — bad requests rejected here.
- If cached, response served immediately. Cache miss → forward to origin (ALB).

**Step 3: Presentation tier (ALB)**
- Request reaches ALB in public subnet.
- ALB applies listener rules: path-based or host-based routing.
- ALB checks target health; routes only to healthy targets.
- For static content: returns S3-served React/Angular bundle.
- For API/dynamic content: forwards to application tier.

**Step 4: Application tier (EKS pods)**
- ALB forwards to a pod in private subnet (via AWS Load Balancer Controller).
- The pod's container processes the request.
- Application calls downstream:
  - Reads from cache (Redis) first — fast path.
  - On cache miss, queries database.
  - May call other microservices via internal Service DNS.

**Step 5: Data tier (RDS, Redis, S3)**
- Database query goes through private subnet to RDS endpoint.
- DNS resolves to writer instance for writes, reader endpoint for reads.
- RDS returns result.
- Application aggregates response, returns to ALB.

**Step 6: Response path (reversed)**
- ALB → CloudFront (may cache based on headers) → user.

**Why three tiers specifically — design rationale:**

| Concern | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| Internet-facing | Yes | No | No |
| Scalability driver | Traffic volume | Compute | Storage / IOPS |
| Replaceable | Frontend rewrite | Service-by-service | Migrations require care |
| Security boundary | DDoS, WAF | App-level auth | Data-at-rest encryption |

The separation gives each tier independent scaling, security, and deployment lifecycle.

**Architect's framing:**
> "The three-tier model isn't just a diagram — it's a separation of concerns that lets each layer evolve independently. Presentation handles user experience and edge concerns; application handles business logic; data handles persistence. The decisions that matter most are CIDR planning for the VPC, AZ distribution for HA, and tight security group rules between tiers so traffic only flows in the intended direction."

---

## 2. CI/CD with Maven Build

> "Walking through both CI/CD concepts and concrete Maven commands.

**CI/CD overview:**

**Continuous Integration:** every code change automatically built, tested, scanned, and packaged into an artifact. Goal: catch issues immediately.

**Continuous Delivery:** the artifact is always deployable. Promotion to higher environments is gated but automated.

**Continuous Deployment:** the artifact reaches production automatically without manual intervention.

**Pipeline stages for a Java application:**

```
[1] Checkout         → git checkout the commit
[2] Build            → mvn compile
[3] Unit test        → mvn test
[4] Code coverage    → JaCoCo report
[5] SonarQube scan   → mvn sonar:sonar (quality gate)
[6] Dependency scan  → mvn dependency-check:check (OWASP)
[7] Package          → mvn package (.jar/.war)
[8] Image build      → docker build
[9] Image scan       → trivy image
[10] Image sign      → cosign sign
[11] Push to registry → docker push
[12] Update GitOps   → bump tag in manifests repo
[13] Deploy          → ArgoCD reconciles or kubectl apply
```

**Maven commands explained:**

```bash
# Clean previous build outputs
mvn clean

# Compile source code into target/classes
mvn compile

# Run unit tests (also runs compile)
mvn test

# Package the application as JAR/WAR (also runs test)
mvn package

# Install to local Maven repo (~/.m2) for use by other projects
mvn install

# Deploy to remote Maven repository (Nexus, Artifactory)
mvn deploy

# Common chained command — the workhorse
mvn clean install -DskipTests=false

# Production CI command
mvn -B clean verify         # -B = batch mode, no interactive prompts
                            # verify = runs test + integration tests + checks
```

**The Maven lifecycle (each phase runs all prior phases):**

```
validate → compile → test → package → verify → install → deploy
```

So `mvn package` automatically runs validate, compile, test before packaging.

**Production-grade Maven CI:**

```bash
# In CI pipeline
mvn -B clean verify \
    -Dmaven.repo.local=$WORKSPACE/.m2 \
    -Djacoco.destFile=$WORKSPACE/coverage/jacoco.exec

# With code quality
mvn sonar:sonar \
    -Dsonar.host.url=https://sonar.company.com \
    -Dsonar.qualitygate.wait=true

# With dependency vulnerability check
mvn org.owasp:dependency-check-maven:check \
    -DfailBuildOnCVSS=7
```

**Real CI workflow (GitHub Actions example):**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with: { distribution: temurin, java-version: '17', cache: maven }
    - run: mvn -B clean verify
    - uses: dorny/test-reporter@v1
      with: { name: JUnit, path: 'target/surefire-reports/*.xml', reporter: java-junit }
    - run: mvn sonar:sonar -Dsonar.qualitygate.wait=true
      env: { SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }} }
```

**The principle: build once, promote artifact.**
The same JAR/image deploys to dev, staging, and prod. Never rebuild for different environments — that breaks reproducibility and audit trail.

---

## 3. GitHub Workflow — Clone, Pull, Commit

> "Going through the day-to-day Git workflow.

**Initial setup (one-time):**

```bash
git config --global user.name "Rajesh Singh"
git config --global user.email "rajesh10.1990@gmail.com"
git config --global init.defaultBranch main
git config --global pull.rebase false           # use merge by default
```

**Cloning a repository:**

```bash
# HTTPS — works everywhere, requires PAT for push
git clone https://github.com/company/repo.git

# SSH — preferred for daily use, key-based auth
git clone git@github.com:company/repo.git

# Shallow clone (only recent history, faster for large repos)
git clone --depth 1 git@github.com:company/repo.git

# Clone specific branch
git clone -b develop git@github.com:company/repo.git
```

**Daily workflow — feature branch pattern:**

```bash
# Start from up-to-date main
git checkout main
git pull origin main

# Create feature branch
git checkout -b feature/add-payment-validation

# Make changes, then check what changed
git status                              # what's modified
git diff                                # see actual changes
git diff --staged                       # changes already staged

# Stage changes
git add src/payments/Validator.java     # specific file
git add src/payments/                   # specific directory
git add .                               # everything in current dir
git add -A                              # everything including deletions
git add -p                              # interactive — stage hunks selectively

# Commit
git commit -m "feat(payments): add input validation for amount field"

# Push to remote
git push origin feature/add-payment-validation

# First push of a new branch — sets upstream tracking
git push -u origin feature/add-payment-validation
```

**Pulling updates:**

```bash
# Fetch + merge (creates merge commit if both sides changed)
git pull origin main

# Fetch + rebase (rewrites local commits on top of remote, cleaner history)
git pull --rebase origin main

# Just fetch — see what's there without merging
git fetch origin
git log HEAD..origin/main               # see what would come in
```

**Working with PRs:**

```bash
# Update branch with latest main (while feature is in progress)
git checkout main
git pull origin main
git checkout feature/add-payment-validation
git rebase main                         # replay my commits on top of latest main
git push --force-with-lease             # safer than --force

# After PR merged
git checkout main
git pull origin main
git branch -d feature/add-payment-validation     # delete local
git push origin --delete feature/add-payment-validation   # delete remote (or use GitHub UI)
```

**Useful commands for daily work:**

```bash
git log --oneline -10                   # last 10 commits, compact
git log --graph --oneline --all         # visual branch graph
git blame src/payments/Validator.java   # who changed which line
git show <commit-sha>                   # full diff of a specific commit
git stash                               # save uncommitted work temporarily
git stash pop                           # restore stashed work
git reflog                              # safety net — see everything you've done
```

**Undoing things:**

```bash
git checkout -- file.java               # discard unstaged changes to file
git restore file.java                   # modern equivalent
git restore --staged file.java          # unstage (keep changes)
git reset --soft HEAD~1                 # undo last commit, keep changes staged
git reset --hard HEAD~1                 # nuke last commit AND changes — careful
git revert <commit-sha>                 # create new commit that undoes a past commit (safe)
```

**Commit message conventions (Conventional Commits):**

```
feat(payments): add input validation for amount field
fix(auth): handle expired token gracefully
chore(deps): bump spring-boot to 3.2.1
docs(readme): add Kubernetes deployment instructions
test(payments): cover edge cases in validator
refactor(api): extract pagination into utility
```

**Branching strategy I follow:**

- **`main`** — always deployable, protected.
- **`develop`** — integration branch (in GitFlow setups).
- **`feature/<name>`** — for new work.
- **`hotfix/<name>`** — emergency fixes branching from main.
- **`release/<version>`** — release stabilization.

For most modern teams, **trunk-based development** is simpler: everyone works off main with short-lived feature branches, feature flags for incomplete work.

**GitHub-specific workflow:**

1. Push branch to GitHub.
2. Open PR via web UI or `gh pr create`.
3. CI runs checks, PR template prompts for context.
4. Reviewers approve.
5. Merge via squash, rebase, or merge commit (depending on team policy).
6. Branch deleted automatically.

---

## 4. Pod Disruption Budget (PDB)

> "PDB is a Kubernetes resource that limits voluntary disruptions to ensure minimum availability during operations like node drains, cluster upgrades, or autoscaling scale-downs.

**The problem PDB solves:**

Imagine a 3-replica Deployment. During a node upgrade, Kubernetes wants to drain a node — which may evict 2 of your 3 pods simultaneously. Now you have 1 pod serving traffic, possibly overloaded. Without coordination, you'd have a brief outage during otherwise routine operations.

**PDB tells the cluster: 'don't take more than X pods out at once.'**

**Two ways to express it:**

```yaml
# Option A: minAvailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments-api-pdb
  namespace: payments
spec:
  minAvailable: 2          # at least 2 pods must stay up
  selector:
    matchLabels:
      app: payments-api

---
# Option B: maxUnavailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments-api-pdb
  namespace: payments
spec:
  maxUnavailable: 1        # at most 1 pod can be down at a time
  selector:
    matchLabels:
      app: payments-api
```

You can express the value as an integer or a percentage:

```yaml
spec:
  minAvailable: 75%        # for a Deployment with replicas=4 → at least 3 up
```

**Critical: PDB only protects against VOLUNTARY disruptions.**

| Disruption type | PDB protects? |
|---|---|
| `kubectl drain` (planned node maintenance) | Yes |
| Cluster autoscaler scale-down | Yes |
| Cluster upgrade pod evictions | Yes |
| Voluntary pod evictions via Eviction API | Yes |
| Node hardware failure | No |
| Kernel panic | No |
| OOMKill | No |
| Application crash | No |

PDB is a *cooperation contract* between you and the cluster operators, not a hard guarantee.

**Production patterns I follow:**

**For tier-1 services with 3+ replicas:**
```yaml
spec:
  minAvailable: 2          # keep at least 2 even during disruption
```

**For larger fleets:**
```yaml
spec:
  maxUnavailable: 25%      # at most 25% can be down at once
```

**Pitfalls and gotchas:**

**1. PDB with single-replica Deployment** — locks operations. You can't drain a node if `minAvailable: 1` and there's only 1 pod. Use `maxUnavailable: 0` only with explicit understanding.

**2. PDB blocks cluster autoscaler scale-down** — if removing a node would violate PDB, autoscaler skips that node. Underutilized clusters stay underutilized.

**3. PDB doesn't help with rolling updates** — Deployment's `maxUnavailable` in the rollout strategy handles that. PDB is for *outside* the normal rollout (node maintenance, etc.).

**4. PDB selector must match pods correctly** — wrong labels = no protection.

**Verifying:**

```bash
kubectl get pdb -n payments
# NAME              MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# payments-api-pdb  2               N/A               1

kubectl describe pdb payments-api-pdb -n payments
# Shows current state, disruptions allowed
```

**Real-world example from banking:**

At Finastra, for payment processing pods, we use `minAvailable: 2` on a 3-replica Deployment. Why: payment processing has tight SLAs. Even during routine node patching, we need 2 pods serving so spikes don't overwhelm a single replica.

**Architect's framing:**
> "PDB is the operational contract that tells the platform 'this is the minimum capacity my workload needs during routine operations.' Without PDBs, cluster maintenance becomes risky — operators must coordinate with every workload owner. With PDBs, operations are safe by default. The principle: every production workload with replicas > 1 should have a PDB. It's three lines of YAML that prevents an entire class of self-inflicted outages."

---

## 5. Readiness and Liveness Probes

> "Three probe types, each with a different purpose. Critical to understand the distinction.

**Liveness probe** — 'is this container still alive?'
- If fails: kubelet kills the container, Deployment restarts it.
- Use to recover from deadlocks, infinite loops, or wedged states.

**Readiness probe** — 'is this container ready to accept traffic?'
- If fails: kubelet removes pod from Service endpoints; pod still runs.
- Use to control when traffic flows to a pod.

**Startup probe** — 'has this container finished starting?'
- Runs before liveness/readiness take over.
- Use for slow-starting apps (JVM, large data loads) so liveness doesn't kill them prematurely.

**Common mistake:** treating liveness and readiness as the same thing. They're not.

**Full syntax in deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
spec:
  replicas: 3
  selector: { matchLabels: { app: payments-api } }
  template:
    metadata: { labels: { app: payments-api } }
    spec:
      containers:
      - name: api
        image: myapp:1.4.2
        ports:
        - containerPort: 8080
          name: http
        
        # Startup probe — gives slow-starting apps time
        startupProbe:
          httpGet:
            path: /actuator/health/startup
            port: http
          failureThreshold: 30        # 30 × 10s = 5 minutes max for startup
          periodSeconds: 10
          timeoutSeconds: 3
        
        # Readiness probe — controls traffic routing
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
          initialDelaySeconds: 0      # startupProbe handles delay
          periodSeconds: 5            # check every 5s
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 2         # fail after 2 consecutive failures
        
        # Liveness probe — restarts container if broken
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
          initialDelaySeconds: 0      # startupProbe handles delay
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3         # restart after 3 consecutive failures
```

**Three probe execution methods:**

```yaml
# HTTP GET (most common)
readinessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: probe-call

# TCP socket (when no HTTP endpoint)
livenessProbe:
  tcpSocket:
    port: 5432               # for databases

# Exec command (for custom checks)
livenessProbe:
  exec:
    command: ["/bin/sh", "-c", "pg_isready -U postgres"]
```

**Critical design principles:**

**1. Liveness should NOT check downstream dependencies.**

```yaml
# BAD
livenessProbe:
  httpGet:
    path: /health/deep        # this checks DB connection
```

If the DB has a 30-second blip, *all* your pods get restarted by the liveness probe. Cascade failure. Recovery takes way longer than the original DB issue.

```yaml
# GOOD
livenessProbe:
  httpGet:
    path: /health/liveness    # only checks the app process is alive
```

**2. Readiness CAN check downstream — that's its job.**

If the DB is down, the pod isn't ready to serve traffic. Stop routing requests to it.

```yaml
readinessProbe:
  httpGet:
    path: /health/readiness   # checks DB connectivity, cache, etc.
```

**3. Use startupProbe for slow-starting apps.**

JVM apps especially — Spring Boot can take 60+ seconds to start. Without startupProbe, liveness fails during startup and the pod gets killed in a loop.

```yaml
startupProbe:
  failureThreshold: 30        # 30 × 10s = 5 min total budget
  periodSeconds: 10
```

**Common values for a Spring Boot app:**

| Parameter | Startup | Readiness | Liveness |
|---|---|---|---|
| initialDelaySeconds | 0 | 0 | 0 |
| periodSeconds | 10 | 5 | 10 |
| timeoutSeconds | 3 | 3 | 3 |
| failureThreshold | 30 | 2 | 3 |
| successThreshold | 1 | 1 | 1 |

**Endpoints expected in code:**

```java
// Spring Boot Actuator
@RestController
@RequestMapping("/actuator/health")
public class HealthController {
    
    // Liveness — just checks the process can respond
    @GetMapping("/liveness")
    public Map<String, String> liveness() {
        return Map.of("status", "UP");
    }
    
    // Readiness — checks dependencies
    @GetMapping("/readiness")
    public ResponseEntity<Map<String, String>> readiness() {
        if (!databaseConnected() || !cacheReachable()) {
            return ResponseEntity.status(503).body(Map.of("status", "DOWN"));
        }
        return ResponseEntity.ok(Map.of("status", "UP"));
    }
}
```

**Architect's framing:**
> "Liveness recovers from wedged processes; readiness controls traffic. Mixing them — putting downstream checks on liveness — is the single most common probe mistake, and it causes massive cascade failures during dependency outages. The discipline: liveness is a 'can this process respond at all' check; readiness reflects 'should I send work here right now.'"

---

## 6. Practical Steps After CI/CD Build

> "Once the build completes, the operational reality starts. Here's what happens next:

**Stage 1: Artifact handling**

- **Image push to registry** — to ECR/ACR/GCR with both immutable tag and any floating tags (rare in prod).
- **Helm chart packaging** — if using Helm, package and push the chart as an OCI artifact.
- **SBOM generation** — Software Bill of Materials for compliance.
- **Image signing** — Cosign signs the image; signature stored in registry.

```bash
# Image push
docker tag myapp:local 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2-a1b2c3d
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2-a1b2c3d

# SBOM
syft myapp:local -o spdx-json > sbom.json

# Signing
cosign sign --yes 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2-a1b2c3d
```

**Stage 2: Image vulnerability scanning**

- **Trivy / Snyk / Inspector** scans the pushed image.
- Findings published to Security Hub or dashboard.
- Alert on critical/high CVEs without fixes.

```bash
trivy image --severity CRITICAL,HIGH --exit-code 1 \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2-a1b2c3d
```

**Stage 3: Deployment preparation (GitOps pattern)**

- **Image tag bump in GitOps repo** — pipeline commits to a separate manifests repo.

```bash
git clone https://github.com/company/gitops-config
cd gitops-config
yq -i '.image.tag = "1.4.2-a1b2c3d"' environments/dev/values.yaml
git add . && git commit -m "chore(dev): bump myapp to 1.4.2-a1b2c3d"
git push
```

**Stage 4: Automated deployment**

- **ArgoCD watches the GitOps repo** — detects the change.
- **Reconciles to Kubernetes** — applies the new manifest.
- **Kubernetes does rolling update** — new pods come up, old ones drain.

```bash
# Or if not using GitOps
helm upgrade --install myapp ./chart \
    -f values-dev.yaml \
    --set image.tag=1.4.2-a1b2c3d \
    --atomic --wait --timeout 10m
```

**Stage 5: Verification**

- **Health checks** — verify all pods are Running and Ready.
- **Smoke tests** — run a small test suite against the deployed environment.
- **Synthetic monitoring** — verify the user-facing endpoints work.

```bash
kubectl rollout status deployment/myapp -n dev --timeout=5m
kubectl get pods -n dev -l app=myapp

# Run smoke tests
curl -f https://myapp.dev.example.com/health
pytest tests/smoke/ --url=https://myapp.dev.example.com
```

**Stage 6: Observability hookup**

- **Metrics flowing** — Prometheus scraping the new pods.
- **Logs aggregating** — Fluent Bit → Loki / CloudWatch.
- **Traces** — OpenTelemetry sending to Tempo / Jaeger.
- **Dashboards updated** — new deploy annotation on Grafana dashboards.

**Stage 7: Promotion gates**

- **Dev to Staging** — auto-promote after dev smoke tests pass.
- **Staging to UAT** — approval gate (often QA team approval).
- **UAT to Production** — formal approval, change advisory board ticket linked, two-person review for tier-1 services.

**Stage 8: Production deployment**

- **Canary rollout** for tier-1 services (Argo Rollouts).
- **Blue-green** for major version cuts.
- **Standard rolling** for most services.
- **SLO-based monitoring** during rollout — auto-abort if error rate or latency regresses.

**Stage 9: Post-deploy**

- **Tag the release in Git** for traceability.
- **Update changelog** with deployed version.
- **Notify stakeholders** via Slack/Teams (automated).
- **Update internal status page** if customer-impacting.

**Stage 10: Day-2 monitoring**

- **Watch SLO dashboards** for 24-48 hours.
- **Compare error budget burn rate** before/after.
- **Check application logs** for any new error patterns.
- **Review APM data** — latency distribution, throughput, dependencies.

**My typical pipeline at Finastra:**

```
Build (5 min) → Scan (3 min) → Push (2 min) →
Deploy Dev (auto, 5 min) → Smoke test (2 min) →
PR to Staging (manual, hours) → Deploy Staging → UAT testing →
PR to Production (formal, days) → Canary deploy →
Monitor → Promote → Complete rollout
```

Total time from commit to production: typically 2-5 days for non-urgent changes, hours for urgent fixes.

**Architect's framing:**
> "The build completing is the *start* of the deployment journey, not the end. Production-grade deployment includes signed artifacts, scanned images, GitOps reconciliation, canary verification, SLO monitoring, and stakeholder communication. The discipline is making each step automated and observable — manual deployment steps are where incidents are born."

---

## 7. Deployment.yaml Manifest

> "Production-grade Deployment manifest with all the elements I'd include for a tier-1 service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  namespace: payments
  labels:
    app: payments-api
    tier: backend
    team: platform
    cost-center: payments-platform
  annotations:
    description: "Payments processing API"
    repository: "github.com/company/payments-api"

spec:
  replicas: 3
  revisionHistoryLimit: 10              # keep 10 ReplicaSets for rollback
  progressDeadlineSeconds: 600

  selector:
    matchLabels:
      app: payments-api

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2                       # up to 2 extra pods during rollout
      maxUnavailable: 0                 # never go below desired replicas

  template:
    metadata:
      labels:
        app: payments-api
        tier: backend
        version: v1.4.2
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
        checksum/config: "{{ include 'mychart/configmap.yaml' . | sha256sum }}"

    spec:
      # Use a specific service account (for IRSA on EKS)
      serviceAccountName: payments-api-sa
      automountServiceAccountToken: true

      # Graceful shutdown
      terminationGracePeriodSeconds: 30

      # Security context at pod level
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      # Distribute pods across AZs
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: payments-api

      # Prefer not to schedule on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: payments-api

      # Allow scheduling on tainted nodes (if applicable)
      tolerations:
      - key: workload-type
        operator: Equal
        value: payments
        effect: NoSchedule

      # Image pull secret (if not using IRSA-based ECR auth)
      imagePullSecrets:
      - name: ecr-credentials

      # Init container — wait for dependencies
      initContainers:
      - name: wait-for-db
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          until nc -z payments-db.payments.svc.cluster.local 5432; do
            echo "waiting for db..."
            sleep 2
          done
        resources:
          requests: { cpu: 10m, memory: 32Mi }
          limits: { cpu: 100m, memory: 64Mi }

      containers:
      - name: api
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payments-api:1.4.2-a1b2c3d
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP

        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        - name: POD_NAMESPACE
          valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
        - name: POD_IP
          valueFrom: { fieldRef: { fieldPath: status.podIP } }

        # Bulk config
        envFrom:
        - configMapRef:
            name: payments-config
        - secretRef:
            name: payments-secrets

        # Resource boundaries
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi

        # Probes
        startupProbe:
          httpGet:
            path: /actuator/health/startup
            port: http
          failureThreshold: 30
          periodSeconds: 10
          timeoutSeconds: 3

        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2

        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

        # Lifecycle hooks
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - "sleep 15 && /app/graceful-shutdown.sh"

        # Container-level security
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]

        # Volumes for read-only root filesystem
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /var/log
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true

      volumes:
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}
      - name: secrets
        secret:
          secretName: payments-secrets

---
# Service to expose the deployment
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  namespace: payments
spec:
  selector:
    app: payments-api
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: metrics
    port: 9090
    targetPort: metrics
  type: ClusterIP

---
# PDB for HA during disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments-api
  namespace: payments
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payments-api

---
# HPA for autoscaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payments-api
  namespace: payments
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payments-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - { type: Percent, value: 100, periodSeconds: 30 }
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - { type: Percent, value: 25, periodSeconds: 60 }
```

**What this manifest gets right:**

- **Probes** for proper lifecycle handling (startup, readiness, liveness).
- **Resources** defined — requests for scheduling, limits for fairness.
- **Security context** — non-root, read-only root filesystem, dropped capabilities.
- **PDB** for HA during disruptions.
- **HPA** for elastic scaling.
- **Topology spread** for AZ distribution.
- **Anti-affinity** to avoid co-location.
- **Init container** to wait for dependencies.
- **PreStop hook** for graceful shutdown.
- **Prometheus annotations** for metrics scraping.

**For production, also create:**
- **NetworkPolicy** for zero-trust pod-to-pod.
- **ServiceAccount + Role + RoleBinding** for RBAC.
- **Ingress** for external traffic.

---

## 8. Multi-Stage Dockerfile

> "Multi-stage builds are essential for production images — separate build environment from runtime. I'll show one for a Spring Boot Java app:

```dockerfile
# syntax=docker/dockerfile:1.6

# ─── Stage 1: Build dependencies (separate layer for caching) ──────────
FROM maven:3.9-eclipse-temurin-17 AS deps
WORKDIR /build

# Copy pom.xml first — cached if deps don't change
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B dependency:go-offline

# ─── Stage 2: Build application ──────────────────────────────────────────
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build

# Copy dependencies from previous stage
COPY --from=deps /root/.m2 /root/.m2

# Copy pom and source
COPY pom.xml .
COPY src ./src

# Build with cache mount for Maven repo
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B clean package -DskipTests

# Extract layered jar (Spring Boot 2.3+ feature for better Docker caching)
RUN java -Djarmode=layertools -jar target/*.jar extract --destination target/extracted

# ─── Stage 3: Runtime ────────────────────────────────────────────────────
FROM eclipse-temurin:17-jre-alpine AS runtime

# Install tini for proper signal handling
RUN apk add --no-cache tini

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring -u 1000

WORKDIR /app

# Copy layered jar contents — each layer cached separately
# Order matters: changes least to most frequent
COPY --from=build --chown=spring:spring /build/target/extracted/dependencies/ ./
COPY --from=build --chown=spring:spring /build/target/extracted/spring-boot-loader/ ./
COPY --from=build --chown=spring:spring /build/target/extracted/snapshot-dependencies/ ./
COPY --from=build --chown=spring:spring /build/target/extracted/application/ ./

USER spring:spring

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["/sbin/tini", "--", "java", "org.springframework.boot.loader.JarLauncher"]
```

**Companion `.dockerignore`:**

```
# Git
.git
.gitignore
.gitattributes

# IDE
.idea
.vscode
*.iml

# Build artifacts not needed for the build itself
target/*.jar
target/classes
target/test-classes

# Logs and temp files
*.log
*.tmp
*.swp

# Docs
README.md
*.md

# Secrets (defense in depth)
.env*
*.pem
*.key

# Dockerfile itself
Dockerfile*
.dockerignore
```

**Build:**

```bash
DOCKER_BUILDKIT=1 docker build -t payments-api:1.4.2 .
```

**For Node.js — different but same principle:**

```dockerfile
# syntax=docker/dockerfile:1.6

# ─── Build stage ──────────────────────────────────────────────────────
FROM node:20-alpine AS build
WORKDIR /build

COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

COPY . .
RUN npm run build

# ─── Production dependencies (separate stage for caching) ─────────────
FROM node:20-alpine AS deps
WORKDIR /deps
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --no-audit --no-fund

# ─── Runtime ──────────────────────────────────────────────────────────
FROM node:20-alpine AS runtime
RUN apk add --no-cache tini
WORKDIR /app

ENV NODE_ENV=production PORT=3000

COPY --chown=node:node --from=deps  /deps/node_modules ./node_modules
COPY --chown=node:node --from=build /build/dist ./dist
COPY --chown=node:node package.json ./

USER node
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
    CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/server.js"]
```

**Size impact (real data):**

| Approach | Image Size |
|---|---|
| Single-stage with `maven:latest` | ~900 MB |
| Multi-stage with `openjdk:17` runtime | ~280 MB |
| Multi-stage with `eclipse-temurin:17-jre-alpine` | ~180 MB |
| Multi-stage with distroless | ~150 MB |

**What multi-stage achieves:**

- **No build tools in production image** (no Maven, no JDK in runtime — only JRE).
- **Layered jar** (Spring Boot 2.3+) means dependency changes don't invalidate the entire jar layer.
- **Cache mount for Maven repo** speeds rebuilds.
- **Non-root user** for security.
- **Tini** for proper PID 1 signal handling.
- **HEALTHCHECK** for orchestrator visibility.

**Architect's framing:**
> "Multi-stage is non-negotiable for production. It separates build-time concerns from runtime, dramatically reduces image size, and reduces attack surface. The build stage has Maven, JDK, source — none of which should ship to production. The runtime stage has only what's needed to run. The layered jar approach is icing on the cake — when only application code changes, dependency layers stay cached, making rebuilds and pulls much faster."

---

## 9. Pod Crashes — Changes in deployment.yaml

> "Depends on the type of crash. Here's the diagnostic-to-fix mapping:

**If OOMKilled (exit code 137):**

```yaml
# BEFORE — memory too tight
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { cpu: 1000m, memory: 512Mi }

# AFTER — proper headroom based on observed usage + buffer
resources:
  requests: { cpu: 250m, memory: 768Mi }
  limits:   { cpu: 1000m, memory: 1.5Gi }
```

For JVM apps, also tune heap explicitly:
```yaml
env:
- name: JAVA_OPTS
  value: "-XX:MaxRAMPercentage=75 -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError"
```

**If liveness probe killing the pod:**

```yaml
# BEFORE — probe too aggressive for slow start
livenessProbe:
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

# AFTER — add startup probe so liveness doesn't fire during startup
startupProbe:
  httpGet: { path: /actuator/health/startup, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10

livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
```

**If readiness probe causing traffic to wrong pods:**

```yaml
# Add proper readiness with downstream dependency checks
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2
```

**If pod evicted due to node pressure:**

```yaml
# Add explicit QoS class — guaranteed pods evicted last
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 500m       # same as requests = Guaranteed QoS
    memory: 1Gi

# Or add priority class
spec:
  priorityClassName: high-priority
```

**If pod crashes from missing configuration:**

```yaml
# Validate config exists via init container
initContainers:
- name: config-check
  image: busybox
  command:
  - sh
  - -c
  - |
    if [ ! -f /etc/config/required.yaml ]; then
      echo "ERROR: required config missing"
      exit 1
    fi
  volumeMounts:
  - name: config
    mountPath: /etc/config
```

**If pod crashes during shutdown (preStop issues):**

```yaml
# Increase grace period; add preStop hook
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command:
          - sh
          - -c
          - "sleep 15 && /app/graceful-shutdown.sh"
```

**Adding observability for diagnosis:**

```yaml
# Heap dump on OOM
env:
- name: JAVA_OPTS
  value: "-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps"
volumeMounts:
- name: dumps
  mountPath: /dumps
volumes:
- name: dumps
  emptyDir: { sizeLimit: 1Gi }
```

**For repeated crashes — increase replicas and add PDB:**

```yaml
spec:
  replicas: 5     # more replicas so individual crashes don't impact availability

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: myapp-pdb }
spec:
  minAvailable: 3
  selector: { matchLabels: { app: myapp } }
```

**If crash is during scale events — graceful handling:**

```yaml
# Better autoscaling — scale up faster, scale down slower
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600   # wait 10min before scaling down
      policies:
      - { type: Percent, value: 10, periodSeconds: 120 }
```

**General principle:** the deployment.yaml change should be informed by *what kind of crash* is happening:

| Crash type | Fix in deployment.yaml |
|---|---|
| OOMKill | Higher memory limit, JVM heap tuning |
| Liveness probe killing | Add startupProbe, decouple liveness from dependencies |
| Eviction due to node pressure | Higher QoS, priority class, more replicas |
| Missing config | Init container validation, ensure ConfigMap exists |
| Shutdown crash | Longer grace period, preStop hook |
| Repeated app crashes | More replicas + PDB while investigating root cause |

**Architect's framing:**
> "The diagnostic question 'why is the pod crashing' must come *before* changing the manifest. OOMKill is a different problem from liveness-probe-kill, which is different from eviction. Treating them all as 'increase memory' wastes resources and obscures the real issue. The right sequence: check `kubectl describe pod` for the actual reason, read `kubectl logs --previous` for the application's view, then make the targeted change."

---

## 10. ADD vs COPY in Docker

> "Both copy files from the build context into the image, but with different capabilities and use cases.

**COPY** — simple, predictable. Just copies files/directories from build context to image.

```dockerfile
COPY app.jar /app/app.jar
COPY src/ /build/src/
COPY --from=build /build/dist /app/dist     # multi-stage
COPY --chown=user:group config.yaml /etc/app/
```

**ADD** — does everything COPY does, *plus*:
- Auto-extracts local tar archives (`ADD app.tar.gz /app` extracts into /app).
- Can fetch a remote URL (`ADD https://example.com/file.tar.gz /tmp/`).

```dockerfile
ADD app.tar.gz /app                          # auto-extracts
ADD https://example.com/release.tar.gz /tmp/ # fetches and stores (no extract for URL)
```

**The Docker documentation explicitly recommends COPY over ADD** for most cases.

**Comparison table:**

| Feature | COPY | ADD |
|---|---|---|
| Copy local files | ✓ | ✓ |
| Copy with permissions | ✓ | ✓ |
| Multi-stage `--from=` | ✓ | ✓ |
| Auto-extract tar | ✗ | ✓ |
| Fetch from URL | ✗ | ✓ |
| Predictable behavior | ✓ | ✗ (varies by source) |
| Recommended | ✓ | Specific cases only |

**Why prefer COPY:**

**1. Predictable behavior**
COPY does one thing: copies files. ADD might silently extract a tar based on file extension, which can confuse readers.

```dockerfile
# Reader's question: "what does this do?"
ADD app.tar.gz /app

# COPY makes intent explicit
COPY app.tar.gz /app/app.tar.gz
RUN tar -xzf /app/app.tar.gz -C /app && rm /app/app.tar.gz
```

**2. URL fetching via ADD is discouraged**
- Can't verify checksums in one step.
- Adds remote dependency at build time.
- Better to use `RUN` with explicit checksum verification:

```dockerfile
# BAD — opaque, no integrity check
ADD https://example.com/release.tar.gz /tmp/

# GOOD — explicit and verifiable
RUN wget https://example.com/release.tar.gz -O /tmp/release.tar.gz \
    && echo "expected-sha256 *release.tar.gz" | sha256sum -c \
    && tar -xzf /tmp/release.tar.gz -C /opt \
    && rm /tmp/release.tar.gz
```

**3. COPY is faster**
ADD has overhead from URL detection and tar inspection. COPY is a pure file operation.

**4. Better caching behavior**
With ADD URL, Docker can't easily determine if the remote file changed without fetching it. COPY's caching is based on file contents — deterministic.

**When to use ADD:**

- **Pre-tar'd application bundles** where extraction is the intent:
```dockerfile
ADD release-bundle.tar.gz /opt/myapp/   # extracts into /opt/myapp
```

- **Multi-stage builds with extraction** — but `RUN tar` is usually clearer:
```dockerfile
# Slightly cleaner with ADD for bundled apps
ADD --from=build /build/release.tar.gz /opt/app/
```

**Practical recommendation:**

```dockerfile
# 95% of the time
COPY src/ /app/src/
COPY package.json /app/
COPY --from=build /build/dist /app/dist

# When extracting a tarball
RUN curl -fsSL https://example.com/release.tar.gz | tar -xz -C /opt
# OR (with explicit verification)
COPY release.tar.gz /tmp/
RUN sha256sum -c /tmp/release.tar.gz.sha256 \
    && tar -xzf /tmp/release.tar.gz -C /opt \
    && rm /tmp/release.tar.gz
```

**Architect's framing:**
> "COPY is boring and predictable — exactly what you want in infrastructure code. ADD's magic features (auto-extract, URL fetch) sound convenient but make Dockerfiles harder to read and reason about. The principle: prefer explicit operations (RUN curl + verify + extract) over implicit ones (ADD URL). In code review, ADD with URL is a red flag because it can't be reproduced if the URL changes or disappears."

---

## 11. CI/CD Pipeline for Java App — Dev and Prod

> "Full GitHub Actions pipeline for a Spring Boot app deploying to Kubernetes:

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  id-token: write       # OIDC for AWS
  contents: read
  pull-requests: write

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: payments-api
  JAVA_VERSION: '17'

jobs:
  # ────────────────────────────────────────────────────────────────────
  # CI: Build, test, scan
  # ────────────────────────────────────────────────────────────────────
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
      sha_short: ${{ steps.meta.outputs.sha_short }}
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      with: { fetch-depth: 0 }
    
    - name: Set image tag
      id: meta
      run: |
        SHORT_SHA="${GITHUB_SHA::7}"
        TAG="$(date +%Y%m%d)-${SHORT_SHA}-${GITHUB_RUN_NUMBER}"
        echo "tag=$TAG" >> $GITHUB_OUTPUT
        echo "sha_short=$SHORT_SHA" >> $GITHUB_OUTPUT
    
    - name: Setup Java
      uses: actions/setup-java@v4
      with:
        distribution: temurin
        java-version: ${{ env.JAVA_VERSION }}
        cache: maven
    
    - name: Cache SonarQube packages
      uses: actions/cache@v4
      with:
        path: ~/.sonar/cache
        key: ${{ runner.os }}-sonar
    
    - name: Compile
      run: mvn -B compile
    
    - name: Unit tests
      run: mvn -B test
    
    - name: Code coverage report
      run: mvn -B jacoco:report
    
    - name: Publish test results
      uses: dorny/test-reporter@v1
      if: always()
      with:
        name: JUnit Tests
        path: target/surefire-reports/*.xml
        reporter: java-junit
    
    - name: Integration tests
      run: mvn -B verify -DskipUnitTests
    
    - name: SonarQube analysis
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      run: |
        mvn -B sonar:sonar \
          -Dsonar.host.url=${{ secrets.SONAR_URL }} \
          -Dsonar.qualitygate.wait=true \
          -Dsonar.projectKey=payments-api
    
    - name: OWASP Dependency Check
      run: |
        mvn -B org.owasp:dependency-check-maven:check \
          -DfailBuildOnCVSS=7 \
          -DsuppressionFiles=dependency-check-suppressions.xml
    
    - name: Snyk dependency scan
      uses: snyk/actions/maven@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

    - name: Package JAR
      run: mvn -B package -DskipTests
    
    - name: Upload JAR artifact
      uses: actions/upload-artifact@v4
      with:
        name: jar-${{ steps.meta.outputs.tag }}
        path: target/*.jar
        retention-days: 7

  # ────────────────────────────────────────────────────────────────────
  # Build container image, scan, sign, push
  # ────────────────────────────────────────────────────────────────────
  build-image:
    name: Build & Push Image
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/gha-ci-role
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to ECR
      id: ecr
      uses: aws-actions/amazon-ecr-login@v2
    
    - name: Setup Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Build and push image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        platforms: linux/amd64,linux/arm64
        tags: |
          ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ needs.build-and-test.outputs.image_tag }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        provenance: true
        sbom: true
    
    - name: Trivy image scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ needs.build-and-test.outputs.image_tag }}
        severity: CRITICAL,HIGH
        exit-code: '1'
        ignore-unfixed: true
        format: sarif
        output: trivy-results.sarif
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with: { sarif_file: trivy-results.sarif }
    
    - name: Install Cosign
      uses: sigstore/cosign-installer@v3
    
    - name: Sign image (keyless OIDC)
      run: |
        cosign sign --yes \
          ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ needs.build-and-test.outputs.image_tag }}

  # ────────────────────────────────────────────────────────────────────
  # Deploy to Dev — automatic on develop branch
  # ────────────────────────────────────────────────────────────────────
  deploy-dev:
    name: Deploy to Dev
    needs: [build-and-test, build-image]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: dev
      url: https://payments-api.dev.example.com
    
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: company/gitops-config
        token: ${{ secrets.GITOPS_PAT }}
        path: gitops
    
    - name: Update image tag in dev values
      working-directory: gitops
      run: |
        yq -i '.image.tag = "${{ needs.build-and-test.outputs.image_tag }}"' \
          environments/dev/payments-api/values.yaml
    
    - name: Commit & push to GitOps
      working-directory: gitops
      run: |
        git config user.name "ci-bot"
        git config user.email "ci-bot@company.com"
        git add .
        git commit -m "chore(dev): bump payments-api to ${{ needs.build-and-test.outputs.image_tag }}"
        git push
    
    - name: Wait for ArgoCD sync
      run: |
        # ArgoCD picks up the change and syncs within seconds
        sleep 30
        # Could also poll ArgoCD API for sync status
    
    - name: Smoke tests
      run: |
        curl -f https://payments-api.dev.example.com/actuator/health/readiness || exit 1
        # More comprehensive smoke tests
        pytest tests/smoke/ --url=https://payments-api.dev.example.com
    
    - name: Notify Slack
      uses: 8398a7/action-slack@v3
      if: always()
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        text: 'Dev deploy: ${{ needs.build-and-test.outputs.image_tag }}'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # ────────────────────────────────────────────────────────────────────
  # Deploy to Production — requires approval, only on main
  # ────────────────────────────────────────────────────────────────────
  deploy-prod:
    name: Deploy to Production
    needs: [build-and-test, build-image]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production               # GitHub Environment with required reviewers
      url: https://payments-api.example.com
    
    steps:
    - name: Checkout GitOps repo
      uses: actions/checkout@v4
      with:
        repository: company/gitops-config
        token: ${{ secrets.GITOPS_PAT }}
        path: gitops
    
    - name: Validate change ticket
      run: |
        # Verify a change ticket exists for this deploy
        # (e.g., query ServiceNow API)
        # Block deploy if no approved CR
    
    - name: Update image tag in prod values
      working-directory: gitops
      run: |
        yq -i '.image.tag = "${{ needs.build-and-test.outputs.image_tag }}"' \
          environments/prod/payments-api/values.yaml
    
    - name: Create deployment PR
      working-directory: gitops
      run: |
        BRANCH="deploy/prod-${{ needs.build-and-test.outputs.image_tag }}"
        git checkout -b "$BRANCH"
        git config user.name "ci-bot"
        git config user.email "ci-bot@company.com"
        git add .
        git commit -m "chore(prod): deploy payments-api ${{ needs.build-and-test.outputs.image_tag }}"
        git push origin "$BRANCH"
        gh pr create \
          --title "Deploy payments-api ${{ needs.build-and-test.outputs.image_tag }} to prod" \
          --body "Promotion of ${{ needs.build-and-test.outputs.sha_short }} validated in dev/staging" \
          --base main \
          --reviewer release-managers
      env:
        GH_TOKEN: ${{ secrets.GITOPS_PAT }}
    
    - name: Wait for canary analysis
      run: |
        # ArgoCD picks up the PR after merge
        # Argo Rollouts performs canary deployment with SLO analysis
        # Wait for rollout to complete
        sleep 300   # rough placeholder; in reality, poll Argo Rollouts
    
    - name: Verify SLO compliance
      run: |
        # Query Prometheus for SLO metrics
        # Fail if error rate or latency regressed
        ./scripts/verify-slo.sh prod payments-api
    
    - name: Tag release
      run: |
        git tag -a "v${{ needs.build-and-test.outputs.image_tag }}" \
          -m "Production release ${{ needs.build-and-test.outputs.image_tag }}"
        git push origin "v${{ needs.build-and-test.outputs.image_tag }}"
    
    - name: Update changelog
      run: ./scripts/update-changelog.sh ${{ needs.build-and-test.outputs.image_tag }}
    
    - name: Notify stakeholders
      uses: 8398a7/action-slack@v3
      if: always()
      with:
        status: ${{ job.status }}
        channel: '#deployments-prod'
        text: '🚀 Prod deploy: payments-api ${{ needs.build-and-test.outputs.image_tag }}'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**Supporting GitOps repo structure:**

```
gitops-config/
├── environments/
│   ├── dev/
│   │   └── payments-api/
│   │       ├── values.yaml          # ← pipeline bumps image.tag here
│   │       └── application.yaml     # ArgoCD Application
│   └── prod/
│       └── payments-api/
│           ├── values.yaml          # ← pipeline bumps image.tag here
│           └── application.yaml
└── apps/
    └── payments-api/                 # Helm chart (or Kustomize base)
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml
            ├── hpa.yaml
            └── pdb.yaml
```

**ArgoCD Application example:**

```yaml
# environments/dev/payments-api/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-config
    targetRevision: HEAD
    path: apps/payments-api
    helm:
      valueFiles:
      - ../../environments/dev/payments-api/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: payments-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Argo Rollouts for production canary:**

```yaml
# Used in production helm template
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: payments-api }
spec:
  replicas: 6
  strategy:
    canary:
      maxSurge: 2
      maxUnavailable: 1
      steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 25
      - pause: { duration: 10m }
      - setWeight: 50
      - pause: { duration: 10m }
      - setWeight: 100
```

**Key principles in this pipeline:**

- **Build once, promote artifact** — same image goes from dev to prod.
- **OIDC federation** — no static AWS credentials in GitHub.
- **Quality gates** — Sonar, Snyk, Trivy all gate.
- **Image signing** — Cosign signs, Kyverno verifies in cluster.
- **GitOps separation** — pipeline commits to GitOps repo; ArgoCD does the actual deploy.
- **Environment-gated deploys** — GitHub Environments with required reviewers for prod.
- **Dev: auto, Prod: PR-based** — Dev gets continuous flow, Prod gets explicit promotion.
- **Canary in production** — Argo Rollouts with SLO-based abort.
- **Audit trail** — every deploy ties back to a Git commit and approval.

**The full flow from commit to prod:**

```
1. Dev pushes to develop
2. CI runs: build + test + scan + image build + push to ECR
3. CI commits to GitOps repo bumping dev/values.yaml
4. ArgoCD syncs dev cluster
5. Smoke tests run; CI continues
6. After validation in dev, dev opens PR to main
7. PR merged to main triggers prod deployment job
8. Job opens PR in GitOps repo bumping prod/values.yaml
9. Release manager approves the GitOps PR
10. ArgoCD picks up merged change
11. Argo Rollouts canary deployment starts
12. SLO monitoring during canary; auto-abort on regression
13. Full rollout if canary healthy
14. Notify stakeholders + tag release
```

**Architect's framing:**
> "This pipeline embodies several principles: build once and promote, separate CI from CD via GitOps, gate prod with formal approval, use canary deployments with SLO-based safety nets, sign and verify images, federate identity instead of using static credentials. The result is a fast path for dev iteration and a safe path for production changes — and the same artifact flows through both, with the only difference being the level of process applied. Banking-specific additions are the change ticket validation and the audit trail through the GitOps repo."

---

That covers all 11 questions. Likely follow-ups to be ready for:

- **"How do you handle a Helm chart upgrade that breaks?"** — `helm rollback`, atomic upgrades with `--atomic --wait`.
- **"How do you handle schema migrations alongside Kubernetes deploys?"** — pre-deploy migrations via Helm hooks or init containers, expand-contract for breaking changes.
- **"What's your approach to secret rotation?"** — External Secrets Operator with auto-rotation, Vault dynamic secrets, IAM auth where possible.
- **"How do you handle a failed canary that needs immediate rollback?"** — Argo Rollouts auto-aborts on SLO regression; pre-promotion analysis gates each step; manual `kubectl argo rollouts abort` if needed.

Your Finastra/banking background lets you go deep on the operational and compliance aspects — emphasize the audit trail, segregation of duties, and security gates throughout the pipeline.