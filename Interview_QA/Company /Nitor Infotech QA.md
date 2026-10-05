These are model answers for a **six-year DevOps interview**. Adapt the project examples to your actual experience, especially the tools, versions, and implementation details.

**1. Explain the CI/CD setup in your current project**

A good answer should cover the complete path from a commit to a verified production release.

A typical flow is:

| Step | What happens |
|---|---|
| Commit or pull request | A webhook triggers Jenkins |
| Checkout | Jenkins retrieves the specific commit |
| Validation | Linting, unit tests, dependency scanning, and code-quality checks run |
| Build | The application artifact and container image are created |
| Image validation | The image is scanned and checked against release policies |
| Publish | The image is pushed to a registry using an immutable version |
| Dev/QA deployment | Automated integration and smoke tests run |
| Production promotion | The tested artifact is promoted through required approvals |
| Verification | Health, errors, latency, and business transactions are checked |

For example:

> “Jenkins handles CI and publishes the validated image to ECR. The pipeline then proposes an image-version change in our deployment repository. Once approved, Argo CD reconciles that configuration into the target environment.”

**Build once and promote the same artifact across environments.** Environment-specific configuration should be supplied separately.

Keep the `Jenkinsfile` in source control so pipeline changes receive the same review and versioning as application changes. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/?utm_source=chatgpt.com)

**2. Explain GitOps, Argo CD, and Flux**

GitOps means storing the desired deployment configuration in version control and using a controller to reconcile the running environment with that configuration.

A practical workflow is:

1. CI builds, tests, and publishes an image.
2. A pull request updates the image reference in the deployment repository.
3. Reviewers approve the change.
4. The GitOps controller detects the new desired state.
5. It applies changes and reports synchronization and health status.

| Tool | Main approach |
|---|---|
| Argo CD | Application-oriented deployment management with synchronization, health reporting, CLI, and UI |
| Flux | A set of Kubernetes controllers for sources, Kustomize, Helm, notifications, and image automation |

Argo CD compares live resources against the desired configuration and reports differences as `OutOfSync`. Synchronization can be manual or automated. [Declarative GitOps CD for Kubernetes](https://argo-cd.readthedocs.io/en/stable/?utm_source=chatgpt.com)

Flux separates responsibilities across controllers—for example, fetching a source, applying Kustomize configuration, or reconciling a Helm release. [Flux](https://fluxcd.io/flux/concepts/?utm_source=chatgpt.com)

In Argo CD, enabling automated synchronization does **not automatically enable every drift-remediation behavior**. `selfHeal` controls reconciliation of live changes, while `prune` controls deletion of resources removed from Git. [Declarative GitOps CD for Kubernetes](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/?utm_source=chatgpt.com)

For rollback, restore the known-good configuration in Git. Database changes require their own recovery plan.

**3. What are shared libraries in Jenkins?**

Shared libraries provide reusable pipeline code across repositories. They help standardize activities such as builds, scans, image publication, and deployment validation.

Typical structure:

| Location | Purpose |
|---|---|
| `vars/` | Pipeline-facing functions and global variables |
| `src/` | Groovy classes |
| `resources/` | Supporting files loaded by library code |

For example, `vars/buildJava.groovy`:

```groovy
def call() {
    sh './mvnw -B clean verify'
}
```

A Jenkinsfile can invoke it:

```groovy
@Library('platform-library@v1.0.0') _

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                buildJava()
            }
        }
    }
}
```

Configure the library name, SCM repository, credentials, and version in Jenkins library settings.

Pinning a reviewed tag or commit makes releases reproducible. Restrict write access to trusted libraries because their code can have significant Jenkins privileges. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/shared-libraries/?utm_source=chatgpt.com)

**4. How is caching implemented in Jenkins?**

Jenkins does not automatically provide a universal dependency cache. Caching depends on the build tools, agents, and storage configuration.

Common implementations include:

| Cache | Implementation |
|---|---|
| Maven/Gradle dependencies | Persistent agent directories or mounted cache volumes |
| npm packages | Cache the package-download directory; install using the lockfile |
| Container build layers | BuildKit cache, optionally stored in a registry |
| Downloaded dependencies | Nexus or Artifactory proxy repositories |
| Shared-library retrieval | Jenkins library retrieval caching |

Containerized Jenkins agents can mount persistent dependency-cache directories to avoid downloading everything for every build. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/docker/?utm_source=chatgpt.com)

For ephemeral image builders, a registry cache is useful:

```bash
docker buildx build \
  --cache-from type=registry,ref=registry.example.com/web:cache-main \
  --cache-to type=registry,ref=registry.example.com/web:cache-main,mode=max \
  --tag registry.example.com/web:release-123 \
  --push .
```

Use a Buildx builder configured for the registry cache backend. Separate cache references by branch when concurrent branches would otherwise overwrite the same cache. [Docker Docs](https://docs.docker.com/build/cache/backends/?utm_source=chatgpt.com)

Cache keys should account for dependency lockfiles, tool versions, and platform. A cache accelerates the build; correctness must not depend on a cache hit.

**5. What is the current Jenkins version?**

Distinguish **your installed project version** from the latest published release.

As checked on **5 October 2026**, the official changelogs list:

| Release channel | Version | Publication date |
|---|---|---|
| LTS | **2.580.1** | 30 September 2026 |
| Weekly | **2.584** | 28 September 2026 | :chatgpt-content-reference{index="7"}


For your project, report the actual controller version. You can check its system information or, for a WAR installation:

```bash
java -jar jenkins.war --version
```

A strong interview answer also explains how you validate plugin compatibility, agent compatibility, and the required Java version before upgrading.

**6. How have you implemented parallelism in pipelines?**

Run **independent activities** concurrently—for example, unit tests, static analysis, and dependency scanning.

Example using repository-owned scripts and separately allocated agents:

```groovy
pipeline {
    agent none

    stages {
        stage('Parallel checks') {
            failFast true

            parallel {
                stage('Unit tests') {
                    agent { label 'linux' }
                    steps {
                        checkout scm
                        sh './ci/unit-tests.sh'
                    }
                }

                stage('Dependency scan') {
                    agent { label 'linux' }
                    steps {
                        checkout scm
                        sh './ci/dependency-scan.sh'
                    }
                }
            }
        }
    }
}
```

The repository must contain these scripts, and Jenkins needs sufficient matching agent capacity.

- `parallel` runs the branches concurrently.
- `failFast true` aborts the other branches when one fails.
- Separate workspaces avoid concurrent modification of the same build directory.
- Dependent stages still run in order—for example, publishing requires a successful build.

Jenkins also supports matrix execution for combinations such as operating systems or runtime versions. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com)

**7. Explain StatefulSet, DaemonSet, and Deployment**

| Controller | Main purpose | Example |
|---|---|---|
| Deployment | Manage interchangeable replicas and application rollouts | Stateless API |
| StatefulSet | Manage pods with stable identity and persistent-storage associations | Database cluster |
| DaemonSet | Run a pod on every eligible node | Node monitoring or logging agent |

A **Deployment** manages ReplicaSets and supports rolling replacement of application pods. Pod identities are replaceable. [kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com)

A **StatefulSet** provides identities such as `database-0` and `database-1`. If `database-0` is replaced, the replacement retains that ordinal identity. With volume-claim templates, each replica has its own persistent-storage association. StatefulSet does not itself implement database replication or backups. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com)

A **DaemonSet** normally maintains one pod per eligible node. Node selectors, affinity, and tolerations determine eligibility. It is useful for agents that need node-level coverage. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/?utm_source=chatgpt.com)

**8. Explain Kubernetes networking and how you implemented it**

Describe networking in layers:

1. **Pod networking:** The CNI implementation establishes pod connectivity.
2. **Service discovery:** CoreDNS resolves Kubernetes Service names.
3. **Service routing:** Services provide stable access to changing pod endpoints.
4. **External access:** An ingress or gateway implementation connects external traffic to applications.
5. **Network isolation:** NetworkPolicies restrict permitted communication.

Kubernetes expects pod networking to support communication across nodes; the network implementation supplies that connectivity. [Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/networking/?utm_source=chatgpt.com)

For a typical EKS implementation:

- Amazon VPC CNI assigns pod addresses from VPC networking.
- Private worker subnets provide application compute.
- Services expose applications internally.
- AWS Load Balancer Controller provisions the appropriate load-balancer resources.
- NetworkPolicies restrict application communication. [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html?utm_source=chatgpt.com)

**Traffic flow depends on the implementation.** An ALB using IP targets can forward directly to pod IPs; an NGINX ingress design sends traffic through ingress-controller pods.

NetworkPolicies only work when the networking implementation enforces them. For isolated workloads, allow required application paths and DNS explicitly. [Kubernetes](https://kubernetes.io/docs/concepts/services-networking/network-policies/?utm_source=chatgpt.com)

**9. Explain taints, tolerations, and affinity**

| Concept | Purpose |
|---|---|
| Taint | Restrict which pods can use a node |
| Toleration | Allow a pod to tolerate a matching taint |
| Node affinity | Select or prefer nodes based on labels |
| Pod affinity | Place pods near matching pods |
| Pod anti-affinity | Separate pods from matching pods |

For example, reserve nodes for payments workloads:

```bash
kubectl taint node worker-1 dedicated=payments:NoSchedule
kubectl label node worker-1 workload=payments
```

The workload can include:

```yaml
# Under spec.template.spec in a Deployment
tolerations:
  - key: dedicated
    operator: Equal
    value: payments
    effect: NoSchedule

nodeSelector:
  workload: payments
```

The toleration permits use of the tainted node; the selector restricts placement to matching nodes.

Taint effects:

- `NoSchedule`: Blocks new pods without a matching toleration.
- `PreferNoSchedule`: A scheduling preference.
- `NoExecute`: Can also evict existing pods without a matching toleration. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/?utm_source=chatgpt.com)

Affinity supports **required** rules and **preferred** rules. Use required rules carefully: if no node satisfies them, the pod remains Pending. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/?utm_source=chatgpt.com)

**10. How would you reduce pipeline runtime? How do multi-stage builds help?**

First measure the pipeline’s critical path. Distinguish time spent waiting for agents from time spent executing stages.

Then optimize the largest contributors:

- Parallelize independent checks.
- Cache dependencies and container layers.
- Build only affected components where dependency analysis permits.
- Avoid repeated checkout, compilation, or image builds.
- Use appropriately sized agents and local artifact mirrors.
- Reduce unnecessary artifact transfers.
- Promote an existing tested image across environments.

Preserve required quality and security checks; reduce duplicated work and unnecessary waiting.

Jenkins controller-heavy Groovy processing can also affect performance. Run substantial processing in build tools or scripts on agents where appropriate. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/scaling-pipeline/?utm_source=chatgpt.com)

A **multi-stage Docker build** separates compilation from runtime packaging:

1. A builder stage contains compilers and build dependencies.
2. It produces the application artifact.
3. A runtime stage copies only the necessary output.

This usually reduces final image size and therefore upload, download, and deployment time. **It does not automatically make compilation faster**; dependency ordering and cache reuse provide that benefit. [Docker Docs](https://docs.docker.com/build/building/multi-stage/?utm_source=chatgpt.com)

**11. What is SCP?**

In this AWS context, SCP means **Service Control Policy**.

An SCP is an AWS Organizations policy that limits the maximum permissions available to principals in member accounts. It can be attached at organization-root, organizational-unit, or account levels.

For example, an SCP can prevent disabling CloudTrail:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

Key points:

- An SCP **does not grant permissions**.
- IAM or resource policies must still grant access.
- An applicable explicit deny prevents the action.
- SCP restrictions apply to member-account principals, including their root users.
- SCPs do not restrict users or roles in the management account or service-linked roles. [AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html?utm_source=chatgpt.com)

In a Linux context, lowercase `scp` instead refers to a file-copy command.

**12. Explain VPC Endpoint, IGW, Transit Gateway, and Virtual Gateway**

| Component | Purpose | Typical use |
|---|---|---|
| Interface VPC endpoint | Private access to a supported service through endpoint network interfaces | Access Secrets Manager without an internet route |
| Gateway VPC endpoint | Route-table-based access to S3 or DynamoDB | Private EC2 access to S3 |
| Internet Gateway | Connect a VPC to the internet | Internet access for appropriately configured public resources |
| Transit Gateway | Connect multiple networks through a routing hub | Many VPCs and on-premises networks |
| Virtual Private Gateway | AWS-side gateway attached to a VPC | Site-to-Site VPN connectivity to that VPC |

Interface endpoints use AWS PrivateLink. Gateway endpoints for S3 and DynamoDB use route tables and do not use PrivateLink. [Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html?utm_source=chatgpt.com)

An IGW alone does not make an IPv4 EC2 instance internet-accessible: the instance also needs suitable addressing, routes, and security rules. [Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

Transit Gateway provides a hub for attached networks, with route tables controlling connectivity. [Amazon VPC](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html?utm_source=chatgpt.com)

“Virtual Gateway” normally means **Virtual Private Gateway**, an AWS-side VPN termination option. Site-to-Site VPN can alternatively terminate on Transit Gateway. [AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html?utm_source=chatgpt.com)

**13. Provide a user only EC2 start and stop access**

Grant the required operations on the approved instance ARNs.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StartStopApprovedInstance",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "arn:aws:ec2:ap-south-1:123456789012:instance/i-0123456789abcdef0"
    }
  ]
}
```

Replace the example account, region, and instance ID.

This grants start and stop operations for that instance. It does not grant creation, termination, or modification.

For console use, the user may also need selected read-only `Describe` permissions. Add those separately because many describe operations require `"Resource": "*"`. [Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ExamplePolicies_EC2.html?utm_source=chatgpt.com)

Prefer assigning the policy to an appropriate federated role. Review other attached policies: adding a narrow policy does not remove broader permissions already granted elsewhere.

**14. A user has an S3-access role but still cannot access the bucket. Why?**

First verify the identity actually making the request:

```bash
aws sts get-caller-identity
```

Then investigate the failed operation:

| Possible cause | Check |
|---|---|
| Wrong credentials | Whether the intended role was assumed |
| Missing permission | Exact operation: listing, reading, writing, or deleting |
| Wrong resource ARN | Bucket ARN versus object ARN |
| Explicit deny | Bucket policy, SCP, RCP, or endpoint policy |
| Restricted session | Session policy or permissions boundary |
| Cross-account access | Required identity and resource permissions |
| SSE-KMS encryption | Required KMS authorization |
| Policy conditions | Source endpoint, IP, organization, TLS, or other conditions |

For example:

- `s3:ListBucket` applies to `arn:aws:s3:::example-bucket`.
- `s3:GetObject` applies to `arn:aws:s3:::example-bucket/*`.

Having permission to read objects does not automatically grant permission to list them.

S3 authorization can fail for several reasons simultaneously; the error may report only one. Check the exact error and applicable policies before changing access. [Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/troubleshoot-403-errors.html?utm_source=chatgpt.com)

If the request times out rather than returning AccessDenied, investigate DNS, routes, NAT or endpoint connectivity, and network controls.

**15. Explain `terraform refresh`**

`terraform refresh` reads managed remote objects and updates Terraform state to reflect what currently exists.

It:

- Updates state.
- Does not modify the configuration files.
- Does not change the actual infrastructure.
- Does not restore infrastructure to the configuration’s desired state.

The command is **deprecated**. It effectively performs an automatically approved refresh-only apply. Prefer a reviewable workflow:

```bash
terraform plan -refresh-only
terraform apply -refresh-only
``` :chatgpt-content-reference{index="26"}


For example, if someone changes an instance outside Terraform, refresh-only records the observed setting. If configuration still specifies the previous setting, a subsequent normal plan can propose changing it back.

Similarly, accepting a refresh that detects a deleted object does not recreate it. A normal plan/apply handles reconciliation against configuration.

**16. How do you manage Terraform code for multiple environments?**

Use reusable modules with separate environment configurations and states.

Example structure:

| Path | Contents |
|---|---|
| `modules/vpc/` | Shared network implementation |
| `modules/eks/` | Shared cluster implementation |
| `environments/dev/` | Dev root configuration and backend |
| `environments/qa/` | QA root configuration and backend |
| `environments/prod/` | Production root configuration and backend |

Each environment consumes shared modules:

```hcl
module "network" {
  source = "../../modules/vpc"

  environment = var.environment
  cidr_block  = var.vpc_cidr
}
```

Environment inputs vary—for example, CIDRs, node sizes, replica capacity, and tags.

Keep these separate:

- State files.
- Deployment credentials.
- Accounts or subscriptions where required.
- Approval and access rules.
- Backend locations.

CLI workspaces provide separate states within a working directory, but they are not a strong access-control boundary. Terraform documentation advises against using them for deployments requiring separate credentials and access controls. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/workspaces?utm_source=chatgpt.com)

Run formatting, validation, policy checks, and a reviewed plan before applying the saved plan through CI/CD.

**17. Explain state locking in Terraform**

State locking prevents concurrent Terraform operations from writing the same state.

For an S3 backend:

```hcl
terraform {
  backend "s3" {
    bucket       = "example-company-terraform-state"
    key          = "prod/platform/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The bucket must already exist. Enable versioning, restrict access, and use suitable encryption.

Native S3 locking is enabled with `use_lockfile = true`. **DynamoDB-based locking is deprecated.** The runner needs appropriate permissions on both the state object and its `.tflock` object. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/backend/s3?utm_source=chatgpt.com)

A run can wait for a lock:

```bash
terraform apply -lock-timeout=5m tfplan
```

If a run crashes and leaves a stale lock, identify its owner and confirm that the original operation has stopped before using `terraform force-unlock`. Do not routinely bypass locking with `-lock=false`. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/state/locking?utm_source=chatgpt.com)

Locking coordinates Terraform runs; it does not prevent manual console changes. Also, `.terraform.lock.hcl` records provider dependency selections—it is not the state lock.

**18. Create the requested Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - name: http
              containerPort: 80
```

Explanation:

- The Deployment is named `web-app`.
- It requests three replicas.
- Its selector matches the pod-template label `app: web`.
- Each pod runs the requested nginx image and declares container port 80.

Declaring `containerPort` documents the port; the application itself must listen on it.

**19. Write a NodePort Service exposing the application on port 8080**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  labels:
    app: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080
```

Create it in the same namespace as the Deployment.

| Field | Meaning |
|---|---|
| `port: 8080` | Service port inside the cluster |
| `targetPort: 80` | nginx port receiving traffic |
| `nodePort: 30080` | Port exposed through eligible node addresses |

Access examples:

- Inside the cluster: `http://web-app:8080`
- Through a reachable node: `http://NODE_IP:30080`

The default NodePort range is **30000–32767**, so `nodePort: 8080` is normally invalid. An external listener specifically using port 8080 can instead be configured through a suitable load balancer. [Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/?utm_source=chatgpt.com)

Apply and verify:

```bash
kubectl apply -f deployment.yaml -f service.yaml
kubectl rollout status deployment/web-app
kubectl get deployments,pods,services -l app=web
```

**20. Write a script counting processes owned by `ubuntu`**

```bash
#!/usr/bin/env bash
set -euo pipefail

user_name="ubuntu"

if ! user_uid=$(id -u "$user_name" 2>/dev/null); then
  printf 'User does not exist: %s\n' "$user_name" >&2
  exit 1
fi

process_count=$(
  ps -eo euid= |
    awk -v target="$user_uid" '
      $1 == target { count++ }
      END { print count + 0 }
    '
)

printf 'Processes owned by %s: %s\n' \
  "$user_name" "$process_count"
```

Explanation:

- `id -u` obtains Ubuntu’s numeric UID.
- `ps -eo euid=` lists effective user IDs without a header.
- `awk` counts matching processes.
- If the user owns no visible processes, the result is zero.

The effective UID determines the identity used for process access permissions. [Linux manual page](https://man7.org/linux/man-pages/man1/ps.1.html?utm_source=chatgpt.com)

Here, “running under ubuntu” means processes owned by that user, including sleeping processes. If the interviewer specifically means Linux’s running/runnable `R` state, also filter the process-state column.

**21. How have you implemented RBAC in EKS?**

Explain **authentication and authorization separately**.

- **Authentication:** Establish which IAM principal is accessing the cluster.
- **Authorization:** Grant that principal appropriate Kubernetes permissions.

A modern approach uses EKS access entries. The cluster must support them through its authentication mode, such as `API` or `API_AND_CONFIG_MAP`.

For custom RBAC, an administrator can map an IAM role to a Kubernetes group:

```bash
aws eks create-access-entry \
  --region ap-south-1 \
  --cluster-name production-eks \
  --principal-arn arn:aws:iam::123456789012:role/PlatformReadOnly \
  --type STANDARD \
  --kubernetes-groups platform-readers
```

Use the IAM role ARN, not an STS assumed-role session ARN. AWS supports group-based authorization through Kubernetes RBAC for `STANDARD` entries. [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/creating-access-entries.html?utm_source=chatgpt.com)

Then create namespace-scoped permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: platform-pod-readers
  namespace: prod
subjects:
  - kind: Group
    name: platform-readers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This allows the group to inspect pods and retrieve logs in `prod`. It does not grant permission to modify workloads or read Secrets.

A `RoleBinding` grants permissions within its namespace; use cluster-wide bindings only when that scope is required. [Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/?utm_source=chatgpt.com)

Using the intended role/session, verify:

```bash
kubectl auth can-i list pods -n prod
kubectl auth can-i delete pods -n prod
kubectl auth can-i get secrets -n prod
```

Expected results for this RBAC example are `yes`, `no`, and `no`.

EKS access policies are another authorization option, and their permissions combine with RBAC permissions. A narrow RoleBinding does not restrict a broader EKS access-policy grant. Also, `kubectl auth can-i --list` does not display permissions supplied by EKS access policies. [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/access-policies.html?utm_source=chatgpt.com)
