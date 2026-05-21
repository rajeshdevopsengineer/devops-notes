# Senior DevOps / Azure Cloud Interview Answers

A complete walkthrough of the questions, the way I'd answer them in an interview — concise on the warm-up, detailed where the depth matters.

---

## 🚪 Getting Started

### Can you introduce yourself and your DevOps/Azure experience?

This one needs to be your own story, but here's the structure I'd use as a Senior DevOps/SRE candidate:

> "I'm a Senior DevOps Engineer with ~X years building and operating cloud-native platforms, with the last ~Y years focused on Azure. I've led the design and delivery of CI/CD platforms serving multiple product teams — Jenkins and Azure DevOps for build/release, Terraform for infrastructure, AKS for container orchestration, ArgoCD for GitOps, and Prometheus/Grafana for observability. Most recently I owned the platform for a [domain — e.g., regulated financial workload, e-commerce, healthcare] running on AKS across two regions, handling [scale — e.g., ~50 microservices, ~2k pods, multi-region active-active]. I care about reliability through automation, treating infrastructure as a product for internal customers, and reducing toil through self-service tooling."

Three principles for this answer: keep it under 90 seconds, anchor in concrete scale, and end with what you *value* (it sets the tone for the rest of the interview).

### Azure services and DevOps tools I've actively worked with

Grouped by capability:

| Capability | Azure | DevOps Tooling |
|---|---|---|
| Compute & orchestration | AKS, VMSS, App Service, Functions, Container Apps | Helm, Kustomize, ArgoCD |
| IaC | ARM, Bicep | Terraform (primary), Pulumi |
| CI/CD | Azure DevOps Pipelines, GitHub Actions | Jenkins, GitLab CI |
| Containers & registry | ACR with content trust, image scanning | Docker, BuildKit, Trivy, Cosign |
| Identity | Entra ID, Managed Identity, Workload Identity, RBAC | OIDC federation to pipelines |
| Networking | VNet, NSG, Private Link, App Gateway, Front Door, AFD WAF | NGINX Ingress, cert-manager |
| Data | Azure SQL, Cosmos DB, PostgreSQL Flexible Server, Storage, Service Bus, Event Hubs | — |
| Secrets | Key Vault + CSI driver | External Secrets Operator, SOPS |
| Observability | Azure Monitor, App Insights, Log Analytics, Managed Prometheus | Prometheus, Grafana, Loki, Alertmanager, OpenTelemetry |
| Governance | Azure Policy, Management Groups, Defender for Cloud, Blueprints (now Landing Zones) | OPA/Gatekeeper, Kyverno |
| Code quality | — | SonarQube, CodeQL, Snyk, Checkov, tfsec |

The honest version of this answer in an interview: I name the 6–8 I've used deepest and add "and I've worked with these adjacent ones at a familiar level" for the rest. Padding the list invites a trap question on something you've barely touched.

### Technologies I've built CI/CD pipelines for

- **Java** (Maven + Gradle) — Spring Boot microservices, packaged as fat JARs, containerized, deployed to AKS via Helm.
- **.NET** (.NET 6/8) — `dotnet build/test/publish`, multi-stage Dockerfile, deployed to AKS or App Service.
- **Python** — FastAPI and Django services; `pip` or Poetry, pytest, Black/Ruff, Bandit for SAST, image build + push.
- **Angular** (and React) — `npm ci` → `ng build --prod` → static assets to Azure Storage + Front Door, or served from an NGINX-based image behind an Ingress.
- **Node.js** — Express/NestJS backends, similar to Python flow.
- **Infrastructure (Terraform)** — separate IaC pipelines with `fmt`, `validate`, `tflint`, `tfsec`/`checkov`, plan-on-PR, apply-on-merge.

Cross-language patterns: build once → promote the same artifact across environments, SAST + dependency scan + image scan in every pipeline, GitOps for deploys.

---

## 🔧 CI Pipeline & Builds

### Typical steps in a CI pipeline for a Java application

In order, with what each step actually does:

1. **Checkout** — clone the repo at the triggering commit. Shallow clone (`--depth=1`) where history isn't needed, speeds things up.
2. **Cache restore** — Maven `~/.m2/repository` or Gradle `~/.gradle/caches` from prior builds. Massive build-time saver.
3. **Compile** — `mvn compile` or `gradle compileJava`. Fail-fast on syntax/compile errors.
4. **Unit tests** — `mvn test` with JUnit. Publish test results in JUnit XML format so the CI UI shows pass/fail per test.
5. **Code coverage** — JaCoCo agent attached during tests; XML report generated at `target/site/jacoco/jacoco.xml`.
6. **Static analysis & quality gate** — SonarQube scan (`mvn sonar:sonar`), then `waitForQualityGate` blocks the pipeline until the gate passes/fails.
7. **Dependency vulnerability scan** — OWASP Dependency-Check, Snyk, or `mvn dependency-check:check`. Catches CVEs in transitive deps.
8. **SAST** — CodeQL, Checkmarx, or SonarQube's security rules. Surfaces source-code-level vulns.
9. **License compliance** — FOSSA or similar to ensure no GPL leaking into a proprietary product.
10. **Package** — `mvn package -DskipTests` produces the JAR/WAR.
11. **Build container image** — multi-stage Dockerfile, tag with `${git-short-sha}-${build-number}`.
12. **Image vulnerability scan** — Trivy, Grype, or Defender for Containers. Fail on CRITICAL/HIGH.
13. **Sign image** — Cosign signs with a key from Key Vault.
14. **Push to ACR** — image lands in the dev/snapshot repo first; promotion to release repo happens after additional gates.
15. **Update GitOps repo** — bump the image tag in `values-dev.yaml` and commit. ArgoCD takes it from there.
16. **Notify** — Slack/Teams on success/failure with links to logs and Sonar report.

A few principles worth stating in the interview: each stage fails fast and short-circuits the pipeline; the artifact is built **once** and the same image is promoted across environments; secrets come from Key Vault via the pipeline's managed identity / OIDC federation, never plain variables.

### When and how I scan code for vulnerabilities

Layered scanning at multiple lifecycle stages — defense in depth, each catches something the others miss.

**Pre-commit (developer machine):**
- `pre-commit` hooks: `gitleaks` for secret detection, language linters.
- IDE plugins (SonarLint, Snyk).

**Pull request build:**
- **SAST** (SonarQube/CodeQL) — source code analysis for security bugs (SQL injection, XSS, deserialization issues).
- **SCA / Dependency scanning** (Snyk, OWASP Dependency-Check) — known CVEs in libraries. This catches the Log4Shell-class issues.
- **Secret scanning** (gitleaks, TruffleHog, GitHub Advanced Security) — accidentally committed credentials.
- **IaC scanning** (Checkov, tfsec, Terrascan) — misconfig in Terraform/ARM/Bicep.

**Container build:**
- **Image vulnerability scan** (Trivy, Grype, Defender for Containers) — OS package CVEs in the base image and layers.
- **Dockerfile linting** (hadolint) — security and best-practice issues.

**Pre-deployment:**
- **Manifest/policy scan** (Kyverno or OPA Gatekeeper in dry-run, kubesec).
- **DAST** (OWASP ZAP, Burp) against a deployed staging instance.

**Continuous (post-deploy):**
- **Registry rescans** — ACR/Defender re-scan stored images as new CVEs are disclosed.
- **Runtime** (Falco, Defender for Containers) — behavior anomaly detection.

**Gating rules I apply:**
- Block PR merge on any CRITICAL CVE with a known fix.
- Block image push on CRITICAL/HIGH with fix available.
- Notify-only for medium/low; track in a dashboard.
- Quality gate: zero new vulnerabilities introduced in this change.

### Artifacts from Java code (JAR vs WAR) and how I choose

- **JAR (Java Archive)** — a packaged set of `.class` files + resources + manifest. Used for libraries and standalone applications.
- **Fat / Uber JAR** — JAR with all dependencies embedded, runnable directly: `java -jar app.jar`. Spring Boot, Quarkus, Micronaut produce these.
- **WAR (Web Archive)** — packaged web application meant to be deployed *into* a servlet container (Tomcat, JBoss, WebLogic). Contains `WEB-INF/`, `web.xml`, the compiled classes, and bundled libs.
- **EAR (Enterprise Archive)** — bundles multiple WARs/JARs for traditional JEE app servers. Rarely seen in modern microservice work.

**How I choose:**

| Situation | Artifact |
|---|---|
| Spring Boot / Micronaut / Quarkus microservice in a container | **Fat JAR** |
| Legacy app deployed to managed Tomcat/JBoss | **WAR** |
| Reusable library consumed by other apps | **Plain JAR** to artifact repo |
| Old-school JEE monolith | **EAR** |

In modern containerized work it's almost always a fat JAR — the container *is* the deployment unit; you don't want a servlet container as another layer in the stack. Spring Boot apps build a fat JAR by default with the spring-boot-maven-plugin.

### Handling build errors or failed unit tests

**Differentiate the failure first:**

1. **Compile error** — code is broken. Fail the build, notify the author, no retry. The fix is a code change.
2. **Failed unit test** — could be a real regression *or* a flaky test. Look at the failure:
   - Deterministic and reproducible → regression. Block the merge. Author fixes.
   - Intermittent with no clear cause → flaky test. Mark it flaky, file a bug, quarantine if pervasive, but **never just retry until green** — that's how teams ship bugs.
3. **Infrastructure failure** (network blip, agent ran out of disk, registry timeout) → safe to retry with backoff. Always alert when retries succeed so the root cause gets fixed.

**Pipeline hygiene that makes this work:**
- Publish test results in JUnit XML format so the CI UI shows which tests failed.
- Fail fast — early stages should be the fastest (lint, compile, unit) so feedback is quick.
- Test reports linked in the failure notification: Slack message includes a link to the Sonar report, the test report, and the failing build's console.
- "Fix the build" culture — a broken `main` build is a top priority; nobody merges over a red main.
- Branch protection: PR must pass CI before merge.
- Flaky test tracking — a dashboard of flaky tests with an SLA to fix them. Quarantine, don't ignore.

**Real example I'd cite:** Pipeline started failing 1 in 10 builds on a specific JUnit test exercising a date boundary. Turned out to be timezone-sensitive — the test ran in UTC but assumed local time. Fix: explicit `TimeZone.setDefault(...)` in test setup. The temptation was to mark it flaky and retry; the right move was to find the actual bug because the same logic was almost certainly broken in production for non-UTC users.

---

## 🐳 Docker & Containerization

### Where and how I store Docker images

**Storage:**
- **Azure Container Registry (ACR)** — primary for Azure-native workloads. Premium SKU for geo-replication, private endpoint, content trust, customer-managed keys.
- **JFrog Artifactory** — when the org already uses Artifactory for other artifact types (Maven, npm) and wants a unified store with promotion semantics.
- **AWS ECR** — if multi-cloud or AWS-primary.
- **Docker Hub** — only for public images of OSS projects we publish.

**Repo organization I use:**

Per-environment repos with promotion:
```
acrcompany.azurecr.io/dev/myapp:1.4.2-a1b2c3d
acrcompany.azurecr.io/staging/myapp:1.4.2-a1b2c3d
acrcompany.azurecr.io/prod/myapp:1.4.2-a1b2c3d
```
Promotion happens by re-tagging/copying the same image digest across repos (`az acr import` or `docker buildx imagetools create`) — not by rebuilding. This guarantees prod runs *exactly* what passed staging.

**Tagging strategy:**
- **Immutable build tag**: `1.4.2-a1b2c3d-2087` (semver + git short SHA + build number). Production references this.
- **Floating convenience tags**: `1.4`, `1`, `latest` — for local dev only. Never referenced from prod manifests.
- **Tag immutability enabled** on prod repos so tags can't be overwritten.

**Hardening I apply:**
- Private registry only — no anonymous pull, no public endpoints unless absolutely needed.
- **Workload Identity / Managed Identity** for pulls; no service principal secrets.
- ACR Tasks or scheduled scans (Defender for Containers) — re-scan images as new CVEs surface.
- **Retention policy** — purge untagged manifests after 30 days; keep tagged for 1 year (longer for prod releases).
- **Content trust / Cosign signing** — images signed in CI, verified by Kyverno at admission.
- **Geo-replication** for multi-region deploys (Premium SKU).
- **Diagnostic logs to Log Analytics** for audit.

### How the CD pipeline knows which image to deploy

The pattern depends on whether you're using GitOps or push-based CD.

**Push-based CD (e.g., Jenkins → kubectl/helm directly):**
The pipeline knows the image tag because *it just built it*. The build stage exports the tag as a pipeline variable; the deploy stage references it:
```groovy
sh "helm upgrade myapp ./chart --set image.tag=${env.IMAGE_TAG}"
```

**GitOps (my preferred pattern — ArgoCD/Flux):**
The cluster pulls from Git, so the source of truth is the values file in a Git repo. The CI pipeline:

1. Builds the image with tag `1.4.2-a1b2c3d`, pushes to ACR.
2. Clones the GitOps repo.
3. Updates `environments/dev/values.yaml` — `image.tag: 1.4.2-a1b2c3d`.
4. Commits, pushes, opens a PR (or auto-merges for dev).
5. ArgoCD sees the change, syncs the cluster.

Tools that automate step 3: **kustomize edit set image**, **yq**, **Helm chart Releaser**, or **image updaters** like ArgoCD Image Updater (which can watch the registry and auto-update Git).

```bash
yq -i '.image.tag = "1.4.2-a1b2c3d"' environments/dev/values.yaml
git add . && git commit -m "chore(dev): bump myapp to 1.4.2-a1b2c3d" && git push
```

**Why I prefer GitOps:** the cluster's current state is fully reconstructable from Git, every deploy is an auditable commit, drift is automatically detected, and rollback is `git revert`. The pipeline no longer needs cluster credentials, which is a big security win — only the in-cluster ArgoCD has cluster-admin scope, behind firewalls.

### Connecting Docker images to AKS — the steps

The flow end-to-end:

**1. Cluster ↔ Registry trust:**
- Create the AKS cluster with ACR integration: `az aks update -g rg -n aks --attach-acr myacr`. This grants the cluster's kubelet identity `AcrPull` on the registry — no `imagePullSecrets` needed in manifests.
- For Workload Identity-based pull (newer pattern), assign `AcrPull` to the pod's federated identity instead of the kubelet.

**2. Network path:**
- If ACR has Private Endpoint (recommended for prod), ensure AKS has DNS resolution and network reachability to the ACR private FQDN — typically via a Private DNS Zone linked to the AKS VNet.
- Otherwise, AKS pulls over the public endpoint with TLS.

**3. Image reference in the manifest:**
```yaml
spec:
  containers:
  - name: app
    image: myacr.azurecr.io/prod/myapp:1.4.2-a1b2c3d
```
Always full-path with registry FQDN; never just `myapp:tag`.

**4. Pull-time verification (production-grade):**
- **Image signature verification** at admission via Kyverno or OPA Gatekeeper — block unsigned images.
- **Policy on allowed registries** — only `myacr.azurecr.io/*` and trusted publishers allowed.

**5. Pod execution:**
- Kubelet on the node fetches the image, caches it locally, runs the container.
- Resource requests/limits applied via the spec.
- Probes (readiness/liveness/startup) determine when traffic is routed.

**6. Observability:**
- Container Insights / Prometheus scrape the pod.
- Logs flow to Log Analytics or Loki via Fluent Bit.

**Common gotchas I'd mention:**
- `ImagePullBackOff` is usually one of: wrong tag, missing `AcrPull` permission, DNS resolution failing for private endpoint, or rate-limiting (rare on ACR; common with Docker Hub).
- Forgetting to attach ACR to the cluster after creating both → pods can't pull → "is it the network or the auth?" debugging session. Always check `kubectl describe pod` first.

---

## 🚢 CD Pipeline & Kubernetes Deployment

### Deploying applications to AKS using Helm charts or manifest files

I use **Helm** as the default in production. Raw manifests are fine for simple cases but Helm gives you templating, packaging, versioning, and rollback.

**Helm chart structure:**
```
chart/
├── Chart.yaml          # name, version, appVersion
├── values.yaml         # default values
├── values-dev.yaml     # env-specific overrides
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── README.md
```

**Deploy commands:**
```bash
# Install or upgrade
helm upgrade --install myapp ./chart \
  --namespace prod --create-namespace \
  -f values-prod.yaml \
  --set image.tag=1.4.2-a1b2c3d \
  --wait --timeout 5m \
  --atomic

# Rollback
helm rollback myapp 0
helm history myapp
```

`--atomic` is critical: if the upgrade fails (probes don't pass, etc.), Helm rolls back automatically.

**Kustomize alternative** — overlay-based, no templating syntax. Cleaner for environment overrides, less powerful for complex logic. I use Helm when templating is needed, Kustomize when the variation is purely declarative overlays.

**In GitOps:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: myapp-prod, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/company/charts
    targetRevision: main
    path: charts/myapp
    helm:
      valueFiles: [../../environments/prod/values-prod.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

### How the pipeline updates the cluster after manifest changes

Two patterns again — push vs pull.

**Push (Jenkins → kubectl/helm):**
```groovy
stage('Deploy') {
  steps {
    withKubeConfig(credentialsId: 'aks-prod') {
      sh '''
        helm upgrade --install myapp ./chart \
          -n prod -f values-prod.yaml \
          --set image.tag=${IMAGE_TAG} \
          --atomic --wait
      '''
    }
  }
}
```
The pipeline holds cluster credentials and applies changes directly. Simple, works, but the pipeline has prod cluster access, which is a security surface.

**Pull (GitOps with ArgoCD):**
1. Pipeline commits the new image tag to the GitOps repo.
2. ArgoCD's reconciliation loop (every 3 minutes by default, or via webhook for immediate sync) detects the diff between Git and cluster.
3. ArgoCD applies the chart with the new values — same Helm engine, but running inside the cluster.
4. Pods roll out per the Deployment's `strategy` (rolling update).
5. ArgoCD reports `Synced` and `Healthy` once probes pass.

**Webhook for instant sync** (avoid the 3-minute lag):
```bash
# Bitbucket/GitHub webhook → ArgoCD's /api/webhook endpoint
```

**Failure handling:**
- ArgoCD has `retry` config with backoff.
- If `selfHeal: true`, ArgoCD reverts any manual kubectl changes.
- Argo Rollouts for canary/blue-green with automated SLO-based rollback.
- Health checks via Lua scripts for custom resource types.

**Notification:**
- ArgoCD Notifications controller → Slack/Teams on sync success/failure.
- Prometheus metrics (`argocd_app_info{sync_status="OutOfSync"}`) → Alertmanager for SLO-violation alerts.

### How the CD pipeline authenticates to AKS

**Preferred: Workload Identity Federation / OIDC** — no long-lived secrets.

For **GitHub Actions**:
```yaml
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: azure/login@v2
      with:
        client-id:       ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id:       ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        # no client-secret — federated identity does the auth
    - run: az aks get-credentials -g rg-prod -n aks-prod --overwrite-existing
    - run: kubectl apply -f manifests/
```

The setup:
1. Create an Entra ID application + service principal.
2. Add a federated credential trusting GitHub's OIDC issuer scoped to a specific repo + branch (`repo:company/myapp:ref:refs/heads/main`).
3. Grant the SP `Azure Kubernetes Service Cluster User` (and namespace-level RBAC inside the cluster for least privilege).
4. At runtime, GitHub mints an OIDC token, exchanges it for an Azure AD token via the federated credential, and the pipeline acts as that SP — for a single job, no persistent secret.

For **Azure DevOps**:
- **Service connection** to Azure subscription (workload identity federation supported natively now).
- Pipeline targets an **Environment** with approvals; the environment-scoped SP has prod cluster access only.

For **Jenkins** (typically self-hosted):
- Run Jenkins agents on an AKS-attached node (managed identity grants cluster access).
- Or use **Azure CLI** task with a credential bound to a Key Vault-stored SP secret rotated frequently.
- For pulls to private ACR from the build agent: agent identity has `AcrPull`.

**Inside the cluster (Kubernetes RBAC):**
The Azure-AD authenticated identity needs **RoleBinding** to operate. Don't bind to `cluster-admin` — bind to a scoped role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer
  namespace: prod
subjects:
- kind: User                # for OIDC users
  name: <object-id-of-SP>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: deployer            # custom role with deploy/services/pods get/list/create/update
  apiGroup: rbac.authorization.k8s.io
```

**For ArgoCD-based deploys** the pipeline doesn't need cluster access at all — only Git access. ArgoCD itself runs inside the target cluster and uses an in-cluster service account. This is the security win of GitOps.

---

## 🌐 Networking & Application Connectivity

### Which ingress controller did I use and why

Primary choice in most projects: **NGINX Ingress Controller**.

**Why:**
- Mature, battle-tested, huge community.
- Rich annotations for rewrite, rate limiting, auth, body size, timeouts.
- Pairs well with `cert-manager` for automatic Let's Encrypt or internal CA cert provisioning.
- Works with `external-dns` to auto-create Azure DNS records.
- Portable — same controller works on AKS, EKS, GKE, on-prem.

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.replicaCount=3 \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set controller.config.use-forwarded-headers=true \
  --set controller.metrics.enabled=true \
  --set controller.podDisruptionBudget.enabled=true
```

**Alternatives I've used and their fit:**

- **Application Gateway Ingress Controller (AGIC)** — uses Azure Application Gateway as the data plane. Native Azure WAF, Entra ID auth, integrates with Azure Front Door. Pros: managed, WAF rules at the edge. Cons: more expensive, Azure-locked, less flexible than NGINX.
- **Azure Application Gateway for Containers** (newer service) — Microsoft's evolution of AGIC; better performance characteristics and gRPC support.
- **Traefik** — great for label-based routing, dynamic config, dashboard UI. Used it for smaller deployments.
- **Istio Gateway** — if a service mesh is already in play, Istio's gateway eliminates the need for a separate ingress controller and adds mTLS, traffic shifting, etc.
- **HAProxy Ingress** — performance-focused; used in latency-sensitive cases.

**Decision criteria:**

| Need | Pick |
|---|---|
| Azure-native WAF, Entra ID auth, deep Azure integration | AGIC / App Gateway for Containers |
| Multi-cloud portability, flexible config, community plugins | NGINX |
| Service mesh in play | Istio Gateway / Linkerd |
| Edge global routing + WAF + CDN before regional ingress | Front Door + (NGINX inside AKS) |

Common pattern in prod: **Front Door (global) → App Gateway WAF (regional) → NGINX (in-cluster)**. Each layer does what it's best at.

### Protocols, ports, and whether a load balancer could replace it

**NGINX Ingress speaks:**
- HTTP/1.1, HTTP/2, gRPC, WebSockets — all over **80/443**.
- TCP/UDP traffic via ConfigMap-based stream config (less common).

**Ports:**
- Externally: 80 (HTTP, usually redirected to HTTPS) and 443 (TLS).
- Inside the cluster: the controller pod listens on whatever the Service points at; the controller-managed LoadBalancer Service binds 80/443 to the Azure Load Balancer's frontend IP.

**Could a plain Load Balancer replace the ingress controller?**

Conceptually yes, practically no for HTTP workloads. Trade-offs:

| | LoadBalancer Service | Ingress Controller |
|---|---|---|
| Layer | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| One LB per service | Yes — each Service of type `LoadBalancer` provisions a new Azure LB rule with a public IP | No — single ingress shares one IP across many services |
| Cost | Scales linearly with # services (public IPs add up) | One IP, many hosts/paths |
| Path-based routing | No | Yes (`/api`, `/web`) |
| Host-based routing | No | Yes (`api.example.com`, `web.example.com`) |
| TLS termination | Pass-through (TCP) | Yes, with cert-manager auto-renewal |
| Routing rules, rewrites, rate limits | No | Yes |
| Suitable for | Non-HTTP protocols, single-service exposure | HTTP/HTTPS microservice fleets |

For a small setup with a handful of services, LoadBalancer Services work. For a microservice platform with 50+ services on shared domains, an Ingress Controller is the only sensible answer — otherwise you're managing 50 public IPs and 50 TLS certs.

In production AKS, the typical architecture is: **one LoadBalancer Service (provisioned by the ingress controller's Helm install) → that's the single public entry point → the controller routes by host/path to backend Services (ClusterIP) → those Services route to pods.**

### Connecting front-end and back-end services in AKS

The standard pattern uses Kubernetes Services + DNS.

**Internal connectivity (frontend pod → backend pod):**
- Backend is exposed as a `ClusterIP` Service.
- Frontend pod resolves it via in-cluster DNS: `backend-service.namespace.svc.cluster.local` (or just `backend-service` if in the same namespace).
- Connection goes pod → kube-proxy iptables/IPVS rule → backend pod IP. No public network involved.

```yaml
# Backend Service
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: app
spec:
  type: ClusterIP
  selector: { app: backend }
  ports:
  - port: 80
    targetPort: 8080
```

Frontend config (env var or config map): `BACKEND_URL=http://backend.app.svc.cluster.local`. Often simplified to just `http://backend` for same-namespace.

**External connectivity (user → frontend):**
- Ingress routes `myapp.company.com` → `frontend` Service → frontend pods.
- Frontend pods call `backend` Service internally.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
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
        backend: { service: { name: frontend, port: { number: 80 } } }
      - path: /api
        pathType: Prefix
        backend: { service: { name: backend, port: { number: 80 } } }
```

**More advanced patterns I've used:**

- **Service mesh (Istio / Linkerd)** — adds mTLS between services, retries, circuit breakers, observability. Backend isn't called by URL but via mesh-aware sidecar proxies.
- **NetworkPolicies** — explicit allow-rules; deny-all by default, then `frontend → backend on port 8080` only. Critical for zero-trust posture.
- **External services** (managed DB, Service Bus) — accessed via Private Endpoints from AKS subnets; **External Services** in Kubernetes can map an external endpoint to an internal Service name for clean DNS.

**NetworkPolicy example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: app
spec:
  podSelector: { matchLabels: { app: backend } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: { matchLabels: { app: frontend } }
    ports:
    - protocol: TCP
      port: 8080
```

The principle to state: **DNS is the contract between services**, not IPs. Pods come and go; DNS names are stable.

---

## 📈 Scaling & Availability

### Autoscalers I've used and how they differ in Azure

Three layers of autoscaling in Kubernetes, each addressing different scaling concerns:

**1. Horizontal Pod Autoscaler (HPA)** — scales the *number of pod replicas*.
- Built into Kubernetes. Watches metrics, adjusts the Deployment's replica count.
- Default: CPU/memory from metrics-server.
- Custom metrics: `prometheus-adapter` → scale on app metrics (RPS, queue depth, p95 latency).
- External metrics: KEDA → scale on Azure Service Bus queue length, Event Hubs lag, Storage queue, Kafka lag, virtually anything.
- Scale-to-zero with KEDA for event-driven workloads.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: backend }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
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

**2. Vertical Pod Autoscaler (VPA)** — adjusts *requests/limits* of pods.
- Right-sizes resource requests based on observed usage.
- Modes: `Off` (recommendations only), `Initial` (set on pod create), `Auto` (evict and restart pods to apply).
- Cannot be used simultaneously with HPA on CPU/memory (they fight). Combine: HPA on custom metrics, VPA for resource sizing.
- Available on AKS via the VPA add-on or Goldilocks for recommendations.

**3. Cluster Autoscaler (CA)** — adjusts the *number of nodes*.
- Watches for unscheduled pods (pending due to resource pressure), adds nodes from the node pool.
- Scales down nodes when underutilized and pods can be safely rescheduled.
- On AKS: enabled per node pool: `--enable-cluster-autoscaler --min-count 3 --max-count 20`.

**4. KEDA (Kubernetes Event-Driven Autoscaling)** — the Azure-native go-to.
- Open-source, Microsoft-supported, available as an AKS add-on.
- Provides 60+ scalers including all major Azure services (Service Bus, Event Hubs, Storage Queue, Cosmos DB change feed, Azure Monitor metrics, Log Analytics queries).
- Supports scale-to-zero, which HPA can't do alone.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: worker-scaler }
spec:
  scaleTargetRef: { name: worker-deploy }
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      messageCount: "10"
    authenticationRef: { name: keda-azure-auth }
```

**5. Karpenter** — newer node autoscaler (originated on AWS, increasingly available on Azure). Provisions right-sized nodes on demand rather than scaling pre-defined node pools. Faster and more efficient than Cluster Autoscaler. **Karpenter for Azure** is GA.

**Differences in Azure specifically:**

| | AKS | Other |
|---|---|---|
| HPA | Standard K8s | Same |
| VPA | Add-on | Native or add-on |
| Cluster Autoscaler | Built-in, per node pool | Similar on EKS |
| KEDA | Managed add-on (AKS Workload Identity supported) | Self-installed |
| Spot node pools | First-class via VMSS Spot | EKS Spot |
| Karpenter | "Node Autoprovisioning" (AKS NAP) — Karpenter under the hood | EKS-native |

**My typical production setup:**
- **HPA** on every deployment with min/max replicas based on traffic profile.
- **KEDA** for event-driven workers (queues, topics, scheduled scalers for predictable patterns).
- **Cluster Autoscaler or AKS NAP** for node scaling.
- **PDBs + topology spread constraints** to keep autoscaler-driven changes safe (no draining the last available zone).
- **VPA in recommendation mode** to surface right-sizing opportunities; apply manually rather than auto-evict.

### How I handle high availability and disaster recovery

HA and DR are different things — say so explicitly in the interview. HA = surviving component failures without downtime; DR = recovering from large-scale disasters (region loss, data corruption) within defined RTO/RPO.

**High Availability — design principles:**

**Compute:**
- AKS node pools spanning **3 Availability Zones** (`--zones 1 2 3`). One zone failure = ~33% capacity loss, not an outage.
- Multiple replicas of every workload, never `replicas: 1` in prod.
- **PodDisruptionBudget** (`minAvailable: 2` or `maxUnavailable: 25%`) so voluntary disruptions (node upgrades) don't take services down.
- **topologySpreadConstraints** to enforce one pod per zone.
- **Pod anti-affinity** so two replicas don't land on the same node.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: backend } }
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector: { matchLabels: { app: backend } }
```

**Networking:**
- **Standard SKU Load Balancer** with zone-redundant frontend IPs.
- **Application Gateway / Front Door** for L7 with zone redundancy.
- **Private Endpoints** for managed services, also zone-redundant.

**Data:**
- **Azure SQL** with Auto-Failover Groups or Zone-Redundant deployment.
- **Cosmos DB** with multi-region writes (or single-write multi-read for cost).
- **PostgreSQL Flexible Server** with zone-redundant HA.
- **Storage** RA-GZRS (read-access geo-zone-redundant).
- **AKS etcd** is managed by Azure, replicated across zones automatically.

**Control plane:**
- AKS control plane has an SLA (99.95% with uptime SLA enabled).
- Multiple Ingress controller replicas with PDB.
- ArgoCD HA mode in production.

**Disaster Recovery — strategies by tier:**

| RTO/RPO | Pattern | Cost |
|---|---|---|
| Minutes/seconds | Active-active multi-region | $$$$ |
| < 1h / minutes | Pilot light (warm secondary) | $$$ |
| Hours / minutes-hours | Backup-restore to standby | $$ |
| Best-effort | Backup only | $ |

**Active-active pattern (what I've built for tier-1 systems):**

1. **Compute:** Identical AKS clusters in two regions (e.g., `westeurope` + `northeurope`).
2. **Traffic management:** **Azure Front Door** with both regional ingresses as backends; health probes route around an unhealthy region in seconds.
3. **Data — the hard part:**
   - **Cosmos DB** with multi-region writes (multi-master) — simplest for new apps.
   - **Azure SQL** with auto-failover groups — automatic failover with read-only secondary in normal operation.
   - **PostgreSQL/MySQL** — geo-replicas; promote on disaster.
   - **Storage** — RA-GZRS for read-anywhere with manual failover for writes.
   - **Key Vault** — built-in geo-replication; failover automatic for read.
4. **GitOps:** Same ArgoCD-managed apps in both clusters; manifests in Git apply identically.
5. **Stateful operations** — global locks via Redis with active-passive or consensus algorithms; idempotent processing for at-least-once semantics.
6. **Backups:** Velero backing up AKS persistent volumes to Azure Storage in a third region. Cosmos DB continuous backup with PITR. DB exports nightly to a separate subscription.

**DR runbook practices:**
- **Define RTO and RPO per system explicitly** — agreed with business stakeholders, not technical wishful thinking.
- **Game days** — quarterly, deliberately fail over a region. Measure actual RTO. Without this, "DR plan" is fiction.
- **Document failover procedures** as runbooks with screenshots and exact commands. Test that the on-call who's never done it before can follow it.
- **Avoid pet automations** that only the architect understands.
- **Practice failback** too — going back to the primary is often harder than failing over.
- **Monitor cross-region replication lag** with alerts before it becomes an RPO violation.

**A failure mode I always plan for:** the disaster isn't a clean region outage. It's *data corruption* that replicated to the standby before you noticed. Defense: **PITR + backup retention in a separate subscription with separate identity** — so a compromised admin account can't delete the backups. Bank-grade DR plans explicitly air-gap backup credentials.

---

That's the full set. A few likely follow-ups to be ready for:

- "Walk me through a real production incident you led" — have a structured story ready (situation, action, decisions, outcome, lessons).
- "What's your approach to cost optimization on AKS?" — VPA recommendations, spot node pools for non-critical workloads, KEDA scale-to-zero, Kubecost for attribution, right-sized requests.
- "How do you handle a noisy neighbor problem on a shared cluster?" — ResourceQuotas, LimitRanges, PriorityClasses, dedicated node pools with taints for critical workloads.
- "What's wrong with this Terraform / Helm chart / Kubernetes manifest?" — they may share a snippet and ask for review. Have a checklist mentally: security (non-root, secrets, RBAC), reliability (probes, PDB, resources), observability (labels, metrics annotations).


# DevOps Interview — Complete Answers

Going through each with the depth a senior engineer brings.

---

## 🔹 Reliability Engineering

### SLA, SLO, SLI — definitions and how they relate

These three are often used interchangeably and incorrectly. They form a hierarchy:

**SLI — Service Level Indicator**

A specific, measurable metric describing service behavior. The raw measurement.

Examples:
- Request success rate: `(successful_requests / total_requests) × 100`
- p99 latency for a specific endpoint
- Availability: `(uptime_seconds / total_seconds) × 100`
- Freshness of data: time between event and processing completion

**SLO — Service Level Objective**

An internal *target* for an SLI, agreed within the engineering org. The bar we hold ourselves to. Always tighter than the SLA.

Examples:
- 99.95% of requests return successfully over a 30-day window.
- p99 latency under 300ms over a 7-day window.
- 99.99% availability over rolling 30 days.

**SLA — Service Level Agreement**

A *contractual* commitment to customers, with consequences for breach (service credits, refunds, escalation). External-facing, legally binding.

Examples:
- "We guarantee 99.9% monthly uptime; below that, customers receive a 10% credit."
- "If p99 latency exceeds 500ms for more than 1% of requests in a billing month, customers get a 25% credit."

**The relationship:**

```
SLI (measured)  →  SLO (target)  →  SLA (contract)

  Reality           Internal goal     External promise

  99.97%      →    99.95%        →    99.9%
```

Always: **SLI is reality, SLO is stricter than SLA**. The gap between SLO and SLA is your safety margin — your **error budget**.

**Error budget = 100% − SLO**

If SLO is 99.95% over 30 days, the error budget is 0.05% = ~21.6 minutes of allowed downtime per month. Once you spend it, the policy is to slow feature velocity and focus on reliability.

**Architect's framing:**
> "SLA is what the lawyers care about; SLO is what engineering cares about; SLI is what monitoring measures. They have to align — measured SLIs feed into SLO compliance, and SLO must comfortably exceed SLA to give us margin. The single most important practice is **error budgets**: when we've burned through the budget, we stop shipping features and focus on reliability. That's the discipline that prevents 'SLO theater' — having an SLO on paper that nobody acts on."

### 99.9% SLA — what it practically means

99.9% ("three nines") sounds high but allows surprising amounts of downtime.

**Allowed downtime budget at 99.9%:**

| Period | Allowed downtime |
|---|---|
| Per day | 1 minute 26 seconds |
| Per week | 10 minutes 4 seconds |
| Per month (30 days) | **43 minutes 49 seconds** |
| Per quarter | 2 hours 11 minutes |
| Per year | 8 hours 45 minutes 57 seconds |

**For comparison:**

| Nines | Yearly downtime | Use case |
|---|---|---|
| 99% | 3 days 15 hours | Internal tools, dev |
| 99.9% | 8 hours 45 min | Most SaaS, B2B APIs |
| 99.95% | 4 hours 22 min | Enterprise SaaS |
| 99.99% | 52 minutes | Tier-1 services, payments |
| 99.999% | 5 minutes 15 sec | Telco, financial trading |

**What "downtime" actually means:**

It's not just "the site is completely down." Usually counted as:
- Error rate exceeds X% (often 1-5%) for a sustained period.
- p99 latency exceeds Y ms for a sustained period.
- Health checks fail from external monitors for N consecutive minutes.

The exact definition is in the SLA contract. The unambiguous mode (total outage) is rare; the ambiguous mode (degraded service) is what most "downtime" actually looks like.

**Practical implications of a 99.9% SLA:**

- A 45-minute incident in a month consumes your *entire* monthly budget. No room for any other issue.
- Two 30-minute incidents = SLA breach.
- A bad deploy that takes 1 hour to roll back = SLA breach.
- This forces operational discipline: fast rollback, canary deploys, observability that catches issues in minutes, not hours.

**Why teams aim for 99.95% SLO at 99.9% SLA:**

The gap is your *error budget* and your *safety margin*. If your SLO is 99.95% (~22 min/month) and SLA is 99.9% (~44 min/month), you have ~22 minutes of buffer. Burn through the SLO budget? Slow down. Approach the SLA? Emergency response.

**Architect's framing:**
> "99.9% sounds great in marketing but is genuinely tight in practice — 44 minutes of monthly downtime is one bad deploy or one slow rollback. The SLA itself doesn't make a service reliable; the engineering discipline around it does. I'd rather see a team commit to 99.5% they can actually meet consistently than 99.9% they breach quarterly."

### Monitoring SLA using monitoring tools

The pattern: define SLIs as queries, compute SLO compliance from them, alert on **error budget burn rate** rather than instantaneous thresholds.

**Step 1: Define SLIs as Prometheus queries**

```promql
# Availability SLI — success ratio over time
sum(rate(http_requests_total{job="api", code!~"5.."}[5m]))
/
sum(rate(http_requests_total{job="api"}[5m]))

# Latency SLI — % of requests below threshold
sum(rate(http_request_duration_seconds_bucket{job="api", le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count{job="api"}[5m]))
```

**Step 2: Compute SLO compliance over the rolling window**

```promql
# 30-day availability rate
1 - (
  sum(increase(http_requests_total{code=~"5.."}[30d]))
  /
  sum(increase(http_requests_total[30d]))
)
```

If this is below 0.9995 (for a 99.95% SLO), we've breached.

**Step 3: Error budget burn rate alerting**

The Google SRE workbook approach — instead of alerting on instantaneous threshold, alert based on how fast you're burning the budget.

```promql
# Burn rate = (current error rate) / (allowed error rate from SLO)
# If burn_rate > 14.4 for 1h → at this rate you'd burn 30 days of budget in 2 days. Page.
# If burn_rate > 6 for 6h → at this rate you'd burn 30 days in 5 days. Ticket.
```

Multi-window, multi-burn-rate alerts:

```yaml
groups:
- name: slo.api
  rules:
  - alert: APIErrorBudgetBurnFast
    expr: |
      (
        slo:api_error_rate:rate5m > (14.4 * 0.0005)   # 14.4× burn rate
        and
        slo:api_error_rate:rate1h > (14.4 * 0.0005)
      )
    for: 2m
    labels: { severity: page }
    annotations:
      summary: "API burning error budget 14× too fast — 2% spent in last hour"
      runbook: "https://wiki.company.com/runbooks/api-error-budget"

  - alert: APIErrorBudgetBurnSlow
    expr: |
      (
        slo:api_error_rate:rate1h > (6 * 0.0005)
        and
        slo:api_error_rate:rate6h > (6 * 0.0005)
      )
    for: 15m
    labels: { severity: ticket }
```

**Step 4: Tooling stack**

**Open-source / cloud-native:**
- **Prometheus** for SLI collection.
- **Grafana** for dashboards — SLO compliance bar, error budget consumption, burn rate over time.
- **Alertmanager** for routing burn-rate alerts to PagerDuty/Slack.
- **Sloth** or **OpenSLO** — tools that generate Prometheus rules and dashboards from SLO definitions in YAML.

```yaml
# Sloth example
version: "prometheus/v1"
service: "api"
labels: { team: "platform" }
slos:
- name: "availability"
  objective: 99.95
  description: "API request success rate"
  sli:
    events:
      error_query: sum(rate(http_requests_total{job="api", code=~"5.."}[{{.window}}]))
      total_query: sum(rate(http_requests_total{job="api"}[{{.window}}]))
  alerting:
    name: APIHighErrorRate
    page_alert: { labels: { severity: page } }
    ticket_alert: { labels: { severity: ticket } }
```

**AWS-native:**
- **CloudWatch** metrics for SLIs.
- **CloudWatch Service Lens** for distributed tracing context.
- **CloudWatch Synthetics Canaries** for external availability monitoring (more honest than internal scrapes — measures from user perspective).
- **CloudWatch Alarms** with `evaluation_periods` and `datapoints_to_alarm` for burn-rate-like alerts.

**Commercial:**
- **Datadog SLO** — purpose-built SLO product with burn-rate alerts and budget dashboards.
- **New Relic SLM** — similar.
- **Nobl9** — vendor-agnostic SLO platform.

**Dashboard I'd build:**
- **SLO compliance % over 30 days** — single big number, green/yellow/red.
- **Error budget remaining** — gauge: "76% of monthly budget remaining."
- **Burn rate over time** — line chart with multiple burn-rate thresholds marked.
- **Recent incidents that consumed budget** — table with timestamps and budget impact.
- **Per-service SLO compliance** — heatmap across all services.

**Architect's framing:**
> "Monitoring an SLA isn't about checking if a number is above 99.9% — it's about catching budget burn early enough to react. Burn-rate alerting based on multi-window evaluation is the SRE-standard approach: short-window alerts catch acute issues, long-window alerts catch slow degradation. SLO dashboards visible to engineering and product create the cultural feedback loop — when error budget is healthy, ship features; when it's burning, focus on reliability. Without that loop, SLOs are theater."

---

## 🔹 Cloud Architecture (AWS)

### Deploying a 3-tier application in AWS

Walking through end-to-end:

**Networking foundation:**

```
VPC: 10.0.0.0/16, 3 AZs
├── Public subnets (10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24)
│     ALB, NAT Gateways, bastion
├── Private app subnets (10.0.16.0/20, 10.0.32.0/20, 10.0.48.0/20)
│     EKS nodes, EC2 app servers
└── Private data subnets (10.0.64.0/22, 10.0.68.0/22, 10.0.72.0/22)
      RDS, ElastiCache
```

- Internet Gateway attached to VPC.
- NAT Gateways per AZ (no SPOF) for private subnet egress.
- Route tables per tier.
- VPC Endpoints for S3, DynamoDB (gateway), ECR/Secrets Manager/KMS (interface).
- VPC Flow Logs to CloudWatch/S3.

**Tier 1 — Presentation:**

- **CloudFront** for global edge — caches static assets, TLS termination, WAF integration, DDoS protection.
- **S3** for static frontend bundles (React/Angular/Vue), behind CloudFront with OAC.
- **Route 53** for DNS with health-check-based failover or latency-based routing.
- **ACM** for certificates.
- **AWS WAF** + Shield for DDoS protection.

**Tier 2 — Application:**

- **ALB** in public subnets fronting the application tier. WAF attached, TLS terminated at ALB.
- **EKS** or **ECS Fargate** for containerized microservices, *or* **EC2 Auto Scaling Group** for traditional apps.
  - For EKS: AWS Load Balancer Controller provisions ALBs from Ingress resources.
  - HPA + Cluster Autoscaler / Karpenter for elastic scaling.
- **API Gateway** for REST/HTTP/WebSocket APIs.
- **Lambda** for event-driven workloads.
- **SQS / SNS / EventBridge** for async messaging.
- **ElastiCache Redis** for cache and sessions.

**Tier 3 — Data:**

- **RDS Aurora PostgreSQL** Multi-AZ for transactional data — writer + readers, auto-failover.
- **DynamoDB** for high-write low-latency NoSQL.
- **S3** for object storage with lifecycle policies.
- **OpenSearch** for search/analytics (if needed).
- All in **private data subnets**, no internet route.

**Cross-cutting:**

- **IAM**: roles for EC2 instances, IRSA for EKS pods. No static credentials.
- **Secrets Manager** for credentials, External Secrets Operator on EKS to sync.
- **KMS** customer-managed CMKs for all encryption.
- **Observability**: CloudWatch + Prometheus/Grafana/Loki on EKS.
- **CI/CD**: GitHub Actions → ECR → ArgoCD → EKS.
- **DR**: warm-standby in second region with Aurora Global Database, S3 CRR, Route 53 failover.

**Security:**

- **Security groups** at every tier — ALB allows 80/443 from internet; EKS nodes allow 8080 from ALB SG only; RDS allows 5432 from EKS SG only. Zero-trust between tiers.
- **WAF** on ALB and CloudFront.
- **GuardDuty** for threat detection.
- **CloudTrail** to a separate logging account.
- **AWS Inspector** for vulnerability scanning.

**Architecture diagram (in words):**

```
User → Route 53 → CloudFront (WAF) →
       ├── S3 (static assets)
       └── ALB (public subnets, WAF) →
                 EKS / EC2 ASG (private app subnets) →
                       ├── Aurora (private data subnets, Multi-AZ)
                       ├── ElastiCache Redis (private data subnets)
                       ├── DynamoDB
                       └── S3 (via gateway endpoint)
```

### Ensuring high availability

HA is achieved by **eliminating single points of failure at every layer** and designing for the next-most-likely failure.

**Compute layer:**
- **Multi-AZ deployment** — at least 3 AZs for production.
- **Auto Scaling Group** with `min_size ≥ 2` and instances spread across AZs.
- For EKS: node groups spanning AZs, **topologySpreadConstraints** distributing pods, **PodDisruptionBudgets** enforcing minimum availability.

**Network layer:**
- **Multiple NAT Gateways** (one per AZ) so a single-AZ failure doesn't kill outbound.
- **Application Load Balancer** is regional and inherently multi-AZ.
- **Health checks** on every component — ALB target health checks, Route 53 health checks, ELB health checks for ASGs.

**Data layer:**
- **RDS Multi-AZ** — synchronous replication to standby in another AZ, auto-failover (~60-120s).
- **Aurora** — even better; 6-way storage replication across 3 AZs, near-zero failover.
- **DynamoDB** — multi-AZ by default; Global Tables for multi-region.
- **ElastiCache** — cluster mode with replication across AZs.
- **S3** — 99.999999999% (11 nines) durability by default; CRR for cross-region.

**Application design:**
- **Stateless application servers** — any instance can serve any request.
- **Session state externalized** to Redis or DynamoDB, not in-memory.
- **Graceful shutdown** — preStop hooks, connection draining, terminationGracePeriodSeconds.
- **Retries with exponential backoff** for downstream calls.
- **Circuit breakers** to prevent cascade failures.
- **Bulkheads** — separate thread pools / connection pools per downstream.

**Multi-region for higher tiers:**
- **Active-active or active-passive** across regions.
- **Route 53 health-check-based failover** or latency-based routing.
- **Aurora Global Database** for cross-region DB replication.
- **CloudFront** as global edge.

**Observability supports HA:**
- **Synthetic monitoring** (CloudWatch Synthetics, Pingdom) — measures from user perspective.
- **Multi-region monitoring** — if monitoring is in the same region as the workload, both go down together.
- **Alerts on SLO burn**, not just hard down.

**Operational practices:**
- **Game days** — quarterly DR drills that actually fail over.
- **Chaos engineering** — Chaos Mesh, Gremlin to validate failure handling.
- **Blameless postmortems** to capture lessons.
- **Capacity planning** — never run at >70% baseline so spikes have headroom.
- **Tested backups** — restore drills, not just backup completion.

**Architect's framing:**
> "HA is layered, and the failure mode you didn't plan for is the one that bites you. The first 9s come from multi-AZ; the second 9s come from multi-region; subsequent 9s come from operational discipline — fast detection, fast rollback, well-tested runbooks. The most-missed element is **data tier HA** — people build redundant app servers but run a single-AZ RDS. The other most-missed is **observability fault tolerance** — your monitoring has to survive what you're monitoring."

### Microservice communication across AZs

The good news: in AWS, **cross-AZ traffic is transparent at the application layer**. You don't write different code for cross-AZ communication; the network handles it.

**The mechanisms:**

**1. Service discovery via DNS**

In EKS, a Service named `payments-service` is reachable at `payments-service.namespace.svc.cluster.local`. The Kubernetes DNS resolves to a ClusterIP, which kube-proxy routes to backend pods *regardless of which AZ they're in*.

```python
# Application code — no AZ awareness
response = requests.get("http://payments-service/api/v1/transactions")
```

**2. Load balancer routing**

When microservices communicate via an internal ALB or NLB:
- The LB has targets in all AZs.
- Cross-zone load balancing distributes traffic across AZs evenly.
- The LB selects a target; the traffic goes to whichever AZ has it.

**3. Service mesh (Istio, App Mesh)**

If using a mesh:
- Sidecar proxies (Envoy) handle service-to-service routing.
- Identity-based routing via SPIFFE — caller authenticates regardless of network position.
- mTLS encrypts traffic regardless of which AZ it crosses.
- Locality-aware routing — mesh prefers same-AZ targets to reduce cross-AZ data charges.

**Cross-AZ networking properties:**

- **Latency**: <2ms typically within a region across AZs. Imperceptible for most workloads.
- **Bandwidth**: 10+ Gbps between AZs.
- **Reliability**: AWS guarantees high inter-AZ availability.
- **Cost**: **cross-AZ traffic incurs data transfer charges** ($0.01/GB each direction in most regions). For chatty microservices, this adds up.

**Cost optimization patterns:**

```yaml
# Topology-aware routing — prefer same-AZ targets
apiVersion: v1
kind: Service
metadata:
  name: payments-service
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  selector: { app: payments }
  ports: [{ port: 80, targetPort: 8080 }]
```

This tells Kubernetes to prefer routing to pods in the same AZ as the caller, falling back to cross-AZ only if needed.

**For EKS specifically:**
- **AWS VPC CNI** gives pods VPC IPs — same VPC fabric handles cross-AZ traffic.
- **Cilium** with topology-aware load balancing.

**Security across AZs:**

- **Security groups** apply regardless of AZ — same rules enforced.
- **NetworkPolicies** apply cluster-wide.
- **mTLS** in service mesh secures traffic regardless of AZ.

**Architect's framing:**
> "From the application's perspective, cross-AZ communication is invisible — Service DNS, load balancers, and service meshes abstract it. The architecturally interesting parts are: pay attention to cost (cross-AZ data charges are real at scale), latency-sensitive workloads should use topology-aware routing to prefer same-AZ, and security policies must work across AZs because traffic *will* cross AZs whenever any layer rebalances. The principle: **don't pin services to specific AZs; design for any-AZ-to-any-AZ traffic with sensible defaults**."

### Accessing an app in a private subnet from the internet

The whole point of a private subnet is that resources aren't directly reachable from the internet. So we put something *in front* that *is* reachable.

**Pattern 1: Application Load Balancer (most common for web apps)**

```
Internet → ALB (public subnets) → App instances (private subnets)
```

- ALB sits in **public subnets** (has IGW route).
- ALB has a public DNS name (or alias via Route 53).
- ALB forwards requests to targets in private subnets.
- Targets only need to allow traffic from the ALB's security group, not from the internet.

```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false                    # public-facing
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id  # in public subnets
  security_groups    = [aws_security_group.alb.id]
}

resource "aws_lb_target_group" "app" {
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  health_check { path = "/health" }
}

resource "aws_security_group" "app" {
  # App instances in private subnets
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # only from ALB
  }
}
```

**Pattern 2: NLB for non-HTTP / TCP workloads**

Same shape — NLB in public subnets, targets in private subnets. Use when you need TCP/UDP, static IP, or extremely high throughput.

**Pattern 3: API Gateway → VPC Link → private resources**

For serverless or per-route management:
```
Internet → API Gateway → VPC Link → NLB or PrivateLink → private targets
```

API Gateway provides authentication, rate limiting, request transformation, etc.

**Pattern 4: CloudFront → ALB**

For global edge + caching + WAF:
```
User → CloudFront (edge) → ALB (regional, public) → App (private)
```

CloudFront also enables Origin Access Identity-style restrictions so ALB can only be reached via CloudFront.

**Pattern 5: Bastion / Session Manager for admin access**

For SSH or admin access to instances (not user traffic):
- **AWS Systems Manager Session Manager** — preferred. No public IPs, no bastion host, no SSH keys. IAM-based auth, audited via CloudTrail.
- **Bastion host** in public subnet (legacy approach) — instance with public IP, SSH/RDP from admin IPs only, jump to private instances.

**Pattern 6: Client VPN**

For users needing direct private network access:
- **AWS Client VPN** — managed VPN endpoint, users connect with OpenVPN client.
- **Site-to-site VPN / Direct Connect** — for corporate networks.

**Pattern 7: ECS/EKS LoadBalancer Service**

For Kubernetes, the Service of type LoadBalancer (or Ingress) is the public entry:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector: { app: myapp }
  ports: [{ port: 80, targetPort: 8080 }]
```

This provisions an NLB in public subnets pointing at pods in private subnets.

**The common pattern in all of these:**

The thing in the private subnet is **never directly internet-reachable**. There's always a public-subnet intermediary (LB, Gateway, VPN endpoint) that the internet reaches, and that intermediary forwards to the private resource via internal-only paths.

**Architect's framing:**
> "The rule: private subnet workloads are reached *through* a public-subnet intermediary, never directly. The choice of intermediary depends on the protocol and pattern — ALB for HTTP, NLB for TCP, API Gateway for managed APIs, Session Manager for admin access. The instance in the private subnet should never have a public IP and should only allow traffic from the intermediary's security group. This separation is what makes 'private' meaningful — internet-facing surface is concentrated in well-defined choke points where WAF, rate limiting, and access control live."

### Route 53 failover routing

Route 53 failover routing is a DNS-based active-passive failover mechanism: a primary record gets traffic; if the primary's health check fails, traffic shifts to a secondary record.

**The components:**

**1. Health Check**

Route 53 monitors an endpoint (HTTP, HTTPS, or TCP) from multiple AWS regions. Defines:
- Endpoint to check.
- Protocol and port.
- Interval (10s or 30s).
- Failure threshold (consecutive checks).
- Calculated health checks (combine multiple checks with AND/OR logic).

```hcl
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
  regions           = ["us-east-1", "us-west-2", "eu-west-1"]
}
```

**2. Primary and Secondary records**

Two records for the same DNS name, distinguished by `set_identifier` and failover role:

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
  health_check_id = aws_route53_health_check.secondary.id
  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

**3. How the resolution works:**

When a client queries `app.example.com`:

1. Route 53 checks the primary record's associated health check.
2. **If primary is healthy**: returns the primary record's value.
3. **If primary is unhealthy**: returns the secondary record's value.
4. The client connects to whichever endpoint was returned.

The client doesn't know about the failover — DNS resolution returns one IP, and the client connects to it.

**Failover timing:**

- **Detection**: health check fails over `failure_threshold × interval`. With threshold=3, interval=30s, detection takes 90s.
- **DNS propagation**: depends on TTL. With TTL=60s, half of clients see the new IP within 60s, most within 2-3× TTL.
- **Total failover time**: roughly 2-5 minutes typically.

**Important caveats:**

**1. DNS caching at clients**

Some clients (browsers, older Java JVMs) cache DNS aggressively, ignoring TTL. They may stick to the failed IP even after Route 53 returns the new one.

Mitigations:
- Low TTL on the records (60s typical).
- Connection-level retry with re-resolution.
- For Java: set `networkaddress.cache.ttl=60` in `java.security`.

**2. Asymmetric failover**

Failover is one-way detection — Route 53 doesn't fail back automatically. Once primary recovers, traffic returns to primary on the next DNS resolution.

You can configure the behavior:
- **Primary returns**: traffic shifts back to primary automatically.
- Want to manually control? Disable the health check temporarily.

**3. Evaluate Target Health**

`evaluate_target_health = true` on alias records adds another layer: even without a Route 53 health check, the underlying target (ALB, NLB) signals its own health.

**Other Route 53 routing policies (for context):**

- **Simple**: basic single-record routing.
- **Weighted**: split traffic by percentage.
- **Latency-based**: route to lowest-latency region.
- **Geolocation**: route based on user geography.
- **Geoproximity**: based on distance, with bias.
- **Multivalue answer**: return up to 8 healthy records.
- **IP-based**: route based on source IP ranges.

**Combining policies:**

```
Latency-based (route to closest region)
    → Failover (within each region, primary/secondary)
        → Weighted (canary deployment within each tier)
```

**Architect's framing:**
> "Route 53 failover is DNS-level failover — coarse-grained but simple and globally distributed. The actual failover time is dominated by DNS propagation (TTL) and client-side caching, not Route 53's detection speed. For RTOs measured in minutes, it works well; for RTOs measured in seconds, you need application-level failover (LB-based, like AWS Global Accelerator's anycast). The classic mistake is setting a long TTL (3600s) and expecting fast failover — Route 53 returns the new value quickly, but clients keep using the cached old value. Low TTL + Route 53 health checks + evaluate_target_health is the standard combination."

---

## 🔹 AWS Networking

### VPC Endpoint

**VPC Endpoints** enable private connectivity between your VPC and AWS services without traversing the public internet, the NAT Gateway, or VPN.

**Two types:**

**1. Gateway Endpoints** (free)

For **S3** and **DynamoDB** only. Adds an entry to route tables that routes traffic to those services through the AWS internal network.

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id
}
```

Properties:
- **Free** — no hourly or data charges.
- **No ENI required** — just route table updates.
- **No DNS changes** — traffic uses the normal S3/DynamoDB DNS names.
- **Only S3 and DynamoDB** — no other services.

**2. Interface Endpoints** (PrivateLink — hourly + data charges)

For most other AWS services and for cross-account/private SaaS services. Creates an ENI in your VPC with a private IP.

```hcl
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.endpoints.id]
  private_dns_enabled = true   # so SDK calls resolve to the endpoint
}
```

Properties:
- **Hourly charges** (~$0.01/hour per AZ per endpoint).
- **Data processing charges** (~$0.01/GB).
- **Creates ENIs in your subnets** — uses private IPs from your subnet ranges.
- **Private DNS** (when enabled) makes SDK calls automatically use the endpoint.
- **PrivateLink** — same technology used for private SaaS access.

**Services that commonly use Interface Endpoints:**

- **ECR** (api + dkr) — for private container pulls.
- **Secrets Manager / SSM Parameter Store** — for secrets retrieval.
- **KMS** — for encryption operations.
- **CloudWatch Logs / Monitoring** — for log/metric submission.
- **STS** — for token operations.
- **SNS / SQS** — for messaging.
- **API Gateway**, **EventBridge**, **Step Functions**.

**Why use VPC Endpoints:**

**1. Security**
- Traffic stays on the AWS network, never traverses the public internet.
- Endpoint policies restrict which buckets/services can be accessed through the endpoint.
- For S3: combine with bucket policies that require `aws:sourceVpce` to enforce private-only access.

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::sensitive-bucket/*",
  "Condition": {
    "StringNotEquals": { "aws:sourceVpce": "vpce-abc123" }
  }
}
```

**2. Cost reduction**
- **No NAT Gateway data charges** for traffic to those services. Cross-AZ NAT charges add up fast for chatty AWS API usage.
- **Gateway endpoints (S3, DynamoDB) are free** and save the most.

**3. Performance**
- Lower latency than going via NAT Gateway.
- Higher throughput.

**4. Compliance**
- Some regulatory frameworks require that workload traffic never traverse the public internet.

**Common setup for a private EKS workload:**

```hcl
# Gateway endpoints (free)
resource "aws_vpc_endpoint" "s3" { ... }
resource "aws_vpc_endpoint" "dynamodb" { ... }

# Interface endpoints (paid but worth it)
locals {
  interface_endpoints = [
    "ecr.api", "ecr.dkr",
    "secretsmanager",
    "ssm", "ssmmessages", "ec2messages",
    "logs", "monitoring",
    "kms",
    "sts",
  ]
}

resource "aws_vpc_endpoint" "interface" {
  for_each            = toset(local.interface_endpoints)
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.${each.value}"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.endpoints.id]
  private_dns_enabled = true
}
```

**Architect's framing:**
> "VPC Endpoints are the difference between 'private subnet that secretly hairpins AWS API traffic through NAT to the public internet' and 'truly private network with AWS API traffic on the internal fabric.' Gateway endpoints for S3 and DynamoDB are free and obvious — always enable. Interface endpoints cost money but pay for themselves quickly through NAT data savings on chatty workloads (ECR pulls, Secrets Manager calls). The compliance benefit is separate: when an auditor asks 'does any production traffic traverse the public internet,' VPC Endpoints let you say 'no' honestly."

### NACL, NAT Gateway, ALB

**NACL — Network Access Control List**

Subnet-level stateless firewall.

| Property | Value |
|---|---|
| Level | Subnet boundary |
| State | Stateless |
| Rules | Allow AND deny |
| Default behavior | Default NACL allows all; custom NACLs deny all |
| Rule evaluation | Numbered, first match wins |
| Applies to | All resources in associated subnets |

**Stateless** is the critical property — if you allow inbound on 443, you must *also* explicitly allow outbound on ephemeral ports (1024-65535) for the response.

Used for:
- Subnet-level allow/deny by IP CIDR.
- Defense in depth (alongside security groups).
- Blocking specific malicious IP ranges that SGs can't (SGs are allow-only).
- Compliance requiring subnet-level controls.

For most workloads, default NACLs (allow all) work fine and security groups do the heavy lifting.

**NAT Gateway**

Managed AWS service enabling **outbound** internet access for resources in private subnets while keeping them **unreachable from the internet**.

```
Private Instance → Private subnet route table → NAT Gateway (in public subnet) → IGW → Internet
```

Properties:
- **AZ-scoped** — one NAT Gateway per AZ for HA (failure in AZ-A's NAT affects only AZ-A).
- **Managed** — AWS handles scaling, HA, patching.
- **Outbound only** — cannot initiate inbound connections.
- **Costs**: hourly charge (~$0.045/hour per NAT) + per-GB data charge (~$0.045/GB). At scale, this is significant — the data charge often dominates.

```hcl
resource "aws_nat_gateway" "main" {
  count         = length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

resource "aws_route_table" "private" {
  count  = length(var.azs)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
}
```

Used for: EC2 in private subnets needing to download packages, call third-party APIs, or hit public AWS endpoints when VPC Endpoints aren't configured.

**Modern best practice — combine NAT with VPC Endpoints**:
- NAT Gateway for general internet egress.
- VPC Endpoints for AWS APIs (S3, ECR, etc.) — bypasses NAT data charges entirely.

**ALB — Application Load Balancer**

Layer 7 (HTTP/HTTPS) load balancer with content-based routing.

Properties:
- **L7 routing** — path-based, host-based, header-based, query-string-based.
- **TLS termination** with ACM certificates.
- **WAF integration** for OWASP rules, bot protection.
- **Authentication integration** with Cognito, OIDC providers.
- **Per LCU pricing** (Load Balancer Capacity Units — connections + new connections per sec + data + rules).
- **Health checks** at L7 (HTTP/HTTPS path-based).
- **Sticky sessions** via cookie.
- **Cross-zone load balancing** enabled by default (free for ALB).

```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  security_groups    = [aws_security_group.alb.id]

  enable_deletion_protection = true
  drop_invalid_header_fields = true
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.app.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
  condition {
    path_pattern { values = ["/api/*"] }
  }
}
```

When to use ALB vs NLB:
- **ALB**: HTTP/HTTPS, gRPC, WebSocket. Need path/host routing, WAF, OIDC auth.
- **NLB**: TCP/UDP, ultra-high throughput, need static IPs, source IP preservation, ultra-low latency.

**How they fit together in a typical architecture:**

```
Internet
    ↓
ALB (public subnet) — terminates TLS, routes by path
    ↓
EKS pods / EC2 instances (private subnet) — accept traffic from ALB SG only
    ↓
NAT Gateway (public subnet) — when pods need to call external APIs
    ↓
Internet (or VPC Endpoint for AWS services)

NACL guards subnet boundaries (defense in depth)
```

---

## 🔹 Kubernetes

### How Kubernetes was deployed in your banking project

> "On the banking platform I worked on, Kubernetes was deployed as managed **EKS** in a regulated AWS landing zone:
>
> **Cluster topology:**
> - Production EKS clusters in two regions: us-east-1 (active) and us-west-2 (warm standby for DR).
> - Three node pools per cluster:
>   - **System pool** — t3.medium, 2-4 nodes, tainted `dedicated=system:NoSchedule`. Runs cluster-critical workloads (CoreDNS, ingress controller, monitoring).
>   - **General workload pool** — m6i.large, 3-30 nodes via Karpenter. On-demand for most services.
>   - **Spot workload pool** — m6i.large variants, for fault-tolerant batch and non-critical services.
> - All node groups spanned three AZs with topologySpreadConstraints enforcing distribution.
>
> **Provisioning:**
> - **Terraform** for everything — VPC, EKS, IAM, KMS, node groups, supporting AWS services.
> - Per-environment state files in S3 with DynamoDB locking.
> - Cluster encryption at rest with customer-managed KMS keys for secrets.
> - IRSA (IAM Roles for Service Accounts) — every pod that needed AWS access had a service account bound to a specific IAM role. No static credentials anywhere.
>
> **Cluster add-ons:**
> - AWS Load Balancer Controller (for ALB/NLB provisioning from Ingress).
> - Karpenter for node autoscaling.
> - ArgoCD for GitOps deployment.
> - cert-manager for TLS.
> - External Secrets Operator syncing from AWS Secrets Manager.
> - kube-prometheus-stack for metrics, Loki for logs, Tempo for traces.
> - Kyverno for admission policy enforcement.
> - Cilium as the CNI (we needed L7 NetworkPolicies for zero-trust).
>
> **Security posture (banking-specific):**
> - **PCI-DSS compliance** required: dedicated node pools for cardholder data workloads with taints.
> - **NetworkPolicies** default-deny per namespace, explicit allows.
> - **Pod Security Standards** restricted profile enforced via Kyverno — no privileged pods, no hostNetwork, no hostPath.
> - **Image signing with Cosign** — Kyverno blocked unsigned images at admission.
> - **CIS Kubernetes benchmark** compliance verified quarterly.
> - **CloudTrail + EKS audit logs** to a separate logging account.
> - **GuardDuty for EKS** for threat detection.
>
> **CI/CD pattern:**
> - GitHub Actions for build/test/scan → push image to ECR.
> - Pipeline commits image tag bump to a GitOps repo.
> - ArgoCD watching the GitOps repo, reconciles via Helm.
> - Argo Rollouts canary for tier-1 services (payments, account services) with SLO-based auto-abort.
> - Prod promotions required two-person PR approval and a Change Advisory Board ticket for regulatory traceability.
>
> **Observability:**
> - SLO-based monitoring with burn-rate alerts.
> - PagerDuty integration with severity-tiered routing.
> - Audit dashboards for compliance teams showing who deployed what when.
>
> **DR architecture:**
> - DR cluster runs at 25% of prod capacity — warm standby.
> - Aurora Global Database for transactional data.
> - Same GitOps repo, ApplicationSet fans deployments out to both clusters.
> - Quarterly DR drills with measured RTO (target: 15 minutes).
>
> The thing that mattered most for banking: **everything auditable**. Every deploy traced back to a Git commit, every cluster change to a Terraform apply, every secret access to CloudTrail. Compliance teams could reconstruct the state of the system at any historical timestamp."

### Rolling update manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  namespace: payments
  labels:
    app: payments-api
    tier: backend
spec:
  replicas: 6
  revisionHistoryLimit: 10                   # keep 10 ReplicaSets for rollback

  selector:
    matchLabels:
      app: payments-api

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2                            # up to 2 extra pods during rollout
      maxUnavailable: 0                      # never go below 6 ready pods

  template:
    metadata:
      labels:
        app: payments-api
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"

    spec:
      serviceAccountName: payments-api-sa
      terminationGracePeriodSeconds: 30

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels: { app: payments-api }

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels: { app: payments-api }

      containers:
      - name: api
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payments-api:1.4.2-a1b2c3d
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production
        envFrom:
        - configMapRef: { name: payments-config }
        - secretRef:    { name: payments-secrets }

        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi

        startupProbe:
          httpGet: { path: /actuator/health/startup, port: http }
          failureThreshold: 30
          periodSeconds: 10                  # up to 5 min for slow JVM start

        readinessProbe:
          httpGet: { path: /actuator/health/readiness, port: http }
          periodSeconds: 5
          failureThreshold: 2

        livenessProbe:
          httpGet: { path: /actuator/health/liveness, port: http }
          periodSeconds: 10
          failureThreshold: 3

        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]   # let LB stop sending traffic

        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]

        volumeMounts:
        - name: tmp
          mountPath: /tmp

      volumes:
      - name: tmp
        emptyDir: {}

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments-api
  namespace: payments
spec:
  minAvailable: 5                            # at least 5 of 6 always available
  selector:
    matchLabels: { app: payments-api }
```

**Key rolling-update properties:**

- **`maxSurge: 2`** — during update, up to 2 extra pods can be created beyond the desired count.
- **`maxUnavailable: 0`** — never go below the desired count. Zero-downtime guarantee.
- **`readinessProbe`** — new pods only get traffic after probe passes.
- **`preStop` sleep** — gives the load balancer time to remove the pod from rotation before SIGTERM.
- **`terminationGracePeriodSeconds: 30`** — enough time for in-flight requests to drain.
- **PodDisruptionBudget** — ensures voluntary disruptions (node upgrades) don't take us below 5 pods.

**Triggering and watching the rollout:**

```bash
# Update the image — triggers rolling update
kubectl set image deployment/payments-api api=...:1.4.3

# Watch progress
kubectl rollout status deployment/payments-api

# Pause if something looks wrong
kubectl rollout pause deployment/payments-api

# Resume
kubectl rollout resume deployment/payments-api

# Rollback
kubectl rollout undo deployment/payments-api

# History
kubectl rollout history deployment/payments-api
```

### CrashLoopBackOff or ImagePullBackOff

**CrashLoopBackOff** — pod's container keeps starting and crashing; Kubernetes is backing off before retrying.

The flow:
1. Container starts.
2. Container exits (success or failure).
3. kubelet waits (backoff: 10s, 20s, 40s, 80s, 160s, up to 5 min).
4. Restarts.
5. Loops.

**Diagnosis:**

```bash
# Step 1: Look at recent events and pod status
kubectl describe pod <pod>
# Look at: Last State, Reason, Exit Code, Events section

# Step 2: Logs from the CURRENT container
kubectl logs <pod>

# Step 3: Logs from the PREVIOUS crashed container (the critical one)
kubectl logs <pod> --previous

# Step 4: If multiple containers
kubectl logs <pod> -c <container-name> --previous

# Step 5: Exec into the container if it lives long enough
kubectl exec -it <pod> -- sh
```

**Common root causes ranked by likelihood:**

1. **Application error on startup** — bad config, missing env var, can't connect to DB.
2. **Misconfigured probes** — liveness probe failing immediately because app needs more startup time. Fix: add `startupProbe` or increase `initialDelaySeconds`.
3. **OOM kill** — exit code 137. App exceeds memory limit.
4. **Missing config or secrets** — ConfigMap/Secret referenced but doesn't exist.
5. **Volume mount issues** — PVC unbound, file permissions wrong (especially with non-root users).
6. **Image binary issues** — wrong architecture (ARM image on x86 node), missing dynamic libraries.
7. **PID 1 problem** — wrapper script exits because main process detaches.

**Diagnosis by exit code:**

- `0` — clean exit but kubelet expects long-running (probably PID 1 ending too soon).
- `1` — generic application error.
- `137` — SIGKILL, usually OOM.
- `139` — segfault.
- `143` — SIGTERM, graceful exit.
- `255` — abnormal termination.

**Fix and verify:**

After fix, restart triggers a fresh attempt:
```bash
kubectl delete pod <pod>   # Deployment recreates with new config
```

---

**ImagePullBackOff** — kubelet failed to pull the container image and is in exponential backoff.

**Diagnosis:**

```bash
kubectl describe pod <pod>
# Events show the specific pull error:
#   - "manifest for ...:1.4.2 not found"          → wrong tag
#   - "unauthorized" / "authentication required"  → missing imagePullSecrets
#   - "no such host"                              → DNS/network issue
#   - "rate limit exceeded"                       → DockerHub rate limit
#   - "no space left on device"                   → node disk full
```

**Common root causes:**

1. **Wrong image name or tag** — typo, image doesn't exist in registry, tag was deleted.
2. **Missing imagePullSecrets** for private registries (not needed on EKS+ECR if node has `AmazonEC2ContainerRegistryReadOnly` or IRSA).
3. **Authentication expired** — registry token rotation needed.
4. **Network can't reach registry** — pod's network can't egress; missing VPC endpoint for ECR; DNS resolution failure.
5. **Rate limit exceeded** — DockerHub anonymous pull limit; mitigate via authenticated pulls or ECR mirror.
6. **Image is for a different architecture** — ARM image on x86 nodes.
7. **KMS issue** — image encrypted with CMK the node can't decrypt.
8. **Node disk full** — `kubelet image GC` failed.

**Diagnostic commands:**

```bash
# Verify image exists in ECR
aws ecr describe-images --repository-name myapp \
  --image-ids imageTag=1.4.2

# Verify imagePullSecrets
kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets}'

# Test pull from a node
crictl pull <image>

# Check node's IAM role (EKS)
aws iam get-role --role-name <node-role-name>
```

**Fix it forward:** once the underlying issue is resolved, kubelet retries on its backoff schedule. Or delete the pod for immediate retry:
```bash
kubectl delete pod <pod>
```

**For both states — production prevention:**

- **CrashLoopBackOff prevention:**
  - Sensible probes (startup probe for slow apps, separate liveness vs readiness).
  - Resource limits sized from VPA recommendations.
  - Init containers verifying dependencies before main container starts.
  - Comprehensive config validation at startup with clear error messages.

- **ImagePullBackOff prevention:**
  - Use immutable image tags (no `latest` in prod).
  - Verify images exist before deploying (`docker manifest inspect`).
  - VPC endpoints for ECR to avoid NAT/internet dependency.
  - Image lifecycle policies in ECR so referenced images don't get cleaned up.
  - For multi-arch needs, use manifest lists with both amd64 and arm64.

**Architect's framing:**
> "CrashLoopBackOff is an application issue 95% of the time — bad config, missing dependency, misconfigured probe. The systematic mistake is looking at the current pod's logs instead of `--previous` for the actual crash. ImagePullBackOff is an infrastructure issue — auth, network, or wrong reference. The `describe pod` Events section tells you in 5 seconds which it is."

---

## 🔹 Monitoring & Observability

### Why Grafana?

Grafana is an open-source **visualization and dashboard platform** that connects to multiple data sources and presents observability data in interactive, customizable views.

**Why it's the default choice:**

**1. Data-source agnostic**

Grafana connects to dozens of data sources via plugins:
- **Metrics**: Prometheus, InfluxDB, Graphite, CloudWatch, Azure Monitor, Datadog.
- **Logs**: Loki, Elasticsearch, CloudWatch Logs, Splunk.
- **Traces**: Tempo, Jaeger, Zipkin, X-Ray.
- **Databases**: PostgreSQL, MySQL, MSSQL — for business metrics.
- **Cloud-specific**: AWS, Azure, GCP native services.

This makes Grafana the single pane of glass over a polyglot observability stack.

**2. Rich visualization options**

Beyond simple line charts:
- Time series, bar charts, pie charts.
- Heatmaps for distributions.
- Stat panels for single big numbers.
- Tables with sortable columns.
- Geographic maps with metric overlays.
- Status panels with thresholds and colors.
- Flame graphs for traces.
- Logs panels with full-text search.

**3. Powerful query languages exposed**

Grafana doesn't hide the query language — you write PromQL, LogQL, KQL directly. This means dashboards are reproducible, exportable, and version-controllable.

**4. Templating with variables**

Variables (`$namespace`, `$pod`, `$instance`) let one dashboard serve many contexts. Click a dropdown to switch from staging to production, from one service to another.

```promql
sum(rate(http_requests_total{namespace="$namespace", service="$service"}[5m]))
```

**5. Alerting**

Grafana has its own alerting engine that can fire alerts based on dashboard queries, route to PagerDuty/Slack/email, and respect silences and maintenance windows. Alternative: use the data source's native alerting (Prometheus Alertmanager, CloudWatch Alarms).

**6. Annotations**

Overlay events on time series — "this is when the deploy happened, this is when the alert fired, this is when the user reported the issue." Critical for incident analysis.

**7. Dashboards as code**

Dashboards are JSON; they can be:
- Version-controlled in Git.
- Generated programmatically (Grafonnet, Jsonnet).
- Provisioned via ConfigMaps in Kubernetes.
- Reviewed in PRs alongside service code.

**8. Open source with enterprise option**

Free for most use, with Grafana Cloud and Grafana Enterprise for managed/scaled deployments. The OSS version is genuinely capable.

**9. Plugin ecosystem**

Custom panels (status timeline, business panels, geo maps), authentication integrations (OIDC, SAML), data source connectors.

**10. Multi-tenancy and RBAC**

Folders, teams, role-based access — supports large orgs with multiple teams sharing one Grafana.

**Why not just use the data source's UI directly?**

Most data sources have their own UI (CloudWatch Console, Prometheus's UI, Kibana). Grafana wins because:
- **Unified experience** — same UI for metrics, logs, and traces across sources.
- **Cross-source correlation** — a single dashboard shows app metrics from Prometheus alongside logs from Loki alongside infra metrics from CloudWatch.
- **Polish** — Grafana's UX is significantly better than most native data source UIs.

**Architect's framing:**
> "Grafana wins because it's the data source-agnostic visualization layer in an ecosystem where you'll never use just one data source. Your metrics might live in Prometheus, logs in Loki, traces in Tempo, business data in Postgres — Grafana shows all of it in one dashboard. The alternative is a tab per data source, which is what every team starts with and grows to hate. Grafana is the unification."

### Types of graphs and dashboards in Grafana

**Panel types (the building blocks):**

**Time series panels:**
- **Time series (line/area/bar)** — the workhorse. Show metric over time. RED metrics, resource usage, throughput.
- **State timeline** — show categorical state changes over time (e.g., pod status: Running/Pending/Failed).
- **Status history** — like state timeline but for discrete events.

**Statistical panels:**
- **Stat** — single big number (current value, with delta from previous period).
- **Gauge** — circular meter with thresholds.
- **Bar gauge** — horizontal bars with thresholds, good for top-N lists.

**Comparison panels:**
- **Bar chart** — categorical comparison.
- **Pie chart** — proportional breakdown (overused; usually bar is better).
- **Histogram** — distribution of values.
- **Heatmap** — 2D distribution over time, great for latency distributions.

**Data panels:**
- **Table** — sortable rows with conditional formatting.
- **Logs** — log lines with filtering and full-text search.
- **Trace view** — distributed trace waterfall.

**Specialized panels:**
- **Geomap** — world map with metric values by region/country.
- **Node graph** — service dependencies visualized.
- **Canvas** — custom graphical compositions.
- **Flame graph** — for profile data, exec stack analysis.
- **Annotation list** — recent annotations as a list.
- **Dashboard list** — links to other dashboards (navigation).

**Dashboard types I typically build:**

**1. Service Overview dashboard**

Per microservice. Top section: SLO compliance, error budget remaining, current request rate, error rate, p99 latency. Middle: RED metrics (Rate, Errors, Duration) over time. Bottom: resource usage, downstream dependency health.

```
┌────────────┬────────────┬────────────┬────────────┐
│ SLO: 99.95%│ Budget: 73%│ Req/s: 1.2k│ p99: 184ms │
│ ✓ Healthy  │ ✓          │            │            │
├────────────┴────────────┴────────────┴────────────┤
│              Request Rate (line)                  │
├──────────────────┬────────────────────────────────┤
│ Error Rate (line)│ p50/p95/p99 Latency (lines)    │
├──────────────────┴────────────────────────────────┤
│              CPU/Memory per pod (lines)           │
└───────────────────────────────────────────────────┘
```

**2. Infrastructure dashboard**

Node-level: CPU, memory, disk, network per node. Cluster-level: pod count, pending pods, eviction count. Cost: spend per namespace.

**3. SLO dashboard**

Org-wide view of all services' SLO compliance, error budgets remaining, recent breaches. Heatmap of which services are red/yellow/green.

**4. Incident response dashboard**

What an on-call engineer pulls up first. RED metrics for the affected service, deploy timeline (annotations), error rate by endpoint, recent log volume, downstream dependency status. Designed for fast triage.

**5. Business KPIs dashboard**

Orders/sec, signups/min, payments processed, revenue rate. The numbers that map directly to business value. Often a dashboard the product/business team also looks at.

**6. Cost dashboard**

EKS spend per namespace via Kubecost. Top 10 most expensive workloads. Cost trend week-over-week. Right-sizing opportunities.

**7. Security dashboard**

Failed auth attempts, suspicious admission policy violations, container runtime alerts from Falco, image scan findings.

**8. Capacity planning dashboard**

CPU/memory utilization vs requests across the cluster. Forecast lines. HPA scaling events. Cluster autoscaler activity.

**Dashboard best practices I follow:**

- **One purpose per dashboard** — a Service Overview is different from an Incident Response dashboard, even for the same service.
- **Variables for context switching** — `$namespace`, `$environment`, `$service` dropdowns at the top.
- **Annotations for deploys** — every deploy posts an annotation to Grafana so you can correlate metric changes with releases.
- **Links between dashboards** — drill from cluster view to service view to pod view.
- **Sensible defaults for time range** — last 1h on incident dashboards, last 30d on SLO dashboards.
- **Versioned in Git** — dashboards are JSON, kept in a repo, provisioned to Grafana.
- **Consistent color coding** — green/yellow/red for status, blue/red for normal/anomaly.
- **Document the queries** — comments explaining what each panel measures.

**Architect's framing:**
> "The dashboard hierarchy matters: overview dashboards for monitoring, incident dashboards for response, deep-dive dashboards for analysis. The mistake teams make is one dashboard with everything — 50 panels, all important, none actionable in the moment. Each dashboard should answer a specific question: 'Is my service healthy right now?' or 'Where is the bottleneck during this incident?' If a dashboard can't answer that question in 10 seconds, it's wrong."

---

## 🔹 CI/CD & Infrastructure as Code

### Pipeline that executes from the current repository

Going with GitHub Actions since it's natively repository-aware. A representative pipeline:

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  id-token: write       # OIDC to AWS
  contents: read
  pull-requests: write

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp

jobs:
  # ─────────────────────────────────────────────────────────
  # Build & test on every PR and push
  # ─────────────────────────────────────────────────────────
  build:
    name: Build & Test
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Set image tag
        id: meta
        run: |
          TAG="$(date +%Y%m%d)-${GITHUB_SHA::7}-${GITHUB_RUN_NUMBER}"
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - name: Unit tests + coverage
        run: mvn -B clean verify

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: JUnit Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit

      - name: SonarQube scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.host.url=${{ secrets.SONAR_URL }} \
            -Dsonar.qualitygate.wait=true \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

      - name: Dependency vulnerability scan
        uses: snyk/actions/maven@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  # ─────────────────────────────────────────────────────────
  # Build, scan, sign, and push image
  # ─────────────────────────────────────────────────────────
  image:
    name: Build & Push Image
    needs: build
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

      - name: Build & push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ needs.build.outputs.image_tag }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ needs.build.outputs.image_tag }}
          severity: CRITICAL,HIGH
          exit-code: '1'
          ignore-unfixed: true

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign image (keyless via OIDC)
        run: |
          cosign sign --yes ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ needs.build.outputs.image_tag }}

  # ─────────────────────────────────────────────────────────
  # Deploy: dev (auto on develop), prod (manual approval)
  # ─────────────────────────────────────────────────────────
  deploy-dev:
    name: Deploy to Dev
    needs: [build, image]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: company/gitops-config
          token: ${{ secrets.GITOPS_PAT }}

      - name: Bump image tag in dev values
        run: |
          yq -i '.image.tag = "${{ needs.build.outputs.image_tag }}"' \
            environments/dev/values.yaml

      - name: Commit & push
        run: |
          git config user.name "ci-bot"
          git config user.email "ci-bot@company.com"
          git add .
          git commit -m "chore(dev): bump myapp to ${{ needs.build.outputs.image_tag }}"
          git push

  deploy-prod:
    name: Deploy to Production
    needs: [build, image]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production    # GitHub Environment with required reviewers
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: company/gitops-config
          token: ${{ secrets.GITOPS_PAT }}

      - name: Bump image tag in prod values
        run: |
          yq -i '.image.tag = "${{ needs.build.outputs.image_tag }}"' \
            environments/prod/values.yaml

      - name: Open PR for prod deployment
        run: |
          git checkout -b deploy/prod-${{ needs.build.outputs.image_tag }}
          git config user.name "ci-bot"
          git config user.email "ci-bot@company.com"
          git add . && git commit -m "chore(prod): bump myapp"
          git push origin HEAD
          gh pr create --title "Deploy ${{ needs.build.outputs.image_tag }} to prod" \
            --body "Promotion from develop" --base main
        env:
          GH_TOKEN: ${{ secrets.GITOPS_PAT }}
```

**Key principles in this pipeline:**

- **OIDC federation to AWS** — no long-lived credentials in GitHub secrets.
- **Build once, promote artifact** — same image deploys to dev and prod.
- **Quality gates fail fast** — Sonar, Snyk, Trivy all gate.
- **Image signing** — Cosign in CI; Kyverno verifies at admission.
- **GitOps separation** — pipeline doesn't deploy directly; it commits the image tag to a GitOps repo. ArgoCD does the cluster work.
- **Environment-gated deploys** — GitHub Environments enforce approval and secret scoping.

### Terraform — what is it

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp that lets you define cloud and on-premises resources declaratively in human-readable configuration files, then provision and manage them through a consistent workflow.

**Core properties:**

**1. Declarative**

You describe the *desired state* of infrastructure — what should exist — rather than the steps to create it. Terraform figures out what to add, change, or delete to match.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "company-app-logs"
}
```

No "create bucket if doesn't exist" — just "this bucket should exist."

**2. Multi-cloud / multi-provider**

Terraform works across AWS, Azure, GCP, Kubernetes, GitHub, Datadog, Cloudflare, and 3,000+ other providers. One tool, one language, many platforms.

```hcl
provider "aws" {}
provider "azurerm" {}
provider "kubernetes" {}
```

**3. State-backed**

Terraform maintains a state file mapping code resources to real-world resources. State is what enables drift detection, dependency tracking, and incremental updates.

**4. Plan-before-apply**

```bash
terraform plan       # show me what would change
terraform apply      # do it
```

You see exactly what will be created, modified, or destroyed before any change reaches the cloud.

**5. Modular**

Reusable building blocks (modules) let you compose infrastructure from versioned, parameterized components.

**6. Provider ecosystem**

The Terraform Registry has thousands of community modules and providers. The AWS provider alone supports hundreds of services.

**7. HCL — HashiCorp Configuration Language**

Purpose-built syntax that's more readable than JSON/YAML and more powerful than YAML (functions, expressions, conditionals).

**Why it became the de facto standard:**

- **Cloud-agnostic** — same workflow regardless of provider.
- **Mature ecosystem** — modules, examples, community.
- **State management** — the missing piece in earlier IaC tools.
- **Plan output** — reviewable, auditable changes.
- **GitOps-friendly** — config in Git, changes via PR.

**Modern context:**

- HashiCorp changed Terraform's license in 2023 from MPL to BSL.
- **OpenTofu** is an MIT-licensed fork maintained by the Linux Foundation.
- Functionally equivalent at the time of writing; the API and HCL syntax are identical.
- For new projects in license-sensitive orgs, OpenTofu is increasingly the choice.

**Architect's framing:**
> "Terraform is the default IaC tool because it solved the right problems at the right time — declarative, multi-cloud, state-backed, plan-before-apply. The alternatives (CloudFormation, ARM, CDK, Pulumi) each have advantages but Terraform's combination of breadth and discipline made it the universal answer. The skill is portable; what I know about Terraform on AWS transfers to Azure, GCP, and managing Kubernetes, Datadog, GitHub repos, anything with a provider."

### Terraform block to launch an EC2 instance

A production-quality EC2 launch goes beyond the minimal `resource "aws_instance"` block. Here's what I'd actually write:

```hcl
# Pin Terraform and provider versions
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = var.project
    }
  }
}

# Look up the latest Amazon Linux 2023 AMI rather than hardcoding
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Security group restricting access
resource "aws_security_group" "web" {
  name        = "${var.name}-sg"
  description = "Security group for ${var.name}"
  vpc_id      = var.vpc_id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [var.alb_security_group_id]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.name}-sg" }
}

# IAM instance profile for SSM and CloudWatch
resource "aws_iam_role" "instance" {
  name = "${var.name}-instance-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.instance.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.instance.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

resource "aws_iam_instance_profile" "instance" {
  name = "${var.name}-instance-profile"
  role = aws_iam_role.instance.name
}

# The EC2 instance itself
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.instance.name

  # Use IMDSv2 only (more secure than v1)
  metadata_options {
    http_tokens                 = "required"
    http_endpoint               = "enabled"
    http_put_response_hop_limit = 1
  }

  # Encrypted root volume
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 30
    encrypted             = true
    kms_key_id            = var.kms_key_arn
    delete_on_termination = true
    tags = { Name = "${var.name}-root" }
  }

  # Bootstrap via user data
  user_data = <<-EOF
    #!/bin/bash
    set -e
    dnf update -y
    dnf install -y amazon-cloudwatch-agent

    # Install application
    cat > /etc/systemd/system/myapp.service <<'SERVICE'
    [Unit]
    Description=MyApp
    After=network-online.target

    [Service]
    Type=simple
    ExecStart=/usr/local/bin/myapp
    Restart=always

    [Install]
    WantedBy=multi-user.target
    SERVICE

    systemctl daemon-reload
    systemctl enable --now myapp
  EOF

  user_data_replace_on_change = true   # force replacement when user_data changes

  monitoring                  = true   # detailed CloudWatch monitoring
  disable_api_termination     = var.environment == "prod"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = var.name
  }
}

# Variables
variable "name"               { type = string }
variable "region"             { type = string  default = "us-east-1" }
variable "environment"        { type = string }
variable "project"            { type = string }
variable "vpc_id"             { type = string }
variable "subnet_id"          { type = string }
variable "instance_type"      { type = string  default = "t3.small" }
variable "alb_security_group_id" { type = string }
variable "kms_key_arn"        { type = string }

# Outputs
output "instance_id"  { value = aws_instance.web.id }
output "private_ip"   { value = aws_instance.web.private_ip }
output "instance_arn" { value = aws_instance.web.arn }
```

**Key production considerations included:**

- **Versions pinned** for Terraform and provider.
- **AMI looked up dynamically** rather than hardcoded — gets the latest patched image.
- **Security group with SG references** (not CIDR) for ALB → instance access.
- **IAM instance profile** for SSM (no SSH needed) and CloudWatch (metrics + logs).
- **IMDSv2 only** — prevents SSRF-based credential theft.
- **Encrypted root volume** with customer-managed KMS key.
- **gp3 storage** (cheaper and equivalent performance vs gp2).
- **`user_data_replace_on_change`** — changes to user_data trigger replacement.
- **Detailed monitoring** enabled for granular CloudWatch metrics.
- **API termination protection** in production.
- **`create_before_destroy`** lifecycle for zero-downtime replacement.
- **Default tags** applied via provider for cost allocation.

**The minimal version for context:**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abc123def456"
  instance_type = "t3.micro"
}
```

This works but is the absolute minimum — no security group, no IAM, hardcoded AMI, no encryption, no monitoring. Suitable for a quick test, never for production.

**Architect's framing:**
> "The minimal EC2 block is two lines; the production block is fifty. The difference is everything that's not the instance itself — security group, IAM profile, KMS encryption, IMDSv2 enforcement, monitoring, lifecycle protection. None of those are about EC2 per se; they're about operating EC2 safely. The skill in writing Terraform isn't knowing the resource block syntax — it's knowing all the surrounding resources that make a workload production-ready."

---

That covers all the questions. Likely follow-ups to be ready for:

- **"How would you measure that your SLO is appropriate?"** — derived from real user pain, not engineering vibes; periodically re-evaluated; compared against competitor SLAs.
- **"Walk me through a real incident where error budget burn rate alerted you in time."** — have a specific story ready.
- **"What's the difference between `terraform import` and `data` blocks?"** — import brings unmanaged resources under TF management; data blocks read existing resources for reference without managing them.
- **"How do you handle stateful workloads in K8s for a regulated environment?"** — operators, encrypted PVs, strict backup, PDBs, and honestly: prefer managed services where possible.