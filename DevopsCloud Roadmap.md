# 30-Day DevOps / Cloud Engineering Interview Roadmap
### For 8–15 Years Experience | Azure, Kubernetes, Docker, Terraform, Ansible, CI/CD, Observability, ArgoCD

---

## How to Use This Roadmap

At your experience level, interviewers aren't testing whether you know commands — they're testing **judgment**: design trade-offs, failure handling, cost/scale decisions, and how you've solved real production problems. For every topic below, prepare:
1. A **concept explanation** (as if teaching a mid-level engineer)
2. A **war story** from your own experience (what broke, what you decided, why)
3. **2–3 likely interview questions** and how you'd answer them out loud

Spend mornings on hands-on labs, evenings on theory + STAR-format stories. Keep a running doc of "war stories" — you'll reuse them across System Design, Behavioral, and Technical rounds.

---

## Week 1: Docker, Kubernetes Core, and Azure Fundamentals

### Day 1 — Docker Deep Dive
- Image layers, build cache, multi-stage builds, distroless/minimal images
- Docker networking modes (bridge, host, overlay), volumes vs bind mounts
- Container security: rootless containers, capabilities, seccomp, image scanning (Trivy/Grype)
- **Practice:** Write a multi-stage Dockerfile reducing image size by 70%+
- **Be ready to discuss:** "How did you reduce image size/build time in production?"

### Day 2 — Kubernetes Architecture
- Control plane deep dive: API server, etcd, scheduler, controller manager
- kubelet, kube-proxy, CNI plugins (Calico, Azure CNI, Cilium)
- Pod lifecycle, init containers, sidecars, probes (liveness/readiness/startup)
- **Practice:** Explain etcd backup/restore and what happens if etcd quorum is lost
- **Be ready to discuss:** A cluster outage you diagnosed and root-caused

### Day 3 — Kubernetes Workloads & Scheduling
- Deployments, StatefulSets, DaemonSets, Jobs/CronJobs — when to use each
- Resource requests/limits, QoS classes, OOMKilled troubleshooting
- Node affinity/anti-affinity, taints/tolerations, topology spread constraints
- HPA, VPA, Cluster Autoscaler, KEDA (event-driven scaling)
- **Practice:** Design a scheduling strategy for a multi-tenant cluster

### Day 4 — Kubernetes Networking & Security
- Services (ClusterIP, NodePort, LoadBalancer), Ingress vs Gateway API
- Network Policies, mTLS, service mesh basics (Istio/Linkerd) — trade-offs
- RBAC design, Pod Security Standards/Admission, Secrets management (Azure Key Vault CSI driver)
- **Practice:** Design RBAC for 3 teams sharing one cluster with namespace isolation

### Day 5 — Azure Core Services for DevOps
- AKS architecture: node pools, managed identity, AKS networking (kubenet vs Azure CNI vs Azure CNI Overlay)
- Azure Virtual Networks, NSGs, Private Link, Azure Firewall
- Azure Storage (Blob, Files, Disks), Azure AD / Entra ID, Managed Identities vs Service Principals
- **Practice:** Design a secure AKS landing zone (hub-spoke topology)

### Day 6 — Azure Cost, Governance, and Well-Architected Framework
- Azure Policy, Management Groups, subscription design, Cost Management/Budgets
- Reserved Instances, Spot node pools in AKS, right-sizing strategies
- **Be ready to discuss:** A cost optimization initiative you led and the $ impact

### Day 7 — Review & Mock Round 1
- Whiteboard: Design a production-grade AKS cluster from scratch (networking, security, scaling, DR)
- Review all war stories from Days 1–6

---

## Week 2: Infrastructure as Code (Terraform, Ansible) + CI/CD

### Day 8 — Terraform Fundamentals to Advanced
- State management: remote backends (Azure Storage + state locking), workspaces
- Module design, versioning, and reusability patterns
- `terraform plan/apply` internals, drift detection, import
- **Practice:** Design a Terraform module structure for multi-environment (dev/stage/prod) AKS

### Day 9 — Terraform at Scale
- State file splitting strategies, remote state data sources
- Terragrunt or native workspaces — trade-offs
- Handling secrets in Terraform (Azure Key Vault integration, never in state)
- CI/CD for Terraform: plan on PR, apply on merge, approval gates
- **Be ready to discuss:** A time state got corrupted/locked and how you recovered

### Day 10 — Ansible Deep Dive
- Idempotency, inventory management (dynamic inventory for Azure), roles vs playbooks
- Ansible Vault, handlers, templates (Jinja2)
- Ansible vs Terraform — when to use which (config management vs provisioning)
- **Practice:** Write a role to configure/patch a fleet of VMs with drift detection

### Day 11 — Jenkins CI/CD
- Declarative vs scripted pipelines, shared libraries, Jenkinsfile best practices
- Agent architecture (static vs dynamic/K8s agents), plugin management overhead
- Pipeline security: credentials binding, approved script sandboxing
- **Be ready to discuss:** Migrating Jenkins to K8s-based dynamic agents — why and how

### Day 12 — GitHub Actions
- Workflow design, reusable workflows, composite actions, matrix builds
- Self-hosted runners on AKS, OIDC federation with Azure (no static secrets)
- GitHub Actions vs Jenkins — cost, maintainability, scaling trade-offs
- **Practice:** Design a pipeline: build → test → scan → push to ACR → deploy via ArgoCD

### Day 13 — CI/CD Strategy & GitOps Principles
- Branching strategies (trunk-based vs GitFlow) for CI/CD velocity
- Blue-green, canary, rolling deployments — how each is implemented in K8s
- Artifact management (ACR, versioning/tagging strategy, image promotion across environments)

### Day 14 — Review & Mock Round 2
- Whiteboard: Design end-to-end CI/CD from commit to production with approvals, rollback, and security scanning gates
- Review IaC and CI/CD war stories

---

## Week 3: GitOps (ArgoCD) + Observability + Reliability

### Day 15 — ArgoCD Fundamentals
- App of Apps pattern, ArgoCD Projects, RBAC, sync policies (auto vs manual, prune, self-heal)
- Sync waves and hooks for ordered deployments
- ArgoCD vs Flux — trade-offs
- **Practice:** Design a multi-cluster, multi-environment ArgoCD setup

### Day 16 — GitOps at Scale
- Repo structure strategies (mono-repo vs poly-repo for manifests), Kustomize vs Helm with ArgoCD
- Secrets in GitOps (Sealed Secrets, External Secrets Operator + Azure Key Vault)
- Progressive delivery: Argo Rollouts for canary/blue-green with automated analysis
- **Be ready to discuss:** How you handled a bad deployment — detection and rollback time

### Day 17 — Prometheus Deep Dive
- Architecture: scraping, PromQL, service discovery, federation
- Alerting: Alertmanager routing, silencing, grouping, avoiding alert fatigue
- Long-term storage options (Thanos/Mimir/Cortex) — why needed at scale
- **Practice:** Write PromQL for error rate, p99 latency, and saturation (RED/USE methods)

### Day 18 — Grafana & Dashboards
- Dashboard-as-code (JSON models, Grafana provisioning via Terraform/GitOps)
- Data source federation (Prometheus, Loki, Azure Monitor)
- Designing SLO/SLI dashboards, burn-rate alerts
- **Be ready to discuss:** A dashboard/alert that caught a real incident before customers did

### Day 19 — ELK / EFK Stack
- Elasticsearch cluster design (shards, replicas, index lifecycle management)
- Logstash/Fluentd/Fluent Bit pipelines, parsing/enrichment
- Kibana for log correlation during incidents; log retention & cost trade-offs
- **Practice:** Design a centralized logging pipeline for a multi-cluster AKS environment

### Day 20 — SRE Practices: SLOs, Incident Management, Chaos
- Defining SLIs/SLOs/error budgets, and how they drive engineering priorities
- Incident response process, postmortems (blameless), runbooks
- Chaos engineering basics — have you deliberately broken things to test resilience?
- **Be ready to discuss:** Your best postmortem — root cause, contributing factors, action items

### Day 21 — Review & Mock Round 3
- Whiteboard: Design observability stack for a 50-microservice platform (metrics, logs, traces, alerting)
- Review GitOps/observability war stories

---

## Week 4: System Design, Behavioral, Leadership, and Final Polish

### Day 22 — Cloud Architecture / System Design Practice 1
- Design a highly available, multi-region AKS platform on Azure (DR strategy, RPO/RTO)
- Cover: traffic management (Front Door/Traffic Manager), data replication, failover automation

### Day 23 — System Design Practice 2
- Design a secure CI/CD platform for 50+ teams (multi-tenancy, self-service, guardrails)
- Cover: golden pipelines, policy-as-code (OPA/Gatekeeper/Kyverno), platform engineering mindset

### Day 24 — Security & Compliance (DevSecOps)
- Shift-left security: SAST/DAST, container image scanning, SBOM
- Secrets management end-to-end, supply chain security (SLSA, image signing with Cosign)
- Compliance frameworks relevant to your industry (SOC2/ISO/HIPAA) — how CI/CD enforces them

### Day 25 — Networking & Troubleshooting Scenarios
- Practice live troubleshooting: pod can't reach service, DNS failure in K8s, node NotReady, certificate expiry
- Azure networking troubleshooting: NSG blocking traffic, private endpoint DNS resolution issues
- **Practice:** Talk through your troubleshooting methodology out loud (this is what's being evaluated)

### Day 26 — Behavioral / Leadership Prep (STAR Method)
- Prepare 6–8 stories covering: conflict resolution, mentoring, driving a migration, handling a major outage, disagreeing with leadership, cost savings, cross-team influence
- At 8–15 YOE, expect: "Tell me about a time you influenced architecture decisions without direct authority"

### Day 27 — Cost, FinOps & Trade-off Discussions
- Be ready to discuss trade-offs explicitly: cost vs reliability, speed vs governance, build vs buy
- Practice articulating **why**, not just **what** — interviewers probe reasoning at senior levels

### Day 28 — Mock Interview (Full Loop)
- Simulate: 1 technical deep-dive round, 1 system design round, 1 behavioral round back-to-back
- Get feedback (peer, mentor, or record yourself) on clarity and structure of answers

### Day 29 — Gap-Filling & Weak Area Review
- Revisit whichever domain felt shakiest in the mock (commonly: Terraform state at scale, ArgoCD sync strategies, or PromQL)
- Skim official docs changelogs for Kubernetes, Terraform, ArgoCD — know current stable versions and recent notable features

### Day 30 — Final Review & Rest
- Review your war-story doc end-to-end
- Prepare 3–5 thoughtful questions to ask the interviewer (team structure, on-call model, platform maturity, tech debt priorities)
- Light review only — no new topics. Rest well.

---

## Quick-Reference: High-Frequency Senior-Level Questions

| Area | Sample Question |
|---|---|
| Kubernetes | "Walk me through what happens when a node goes NotReady in production." |
| Terraform | "How do you manage Terraform state across 50+ microservices and multiple environments?" |
| ArgoCD | "How do you handle secrets in a GitOps workflow without committing them to Git?" |
| CI/CD | "How would you design a pipeline that can safely roll back within 2 minutes of a bad deploy?" |
| Observability | "How do you decide what to alert on vs what to just log?" |
| Azure | "How do you design network isolation for a multi-tenant AKS cluster?" |
| Behavioral | "Tell me about a production incident you caused. What did you learn?" |
| Architecture | "How would you migrate a monolith's CI/CD to a GitOps model with zero downtime?" |

---

## Suggested Daily Time Split
- **2 hrs** hands-on labs (build it, break it, fix it)
- **1 hr** theory/documentation review
- **30 min** writing/refining war stories in STAR format
- **30 min** mock Q&A (say answers out loud, don't just think them)

Good luck — at your experience level, the strongest signal you can give is **calm, structured reasoning under ambiguity**, not memorized facts.
