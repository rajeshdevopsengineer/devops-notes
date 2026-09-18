# AKS and Azure DevOps Interview Questions and Answers

> These answers are structured for a mid-to-senior DevOps interview. They cover implementation, security, availability, cost, and operational trade-offs.

---

## 1. How do you use Azure Key Vault secrets in AKS?

The recommended pattern is **Azure Key Vault Provider for Secrets Store CSI Driver** with **Microsoft Entra Workload ID**. It avoids storing long-lived client secrets in Kubernetes. A Pod's Kubernetes service account is federated with a Microsoft Entra managed identity, which receives only the required Key Vault permissions. The CSI driver mounts secrets, keys, or certificates into the Pod as files. Microsoft also supports managed identity and Service Connector access patterns. citeturn8search194turn8search195turn8search197

### High-level flow

```text
Pod
  -> Kubernetes ServiceAccount token
  -> AKS OIDC issuer
  -> Microsoft Entra federated identity
  -> User-assigned managed identity
  -> Azure Key Vault
  -> CSI-mounted secret files
```

### Step 1: Enable OIDC, Workload Identity, and the Key Vault CSI add-on

For a new AKS Standard cluster:

```bash
az aks create \
  --resource-group rg-platform-prod \
  --name aks-prod \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-addons azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --generate-ssh-keys
```

For an existing cluster:

```bash
az aks update \
  --resource-group rg-platform-prod \
  --name aks-prod \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-addons azure-keyvault-secrets-provider \
  --enable-secret-rotation
```

AKS Standard requires Workload Identity and the OIDC issuer to be enabled. AKS Automatic enables these cluster-level capabilities by default. citeturn8search194turn8search196

### Step 2: Create a user-assigned managed identity

```bash
IDENTITY_NAME="id-payment-api"
RESOURCE_GROUP="rg-platform-prod"

az identity create \
  --name "$IDENTITY_NAME" \
  --resource-group "$RESOURCE_GROUP"

CLIENT_ID=$(az identity show \
  --name "$IDENTITY_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query clientId -o tsv)

PRINCIPAL_ID=$(az identity show \
  --name "$IDENTITY_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query principalId -o tsv)
```

### Step 3: Grant least-privilege access to Key Vault

If the vault uses Azure RBAC, assign `Key Vault Secrets User` at the narrowest practical scope:

```bash
KEYVAULT_ID=$(az keyvault show \
  --name kv-platform-prod \
  --resource-group rg-security-prod \
  --query id -o tsv)

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope "$KEYVAULT_ID"
```

Use separate identities and vault scopes for different applications or trust boundaries. For keys or certificates, grant the matching data-plane role rather than over-permissioning the identity.

### Step 4: Create the Kubernetes service account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: payments
  annotations:
    azure.workload.identity/client-id: "<managed-identity-client-id>"
```

### Step 5: Create the federated identity credential

```bash
OIDC_ISSUER=$(az aks show \
  --resource-group rg-platform-prod \
  --name aks-prod \
  --query oidcIssuerProfile.issuerUrl -o tsv)

az identity federated-credential create \
  --name fic-payment-api \
  --identity-name id-payment-api \
  --resource-group rg-platform-prod \
  --issuer "$OIDC_ISSUER" \
  --subject system:serviceaccount:payments:payment-api \
  --audience api://AzureADTokenExchange
```

### Step 6: Define `SecretProviderClass`

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: payment-api-keyvault
  namespace: payments
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<managed-identity-client-id>"
    keyvaultName: "kv-platform-prod"
    tenantId: "<tenant-id>"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
          objectAlias: db-password
          objectVersion: ""
        - |
          objectName: api-certificate
          objectType: secret
          objectAlias: tls.pfx
          objectVersion: ""
```

Leaving `objectVersion` empty allows the provider to follow the current secret version during rotation.

### Step 7: Mount secrets in the Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: payments
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: payment-api
      containers:
        - name: payment-api
          image: contoso.azurecr.io/payment-api:1.8.0
          volumeMounts:
            - name: keyvault-secrets
              mountPath: /mnt/secrets-store
              readOnly: true
      volumes:
        - name: keyvault-secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: payment-api-keyvault
```

The Azure provider supports CSI inline volumes, multiple Key Vault objects, Kubernetes Secret synchronization, and secret autorotation. citeturn8search197turn8search199

### Network and security considerations

- Use an Azure Key Vault private endpoint for a network-isolated cluster.
- Configure Private DNS correctly for the vault endpoint.
- Restrict Key Vault public access where the architecture permits.
- Use Azure RBAC, least privilege, soft delete, purge protection, diagnostic logs, and alerts.
- Do not log secret contents or expose the mount through debugging endpoints.
- Use a read-only filesystem and narrowly scoped identities.
- Apply NetworkPolicy and namespace isolation around sensitive applications.

Microsoft recommends private endpoint access for Key Vault from network-isolated AKS clusters and documents required egress allowances when user-defined routing and firewalls are used. citeturn8search197

---

## 2. How does a Pod fetch the latest rotated secret from Azure Key Vault?

Enable **autorotation** on the Azure Key Vault CSI Driver. The provider periodically polls Key Vault and updates the mounted files. It can also update a synchronized Kubernetes Secret configured in the `secretObjects` section. The default documented rotation polling interval is two minutes, and it can be customized. citeturn8search199

### Rotation behavior depends on how the application consumes the secret

#### A. Direct CSI file mount, preferred

```text
Key Vault secret rotates
  -> CSI rotation reconciler detects new version
  -> mounted file is updated
  -> application watches/reloads the file
```

The application must watch the file, periodically reread it, or expose a safe reload mechanism. A long-running application that reads the file only at startup will continue using the old value until it reloads or restarts. citeturn8search197turn8search199

#### B. Synced Kubernetes Secret mounted as a volume

The driver updates the Kubernetes Secret and the projected volume content. The application still needs to detect and reload the changed file. citeturn8search199

Example synchronization:

```yaml
spec:
  provider: azure
  secretObjects:
    - secretName: payment-api-runtime
      type: Opaque
      data:
        - objectName: db-password
          key: DB_PASSWORD
  parameters:
    keyvaultName: kv-platform-prod
    tenantId: "<tenant-id>"
    clientID: "<managed-identity-client-id>"
    objects: |
      array:
        - |
          objectName: database-password
          objectAlias: db-password
          objectType: secret
          objectVersion: ""
```

#### C. Kubernetes Secret consumed as an environment variable

Environment variables are fixed when the container starts. Even if the Kubernetes Secret is updated, the running process does not receive a new environment value. Restart or roll out the Pods. Microsoft suggests using a controller such as Reloader to watch the synchronized Secret and trigger a rolling restart. citeturn8search199

```bash
kubectl rollout restart deployment/payment-api -n payments
```

#### D. `subPath` mount

Do not use `subPath` when automatic secret updates are required. Kubernetes does not propagate updates to a Secret or ConfigMap mounted through `subPath`; the Pod must be restarted. citeturn8search197

### Enable and configure rotation

```bash
az aks addon update \
  --resource-group rg-platform-prod \
  --name aks-prod \
  --addon azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 5m
```

### Zero-downtime rotation pattern

For database credentials or certificates, use an overlap window:

1. Create or rotate the new credential in Key Vault.
2. Ensure the destination system temporarily accepts old and new credentials if supported.
3. Wait for CSI synchronization and application reload.
4. Validate authentication with the new credential.
5. Revoke the old credential.
6. Monitor access failures and roll back if necessary.

This avoids breaking all replicas simultaneously during rotation.

---

## 3. How do you integrate SonarQube and Snyk in an Azure Pipeline?

Use the official Azure DevOps extensions and service connections. SonarQube normally performs code-quality and static-analysis checks, while Snyk can scan open-source dependencies, source code, and container images. Configure quality gates so high-risk findings fail pull-request validation before deployment.

The SonarQube Azure DevOps extension uses Prepare, Analyze, and Publish Quality Gate tasks. It supports .NET, Maven/Gradle, and CLI analysis modes. citeturn8search200turn8search202turn8search204

The Snyk Azure Pipelines task supports application dependency scanning, container scanning, and Snyk Code. It uses a Snyk Authentication service connection and can fail the build according to the configured threshold. citeturn8search206turn8search208turn8search210

### Prerequisites

1. Install the correct SonarQube Server or SonarQube Cloud Azure DevOps extension.
2. Create a SonarQube service connection with the minimum required permissions.
3. Install the Snyk extension.
4. Create a Snyk service connection using an organization-level service account token rather than a personal token where available.
5. Limit who can use and administer each service connection.
6. Configure branch policies so the Pipeline and quality gates are required before merge.

SonarQube Server and SonarQube Cloud use distinct Azure DevOps extensions, even though their tasks have similar behavior. citeturn8search202turn8search205

### Example Azure Pipeline

Task versions can change, so validate the installed extension's current task version before use.

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  imageRepository: payment-api
  containerRegistry: acr-prod-service-connection
  imageTag: $(Build.SourceVersion)

stages:
  - stage: QualityAndSecurity
    jobs:
      - job: ScanAndBuild
        steps:
          - checkout: self
            fetchDepth: 0

          - task: SonarQubePrepare@8
            inputs:
              SonarQube: sonar-prod
              scannerMode: cli
              configMode: manual
              cliProjectKey: payment-api
              cliProjectName: payment-api
              cliSources: src

          - script: |
              npm ci
              npm test -- --coverage
            displayName: Build and unit tests

          - task: SonarQubeAnalyze@8
            displayName: Run SonarQube analysis

          - task: SonarQubePublish@8
            inputs:
              pollingTimeoutSec: '300'
            displayName: Publish quality gate

          - task: SnykSecurityScan@1
            inputs:
              serviceConnectionEndpoint: snyk-prod
              testType: app
              monitorWhen: always
              failOnIssues: true
              severityThreshold: high
            displayName: Snyk dependency scan

          - task: Docker@2
            inputs:
              command: build
              repository: $(imageRepository)
              containerRegistry: $(containerRegistry)
              Dockerfile: Dockerfile
              tags: |
                $(imageTag)

          - task: SnykSecurityScan@1
            inputs:
              serviceConnectionEndpoint: snyk-prod
              testType: container
              dockerImageName: payment-api:$(imageTag)
              dockerfilePath: Dockerfile
              failOnIssues: true
              severityThreshold: high
            displayName: Snyk container scan

          - task: Docker@2
            inputs:
              command: push
              repository: $(imageRepository)
              containerRegistry: $(containerRegistry)
              tags: |
                $(imageTag)
```

Snyk's Azure task supports `app`, `container`, and `code` test types. Its documented code-scan example uses `testType: code` and can fail the job based on severity. citeturn8search208turn8search209

### Pipeline design recommendations

- Run fast scans on pull requests and comprehensive scans on `main` or scheduled pipelines.
- Fail on new high/critical issues, not an unmanaged historical backlog.
- Establish an exception process with owner, justification, compensating control, and expiry date.
- Publish reports and retain audit evidence.
- Do not expose Sonar or Snyk tokens in scripts or logs.
- Scan the final image before pushing or promoting it.
- Generate an SBOM and sign the image.
- Promote the same immutable digest across environments.
- Pin extension/task versions after validation and maintain an upgrade process.

SonarQube can report quality gate results in the Azure Pipeline summary and can be used to prevent pull-request merging or block release progression. citeturn8search201turn8search203

---

## 4. How do you secure an AKS cluster?

Use a layered model covering identity, network, workload, supply chain, data, policy, observability, and operations.

### Identity and access

- Integrate AKS with Microsoft Entra ID.
- Use Azure RBAC for Kubernetes authorization or Kubernetes RBAC according to the governance model.
- Assign access to Entra groups, not individual users.
- Separate platform-admin, namespace-admin, developer, auditor, and read-only roles.
- Disable or tightly control local accounts and cluster-admin credentials.
- Use Privileged Identity Management for just-in-time elevated Azure roles.
- Use Microsoft Entra Workload ID for Pod-to-Azure authentication instead of client secrets.

Workload ID uses projected Kubernetes service-account tokens and OIDC federation to let Pods access Entra-protected resources without embedded long-lived Azure credentials. citeturn8search194turn8search196

### API server and network

- Prefer a private AKS cluster for sensitive production workloads.
- If the API is public, restrict authorized IP ranges.
- Use hub-spoke or Virtual WAN designs, Azure Firewall, Private Link, and controlled DNS where appropriate.
- Apply default-deny Kubernetes NetworkPolicies and allow only required flows.
- Use private endpoints for ACR, Key Vault, Storage, and other platform services.
- Restrict outbound traffic and document required AKS FQDNs.
- Protect public applications with Application Gateway or another ingress, WAF, TLS, and DDoS controls.

### Workload hardening

- Enforce Pod Security Standards, normally `restricted` where applications support it.
- Run as non-root, drop Linux capabilities, use seccomp, disallow privilege escalation, and make the root filesystem read-only.
- Set CPU/memory requests and limits.
- Avoid host networking, host PID, host IPC, privileged containers, and unrestricted hostPath mounts.
- Use namespaces, quotas, LimitRanges, priority classes, and disruption budgets.
- Apply Azure Policy for AKS or another admission-policy engine to enforce controls.

Example `securityContext`:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

### Image and supply-chain security

- Use private ACR repositories and managed identity-based pull access.
- Scan code, dependencies, IaC, and final container images.
- Use minimal, patched, pinned base images.
- Generate an SBOM and sign artifacts.
- Enforce deployment policies for approved registries and signatures where supported.
- Protect branches and use workload identity or federated service connections in CI/CD.

### Secrets and data

- Use Key Vault with Workload ID and CSI Driver.
- Avoid plaintext secrets, Git-stored secrets, and environment variables where file-based reload is practical.
- Encrypt disks and data services, control backup access, and test recovery.
- Use separate Key Vaults and managed identities for isolation where required.

The Key Vault CSI integration supports identity-based access, private endpoint designs, inline secret mounting, secret synchronization, and rotation. citeturn8search195turn8search197turn8search199

### Monitoring and operations

- Enable AKS control-plane diagnostic logs, activity logs, container logs, metrics, and audit logs.
- Integrate Microsoft Defender for Cloud/Containers where licensed.
- Alert on privileged deployments, role changes, denied requests, node pressure, certificate expiry, and anomalous egress.
- Keep AKS and node images on supported versions using planned maintenance windows.
- Back up cluster state that cannot be recreated and application data, then test restoration.
- Conduct regular access reviews, penetration tests, and incident-response exercises.

---

## 5. How do you make an AKS cluster highly available?

High availability must cover the control plane, nodes, Pods, networking, state, dependencies, and recovery.

### Cluster and node availability

- Use an AKS tier/SLA appropriate for production requirements.
- Spread system and user node pools across Availability Zones supported by the chosen region and VM size.
- Maintain at least two or three system nodes for production, according to scale and SLA requirements.
- Use separate system and user node pools.
- Use multiple node pools for different workloads, operating systems, or failure domains.
- Enable Cluster Autoscaler with realistic minimum capacity.
- Configure planned maintenance, node image upgrades, surge upgrades, and Pod disruption controls.

### Application availability

- Run multiple replicas across nodes and zones.
- Configure readiness, liveness, and startup probes correctly.
- Use topology spread constraints and Pod anti-affinity.
- Define PodDisruptionBudgets.
- Use HPA/KEDA for Pod scaling and Cluster Autoscaler for node scaling.
- Set requests and limits so scheduling and autoscaling decisions are meaningful.
- Design graceful shutdown, retry, timeout, circuit-breaker, and idempotency behavior.

Example:

```yaml
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment-api
```

### Network and ingress

- Use zone-resilient ingress/load-balancing designs.
- Run multiple ingress-controller replicas and spread them.
- Use stable health probes and sensible connection draining.
- Provide redundant NAT/firewall paths according to the network design.
- Avoid a single DNS, certificate, or outbound dependency.

### Stateful workloads

- Prefer managed zone-redundant data services where possible.
- Understand storage-zone constraints and failover behavior.
- Configure backups, replication, recovery point objective, and recovery time objective.
- Test restore and regional failover.

### Regional disaster recovery

Availability Zones protect from a datacenter/zone failure, not a full regional outage. For critical systems:

```text
Primary AKS region + secondary AKS region
  -> replicated container images
  -> replicated/configured data tier
  -> GitOps/IaC rebuild capability
  -> global traffic routing
  -> tested failover runbook
```

Avoid stretching one stateful application blindly across regions. Choose active-active or active-passive based on data consistency, latency, cost, and business RTO/RPO.

---

## 6. How do you perform cost optimization in AKS?

Start with measurement and rightsizing, then optimize capacity, architecture, and operational waste without compromising availability.

### Rightsize Pods

- Set realistic CPU and memory requests from historical usage.
- Use limits carefully, especially CPU limits that can cause throttling.
- Use Vertical Pod Autoscaler recommendations or controlled VPA modes.
- Identify unused namespaces, stale workloads, oversized replicas, and idle services.
- Tune JVM/runtime heap and concurrency rather than only increasing Pod size.

### Autoscale effectively

- Use HPA for resource or application metrics.
- Use KEDA for event-driven workloads such as queues and streams.
- Use Cluster Autoscaler to remove unused nodes and add capacity for pending Pods.
- Configure sensible minimums, maximums, stabilization windows, and scale-down policies.

AKS provides a managed KEDA add-on and supports Workload Identity for securely accessing trigger sources such as Azure Service Bus. citeturn8search198

### Optimize node pools

- Choose VM families based on actual CPU, memory, disk, network, and accelerator requirements.
- Use multiple node pools and labels/taints for workload-specific sizing.
- Use Spot node pools for fault-tolerant batch, CI, or interruptible workloads.
- Use reserved capacity or savings constructs for stable baseline demand where financially appropriate.
- Scale non-production node pools down or stop clusters on schedules where supported by the operating model.
- Use ephemeral OS disks where supported and operationally appropriate.
- Review daemonset overhead because every node may run monitoring, security, CNI, and logging agents.

### Reduce supporting-service costs

- Apply log filtering, sampling, table retention, and archive policies.
- Avoid collecting duplicate logs and high-cardinality metrics.
- Review load balancers, public IPs, disks, snapshots, NAT, firewalls, and cross-region/cross-zone data transfer.
- Use private connectivity thoughtfully because it improves security but adds endpoint and DNS costs.
- Remove unattached disks, unused public IPs, old snapshots, stale ACR images, and orphaned resources.

### Governance

- Tag resources by application, owner, environment, and cost center.
- Use Azure Cost Management budgets and anomaly alerts.
- Measure unit cost, such as cost per request, tenant, transaction, or build.
- Conduct monthly rightsizing reviews with engineering and finance.
- Treat cost regressions like reliability regressions in architecture and pull-request reviews.

### Key warning

Do not optimize only node utilization. Consolidating too aggressively may increase eviction risk, startup latency, zonal imbalance, or outage impact. Preserve headroom for failures, deployments, traffic spikes, and autoscaler reaction time.

---

## 7. HPA triggers and metric types

The Kubernetes Horizontal Pod Autoscaler changes replica count based on observed metrics. HPA commonly uses `autoscaling/v2`.

### Resource metrics

- CPU utilization or average CPU value
- Memory utilization or average memory value

These normally require Metrics Server and accurate Pod resource requests.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api
  namespace: payments
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

### Container resource metrics

Scale using one container's CPU or memory rather than the whole Pod. This is useful when a sidecar would otherwise distort the metric.

### Pods metrics

Custom metrics averaged across Pods, such as:

- Requests per second per Pod
- Active sessions
- Work queue inside the application
- In-flight requests

### Object metrics

A metric associated with a Kubernetes object, such as requests per second on an Ingress or queue depth represented by a custom-metrics adapter.

### External metrics

Metrics outside the cluster, such as:

- Azure Service Bus queue length
- Event Hubs lag
- Azure Storage Queue messages
- Prometheus query result
- Application Insights or Azure Monitor-derived metrics
- Kafka consumer lag

### KEDA event triggers

KEDA creates or manages HPA behavior based on event sources. Common triggers include:

- Azure Service Bus
- Azure Storage Queue
- Azure Event Hubs
- Kafka
- Prometheus
- Cron
- HTTP add-on patterns
- RabbitMQ
- Redis

With the AKS KEDA add-on, Workload Identity can authenticate scalers to Azure resources without storing static credentials. citeturn8search198turn8search194

### Interview answer from experience

A good answer is:

> I have used CPU and memory HPA for APIs, Prometheus custom metrics for request rate, and KEDA with Azure Service Bus queue length for asynchronous workers. For each trigger, I configured min/max replicas, stabilization windows, scale-down behavior, resource requests, and Cluster Autoscaler capacity. I validated that the downstream system could handle the additional concurrency.

### Common HPA mistakes

- Missing or unrealistic resource requests
- Scaling on memory for an application that does not release memory
- Oscillation from aggressive thresholds
- HPA reaches maximum while nodes have no capacity
- Scaling Pods without scaling downstream database connections
- Using average CPU when queue depth is the real demand signal
- Not considering startup time and warm-up
- Conflicting HPA and VPA policies

---

## 8. Stateful vs stateless applications

### Stateless application

A stateless instance does not depend on local runtime state to serve the next request. Any healthy replica can process it.

Characteristics:

- Easy horizontal scaling
- Pods are replaceable
- Sessions and durable data are externalized
- Usually deployed with a Kubernetes Deployment
- Rolling, canary, and blue-green deployments are easier

Examples include a REST API whose data is in Azure SQL/Cosmos DB and whose sessions are in Redis.

### Stateful application

A stateful instance has persistent data or stable identity/order requirements.

Characteristics:

- Stable network or storage identity may be required
- Data consistency and replication matter
- Ordered startup or shutdown may matter
- Storage failover and backup require explicit design
- Often uses StatefulSet, PVCs, headless Services, and application-specific clustering

Examples include databases, Kafka brokers, ZooKeeper-like systems, and some search clusters.

### Important distinction

A StatefulSet does not automatically make an application highly available or protect its data. The application still needs replication, quorum, backup, failure detection, and recovery. Where possible, use managed databases and messaging services instead of operating complex stateful systems in AKS.

---

## 9. How do you manage data for a stateless application?

The compute tier remains disposable by moving durable and shared state outside the Pod.

### Data patterns

- **Relational data:** Azure SQL Database or Azure Database for PostgreSQL/MySQL
- **Document/global data:** Azure Cosmos DB
- **Session/cache:** Azure Cache for Redis
- **Files and objects:** Azure Blob Storage or ADLS
- **Shared file semantics:** Azure Files where appropriate
- **Messages/events:** Azure Service Bus, Event Hubs, or Event Grid
- **Secrets:** Azure Key Vault
- **Logs and metrics:** stdout/stderr plus Azure Monitor, managed Prometheus, and Log Analytics

### Design rules

- Do not store required data in a container writable layer or `emptyDir`.
- Use retries with exponential backoff and jitter.
- Add timeouts, idempotency keys, circuit breakers, and connection-pool limits.
- Use managed identity/Workload Identity rather than connection secrets where the service supports Entra authentication.
- Cache only recreatable data and define TTL/invalidation behavior.
- Configure backup, replication, retention, encryption, and restore tests for each external data service.
- Use schema migration tools with backward-compatible deployment practices.
- Keep uploaded files outside the Pod before acknowledging success.

### Session example

```text
Client -> Ingress -> any API Pod
                    -> Redis session/cache
                    -> Azure SQL durable records
                    -> Blob Storage uploaded files
```

This lets a failed Pod be replaced without losing business data or requiring sticky sessions.

---

## 10. How do you integrate Microsoft Entra ID with AKS authentication?

There are two separate identity scenarios:

1. **Human/control-plane access:** Users authenticate to the AKS API with Microsoft Entra ID, then RBAC authorizes their Kubernetes actions.
2. **Workload access:** Pods use Microsoft Entra Workload ID to access Azure services.

Do not confuse the identity used by a developer running `kubectl` with the identity used by an application Pod.

### Enable Entra integration and Azure RBAC

For a new cluster, an example is:

```bash
az aks create \
  --resource-group rg-platform-prod \
  --name aks-prod \
  --enable-aad \
  --enable-azure-rbac \
  --disable-local-accounts \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys
```

Use Entra groups for role assignment. Example namespace-scoped role assignment:

```bash
AKS_ID=$(az aks show \
  --resource-group rg-platform-prod \
  --name aks-prod \
  --query id -o tsv)

az role assignment create \
  --assignee-object-id <entra-group-object-id> \
  --assignee-principal-type Group \
  --role "Azure Kubernetes Service RBAC Writer" \
  --scope "${AKS_ID}/namespaces/payments"
```

Cluster administrators can receive an appropriate AKS RBAC administrator role at cluster scope, while developers receive Writer or Reader access only to their namespaces.

### User authentication flow

```text
User runs az login
  -> obtains AKS credentials
  -> kubelogin/credential plugin obtains Entra token
  -> kube-apiserver validates token
  -> Azure RBAC or Kubernetes RBAC authorizes the operation
```

Example:

```bash
az login
az aks get-credentials \
  --resource-group rg-platform-prod \
  --name aks-prod

kubelogin convert-kubeconfig -l azurecli
kubectl auth can-i create deployments -n payments
kubectl auth can-i get secrets -n payments
```

### Governance recommendations

- Assign permissions to groups, not individual users.
- Use least privilege and namespace scope.
- Disable local accounts where emergency-access requirements permit.
- Protect emergency access through an audited break-glass process.
- Use PIM for time-bound administrative elevation.
- Perform access reviews and remove stale memberships.
- Keep Azure role assignment authority tightly controlled.
- Log control-plane authentication and authorization events.
- Separate deployment identities from human identities.

For Pods, enable OIDC and Workload Identity, annotate a Kubernetes service account with a managed identity client ID, create the federated credential, and grant the managed identity only the required Azure resource role. Workload Identity is specifically designed for Pod-to-Azure authentication through Kubernetes token projection and OIDC federation. citeturn8search194turn8search196

---

## Rapid Revision

- Use **Key Vault CSI Driver + Entra Workload ID** for secret mounting without long-lived credentials.
- CSI autorotation updates mounted files; the application must reload them.
- Environment variables require a Pod restart to receive a rotated secret.
- Do not use `subPath` when automatic secret updates are required.
- SonarQube uses Prepare, Analyze, and Publish Quality Gate tasks in Azure Pipelines.
- Snyk scans dependencies, source code, and container images through its Azure task or CLI.
- Secure AKS with Entra/RBAC, private networking, NetworkPolicy, workload hardening, supply-chain checks, Key Vault, policy enforcement, and audit logging.
- Achieve HA across zones, nodes, Pods, ingress, data services, and regions according to RTO/RPO.
- Optimize cost using rightsizing, HPA/KEDA, Cluster Autoscaler, appropriate node pools, Spot capacity, log controls, and waste removal.
- HPA supports resource, container-resource, Pods, object, and external metrics; KEDA adds event-driven triggers.
- Stateless compute externalizes sessions, durable records, objects, queues, secrets, logs, and metrics.
- Human AKS access and Pod workload identity are separate Entra integration scenarios.

---

## Official References

- Microsoft Entra Workload ID for AKS: https://learn.microsoft.com/azure/aks/workload-identity-overview
- Deploy AKS with Workload ID: https://learn.microsoft.com/azure/aks/workload-identity-deploy-cluster
- Key Vault CSI identity access: https://learn.microsoft.com/azure/aks/csi-secrets-store-identity-access
- Azure Key Vault CSI Driver for AKS: https://learn.microsoft.com/azure/aks/csi-secrets-store-driver
- Key Vault CSI rotation: https://learn.microsoft.com/azure/aks/csi-secrets-store-configuration-options
- KEDA with Workload Identity on AKS: https://learn.microsoft.com/azure/aks/keda-workload-identity
- SonarQube Azure Pipelines integration: https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/ci-integration/azure-pipelines/integration-overview
- SonarQube Azure DevOps extension: https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/scanners/sonarqube-extension-for-azure-devops
- Snyk Azure Pipelines integration: https://docs.snyk.io/developer-tools/integrations/snyk-ci-cd-integrations/azure-pipelines-integration
- Snyk task parameters: https://docs.snyk.io/developer-tools/integrations/snyk-ci-cd-integrations/azure-pipelines-integration/snyk-security-scan-task-parameters-and-values
