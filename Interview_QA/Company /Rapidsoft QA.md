# DevOps, Cloud, and SRE Interview Questions

## Rapidsoft | Experience: 10.5 Years Overall, 4–5 Years in DevOps

This guide includes interview-ready explanations, troubleshooting workflows, configuration examples, and production considerations.

> **Important:** The experience figures are from the interview description, not verified personal history. Examples are illustrative and have not been executed against a live environment. Operational workflows marked as recommendations must be adapted to your application and change-management process.

---

## 1. What have you worked on? Explain your experience briefly.

### How to structure the answer

My recommended structure is:

1. Current role and actual experience.
2. Infrastructure and applications supported.
3. Tools used and responsibilities owned.
4. One concrete project or incident.
5. Verified outcome.

Avoid listing tools without explaining what you did with them.

### Sample answer template

> “I currently work as a Senior DevOps Engineer. I have [actual total experience] years of overall experience, including [actual DevOps experience] years focused on infrastructure automation, delivery pipelines, and production reliability.
>
> My responsibilities include [choose your actual responsibilities: Terraform infrastructure, Jenkins pipelines, container deployments, Kubernetes operations, configuration management, monitoring, and incident response].
>
> In a recent project, I worked on [specific project]. My contribution was [specific ownership], including [implementation details].
>
> The main challenge was [real technical challenge]. I addressed it through [actions you actually performed], and the outcome was [verified result].
>
> My focus is making deployments repeatable, infrastructure changes controlled, and production issues easier to diagnose.”

### Example project topics

Use only those that match your experience:

- Migrating manual infrastructure to Terraform.
- Standardizing Jenkins pipelines.
- Containerizing an application.
- Improving Kubernetes deployment health checks.
- Implementing monitoring and operational runbooks.
- Reducing deployment risk through reviewed plans and staged releases.

### Senior-level follow-up preparation

Be ready to explain:

- What you personally owned.
- Why you selected the approach.
- What alternatives you considered.
- How you handled secrets and permissions.
- What failed and how you diagnosed it.
- How you measured the result.

> **Interview tip:** Do not invent percentages, availability figures, project scale, or production ownership.

---

## 2. Terraform has errors while provisioning infrastructure. How do you investigate and validate the files?

### Understand the failure stage

Terraform problems can involve configuration language, state, Terraform core, or provider behavior. Start by identifying which command failed and reading the resource address, file location, and error details. [Official troubleshooting guide](https://developer.hashicorp.com/terraform/tutorials/configuration-language/troubleshooting-workflow).

### Validation versus provisioning

| Command | Purpose |
|---|---|
| `terraform fmt` | Formats configuration consistently. |
| `terraform init` | Initializes the working directory and installs referenced dependencies. |
| `terraform validate` | Checks syntax and internal configuration consistency. |
| `terraform plan` | Evaluates the proposed changes for a particular run. |
| `terraform apply` | Performs the planned infrastructure changes. |

`terraform validate` does not validate remote services, provider APIs, or remote state. A valid configuration is not proof that cloud provisioning will succeed. [Validate command reference](https://developer.hashicorp.com/terraform/cli/commands/validate).

### Static validation example

```bash
terraform version

terraform fmt -check -recursive

# Initialize dependencies without accessing the configured backend.
terraform init -backend=false

terraform validate
```

Run these in the intended root-module directory.

For a real environment plan, initialize the approved backend normally:

```bash
terraform init

terraform plan \
  -var-file=environments/dev.tfvars \
  -out=tfplan
```

The plan evaluates configuration in the context of the selected environment and input values. [Validate command reference](https://developer.hashicorp.com/terraform/cli/commands/validate).

### My recommended investigation checklist

| Symptom | Suggested checks |
|---|---|
| Unsupported argument | Installed provider version and resource schema. |
| Invalid reference | Resource/module names and declared outputs. |
| Authentication failure | Effective identity, expired credentials, account and region. |
| Access denied | Required permissions and applicable policy restrictions. |
| Resource already exists | Existing infrastructure ownership and import requirements. |
| Quota or capacity failure | Relevant service limits and deployment location. |
| State lock error | Whether another operation is genuinely running. |
| Timeout | Provider error details, service status, dependencies, and connectivity. |

### Example input validation

```hcl
variable "environment" {
  type = string

  validation {
    condition = contains(
      ["dev", "staging", "prod"],
      var.environment
    )

    error_message = "Environment must be dev, staging, or prod."
  }
}
```

Input validation documents and enforces allowed values. [Configuration validation](https://developer.hashicorp.com/terraform/language/validate).

### After a failed apply

My recommendation:

1. Preserve the error details.
2. Inspect what actually exists.
3. Review Terraform state and generate a fresh plan.
4. Correct the underlying problem.
5. Review the recovery plan before applying.

Do not assume a failed apply rolled back every successful operation.

Avoid deleting state, disabling locking, or force-unlocking without establishing the cause.

### Interview-ready answer

> “I identify whether the failure is in initialization, validation, planning, or apply. I use fmt and validate for configuration checks, then plan for the environment-specific evaluation. For provisioning failures, I investigate identity, permissions, provider errors, dependencies, and state before reviewing a recovery plan.”

---

## 3. How do you execute an Ansible task as root?

### Explanation

Use privilege escalation with `become: true`.

The login user and the execution user are different concepts:

- `remote_user`: User connecting to the host.
- `become`: Enables privilege escalation.
- `become_user`: User the task executes as.
- `become_method`: Escalation mechanism, such as `sudo`.

Setting `become_user: root` alone does not enable escalation. [Ansible privilege escalation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html).

### Play-level example

```yaml
---
- name: Manage application hosts
  hosts: application
  remote_user: deploy

  become: true
  become_user: root
  become_method: sudo

  tasks:
    - name: Ensure application directory exists
      ansible.builtin.file:
        path: /opt/myapp
        state: directory
        owner: root
        group: root
        mode: "0755"
```

### Task-level example

Use escalation only where needed:

```yaml
---
- name: Check host and create protected directory
  hosts: application
  remote_user: deploy

  tasks:
    - name: Display connection user
      ansible.builtin.command: whoami
      changed_when: false

    - name: Create protected application directory
      become: true
      become_user: root

      ansible.builtin.file:
        path: /opt/myapp
        state: directory
        mode: "0755"
```

### Command-line example

```bash
ansible-playbook \
  -i inventory.ini \
  site.yml \
  --become \
  --ask-become-pass
```

`--ask-become-pass`, also written `-K`, prompts for the escalation password. [Ansible privilege escalation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html).

### Production recommendations

- Use an approved automation account.
- Configure escalation deliberately.
- Do not hardcode passwords in playbooks or inventory.
- Limit elevated execution to required tasks.
- Prefer idempotent modules over unnecessary shell commands.

### Interview-ready answer

> “I connect using the automation account and use become for tasks requiring elevated privileges. I explicitly distinguish the connection user from the execution user and avoid storing escalation passwords in plaintext.”

---

## 4. Your Jenkins CI/CD pipeline failed. How do you investigate?

### Start with the failed stage

My recommended approach is to identify:

- Whether the build started.
- Which stage failed.
- The first actionable error.
- Whether the issue affects one agent, one project, or multiple pipelines.
- What changed since the last successful run.

### Suggested investigation checklist

| Stage or symptom | Suggested checks |
|---|---|
| Build remains queued | Agent availability, label matching, available executors. |
| Checkout fails | Repository access, branch/ref, credentials, connectivity. |
| Compilation fails | Source change, dependency versions, compiler output. |
| Tests fail | Test reports, fixtures, dependency availability. |
| Image build fails | Dockerfile, build context, base image, registry access. |
| Deployment fails | Effective identity, permissions, target context, rollout events. |
| Failure only on one agent | Tool versions, disk space, workspace state, connectivity. |
| Failure after configuration change | Jenkinsfile, shared-library version, plugin/configuration changes. |

These are diagnostic suggestions, not definitive causes.

### Preserve evidence

Jenkins pipelines can publish test reports and archive selected artifacts. Keep useful diagnostic outputs without collecting secrets. [Using a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/).

### Example Jenkinsfile

```groovy
pipeline {
    agent { label 'linux-build' }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            steps {
                sh './scripts/build.sh'
            }
        }

        stage('Test') {
            steps {
                sh './scripts/test.sh'
            }
        }
    }

    post {
        always {
            junit(
                testResults: 'reports/**/*.xml',
                allow***tyResults: true
            )

 ***        archiveArtifacts(
      ***       artifacts:***iagnostics/**/*.txt',
                allowEmptyArchive: true
            )
        }

        failure {
            echo 'Review the failed stage, console output, and reports.'
        }
    }
}
```

The scripts, agent label, report paths, and installed plugins must match your environment.

### Rerun carefully

Completed Declarative Pipelines support restarting from eligible top-level stages. Preserved stashes can support artifact reuse. [Running pipelines](https://www.jenkins.io/doc/book/pipeline/running-pipelines/).

My recommendation is to rerun only after understanding whether the stage is safe to repeat.

Do not blindly retry:

- Database migrations.
- Non-idempotent infrastructure operations.
- Deployment steps with unknown partial results.

### Interview-ready answer

> “I locate the failed stage and first actionable error, compare with the last successful run, and check the relevant agent, credentials, tools, and target system. I preserve reports and rerun only when the operation is safe.”

---

## 5. How do you store passwords and other sensitive information in Jenkins?

### Explanation

Use the Jenkins credentials store and reference credentials by ID.

Supported credential types include:

- Secret text.
- Username and password.
- Secret file.
- SSH username with private key.
- Certificates.

Jenkins stores credentials in encrypted form on the controller. [Using credentials](https://www.jenkins.io/doc/book/using/using-credentials/).

### Example: Bind a database credential

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'application-db',
        usernameVariable: 'DB_USER',
        passwordVariable: 'DB_PASSWORD'
    )
]) {
    sh '''
        set +x
        ./scripts/database-check.sh
    '''
}
```

The script should consume the environment variables without printing them.

### Why single-quoted Groovy strings?

Using single-quoted Groovy strings lets the shell expand environment variables instead of interpolating secrets into the Groovy command string. [Credentials Binding reference](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/).

### Security limitations

Credential masking helps prevent accidental disclosure, but it is not a security boundary.

A pipeline author with access to credentials can misuse them. Other processes running under the same operating-system account may also access environment variables. [Credentials Binding reference](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/).

### My production recommendations

- Scope credentials to the smallest appropriate folder or workload.
- Isolate trusted deployment jobs from untrusted pull-request jobs.
- Prefer short-lived workload credentials where supported.
- Rotate secrets.
- Protect controller backups and encryption material.
- Avoid secrets in command arguments, logs, artifacts, and Git.
- Review external secret-manager integrations before adoption.

### Interview-ready answer

> “I store secrets in the credentials store and inject them only into the required stage using credential IDs. I avoid Groovy interpolation and secret logging, and isolate jobs that can access production credentials.”

---

## 6. How do you manage pipelines in a multi-cloud environment?

### Recommended design

Standardize the delivery process while keeping cloud-specific implementation details separate.

Common stages can include:

1. Checkout.
2. Build.
3. Test.
4. Security checks.
5. Publish an immutable artifact.
6. Infrastructure plan.
7. Approval.
8. Cloud-specific deployment.
9. Post-deployment validation.

This is a suggested architecture, not a claim that every cloud deployment is interchangeable.

### Shared pipeline logic

Jenkins Shared Libraries allow reusable pipeline code stored in source control. [Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/).

### Suggested repository structure

```text
delivery/
├── Jenkinsfile
├── scripts/
│   ├── build.sh
│   ├── test.sh
│   └── verify-release.sh
├── deploy/
│   ├── aws/
│   ├── gcp/
│   └── azure/
└── infrastructure/
    ├── aws/
    ├── gcp/
    └── azure/
```

### Suggested separation

| Shared concern | Cloud-specific concern |
|---|---|
| Application tests | Authentication and role assumption. |
| Artifact identity | Registry access and replication. |
| Approval rules | Account/project/subscription selection. |
| Release metadata | Network and managed-service configuration. |
| Validation contract | Cloud-specific deployment commands. |

### Example parameterized pipeline

```groovy
pipeline {
    agent { label 'trusted-deploy' }

    parameters {
        choice(
            name: 'TARGET_CLOUD',
            choices: ['aws', 'gcp', 'azure'],
            description: 'Deployment destination'
        )

        choice(
            name: 'TARGET_ENV',
            choices: ['dev', 'staging', 'prod'],
            description: 'Environment'
        )
    }

    stages {
        stage('Build and Test') {
            steps {
                sh './scripts/build.sh'
                sh './scripts/test.sh'
            }
        }

        stage('Production Approval') {
            when {
                expression {
                    params.TARGET_ENV == 'prod'
                }
            }

            steps {
                input(
                    message: 'Approve production deployment?',
                    submitter: 'release-approvers'
                )
            }
        }

        stage('Deploy') {
            steps {
                withEnv([
                    "TARGET_CLOUD=${params.TARGET_CLOUD}",
                    "TARGET_ENV=${params.TARGET_ENV}"
                ]) {
                    sh '''
                        ./deploy/"$TARGET_CLOUD"/deploy.sh "$TARGET_ENV"
                    '''
                }
            }
        }
    }
}
```

This is a skeleton. Configure approved identities, immutable artifact selection, authorization, and cloud-specific scripts before use.

### Senior-level considerations

My recommendations:

- Keep state and deployment identities isolated by cloud and environment.
- Do not use one unrestricted credential for every destination.
- Promote the same artifact rather than rebuilding separately.
- Track deployment results independently.
- Define what happens if one destination succeeds and another fails.
- Treat shared-library changes as privileged changes.
- Do not assume networking, storage, or IAM semantics are identical.

### Interview-ready answer

> “I reuse build, test, and release logic, but isolate cloud-specific authentication, infrastructure, and deployment adapters. I promote immutable artifacts and keep state, credentials, and failure handling separate.”

---

## 7. How would you structure disaster recovery for an application?

### Start with business requirements

- **RTO:** Maximum acceptable delay before service restoration.
- **RPO:** Maximum acceptable age of the recovery point, representing tolerated data loss.

These objectives guide the recovery strategy. [AWS disaster-recovery workshop](https://disaster-recovery.workshop.aws/en/intro/disaster-recovery.html).

### Common strategies

| Strategy | Description |
|---|---|
| Backup and restore | Recover data and rebuild the workload when required. |
| Pilot light | Keep critical core components available and activate the remaining workload during recovery. |
| Warm standby | Keep a reduced-capacity functional environment ready. |
| Multi-site active/active | Serve traffic from multiple sites, with more complex data and failure handling. |

AWS documents these four approaches and emphasizes testing the selected strategy. [Disaster-recovery options](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html).

### Suggested warm-standby design

For an application requiring faster recovery than rebuild-from-backup, I would evaluate:

- Primary environment serving production traffic.
- Secondary environment with reduced application capacity.
- Supported database replication.
- Independently protected backups.
- Available application images and infrastructure configuration.
- Recovery-site secrets, certificates, permissions, and monitoring.
- A controlled traffic-switching mechanism.

This is an architecture recommendation; suitability depends on the workload.

### Suggested recovery runbook

1. Confirm the incident and authorize failover.
2. Establish the available recovery point.
3. Prevent conflicting writes from the old primary.
4. Promote or restore the recovery database.
5. Activate application capacity.
6. Validate dependencies and application behavior.
7. Switch traffic through the approved mechanism.
8. Monitor errors, latency, and data integrity.
9. Plan controlled failback and data reconciliation.

### What must be tested?

My recommended drill checks:

- Backup restoration.
- Replication lag.
- Recovery-site capacity.
- Credentials and key access.
- DNS and routing behavior.
- External dependencies.
- Measured recovery time.
- Data loss relative to the objective.
- Failback.

### Important distinctions

- High availability within one region does not automatically cover regional failure.
- Replication does not replace recoverable backups.
- Kubernetes manifests alone do not restore database contents.
- Do not claim an RTO or RPO without measuring the recovery process.

### Interview-ready answer

> “I define RTO and RPO first, select the appropriate recovery strategy, and include data, infrastructure, identities, artifacts, and dependencies. I test failover and failback and measure the actual recovery outcome.”

---

## 8. How would you perform a database migration?

### Clarify the migration type

My first step would be to distinguish:

- Moving data between hosts or services.
- Upgrading the database engine.
- Changing database engines.
- Applying an application schema change.

The tools and risks differ.

### Suggested migration workflow

#### 1. Assess compatibility

Review:

- Engine and version.
- Data volume and write rate.
- Data types and encoding.
- Indexes and constraints.
- Stored procedures and triggers.
- Application queries.
- Permissions.
- Downtime allowance.
- Recovery requirements.

#### 2. Prepare and rehearse

My recommendation:

- Take and test a recoverable backup.
- Prepare the destination.
- Test the migration on representative data.
- Validate application compatibility.
- Document cutover and rollback criteria.

#### 3. Choose a transfer method

| Requirement | Option to evaluate |
|---|---|
| Acceptable maintenance window | Native export/import or backup/restore. |
| Reduced cutover downtime | Initial load plus ongoing replication. |
| Cross-engine migration | Compatibility assessment and schema conversion where required. |
| Application schema evolution | Versioned, backward-compatible migrations. |

### Example: Full load plus CDC

AWS DMS supports full load followed by change data capture, or CDC-only tasks after initial data is present.

CDC reads engine-specific change logs. It is not guaranteed real-time replication; latency varies with workload and infrastructure. [DMS ongoing replication](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Task.CDC.html).

### Suggested cutover sequence

1. Validate the initial load.
2. Monitor replication and errors.
3. Pause or fence source writes through the approved process.
4. Confirm the required changes have reached the destination.
5. Validate data and application behavior.
6. Switch application connections.
7. Monitor closely before retiring the source.

### Data validation

AWS DMS can compare source and target rows and report mismatches for supported configurations. Validation adds source, target, and network load. [DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html).

My additional recommended checks:

- Business-level totals.
- Critical queries.
- Constraints and indexes.
- Sequence or identity behavior.
- Application transactions.
- Performance.

Matching row counts alone is insufficient.

### Schema migration recommendation: Expand and contract

1. Add a backward-compatible structure.
2. Deploy compatible application code.
3. Backfill and validate.
4. Switch usage.
5. Remove the old structure only after the compatibility window.

### Rollback caveat

After the destination accepts new writes, switching back to the old source can lose those writes.

Define reverse replication, reconciliation, or a forward-fix strategy before cutover.

### Interview-ready answer

> “I assess compatibility, rehearse with a tested backup, choose a transfer method based on downtime requirements, and validate both data and application behavior. I define rollback before cutover, especially for writes accepted by the new database.”

---

## 9. How do you fix CrashLoopBackOff?

### Explanation

The correct term is `CrashLoopBackOff`.

It indicates that a container repeatedly exits and Kubernetes delays subsequent restart attempts.

It is a symptom, not the underlying cause. Common categories include application errors, configuration problems, resource exhaustion, and probe failures. [Official GKE troubleshooting guide](https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/crashloopbackoff-events).

### Diagnostic commands

```bash
kubectl get pods -n NAMESPACE -o wide

kubectl describe pod POD_NAME -n NAMESPACE

kubectl logs POD_NAME \
  -n NAMESPACE \
  -c CONTAINER_NAME

kubectl logs POD_NAME \
  -n NAMESPACE \
  -c CONTAINER_NAME \
  --previous

kubectl get events \
  -n NAMESPACE \
  --sort-by=.metadata.creationTimestamp
```

Check the failing container, including init containers where applicable.

### Suggested cause-to-action checklist

| Evidence | Suggested action |
|---|---|
| `OOMKilled` | Investigate memory demand, limits, and possible leaks. |
| Application exception | Correct the application or configuration problem. |
| Failed startup/liveness probe | Verify path, port, timeout, and startup behavior. |
| Permission denied | Review file ownership, mounts, and security context. |
| Dependency connection failure | Review dependency availability, DNS, credentials, and network policy. |
| Successful immediate exit | Verify whether the workload should be a Job rather than a long-running service. |

Do not treat an exit code alone as a definitive diagnosis.

### Probe distinction

- Startup probe: Protects slow initialization.
- Liveness probe: Can trigger container restart.
- Readiness probe: Controls readiness for traffic; failure alone does not restart the container. [Probe documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

### Example probe configuration

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 30

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /live
    port: 8080
  periodSeconds: 10
```

The application must implement these endpoints. Tune the values from observed behavior.

### Validate the fix

```bash
kubectl rollout status \
  deployment/APP_NAME \
  -n NAMESPACE

kubectl get pods -n NAMESPACE
```

My recommendation is to confirm:

- Restart counts stop increasing.
- Pods become ready.
- Application requests succeed.
- Resource usage remains stable.

> **Interview trap:** Deleting the Pod can restart the cycle but does not fix the underlying problem.

### Interview-ready answer

> “I inspect the termination reason, events, and previous container logs. I fix the demonstrated cause, such as memory exhaustion, application configuration, or probe behavior, then verify rollout health and application success.”

---

## 10. What is the difference between Deployment and StatefulSet?

### Comparison

| Aspect | Deployment | StatefulSet |
|---|---|---|
| Typical workload | Interchangeable application replicas. | Workloads needing stable identity or storage association. |
| Pod identity | Replacement Pods do not retain a fixed ordinal identity. | Stable ordinal identity, such as `database-0`. |
| Storage | Can use persistent volumes, but does not provide StatefulSet-style per-Pod claim templates. | Supports `volumeClaimTemplates` for per-Pod storage. |
| Ordering | Controlled Deployment rollout. | Ordered behavior by default, with configurable policies. |
| Network identity | Usually accessed through a Service. | Stable identity supported through a governing headless Service. |

Deployments manage Pods and ReplicaSets. StatefulSets maintain sticky identities across Pod replacement. [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/).

### Example use cases

My suggested choices:

- Stateless REST API: Deployment.
- Application requiring stable per-replica identity and storage: Evaluate StatefulSet.
- Database cluster: Evaluate the database's supported operator or deployment model.

### Important caveats

- A Deployment can use a PVC.
- A StatefulSet does not automatically configure database replication.
- Stable Pod identity does not mean stable Pod IP.
- StatefulSet storage retention depends on configured retention policies.
- Ordered startup does not prove application-level consistency.

### Interview-ready answer

> “I use Deployment for interchangeable replicas. I use StatefulSet when the application needs stable per-Pod identity, storage association, or ordered behavior. StatefulSet does not replace database replication, backup, or recovery logic.”

---

## 11. Explain the terms in a Kubernetes deployment.yml file

### Example manifest

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: interview-api
  namespace: demo
  labels:
    app: interview-api

spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 10
  progressDeadlineSeconds: 300

  selector:
    matchLabels:
      app: interview-api

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        app: interview-api

    spec:
      terminationGracePeriodSeconds: 30

      containers:
        - name: api
          image: registry.example.com/interview-api:1.2.3

          ports:
            - name: http
              containerPort: 8080

          env:
            - name: APP_ENV
              value: "demo"

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"

          startupProbe:
            httpGet:
              path: /startup
              port: http
            periodSeconds: 5
            failureThreshold: 30

          readinessProbe:
            httpGet:
              path: /ready
              port: http
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /live
              port: http
            periodSeconds: 10
```

The namespace must exist, the image must be available, and the application must implement the endpoints.

### Field explanations

| Field | Meaning |
|---|---|
| `apiVersion` | API group and version for the object. |
| `kind` | Resource type. |
| `metadata.name` | Deployment name. |
| `metadata.namespace` | Namespace containing the object. |
| `metadata.labels` | Labels on the Deployment object. |
| `replicas` | Desired Pod count. |
| `selector.matchLabels` | Labels identifying managed Pods. |
| `template` | Pod template used to create replicas. |
| `template.metadata.labels` | Labels assigned to Pods. |
| `strategy` | How updates replace existing Pods. |
| `maxSurge` | Additional Pods permitted during rollout. |
| `maxUnavailable` | Unavailable Pods permitted during rollout. |
| `revisionHistoryLimit` | Old ReplicaSets retained for revision history. |
| `minReadySeconds` | Required ready duration before a Pod is considered available. |
| `progressDeadlineSeconds` | Deadline for reporting a stalled rollout. |
| `containers` | Containers included in each Pod. |
| `image` | Container image reference. |
| `containerPort` | Declared container port; it does not create a Service. |
| `env` | Container environment variables. |
| `resources.requests` | Resource requests used for scheduling and allocation. |
| `resources.limits` | Configured resource limits. |
| `terminationGracePeriodSeconds` | Grace period for Pod termination. |

Deployment rollout fields are documented in the [Deployment reference](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

### Probe fields

| Probe | Purpose |
|---|---|
| `startupProbe` | Detect successful initialization before other probes begin. |
| `readinessProbe` | Determine whether the container is ready for traffic. |
| `livenessProbe` | Detect conditions requiring container restart. |

Probe behavior is documented in the [probe reference](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

### Important interview points

- The selector must match the Pod-template labels.
- Deployment selectors are immutable after creation.
- `maxUnavailable: 0` requires capacity for surge Pods.
- A stalled rollout is reported; it does not automatically guarantee rollback.
- `containerPort` does not expose the application externally.
- Requests and limits must be selected from workload measurements.
- Three replicas alone do not guarantee distribution across failure domains.

### Suggested validation commands

```bash
kubectl apply \
  --dry-run=client \
  -f deployment.yml

# Contacts the cluster and evaluates server-side handling.
kubectl apply \
  --dry-run=server \
  -f deployment.yml
```

My recommendation is to review cluster policy results and rollout capacity before applying.

### Interview-ready answer

> “A Deployment declares desired replicas and a Pod template. The selector connects the controller to Pod labels, the strategy controls replacement, resources define allocation requirements, and probes separate startup, readiness, and liveness.”

---

## 12. Have you worked on Docker Swarm?

### Answer honestly

Use one of these templates based on your actual experience.

### Template A: Production experience

> “Yes, I worked with Docker Swarm in [actual environment]. My responsibilities included [actual responsibilities], such as service deployment, scaling, update management, node maintenance, or troubleshooting. One example was [real task or incident], where I [actual action and result].”

### Template B: Lab experience only

> “I have hands-on lab experience with Swarm, but I have not operated it in production. I have practiced [actual exercises], and I understand the distinction between manager nodes, worker nodes, services, and tasks.”

### Template C: No hands-on experience

> “I have not worked hands-on with Swarm. My container-orchestration experience is in [actual platform]. I would not claim production Swarm experience, but I can explain its documented architecture.”

### Concepts to understand

- Manager nodes manage cluster state.
- Worker nodes execute tasks.
- Services define desired workloads.
- Stacks group application services.
- Manager consensus requires a majority, called quorum.

If manager quorum is lost, existing tasks can continue running, but management and scheduling operations are restricted. [Swarm administration](https://docs.docker.com/engine/swarm/admin_guide/).

### Manager availability

Three managers require two available managers for quorum and tolerate one manager failure.

Docker recommends an odd number of managers for fault tolerance. [Swarm administration](https://docs.docker.com/engine/swarm/admin_guide/).

### Illustrative lab commands

Run initialization only on a designated lab manager:

```bash
docker swarm init \
  --advertise-addr MANAGER_PRIVATE_IP
```

Inspect the cluster:

```bash
docker node ls
docker service ls
```

### Example stack.yml

```yaml
version: "3.8"

services:
  web:
    image: nginx:1.28

    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: ingress

    deploy:
      replicas: 3

      update_config:
        parallelism: 1
        delay: 10s

      restart_policy:
        condition: on-failure
```

### Deploy and inspect

```bash
docker stack deploy \
  -c stack.yml \
  interview

docker stack services interview

docker service ps \
  interview_web \
  --no-trunc
```

Stack and service management commands run from a manager node. `docker stack deploy` uses the legacy Compose version 3 format, not every feature of the latest Compose specification. [Deploy a stack](https://docs.docker.com/engine/swarm/stack-deploy/).

### Production recommendations

- Preserve manager quorum during maintenance.
- Protect cluster administration access.
- Use approved images and registries.
- Review update and rollback behavior.
- Design persistent storage deliberately.
- Monitor service tasks and node health.
- Test recovery procedures.

### Interview-ready answer

> “My Swarm experience is [production, lab, or none]. I understand manager quorum, worker tasks, service desired state, and stack deployment. I would describe only the operations I have actually performed.”

---

# Quick Revision

1. Explain ownership and outcomes, not only tool names.
2. Terraform validation does not prove cloud provisioning will succeed.
3. A failed apply requires investigation, not automatic state deletion.
4. `become_user` alone does not enable Ansible escalation.
5. Classify Jenkins failures before retrying.
6. Credential masking is not a security boundary.
7. Share pipeline logic, but isolate cloud identities and state.
8. Define and measure RTO and RPO.
9. Replication does not replace recoverable backups.
10. Database rollback becomes harder after destination writes begin.
11. CrashLoopBackOff is a symptom, not the root cause.
12. Readiness failure alone does not restart a container.
13. StatefulSet does not automatically provide database replication.
14. `containerPort` does not create external access.
15. A Deployment progress deadline does not automatically roll back.
16. Preserve Swarm manager quorum.
17. Never invent production experience or numerical achievements.
