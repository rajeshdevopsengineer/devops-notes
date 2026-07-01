Two more companies' worth of questions. I'll keep the ones that overlap with earlier rounds (OOMKilled, stuck state lock, exposed secret, 60s latency, observability, ArgoCD, HPA, CrashLoopBackOff) brief and point back, and spend the depth on the genuinely new material — the Capgemini-specific items (Git, Maven, GCP networking/storage, probes, Terraform count vs for_each) and the two fresh advanced ones: **error budget/burn rate** and **Platform Engineering/IDP**.

## Capgemini — Git / Source Control

**1. Git Fetch vs Git Merge**

Different operations entirely. `git fetch` **downloads** commits/refs from the remote into your local repo (updates remote-tracking branches like `origin/main`) but does **not** change your working branch — it's safe, non-destructive, lets you inspect before integrating. `git merge` **integrates** changes from one branch into your current branch, creating a merge commit (or fast-forward). `git pull` = `fetch` + `merge` in one step. The distinction to state: *"Fetch retrieves but doesn't touch my working branch; merge actually combines the changes into it. I often fetch first to review what changed before merging."*

**2. Conflicts you've seen in Git**

Talk from experience: **merge conflicts** when two branches edit the same lines of the same file (most common); **conflicts during rebase** (replaying commits onto a moved base); **delete/modify conflicts** (one branch deletes a file another edited); and binary file conflicts. How you resolve: open the conflicted file, find the `<<<<<<<`/`=======`/`>>>>>>>` markers, decide which changes to keep (or combine), then `git add` and complete the merge/rebase. Prevention: small frequent merges, communicate on shared files, pull often.

## Docker

**3. Docker Client Architecture**

Docker uses a **client-server architecture**. The **Docker client** (the `docker` CLI) is what you type commands into; it talks to the **Docker daemon (`dockerd`)** via a REST API (over a Unix socket or network). The daemon does the heavy lifting — building images, running containers, managing volumes/networks. Below the daemon, **containerd** manages the container lifecycle and **runc** is the low-level runtime that actually creates containers using Linux namespaces and cgroups. The client and daemon can be on the same host or remote. **Registry** (Docker Hub/ECR/etc.) is where images are pulled/pushed. Mental model: *"client sends commands → daemon executes via containerd/runc → images come from a registry."*

## Maven

**4. Maven Phases**

Maven's default **build lifecycle** runs phases in order, each including the previous: **validate** → **compile** → **test** (unit tests) → **package** (build the JAR/WAR) → **verify** (integration tests/checks) → **install** (put artifact in local repo) → **deploy** (push to remote repo). Running a phase runs all phases before it (e.g., `mvn package` also validates, compiles, tests). There are also `clean` (separate lifecycle, removes `target/`) and `site` lifecycles. The point to land: *"Phases are sequential and cumulative — `mvn install` runs everything from validate through install."*

## CI/CD

**5. Different types of pipelines**

A few ways to interpret this — cover the main framings: **CI pipelines** (build + test on commit) vs **CD pipelines** (deploy to environments). In Jenkins specifically: **Declarative** vs **Scripted** pipelines, and **single-branch** vs **Multibranch** pipelines. By execution: **sequential vs parallel** stages. By trigger/pattern: **push-based** (CI triggers deploy) vs **pull-based/GitOps** (ArgoCD reconciles). If it's a Jenkins-context question, lead with **Declarative vs Scripted**; if general, lead with **CI vs CD**.

## GCP Networking

**6. Shared VPC vs VPC Peering**

- **Shared VPC**: a single VPC in a **host project** is shared with multiple **service projects** in the *same organization*. Centralized network admin owns the network; service projects deploy resources into the shared subnets. Used for **centralized control across projects in one org**.
- **VPC Peering**: connects **two separate VPCs** (can be in different projects/orgs) so they communicate via private IPs. Each side manages its own VPC; it's a **peer relationship**, not shared ownership. Non-transitive (A↔B and B↔C doesn't give A↔C).

One-liner: *"Shared VPC = one network shared across projects under central admin; VPC Peering = two independent networks connected privately, each self-managed."*

## Kubernetes Basics

**7. DaemonSet**

Ensures a copy of a pod runs on **every node** (or a selected subset). When nodes join, the pod is automatically added; when they leave, it's removed. Used for **node-level agents**: log collectors (Fluentd), monitoring agents (node-exporter), networking (CNI, kube-proxy), storage daemons. Mental model: *"one pod per node, automatically."*

**8. Taints and Tolerations**

A mechanism to control which pods schedule onto which nodes. A **taint** on a node *repels* pods ("don't schedule here unless you tolerate me"). A **toleration** on a pod lets it *tolerate* a matching taint and schedule there anyway. Effects: `NoSchedule`, `PreferNoSchedule`, `NoExecute` (also evicts running pods). Use cases: dedicating nodes to specific workloads (GPU nodes, by reserving them), keeping pods off control-plane/special nodes. **Key distinction from affinity:** taints *repel* pods from nodes; node affinity *attracts* pods to nodes — they're complementary.

**9. Issues you identified in deployment times (experience question)**

Answer from experience with concrete examples: *"We saw slow deployments due to large Docker images (long pull times), missing/poor layer caching causing full rebuilds, no readiness probes so rollouts waited on arbitrary delays, sequential pipeline stages that could run in parallel, slow image scanning, and conservative `maxSurge`/`maxUnavailable` settings."* Pick the ones you've actually dealt with.

**10. Resolutions implemented (experience question)**

Pair each issue with a fix: *"Multi-stage builds and a smaller base image cut image size and pull time; better Dockerfile layer ordering restored caching; we added proper readiness probes so rollouts progressed as soon as pods were truly ready instead of fixed waits; parallelized independent test stages; cached dependencies; and tuned rollout surge settings."* The interviewer wants problem→action→result, ideally with a number ("cut deploy time from 12 to 4 minutes").

**11. Command to describe pod status**

`kubectl describe pod <pod-name>` — shows events, container states, exit codes, probe results, and scheduling info. Also useful: `kubectl get pod <name> -o wide` (status + node/IP), `kubectl get events`, and `kubectl logs`.

**12. Use of Namespace**

Namespaces provide **logical isolation/partitioning** within a cluster — a way to divide cluster resources among teams/projects/environments. They enable **scoping of names** (same resource name can exist in different namespaces), **RBAC boundaries** (permissions per namespace), **resource quotas/limits** per namespace, and **network isolation** (with NetworkPolicies). Used for multi-tenancy, separating dev/test/prod within a cluster, and organizing resources. Note: some resources are cluster-scoped (nodes, PVs) and not namespaced.

## Kubernetes Troubleshooting

**13. CrashLoopBackOff vs ImagePullBackOff**

Different failure stages. **ImagePullBackOff** — Kubernetes **can't pull the image** (wrong image name/tag, image doesn't exist, private registry without credentials, registry unreachable, rate limits). The container never even starts. **CrashLoopBackOff** — the image pulled fine and the container **started but keeps crashing** (app error, OOM, bad config, failing liveness probe). Both use exponential backoff on retries. One-liner: *"ImagePullBackOff = can't get the image; CrashLoopBackOff = got it but the container keeps dying."*

**14. Reasons for CrashLoopBackOff**

(Covered repeatedly.) App error/unhandled exception on startup, missing config/secrets/env vars, **OOMKilled** (exit 137), failing **liveness probe** killing a slow-starting app, wrong command/entrypoint, unreachable dependency at startup, permission issues. Debug with `kubectl logs --previous` + `describe`.

**15. Types of Probes**

Three probes, different purposes:
- **Liveness** — "is the container alive?" If it fails, kubelet **restarts** the container. Catches deadlocks/hangs.
- **Readiness** — "is it ready to serve traffic?" If it fails, the pod is **removed from Service endpoints** (no traffic) but **not restarted**. For startup dependencies or temporary unavailability.
- **Startup** — "has the app finished starting?" Disables liveness/readiness until it succeeds. Protects **slow-starting apps** from being killed by liveness probes prematurely.

Each can be HTTP, TCP, exec, or gRPC. **Key distinction:** liveness failing → restart; readiness failing → stop traffic but keep running.

## Terraform

**16. Major use of Terraform Lint (tflint)**

`tflint` catches errors and enforces best practices that `terraform validate` misses — **provider-specific errors** (invalid instance types, deprecated syntax), **unused declarations**, **naming convention enforcement**, and possible **mistakes before apply**. It's about catching issues early in CI and enforcing standards, complementing `validate` (which only checks syntax/config validity) and `tfsec`/`checkov` (which check security).

**17. Terraform Drift**

When real infrastructure diverges from the Terraform state/config — usually from manual changes in the console. Detect with `terraform plan` (shows the diff) or `terraform plan -refresh-only`. Manage by re-applying to restore desired state, or updating config to match reality. Prevent by enforcing IaC-only changes and restricting console write access.

**18. Terraform Taint**

`terraform taint` **marks a resource for recreation** on the next apply (destroy + recreate) — useful when a resource is degraded/misbehaving and you want to force its replacement. Note: in newer Terraform versions, `terraform taint` is **deprecated** in favor of `terraform apply -replace=<resource>`, which is the current recommended way. Worth mentioning that to show you're current.

**19. for_each in Terraform**

Creates **multiple instances** of a resource/module from a **map or set of strings**. Each instance is keyed by its map key/set value, so resources are addressed by key (e.g., `aws_instance.web["app1"]`). Because instances are keyed (not indexed), adding/removing one doesn't disturb the others.

**20. count in Terraform**

Creates **N instances** of a resource based on an integer. Instances are addressed by **numeric index** (`aws_instance.web[0]`, `[1]`...). Good for identical copies or conditional creation (`count = var.create ? 1 : 0`).

**21. for_each vs count**

The critical difference is **how instances are tracked**:
- `count` uses **numeric indices** — removing an item from the *middle* shifts all subsequent indices, causing Terraform to destroy/recreate resources that just moved position. Best for identical resources or simple conditional toggles.
- `for_each` uses **stable keys** (map keys/set values) — removing one item only affects that one; others stay put. Best when resources are distinct or the list changes over time.

**The interviewer's point:** *"Use `for_each` when items are distinct or the collection changes — its keyed addressing avoids the index-shifting destroy/recreate problem that `count` has. Use `count` for identical copies or simple on/off conditionals."*

## Kubernetes Autoscaling

**23. HPA vs VPA**

(Covered before.) **HPA** scales the *number of pod replicas* based on metrics — scaling **out**. **VPA** adjusts a pod's *CPU/memory requests/limits* — scaling **up**. HPA adds more pods; VPA makes pods bigger. They conflict on the same metric, so usually not combined on CPU. HPA is far more common in production.

## Kubernetes / GKE Services

**24. Types of Services**

- **ClusterIP** (default) — internal-only stable virtual IP for pod-to-pod.
- **NodePort** — exposes the service on a static port on every node's IP.
- **LoadBalancer** — provisions an external cloud load balancer (GCP/AWS LB).
- **ExternalName** — maps the service to an external DNS name (CNAME), no proxying.
- (Plus **Headless** — `clusterIP: None`, returns pod IPs directly for stateful sets/direct discovery.)

**25. NodePort IP/port range**

The default NodePort range is **30000–32767**. The service is reachable on `<any-node-IP>:<nodePort>` within that range. (The range is configurable via the API server's `--service-node-port-range` flag.)

**26. Types of Load Balancers**

In a GCP context: **External vs Internal**, and by layer — **HTTP(S) Load Balancer** (global, L7, content-based routing), **TCP/SSL Proxy** and **Network Load Balancer** (L4). In a general/AWS sense: **ALB** (L7), **NLB** (L4), **GLB/Classic**. Tailor to the cloud they're asking about. Lead with the L7 (application/HTTP) vs L4 (network/TCP) distinction.

**27. Load Balancer vs Ingress — which is best?**

A **LoadBalancer Service** provisions **one external LB per service** — simple but expensive and doesn't scale well when you have many services (one cloud LB each). **Ingress** provides a **single entry point** with **L7 host/path-based routing** to many backend services, **TLS termination**, and needs an **Ingress Controller**. 

"Which is best?" → *"It depends. For a single service or non-HTTP (TCP/UDP) traffic, a LoadBalancer Service is simpler. For multiple HTTP services behind one entry point with path/host routing and shared TLS, **Ingress is better and more cost-effective** — you avoid paying for a separate LB per service."* Showing it's a trade-off, not an absolute, is the right answer.

## GCP Serverless

**28. Cloud Run vs Cloud Functions**

- **Cloud Functions** — event-driven, **single-function** FaaS. You deploy one function triggered by an event (HTTP, Pub/Sub, storage event). Best for small, single-purpose, event-reactive code.
- **Cloud Run** — runs **full containers** (any language/runtime that fits in a container), serving HTTP, scales to zero. More flexible — you control the container, can run a whole app/API, longer requests. 

One-liner: *"Cloud Functions = deploy a single function for an event; Cloud Run = deploy a whole container with full control. Cloud Run is more flexible; Functions is simpler for event glue."* (Note: the lines have blurred as both have grown, but the container-vs-function distinction is the core.)

## GCP Storage

**29. Types of Buckets / storage classes in GCS**

GCS has one bucket type but several **storage classes** based on access frequency: **Standard** (frequent access, hot data), **Nearline** (accessed <once/month, ~30-day min), **Coldline** (accessed <once/quarter, ~90-day min), **Archive** (rarely accessed, ~365-day min, cheapest). Buckets also differ by **location type**: **regional**, **dual-region**, **multi-region** (for availability/latency). Choose class by access pattern (cost vs retrieval cost) and location by availability/latency needs. Lifecycle policies can auto-transition objects to colder classes.

---

## Real DevOps Interview Questions (advanced)

Most of these (1, 4, 5, 6, 7, 9, 10, 11) repeat earlier rounds, so I'll point you back and answer the two genuinely new ones in depth.

**1. OOMKilled internally** — see this round's Capgemini section / earlier: cgroup memory limit exceeded → kernel OOM killer SIGKILLs the process → exit 137 → kubelet restarts. Debug with `describe` (OOMKilled reason) + `top`; prevent by right-sizing limits, fixing leaks, cgroup-aware runtimes.

**2. Pipeline fails in prod with "permission denied"** — almost always an **IAM/credentials/identity** difference between environments. The prod pipeline's service account/role lacks a permission the staging one had, expired prod credentials, a different secret, or stricter prod IAM policies/resource policies. Debug: read the exact error (which resource/action was denied), compare the prod vs staging service account permissions, check credential validity and scope, verify the role's trust/resource policies. Lead with: *"'Permission denied' across environments = the prod identity has different (usually narrower) permissions than staging. I find exactly which action/resource was denied from the error, then diff the two environments' IAM roles."*

**3. Kubernetes DNS across namespaces & what breaks it** — CoreDNS resolves services via FQDN `<service>.<namespace>.svc.cluster.local`. Within a namespace you can use just `<service>`; across namespaces you must qualify with the namespace (`<service>.<namespace>`). What breaks it: **CoreDNS pods down/overloaded**, wrong/missing namespace in the name, NetworkPolicy blocking DNS (port 53) or cross-namespace traffic, misconfigured `ndots`/search domains causing slow or failed resolution, or a Service with no ready endpoints (resolves but nothing answers). Debug: `nslookup` from a pod, check CoreDNS logs/health, verify the service exists in the target namespace.

**4. Stuck Terraform lock** — (covered) verify nothing's actually running, then `terraform force-unlock <LOCK_ID>`; never force-unlock during a genuine apply.

**5. Observability at scale** — (covered) metrics (Prometheus + Thanos/Mimir), logs (Loki/ELK, structured + sampled), traces (OTel + Tempo, tail-sampled), correlated by trace ID; cardinality and cost are the real scale challenges; SLOs on top drive alerting.

**6. Exposed secret response** — (covered) **rotate first** (it's burned the moment it's pushed; bots scrape public GitHub within minutes), assess blast radius via logs, contain, purge history (cleanup not remediation), add push-protection/scanning, blameless postmortem.

**7. HPA internals** — The HPA controller runs a control loop (default every 15s). It **queries the metrics API** (metrics-server for CPU/memory, or custom/external metrics adapters) for current pod metrics, computes `desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))`, and updates the Deployment's replica count. It respects min/max replica bounds and has a **stabilization window** (cooldown) to avoid flapping. It depends entirely on **resource requests** being set (utilization is measured against requests). Verbalize the formula — interviewers love that you know the ratio math.

**8. Error budget & burn rate (NEW — answer in depth)**

This is core SRE and worth nailing. An **SLO** (Service Level Objective) is your reliability target — say 99.9% availability over 30 days. The **error budget** is the *allowed* unreliability: 100% − 99.9% = **0.1%**, which over 30 days is ~43 minutes of downtime you're "allowed" to spend. 

The error budget reframes reliability as a *budget you spend*, not perfection you chase. It enables decisions: if you have budget remaining, you can ship features/take risks; if you've burned it, you **freeze risky changes and focus on reliability**. It aligns dev (wants to ship) and ops (wants stability) around one number.

**Burn rate** is *how fast* you're consuming the error budget. A burn rate of 1 means you'll exactly exhaust the budget by the end of the window; a burn rate of 10 means you're burning 10× too fast and will exhaust it in a tenth of the time. **Multi-window, multi-burn-rate alerting** (e.g., alert if you're burning fast over both a 1-hour and 5-minute window) pages you for *fast* burns (real incidents) while ignoring slow harmless ones — this is the modern fix for alert fatigue.

**Verbalize:** *"The SLO sets the target; the error budget is the allowed failure (1 − SLO); burn rate is how fast you're spending it. I alert on burn rate over multiple windows so I page on fast burns that threaten the budget, not on every blip — and when the budget's exhausted, the policy is to freeze risky releases and invest in reliability."* That "freeze releases when budget is gone" detail is the cultural signal SRE interviewers want.

**9. 60-second latency spikes** — (covered in depth earlier) periodicity is the clue: cron jobs, GC pauses on a cycle, cache TTL expiry causing a stampede, 60s metrics scrape/flush, connection pool recycling. Correlate spike timestamps with everything periodic.

**10. Push vs Pull CD / ArgoCD** — **Push** (traditional): the CI pipeline has cluster credentials and pushes changes *to* the cluster (`kubectl apply`). **Pull** (GitOps/ArgoCD): an in-cluster agent *pulls* desired state from Git and reconciles — the cluster credentials never leave the cluster (more secure), Git is the source of truth, and drift is auto-corrected. ArgoCD continuously diffs Git vs cluster, syncs on change, and with self-heal reverts manual drift. Rollback = `git revert`. Pull/GitOps advantages: better security (no external creds), auditability (Git history), and automatic drift correction.

**11. Dockerfile best practices (security & performance)** — (covered) **Security**: minimal/distroless base, run as non-root (`USER`), no secrets in image, scan in CI (Trivy), drop capabilities, pin versions, sign images. **Performance**: multi-stage builds, small base, order layers by change frequency (deps before source for caching), combine `RUN` + clean caches in same layer, `.dockerignore`.

**12. Platform Engineering & designing an IDP (NEW — answer in depth)**

**Platform Engineering** is the discipline of building an **Internal Developer Platform (IDP)** — a self-service layer that lets developers ship software without needing deep infra/ops expertise for every task. It's the evolution of DevOps: instead of every team reinventing CI/CD, environments, and infra, a platform team productizes these as **paved roads / golden paths** that developers consume self-service. The goal is to **reduce cognitive load** on developers and reduce toil, while keeping consistency, security, and governance baked in.

Designing an IDP — the key components:
- **Self-service provisioning** — developers request environments, databases, pipelines via a portal/CLI/API without filing tickets (e.g., Backstage as the developer portal).
- **Golden paths / templates** — opinionated, paved-road templates for common tasks (new service scaffolding, standard CI/CD) so the easy way is also the secure, compliant way.
- **Abstraction over infra** — developers describe *what* they need (a service, a DB); the platform handles the *how* (Terraform, K8s manifests) underneath. Often via a control plane like **Crossplane** or Kubernetes operators.
- **Standardized CI/CD & GitOps** — consistent pipelines and ArgoCD-style deployment built in.
- **Observability & security baked in** — logging, metrics, tracing, scanning, and policy (OPA/Kyverno) provided by default, not bolted on per team.
- **Guardrails not gates** — policy-as-code enforces standards automatically rather than manual approval bottlenecks.

**Verbalize:** *"Platform engineering productizes infrastructure as a self-service IDP so developers ship via golden paths without deep ops knowledge — reducing their cognitive load. I'd design it around a developer portal (Backstage), self-service provisioning abstracted over Terraform/K8s (Crossplane), standardized GitOps pipelines, and built-in observability and policy-as-code guardrails — so the paved road is the secure, compliant, easy road."* The phrases **"golden paths," "reduce cognitive load," "guardrails not gates,"** and **"self-service"** are exactly the vocabulary that signals you understand the discipline.

---

A couple of patterns across this set:

Capgemini's questions lean **fundamentals + experience** ("issues you identified in deployment times," "conflicts you've seen") — so have **2-3 concrete stories ready** with problem→action→measurable result. The "which is best, LB or Ingress?" type questions are traps for absolutism — always answer *"it depends, here's the trade-off."*

The two new advanced ones — **error budget/burn rate** and **platform engineering/IDP** — are increasingly common at senior level and have distinctive vocabulary. Dropping the right terms ("freeze releases when budget is exhausted," "multi-burn-rate alerting," "golden paths," "reduce cognitive load," "guardrails not gates") signals fluency faster than a long explanation.
