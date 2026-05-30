This is a large batch spanning several more interviews. There's a lot of repetition with earlier rounds, so I'll move fast on those and spend real depth on the genuinely new material — Route 53 routing policies, the Prometheus/Grafana setup specifics, the Git migration steps, the Ansible-100-servers scenario, RDS scaling, SSL-on-S3 (a trick question), Terraform vs Boto3, connection draining, and a few others. I'll group by topic rather than re-answering duplicates five times.

I caught one question (#4 "difference between terraform and CloudFront") that's almost certainly meant to be **Terraform vs CloudFormation** — I'll answer that since CloudFront is a CDN and wouldn't make sense as a Terraform comparison.

## AWS Compute & Serverless

**Lambda — what is the function / how it works / invocation / scaling**

AWS Lambda is **serverless compute** — you upload code (a "function"), and AWS runs it on demand without you managing servers. You pay only for execution time (per ms) and number of invocations.

- **How it works**: an event triggers the function; Lambda provisions an execution environment, runs your handler, and tears it down (or keeps it warm). Stateless by design.
- **Invocation/configuration**: triggered by events — API Gateway (HTTP), S3 (object upload), Pub/Sub equivalents (SNS/SQS), EventBridge (schedule/events), DynamoDB streams, or directly via SDK/CLI. You configure the trigger, memory, timeout, IAM execution role, and env vars in the Lambda console/IaC.
- **Scaling & events**: Lambda **auto-scales by running concurrent execution environments** — each concurrent invocation gets its own environment, scaling horizontally and automatically with load. **Synchronous** (API Gateway — caller waits) vs **asynchronous** (S3/SNS — queued, retried on failure) vs **stream/poll-based** (SQS/Kinesis). Cold starts happen when a new environment spins up; mitigate with Provisioned Concurrency.

## AWS Networking — Route 53 & VPC

**Route 53 routing policies (NEW — answer in full)**

Route 53 offers several routing policies, each for a different goal:
- **Simple** — one record, one or more values (no health-based logic). Basic DNS.
- **Weighted** — split traffic by assigned weights (e.g., 90/10) across resources. Great for canary/gradual rollout or A/B testing.
- **Latency-based** — route users to the region with the lowest latency for them. For global low-latency apps.
- **Failover** — active-passive; route to primary, fail over to secondary on health-check failure. For DR.
- **Geolocation** — route based on the user's geographic location (compliance, localized content).
- **Geoproximity** — route based on geographic distance, with a "bias" to shift traffic between regions.
- **Multivalue answer** — returns multiple healthy IPs (with health checks), a simple form of load distribution.

The framing: *"Weighted for canary/splitting, latency for global performance, failover for DR, geolocation for compliance/localization. Most policies can be tied to health checks."*

**VPC troubleshooting — systematic approach**

Work outside-in along the connectivity path: confirm the **symptom and scope** (which resources, which direction), then check layer by layer — **route tables** (is there a route to the destination/IGW/NAT?), **security groups** (stateful, allow rules — check both directions of the conversation), **NACLs** (stateless — need explicit allow *both* inbound and outbound), **subnet type** (public needs IGW route; private needs NAT for outbound), **DNS resolution**, and **VPC Flow Logs** to see if traffic is being ACCEPTED or REJECTED and where. Tools: VPC Flow Logs, **Reachability Analyzer** (traces the path and tells you exactly what blocks it), and CloudWatch. Lead with: *"Flow Logs tell me if packets are accepted or rejected and at which hop, and Reachability Analyzer traces the path automatically — those two narrow it fast. The classic gotcha is NACLs being stateless, so people forget to allow the return traffic."*

**Subnet issues — insufficient IPs or overlapping CIDR**

- **Insufficient IPs**: the subnet's CIDR is too small (remember AWS reserves 5 IPs per subnet). Fix: add a secondary CIDR / additional subnet, or for new builds size subnets generously. Can't resize an existing subnet — you add capacity alongside.
- **Overlapping CIDR**: two VPCs/subnets with the same range can't be peered or routed between (ambiguous routing). Fix: plan **non-overlapping CIDR ranges** up front (IPAM helps); to connect overlapping VPCs you'd need NAT/PrivateLink as a workaround. Prevention is the real answer — **central IP address planning (AWS IPAM)**.

**Managing multiple VPCs across teams/environments**

Centralize and standardize: use a **hub-and-spoke with Transit Gateway** to interconnect VPCs at scale (instead of a mesh of peering connections, which is non-transitive and unmanageable beyond a few). Use **AWS Organizations + multiple accounts** (account-per-environment/team for blast-radius isolation), **shared services VPC** for common resources, **IPAM** for non-overlapping CIDR planning, centralized logging/monitoring, and standardized VPC creation via a **reusable Terraform module**. Optimize cost by sharing NAT/endpoints where appropriate and using VPC endpoints to avoid NAT data charges.

## AWS Storage, Database, CDN

**RDS — what it is & scaling under heavy load (NEW depth)**

RDS is AWS's **managed relational database** service (handles provisioning, patching, backups, failover for MySQL/Postgres/etc.). Under heavy load, your scaling options:
- **Vertical scaling** — bump to a larger instance class (more CPU/RAM). Quick but has a ceiling and brief downtime (or use a Multi-AZ failover to minimize it).
- **Read replicas** — offload **read** traffic to replicas (read-heavy workloads benefit most). Horizontal read scaling.
- **Multi-AZ** — for availability/failover, not load (a standby, not active).
- **Caching** — put **ElastiCache** (Redis/Memcached) in front to absorb repeated reads — often the biggest win.
- **Connection pooling** — **RDS Proxy** to handle connection storms (a very common cause of "degradation under load").
- **Aurora** — if you can migrate, Aurora scales reads better (up to 15 replicas) and Aurora Serverless auto-scales capacity.
- **Optimize first** — slow queries/missing indexes via Performance Insights often beat throwing hardware at it.

Framing: *"Read-heavy → read replicas + ElastiCache; connection storms → RDS Proxy; write-bound → vertical scale or shard/Aurora. But I'd check Performance Insights first — often it's a missing index, not a capacity problem."*

**Attach an SSL certificate to an S3 bucket (TRICK QUESTION)**

You **cannot attach an SSL certificate directly to an S3 bucket** — S3 static website hosting only serves HTTP. To get HTTPS on S3 content, you put **CloudFront in front of the bucket** and attach the **ACM SSL certificate to the CloudFront distribution** (CloudFront serves the content over HTTPS, S3 stays private behind an Origin Access Control). The correct answer is recognizing the premise is flawed: *"You don't attach a cert to S3 directly — S3 website endpoints are HTTP-only. You front it with CloudFront and put the ACM certificate on the distribution."* Spotting that earns points.

**Read-only S3 bucket policy — can you modify objects?**

**No.** A read-only policy grants `s3:GetObject` (and maybe `ListBucket`) but not `s3:PutObject`/`DeleteObject`, so you can't modify or upload. (Caveat: effective permissions are the union of IAM policy + bucket policy minus any explicit denies — but if the access is genuinely read-only across the board, modification is blocked.)

**CDN — what & how**

A **Content Delivery Network** caches content at **edge locations** geographically close to users, so requests are served from a nearby edge instead of the origin — reducing latency and offloading the origin. CloudFront caches static (and configurable dynamic) content, terminates TLS at the edge, and improves performance + availability + DDoS resilience. Flow: user request → nearest edge → cache hit serves immediately, cache miss fetches from origin and caches it.

## Terraform

**Terraform vs CloudFormation** (the likely intended question)

Both are IaC. **Terraform** is cloud-agnostic (multi-cloud + many providers), uses HCL, has a large module ecosystem, and manages state explicitly. **CloudFormation** is AWS-native (deep AWS integration, no separate state to manage — AWS tracks it, native rollback on failure) but AWS-only, using JSON/YAML. Choose Terraform for multi-cloud/portability and a richer ecosystem; CloudFormation for pure-AWS shops wanting native integration and managed state.

**Terraform plan & purpose**

`terraform plan` produces a **preview of changes** — it compares desired config against current state and shows what will be created, changed, or destroyed *without making any changes*. Purpose: review and validate before applying, catch unintended destroys, and (in CI) post the plan to a PR for human review before `apply`.

**Why `terraform import`**

To bring **existing infrastructure** (created manually or outside Terraform) **under Terraform management** — it writes the existing resource into state so Terraform stops trying to create it and starts managing it. Used when adopting Terraform for pre-existing resources or recovering after state loss. (Newer Terraform supports declarative `import` blocks.)

**Terraform over Boto3 — why (NEW)**

Terraform is **declarative** (you describe desired state; Terraform figures out the changes and tracks them in state), **idempotent**, handles **dependency ordering** automatically, gives a **plan/preview**, and manages the full lifecycle including deletion of removed resources. Boto3 is **imperative** Python (you script each API call, handle ordering, idempotency, and state tracking yourself) — more work and error-prone for infra management. Boto3 shines for **operational scripting/automation** (one-off tasks, Lambda automation, data operations), not declarative infra provisioning. Framing: *"Terraform for declarative, stateful, idempotent infra provisioning with a plan step; Boto3 for imperative operational scripting. Using Boto3 to provision infra means reinventing state tracking and idempotency."*

## CloudWatch / Monitoring / Grafana

**High CPU on EC2 — use CloudWatch to troubleshoot**

Start with the **CloudWatch metrics** for the instance: confirm the `CPUUtilization` spike and its pattern (sustained vs spiky, when it started). Correlate with other metrics — network, disk I/O, and (with the **CloudWatch agent**) memory and per-process data, since basic EC2 metrics don't include memory or process-level detail. Check **CloudWatch Logs** (app logs) around the spike, and look for a correlated **deploy/change** (CloudTrail). Set up an alarm for recurrence. If it's a runaway process, the agent's process metrics or logging in to check `top` pinpoints it. Mention: *"Default EC2 metrics don't cover memory or per-process CPU — I'd install the CloudWatch agent for that depth."*

**Set up Prometheus to monitor a custom app (NEW depth)**

The standard flow:
1. **Instrument the app** — add a Prometheus client library (e.g., `prometheus_client` for Python) and expose a **`/metrics`** endpoint emitting your custom metrics (counters, gauges, histograms) in Prometheus format.
2. **Configure Prometheus** to **scrape** that endpoint — add a job to `prometheus.yml` with the target's address and scrape interval (or use service discovery / a ServiceMonitor if on Kubernetes with the Prometheus Operator).
3. **Verify** the target is UP in Prometheus and query the metrics in PromQL.
4. **Visualize** in Grafana and **alert** via Alertmanager on the metrics that matter (SLO-aligned).

Framing: *"Instrument with a client library exposing `/metrics`, add a scrape job (or ServiceMonitor on K8s), then build Grafana dashboards and Alertmanager rules on the custom metrics."*

**Grafana alert when EC2 CPU > 90% (NEW)**

Configure it as: a **data source** (CloudWatch or Prometheus) providing the EC2 CPU metric → create a **Grafana alert rule** on that query with the **condition** `CPUUtilization > 90` → set the **evaluation interval and "for" duration** (e.g., sustained for 5 minutes, to avoid alerting on brief spikes) → attach a **contact point** (email/Slack/PagerDuty) and notification policy. Key detail to mention: *"I'd set a 'for' duration so it only fires on sustained high CPU, not transient spikes — otherwise it's alert noise."*

**Monitoring/logging experience (#7)** — Answer honestly from your experience with specifics: the stack you used (Prometheus/Grafana, ELK/CloudWatch, Datadog), what you instrumented, dashboards/alerts you built, and a concrete win ("set up SLO-based alerting that cut noise by X"). Tailor to your real background.

## CI/CD & Pipelines

**Complete CI/CD on AWS — tools & why (NEW)**

A coherent AWS-native answer: **CodePipeline** (orchestration) + **CodeBuild** (build/test) + **CodeDeploy** (deployment, supports blue-green/rolling) + **CodeCommit or GitHub** (source) + **ECR** (image registry) + deploy target (**ECS/EKS/Lambda/EC2**). Add **CloudWatch** for monitoring and **IAM** for least-privilege. Why: fully managed, integrated, no servers to maintain, native IAM/CloudWatch integration. *Or* the common real-world hybrid: **GitHub + Jenkins/GitHub Actions → ECR → EKS via ArgoCD**. Justify your choice by team familiarity, multi-cloud needs, and existing tooling — there's no single right answer, so explain the trade-off.

**Ensure CI/CD quality & reliability + metrics (#17)**

Practices: automated testing at every stage (unit→integration→e2e), quality/security gates that fail fast, rollback mechanisms, deploy safely (canary/blue-green), automate everything, monitor proactively. **Metrics — the DORA four**: **Deployment Frequency**, **Lead Time for Changes**, **Change Failure Rate**, **MTTR**. Also pipeline success rate, build duration, and test coverage. Naming the **DORA metrics** explicitly is the senior signal here.

**CI/CD YAML Dev→UAT from scratch (#11)** — Describe a pipeline (GitHub Actions/GitLab CI) with stages: checkout → build → test → build+push image → deploy to Dev → run smoke/integration tests → manual approval gate → deploy to UAT. Use environment-specific variables/secrets and an approval gate before UAT. Narrate the *why* (fail fast, gate before promotion).

**Jenkins/pipeline fails in prod but works in Dev** — (covered repeatedly) Environment difference: usually **permissions/credentials** (prod IAM role narrower, expired creds), missing prod config/secrets, different prod endpoints/dependencies, or stricter prod resource policies. Read the exact error, diff the two environments' config and permissions.

**Jenkins pipeline for your project (#14, experience)** — Answer yes with your real stages (checkout, build, test, scan, dockerize, push, deploy) and any specifics (shared libraries, multibranch, credentials store).

## Git

**What is Git / why / branching strategy**

Git is a **distributed version control system** — tracks code changes, enables collaboration, history, and branching. Why: history/rollback, parallel work via branches, collaboration, and it underpins CI/CD. **Branching strategies**: **GitFlow** (main/develop/feature/release/hotfix — structured, heavier), **GitHub Flow** (main + short-lived feature branches, deploy from main — simple, CD-friendly), **trunk-based** (everyone commits to main frequently behind feature flags — best for high-velocity CD). Mention which you've used and why.

**Migrate Git repo with full history (GitHub → GitLab) (NEW)**

The clean way uses a **mirror clone** to preserve all branches, tags, and history:
1. `git clone --mirror <source-repo-url>` — bare mirror clone with all refs/history.
2. Create the empty destination repo in GitLab.
3. `cd` into the mirror, then `git push --mirror <destination-repo-url>` — pushes all branches, tags, and full commit history.
4. Verify branches/tags/history landed, update remotes for the team, and migrate any extras separately (issues, PRs, CI config, LFS objects, webhooks aren't carried by git push).

Framing: *"`git clone --mirror` then `git push --mirror` preserves complete history, all branches, and tags. The git data migrates cleanly; issues, MRs, and pipelines need separate migration since they live outside git."*

## Kubernetes

**CrashLoopBackOff troubleshoot** — (covered) `kubectl logs --previous` + `describe`; map exit code (137=OOM), check config/secrets, probes, entrypoint, dependencies.

**Deployed successfully but can't access externally (NEW angle)** — Work the exposure path: is there a **Service**, and does its **selector match the pod labels**? `kubectl get endpoints <svc>` — **empty endpoints = selector matches no ready pods** (very common). Is the Service type right (ClusterIP is internal-only; need NodePort/LoadBalancer/Ingress for external)? Is the **`targetPort`** correct? For Ingress, is the **controller running** and rules correct? Are **security groups/firewall/NSG** allowing the traffic? Are pods passing **readiness probes** (failing readiness = removed from endpoints)? Lead with: *"First `kubectl get endpoints` — empty endpoints means the Service selector doesn't match ready pods, which is the most common cause. Then check Service type, targetPort, and the LB/Ingress + firewall."*

**When a node fails — what happens** — The node's status goes `NotReady` (kubelet stops heartbeating). After the **eviction timeout** (~5 min default), the node controller marks pods for eviction and the controllers (ReplicaSet/Deployment) **reschedule replacement pods onto healthy nodes** to restore desired replica count. StatefulSet/PV-bound pods need their volumes to be reattachable (multi-attach constraints apply). DaemonSet pods just disappear with the node. PDBs are respected for voluntary disruptions but a hard node failure is **involuntary** — pods are lost and rescheduled.

**Pod config from a production perspective (#9)** — The production checklist: **resource requests & limits** set, **readiness + liveness + startup probes**, **multiple replicas** + **PodDisruptionBudget**, **anti-affinity** to spread across nodes/AZs, **non-root securityContext** + read-only filesystem, **secrets from a vault** not env, **graceful shutdown** (preStop + terminationGracePeriod), image pinned by digest/tag (not `latest`), and appropriate **HPA**. This is a great checklist to rattle off — it signals real production experience.

**HPA vs VPA**, **CNI**, **DaemonSet**, **Namespaces**, **recent EKS problem (#16, experience)** — covered earlier; for the EKS one, have a real story ready (e.g., a node-group upgrade issue, IAM/IRSA misconfiguration, CNI IP exhaustion on the VPC CNI, or a CoreDNS scaling problem — the **VPC CNI IP exhaustion** one is very common and credible on EKS).

## Other Tools

**Maven & repositories (NEW detail)** — Maven is a **Java build & dependency management tool** driven by a `pom.xml`. **Repositories**: **local** (`~/.m2`, cached on your machine), **central** (Maven's public default repo), and **remote/private** (e.g., **Nexus/Artifactory** — your org's hosted repo for internal artifacts and proxying central). Maven resolves dependencies from local first, then remote/central.

**Deploy app on 100 servers with Ansible (NEW)** — Define the 100 servers in an **inventory** (grouped, ideally dynamic inventory if cloud), write a **playbook** with the deployment tasks (idempotent modules), and run it — Ansible is **agentless over SSH** and runs across all hosts in parallel (tune with `forks` and `serial`). Crucially, use **`serial`** for a **rolling/batched deployment** (e.g., `serial: 10` → 10 servers at a time) so you don't take all 100 down at once, combine with health checks, and use `--limit` for targeting. Framing: *"Inventory of 100 hosts, idempotent playbook, run with `serial` to roll out in batches so it's a controlled rolling deployment with health checks between batches — not all 100 at once."* The **`serial`/rolling** detail is what makes this a senior answer.

**Docker containers stop suddenly after starting (NEW)** — `docker ps -a` to see exit code, `docker logs <container>` for output. Exit **137** = OOM/killed; non-zero = app error; **0** = the main process simply finished (a container exits when PID 1 exits — common when there's no long-running foreground process). Check: is the entrypoint a long-running foreground process (not daemonized)? OOM (memory limit)? Crash/misconfig? Missing dependency/env var? Lead with: *"`docker logs` + exit code. A very common cause is the main process exiting/daemonizing — the container lives only as long as PID 1 runs. 137 means OOM."*

**ArgoCD & GitOps** — (covered) ArgoCD = GitOps continuous delivery for K8s; continuously reconciles cluster state to match Git (source of truth), auto-syncs and self-heals drift, rollback via git revert. GitOps = Git as the single source of truth with automated pull-based reconciliation; benefits are auditability, consistency, security (no external cluster creds), easy rollback.

## AWS — IAM, Load Balancing, Auto Scaling, AMIs, Encryption

**IAM roles & policies** — **Policies** define permissions (allow/deny on actions/resources). **Roles** are assumable identities with policies attached, used by services/users/cross-account principals to get **temporary** credentials (no long-lived keys). Attach policies to **users** directly or via **groups** (preferred — manage permissions by group, add users to groups), and to **roles** for services/cross-account. Best practice: **least privilege**, prefer roles + groups over per-user inline policies.

**Encryption in your project (experience)** — Cover **at rest** (S3 SSE-KMS, EBS/RDS encryption with KMS keys) and **in transit** (TLS/HTTPS, ACM certs). Mention KMS for key management. Tailor to what you actually used.

**Connection draining (NEW)** — When an instance is being deregistered or is unhealthy, connection draining (called **Deregistration Delay** on ALB/NLB target groups) lets **in-flight requests complete** before the LB stops sending traffic and removes the instance — rather than abruptly cutting active connections. You set a timeout (e.g., 300s). It's what makes scale-in and deployments graceful. Framing: *"It lets existing connections finish before the instance is removed, so users mid-request aren't dropped during scale-in or deployment."*

**How ELB distributes traffic** — The LB distributes incoming requests across **healthy registered targets** (failing health checks → removed from rotation) across AZs. ALB routes at L7 (round-robin / least-outstanding-requests, content-based rules) and NLB at L4 (flow hash). It only sends to targets passing **health checks**, and supports cross-zone load balancing to spread evenly across AZs.

**Auto Scaling** — An **Auto Scaling Group** maintains a desired count of instances across AZs, **replacing unhealthy ones** automatically, and scales in/out via policies (**target tracking** e.g. keep CPU at 50%, **step**, or **scheduled**). Combined with an ELB it gives elastic, self-healing capacity.

**Deployment strategies in your project (experience)** — State which you use (rolling/blue-green/canary) and why, tied to your real setup.

**Custom AMIs — tools (experience)** — **Packer** (HashiCorp) is the standard answer for building customized, versioned AMIs as code; also EC2 Image Builder. Mention Packer + a base image + provisioners (Ansible/shell) if you've used it.

**Application behind LB suddenly unavailable — troubleshoot** — Work the path: **health checks** (are targets healthy? failing health checks pull them from rotation — #1 cause), **target/instance health** (app crashed, OOM, disk full), **security groups** (LB→instance and client→LB), **target group registration**, **LB listener config**, and recent **deploys/changes**. Lead with health checks: *"If targets fail health checks the LB removes them and serves nothing — so I check target group health first, then the app itself, then security groups and recent changes."*

**AWS billing spike — identify costs** — (covered) **Cost Explorer** grouped by service → usage type → region to isolate what jumped; **CloudTrail** for what changed and who; rule out **compromised credentials** for sudden overnight spikes (security incident); check runaway autoscaling, data egress, forgotten resources.

**Developer wants an EC2 for local deployment (NEW)** — Provision a right-sized instance for dev work: a **general-purpose burstable type (t3/t4g)** is the sensible default — cheap, burstable, fine for dev/test. Give them access via a dedicated IAM role/key with least privilege, a security group scoped to their needs, in a non-prod account/VPC, and tag it for cost tracking. Mention: *"I'd give a t3 (general-purpose burstable) since it's dev workload — cost-effective and sufficient — in a non-prod account with least-privilege access and proper tagging, ideally with auto-stop off-hours to save cost."*

---

A few patterns worth flagging across this batch:

Several are **trick/premise-check questions** — "attach SSL cert to S3" (you can't, use CloudFront), "modify objects with read-only policy" (no), and the "terraform vs CloudFront" typo. Catching the flawed premise scores more than a confident wrong answer.

The **experience questions** ("how much monitoring experience," "recent EKS problem," "encryption in your project," "Jenkins pipeline for your project") want **specific stories from your real work** — prepare 3-4 concrete examples with problem→action→result and numbers where you can.

And the genuinely-new scenario answers have **distinctive "right details"** that signal seniority: Ansible **`serial`** for batched rollout, Grafana **"for" duration** to avoid noise, **`git clone/push --mirror`** for full history, **DORA metrics** for CI/CD quality, **RDS Proxy** for connection storms, and **empty endpoints** as the first check for "can't access the app."
