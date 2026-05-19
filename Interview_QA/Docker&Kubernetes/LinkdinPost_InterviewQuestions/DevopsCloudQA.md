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