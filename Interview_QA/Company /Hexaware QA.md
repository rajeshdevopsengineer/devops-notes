Hexaware
Below are detailed answers using **Azure DevOps, Azure, Kubernetes, Helm, and Terraform**. I’ve numbered the duplicated question 13 separately, giving 15 answers.

**1. Write a YAML pipeline for CI/CD—overall structure**

An Azure DevOps pipeline generally contains:

| Element | Purpose |
|---|---|
| `trigger` | Determines which repository changes start the pipeline |
| `pool` | Selects the agent that executes jobs |
| `variables` | Stores reusable configuration |
| `stages` | Separates activities such as build and deployment |
| `jobs` | Groups work executed by an agent |
| `steps` | Runs scripts or predefined tasks |
| `deployment` | Defines a deployment job associated with an environment |

Stages normally run sequentially, with `dependsOn` making their dependencies explicit. Deployment jobs can record deployment history against an Azure DevOps environment. :chatgpt-content-reference{index="0"}

**Example: build a .NET application once and deploy the same package to dev and production**

Assumptions:

- The API project is under `src/Orders.Api/`.
- Test projects are under `tests/`.
- The App Services, service connections, and Azure DevOps environments already exist.
- Production approval is configured on `orders-prod`, as explained in question 8.

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: Release

stages:
  - stage: Build
    displayName: Build, test, and package

    jobs:
      - job: BuildApplication

        steps:
          - checkout: self

          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: '8.0.x'

          - task: DotNetCoreCLI@2
            displayName: Run tests
            inputs:
              command: test
              projects: 'tests/**/*.csproj'
              arguments: '--configuration $(buildConfiguration)'

          - task: DotNetCoreCLI@2
            displayName: Publish application
            inputs:
              command: publish
              publishWebProjects: false
              projects: 'src/Orders.Api/Orders.Api.csproj'
              arguments: >
                --configuration $(buildConfiguration)
                --output $(Build.ArtifactStagingDirectory)/app
              zipAfterPublish: true
              modifyOutputPath: false

          - publish: $(Build.ArtifactStagingDirectory)
            artifact: drop

  - stage: Dev
    displayName: Deploy to development
    dependsOn: Build
    condition: succeeded()

    jobs:
      - deployment: DeployDev
        environment: orders-dev

        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: azure-dev-wif
                    appType: webAppLinux
                    appName: orders-api-dev
                    package: $(Pipeline.Workspace)/drop/app.zip

  - stage: Prod
    displayName: Deploy to production
    dependsOn: Dev
    condition: succeeded()

    jobs:
      - deployment: DeployProd
        environment: orders-prod

        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: azure-prod-wif
                    appType: webAppLinux
                    appName: orders-api-prod
                    package: $(Pipeline.Workspace)/drop/app.zip
```

`DotNetCoreCLI@2` can publish and zip application output, while `AzureWebApp@1` deploys the resulting package to App Service. Match the SDK and App Service runtime to your application. :chatgpt-content-reference{index="1"}

**What to explain in the interview:**

- CI builds, tests, analyzes, and publishes an artifact.
- CD downloads that artifact and deploys it.
- Dev and production consume the same package.
- Environment-specific configuration and secrets are supplied separately.
- Add QA/staging using the same deployment-job pattern.
- Add quality/security gates before promotion and application health checks after deployment.

For Azure Repos Git, configure pull-request build validation through branch policies; YAML `pr` triggers are not supported for that repository provider. :chatgpt-content-reference{index="2"}

---

**2. What is the Terraform command for automatic approval?**

```bash
terraform apply -auto-approve
```

This skips Terraform’s interactive confirmation prompt.

Example with environment variables supplied through a file:

```bash
terraform apply \
  -var-file=dev.tfvars \
  -auto-approve
```

**A useful production workflow is:**

```bash
terraform plan \
  -var-file=prod.tfvars \
  -out=tfplan

# Review and approve the saved plan.

terraform apply tfplan
```

Applying a saved plan already proceeds without an interactive confirmation, so `-auto-approve` is unnecessary in that case. :chatgpt-content-reference{index="3"}

**Interview distinction:** `-auto-approve` skips Terraform’s prompt. It does not bypass Azure DevOps approvals, cloud permissions, or state locking.

---

**3. What is `deployment.yml`?**

In this Kubernetes context, `deployment.yml` is a file containing a **Deployment manifest**.

The filename is a convention. Kubernetes identifies the resource through:

```yaml
kind: Deployment
```

A Deployment declares the desired application configuration, including its image, replica count, update strategy, and pod template.

**Example:**

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: orders
  namespace: dev

spec:
  replicas: 3

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: orders

  template:
    metadata:
      labels:
        app: orders

    spec:
      containers:
        - name: orders
          image: registry.example.com/orders:1.0.0

          ports:
            - name: http
              containerPort: 8080

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"

          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 5
```

This assumes the application provides `/healthz`, the namespace exists, and the illustrative image is replaced with your published image.

**Important fields:**

- `replicas`: Desired number of pods.
- `selector`: Identifies the pods managed by the Deployment.
- `template`: Defines the pods to create.
- `image`: Application container image.
- `strategy`: Controls replacement during an update.

The selector must match the pod-template labels. Updating the pod template triggers a rollout managed through ReplicaSets. :chatgpt-content-reference{index="4"}

Commands:

```bash
kubectl apply -f deployment.yml

kubectl -n dev rollout status deployment/orders

kubectl -n dev get pods
```

For production releases, pin the approved image digest and ensure sufficient capacity and graceful application shutdown.

---

**4. What is `service.yml`?**

`service.yml` usually contains a Kubernetes **Service manifest**.

A Service provides a stable access point for workloads whose pod IP addresses can change.

**Example matching the previous Deployment:**

```yaml
apiVersion: v1
kind: Service

metadata:
  name: orders
  namespace: dev

spec:
  type: ClusterIP

  selector:
    app: orders

  ports:
    - name: http
      port: 80
      targetPort: http
```

Here:

- `selector.app: orders` selects the application pods.
- `port: 80` is the Service port.
- `targetPort: http` resolves to the named container port, `8080`.
- `ClusterIP` provides internal cluster access.

The application must actually listen on its configured port; declaring `containerPort` does not start a listener.

| Service type | Purpose |
|---|---|
| `ClusterIP` | Internal cluster access |
| `NodePort` | Exposes a port on cluster nodes |
| `LoadBalancer` | Requests an external load-balancer integration |
| `ExternalName` | Provides a DNS alias |

With the default cluster domain, clients in the cluster can use:

```text
http://orders.dev.svc.cluster.local
```

Service selectors and EndpointSlices allow traffic routing to track changing backend pods. :chatgpt-content-reference{index="5"}

```bash
kubectl apply -f service.yml
kubectl -n dev get services
kubectl -n dev get endpointslices
```

---

**5. What is a ReplicaSet?**

A ReplicaSet maintains a specified number of matching pods.

If the desired count is three:

- When a pod disappears, it creates a replacement.
- When the desired count increases, it creates additional pods.
- When the desired count decreases, it removes excess pods.

The usual ownership relationship is:

**Deployment → ReplicaSet → Pods**

During a Deployment update, Kubernetes creates a new ReplicaSet and gradually adjusts the old and new replica counts according to the update strategy.

You normally manage application replicas through a Deployment. Editing a standalone ReplicaSet’s pod template does not orchestrate a rollout of all existing pods. :chatgpt-content-reference{index="6"}

Commands:

```bash
kubectl -n dev get deployments
kubectl -n dev get replicasets
kubectl -n dev get pods
```

**Interview answer:**

> “A ReplicaSet maintains the required number of pods. A Deployment adds release-management behavior, including controlled updates and rollback, by managing ReplicaSets.”

Container restarts inside an existing pod are a separate responsibility handled on the node.

---

**6. What is the output of a Helm chart?**

A Helm chart combines **templates and values** to produce Kubernetes manifests.

The exact output depends on the command:

| Command | Result |
|---|---|
| `helm template` | Renders Kubernetes YAML locally |
| `helm install` | Creates a Helm release and submits resources to Kubernetes |
| `helm get manifest` | Displays the stored manifests for an installed release |
| `helm package` | Creates a versioned `.tgz` chart archive |

For example, a template may contain:

```yaml
replicas: {{ .Values.replicaCount }}
```

If `values.yaml` contains:

```yaml
replicaCount: 3
```

the rendered output contains:

```yaml
replicas: 3
```

Render a chart:

```bash
helm template orders ./orders -f values-dev.yaml
```

Rendering alone does not deploy the resources and does not perform all server-side Kubernetes validation. :chatgpt-content-reference{index="7"}

Install it:

```bash
helm install orders ./orders \
  --namespace dev \
  --create-namespace \
  -f values-dev.yaml
```

Installation also produces release information and any chart notes. :chatgpt-content-reference{index="8"}

Package it:

```bash
helm package ./orders
```

For a chart named `orders` with chart version `0.1.0`, the package is `orders-0.1.0.tgz`. Container images are built separately. :chatgpt-content-reference{index="9"}

---

**7. What is Storage Explorer?**

**Microsoft Azure Storage Explorer** is a desktop application for working with Azure Storage data. It is available for Windows, macOS, and Linux.

Typical uses include:

- Browsing storage accounts and containers.
- Uploading and downloading blobs.
- Working with file shares, queues, and tables.
- Inspecting object properties and metadata.
- Connecting through an Azure identity or supported connection credentials.
- Investigating storage-access issues. :chatgpt-content-reference{index="10"}

**Example use case:**

> “After an application uploads a report to Blob Storage, I can use Storage Explorer to verify its location, size, metadata, and contents.”

**Access troubleshooting:**

Check both authorization and connectivity:

- Does the identity have the required data access?
- Is the account firewall allowing the connection?
- If using a private endpoint, can the client reach it and resolve the correct private DNS address?

Storage Explorer is the client application; the underlying data service remains Azure Storage.

---

**8. How do you set approval in a CD pipeline?**

For an Azure DevOps **YAML pipeline**, a common approach is to configure an approval check on the production environment.

**Steps:**

1. Open **Pipelines → Environments**.
2. Select or create the production environment.
3. Open **Approvals and checks**.
4. Add an approval check.
5. Select the approvers.
6. Configure self-approval restrictions, instructions, and timeout.
7. Reference that environment from the deployment job.

```yaml
jobs:
  - deployment: DeployProduction
    environment: orders-prod

    strategy:
      runOnce:
        deploy:
          steps:
            - script: echo "Production deployment steps run here"
```

When the stage attempts to use the protected environment, Azure DevOps evaluates its checks before allowing execution.

**The approval configuration is managed outside the YAML.** Resource owners control these checks. Production service connections and their permissions should also be protected. :chatgpt-content-reference{index="11"}

**Other cases:**

- In a classic release pipeline, configure the stage’s pre-deployment approvals.
- To pause within a YAML workflow for manual validation, use `ManualValidation@1` in an agentless job with `pool: server`. This is distinct from an environment approval check. :chatgpt-content-reference{index="12"}

---

**9. What branching strategies do you use?**

Explain the strategy that matches your project’s release frequency and support requirements.

| Strategy | Typical use |
|---|---|
| Trunk-based development | Frequent integration and delivery |
| Gitflow | Planned releases with development, release, and hotfix branches |
| Release branches | Maintaining supported versions while development continues |

**Example answer for frequent delivery:**

> “We use short-lived feature branches and pull requests into a protected main branch. Automated validation runs before merging. Successful builds produce artifacts that are promoted through environments.”

Typical flow:

1. Create a feature branch.
2. Make a small, focused change.
3. Open a pull request.
4. Run tests and analysis.
5. Review and merge.
6. Build and promote the approved artifact.

**Gitflow alternative:**

- `main`: Production release history.
- `develop`: Integration branch.
- `feature/*`: Feature development.
- `release/*`: Release stabilization.
- `hotfix/*`: Urgent production corrections.

Release and hotfix changes must be incorporated into ongoing development so fixes are not lost. Gitflow’s original author recommends simpler workflows for many continuous-delivery teams. :chatgpt-content-reference{index="13"}

Permanent branches for every deployment environment are not required; artifact promotion can handle environment progression.

---

**10. Have you written automation scripts for daily tasks?**

Use examples you have actually implemented. Common DevOps automation includes:

- Application health checks.
- Backup verification.
- Certificate-expiry checks.
- Cloud inventory reports.
- Deployment validation.
- Log processing.
- Cleanup governed by retention rules.
- Repetitive environment configuration.

Explain the problem, inputs, error handling, scheduling, and result.

**Example: check several application health endpoints**

```bash
#!/usr/bin/env bash
set -euo pipefail

if (( $# == 0 )); then
  printf 'Usage: %s URL [URL...]\n' "$0" >&2
  exit 2
fi

failed=0

for url in "$@"; do
  if status=$(curl \
      --silent \
      --show-error \
      --output /dev/null \
      --write-out '%{http_code}' \
      --connect-timeout 5 \
      --max-time 15 \
      "$url"); then

    case "$status" in
      2[0-9][0-9])
        printf 'OK   %s HTTP %s\n' "$url" "$status"
        ;;
      *)
        printf 'FAIL %s HTTP %s\n' "$url" "$status"
        failed=1
        ;;
    esac

  else
    printf 'FAIL %s connection or request error\n' "$url"
    failed=1
  fi
done

exit "$failed"
```

Example invocation:

```bash
bash check-health.sh \
  https://orders-dev.example.com/healthz \
  https://orders-stage.example.com/healthz
```

The script:

- Accepts endpoints as arguments.
- Applies connection and overall timeouts.
- Treats non-2xx responses as failures.
- Continues checking other endpoints.
- Returns a nonzero status if any check fails.

The curl options separately expose HTTP status and request failures. :chatgpt-content-reference{index="14"}

A pipeline can use the exit status to fail post-deployment validation. This checks the health endpoint’s contract; business transactions need additional tests.

---

**11. How do you move code from one environment to another?**

In a controlled delivery process, you **promote a versioned artifact**.

For a .NET application, that might be a ZIP package. For Kubernetes, it is commonly a container image identified by digest.

**Example flow:**

1. Merge an approved change.
2. Build and test it.
3. Publish an artifact linked to the Git commit and pipeline run.
4. Deploy that artifact to dev.
5. Validate it in QA or staging.
6. Approve production promotion.
7. Deploy the same artifact to production.
8. Record the release and verify health.

| Environment | Artifact | Environment-specific settings |
|---|---|---|
| Dev | Build 1042, `app.zip` | Dev endpoints and credentials |
| QA | Build 1042, `app.zip` | QA endpoints and credentials |
| Prod | Build 1042, `app.zip` | Production endpoints and credentials |

Configuration can come from variable groups, App Service settings, ConfigMaps, and secret stores.

**Interview answer:**

> “We build once and promote the same tested artifact. Configuration changes by environment, while the application version remains fixed.”

Deployment environments provide a history of which pipeline runs deployed to them. :chatgpt-content-reference{index="15"}

If a release fails, redeploy the previous known-good artifact and compatible configuration. A source-code revert alone does not immediately change the running application.

---

**12. What is a variable group in Azure DevOps?**

A variable group is a centrally managed collection of values that authorized pipelines can reuse.

Examples include:

- Application names.
- Resource-group names.
- Environment identifiers.
- Deployment configuration.
- Secret values or linked Key Vault secrets.

Create one under:

**Pipelines → Library → Variable groups**

Example group: `orders-dev`

| Variable | Example value |
|---|---|
| `appName` | `orders-api-dev` |
| `resourceGroup` | `rg-orders-dev` |
| `environmentName` | `dev` |
| `deploymentToken` | Secret |

Reference it in YAML:

```yaml
variables:
  - group: orders-dev

steps:
  - script: echo "Deploying $(appName)"

  - bash: |
      set +x
      ./ci/deploy.sh
    env:
      DEPLOYMENT_TOKEN: $(deploymentToken)
```

**Important points:**

- Authorize the intended pipeline to use the group.
- Mark sensitive values as secrets.
- Map secrets explicitly into script environment variables.
- Do not print secrets.
- Restrict group editing and access.
- Keep development and production settings appropriately separated. :chatgpt-content-reference{index="16"}

A Key Vault-linked variable group stores mappings to selected secret names and retrieves their values during pipeline execution. Changes to which secrets are included require updating the group’s mappings. :chatgpt-content-reference{index="17"}

---

**13. What is SonarQube, and why is it used?**

SonarQube analyzes source code to identify quality and supported security issues and report code metrics.

Typical findings include:

- Reliability problems.
- Maintainability issues, traditionally called code smells.
- Security vulnerabilities supported by its analysis.
- Duplicated code.
- Test-coverage metrics.
- Issues introduced in new code.

Its purpose is to make code-quality expectations measurable and enforce them during delivery.

**Two concepts matter:**

| Concept | Meaning |
|---|---|
| Quality profile | The analysis rules enabled for a language |
| Quality gate | The conditions used to determine whether results are acceptable |

For example, a team may require acceptable new-code coverage, no prohibited security findings, and duplication below its agreed threshold. Quality gates can evaluate new code or overall code, depending on the analysis context. :chatgpt-content-reference{index="18"}

**Typical Azure DevOps integration:**

1. Prepare analysis configuration.
2. Build the application and run tests.
3. Generate coverage reports.
4. Run analysis.
5. Evaluate the quality gate.
6. Permit deployment only when the required checks pass.

SonarQube imports coverage from testing tools; it does not create test coverage by running tests itself. :chatgpt-content-reference{index="19"}

The Azure integration includes Prepare, Analyze, and Publish tasks. Publishing the quality-gate result displays it in the build summary. :chatgpt-content-reference{index="20"}

To enforce failure, configure an actual pipeline gate. One supported approach is:

```properties
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300
```

This makes analysis wait for the gate and fail when the gate fails. :chatgpt-content-reference{index="21"}

---

**14. Write sample Terraform code—overall skeleton**

A typical Terraform root configuration contains:

| File | Purpose |
|---|---|
| `versions.tf` | Terraform and provider requirements |
| `providers.tf` | Provider configuration |
| `main.tf` | Resources or module calls |
| `variables.tf` | Input declarations |
| `outputs.tf` | Values exposed by the configuration |
| `dev.tfvars` | Non-secret environment-specific inputs |
| `backend.tf` | Remote state configuration |
| `.terraform.lock.hcl` | Provider selections and checksums |

Terraform loads the `.tf` files in a directory together; the filenames organize the project rather than define execution order.

**Example: Azure resource group and storage account**

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.10.0, < 2.0.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.4"
    }
  }
}
```

`providers.tf`:

```hcl
provider "azurerm" {
  features {}
}
```

Supply Azure authentication and the target subscription through your approved CLI or pipeline identity configuration. The provider supports subscription selection through configuration, environment settings, or the Azure CLI. :chatgpt-content-reference{index="22"}

`variables.tf`:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "location" {
  type    = string
  default = "Central India"
}

variable "storage_account_name" {
  type        = string
  description = "Globally unique lowercase storage account name"
}
```

`main.tf`:

```hcl
locals {
  prefix = "orders-${var.environment}"
}

resource "azurerm_resource_group" "app" {
  name     = "rg-${local.prefix}"
  location = var.location

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

resource "azurerm_storage_account" "app" {
  name                     = var.storage_account_name
  resource_group_name      = azurerm_resource_group.app.name
  location                 = azurerm_resource_group.app.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}
```

The reference to the resource group creates an implicit dependency, so Terraform knows the group must exist before creating the storage account. Storage account names must be globally unique and use allowed lowercase alphanumeric characters. :chatgpt-content-reference{index="23"}

`outputs.tf`:

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.app.name
}

output "storage_account_id" {
  value = azurerm_storage_account.app.id
}
```

`dev.tfvars`:

```hcl
environment          = "dev"
location             = "Central India"
storage_account_name = "stordersdev123456"
```

Replace the illustrative storage name with an available one.

**Remote backend for team use**

`backend.tf`:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "stcompanytfstate"
    container_name       = "tfstate"
    key                  = "orders/dev.tfstate"

    use_azuread_auth = true
  }
}
```

The state storage must already exist. Configure the backend’s authentication separately as needed; federated CI authentication can use its OIDC support. Azure Blob Storage provides native state-locking capabilities. :chatgpt-content-reference{index="24"}

Run:

```bash
terraform init
terraform fmt -check
terraform validate

terraform plan \
  -var-file=dev.tfvars \
  -out=tfplan

terraform apply tfplan
```

Keep production state and permissions isolated, and protect state and plan files because they can contain sensitive resource data.

---

**15. What files are available inside a Helm chart?**

A typical application chart contains:

| File or directory | Purpose |
|---|---|
| `Chart.yaml` | Chart name, version, metadata, and dependency declarations |
| `values.yaml` | Default configurable values |
| `templates/` | Kubernetes resource templates |
| `templates/deployment.yaml` | Deployment template |
| `templates/service.yaml` | Service template |
| `templates/ingress.yaml` | Optional Ingress template |
| `templates/_helpers.tpl` | Reusable named template definitions |
| `templates/NOTES.txt` | Information displayed to users after installation |
| `templates/tests/` | Optional Helm test resources |
| `charts/` | Dependency charts |
| `Chart.lock` | Locked dependency versions |
| `values.schema.json` | Optional validation schema for values |
| `crds/` | CustomResourceDefinition files |
| `.helmignore` | Files excluded from chart packaging |
| `README.md` | Usage documentation |

Not every chart contains every optional file. :chatgpt-content-reference{index="25"}

Create a starter chart:

```bash
helm create orders
```

Inspect and validate it:

```bash
helm lint ./orders

helm template orders ./orders
```

Package it:

```bash
helm package ./orders
```

The starter chart provides templates and defaults that you customize for the application. :chatgpt-content-reference{index="26"}

**Important version distinction:**

```yaml
apiVersion: v2
name: orders
version: 0.1.0
appVersion: "1.0.0"
```

- `version` identifies the chart package.
- `appVersion` describes the application version.
- The container image version comes from the chart’s template logic; changing `appVersion` only changes the image if the template uses it.
