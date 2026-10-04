**1. How Terraform, Azure, Azure DevOps, Docker, and Git fit together**

A typical workflow connects these tools as follows:

| Tool | Responsibility |
|---|---|
| Git/Azure Repos | Version application code, infrastructure code, and pipeline definitions |
| Azure DevOps | Run validation, builds, infrastructure deployment, and application releases |
| Terraform | Provision and manage Azure infrastructure |
| Docker | Package applications into reproducible images |
| Azure | Run applications and provide networking, identity, storage, and monitoring |

For example, a pull request triggers tests and Terraform validation. After approval, the pipeline builds an image, scans it, publishes it to Azure Container Registry, and deploys the approved image to AKS or another hosting service.

Keep infrastructure provisioning and application deployment clearly owned, while sharing authentication, approvals, and audit controls.

**2. How do you optimize a Terraform pipeline taking more than 25 minutes?**

I first measure where the time goes:

- Waiting for an agent.
- Installing Terraform and downloading providers.
- Refreshing resources and generating the plan.
- Running security checks.
- Waiting for approval.
- Creating or updating Azure resources.

The solution depends on the bottleneck.

| Bottleneck | Improvement |
|---|---|
| Repeated downloads | Cache provider packages using the committed dependency lock file |
| Agent queue time | Increase appropriately sized agent capacity |
| Oversized state | Split infrastructure into independently owned deployment units |
| Independent workloads | Run separate root configurations concurrently |
| Slow Azure operations | Inspect resource provisioning and extension failures |
| API throttling | Reduce concurrency and investigate retry behavior |
| Duplicate work | Apply the reviewed saved plan rather than generating another plan |

Terraform’s operation concurrency defaults to **10**. Increasing it can help independent operations, but Azure throttling can make excessive concurrency slower. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/plan?utm_source=chatgpt.com)

A measured experiment might use:

```bash
terraform plan -parallelism=15 -out=tfplan
terraform apply -parallelism=15 tfplan
```

I compare execution time, throttling, and reliability before retaining that setting.

For a large deployment, separate states might cover:

```text
Networking
Shared services
AKS platform
Application infrastructure
```

These are separate Terraform root configurations with clear dependencies and ownership. Concurrent jobs should not modify the same state.

**Interview answer:** “I profile the pipeline first, optimize downloads and agent capacity, then review state boundaries and operation concurrency. Resource provisioning time needs different treatment from pipeline overhead.”

**3. What happens to Terraform state if someone deletes resources from Azure?**

The state does **not immediately change** when someone deletes a resource through the Azure portal or CLI.

At that moment:

- Azure no longer contains the resource.
- Terraform configuration may still declare it.
- Terraform state may still contain its previous identity.

During the next normal plan, Terraform refreshes its view of the managed objects. If the provider confirms that an object is absent and configuration still requires it, the plan normally proposes creating it again.

A plan refresh is generally in memory; completing an apply persists the resulting state changes. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/cloud-get-started/cloud-refresh-only?utm_source=chatgpt.com)

My response would be:

1. Verify the subscription, backend, and workspace.
2. Inspect Azure Activity Log to identify the deletion.
3. Run a fresh plan.
4. Decide whether to restore the resource or update the intended configuration.
5. Review dependencies and data-recovery requirements before applying.

If the deletion was intentional, update the configuration accordingly.

A refresh-only operation can reconcile state without recreating infrastructure:

```bash
terraform plan -refresh-only
terraform apply -refresh-only
```

However, if configuration still declares the resource, a later normal plan can propose recreation. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/state/refresh?utm_source=chatgpt.com)

**Important distinction:** Recreating a database or disk resource does not restore deleted business data. Data recovery requires backups or other recovery mechanisms.

**4. If a pipeline fails because resources already exist, how do you handle RIP—Remove, Import, Plan?**

“RIP” is a troubleshooting shorthand, rather than a Terraform command.

First establish why the resource already exists.

| Situation | Appropriate action |
|---|---|
| Resource exists but this state does not manage it | Import it if this configuration should own it |
| Wrong backend or workspace selected | Correct that selection |
| Resource address changed | Use a `moved` block or an appropriate state move |
| State address references the wrong object | Carefully correct the binding |
| Another state owns the resource | Resolve ownership before importing |
| Resource is intentionally externally managed | Reference it through a data source where appropriate |

**Remove:** `terraform state rm` makes Terraform forget a binding. It leaves the Azure resource in place and can cause Terraform to propose creating another object if configuration still declares it. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/state/rm?utm_source=chatgpt.com)

Therefore, removal is only appropriate when the existing binding is wrong or ownership is intentionally changing.

**Import:** Define the intended resource and import the existing Azure object:

```hcl
resource "azurerm_resource_group" "app" {
  name     = "rg-app"
  location = "Central India"
}
```

```bash
terraform import azurerm_resource_group.app \
  "/subscriptions/<subscription-id>/resourceGroups/rg-app"
```

Alternatively, use a reviewable import block:

```hcl
import {
  to = azurerm_resource_group.app
  id = "/subscriptions/<subscription-id>/resourceGroups/rg-app"
}
```

Import associates the existing object with its Terraform address. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import/single-resource?utm_source=chatgpt.com)

**Plan:**

```bash
terraform plan -out=reconciliation.tfplan
```

Inspect every proposed update or replacement. Adjust the configuration to match the intended existing infrastructure before applying.

I also preserve a recoverable state version before correcting bindings and ensure only one owner manages each Azure resource.

**5. How do you export Azure resources into Terraform code?**

For a large existing environment, I would use **Azure Export for Terraform**, then review and refactor the output.

Example:

```bash
az login

az account set --subscription "<subscription-id>"

mkdir terraform-export
cd terraform-export

aztfexport resource-group rg-app
```

The tool maps discovered Azure resources to Terraform resources and can export configuration together with state. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/terraform/azure-export-for-terraform/export-first-resources?utm_source=chatgpt.com)

For a configuration-only export, use the supported `--hcl-only` mode. Microsoft also documents using that mode when incorporating existing resources into an established Terraform environment. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/terraform/azure-export-for-terraform/export-advanced-scenarios?utm_source=chatgpt.com)

For selected resources, Terraform can generate configuration from import blocks:

```hcl
import {
  to = azurerm_resource_group.app
  id = "/subscriptions/<subscription-id>/resourceGroups/rg-app"
}
```

```bash
terraform init
terraform plan -generate-config-out=generated.tf
```

This generates configuration for imported resources lacking an existing resource block. Review the generated file before applying. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import/generating-configuration?utm_source=chatgpt.com)

After export, I would:

- Review resource coverage and unsupported properties.
- Replace generated names with meaningful addresses.
- Introduce variables and reusable modules.
- Review dependency relationships.
- Configure the correct protected backend.
- Remove secrets from configuration.
- Confirm that the plan contains only intended changes.

An exported configuration is a starting point; it does not automatically capture the organization’s intended architecture.

**6. How do you enforce Azure Policies such as tag or location restrictions using Terraform at scale?**

I manage policy definitions, initiatives, assignments, and exceptions through a dedicated governance configuration.

Assign policies at the appropriate scope:

- **Management group:** Common controls across subscriptions.
- **Subscription:** Environment or business-unit controls.
- **Resource group:** Specific workload controls.

For related requirements, group policies into an **initiative**.

Example assignment using the built-in Allowed Locations policy’s ID:

```hcl
variable "management_group_id" {
  type = string
}

variable "allowed_locations_policy_definition_id" {
  type = string
}

resource "azurerm_management_group_policy_assignment" "locations" {
  name                 = "allowed-locations"
  management_group_id  = var.management_group_id
  policy_definition_id = (
    var.allowed_locations_policy_definition_id
  )

  enforce = true

  parameters = jsonencode({
    listOfAllowedLocations = {
      value = [
        "centralindia",
        "southindia"
      ]
    }
  })
}
```

Terraform supports management-group policy assignments and their enforcement setting. [Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/management_group_policy_assignment?utm_source=chatgpt.com)

For tags, choose the effect according to the requirement:

| Effect | Use |
|---|---|
| Audit | Identify resources that violate requirements |
| Deny | Block applicable noncompliant creation/update requests |
| Modify | Add or correct supported properties such as tags |
| DeployIfNotExists | Deploy required supporting configuration |

For existing resources, `Modify` and `DeployIfNotExists` remediation requires a managed identity with suitable permissions and a remediation task. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources?utm_source=chatgpt.com)

Roll out through a pilot scope, examine compliance, then expand enforcement. Maintain approved exceptions and align Terraform’s desired tags with policy-managed tags.

Also check each policy’s resource-type coverage, including resource groups and global services.

**7. What are good repository and pipeline practices for a large DevOps project?**

Organize by ownership and deployment boundaries.

| Example path | Contents |
|---|---|
| `infra/modules/network/` | Reusable networking module |
| `infra/modules/aks/` | Reusable AKS module |
| `infra/live/dev/platform/` | Development root configuration |
| `infra/live/prod/platform/` | Production root configuration |
| `pipelines/templates/` | Shared pipeline templates |
| `services/api/` | Application code |
| `charts/api/` | Deployment packaging |
| `policy/` | Infrastructure and security checks |
| `docs/` | Architecture and operational runbooks |

Important practices include:

- Separate state and access permissions by environment and ownership.
- Pin Terraform, provider, and module versions.
- Commit `.terraform.lock.hcl`.
- Protect important branches with reviews and validation.
- Use versioned pipeline templates.
- Build application artifacts once and promote the same artifact.
- Generate environment-specific Terraform plans.
- Protect plans because they can contain sensitive information.
- Use production environment approvals and restricted service connections.
- Limit untrusted pull-request workloads’ access to privileged agents.

A monorepo works when teams benefit from coordinated changes. Separate repositories suit independent ownership and access requirements.

Choose based on team boundaries, while keeping common controls reusable.

**8. A pipeline fails only on Tuesdays, with no code changes. How do you debug it?**

I treat the recurring timing as evidence of an external or scheduled dependency.

Compare successful and failed runs:

- Agent hostname, pool, image, and tool versions.
- Exact failure time and timezone.
- Resolved dependencies and container digests.
- Service connection identity.
- Network responses and error codes.
- Concurrent jobs and available resources.

Likely causes include:

| Cause | Evidence to check |
|---|---|
| Scheduled patching | Agent restart and OS update history |
| Backups or maintenance | Storage/network saturation at the failure time |
| Weekly cleanup | Removed caches, credentials, or artifacts |
| Scheduled credential rotation | Authentication changes |
| Dependency updates | Different package/base-image versions |
| Weekly workloads | Agent, database, or API contention |
| Date-dependent logic | Cron expressions and script conditions |

I preserve logs and reproduce the failure on the same agent pool. Then change one variable at a time: agent image, dependency source, execution time, or concurrency.

“No code changes” does not mean “no environment changes.” A mutable container tag or unlocked dependency can change the executed software.

Retries with backoff help genuine transient failures, but recurring failures need a documented root cause and correction.

**9. Logs are incomplete. How do you troubleshoot across AKS, Ingress, application, and infrastructure?**

I trace a specific failed request using its timestamp, hostname, path, and correlation ID.

| Layer | Checks |
|---|---|
| Client/DNS/TLS | Resolution, certificate, connection, HTTP response |
| Gateway/Ingress | Routing rules, backend status, rewrites, WAF decisions |
| Kubernetes Service | Selectors, ports, EndpointSlices, ready endpoints |
| Application | Exceptions, dependency calls, request timings |
| AKS infrastructure | Restarts, OOM kills, node pressure, networking |
| Telemetry pipeline | Collection configuration, filters, ingestion, sampling |

Useful commands:

```bash
kubectl get pods -n app -o wide

kubectl get endpointslices -n app \
  -l kubernetes.io/service-name=api

kubectl describe pod <pod-name> -n app

kubectl logs <pod-name> -n app \
  -c api --previous --timestamps

kubectl get events -n app \
  --sort-by=.metadata.creationTimestamp
```

If application logs are missing, inspect container termination reasons and events. A process killed before logging can still leave evidence in its status and metrics.

For Azure Container insights:

```kusto
ContainerLogV2
| where TimeGenerated > ago(30m)
| where PodNamespace == "app"
| project TimeGenerated, PodName, LogMessage
| order by TimeGenerated asc
```

**Current Azure detail:** Microsoft states that the legacy `ContainerLog` table stopped receiving data after September 30, 2026. Check collection configuration and queries against `ContainerLogV2`. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-logs-schema?utm_source=chatgpt.com)

For ongoing observability, correlate structured logs, metrics, and distributed traces. Also monitor the collectors themselves: missing telemetry can result from filtering, exporter failures, ingestion problems, or excessive sampling.

**10. How do you monitor Azure VM memory and alert above 80%?**

First define the metric: physical memory pressure and Windows committed-memory utilization have different meanings.

Azure currently documents **Available Memory Percentage**. If that metric is populated for the target VM, an available-memory threshold below **20%** represents usage above 80% under the total-minus-available definition. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-compute-virtualmachines-metrics?utm_source=chatgpt.com)

For guest telemetry, configure:

1. Azure Monitor Agent.
2. An associated Data Collection Rule or VM insights configuration.
3. The intended workspace/destination.
4. An alert rule and Action Group.

Guest operating system monitoring requires configured collection rather than relying on host telemetry alone. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/monitor-virtual-machine?utm_source=chatgpt.com)

With VM insights memory data in `InsightsMetrics`, a query can calculate usage:

```kusto
InsightsMetrics
| where TimeGenerated > ago(5m)
| where Origin == "vm.azm.ms"
| where Namespace == "Memory" and Name == "AvailableMB"
| extend TotalMemoryMB =
    todouble(todynamic(Tags)["vm.azm.ms/memorySizeMB"])
| where TotalMemoryMB > 0
| extend UsedMemoryPct =
    100.0 * (1.0 - todouble(Val) / TotalMemoryMB)
| summarize UsedMemoryPct = avg(UsedMemoryPct)
    by Computer, _ResourceId
| where UsedMemoryPct > 80.0
```

Microsoft documents this available-memory counter and total-memory tag for VM insights alerts. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/monitor-virtual-machine-alerts?utm_source=chatgpt.com)

Configure a log alert when matching rows exist, split by VM resource ID. Use an agreed evaluation window—for example, sustained usage over five minutes—and attach the notification/action group.

The query assumes VM insights data. A custom performance-counter DCR may write to `Perf`, requiring a different query. Add separate monitoring for missing telemetry.

**11. Write a multistage Dockerfile for a Node.js app while keeping secrets and unnecessary files out**

This example assumes `npm run build` creates `dist/server.js`.

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-bookworm-slim AS build

WORKDIR /app

COPY package.json package-lock.json ./

RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .

RUN npm run build && npm prune --omit=dev


FROM node:24-bookworm-slim AS runtime

WORKDIR /app

ENV NODE_ENV=production

COPY --from=build --chown=node:node \
    /app/package.json ./package.json

COPY --from=build --chown=node:node \
    /app/node_modules ./node_modules

COPY --from=build --chown=node:node \
    /app/dist ./dist

USER node

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

Example `.dockerignore`:

```text
.git
node_modules
dist
coverage
.env
.env.*
.npmrc
npm-debug.log*
```

Pass a private npm configuration through a BuildKit secret:

```bash
docker build \
  --secret id=npmrc,src=/secure/npmrc \
  -t application:release-001 .
```

Secret mounts expose credentials during the relevant build instruction. Docker recommends them instead of build arguments or environment variables for build secrets. [Docker Docs](https://docs.docker.com/build/building/secrets/?utm_source=chatgpt.com)

The runtime image receives only production dependencies and compiled output. Source files, build tooling, and development dependencies stay in the build stage. [Docker Docs](https://docs.docker.com/guides/nodejs/containerize/?utm_source=chatgpt.com)

For production:

- Pin an approved base-image digest.
- Scan and test the final image.
- Supply application secrets at runtime.
- Review dependency lifecycle scripts.
- Verify native dependencies and required runtime libraries.

Deleting a copied secret in a later layer leaves it in earlier layers. If a secret was published, revoke it and rebuild from clean inputs.

**12. Which tools would you recommend for a hybrid on-premises and Azure setup?**

For the stated stack, I would start with this combination:

| Requirement | Starting choice |
|---|---|
| Source control | Azure Repos |
| CI/CD | Azure Pipelines with agents in the required networks |
| Package feeds | Azure Artifacts |
| Container registry | Azure Container Registry |
| Image vulnerability scanning | Trivy |
| Terraform misconfiguration checks | Checkov |
| Secret storage | Azure Key Vault |
| Monitoring | Azure Monitor, Application Insights, and the existing on-premises monitoring platform |

Azure Pipelines supports self-hosted agents for both Azure DevOps Services and Server. Place agents where they can securely reach the required private services. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops\&utm_source=chatgpt.com)

Azure Artifacts supports package feeds such as npm, NuGet, Maven, Python, and Universal Packages. ACR provides the container registry role. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/artifacts/start-using-azure-artifacts?view=azure-devops\&utm_source=chatgpt.com)

Trivy provides container scanning, while Checkov supports Terraform and other infrastructure formats. [Trivy](https://trivy.dev/latest/docs/target/container_image/?utm_source=chatgpt.com)

Before choosing, assess:

- Connectivity and disconnected-operation requirements.
- Existing enterprise tooling and licenses.
- Artifact availability during network outages.
- Data residency.
- Agent isolation and maintenance.
- Expected storage, scanning, and transfer costs.

Use short-lived credentials where supported, separate privileged deployment agents from untrusted builds, and preserve artifact provenance.

**13. How do you assess Azure DevOps migration readiness and plan the transition?**

Clarify whether the migration is:

- Azure DevOps Server to Services.
- Another platform to Azure DevOps.
- An organization/project restructuring.

Inventory:

- Repositories, branches, tags, LFS objects, and history.
- Pipelines, templates, extensions, and agents.
- Service connections, secrets, and package feeds.
- Work items, test plans, permissions, and policies.
- Custom integrations and process definitions.

Then assess identity mapping, network access, compliance requirements, supported features, and the required downtime.

For **Server to Services**, Microsoft’s Data Migration Tool supports a collection-to-new-organization migration, subject to its requirements and limitations. Validate the supported server version and destination before scheduling the move. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/migrate/migration-overview?view=azure-devops\&utm_source=chatgpt.com)

My transition plan would be:

1. Define what history and assets must be preserved.
2. Back up and validate the source.
3. Resolve compatibility and customization issues.
4. Perform a test migration.
5. Test builds, releases, permissions, and integrations.
6. Agree on a freeze/cutover window.
7. Perform the final migration and business validation.
8. Keep the source available under the agreed retention plan.

Microsoft’s validation checks identify problems such as unsupported process customizations. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/migrate/migration-validate?view=azure-devops\&utm_source=chatgpt.com)

For a Git-only migration, transferring Git history does not automatically transfer pipeline configuration, policies, issues, or secrets.

Define rollback carefully: once users create new work in the destination, returning to the source requires reconciliation.

**14. How do you manage AWS and Azure through one DevOps process, focusing on security and cost?**

Standardize the workflow while keeping cloud-specific implementation explicit.

A shared process can include:

```text
Pull request → validation → security checks → plan/build
→ approval → deployment → verification
```

Shared controls:

- Versioned pipeline templates.
- Immutable application artifacts.
- Common tagging and ownership rules.
- Security and policy checks.
- Production approvals.
- Central audit records and observability standards.

Cloud-specific controls:

| Area | AWS | Azure |
|---|---|---|
| Deployment identity | Scoped IAM roles | Scoped service principals/managed identities |
| Secrets | Secrets Manager or equivalent | Key Vault |
| Governance | Organizational and IAM controls | Management groups, Azure Policy, RBAC |
| Cost visibility | AWS billing/cost tools | Azure Cost Management |

Use short-lived federation where supported. Azure Resource Manager service connections support workload identity federation. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?WT.mc_id=MVP_319025\&view=azure-devops\&utm_source=chatgpt.com)

Keep Terraform states separated by cloud, environment, and ownership. A shared process does not require one enormous cross-cloud state.

For cost, measure idle resources, utilization, storage retention, cross-cloud transfer, and cost per workload. Apply scheduling and right-sizing, then consider commitments for predictable demand.

Cross-cloud deployment is not automatically transactional. Define recovery for partial success, and test data replication and recovery independently.

**15. How would you use Azure DevOps REST APIs to apply a security policy to all repositories?**

First define the control:

- **Branch policies:** Review requirements, build validation, merge rules.
- **Repository permissions:** Force-push rights, policy bypass, administration.

For example, require **two reviewers on every `main` branch in a project** using the Policy Configurations API:

```http
POST https://dev.azure.com/{organization}/{project}/_apis/policy/configurations?api-version=7.1
Authorization: Bearer <short-lived-token>
Content-Type: application/json
```

```json
{
  "isEnabled": true,
  "isBlocking": true,
  "type": {
    "id": "fa4e907d-c16b-4a4c-9dfa-4906e5d171dd"
  },
  "settings": {
    "minimumApproverCount": 2,
    "creatorVoteCounts": false,
    "scope": [
      {
        "repositoryId": null,
        "refName": "refs/heads/main",
        "matchKind": "exact"
      }
    ]
  }
}
```

`repositoryId: null` provides project-wide scope for matching branches. Microsoft documents this approach for policies across all repositories in a project. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/repos/git/repository-settings?view=azure-devops\&utm_source=chatgpt.com)

For organization-wide automation:

1. Enumerate authorized projects.
2. Read existing policy configurations.
3. Match policies by type and scope.
4. Create missing policies.
5. Update changed policies through the configuration ID.
6. Read back and verify the result.
7. Record configuration IDs and outcomes.

This makes the process idempotent instead of creating duplicates each run. The API supports listing, creating, updating, and deleting configurations. [Microsoft Learn](https://learn.microsoft.com/en-us/rest/api/azure/devops/policy/configurations?utm_source=chatgpt.com)

If repositories use different protected branch names, discover and apply the appropriate scopes.

Use a managed identity or service principal with short-lived Entra tokens and the necessary **Azure DevOps permissions**. Azure subscription permissions alone are insufficient. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops\&utm_source=chatgpt.com)

For controls such as force-push permission, use the Git security namespace and ACL APIs, preserving existing entries and inheritance. Branch-policy configuration does not replace permission management. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/organizations/security/namespace-reference?view=azure-devops\&utm_source=chatgpt.com)

**16. What is the difference between Git merge and rebase?**

| Aspect | Merge | Rebase |
|---|---|---|
| Operation | Combines development histories | Replays commits on another base |
| Existing commits | Preserved | Replayed commits receive new identities |
| History | Can retain branch structure | Commonly produces a linear sequence |
| Conflicts | Resolved during the merge | Can require resolution across replayed commits |
| Typical use | Integrating shared branches | Updating a feature branch |

Merge example:

```bash
git switch main
git merge feature
```

Depending on the history, Git may fast-forward or create a merge commit. [git-merge Documentation](https://git-scm.com/docs/git-merge?utm_source=chatgpt.com)

Rebase example:

```bash
git fetch origin
git switch feature
git rebase origin/main
```

Git replays the feature commits on the current main branch.

If conflicts occur:

```bash
# Resolve conflicting files
git add <resolved-files>
git rebase --continue
```

To abandon the rebase:

```bash
git rebase --abort
```

Rebase changes the identities of replayed commits. Coordinate before rewriting history other people use. [git-rebase Documentation](https://git-scm.com/docs/git-rebase?utm_source=chatgpt.com)

**17. Someone force-pushed and lost the main branch’s history. How do you recover it?**

First coordinate a pause in writes and preserve existing clones. Avoid cleanup operations that could remove recoverable objects.

Find the last correct commit through:

- A developer’s local branch.
- Local or remote-tracking reflogs.
- Release tags.
- Other branches containing the commits.
- CI checkout records.
- Repository backups.

Inspect reflogs:

```bash
git reflog show main

git reflog show refs/remotes/origin/main
```

Reflogs record previous reference values in the local repository. Another clone may contain useful history your clone lacks. [git-reflog Documentation](https://git-scm.com/docs/git-reflog?utm_source=chatgpt.com)

After identifying a candidate:

```bash
git show <good-commit-sha>

git branch recovery/main <good-commit-sha>

git log --oneline --decorate recovery/main
```

Validate its content, ancestry, release relationship, and build results before restoring main.

If necessary, inspect locally retained unreachable objects:

```bash
git fsck --full --no-reflogs --unreachable
```

This can identify commits no longer reachable through current references. [git-fsck Documentation](https://git-scm.com/docs/git-fsck?utm_source=chatgpt.com)

Preserve the current remote history too, because it may contain legitimate later work.

Recovery requires retained objects or backups. If every copy has lost and pruned the relevant objects, Git cannot reconstruct their content.

**18. How do you push the recovered branch back to the remote?**

Publish the recovery branch first so others can inspect it:

```bash
git push origin recovery/main:refs/heads/recovery-main
```

If the repair can be delivered through ordinary commits, use the normal reviewed workflow.

Restoring the exact previous main reference after rewritten history may require an authorized non-fast-forward update. Capture the remote’s current value and use an explicit lease:

```bash
set -euo pipefail

expected_sha=$(
  git ls-remote --heads origin refs/heads/main |
    awk '{print $1}'
)

git push \
  --force-with-lease="refs/heads/main:$expected_sha" \
  origin recovery/main:refs/heads/main
```

The explicit lease makes Git reject the push if main has changed since the expected value was captured. An empty expected value requires that the remote reference does not exist. [git-push Documentation](https://git-scm.com/docs/git-push?utm_source=chatgpt.com)

Before replacing main, review and preserve any legitimate commits currently on it. Branch protection may require an authorized administrator to permit the specific restoration.

Afterward:

- Verify the remote commit and build.
- Restore the intended protections.
- Review force-push and bypass permissions.
- Guide collaborators to synchronize their branches while preserving local work.
