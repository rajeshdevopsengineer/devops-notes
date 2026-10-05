These answers cover the **Oracle DevOps interview questions for 8 years’ experience**, with practical examples and production considerations.

**1. What is the purpose of init containers in Kubernetes?**

An **init container performs initialization before the application containers start**.

Regular init containers execute sequentially. Each must finish successfully before Kubernetes starts the next one, and all must complete before the application starts.

Typical uses include:

- Generating configuration files.
- Preparing directories and setting permissions.
- Downloading required initialization data into a shared volume.
- Waiting for a prerequisite to become available.
- Running initialization tools that are unnecessary in the application image.

For example, an init container can generate configuration in an `emptyDir` volume. The application container mounts that same volume and reads the generated files. Each container has its own filesystem, so a shared volume is required for exchanging files. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/?utm_source=chatgpt.com)

Important points:

- Initialization should be **idempotent**, because Kubernetes may retry it.
- A failing init container can prevent application startup.
- Regular init containers do not use readiness, liveness, or startup probes.
- Kubernetes native sidecars can also be declared under `initContainers` with special restart behaviour; they continue running and differ from ordinary initialization containers. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/?utm_source=chatgpt.com)

To troubleshoot initialization:

```bash
kubectl describe pod my-pod -n app
kubectl logs my-pod -n app -c initialize
```

For database schema migrations, I would usually use a controlled migration Job with concurrency safeguards, rather than run the migration independently in every application replica.

---

**2. What is the difference between a StatefulSet and a Deployment?**

Both manage Pods, but they provide different identity and lifecycle guarantees.

| Aspect | Deployment | StatefulSet |
|---|---|---|
| Typical workload | Interchangeable application replicas | Workloads requiring stable identity or storage |
| Replacement Pod name | Usually changes | Preserves the ordinal identity, such as `database-0` |
| Network identity | Normally accessed through a Service | Stable per-Pod identity, usually supported by a headless Service |
| Storage | Can mount persistent storage | Can create a separate PVC for each replica using `volumeClaimTemplates` |
| Creation and scaling | No ordinal startup requirement | Ordered by default |
| Rolling updates | Controlled by `maxSurge` and `maxUnavailable` | Normally updates Pods in descending ordinal order |
| Examples | Web applications, APIs | Database clusters, brokers requiring stable identities |

A StatefulSet is useful when application members need to recognize one another consistently across replacement or rescheduling. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com)

For example, if `mongo-0` is replaced:

- Its replacement is named `mongo-0`.
- The replacement has a new Pod UID.
- Its IP address may change.
- Its associated persistent storage can be reused.

PVCs are retained by default, although StatefulSet PVC retention behaviour is configurable. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com)

**Interview nuance:** A Deployment can use a PVC. A StatefulSet does not automatically provide database replication, quorum management, backups, or disaster recovery; those remain application or operator responsibilities.

---

**3. What is the difference between ConfigMaps and Secrets?**

| Aspect | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive configuration | Sensitive information |
| Examples | Feature flags, application mode, service URLs | Passwords, tokens, certificates |
| Common fields | `data`, `binaryData` | `data`, `stringData` |
| Consumption | Environment variables or mounted files | Environment variables or mounted files |
| Security consideration | Should contain non-confidential values | Requires appropriate access and encryption controls |

A ConfigMap separates configuration from the container image, allowing the same image to run with different settings. [Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/?utm_source=chatgpt.com)

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: app
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: app
type: Opaque
stringData:
  DB_PASSWORD: "example-only"
```

`stringData` accepts ordinary strings. Values supplied through the Secret’s `data` field must be base64 encoded. **Base64 is encoding, not encryption.** [Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/?utm_source=chatgpt.com)

A container can consume selected keys:

```yaml
env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_MODE
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: DB_PASSWORD
```

For production Secrets, I would:

- Verify encryption at rest.
- Restrict access through RBAC.
- Retrieve credentials from an approved secret manager.
- Rotate credentials and avoid committing plaintext secrets to Git.

Permission to create workloads in a namespace also needs careful control, because those workloads can potentially consume Secrets in that namespace. 

**Update behaviour matters:** Environment variables are not automatically refreshed when configuration changes. Projected ConfigMap and Secret volumes can update, but the application must reload the files; `subPath` mounts do not receive these updates. [Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/?utm_source=chatgpt.com)

---

**4. What is a PodDisruptionBudget?**

The Kubernetes object is called a **PodDisruptionBudget**, or **PDB**.

It limits voluntary evictions so that maintenance operations, such as a node drain, preserve the configured number of healthy application replicas.

For three replicas, this example requires at least two to remain available:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: app
spec:
  minAvailable: 2
  unhealthyPodEvictionPolicy: AlwaysAllow
  selector:
    matchLabels:
      app: web
```

If all three are healthy, one can normally be voluntarily evicted. If only two are healthy, another healthy Pod cannot be evicted without violating the budget.

You configure either:

- `minAvailable`: the minimum number that must remain available.
- `maxUnavailable`: the maximum number that may be unavailable.

You cannot configure both in the same PDB. [Kubernetes](https://kubernetes.io/docs/tasks/run-application/configure-pdb/?utm_source=chatgpt.com)

Important limitations:

- A PDB cannot prevent node crashes or other involuntary failures.
- Direct Pod deletion does not provide the same protection as the eviction API.
- Deployment and StatefulSet rolling updates use their own update settings.
- `AlwaysAllow` permits eviction of unhealthy running Pods, helping avoid maintenance being blocked by an application that cannot become healthy. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/?utm_source=chatgpt.com)

For reliable maintenance, combine a suitable PDB with enough replicas, spare capacity, and distribution across failure domains.

---

**5. Explain all the steps in a multi-stage Docker build.**

A multi-stage build uses multiple `FROM` statements. One stage builds the application, and a later stage copies the required output into the runtime image.

This keeps compilers, build tools, and intermediate files out of the final image. [Docker Docs](https://docs.docker.com/build/building/multi-stage/?utm_source=chatgpt.com)

Consider a Java application whose Maven build produces an executable `target/app.jar` and which listens on port 8080.

**Step 1: Select builder and runtime images.**

The builder requires a JDK. The runtime can use a JRE. The following example uses Java 21 Temurin images. [GitHub](https://github.com/docker-library/official-images/blob/master/library/eclipse-temurin?utm_source=chatgpt.com)

**Step 2: Create the Dockerfile.**

```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:21-jdk-jammy AS build
WORKDIR /workspace

COPY .mvn/ .mvn/
COPY --chmod=0755 mvnw .
COPY pom.xml .

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B dependency:go-offline

COPY src/ src/

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B verify

FROM eclipse-temurin:21-jre-jammy AS runtime
WORKDIR /app

COPY --from=build --chown=10001:10001 \
    /workspace/target/app.jar /app/app.jar

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

What happens:

1. Docker starts the named `build` stage.
2. Maven configuration is copied before frequently changing source files.
3. Dependencies are downloaded, using a BuildKit cache mount.
4. Source files are copied.
5. Maven builds and verifies the application.
6. Docker starts a separate runtime stage.
7. Only the resulting JAR is copied from the builder.
8. The application runs under a non-root UID.

Separating dependency inputs from source code improves cache reuse when only application code changes. [Docker Docs](https://docs.docker.com/build/building/multi-stage/?utm_source=chatgpt.com)

**Step 3: Restrict the build context.**

Example `.dockerignore`:

```text
.git
target/
.env
.env.*
*.log
```

This reduces unnecessary context transfer and excludes files that the build does not need.

**Step 4: Build and run the final image.**

```bash
docker build -t web-app:1.0 .

docker run --rm \
  --name web-app \
  -p 8080:8080 \
  web-app:1.0
```

`EXPOSE` documents the container port. The `-p` option publishes it on the host. [Docker Docs](https://docs.docker.com/reference/dockerfile/?utm_source=chatgpt.com)

**Step 5: Inspect and promote the image.**

Check application behaviour, scan the final image, push it to the registry, and promote the same immutable image between environments.

Build credentials should use BuildKit secret mounts. Passing credentials through `ARG`, `ENV`, or ordinary copied files can expose them in build artifacts or metadata. [Docker Docs](https://docs.docker.com/build/building/secrets/?utm_source=chatgpt.com)

---

**6. What cron expression schedules a job in Linux?**

A standard user crontab has **five time fields followed by the command**.

| Field | Meaning | Typical range |
|---|---|---|
| 1 | Minute | `0–59` |
| 2 | Hour | `0–23` |
| 3 | Day of month | `1–31` |
| 4 | Month | `1–12` |
| 5 | Day of week | `0–7`, with Sunday represented by `0` or `7` |

Examples:

| Expression | Schedule |
|---|---|
| `*/5 * * * *` | Every five minutes |
| `0 * * * *` | Every hour |
| `0 2 * * *` | Daily at 02:00 |
| `0 2 * * 1-5` | Weekdays at 02:00 |
| `0 3 * * 0` | Sundays at 03:00 |
| `0 0 1 * *` | First day of each month |

Edit and inspect a user’s schedule:

```bash
crontab -e
crontab -l
```

Example entry:

```cron
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Verify execution permissions, writable log paths, required environment variables, and the scheduler’s timezone. System crontabs additionally contain a username field.

A common trap: when day-of-month and day-of-week are both restricted, standard cron matches **either** field. [Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html?utm_source=chatgpt.com)

---

**7. What layers are created while building a Docker image?**

A Docker image contains filesystem layers inherited from its base image and added during the build.

Distinguish **filesystem layers** from **image configuration and history entries**.

| Instruction | Effect |
|---|---|
| `FROM` | Selects a base image and starts a stage |
| `RUN` | Usually produces a filesystem change layer |
| `COPY` | Adds copied filesystem content |
| `ADD` | Adds content, with additional supported behaviours |
| `ENV`, `CMD`, `ENTRYPOINT`, `EXPOSE`, `USER`, `LABEL` | Primarily configure image metadata |

Not every Dockerfile instruction adds a non-empty filesystem layer. The OCI image format explicitly supports history entries without an associated layer. [Docker Docs](https://docs.docker.com/reference/dockerfile/?utm_source=chatgpt.com)

For the multi-stage example:

- The builder contains JDK base layers, source files, and build outputs.
- The final image contains JRE base layers and the copied application JAR.
- Builder-stage content is not automatically included in the final image.

Image layers are read-only. A running container adds a writable layer for its filesystem changes. 

Important consequences:

- Removing a file in a later layer does not remove its bytes from an earlier layer.
- Cache reuse depends on instructions and their inputs.
- Changing an earlier build input can invalidate subsequent cached steps. [Docker Docs](https://docs.docker.com/build/cache/invalidation/?utm_source=chatgpt.com)

Inspect the results:

```bash
docker history --no-trunc web-app:1.0

docker image inspect web-app:1.0 \
  --format '{{json .RootFS.Layers}}'
```

---

**8. A Pod deployment throws an error. How would you investigate?**

First determine whether the failure occurs during **API submission, scheduling, container creation, or application execution**.

**Step 1: Validate the manifest and access.**

```bash
kubectl apply --dry-run=server -f pod.yaml

kubectl auth can-i create pods -n app
```

Check the API version, field names, indentation, namespace, admission policies, and RBAC permissions. Server-side dry run validates the request without persisting the resource. 

**Step 2: Inspect the Pod and events.**

```bash
kubectl get pods -n app -o wide

kubectl describe pod POD_NAME -n app

kubectl get events -n app \
  --sort-by=.metadata.creationTimestamp
```

The Pod’s events often identify the failing layer.

| Status or error | Main checks |
|---|---|
| `Pending` | CPU/memory requests, taints, affinity, PVC binding, available nodes |
| `ImagePullBackOff` | Image name/tag, registry credentials, network access |
| `CreateContainerConfigError` | Missing Secrets, ConfigMaps, or referenced keys |
| `ContainerCreating` | Volume mounts, CSI, CNI, kubelet/runtime errors |
| `Init:...` | Init-container status and logs |
| `CrashLoopBackOff` | Exit reason, startup command, configuration, probes, dependencies |
| `Running` but not Ready | Readiness path, port, timeout, and application readiness |

For a Pending Pod, remember that scheduling primarily considers **resource requests**, not current CPU utilization. [Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/?utm_source=chatgpt.com)

**Step 3: Inspect current and previous logs.**

```bash
kubectl logs POD_NAME -n app \
  -c APP_CONTAINER --tail=100

kubectl logs POD_NAME -n app \
  -c APP_CONTAINER --previous --tail=100

kubectl get pod POD_NAME -n app -o yaml
```

Inspect `state` and `lastState` for exit codes and reasons such as `OOMKilled`.

When logs are empty, the container may never have started, may exit before logging, or may write logs to a file instead of standard output. [Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/?utm_source=chatgpt.com)

**Step 4: Debug at the appropriate layer.**

If the application crashes too quickly or its image lacks a shell, use an authorized ephemeral debug container or a copied debugging Pod with changed configuration.

For example, if the application image contains a shell:

```bash
kubectl debug POD_NAME -n app -it \
  --copy-to=application-debug \
  --container=APP_CONTAINER -- sh
```

If several Pods on the same node fail, investigate node conditions, kubelet, container runtime, CNI, storage, and disk pressure. [Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/?utm_source=chatgpt.com)

**Step 5: Correct the declarative configuration and verify recovery.**

Update the manifest, Helm values, or GitOps configuration. For a Deployment:

```bash
kubectl rollout status deployment/web-app \
  -n app --timeout=180s
```

Verify readiness and an actual application request, rather than relying only on the `Running` status.

---

**9. How do you deploy a Pod to a particular node?**

A common approach is to label the node and use `nodeSelector`.

Label the intended node:

```bash
kubectl label node worker-02 \
  placement=oracle-app --overwrite

kubectl get nodes -L placement
```

Then select that label:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selected-node-pod
  namespace: app
spec:
  nodeSelector:
    placement: oracle-app
  containers:
    - name: nginx
      image: nginx:1.30-alpine
      ports:
        - containerPort: 80
```

Every specified selector label must match. If multiple nodes carry the label, the scheduler can choose among them.

Use **node affinity** when you need richer rules, such as multiple acceptable pools or preferred placement. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/?utm_source=chatgpt.com)

Important considerations:

- The selected node still needs sufficient capacity.
- Its taints may require matching tolerations.
- A toleration permits placement but does not itself select a node.
- Strictly pinning to one node can leave the Pod unavailable when that node fails. 

You can also set:

```yaml
spec:
  nodeName: worker-02
```

However, `nodeName` bypasses the scheduler and its placement decisions. For ordinary workloads, I would use selectors or affinity. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/?utm_source=chatgpt.com)

---

**10. Explain the components of a `deployment.yaml` file.**

A Deployment contains four top-level fields:

| Field | Purpose |
|---|---|
| `apiVersion` | API group and version |
| `kind` | Resource type |
| `metadata` | Name, namespace, labels, and annotations |
| `spec` | Desired configuration |

The Deployment manages ReplicaSets, which manage the application Pods.

Here is an example, assuming namespace `app` exists:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: app
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 5
  progressDeadlineSeconds: 300

  selector:
    matchLabels:
      app: web

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: web
          image: nginx:1.30-alpine
          ports:
            - name: http
              containerPort: 80

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          startupProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 2
            failureThreshold: 30

          readinessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
```

**Deployment-level settings:**

| Setting | Meaning |
|---|---|
| `replicas` | Desired number of Pods |
| `selector` | Labels identifying Pods managed by this Deployment |
| `template` | Specification used to create Pods |
| `revisionHistoryLimit` | Number of old ReplicaSets retained for rollback history |
| `minReadySeconds` | Time a new Pod must remain ready before being considered available |
| `progressDeadlineSeconds` | Deadline for reporting a stalled rollout |
| `maxSurge` | Additional Pods permitted during the update |
| `maxUnavailable` | Desired replicas permitted to be unavailable during the update |

The selector must match the template labels. A progress deadline reports rollout failure; it does not automatically roll back the Deployment. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com)

**Pod and container settings:**

- `containers`: application containers and their images.
- `ports`: declared container ports; this does not expose the application outside the cluster.
- `terminationGracePeriodSeconds`: time allowed for graceful termination.
- `requests`: resources considered for scheduling.
- `limits`: resource constraints; CPU can be throttled, and memory overuse can lead to an OOM termination. 

**Probe behaviour:**

- **Startup:** allows time for initialization before readiness and liveness probing begins.
- **Readiness:** determines eligibility for normal Service traffic.
- **Liveness:** triggers a container restart after the configured failure threshold. 

Other common fields include `env`, `volumes`, `volumeMounts`, `initContainers`, `serviceAccountName`, `securityContext`, `imagePullSecrets`, affinity, and topology spread constraints. Select them according to the workload’s requirements. 

---

**11. If a Terraform state file is corrupted, how do you fix it?**

Terraform state maps configuration addresses to real infrastructure objects. State corruption does not itself mean the cloud resources have disappeared.

My recovery procedure would be:

**Step 1: Stop concurrent changes and preserve evidence.**

Pause applies and confirm no other process is writing state. Preserve the damaged local file or raw backend object. Confirm the correct backend, workspace, account, and Terraform configuration.

Do not force-unlock a backend until you establish that its owning process has stopped. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/state/recover?utm_source=chatgpt.com)

**Step 2: Locate and validate a known-good backup.**

Possible sources include:

- Local state backups.
- Previous S3 object versions.
- State-version history in a managed Terraform backend.
- Approved backup systems.

For default local-state paths, a simplified restore is:

```bash
cp terraform.tfstate terraform.tfstate.corrupt

python3 -m json.tool terraform.tfstate.backup > /dev/null

cp terraform.tfstate.backup terraform.tfstate
```

JSON validation only checks syntax. Also verify that the backup belongs to the correct infrastructure and contains the expected resource mappings. Local backend paths can be configured, so use the actual paths for the selected workspace. 

**Step 3: Restore the remote backend appropriately.**

For S3, previous versions can be recovered if versioning was enabled. Terraform recommends enabling bucket versioning for state recovery. 

Where appropriate, Terraform can upload a reviewed snapshot:

```bash
terraform state push recovered.tfstate
```

The command checks lineage and serial numbers. A different lineage or a higher remote serial can prevent the upload. If it is rejected, investigate the differences and follow the backend’s recovery procedure; `-force` disables those safeguards. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/state/push?utm_source=chatgpt.com)

**Step 4: Reconcile with actual infrastructure.**

```bash
terraform state list

terraform plan -refresh-only

terraform plan
```

Review both plans. A refresh-only plan reconciles recorded objects with their current remote attributes; it does not discover and import every untracked resource. [HashiCorp Developer](https://developer.hashicorp.com/terraform/cli/commands/plan?utm_source=chatgpt.com)

An older backup may omit resources created later. Import those existing objects before applying a normal plan that proposes creating them.

**Step 5: If there is no usable backup, rebuild the mappings through import.**

Create matching configuration and import existing resources:

```bash
terraform import \
  aws_instance.web \
  i-0123456789abcdef0
```

Use the correct resource addresses and provider-specific import IDs. Imports establish management of existing objects; review the resulting plan for configuration differences. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/import?utm_source=chatgpt.com)

To reduce future recovery risk, use versioned remote state, encryption, locking, restricted access, and tested recovery procedures.

---

**12. Explain Terraform lifecycle: create before or after destroy.**

For a resource replacement, Terraform normally:

1. Destroys the existing object.
2. Creates the replacement.

The lifecycle setting that reverses this order is **`create_before_destroy`**.

```hcl
resource "aws_instance" "web" {
  ami                    = var.approved_ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  lifecycle {
    create_before_destroy = true
  }
}
```

When replacement is required, Terraform creates the new instance before destroying the old one. Ordinary in-place updates do not require this replacement sequence. 

Conditions to consider:

- Both resources must temporarily fit within capacity and quotas.
- Unique-name constraints must permit coexistence.
- Successful resource creation does not necessarily mean the application is healthy.
- Traffic switching and connection draining must be designed separately.

For an application, I would combine replacement capacity with load-balancer health checks and graceful draining to maintain availability.

Other common lifecycle settings are:

| Setting | Purpose |
|---|---|
| `prevent_destroy` | Reject destructive plans while the protection remains configured |
| `ignore_changes` | Ignore selected attributes when planning updates |
| `replace_triggered_by` | Replace a resource when a referenced managed object changes |

Removing a resource’s configuration also removes its `prevent_destroy` protection; it does not prevent manual cloud deletion. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle?utm_source=chatgpt.com)

**If “after destroy” means executing an operation:** a provisioner with `when = destroy` runs **before** deletion. Enabling `create_before_destroy` prevents destroy-time provisioners from running. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/provisioners?utm_source=chatgpt.com)

Current Terraform also documents provider-defined lifecycle actions through `action_trigger`, including an `after_destroy` event. Use those only with compatible Terraform and provider versions. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle?utm_source=chatgpt.com)
