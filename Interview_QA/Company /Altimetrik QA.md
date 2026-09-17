# Senior DevOps/SRE Interview Questions and Detailed Answers

> **Profile focus:** Terraform, AWS, GCP, GKE, Kubernetes, Helm, observability, SRE, incident management, cost optimization, and production operations.
>
> **How to use this guide:** The answers are written in interview-ready first-person language. Replace example tools, project details, environment names, measurements, and achievements with information from your actual experience. Do not claim experience with a tool or result unless you can explain it confidently.

---

## Table of Contents

1. [Importing manually created resources into Terraform](#1-how-do-you-import-a-resource-into-terraform-that-was-created-manually-in-aws-or-gcp)
2. [Experience across Dev, QA, UAT, and Prod](#2-can-you-describe-your-exposure-to-environments-like-dev-qa-and-prod)
3. [Software architecture components](#3-what-is-your-understanding-of-software-architecture-components)
4. [Alerts, logging, and incident resolution](#4-what-is-your-experience-with-alerts-logging-and-incidentproblem-resolution)
5. [Production sizing and lifecycle management](#5-what-is-your-knowledge-of-production-system-sizing-provisioning-setup-maintenance-and-closure)
6. [Licensing, billing, cost, and security](#6-describe-your-experience-with-licensing-billing-cost-reduction-and-security)
7. [Helm chart structure and integration](#7-walk-me-through-a-helm-chart-and-explain-how-it-is-integrated)
8. [Contents of values.yaml](#8-what-is-defined-in-valuesyaml)
9. [Deploying a microservice into GKE](#9-how-does-a-microservice-get-deployed-into-a-gke-cluster)
10. [Troubleshooting failing Kubernetes pods](#10-how-do-you-troubleshoot-a-failing-or-terminating-pod-in-gke)
11. [General troubleshooting approach](#11-what-is-your-general-approach-to-troubleshooting)
12. [Observability frameworks and tools](#12-which-observability-frameworks-and-tools-do-you-use)
13. [Creating dashboards and alerts](#13-are-you-familiar-with-creating-dashboards-and-alerts-yourself)
14. [Dashboards and alerts created](#14-what-types-of-dashboards-and-alerts-have-you-created)
15. [SRE golden signals](#15-have-you-used-sre-golden-signals-for-dashboarding-and-alerting)
16. [SLO-based monitoring](#16-have-you-implemented-slo-based-monitoring)
17. [Error budgets](#17-what-is-an-error-budget)
18. [SLO-based alerting](#18-have-you-configured-slo-based-alerting)
19. [Burn rate](#19-what-is-an-ideal-burn-rate)
20. [Deploying cluster nodes using Terraform](#20-how-do-you-use-terraform-to-deploy-cluster-nodes)
21. [Terraform file structure](#21-what-do-you-write-in-providertf-maintf-variablestf-and-other-terraform-files)
22. [AWS VPC using Terraform](#22-and-23-write-terraform-code-to-create-an-aws-vpc-and-subnet)
24. [Kubernetes deployment optimization](#24-how-have-you-optimized-kubernetes-deployment-configurations)
25. [Blameless postmortem framework](#25-how-did-you-build-a-blameless-postmortem-framework-and-prevent-recurring-incidents)
26. [Automation to reduce MTTR](#26-what-automation-have-you-implemented-to-reduce-mttr)

---

## 1. How do you import a resource into Terraform that was created manually in AWS or GCP?

### Interview-ready answer

If a resource already exists in AWS or GCP but was created manually, I first define a corresponding resource block in Terraform. I then initialize the Terraform working directory and use `terraform import` to associate the existing cloud resource with a Terraform resource address in the state file.

The generic command is:

```bash
terraform import <terraform-resource-address> <cloud-resource-id>
```

For example, to import an existing AWS EC2 instance:

```hcl
resource "aws_instance" "existing_server" {
  # Arguments will be aligned with the existing instance
}
```

```bash
terraform init
terraform import aws_instance.existing_server i-0123456789abcdef0
terraform state show aws_instance.existing_server
terraform plan
```

For an existing GCP storage bucket:

```hcl
resource "google_storage_bucket" "existing_bucket" {
  name     = "application-data-bucket"
  project  = "my-gcp-project"
  location = "ASIA-SOUTH1"
}
```

```bash
terraform import \
  google_storage_bucket.existing_bucket \
  my-gcp-project/application-data-bucket
```

### What actually happens during import?

`terraform import` does not automatically create a complete, maintainable Terraform configuration in the traditional workflow. Its primary purpose is to create a mapping in Terraform state between:

- The Terraform resource address, such as `aws_instance.existing_server`
- The real cloud resource ID, such as `i-0123456789abcdef0`

After importing, I run:

```bash
terraform state show aws_instance.existing_server
terraform plan
```

`terraform state show` helps me inspect the attributes Terraform discovered. I then update the `.tf` configuration until `terraform plan` shows either no changes or only the changes I intentionally want to make.

### Configuration-driven import

Modern Terraform also supports an `import` block, which is useful because it makes the import reviewable and repeatable through normal Terraform workflows:

```hcl
import {
  to = aws_instance.existing_server
  id = "i-0123456789abcdef0"
}
```

The import is then processed during `terraform apply`.

### Important precautions

1. I back up the remote state before a sensitive import.
2. I confirm that the resource is not already managed at another Terraform address.
3. I use the exact resource ID format required by the provider.
4. I check dependencies, such as subnet, security groups, disks, and IAM roles.
5. I run `terraform plan` after the import to prevent accidental modification or replacement.
6. I never import one remote object into multiple Terraform addresses.
7. I avoid editing the state file manually. If state correction is required, I use supported commands such as `terraform state mv` or `terraform state rm` with appropriate review and backup.

---

## 2. Can you describe your exposure to environments like Dev, QA, and Prod?

### Interview-ready answer

I have worked with isolated environments such as Development, QA, UAT, staging, pre-production, and Production. My responsibility is to keep the delivery process consistent across environments while applying stricter controls as an application moves closer to production.

### Environment purposes

#### Development

The Development environment is optimized for rapid feedback. Developers deploy frequently, test new functionality, and troubleshoot integration problems. It may use smaller infrastructure and relaxed approval controls, but basic security, logging, and tagging standards still apply.

#### QA

QA is used for functional, integration, regression, API, and automation testing. It should be more stable than Development. Test data, dependent services, and configuration should be controlled enough to produce repeatable results.

#### UAT or staging

UAT or staging should closely resemble Production in architecture, deployment method, security controls, and configuration shape. It is used for acceptance, release validation, performance tests, and operational readiness checks.

#### Production

Production requires the strongest controls, including:

- High availability and redundancy
- Restricted access and least privilege
- Change approvals and audit trails
- Monitoring, alerting, and on-call ownership
- Backup and disaster-recovery procedures
- Capacity planning and autoscaling
- Controlled deployment and rollback
- Security and compliance validation

### How I separate environments

Depending on organizational requirements, I use:

- Separate AWS accounts or GCP projects
- Separate VPCs and subnets
- Separate Kubernetes clusters or namespaces
- Separate IAM roles and service accounts
- Separate Terraform state paths or backends
- Separate secret paths and encryption keys
- Environment-specific Helm values
- CI/CD approval gates for UAT and Production

A possible repository structure is:

```text
infrastructure/
├── modules/
│   ├── network/
│   ├── kubernetes/
│   └── database/
└── environments/
    ├── dev/
    ├── qa/
    ├── uat/
    └── prod/
```

### Artifact promotion principle

I prefer to build the deployable artifact once and promote the same immutable artifact across environments. For containers, that means promoting the same image digest or immutable image tag from Dev to QA, UAT, and Production. Only environment configuration changes. Rebuilding the application for each environment can introduce inconsistencies and weaken traceability.

---

## 3. What is your understanding of software architecture components?

### Interview-ready answer

I understand software architecture as the complete path a request follows, from the client to the application and its downstream dependencies. From a DevOps and SRE perspective, I evaluate every component for availability, security, capacity, observability, scalability, and failure handling.

```text
User or Client
      |
      v
DNS -> CDN -> WAF
      |
      v
Load Balancer / Kubernetes Ingress
      |
      v
Web or Frontend Layer
      |
      v
Application Services / Microservices
      |
      +-----------------------------+
      |              |              |
      v              v              v
   Database        Cache       Queue/Event Bus
      |
      v
Internal and External Integrations
```

### Load balancer

A load balancer distributes incoming traffic across healthy backends. It may also perform:

- TLS termination
- Health checks
- Path-based or host-based routing
- Session persistence, if required
- Connection draining
- Cross-zone traffic distribution
- Integration with a WAF

In Kubernetes, an Ingress controller or Gateway implementation often performs Layer 7 routing, while a cloud load balancer exposes the service externally.

### Web server

A web server such as NGINX or Apache can serve static content and act as a reverse proxy. It can handle compression, caching headers, TLS, request routing, and connection management.

### Application server

The application layer implements business logic. In microservice architecture, it usually communicates with databases, caches, queues, and other APIs. Important operational characteristics include:

- Stateless versus stateful behavior
- Startup and shutdown behavior
- Health endpoints
- Connection-pool configuration
- Timeouts and retries
- Circuit breaking
- Idempotency
- Horizontal scalability

### Database

A database provides persistent storage. For Production, I consider:

- High availability and replication
- Backup and point-in-time recovery
- Recovery point objective and recovery time objective
- Query performance and indexing
- Connection pooling
- Encryption
- Storage growth
- Failover behavior
- Maintenance windows

### Cache

A cache reduces latency and database load. It improves performance but introduces issues such as expiration, stale data, memory pressure, and cache stampedes. The application should have a defined response when the cache is unavailable.

### Messaging and integrations

Queues and event streams decouple services and support asynchronous processing. I monitor publish errors, queue depth, consumer lag, dead-letter queues, and processing latency. For external integrations, I review authentication, rate limits, retries, timeouts, and fallback behavior.

---

## 4. What is your experience with alerts, logging, and incident/problem resolution?

### Interview-ready answer

I use metrics, logs, traces, Kubernetes events, cloud audit logs, and deployment history together. My objective during an incident is first to restore service safely and then identify and remove the root cause.

### Incident-response lifecycle

1. **Detection:** Monitoring or a user report identifies an issue.
2. **Triage:** I validate the alert and identify the affected service and environment.
3. **Severity:** I classify the incident based on customer and business impact.
4. **Ownership:** An incident commander and relevant technical owners are engaged.
5. **Mitigation:** We roll back, scale, fail over, disable a feature, or correct configuration.
6. **Communication:** Stakeholders receive clear and regular updates.
7. **Validation:** We confirm technical recovery and customer recovery.
8. **Root-cause analysis:** Evidence is reviewed after stabilization.
9. **Prevention:** Corrective actions are assigned owners and target dates.

### Alerting principles

I try to make every paging alert actionable. A good alert should contain:

- Service and environment
- Severity and observed condition
- Customer or system impact
- Dashboard and log links
- Runbook link
- Service owner and escalation route
- Relevant labels, such as cluster, namespace, region, or version

I reduce alert fatigue through appropriate thresholds, evaluation windows, grouping, deduplication, routing, inhibition, and regular alert reviews.

### Logging practices

I prefer structured logs, normally JSON, with fields such as:

```json
{
  "timestamp": "2026-09-17T10:30:00Z",
  "severity": "ERROR",
  "service": "order-service",
  "environment": "production",
  "traceId": "abc123",
  "requestId": "req456",
  "message": "Database request timed out"
}
```

I avoid logging secrets, passwords, tokens, or unnecessary personal data. Log retention and access are controlled according to security and compliance requirements.

### Incident versus problem management

- **Incident management** focuses on restoring service quickly.
- **Problem management** focuses on identifying systemic causes and preventing recurrence.

A restart may mitigate an incident, but it is not necessarily the root-cause fix. I investigate why the service required a restart and whether monitoring, capacity, code, configuration, or architecture must change.

---

## 5. What is your knowledge of production system sizing, provisioning, setup, maintenance, and closure?

### Interview-ready answer

I treat production infrastructure as a complete lifecycle. I start with measurable workload and availability requirements, provision using Infrastructure as Code, validate operational readiness, maintain the platform proactively, and decommission resources through an approved closure process.

### Sizing inputs

I collect:

- Current and forecast requests per second
- Concurrent users or active sessions
- CPU and memory utilization
- Latency percentiles
- Data volume and growth rate
- Storage IOPS and throughput
- Network ingress and egress
- Batch window and queue depth
- Peak traffic and seasonal patterns
- Availability target and redundancy requirements
- RTO and RPO
- Dependency quotas and rate limits

I use historical metrics, performance testing, capacity tests, and failure tests instead of relying only on estimates. I also retain headroom so the service can tolerate traffic bursts and component failure.

### Provisioning and setup

I provision infrastructure using Terraform or another approved IaC solution. The setup includes:

- Networking and routing
- IAM and service identities
- Encryption keys
- Compute or Kubernetes resources
- Storage and database configuration
- Logging and monitoring
- Backup policies
- Alerts and runbooks
- Autoscaling
- Resource tags and ownership

Before release, I complete an operational-readiness review that verifies dashboards, alerts, backups, recovery procedures, capacity, security, ownership, and escalation paths.

### Maintenance

Regular maintenance includes:

- OS, node, and application patching
- Kubernetes and add-on upgrades
- Certificate and secret rotation
- Backup and restore testing
- Capacity reviews
- Cost reviews
- Vulnerability remediation
- Database maintenance
- Removal of obsolete resources
- Review of incidents and recurring alerts

### Closure or decommissioning

I use a controlled sequence:

1. Confirm the business owner and obtain approval.
2. Map upstream and downstream dependencies.
3. Remove traffic and confirm no active consumers remain.
4. Back up or archive data according to retention policy.
5. Revoke credentials, DNS records, firewall rules, and integrations.
6. Destroy infrastructure using IaC where possible.
7. Remove monitoring, alerts, licenses, and inventory entries.
8. Confirm that billing has stopped.
9. Document completion and retained data.

---

## 6. Describe your experience with licensing, billing, cost reduction, and security

### Interview-ready answer

I consider administration to include governance as well as technical operations. I track ownership and licensing, improve cost visibility, remove waste, and implement security controls through automation and policy.

### Licensing

My approach includes:

- Maintaining ownership and subscription inventory
- Tracking renewal and expiry dates
- Comparing allocated licenses with actual usage
- Removing unused licenses
- Validating product edition and deployment rights
- Monitoring license servers where applicable
- Including licensing in decommissioning procedures

### Billing and cost allocation

I use mandatory tags or labels such as:

```text
Environment = Production
Application = OrderPlatform
Owner       = PaymentsTeam
CostCenter  = CC1001
ManagedBy   = Terraform
```

This allows cost reporting by environment, application, team, and business unit. I also configure budgets, forecast notifications, and anomaly alerts.

### Cost optimization

Typical actions include:

- Rightsizing compute instances and Kubernetes requests
- Using cluster and workload autoscaling
- Scheduling non-production shutdowns where permitted
- Deleting unattached disks, snapshots, idle IPs, and unused load balancers
- Applying object-storage lifecycle policies
- Reducing unnecessary log volume and retention
- Using reserved capacity or commitments for stable workloads
- Selecting spot or preemptible capacity for fault-tolerant workloads
- Optimizing network egress
- Reviewing database size and storage class

Cost optimization must not compromise availability, security, recovery, or compliance.

### Security

The controls I focus on include:

- Least-privilege IAM
- Strong authentication and short-lived credentials
- Encryption at rest and in transit
- Secrets management and rotation
- Network segmentation
- Private endpoints where suitable
- Vulnerability and container-image scanning
- Patch management
- Audit logging
- Backup protection
- Policy as code
- Secure CI/CD controls
- Kubernetes admission and workload-security policies

---

## 7. Walk me through a Helm chart and explain how it is integrated

### Interview-ready answer

A Helm chart is a versioned package of templated Kubernetes resources. It allows the same application deployment definition to be reused across Development, QA, UAT, and Production by supplying different values.

### Typical structure

```text
order-service/
├── Chart.yaml
├── values.yaml
├── values.schema.json
├── README.md
├── charts/
├── crds/
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    ├── pdb.yaml
    ├── networkpolicy.yaml
    └── NOTES.txt
```

### Explanation of files

#### `Chart.yaml`

Contains chart metadata:

```yaml
apiVersion: v2
name: order-service
description: Helm chart for the order microservice
type: application
version: 1.3.0
appVersion: "2.7.4"
```

`version` is the chart version, while `appVersion` describes the application version. They serve different purposes.

#### `values.yaml`

Contains default input values used by templates. Environment files override these defaults.

#### `templates/`

Contains Kubernetes resources written as Go templates. Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: order-service
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

#### `_helpers.tpl`

Contains reusable helpers for names, labels, and selectors.

#### `values.schema.json`

Validates required values and data types before deployment. This reduces failures caused by malformed configuration.

### CI/CD integration

The pipeline generally performs:

```bash
helm dependency update ./charts/order-service
helm lint ./charts/order-service

helm template order-service ./charts/order-service \
  -f ./environments/prod.yaml \
  --set image.tag="$IMAGE_TAG"

helm upgrade --install order-service ./charts/order-service \
  --namespace production \
  --create-namespace \
  -f ./environments/prod.yaml \
  --set image.tag="$IMAGE_TAG" \
  --atomic \
  --wait \
  --timeout 10m
```

The CI stage builds, tests, scans, and pushes the container image. The CD stage authenticates to the cluster and deploys the chart. `--atomic` helps roll back a failed upgrade, while `--wait` waits for supported resources to become ready before reporting success.

### Validation and rollback

Useful commands include:

```bash
helm list -n production
helm status order-service -n production
helm history order-service -n production
helm get values order-service -n production
helm get manifest order-service -n production
helm rollback order-service <revision> -n production
```

---

## 8. What is defined in `values.yaml`?

### Interview-ready answer

`values.yaml` contains the default configuration passed into Helm templates. I use it for values that can vary between applications or environments, while the Kubernetes resource structure remains in the templates.

```yaml
replicaCount: 3

image:
  repository: asia-south1-docker.pkg.dev/project/apps/order-service
  tag: "2.7.4"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: orders.example.com
  tls:
    enabled: true
    secretName: orders-tls

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "1Gi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

probes:
  startup:
    path: /health/startup
    initialDelaySeconds: 5
    periodSeconds: 5
    failureThreshold: 30
  readiness:
    path: /health/ready
    periodSeconds: 10
  liveness:
    path: /health/live
    periodSeconds: 20

config:
  logLevel: INFO
  dependencyTimeoutSeconds: 5

serviceAccount:
  create: true
  name: ""

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

### Environment overrides

```text
values.yaml
values-dev.yaml
values-qa.yaml
values-prod.yaml
```

```bash
helm upgrade --install order-service ./chart \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag="$IMAGE_TAG"
```

The later value source overrides the earlier one. I use `--set` sparingly for pipeline-generated values such as an image tag.

### Secret-handling rule

I do not commit plain-text passwords, private keys, or tokens to `values.yaml`. The chart should reference a Kubernetes Secret or integrate with an approved external secret-management solution.

---

## 9. How does a microservice get deployed into a GKE cluster?

### Interview-ready answer

A source-code change passes through CI to create an immutable container image. After testing and security checks, CD deploys that exact image to GKE using Helm or a GitOps controller. Kubernetes then creates the required ReplicaSet and pods, and readiness checks determine when traffic can reach the new version.

### End-to-end flow

1. A developer creates a pull request.
2. Automated unit, static-analysis, and policy checks run.
3. The change is reviewed and merged.
4. CI builds a container image.
5. The image is scanned for vulnerabilities.
6. The approved image is pushed to Artifact Registry.
7. The image digest or immutable tag is recorded.
8. The CD pipeline authenticates to GCP and GKE.
9. Helm renders the Kubernetes manifests.
10. Kubernetes applies the Deployment change.
11. The Deployment creates a new ReplicaSet.
12. The scheduler places pods on eligible nodes.
13. The kubelet pulls and starts the image.
14. Startup and readiness probes are evaluated.
15. Ready pods are added to service endpoints.
16. The pipeline verifies rollout and service health.

### Example deployment commands

```bash
gcloud container clusters get-credentials application-cluster \
  --region asia-south1 \
  --project production-project

helm upgrade --install order-service ./charts/order-service \
  --namespace production \
  --create-namespace \
  -f ./environments/prod.yaml \
  --set image.tag="$COMMIT_SHA" \
  --atomic \
  --wait \
  --timeout 10m

kubectl rollout status deployment/order-service \
  --namespace production \
  --timeout 5m
```

### Deployment strategies

- **Rolling update:** Gradually replaces old pods.
- **Blue-green:** Runs old and new versions separately and switches traffic.
- **Canary:** Sends a small percentage of traffic to the new version first.

For production, I combine deployment progress with application-level metrics such as error rate, latency, availability, and critical business transactions.

---

## 10. How do you troubleshoot a failing or terminating pod in GKE?

### Interview-ready answer

I first identify the exact Kubernetes state and reason. I do not begin by repeatedly restarting the pod because a restart can hide evidence and does not fix configuration, scheduling, image, resource, or application problems.

### Initial triage

```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --all-containers=true
kubectl logs <pod-name> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

I examine:

- Pod phase and container state
- `reason`, `exitCode`, and restart count
- Last terminated state
- Pod conditions
- Scheduler and kubelet events
- Image name and image-pull events
- Requests, limits, probes, volumes, and mounts
- Node assignment
- Recent deployment or configuration changes

### Troubleshooting by status

#### `CrashLoopBackOff`

Possible causes include an application exception, invalid command, missing configuration, unavailable dependency, or failed liveness probe.

```bash
kubectl logs <pod> -n <namespace> --previous
kubectl describe pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
```

#### `OOMKilled`

The container exceeded its memory limit. I compare actual usage with requests and limits, inspect application memory behavior, and determine whether it is a sizing problem or a memory leak.

```bash
kubectl top pod <pod> -n <namespace> --containers
kubectl describe pod <pod> -n <namespace>
```

#### `ImagePullBackOff` or `ErrImagePull`

I check:

- Repository and tag
- Whether the image exists
- Registry credentials or workload identity
- Network and DNS connectivity
- Image pull secrets
- Registry quota or permissions

#### `Pending`

I look for insufficient CPU or memory, taints, affinity rules, topology constraints, quota, unbound PVCs, or autoscaler limitations.

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get nodes
kubectl describe node <node>
kubectl get resourcequota,limitrange -n <namespace>
kubectl get pvc -n <namespace>
```

#### `Running` but not `Ready`

I examine readiness-probe failures, service ports, endpoint membership, application dependencies, and network policies.

```bash
kubectl get svc,endpoints,endpointslices -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl port-forward pod/<pod> 8080:8080 -n <namespace>
```

#### `Evicted`

I investigate node memory, disk, PID, or ephemeral-storage pressure. I also check whether requests are realistic and whether application logs or temporary files are filling local storage.

#### Stuck in `Terminating`

I check:

- Finalizers
- Long `preStop` hooks
- `terminationGracePeriodSeconds`
- Unresponsive processes
- Volume detach problems
- Admission webhooks
- Node availability

```bash
kubectl get pod <pod> -n <namespace> -o yaml
kubectl describe pod <pod> -n <namespace>
```

Force deletion is a last resort because it can leave live processes or storage operations in an uncertain state.

### Workload, service, and node checks

```bash
kubectl get deployment,replicaset,pod -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
kubectl rollout history deployment/<deployment> -n <namespace>
kubectl top nodes
kubectl top pods -n <namespace>
kubectl get networkpolicy -n <namespace>
```

If the image lacks troubleshooting tools, I use an approved ephemeral debug container where supported:

```bash
kubectl debug <pod-name> -it \
  --image=busybox \
  --target=<container-name> \
  -n <namespace>
```

### Resolution and validation

I correct the source configuration in Git or IaC, deploy through the approved pipeline, verify rollout status, confirm readiness and service endpoints, and monitor errors, latency, saturation, and customer-facing checks.

---

## 11. What is your general approach to troubleshooting?

### Interview-ready answer

My troubleshooting method is evidence-driven and impact-focused. I first establish the symptoms and scope, stabilize the service when necessary, and then move systematically through the request path while testing one hypothesis at a time.

### Structured method

1. **Understand the symptom:** What exactly is failing?
2. **Establish the timeline:** When did it start?
3. **Determine scope:** One user, pod, service, zone, region, or the entire platform?
4. **Assess impact:** Is it a customer-facing outage, degradation, or internal issue?
5. **Check recent changes:** Code, configuration, infrastructure, certificate, secret, or dependency.
6. **Follow the request path:** DNS, load balancer, ingress, service, pod, database, and external dependency.
7. **Collect evidence:** Metrics, logs, traces, events, audit logs, and deployment history.
8. **Form a hypothesis:** State what should be true if the hypothesis is correct.
9. **Test safely:** Change one variable at a time where possible.
10. **Mitigate:** Roll back, scale, isolate, fail over, or correct the problem.
11. **Validate recovery:** Confirm both system metrics and user experience.
12. **Document and prevent:** Record root cause and track corrective actions.

### Example request-path checks

```text
DNS resolution
   -> Load balancer health
      -> Ingress/Gateway routing
         -> Kubernetes Service selector
            -> EndpointSlice membership
               -> Pod readiness
                  -> Application logs
                     -> Database/dependency health
```

My guiding principle is: **restore service safely, preserve evidence, identify the root cause, and prevent recurrence.**

---

## 12. Which observability frameworks and tools do you use?

### Interview-ready answer

I use the three major observability signals, metrics, logs, and traces, along with events and deployment context. The actual toolset varies by project, but the objective is to correlate a customer symptom with the component and change responsible for it.

### Metrics

Possible tools include Prometheus, Grafana, Google Cloud Monitoring, and AWS CloudWatch. Metrics are useful for rates, ratios, percentiles, capacity trends, and SLO measurement.

### Logs

Centralized logging can use Google Cloud Logging, CloudWatch Logs, Elasticsearch or OpenSearch with Kibana, or Loki. I prefer structured logs with consistent fields and trace correlation.

### Traces

OpenTelemetry can instrument applications and export telemetry to a supported backend. Distributed traces help identify which service, database call, or external API contributes to latency or errors.

### Alerting and incident management

Alerting tools may include Alertmanager or cloud-native alerting, integrated with PagerDuty, Opsgenie, ServiceNow, Microsoft Teams, Slack, or another approved system.

### Observability model

```text
Application and platform telemetry
        |
        +--> Metrics --> Dashboards/SLOs --> Alerts
        |
        +--> Logs ------------------------> Investigation
        |
        +--> Traces ----------------------> Request analysis
        |
        +--> Events/Changes --------------> Correlation
```

In an interview, I would only name tools I have actually used and then explain one dashboard, alert, or incident where that tool helped.

---

## 13. Are you familiar with creating dashboards and alerts yourself?

### Interview-ready answer

Yes. I have created dashboards and alerts by first understanding the service architecture, customer journey, expected behavior, ownership, and operational risks. I do not begin with all available metrics. I begin with the decisions the dashboard or alert must support.

### Dashboard process

1. Identify the audience: developer, on-call engineer, service owner, or management.
2. Define customer-facing service indicators.
3. Add golden-signal metrics.
4. Add dependency and infrastructure drill-down panels.
5. Add dimensions such as environment, region, cluster, namespace, and version.
6. Add deployment annotations.
7. Test the dashboard during normal conditions and incidents.
8. Review performance and label cardinality.

### Alert process

1. Identify a user-impacting or urgent condition.
2. Select a reliable metric or SLI.
3. Choose a meaningful threshold and duration.
4. Define warning, ticket, and paging severity.
5. Add ownership, runbook, dashboard, and log links.
6. Configure routing and deduplication.
7. Test notification delivery.
8. Review false positives and missed incidents.

I separate dashboards from alerts. A dashboard can contain broad diagnostic detail, but a page should indicate a condition requiring timely human action.

---

## 14. What types of dashboards and alerts have you created?

### Interview-ready answer

I build dashboards in layers so an engineer can move from service impact to the affected workload or dependency.

### Service overview dashboard

- Availability or successful-request ratio
- Requests per second
- p50, p95, and p99 latency
- HTTP 4xx and 5xx rate
- Top failing endpoints
- SLO status and remaining error budget
- Current deployed version

### Kubernetes dashboard

- Desired, available, and unavailable replicas
- Pod phase and readiness
- Container restart count
- CPU and memory usage versus requests and limits
- OOM kills and evictions
- Node readiness and capacity
- Deployment rollout status
- HPA current and desired replicas

### Infrastructure dashboard

- Compute utilization
- Network throughput and errors
- Disk or volume usage
- Load-balancer requests and backend health
- Cluster capacity
- Quota usage

### Database dashboard

- Connection count and pool saturation
- Query latency
- Error and timeout rates
- CPU and memory
- Storage utilization
- Replication lag
- Lock or deadlock behavior
- Backup status

### Typical alerts

- Availability below target
- Rapid or sustained error-budget burn
- High 5xx ratio
- Sustained p95/p99 latency
- No traffic when traffic is expected
- Deployment unavailable replicas
- Pod crash loops or frequent restarts
- OOM kills
- Node not ready
- High memory, CPU, disk, or ephemeral-storage pressure
- PVC approaching capacity
- Database connection saturation or replication failure
- Queue backlog or consumer lag
- Certificate expiry
- Backup failure

A good alert message includes service, environment, observed value, threshold, duration, severity, owner, dashboard, and runbook.

---

## 15. Have you used SRE golden signals for dashboarding and alerting?

### Interview-ready answer

Yes. I use the four golden signals as the foundation of service dashboards: latency, traffic, errors, and saturation. They provide a balanced view of user experience and system capacity.

### Latency

Latency shows how long requests take. I prefer percentiles because averages can hide slow requests.

- p50 represents a typical request.
- p95 and p99 show tail latency.
- Successful and failed requests may be separated because failed requests can complete quickly and make latency appear healthy.

### Traffic

Traffic measures demand on the service, such as:

- Requests per second
- Transactions per minute
- Messages published or consumed
- Concurrent sessions
- Bytes transferred

### Errors

Errors may include:

- HTTP 5xx responses
- Failed business transactions
- Exceptions
- Dependency timeouts
- Queue-processing failures
- Invalid responses

I define errors from the customer's perspective rather than only from infrastructure status.

### Saturation

Saturation indicates how close a constrained resource is to its limit:

- CPU throttling
- Memory usage
- Thread-pool saturation
- Database connection-pool utilization
- Queue depth
- Disk utilization
- Network or API quota usage

Not every golden-signal panel should become a page. I page on actionable user impact or a condition likely to cause imminent impact.

---

## 16. Have you implemented SLO-based monitoring?

### Interview-ready answer

Yes. I start with the customer journey, define a measurable SLI, agree on an SLO target and window, calculate the error budget, and then create dashboards and burn-rate alerts.

### Example availability SLI

```text
SLI = Successful eligible requests / Total eligible requests
```

A request may be considered successful when it returns a valid response within an agreed latency. Health checks, load tests, and explicitly excluded traffic should be handled according to the SLI specification.

### Example SLO

```text
99.9% of eligible API requests should succeed
over a rolling 30-day window.
```

### Implementation process

1. Identify the service and owner.
2. Define what a customer considers successful.
3. Select the telemetry source.
4. Define eligible and excluded events.
5. Set the objective and compliance window.
6. Validate historical performance.
7. Create SLO and error-budget dashboards.
8. Configure fast-burn and slow-burn alerts.
9. Define an error-budget policy.
10. Review the SLO as the service changes.

SLO-based monitoring is useful because it connects engineering signals to customer impact and provides a shared reliability target.

---

## 17. What is an error budget?

### Interview-ready answer

An error budget is the amount of unreliability permitted while still meeting the service-level objective.

```text
Error budget percentage = 100% - SLO percentage
```

For a 99.9% SLO:

```text
Error budget = 100% - 99.9% = 0.1%
```

For a time-based interpretation over 30 days:

```text
30 days x 24 hours x 60 minutes = 43,200 minutes
43,200 x 0.001 = 43.2 minutes
```

This means approximately 43.2 minutes of allowed unavailability, provided that the SLI is directly time-based. A request-based SLO uses failed eligible events rather than simply downtime.

### Why it matters

An error budget provides a measurable balance between delivery speed and reliability:

- If the service is comfortably within budget, normal feature delivery can continue.
- If the budget is being consumed too quickly, reliability work receives greater priority.
- If the budget is exhausted, a predefined policy may restrict risky changes until reliability improves.

The policy should not be used to blame individuals. It is a decision-making mechanism agreed by development, SRE, product, and business stakeholders.

---

## 18. Have you configured SLO-based alerting?

### Interview-ready answer

Yes. I use error-budget burn-rate alerting so that the team is notified when the service consumes its budget too quickly. I use multiple windows because a single threshold often either reacts too slowly to a severe outage or creates noise from short spikes.

### Burn-rate concept

```text
Burn rate = Observed bad-event ratio / Allowed bad-event ratio
```

For a 99.9% SLO, the allowed bad-event ratio is:

```text
1 - 0.999 = 0.001
```

If the measured error ratio is 1%:

```text
Burn rate = 0.01 / 0.001 = 10
```

At that moment, the budget is being consumed ten times faster than the sustainable rate.

### Conceptual PromQL

```promql
(
  sum(rate(http_requests_total{service="order",status=~"5.."}[1h]))
/
  sum(rate(http_requests_total{service="order"}[1h]))
)
/
(1 - 0.999)
```

The production expression must correctly define eligible traffic, success criteria, low-traffic behavior, and missing telemetry.

### Multi-window approach

- A short and long pair can identify a severe fast burn and page the on-call engineer.
- Longer windows can identify gradual degradation and create a ticket or lower-severity notification.
- Both windows should usually agree before an alert fires, reducing noise from brief spikes.

I include the SLO, current burn, affected service, remaining budget, dashboard, and runbook in the alert.

---

## 19. What is an ideal burn rate?

### Interview-ready answer

There is no single alert threshold that is ideal for every service. However, the interpretation is straightforward:

- **Burn rate 0:** No error budget is currently being consumed.
- **Burn rate below 1:** The service is consuming budget more slowly than the sustainable rate.
- **Burn rate 1:** The service is consuming budget at exactly the sustainable rate.
- **Burn rate above 1:** If sustained, the service will exhaust its budget before the compliance period ends.

Operationally, I want the long-term burn rate below 1. I do not necessarily page immediately when it briefly crosses 1. Paging thresholds depend on how much budget a failure would consume and how quickly the team must respond.

### Time-to-exhaustion intuition

If a 30-day error budget burns continuously at:

```text
Burn rate 1   -> approximately 30 days to consume the budget
Burn rate 2   -> approximately 15 days
Burn rate 10  -> approximately 3 days
Burn rate 30  -> approximately 1 day
```

This is a simplified explanation assuming the rate stays constant. Real alert design uses multiple windows and organization-specific policies.

---

## 20. How do you use Terraform to deploy cluster nodes?

### Interview-ready answer

For managed Kubernetes, I use Terraform to provision the cluster, node pools, networking, IAM, service accounts, encryption, logging, and autoscaling. I prefer a reusable module and separate environment inputs.

### GKE node-pool example

```hcl
resource "google_container_cluster" "primary" {
  name     = var.cluster_name
  location = var.region

  network    = var.network
  subnetwork = var.subnetwork

  remove_default_node_pool = true
  initial_node_count       = 1

  release_channel {
    channel = "REGULAR"
  }
}

resource "google_service_account" "gke_nodes" {
  account_id   = "${var.cluster_name}-nodes"
  display_name = "GKE node service account"
}

resource "google_container_node_pool" "application" {
  name     = "application-pool"
  cluster  = google_container_cluster.primary.name
  location = var.region

  autoscaling {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    machine_type    = var.machine_type
    disk_type       = "pd-balanced"
    disk_size_gb    = 100
    service_account = google_service_account.gke_nodes.email

    labels = {
      workload   = "application"
      environment = var.environment
    }

    tags = ["gke-application"]

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }
}
```

### Pipeline process

```bash
terraform fmt -check
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

### Production considerations

- Remote state and state locking where supported
- Separate state per environment or blast radius
- Peer review of the saved plan
- Narrow CI/CD permissions
- Version constraints for Terraform and providers
- Private networking where required
- Autoscaling and multiple zones
- Node taints and labels for workload isolation
- Maintenance windows and upgrade strategy
- Logging, monitoring, and security configuration

---

## 21. What do you write in `provider.tf`, `main.tf`, `variables.tf`, and other Terraform files?

### Interview-ready answer

Terraform loads all `.tf` files in a module together, so these names are organizational conventions. I separate files to make ownership and review easier.

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }

  backend "gcs" {
    bucket = "company-terraform-state"
    prefix = "gke/production"
  }
}
```

### `provider.tf`

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}
```

I do not hardcode credentials. Authentication should use an approved workload identity, federation, service account, or environment-based mechanism.

### `main.tf`

Contains resources, data sources, locals where appropriate, and module calls:

```hcl
module "gke" {
  source = "../../modules/gke"

  project_id   = var.project_id
  region       = var.region
  cluster_name = var.cluster_name
  network      = var.network
  subnetwork   = var.subnetwork
}
```

### `variables.tf`

```hcl
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "region" {
  type        = string
  description = "GCP region"
  default     = "asia-south1"

  validation {
    condition     = length(var.region) > 0
    error_message = "Region must not be empty."
  }
}
```

### `outputs.tf`

```hcl
output "cluster_name" {
  description = "Name of the GKE cluster"
  value       = module.gke.cluster_name
}
```

Sensitive outputs should be marked as sensitive, although this does not remove the need to protect state.

### `locals.tf`

```hcl
locals {
  common_labels = {
    environment = var.environment
    application = var.application
    managed_by  = "terraform"
  }
}
```

### Environment input

```hcl
# prod.tfvars
project_id   = "company-production"
region       = "asia-south1"
cluster_name = "production-gke"
environment  = "production"
```

```bash
terraform plan -var-file=prod.tfvars
```

---

## 22 and 23. Write Terraform code to create an AWS VPC and subnet

### Interview-ready answer

The following example creates a VPC, one public subnet, an internet gateway, a public route table, and a route-table association. A subnet becomes publicly routable when its route table has a default route to an internet gateway. Public IPv4 assignment is also enabled for resources launched into this subnet.

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Application = var.application
      ManagedBy   = "Terraform"
    }
  }
}

variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "application" {
  type    = string
  default = "interview-demo"
}

variable "vpc_cidr" {
  type    = string
  default = "10.10.0.0/16"
}

variable "public_subnet_cidr" {
  type    = string
  default = "10.10.1.0/24"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.environment}-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-subnet-a"
    Tier = "public"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-internet-gateway"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.environment}-public-route-table"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}
```

### Commands

```bash
terraform fmt
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

### Private-subnet extension

A private subnet would normally use a separate route table. If private workloads require outbound internet access, their default route can point to a NAT gateway in a public subnet. NAT gateways, routes, and high-availability design should be evaluated for cost and resilience.

### Production improvements

For Production, I would normally add:

- Multiple Availability Zones
- Public and private subnets per zone
- NAT design based on availability and cost requirements
- VPC flow logs
- Network ACL and security-group design
- VPC endpoints for selected AWS services
- Centralized tagging
- Reusable modules
- Remote state and controlled deployment

---

## 24. How have you optimized Kubernetes deployment configurations?

### Interview-ready answer

I optimize Kubernetes deployments by reviewing actual utilization, application startup behavior, rollout failures, and availability requirements. The objective is to improve reliability and efficiency without hiding application problems through excessive resources or aggressive restarts.

### Resource requests and limits

Requests influence scheduling and limits provide an upper boundary. I compare historical CPU and memory usage with configured values and load-test results.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

Very low memory limits can cause OOM kills. Very high requests waste cluster capacity and may leave pods Pending.

### Health probes

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: http
  periodSeconds: 5
  failureThreshold: 30

readinessProbe:
  httpGet:
    path: /health/ready
    port: http
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health/live
    port: http
  periodSeconds: 20
  failureThreshold: 3
```

- Startup probes protect slow-starting applications from premature liveness failures.
- Readiness controls whether a pod receives traffic.
- Liveness detects a process that cannot recover without restart.

A liveness probe should not fail simply because an external dependency is temporarily unavailable, otherwise it can create a restart storm.

### Rolling-update strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1

minReadySeconds: 15
progressDeadlineSeconds: 600
```

### Graceful termination

```yaml
terminationGracePeriodSeconds: 60

containers:
  - name: application
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```

The application should stop accepting new work and finish or safely return in-flight requests before termination.

### Availability and scheduling

I may add:

- Horizontal Pod Autoscaler
- PodDisruptionBudget
- Topology spread constraints
- Pod anti-affinity
- Priority classes where justified
- Node affinity, tolerations, and dedicated node pools

### Security optimization

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault

containers:
  - name: application
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

### Measuring improvement

I evaluate:

- Pod restart and OOM frequency
- Deployment success and rollback rate
- Time to readiness
- Unavailable replicas during rollout
- CPU and memory efficiency
- Pending-pod frequency
- p95/p99 latency and error rate
- Availability during node maintenance

---

## 25. How did you build a blameless postmortem framework and prevent recurring incidents?

### Interview-ready answer

I introduced a standardized blameless postmortem process that focuses on system conditions and decision context rather than individual blame. The purpose is to learn why the safeguards, processes, or architecture allowed an incident to occur and to convert those lessons into tracked engineering work.

### Postmortem template

```text
1. Incident title and date
2. Severity and duration
3. Executive summary
4. Customer and business impact
5. Detection method
6. Detailed timeline
7. Technical root cause
8. Contributing factors
9. What went well
10. What did not go well
11. Where we were fortunate
12. Corrective and preventive actions
13. Owners and target dates
14. Supporting dashboards, logs, and changes
```

### Blameless principles

Instead of writing:

> An engineer deployed the wrong configuration.

I frame the investigation as:

> The deployment process allowed an incompatible configuration to reach Production because schema validation and an approval check were absent.

This approach does not remove accountability. It improves accuracy by identifying the system conditions that made an error possible and likely to recur.

### Action categories

- **Prevention:** Validation, testing, architecture, or policy changes
- **Detection:** Better SLIs, alerts, logs, synthetic checks, or change correlation
- **Mitigation:** Automated rollback, failover, circuit breaker, or feature flag
- **Recovery:** Better runbooks, tools, backups, or access procedures

### Preventing recurrence

I track every action with:

- A specific deliverable
- An owner
- Priority
- Due date
- Verification criteria
- Status review

I also classify incidents by cause and review trends. If several incidents involve certificate expiry, resource exhaustion, or configuration drift, that pattern becomes a platform-level improvement instead of several isolated fixes.

### Strong interview closing

> The framework improved continuous learning because postmortems were no longer just documents. Actions were assigned, reviewed, and validated. Repeated patterns were used to prioritize automation, platform safeguards, and architectural improvements.

Use actual reduction percentages only if you have reliable records and can explain how the measurement was calculated.

---

## 26. What automation have you implemented to reduce MTTR?

### Interview-ready answer

I reduce mean time to recovery by automating the repetitive work between alert detection and safe mitigation. The goal is to give the on-call engineer immediate context and approved recovery actions, not to automate risky actions without safeguards.

### Alert enrichment

Instead of a generic message such as `High error rate`, an enriched alert can include:

- Service and environment
- Current error rate and threshold
- Affected region, cluster, and namespace
- Current application version
- Last deployment details
- Dashboard and log links
- Runbook
- Service owner and escalation route

This reduces time spent locating the affected workload.

### Automated Kubernetes diagnostic collection

A controlled script or function can collect:

```bash
kubectl get pod -n "$NAMESPACE" -o wide
kubectl describe pod "$POD" -n "$NAMESPACE"
kubectl logs "$POD" -n "$NAMESPACE" --all-containers=true
kubectl logs "$POD" -n "$NAMESPACE" --previous
kubectl get events -n "$NAMESPACE" --sort-by=.lastTimestamp
kubectl top pod "$POD" -n "$NAMESPACE" --containers
```

The output can be attached to the incident while applying access control and redacting sensitive information.

### Deployment safety automation

I can reduce recovery time through:

- Automated smoke tests
- Deployment health gates
- Rollout timeout
- Helm atomic rollback
- Canary analysis
- Automatic stop or rollback when critical service indicators degrade

### Self-healing and preventive automation

Examples include:

- Health probes and controller-based pod replacement
- Horizontal and cluster autoscaling
- Automated certificate renewal
- Disk and log-retention controls
- Backup completion and restore verification
- Safe node auto-repair
- Queue-consumer scaling
- Expiry alerts for secrets and certificates

### ChatOps and runbooks

For approved operations, a ChatOps command or runbook automation can:

- Retrieve service health
- Display recent deployments
- Scale a workload within limits
- Initiate an approved rollback
- Create an incident channel
- Notify stakeholders
- Update incident status

Sensitive actions must require authorization, audit logging, input validation, and rollback capability.

### Example interview story structure

> During incidents, the team previously spent significant time identifying the cluster, namespace, pod, application version, and recent deployment. I automated the initial diagnostic collection and enriched alerts with the relevant dashboards, logs, ownership, and runbook. For deployment-related failures, the pipeline could stop the rollout or perform an approved rollback. This shortened triage and allowed the engineer to start mitigation earlier.

When explaining results, I use genuine before-and-after data, such as median acknowledgement time, median recovery time, or percentage of incidents automatically enriched.

---

# Rapid Revision Notes

## Terraform import

```bash
terraform import <resource-address> <resource-id>
terraform state show <resource-address>
terraform plan
```

## Kubernetes triage

```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --all-containers=true
kubectl logs <pod> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

## Helm deployment

```bash
helm lint ./chart
helm template release ./chart -f values-prod.yaml
helm upgrade --install release ./chart \
  -n production \
  -f values-prod.yaml \
  --atomic --wait
```

## SRE golden signals

```text
Latency, Traffic, Errors, Saturation
```

## Error budget

```text
Error budget = 1 - SLO
```

## Burn rate

```text
Burn rate = Observed bad-event ratio / Allowed bad-event ratio
```

---

# Interview Delivery Tips

1. Start with a direct answer in one or two sentences.
2. Explain your process in logical steps.
3. Give one real project example.
4. Mention safety, validation, rollback, and monitoring.
5. Finish with a measurable outcome if you have verified data.
6. Do not list tools without explaining how you used them.
7. If you have not implemented something directly, state that you understand the approach and explain how you would implement it.
8. Keep Production credentials, customer names, URLs, IP addresses, and sensitive architecture details confidential.

---

# Suggested Reference Documentation

- HashiCorp Terraform import documentation: <https://developer.hashicorp.com/terraform/cli/commands/import>
- Terraform import workflow: <https://developer.hashicorp.com/terraform/cli/import/usage>
- Google Cloud Terraform resource import: <https://cloud.google.com/docs/terraform/resource-management/import>
- Kubernetes pod debugging: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/>
- Kubernetes running-pod debugging: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/>
- Helm chart structure: <https://helm.sh/docs/topics/charts/>
- Helm values best practices: <https://helm.sh/docs/chart_best_practices/values/>
- Google SRE Workbook, Alerting on SLOs: <https://sre.google/workbook/alerting-on-slos/>
- Google SRE Workbook, Error Budget Policy: <https://sre.google/workbook/error-budget-policy/>

---

> **Final note:** Treat every sample as a framework. The strongest interview response combines a correct technical explanation with one concise situation from your own experience, the action you performed, and a verified result.
