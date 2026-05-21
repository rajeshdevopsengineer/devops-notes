# Senior DevOps Engineer — Barclays Interview Answers (Azure Focus)

Combining both question sets here. I'll answer as I would in the actual interview — concise where the question is concise, detailed where it warrants depth.

---

## 🚀 CI/CD & DevOps Workflows

**How do you implement multi-stage deployments in Azure DevOps Pipelines?**

Use a YAML pipeline with multiple `stages`, each containing jobs and deployment jobs that target environments. Each stage represents a logical promotion step (Build → Dev → QA → Staging → Prod):

```yaml
trigger:
  branches: { include: [main] }

variables:
  imageRepo: myacr.azurecr.io/myapp
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  jobs:
  - job: BuildAndPush
    pool: { vmImage: 'ubuntu-latest' }
    steps:
    - task: Docker@2
      inputs:
        containerRegistry: 'acr-connection'
        repository: 'myapp'
        command: 'buildAndPush'
        tags: |
          $(tag)
          latest

- stage: DeployDev
  dependsOn: Build
  jobs:
  - deployment: Deploy
    environment: 'dev'        # references an Environment object in ADO
    strategy:
      runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              command: 'upgrade'
              chartType: 'FilePath'
              chartPath: './charts/myapp'
              releaseName: 'myapp'
              valueFile: 'values-dev.yaml'
              overrideValues: 'image.tag=$(tag)'

- stage: DeployProd
  dependsOn: DeployDev
  condition: succeeded()
  jobs:
  - deployment: Deploy
    environment: 'prod'       # this Environment has approval checks attached
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo "Deploying to prod"
```

Each stage gets its own variable groups (linked from Key Vault) and service connections per environment. The `environment:` directive is what hooks approvals and gates in.

**How would you configure pipeline approvals and gates before production deployment?**

Approvals and gates are attached to **Environments** (not the pipeline itself), via *Pipelines → Environments → \[env\] → Approvals and checks*. Options:

- **Pre-deployment approvals** — named users/groups must approve. Configure timeout, notify-on-creation, and prevent the requester from approving their own deploy.
- **Branch control** — only artifacts built from `main` (or a tagged release branch) can deploy to prod.
- **Business hours** — restrict deploys to allowed windows (e.g., no Friday-after-3pm deploys).
- **Invoke REST API / Azure Function** — call an external system (ServiceNow change ticket check, Datadog SLO check) and pass only if it returns 200.
- **Query Azure Monitor** — fail the deploy if active P1 alerts exist for the target service.
- **Required template** — enforces that prod deploys must use the approved security/compliance template.
- **Exclusive lock** — only one deployment to prod at a time.

In Barclays-style regulated orgs, I'd combine: ServiceNow change ticket gate + two-person approval + Azure Monitor health check + branch policy restricting to `main`.

**How do you trigger automated testing after each build in a CI pipeline?**

In Azure DevOps YAML, tests run as build steps (unit/integration in the Build stage, post-deploy smoke/E2E in environment stages):

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Run unit tests'
  inputs:
    command: 'test'
    arguments: '--collect:"XPlat Code Coverage" --logger trx'

- task: PublishTestResults@2
  condition: succeededOrFailed()
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '**/*.trx'

- task: PublishCodeCoverageResults@2
  inputs:
    summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'
```

Layered test pyramid: unit + static analysis on every commit; integration tests in the Build stage; smoke tests post-deploy to dev; full E2E + performance baseline in staging; canary smoke in prod. `condition: succeededOrFailed()` on `PublishTestResults` ensures results are visible even when tests fail.

---

## 🚀 Docker & Image Management

**How do you manage multiple application versions in the same Docker repository?**

Tagging strategy is everything. I use a multi-tag approach:

- **Immutable build tag** — `myapp:1.4.2-a1b2c3d` (semver + short git SHA). This is what production manifests reference. Never overwritten.
- **Floating tags** — `myapp:1.4`, `myapp:1`, `myapp:latest` — point at the current build. Useful for dev/local but never for prod.
- **Environment tags** — `myapp:prod`, `myapp:staging` — convenience pointers.

Enforce **tag immutability** in ACR (`az acr config retention update`) so production tags can't be silently overwritten. Use **retention policies** to auto-purge old untagged manifests. For production, always pin to the SHA-suffixed tag in Helm values; "latest" in prod is how you get 3am pages.

**Advantage of Alpine images:**

- **Tiny size** — Alpine base is ~5MB vs ~70MB+ for Debian-slim, ~200MB+ for Ubuntu. Faster pulls, smaller registry storage, faster cluster autoscaling.
- **Smaller attack surface** — fewer packages = fewer CVEs.
- **Cleaner images for static binaries** (Go, Rust) where you don't need libc gymnastics.

Caveats: Alpine uses **musl libc**, not glibc. That bites Python (slow wheel installs because precompiled manylinux wheels don't work — you end up compiling from source), Java with native libs, and anything doing DNS edge cases. For Java I often prefer `eclipse-temurin:17-jre-alpine` (works well) or for Python `python:3.12-slim` (Debian-slim is a better trade-off than `python:3.12-alpine`). For paranoid prod, **distroless** is even better — no shell, no package manager, just your binary + runtime.

**How can you securely inject environment variables into a running container?**

Bad: `ENV DB_PASSWORD=...` in Dockerfile, or `-e DB_PASSWORD=secret` in plain text.

Good:
- **Kubernetes Secrets** mounted as env vars: `valueFrom.secretKeyRef`. Encrypt etcd at rest (`--encryption-provider-config`).
- **External secret stores** — Azure Key Vault via the **Secrets Store CSI Driver** with Workload Identity. App reads from a mounted volume or env populated from the vault. No secrets in Git, no secrets in the cluster's etcd.
- **Docker secrets** (Swarm mode) or **`--env-file`** for local dev with the file `.gitignore`d.
- **Sealed Secrets / SOPS / External Secrets Operator** if you must store sealed encrypted values in Git for GitOps.

For Azure specifically: **AKS + Workload Identity + CSI Secrets Store driver → Key Vault**. The pod gets a federated token, exchanges it for a Key Vault token, and pulls secrets at startup. No long-lived credentials anywhere.

**Docker Content Trust (DCT):**

DCT uses **Notary** under the hood to sign image tags with publisher keys. When `DOCKER_CONTENT_TRUST=1`, the Docker client refuses to pull unsigned images and verifies signatures on pull. Why it matters: it prevents tag-substitution attacks (someone pushing a malicious image under the same tag) and provides cryptographic publisher verification.

In practice, the modern ecosystem has largely moved to **Cosign / Sigstore** (keyless signing with OIDC + transparency log via Rekor) because DCT/Notary v1 was operationally heavy. Either way, the principle is the same: sign images in the CI pipeline, verify at admission time in Kubernetes (Kyverno, Gatekeeper, or Connaisseur policies). For a regulated environment like Barclays, image signing + admission verification is essentially mandatory now.

---

## 🚀 Kubernetes & Platform Engineering

**Pod Disruption Budgets (PDBs):**

PDBs limit how many pods of a workload can be down *voluntarily* (node drains, cluster upgrades, autoscaler scale-down). They don't protect against involuntary disruptions (node crashes).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2          # or use maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

Rules of thumb: for 3 replicas, `minAvailable: 2` (tolerates 1 down). For larger fleets, `maxUnavailable: 25%`. Always pair PDBs with `topologySpreadConstraints` (spread across zones/nodes) and anti-affinity, otherwise a single node holding all your replicas defeats the PDB. Watch out for setting `minAvailable` equal to `replicas` — that blocks all voluntary drains and breaks cluster upgrades.

**Ingress Controllers on AKS:**

An Ingress Controller is the actual proxy (NGINX, Traefik, HAProxy, Azure Application Gateway) that watches `Ingress` resources and configures itself to route HTTP(S) traffic to Services. The `Ingress` resource is just declarative config — you need a controller to act on it.

On AKS two common patterns:

1. **NGINX Ingress Controller** (via Helm or the AKS app routing add-on) — install once, fronted by an Azure LoadBalancer (Standard SKU). Most flexibility, huge community.
2. **Application Gateway Ingress Controller (AGIC)** or the newer **Application Gateway for Containers** — uses Azure App Gateway as the data plane. Better integration with WAF, native Azure AD auth, but pricier and Azure-locked.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set controller.replicaCount=3
```

Then per-app:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [myapp.company.com]
    secretName: myapp-tls
  rules:
  - host: myapp.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: myapp, port: { number: 80 } }
```

Pair with **cert-manager** for automatic TLS via Let's Encrypt or internal CA, and **external-dns** to auto-create Azure DNS records.

**HPA with custom metrics:**

HPA scales replicas based on observed metrics. Three metric API tiers:
- `metrics.k8s.io` — built-in CPU/memory (metrics-server)
- `custom.metrics.k8s.io` — Prometheus-exposed app metrics (RPS, queue depth, p95 latency)
- `external.metrics.k8s.io` — metrics from outside the cluster (Azure Service Bus queue length, Azure Monitor)

For custom metrics you need an **adapter**: `prometheus-adapter` for Prometheus, **KEDA** for everything else (and KEDA is the de facto standard on AKS now).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Pods
    pods:
      metric: { name: http_requests_per_second }
      target: { type: AverageValue, averageValue: "100" }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{ type: Percent, value: 50, periodSeconds: 60 }]
    scaleUp:
      stabilizationWindowSeconds: 0
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
```

For Azure-native, KEDA's `azure-servicebus` scaler is fantastic — scale workers based on queue depth, scale to zero when idle.

---

## 🚀 Git & GitOps

**Git Tag and triggering production deployments:**

A tag is an immutable pointer to a commit, typically `v1.4.2`. Two flavors: lightweight (just a ref) and **annotated** (`git tag -a v1.4.2 -m "Release 1.4.2"`) — annotated tags carry metadata and a SHA, and are what you should use for releases. Signed tags (`-s`) add GPG verification.

Production trigger pattern (Azure DevOps):

```yaml
trigger:
  tags:
    include: ['v*']
```

Or for GitHub Actions:

```yaml
on:
  push:
    tags: ['v*.*.*']
```

The pattern I prefer: CI runs on every push to `main`, but **only tagged commits trigger the production stage**. Combined with branch protection (no direct pushes to main, signed commits required), this gives you a clean release record: every prod deploy maps to an annotated tag in Git.

**Automatic reconciliation and drift detection in ArgoCD:**

ArgoCD continuously compares Git desired state vs cluster live state. Configure it via the Application spec:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-config
    targetRevision: HEAD
    path: apps/myapp/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true          # delete resources removed from Git
      selfHeal: true       # revert manual cluster changes
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff: { duration: 5s, factor: 2, maxDuration: 3m }
```

`selfHeal: true` is the drift killer — any `kubectl edit` against the cluster gets reverted within the sync interval (default 3m, tunable via `timeout.reconciliation`). For early detection: ArgoCD Notifications + Prometheus metrics (`argocd_app_info{sync_status="OutOfSync"}`) → Alertmanager → Slack/PagerDuty. For prod, many teams keep `selfHeal: false` and require explicit sync (with approval) but still alert on drift.

**Resolving merge conflicts in GitOps multi-environment deployments:**

The trick is to *prevent* them by design more than resolving them. Patterns:

1. **Kustomize overlays** — one `base/` directory + per-env `overlays/dev/`, `overlays/prod/`. Each env owns its own `kustomization.yaml` and patches; promotions touch only that env's file. No cross-env file conflicts.
2. **Separate values files** — `values-dev.yaml`, `values-prod.yaml`. Each env's PR only modifies its own file.
3. **Promotion via PR, not parallel commits** — image tag promotion goes dev → staging → prod as sequential PRs. No two pipelines write to the same file simultaneously.
4. **Auto-merge with rebase** for image-tag-bump PRs from CI (they're trivially mergeable).

When conflicts *do* happen (usually in shared base manifests):
```bash
git fetch origin
git checkout my-branch
git rebase origin/main
# resolve, validate with: kustomize build overlays/prod | kubectl diff -f -
git rebase --continue
git push --force-with-lease
```

`--force-with-lease` instead of `--force` so you don't clobber someone else's push. Always `kubectl diff` or `helm template` the result before pushing — a YAML conflict that "looks resolved" can still produce broken manifests.

---

## 🚀 Cloud & Infrastructure (Azure Focus)

**HA architecture across Azure regions:**

Layers:

1. **Compute** — AKS clusters or VMSS in two (or more) regions, each spanning 3 Availability Zones. Active-active where the app supports it; active-passive (with warm standby) where stateful constraints make active-active painful.
2. **Global traffic management** — **Azure Front Door** (anycast, global L7, WAF, fast failover) for HTTP. **Traffic Manager** (DNS-based) for protocols Front Door doesn't speak. Front Door does health probes per backend and fails over typically in seconds.
3. **Data** — region-pair replication. Cosmos DB multi-region writes (single-digit ms reads globally). Azure SQL with active geo-replication or auto-failover groups. Storage with RA-GZRS. For Postgres/MySQL Flexible Server, cross-region read replicas + failover.
4. **Secrets/config** — Key Vault with geo-redundancy; replicate critical secrets cross-region.
5. **DNS** — Azure DNS with short TTLs on records that may fail over.
6. **Backups** — cross-region restore tested quarterly. RTO/RPO measured, not aspirational.
7. **Observability** — Log Analytics workspaces per region, aggregated into a central workspace; cross-region dashboards in Grafana/Azure Monitor.

The trap people fall into: declaring "multi-region HA" but never actually failing over. Run regular game days that force a region offline.

**Azure Blueprints for enterprise governance:**

Blueprints package **ARM templates + policies + RBAC role assignments + resource groups** as a single artifact you assign to a subscription. Use case: when a new business unit needs a subscription, instead of clicking through dozens of policies, role assignments, and baseline resources, you assign the "Production Subscription Blueprint" — it lays down the landing zone (log analytics workspace, network watcher, locked policies for tagging/encryption/region restrictions, baseline RBAC) consistently.

**Important caveat:** Microsoft has deprecated Azure Blueprints. New deployments should use **Azure Landing Zones** (the Cloud Adoption Framework reference architecture), implemented via **Terraform/Bicep + Azure Policy + Management Groups + Deployment Stacks**. The functional goal is the same — consistent, governed environments — but the tooling has shifted. If asked in an interview I'd mention the deprecation; it shows current awareness.

**Application Gateway vs Front Door:**

| | Application Gateway | Front Door |
|---|---|---|
| Scope | Regional | Global (anycast) |
| Layer | L7 (HTTP/HTTPS/WebSocket) | L7 (HTTP/HTTPS) |
| Use case | In-region load balancing, AKS ingress | Multi-region routing, edge acceleration |
| WAF | Yes (regional WAF) | Yes (global WAF) |
| CDN | No | Yes (built-in caching) |
| TLS termination | Yes | Yes |
| Backend | Anything reachable in the VNet | Public endpoints, App Gateway, App Service, etc. |

The clean pattern is **stack them**: Front Door at the edge for global routing, WAF, and DDoS; App Gateway inside each region for the actual ingress to AKS. Front Door for global failover, App Gateway for regional traffic management.

---

## 🚀 Monitoring, Alerting & Troubleshooting

**Centralized logging for multi-region AKS:**

Two solid patterns:

**Pattern A — Azure-native:**
- AKS clusters → **Azure Monitor / Container Insights** (Log Analytics workspace per region for data residency + cost containment)
- Cross-region querying with Log Analytics workspace federation (`workspace("prod-eu").ContainerLog | union workspace("prod-us").ContainerLog`)
- Long-term archive: export to Azure Storage / Data Explorer

**Pattern B — OSS stack:**
- **Fluent Bit** as DaemonSet on each cluster → ships to either:
  - Central **Loki** (cost-efficient, label-based, pairs beautifully with Grafana/Prometheus)
  - **Elasticsearch/OpenSearch** if you need full-text search at scale
- Per-region collectors with regional buffering (so a network blip doesn't lose logs), then forward to central via cross-region peering or private endpoints
- **Grafana** as the single pane, multiple data sources

Either way: structured JSON logs at the app level, consistent labels (`cluster`, `region`, `namespace`, `app`, `version`), retention tiers (hot 7d, warm 30d, cold 1y), and never ship secrets to logs (use redaction filters in Fluent Bit).

**Detecting resource misconfigurations with Azure Log Analytics:**

Combine **Azure Policy compliance data** + **Resource Graph** + **Log Analytics** queries via KQL. Examples:

```kql
// Storage accounts with public network access enabled
AzureActivity
| where OperationNameValue contains "MICROSOFT.STORAGE/STORAGEACCOUNTS/WRITE"
| where Properties contains "publicNetworkAccess: Enabled"

// NSG rules that allow * to 22/3389
AzureDiagnostics
| where Category == "NetworkSecurityGroupEvent"
| where msg_s contains "Allow" and (destinationPort_s == "22" or destinationPort_s == "3389")
| where sourceAddressPrefix_s == "*" or sourceAddressPrefix_s == "0.0.0.0/0"

// Resources missing required tags
resources
| where tags !has "CostCenter" or tags !has "Owner"
| project name, type, location, tags
```

Schedule these as **Log Search Alerts**, route to **Action Groups** (email, webhook, Logic App that posts to Slack/ServiceNow). Pair with **Microsoft Defender for Cloud** which automates much of this via Secure Score and regulatory compliance dashboards.

**Intermittent 504 Gateway Timeout — troubleshooting:**

504 means a proxy/gateway upstream couldn't get a timely response from the backend. Methodical triage:

1. **Where exactly is the 504 coming from?** Front Door, App Gateway, NGINX ingress, or the app's own proxy? Check the response headers — `Server: Microsoft-Azure-Application-Gateway/v2` vs `Server: nginx` tells you the layer.
2. **Is it specific paths or endpoints?** Single endpoint = app code (slow query, blocking call); broad = infra.
3. **Backend health probe pass/fail correlation** — check App Gateway/Front Door backend health metrics. Intermittent probe failures point at backend capacity.
4. **App metrics** — p95/p99 latency, GC pauses, thread pool exhaustion, DB connection pool starvation. Container Insights and APM (Application Insights) here.
5. **Downstream dependencies** — DB slow queries, external API latency. Check `dependencies` table in Application Insights.
6. **Resource saturation** — node CPU/memory throttling, network egress limits, SNAT port exhaustion (very common on AKS with high-fanout outbound calls — symptom is intermittent timeouts to external services).
7. **Timeouts mismatched across layers** — if Front Door's timeout is 30s but App Gateway is 60s, and your app sometimes takes 35s, only Front Door fails. Align timeouts top-down (edge ≥ middle ≥ backend) or fix the slow path.
8. **HPA lag** — traffic spike, autoscaler hasn't caught up yet. Check HPA events.
9. **Connection pooling on the keep-alive boundary** — idle connection reuse where the backend already closed the socket. Tune `keepalive_timeout` and idle connection settings.

In Azure specifically, **SNAT port exhaustion** on the cluster's outbound IP is the #1 cause of intermittent timeouts I see. Fix: NAT Gateway with more public IPs, or use Private Link for Azure service calls so they don't traverse SNAT.

---

## 🚀 Scripting & Infrastructure Automation

**Bash — monitor Azure VM CPU and alert at 80%:**

```bash
#!/usr/bin/env bash
set -euo pipefail

# Usage: ./vm-cpu-alert.sh <resource-group> <vm-name>
RG="${1:?resource group required}"
VM="${2:?vm name required}"
THRESHOLD=80
WEBHOOK="${SLACK_WEBHOOK:?SLACK_WEBHOOK env var not set}"

VM_ID=$(az vm show -g "$RG" -n "$VM" --query id -o tsv)

# Pull the last 5 minutes of CPU and take the max
END=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
START=$(date -u -d '5 minutes ago' +"%Y-%m-%dT%H:%M:%SZ")

CPU_MAX=$(az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Percentage CPU" \
  --start-time "$START" --end-time "$END" \
  --interval PT1M \
  --aggregation Maximum \
  --query "value[0].timeseries[0].data[].maximum" -o tsv \
  | sort -nr | head -1)

CPU_INT=${CPU_MAX%.*}

if (( CPU_INT >= THRESHOLD )); then
  PAYLOAD=$(cat <<EOF
{"text":":rotating_light: VM *${VM}* in *${RG}* CPU at *${CPU_MAX}%* (threshold ${THRESHOLD}%)"}
EOF
)
  curl -sS -X POST -H 'Content-Type: application/json' \
    -d "$PAYLOAD" "$WEBHOOK"
  echo "ALERT: CPU=${CPU_MAX}%"
  exit 1
fi

echo "OK: CPU=${CPU_MAX}%"
```

Schedule via cron, an Azure Function on timer trigger, or — in real life — just use a proper **Azure Monitor metric alert rule** with an action group. Scripts like this are useful for custom logic Azure Monitor can't express, not as a replacement for it.

**Python — auto-scale AKS nodes using Azure CLI:**

```python
#!/usr/bin/env python3
"""Scale an AKS node pool based on pending-pod pressure."""
import json
import subprocess
import sys
import time

RG = "prod-rg"
CLUSTER = "prod-aks"
NODEPOOL = "userpool"
MIN_NODES = 3
MAX_NODES = 20
PENDING_THRESHOLD = 5    # if >= this many pods are Pending, scale up

def run(cmd):
    r = subprocess.run(cmd, capture_output=True, text=True, check=True)
    return r.stdout

def pending_pod_count():
    out = run(["kubectl", "get", "pods", "-A",
               "--field-selector=status.phase=Pending", "-o", "json"])
    return len(json.loads(out)["items"])

def current_node_count():
    out = run(["az", "aks", "nodepool", "show",
               "-g", RG, "--cluster-name", CLUSTER, "-n", NODEPOOL,
               "--query", "count", "-o", "tsv"])
    return int(out.strip())

def scale_to(n):
    n = max(MIN_NODES, min(MAX_NODES, n))
    print(f"Scaling {NODEPOOL} -> {n}")
    run(["az", "aks", "nodepool", "scale",
         "-g", RG, "--cluster-name", CLUSTER, "-n", NODEPOOL,
         "--node-count", str(n), "--no-wait"])

def main():
    pending = pending_pod_count()
    current = current_node_count()
    print(f"Pending pods: {pending} | Current nodes: {current}")

    if pending >= PENDING_THRESHOLD and current < MAX_NODES:
        scale_to(current + 2)
    elif pending == 0 and current > MIN_NODES:
        # naive scale-down; real impl should check node utilization
        scale_to(current - 1)

if __name__ == "__main__":
    try:
        main()
    except subprocess.CalledProcessError as e:
        print(f"Command failed: {e.stderr}", file=sys.stderr)
        sys.exit(1)
```

Disclaimer worth saying in the interview: **for production, use the Cluster Autoscaler or KEDA with `cluster-autoscaler` rather than rolling your own.** Custom scripts are useful when you need scaling logic the autoscaler doesn't support (e.g., predictive scaling based on a forecast model).

**Terraform module — Key Vault with RBAC policies:**

```hcl
# modules/keyvault/variables.tf
variable "name"                       { type = string }
variable "location"                   { type = string }
variable "resource_group_name"        { type = string }
variable "tenant_id"                  { type = string }
variable "sku_name"                   { type = string  default = "standard" }
variable "purge_protection_enabled"   { type = bool    default = true }
variable "soft_delete_retention_days" { type = number  default = 90 }
variable "public_network_access_enabled" { type = bool default = false }
variable "allowed_subnet_ids"         { type = list(string) default = [] }
variable "rbac_assignments" {
  description = "Map of principal_id => role to assign on this vault"
  type = map(object({
    principal_id = string
    role         = string   # e.g. "Key Vault Secrets User"
  }))
  default = {}
}
variable "tags" { type = map(string)  default = {} }

# modules/keyvault/main.tf
resource "azurerm_key_vault" "this" {
  name                          = var.name
  location                      = var.location
  resource_group_name           = var.resource_group_name
  tenant_id                     = var.tenant_id
  sku_name                      = var.sku_name
  enable_rbac_authorization     = true                 # <-- key bit: RBAC, not access policies
  purge_protection_enabled      = var.purge_protection_enabled
  soft_delete_retention_days    = var.soft_delete_retention_days
  public_network_access_enabled = var.public_network_access_enabled

  network_acls {
    default_action             = "Deny"
    bypass                     = "AzureServices"
    virtual_network_subnet_ids = var.allowed_subnet_ids
  }

  tags = var.tags
}

resource "azurerm_role_assignment" "this" {
  for_each             = var.rbac_assignments
  scope                = azurerm_key_vault.this.id
  role_definition_name = each.value.role
  principal_id         = each.value.principal_id
}

# modules/keyvault/outputs.tf
output "id"        { value = azurerm_key_vault.this.id }
output "name"      { value = azurerm_key_vault.this.name }
output "vault_uri" { value = azurerm_key_vault.this.vault_uri }
```

Usage:

```hcl
module "kv_prod" {
  source              = "../../modules/keyvault"
  name                = "kv-myapp-prod"
  location            = "westeurope"
  resource_group_name = azurerm_resource_group.prod.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  allowed_subnet_ids  = [module.vpc.private_subnet_ids[0]]

  rbac_assignments = {
    app_pods = {
      principal_id = azurerm_user_assigned_identity.app.principal_id
      role         = "Key Vault Secrets User"
    }
    sre_team = {
      principal_id = data.azuread_group.sre.object_id
      role         = "Key Vault Secrets Officer"
    }
  }

  tags = { env = "prod", owner = "platform" }
}
```

Notes: `enable_rbac_authorization = true` disables legacy access policies and forces RBAC — Microsoft's recommended path. `purge_protection_enabled = true` is required by most compliance frameworks (and once on, can't be turned off — that's the point). Network ACLs default-deny with explicit subnet allow-list.

---

## 🔧 CI/CD with GitHub Actions & Jenkins

**Multi-environment CI/CD pipeline design:**

The mental model: **one build, many deploys**. Build the artifact once, promote the same artifact through environments. Never rebuild for prod.

GitHub Actions structure:

```yaml
# .github/workflows/ci-cd.yml
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "tag=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
      # ... build, test, scan, push image

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: dev               # GitHub Environment; no approval
    runs-on: ubuntu-latest
    steps:
      - run: helm upgrade myapp ./chart -f values-dev.yaml --set image.tag=${{ needs.build.outputs.image_tag }}

  deploy-staging:
    needs: deploy-dev
    environment: staging           # GitHub Environment; required reviewers
    runs-on: ubuntu-latest
    steps: [ ... ]

  deploy-prod:
    needs: deploy-staging
    environment: production        # required reviewers + branch restriction + wait timer
    runs-on: ubuntu-latest
    steps: [ ... ]
```

GitHub Environments provide approvals, secret scoping per env, and branch protection. Jenkins equivalent: separate stages with `input` steps and credentialed bindings per environment, ideally driven by a shared library.

**Securely managing secrets in GitHub Actions:**

Layered approach:
- **GitHub Encrypted Secrets** (repo or org level) for static low-value secrets
- **Environment secrets** — scoped to a specific GitHub Environment; only jobs targeting that environment can access them. Pair with required reviewers so prod secrets need approval to use.
- **OIDC federation to cloud providers** — best practice: no long-lived cloud credentials in GitHub at all. Configure a federated identity in Azure AD / AWS IAM that trusts GitHub's OIDC issuer for a specific repo + branch. The Action gets a short-lived token at runtime.
  ```yaml
  permissions:
    id-token: write
    contents: read
  steps:
  - uses: azure/login@v2
    with:
      client-id:       ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      # no client-secret! federated identity does the auth
  ```
- **External secret managers** — Azure Key Vault / HashiCorp Vault. The Action authenticates via OIDC, pulls secrets at job time.
- **Never** `echo $SECRET`, never write secrets to files committed to artifacts, mask custom secrets with `::add-mask::`.

**GitHub Actions vs Jenkins in enterprise:**

| | GitHub Actions | Jenkins |
|---|---|---|
| Hosting | SaaS (or self-hosted runners) | Self-hosted, you own ops |
| Config | YAML in repo | Groovy DSL / declarative; shared libraries |
| Ecosystem | Marketplace actions, GitHub-native | Massive plugin ecosystem (also: massive plugin maintenance burden) |
| Secrets | Native + OIDC federation | Credentials plugin, Vault integration |
| Enterprise control | GitHub Enterprise, audit logs, OIDC | Full control, isolated network |
| Cost model | Per-minute billing on GitHub-hosted | Infra cost + ops time |
| Cold start | Fast | Variable; cold agents take time |
| Best for | Cloud-native shops, GitHub-centric orgs | Complex existing pipelines, on-prem, regulated environments needing data residency |

In Barclays-type environments, Jenkins still dominates for on-prem/regulated workloads (data residency, network isolation, deep integrations); GitHub Actions is gaining ground rapidly for greenfield work.

**Handling rollback in pipelines:**

Multi-layered:
- **Artifact-based** — keep the previous image tag known. `helm rollback myapp 0` reverts to previous release. `kubectl rollout undo deployment/myapp` for raw manifests.
- **GitOps rollback** — revert the PR in the GitOps repo. ArgoCD reconciles to the previous state automatically.
- **Blue-green rollback** — flip the service selector back to blue. Near-instant.
- **Database migrations** — the hard part. Always **expand/contract** (additive migrations only): step 1 add new column, step 2 deploy code that writes to both, step 3 backfill, step 4 deploy code that reads only new, step 5 drop old. Rollback at any step is safe because old code still works.
- **Automated rollback triggers** — Argo Rollouts with analysis templates: if error rate exceeds threshold during canary, auto-rollback.

The cardinal rule: **make rollback boring**. If rollback is exciting, your deploy strategy is broken.

---

## ☁️ Kubernetes on Azure (AKS)

**Create and scale AKS deployments:**

```bash
az aks create -g rg-prod -n aks-prod \
  --node-count 3 --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler --min-count 3 --max-count 20 \
  --enable-managed-identity --enable-workload-identity --enable-oidc-issuer \
  --network-plugin azure --network-policy calico \
  --zones 1 2 3

az aks get-credentials -g rg-prod -n aks-prod

kubectl create deployment myapp --image=myacr.azurecr.io/myapp:1.0
kubectl scale deployment myapp --replicas=5
kubectl autoscale deployment myapp --min=3 --max=20 --cpu-percent=70
```

For production I'd never use raw `kubectl create` — everything goes through Helm + ArgoCD. The cluster itself is provisioned via Terraform.

**Liveness and readiness probes:**

- **Liveness** — "is this container alive?" Fail → kubelet kills and restarts the container. Use for deadlock detection. Should be cheap and reliable; a bad liveness probe is worse than none (restart loops).
- **Readiness** — "is this container ready to serve traffic?" Fail → pod is removed from Service endpoints (no traffic sent), but the pod is *not* killed. Use for warm-up, dependency-not-ready, temporary back-pressure.
- **Startup** (often forgotten) — disables liveness/readiness until it passes. Use for slow-starting apps (Spring Boot, JVMs) so liveness doesn't kill them during boot.

```yaml
startupProbe:
  httpGet: { path: /health/startup, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10        # gives 5 minutes to start
livenessProbe:
  httpGet: { path: /health/live, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2
```

Liveness should check *the process is healthy* (not dependencies). Readiness should check *dependencies are reachable* (DB, cache). Mixing these up is a classic outage cause — DB blips kill all pods because liveness fails everywhere.

**Blue-green and canary in Kubernetes:**

**Blue-green:** two Deployments (`myapp-blue`, `myapp-green`); single Service with a selector. Deploy green, validate, switch the service's label selector to green. Old version stays around for instant rollback. Tools: Argo Rollouts, Flagger, or hand-rolled with `kubectl patch service`.

**Canary** with Argo Rollouts:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: myapp }
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: { duration: 5m }
      - analysis:
          templates: [{ templateName: success-rate }]
      - setWeight: 25
      - pause: { duration: 10m }
      - setWeight: 50
      - pause: { duration: 10m }
      - setWeight: 100
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        nginx: { stableIngress: myapp-ingress }
```

The AnalysisTemplate queries Prometheus for error rate / latency and auto-aborts if SLOs are violated. **Flagger** is a similar tool; both are excellent.

**Monitoring & logging tools for K8s:**

Production stack I default to on AKS:
- **Metrics:** Prometheus (kube-prometheus-stack) + Grafana, or Azure Monitor managed Prometheus
- **Logs:** Loki + Promtail/Fluent Bit, or Container Insights → Log Analytics
- **Traces:** Tempo or Application Insights, instrumented via OpenTelemetry
- **Alerting:** Alertmanager → PagerDuty + Slack, or Azure Monitor alert rules + Action Groups
- **Cluster health:** kube-state-metrics + node-exporter (bundled in kube-prometheus-stack)
- **Cost:** Kubecost or OpenCost
- **Security/posture:** Defender for Containers, Falco, Trivy Operator

---

## 🧱 Terraform (IaC)

**init → plan → apply → destroy workflow:**

- `terraform init` — downloads providers and modules, initializes the backend (where state lives). Run after cloning, after any backend/provider/module change.
- `terraform plan` — computes the diff between desired (code) and actual (state + refresh from cloud). Outputs what will be added/changed/destroyed. **Never apply without reading plan output.**
- `terraform apply` — executes the plan. Use `-auto-approve` only in CI with a previously saved plan: `terraform plan -out=tfplan && terraform apply tfplan`.
- `terraform destroy` — tears it all down. Reserved for ephemeral environments; never run against shared infra without explicit guardrails.

I always pair these with `terraform validate`, `terraform fmt -check`, `tflint`, and `tfsec`/`checkov` in CI.

**`terraform state` vs `terraform output`:**

- **`terraform state`** — a *subcommand family* for inspecting and manipulating the state file: `state list`, `state show`, `state mv`, `state rm`, `state pull`, `state push`, `state replace-provider`. Operational tool for state surgery.
- **`terraform output`** — reads declared `output` values from state and prints them. Used by CI to pass values downstream (`-raw`, `-json`).

```bash
terraform state list                            # everything tracked
terraform state show aws_instance.web           # detailed attrs
terraform output -raw load_balancer_dns         # consumable in CI
```

**Managing infra for multiple environments:**

Three common patterns; pick one and be consistent:

1. **Workspaces** — same config, different state files. Cheap to set up, but easy to mix up (`terraform workspace select prod` mistakes happen). I avoid for prod.
2. **Per-environment directory** (my preference):
   ```
   environments/
     dev/    { main.tf, backend.tf, terraform.tfvars }
     stage/  { ... }
     prod/   { ... }
   modules/
     network/  app/  database/
   ```
   Each env directory has its own backend, its own state, calls shared modules with env-specific values.
3. **Terragrunt** — DRY wrapper over Terraform; great for large multi-env, multi-account setups. Adds a learning curve.

For Barclays-scale orgs with strict governance, pattern 2 or Terragrunt with separate state per environment per region.

**Remote backends in Azure:**

```hcl
terraform {
  required_version = ">= 1.6"
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstatecompanyprod"
    container_name       = "tfstate"
    key                  = "prod/network.tfstate"
    use_azuread_auth     = true
  }
}
```

Configuration on the storage account:
- **Versioning + soft delete** for state recovery
- **Private endpoint** so state isn't reachable over public internet
- **RBAC**: pipeline service principal gets `Storage Blob Data Contributor` scoped to the container
- **State locking** is built-in (uses blob lease)
- **Encryption at rest** is on by default; use a customer-managed key in Key Vault for sensitive infrastructure
- **Diagnostic logs** to Log Analytics for audit trail (who touched state when)

---

## 📦 Docker & Containerization

**Efficient and secure Dockerfiles:**

Already covered above. Quick checklist:
- Multi-stage build
- Pinned, minimal base image (distroless or alpine)
- Non-root user, read-only root filesystem where possible
- `.dockerignore` to exclude `.git`, `node_modules`, secrets, build artifacts
- Layer ordering: deps before code
- Single concern per image, single process per container (use init like `tini` if reaping zombies)
- `HEALTHCHECK` instruction
- No secrets in build args/layers — use BuildKit `--mount=type=secret`
- Scanned in CI (Trivy / Snyk / Defender)
- Signed (Cosign) and verified at admission

**Container vs VM:**

- **VM** — full OS instance, virtualized hardware via a hypervisor (KVM, Hyper-V). Strong isolation, slow boot (seconds-minutes), large footprint (GBs).
- **Container** — process(es) sharing the host kernel, isolated via namespaces (PID, net, mount, user, IPC, UTS) and constrained via cgroups. Fast boot (ms), tiny footprint (MBs), weaker isolation than VMs.

Containers and VMs are complementary, not competing — containers run *inside* VMs in virtually all production cloud deployments (AKS nodes are VMs running containers).

**Multi-container apps — Compose vs Kubernetes:**

- **Docker Compose** — single-host orchestration. Excellent for local dev, integration testing in CI, small single-machine deployments. Declarative YAML, simple, fast iteration. Not for production-scale.
- **Kubernetes** — multi-host cluster orchestration with scheduling, self-healing, autoscaling, service discovery, rolling updates, secrets management. The right tool when you need HA, scale, multi-tenant, multi-team.

Rule of thumb: Compose for dev, Kubernetes for everything multi-node and production-grade.

**Scanning Docker images for vulnerabilities:**

Multiple layers, defense in depth:
- **Build-time** — Trivy, Snyk, Grype scan in CI. Fail build on critical/high CVEs.
- **Registry-side** — ACR Defender, ECR enhanced scanning, JFrog Xray. Continuous scanning as new CVEs are disclosed against already-pushed images.
- **Runtime** — Falco, Defender for Containers, Sysdig. Detect runtime anomalies.
- **Admission control** — Kyverno or Gatekeeper policies that block deploys of unsigned or unscanned images.
- **Base image hygiene** — distroless or minimal images mean fewer CVEs by construction; periodically rebuild to pick up patched base images.

---

## 🔄 DevOps Best Practices & Collaboration

**Branching strategy in CI/CD:**

The branching model defines how code flows from commit to production:

- **Trunk-based development** — short-lived feature branches (hours-days), merge to `main` behind feature flags. Pairs naturally with CI/CD and continuous deployment. My preference for high-velocity teams.
- **GitFlow** — long-lived `develop`, `release/*`, `hotfix/*`, `feature/*`. Heavier, suited to versioned products with scheduled releases. Heavy for SaaS.
- **GitHub Flow** — `main` is always deployable; branch, PR, merge, deploy. Simple, opinionated.

Branching directly shapes pipelines: trunk-based encourages continuous deployment; GitFlow encourages release-pipeline gates. The most common antipattern: GitFlow with long-lived branches + the goal of "deploy daily." Pick a strategy that matches your release cadence.

**Shift-left testing:**

Move quality activities earlier in the SDLC instead of catching things in late-stage QA or prod. Practical implementations:
- Pre-commit hooks (linters, formatters, secret scanners — `gitleaks`, `talisman`)
- Static analysis in PR builds (SonarQube, CodeQL)
- Unit & contract tests run on every commit
- IaC scanning (`tfsec`, `checkov`) on Terraform PRs
- Security scanning (SAST, DAST, SCA) integrated into the pipeline
- Developer-runnable integration test environments (ephemeral preview environments per PR)
- Threat modeling in design phase, not after code is written
- Test data and observability concerns owned by dev, not "thrown over the wall"

Culture matters more than tools: developers see test failures and security findings as their problem, not QA's or Security's.

**GitOps vs traditional CI/CD:**

| | Traditional CI/CD | GitOps |
|---|---|---|
| Source of truth | Pipeline + cluster state | Git repo |
| Deploy mechanism | Pipeline pushes to cluster | Cluster pulls from Git (operator) |
| Credentials | Pipeline has prod cluster credentials | Only the in-cluster operator has cluster creds |
| Drift detection | Manual | Automatic, continuous |
| Rollback | Rerun pipeline | Revert Git commit |
| Audit | Pipeline logs | Git history |

GitOps (ArgoCD, Flux) inverts the deployment model: the cluster reconciles itself to match Git, instead of pipelines pushing changes in. Security wins (fewer external creds needed in the pipeline), audit wins (Git is the audit log), and drift detection comes for free. Traditional CI/CD is still excellent for the *CI* portion (build, test, scan, push image, commit GitOps manifest); GitOps handles the *CD* part better.

**Infrastructure immutability:**

Servers/images are never modified in place — to change them, you replace them. Benefits:
- **Reproducibility** — what runs in prod is identical to what was tested; no "config drift over time"
- **Rollback simplicity** — redeploy the previous image
- **Disaster recovery** — rebuilding is well-rehearsed because that's how every change works
- **Security** — patching = redeploy, not SSH and `apt upgrade`; reduces toil and ensures consistency
- **Audit** — every change is a versioned artifact

Practical implementation: containers (the image is immutable), AMIs/VHDs baked via Packer, Kubernetes pods (treat as cattle, not pets), declarative IaC. The cultural shift: nobody SSHes into prod boxes to "just fix one thing." All changes go through code review and pipelines.

---

## 🔍 Monitoring, Troubleshooting & Security

**Tools for log aggregation and alerting:**

Stacks I've used and when each fits:
- **Prometheus + Grafana + Alertmanager + Loki** — cloud-native default. Best for K8s, label-based queries, low operational cost for Loki.
- **ELK / OpenSearch + Filebeat/Logstash + Kibana** — heavy but full-text search at scale; mature ecosystem.
- **Azure Monitor + Log Analytics + Application Insights** — native Azure, deep integration, KQL is genuinely good. The default for Azure-heavy shops.
- **Datadog / New Relic / Dynatrace** — full-stack SaaS observability; expensive but excellent if you have the budget.
- **Splunk** — incumbent in big enterprises (including banks). Powerful, expensive, lots of skilled operators around.

Real-world setups usually mix: Prometheus for K8s metrics, Loki/ELK for logs, Azure Monitor for cloud resource telemetry, all aggregated into Grafana for dashboards and PagerDuty for paging.

**Incident response for failed deployments:**

Defined sequence I follow:
1. **Detect** — automated alert from health checks, error-rate jump, synthetic monitor failure. SLO burn-rate alerts catch issues before customers complain.
2. **Stop the bleeding** — rollback first, investigate after. `kubectl rollout undo` or revert the GitOps PR. Don't try to roll forward under fire.
3. **Declare** — open an incident channel (Slack), assign IC (Incident Commander), Comms lead, Subject Matter Expert roles. Keep customer status page updated.
4. **Investigate in parallel** — logs, traces, metrics, recent deploys diff (`argocd app diff`), Git history. Time-bound investigation; if stuck, escalate.
5. **Mitigate fully** — rollback may not be enough (DB migration, cache poisoning, data corruption). Address each surface.
6. **Restore** — confirm SLOs recovered, customer impact ended.
7. **Postmortem** (within 48-72h) — blameless. Timeline, root causes (5-whys, fishbone), contributing factors, action items with owners and due dates. Share widely.
8. **Action items tracked** — postmortems are worthless if action items don't get done. Treat them like P2 tickets with deadlines.

Prevention: pre-prod canary, gradual rollouts, feature flags, automated rollback on SLO violation, well-tested runbooks, regular game days.

**Cloud security best practices in Azure DevOps pipelines:**

- **Identity** — Managed Identities / Workload Identity, OIDC federation; never long-lived service principal secrets.
- **Least privilege** — pipeline service principals scoped to the smallest resource group needed. Separate identities per environment.
- **Secrets** — Key Vault references in variable groups, not plain-text variables. Never log secret values.
- **Approval gates** for prod deploys; segregation of duties (the person who wrote the code can't be the only approver).
- **Self-hosted agents** in a private VNet for pipelines that touch internal resources. Microsoft-hosted agents for public-internet work only.
- **IaC scanning** — `tfsec`, `checkov`, `terrascan` on every IaC PR.
- **Image scanning** — Defender for Containers, Trivy in pipeline.
- **Signed commits** required on `main`.
- **Audit** — Azure DevOps audit logs to Log Analytics; alerts on suspicious actions (service connection changes, permission changes).
- **Defender for DevOps** — surfaces misconfig and exposed secrets across DevOps platforms.
- **Pipeline-as-code only** — no UI clicks creating release pipelines; everything reviewable in Git.

**Monitoring cost and usage:**

- **Azure Cost Management + Billing** — built-in budgets, alerts, anomaly detection, recommendations. Tag everything with `CostCenter`, `Owner`, `Environment`, `Application` so reports are meaningful.
- **Azure Advisor** — surfaces unused resources, right-sizing opportunities.
- **Kubecost / OpenCost** for AKS workload-level cost attribution down to namespace/pod/label. Critical for chargeback/showback.
- **Reserved Instances and Savings Plans** for predictable workloads; **Spot instances** for fault-tolerant workloads (CI runners, batch).
- **Budget alerts** in Azure routed to Slack/Teams via Action Groups at 50/75/90/100% thresholds.
- **FinOps practices** — monthly cost reviews per team, unit economics ($/transaction, $/customer), policy guardrails (Azure Policy denying expensive SKUs without approval).
- **Lifecycle policies** — blob tiering to cool/archive, log retention reduced for non-prod, dev environments shut down nightly via automation.

---

That's the full set. Common Barclays-style follow-ups would dig into: workload identity / OIDC federation specifics, multi-region failover game day design, KQL query writing live, and walking through a real production incident you handled. Worth rehearsing a specific incident story (5-10 min, structured: situation, action, decision points, outcome, lessons learned) — that almost always comes up.