Below are detailed model answers for the Nisum Technologies questions. For experience-based questions, adapt the examples to work you have actually performed.

**1. Explain a CI/CD pipeline**

A CI/CD pipeline automates the process of validating code, creating a deployable artifact, and releasing it to an environment.

- **Continuous Integration:** Developers regularly merge changes, and each change triggers automated builds, tests, and security checks.
- **Continuous Delivery:** Validated changes are ready for deployment; production deployment may require approval.
- **Continuous Deployment:** Changes that pass all required checks are automatically deployed to production.

A typical pipeline follows these steps:

| Stage | What happens |
|---|---|
| Checkout | Retrieve the source code for a specific commit |
| Validate | Run linting, unit tests, code-quality checks, and dependency scans |
| Build | Compile the application or package its dependencies |
| Containerize | Build a Docker image and scan it |
| Publish | Push the versioned image to ECR, ACR, or another registry |
| Deploy to dev/QA | Deploy and run integration and smoke tests |
| Promote to production | Deploy the same tested artifact, with required approvals |
| Verify | Check health, error rate, latency, and deployment results |

**An important practice is to build once and promote the same image digest across environments.** Rebuilding separately for production can produce an artifact different from the one tested.

For Jenkins, I would define the pipeline in a version-controlled `Jenkinsfile`, so pipeline changes also go through review. Jenkins supports both Declarative and Scripted Pipeline syntax. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/?utm_source=chatgpt.com)

An interview answer can connect the tools to their purpose:

> “Git manages source code, Jenkins runs the pipeline, SonarQube checks code quality, Docker packages the application, ECR stores images, and Helm or Argo CD deploys to Kubernetes. Prometheus and Grafana help validate the release after deployment.”

---

**2. What is a “scrapper”?**

In a monitoring interview, this probably means **scraper**.

A scraper periodically collects metrics from an application or exporter. Prometheus usually collects metrics through an HTTP pull model and stores them as time series with timestamps and labels. [Prometheus](https://prometheus.io/docs/introduction/overview/?utm_source=chatgpt.com)

For example:

- An application exposes request counts and latency at `/metrics`.
- Node Exporter exposes operating-system metrics.
- Prometheus periodically requests these endpoints.
- Grafana queries the collected metrics to display dashboards.

Example Prometheus configuration:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: application
    metrics_path: /metrics
    static_configs:
      - targets:
          - "application:8080"
```

Here, Prometheus collects application metrics every 15 seconds. The hostname must resolve and the endpoint must be reachable from Prometheus.

A **web scraper** is different: it extracts information from web pages. Clarify the context if the interviewer uses this term without mentioning monitoring.

---

**3. Can we deploy services on the master node?**

**Application pods can run on a self-managed control-plane node if its configuration permits scheduling.**

“Master node” is the older term for a **control-plane node**. In a kubeadm cluster, control-plane nodes normally have this taint:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

This prevents ordinary application pods from being scheduled there. [Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/?utm_source=chatgpt.com)

A matching toleration can allow a pod onto such a node:

```yaml
# Under Pod spec, or Deployment spec.template.spec
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

A toleration **permits** scheduling; it does not force placement on that node. Use node affinity or a node selector when you also need to control placement. [Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/?utm_source=chatgpt.com)

For production, I would normally keep application workloads on worker nodes to protect control-plane capacity.

In **Amazon EKS**, AWS manages the control plane. Applications run on the cluster’s supported compute options, such as EC2 workers or Fargate, rather than the managed control-plane machines. [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html?utm_source=chatgpt.com)

Also, a Kubernetes **Service object** is a networking abstraction; the application pods behind it are the workloads scheduled onto nodes.

---

**4. Did you upgrade any services?**

This is an experience question. Explain one actual upgrade, including preparation, execution, validation, and recovery.

For an application upgrade, a strong answer would cover:

1. **Preparation:** Review release notes, dependency compatibility, configuration changes, and database implications.
2. **Testing:** Upgrade in a lower environment and run integration, smoke, and performance tests.
3. **Recovery planning:** Preserve the previous image and configuration; verify backups where persistent data is involved.
4. **Deployment:** Release through a rolling update or canary.
5. **Validation:** Check readiness, error rate, latency, resource usage, and business transactions.
6. **Recovery:** Restore the previous application version if validation fails.

For example:

> “I upgraded an application through its Helm chart. I tested the new image in staging, verified configuration compatibility, deployed gradually, and monitored errors and latency. The previous release remained available for rollback.”

If discussing an **EKS upgrade**, explain upgrade-readiness checks, deprecated APIs, control-plane upgrades one minor version at a time, and compatible updates to workers and cluster components. [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html?utm_source=chatgpt.com)

Application rollback and database rollback need separate plans: restoring an older image does not reverse a database migration.

---

**5. What deployment strategy are you following?**

Explain your actual strategy and why it suits the workload.

| Strategy | How it works | Suitable use |
|---|---|---|
| Rolling update | Gradually replaces existing pods | Routine compatible releases |
| Blue-green | Runs old and new environments, then switches traffic | Releases requiring a fast traffic switch |
| Canary | Sends a controlled portion of traffic to the new version | Higher-risk changes requiring production validation |
| Recreate | Stops the old version before starting the new version | Workloads where simultaneous versions are unsuitable and downtime is acceptable |

For a typical stateless application, a rolling-update configuration could be:

```yaml
spec:
  replicas: 3
  minReadySeconds: 10
  progressDeadlineSeconds: 300

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

- `maxUnavailable: 0` prevents the rollout from deliberately reducing availability below the desired replica count.
- `maxSurge: 1` allows one additional pod during replacement.
- `minReadySeconds` requires a new pod to remain ready before it becomes available.

This needs correct readiness probes and sufficient capacity for the extra pod. A Deployment reports a stalled rollout; it does not automatically roll back simply because its progress deadline expires. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com)

```bash
kubectl rollout status deployment/api -n prod --timeout=5m
kubectl rollout undo deployment/api -n prod
```

When using GitOps, also restore the desired version in Git so reconciliation preserves the recovery.

---

**6. What is the difference between Docker `COPY` and `ADD`?**

Both put files into an image, but `ADD` supports additional source-processing behavior.

| Capability | `COPY` | `ADD` |
|---|---|---|
| Copy files from the build context | Yes | Yes |
| Automatically extract a supported local tar archive | No | Yes, by default |
| Download a remote URL directly | No | Yes |
| Fetch a Git repository directly | No | Yes |
| Copy from another build stage | Yes, using `--from` | Use `COPY --from` for this purpose |

Examples:

```dockerfile
# Copy application files
COPY src/ /app/src/

# Copy a compiled artifact from another stage
COPY --from=builder /app/application.jar /app/application.jar

# Extract a local tar archive
ADD application.tar.gz /app/
```

Prefer **`COPY` for ordinary file copying** because its intent is explicit. Use `ADD` when you deliberately need its additional behavior.

A remote tar archive downloaded using `ADD` is not automatically extracted by default, unlike a supported local tar archive. [Docker Docs](https://docs.docker.com/reference/dockerfile/?utm_source=chatgpt.com)

---

**7. How do you fix security issues in Docker images?**

I first identify **which component is vulnerable**, then fix the source of the finding and rebuild the image.

A practical process is:

1. **Scan the image.** Identify affected packages, severity, fixed versions, and exposure.
2. **Update the base image.** Move to a supported, patched version.
3. **Update application dependencies.** A patched operating system does not fix a vulnerable Python, Java, or Node.js library.
4. **Remove unnecessary components.** Use multi-stage builds and keep build tools out of the runtime image.
5. **Rebuild and test.** Run application tests and scan the resulting image again.
6. **Deploy the patched image.** Track it using an immutable tag or digest.

Docker Scout can identify vulnerabilities and recommend base-image updates:

```bash
docker scout cves application:current
docker scout recommendations application:current
```

The scan reports findings; remediation still requires changes to the image or its dependencies. [Docker Docs](https://docs.docker.com/reference/cli/docker/scout/cves/?utm_source=chatgpt.com)

When picking up updated base images or packages, an explicit rebuild might be:

```bash
docker build --pull --no-cache -t application:patched .
docker scout cves application:patched
```

Use minimal trusted base images, a non-root runtime user, and only required packages. Add CI gates for unacceptable findings and rescan stored images as new vulnerabilities become known. [Docker Docs](https://docs.docker.com/build/building/best-practices/?utm_source=chatgpt.com)

**Keep credentials out of image layers.** Use BuildKit secret mounts for build-time credentials rather than Dockerfile `ARG` or `ENV`. Deleting a copied secret in a later layer does not reliably remove it from earlier layers. [Docker Docs](https://docs.docker.com/build/building/secrets/?utm_source=chatgpt.com)

Running as non-root reduces impact, but it does not patch a vulnerable package.

---

**8. What is the difference between “content” and tuple in Terraform?**

**Terraform does not have a data type named `content`.** If the interviewer literally means `content`, it is usually the body of a `dynamic` block.

These concepts serve different purposes:

| Concept | Meaning |
|---|---|
| `content` block | Defines the nested configuration generated by a `dynamic` block |
| `tuple` | An ordered value with a declared type for each position |
| `list` | An ordered collection with one common element type |

For example, inside a resource supporting `ingress` blocks:

```hcl
dynamic "ingress" {
  for_each = [443, 8443]

  content {
    from_port   = ingress.value
    to_port     = ingress.value
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }
}
```

The `content` block describes the body of each generated ingress rule. It is configuration syntax, not a collection type. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks?utm_source=chatgpt.com)

A tuple example:

```hcl
variable "service_details" {
  type    = tuple([string, number, bool])
  default = ["payments", 8080, true]
}
```

This tuple has three positions: a string, a number, and a Boolean.

If the intended question was **list versus tuple**, a list has one common element type and variable length; a tuple constraint specifies the number and types of its positions. Terraform can automatically convert compatible list and tuple values. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/expressions/type-constraints?utm_source=chatgpt.com)

---

**9. Any experience with Python or shell scripting? Explain one file**

Describe the problem your script solved, its inputs, error handling, output, and operational benefit.

One useful example is a shell script that extracts error and warning lines from a supplied log file:

```bash
#!/usr/bin/env bash
set -euo pipefail

if (( $# != 1 )); then
  printf 'Usage: %s LOG_FILE\n' "$0" >&2
  exit 2
fi

log_file=$1

if [[ ! -f "$log_file" || ! -r "$log_file" ]]; then
  printf 'Cannot read log file: %s\n' "$log_file" >&2
  exit 1
fi

awk '
  /ERROR|WARNING/ {
    print
    matches++
  }
  END {
    printf "Matched entries: %d\n", matches > "/dev/stderr"
  }
' "$log_file"
```

Execution:

```bash
bash log-report.sh /var/log/application.log > report.log
```

Explanation:

- The filename is supplied as an argument.
- The script rejects missing arguments and unreadable files.
- `awk` processes the file line by line.
- Lines containing uppercase `ERROR` or `WARNING` go to standard output.
- The match count goes to standard error, keeping it separate from the report.
- Quoted variables handle paths containing spaces.

For Python, suitable examples include cloud inventory reports, API integrations, backup verification, or structured JSON-log processing. Explain a script you actually understand well enough to modify during the interview.

---

**10. Any experience with Ansible?**

Ansible automates configuration management and deployments using inventories, variables, modules, and YAML playbooks.

A useful experience answer should identify:

- Which systems you managed.
- What manual process you automated.
- Which modules and roles you used.
- How you handled credentials.
- How you validated changes.

Example playbook for Ubuntu web servers:

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Ensure nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Here, the inventory defines `webservers`; the tasks install nginx and ensure it starts automatically.

`state: started` is idempotent: Ansible does not restart an already running service merely because the playbook runs again. [Ansible Community Documentation](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/service_module.html?utm_source=chatgpt.com)

Useful commands:

```bash
ansible-playbook -i inventory.ini web.yml --syntax-check
ansible-playbook -i inventory.ini web.yml --check
ansible-playbook -i inventory.ini web.yml
```

Check mode predicts supported changes; it does not fully prove that a production run will succeed. [Ansible Community Documentation](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html?utm_source=chatgpt.com)

For larger automation, roles organize related tasks, handlers, templates, files, and variables into reusable units. [Ansible Community Documentation](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html?utm_source=chatgpt.com)

---

**11. Explain AWS Fargate**

**AWS Fargate provides managed compute for containers**, so you do not provision and maintain the underlying EC2 worker fleet.

You still configure the container image, CPU, memory, networking, IAM permissions, and application scaling. AWS manages the underlying compute infrastructure. [Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html?utm_source=chatgpt.com)

| Usage | How workloads run |
|---|---|
| ECS with Fargate | ECS launches tasks using Fargate-compatible task definitions |
| EKS with Fargate | Kubernetes pods matching Fargate-profile selectors run on Fargate |

For EKS, profiles select workloads using namespaces and optionally labels.

Important **EKS Fargate** considerations include:

- DaemonSets and privileged containers are unsupported.
- GPU workloads are unsupported.
- Pods run in private subnets.
- Amazon EBS volumes cannot be mounted to Fargate pods.
- EKS Fargate does not support Fargate Spot. [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html?utm_source=chatgpt.com)

It can suit stateless APIs and workers where reducing node-management effort is valuable. EC2-backed nodes offer more host-level flexibility.

**Fargate handles compute provisioning; you still need to configure application scaling**, such as ECS Service Auto Scaling or Kubernetes HPA.

---

**12. If your pod is not running, how do you troubleshoot it?**

I first identify **where the failure occurs: scheduling, image retrieval, container creation, or application startup**.

Start with:

```bash
kubectl get pods -n prod -o wide
kubectl describe pod my-pod -n prod
kubectl get events -n prod --sort-by=.metadata.creationTimestamp
```

Then follow the observed status:

| Observed status | Checks |
|---|---|
| `Pending` | CPU/memory availability, node readiness, taints, affinity, quotas, and PVC binding |
| `ErrImagePull` / `ImagePullBackOff` | Image name/tag, registry credentials, registry availability, DNS and network access |
| `CreateContainerConfigError` | Missing ConfigMaps, Secrets, or referenced keys |
| `ContainerCreating` | Volume mounts, CSI/CNI errors, image preparation and node conditions |
| `CrashLoopBackOff` | Exit reason, command, configuration, dependencies, memory and failing startup/liveness probes |
| `Running` but not Ready | Readiness endpoint, port, timeout and application dependency health |

Pod descriptions and events are particularly useful when the application has not started and therefore has no application logs. [Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/?utm_source=chatgpt.com)

For a restarting container:

```bash
kubectl logs my-pod -n prod -c app --tail=100
kubectl logs my-pod -n prod -c app --previous --tail=100
kubectl get pod my-pod -n prod -o yaml
```

Inspect:

- Container state and last terminated state.
- Exit code and termination reason.
- Restart count.
- `OOMKilled`.
- Init-container status.

`CrashLoopBackOff` describes repeated container failures with restart backoff; it is not itself a Pod phase. [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/?utm_source=chatgpt.com)

If logs are empty, the process may exit before writing anything, write to a file instead of stdout, or fail before the container starts. If `exec` is impossible, use `kubectl debug` with an approved debugging image or a disposable pod copy. [Kubernetes](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/?utm_source=chatgpt.com)

Finally, fix the Deployment, Helm values, or configuration source and verify the rollout and actual application traffic.

---

**13. What is the difference between list and string in Terraform?**

A **string** contains one text value. A **list** contains an ordered sequence of values sharing one element type.

| Property | String | List |
|---|---|---|
| Example | `"prod"` | `["dev", "qa", "prod"]` |
| Type constraint | `string` | `list(string)` |
| Typical use | Environment name or resource name | Multiple names, subnet IDs, or ports |
| Access | Use the text value directly | Access elements by index or iterate |

Example:

```hcl
variable "environment" {
  type    = string
  default = "prod"
}

variable "environments" {
  type    = list(string)
  default = ["dev", "qa", "prod"]
}
```

```hcl
locals {
  first_environment = var.environments[0]
  combined_names    = join(",", var.environments)
}
```

`"dev,qa,prod"` is still one string. To turn it into a sequence, use:

```hcl
split(",", "dev,qa,prod")
```

Lists preserve order and can contain duplicate values. Bracketed defaults are converted to the declared list type when compatible. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/expressions/types?utm_source=chatgpt.com)

---

**14. Did you work on Helm charts?**

Helm charts package Kubernetes resource templates, metadata, and configurable values. Helm combines templates with values to produce Kubernetes manifests.

A strong experience answer explains which resources your charts managed and how environments supplied their configuration.

| Chart component | Purpose |
|---|---|
| `Chart.yaml` | Chart metadata and dependencies |
| `values.yaml` | Default configuration |
| `templates/` | Kubernetes resource templates |
| `templates/_helpers.tpl` | Reusable template definitions |
| `charts/` | Dependency charts |
| `values.schema.json` | Optional validation of supplied values |

For example, one chart can deploy a Deployment, Service, Ingress, HPA, and related configuration. Dev and production can use different value files for replica counts, resources, hostnames, and image references. [Helm](https://helm.sh/docs/topics/charts/?utm_source=chatgpt.com)

Typical commands:

```bash
helm create application

helm lint ./application

# Render manifests without deploying
helm template application ./application \
  -n dev -f values-dev.yaml

# Install or upgrade the release
helm upgrade --install application ./application \
  -n dev --create-namespace \
  -f values-dev.yaml \
  --wait --timeout 5m

helm history application -n dev
```

For failure handling, **Helm 4 uses `--rollback-on-failure`**, while **Helm 3 uses `--atomic`** for automatic rollback of a failed upgrade. [Helm](https://helm.sh/docs/helm/helm_upgrade/?utm_source=chatgpt.com)

A Helm rollback restores release resources; database changes and external side effects require their own recovery process.
