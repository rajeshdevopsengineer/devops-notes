# Atlassian DevOps Architect — Complete Interview Walkthrough

This is a senior architect interview, so the answers go deeper into trade-offs, design judgement, and operational nuance.

---

# 🎯 Round 1 — Infra, Kubernetes, and Cloud Patterns

## 1. Multi-tenant EKS cluster: dev/QA/prod isolation with no noisy neighbors

**First, a critical caveat I'd state explicitly:** "Multi-tenant" across **dev/QA/prod environments** is different from multi-tenant across **teams within an environment**. The architect's choice here matters.

My strong recommendation for **dev/QA/prod**: **separate clusters**, not separate namespaces in one cluster. The blast-radius asymmetry — a dev workload causing a control-plane incident that affects prod — is unacceptable. Atlassian-scale shops run separate EKS clusters per environment, often per region.

That said, the question is "design it," so here's the **single-cluster multi-tenant** design when the constraint is given:

**Isolation layers, from coarsest to finest:**

**1. Namespace isolation**
- Namespace per tenant/environment: `dev-team-a`, `qa-team-a`, `prod-team-a`.
- ResourceQuotas: CPU, memory, pod count, PVC count caps per namespace.
- LimitRanges: default requests/limits for pods without them; min/max boundaries.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: tenant-quota, namespace: prod-team-a }
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "200"
    services.loadbalancers: "5"
```

**2. Compute isolation (the noisy-neighbor killer)**
- Separate **node groups per environment** with taints:
  - `env=prod:NoSchedule` — only prod-tolerating pods land here.
  - `env=dev:NoSchedule` — dev workloads.
- **Karpenter or Cluster Autoscaler** per node group.
- **GuaranteedQoS** for prod (requests = limits) → highest scheduling and eviction priority.
- **Reserved system pool** (taint) for cluster-critical workloads (ingress, CoreDNS, monitoring).
- For hardest isolation: **separate node groups using different instance types and AZ spread** so dev workloads can't physically saturate hardware that prod might burst onto.

**3. PriorityClasses** — explicit eviction order
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: { name: prod-critical }
value: 1000000
globalDefault: false
description: "Production workloads — evicted last"
```
Combined with PodDisruptionBudgets, prod pods are evicted only after dev/QA pods are gone.

**4. Network isolation**
- **NetworkPolicy** with default-deny per namespace.
- **Cilium** (or Calico) for L7 policies — block lateral movement at HTTP method/path level, not just port.
- Separate ingress controllers per environment (different ALBs/NLBs).
- **Egress controls** — prod can reach prod data plane only; dev cannot reach prod databases at all.

**5. RBAC**
- Namespace-scoped Roles, not ClusterRoles, for tenant access.
- Bind to **groups** (OIDC), never users.
- **Kyverno / OPA Gatekeeper** policies enforcing:
  - Required labels (owner, env, cost-center).
  - Image registry allowlist.
  - Resource requests/limits required.
  - No `privileged: true`, no `hostPath` mounts.

**6. Storage isolation**
- Per-environment StorageClasses with different KMS keys.
- Separate snapshot retention per env.

**7. Control plane**
- API server priority and fairness (APF) — protect critical control-plane traffic from a busy tenant.
- Etcd encryption with KMS.
- Audit logging with namespace filtering routed to a separate logging account.

**8. Observability with tenant cardinality**
- Prometheus federation with per-tenant retention.
- Grafana folders + dashboards per tenant.
- Per-tenant alert routes in Alertmanager.

**Sketch of the cluster shape:**
```
EKS cluster (single)
├── System node pool   (taint=system,  cluster-critical only)
├── Prod node pool     (taint=env=prod,  PriorityClass=prod-critical)
├── QA node pool       (taint=env=qa)
└── Dev node pool      (taint=env=dev,   uses Spot instances)
```

**The architect's answer:** "For Atlassian's scale, I'd push back on cross-environment multi-tenancy and run separate clusters per environment. But if the constraint is real — typically cost or operational simplicity — then taint-isolated node groups + PriorityClasses + ResourceQuotas + default-deny NetworkPolicy + admission controllers covers 95% of the noisy-neighbor surface. The remaining 5% — kernel-level resource contention, etcd hot keys, control-plane saturation — can only be solved by physical cluster separation."

## 2. Managing 10+ Kustomize overlays without drift or duplication

The pattern I use is **strict layering with discipline at the base**:

```
manifests/
├── base/                              # shared, environment-agnostic
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── components/                        # reusable optional features
│   ├── istio-injection/
│   ├── prometheus-scrape/
│   └── network-policy-default-deny/
└── overlays/
    ├── _common/                       # all-env shared overlay layer
    │   ├── kustomization.yaml
    │   └── patches/
    ├── dev/
    │   ├── kustomization.yaml         # references _common + dev-specific
    │   └── patches/
    ├── qa/
    ├── staging/
    ├── prod-us-east-1/
    ├── prod-us-west-2/
    ├── prod-eu-west-1/
    └── prod-ap-southeast-1/
```

**Patterns that prevent drift:**

**1. Strict base — no defaults that bleed into prod**
The base defines structure, not values. Replica count, resource requests, ingress hostnames — all overlay-set. Base has placeholders or sensible-but-low defaults that would fail loudly in prod if not overridden.

**2. Common overlay layer between base and env-specific**
The `_common` overlay applies things every environment needs: prometheus annotations, default labels, security context, NetworkPolicy. Env overlays inherit from it.

```yaml
# overlays/prod-us-east-1/kustomization.yaml
resources:
- ../_common
patches:
- path: patches/replicas.yaml
- path: patches/resources.yaml
- path: patches/ingress-hostname.yaml
```

**3. Kustomize Components for cross-cutting features**
Components are mountable feature flags. "Add Istio sidecar injection" is a Component, not duplicated YAML.

```yaml
# overlays/prod-us-east-1/kustomization.yaml
components:
- ../../components/istio-injection
- ../../components/prometheus-scrape
```

**4. JSON Patches over strategic-merge for surgical changes**
For one-line changes (replica count, image tag), use 6902 JSON Patches — explicit, smaller, less ambiguity:
```yaml
patches:
- target: { kind: Deployment, name: myapp }
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 12
```

**5. Generators over hand-written ConfigMaps/Secrets**
`configMapGenerator` and `secretGenerator` produce hashed names → rolling-update happens automatically when config changes. No "I changed the ConfigMap but pods didn't restart" surprises.

**6. CI validation that catches drift early**
On every PR:
```bash
# Validate every overlay builds
for o in overlays/*/; do
  kustomize build "$o" | kubeconform -strict || exit 1
done

# Diff against current cluster (read-only)
for env in prod-us-east-1 prod-us-west-2; do
  kustomize build overlays/$env | kubectl --context=$env diff -f -
done

# Drift detection — what's between overlays?
diff <(kustomize build overlays/prod-us-east-1) <(kustomize build overlays/prod-us-west-2)
```

**7. ArgoCD ApplicationSets** to fan out one definition across N environments
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: myapp-all-envs }
spec:
  generators:
  - list:
      elements:
      - env: prod-us-east-1
        cluster: https://prod-us-east-1.example.com
      - env: prod-us-west-2
        cluster: https://prod-us-west-2.example.com
      # ...
  template:
    metadata: { name: 'myapp-{{env}}' }
    spec:
      source:
        repoURL: https://github.com/company/manifests
        path: 'overlays/{{env}}'
      destination: { server: '{{cluster}}' }
```

**8. Policy at the kustomize boundary**
Conftest / OPA policies that run against the *built* output, not the source. Catches things like "did anyone accidentally remove the resource limits patch in this overlay?"

**The discipline that actually prevents drift:**
- **Code review** — overlays go through PR with the rendered output diffed in CI comment.
- **Drift detection job** — daily comparison of cluster state vs `kustomize build` output. Alert on mismatch.
- **Rotation of overlay-owners** — no single team owns the prod overlay alone; platform team co-reviews.
- **"No env-specific in base"** — code-review rule that gets enforced.

**Alternative I'd mention:** for 10+ environments with complex variation, **Helm with values files** can be cleaner than Kustomize, or a hybrid (Helm-templated base, Kustomize overlays for last-mile environment patches). The "right" choice depends on whether your variation is structural (Kustomize) or value-substitution (Helm).

## 3. Securing cross-region S3 replication and validating data integrity at scale

**Encryption end-to-end:**
- Source bucket with SSE-KMS using a customer-managed key (CMK) in the source region.
- Destination bucket with SSE-KMS using a separate CMK in the destination region.
- **Replication configuration with `encryption_configuration.replica_kms_key_id`** so replicated objects are re-encrypted with the dest CMK.
- KMS key policies in both regions grant the replication role decrypt-on-source and encrypt-on-destination.
- Bucket policies require `aws:SecureTransport=true` for all access.

**IAM scoping:**
- Replication role has minimum permissions: `s3:GetReplicationConfiguration`, `s3:ListBucket` on source; `s3:GetObjectVersion*` on source objects; `s3:ReplicateObject`, `s3:ReplicateDelete`, `s3:ReplicateTags` on destination.
- Trust policy limits to `s3.amazonaws.com`.

**Access control:**
- Both buckets have **Block Public Access** enabled.
- Bucket policies enforce `bucket-owner-full-control` ownership on replicated objects via `access_control_translation`.
- **S3 Object Ownership** set to "Bucket owner enforced" (ACLs disabled).
- Cross-account replication uses explicit principal grants in the destination bucket policy.

**Network:**
- For private connectivity, **S3 Gateway Endpoints** in source and destination VPCs.
- For traffic that must traverse internet, AWS uses its private backbone for cross-region S3 replication — never goes over the public internet anyway.

**Immutability and ransomware protection:**
- **S3 Object Lock** in compliance mode on both buckets for regulated data.
- **Versioning** mandatory (required by replication anyway).
- **MFA delete** on the destination bucket.
- **Replicate to a separate account** owned by a different trust boundary — even if the source account is compromised, the destination is unreachable.

**Replication time control:**
- **S3 RTC (Replication Time Control)** for SLA-backed 15-minute replication — and the metrics it surfaces (`ReplicationLatency`, `BytesPendingReplication`, `OperationsPendingReplication`) are how you monitor.

**Data integrity validation at scale:**

This is the operationally hard part. Several layers:

**1. ETag and checksum verification**
- S3 returns an ETag (MD5 for non-multipart, composite for multipart). Replication preserves it.
- **Additional checksums** (CRC32C, CRC32, SHA-1, SHA-256) configurable on PutObject; replication preserves these. Use SHA-256 for crypto-strength integrity.
- After replication, compare ETag/checksum between source and destination via head-object.

**2. S3 Inventory + Athena**
- Enable S3 Inventory on both buckets (daily/weekly CSV reports of all objects).
- Athena query joins source and destination inventories on key, compares size + ETag + checksum + storage class.
- Anomalies flagged into a CloudWatch alarm or SNS.

```sql
-- Athena: find replication mismatches
SELECT s.key, s.size as src_size, d.size as dst_size, s.etag as src_etag, d.etag as dst_etag
FROM source_inventory s
LEFT JOIN dest_inventory d ON s.key = d.key
WHERE d.key IS NULL                        -- missing from destination
   OR s.size <> d.size                     -- size mismatch
   OR s.etag <> d.etag                     -- content mismatch
   OR s.last_modified > d.last_modified + interval '15' minute;  -- replication lag
```

**3. CloudWatch metrics on replication**
- `ReplicationLatency` per replication rule.
- `BytesPendingReplication` — should trend to zero.
- `OperationsFailedReplication` — alert on any non-zero value.

**4. EventBridge events for replication failures**
- S3 emits `s3:Replication:OperationFailedReplication` events. Route to SNS/Lambda for immediate visibility.

**5. Periodic sampling reconciliation**
- For very large buckets, full-inventory comparison is expensive. Sample 0.1% of objects daily, run full integrity check including range-byte SHA-256.

**6. Batch operations for repair**
- When mismatches are detected, **S3 Batch Replication** can re-replicate specific objects from a manifest CSV. Don't manually `aws s3 cp` at petabyte scale.

**Cost note worth mentioning:**
Replication storage, requests, and KMS calls add up at scale. For non-critical buckets, **One Zone-IA or Glacier Instant Retrieval** on the destination cuts cost ~40%. RTC is more expensive but worth it for tier-1 data.

## 4. systemd failing unit on a containerized node — auto-recovery

On EKS/GKE nodes, the "containerized node" still has systemd on the host managing services like `kubelet`, `containerd`/`docker`, `aws-node`, `kube-proxy`. A failing systemd unit on the node ≠ a failing pod — it's an infrastructure-level failure.

**What happens by default:**
- systemd respects the unit's `Restart=` directive. If absent or set to `no`, the unit stays failed.
- The kubelet, containerd, and other critical units typically have `Restart=always` and `RestartSec=` set, but with a finite `StartLimitBurst` — too many fast failures and systemd gives up.
- A failed kubelet means the node stops reporting status → control plane marks node `NotReady` → pods are evicted after `pod-eviction-timeout` (default 5min) → other nodes absorb the load.

**Auto-recovery layered design:**

**1. systemd-level restart with backoff**
```ini
# /etc/systemd/system/kubelet.service.d/override.conf
[Service]
Restart=always
RestartSec=5
StartLimitBurst=10
StartLimitIntervalSec=300
# 10 restarts allowed per 5 minutes; beyond that, escalate
```

**2. OnFailure unit chains** — trigger remediation when a unit fails
```ini
[Unit]
OnFailure=node-remediate@%n.service
```

```ini
# /etc/systemd/system/node-remediate@.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/remediate.sh %i
```

The script can: dump diagnostics to S3, attempt service restart, or cordon the node.

**3. Self-cordoning / self-termination**
A failing kubelet that won't recover should remove itself from service:
```bash
# remediate.sh
SERVICE=$1
journalctl -u "$SERVICE" -n 200 | aws s3 cp - s3://node-diagnostics/$INSTANCE_ID-$SERVICE.log
# If kubelet has failed irrecoverably:
if [ "$SERVICE" = "kubelet.service" ]; then
  aws autoscaling set-instance-health \
    --instance-id "$INSTANCE_ID" \
    --health-status Unhealthy
fi
```
The ASG terminates the unhealthy instance and launches a replacement.

**4. Health checks at the ASG level**
ASG with **ELB health checks** (if nodes are behind an NLB) or **EC2 status checks** — failed instance gets terminated and replaced automatically.

**5. Node Problem Detector (NPD)** — Kubernetes-native
Runs as DaemonSet, monitors host conditions:
- Kernel issues (kernel deadlock, OOM events).
- Container runtime issues.
- systemd unit failures (with the `system-stats-monitor` plugin).
- Exposes problems as Node Conditions in the K8s API.

```yaml
apiVersion: v1
kind: Node
status:
  conditions:
  - type: KubeletUnhealthy
    status: "True"
    reason: KubeletProblem
```

**6. Auto-remediation operators**
- **Medik8s / NHC (Node Health Check)** — when an NPD condition fires, automatically cordon, drain, and replace the node.
- **AWS Node Termination Handler** — handles spot interruptions + scheduled maintenance.
- **Karpenter consolidation** — replaces underperforming nodes opportunistically.

**The architect's framing:**
> "There's no single answer — auto-recovery is a layered defense. systemd handles transient failures via Restart=. Persistent failures escalate via OnFailure to self-termination via ASG health status. Node Problem Detector + a remediation operator handle higher-level conditions like kernel deadlocks. And the cluster autoscaler / Karpenter ensures replacement capacity is provisioned. The principle is: **a node should never linger in a broken state**. If it can't self-heal in 60 seconds, it should self-terminate."

## 5. Detecting and mitigating pod-to-pod lateral movement

This is a security architecture question. The threat model: a compromised pod (RCE, supply-chain attack, malicious dep) attempting to reach other pods or cluster services it shouldn't.

**Defense in depth, from prevention to detection:**

**1. Default-deny NetworkPolicy in every namespace** (prevention)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: app }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
# No rules = deny all
```

Then explicit allows only for required flows. CNI must enforce — Calico, Cilium, or Antrea (default AWS VPC CNI doesn't enforce on its own).

**2. Cilium for L7 policies** (prevention with depth)
Standard NetworkPolicy is L3/L4 — TCP port 8080 between labeled pods. Cilium extends to L7: only `GET /api/v1/users` is allowed, `POST /admin/*` is denied. Much harder to abuse a legitimate flow for malicious purposes.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: frontend-to-backend }
spec:
  endpointSelector: { matchLabels: { app: backend } }
  ingress:
  - fromEndpoints: [{ matchLabels: { app: frontend } }]
    toPorts:
    - ports: [{ port: "8080", protocol: TCP }]
      rules:
        http:
        - method: GET
          path: /api/v1/.*
```

**3. Service mesh with mTLS** (prevention)
Istio or Linkerd enforces mTLS for inter-pod traffic. A rogue pod can't impersonate another service without the right cert (issued via SPIFFE/SPIRE).

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: default, namespace: app }
spec:
  mtls: { mode: STRICT }
```

Combine with **AuthorizationPolicy** for identity-based access control beyond network reachability.

**4. Workload identity** (prevention)
- IRSA on EKS, Workload Identity on GKE — pod identity bound to specific service accounts.
- API access requires identity match; stealing a network address gives the attacker nothing without a token.

**5. Pod Security Standards (PSS)** (prevention)
Enforce `restricted` profile:
- No `privileged: true`.
- No `hostNetwork: true`, `hostPID: true`, `hostIPC: true`.
- No `hostPath` volumes.
- Drop all Linux capabilities.
- `readOnlyRootFilesystem: true`.

Without these, a pod can break out of the pod sandbox or read sensitive node-level data.

**6. Egress controls** (prevention + detection)
- All external egress through a controlled egress proxy (Squid, Cilium Egress Gateway).
- DNS resolution restricted (NodeLocal DNS + firewall).
- AWS VPC egress to a NAT Gateway with VPC Flow Logs analyzed for anomalies.

**7. Runtime detection — Falco** (detection)
Falco monitors syscalls and detects suspicious behavior in real time:
```yaml
- rule: Pod-to-Pod Lateral Movement
  desc: Detect pod opening connection to non-standard internal endpoint
  condition: >
    container and
    evt.type=connect and
    fd.sip in (internal_ranges) and
    not allowed_internal_connections
  output: "Lateral movement attempt %container.id %container.image"
  priority: WARNING
```

Falco also catches:
- Shell spawned inside a container.
- Reads of `/etc/shadow` or sensitive paths.
- Outbound network from a container that shouldn't have any.
- Crypto-mining patterns.

**8. eBPF-based observability — Cilium Hubble or Tetragon** (detection + forensics)
- Hubble: real-time flow visibility with policy verdict.
- Tetragon: kernel-level process and network event tracing per container.

Lets you ask: "Did pod X make any connections to pod Y in the last hour?" — instantly, with policy verdict (allowed / denied / dropped).

**9. Audit log analysis** (detection)
- Kubernetes audit logs shipped to a SIEM.
- Detection rules for: ServiceAccount token use from unexpected pods, mass listing of secrets, escalation attempts.

**10. Cluster admission policies** (prevention)
Kyverno or OPA Gatekeeper blocking at admission:
- Images must come from approved registries.
- Pods must declare resource limits.
- ServiceAccount tokens must not auto-mount unless required.
- No pod can request `hostNetwork`.

**11. Network segmentation by trust zone**
- Production prod namespaces in dedicated node pools.
- Data tier pods in a separate node pool with tighter NetworkPolicy.
- Edge/web tier physically separated from data tier nodes.

**12. Threat-modeling exercise at design**
For each new service: what does it need to talk to? Anything else is blocked. Document in NetworkPolicy. Reviewed at PR time, audited monthly.

**Mitigation when detected:**

1. **Automatic isolation** — Falco webhook triggers a controller that adds a `quarantine: true` label, which a NetworkPolicy targets to drop all traffic to/from the pod.
2. **Snapshot for forensics** — capture memory and disk before terminating; CRI image snapshots, /proc dumps.
3. **Terminate the pod** with `kubectl delete` (Deployment respawns; the new pod is the same image — only useful if you've also blocked the image).
4. **Block the compromised image** at the registry; admission controller refuses new instances.
5. **Rotate any credentials** the pod could have accessed — service account tokens, KMS keys, IRSA roles.

**The architect's framing:**
> "Lateral movement detection has to assume prevention will sometimes fail. So we layer: zero-trust at the network (NetworkPolicy + mTLS), identity at the workload (workload identity, not network position, gates access), runtime detection (Falco/Tetragon) for behavioral anomalies, and audit-log analytics for post-hoc forensics. The goal isn't to make compromise impossible — it's to make blast radius small and detection fast."

## 6. Zero-downtime upgrade of a stateful workload with Helm 3

The honest first answer: "Stateful workloads on Kubernetes are operationally expensive. For most production data tiers, I prefer managed services (RDS, Aurora, MSK). But when running stateful in-cluster — Kafka, Elasticsearch, Cassandra — here's the pattern."

**The Helm part is easy; the data-safety part is hard.**

**1. Pre-flight: backup + verify backup**
- Snapshot via Velero, CSI snapshots, or the storage system's native tool (e.g., `kafka-mirror-maker` to a parallel cluster).
- Restore the snapshot to a scratch cluster and verify integrity. **An unverified backup is no backup.**

**2. Pre-flight: protocol compatibility check**
- Read release notes for breaking changes.
- For Kafka: brokers can speak older protocol versions. Set `inter.broker.protocol.version` and `log.message.format.version` to the *old* version during the rolling upgrade — upgrade brokers one by one — then bump these settings only after all brokers are on the new version.
- For Elasticsearch: respect the upgrade path (no skipping minor versions).
- For Cassandra: respect Java version, sstable format, gossip protocol versions.

**3. The Helm upgrade itself**

StatefulSets with `updateStrategy: RollingUpdate` and `partition: N` give surgical control:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 3      # Only pods with ordinal >= 3 are updated
```

Start with partition = N (don't update anyone), then progressively reduce:

```bash
# Pin Helm chart values to control rollout pace
helm upgrade kafka ./chart \
  -f values-prod.yaml \
  --set updateStrategy.rollingUpdate.partition=2 \
  --wait --timeout=10m

# Validate the canary (broker 2)
kubectl exec kafka-2 -- kafka-broker-api-versions --bootstrap-server kafka-2:9092
# Run consumer/producer integration tests
# Watch metrics for replication lag, request errors

# Continue rolling
helm upgrade kafka ./chart --set updateStrategy.rollingUpdate.partition=1 ...
helm upgrade kafka ./chart --set updateStrategy.rollingUpdate.partition=0 ...
```

**4. Per-pod safety**
- **PodDisruptionBudget**: `minAvailable: N-1` so you always have quorum.
- **preStop hook**: graceful shutdown — for Kafka, demote leader replicas off this broker before stopping.
- **terminationGracePeriodSeconds**: long enough for graceful shutdown (Kafka, Elasticsearch can take minutes).
- **Readiness probe**: should only pass after the workload is *fully* caught up — not just "process is running."

**5. Coordinated updates**

For very stateful systems, Helm-rolled StatefulSet isn't enough. Use an **Operator** that understands the data plane:
- **Strimzi** (Kafka), **ECK** (Elasticsearch), **K8ssandra** (Cassandra) — coordinate cluster state during upgrades, demote leaders, wait for replication.
- The Helm chart manages the operator; the operator manages the data plane.

**6. PVC handling**
- `volumeClaimTemplates` in StatefulSet — PVCs are retained across pod restarts/upgrades.
- `persistentVolumeReclaimPolicy: Retain` on the StorageClass — accidental PVC deletion doesn't lose data.
- Helm's `helm upgrade` won't touch existing PVCs unless you explicitly tell it to. Verify in `--dry-run` output.

**7. Observability during upgrade**
- Real-time dashboards: replication lag, leader changes, consumer group lag, query latency p99.
- Alerts at lower thresholds during upgrades (`alert if lag > 5s` instead of normal `> 30s`).

**8. Roll-forward, not roll-back, where possible**
- Stateful rollbacks are dangerous — older versions may not read newer storage formats.
- Plan upgrades that allow roll-forward if issues are caught early.

**9. Helm 3-specific notes**
- `--atomic` rolls back the entire Helm release on failure (useful for stateless, risky for stateful).
- `helm rollback` reverts manifests but doesn't restore data — make this explicit in runbooks.
- `helm history` for audit trail.
- `helm diff upgrade` (plugin) before applying — non-negotiable in prod.

**The architect's framing:**
> "Zero-downtime for stateful workloads is less a Helm problem and more a data-plane coordination problem. Helm gives me the StatefulSet rollout mechanics — partition, PDB, probes. The real work is in protocol compatibility, leader demotion, replication-lag-aware probes, and pre-verified backups. For anything tier-1, I'd use an operator that encapsulates this knowledge — Strimzi for Kafka, ECK for ES — and treat the Helm chart as the way to deploy the operator, not the data plane itself."

## 7. Hybrid cloud routing — GCP and AWS

**Connectivity layer:**

Two patterns by maturity and need:

**Pattern A: VPN — fast to deploy, lower SLA**
- Site-to-site IPsec VPN between AWS Transit Gateway and GCP Cloud VPN.
- BGP over VPN for dynamic routing.
- Throughput limited (~1.25 Gbps per tunnel, multiple tunnels for HA).

**Pattern B: Dedicated interconnect — production scale**
- **AWS Direct Connect** + **GCP Cloud Interconnect** terminating in the same colocation facility.
- Routed via a partner like Megaport, Equinix Fabric, or PacketFabric — single MPLS-like backbone.
- Throughput: 10/100 Gbps per circuit, HA via redundant circuits in separate facilities.

**Topology — hub-and-spoke per cloud, peered across:**

```
AWS region:
  Transit Gateway (hub)
  ├── VPC: prod-app
  ├── VPC: prod-data
  ├── VPC: shared-services (DNS, AD, monitoring)
  └── DXGW → AWS Direct Connect → Megaport →
                                              GCP Cloud Router → 
                                                                  GCP Shared VPC (hub)
                                                                  ├── VPC: prod-app
                                                                  ├── VPC: prod-data
                                                                  └── VPC: shared-services
```

BGP advertises networks across; route tables filter what's advertised in each direction.

**DNS:**
- **Route 53 Resolver inbound endpoint** in AWS exposing `*.aws.company.internal` resolvable from GCP.
- **Cloud DNS** in GCP with private zone for `*.gcp.company.internal`.
- Forwarders both ways so workloads in either cloud can resolve services in the other by name.

**Identity:**
- **Workload Identity Federation** — GCP workloads can assume AWS IAM roles via OIDC, and vice versa with Identity Pool.
- No long-lived cross-cloud credentials. The IdP (Entra ID, Okta) federates to both.

**Boundary enforcement — where it matters most for the question:**

**1. At the cloud network edge**
- AWS: Security Groups + NACLs + GWLB for inline inspection of cross-cloud traffic.
- GCP: VPC firewall rules + Cloud Armor + Cloud NGFW.
- Deny-all default, explicit allows for known cross-cloud flows.

**2. At the interconnect**
- BGP route filters — only advertise the CIDRs that *should* be reachable cross-cloud. Don't leak the entire 10.0.0.0/8.
- Separate "shared services" VPCs whose CIDRs are advertised, vs "private" VPCs whose CIDRs are not.

**3. At the service mesh**
- If you have Istio/Anthos/AWS App Mesh spanning both clouds, **mTLS terminates at the mesh boundary**. Cross-cloud requests authenticate via SPIFFE identities, not source IP.
- **East-west gateway** in each cluster handles cross-cluster (and cross-cloud) traffic with explicit policy.

**4. At the application layer**
- Auth tokens validated by the receiving service.
- Service-to-service authorization (not just authentication) — even if a service in GCP can *reach* a service in AWS, it must present a credential indicating it's *allowed* to call that specific endpoint.

**5. At egress/data sovereignty**
- **Egress-only flows for regulated data** — data classified as regulated stays in the cloud/region where it was created.
- Data exfiltration controls: VPC Service Controls (GCP), AWS Verified Access, AWS Network Firewall with FQDN allowlists.

**6. At observability and audit**
- VPC Flow Logs on both sides shipped to a central SIEM (split-account / split-project so neither cloud admin alone can tamper).
- Cross-cloud trace correlation via OpenTelemetry with shared trace ID.

**The architect's framing on where to enforce boundaries:**
> "I enforce at every layer — defense in depth. But the most important boundary is the **identity layer in the service mesh or app-level auth** — not the network. Network-layer enforcement is necessary but insufficient; an attacker who breaches one cloud can pivot via legitimate network paths, so the receiving service must verify the *identity* of the caller, not just trust the source CIDR. Network rules limit attack surface; identity gates limit what can be done across that surface."

## 8. Rebuilding Terraform state after backend migration corruption

The order matters: **stop the bleeding first**, then rebuild.

**1. Halt all writes**
- Disable the CI pipeline that runs `terraform apply`.
- Notify the team — no manual applies.
- Take a lock on the backend (`terraform force-unlock <id>` only if a stale lock is blocking; otherwise leave any active lock alone).

**2. Inventory what you have**
```bash
# What's in each backend?
terraform state pull > /tmp/current-state-broken.tfstate
# Check old backend if accessible
aws s3 ls s3://old-state-bucket/key/
aws s3api list-object-versions --bucket old-state-bucket --prefix prod/terraform.tfstate
# New backend
aws s3api list-object-versions --bucket new-state-bucket --prefix prod/terraform.tfstate
```

**3. Recovery from versioned backups (best case)**
If either backend has versioning enabled (and it should — that's mandatory):
```bash
# Find a known-good version (before migration)
aws s3api list-object-versions \
  --bucket old-state-bucket \
  --prefix prod/terraform.tfstate \
  --query 'Versions[?LastModified < `2025-01-15`]'

# Download last known good
aws s3api get-object \
  --bucket old-state-bucket \
  --key prod/terraform.tfstate \
  --version-id <good-version> \
  /tmp/recovered.tfstate

# Validate it's parseable
jq . /tmp/recovered.tfstate > /dev/null
terraform show -json /tmp/recovered.tfstate | jq '.values.root_module.resources | length'

# Push to the new backend
terraform state push /tmp/recovered.tfstate

# Verify with plan
terraform plan
```

If plan shows minimal/expected drift → done. Re-enable pipeline.

**4. Recovery from local copies**
Anyone on the team who ran `terraform plan` recently has a local cached state in `.terraform/terraform.tfstate.backup` or similar. Check personal laptops and CI runners' last-known-good artifacts.

**5. Full rebuild (worst case — no backup exists)**

This is where the discipline of well-named, well-tagged resources pays off.

```bash
# Bootstrap fresh state
rm -rf .terraform terraform.tfstate*
terraform init

# Bulk import via terraformer
terraformer import aws \
  --resources=vpc,subnet,route_table,security_group,iam_role,s3,rds,eks \
  --regions=us-east-1 \
  --filter="Name=tags.Environment;Value=prod"

# Or use Terraform 1.5+ import blocks
cat > imports.tf <<EOF
import {
  to = aws_vpc.main
  id = "vpc-0abc123def456"
}
import {
  to = aws_subnet.private["a"]
  id = "subnet-0aaa..."
}
# ... for every resource
EOF

terraform plan -generate-config-out=generated.tf
# Review generated.tf, merge into real config
terraform plan
# Iterate until plan = "No changes"
```

For petabyte-scale infra, full rebuild can take days. **terraformer + filtering by tags** is the only practical path.

**6. Validate exhaustively**
- `terraform plan` shows zero changes — that's the contract.
- Run `plan` against every workspace/environment.
- Compare resource counts pre- and post-rebuild against cloud-side queries (`aws resourcegroupstaggingapi get-resources`).

**7. Document the incident and prevent recurrence**

The real interview answer is the prevention list:
- **Versioning + MFA delete** on state buckets — mandatory.
- **DynamoDB locking** (AWS) or equivalent — every backend.
- **Backend migration runbook** — `terraform state pull` before, `terraform state push` after, side-by-side plan verification before any cleanup.
- **No deletion of old state for at least 30 days** after migration.
- **Backup state nightly** to a separate account/region — independent of bucket versioning.
- **State integrity validation** — schedule a daily job: `terraform plan -detailed-exitcode`. Exit code 2 = drift; investigate.
- **Separate state files per environment** — corruption in dev is annoying; corruption in prod is a Sev-1. Don't share state files across environments.
- **Pin Terraform and provider versions** — schema differences between versions can cause state-write corruption.

**The architect's framing:**
> "Corruption during backend migration almost always traces to one of three causes: a partial write because the migration tool was killed mid-run, a schema mismatch from running different TF versions against the same state, or — most commonly — *no backup existed* and a manual edit went wrong. Recovery hinges entirely on whether versioning was enabled. The pre-incident hardening matters more than the recovery procedure: every backend I architect has versioning, locking, encrypted-with-CMK storage in a separate account, and nightly snapshots to a third location."

## 9. Bash one-liner: containers using >500MB RSS

Several valid forms; I'd give a couple to show flexibility:

**Docker:**
```bash
docker stats --no-stream --format "{{.Name}} {{.MemUsage}}" | awk '{split($2,a,"MiB"); if (a[1]+0 > 500) print}'
```

**Cleaner with explicit numeric parsing:**
```bash
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}" | \
  awk 'NR>1 { split($2,a,"MiB|GiB"); val=a[1]; if ($2 ~ /GiB/) val*=1024; if (val>500) print $1, val"MB" }'
```

**containerd / nerdctl (modern EKS nodes use containerd, not Docker):**
```bash
crictl stats --output json | jq -r '.stats[] | select(.memory.usageBytes > 500*1024*1024) | "\(.attributes.metadata.name) \(.memory.usageBytes/1024/1024)MB"'
```

**Generic from cgroup v2 (works for any container runtime on modern kernels):**
```bash
for c in /sys/fs/cgroup/system.slice/docker-*.scope /sys/fs/cgroup/kubepods.slice/*/*/cgroup.procs; do
  [ -d "$c" ] || continue
  mem=$(cat $(dirname $c)/memory.current 2>/dev/null)
  [ -z "$mem" ] && continue
  [ $mem -gt $((500*1024*1024)) ] && echo "$c: $((mem/1024/1024))MB"
done
```

**From `ps` filtering by container PID:**
```bash
ps -e -o pid,rss,cmd --sort=-rss | awk '$2 > 512000 {printf "%s %dMB %s\n", $1, $2/1024, $3}'
```

The one-liner I'd write in the interview is the first `docker stats` form. I'd then mention: "On modern EKS this is `crictl`, not `docker`, because containerd is the runtime."

---

# 🎯 Round 2 — RCA, Fire, and Chaos Control

## 1. ALB TLS handshake failures (intermittent)

Intermittent TLS failures are particularly nasty — they survive obvious testing.

**RCA path:**

**Step 1: Characterize the failure**
- What % of handshakes fail? Constant percentage or bursts?
- Specific client IPs / regions / TLS versions / cipher suites?
- Specific time windows?
- Browser vs CLI client behavior?

Pull from ALB access logs (S3) or CloudWatch metrics:
```sql
-- Athena query against ALB logs
SELECT
  client_port,
  ssl_protocol,
  ssl_cipher,
  COUNT(*) FILTER (WHERE elb_status_code = '-') AS handshake_failed,
  COUNT(*) AS total
FROM alb_logs
WHERE date_trunc('hour', time) >= now() - interval '6' hour
GROUP BY 1,2,3
HAVING handshake_failed > 0
ORDER BY handshake_failed DESC;
```

**Step 2: Hypothesis — typical causes ranked**

1. **Certificate / SNI mismatch** — new listener cert added, but a subset of clients send no SNI or different SNI. Old clients on TLS 1.0/1.1 don't send SNI by default.
2. **Cipher / protocol policy too restrictive** — new ALB security policy dropped legacy ciphers; some clients only support those.
3. **OCSP stapling issues** — ALB attempts OCSP stapling, CA endpoint is slow/down → handshake stalls.
4. **Cert chain issue** — intermediate CA cert missing or in wrong order; some clients can't build chain.
5. **Multiple certs on one ALB with SNI conflict** — added a new cert that overlaps; ALB picks wrong cert for some clients.
6. **MTU / fragmentation** — TLS records >MTU getting fragmented, some path drops fragmented IP.
7. **Cross-zone load balancing imbalance** + one node serving stale config — old listener config still on some LCU.
8. **ACM cert auto-renewal mid-flight** — new cert version not propagated to all LB nodes yet.

**Step 3: Reproduce + isolate**
```bash
# Test TLS 1.0/1.1/1.2/1.3 separately
for v in tls1 tls1_1 tls1_2 tls1_3; do
  echo "=== $v ==="
  for i in {1..20}; do
    openssl s_client -connect alb.example.com:443 -$v -servername alb.example.com \
      -timeout 5 < /dev/null 2>&1 | grep -E "Cipher|Protocol|verify"
  done
done

# Test SNI vs no-SNI
openssl s_client -connect alb.example.com:443 -servername alb.example.com < /dev/null
openssl s_client -connect alb.example.com:443 < /dev/null   # no SNI

# Test from multiple AZs / source IPs (different ALB nodes resolved)
dig +short alb.example.com   # multiple IPs
for ip in $(dig +short alb.example.com); do
  echo "=== $ip ==="
  curl -v --resolve alb.example.com:443:$ip https://alb.example.com/ 2>&1 | grep -E "TLS|SSL|cipher"
done

# Test cipher suite negotiation
nmap --script ssl-enum-ciphers -p 443 alb.example.com

# Check OCSP stapling
openssl s_client -connect alb.example.com:443 -status < /dev/null 2>&1 | grep -A20 "OCSP response"
```

**Step 4: Check the recent ALB change**
- `aws elbv2 describe-listeners` — compare cert ARNs and SSL policy vs previous.
- CloudTrail for the ALB config change: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<alb-arn>`.
- Was the cert in ACM rotated? `aws acm describe-certificate`.
- Was the security policy changed? Compare against `ELBSecurityPolicy-TLS13-1-2-2021-06` vs older policies.

**Step 5: Common root causes I've actually seen**

- **Most likely:** SSL policy upgraded to TLS 1.3-only, breaking older clients (mobile apps with pinned old TLS, IoT devices). Fix: use a policy that includes 1.2.
- **Second most likely:** ACM cert renewed and the new cert is missing an intermediate that the old chain had; ALB serves a partial chain. Fix: include full chain when importing.
- **Third:** Multiple SNI certs on the listener; new cert's CN overlaps and ALB picks the wrong one for some hostnames. Fix: explicit listener rule conditions per SNI host.

**Step 6: Mitigate**
- Revert to previous SSL policy or cert via `aws elbv2 modify-listener`.
- If can't roll back immediately (e.g., cert was deleted), add a second listener with broader compatibility on a different port and CNAME-route legacy clients there temporarily.
- Verify resolution with synthetic monitoring (Pingdom / CloudWatch Synthetics) from multiple regions and TLS configurations.

**Step 7: Post-incident**
- Add **synthetic TLS handshake tests** from multiple regions and TLS versions as a permanent canary.
- ALB security policy changes go through a phased rollout: shadow listener with new policy, traffic-mirror 1% of clients, validate, then promote.
- ACM cert renewal monitored with alarm on `DaysToExpiry` < 30 and on cert change events (CloudTrail rule).

## 2. Nodes healthy, kubectl logs blank for critical pods

The setup: pods are running (status `Running`, ready `1/1`), but `kubectl logs <pod>` returns empty or stale output.

**What's actually happening — the candidate causes:**

**1. Container is writing to a file, not stdout/stderr**
The most common cause. App was configured with `LOG_FILE=/var/log/app.log` instead of stdout. Kubelet only collects what's on stdout/stderr.

```bash
# Verify:
kubectl exec <pod> -- ls -la /var/log/
kubectl exec <pod> -- tail /var/log/app.log
kubectl exec <pod> -- ls -la /proc/1/fd/1 /proc/1/fd/2
# fd/1 should symlink to a pipe; if it's redirected to /dev/null or a file, that's the issue
```

**Fix:** redirect to stdout (sidecar tailing the file, or app reconfig to log to stdout).

**2. Logs were already rotated by kubelet**
Kubelet has a default log rotation: 10MB per file, 5 files max. A chatty app blows through 50MB fast; old logs are gone.

```bash
# Find on-node log location
kubectl get pod <pod> -o jsonpath='{.spec.nodeName}'
ssh <node>
ls -la /var/log/pods/<namespace>_<pod>_<uid>/<container>/
ls -la /var/log/containers/<pod>_<ns>_<container>-<id>.log
# These are symlinks; check the actual location
```

If you need history, you'd need to ship logs externally (Fluent Bit → Loki/CloudWatch).

**3. Container's log driver is misconfigured**
- Docker `log-driver=none` or `log-driver=splunk` (failing silently).
- containerd misconfigured in `/etc/containerd/config.toml`.

```bash
crictl info | jq '.config.containerd.runtimes'
# Check log driver config
journalctl -u containerd | grep -i error
```

**4. App writes to stdout but is buffered**
Python without `PYTHONUNBUFFERED=1`, or Java without `flush` — output sits in the process's stdio buffer until the buffer fills or the process exits. Pods that have been running for a while with low log volume show empty logs.

```bash
kubectl exec <pod> -- env | grep -i buffer
# For Python: PYTHONUNBUFFERED=1
# Or invoke with python -u
```

**5. CRI socket / kubelet bug**
Rare but real. Kubelet has a bad cached pod sandbox state.

```bash
# Check kubelet
ssh <node>
systemctl status kubelet
journalctl -u kubelet -n 200 | grep -i error
# Check container runtime
crictl ps | grep <container>
crictl logs <container-id>     # bypass kubelet, go direct
```

If `crictl logs` works but `kubectl logs` doesn't, it's the kubelet log-fetching path — restart kubelet on the affected nodes.

**6. PID 1 ate the logs**
A common gotcha: if a shell wrapper script doesn't `exec` the real binary, the real process writes to its own stdout but PID 1 (the wrapper) doesn't pipe it. The container "runs" (wrapper is alive) but logs are invisible.

```dockerfile
# Bad:
CMD ./run.sh
# In run.sh: java -jar app.jar    ← no exec, logs lost

# Good:
CMD ./run.sh
# In run.sh: exec java -jar app.jar
```

**7. Node disk full → kubelet can't write logs**
```bash
df -h /var/log /var/lib/containerd
# Logs can't be written because the disk is full
```

**8. SELinux / AppArmor blocking log writes**
```bash
ausearch -m AVC -ts recent
journalctl | grep -i "denied"
```

**Diagnostic sequence I'd run:**

```bash
# 1. Are the pods actually producing output?
kubectl logs <pod> --previous   # last container instance
kubectl logs <pod> --tail=10000 --since=1h

# 2. Can the container runtime see them?
ssh <node>
crictl ps | grep <pod>
crictl logs <container-id>

# 3. Are logs on disk?
ls -la /var/log/pods/<ns>_<pod>_*/
cat /var/log/pods/<ns>_<pod>_*/<container>/0.log | head

# 4. Where is the app writing?
kubectl exec <pod> -- ls -l /proc/1/fd/1 /proc/1/fd/2
kubectl exec <pod> -- lsof -p 1 | grep -E "1u|2u"

# 5. What's the kubelet doing?
journalctl -u kubelet --since "10 min ago" | grep -i "<pod>"

# 6. Is the node OK?
df -h
dmesg | tail -50
```

**Most likely answer in an interview:** "Probably the app is writing to a file instead of stdout — common when the app config wasn't adapted for containers. Or kubelet's rotated the logs already and we're looking at empty rotated files. I'd start with `crictl logs` to bypass the kubelet path, and `kubectl exec` to check where fd 1 and 2 point."

## 3. Sidecar logging agent causing CPU throttling

**Diagnose:**

**1. Confirm CPU throttling is from the sidecar, not the main app**
```bash
kubectl top pod <pod> --containers
kubectl exec <pod> -c logging-agent -- cat /sys/fs/cgroup/cpu.stat
# Look at nr_throttled and throttled_time
# Or for cgroup v1:
# cat /sys/fs/cgroup/cpu,cpuacct/cpu.stat
```

Throttling is the signal: `nr_throttled` increasing means the container is hitting its CPU limit.

**2. Profile the agent**
```bash
# What's it actually doing?
kubectl exec <pod> -c logging-agent -- top -bn1 | head
kubectl exec <pod> -c logging-agent -- ps -eo pid,pcpu,pmem,cmd --sort=-pcpu

# If it's Fluent Bit / Fluentd / Vector, check its own metrics endpoint
kubectl port-forward <pod> 2020:2020
curl localhost:2020/api/v1/metrics/prometheus
```

**3. Common root causes for sidecar log agent CPU spikes:**

- **Parsing regex catastrophic backtracking** — a new multiline parser rule that backtracks on certain log shapes. CPU pegged at 100%.
- **Log volume spike** — the app started emitting 10× more logs (debug logging accidentally enabled).
- **Output destination latency causing backpressure** — Loki/CloudWatch endpoint slow, agent buffering and retrying.
- **No buffer flush limits** — agent reads as fast as the app emits, hits CPU ceiling.
- **No CPU request set on the sidecar** — it got a tiny CFS share, and any work causes throttling.
- **JSON parsing on every record when records aren't all JSON** — try-parse-fail loops.

**4. Quick diagnostic — is it actually CPU-bound or throttled-bound?**

The two look identical from `top` but have different fixes.
- **CPU-bound**: the agent legitimately needs more CPU.
- **Throttle-bound**: the limit is too low even though average usage is fine — bursts get clipped.

```bash
# In the agent container
cat /sys/fs/cgroup/cpu.max     # cgroup v2: <quota> <period>
# Throttle ratio:
cat /sys/fs/cgroup/cpu.stat | grep -E "throttled|nr_periods"
# If nr_throttled / nr_periods > 5%, it's throttle-bound
```

**Rollback:**

If this was a recent deploy:
```bash
# Helm
helm rollback myapp <previous-revision>

# Or via GitOps
git revert <bad-commit>
git push    # ArgoCD reconciles

# Or just remove the sidecar from the deployment temporarily
kubectl patch deployment myapp --type=json -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/1"}
]'
```

For a fast tactical fix without removing the sidecar:
```bash
# Bump the CPU limit
kubectl set resources deployment myapp -c logging-agent --limits=cpu=500m,memory=256Mi
```

But understand: increasing the limit *masks* the issue. If the agent's config has a bad regex, you're just letting it consume more.

**Post-incident hardening:**

- **Test sidecar changes against representative log volume** in staging — the issue often only shows at scale.
- **Set both requests and limits on sidecars** — request reserves capacity, limit prevents runaway impact on the main container.
- **Sidecar resource requests sized empirically** from production observation (kubectl top + Prometheus over a week).
- **CPU throttling alerts**: `rate(container_cpu_cfs_throttled_seconds_total{container="logging-agent"}[5m]) > 0.1` page on this.
- **Vertical Pod Autoscaler in recommendation mode** on sidecars — gives you data to right-size.
- **Canary-deploy sidecar config changes** — change config in one Pod, verify, then propagate.

**The architect's framing:** "Sidecars sharing the pod's resource budget is operationally fragile. CPU throttling on a logging sidecar can cascade — the main app stalls because it's writing to a stdout the kubelet can't flush fast enough because the agent is throttled. I'd consider DaemonSet-deployed log agents (one per node) over per-pod sidecars when the log volume is significant — better resource bin-packing and clearer ownership."

## 4. HPA not scaling despite CPU over threshold

This is a checklist question. Walk through systematically.

**Layer 1: metrics**

```bash
# Is metrics-server installed and healthy?
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system deployment/metrics-server

# Can it actually report metrics for the target pod?
kubectl top pod <pod-name>
# If "Unknown" or error — metrics-server is broken
```

If `kubectl top pod` works, metrics flow is OK. If not, fix metrics-server first.

**Layer 2: HPA's view of metrics**

```bash
kubectl describe hpa <hpa-name>
# Look at:
#   - Metrics:                                       (most recent value)
#     resource cpu on pods  (as a percentage of request):  85% (170m) / 70%
#   - Current/Desired replicas
#   - Conditions:  AbleToScale, ScalingActive, ScalingLimited
#   - Events at the bottom — these are gold
```

**Failure modes the describe will show:**

- `the HPA was unable to compute the replica count: failed to get cpu utilization` → metrics-server can't read the pod's metrics. Often because the pod doesn't have a CPU request set (HPA divides usage by request).
- `the desired replica count is more than the maximum replica count` → `maxReplicas` is the actual cap.
- `the HPA controller was unable to update the number of replicas: deployments.apps "myapp" is forbidden` → RBAC issue, HPA controller can't patch the Deployment.
- `ScalingActive: False, BackoffBoth` → in cooldown after a scale event.

**Layer 3: Common gotchas**

**A. CPU request not set on the pod**
HPA uses `usage / request` as the utilization metric. Without a request, no percentage can be computed.

```bash
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'
```

**B. Stabilization windows / behavior config**
HPA v2 added `behavior` with separate scale-up / scale-down policies and stabilization windows. A 5-minute stabilization window on scale-up means even with CPU pegged, HPA waits 5 minutes before scaling.

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 300   # ← Will not scale up faster than this
```

**C. Multiple metrics, one not available**
If HPA uses both CPU and a custom metric, and the custom metric is unavailable, HPA falls back to weird behavior. Check `kubectl describe hpa` for both metrics' state.

**D. Deployment is at maxReplicas**
```bash
kubectl get hpa <name> -o yaml | grep -E "min|max|current|desired"
```

**E. Pod not Ready**
HPA only counts Ready pods. If new pods aren't passing readiness, HPA sees fewer pods than expected, gets confused.

**F. PodDisruptionBudget blocking**
Edge case: HPA wants to scale down, but PDB blocks. Not relevant to scale-up.

**Layer 4: API server / controller-manager**

```bash
# Is the HPA controller running?
kubectl logs -n kube-system kube-controller-manager-<node> | grep -i hpa

# On managed clusters (EKS/GKE), control-plane logs are in CloudWatch / Stackdriver
aws logs tail /aws/eks/<cluster>/cluster --filter-pattern hpa
```

If controller-manager is restarting or its leader-election is failing, HPA decisions don't get made.

**Layer 5: API throttling**
At very large scale, HPA can get rate-limited by the API server. Symptoms: scale decisions take minutes instead of seconds.

**Most common answer in interviews:** "Almost always the pod is missing a CPU request, or there's a `behavior.scaleUp.stabilizationWindowSeconds` value that's higher than expected. `kubectl describe hpa` tells you in 5 seconds which one it is."

## 5. 504s but ELB health checks green

The ELB-says-healthy / users-see-504 disconnect always traces to one of these: the health check is too lenient, or the failure is not on the path the health check tests.

**Triage sequence:**

**1. Confirm where the 504 is coming from**
```bash
# Headers in the response reveal the proxy that emitted the 504
curl -v https://api.example.com/some-real-endpoint 2>&1 | grep -E "Server|x-amzn"
# Server: awselb/2.0  → ELB itself emitted the 504
# Server: nginx       → in-cluster ingress
# Server: <upstream>  → app's own proxy
```

**2. ELB-emitted 504 means the LB timed out waiting for backend**
- ELB idle timeout default 60s; if backend takes >60s, ELB sends 504.
- Different from health check — health check pings `/health`, but user traffic hits `/expensive-query`.

**3. Look at ELB access logs / target group metrics**
```bash
# CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=<arn-suffix> \
  --start-time $(date -u -d '1 hour ago' +%FT%T) \
  --end-time $(date -u +%FT%T) \
  --period 60 --statistics Average,Maximum
```

If `TargetResponseTime` p99 is near or above the ELB idle timeout, that's the 504 source.

**4. Hypotheses ranked by likelihood:**

**A. Health check tests a trivial endpoint, real traffic hits a slow one**
`/health` returns 200 with no DB call. `/api/orders` does a 30-second query. ELB sees healthy backend, user sees 504.

**Fix:** health check that reflects actual readiness — `/health/ready` validates DB connectivity, downstream APIs, cache warm-up.

**B. Per-pod slowness, not per-target**
ELB target group sees 10 pods, all "healthy." But one pod has a connection pool exhausted and serves slowly. ELB doesn't notice because health checks pass; load balancing is round-robin, so 1/10 requests hit the slow pod.

**Diagnostic:**
```bash
# Look for per-pod variation
kubectl top pods -l app=api
# Check each pod individually
for p in $(kubectl get pods -l app=api -o name); do
  echo "=== $p ==="
  kubectl exec $p -- curl -w "%{time_total}\n" -s -o /dev/null http://localhost:8080/api/test
done
```

**C. Long-running connections + new pod started slowly**
A new pod becomes ready by health-check standards (process up, port open) but hasn't warmed JIT / cache / connection pool. First few requests time out.

**Fix:** longer `startupProbe`, warm-up endpoint hit before passing readiness.

**D. Upstream dependency slow**
DB or third-party API slowed down. App times out waiting; ELB times out waiting for app; user sees 504.

**Diagnostic:** distributed tracing (X-Ray / Tempo) — find a slow span. Or DB metrics (RDS CPU, connection count, slow queries).

**E. DNS resolution slow inside pods**
CoreDNS overloaded or rate-limited. App hangs on outbound DNS lookups. Symptoms: occasional 30s pauses (DNS retry timeout).

```bash
kubectl exec <pod> -- nslookup downstream-api.namespace.svc.cluster.local
# Time it. Should be < 100ms.
```

Fix: NodeLocal DNSCache.

**F. SNAT port exhaustion (AWS)**
EKS nodes share SNAT ports for outbound traffic via NAT Gateway. At high concurrency, ports exhaust, new connections fail or hang.

```bash
# On node
ss -s
# Look at "ports" / "estab"
```

Fix: more NAT Gateway IPs, VPC endpoints for AWS services, or eliminate SNAT via per-pod source IP.

**G. Kubernetes Service / endpoints mismatch**
Pod was terminated, kube-proxy hasn't updated iptables yet, ELB target group still has the IP. Traffic to a dead pod → connection reset / timeout.

```bash
kubectl get endpoints <svc>
# Compare to actual pod IPs
kubectl get pods -l <selector> -o wide
```

**H. Asymmetric routing / mTLS handshake**
In a mesh, mTLS handshake fails for some requests if certs are rotating. Symptoms: bursts of 504s correlated with cert rotation events.

**5. Mitigation playbook:**

- Increase ELB idle timeout if upstream legitimately needs more time.
- Make health check more representative of real readiness.
- Add per-target response time alarms (not just unhealthy host count).
- Add panic-button feature flags so the slow code path can be disabled.

**6. Long-term fix:** **canary + automated rollback on p99 latency regression**. The deploy that introduced this would have been caught by SLO-aware progressive delivery.

## 6. journald logs vanishing on reboot across some AMIs

This is an AMI build problem. systemd-journald has two storage modes:

- **Volatile** (default on many minimal AMIs): logs stored in `/run/log/journal` (tmpfs). Lost on reboot.
- **Persistent**: logs in `/var/log/journal`. Survive reboot.

**What to check in the image build:**

```bash
# In the AMI build script / Packer config / cloud-init
# Is journal storage configured persistent?
grep "^Storage=" /etc/systemd/journald.conf
# Should be: Storage=persistent
# (or "auto" with /var/log/journal directory present)

ls -ld /var/log/journal
# If this dir doesn't exist, journald falls back to volatile

# Permissions matter:
# /var/log/journal owned by root:systemd-journal, mode 2755
```

**Fix in the AMI build:**
```bash
# In Packer / user-data / Ansible playbook:
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald

# Or set explicitly in config:
sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf
```

**What to check in the boot sequence:**

- **Disk mount order** — if `/var/log` is on a separate volume (EBS), is it mounted before journald starts? Check `/etc/fstab` ordering and `Before=` dependencies.
- **cloud-init** wiping logs — some cloud-init configs explicitly clear `/var/log` on first boot.
- **Bootloader regenerated initramfs** — if an AMI build re-runs `dracut`, journal files in initramfs might not persist.
- **Disk-prep userdata** that reformats `/var/log` volume.

**Forensics:**
```bash
# Look at journal config in the running system
journalctl --header | head
# "Persistent journal" or "Volatile journal" — definitive

# Check whether journal dir exists and was populated pre-reboot
ls -la /var/log/journal/
# Each entry is a 128-bit machine ID; should have files

# When did journald last start?
journalctl -u systemd-journald --since "1 hour ago"
```

**Persistent vs volatile diagnosis:**
- If `/run/log/journal` exists and `/var/log/journal` doesn't → volatile.
- If `/var/log/journal` exists but is empty → mount issue; check whether something wiped it on boot.
- If `/var/log/journal` exists and has files but logs are gone → journald rotation/size limits aggressive.

**Standardization fix:** publish a **golden AMI** with verified journald config, and add a smoke test to the AMI pipeline:
```bash
# AMI test (run after first boot)
test "$(journalctl --header | grep -c 'Persistent')" -ge 1 || exit 1
```

## 7. OOMKilled pod, no logs

**Why logs might be missing:**
- Kubelet's log rotation already cycled out the OOM-time logs.
- App was writing to a file, not stdout (see Q2).
- OOM happened mid-write, last buffer never flushed to disk.

**Forensic-level debug:**

**1. Confirm OOMKill, not OOMKilled-by-cgroup-limit-vs-host-OOM**
```bash
kubectl describe pod <pod> | grep -A5 "Last State"
# Look for: Reason: OOMKilled, Exit Code: 137

kubectl get events --field-selector involvedObject.name=<pod>
```

**2. Get the *previous* container's logs** (the one that was OOMKilled, not the new restart)
```bash
kubectl logs <pod> --previous
kubectl logs <pod> --previous -c <container-name>
```
This is the single most-missed step. The OOMed container is gone; logs from its lifetime live in `--previous`.

**3. On-node forensics**
```bash
# Find the node
NODE=$(kubectl get pod <pod> -o jsonpath='{.spec.nodeName}')
ssh $NODE

# Check kernel OOM messages
dmesg -T | grep -i "killed process\|oom"
journalctl -k --since "30 min ago" | grep -i oom

# Sample output:
# [Wed Sep 13 11:23:45 2025] Memory cgroup out of memory: Killed process 12345 (java) total-vm:2058920kB, anon-rss:1048576kB...
# This tells you exactly which process, in which cgroup, used how much memory at the moment of death

# Find the pod's cgroup memory.events
ls /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<UID>.slice/
cat .../memory.events
# oom_kill counter, oom counter, low/high events

# Container logs on disk (if not rotated)
ls /var/log/pods/<ns>_<pod>_<uid>/<container>/
cat /var/log/pods/<ns>_<pod>_<uid>/<container>/0.log
# Number suffix increments per restart; look at the highest one
```

**4. Heap / memory profile if app had one**
- Java with `-XX:+HeapDumpOnOutOfMemoryError` writes a heap dump to a configured path. If a PV was mounted, the heap dump survives.
- Go binaries with pprof endpoint — pull periodic profiles to S3 via a sidecar.

**5. Prometheus / metrics historical data**
```promql
# Memory trend leading up to the kill
container_memory_working_set_bytes{pod="<pod>"}[30m]
# RSS growth
container_memory_rss{pod="<pod>"}[30m]
# Page faults — sudden spike often signals a memory pressure event
rate(container_memory_pgfault{pod="<pod>"}[5m])
```

If you can see usage climbing steadily toward the limit, that's a leak. If usage was flat then jumped, that's a workload-shape change (large request, big batch).

**6. APM / tracing**
- Application Insights / Datadog / New Relic: look at the spans around the OOM time.
- Tempo / Jaeger: traces leading up to the kill.

**7. Recreate with debug instrumentation**
```bash
kubectl debug -it <pod> --image=busybox --target=<container>
# Or restart with more memory + heap dump enabled:
kubectl set resources deployment/<dep> -c <container> --limits=memory=4Gi
# Watch with kubectl top pod -n <ns> --watch
```

**Common root causes:**
- **Limit too low for actual workload** — limit was set conservatively, real traffic exceeded it. VPA recommendations would have caught this.
- **Memory leak** — heap grows over time, hits limit. Heap dump → leak suspect class.
- **Native memory leak** — Java direct buffers, JNI, off-heap caches. JVM heap looks fine; native RSS climbs. Track via NMT (`-XX:NativeMemoryTracking=summary`).
- **Memory amplification** — a request that loads a large dataset into memory; one bad query OOMs.
- **JVM container-unaware** — without `UseContainerSupport`, the JVM thinks it can use the whole node, sets a huge heap, gets killed.

**Prevention:**
- **VPA in recommendation mode** for sizing.
- **Heap dumps on OOM** with a persistent volume for collection.
- **Memory alerts at 80% of limit** so leaks are caught before the kill.
- **Application-level metrics** — JVM heap, GC time, allocation rate — beyond just container memory.

## 8. Kernel panic on GKE node mid-deploy

A kernel panic on a managed-K8s node is unusual — Google maintains the COS (Container-Optimized OS) base. So the first instinct is: it's probably workload-induced, not the kernel itself.

**Triage — is it infra, base image, or app-level?**

**1. Capture the panic if possible**
- GKE serial console logs (from GCP Console → VM → Serial Port 1) — kernel panics typically print stack trace here.
- `gcloud compute instances get-serial-port-output <node>` — pull serial logs.
- GKE node-problem-detector should also report; check `kubectl describe node` for conditions.

**2. Pattern: what was deploying?**
- Was this the first node to receive the new workload?
- New base image for app containers?
- New CSI driver, new CNI plugin, new privileged DaemonSet, new device-plugin?
- Did `kubectl apply` add a workload requesting hostPath, hostNetwork, hostPID, or hostIPC?

**3. Classification by symptom:**

**A. Infra / underlying GCP issue**
- Reproducibility: random nodes panic without your changes.
- Other tenants/projects affected (GCP status page).
- Panic stack mentions hardware: `MCE`, `EDAC`, page table corruption with no userspace context.
- Action: file ticket with Google Cloud Support; consider draining and replacing the node pool.

**B. Base image issue**
- COS auto-upgrades from one minor version to another and the new kernel has a bug.
- Reproducibility: all nodes panic after the upgrade.
- Stack often points at recently-changed kernel modules.
- Action: pin node pool to a known-good COS image version (`--image-family`). File issue with Google.

**C. App / workload-induced**
- Reproducibility: panic happens when a specific workload runs.
- Stack trace shows a kernel module the workload pulled in (network namespace, eBPF program, FUSE mount).
- Action: identify the workload, isolate to dedicated nodes for testing, fix or roll back.

**4. Forensics to differentiate:**

```bash
# Node-level diagnostics
gcloud compute ssh <node> --command="dmesg -T | tail -100"
gcloud compute instances get-serial-port-output <node> > serial.log
grep -i "panic\|BUG\|Oops\|RIP:" serial.log

# Node-problem-detector reports
kubectl describe node <node> | grep -i kernel

# What workload was on the node at panic time?
kubectl get events --all-namespaces | grep <node>
kubectl logs -n kube-system <kube-scheduler-pod> | grep <node>

# Was a privileged pod recently scheduled there?
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node> | \
  xargs -I {} kubectl get pod {} -o json | jq '.spec.securityContext, .spec.hostNetwork, .spec.hostPID'
```

**5. Common workload-induced kernel panics in my experience:**

- **eBPF program from a security agent** (Falco, Sysdig, Pixie) loading a broken probe → kernel verifier accepted but probe causes panic.
- **CSI driver bug** — FUSE-based CSI drivers can panic the kernel under load.
- **Network plugin bug** — Calico/Cilium kernel datapath under specific traffic patterns.
- **GPU driver / device plugin** — out-of-tree NVIDIA driver versions mismatched with kernel.
- **Privileged container syscalls** — extremely rare on locked-down COS.

**6. Stabilize:**

- **Cordon affected nodes** and let workloads reschedule.
- **Isolate suspect workload** with `nodeAffinity` to a separate, less-critical node pool.
- **Roll back the deploy** that introduced the change.
- **Open a GCP support ticket** with serial console output — Google's kernel team can read the stack.

**7. Prevent:**

- **Test new privileged workloads, CSI drivers, eBPF tools in a sandbox node pool** before promoting to prod node pools.
- **Pin node pool image versions** for production; only auto-upgrade non-prod first.
- **Use surge upgrades** so a bad image rollout doesn't bring down all nodes simultaneously.
- **Monitor node-problem-detector** with alerting on `KernelDeadlock`, `KernelPanic`, etc.

**The architect's framing:** "Kernel panics in managed K8s are almost always either the cloud provider's problem (file a ticket) or a privileged-workload's problem (CSI, eBPF, hardware drivers). The diagnostic that separates the two is the serial console output — userspace context in the panic frame points at app, no userspace context points at hardware or kernel. Either way, the immediate move is to cordon and rebalance; root cause is a calmer parallel investigation."

---

# 🎯 Round 3 — Leadership, Influence, Production Principles

## 1. Empower devs without footguns

The philosophy I'd lead with: **paved roads, not paved walls**. Make the easy path the right path; don't gate the wrong path with bureaucracy.

**Concrete patterns:**

**1. Golden paths via templates**
- Backstage / Port / internal portal: "Create a new service" produces:
  - GitHub repo from a template (with `CODEOWNERS`, branch protection, Dependabot).
  - CI/CD pipeline already wired up.
  - Helm chart in a GitOps repo with sensible defaults.
  - Observability dashboards auto-generated.
  - Slack channel and PagerDuty service.
  - Secrets-manager namespace created.
- A new service is in production in an afternoon, on the paved road, with everything wired correctly.

**2. Self-service, opinionated infrastructure**
- Terraform modules that codify the platform team's standards. "Want an RDS? Use our module — it enforces encryption, backup, multi-AZ, parameter group baseline."
- Modules accept a small set of inputs; no escape hatches that bypass guardrails.
- Modules are versioned; teams pin versions and upgrade deliberately.

**3. Policy-as-code at the edge, not the gate**
- Kyverno / OPA Gatekeeper in *audit mode* first — warn, don't block. Get teams to fix violations on their own timeline.
- Then move to enforce. By then, almost no one is violating.
- Cost: 6 months of polite reports. Benefit: cultural buy-in.

**4. Sane defaults that beat the alternatives**
- Default Helm chart includes: probes, resource requests, PDB, NetworkPolicy, Prometheus annotations, security context. Teams who use the default get all of it free.
- Want to opt out? Possible, but requires explicit justification.

**5. Observability is a paved road, not a checklist**
- Auto-instrumentation libraries (OpenTelemetry SDK with auto-config) added to standard base images.
- A new service emits RED metrics, traces, structured logs automatically — without the team writing code.

**6. Pre-prod environments that match prod**
- Dev clusters with same admission policies, same NetworkPolicy enforcement, same CSI driver, same OS. If it works in dev, it works in prod.
- "It worked in dev" is the team's problem; "it didn't even run in dev because of policy" is something they can fix immediately.

**7. Make recovery easier than mistakes**
- Backups everywhere, restore tested quarterly.
- Easy rollback (helm rollback, git revert).
- Feature flags for risky changes — turn off without redeploying.
- Idempotent migrations.

**8. Production access tiers**
- Dev environment: full power, including `kubectl exec` to anything.
- Staging: same as dev for the team's own services; read-only on others.
- Prod: read-only by default; "break glass" for incident response logged and reviewed.
- Service identity (IRSA, workload identity) is how apps authenticate; humans don't have prod keys.

**9. What I'd NOT do:**
- Block all customizations. The platform team can't anticipate everything; rigid platforms produce shadow IT.
- Wrap every operation in a ticket. Engineers won't follow it; they'll find workarounds.
- Build a "platform" that's a slow proxy over kubectl. If kubectl is what they need, give them kubectl with audit.

**The principle I'd articulate:** "Empowerment is providing tools where the easy thing is the correct thing. Footguns are removed by making the safer alternative cheaper, not by hiding the gun behind a process. The platform team's success metric is how few times anyone has to ask permission."

## 2. Linux-level checklist before approving a custom AMI to production

**Filesystem and storage:**
- `/` mounted with `noexec`, `nosuid`, `nodev` where applicable? (`/tmp`, `/var/tmp`, `/dev/shm` should have these.)
- Filesystem journaling enabled (`ext4` with journal, `xfs` with reflinks for snapshots).
- Disk encryption (LUKS or cloud-native — EBS encryption flag).
- `/var/log` on a separate partition to prevent root fill.
- `noatime` on busy filesystems for performance.

**Kernel and modules:**
- Latest patched kernel for the distribution (CIS benchmark-aligned).
- Unnecessary kernel modules blacklisted (`/etc/modprobe.d/`) — usb-storage, firewire, bluetooth on a server.
- `sysctl` baseline:
  - `net.ipv4.tcp_syncookies=1`
  - `net.ipv4.conf.all.rp_filter=1`
  - `net.ipv4.ip_forward=1` (for K8s nodes)
  - `vm.swappiness=10` or `0` (K8s requires swap off)
  - `vm.max_map_count=262144` (for Elasticsearch / containers)
  - `fs.file-max=2097152`
  - `kernel.pid_max=4194304` (containers create lots of PIDs)
- Kernel hardening: `kernel.kptr_restrict=2`, `kernel.dmesg_restrict=1`.
- ASLR enabled.
- AppArmor or SELinux in enforcing mode (where compatible).

**systemd / init:**
- journald with `Storage=persistent` (see Q6 in Round 2).
- Unnecessary services masked: `cups`, `bluetooth`, `avahi-daemon`, `rpcbind`, `nfs` if not used.
- `systemd-resolved` configured correctly (or disabled if using cluster DNS).
- Timezone set, NTP enabled (chrony or systemd-timesyncd) with verified upstream.

**Networking:**
- `iptables-persistent` or `nftables` baseline rules.
- IPv6 disabled if not used (or configured properly if it is).
- `sshd_config` hardened:
  - `PermitRootLogin no`
  - `PasswordAuthentication no` (key-only)
  - `Protocol 2`
  - `ClientAliveInterval 300`
  - `MaxAuthTries 3`
  - Only specific KEX/cipher algorithms (CIS benchmark).
- Host firewall (iptables/nftables) — default deny inbound except known ports.

**Logging and auditing:**
- `auditd` running with a baseline rule set (CIS-recommended rules: file changes, command execution by root, privilege changes).
- Logs forwarded to central log aggregation (Fluent Bit / CloudWatch agent).
- No local-only logs for critical events; everything ships off-box.

**Package management:**
- `unattended-upgrades` configured for security patches only (Debian/Ubuntu).
- `dnf-automatic` for security patches (RHEL/Fedora).
- Or — if immutable: AMI rebuilt frequently, no in-place patching.
- Package signing verification enabled.
- Removed development tools, compilers, scripting languages not needed at runtime.

**Users and access:**
- No password-based local accounts beyond root (locked).
- No default service accounts with shell access.
- `sudo` configured with logging, NOPASSWD only where automation requires.
- Login banner with legal notice.
- `/etc/securetty` restricted.
- Failed login lockout (`pam_tally2` or `pam_faillock`).

**Container runtime (for K8s nodes):**
- containerd configured with appropriate snapshotter, registry mirrors, image GC thresholds (`image-gc-high-threshold=85, image-gc-low-threshold=70`).
- Container runtime cgroup driver matches kubelet (`systemd` for modern systems).
- containerd has `discard_unpacked_layers = true` for disk efficiency.

**Security agents:**
- EDR / antivirus (CrowdStrike, SentinelOne) installed and reporting.
- Falco or equivalent for runtime detection.
- SSM Agent (AWS) or equivalent for patch management and session manager — preferred over SSH.

**Compliance and audit trail:**
- AMI built via Packer with versioned config in Git.
- AMI tagged with: build date, source AMI, OS version, kernel version, Packer git SHA, builder identity.
- AMI scanned with Trivy / Inspector / Clair for CVEs — no CRITICAL.
- AMI promotion follows a pipeline: build → scan → smoke test → publish to staging → promote to prod after validation.
- AMI deprecation policy: AMIs older than 90 days flagged; older than 180 days blocked from new launches.

**Smoke tests on the AMI:**
- Boots cleanly without manual intervention.
- All expected services start (`systemctl --failed` returns nothing).
- Networking works (DNS resolves, time syncs).
- kubelet/agent registers with cluster.
- Sample pod runs successfully.
- Reboot cycle completes cleanly.

**The architect's framing:** "An AMI is a contract about how a Linux host behaves in production. Every item on the checklist exists because something at some point went wrong without it. The checklist isn't theater; it's institutional memory. I review it annually with the security team and trim items that have become irrelevant; bloated checklists get ignored."

## 3. Centralized logging → service-mesh-based observability — tradeoffs

First, a precision: "service mesh observability" usually means telemetry generated by mesh sidecars (Envoy) — request traces, latency histograms, error rates per service-to-service edge — rather than logs themselves. The question is more accurately "centralized logging-and-metrics → mesh-native telemetry + distributed tracing."

**What the move actually changes:**

**Centralized logging model:**
- Apps emit logs (often unstructured) → shipped to ELK/Loki → searched after the fact.
- Metrics: ad-hoc, Prometheus scraping `/metrics` per app.
- Tracing: optional, app-instrumented (and inconsistent).

**Service-mesh observability model:**
- Mesh sidecars emit standardized telemetry: request rate, error rate, latency histograms, trace spans for every inter-service call.
- Apps may still emit logs, but mesh data is the primary source for service-level visibility.
- Cross-service correlation is automatic via shared trace IDs.

**Tradeoffs:**

**Pros:**

- **Uniform telemetry across all services** without per-team instrumentation work. A service joins the mesh and immediately has RED metrics and traces.
- **Cross-service visibility** — the trace shows the full request path through the platform. Service ownership boundaries don't break visibility.
- **Standardization** — same labels, same percentiles, same dimensions everywhere. Dashboards and alerts portable across teams.
- **Less app-side instrumentation burden** — no per-language SDKs to maintain. Polyglot environments benefit most.
- **Identity-aware telemetry** — mesh knows the SPIFFE identity of caller and callee. Security and access analysis layer on for free.
- **Black-box observability** — works for services you don't control (vendor binaries, legacy apps).

**Cons:**

- **Telemetry is at the network boundary, not inside the app** — you see the call took 800ms, but mesh doesn't know it was the DB query inside that took 750ms. Need app-level metrics + traces to drill down.
- **Sidecar overhead** — every pod gets an Envoy sidecar. CPU/memory cost (~50-100m CPU, 100-200Mi memory per pod). At thousands of pods, this is real money.
- **Operational complexity** — mesh is itself a distributed system. Istio/Linkerd upgrades have failure modes. Mesh control-plane bugs can cause cluster-wide incidents.
- **Logging is still needed** — for "what did the app actually do," not just "what HTTP request did it serve." Mesh doesn't replace structured logging; it augments it.
- **Trace sampling complexity** — at high scale, you can't keep every trace. Tail-based sampling vs head-based sampling is a non-trivial design decision.
- **Increased data volume** — mesh emits a lot of telemetry. Backend (Prometheus, Tempo, Loki) sized accordingly.
- **Cross-mesh blind spots** — non-mesh services (databases, external APIs) appear as opaque endpoints in traces. Mesh telemetry doesn't extend to them.

**Practical hybrid model I'd advocate:**

Don't replace centralized logging — augment it.

| Use mesh for | Keep centralized logging for |
|---|---|
| Service-to-service RED metrics | Application business logic logs |
| Distributed tracing | Audit/security events |
| mTLS-based identity | Compliance log retention |
| Traffic shaping & retry visibility | Stack traces, debug context |
| L7 policy verdicts | Slow query logs, dependency events |

**The transition I'd plan:**

1. Stand up the mesh in parallel — Istio/Linkerd in `permissive` mode (mesh observes traffic; doesn't enforce).
2. Build mesh-native dashboards alongside existing ELK ones.
3. Get teams comfortable reading both.
4. Migrate alerts gradually from log-grep to mesh-metric-based.
5. Move enforcement (mTLS strict, AuthorizationPolicy) only after observability is solid.
6. Reduce log retention for ephemeral logs once mesh-telemetry meets the same use cases.

**The architect's framing:** "Mesh observability is a complement, not a replacement. The platform-team value comes from standardization — every team gets the same RED metrics for free. The cost is sidecar resource overhead and an additional distributed system to operate. I wouldn't undertake this transition unless the org has the operational maturity to run a service mesh, and unless cross-team telemetry standardization is a stated business priority. For Atlassian, given the polyglot service ownership and cross-team service interaction, it's likely worth it. For smaller orgs with 10 services, it's overkill."

## 4. Simulating production-level chaos in staging for Kubernetes

**Principles first:**

1. **Chaos in staging only makes sense if staging looks like production.** Same architecture, similar scale, similar traffic shape.
2. **Hypothesis-driven** — "I believe service X can tolerate failure Y; let's prove it." Not "let's break things and see."
3. **Steady-state metric defined upfront** — before injecting failure, define what "still working" means.
4. **Blast radius bounded** — chaos has a kill switch and explicit scope.
5. **Run results inform real changes** — if a failure exposes a weakness, file work to fix it.

**Tools I use:**

- **Chaos Mesh** — Kubernetes-native, declarative chaos experiments via CRDs. My default.
- **Litmus** — similar, broader experiment library.
- **AWS Fault Injection Simulator (FIS)** — for AWS infrastructure chaos (terminating instances, throttling APIs).
- **Gremlin** — commercial, broader but expensive.
- **PowerfulSeal**, **kube-monkey** — older, simpler.

**Chaos categories and example experiments:**

**Infrastructure-level:**
- **Node failure** — terminate a node mid-traffic. Verify pods reschedule, no SLO breach.
- **AZ failure** — block all traffic to/from one AZ via NetworkPolicy + VPC route changes. Verify multi-AZ workloads continue.
- **Disk fill** — fill a node's disk to 95%; verify kubelet evicts the right pods and disk-pressure alerts fire.
- **CPU exhaustion** — stress-ng on a node; verify other pods aren't starved.

**Network-level:**
- **Pod network partition** — inject 100% packet loss between Service A and Service B. Verify retries, circuit breakers, fallback paths.
- **High latency** — 500ms artificial latency on egress. Verify timeouts are tuned correctly.
- **DNS failure** — block CoreDNS for 30s. Verify NodeLocal DNS cache or service-name resilience.
- **Connection drops mid-request** — TCP RST injection.

**Application-level:**
- **Pod kill** — kill a random pod every 30 minutes. Verify graceful shutdown works (no in-flight requests dropped).
- **CPU throttle** — clamp a pod's CPU to 50m for 2 minutes. Verify HPA scales out.
- **Memory pressure** — allocate to near-limit. Verify OOM-handling.
- **Stuck process** — SIGSTOP a container. Verify liveness probe catches it.

**Dependency-level:**
- **Downstream service slow** — inject 5s latency on the mesh edge for one upstream. Verify caller times out cleanly.
- **Database failover** — trigger RDS failover. Verify connection pool reconnects, queries retry.
- **Cache miss** — flush Redis. Verify graceful degradation, not stampede on DB.
- **External API down** — block egress to a third-party. Verify circuit breakers fire.

**Control-plane:**
- **API server pressure** — load the API with read-list queries. Verify in-cluster controllers (HPA, ArgoCD) still function.
- **etcd slow** — inject latency on etcd peers. Verify cluster stays operational.

**Practical chaos engineering workflow:**

```yaml
# Chaos Mesh example: kill a random pod in payments namespace every 10 min
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: payments-pod-kill
  namespace: chaos-experiments
spec:
  action: pod-kill
  mode: random-one
  selector:
    namespaces: [payments]
    labelSelectors:
      tier: backend
  scheduler:
    cron: "@every 10m"
  duration: "30s"
```

```yaml
# Network latency between two services
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata: { name: payments-to-db-latency }
spec:
  action: delay
  mode: all
  selector:
    namespaces: [payments]
    labelSelectors: { tier: backend }
  delay:
    latency: "500ms"
    jitter: "100ms"
  direction: to
  target:
    mode: all
    selector:
      namespaces: [data]
      labelSelectors: { app: postgres }
  duration: "5m"
```

**Game days:**

- Quarterly, 4-hour blocked window.
- Multi-team participation.
- Pre-defined hypotheses ("if the EU region goes down, traffic shifts to US within 5 min without user impact").
- Live observability dashboard with on-call mentally rehearsing real incident response.
- Blameless retrospective afterward: what worked, what didn't, what's the fix.

**What I track post-experiment:**
- Did steady-state metrics stay green?
- Did alerts fire correctly (not false-positive, not silent)?
- Did the right on-call get paged?
- Did runbooks work?
- What broke that we didn't expect?

**The architect's framing:** "Chaos engineering is the discipline of building confidence through controlled experimentation. Run it in staging first to verify your tooling and hypotheses, but the real value is running it in production — gradually, with bounded blast radius, on services that are designed for it. The biggest chaos lesson isn't 'we found a bug.' It's 'we found that our monitoring was blind to this failure mode' or 'our runbook was wrong.' Operational learning compounds; the systems get more reliable the more chaos you run."

## 5. Pushback from leadership when SLOs threaten velocity

This is a leadership-influence question; the answer must show how I'd reason about it, not just what I'd recommend.

**Framing the conversation:**

> "I don't see SLOs as a velocity tax. I see them as the explicit budget for risk. When velocity threatens SLO compliance, the question isn't 'should we slow down?' — it's 'how much reliability are we willing to spend, and what are we spending it on?'"

**The actual recommendation pattern:**

**1. Acknowledge the real concern**
Leadership pushback usually isn't "SLOs are bad." It's:
- "We need to ship feature X to win deal Y."
- "Engineering is slowing down and I don't know why."
- "Compliance / SREs are blocking us."

I take that concern seriously. Velocity matters. Customers don't reward us for great SLOs; they reward us for great products.

**2. Re-frame SLOs as a tool, not a constraint**
SLOs and **error budgets** are how the team makes that trade-off explicit. The error budget says: "We promise 99.9% to customers; that means we have 43 minutes of error budget per month. If we have 30 minutes left, we can ship the risky feature. If we have 5 minutes left, we should stabilize."

The conversation becomes data-driven, not opinion-driven. Leadership sees the budget. The team sees the budget. Engineers don't get blamed for a slowdown — the budget did.

**3. Show, don't tell**
- Pull data: when have SLO violations actually slowed velocity? In my experience, the answer is "the times we ignored the budget and shipped anyway, which produced incidents that consumed three sprints of engineering time."
- The choice isn't velocity vs reliability. It's *predictable velocity with occasional planned slowdowns* vs *fast velocity punctuated by unpredictable outages*. The second is slower in aggregate.

**4. Tune the SLO, not the deployment cadence**
Often the SLO is too tight for the actual business need. If we set 99.99% and customers only need 99.5%, we're spending engineering for compliance to nothing. Push to re-derive SLOs from actual user pain — what % of requests must succeed for the user to perceive the system as "working"? That number is often lower than engineers initially proposed.

**5. Offer alternatives that protect both**
- **Progressive delivery** — canary deploys with automatic rollback consume *less* error budget than monolithic deploys. So we can ship more by deploying smaller, safer chunks.
- **Feature flags** — ship code without exposing it; flip on for 1% of users; expand if metrics hold. Velocity preserved, risk bounded.
- **Pre-prod realism** — fewer production surprises mean less budget consumed, so we can deploy faster.
- These tactics often unlock 2-3× the velocity at the same SLO.

**6. Bring leadership into the decision, don't shield them**
If the situation is truly "we must ship this risky thing for a strategic deal, we know the SLO will burn":
- Bring the data. "Shipping this will consume X% of our error budget. We have Y% remaining. The expected breach probability is Z."
- Get explicit sign-off. "If we agree to spend this budget, we commit to a freeze afterward to recover. Are we OK with that trade?"
- This makes leadership a partner in the decision, not an adversary.

**7. Don't die on a hill that doesn't matter**
Some SLOs are vanity. Latency p99 for an internal admin tool? Probably nobody cares. Don't expend political capital defending SLOs that don't map to business value. Save the influence for the SLOs that do.

**8. Build relationship before need**
The time to establish trust with leadership about SLOs is *before* the deal-pressure moment. Regular reliability reviews where I show "here's what we deferred to protect velocity, here's what we protected to keep customers happy, here's the math." When the pressure comes, they trust the framework.

**The framing I'd close with:**
> "SLOs aren't a velocity tax. They're a forcing function for explicit trade-offs. The teams I've worked on that took SLO discipline seriously shipped *faster* over a quarter, because they spent less time firefighting and more time building. My job when leadership pushes back isn't to defend SLOs as a sacred principle — it's to show that the discipline produces better outcomes for the things leadership actually cares about. If it doesn't, then maybe the SLO is wrong and we should change it together."

---

That's the full set across all three rounds. Likely Atlassian-specific follow-ups to be ready for:

- **"You're describing platform engineering — how do you measure platform team success?"** — DORA metrics + developer satisfaction + platform NPS, not infrastructure-internal metrics like "uptime of internal services."
- **"How do you handle a long-tenured engineer who refuses to follow platform standards?"** — direct conversation, understand the concern, find what's right in their objection, fix the standard if needed, expect compliance once that's done.
- **"How would you onboard a 20-person team to your platform?"** — paved roads, office hours, internal Slack support, documentation that anticipates failure modes, and explicit "platform service-level commitments" so teams know what to expect from us.
- **"Describe a time you were wrong about an architecture decision."** — have one ready, structured: what I decided, why, what went wrong, how I noticed, what I did, what I learned and changed.

# Amazon Senior DevOps Interview — Complete Walkthrough

Amazon's bar is high on systems thinking and operational depth. They want to see structured reasoning, ownership instincts, and the ability to handle ambiguity — not just textbook answers. Going through each with the depth a senior engineer brings.

---

# 🎯 Round 1 — System Thinking

## 1. Request flow: DNS → LB → Service → Pod → Database

Walking through every hop, what happens, and where things can go wrong.

**Hop 1: DNS resolution**

The user types `app.example.com` or the browser makes a request.

- Browser checks its own DNS cache, then the OS resolver cache.
- If miss, OS asks the configured resolver (usually ISP or corporate DNS).
- Resolver walks the DNS hierarchy: root → TLD (`com`) → authoritative (`example.com`).
- For us, that authoritative server is **Route 53**.
- Route 53 returns an answer based on the configured routing policy:
  - **Simple**: one record.
  - **Latency-based**: returns the IP for the lowest-latency region.
  - **Geolocation**: returns based on user's geographic region.
  - **Failover with health checks**: returns primary if healthy, secondary otherwise.
  - **Weighted**: splits by percentage (canary, A/B).
- The returned record is typically an **alias** to an ALB or CloudFront distribution.
- Response cached at each hop per TTL.

What can go wrong: stale DNS caches at clients ignoring TTL, Route 53 health checks misfiring, third-party DNS provider issues.

**Hop 2: CloudFront (if used)**

If the alias resolves to CloudFront:
- User hits the nearest edge POP via anycast.
- CloudFront checks its cache; serves cached response if hit.
- On miss, forwards to the origin (typically an ALB) over the AWS backbone.
- TLS terminated at the edge (faster handshake, AWS-managed certificate).
- WAF rules evaluated; bad requests rejected here.

What can go wrong: cache poisoning, origin failover misconfigured, WAF rules blocking legitimate traffic.

**Hop 3: Application Load Balancer**

CloudFront (or the client directly) hits the ALB:
- ALB sits in public subnets, spanning multiple AZs.
- TLS handshake (if not terminated at CloudFront).
- ALB applies listener rules: path-based / host-based routing to a target group.
- ALB checks target health; only routes to healthy targets.
- Picks a target via round-robin or least-outstanding-requests algorithm.
- Forwards the request, preserving original headers (`X-Forwarded-For`, `X-Forwarded-Proto`).

What can go wrong: TLS handshake failures (cert mismatch, policy too strict), target group health-check misconfiguration, sticky session conflicts.

**Hop 4: Kubernetes Service / Ingress**

The ALB target is typically managed by the **AWS Load Balancer Controller**:
- For an Ingress, the ALB targets are pod IPs (via IP target type) or node-port targets.
- For a LoadBalancer Service, similar but at L4.
- Traffic arrives at the pod's IP directly (if using IP target mode + VPC CNI).

If using an in-cluster ingress controller (NGINX) instead:
- ALB forwards to NGINX pods.
- NGINX reads the Host header / path, maps to a Service.
- NGINX forwards to a backend pod via the Service.

**Hop 5: kube-proxy / Service DNS / pod**

Inside the cluster:
- Pod-to-pod calls hit a `ClusterIP` Service.
- The pod's DNS resolver (CoreDNS) resolves `myservice.namespace.svc.cluster.local` to the ClusterIP.
- kube-proxy (via iptables or IPVS) intercepts traffic to the ClusterIP and DNATs it to a backing pod IP.
- Traffic arrives at the pod.

What can go wrong: CoreDNS overloaded, kube-proxy iptables stale after pod churn, EndpointSlice out of sync.

**Hop 6: Pod / container / application**

The container receives the request:
- Sidecar (Envoy, Istio) intercepts if there's a service mesh — adds mTLS, retries, telemetry.
- Application code processes the request.
- App calls downstream services (other pods, AWS APIs) via similar paths.

**Hop 7: Database**

The application opens (or reuses) a connection to RDS:
- Connection goes through the Service / pod network to the RDS endpoint.
- RDS endpoint is a DNS name; resolves to the writer instance's private IP.
- Traffic routes through private subnets to the RDS network interface.
- DB processes the query, returns a result.
- For reads, the app may use a separate reader endpoint that load-balances across read replicas.

What can go wrong: connection pool exhausted, DNS failover during failover event, slow queries, lock contention.

**The response path:**

Reversed back through the stack. Some layers (like CloudFront) may cache the response based on headers.

**Architect's framing:**
> "Every hop is an opportunity for latency, failure, or observability gap. The skill is knowing which hop owns which class of problem — DNS owns name resolution and failover speed, ALB owns connection handling and L7 routing, the service mesh owns identity and retry policy, the database owns query time. When something goes wrong, the diagnostic question is always 'which hop is responsible' before 'what's broken.'"

## 2. Kubernetes pods restarting every few minutes

Structured debugging flow:

**Step 1: Confirm and characterize**

```bash
# How often, and what's the pattern?
kubectl get pods -n <ns> -w
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Restart count tells me how long this has been happening
kubectl get pods -n <ns> -o wide
```

Single pod restarting vs. all pods restarting tells me very different things.

**Step 2: Look at the pod's exit reason**

```bash
kubectl describe pod <pod>
# Look at "Last State" section:
#   - Reason: OOMKilled, Error, Completed
#   - Exit Code
kubectl get pod <pod> -o yaml | yq '.status.containerStatuses'
```

Common patterns:
- **OOMKilled (exit 137)**: container exceeded memory limit.
- **Error with non-zero exit**: application crash.
- **Completed with restart policy Always**: app exiting cleanly but kubelet restarting because it's expected to stay up.
- **Liveness probe failed**: kubelet killed the container.

**Step 3: Check logs from BOTH current and previous containers**

The critical command: `--previous` gives you the logs from the crashed container.

```bash
kubectl logs <pod> --previous          # last crash
kubectl logs <pod>                     # current
kubectl logs <pod> -c <container>      # multi-container
```

The most common mistake here is looking only at the current pod's logs, missing the actual crash.

**Step 4: Probes**

Probes are a leading suspect when "restarting every few minutes" matches the liveness probe cadence.

```bash
kubectl get pod <pod> -o yaml | yq '.spec.containers[0].livenessProbe'
```

Check:
- Is the probe path correct? Did it change in the latest deploy?
- Is `initialDelaySeconds` long enough for slow-starting apps (JVM, Spring Boot)?
- Is the timeout (`timeoutSeconds`) realistic for the endpoint's actual response time?
- Does the probe accidentally check a downstream dependency (DB, cache)? That's a common cascade-failure pattern — DB blip → all probes fail → all pods restart.

**Step 5: Resource constraints**

```bash
kubectl top pod <pod>
kubectl describe pod <pod> | grep -A5 "Limits"

# At node level
kubectl top nodes
kubectl describe node <node> | grep -A5 "Allocated resources"
```

OOMKill indicates memory limit too low. CPU throttling can cause request timeouts that *look* like crashes if the probe times out.

**Step 6: Recent changes**

```bash
kubectl rollout history deployment/<deploy>
kubectl describe deployment/<deploy> | head -50
```

What changed recently? Image tag, env vars, probe config, resource limits, ConfigMap?

**Step 7: Node-level health**

```bash
kubectl describe node <node>
# Look at Conditions: MemoryPressure, DiskPressure, PIDPressure

ssh <node>
dmesg -T | tail -50
journalctl -u kubelet -n 100
df -h
free -m
```

**Step 8: Dependencies**

If the app dies because it can't reach a downstream:
- DB available?
- Cache reachable?
- Service mesh sidecar healthy?
- DNS resolution working from the pod?

```bash
kubectl exec <pod> -- nslookup mydb.namespace.svc.cluster.local
kubectl exec <pod> -- nc -zv mydb 5432
```

**Common root causes ranked by likelihood:**

1. **Liveness probe failing** because of slow startup or accidental dependency check — fix by adding `startupProbe` and decoupling liveness from dependencies.
2. **OOMKill** from undersized limits — fix by VPA recommendations and right-sizing.
3. **App crashes on startup** because of missing config — check `--previous` logs.
4. **Recent deploy regression** — `kubectl rollout undo deployment/<deploy>` while investigating.
5. **Dependency unavailable** causing app to exit — check service connectivity from pod.
6. **Node-level pressure** evicting pods — check node conditions.

**Architect's framing:**
> "The single most diagnostic command is `kubectl logs --previous`. It tells you what the application said before dying. Combined with `kubectl describe pod` for the kubelet's view of the death, those two answer 80% of restart questions in under a minute. The trap is jumping to conclusions before reading those two outputs."

## 3. Latency increased after deployment

Walking through what I'd check first, in priority order:

**Step 1: Confirm the correlation**

Look at the deployment timeline vs latency graph. Is the increase actually correlated with the deploy, or coincidental?

```promql
# Latency p99
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

Overlay deployment annotations on the dashboard. If latency increased exactly at deploy time → deploy is the cause. If it increased an hour later → maybe traffic patterns changed.

**Step 2: Rollback first, investigate second**

For a customer-facing service with measurable latency regression, my first action is rollback. The system spends less time degraded; I have time to investigate calmly.

```bash
kubectl rollout undo deployment/<deploy>
# or in GitOps
git revert <bad-commit> && git push
```

This isn't avoiding investigation — it's reducing customer impact while investigating.

**Step 3: Diff the change**

What actually shipped?

```bash
# Image diff
kubectl rollout history deployment/<deploy> --revision=N
# Git diff between current and previous tags
git log <previous-tag>..<current-tag> --oneline
git diff <previous-tag>..<current-tag>
```

Categorize the changes:
- **Code changes** — algorithm changes, new dependencies, removed caching.
- **Config changes** — env vars, ConfigMaps, feature flags.
- **Dependency bumps** — library upgrades that might have behavioral changes.
- **Infrastructure changes** — resource limits, replicas, probe config.

**Step 4: Look at RED metrics broken down**

Latency in aggregate hides what's actually happening. Break it down:

```promql
# Per-endpoint p99
histogram_quantile(0.99, sum by (le, endpoint) (rate(http_request_duration_seconds_bucket[5m])))

# Per-pod
histogram_quantile(0.99, sum by (le, pod) (rate(http_request_duration_seconds_bucket[5m])))
```

If one endpoint regressed but others didn't → look at the code path for that endpoint.

If one pod is slow but others are fine → pod-level issue (bad node, image pull mid-deployment, etc).

**Step 5: Downstream dependencies**

```promql
# DB query duration
histogram_quantile(0.99, sum by (le) (rate(db_query_duration_seconds_bucket[5m])))

# External API calls
histogram_quantile(0.99, sum by (le, target) (rate(http_client_duration_seconds_bucket[5m])))
```

Did the new version start hitting the DB more (N+1 query introduced)? Did it switch to a slower downstream?

**Step 6: Connection pool / resource saturation**

```promql
# DB connection pool
hikari_connections_active
hikari_connections_pending

# Thread pool
jvm_threads_states_threads
```

New version might have lower pool size, or new code path holds connections longer.

**Step 7: GC and JVM behavior (for JVM apps)**

```promql
rate(jvm_gc_pause_seconds_sum[5m])
jvm_memory_used_bytes
```

A memory leak introduced in the new version causes longer GC pauses, which look like increased latency.

**Step 8: Trace a slow request end-to-end**

If distributed tracing is set up:

```bash
# Look at a recent slow request in Tempo/Jaeger
# Find the span that consumed the time
```

Tracing surfaces "we spent 2s in the DB call" vs "we spent 2s in the app" instantly.

**Step 9: Configuration deltas**

```bash
# Diff ConfigMaps
kubectl get cm -n <ns> -o yaml > current.yaml
git checkout <previous-commit> -- cm.yaml
diff current.yaml cm.yaml

# Diff resource specs
kubectl get deploy <name> -o yaml > current.yaml
```

Often the code is fine but config changed (smaller cache size, lower timeout, new logging level that flooded I/O).

**Step 10: Check resource throttling**

```bash
# CPU throttling
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu.stat | grep throttled
```

CPU limit too low + busy traffic → throttling → latency. Common when teams add CPU limits without realizing how aggressive CFS throttling is.

**Common root causes ranked by likelihood:**

1. **N+1 query** introduced by ORM change or refactor.
2. **Missing cache** — code removed a cache or changed cache key, causing more DB calls.
3. **Smaller connection pool** in new version.
4. **CPU/memory limit too tight** for new version's resource needs.
5. **Slower dependency** — new library version with different perf characteristics.
6. **More logging** flooding I/O.
7. **Memory leak** causing GC pressure.
8. **New feature flag** enabled an expensive code path.

**Architect's framing:**
> "Rollback first, investigate second — that's the operational discipline. Then methodically: confirm correlation with deploy time, diff what changed, break down RED metrics by dimension to find which slice regressed, follow that thread. The diff is usually small; one of those code changes is the cause. Distributed tracing turns this from forensics into a 30-second investigation, which is why I push hard for tracing on every service."

## 4. 10× traffic during peak — bottlenecks

Walking through the stack from edge to data layer:

**Edge / CDN tier**

Usually scales transparently:
- **CloudFront** auto-scales to billions of requests; rarely a bottleneck.
- **Route 53** scales similarly.
- **Risk:** misconfigured cache headers cause too many origin hits.

**Load balancer tier**

- **ALB** uses LCUs (Load Balancer Capacity Units) — scales but with a warm-up period for very sudden spikes.
- **Risk:** brief 5xx during scale-up if the spike is genuinely instant.
- **NLB** scales faster but at L4 only.
- **Mitigation:** for predictable traffic events (Black Friday, sale launches), AWS recommends pre-warming via support ticket.

**Compute tier (the most common bottleneck)**

- **Pods** — HPA scales replicas but takes time. Typical HPA reaction: 1-3 minutes to start scaling, plus pod startup time.
- **Nodes** — Cluster Autoscaler or Karpenter must provision new EC2 instances. Cluster Autoscaler: 60-90s. Karpenter: 20-60s.
- **Spot capacity** — if depending on Spot, capacity may not be available during peak.

Common compute bottlenecks:
- **CPU throttling** on pods with limits set too low.
- **Memory exhaustion** triggering OOMKills.
- **Thread pool exhaustion** in the app.
- **File descriptor limits** on nodes.
- **Connection pool exhaustion** for downstream calls.

**Cache tier**

- **ElastiCache Redis** scales but cluster mode resharding takes time. Pre-warm cluster before known traffic events.
- **Risk:** cache miss storms — cold cache during scale-up causes DB stampede.
- **Mitigation:** request coalescing, jittered cache TTLs, cache warming.

**Database tier (usually the hardest to scale)**

- **RDS writer** is the throughput ceiling. You can vertical-scale (downtime) or shard (architectural change).
- **Read replicas** scale reads but have replication lag.
- **Connection limits** — RDS instance classes have max connections. Hit this and queries fail.
- **Aurora** scales storage transparently but write throughput is still bound by the writer.
- **DynamoDB** can scale, but on-demand mode is required for unpredictable spikes; provisioned mode hits throttling.

**Async / messaging tier**

- **SQS** scales transparently for throughput.
- **Kafka (MSK)** — partition count is the scaling unit. Adding partitions mid-stream requires consumer rebalancing.
- **Risk:** consumer lag grows; downstream processing falls behind.

**External dependencies**

- **Third-party APIs** with rate limits. Even if you scale, the API doesn't.
- **Payment processors, identity providers** — your scaling doesn't matter if their rate limit hits.

**The order in which bottlenecks usually appear:**

1. **HPA / autoscaler lag** — 1-3 min before new pods are ready.
2. **Node provisioning** — 30-90s for new nodes.
3. **Database connection pool exhausted** — pod-level pool sized for normal traffic.
4. **Database CPU** at writer.
5. **External API rate limits**.
6. **Cache miss storm during cold start**.
7. **External API throttling**.

**Mitigations I'd implement:**

- **Pre-scaling for predictable events** — manually scale up before known traffic spikes via scheduled HPA or override.
- **Cluster overprovisioning** — pause pods or low-priority workloads that get evicted to make room for real workload spikes.
- **Connection pool sizing** for peak, not average.
- **Read replicas pre-provisioned**.
- **Cache warming jobs** run before traffic spike.
- **Rate limiting at the edge** so the system degrades gracefully rather than collapses.
- **Circuit breakers** on downstream calls.
- **Graceful degradation** — turn off expensive features under load (recommendations, related-products).
- **Async-ify where possible** — turn synchronous calls into queued processing during spikes.

**Architect's framing:**
> "10× traffic finds the weakest link. The compute tier is the obvious one but scales easiest. The hidden bottlenecks are usually database write throughput, connection pools, external API rate limits, and cold cache. The discipline isn't predicting which will break — it's load-testing the system at 10× ahead of time to find them. For known events like sale launches, pre-warming is the operational answer."

## 5. Monitoring didn't detect an outage for 10 minutes

This is a postmortem-style question. Walking through the layered failure analysis.

**Step 1: What "didn't detect" means**

Several variants:
- Alert never fired.
- Alert fired but didn't page anyone.
- Alert fired but went to a channel nobody watched.
- Alert fired but was misclassified as low priority.

Each has different root causes. Establish which one first.

**Step 2: Common failure modes**

**A. Alert threshold wrong**

Alert was on "5xx rate > 5%" — but during the outage, the service returned 200 with empty body (broken UI), not 5xx. So alert never fired.

Fix: alerts based on user-visible outcomes (synthetic monitoring, business KPIs) rather than just HTTP status codes.

**B. Metric source went down with the system**

Monitoring agent runs on the same nodes that are failing. Pods can't push metrics. Prometheus can't scrape unhealthy pods. The system became "metric-dark."

Looking at the dashboard during the outage shows flat lines — looks like "everything fine" but actually means "no data."

Fix:
- **Stale-metric alerts** — alert when expected metrics stop arriving (`absent(http_requests_total{job="api"})`).
- **External synthetic monitoring** (Pingdom, CloudWatch Synthetics) — measures from outside the system.
- **Heartbeat metrics** that should always be present.

**C. Alert routing misconfigured**

Alert fired but was routed to the wrong channel or to a paused PagerDuty service.

Fix: regular fire drills — deliberately trigger test alerts to verify routing actually reaches the on-call.

**D. Alert noise** — boy who cried wolf

Hundreds of low-value alerts per day caused on-call to mute or ignore. The critical alert was lost in the noise.

Fix:
- **SLO-based alerting** that only pages on user-visible impact.
- **Alert review meetings** — kill alerts that didn't lead to action.
- **Symptom-based alerts** (user experience), not cause-based (CPU > 80%).

**E. Threshold too lenient or window too long**

Alert: "Error rate > 5% for 15 minutes" — outage was 10 minutes, so alert didn't fire.

Fix: multi-window burn-rate alerts (short window for acute, long window for chronic). The Google SRE workbook approach.

**F. Wrong dimensions**

Alert on "error rate across all of /api/* > 1%" — but only `/api/payments` was broken at 50% error rate. Aggregated across all endpoints, it looked like 2% — below threshold but not flagged urgently.

Fix: alert on per-endpoint SLOs for tier-1 endpoints, not just aggregated metrics.

**G. Cascading failure obscured the signal**

The outage triggered so many other alarms (downstream services failing) that the on-call missed which one was the actual root cause for 10 minutes.

Fix:
- **Alert dependencies / inhibition rules** in Alertmanager — suppress downstream alerts when upstream alerts fire.
- **Service maps** showing dependencies so the on-call can quickly identify root cause vs symptom.

**H. Customer-perceived outage that's not visible in technical metrics**

Sometimes the system is "up" (returning 200s) but actually broken (returning wrong data, broken JavaScript, payment flow not completing). Technical metrics look fine; users are impacted.

Fix:
- **Business metric alerts** — orders/sec, signups/min. If those drop, something's wrong even if the system "looks healthy."
- **Real User Monitoring (RUM)** — catches client-side issues that server-side monitoring misses.
- **Synthetic transactions** that complete an actual user journey.

**Step 3: Process failure**

Sometimes monitoring detected but humans didn't react:
- On-call ack'ed and then forgot.
- Paging fatigue.
- Vacation coverage gap.

These aren't monitoring problems — they're process problems. Distinct fix.

**Step 4: The postmortem**

A 10-minute detection gap warrants a postmortem regardless of severity. Output:

1. **Timeline** — when did the outage actually start, when did first signal appear, when did the alert fire (if at all), when did the on-call notice.
2. **Root cause of the detection gap** — which of the above categories.
3. **Action items**:
   - Add the missing alert.
   - Tune the threshold.
   - Add synthetic monitoring for this code path.
   - Improve runbook so future on-call recognizes the signal faster.

**Architect's framing:**
> "Monitoring isn't 'do we have alerts' — it's 'do alerts reliably surface user impact within a target time.' MTTD (Mean Time to Detect) is a measurable SLO. When MTTD exceeds target, that itself is an incident worthy of postmortem. Common root causes: alerts based on cause not symptom, no external synthetic monitoring, alert fatigue causing real signals to be ignored, and missing alerts for things like 'metrics stopped arriving.' The fix is layered: SLO-based alerts for user-visible symptoms, synthetic monitoring from outside the system, stale-data alerts, and disciplined alert review to kill noise."

---

# 🎯 Round 2 — Production Debugging

## 1. Nodes healthy, users get 502 errors

502 (Bad Gateway) means an intermediate proxy got an invalid response from the upstream. Working through systematically.

**Step 1: Where exactly is the 502 coming from?**

```bash
curl -v https://app.example.com/ 2>&1 | grep -i "server"
# Server: awselb/2.0       → ALB emitted the 502
# Server: nginx            → in-cluster ingress emitted it
# Server: cloudfront       → CloudFront emitted it
```

Different sources, different root causes.

**Step 2: ALB-emitted 502**

ALB returns 502 when it got an invalid response from the target. Common causes:

- **Target returned non-HTTP response** — app crashed mid-response, returned malformed bytes.
- **Connection reset by target** — app crashed and closed the connection.
- **Keep-alive timeout mismatch** — ALB's idle timeout is longer than app's; app closes idle connection while ALB tries to reuse.
- **Target health check passing but target actually unhealthy** — e.g., `/health` returns 200 but `/api` is broken.

```bash
# Check target group health
aws elbv2 describe-target-health --target-group-arn <arn>
# All healthy? But 502s? → keep-alive or per-request failure

# Check ALB access logs for the specific 502
# Athena query:
SELECT * FROM alb_logs
WHERE elb_status_code = '502'
ORDER BY time DESC
LIMIT 100;
# Look at target_status_code and target_processing_time
```

If `target_status_code = '-'` and `target_processing_time = -1`, the target never responded properly (connection reset, crash).

**Step 3: NGINX-emitted 502**

NGINX returns 502 when its upstream (backend pod) didn't respond properly:

```bash
kubectl logs -n ingress-nginx <controller-pod> | grep "502"
# Look for:
#   upstream prematurely closed connection
#   connect() failed (111: Connection refused)
#   upstream timed out
```

**Step 4: Pod-level investigation**

```bash
# Are pods actually serving traffic?
kubectl get pods -n <ns> -l app=<app>
kubectl get endpoints -n <ns> <service>
# Endpoints should list pod IPs. If empty or wrong → service selector issue or all pods unhealthy

# Are pods restarting?
kubectl get pods -n <ns> -l app=<app> -o wide
# Restart count? Recent restarts mean intermittent crashes → 502s during the crashes

# Logs from a pod
kubectl logs <pod> -n <ns> | tail -100
kubectl logs <pod> -n <ns> --previous   # if recently restarted

# Resource pressure
kubectl top pod -n <ns> -l app=<app>
```

**Step 5: Connection-level issues**

Even if pods look fine individually, connection issues cause 502s:

- **SNAT port exhaustion** on EKS — high outbound connection rate exhausts NAT ports, causing intermittent connection failures.
- **Connection pool exhaustion** in the app — incoming requests succeed but app can't open a downstream connection.
- **DNS resolution failures** inside pods — CoreDNS overloaded.
- **Keep-alive mismatch** — covered above.

```bash
# Check SNAT ports
aws ec2 describe-nat-gateways --filter Name=state,Values=available
# Each NAT GW has ~55,000 ports per destination IP combo

# DNS
kubectl exec <pod> -n <ns> -- nslookup mydb.namespace.svc.cluster.local
kubectl logs -n kube-system -l k8s-app=kube-dns | tail
```

**Step 6: Downstream dependencies**

If pods are serving but their downstream calls fail:
- Pod returns an error 502 to NGINX, which propagates.
- Could be DB unavailable, third-party API timing out, cache down.

```bash
# Check app error logs for downstream failures
kubectl logs <pod> -n <ns> | grep -i "error\|failed\|timeout"
```

**Step 7: Cascade pattern**

Sometimes 502s come and go in waves. Pattern: pod crashes → pod restarts → traffic rebalances → other pods overloaded → more crashes.

Looking at:
- Pod restart count over time.
- Service Endpoints membership changes.
- 502 rate correlated with deploy or pod-churn events.

**Common root causes ranked by likelihood:**

1. **Keep-alive timeout mismatch** — ALB idle timeout 60s, app idle 30s. ALB tries to reuse closed connection.
2. **App crashing intermittently** under load — exit during request → 502.
3. **Health check too lenient** — pods pass health check but actual endpoints fail.
4. **Downstream timeout** — app waits, ALB times out at 60s, returns 502.
5. **Pod-to-pod connectivity issues** — NetworkPolicy blocking, CoreDNS issues.
6. **SNAT exhaustion** for high-outbound-traffic clusters.

**Mitigation patterns:**

```yaml
# Tune timeouts to match
# In Spring Boot
server.tomcat.connection-timeout: 60s
server.tomcat.keep-alive-timeout: 65s   # slightly longer than ALB idle (60s)

# Annotation on ALB
service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
```

**Architect's framing:**
> "502 means a proxy got an invalid response from upstream. The diagnosis order: which proxy emitted it (header), what does that proxy's log say about the upstream failure, is the upstream actually responding, why not. Keep-alive mismatch and app-side crashes are the most common at scale. Healthy nodes don't mean healthy pods, and healthy pods don't mean healthy responses — each layer needs its own observability."

## 2. Redis latency spikes during peak

Walking through the diagnostic flow:

**Step 1: Characterize the spike**

```promql
# Redis command latency p99
histogram_quantile(0.99,
  sum by (le, command) (rate(redis_command_duration_seconds_bucket[1m]))
)

# Is it certain commands? All commands?
```

If it's all commands → infrastructure or capacity issue. If it's specific commands → command-specific (slow query, large value).

**Step 2: Check ElastiCache / Redis metrics**

```
CPUUtilization              — Redis is single-threaded, one core matters most
EngineCPUUtilization        — actual Redis engine, not the host
NetworkBytesIn/Out          — bandwidth saturation?
CurrConnections             — connection storm?
NewConnections              — high churn?
Evictions                   — running out of memory, evicting keys
ReplicationLag              — if using replicas
```

**Common patterns:**

**A. EngineCPU saturated**
Redis is single-threaded. If EngineCPU is 100%, you're maxed out regardless of how many cores the instance has. Need to either shard (cluster mode), scale up (bigger instance), or reduce command rate.

**B. Slow commands blocking the event loop**
Single-threaded Redis means one slow `KEYS *`, `SUNION`, `SORT`, or `LRANGE` on a huge list blocks every other client.

```bash
# Check slow log
redis-cli SLOWLOG GET 100
redis-cli SLOWLOG RESET
```

Look for `KEYS`, large `SORT`, large `SUNION`, anything operating on big collections.

**C. High connection churn**
Apps not using connection pools open/close connections per request. Each connection costs CPU.

```bash
redis-cli INFO clients
# Look at:
#   connected_clients
#   total_connections_received   ← high churn rate is bad
```

Fix in the application: connection pool with appropriate size and timeout settings.

**D. Memory pressure / evictions**
If `maxmemory-policy` is `allkeys-lru` and memory is full, every write triggers an eviction (LRU sampling overhead).

```bash
redis-cli INFO memory
# Look at:
#   used_memory
#   used_memory_peak
#   maxmemory
#   maxmemory_policy
#   evicted_keys
```

If evictions are happening, cache hit ratio drops, application makes more DB calls, which often cascades.

**E. Hot keys**
A single key being hammered. All requests for that key serialize behind one another.

```bash
redis-cli --hotkeys
# Or with newer Redis:
redis-cli CLUSTER COUNTKEYSINSLOT <slot>
```

Fix: split the hot key (sharded counter), use a local in-process cache for read-heavy hot keys, use Redis cluster mode and distribute.

**F. Network bandwidth saturation**
Especially with large values. ElastiCache instance has a fixed network ceiling.

Check `NetworkBytesIn/Out` against the instance's bandwidth spec. If close to ceiling, you're network-bound.

**G. Replication lag affecting reads**
If reading from replicas and lag spikes, queries hit slow replicas.

**H. Backup taking resources**
ElastiCache nightly snapshots can cause brief latency spikes. Confirm via the snapshot schedule.

**Step 3: Application-side investigation**

```bash
# What's the app doing with Redis?
# Trace a slow request — what Redis commands were issued?
```

Common app-side issues:
- **N+1 cache calls** — fetching items one at a time instead of MGET batch.
- **Large values** — serializing huge objects as single Redis values.
- **Wrong data structures** — using a single SET with millions of members instead of multiple smaller SETs.
- **Cache stampede** — many requests miss simultaneously and all rebuild the cache, hammering both Redis and the DB.

**Step 4: Mitigation patterns**

- **Pipeline commands** to reduce round-trips.
- **MGET / MSET** for batch operations.
- **Cluster mode** if hitting single-instance limits.
- **Connection pooling** with reasonable pool size.
- **Read replicas** for read-heavy workloads.
- **Cache stampede prevention** — request coalescing, jittered TTLs, probabilistic early expiration.
- **Eliminate or rewrite slow commands** — never `KEYS *` in production.

**Step 5: Long-term architectural patterns**

- **Local in-process cache** for hot data (Caffeine in Java, lru-cache in Node) — Redis becomes the second tier.
- **Multi-tier caching** — local → Redis → DB.
- **Async cache warming** for predictable access patterns.

**Architect's framing:**
> "Redis is single-threaded — once EngineCPU saturates, no amount of instance scaling helps. The diagnostic path: EngineCPU first (is the engine maxed?), slow log second (which commands?), eviction/memory third (are we thrashing?). Most production Redis latency issues at scale trace to slow commands on the wrong data structure, connection churn from missing pools, or hot keys that need to be sharded. The architectural answer for cache-heavy systems is multi-tier caching — local cache in process, Redis for shared cache."

## 3. HPA didn't scale despite high CPU

Systematic check, in priority order:

**Step 1: What does HPA actually see?**

```bash
kubectl describe hpa <name> -n <ns>
# Look at the "Metrics" section:
#   - resource cpu on pods (as a percentage of request):  85% (170m) / 70%
#   - Current/Desired replicas
# Look at "Conditions":
#   - AbleToScale, ScalingActive, ScalingLimited

# Events at the bottom are crucial:
kubectl describe hpa <name> -n <ns> | tail -30
```

The events tell you in plain English why HPA isn't scaling.

**Step 2: Common root causes**

**A. Pod doesn't have CPU request set**

HPA computes utilization as `usage / request`. Without a request, there's no denominator — HPA can't compute utilization.

```bash
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'
# If empty or no "requests" → that's the problem
```

Fix: set CPU requests on the pod template.

**B. Metrics server not running or unhealthy**

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl logs -n kube-system <metrics-server-pod>

# Can it actually report metrics?
kubectl top pod <pod>
# If "Unknown" or error, metrics-server is broken
```

**C. HPA at maxReplicas**

```bash
kubectl get hpa <name> -o yaml | grep -E "min|max|current|desired"
```

If `desired_replicas == max_replicas` and HPA wanted more, the `ScalingLimited` condition would be true.

**D. Stabilization window**

HPA v2 has scale-up stabilization. If `behavior.scaleUp.stabilizationWindowSeconds` is set high (e.g., 5 min), HPA waits before scaling up even with high CPU.

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 300   # waits 5min before scaling
```

For aggressive scale-up:
```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
```

**E. Cool-down from recent scale event**

If HPA scaled down recently, the default 5-min cool-down may be preventing scale-up.

**F. Multiple metrics, one unavailable**

If HPA uses both CPU and a custom metric, and the custom metric returns errors, HPA may not act.

**G. Custom metrics adapter missing**

If HPA references custom metrics that aren't being collected (Prometheus adapter not installed or misconfigured), it can't compute desired replicas.

**H. RBAC issue**

HPA needs permission to patch the Deployment. If RBAC was changed and the HPA controller's service account lost permission, scaling fails.

```bash
kubectl logs -n kube-system -l component=kube-controller-manager | grep -i hpa
# Look for "forbidden" errors
```

**I. CPU utilization vs. CPU usage confusion**

HPA's `averageUtilization` is *percent of requests*. If pod requests 1 CPU and uses 0.5 CPU, that's 50% utilization. If utilization target is 70%, no scale-up.

But the user might be looking at *raw CPU usage* on the node (e.g., 90% of node CPU) and assume HPA should scale. These are different things.

**J. Pod template hash matches existing RS**

This is more subtle: if HPA tries to scale but the Deployment can't roll due to some other constraint (PDB blocking, resource quota, no nodes available), scaling stalls.

**K. ResourceQuota at the namespace level**

```bash
kubectl get resourcequota -n <ns>
```

If quota cap is reached, new pods can't be created regardless of what HPA wants.

**Step 3: Verify the chain end-to-end**

```bash
# 1. Are metrics flowing?
kubectl top pod <pod>

# 2. Is HPA reading them?
kubectl get hpa <name> -o yaml | grep currentMetrics -A10

# 3. Is HPA computing correctly?
kubectl describe hpa <name>

# 4. Is the deployment being patched?
kubectl get deploy <name> -o yaml | grep replicas

# 5. Are pods being scheduled?
kubectl get pods -n <ns> -l app=<app>
```

**Architect's framing:**
> "HPA failures almost always trace to one of three: missing CPU request (no denominator), metrics-server broken, or stabilization window throttling. `kubectl describe hpa` events tell you which one. Less common: RBAC, ResourceQuota, or wrong metric type. The systematic mistake is debugging by changing config instead of reading the HPA's own status, which explicitly explains why it isn't scaling."

## 4. Memory leak in deployment — early signals

Signals ranked from earliest to most obvious:

**Earliest signals (would catch a leak in hours):**

**1. RSS growth without level-off**

In a healthy app under steady traffic, memory grows during startup and warm-up, then plateaus. A leak shows monotonic growth that doesn't plateau.

```promql
# Memory trend
container_memory_working_set_bytes{pod=~"myapp.*"}

# Slope over the last hour
deriv(container_memory_working_set_bytes{pod=~"myapp.*"}[1h])
# Positive and not decreasing → potential leak
```

Alert: memory derivative consistently positive over 24h.

**2. Increasing GC pause frequency (JVM specifically)**

```promql
rate(jvm_gc_pause_seconds_count{action="end of major GC"}[5m])
# Increasing trend over time → heap pressure
```

As heap fills, major GC runs more frequently. Each GC reclaims less. This pattern precedes the OOM by hours or days.

**3. Increasing GC duration**

```promql
rate(jvm_gc_pause_seconds_sum[5m])
/
rate(jvm_gc_pause_seconds_count[5m])
# Average GC pause time — increasing → leak symptoms
```

**4. Old generation heap usage trending up**

```promql
jvm_memory_used_bytes{area="heap", id="G1 Old Gen"}
# Trending up over time after each GC → objects being retained that shouldn't be
```

This is the cleanest leak signal in JVMs: even after GC, heap stays elevated.

**5. Native memory growth (non-heap)**

```promql
process_resident_memory_bytes - jvm_memory_used_bytes{area="heap"}
# Native portion of RSS
```

If JVM heap is stable but RSS grows, the leak is in native memory (direct buffers, JNI, native libraries). Pure-Java leak detection misses this.

**Mid-term signals (hours to days):**

**6. Decreasing throughput**

GC stop-the-world pauses reduce effective throughput. Even before OOM, requests per second drops.

```promql
rate(http_requests_total[5m])
# Trending down without traffic decrease → app slowing down
```

**7. Increasing latency**

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
# Climbing p99 → GC pauses lengthening request times
```

**8. Connection pool / thread pool saturation**

Leaked objects might be holding connections or threads. Pool exhaustion follows.

```promql
hikari_connections_active
jvm_threads_states_threads{state="blocked"}
```

**Late signals (about to break):**

**9. Approaching memory limit**

```promql
container_memory_working_set_bytes / container_spec_memory_limit_bytes
# > 0.8 → close to OOM
```

**10. OOMKill (the post-mortem signal)**

```bash
kubectl describe pod <pod>
# Last State: Terminated, Reason: OOMKilled, Exit Code: 137
```

Once you see OOMKills, the leak has been there for a while and only now manifested as crash-and-restart cycles.

**Comprehensive alerting strategy:**

```yaml
groups:
- name: memory-leak-detection
  rules:
  # Earliest: long-term memory growth
  - alert: PossibleMemoryLeak
    expr: |
      deriv(
        container_memory_working_set_bytes{
          container="app", pod=~"myapp.*"
        }[6h]
      ) > 1024 * 1024   # growing >1MB/sec over 6h
    for: 1h
    labels: { severity: ticket }
    annotations:
      summary: "Memory growing in {{ $labels.pod }} — possible leak"

  # JVM heap pressure
  - alert: JVMOldGenGrowing
    expr: |
      deriv(
        jvm_memory_used_bytes{
          area="heap", id="G1 Old Gen", pod=~"myapp.*"
        }[2h]
      ) > 0
    for: 30m
    labels: { severity: ticket }

  # GC time increasing
  - alert: GCTimeHigh
    expr: |
      (rate(jvm_gc_pause_seconds_sum[5m]) / rate(jvm_gc_pause_seconds_count[5m])) > 0.5
    for: 10m
    labels: { severity: page }

  # Near OOM
  - alert: MemoryNearLimit
    expr: |
      container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.85
    for: 5m
    labels: { severity: page }
```

**Investigative tools when alerts fire:**

- **Heap dump on OOM** — `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dumps/`
- **Periodic heap dumps** — Cron-triggered every hour, analyzed offline.
- **JFR (Java Flight Recorder)** — continuous, low-overhead profiling.
- **Memory profiler (async-profiler, YourKit, JProfiler)** for live diagnosis.
- **Pyroscope / Grafana Profiles** for continuous profiling in production.

**Architect's framing:**
> "The earliest signal is monotonic memory growth without plateau — visible hours or days before the OOM. The next signals are GC behavior changes — frequency, duration, old-gen retention. By the time you see OOMKill, the leak has been brewing. Investment in heap-dump-on-OOM + continuous profiling means leak diagnosis goes from 'why did pod crash' (forensics) to 'why is heap growing' (proactive). Most teams have OOMKill alerting; few have leak-detection alerting."

---

# 🎯 Round 3 — Engineering Depth

## 1. Designing systems that fail gracefully

Graceful degradation means: when something fails, the system continues operating at reduced capability rather than collapsing. This is fundamentally a *design* property, not something you bolt on afterwards.

**Core principles:**

**1. Identify what's critical vs nice-to-have**

Not every feature is equal. In an e-commerce checkout:
- **Critical**: take payment, create order.
- **Important**: send confirmation email.
- **Nice-to-have**: show recommended products, update loyalty points.

Design so failure of nice-to-have features doesn't break critical paths. If the recommendations service is down, checkout still works.

**2. Circuit breakers**

Stop calling downstream services that are clearly failing. Prevents cascade failures where one slow service drags down all callers.

```java
// Resilience4j example
CircuitBreaker breaker = CircuitBreaker.of("recommendations",
  CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .slowCallRateThreshold(50)
    .slowCallDurationThreshold(Duration.ofMillis(500))
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .build()
);

return breaker.executeSupplier(() -> recommendationsClient.fetch())
  .orElse(Collections.emptyList());   // fallback to empty
```

State machine: closed (healthy) → open (failing, short-circuit) → half-open (probing). Failure doesn't propagate; users see empty recommendations instead of failed checkouts.

**3. Timeouts and retries**

Every external call has a timeout. Default to "no timeout" means default to "system hangs forever when downstream is slow."

```yaml
# Sensible defaults
connect-timeout: 1s
read-timeout: 3s

# Retries with backoff and jitter
retries: 3
backoff:
  initial: 100ms
  multiplier: 2
  max: 5s
  jitter: 0.3
```

Retries with jitter prevent thundering herds.

**4. Bulkheads**

Isolate resources so failure in one area can't exhaust resources used by another.

- Separate connection pools per downstream service.
- Separate thread pools per workload type.
- Separate request queues with bounded capacity.

If downstream A is slow, only the A pool is exhausted; B keeps working.

**5. Graceful degradation paths**

Pre-design what "degraded" looks like:
- **Cache miss → serve stale data** with a warning.
- **Recommendation service down → return empty list.**
- **Search down → fallback to category browsing.**
- **Payment provider A down → failover to provider B.**
- **Image CDN down → serve from origin with degraded latency.**

This must be in the design phase, not invented during an incident.

**6. Feature flags for emergency degradation**

```python
if feature_flag("expensive_recommendation_algorithm"):
    return ml_recommendations(user)
else:
    return simple_popularity_recommendations()
```

When ML inference latency spikes, flip the flag, switch to cheap recommendations. No deploy needed.

**7. Load shedding**

Under extreme load, reject some requests gracefully rather than collapse entirely:
- Rate limiting at the edge (CloudFront, API Gateway, NGINX).
- Priority queues — drop low-priority requests first.
- Adaptive concurrency limits — reduce concurrency as latency rises.

Better to serve 80% of users well than 100% of users poorly.

**8. Async patterns for non-critical paths**

```
Order placed → Queue (durable) →
  ├── Charge payment (sync, critical)
  ├── Send confirmation email (async via SQS)
  ├── Update loyalty points (async)
  └── Refresh recommendations (async)
```

If the email service is down, the order still completes. Email retries from the queue when the service recovers.

**9. Idempotency**

Retries are meaningless without idempotency. Every write operation should have an idempotency key so it's safe to retry without double-effect.

```http
POST /api/payments
Idempotency-Key: 7f8e9d10-payment-attempt-1
```

**10. Health checks that mean something**

Health endpoints should reflect *real* health:
- Dependent on internal queue depth, downstream availability, cache reachability.
- Not just "process is running."

But carefully — health check that depends on downstream causes cascading failures. Use **deep health** for operator dashboards and **shallow health** for load balancer probes.

**11. Default to denying gracefully**

When in doubt: fail closed (deny) for security-sensitive, fail open (allow) for non-critical features. Either way, **fail consistently** so callers know what to expect.

**12. Observability for graceful degradation**

You need to know when degradation is happening:
- Metric: "circuit breaker open" count.
- Metric: "fallback path taken" count.
- Alert: "more than X% of requests using degraded path."

Without this, degradation is invisible until it crosses into outage.

**Practical example — checkout flow:**

```
User clicks "place order"
    │
    ▼
[Validate cart] ──────── DB down? → return cached cart with warning
    │
    ▼
[Reserve inventory] ──── timeout? → retry with backoff; if still fails, page on-call
    │
    ▼
[Charge payment] ─────── provider A down? → failover to provider B
    │
    ▼
[Create order in DB] ─── critical path; cannot fail
    │
    ├──→ [Send email] ───────── async via SQS; retries
    ├──→ [Update loyalty] ──── async; non-blocking
    └──→ [Fraud check async] ─ async; flag for review if fails
    │
    ▼
Return success to user
```

Critical path is short (validate → reserve → charge → create order). Everything else is async. A failure in non-critical paths doesn't break the order.

**Architect's framing:**
> "Graceful degradation is a design discipline, not a feature. The skill is knowing which parts are critical and which can degrade — and that conversation happens during design, not during an incident. The patterns are well-known: timeouts, circuit breakers, bulkheads, async non-critical paths, feature flags, load shedding. The hard part is the cultural discipline of asking 'what happens if this dependency fails?' for every dependency, every time."

## 2. Reducing MTTR in production

**MTTR (Mean Time To Recovery)** is the time from outage start to restoration. Reducing it has compounding value — every minute saved is one less minute of customer impact.

Breaking it down: MTTR = MTTD (detect) + MTTI (investigate) + MTTM (mitigate) + MTTR-recover (verify).

Each component has different levers.

**Reducing MTTD — Detection time**

- **SLO-based alerting** based on user-visible symptoms (error rate, latency), not infrastructure (CPU).
- **External synthetic monitoring** that catches issues internal monitoring misses.
- **Multi-window burn-rate alerts** — short window catches acute issues, long window catches slow degradation.
- **Business KPI alerts** — orders/sec dropping is a faster signal than 5xx rate sometimes.
- **Aliveness alerts** — alert when expected metrics stop arriving (`absent()`).
- **Stale data alerts** — last-update timestamp on dashboards so flat graphs don't masquerade as healthy.

Goal: MTTD < 2 minutes for tier-1 issues.

**Reducing MTTI — Investigation time**

The longest MTTR contributor for most incidents. Levers:

- **Distributed tracing** — answers "where is the time being spent" in seconds, not minutes.
- **Structured logging** with correlation IDs — find all log lines for a slow request in one query.
- **Per-service runbooks** linked from alerts. "API has 5xx spike" alert → click → runbook with diagnostic queries.
- **Dependency maps** — see at a glance which downstreams are affected.
- **Recent-change dashboards** — what deployed in the last hour, what configs changed.
- **Pre-built incident dashboards** per service so the on-call doesn't build queries during an incident.
- **Correlation tools** — automatic anomaly detection that points at "this metric is unusual right now."

Goal: identify root cause within 10 minutes.

**Reducing MTTM — Mitigation time**

This is where rollback culture matters.

- **One-click rollback** — `kubectl rollout undo`, `helm rollback`, GitOps `git revert`. Should take < 60 seconds.
- **Feature flag kill switches** — disable risky features without redeploy.
- **Pre-tested runbooks** — manual mitigation steps tested in DR drills, not invented during incidents.
- **Automated runbook execution** — for known patterns, automate the mitigation (Rundeck, AWS SSM Documents).
- **Decision frameworks** — rollback first vs investigate first? Default: rollback first for any tier-1 customer-impacting issue.

Goal: mitigation within 5 minutes of identifying root cause.

**Reducing MTTR-recover — Verification time**

- **Automated synthetic tests** post-mitigation to verify recovery.
- **SLO dashboards** with real-time burn-rate showing recovery.
- **Customer impact metrics** clearly visible (orders/sec, error rate by region).

**Cross-cutting practices:**

**1. Blameless postmortems with action items**

Every incident produces tracked action items. MTTR improvements compound only if past incidents inform future architecture.

**2. Game days**

Quarterly DR drills, chaos exercises. Practiced incidents have faster MTTR than novel ones.

**3. On-call training**

New on-call shadows experienced on-call for several rotations. Documented decision trees. Runbooks reviewed regularly.

**4. Reduced cognitive load during incidents**

- **Incident commander role** — one person directing, others investigating.
- **Standardized comm channels** — every incident has a Slack channel with the same template.
- **Status pages** — automated, not manually updated during incidents.

**5. Pre-built mitigation tooling**

- **Rollback buttons** in deployment dashboards.
- **Traffic shifting tools** (Route 53, ALB weighted target groups) at the ready.
- **Database failover scripts** tested and documented.

**6. Eliminate manual steps**

Manual = slow + error-prone. Automate any step that's been done more than twice during incidents.

**7. Observability investments**

- **Continuous profiling** — when investigating "why is service slow," profile is ready.
- **Pre-aggregated dashboards** — don't query raw data during incidents.
- **Audit trails** — who deployed what, who ran what command.

**8. Architectural patterns that enable fast recovery**

- **Immutable infrastructure** — replace, don't repair.
- **Blue-green or canary deployments** — rollback is just a traffic flip.
- **Multi-region** — fail over instead of fixing.
- **Idempotent operations** — safely retry without double-effect.

**MTTR metric tracking:**

Measure and review:
- MTTR per quarter, trending down or up.
- MTTR by severity tier.
- Breakdown: MTTD + MTTI + MTTM.

If MTTI dominates, invest in observability. If MTTM dominates, invest in mitigation tooling. If MTTD dominates, invest in alerts.

**Architect's framing:**
> "MTTR isn't one thing — it's four phases, each with different levers. Detection is alerts; investigation is observability; mitigation is rollback tooling; verification is monitoring. The biggest wins for most teams are in MTTI — distributed tracing alone often cuts MTTR by half. The cultural piece: every incident either produces an MTTR improvement, or we're not learning. Postmortem action items should be tracked with the same rigor as feature work — MTTR improvement is product work, not toil."

## 3. Preventing repeated incidents in a platform team

The fundamental insight: repeated incidents indicate systemic gaps, not bad luck. The goal is structural prevention, not "tell people to be more careful."

**Layer 1: Blameless postmortems with action items**

Non-negotiable for any non-trivial incident. Structure:

- **What happened** — timeline with timestamps.
- **What was the impact** — customer-visible, business metrics.
- **Why it happened** — root cause analysis (5-whys, fishbone). Look for systemic causes, not "operator error."
- **What did we do well** — preserve the practices that worked.
- **What did we miss** — gaps in monitoring, runbooks, design.
- **Action items** with owners, deadlines, and tracked completion.

Blameless means: assume people made the best decision they could with the information available. Look at why the system allowed the mistake.

**Layer 2: Action item tracking**

Action items must close. The single biggest postmortem failure mode is producing great action items that never ship.

- Tracked in Jira/Linear like any other work.
- Quarterly review: action items overdue >30 days get escalated.
- Action items have priority tiers — P0 must ship in current sprint, P1 in current quarter.

**Layer 3: Pattern detection across incidents**

After enough incidents, patterns emerge. Quarterly review:

- Same root cause appearing repeatedly? → invest in fixing the underlying class.
- Same service in multiple incidents? → architectural review of that service.
- Same on-call pattern (Monday morning deploys)? → process change.

**Layer 4: Pre-incident investments**

Most incidents are preventable with upfront engineering. The hard part is justifying preventive work that has no immediate visible payoff.

- **Capacity planning** before traffic events.
- **Chaos engineering** to surface unknown failure modes.
- **DR drills** that find latent issues.
- **Load testing** for new services before launch.
- **Service Level Objectives** with error budgets to allocate engineering time between reliability and features.

**Layer 5: Process for high-risk changes**

Not every change needs heavy process, but high-risk changes need it:

- **Change Review Board** for schema migrations, infrastructure changes, security changes.
- **Pre-deploy checklists** for production changes.
- **Mandatory canary** for tier-1 services.
- **Pre-deploy approvals** for production database changes.
- **Backout plan documented** for any non-trivial change.

This isn't bureaucracy if calibrated to risk — light process for low-risk, heavy process for high-risk.

**Layer 6: Architectural improvements**

Some incidents prevent themselves only via architecture:

- **Multi-region** if single-region outages keep happening.
- **Idempotency everywhere** if retry-induced data corruption recurs.
- **Service mesh with circuit breakers** if cascade failures recur.
- **Better isolation** if multi-tenant noisy-neighbor recurs.
- **Replace fragile components** that keep breaking.

**Layer 7: Observability investments**

If incidents recur because nobody saw them coming:

- **Add the missing metric/alert** identified in postmortems.
- **Synthetic monitoring** for previously-blind code paths.
- **Distributed tracing** if "we couldn't tell where the time went" appears repeatedly.
- **Continuous profiling** if performance issues are recurring.

**Layer 8: Runbook investments**

If incidents recur because operators didn't know how to respond:

- **Runbooks linked from alerts** — alert annotation includes runbook URL.
- **Runbook review cadence** — runbooks tested in DR drills, updated when stale.
- **Onboarding documentation** for new on-call.

**Layer 9: Training and team practices**

- **Game days** quarterly — practice incident response.
- **Cross-team on-call shadowing** — share knowledge.
- **Postmortem readouts** across the org — lessons aren't locked to one team.
- **Reliability office hours** — platform team available for design reviews.

**Layer 10: Culture and incentives**

The cultural piece is what makes everything else work:

- **Reliability is a feature, not a tax** — leadership treats SRE work with the same priority as product work.
- **Blameless culture** — engineers report issues without fear, leading to better data.
- **SLO compliance reviewed in product roadmaps** — error budget burn slows feature shipping.
- **Reliability metrics in performance reviews** — engineers rewarded for reliability work, not just shipping.

**Anti-patterns that perpetuate repeated incidents:**

- **Postmortems with no action items** — fundamental cause not addressed.
- **Action items that don't ship** — same issue happens again.
- **Blame culture** — people hide issues; you lose data.
- **"Fix it manually" repeatedly** — symptoms patched, not causes.
- **Ignoring small incidents** — they're warnings; the next one is bigger.
- **"It only happens occasionally"** — that means it'll happen at the worst time.

**The platform team's responsibility specifically:**

A platform team prevents incidents *for product teams*:

- **Paved roads** — make the easy path the right path. Golden Helm charts, templated CI/CD, secure-by-default infrastructure.
- **Guardrails** — admission policies preventing dangerous configurations.
- **Self-service tooling** so teams don't depend on platform team for routine operations.
- **Standard observability** in base images — RED metrics, traces, logs automatic.
- **Platform-level SLOs** that aggregate across product teams' issues.

**Architect's framing:**
> "Repeated incidents indicate systemic gaps that individual operators can't fix. The pattern: postmortem → identify systemic cause → tracked action item → architectural or process change → measured outcome. The hard part isn't producing postmortems — it's closing the action items. Many teams produce great postmortems and then revisit the same issues 6 months later because the action items sat in backlog. The platform team's contribution is structural: paved roads, guardrails, and observability that make incidents harder to cause and faster to detect. The cultural piece is treating reliability as product work, not toil — that's what enables engineering time to be allocated to prevention rather than just response."

---

That covers all three rounds. Likely Amazon-style follow-ups to be ready for:

- **"Tell me about a time you owned a difficult production incident."** — they want LP examples: Ownership, Bias for Action, Dive Deep, Earn Trust.
- **"You disagreed with a team about the right architecture. What happened?"** — Disagree and Commit, Have Backbone.
- **"How do you mentor junior engineers on reliability?"** — Hire and Develop the Best.
- **"How would you decide between cost and reliability when budgets are tight?"** — Frugality, Customer Obsession, balancing trade-offs.
- **"Describe your highest-impact engineering improvement in the last year."** — Deliver Results, with measurable outcomes.

Amazon interviews lean heavily on STAR-format Leadership Principle stories. Have 6-8 strong incident/architecture stories prepared, each mappable to multiple LPs.