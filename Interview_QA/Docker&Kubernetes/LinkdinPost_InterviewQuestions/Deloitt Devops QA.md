# Deloitte AWS DevOps Interview — Complete Answers

Going through each with the depth a senior DevOps engineer should bring.

---

## 1. VPC architecture

> "I'll walk through a production-grade VPC design — three-tier, multi-AZ, with the components and design decisions I'd defend in an interview."

**Foundation:**

- **VPC CIDR**: `10.0.0.0/16` — large enough (65,536 IPs) for growth. Pick a CIDR that doesn't overlap with corporate networks, other AWS accounts, or potential peers.
- **Three Availability Zones** for production HA.

**Subnet tiers (per AZ):**

```
VPC: 10.0.0.0/16

Public subnets (small — only LBs and NAT GWs):
  10.0.0.0/24    us-east-1a   (256 IPs)
  10.0.1.0/24    us-east-1b
  10.0.2.0/24    us-east-1c

Private app subnets (large — EKS nodes, app servers):
  10.0.16.0/20   us-east-1a   (4,096 IPs)
  10.0.32.0/20   us-east-1b
  10.0.48.0/20   us-east-1c

Private data subnets (medium — RDS, ElastiCache):
  10.0.64.0/22   us-east-1a   (1,024 IPs)
  10.0.68.0/22   us-east-1b
  10.0.72.0/22   us-east-1c

Reserved for future growth: 10.0.96.0/19 onwards
```

**Connectivity components:**

- **Internet Gateway** — attached to the VPC, gives public subnets internet access.
- **NAT Gateways** — one per AZ (no SPOF), in public subnets, give private subnets outbound internet egress. Each AZ's private subnets route through that AZ's NAT.
- **Route Tables** — one per subnet tier:
  - Public RT: `0.0.0.0/0 → IGW`
  - Private app RT (per AZ): `0.0.0.0/0 → NAT-GW-in-same-AZ`
  - Private data RT: no internet route (or restricted egress via NAT)
- **VPC Endpoints**:
  - **Gateway endpoints** (free): S3, DynamoDB. Adds entries to route tables.
  - **Interface endpoints** (PrivateLink): ECR, Secrets Manager, KMS, CloudWatch, SSM. Keeps AWS API traffic off NAT and saves cost.

**Security layers:**

- **Security Groups** — stateful, instance-level. Reference each other rather than CIDRs (`app-sg` allows ingress from `alb-sg`).
- **NACLs** — stateless, subnet-level. Default NACL is permissive; custom NACLs for defense in depth on regulated tiers.
- **VPC Flow Logs** — to CloudWatch or S3, fed into a SIEM for traffic auditing.
- **GuardDuty** — threat detection on VPC traffic.

**Multi-VPC patterns (when one VPC isn't enough):**

- **Transit Gateway** — hub-and-spoke for connecting many VPCs and on-prem.
- **VPC Peering** — point-to-point, no transitive routing. Cheaper for small numbers of VPCs.
- **PrivateLink** — expose a service privately to other VPCs without full peering.

**For multi-region DR:** separate VPCs per region, connected via Transit Gateway peering or VPN. Routes carefully filtered to avoid CIDR overlap.

**Architect's framing:**
> "The decisions that matter most aren't about how to draw the diagram — they're CIDR sizing for the long term, subnet-tier separation for security, per-AZ NAT for resilience, and VPC endpoints to keep AWS API traffic private and cheap. The biggest mistakes I see are too-small VPC CIDRs that can't grow, public-by-default subnets, and skipping Flow Logs because 'we don't need them now.'"

---

## 2. NACL vs Security Group

| | Security Group | NACL |
|---|---|---|
| Operates at | ENI (instance) level | Subnet level |
| State | Stateful | Stateless |
| Rule types | Allow only | Allow AND deny |
| Default behavior | Default deny; you add allow rules | Default NACL allows all; custom NACL denies all |
| Rule evaluation | All rules considered together (union of allows) | Numbered, evaluated in order, first match wins |
| Scope | Per-resource (attach to ENIs of instances, LBs, RDS, etc.) | Applies to all resources in associated subnets |
| Quotas | 60 inbound + 60 outbound rules per SG | 20 rules per NACL direction |

**Statefulness implications:**

SG: if I allow inbound on 443, the response is automatically allowed back out. One rule.

NACL: if I allow inbound on 443, I must *also* explicitly allow outbound on ephemeral ports (typically 1024-65535) for the response. Two rules, and forgetting the second one is the classic NACL trap.

**Practical example — three-tier app:**

```
ALB SG:
  Ingress: 443 from 0.0.0.0/0
  Egress:  to app-sg on 8080

App SG:
  Ingress: 8080 from alb-sg
  Egress:  to db-sg on 5432, to 0.0.0.0/0 on 443 (outbound APIs)

DB SG:
  Ingress: 5432 from app-sg
  Egress:  none
```

This is referenced by SG ID, not CIDR — self-documenting, resilient to IP changes.

**When to use NACLs (alongside SGs, not instead):**

- Block a specific malicious IP range at the subnet boundary.
- Defense in depth — if an SG is accidentally permissive, NACL still enforces.
- Compliance requirements mandating subnet-level controls.
- Deny-rules that SGs can't express.

**Practical advice:** for most workloads, SGs are sufficient. Most teams leave default NACLs as-is and rely on SGs for fine-grained control. NACLs come in when you specifically need IP-based denies or compliance demands subnet-layer enforcement.

---

## 3. CI/CD in my organization and role

Tailor to your real experience, but here's the structure:

> "On the platform I currently support, CI/CD follows a GitOps model. Here's the end-to-end flow:
>
> **CI side (per-service pipeline):** Developer pushes to GitHub → GitHub Actions triggers → checkout, Maven build with unit tests + JaCoCo coverage, SonarQube scan with quality-gate block, Snyk dependency scan, Trivy image scan, Cosign signing, push to ECR with an immutable tag like `1.4.2-a1b2c3d-build2087`.
>
> **CD side:** The same pipeline then commits the new image tag to a GitOps repo (separate from the application repo) — updating `environments/dev/values.yaml`. ArgoCD watches that repo, detects the change, and syncs the EKS cluster via Helm.
>
> **Promotion:** Dev is automatic on merge to `develop`. Staging requires the dev smoke test to pass. Production is a PR in the GitOps repo with two-person approval; once merged, ArgoCD reconciles. For tier-1 services, prod uses Argo Rollouts canary — 10% → 25% → 50% → 100% with SLO-based analysis at each step that auto-aborts if error rate or p99 latency regresses.
>
> **My role on this:**
> - I designed the GitHub Actions reusable workflows and the Jenkins shared library that came before them.
> - I own the Helm chart library that all services consume.
> - I drove the migration from push-based CD (Jenkins → kubectl) to GitOps with ArgoCD.
> - I maintain the OIDC federation setup so no long-lived AWS credentials live in GitHub secrets.
> - I'm the escalation point when pipelines break in ways teams can't diagnose themselves."

**Key principles I'd articulate:**
- **Build once, promote artifact** — never rebuild for prod.
- **Immutable tags** — production references SHA-suffixed tags, not floating ones.
- **OIDC federation** — no static cloud credentials.
- **GitOps for deploy** — pipeline doesn't have prod cluster credentials; only ArgoCD does.
- **Security at every stage** — SAST, dep scan, image scan, image signing, admission verification.

---

## 4. Cardinal steps to create an EKS cluster

> "Going through it in the order I actually execute, not just the components:"

**Phase 1: Pre-requisites and planning**

1. **Account and networking baseline** — VPC with three private subnets across AZs (cluster subnets), public subnets for LBs, NAT Gateways per AZ. Tag subnets correctly (`kubernetes.io/role/elb=1` for public, `kubernetes.io/role/internal-elb=1` for private — required for AWS Load Balancer Controller).
2. **IAM roles** — cluster role with `AmazonEKSClusterPolicy`, node role with `AmazonEKSWorkerNodePolicy + AmazonEC2ContainerRegistryReadOnly + AmazonEKS_CNI_Policy`.
3. **KMS key** for envelope encryption of Kubernetes secrets in etcd.

**Phase 2: Cluster provisioning (Terraform)**

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "prod-eks"
  cluster_version = "1.30"

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.private_subnets

  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true  # restrict source CIDRs in prod

  cluster_encryption_config = {
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }

  enable_irsa = true   # OIDC provider for IAM Roles for Service Accounts

  cluster_addons = {
    coredns                = { most_recent = true }
    kube-proxy             = { most_recent = true }
    vpc-cni                = { most_recent = true }
    aws-ebs-csi-driver     = { most_recent = true }
    eks-pod-identity-agent = { most_recent = true }
  }

  eks_managed_node_groups = {
    system = {
      desired_size = 2
      min_size     = 2
      max_size     = 4
      instance_types = ["t3.medium"]
      labels = { role = "system" }
      taints = [{ key = "system", value = "true", effect = "NO_SCHEDULE" }]
    }
    workload = {
      desired_size = 3
      min_size     = 3
      max_size     = 20
      instance_types = ["m6i.large"]
      capacity_type  = "ON_DEMAND"
    }
  }
}
```

4. **Apply Terraform** — creates control plane, node groups, OIDC issuer, encryption config.

**Phase 3: Post-cluster setup**

5. **Update kubeconfig**: `aws eks update-kubeconfig --name prod-eks --region us-east-1`.
6. **Map IAM roles/users to Kubernetes RBAC** via the `aws-auth` ConfigMap (or modern EKS Access Entries — Microsoft's new preferred approach since 2023).
7. **Install cluster-critical add-ons:**
   - **AWS Load Balancer Controller** — for ALB/NLB ingress provisioning.
   - **Karpenter** or **Cluster Autoscaler** — for node-level autoscaling.
   - **External-DNS** — auto-creates Route 53 records from Ingress/Service annotations.
   - **cert-manager** — automated TLS via Let's Encrypt or ACM Private CA.
   - **NGINX Ingress** (or AGIC/Traefik) — Ingress controller.
   - **External Secrets Operator** — syncs secrets from AWS Secrets Manager.
   - **Metrics Server** — for HPA.

**Phase 4: Observability and security**

8. **Install kube-prometheus-stack** (Prometheus, Alertmanager, Grafana) via Helm.
9. **Install Loki + Promtail** or **Fluent Bit → CloudWatch** for logs.
10. **Install Kyverno or OPA Gatekeeper** for admission policies (image registry allowlist, required labels, no privileged pods).
11. **Enable EKS control-plane logging** to CloudWatch.
12. **NetworkPolicy enforcement** — install Calico or enable Cilium. Default-deny baseline per namespace.
13. **IRSA setup** — for each app namespace, create the IAM role and service account binding.

**Phase 5: GitOps platform**

14. **Install ArgoCD** via Helm in `argocd` namespace.
15. **Configure ArgoCD with OIDC** to corporate identity provider.
16. **Bootstrap App-of-Apps pattern** — one root ArgoCD Application that manages all other Applications declaratively.

**Phase 6: Validation**

17. **Smoke test** — deploy a hello-world app via ArgoCD, verify ingress, TLS, autoscaling, metrics, logs all work end-to-end.
18. **Run cluster conformance test** (`sonobuoy`) to validate the cluster behaves per the Kubernetes spec.
19. **Run a chaos experiment** — kill a node, verify recovery.
20. **DR drill** — simulate region failure if applicable.

**Phase 7: Day-2 operations**

21. **Document runbooks** — node replacement, cluster upgrade, common incidents.
22. **Enable cluster upgrade plan** — minor version upgrades approximately quarterly.
23. **Cost monitoring** — Kubecost or OpenCost installed.

**Architect's framing:**
> "I'd never click through the console for any of this in production. The whole cluster — VPC, IAM, EKS, node groups, add-ons, ArgoCD — is in Terraform and Helm, version-controlled, peer-reviewed, applied via pipeline. The reason matters: clusters get rebuilt, replaced, replicated to new regions. If the only thing in Git is the application manifests but the cluster setup is click-ops, you can't rebuild deterministically. Day-2 starts the moment the cluster is up — observability, policy, GitOps are not afterthoughts."

---

## 5. Pods in Pending — troubleshooting

Systematic triage:

**Step 1: Get the events — they almost always tell you the answer.**

```bash
kubectl describe pod <pod>
# Events section at the bottom is where the answer is
```

**Step 2: Categorize the failure mode**

**A. Insufficient resources on any node**
```
Warning  FailedScheduling  0/8 nodes are available: 8 Insufficient cpu, 6 Insufficient memory
```

Cause: requested CPU/memory exceeds what any node has free.

Fix:
- Lower the pod's resource requests.
- Scale up the node pool (more nodes or larger instance type).
- If using Cluster Autoscaler / Karpenter, check why it isn't scaling — look at controller logs.
- Check `kubectl top nodes` to confirm utilization.

**B. Pod requests don't match any node selector / taint**
```
Warning  FailedScheduling  0/8 nodes are available: 8 node(s) didn't match Pod's node affinity/selector
```

Cause: `nodeSelector`, `nodeAffinity`, or tolerations don't match available nodes.

Fix:
- Verify node labels: `kubectl get nodes --show-labels`
- Check taints: `kubectl get nodes -o json | jq '.items[].spec.taints'`
- Add missing tolerations, or fix the affinity/selector.

**C. PVC not bound**
```
Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims
```

Cause: pod references a PVC that can't be bound — no matching PV, no StorageClass, or topology constraints.

Fix:
```bash
kubectl get pvc <pvc>
kubectl describe pvc <pvc>
kubectl get storageclass
# Check provisioner is running (e.g., EBS CSI driver)
kubectl get pods -n kube-system | grep csi
```

**D. IP exhaustion (EKS VPC CNI)**
```
Warning  FailedCreatePodSandBox  ... InsufficientFreeAddressesInSubnet
```

Cause: subnet ran out of IPs. Common at scale with AWS VPC CNI.

Fix:
- Enable VPC CNI prefix delegation (gives 16 IPs per ENI prefix).
- Add a secondary CIDR (`100.64.0.0/16`) for pod IPs.
- Move workloads to a node pool in less-exhausted subnets.

**E. Image pull issues** (Pod might briefly show Pending before flipping to ImagePullBackOff — see next answer).

**F. Admission webhook rejection**
```
Error  FailedCreate  admission webhook "policy.kyverno.io" denied the request
```

Cause: Kyverno/Gatekeeper/OPA policy rejecting the pod (missing labels, disallowed image registry, missing resource limits).

Fix: review the policy violation, update the manifest.

**G. PodDisruptionBudget blocking**
Less common for pending; more for terminating-stuck pods.

**H. ResourceQuota exceeded**
```
Error  exceeded quota: tenant-quota, requested: pods=1, used: pods=200, limited: pods=200
```

Fix: free quota by deleting unneeded pods or raising the quota.

**I. PriorityClass mismatch / preemption pending**
Pod is waiting for higher-priority workloads to free resources.

**Step 3: Diagnostic commands beyond describe**

```bash
# Why is the scheduler not placing this pod?
kubectl get events --sort-by='.lastTimestamp' -n <ns>
kubectl get events --field-selector involvedObject.name=<pod>

# What's the node state?
kubectl get nodes
kubectl describe node <node>
kubectl top nodes

# Capacity vs requests across the cluster
kubectl describe nodes | grep -A5 "Allocated resources"

# If autoscaler should be scaling
kubectl logs -n kube-system deployment/cluster-autoscaler
kubectl logs -n karpenter deployment/karpenter

# Any pending PVC issues
kubectl get pvc -A | grep -v Bound

# Controller manager / scheduler issues (managed EKS shows in CW Logs)
aws logs tail /aws/eks/prod-eks/cluster --filter-pattern scheduler
```

**Step 4: Fix and verify**

After making changes, watch the pod transition:
```bash
kubectl get pod <pod> -w
```

**Architect's framing:**
> "Pending is almost always a scheduling problem. `kubectl describe pod` tells you which scheduler predicate failed — insufficient resources, no matching node, PVC unbound, admission rejection — and from there the fix is direct. The systematic mistake is jumping to logs or `kubectl exec` first; the pod doesn't exist yet, there's nothing to log. Events are the source of truth."

---

## 6. ImagePullBackOff

**ImagePullBackOff** is a Kubernetes pod status indicating the kubelet failed to pull the container image and is in exponential backoff before retrying.

The flow:
1. Pod scheduled to a node.
2. Kubelet tries to pull the image.
3. Pull fails.
4. Status briefly shows `ErrImagePull`.
5. Kubelet waits (backoff: 10s, 20s, 40s, ... up to 5 min).
6. While waiting: status shows `ImagePullBackOff`.
7. Retry, fail again → longer backoff.

**Common root causes ranked by likelihood:**

**1. Wrong image name or tag**
```bash
kubectl describe pod <pod>
# Events show: "manifest for myacr.azurecr.io/myapp:1.4.2 not found"
```

Fix: verify the image exists in the registry: `aws ecr describe-images --repository-name myapp`.

**2. Missing or invalid imagePullSecrets**

For private registries other than the cluster's native one, you need an imagePullSecret:
```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

For EKS + ECR, no imagePullSecret is needed if the cluster's node IAM role has `AmazonEC2ContainerRegistryReadOnly` (or IRSA-based pull). Check:
```bash
kubectl get serviceaccount default -n <ns> -o yaml
aws eks describe-cluster --name <cluster> --query 'cluster.identity.oidc.issuer'
```

**3. Registry authentication expired**
For DockerHub/private registries, the auth token in the imagePullSecret may have expired.

**4. Network can't reach the registry**
- Pod's network can't egress to internet (no NAT, no VPC endpoint for ECR).
- DNS resolution failure for the registry host.
- For ECR private endpoints: missing `com.amazonaws.region.ecr.api` and `com.amazonaws.region.ecr.dkr` interface endpoints.

Diagnose from the node:
```bash
ssh <node>
# Try the pull directly:
docker pull <image>   # or
crictl pull <image>
# Check connectivity
nslookup <registry-host>
curl -v https://<registry-host>
```

**5. Rate limit exceeded (DockerHub specifically)**
DockerHub limits anonymous pulls. Fix: authenticate, use a private mirror (ECR), or use a pull-through cache.

**6. Image is for a different architecture**
ARM image on AMD nodes, or vice versa. Fix: build multi-arch images or correct the platform.

**7. ECR registry policy or KMS issue**
If the image is encrypted with a KMS CMK and the node's role doesn't have decrypt permission on the key.

**Diagnosing commands:**

```bash
# What does the pod say?
kubectl describe pod <pod>
# Look at Events for the exact pull error

# What image is being pulled?
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'

# Try pulling manually from a node
crictl pull <image>

# Check imagePullSecrets
kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets}'
kubectl get secret <pull-secret> -o json | jq -r '.data[".dockerconfigjson"] | @base64d'

# Check node IAM (EKS)
aws iam get-role --role-name <node-role>
```

**Fix it forward:** once the underlying issue is resolved, the kubelet will retry on the backoff schedule (no need to delete the pod, though deleting it does trigger an immediate fresh attempt).

```bash
# Force immediate retry
kubectl delete pod <pod>     # Deployment recreates it
```

---

## 7. Distroless images

**Yes.** Distroless images are container images that contain only the application and its runtime dependencies — **no shell, no package manager, no utilities** like `ls`, `cat`, `bash`. They're built and maintained by Google: `gcr.io/distroless/*`.

**What's in (and not in) a distroless image:**

| Image | Size | Contents |
|---|---|---|
| `ubuntu:22.04` | ~77 MB | Full Ubuntu userspace |
| `alpine:3.20` | ~5 MB | Alpine + busybox + apk |
| `gcr.io/distroless/static-debian12` | ~2 MB | Just CA certs, /etc/passwd, timezone data |
| `gcr.io/distroless/base-debian12` | ~20 MB | + glibc, libssl, libcrypto |
| `gcr.io/distroless/java17-debian12` | ~180 MB | + JRE 17 |
| `gcr.io/distroless/python3-debian12` | ~50 MB | + Python 3 runtime |
| `gcr.io/distroless/nodejs20-debian12` | ~140 MB | + Node.js 20 |

**Why use them:**

**1. Massively reduced attack surface**
No shell means no shell exploits. No `apt`, `apk` means no supply-chain attacks via package installation. An attacker who achieves code execution in a distroless container can't easily pivot — they can't even run `ls`.

**2. Smaller image size**
Fewer layers, less to transfer, faster cold starts.

**3. Fewer CVEs to scan**
Trivy/Snyk reports against distroless are typically a fraction of CVEs vs equivalent Alpine/Debian.

**4. Forces good practices**
Can't shell into a running container — encourages proper logging, observability, debug endpoints rather than `kubectl exec` to poke around.

**Trade-offs / when not to use them:**

**1. Debugging is harder**
No `kubectl exec -it pod -- sh` because there's no shell.

Workarounds:
- **Debug containers** (Kubernetes 1.23+): `kubectl debug -it <pod> --image=busybox --target=<container>` mounts a debug image alongside.
- **Distroless `:debug` tags**: `gcr.io/distroless/static-debian12:debug` includes BusyBox for emergency debugging.
- **Better observability** so you don't *need* to exec.

**2. Application must be self-contained**
Distroless can't do `apt-get install` at runtime. Everything the app needs must be in the image at build time. Multi-stage Dockerfiles solve this.

**3. Statically compiled binaries preferred**
For Go/Rust, distroless `static` is perfect — single statically-linked binary, no libc needed. For Python/Java/Node, use the language-specific distroless images.

**4. Init systems missing**
No `tini` or similar by default — `:debug` tag includes a few utilities. For graceful shutdown, ensure the app handles SIGTERM directly.

**Example multi-stage Dockerfile with distroless:**

```dockerfile
# Build stage
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app

# Distroless runtime
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

That Go binary in distroless static is ~20 MB total. The same Go app on `golang:1.22` would be ~900 MB.

**Architect's framing:**
> "Distroless is the right default for production containers — smaller, fewer CVEs, harder to compromise. The cost is operational: debugging requires `kubectl debug` rather than `kubectl exec`, which is friction the first time you use it. For Go and Rust I use distroless `static`. For Java, distroless `java17` is excellent. For Python/Node, the distroless variants work but I sometimes use `alpine` for the package-availability convenience and accept the slightly larger size."

---

## 8. Current project architecture

Tailor this to your actual experience, but here's a representative answer:

> "Current platform is a microservices-based fintech application running on AWS EKS:
>
> **Edge layer:**
> - CloudFront for global edge caching of static assets and TLS termination.
> - AWS WAF for OWASP rule sets, bot mitigation, geo-blocking.
> - Route 53 with health-check-based failover between us-east-1 and us-west-2.
>
> **Application layer:**
> - EKS clusters in us-east-1 (primary, active) and us-west-2 (warm standby).
> - 35 microservices: Java Spring Boot, Node.js, Python — all stateless behind ClusterIP services.
> - NGINX Ingress Controller as the in-cluster L7 router, fronted by an internal NLB.
> - Istio service mesh for mTLS and traffic policies between services.
> - Karpenter for elastic node provisioning across mixed on-demand and Spot capacity.
>
> **Data layer:**
> - RDS Aurora PostgreSQL Global Database for transactional data — sub-second replication to us-west-2.
> - DynamoDB Global Tables for high-write event log.
> - ElastiCache Redis cluster mode for sessions and cache.
> - MSK (managed Kafka) for event streaming.
> - S3 with cross-region replication for object storage.
>
> **Identity and secrets:**
> - IRSA for pod-to-AWS authentication — no static credentials anywhere.
> - AWS Secrets Manager with rotation, accessed via External Secrets Operator in cluster.
> - KMS multi-region keys for envelope encryption.
>
> **Observability:**
> - Prometheus + Grafana for metrics, Loki for logs, Tempo for distributed tracing.
> - All instrumented via OpenTelemetry auto-instrumentation in the standard service base images.
> - Alertmanager → PagerDuty + Slack.
> - Custom SLO dashboards per service.
>
> **CI/CD:**
> - GitHub Actions for build/test/scan/push to ECR.
> - GitOps via ArgoCD watching a separate manifests repo.
> - Argo Rollouts for canary deploys on tier-1 services with auto-abort on SLO regression.
>
> **Governance:**
> - Multi-account setup via AWS Organizations + Control Tower.
> - Separate accounts for shared services, prod, non-prod, security.
> - SCPs at OU level for guardrails (e.g., no public S3 buckets, specific regions only).
>
> **Operations:**
> - RTO 15 minutes, RPO < 1 minute.
> - Quarterly DR game days that actually fail over a region.
> - Blameless postmortems for any Sev-1 or Sev-2.
> - Capacity reviews monthly; cost reviews monthly.
>
> Volume: ~5,000 transactions per second peak, ~2 million daily active users."

---

## 9. Secret management tools

> "Beyond AWS Secrets Manager, I've used several:
>
> **HashiCorp Vault** — most versatile. Dynamic secrets (Vault generates short-lived DB credentials on demand), encryption-as-a-service, identity-based access via OIDC/JWT. Used in multi-cloud setups where AWS Secrets Manager doesn't fit. Operationally heavier — you own the cluster.
>
> **Azure Key Vault** — Azure equivalent of Secrets Manager. Used on AKS workloads, integrated via CSI Secrets Store driver + Azure Workload Identity. No long-lived credentials.
>
> **Google Secret Manager** — GCP equivalent. Used on GKE with Workload Identity.
>
> **CyberArk Conjur** — enterprise secret management, common in regulated environments (banking). RBAC-heavy, audit-heavy.
>
> **Sealed Secrets** — Bitnami's solution for GitOps. Encrypt secrets with a public key, commit the encrypted blob to Git, the in-cluster controller decrypts. Trade-off: secrets are tied to a specific cluster's key.
>
> **SOPS + age/PGP** — encrypt YAML/JSON values in files; commit encrypted to Git, decrypt at apply time. Lightweight, integrates with Terraform and Helm. I've used this for non-production secrets.
>
> **External Secrets Operator (ESO)** — not a secret store itself, but the *Kubernetes-native interface* to backends like Secrets Manager, Vault, Key Vault. Sync external secrets into native K8s Secrets without manual reconciliation.
>
> **AWS Systems Manager Parameter Store** — used for non-sensitive config and some lightweight secrets (with SecureString). Cheaper than Secrets Manager for high-volume reads.
>
> The default choice depends on the cloud and operational tolerance. For AWS-native EKS workloads, Secrets Manager + ESO covers most needs. For multi-cloud or when dynamic secrets matter, HashiCorp Vault."

---

## 10. AWS Secrets Manager vs Parameter Store

| | Secrets Manager | Parameter Store (SSM) |
|---|---|---|
| Purpose | Sensitive credentials | Configuration + lightweight secrets |
| Encryption | Always (KMS) | Optional (SecureString uses KMS) |
| Automatic rotation | Built-in (with Lambda) | No native rotation |
| Cross-region replication | Built-in (multi-region secrets) | Manual |
| Versioning | Yes, with explicit version stages | Limited (parameter history) |
| Notification on change | EventBridge events | EventBridge events |
| Resource-based policies | Yes | Yes (advanced parameters only) |
| Pricing | $0.40 per secret per month + $0.05 per 10K API calls | Standard: free. Advanced: $0.05 per parameter per month |
| Parameter size | Up to 64 KB | Standard: 4 KB. Advanced: 8 KB |
| Throughput | Higher | Standard: lower; Advanced: same as SM |
| Integration | RDS auto-rotation, ECS, Lambda, etc. | EC2 launch templates, CloudFormation, ECS task definitions |
| Hierarchical naming | Limited | Native (`/app/prod/db/password`) |
| Cross-account sharing | Resource-based policies + KMS grants | Resource-based policies (advanced) |

**When to choose Secrets Manager:**

- **Database credentials with rotation** — Secrets Manager has native rotation Lambdas for RDS, Redshift, DocumentDB. The auth flow is "fetch secret, connect; if connection fails, fetch again (rotation may have happened)."
- **High-security secrets** that must rotate regularly.
- **Cross-region replicated secrets** for DR.
- **API keys, OAuth tokens** with rotation requirements.
- **Cost is justified** for the criticality.

**When to choose Parameter Store:**

- **Configuration values** that aren't truly secret (feature flags, endpoint URLs, log levels).
- **Hierarchical config trees** — `/app/{env}/{component}/...` reads as a logical namespace.
- **Cost-sensitive** workloads with many parameters and high read volume.
- **EC2 / ECS / CloudFormation native integration** scenarios.
- **Build artifact metadata** — AMI IDs, container image tags often live here for cross-pipeline reference.

**My typical pattern:**

- **Secrets Manager** for: DB passwords, third-party API keys, JWT signing keys, anything that rotates.
- **Parameter Store** for: feature flags, environment URLs, AMI IDs, build versions, application config that's environment-specific but not strictly secret.

**Combined access pattern in code:**

```python
import boto3
import json

# Secrets Manager — sensitive, rotated
sm = boto3.client('secretsmanager')
db_creds = json.loads(sm.get_secret_value(SecretId='prod/db')['SecretString'])

# Parameter Store — config
ssm = boto3.client('ssm')
api_endpoint = ssm.get_parameter(Name='/app/prod/api-endpoint')['Parameter']['Value']
feature_flag = ssm.get_parameter(Name='/app/prod/feature-new-checkout')['Parameter']['Value']
```

In Kubernetes both can be synced via External Secrets Operator's `awssm` and `awsssm` providers.

**Architect's framing:**
> "Parameter Store is the right answer 70% of the time — most things we call 'config' are properly Parameter Store territory and Secrets Manager is overkill at $0.40 per secret. Secrets Manager earns its cost when rotation is a real requirement, or when you're integrating with native AWS rotation flows like RDS. I see teams default to Secrets Manager for everything and run up bills — distinguishing 'config' from 'secret with rotation needs' is the discipline."

---

## 11. Challenges faced in current or previous projects

Tailor to your real experience. Several genuine examples that show seniority:

> "A few that stand out across projects:
>
> **1. EKS IP exhaustion at scale**
> We started hitting `InsufficientFreeAddressesInSubnet` as the workload grew. The original subnet design (/24 per AZ) supported maybe 8 nodes per AZ with AWS VPC CNI's per-pod-IP allocation. We migrated to **prefix delegation** — 16 IPs per ENI prefix — which lifted us from ~30 pods per node to ~110, and added a secondary CIDR (100.64.0.0/16) for pod IPs decoupled from VPC subnet sizing. The lesson: subnet capacity planning is a Day-0 conversation, not a Day-100 incident.
>
> **2. Stateful workload migration from EC2 to Kubernetes**
> We had a self-managed Cassandra cluster on EC2 that the business wanted on K8s for consistency. Migrating it to a StatefulSet was straightforward; what wasn't straightforward was understanding the operational implications — backup story changed, monitoring tooling changed, the team's debugging muscle was EC2-focused. We ended up using the Cass-Operator (K8ssandra) rather than DIY StatefulSets, which encoded the operational knowledge. The honest lesson: for stateful workloads on K8s, an Operator is usually worth more than the Kubernetes-native primitives.
>
> **3. Multi-account Terraform state architecture**
> The original setup had all environments sharing one Terraform state file. We had a near-miss where a misconfigured `terraform apply` in dev nearly destroyed prod resources because the state file mixed them. Migrating to per-environment-per-account state files with strict IAM boundaries took two sprints but eliminated a class of accident that could have been catastrophic.
>
> **4. CI/CD secret leak prevention**
> An engineer accidentally committed an AWS access key to a public repo. AWS's automated scanning caught it within minutes; the key was disabled. We then drove a project across the org to eliminate static AWS credentials in GitHub — migrated everything to OIDC federation. No long-lived credentials in any pipeline now. The incident became the forcing function.
>
> **5. Cost spike from misconfigured CloudWatch Logs**
> A team enabled debug-level logging on a high-traffic service before a weekend deploy. CloudWatch Logs ingest costs spiked to $40k over three days before anyone noticed. We added budget alarms at 50/80/100% of monthly forecast, CloudWatch Logs ingest as a top-line dashboard metric, and added log-level enforcement in the standard base image so 'debug in prod' requires explicit opt-in.
>
> **6. Disaster recovery game day exposed broken assumptions**
> Our 'DR plan' said we could fail over to us-west-2 in 15 minutes. The first real game day showed actual RTO was 47 minutes because of: a Lambda function with us-east-1 hardcoded in the source, an Elasticsearch cluster without cross-region snapshots, and an IAM role missing in us-west-2. Quarterly game days now; documented RTO matches measured RTO."

The pattern in good answers: real incidents, what we learned, what we systemically changed.

---

## 12. App on EC2 behind ELB not accessible — troubleshooting

Systematic path, top to bottom of the stack:

**Step 1: Confirm the symptom**
```bash
# Where exactly does it fail?
curl -v https://my-app.example.com/
# Look at response: timeout, refused, 5xx, 4xx, TLS handshake failure
# Different failures point at different layers
```

**Step 2: DNS layer**
```bash
# Does DNS resolve?
nslookup my-app.example.com
dig my-app.example.com

# Resolves to what? Verify it's the ELB's DNS
# Check Route 53 record points at ELB DNS name
aws route53 list-resource-record-sets --hosted-zone-id <id>
```

**Step 3: ELB layer**

```bash
# Is the ELB itself reachable?
curl -v http://<elb-dns-name>/

# Target health
aws elbv2 describe-target-health --target-group-arn <arn>
# Output shows each target's health status:
#   - healthy
#   - unhealthy (with reason: Target.FailedHealthChecks, Target.Timeout)
#   - draining
#   - initial

# ELB security group
aws ec2 describe-security-groups --group-ids <sg-id>
# Inbound: should allow 443 (and/or 80) from 0.0.0.0/0
# Outbound: should allow to instance SG on the target port

# Health check config
aws elbv2 describe-target-groups --target-group-arns <arn>
# Verify: path, port, expected codes, interval, threshold
```

**Common ELB issues:**
- Targets unhealthy because the health check path returns non-2xx.
- Health check port mismatch (LB checks 8080 but app listens on 8081).
- ELB SG doesn't allow traffic to instance SG.
- HTTPS listener has no SSL cert or wrong cert.

**Step 4: EC2 / Security Group / NACL layer**

```bash
# Is the instance running?
aws ec2 describe-instances --instance-ids <id>
# State: running? StatusChecks: 2/2 OK?

# Instance SG — does it allow inbound from ELB's SG?
aws ec2 describe-security-groups --group-ids <instance-sg>
# Inbound: should have rule allowing ELB SG on the app port

# NACL — does the subnet's NACL allow the traffic?
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>
# Default NACL allows all; custom NACL needs explicit rules

# Route table — does the subnet route correctly?
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>
```

**Step 5: Application layer (on the instance)**

```bash
# SSH to the instance (or use SSM Session Manager — preferred)
aws ssm start-session --target <instance-id>

# Is the app process running?
ps aux | grep <app>
systemctl status <app-service>

# Is it listening on the expected port?
ss -tlnp | grep :8080
netstat -tlnp | grep :8080

# Local response works?
curl -v http://localhost:8080/health

# App logs
journalctl -u <app-service> -n 100
tail -f /var/log/<app>/app.log
```

**Step 6: Cross-cutting checks**

```bash
# CloudWatch metrics for the ELB
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name UnHealthyHostCount \
  --dimensions Name=LoadBalancer,Value=<arn-suffix> \
  --statistics Average --period 60 \
  --start-time $(date -u -d '1 hour ago' +%FT%T) \
  --end-time $(date -u +%FT%T)

# ELB access logs (if enabled)
aws s3 ls s3://<elb-logs-bucket>/AWSLogs/...
```

**Step 7: AWS-side issues**
- Check AWS Health Dashboard for region-level issues.
- Check service quotas (rare but possible — hit a limit).

**Triage order I follow:**

1. **Where does the failure happen?** (DNS → LB → instance → app) — the first failure point eliminates everything downstream.
2. **What does the LB target health say?** This alone often gives the answer.
3. **Can the instance reach itself on the app port (localhost)?** If yes, it's network; if no, it's app.
4. **What's between the LB and the instance?** SGs, NACLs, subnets, routes.

**Architect's framing:**
> "I work top-down through the stack: DNS, LB, network, instance, app. The single most diagnostic command in this situation is `describe-target-health` — it tells you 'health check failing' or 'target healthy but traffic not flowing,' and those go down very different troubleshooting paths. I also use VPC Reachability Analyzer in AWS for cross-AZ/subnet routing issues — it surfaces SG/NACL/route problems in seconds."

---

## 13. Route 53 routing policies

Route 53 supports several routing policies. Going through each with use cases:

**1. Simple**
Basic single-record routing. One value or set of values; client gets them in random order. No health checks.
- **Use case:** static records, single endpoints, no failover needed.

**2. Weighted**
Multiple records for the same name with assigned weights. Traffic distributed proportionally.
- **Use case:** canary deployments at the DNS layer (5% to new version), A/B testing, gradual migration to a new region.
- Example: 95 weight on `prod-v1.example.com`, 5 weight on `prod-v2.example.com` — 5% of users hit v2.

**3. Latency-based**
Routes to the AWS region with the lowest measured network latency from the user's perspective.
- **Use case:** multi-region deployments where you want users on the nearest performant region. AWS continuously measures latency; routing adjusts automatically.

**4. Geolocation**
Routes based on the user's geographic location — continent, country, or US state.
- **Use case:** data residency requirements (EU users hit EU region for GDPR), content localization, geo-blocking.
- Always include a default record for unmatched locations or users get nothing.

**5. Geoproximity**
Routes based on geographic *distance* between user and resource, with adjustable bias to expand or shrink a region's footprint.
- **Use case:** more nuanced than geolocation; lets you bias traffic toward specific regions even if technically not closest.
- Requires Route 53 Traffic Flow (more advanced configuration).

**6. Failover**
Active-passive setup: primary record + secondary record. Route 53 monitors primary via health check; on failure, traffic shifts to secondary.
- **Use case:** disaster recovery — primary region down, traffic moves to standby.

```hcl
resource "aws_route53_record" "primary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "primary"
  failover_routing_policy { type = "PRIMARY" }
  health_check_id = aws_route53_health_check.primary.id
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "secondary"
  failover_routing_policy { type = "SECONDARY" }
  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

**7. Multivalue answer**
Returns up to 8 healthy records in response. Poor man's load balancing at DNS layer.
- **Use case:** simple LB without a real load balancer; cheap multi-target high availability.
- Limitation: no traffic-shaping or weighting.

**8. IP-based routing** (relatively newer)
Routes based on the user's source IP address mapped to defined CIDR locations.
- **Use case:** routing specific corporate networks or partner integrations to specific endpoints.

**Combining policies:**

You can combine in hierarchies via **alias chains** or **Traffic Flow**:
- Geolocation → Failover (route EU → eu-failover policy → eu-primary / eu-dr).
- Latency-based → Weighted (route to fastest region, then weighted between two versions).

**Health checks across all of these:**

Most policies support health checks that mark records as unhealthy and skip them. Set up health checks against the actual target (ALB, EC2, custom endpoint), and Route 53 reroutes within seconds of failure detection.

**Architect's framing:**
> "The choice depends on what you're optimizing for: latency (latency-based), data residency (geolocation), HA (failover), gradual rollout (weighted), or balance (multivalue). For multi-region production, the typical pattern is latency-based with failover — get users to the fastest region by default, fail over the entire region if a health check fails. Add weighted for canary releases on top."

---

## 14. Deployment strategies

I've used several:

**1. Recreate**
Kill all old pods, then start new ones. Brief downtime.
- **Use case:** dev environments, singletons, anything with low traffic tolerance.

**2. Rolling Update** (Kubernetes default)
Gradually replace old pods with new, controlled by `maxSurge` and `maxUnavailable`.
- **Use case:** default for stateless services with backwards-compatible API/schema changes.

**3. Blue-Green**
Two identical environments; instant traffic cutover by flipping the service selector or LB target.
- **Use case:** major version upgrades, coordinated DB+app migrations, tier-1 systems needing near-instant rollback.

**4. Canary**
Route a small slice of traffic to the new version, observe, gradually increase.
- **Use case:** risk-sensitive changes, performance-sensitive changes, tier-1 customer-facing services.

**5. A/B Testing**
Similar to canary but split based on user attributes (cookies, headers, geography) rather than random traffic %.
- **Use case:** product experimentation, feature validation with specific cohorts.

**6. Shadow / Mirror**
New version receives a *copy* of production traffic but its responses are discarded. Useful for validating performance and correctness under real load.
- **Use case:** load testing, validation of new database queries, machine learning model evaluation.

**7. Feature Flags + Continuous Deployment**
Deploy code disabled; toggle on for specific users via flags. Velocity decoupled from deploy risk.
- **Use case:** SaaS products with continuous deployment culture.

**In my experience:**

> "For everyday microservice deploys, **rolling update** is the default — cheap, simple, no downtime when readiness probes are configured. For tier-1 production deploys (payments, auth, checkout), I use **canary via Argo Rollouts** with automated SLO-based analysis — error rate or p99 latency regression auto-aborts the rollout. **Blue-green** I reserve for coordinated DB+app cuts or major version upgrades where instant rollback matters more than the 2× infrastructure cost during cutover. **Shadow** I've used twice — once validating a database query rewrite, once validating a recommendation model — useful in narrow cases."

---

## 15. Canary vs Blue-Green

The core difference: **how traffic is shifted and what intermediate states exist**.

| | Blue-Green | Canary |
|---|---|---|
| Traffic during transition | 100% blue, then instant flip to 100% green | Progressive shift: 5% → 25% → 50% → 100% |
| Both versions live simultaneously | Yes, briefly | Yes, throughout the rollout |
| Infrastructure cost during deploy | 2× (both environments at full scale) | 1× plus the canary slice (minimal) |
| Rollback speed | Near-instant (flip selector back) | Stop progression; remaining canary pods drain |
| Validation before cutover | Full pre-traffic validation in green | Validation happens *with* production traffic |
| User exposure on bad deploy | All users hit the new version after cutover | Only the canary % hit the new version |
| Best for | Coordinated DB/app migrations, major versions | Risk-sensitive ongoing changes, gradual rollout |
| Stickiness considerations | None needed during cutover | Session stickiness or feature flags often needed |
| Observability requirements | Can be modest | Strong SLO measurement required for automated promote/abort |

**Blue-Green flow:**
1. Blue is live serving 100%.
2. Deploy green at full capacity, no traffic yet.
3. Run smoke tests / load tests against green.
4. Flip the LB/service to point at green — all traffic moves instantly.
5. Blue stays running for fast rollback if needed.
6. After confidence period, scale down blue.

**Canary flow:**
1. Stable version serves 100%.
2. Deploy canary at small replica count (or use Argo Rollouts' `setWeight: 5`).
3. 5% of traffic goes to canary.
4. Automated analysis runs: error rate < SLO? p99 latency within tolerance? If yes, promote.
5. Progress: 25% → 50% → 100%.
6. Old version retired.
7. If at any step analysis fails, automatically roll back.

**When to choose blue-green:**
- Need instant rollback (one selector flip).
- Database schema changes that require coordinated cutover.
- Major version upgrades where pre-traffic validation is critical.
- You can afford 2× infra during the cutover.
- Traffic shape doesn't tolerate gradual migration (e.g., long-lived connections).

**When to choose canary:**
- High-frequency deploys where 2× infra cost adds up.
- Want production-traffic validation with bounded blast radius.
- Performance-sensitive changes where the only true test is real traffic.
- Strong SLO measurement is in place for automated decisions.
- Want fail-fast behavior — catch issues at 5%, not at 100%.

**A subtler difference:** blue-green tests the new version against pre-prod-style traffic (smoke tests, synthetic tests) before exposing it. Canary tests against real production traffic but limits the blast radius. Each has different validation properties.

**Architect's framing:**
> "Canary is more sophisticated and operationally efficient; blue-green is conceptually simpler with bigger guarantees. For ongoing changes to a service, canary wins. For major coordinated changes (DB schema, breaking API, runtime migration), blue-green wins because you can pre-validate everything before flipping. I run both in the same platform — canary by default, blue-green for the unusual cases where coordinated cutover is the constraint."

---

## 16. Implementing Lambda functions

> "Walking through the full implementation lifecycle:"

**1. Decide if Lambda is the right tool**

Lambda fits when:
- Event-driven workloads (S3 events, SQS messages, EventBridge events, API Gateway requests).
- Spiky or unpredictable traffic.
- Short-duration tasks (max 15 minutes runtime).
- Workloads that benefit from zero-idle-cost.

Lambda is wrong for:
- Long-running processes (>15 min).
- Steady, high-throughput compute (ECS/Fargate is cheaper at scale).
- Workloads requiring custom kernel modules or low-level system access.
- Anything where cold starts can't be tolerated.

**2. Choose deployment package format**

- **ZIP** — up to 250 MB unzipped. Standard for most functions.
- **Container image** — up to 10 GB. Useful for large dependencies, custom runtimes, parity with non-Lambda environments.
- **Lambda Layers** — share common code across functions. Reduces duplication.

**3. Write the function (Python example)**

```python
# handler.py
import json
import logging
import os
import boto3
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit

logger = Logger()
tracer = Tracer()
metrics = Metrics()

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

@logger.inject_lambda_context(correlation_id_path="requestContext.requestId")
@tracer.capture_lambda_handler
@metrics.log_metrics
def lambda_handler(event, context):
    logger.info("Processing event", extra={"event_count": len(event.get('Records', []))})

    for record in event['Records']:
        try:
            body = json.loads(record['body'])
            table.put_item(Item=body)
            metrics.add_metric(name="RecordsProcessed", unit=MetricUnit.Count, value=1)
        except Exception as e:
            logger.exception(f"Failed to process record: {e}")
            metrics.add_metric(name="RecordsFailed", unit=MetricUnit.Count, value=1)
            raise

    return {"statusCode": 200, "body": json.dumps({"processed": len(event['Records'])})}
```

**4. Define infrastructure (Terraform)**

```hcl
resource "aws_iam_role" "lambda" {
  name = "lambda-${var.function_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_custom" {
  role = aws_iam_role.lambda.id
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = ["dynamodb:PutItem", "dynamodb:GetItem"]
      Resource = aws_dynamodb_table.events.arn
    }]
  })
}

resource "aws_lambda_function" "this" {
  function_name = "process-events"
  role          = aws_iam_role.lambda.arn
  handler       = "handler.lambda_handler"
  runtime       = "python3.12"
  architectures = ["arm64"]    # Graviton — cheaper and faster
  memory_size   = 512
  timeout       = 30

  filename         = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")

  environment {
    variables = {
      TABLE_NAME = aws_dynamodb_table.events.name
      LOG_LEVEL  = "INFO"
    }
  }

  tracing_config { mode = "Active" }   # X-Ray

  reserved_concurrent_executions = 100
  
  dead_letter_config { target_arn = aws_sqs_queue.dlq.arn }
}

# Trigger: SQS
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn = aws_sqs_queue.events.arn
  function_name    = aws_lambda_function.this.arn
  batch_size       = 10
  maximum_batching_window_in_seconds = 5
}

# CloudWatch log group with retention
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${aws_lambda_function.this.function_name}"
  retention_in_days = 30
}
```

**5. CI/CD pipeline**

```yaml
# GitHub Actions
- name: Install deps
  run: pip install -r requirements.txt -t package/

- name: Copy source
  run: cp handler.py package/

- name: Create zip
  run: cd package && zip -r ../lambda.zip .

- name: Run tests
  run: pytest tests/

- name: SAST + dep scan
  run: |
    bandit -r .
    safety check

- name: Configure AWS via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/gha-deploy
    aws-region: us-east-1

- name: Terraform apply
  run: terraform apply -auto-approve
```

**6. Operational concerns**

- **Cold start mitigation**: Provisioned Concurrency or Lambda SnapStart (Java/Python/Ruby/.NET).
- **Concurrency limits**: `reserved_concurrent_executions` to cap, `provisioned_concurrent_executions` to pre-warm.
- **DLQ**: dead letter queue for failed async invocations (otherwise errors are silently lost).
- **Idempotency**: events can be delivered more than once; design handlers idempotent.
- **VPC**: avoid unless required (Hyperplane ENIs improved this dramatically, but still adds complexity).
- **Observability**: Lambda Powertools (structured logging, metrics, tracing), X-Ray, CloudWatch Logs Insights.

**7. Common patterns I use**

- **API Gateway + Lambda** for HTTP APIs without managing servers.
- **EventBridge + Lambda** for cross-service events.
- **S3 trigger + Lambda** for object processing (image resize, log analysis).
- **SQS + Lambda** for queue-driven processing with built-in scaling.
- **Scheduled Lambda** via EventBridge cron for periodic jobs.

**Architect's framing:**
> "Lambda's biggest operational wins are the things you don't have to do — no servers to patch, no autoscaling to configure, no capacity planning. The trade-off is the 15-minute runtime cap, cold starts (mostly tamed by SnapStart/Provisioned Concurrency now), and the discipline required to keep functions small and focused. Where Lambda fits, it's the cheapest and most operationally-light option. Where it doesn't fit, ECS Fargate or EKS is the next step up."

---

## 17. Cost optimization work

> "Yes — cost optimization has been a recurring focus. I'd frame it as a continuous discipline rather than a one-off cleanup:"

**Phase 1: Visibility**
- **Tagging strategy** enforced via SCPs and Config rules — every resource has `CostCenter`, `Owner`, `Environment`, `Application`.
- **Cost Explorer + Cost and Usage Reports** to QuickSight for custom team-level dashboards.
- **Budget alerts** at 50/80/100% of forecast, routed to Slack.
- **Kubecost** for EKS workload-level cost attribution.

**Phase 2: Quick wins (typically 20-30% reduction)**
- Right-sized EC2 based on CloudWatch metrics — Cost Explorer's right-sizing recommendations are reliable.
- Deleted orphans: unattached EBS volumes, unattached EIPs (now charged at $0.005/hour), old AMIs, idle ALBs.
- Stopped non-prod environments nightly and weekends via EventBridge + Lambda — typically 65% savings on dev.
- S3 lifecycle policies — Standard → IA → Glacier → Deep Archive. Intelligent-Tiering for unpredictable access patterns.
- EBS gp2 → gp3 (20% cheaper, equal/better performance).
- VPC Endpoints for S3/DynamoDB and Interface endpoints for high-traffic AWS APIs — eliminated significant NAT Gateway data charges.
- CloudWatch Logs retention: most teams default to "never expire." Set per log group based on actual need (typically 30/90/365 days).

**Phase 3: Commitments (10-40% additional)**
- **Compute Savings Plans** for baseline workload (covers EC2 + Fargate + Lambda flexibly). 1-year, no upfront — covered ~75% of steady-state.
- **Reserved Instances** for RDS, ElastiCache (more predictable than EC2).
- **Spot instances** for fault-tolerant workloads — CI runners, EKS data plane via Karpenter with diversified instance types, batch jobs. 60-90% discount.

**Phase 4: Architecture changes**
- Moved a Lambda-heavy batch workflow to Fargate Spot — cheaper at the steady throughput we had.
- Right-sized Kubernetes via VPA recommendations and tighter HPA targets.
- Consolidated low-utilization microservices that didn't need their own pods.
- Moved an over-provisioned RDS to Aurora Serverless v2 for a workload with bursty access patterns.

**Phase 5: Specific examples with numbers**

- **NAT Gateway data charges** were ~$8k/month. Moved AWS API traffic through Interface endpoints; cost dropped to ~$2k/month. ROI in 3 weeks.
- **EKS overprovisioning** — VPA showed average pod CPU usage was 35% of requests. Tightened requests; node count dropped 25%.
- **CloudWatch Logs ingest** for one chatty service was $12k/month at debug level. Moved debug logs to Loki (S3-backed), kept INFO-level in CloudWatch. Saved $9k/month.
- **DataSync vs S3 replication** — DataSync was running daily on 50TB; switched to S3 cross-region replication on a subset of prefixes. Saved $4k/month.
- **Reserved Instances for RDS** — committed 70% of the production RDS fleet to 1-year RIs. Saved 40% on those instances.

**Phase 6: Cultural**
- Monthly FinOps reviews per team with their cost dashboard.
- Cost-aware CI/CD — `infracost` in PRs showing the cost impact of each Terraform change.
- Service Control Policies preventing expensive resource types in dev.
- Unit economics tracking — cost per transaction, cost per user — to ensure cost scales sublinearly with growth.

**What I explicitly don't do for cost cutting:**
- Reduce production HA/redundancy. Outage costs >> infra savings.
- Disable monitoring/logging. Blind operations cost much more in incidents.
- Last-minute spot for tier-1 stateful workloads.

**Architect's framing:**
> "Cost optimization is a continuous discipline owned by engineering, not a quarterly fire drill driven by finance. The biggest wins are usually structural — tagging, SCPs, Savings Plans, killing orphans — rather than clever optimizations on individual services. I treat cost as a first-class non-functional requirement, alongside latency and availability."

---

## 18. Terraform lifecycle

The Terraform workflow is `init → plan → apply` (and `destroy` for cleanup), with `validate` and `fmt` as quality checks. The lifecycle for a working configuration:

**1. `terraform init`**

Initializes the working directory:
- Reads the `terraform` block (required version, providers, backend).
- Initializes the **backend** (connects to S3, Azure Storage, Terraform Cloud, etc., creates state file if absent).
- Downloads providers declared in `required_providers` into `.terraform/providers/`.
- Downloads modules referenced via `source` in `module` blocks.
- Writes `.terraform.lock.hcl` recording exact provider versions and checksums.

Run after: cloning, changing backend, adding/upgrading providers, adding modules.

**2. `terraform validate`**

Syntax check + internal consistency check. Doesn't access cloud APIs. Fast.

**3. `terraform fmt`**

Reformats `.tf` files to canonical style. `-check` flag in CI to enforce.

**4. `terraform plan`**

Computes the diff between desired state (code) and actual state (state file + refresh from cloud APIs):
- Acquires state lock.
- Refreshes state by querying cloud for current resource attributes — surfaces drift.
- Parses configuration, builds the resource DAG.
- Compares current state vs desired config, computes changes.
- Outputs the plan: `+ add`, `~ change`, `- destroy`, with attribute-level detail.
- Optionally saves a binary plan file with `-out=tfplan`.

`terraform plan -out=tfplan` produces a saved plan that `apply` can later execute exactly — what you reviewed is what gets applied.

**5. `terraform apply`**

Executes the plan:
- Acquires state lock.
- Walks the DAG in dependency order.
- Calls provider CRUD operations for each resource.
- Updates state incrementally after each operation (so a crash mid-apply leaves accurate state).
- Releases lock.

`terraform apply tfplan` applies the saved plan from step 4 — no prompt, deterministic.

**6. `terraform destroy`**

Reverses the apply — destroys all managed resources in reverse-dependency order. Reserved for ephemeral environments.

**Lifecycle meta-arguments (the other thing "lifecycle" can mean):**

```hcl
resource "aws_db_instance" "prod" {
  # ...
  lifecycle {
    prevent_destroy       = true                # apply fails if destroy planned
    ignore_changes        = [password, tags["LastModified"]]
    create_before_destroy = true                # replace = create new first, then destroy old
    replace_triggered_by  = [aws_kms_key.db.arn]   # force replacement when this changes
  }
}
```

- **`prevent_destroy`** — guardrail on production resources. Removing the lifecycle block is required before destruction.
- **`ignore_changes`** — Terraform won't try to revert these attributes (useful for ASG `desired_capacity`, rotated passwords).
- **`create_before_destroy`** — zero-downtime replacement (launch templates, certs).
- **`replace_triggered_by`** — force replacement when an upstream resource changes.

**CI/CD lifecycle pattern:**

```
PR opened
  → terraform fmt -check
  → terraform validate
  → tflint, tfsec, checkov
  → terraform plan (post output to PR)
  → reviewer approval

PR merged
  → terraform plan -out=tfplan
  → terraform apply tfplan
  → notify Slack
```

**Architect's framing:**
> "The lifecycle isn't just `init plan apply` — it's the whole discipline of how change reaches infrastructure. The pieces that matter most: peer-reviewed plans, applied via pipeline with saved plan files (never `-auto-approve` against ad-hoc plans), state in a hardened backend with versioning and locking, lifecycle meta-arguments protecting production resources from accidental destruction. The mechanics are simple; the operational discipline around them is what makes Terraform safe in production."

---

## 19. Auto-scaling

Auto-scaling adjusts capacity dynamically based on demand. AWS provides several auto-scaling mechanisms; in Kubernetes there are several more. Going through both:

**AWS Auto Scaling primitives:**

**1. EC2 Auto Scaling Groups (ASG)**
Manages a fleet of EC2 instances, automatically launching/terminating based on:
- **Target tracking** — keep average CPU at 60%, scale automatically to maintain.
- **Step scaling** — add N instances when alarm threshold breached, more as severity increases.
- **Scheduled scaling** — predictable patterns (scale up before business hours).
- **Predictive scaling** — ML-based, learns patterns and pre-scales.

```hcl
resource "aws_autoscaling_group" "app" {
  min_size            = 3
  max_size            = 20
  desired_capacity    = 5
  vpc_zone_identifier = aws_subnet.private[*].id
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  health_check_type         = "ELB"
  health_check_grace_period = 120
  target_group_arns         = [aws_lb_target_group.app.arn]
  
  lifecycle { create_before_destroy = true }
}

resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}
```

**2. Application Auto Scaling**
Generic scaling service for non-EC2 resources:
- **ECS services** — scale task count.
- **DynamoDB** — auto-scale read/write capacity.
- **Aurora replicas** — auto-add/remove read replicas.
- **Lambda Provisioned Concurrency** — auto-scale warm capacity.

**3. AWS Auto Scaling (the central service)**
Unified control plane orchestrating scaling across multiple AWS resources for a workload.

**Kubernetes auto-scaling layers:**

**1. Horizontal Pod Autoscaler (HPA)**
Scales pod replicas in a Deployment/StatefulSet based on metrics:
- CPU/memory utilization (built-in via metrics-server).
- Custom metrics from Prometheus (via prometheus-adapter).
- External metrics (Azure Service Bus depth, SQS depth) via KEDA.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  - type: Pods
    pods:
      metric: { name: http_requests_per_second }
      target: { type: AverageValue, averageValue: "200" }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{ type: Percent, value: 25, periodSeconds: 60 }]
```

**2. Vertical Pod Autoscaler (VPA)**
Adjusts pod CPU/memory requests rather than count. Modes:
- `Off` — recommendations only (my preferred mode — apply manually).
- `Initial` — set on pod creation.
- `Auto` — evict and restart pods with new requests.

VPA shouldn't be used with HPA on the same metric (they fight). Pattern: HPA on custom metrics (RPS), VPA for memory sizing.

**3. Cluster Autoscaler (CA)**
Watches for unschedulable pods, adds nodes from the configured node groups; removes underutilized nodes.

**4. Karpenter** (modern alternative to CA)
- Provisions right-sized nodes on demand from a flexible pool of instance types.
- Faster scale-up (seconds vs minutes).
- Better bin-packing.
- Native consolidation (replaces underutilized nodes with smaller ones).
- Now Microsoft's recommended for AKS too (Node Autoprovisioning).

**5. KEDA (Kubernetes Event-Driven Autoscaling)**
Scales based on external events:
- Azure Service Bus, AWS SQS depth.
- Kafka consumer lag.
- Database query results.
- Scheduled scalers (cron).
- Scale-to-zero for event-driven workloads.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: worker }
spec:
  scaleTargetRef: { name: worker-deploy }
  minReplicaCount: 0       # scale to zero when no work
  maxReplicaCount: 50
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123/orders
      queueLength: "5"
```

**Production stack I run:**

- **HPA** on every deployment for pod scaling.
- **KEDA** for event-driven workers (queues, Kafka).
- **Karpenter** for node scaling.
- **VPA in recommendation mode** to surface right-sizing opportunities.
- **PDBs and topology spread constraints** so scaling decisions don't violate availability.

**Common patterns and pitfalls:**

- **HPA without setting CPU requests on pods** — utilization is undefined, scaling doesn't work.
- **CPU limits causing throttling** — pod gets enough CPU on average but bursts get clipped. Most teams now avoid setting CPU limits.
- **Cluster autoscaler not scaling** — pods stuck Pending because of `nodeSelector`, taints, or PVC topology constraints the autoscaler can't satisfy.
- **Scale-down too aggressive** — pods constantly evicted and re-created. Tune `stabilizationWindowSeconds` and PDBs.
- **No predictive scaling for known patterns** — Black Friday traffic shouldn't be discovered by HPA; pre-scale via scheduled scaling.

**Architect's framing:**
> "Auto-scaling is layered: pod-level (HPA/KEDA) responds in seconds, node-level (Karpenter/CA) responds in 30-60s, regional-level (ASG with capacity providers) for stateful infrastructure. The hardest part isn't configuring any single layer — it's getting them to cooperate. HPA scales pods, can't find capacity, Karpenter provisions a node, pod schedules — that whole cycle is fast on Karpenter and slow on CA. The other hard part is *down*-scaling — services with long-lived connections or warm-up costs don't tolerate aggressive scale-down. I tune scale-up to be eager and scale-down to be conservative."

---

That covers all 19 questions with senior-level depth. Likely Deloitte-style follow-ups to be ready for:

- **"Walk me through the worst production incident you've led."** — have a structured 5-7 min story ready.
- **"You're advising a client on K8s vs ECS — how do you frame the decision?"** — ecosystem, portability, team skills, operational tolerance.
- **"How do you handle a client engineer who insists on a pattern you know is wrong?"** — leadership angle; consultative pushback with data.
- **"What's your view on multi-cloud?"** — honest take with trade-offs; not "more cloud = better."

# Senior DevOps Interview — Complete Answers

Going through each with the depth a senior engineer brings.

---

## 1. Your last project — work and role

Tailor this to your actual experience. Here's the structure of a strong answer:

> "My most recent project was platform engineering for a fintech application running on AWS — a payments and account-management platform serving ~2 million daily active users.
>
> **The platform:** EKS clusters in us-east-1 (active) and us-west-2 (warm standby), running about 35 microservices in Java, Node.js, and Python. Data layer was Aurora PostgreSQL Global Database, DynamoDB Global Tables, ElastiCache Redis, and MSK for event streaming. Edge via CloudFront + Route 53 with health-check-based failover.
>
> **My role:** Senior DevOps engineer leading the platform team of four. End-to-end ownership:
>
> - **Infrastructure:** Owned the Terraform module library — VPC, EKS, RDS, IAM, networking. Per-environment state files in S3 with DynamoDB locking, applied via GitHub Actions pipelines with OIDC federation. No long-lived AWS credentials anywhere.
>
> - **CI/CD:** Designed the GitHub Actions reusable workflows that all 35 services use. Build-test-scan-sign pattern, image to ECR, image tag bump in a GitOps repo, ArgoCD reconciles to EKS. Argo Rollouts canary for tier-1 services with SLO-based auto-rollback.
>
> - **Observability:** Prometheus + Grafana for metrics via kube-prometheus-stack, Loki for logs, Tempo for traces, all instrumented via OpenTelemetry. SLO dashboards per service, Alertmanager → PagerDuty.
>
> - **Day-2:** Quarterly DR game days, monthly capacity reviews, blameless postmortems for any Sev-1 or Sev-2, on-call rotation, runbooks tied to alert annotations.
>
> - **Mentorship:** Led platform office hours weekly. Reviewed Helm charts and Terraform PRs for product teams. Drove the migration from Jenkins to GitHub Actions and from push-based CD to GitOps.
>
> **A specific thing I'm proud of:** we drove our deploy-to-prod time from a 2-hour, manual, error-prone process down to ~12 minutes fully automated, with the Argo Rollouts canary catching three production issues in the first quarter that would otherwise have caused incidents. Mean time to detection on issues that did slip through dropped from ~25 min to ~3 min because of better SLO-burn-rate alerting."

Three principles for this answer: under 2 minutes, anchor in concrete scale and tools, end with a measurable improvement you owned.

---

## 2. CI/CD pipeline in my company

> "We run a GitOps model with separation between build and deploy:
>
> **Build side (per-service, GitHub Actions):**
> - Developer opens PR or pushes to `develop`/`main`.
> - GitHub Actions reusable workflow runs:
>   1. Checkout with full history.
>   2. Maven build + unit tests + JaCoCo coverage.
>   3. SonarQube scan with quality-gate blocking the pipeline.
>   4. Snyk dependency vulnerability scan, fails on critical/high with fix available.
>   5. Build Docker image with multi-stage Dockerfile.
>   6. Trivy CVE scan on the image — fails on critical/high.
>   7. Cosign signing using a Key Vault-managed key.
>   8. Push to ECR with immutable tag: `1.4.2-a1b2c3d-build2087` (semver + git short SHA + build number).
>
> **Deploy side (GitOps):**
> - The same pipeline checks out a separate GitOps manifests repo.
> - Bumps the image tag in `environments/dev/values.yaml` and commits.
> - ArgoCD watches the GitOps repo, detects the change, syncs to EKS via Helm.
> - Webhook from GitHub to ArgoCD makes this near-instant.
>
> **Promotion through environments:**
> - **Dev:** auto on merge to `develop`. ArgoCD has `selfHeal: true` and `automated.prune: true`.
> - **Staging:** dev pipeline triggers a smoke-test job; on pass, raises a PR bumping `environments/staging/values.yaml`. Auto-merge if smoke tests pass.
> - **Prod:** explicit PR in the GitOps repo with two-person approval requirement. Once merged, ArgoCD reconciles.
> - **Tier-1 prod services** (payments, checkout, auth) use Argo Rollouts canary: 10% → 25% → 50% → 100% with analysis at each step against Prometheus SLOs. If error rate or p99 latency regresses, auto-aborts and routes 100% back to stable.
>
> **Security and compliance built in:**
> - **OIDC federation** to AWS — no static credentials in GitHub.
> - **Required reviewers** on prod via GitHub Environments.
> - **Branch protection** on `main`: required status checks, no force-push, no merge without review.
> - **Image admission verification** in cluster via Kyverno — only signed images from approved registries can run.
> - **Audit:** every deploy traces back to a Git commit; ArgoCD UI shows who synced what when.
>
> **Why this shape:**
> - **Build once, promote artifact** — never rebuild for prod.
> - **GitOps over push-based CD** — pipeline doesn't have prod cluster credentials, only ArgoCD inside the cluster does. Smaller security surface.
> - **PR-based promotions** — every prod change is reviewable, auditable, and revertable via `git revert`.
> - **Argo Rollouts for risk-tier-1** — canary catches issues at 10% traffic, not after 100% cutover."

---

## 3. Production and DR in sync via pipeline

This is asking how to keep prod and DR aligned across all environments. The pattern depends on what "in sync" means — typically: same application versions, same configuration, same data, but DR running at warm standby capacity.

> "I'd architect this in a few layers because 'in sync' has multiple meanings — code, config, data, and deployment timing.
>
> **Layer 1: Code and infrastructure parity via GitOps**
>
> Both clusters (prod + DR) point to the same GitOps repo. ArgoCD runs in both clusters watching the same Application definitions, with environment-specific overlays:
>
> ```
> gitops-repo/
> ├── environments/
> │   ├── dev/values.yaml
> │   ├── preprod/values.yaml
> │   ├── prod-primary/values.yaml      ← us-east-1
> │   └── prod-dr/values.yaml           ← us-west-2
> ├── apps/
> └── charts/
> ```
>
> The pipeline promotes through Dev → PreProd → Prod, and the Prod step bumps the image tag in **both** `prod-primary/values.yaml` and `prod-dr/values.yaml` in the same PR. Both ArgoCD instances reconcile simultaneously. Result: code parity in seconds.
>
> Better — use an **ArgoCD ApplicationSet** to fan out a single definition across both prod clusters:
>
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: ApplicationSet
> metadata: { name: myapp-prod }
> spec:
>   generators:
>   - list:
>       elements:
>       - cluster: prod-primary
>         url: https://prod-primary.example.com
>         valuesFile: values-prod-primary.yaml
>       - cluster: prod-dr
>         url: https://prod-dr.example.com
>         valuesFile: values-prod-dr.yaml
>   template:
>     metadata: { name: 'myapp-{{cluster}}' }
>     spec:
>       source:
>         repoURL: https://github.com/company/gitops
>         path: apps/myapp
>         helm:
>           valueFiles: ['../../environments/{{cluster}}/{{valuesFile}}']
>       destination:
>         server: '{{url}}'
>         namespace: prod
>       syncPolicy:
>         automated: { prune: true, selfHeal: true }
> ```
>
> One Git commit bumping the image tag → both clusters reconcile.
>
> **Layer 2: Pipeline structure**
>
> ```
> Dev:          auto-deploy on merge to develop
> PreProd:      auto on merge to main (with smoke test gate)
> Prod-Primary: PR + 2-person approval
>               └─→ ApplicationSet fans out to BOTH prod-primary and prod-dr
> ```
>
> Or, for stricter separation:
> ```
> Prod-Primary: deploy, observe for N hours (canary stable)
> Prod-DR:      automated promotion only after prod-primary green for N hours
> ```
>
> The second pattern is safer for regulated environments — prod-DR is the 'last to deploy' so it remains a known-good fallback if prod-primary has issues.
>
> **Layer 3: Infrastructure parity via Terraform**
>
> Same Terraform modules applied to both regions with region-specific tfvars files:
> ```
> environments/
>   prod-primary/     # us-east-1
>     main.tf
>     terraform.tfvars
>   prod-dr/          # us-west-2
>     main.tf
>     terraform.tfvars
> ```
> Terraform pipeline applies both in the same PR — same module versions, same configuration, just different regions and CIDRs.
>
> **Layer 4: Data sync (the hard part)**
>
> Code parity is easy; data parity is where DR architectures usually break.
> - **Aurora Global Database** — primary in us-east-1, secondary in us-west-2 with sub-second replication lag. No app changes needed.
> - **DynamoDB Global Tables** — multi-region active-active.
> - **S3 cross-region replication (CRR)** — versioning + RTC for SLA-backed 15-min replication.
> - **MSK / Kafka** — MirrorMaker 2 replicating topics cross-region.
> - **ElastiCache Redis Global Datastore** — cross-region replication with sub-second lag.
> - **Secrets Manager** — multi-region replicated secrets so DR can decrypt independently.
> - **KMS multi-region keys** so encrypted data is decryptable in DR.
>
> **Layer 5: Operational discipline**
>
> Code parity doesn't matter if the DR plan is fiction. Quarterly game days that actually fail over to DR for a 4-hour window, run real traffic, then fail back. Without this, you'll find out at the worst possible moment that there's a Lambda with `us-east-1` hardcoded, or an IAM role that wasn't replicated, or a Secrets Manager secret that's only in primary.
>
> **What I'd actually recommend as the pipeline shape:**
>
> One pipeline, one Git commit, fans out to both regions via ApplicationSet. Data sync handled by cloud-native replication, monitored with alerts on lag thresholds. Quarterly drills to verify the whole thing works. RTO/RPO measured during drills, not aspirational."

---

## 4. Handling Terraform drift

> "Drift is when the real cloud state diverges from Terraform's state. It happens because someone clicked in the console, another tool (Ansible, manual scripts) modified resources, or auto-scaling changed counts. My approach is layered: detect early, decide intent, then resolve.
>
> **Detection — continuous, not reactive:**
>
> 1. **Scheduled drift check** in CI:
>    ```bash
>    terraform plan -detailed-exitcode -refresh-only
>    # Exit code 0: no drift
>    # Exit code 2: drift detected → alert
>    ```
>    Runs nightly across all environments. Alerts to Slack on drift.
>
> 2. **driftctl** for continuous detection across providers, including resources Terraform isn't managing (which is also useful).
>
> 3. **Terraform Cloud / Enterprise** built-in drift detection if using that platform.
>
> 4. **AWS Config rules** detecting configuration changes and routing events to EventBridge.
>
> **Once detected — decide what the drift means:**
>
> Three possibilities:
>
> **A. The manual change was wrong** — revert it.
> ```bash
> terraform apply
> # Pushes Terraform's desired state back, overwriting the manual change
> ```
>
> **B. The manual change was correct and should be adopted** — update the code.
> ```bash
> # Inspect what changed
> aws ec2 describe-security-groups --group-ids sg-abc123
>
> # Update the Terraform code to match
> # Now plan is clean again
> terraform plan   # Should show no changes
> ```
>
> **C. The change is managed elsewhere intentionally** — tell Terraform to ignore it.
> ```hcl
> resource "aws_autoscaling_group" "app" {
>   # ...
>   lifecycle {
>     ignore_changes = [desired_capacity]   # AWS autoscaling manages this
>   }
> }
> ```
> Use sparingly — overuse defeats the purpose of IaC.
>
> **The cultural fix (the real answer):**
>
> Detection-and-repair is treating the symptom. The actual fix is making manual changes hard:
> - **IAM lockdown** — engineers don't have permissions to modify Terraform-managed resources directly outside the pipeline role.
> - **SCPs** at the AWS Organizations level — break-glass roles only, with audit logging.
> - **Resource tagging** — `ManagedBy: terraform` tag, with Config rules detecting changes to tagged resources.
> - **CloudTrail event-driven alerts** on resource changes outside the pipeline IAM role — auto-page the operator who made the change.
> - **Postmortem when drift happens repeatedly** — why is someone making manual changes? Is the pipeline too slow? Is there an emergency path that should be formalized?
>
> **For unrecoverable drift** (resource destroyed or completely reconfigured outside Terraform):
> - `terraform import` to re-attach existing resources to state.
> - In extreme cases, `terraform state rm` to detach and re-import.
>
> **Architect's framing:** drift is usually a symptom of a process problem, not a Terraform problem. The fix is in IAM and culture, not in `terraform refresh`. I run drift detection continuously, but I aim for zero drift in steady state — when drift appears, that's the trigger for a conversation about why the pipeline didn't handle the need."

---

## 5. Console changes to a Terraform-managed VPC

> "This is the classic drift scenario. Walking through what happens and what I'd do:
>
> **What just happened:**
> Terraform's state file says the VPC has CIDR X, SG rule Y, etc. Someone went to the AWS console and modified something — added an SG rule, changed a tag, deleted a route. Terraform doesn't know. Next plan will show a diff.
>
> **My response sequence:**
>
> **Step 1: Discover what changed**
>
> ```bash
> terraform plan
> # Output:
> #   ~ resource "aws_security_group" "app" {
> #       ingress {
> #         + cidr_blocks      = ["10.0.0.0/16"]    ← added manually
> #         from_port         = 22
> #         to_port           = 22
> #         protocol          = "tcp"
> #       }
> #     }
> ```
>
> The plan shows me exactly what drifted. I now know what was changed in the console.
>
> **Step 2: Find out who and why**
>
> ```bash
> # CloudTrail — who made the change?
> aws cloudtrail lookup-events \
>   --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-abc123 \
>   --start-time $(date -u -d '24 hours ago' +%FT%T)
> ```
>
> Talk to the person. Was it a hotfix during an incident? A misunderstanding about Terraform ownership? A legitimate need the pipeline didn't address?
>
> **Step 3: Decide based on intent**
>
> **A. The change was wrong — revert it:**
> ```bash
> terraform apply
> # Brings cloud state back to match the code
> ```
> Communicate to the person: 'I'm reverting this change. Please raise a PR for the desired state next time.'
>
> **B. The change was correct — adopt it into code:**
> ```hcl
> resource "aws_security_group" "app" {
>   # ...
>   ingress {
>     description = "SSH from internal network — added during incident #1234"
>     from_port   = 22
>     to_port     = 22
>     protocol    = "tcp"
>     cidr_blocks = ["10.0.0.0/16"]
>   }
> }
> ```
> Submit as a PR, get review, merge. `terraform plan` now shows no diff.
>
> **C. The change is intentionally external — ignore:**
> Rare for VPC components, but if there's a legitimate reason (e.g., another automation manages this resource), use `lifecycle.ignore_changes`. I'd push back hard on this for VPC components — that's the foundation, and external management is a smell.
>
> **Step 4: Prevent recurrence**
>
> - **Remove the IAM permission** that allowed the manual change. The person should have had read-only access to VPC components in prod.
> - **Add a Config rule** detecting changes to that specific SG outside the pipeline role.
> - **Document in the runbook** — 'For emergency SG changes, use this break-glass procedure that still produces an auditable record.'
>
> **What I'd specifically NOT do:**
>
> - **`terraform apply` with `-refresh-only`** as a way to 'accept' the drift silently. That just updates state without updating code; next code change reverts the drift confusingly.
> - **Edit the state file by hand**. Never.
> - **Skip the conversation** with whoever made the change. Drift is a process signal, not a state-file problem.
>
> **Architect's framing:**
> Drift on a VPC is particularly serious because VPC components are foundational — a deleted route or modified SG can cascade. I treat any console change to Terraform-managed infrastructure as a postmortem-worthy event in prod, even if the change itself was benign. The conversation isn't 'you made a change in the console' — it's 'what was the pressure that made bypassing the pipeline feel necessary, and how do we fix that?'"

---

## 6. Terraform state file

> "The state file is Terraform's persistent record of what it manages. It's a JSON file that maps resource declarations in code to real-world cloud resources and stores their attributes.
>
> **What it contains:**
> - **Resource mappings**: `aws_instance.web` → `i-0abc123def456`.
> - **Attribute snapshots**: every property of every managed resource (current as of last refresh).
> - **Dependencies**: the DAG Terraform uses to determine apply/destroy order.
> - **Metadata**: Terraform version, provider versions, output values.
> - **Sensitive data**: passwords, KMS keys, certificates — stored raw, not encrypted.
>
> **Why it exists — five concrete reasons:**
>
> 1. **Mapping** — without state, Terraform can't know which real cloud resource corresponds to which code block. The cloud API returns IDs; the code uses logical names.
> 2. **Performance** — caches resource attributes so plans don't have to API-query every resource.
> 3. **Dependency tracking** — the DAG built from state determines order of operations.
> 4. **Drift detection** — refresh queries cloud, compares to state, surfaces what changed externally.
> 5. **Concurrency safety** — when paired with a locking backend, prevents two simultaneous applies from corrupting state.
>
> **Critical operational properties:**
>
> - **State contains secrets in plaintext** — this is *the* reason state must be encrypted at rest and access-controlled. Any secret you put in a Terraform resource (RDS password, KMS key material) ends up in state.
> - **State must be remote in any team setup** — local state on a laptop is a single point of failure and no concurrency protection.
> - **State is the source of truth** for what Terraform manages. The code is the source of truth for what you *want*; state is what Terraform *currently has*.
> - **Versioning is mandatory** — enable bucket versioning on S3 or soft-delete on Azure Storage so corruption is recoverable.
> - **State is schema-versioned** — different Terraform versions write different state schemas. Mismatched versions across team members can corrupt state.
>
> **What you can do with it:**
> ```bash
> terraform state list                        # what's tracked
> terraform state show aws_instance.web       # detailed attributes
> terraform state mv old_addr new_addr        # rename / refactor
> terraform state rm aws_instance.web         # remove from state (doesn't destroy)
> terraform state pull > backup.tfstate       # download for backup
> terraform state push backup.tfstate         # restore (use carefully)
> ```
>
> **What you should never do:**
> - Edit the JSON by hand. Use `terraform state` subcommands which validate.
> - Commit state to Git. Even encrypted-at-rest. Just don't.
> - Share state files between unrelated environments.
> - Delete state versions immediately after a migration. Wait 30 days minimum.
>
> **The bigger picture:**
> State is what makes Terraform stateful. Tools like Pulumi, OpenTofu work the same way. The alternatives — tag-based discovery, query-the-cloud-every-time — have been tried and have worse properties (slow, ambiguous, can't represent intent vs reality). State is the cost we pay for Terraform's model, and the discipline around it (encryption, versioning, locking, backup) is what makes that cost manageable."

---

## 7. Why DynamoDB for Terraform state locking — not Oracle or MongoDB?

This is a fair question that tests whether you understand the locking mechanism, not just that you use it.

> "The S3 backend in Terraform supports DynamoDB specifically as its locking partner. The reason isn't that Oracle or MongoDB couldn't theoretically do the job — they could, in principle, provide a record with a primary key and conditional writes. The reason is **architectural and practical**:
>
> **1. Native integration in the AWS provider**
>
> The Terraform AWS backend has a built-in code path that talks to DynamoDB for locking. The implementation is `s3.Backend` calling `dynamodb.PutItem` with a conditional expression. There's no integration code for Oracle, MongoDB, Postgres, etc. AWS chose DynamoDB and that's what Terraform supports out of the box for the S3 backend.
>
> **2. The locking primitive Terraform needs is tiny**
>
> Terraform needs:
> - A primary key (`LockID`).
> - Conditional write — 'create this row if it doesn't exist; fail if it does.'
> - Atomic delete — release the lock.
>
> DynamoDB's `PutItem` with `ConditionExpression: attribute_not_exists(LockID)` is purpose-built for this. The whole locking flow is two API calls (acquire + release). It's not a database problem; it's a distributed-lock problem.
>
> **3. Operational properties that fit:**
> - **Managed service** — no servers, no patching, no failover to configure.
> - **Strong consistency** — DynamoDB's conditional write is strongly consistent within a region. No race conditions.
> - **High availability** — multi-AZ by default.
> - **Cost** — pennies per month for typical Terraform workloads. A lock table sees maybe dozens of writes per day.
> - **IAM-integrated** — same auth model as S3.
> - **Available wherever AWS is** — no network plumbing to manage.
>
> **4. Why not Oracle or MongoDB specifically:**
>
> - **Operational overhead** — running Oracle for a tiny locking workload is comically expensive and operationally heavy.
> - **No native integration** — would require a custom Terraform backend, which doesn't exist in the supported list.
> - **MongoDB** could provide the locking semantics via `findAndModify`, but again, no Terraform backend supports it.
> - **HA story** — managing Oracle HA or Mongo replica sets just to coordinate Terraform locks is wildly over-engineered.
>
> **5. Other backends use their native locking:**
>
> - **Azure backend** — uses Azure Storage blob leases natively. No DynamoDB equivalent needed.
> - **GCS backend** — uses GCS object versioning and atomic operations.
> - **Terraform Cloud / Enterprise** — uses its own internal locking.
> - **Consul backend** — uses Consul's KV with sessions for locking.
> - **PostgreSQL backend** — does exist, uses advisory locks. So you *can* use Postgres if you want.
>
> So 'why DynamoDB' is really 'why does the S3 backend specifically use DynamoDB' — and the answer is: it was the natural AWS-native pair when the S3 backend was designed. Could there be a Terraform backend that uses Oracle? Theoretically. But nobody's built one because: there's no demand, the operational cost is absurd, and DynamoDB does the job at almost-zero cost.
>
> **Architect's framing:**
> The lock isn't a 'database' question — it's a distributed-coordination question. The constraints are: must be highly available, must support atomic conditional writes, must be cheap to operate, must be IAM-integrated. DynamoDB checks every box; Oracle and MongoDB introduce operational overhead disproportionate to the problem. If we were on-premises and wanted locking, I'd use Consul or etcd before reaching for Oracle."

---

## 8. Scaling in Kubernetes

> "Kubernetes has three primary scaling axes — pods, pod resources, and nodes — plus event-driven scaling. Going through each:
>
> **1. Horizontal Pod Autoscaler (HPA)** — scales the *number of pod replicas*
>
> Watches metrics, adjusts the Deployment's replica count. Default supports CPU/memory; with adapters supports custom metrics from Prometheus and external metrics from any source (queue depth, etc.) via KEDA.
>
> ```yaml
> apiVersion: autoscaling/v2
> kind: HorizontalPodAutoscaler
> metadata: { name: api }
> spec:
>   scaleTargetRef:
>     apiVersion: apps/v1
>     kind: Deployment
>     name: api
>   minReplicas: 3
>   maxReplicas: 50
>   metrics:
>   - type: Resource
>     resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
>   - type: Pods
>     pods:
>       metric: { name: http_requests_per_second }
>       target: { type: AverageValue, averageValue: "200" }
>   behavior:
>     scaleUp:
>       stabilizationWindowSeconds: 0
>       policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
>     scaleDown:
>       stabilizationWindowSeconds: 300
>       policies: [{ type: Percent, value: 25, periodSeconds: 60 }]
> ```
>
> The `behavior` config matters: aggressive scale-up (double pods in 30s if needed), conservative scale-down (max 25% drop per minute) so we don't oscillate.
>
> **2. Vertical Pod Autoscaler (VPA)** — adjusts pod CPU/memory *requests*
>
> Right-sizes resources based on observed usage. Three modes:
> - `Off` — recommendations only. My preferred mode in prod — surfaces opportunities without auto-evicting pods.
> - `Initial` — sets requests on pod creation.
> - `Auto` — evicts and restarts pods with new requests.
>
> VPA shouldn't run with HPA on the same metric (they fight). Pattern: HPA on RPS (custom metric), VPA for memory sizing.
>
> **3. Cluster Autoscaler (CA) / Karpenter** — scales the *number of nodes*
>
> Watches for unschedulable pods, adds nodes; removes underutilized nodes.
>
> **Karpenter** is the modern replacement on AWS:
> - Provisions right-sized nodes on demand from a flexible pool of instance types.
> - Faster (seconds vs minutes for traditional CA).
> - Better bin-packing — chooses the cheapest instance that fits pending pods.
> - Native consolidation — replaces underutilized nodes with smaller ones.
>
> ```yaml
> apiVersion: karpenter.sh/v1
> kind: NodePool
> metadata: { name: default }
> spec:
>   template:
>     spec:
>       requirements:
>       - key: karpenter.k8s.aws/instance-family
>         operator: In
>         values: [m6i, m6a, m7i]
>       - key: karpenter.k8s.aws/instance-size
>         operator: In
>         values: [large, xlarge, 2xlarge]
>       - key: karpenter.sh/capacity-type
>         operator: In
>         values: [on-demand, spot]
>   limits: { cpu: 1000 }
>   disruption:
>     consolidationPolicy: WhenUnderutilized
>     consolidateAfter: 30s
> ```
>
> **4. KEDA — event-driven autoscaling**
>
> Scales based on external events: SQS queue depth, Kafka lag, Azure Service Bus, scheduled cron, database query results. Supports scale-to-zero — pods drop to 0 when idle.
>
> ```yaml
> apiVersion: keda.sh/v1alpha1
> kind: ScaledObject
> metadata: { name: worker }
> spec:
>   scaleTargetRef: { name: worker-deploy }
>   minReplicaCount: 0       # scale to zero when no work
>   maxReplicaCount: 50
>   triggers:
>   - type: aws-sqs-queue
>     metadata:
>       queueURL: https://sqs.us-east-1.amazonaws.com/123/orders
>       queueLength: "5"
> ```
>
> **The integrated picture:**
>
> In production EKS:
> - **HPA** on every Deployment with appropriate min/max.
> - **KEDA** for queue/event-driven workers.
> - **Karpenter** for node provisioning.
> - **VPA in recommendation mode** for ongoing right-sizing data.
> - **PodDisruptionBudgets** so scaling decisions don't take services below quorum.
> - **topologySpreadConstraints** so scaling doesn't cluster pods on one AZ.
>
> **Common pitfalls:**
> - **HPA without setting CPU requests** — utilization is undefined, scaling broken.
> - **CPU limits causing throttling** — pod is fine on average but bursts get clipped. Many SRE shops now skip CPU limits entirely on latency-sensitive services.
> - **Cluster autoscaler not scaling** — pods stuck Pending because of nodeSelector, taints, or PVC topology constraints the autoscaler can't satisfy.
> - **Scale-down too aggressive** — pods evicted and recreated constantly. Tune `stabilizationWindowSeconds`.
> - **No predictive scaling** for known patterns — Black Friday shouldn't be discovered reactively; use scheduled scaling or pre-scale.
>
> **Architect's framing:**
> Auto-scaling is layered: pod-level responds in seconds, node-level responds in 30-60s on Karpenter (minutes on CA), regional-level for stateful infra is much slower. The hard part isn't any single layer — it's getting them to cooperate, and getting scale-down conservative enough to avoid oscillation. My default tuning: eager scale-up, conservative scale-down."

---

## 9. Designing a load-balanced, auto-scaled architecture

> "Walking through this end-to-end on AWS for a customer-facing web application:
>
> **Layer 1: Global edge**
>
> ```
> User → Route 53 (DNS) → CloudFront (CDN + WAF) → Regional ALB
> ```
>
> - **Route 53** with health-check-based failover or latency-based routing for multi-region.
> - **CloudFront** for global edge caching of static assets, TLS termination, AWS WAF integration, DDoS protection.
> - Reduces origin load and improves user latency globally.
>
> **Layer 2: Regional load balancing**
>
> - **Application Load Balancer (ALB)** in public subnets, spread across at least three AZs.
> - **SSL/TLS termination** at the ALB with ACM certificates.
> - **Path-based and host-based routing** to different target groups.
> - **WAF attached** for OWASP rules, bot mitigation, rate limiting.
> - **Health checks** at L7 against `/health/ready` endpoint that actually validates downstream dependencies.
>
> **Layer 3: Compute tier (the auto-scaled part)**
>
> Two patterns depending on workload:
>
> **Option A: EKS with HPA + Karpenter**
> ```
> ALB → AWS Load Balancer Controller → Ingress → ClusterIP Services → Pods
>                                                                       ↑
>                                                                       HPA scales pods
>                                                                       ↑
>                                                                       Karpenter scales nodes
> ```
> - EKS cluster across three AZs.
> - Pods deployed with HPA (CPU + custom metrics from Prometheus).
> - Karpenter provisions nodes on demand, prefers Spot for stateless workloads with diversified instance types.
> - PodDisruptionBudgets enforce minimum availability during scaling.
> - topologySpreadConstraints distribute pods across AZs.
>
> **Option B: EC2 Auto Scaling Group**
> ```
> ALB → Target Group → ASG (instances launched from a baked AMI)
>                       ↑
>                       Target tracking on ASGAverageCPUUtilization
> ```
> - ASG spans three AZs.
> - Launch Template with a baked Packer AMI containing the application.
> - Target tracking policy maintaining 60% CPU.
> - Mixed Instances Policy combining on-demand + Spot with capacity rebalancing.
> - Health check type: ELB (uses ALB's health checks, not EC2's status checks).
> - Health check grace period of 120s to allow boot + warm-up.
>
> ```hcl
> resource "aws_autoscaling_group" "app" {
>   min_size            = 3
>   max_size            = 30
>   desired_capacity    = 5
>   vpc_zone_identifier = aws_subnet.private[*].id   # three AZs
>   target_group_arns   = [aws_lb_target_group.app.arn]
>   health_check_type   = "ELB"
>   health_check_grace_period = 120
>
>   mixed_instances_policy {
>     launch_template {
>       launch_template_specification {
>         launch_template_id = aws_launch_template.app.id
>         version            = "$Latest"
>       }
>       override { instance_type = "m6i.large" }
>       override { instance_type = "m6a.large" }
>       override { instance_type = "m7i.large" }
>     }
>     instances_distribution {
>       on_demand_base_capacity                  = 3       # baseline
>       on_demand_percentage_above_base_capacity = 25      # 75% spot above baseline
>       spot_allocation_strategy                 = "capacity-optimized"
>     }
>   }
>
>   lifecycle { create_before_destroy = true }
> }
>
> resource "aws_autoscaling_policy" "cpu_target" {
>   name                   = "cpu-tracking"
>   autoscaling_group_name = aws_autoscaling_group.app.name
>   policy_type            = "TargetTrackingScaling"
>   target_tracking_configuration {
>     predefined_metric_specification {
>       predefined_metric_type = "ASGAverageCPUUtilization"
>     }
>     target_value = 60.0
>   }
> }
> ```
>
> **Layer 4: Data tier**
>
> Auto-scaled within sensible limits — not everything scales horizontally:
> - **RDS Aurora** — auto-scaling read replicas based on CPU. Writer is vertical scale only.
> - **DynamoDB** — auto-scaling read/write capacity, or on-demand mode for unpredictable workloads.
> - **ElastiCache** — cluster mode with manual scaling typically (resharding is expensive).
> - **S3** — scales transparently.
>
> **Layer 5: Async tier**
>
> - **SQS / Kafka (MSK)** for buffering between tiers.
> - **KEDA** scaling consumer pods based on queue depth — scale to zero when idle.
>
> **Layer 6: Observability for autoscaling**
>
> Critical to making scaling work:
> - **CloudWatch alarms** on ASG: `GroupInServiceInstances`, `GroupDesiredCapacity`, `CPUUtilization`.
> - **Karpenter metrics** in Prometheus: provisioning latency, consolidation rate, spot interruptions.
> - **HPA events**: `kubectl describe hpa` shows scaling decisions.
> - **Application metrics**: request rate, queue depth, p99 latency — these drive *custom-metric* scaling.
>
> **Layer 7: Cost considerations**
>
> Auto-scaling without cost discipline runs up bills:
> - **Spot for stateless** with capacity-rebalancing and diverse instance types.
> - **Compute Savings Plans** for baseline (on-demand portion).
> - **Scheduled scale-down** for predictable low-traffic windows.
> - **Cluster Autoscaler / Karpenter consolidation** so nodes don't sit idle.
> - **Kubecost** or **AWS Cost Explorer** with tags for visibility.
>
> **The complete picture:**
>
> ```
> Route 53 (health-based DNS failover)
>     │
>     ▼
> CloudFront (CDN + WAF + DDoS)
>     │
>     ▼
> ALB (regional, TLS, L7 routing) ← AWS WAF
>     │
>     ▼
> EKS (3 AZs)
> ├── HPA scales pods on CPU + RPS
> ├── KEDA scales workers on SQS depth
> ├── Karpenter scales nodes (Spot + on-demand mix)
> ├── PDBs protect availability during scaling
> └── topologySpreadConstraints distribute across AZs
>     │
>     ▼
> RDS Aurora (Multi-AZ, read replica auto-scale)
> ElastiCache (cluster mode)
> S3 (lifecycle policies)
> ```
>
> **The principles I'd emphasize in the interview:**
> 1. **Every tier auto-scales independently** — pod, node, replica, capacity.
> 2. **Health checks gate scaling decisions** — readiness probes that test actual readiness, not just process-up.
> 3. **PDBs and graceful shutdown** so scale-down doesn't drop requests.
> 4. **Observability per layer** to diagnose where scaling didn't happen when expected.
> 5. **Cost-aware scaling** — Spot for stateless, savings plans for baseline, consolidation enabled."

---

## 10. Validating image vulnerability in a registry

> "Image vulnerability scanning happens at multiple stages — build-time, registry-side, and admission-time. Going through each layer:
>
> **Build-time scanning (in CI):**
>
> Trivy is what I default to — open-source, fast, multi-format support.
>
> ```yaml
> # GitHub Actions
> - name: Build image
>   run: docker build -t myapp:${{ env.TAG }} .
>
> - name: Trivy scan
>   uses: aquasecurity/trivy-action@master
>   with:
>     image-ref: myapp:${{ env.TAG }}
>     severity: CRITICAL,HIGH
>     exit-code: 1                    # fail build on findings
>     ignore-unfixed: true            # don't fail on CVEs without a fix
>     format: sarif                   # for GitHub Security tab
>     output: trivy-results.sarif
>
> - name: Upload to GitHub Security
>   uses: github/codeql-action/upload-sarif@v3
>   if: always()
>   with: { sarif_file: trivy-results.sarif }
> ```
>
> What this scans:
> - **OS package CVEs** — apt/apk/yum packages with known vulnerabilities.
> - **Language dependencies** — npm, pip, go modules, Maven dependencies.
> - **Misconfigurations** — running as root, world-writable files.
> - **Secrets** — accidentally embedded keys or tokens.
>
> Other build-time scanners I've used:
> - **Snyk Container** — commercial, deeper analysis, includes runtime context.
> - **Grype** — Anchore's open-source scanner.
> - **Docker Scout** — Docker's native tool.
>
> **Registry-side scanning:**
>
> Images get scanned again when pushed to the registry. Even if build-time was clean, new CVEs are disclosed daily — registry scanning catches CVEs that emerged after build.
>
> **ECR scanning options:**
> - **Basic scanning (free)** — scans on push, uses Clair CVE database.
> - **Enhanced scanning (paid)** — uses Amazon Inspector, continuous scanning of stored images, deeper analysis including language-package CVEs.
>
> ```hcl
> resource "aws_ecr_repository" "app" {
>   name                 = "myapp"
>   image_tag_mutability = "IMMUTABLE"
>
>   image_scanning_configuration {
>     scan_on_push = true
>   }
>
>   encryption_configuration {
>     encryption_type = "KMS"
>     kms_key         = aws_kms_key.ecr.arn
>   }
> }
>
> # Enhanced scanning at the registry level
> resource "aws_ecr_registry_scanning_configuration" "this" {
>   scan_type = "ENHANCED"
>   rule {
>     scan_frequency = "CONTINUOUS_SCAN"
>     repository_filter {
>       filter      = "*"
>       filter_type = "WILDCARD"
>     }
>   }
> }
> ```
>
> **Admission-time verification:**
>
> Beyond just scanning — preventing vulnerable or unsigned images from running.
>
> **Image signing with Cosign:**
> ```bash
> # Sign at build time
> cosign sign --yes 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
>
> # Verify at admission via Kyverno
> ```
>
> ```yaml
> apiVersion: kyverno.io/v1
> kind: ClusterPolicy
> metadata: { name: verify-image-signature }
> spec:
>   validationFailureAction: Enforce
>   rules:
>   - name: verify-signature
>     match:
>       any:
>       - resources: { kinds: [Pod] }
>     verifyImages:
>     - imageReferences:
>       - "123456789012.dkr.ecr.us-east-1.amazonaws.com/*"
>       attestors:
>       - count: 1
>         entries:
>         - keys:
>             publicKeys: |-
>               -----BEGIN PUBLIC KEY-----
>               ...
>               -----END PUBLIC KEY-----
> ```
>
> Unsigned images don't run. CVE-blocked images don't run.
>
> **Continuous scanning in production:**
>
> - **AWS Inspector** continuously scans running images on EC2 and in ECR — finds CVEs disclosed after deploy.
> - **Defender for Containers** (Azure) for AKS.
> - **Falco** for runtime anomaly detection — catches behaviors that scanners miss.
>
> **Checking status of a specific image:**
>
> ```bash
> # Get scan results from ECR
> aws ecr describe-image-scan-findings \
>   --repository-name myapp \
>   --image-id imageTag=1.4.2
>
> # Sample output:
> # imageScanFindings.findings[].name: CVE-2024-12345
> # imageScanFindings.findings[].severity: CRITICAL
>
> # Or list by severity
> aws ecr describe-image-scan-findings \
>   --repository-name myapp \
>   --image-id imageTag=1.4.2 \
>   --query 'imageScanFindings.findings[?severity==`CRITICAL`]'
> ```
>
> **The pipeline-wide pattern I run:**
>
> 1. Build image → Trivy scan in CI → fail on critical/high with fix.
> 2. Push to ECR → automatic scan on push (basic) + continuous enhanced scanning.
> 3. Sign image with Cosign.
> 4. Admission controller (Kyverno) blocks unsigned images and images with active critical CVEs.
> 5. AWS Inspector / Falco for runtime detection.
>
> **Architect's framing:**
> Vulnerability scanning isn't one check — it's a continuous discipline across the image lifecycle. Build-time catches what's known at build time; registry-side catches what's newly disclosed; admission-time enforces policy; runtime catches what scanners miss. The principle: **shift left, but don't only shift left** — left-only means the moment something is built, it's never re-evaluated, which is the opposite of what you want for security."

---

## 11. Integrating Trivy with AWS

> "Trivy integrates with AWS at several layers depending on what you're scanning.
>
> **1. Scanning images in ECR**
>
> Trivy can scan ECR images directly without pulling them locally:
>
> ```bash
> # Authenticate Docker to ECR (one-time per session)
> aws ecr get-login-password --region us-east-1 | \
>   docker login --username AWS --password-stdin \
>   123456789012.dkr.ecr.us-east-1.amazonaws.com
>
> # Scan the image
> trivy image \
>   --severity CRITICAL,HIGH \
>   --exit-code 1 \
>   --ignore-unfixed \
>   123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
> ```
>
> Trivy reads ECR credentials from the local Docker config — same auth flow as `docker pull`.
>
> **2. Trivy in GitHub Actions with AWS OIDC**
>
> The CI integration:
>
> ```yaml
> permissions:
>   id-token: write
>   contents: read
>
> jobs:
>   scan:
>     runs-on: ubuntu-latest
>     steps:
>     - uses: actions/checkout@v4
>
>     - name: Configure AWS via OIDC
>       uses: aws-actions/configure-aws-credentials@v4
>       with:
>         role-to-assume: arn:aws:iam::123456789012:role/gha-scan
>         aws-region: us-east-1
>
>     - name: Login to ECR
>       uses: aws-actions/amazon-ecr-login@v2
>
>     - name: Trivy scan
>       uses: aquasecurity/trivy-action@master
>       with:
>         image-ref: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ env.TAG }}
>         severity: CRITICAL,HIGH
>         exit-code: 1
>         ignore-unfixed: true
>         format: sarif
>         output: trivy-results.sarif
>
>     - name: Upload findings to AWS Security Hub
>       run: |
>         python scripts/sarif-to-securityhub.py trivy-results.sarif
>       env:
>         AWS_REGION: us-east-1
> ```
>
> **3. Trivy with AWS Security Hub**
>
> Trivy findings can be pushed to AWS Security Hub for centralized security posture:
>
> ```bash
> # Output Trivy results in ASFF format (Security Hub's native format)
> trivy image --format asff --output findings.json myapp:1.4.2
>
> # Push to Security Hub
> aws securityhub batch-import-findings \
>   --findings file://findings.json \
>   --region us-east-1
> ```
>
> All findings now visible in the Security Hub console alongside Inspector, GuardDuty, and other AWS security service findings.
>
> **4. Trivy Operator on EKS**
>
> For continuous scanning of running workloads in the cluster:
>
> ```bash
> helm repo add aqua https://aquasecurity.github.io/helm-charts/
> helm install trivy-operator aqua/trivy-operator \
>   --namespace trivy-system \
>   --create-namespace \
>   --set trivy.ignoreUnfixed=true \
>   --set serviceMonitor.enabled=true
> ```
>
> Trivy Operator does:
> - Scans all running pod images.
> - Scans Kubernetes manifests for misconfigurations.
> - Scans RBAC configurations for excessive permissions.
> - Exposes results as Custom Resources (`VulnerabilityReport`, `ConfigAuditReport`, `RbacAssessmentReport`).
> - Metrics exposed for Prometheus, alerts on findings.
>
> ```bash
> # Check findings
> kubectl get vulnerabilityreports -A
> kubectl describe vulnerabilityreport <report> -n <ns>
>
> # Visualize in Grafana via Trivy Operator dashboards
> ```
>
> **5. Trivy scanning IaC for AWS resources**
>
> Trivy isn't just for containers — it scans Terraform, CloudFormation, and other IaC for security misconfigurations:
>
> ```bash
> trivy config ./terraform
> # Catches: S3 buckets without encryption, security groups with 0.0.0.0/0,
> # IAM policies with overly broad permissions, etc.
> ```
>
> In CI:
>
> ```yaml
> - name: Trivy IaC scan
>   uses: aquasecurity/trivy-action@master
>   with:
>     scan-type: config
>     scan-ref: ./terraform
>     severity: CRITICAL,HIGH
>     exit-code: 1
> ```
>
> **6. Integration with Amazon Inspector**
>
> Amazon Inspector and Trivy can complement each other:
> - **Inspector** — continuous scanning, AWS-native, integrates deeply with ECR/Lambda.
> - **Trivy** — open-source, runs in any CI, scans more than containers (IaC, filesystems, secrets).
>
> I typically use both: Trivy for build-time gating (because it's faster and more configurable in CI), Inspector for continuous coverage of stored/running images.
>
> **The complete pipeline integration:**
>
> ```
> CI: Trivy scans → fail-fast on critical/high
>       │
>       ▼
> Push to ECR → Inspector enhanced scanning kicks in
>       │
>       ▼
> Deploy to EKS → Trivy Operator continuously scans running images
>       │                                       │
>       ▼                                       ▼
> Findings → AWS Security Hub (central view)    Prometheus → Grafana → Slack alerts
> ```
>
> **Architect's framing:**
> Trivy's value with AWS is that it gives you a consistent scanner across CI, the registry, and the running cluster — single tool, single CVE database, single set of rules. The integration with AWS is primarily through ECR auth (OIDC) and Security Hub for centralized findings. For an AWS-heavy shop, you can complement Trivy with Inspector for continuous registry scanning, but you can also run Trivy-only and feed everything to Security Hub for a unified view."

---

## 12. Syncing repository changes between production and DR on Roman (assuming this means an ArgoCD/GitOps setup)

I'll interpret this as: prod and DR clusters in different regions, both watching the same Git repository for manifests, and the question is how to keep them deploying in sync.

> "Two parts: how the repository sync works, and how the resulting cluster state stays consistent. Going through both:
>
> **The core pattern: same Git repo, both clusters reconcile**
>
> Both ArgoCD instances (one in prod-primary, one in prod-DR) watch the same GitOps repo. Any commit triggers reconciliation in both clusters.
>
> ```
>                    GitOps repo (single source of truth)
>                              │
>                              ├─→ ArgoCD in prod-primary (us-east-1)
>                              │      └─→ EKS cluster reconciles
>                              │
>                              └─→ ArgoCD in prod-DR (us-west-2)
>                                     └─→ EKS cluster reconciles
> ```
>
> **Implementation: ArgoCD ApplicationSet for fan-out**
>
> One Application definition fans out to both clusters:
>
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: ApplicationSet
> metadata:
>   name: myapp-prod-all
>   namespace: argocd
> spec:
>   generators:
>   - list:
>       elements:
>       - cluster: prod-primary
>         url: https://prod-primary-api.example.com
>         valuesFile: values-prod-primary.yaml
>       - cluster: prod-dr
>         url: https://prod-dr-api.example.com
>         valuesFile: values-prod-dr.yaml
>
>   template:
>     metadata:
>       name: 'myapp-{{cluster}}'
>     spec:
>       project: production
>       source:
>         repoURL: https://github.com/company/gitops
>         targetRevision: main
>         path: apps/myapp
>         helm:
>           valueFiles:
>           - '../../environments/{{cluster}}/{{valuesFile}}'
>       destination:
>         server: '{{url}}'
>         namespace: myapp
>       syncPolicy:
>         automated:
>           prune: true
>           selfHeal: true
>         retry:
>           limit: 5
>           backoff:
>             duration: 5s
>             factor: 2
>             maxDuration: 3m
> ```
>
> One commit bumping the image tag → both clusters get the same version, applied within seconds of each other.
>
> **For tighter promotion control:**
>
> Sometimes you don't want truly simultaneous — you want prod-primary to bake for a period before prod-DR follows. Pattern:
>
> ```yaml
> # ApplicationSet with progressive sync
> spec:
>   strategy:
>     type: RollingSync
>     rollingSync:
>       steps:
>       - matchExpressions:
>         - key: cluster
>           operator: In
>           values: [prod-primary]
>       - matchExpressions:
>         - key: cluster
>           operator: In
>           values: [prod-dr]
> ```
>
> Sync waves: prod-primary first; only after it's `Healthy`, then prod-DR. If prod-primary fails, prod-DR doesn't get the update — DR remains a known-good fallback.
>
> **Repository structure that supports this:**
>
> ```
> gitops-repo/
> ├── apps/
> │   └── myapp/
> │       ├── Chart.yaml
> │       ├── values.yaml              ← defaults
> │       └── templates/
> ├── environments/
> │   ├── prod-primary/
> │   │   └── values-prod-primary.yaml ← region-specific overrides
> │   └── prod-dr/
> │       └── values-prod-dr.yaml      ← region-specific overrides
> └── applicationsets/
>     └── myapp.yaml
> ```
>
> Region-specific values files contain things like region-specific endpoints, DR-mode toggles, replica counts (DR runs at lower capacity until activated).
>
> **CI commits to GitOps repo from the build pipeline:**
>
> ```yaml
> # GitHub Actions step after image push to ECR
> - name: Bump tag in BOTH prod values files
>   run: |
>     git clone https://github.com/company/gitops
>     cd gitops
>     yq -i '.image.tag = "${{ env.IMAGE_TAG }}"' environments/prod-primary/values-prod-primary.yaml
>     yq -i '.image.tag = "${{ env.IMAGE_TAG }}"' environments/prod-dr/values-prod-dr.yaml
>     git add . && git commit -m "chore(prod): bump myapp to ${{ env.IMAGE_TAG }}"
>     git push
> ```
>
> One commit, both regions updated. ApplicationSet picks it up, both ArgoCDs sync.
>
> **Multi-region ArgoCD topology choices:**
>
> Two patterns:
>
> **Pattern A: Hub-and-spoke (central ArgoCD)**
> - One ArgoCD instance (in a shared services account or in prod-primary) manages both clusters as remote destinations.
> - Single pane of glass.
> - Downside: ArgoCD itself is a SPoF; if prod-primary goes down, ArgoCD loses ability to reconcile DR.
>
> **Pattern B: Per-cluster ArgoCD (my preference)**
> - ArgoCD running in each cluster, each managing only its local cluster.
> - Both ArgoCDs watch the same Git repo.
> - DR ArgoCD is independent; prod outage doesn't impair DR reconciliation.
> - Downside: two ArgoCDs to operate, but the operational cost is low.
>
> For a true DR setup, Pattern B is what I'd run.
>
> **What about ArgoCD's own configuration syncing?**
>
> ArgoCD itself, its projects, ApplicationSets, and RBAC are also defined in Git via the "App of Apps" pattern:
>
> ```yaml
> # The root Application in each ArgoCD installation
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata: { name: bootstrap, namespace: argocd }
> spec:
>   source:
>     repoURL: https://github.com/company/gitops
>     path: bootstrap
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: argocd
>   syncPolicy:
>     automated: { selfHeal: true, prune: true }
> ```
>
> Both ArgoCDs bootstrap from the same Git path. ArgoCD configurations stay in sync the same way applications do.
>
> **Data-tier sync (the part Git can't help with):**
>
> Code parity is a solved problem with GitOps. Data sync is the harder problem and requires cloud-native replication:
> - **Aurora Global Database** — primary in us-east-1, secondary in us-west-2, sub-second replication.
> - **DynamoDB Global Tables** — multi-region active-active.
> - **S3 cross-region replication** with RTC.
> - **ElastiCache Global Datastore**.
> - **Secrets Manager multi-region secrets**.
> - **KMS multi-region keys**.
>
> Without this, code parity is meaningless — your DR cluster runs the same code but against empty databases.
>
> **Validation and drift:**
>
> - **ArgoCD's `selfHeal: true`** means any manual change to either cluster is reverted to match Git.
> - **Periodic diff job** comparing cluster state between regions:
>   ```bash
>   diff <(kustomize build apps/myapp --enable-helm) <(kustomize build apps/myapp --enable-helm)
>   ```
> - **Drift alerts** in Prometheus: `argocd_app_info{sync_status="OutOfSync"}` per cluster.
>
> **DR game days that actually test this:**
>
> Code parity is theoretical until you fail over and run real traffic. Quarterly game days:
> 1. Cut traffic from primary to DR via Route 53.
> 2. Run for 4 hours under real load.
> 3. Measure RTO actual vs target.
> 4. Fail back to primary.
> 5. Document every issue, fix them.
>
> **Architect's framing:**
> The actual sync mechanism is simple — one Git repo, two ArgoCDs, ApplicationSet fans out. The complexity is in everything around it: progressive sync ordering for safety, data-tier replication, drift detection, and the operational discipline of game days. The principle: **Git is the source of truth, and the truth is the same for both regions; cloud-native replication handles data**. Without both halves, 'prod and DR in sync' is wishful thinking."

---

That covers all 12 questions with senior-level depth. Likely follow-ups to be ready for:

- **"Walk me through a real DR failover you led."** — have a structured 5-7 min story ready.
- **"How do you handle ArgoCD-managed clusters when ArgoCD itself needs upgrading?"** — separate management cluster, or staged upgrades with health-check gates.
- **"What's the difference between Karpenter and Cluster Autoscaler — when would you pick which?"** — Karpenter for flexibility and speed, CA for simplicity and node-group consistency.
- **"How do you handle stateful workloads in a multi-region GitOps setup?"** — operators (Strimzi, ECK), per-region state, replication coordinated separately from Git.

