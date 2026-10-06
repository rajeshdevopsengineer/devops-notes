# DevOps, Cloud, and SRE Interview Questions

## QBurst | Experience: 3–5 Years

This guide covers Kubernetes, cloud networking, Terraform, GCP, and Docker interview questions with explanations, configuration examples, and production considerations.

> **Note:** Examples are illustrative and have not been executed against a live environment. Replace placeholders and validate configurations before production use.

---

# Kubernetes and Cloud Networking

## 1. What are the different types of Kubernetes Services?

A Kubernetes Service provides a stable endpoint for accessing a group of Pods. Individual Pods can be recreated with different IP addresses, but clients continue accessing the same Service.

Services usually identify backend Pods using label selectors.

### Service types

| Type | Purpose | Example use case |
|---|---|---|
| ClusterIP | Exposes the application through an internal cluster IP. This is the default type. | Backend API accessed by frontend Pods. |
| NodePort | Exposes the application through a port on node IP addresses. | Access through a separately managed load balancer. |
| LoadBalancer | Requests a load balancer from a supported infrastructure implementation. | Exposing an application through a cloud load balancer. |
| ExternalName | Maps a Service name to another DNS name using a CNAME response. | Providing a cluster-local alias for an external database. |

### Example: ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

**Explanation:**

- The Service selects Pods with the label `app: backend`.
- Clients connect to Service port `80`.
- The application listens on Pod port `8080`.

### Headless Service

A headless Service uses:

```yaml
spec:
  clusterIP: None
```

It does not allocate a virtual cluster IP. Instead, DNS can return individual endpoint addresses.

This is useful when clients need to discover specific application instances.

> **Interview trap:** A headless Service is not a fifth Service type. Ingress is also not a Service type; it manages HTTP/HTTPS routing to Services.

**Reference:** [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

---

## 2. What is NodePort, and when can we use it?

A NodePort Service exposes an application through:

```text
reachable-node-IP:nodePort
```

The default NodePort allocation range is **30000–32767**, although this range can be configured differently.

### Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

### Port meanings

| Field | Meaning |
|---|---|
| `port: 80` | Service port. |
| `targetPort: 8080` | Application port on the Pod. |
| `nodePort: 30080` | Port used when accessing the Service through a node. |

### Use cases

- Integrating Kubernetes with a separately managed load balancer.
- Infrastructure without a supported cloud load-balancer integration.
- Controlled direct access to node addresses in a lab or test environment.

### Production considerations

NodePort does not automatically make an application publicly accessible. The client must be able to reach the node IP address and port.

A private node IP can be accessed through private connectivity.

**Recommended approach:** For normal production application exposure, prefer a managed load balancer or an appropriate Ingress/Gateway implementation instead of directly publishing node ports.

**Reference:** [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

---

## 3. What are load balancers used for?

Load balancers distribute incoming traffic across application backends.

They are part of highly available architectures, but availability also depends on healthy, redundant application backends.

### Main categories

| Category | Suitable traffic | Example |
|---|---|---|
| Application load balancer | HTTP/HTTPS | Web application or REST API. |
| Proxy network load balancer | TCP | TCP application where proxying is appropriate. |
| Passthrough network load balancer | TCP/UDP and supported additional protocols | UDP service or application requiring preserved client packet information. |
| Internal load balancer | Private clients | Internal application accessed from a VPC or connected network. |
| External load balancer | Internet clients | Public-facing application. |

### Example architectures

**Public web application:**

An external Application Load Balancer routes requests to multiple application backends.

**Internal API:**

An internal load balancer provides a private endpoint accessed through the VPC or connected networks.

### Kubernetes consideration

In GKE, a Service with `type: LoadBalancer` configures a network load balancer.

Do not assume that this automatically provides HTTP host-based or path-based routing.

### Interview-ready answer

> “I select the load balancer based on protocol, internal versus external access, regional versus multi-region requirements, and whether the application needs proxying or client-IP preservation.”

**Reference:** [Choose a Google Cloud load balancer](https://cloud.google.com/load-balancing/docs/choosing-load-balancer)

---

## 4. What command lists Pods running on a specific node?

Use the Pod field selector `spec.nodeName`.

### All namespaces

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=NODE_NAME \
  -o wide
```

### A specific namespace

```bash
kubectl get pods -n production \
  --field-selector spec.nodeName=NODE_NAME \
  -o wide
```

### Only Running Pods on that node

```bash
kubectl get pods -A \
  --field-selector spec.nodeName=NODE_NAME,status.phase=Running
```

### Explanation

- `-A` searches all namespaces.
- `--field-selector` filters using supported resource fields.
- `spec.nodeName` identifies the node assigned to the Pod.
- `-o wide` displays additional information, including node placement.

> **Interview trap:** Use a field selector for `spec.nodeName`, not a label selector, unless relevant labels have separately been assigned to the Pods.

**Reference:** [Kubernetes field selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)

---

## 5. How can you restrict public access to standalone or GKE load balancers?

First distinguish between two requirements:

1. **No public endpoint:** Use an internal load balancer.
2. **Public endpoint with restricted access:** Keep the external endpoint but enforce appropriate access controls.

### A. Internal GKE load balancer

The following annotation requests an internal passthrough Network Load Balancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector:
    app: internal-api
  ports:
    - port: 80
      targetPort: 8080
```

Clients access the private endpoint through supported private connectivity.

Newer supported configurations can use `spec.loadBalancerClass`; requirements depend on the cluster version and configuration.

### B. Restrict source ranges

For a GKE LoadBalancer Service, use `loadBalancerSourceRanges` to configure allowed client source ranges.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: restricted-api
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - "203.0.113.0/24"
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 8443
```

> `203.0.113.0/24` is a documentation example. Replace it with your approved client CIDR.

When source ranges are omitted, the automatically created IPv4 source rule allows `0.0.0.0/0`.

### C. Standalone load balancer

Recommended design:

- Use an internal load balancer when public access is unnecessary.
- Apply access controls appropriate to the load-balancer type.
- Consider IAP for supported web applications requiring identity-based access.
- Verify that clients cannot bypass the intended entry point and access backends directly.

### Important caveat

Proxy load balancers connect to backends using proxy addresses. A backend firewall rule therefore does not necessarily filter the original client IP.

### Interview-ready answer

> “For private-only access, I choose an internal load balancer. For a restricted public endpoint, I apply controls appropriate to the load-balancer type and verify that backend access cannot bypass them.”

**Reference:** [Choose a Google Cloud load balancer](https://cloud.google.com/load-balancing/docs/choosing-load-balancer)

---

## 6. What is the major difference between AWS and GCP VPCs?

The main difference is **resource scope**.

| Aspect | AWS | GCP |
|---|---|---|
| VPC scope | Regional, spanning Availability Zones within a region. | Global. |
| Subnet scope | One Availability Zone. | Regional, usable by resources in zones within that region. |

### Architecture implications

**AWS:**

Workloads in two regions use separate regional VPCs, with explicit connectivity between them.

**GCP:**

One global VPC can contain subnets in multiple regions. Peering is not required merely because those subnets are in different regions.

GCP VPC routes and VPC firewall rules are global resources, while subnets remain regional.

### Interview-ready answer

> “AWS VPCs are regional and subnets are Availability Zone-specific. GCP VPCs are global and subnets are regional. This changes how I design multi-region connectivity and network boundaries.”

**Reference:** [Google Cloud VPC networks](https://docs.cloud.google.com/vpc/docs/vpc)

---

## 7. How do you migrate workloads from one GKE node pool to another?

Migrate the **workloads**, rather than attempting to move existing node VMs between pools.

A node pool contains nodes sharing a configuration. GKE labels each node with:

```text
cloud.google.com/gke-nodepool
```

### Suggested controlled migration

1. Create the destination pool.
2. Verify destination nodes are ready and have sufficient capacity.
3. Review node selectors, affinity, taints, and storage constraints.
4. Cordon source nodes.
5. Drain source nodes one at a time.
6. Validate application health and Pod placement.
7. Retire the source pool only after validation.

### Example commands

```bash
gcloud container node-pools create new-pool \
  --cluster=CLUSTER_NAME \
  --location=CONTROL_PLANE_LOCATION \
  --service-account=NODE_SERVICE_ACCOUNT
```

Check destination nodes:

```bash
kubectl get nodes \
  -l cloud.google.com/gke-nodepool=new-pool
```

Review disruption budgets:

```bash
kubectl get pdb -A
```

Cordon and drain one source node:

```bash
kubectl cordon OLD_NODE_NAME

kubectl drain OLD_NODE_NAME \
  --ignore-daemonsets
```

Validate placement:

```bash
kubectl get pods -A -o wide
```

### Target a specific pool

If a workload must run on the destination pool, its Pod template can include:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: new-pool
```

### Production caveats

- PodDisruptionBudgets can block eviction when availability requirements cannot be met.
- `--ignore-daemonsets` does not remove DaemonSet Pods.
- Review storage and zone constraints before migration.
- Do not promise zero downtime solely because `drain` is used.
- Avoid blindly adding `--force` or data-deletion flags to resolve a blocked drain.

### Interview-ready answer

> “I create a replacement pool, validate capacity and scheduling constraints, then cordon and drain old nodes gradually while monitoring application health. I remove the old pool only after confirming successful migration.”

**References:**

- [GKE node pools](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/node-pools)
- [Safely drain a Kubernetes node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

---

## 8. How can a VM in a private subnet run package updates?

A VM does not need its own public IP to access public package repositories. It needs an appropriate outbound path.

### Cloud options

| Platform | Common outbound option |
|---|---|
| GCP | Cloud NAT Public NAT. |
| AWS | A NAT device, commonly a NAT gateway. |

### Suggested architecture

A private VM accesses public package repositories through controlled outbound NAT connectivity.

For restricted environments, an approved internal package mirror or outbound proxy is another design option.

### Example command

```bash
sudo apt-get update
```

This refreshes package metadata. Actual package installation or upgrades should follow the organization's patching policy.

### GCP checks

Verify:

- NAT covers the VM's subnet/IP range.
- An applicable route uses the default internet gateway.
- Egress firewall rules permit the required traffic.

### Private Google Access versus Cloud NAT

- **Private Google Access:** Access to supported Google APIs and services.
- **Cloud NAT:** Outbound internet connectivity for eligible private resources.

> **Interview trap:** Private Google Access alone is not a general internet path to external package repositories.

**Reference:** [Cloud NAT Public NAT](https://cloud.google.com/nat/docs/public-nat)

---

# Terraform

## 9. How do you remove Terraform configuration without deleting the resource?

Use a `removed` block with `destroy = false` to stop managing the resource while preserving the real infrastructure.

### Example

Replace the resource declaration with:

```hcl
removed {
  from = google_storage_bucket.logs

  lifecycle {
    destroy = false
  }
}
```

Review the plan:

```bash
terraform plan
```

The intended result is removal from Terraform state without destruction of the actual resource.

### Why prevent_destroy alone is not sufficient

While the resource remains configured, this setting rejects plans that destroy it:

```hcl
lifecycle {
  prevent_destroy = true
}
```

However, removing the entire resource configuration also removes that protection.

Terraform can then plan to destroy the resource that remains recorded in state.

### Interview-ready answer

> “For destruction protection while retaining Terraform management, I use `prevent_destroy`. To remove configuration but preserve the resource, I deliberately remove it from management using a `removed` block with `destroy = false`.”

### Important consequence

After removal from state, Terraform no longer manages that resource. Plan the ownership handoff deliberately.

**References:**

- [Terraform lifecycle reference](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)
- [Terraform removed block](https://developer.hashicorp.com/terraform/language/block/removed)

---

## 10. How do you manage different Terraform environments?

A recommended production pattern is:

- Reusable modules.
- Separate root configurations for dev, staging, and production.
- Separate state and credentials.
- Environment-specific variable values.
- A reviewed CI/CD plan-and-apply workflow.

### Suggested directory structure

```text
terraform/
├── modules/
│   └── application/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    ├── staging/
    └── prod/
```

Each environment calls the same module with different values.

```hcl
module "application" {
  source = "../../modules/application"

  environment   = "prod"
  instance_size = "e2-standard-4"
}
```

The module must declare these inputs.

### CLI workspaces

CLI workspaces provide separate state instances for the same working directory.

They can be useful for parallel copies of similar infrastructure, such as temporary test environments.

```bash
terraform workspace new dev

terraform workspace select dev

terraform plan -var-file=dev.tfvars
```

CLI workspaces are not equivalent to independent credential or access-control boundaries.

### HCP Terraform workspaces

HCP Terraform workspaces contain configuration, variables, state, and run history.

They function differently from CLI workspaces.

### Interview-ready answer

> “I reuse modules but isolate production state, credentials, and execution controls. CLI workspaces can suit similar temporary environments, but I do not treat them as a security boundary.”

> **Interview trap:** Different `.tfvars` files alone do not create separate state. State isolation must be explicitly designed.

---

## 11. Can you move an existing local Terraform state to a remote backend?

Yes. Configure the remote backend and run:

```bash
terraform init -migrate-state
```

This attempts to copy existing state into the new backend.

### Example: GCS backend

```hcl
terraform {
  backend "gcs" {
    bucket = "example-company-terraform-state"
    prefix = "production/application"
  }
}
```

The bucket must already exist.

The GCS backend supports state locking. Object Versioning is recommended for recovery.

### Suggested migration workflow

1. Pause competing Terraform runs.
2. Securely back up the existing state.
3. Configure the backend and authenticate.
4. Run migration.
5. Inspect state.
6. Review a plan before making infrastructure changes.

```bash
terraform init -migrate-state

terraform state list

terraform plan
```

### migrate-state versus reconfigure

| Option | Behavior |
|---|---|
| `-migrate-state` | Copies existing state to the new backend. |
| `-reconfigure` | Reinitializes backend configuration without migrating existing state. |

### Interview-ready answer

> “Moving state storage is not the same as migrating cloud resources. I verify that the remote backend tracks the existing infrastructure before applying further changes.”

**Reference:** [Terraform init command](https://developer.hashicorp.com/terraform/cli/commands/init)

---

# GCP Access, Connectivity, and Storage

## 12. What is IAP in GCP?

**Identity-Aware Proxy**, or IAP, controls access based on authentication and authorization.

It can protect supported applications and provide administrative TCP access to VMs.

### Application access

For supported web applications, IAP checks identity and authorization before allowing access to protected resources.

### Administrative VM access

IAP TCP forwarding can tunnel SSH, RDP, and other administrative TCP traffic to VMs without requiring a public IP on the VM.

### SSH example

```bash
gcloud compute ssh VM_NAME \
  --project=PROJECT_ID \
  --zone=ZONE \
  --tunnel-through-iap
```

### Requirements

For IPv4 VM access:

- Permit the required administrative port from IAP's TCP forwarding range: `35.235.240.0/20`.
- Grant the relevant IAM permissions.
- Configure the required VM-login permissions.

### Interview-ready answer

> “IAP provides identity-based access to supported applications and administrative TCP access to private VMs. It lets me avoid assigning public IPs merely for administrative access.”

> **Interview trap:** Enabling IAP does not automatically eliminate every direct access path. Review firewall rules and backend exposure.

---

## 13. What is “VPC connect”?

The wording is ambiguous. Two likely meanings are:

1. VPC Network Peering.
2. Serverless VPC Access connector.

### A. VPC Network Peering

Peering connects two VPC networks so their resources can communicate using internal IP connectivity.

The networks can belong to different projects or organizations.

### Key points

- Both networks need a peering configuration referencing the other.
- Firewall rules still control access.
- Peering is not transitive.
- Overlapping subnet ranges can prevent valid peering configurations.

### Non-transitive example

If VPC A peers with VPC B, and VPC B peers with VPC C, this does not automatically connect VPC A with VPC C.

### B. Serverless VPC Access connector

A connector lets supported serverless workloads send outbound requests to private resources in a VPC.

Examples include internal-IP VMs and Memorystore instances.

### Example architecture

A Cloud Run application uses VPC egress to access a private backend.

For Cloud Run, Google Cloud recommends Direct VPC egress over connectors where applicable.

Direct VPC egress avoids managing connector infrastructure.

Neither option is the inbound path for requests from the VPC to Cloud Run.

### Interview-ready answer

> “If you mean VPC Peering, it connects separate VPC networks privately. If you mean a Serverless VPC Access connector, it provides VPC egress for supported serverless workloads. For Cloud Run, I also evaluate Direct VPC egress.”

**References:**

- [VPC Network Peering](https://docs.cloud.google.com/vpc/docs/vpc-peering)
- [Configure Cloud Run VPC connectivity](https://cloud.google.com/run/docs/configuring/connecting-vpc)

---

## 14. How can you reduce GCP storage bucket costs?

Evaluate the full cost:

- Stored data.
- Operations and data processing.
- Network usage.

Do not optimize only the storage price per GB.

### A. Match storage class to access patterns

| Class | Suggested use | Minimum storage duration |
|---|---|---:|
| Standard | Frequently accessed data. | None |
| Nearline | Infrequently accessed data. | 30 days |
| Coldline | Rarely accessed data. | 90 days |
| Archive | Long-lived, very rarely accessed data. | 365 days |

Colder classes can incur retrieval and early-deletion charges.

Moving frequently read or short-lived data to Archive can increase total cost.

### B. Use lifecycle rules

Lifecycle rules can transition older objects to colder storage classes and manage noncurrent object versions.

Test rules on development data or a limited prefix before applying them broadly.

### Example lifecycle rule

```json
{
  "rule": [
    {
      "action": {
        "type": "SetStorageClass",
        "storageClass": "NEARLINE"
      },
      "condition": {
        "age": 30,
        "matchesStorageClass": ["STANDARD"],
        "matchesPrefix": ["logs/"]
      }
    }
  ]
}
```

This targets Standard objects under `logs/` that are at least 30 days old.

### C. Consider Autoclass

Autoclass transitions objects according to access patterns.

It has management and enablement charges, so evaluate the total cost rather than assuming it is always cheaper.

### D. Review retained versions and soft-deleted data

Soft-deleted objects remain stored during their retention period and incur storage charges.

This can materially affect buckets containing frequently deleted temporary data.

### Recommended approach

Preserve recovery requirements first, then tune:

- Lifecycle transitions.
- Noncurrent version retention.
- Soft-delete settings.
- Bucket location.
- Unnecessary data transfers.

### Interview-ready answer

> “I optimize total workload cost, not just price per GB. I select storage classes based on access patterns, apply tested lifecycle rules, and review retained versions, soft-deleted data, and network charges.”

> **Interview trap:** Changing a bucket's default storage class does not change existing objects.

**References:**

- [Object Lifecycle Management](https://docs.cloud.google.com/storage/docs/lifecycle)
- [Cloud Storage soft delete](https://docs.cloud.google.com/storage/docs/soft-delete)

---

# Docker

## 15. What is the difference between ADD and COPY in a Dockerfile?

Both place content into an image, but `ADD` provides additional source-handling behavior.

| Feature | COPY | ADD |
|---|---|---|
| Copy local files/directories | Yes | Yes |
| Automatically extract recognized local tar archives | No | Yes, by default |
| Fetch remote URL content | No | Yes |
| Fetch Git repository content | No | Yes |
| Copy from another build stage | Yes, using `--from` | Use `COPY --from` for this purpose |

### Examples

```dockerfile
# Copy application files
COPY ./app /app

# Copy an archive without extracting it
COPY application.tar.gz /tmp/

# Extract a recognized local tar archive
ADD application.tar.gz /app/

# Copy an artifact from a build stage
COPY --from=builder /out/application /usr/local/bin/application
```

### Recommended default

Use `COPY` for ordinary file copying.

Use `ADD` when its extraction or remote-source behavior is intentional.

### Interview-ready answer

> “COPY has straightforward copying behavior. ADD supports additional features such as local archive extraction and remote sources. I normally choose COPY unless I explicitly need an ADD feature.”

> **Interview trap:** Remote tar archives are downloaded without unpacking by default. Recognized local tar archives are unpacked by default. Newer syntax provides explicit unpacking controls.

---

## 16. How do you reduce Docker image size?

The most important technique is a **multi-stage build**.

Build the application in one stage, then copy only required runtime artifacts into the final stage.

Build tools and intermediate files remain outside the final image.

### Additional techniques

- Choose a minimal, trusted base image appropriate for the application.
- Avoid unnecessary packages.
- Exclude unnecessary build-context files with `.dockerignore`.
- Keep package installation and cache cleanup in the same relevant build step.
- Copy only files required at runtime.

### Multi-stage Go example

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.26 AS builder
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -trimpath -o /out/app .

FROM scratch

COPY --from=builder /out/app /app

USER 65532:65532

ENTRYPOINT ["/app"]
```

### Explanation

- The builder stage contains the compiler and build dependencies.
- The final stage contains the application binary.
- The compiler and source tree are not copied into the final image.
- The application runs as a non-root numeric user.

### Caveat

This example assumes the application can run as a static binary.

Choose a different runtime base or explicitly include required certificates, timezone data, or other runtime assets when needed.

### Example .dockerignore

```text
.git
.env
*.log
coverage/
tmp/
```

### Inter*iew-ready answer

> “I use multi-s*age builds, select an appropriate *inimal runtime image, avoid unnece*sary packages, and copy only requi*ed runtime artifacts. I also use .*ockerignore to reduce the build co*text.”

> **Interview distinction:** A smaller build context and bett*r cache reuse can speed up builds.*They do not automatically reduce f*nal image size unless unnecessary *ontent is excluded from the final *ayers.

---

## 17. How do you pas* a value while building a Docker i*age?

Use `ARG` in the Dockerfile *nd `--build-arg` in the build comm*nd.

### Dockerfile example

```do*kerfile
FROM alpine:3.22

ARG APP_*ERSION=development

LABEL org.open*ontainers.image.version="${APP_VER*ION}"

RUN printf '%s\n' "$APP_VER*ION" > /app-version

CMD ["cat", "*app-version"]
```

### Build comma*d

```bash
docker build \
  --*uild-arg APP_VERSION=1.*.3 \
* -t interview-app:1.2.3 .
*``

This example deliberately reco*ds the version in image metadata a*d a file.

### ARG versus ENV

| I*struction | Purpose |
|---|---|
| *RG | Supplies build-time values. |*| ENV | Defines environment variab*es retained in the image and avail*ble to containers. |

If*a*build argument is needed as a runt*me environment variable, explicitl* assign it to `ENV`:

```dockerfil*
ARG APP_VERSION=development
ENV A*P_VERSION=${APP_VERSION}
```

### *RG scope

An argument declared bef*re `FROM` can parameterize `FROM`.*
Redeclare it inside a stage when *hat stage needs to use it.

```doc*erfile
ARG BASE_VERSION=3.22

FROM*alpine:${BASE_VERSION}

ARG BASE_V*RSION

RUN printf '%s\n' "$BASE_VE*SION"
```

### Do not pass secrets*through ARG

Build arguments and e*vironment variables are inappropri*te for build secrets.

Use BuildKi* secret mounts instead.

```bash
d*cker build \
  --secret id=repo_to*en,src=./repo-token.txt \
  -t app*ication:1.2.3 .
```

Consume the s*cret during a build instruction:

*``dockerfile
RUN --mount=type=secr*t,id=repo_token \
    your-build-t*ol --token-file /run/secrets/repo_*oken
```

`your-build-tool` is a p*aceholder for the actual build com*and.

Secret mounts temporarily ex*ose the secret during the instruct*on. Do not copy it into an artifac* or print it in logs.

### Intervi*w-ready answer

> “I use ARG and -*build-arg for non-sensitive build-*ime values. If the value must exis* at runtime, I explicitly configur* ENV. For secrets,*I use BuildKit secret mounts rathe* than ARG or ENV.”

**Reference*** [Docker build secrets](https://docs.docker.com/build/building/secrets/)

*--

# Quick Revision: Common Inter*iew Traps

1. **Headless is not a ***th Service type.**
2. **Ingress ***not a Service type.**
3. **NodeP*** does not automatically mean pub*** access.**
4. **A private cluste***oes not replace explicit load-ba***cer access configuration.**
5. ****grate workloads between node poo*** not individual pool VMs.**
6. ****ivate Google Access is not a gen***l internet NAT solution.**
7. *****vent_destroy does not survive re***al of the resource configuration***
8. **Different .tfvars files al*** do not isolate Terraform state.**
9. **terraform init -reconfigure is not state migration.**
10. **VPC Peering is not transitive.**
11. **Cheaper storage per GB does not necessarily mean a cheaper workload.**
12. ***OPY is the usual default***se ADD features***tentionally.**
13. **A smaller b***d context does not automatically***an a smaller final image.**
14. **ARG is for build values, not secrets.**
