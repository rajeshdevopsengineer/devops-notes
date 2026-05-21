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

