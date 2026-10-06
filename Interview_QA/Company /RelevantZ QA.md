# RelevantZ DevOps Interview Questions

**Experience:** 5 years  
**Role:** DevOps Engineer

This guide covers database migration, Kubernetes, Terraform, Azure DevOps, certificates, networking, monitoring, data pipelines, and disaster recovery.

> **Note:** Examples are illustrative and have not been executed against a live environment. Replace placeholders and validate permissions, compatibility, costs, and change approvals before production use.
>
> For experience-based questions, describe your actual contribution. If you have not performed the activity, explain your proposed approach honestly.

---

## 1. Have you migrated an on-premises database to the cloud, especially PostgreSQL? What is its port?

### PostgreSQL port

The default PostgreSQL TCP port is **5432**. It can be configured differently.

### Migration approaches

| Approach | Suitable situation | Main consideration |
|---|---|---|
| Logical dump and restore | A migration where a planned write outage is acceptable. | Changes made after the dump must be accounted for. |
| Online migration | A migration requiring reduced cutover downtime. | Requires change capture, synchronization, and controlled cutover. |

Azure Database for PostgreSQL provides a migration service supporting on-premises and other PostgreSQL sources.

### Suggested migration workflow

This is my recommended operational checklist:

1. **Assess the source**
   - PostgreSQL version.
   - Database size.
   - Extensions.
   - Roles and permissions.
   - Application dependencies.
   - Performance baseline.

2. **Prepare the destination**
   - Select suitable compute and storage.
   - Configure private connectivity and DNS.
   - Configure authentication.
   - Confirm extension and version compatibility.

3. **Rehearse**
   - Run a migration in a non-production environment.
   - Validate schema, data, permissions, and application behavior.
   - Measure the migration and recovery process.

4. **Execute**
   - For offline migration, stop application writes before the final export.
   - For online migration, perform initial loading and synchronize subsequent changes.

5. **Cut over**
   - Stop source writes.
   - Confirm synchronization.
   - Switch application connections.
   - Validate application transactions.

6. **Monitor and retain rollback options**
   - Monitor errors, latency, connections, and database performance.
   - Retain the source according to the approved migration plan.

### Example: Logical export and restore

Use compatible PostgreSQL client tools and approved authentication.

```bash
# Export one database.
pg_dump \
  -h SOURCE_HOST \
  -p 5432 \
  -U migration_user \
  -d application \
  -Fc \
  -f application.dump

# Restore into an existing, empty destination database.
pg_restore \
  -h TARGET_HOST \
  -p 5432 \
  -U migration_user \
  -d application \
  --no-owner \
  --no-acl \
  --exit-on-error \
  application.dump
```

### Important caveats

- `pg_dump` exports one database, not all cluster-wide objects.
- `--no-owner` and `--no-acl` intentionally omit ownership and privilege restoration. Reapply approved permissions separately.
- A consistent dump does not include writes committed after its snapshot.
- Do not put passwords directly in command arguments.
- Restore only trusted dumps.
- After destination writes begin, rollback requires a data-reconciliation strategy. Simply switching back can lose new transactions.

### Interview answer template

> “My migration approach depends on downtime requirements. I assess compatibility, prepare private connectivity, rehearse the migration, validate data and permissions, and perform a controlled cutover. PostgreSQL uses TCP 5432 by default.”

**References:** [PostgreSQL connection settings](https://www.postgresql.org/docs/current/runtime-config-connection.html), [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html), [pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html), [Azure PostgreSQL migration service](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql).

---

## 2. How do you log in to a Pod using kubectl?

### Explanation

Normally, you do not SSH into a Pod. You execute a command inside a running container using `kubectl exec`.

### Open a shell

```bash
kubectl exec -it POD_NAME \
  -n NAMESPACE \
  -- /bin/sh
```

If Bash exists:

```bash
kubectl exec -it POD_NAME \
  -n NAMESPACE \
  -- /bin/bash
```

### Select a container in a multi-container Pod

```bash
kubectl exec -it POD_NAME \
  -n NAMESPACE \
  -c CONTAINER_NAME \
  -- /bin/sh
```

### Execute a single command

```bash
kubectl exec POD_NAME \
  -n NAMESPACE \
  -c CONTAINER_NAME \
  -- ls /app
```

### Meaning of the options

- `-i`: Keep standard input open.
- `-t`: Allocate a terminal.
- `-n`: Namespace.
- `-c`: Container.
- `--`: Separates kubectl arguments from the container command.

### What if the image has no shell?

Minimal or distroless images might not contain `/bin/sh`.

Use an approved debugging image and the required permissions:

```bash
kubectl debug -it POD_NAME \
  -n NAMESPACE \
  --image=APPROVED_DEBUG_IMAGE \
  --target=CONTAINER_NAME
```

### Production recommendation

Use container access for diagnosis, not permanent fixes. Update the image or deployment configuration and redeploy through the approved process.

### Interview answer

> “I use kubectl exec, specifying the namespace and container when needed. If the image has no shell, I use an approved ephemeral debug container.”

**Reference:** [Debug running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/).

---

## 3. Two environments exist. One must not receive deployments; the other must be deployed through Terraform. What strategy would you use?

### Main principle

Enforce the restriction through **identity and access controls**, not only a pipeline condition.

### Suggested design

| Control | Protected environment | Deployable environment |
|---|---|---|
| Deployment identity | No write permission for the pipeline identity. | Least-privilege deployment permission. |
| Terraform root | Separate configuration. | Separate configuration. |
| State | Separate state. | Separate state. |
| Pipeline | No deployment path. | Reviewed plan and controlled apply. |
| Service connection | Not authorized for this pipeline. | Authorized only for approved pipelines. |

### Suggested repository structure

```text
terraform/
├── modules/
│   └── application/
└── environments/
    ├── protected/
    └── deployable/
```

### Execution example

Run Terraform only from the deployable root:

```bash
terraform -chdir=environments/deployable init

terraform -chdir=environments/deployable plan \
  -out=tfplan

# Execute only through the approved deployment workflow.
terraform -chdir=environments/deployable apply tfplan
```

### Azure DevOps controls

Approvals and checks can protect environments and service connections.

They are managed outside pipeline YAML, so changing YAML does not itself change those checks.

### Important traps

- A pipeline condition is not a security boundary.
- `prevent_destroy` prevents certain destruction plans, not all changes.
- Setting `count = 0` for existing managed resources can propose destruction.
- Separate `.tfvars` files alone do not isolate state.
- Terraform controls do not prevent application deployment through another tool. Restrict those deployment identities too.

### Interview answer

> “I isolate configuration, state, and identities. The pipeline has write access only to the deployable environment. I also restrict service connections and use reviewed plans and resource-owner checks.”

**References:** [Terraform modules](https://developer.hashicorp.com/terraform/language/modules), [Azure pipeline approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops).

---

## 4. How do you build a DC/DR setup, and what is the purpose of DR?

### Definitions

- **DC:** Usually the primary data center or production location.
- **DR:** Disaster recovery, the capability to restore business operations after a major disruption.
- **RTO:** Maximum acceptable time to restore operations.
- **RPO:** Maximum acceptable data loss measured in time.

### DR patterns

| Pattern | Design | Trade-off |
|---|---|---|
| Backup and restore | Rebuild infrastructure and restore data after an incident. | Lower standby cost, more recovery work. |
| Cold standby | Minimal or no secondary infrastructure running. | Recovery requires deployment and restoration. |
| Warm standby | Secondary environment runs at reduced capacity. | More standby cost, less recovery work. |
| Active-active | Multiple locations serve production traffic. | Greater operational and data-consistency complexity. |

### Suggested Azure design

For a VM-based application, I would evaluate:

- Primary application resources in one region.
- Recovery networking in another region.
- Azure Site Recovery for supported VM replication.
- Database-native recovery or replication.
- Independent backups.
- Traffic-routing changes during failover.
- Reusable infrastructure code.
- Recovery runbooks and drills.

Azure Site Recovery supports replication of Azure VMs to a target region, failover, and failback.

### Suggested recovery sequence

1. Declare the disaster using predefined criteria.
2. Recover required network and identity dependencies.
3. Recover the database.
4. Recover application services.
5. Validate data and application health.
6. Switch traffic.
7. Monitor recovery.
8. Plan failback and data reconciliation.

This sequence must be adapted to actual application dependencies.

### Production caveats

- High availability is not a complete DR strategy.
- Replication is not a replacement for backups.
- Replication can propagate unwanted changes.
- Test regional capacity, quotas, DNS, secrets, certificates, and external dependencies.
- Prevent simultaneous uncontrolled writes to independent database copies.

### Interview answer

> “I start with business RTO and RPO, select an appropriate standby pattern, protect data, automate infrastructure recovery, and test failover and failback. DR is successful only when the application and its dependencies recover together.”

**References:** [Azure disaster recovery strategies](https://learn.microsoft.com/en-us/azure/well-architected/reliability/disaster-recovery), [Azure-to-Azure Site Recovery architecture](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-architecture).

---

## 5. What cost-optimization methods have you used?

### Answer honestly

Describe methods you actually implemented. Do not invent savings percentages.

### Suggested optimization checklist

| Area | Method to evaluate |
|---|---|
| Compute | Right-size using workload metrics. |
| Non-production | Schedule approved shutdown or deallocation. |
| Scaling | Match capacity to demand. |
| Commitments | Evaluate reservations or savings plans after right-sizing. |
| Storage | Review unused capacity, retained versions, and lifecycle requirements. |
| Logging | Reduce unnecessary ingestion and excessive retention. |
| Networking | Investigate avoidable data transfer. |
| Ownership | Tag resources and identify unused resources. |

### Reservations versus savings plans

- Reservations suit stable, predictable eligible usage.
- Savings plans use an hourly spending commitment for eligible usage.
- Unused hourly savings-plan commitment does not roll over.

### Suggested workflow

1. Establish a cost baseline.
2. Identify the largest cost drivers.
3. Remove waste and right-size.
4. Validate application performance.
5. Evaluate commitments against the optimized baseline.
6. Monitor realized savings and commitment utilization.

### Example interview template

> “I reviewed utilization and cost by workload, removed approved unused resources, and right-sized capacity without violating performance targets. I evaluated commitments only after establishing a stable baseline.”

Add actual evidence from your project: the resource, change, validation, and measured result.

> **Interview trap:** A discount on oversized infrastructure still leaves waste.

**Reference:** [Choose between reservations and savings plans](https://learn.microsoft.com/en-us/azure/cost-management-billing/savings-plan/decide-between-savings-plan-reservation).

---

## 6. I am writing 100 lines of Terraform code. How can I avoid repetition?

### Main techniques

- Modules for reusable groups of resources.
- `for_each` for similar resources.
- Variables for configurable inputs.
- Locals for shared expressions.
- Dynamic blocks for repeated nested configuration when appropriate.

The goal is maintainability, not the smallest possible line count.

### Example: Multiple resource groups

Assuming the AzureRM provider is configured:

```hcl
variable "resource_groups" {
  type = map(object({
    location = string
  }))

  default = {
    "rg-application-dev" = {
      location = "centralindia"
    }
    "rg-monitoring-dev" = {
      location = "centralindia"
    }
  }
}

resource "azurerm_resource_group" "groups" {
  for_each = var.resource_groups

  name     = each.key
  location = each.value.location

  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

### Example: Reusable module

```hcl
module "application" {
  source = "../../modules/application"

  environment = "dev"
  location    = "centralindia"
}
```

The module must declare the inputs and contain the underlying resources.

### Production caveats

- Use stable `for_each` keys.
- Changing a key changes the resource address.
- Refactoring existing resources requires deliberate address migration, such as appropriate `moved` blocks.
- Review plans for unintended replacement.
- Avoid deeply nested modules that make ownership unclear.

### Interview answer

> “I use modules to standardize reusable infrastructure and for_each for similar instances. I preserve stable resource addresses and review refactoring plans to avoid replacement.”

**References:** [Terraform modules](https://developer.hashicorp.com/terraform/language/modules), [for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each).

---

## 7. What Azure DevOps tools do you use, and where do you store CI output?

### Answer structure

Mention only tools you have used, then explain their purpose.

Possible areas include:

- Source repositories.
- YAML pipelines.
- Test execution.
- Artifact publishing.
- Service connections.
- Variable groups.
- Environment deployment controls.

### Where should CI output go?

| Output | Suitable destination |
|---|---|
| Build files used by later pipeline stages | Pipeline artifacts. |
| Versioned npm, NuGet, Maven, Python, or other supported packages | Azure Artifacts feed. |
| Container image | Container registry. |
| Test results | Pipeline test reporting. |

### Publish a pipeline artifact

```yaml
steps:
- publish: '$(Build.ArtifactStagingDirectory)'
  artifact: application
```

### Download it later

```yaml
steps:
- download: current
  artifact: application
```

### Important distinction

**Pipeline artifacts and Azure Artifacts feeds are not the same thing.**

Pipeline artifacts are associated with pipeline runs. Deleting a run deletes its associated artifacts.

Package feeds provide package-management workflows.

### Interview answer

> “I store deployable build output as pipeline artifacts, versioned reusable packages in an Azure Artifacts feed, and container images in a registry. I promote the same validated output rather than rebuilding it for every environment.”

**References:** [Pipeline artifacts](https://learn.microsoft.com/en-us/azure/devops/pipelines/artifacts/pipeline-artifacts?view=azure-devops), [Artifact types](https://learn.microsoft.com/en-us/azure/devops/pipelines/artifacts/artifacts-overview?view=azure-devops).

---

## 8. What steps do you create in an Azure pipeline?

### Suggested CI/CD stages

1. Checkout.
2. Install approved tool versions.
3. Restore dependencies.
4. Lint and validate.
5. Run tests.
6. Run approved security checks.
7. Build and package.
8. Publish artifacts.
9. Deploy to a test environment.
10. Run smoke tests.
11. Apply production approval controls.
12. Deploy the validated artifact.
13. Verify health and use the approved rollback procedure if necessary.

These are suggested stages, not requirements for every application.

### Example YAML

This example assumes a Node.js repository with working `test` and `build` scripts that creates a `dist` directory.

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

stages:
- stage: Build
  jobs:
  - job: BuildAndTest
    steps:
    - checkout: self

    - task: NodeTool@0
      inputs:
        versionSpec: '22.x'

    - script: npm ci
      displayName: Restore dependencies

    - script: npm test
      displayName: Run tests

    - script: npm run build
      displayName: Build application

    - publish: '$(Build.SourcesDirectory)/dist'
      artifact: application

- stage: DeployTest
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeployApplication
    environment: test
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: application

          - bash: |
              set -euo pipefail
              echo "Artifact downloaded."
              echo "Replace this step with an approved deployment task."
            displayName: Deployment placeholder
```

The deployment step is intentionally a placeholder. This pipeline does not deploy an application until a real deployment task is added.

### Production controls

Configure approvals and checks on protected resources outside YAML.

For Terraform delivery, add:

- Formatting and validation.
- Plan generation.
- Plan review.
- Controlled apply of the reviewed plan.
- State and credential protections.

### Interview answer

> “I separate build, test, packaging, and deployment. I publish a validated artifact, deploy it through controlled environments, and verify application health after deployment.”

**References:** [Multi-stage pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/run-stages?view=azure-devops), [Approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops).

---

## 9. How do you store secrets in Azure DevOps?

### Options

- Secret pipeline variables.
- Secret variables in variable groups.
- Variable groups linked to Azure Key Vault.
- Azure Key Vault tasks.
- Workload identity federation for supported Azure authentication, avoiding a stored client secret.

### Example: Retrieve a Key Vault secret

```yaml
steps:
- task: AzureKeyVault@2
  inputs:
    azureSubscription: approved-service-connection
    KeyVaultName: application-vault
    SecretsFilter: database-password
    RunAsPreJob: false

- bash: |
    set -euo pipefail
    test -n "$DB_PASSWORD"
    echo "Required secret is available."
  env:
    DB_PASSWORD: $(database-password)
```

The example checks availability without printing the secret.

### Requirements

- The service-connection identity needs appropriate secret-read permissions.
- The pipeline agent needs network connectivity to the vault.
- The pipeline must be authorized to use the service connection.

### Recommended safeguards

- Do not commit secrets to YAML.
- Do not echo secrets.
- Do not enable shell tracing around secret handling.
- Avoid passing secrets in command arguments.
- Restrict who can modify pipelines that consume secrets.
- Scope permissions and secret access narrowly.

### Interview traps

- Secret variables are not automatically exposed as script environment variables; map them explicitly.
- Log masking does not mask every substring or transformation.
- A malicious pipeline modification can misuse an otherwise securely stored secret.

### Interview answer

> “I use Key Vault-backed secrets or protected secret variables and explicitly map them into tasks. For Azure authentication, I prefer supported workload identity federation over long-lived client secrets.”

**References:** [Secret variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-secret-variables?view=azure-devops), [Key Vault integration](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/azure-key-vault?view=azure-devops), [Workload identity service connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops).

---

## 10. Have you automated CSR, CER, and PFX certificate activities?

### Understand the files

| File | Meaning |
|---|---|
| Private key | Sensitive key material that must remain protected. |
| CSR | Certificate signing request submitted to a certificate authority. |
| CER | Common certificate filename extension; content may be PEM or DER. |
| PFX / PKCS#12 | Container that can include the certificate, private key, and chain. |

A CSR does not contain the private key.

### Suggested automation workflow

1. Generate a protected key and CSR.
2. Submit the CSR through the approved CA integration.
3. Retrieve the issued certificate.
4. Validate identity, expiry, chain, and key matching.
5. Package as PFX if required.
6. Store and deploy through approved secret-management controls.
7. Verify the deployed certificate.
8. Monitor expiry and renewal.

The CA submission mechanism depends on the actual certificate authority.

### Example: Generate a CSR

This example uses OpenSSL 3.x and prompts for key encryption credentials.

```bash
umask 077

openssl req \
  -new \
  -newkey rsa:3072 \
  -sha256 \
  -keyout application.key \
  -out application.csr \
  -subj "/CN=app.example.com" \
  -addext "subjectAltName=DNS:app.example.com"
```

### Inspect the CSR

```bash
openssl req \
  -in application.csr \
  -noout \
  -text \
  -verify
```

### Package an issued certificate as PFX

Assumptions:

- `application.cer` contains the issued certificate in PEM format.
- `chain.pem` contains the required chain.
- `application.key` is the matching private key.

```bash
openssl pkcs12 \
  -export \
  -inkey application.key \
  -in application.cer \
  -certfile chain.pem \
  -name application \
  -out application.pfx
```

OpenSSL prompts for the necessary passwords.

### Automation caveats

- For unattended execution, use approved secret inputs rather than hardcoded passwords.
- Never publish private keys or PFX files as ordinary build artifacts.
- A non-exportable key cannot be packaged into an exportable PFX.
- The filename extension alone does not identify certificate encoding.
- Requesting a SAN does not guarantee that the CA issues it unchanged.
- Do not substitute a self-signed certificate for an approved production CA certificate.

### Interview answer template

> “I automate request generation, approved CA submission, validation, packaging, deployment, and expiry monitoring. I treat private keys and PFX files as secrets and verify the certificate after deployment.”

**References:** [OpenSSL CSR commands](https://docs.openssl.org/master/man1/openssl-req/), [OpenSSL PKCS#12 commands](https://docs.openssl.org/master/man1/openssl-pkcs12/).

---

## 11. What is the purpose of a Log Analytics workspace?

### Explanation

A Log Analytics workspace stores collected log data in tables.

It supports operational analysis, troubleshooting, auditing, alerts, and integrations with other services.

Log Analytics is the query tool used to explore that data, including through Kusto Query Language, or KQL.

### Suggested use cases

- Investigate application and infrastructure incidents.
- Correlate events across resources.
- Analyze collected VM and container logs.
- Build operational dashboards.
- Create log-based alerts.
- Control log access and retention.

### Example KQL

Assuming the `Heartbeat` table is populated:

```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| order by LastHeartbeat asc
```

This identifies the latest heartbeat observed for each computer appearing in the selected period.

It does not identify machines that produced no records during that period unless you compare against an inventory.

### Cost and access recommendations

- Collect useful logs, not everything by default.
- Select appropriate retention and table settings.
- Restrict access to sensitive log data.
- Avoid logging credentials or unnecessary personal information.

### Interview trap

A Log Analytics workspace and an Azure Monitor workspace are different resource types. Managed Prometheus metrics use an Azure Monitor workspace.

### Interview answer

> “I use a Log Analytics workspace to centralize collected logs, query them with KQL, investigate incidents, and support alerts and dashboards while controlling access and retention.”

**References:** [Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview), [Log Analytics query tool](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview), [Managed Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-metrics-overview).

---

## 12. A database is private. How do you allow access only to specific people?

### Main principle

Use both:

1. **Network reachability controls.**
2. **Identity and database authorization.**

A network rule identifies traffic sources, not individual people.

### Suggested Azure design

- Use supported private database connectivity.
- Provide approved users with an authorized private access path.
- Configure DNS correctly.
- Use individual or group-based authentication where supported.
- Grant only required database privileges.
- Audit access.
- Remove access when no longer needed.

For Azure Database for PostgreSQL, Microsoft Entra authentication supports users and groups.

### Example permission model

| Identity | Suggested database access |
|---|---|
| Application identity | Required application operations. |
| Reporting group | Read access to approved reporting objects. |
| DBA group | Controlled administrative access. |
| Unapproved users | No database access. |

### Important distinctions

- Azure resource-management permission does not automatically grant database data access.
- A VPN connection does not grant database privileges.
- A private endpoint does not authenticate users.
- A shared IP address does not reliably identify an individual.
- Do not use an administrative database identity for routine reporting.

### Interview answer

> “I keep the database private, provide an approved network path, and authorize named users or groups at the database layer. Network access and data permissions are separate controls.”

**References:** [Azure Private Link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview), [PostgreSQL Entra authentication](https://learn.microsoft.com/en-us/azure/postgresql/security/security-entra-configure).

---

## 13. What happens when multiple people execute Terraform commands?

### Scope matters

The main concurrency risk is multiple operations against the **same state** or overlapping resource ownership.

Independent configurations with separate state and ownership can operate separately.

### Possible problems

- Concurrent state writes.
- Conflicting infrastructure changes.
- Stale saved plans.
- Inconsistent configuration versions.
- Two states attempting to manage the same resource.

### State locking

A backend supporting locking prevents simultaneous state-writing operations.

The AzureRM backend supports locking using Azure Blob Storage capabilities.

### Example backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "REPLACE_WITH_STORAGE_ACCOUNT"
    container_name       = "tfstate"
    key                  = "production/application.tfstate"

    use_azuread_auth = true
  }
}
```

The storage resources and authentication must be configured separately.

### Suggested team controls

- Centralize production applies in CI/CD.
- Serialize applies per state.
- Review changes through version control.
- Use consistent tool and provider versions.
- Restrict state access.
- Avoid overlapping resource ownership.

### Interview traps

- Do not use `-lock=false` to bypass another active operation.
- Force-unlock only after confirming the operation has ended and the lock is stale.
- State locking does not coordinate two different state files managing the same resource.
- A saved plan can become invalid after another apply changes the state.

### Interview answer

> “I use a locking remote backend and serialize applies per state through CI/CD. Locking prevents concurrent writers, but I also prevent overlapping resource ownership and stale-plan execution.”

**References:** [Terraform state locking](https://developer.hashicorp.com/terraform/language/state/locking), [AzureRM backend](https://developer.hashicorp.com/terraform/language/backend/azurerm).

---

## 14. Have you used monitoring tools?

### Answer with actual experience

A strong answer explains:

- What you monitored.
- How telemetry was collected.
- What alerts were actionable.
- How monitoring helped resolve an incident.

### Example Azure monitoring stack

| Component | Purpose |
|---|---|
| Azure Monitor | Monitoring and alerting capabilities. |
| Log Analytics | Querying collected logs. |
| Managed Prometheus | Collecting and querying Prometheus metrics. |
| Grafana | Visualizing metrics through dashboards. |

### Suggested monitoring priorities

- Request rate.
- Error rate.
- Latency percentiles.
- Capacity saturation.
- Database connections and performance.
- Pod restarts and readiness.
- Node resource pressure.
- Backup and replication health.

### Example PromQL

Assuming the application exposes this counter with a `status` label:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

This expresses a server-error ratio for the selected metrics. Handle zero traffic and label conventions in the actual alert rule.

### Interview answer template

> “I use metrics to detect symptoms, logs to investigate events, and dashboards to understand trends. I connect alerts to runbooks and prioritize application success and latency rather than CPU alone.”

**References:** [Azure Monitor managed Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-metrics-overview), [Grafana integration](https://learn.microsoft.com/en-us/azure/azure-monitor/metrics/prometheus-grafana).

---

## 15. How do you communicate between Azure subscriptions?

### First identify the requirement

A subscription is a management boundary, not a network connection.

Ask whether the requirement is:

- Private network communication.
- Access to one specific service.
- Resource-management access.

### Connectivity options

| Requirement | Option to evaluate |
|---|---|
| Connect VNets privately | VNet peering. |
| Connect VNets across regions | Global VNet peering. |
| Connect VNets through an IPsec tunnel | VNet-to-VNet VPN Gateway connection. |
| Access a supported specific service privately | Private Link/private endpoint. |
| Manage resources in another subscription | Appropriate identity and RBAC, not network peering alone. |

Azure VNet peering supports connectivity across subscriptions and tenants.

VNet-to-VNet VPN connections can also connect different subscriptions.

### Suggested implementation checks

- Non-overlapping address spaces where required.
- Permissions on both sides.
- Peering or connection configuration.
- Routes and security rules.
- DNS resolution.
- Service authentication.
- Cost and bandwidth requirements.

### Hub-and-spoke consideration

A hub-and-spoke design can centralize connectivity and inspection.

Do not assume that ordinary peering automatically provides transitive spoke-to-spoke routing.

### Interview answer

> “For private VNet connectivity, I evaluate peering or VPN Gateway. For one supported service, I consider Private Link. Cross-subscription resource management requires RBAC separately from network connectivity.”

**References:** [VNet peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview), [Cross-subscription VNet VPN](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-vnet-vnet-rm-ps), [Private Link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview).

---

## 16. What Kubernetes commands do you use for troubleshooting?

### Suggested workflow

Start with scope and symptoms, then inspect workload state, events, logs, resources, networking, and dependencies.

### Confirm context

```bash
kubectl config current-context
kubectl get namespaces
```

### Inspect workloads

```bash
kubectl get pods -n NAMESPACE -o wide
kubectl get deployments -n NAMESPACE
kubectl describe pod POD_NAME -n NAMESPACE
```

### Inspect events

```bash
kubectl get events \
  -n NAMESPACE \
  --sort-by=.metadata.creationTimestamp
```

### Inspect logs

```bash
kubectl logs POD_NAME \
  -n NAMESPACE \
  -c CONTAINER_NAME \
  --tail=200

kubectl logs POD_NAME \
  -n NAMESPACE \
  -c CONTAINER_NAME \
  --previous
```

`--previous` retrieves logs from the previous terminated container instance when available.

### Inspect resource usage

```bash
kubectl top pods -n NAMESPACE
kubectl top nodes
```

These commands require a functioning metrics API.

### Inspect service routing

```bash
kubectl get services -n NAMESPACE

kubectl get endpointslices \
  -n NAMESPACE \
  -l kubernetes.io/service-name=SERVICE_NAME

kubectl get networkpolicies -n NAMESPACE
```

### Inspect storage

```bash
kubectl get pvc -n NAMESPACE
kubectl describe pvc PVC_NAME -n NAMESPACE
```

### Inspect nodes

```bash
kubectl get nodes
kubectl describe node NODE_NAME
```

### Check deployment progress

```bash
kubectl rollout status deployment/DEPLOYMENT_NAME \
  -n NAMESPACE

kubectl rollout history deployment/DEPLOYMENT_NAME \
  -n NAMESPACE
```

### Symptom checklist

| Symptom | Suggested investigation |
|---|---|
| Pending | Scheduling events, requests, taints, affinity, PVC binding. |
| ImagePullBackOff | Image reference, registry access, pull credentials. |
| CrashLoopBackOff | Previous logs, exit reason, configuration, probes. |
| OOMKilled | Memory demand and configured limits. |
| Running but not Ready | Readiness probe and dependencies. |
| Service unreachable | Selectors, ready endpoints, ports, policies, DNS. |

### Production recommendation

Collect evidence before restarting or deleting Pods. Restarting can hide the original failure state.

### Interview answer

> “I check context, Pod state, events, current and previous logs, resource usage, service endpoints, storage, and node conditions. I troubleshoot from the observed failure rather than restarting blindly.”

**Reference:** [Debug running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/).

---

## 17. What is Terraform drift?

### Explanation

Drift occurs when managed infrastructure changes outside the Terraform workflow, so the actual resource no longer matches Terraform's recorded state or intended configuration.

### Example

Terraform configures a resource with:

```hcl
tags = {
  Environment = "prod"
}
```

Someone changes the tag manually.

A subsequent plan refreshes resource information and can reveal the difference.

### Detect drift

```bash
terraform plan
```

For a state-focused review:

```bash
terraform plan -refresh-only
```

### Two possible decisions

#### A. Restore the intended configuration

Review a normal plan and apply the approved correction.

#### B. Accept the external change

Update configuration to reflect the approved change and reconcile state through the reviewed workflow.

### Important distinction

```bash
terraform apply -refresh-only
```

updates state to observed infrastructure without modifying deployed resources.

It does **not** rewrite your `.tf` configuration.

### Suggested prevention

- Restrict manual production changes.
- Run scheduled plans.
- Reconcile emergency changes.
- Use narrow, deliberate `ignore_changes` settings only where another system legitimately owns an attribute.

### Interview answer

> “Drift is an out-of-band change to managed infrastructure. I detect it with plans, decide whether to restore or accept the change, and reconcile configuration and state deliberately.”

**Reference:** [Terraform refresh-only operations](https://developer.hashicorp.com/terraform/tutorials/state/refresh).

---

## 18. Data handled by ADF is dynamic. How do you retrieve, analyze, and present it?

### Clarify the type of change

Dynamic data can mean:

- New records.
- Updated records.
- New files.
- Changing source tables.
- Changing schema.

These require different handling.

### Main approach

Use parameterized, metadata-driven pipelines rather than hardcoding every source.

ADF orchestrates data movement and transformation; it is not the database that holds all source data.

### Suggested architecture

- Source databases or files.
- ADF ingestion and orchestration.
- Raw landing storage.
- Transformation and validation.
- Curated analytical storage.
- Reporting or visualization layer.

### Metadata-driven design

A control table can contain:

- Source object.
- Destination.
- Load type.
- Watermark column.
- Last successful watermark.
- Enabled flag.

ADF reads this configuration and applies it through parameterized pipelines.

### Incremental loading

A watermark can identify records changed between an old and new boundary.

Illustrative SQL:

```sql
SELECT *
FROM orders
WHERE updated_at > :old_watermark
  AND updated_at <= :new_watermark;
```

The parameter syntax is illustrative and must match the actual connector.

### Suggested pipeline sequence

1. Read control metadata.
2. Read the previous successful watermark.
3. Capture the new upper boundary.
4. Copy the selected changes.
5. Validate and merge into the destination.
6. Update the watermark only after successful processing.
7. Refresh or expose the curated reporting data.

### Production caveats

- A simple timestamp watermark does not automatically capture deletions.
- Late-arriving records require a deliberate strategy.
- Schema changes need validation and controlled evolution.
- Retries should not create duplicate results.
- Copy success does not prove business-data correctness.
- CDC or change tracking may be more appropriate than a watermark for some sources.

### Interview answer

> “I use metadata-driven pipelines with parameterized sources and destinations. For changing records, I choose watermark or change-tracking ingestion, validate the results, and present curated data through a reporting layer.”

**References:** [Metadata-driven ADF pipelines](https://learn.microsoft.com/en-us/azure/data-factory/copy-data-tool-metadata-driven), [Incremental loading](https://learn.microsoft.com/en-us/azure/data-factory/tutorial-incremental-copy-overview).

---

## 19. What do you do with an Azure Recovery Services vault?

### Explanation

A Recovery Services vault organizes protection and recovery information for supported workloads.

It is used with Azure Backup and Azure Site Recovery scenarios.

### Common activities

- Configure backup protection and policies.
- Manage recovery points.
- Monitor backup and recovery jobs.
- Configure supported replication scenarios.
- Perform restores or failover operations.
- Control access through RBAC.
- Review backup security settings.

Supported examples include Azure VMs and SQL Server running in Azure VMs.

### Important distinction

A **Recovery Services vault** and a **Backup vault** are different resources.

Backup vaults support certain newer backup workloads, including supported Azure Blob and PostgreSQL scenarios.

Choose the vault type according to the workload's current support documentation.

### Interview traps

- Creating a vault alone does not protect a workload.
- Backup and Site Recovery must be configured.
- A vault is not a replacement for an application recovery runbook.
- Backup and replication solve different recovery needs.

### Interview answer

> “I use a Recovery Services vault to configure and monitor supported backup and Site Recovery protection, manage recovery points, and perform tested restores or failovers. I select the vault type based on workload support.”

**References:** [Recovery Services vault](https://learn.microsoft.com/en-us/azure/backup/backup-azure-recovery-services-vault-overview), [Backup vault](https://learn.microsoft.com/en-us/azure/backup/backup-vault-overview).

---

## 20. How do you take backups, and what strategies do you use?

### Start with recovery requirements

Define:

- RPO.
- RTO.
- Retention.
- Restore granularity.
- Security requirements.
- Regional failure requirements.
- Application consistency.

### Backup methods

| Method | Purpose |
|---|---|
| Full backup | Complete backup of the selected scope. |
| Incremental backup | Changes since the preceding backup in the chain. |
| Differential backup | Changes since the last full backup. |
| Logical database export | Portable database objects and data. |
| Physical database backup with transaction logs | Database recovery, including supported point-in-time recovery. |

Actual service behavior and support vary.

### Suggested Azure strategy

- Use Azure Backup for supported VM and workload protection.
- Use managed-database native recovery capabilities.
- Evaluate supported vaulted backup for additional retention or isolation.
- Keep backup access separate from routine application access.
- Monitor failed jobs.
- Test restoration in an isolated environment.

### PostgreSQL distinction

`pg_dump` creates a logical export.

For PostgreSQL point-in-time recovery, use an appropriate physical/base backup and a continuous sequence of archived WAL.

A logical dump cannot be replayed with WAL as a physical recovery base.

### Example: Logical backup

```bash
pg_dump \
  -h DATABASE_HOST \
  -p 5432 \
  -U backup_user \
  -d application \
  -Fc \
  -f application.dump
```

### Example: Restore test

Restore into an existing, empty test database:

```bash
pg_restore \
  -h RESTORE_TEST_HOST \
  -p 5432 \
  -U restore_user \
  -d application_restore_test \
  --no-owner \
  --no-acl \
  --exit-on-error \
  application.dump
```

Reapply and validate approved permissions separately.

### Suggested restore validation

- Backup is accessible.
- Required keys and credentials are available.
- Restore completes.
- Expected data and objects exist.
- Permissions are correct.
- Application smoke tests pass.
- Measured recovery meets the agreed objectives.

### Production caveats

- A successful backup job is not proof of recoverability.
- Replication can propagate corruption and is not a backup substitute.
- Retention must match policy.
- Deleting old backups can break recovery chains.
- Do not overwrite production during a restore test.
- Backup security must account for compromised production credentials.

### Interview answer

> “I design backups around RPO, RTO, retention, and restore granularity. I use workload-supported backup methods, isolate access, monitor failures, and regularly prove recovery through restore tests.”

**References:** [Azure Backup documentation](https://learn.microsoft.com/en-us/azure/backup/), [PostgreSQL continuous archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html), [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html).

---

# Quick Revision

1. PostgreSQL uses TCP **5432** by default.
2. Use `kubectl exec`, not SSH, for normal container command execution.
3. Pipeline conditions are not deployment security boundaries.
4. Separate Terraform state and identities by ownership and environment.
5. Define RTO and RPO before choosing a DR architecture.
6. Replication is not a backup substitute.
7. Right-size before buying commitments.
8. Use modules and `for_each` to reduce repetition.
9. Pipeline artifacts differ from package feeds.
10. Do not print secrets or publish private keys as normal artifacts.
11. A private endpoint does not authorize database users.
12. State locking protects one state, not overlapping ownership across states.
13. Collect Kubernetes evidence before restarting workloads.
14. Refresh-only updates state, not Terraform configuration.
15. Watermark ingestion does not automatically capture deletions.
16. Creating a vault alone does not enable protection.
17. A backup strategy is incomplete until restoration is tested.
