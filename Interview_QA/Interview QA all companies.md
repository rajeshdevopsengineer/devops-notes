Below are senior-level, interview-ready answers. Use the architecture/tools that match your actual experience; don’t claim a technology you haven’t worked with.

1. **What is the difference between Docker and Kubernetes?**

**Answer:**

Docker is primarily a containerization platform. I use it to package an application along with its dependencies into a container image and run that container consistently across environments.

Kubernetes is a container orchestration platform. It manages containers at scale across multiple worker nodes and handles scheduling, service discovery, load balancing, self-healing, autoscaling, rolling deployments, configuration, and secrets.

For example, with Docker I can build and run an application:

```bash
docker build -t myapp:v1 .
docker run -p 8080:8080 myapp:v1
```

In Kubernetes, I deploy that image using a Deployment:

```text
Developer
   |
   v
Docker Image
   |
   v
Container Registry
   |
   v
Kubernetes Deployment
   |
   +---- Pod 1
   +---- Pod 2
   +---- Pod 3
         |
       Service
         |
       Ingress
         |
       Users
```

So I normally summarize it as:

**Docker packages and runs containers; Kubernetes manages and orchestrates containers at production scale.**

---

2. **How do you reduce downtime during deployments?**

**Answer:**

My first preference is to design deployments so that they are effectively zero-downtime.

In Kubernetes, I typically use **rolling deployments** with multiple replicas, readiness probes, PodDisruptionBudgets, and controlled rollout parameters.

For example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

`maxUnavailable: 0` ensures Kubernetes doesn't intentionally remove an existing available pod before a new pod becomes ready.

I also configure:

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
```

The application doesn't receive production traffic until the readiness check passes.

For critical applications, I have additional options such as:

* Blue/green deployments
* Canary deployments
* Feature flags
* Automated rollback
* Database backward-compatible changes
* Pre-deployment validation
* Graceful shutdown using `terminationGracePeriodSeconds`
* `preStop` hooks
* PodDisruptionBudgets
* Multiple replicas across Availability Zones

For example, with a canary release I may send 5% of traffic to the new version, monitor latency, error rate and business KPIs, then gradually move to 25%, 50% and finally 100%.

If the error rate increases, the deployment gets rolled back.

A strong principle is:

**Deployment success doesn't mean the new pods are running; it means the new version is healthy from the user's perspective.**

---

3. **What agents have you deployed?**

**Answer:**

This depends on the organization's monitoring and security stack. A good example from an enterprise Kubernetes/cloud environment would be:

**Monitoring agents:** Prometheus Node Exporter, kube-state-metrics, Datadog Agent or New Relic infrastructure agent.

**Logging agents:** Fluent Bit, Fluentd, Filebeat or Vector.

**APM agents:** Datadog APM, New Relic, Dynatrace OneAgent or OpenTelemetry Collector.

**Security agents:** CrowdStrike Falcon, Qualys, Prisma Cloud/Defender agents depending on company requirements.

In Kubernetes, infrastructure/logging agents are commonly deployed as a **DaemonSet**, because we usually need one instance running on every worker node.

Example:

```text
Kubernetes Worker Node 1
   |- Application Pods
   |- Fluent Bit
   |- Monitoring Agent

Kubernetes Worker Node 2
   |- Application Pods
   |- Fluent Bit
   |- Monitoring Agent

Kubernetes Worker Node 3
   |- Application Pods
   |- Fluent Bit
   |- Monitoring Agent
```

For modern environments, I also prefer **OpenTelemetry** because it provides a vendor-neutral approach for metrics, logs and traces.

---

4. **Suppose there are 100 applications. How do you perform log analysis?**

**Answer:**

With 100 applications, I would never troubleshoot by manually logging into individual servers or pods.

I would implement **centralized logging**.

For example:

```text
100 Applications
       |
       v
stdout/stderr
       |
       v
Fluent Bit / OpenTelemetry
       |
       v
Kafka - optional for buffering
       |
       v
Elasticsearch / OpenSearch / Splunk / Datadog
       |
       v
Dashboards + Alerts
```

Every application should produce structured JSON logs containing fields such as:

```json
{
  "timestamp": "2026-09-14T10:20:31Z",
  "service": "payment-service",
  "environment": "production",
  "pod": "payment-service-xyz",
  "namespace": "payments",
  "level": "ERROR",
  "trace_id": "b92131",
  "request_id": "req-78121",
  "message": "Payment gateway timeout"
}
```

The most important fields for troubleshooting are usually:

`service`, `environment`, `namespace`, `pod`, `level`, `trace_id`, `request_id`, `customer/request identifier`, and timestamp.

If a customer says a transaction failed at 10:20 AM, I can search by request ID or trace ID and follow the request across microservices.

For example:

```text
API Gateway
 trace_id=123
    |
    v
Order Service
 trace_id=123
    |
    v
Payment Service   <-- timeout
 trace_id=123
    |
    v
External Payment Gateway
```

That is why I try to correlate **logs + metrics + distributed traces**, rather than depending only on logs.

---

5. **What is the difference between SRE and DevOps?**

**Answer:**

DevOps is primarily a **culture and operating model** that improves collaboration between development and operations and emphasizes automation, CI/CD and shared ownership.

SRE is a more specific engineering discipline for operating reliable systems using software engineering practices.

I usually explain it as:

| DevOps                        | SRE                                          |
| ----------------------------- | -------------------------------------------- |
| Culture/philosophy            | Engineering discipline                       |
| Dev + Ops collaboration       | Reliability engineering                      |
| CI/CD and automation          | SLI/SLO/error budgets                        |
| Faster software delivery      | Reliability and availability                 |
| Infrastructure automation     | Capacity, resilience and incident management |
| Broad organizational approach | Quantitative reliability practices           |

Google famously described SRE as roughly what happens when you ask a software engineer to design an operations function.

As an SRE, I focus on things like:

```text
Availability
Latency
Error rate
Traffic
Capacity
MTTR
Automation
Observability
Incident response
SLO compliance
```

DevOps and SRE are therefore complementary rather than competing concepts.

---

6. **Have you done an on-premises-to-cloud migration? What challenges did you face?**

A strong interview answer would be:

**Answer:**

Yes. A typical migration I worked on involved moving workloads from an on-premises environment into AWS/Azure using a phased migration approach rather than performing one big cutover.

First, we performed application discovery and classified workloads based on dependencies and migration strategy:

```text
Applications
   |
   +--> Rehost
   +--> Replatform
   +--> Refactor
   +--> Retain
   +--> Retire
```

We then established the cloud foundation:

```text
Cloud Organization / Accounts
          |
          v
Landing Zone
          |
  +-------+-------+
  |               |
Network          IAM
  |               |
VPC/VNet        Roles
Subnets         Policies
Routes          SSO
VPN/DX
```

Some major challenges were **network connectivity** because applications depended on on-premise databases, LDAP, APIs and shared services; **dependency discovery**, because many legacy dependencies were not documented; **database migration**, where we needed replication and controlled cutover to minimize downtime; **security/IAM mapping**, because on-prem permissions didn't directly map to cloud IAM; and **DNS/certificate/firewall changes** during cutover.

Other common challenges included latency between hybrid systems, IP address changes, data transfer volume, application hard-coded configuration, cloud cost optimization, monitoring changes and rollback planning.

To reduce migration risk, I prefer:

```text
Assessment
   ↓
Landing Zone
   ↓
Network Connectivity
   ↓
Pilot Application
   ↓
Migration Waves
   ↓
Validation
   ↓
DNS Cutover
   ↓
Monitoring
   ↓
Decommission On-Prem
```

I always make sure there is a rollback strategy before the production cutover.

---

7. **How does endpoint authentication work in Kubernetes?**

There are two possible meanings to this interview question.

If they're asking about the **Kubernetes API endpoint**, the API request normally goes through:

```text
User / Service Account
        |
        v
Kubernetes API Server
        |
        v
Authentication
        |
        v
Authorization
        |
        v
Admission Control
        |
        v
Kubernetes Resource
```

Authentication establishes **who you are**.

Common mechanisms include client certificates, bearer tokens, ServiceAccount tokens and OIDC integration with identity providers.

Authorization determines **what you're allowed to do**.

The most commonly used mechanism is Kubernetes **RBAC**:

```text
User / ServiceAccount
       |
   RoleBinding
       |
      Role
       |
get/list/create/delete pods
```

For example:

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

That user can view pods but cannot delete them.

For an **application endpoint**, authentication usually happens through components such as:

```text
Client
  |
 HTTPS
  |
Load Balancer
  |
Ingress / API Gateway
  |
OAuth2 / OIDC / JWT / mTLS
  |
Kubernetes Service
  |
Application Pods
```

For service-to-service security, mTLS can also be provided using a service mesh such as Istio.

---

8. **How do you troubleshoot a pod in CrashLoopBackOff?**

**Answer:**

`CrashLoopBackOff` means the container starts, crashes and Kubernetes repeatedly tries to restart it with increasing backoff intervals.

I troubleshoot it systematically.

First:

```bash
kubectl get pods -n <namespace>
```

Then:

```bash
kubectl describe pod <pod> -n <namespace>
```

I'm looking for events such as:

```text
OOMKilled
FailedMount
ImagePullBackOff
Probe failed
Permission denied
ConfigMap missing
Secret missing
```

Then I inspect logs:

```bash
kubectl logs <pod> -n <namespace>
```

If the container has already restarted:

```bash
kubectl logs <pod> -n <namespace> --previous
```

`--previous` is particularly important for CrashLoopBackOff.

Then I check:

```bash
kubectl get pod <pod> \
-o jsonpath='{.status.containerStatuses[*].lastState}'
```

Common causes I've seen include application exceptions, incorrect environment variables, missing Secrets/ConfigMaps, OOMKilled containers, bad startup commands, failed database connectivity, permission problems and liveness probe failures.

For OOM issues:

```bash
kubectl describe pod <pod>
```

may show:

```text
Reason: OOMKilled
Exit Code: 137
```

Then I compare actual memory utilization with Kubernetes resource requests/limits.

My investigation normally follows:

```text
Pod status
   ↓
Events
   ↓
Current logs
   ↓
Previous logs
   ↓
Exit code
   ↓
Resources
   ↓
ConfigMap/Secrets
   ↓
Health probes
   ↓
Network/dependencies
   ↓
Recent deployment changes
```

I also ask **what changed immediately before the incident** because deployment/configuration changes are often the fastest path to the root cause.

---

9. **What are SLA and SLO?**

I would actually explain three terms: **SLI, SLO and SLA**.

**SLI — Service Level Indicator**

A measurement of service reliability.

Examples:

```text
Availability = 99.96%
P95 latency = 180ms
Successful requests = 99.95%
```

**SLO — Service Level Objective**

The internal reliability target.

Example:

```text
99.9% availability per month
```

**SLA — Service Level Agreement**

A contractual commitment made to the customer, often including financial/service consequences if it is violated.

For example:

```text
SLA = 99.9% availability
```

For a 30-day month, 99.9% availability allows roughly:

```text
43 minutes of downtime
```

I also use the concept of an **error budget**.

If the SLO is 99.9%, then:

```text
Error budget = 100% - 99.9%
             = 0.1%
```

Teams can use that error budget to balance feature velocity against reliability.

---

10. **Provide an agent you worked with for a customer or your company.**

This question is usually asking for a concrete example rather than a long list.

You can answer:

**Answer:**

One example was **Fluent Bit**, which we deployed as a DaemonSet across Kubernetes worker nodes for centralized application logging.

Architecture:

```text
Application Pods
      |
      | stdout/stderr
      v
Container Runtime Logs
      |
      v
Fluent Bit DaemonSet
      |
      v
Elasticsearch / OpenSearch
      |
      v
Kibana
```

We added Kubernetes metadata such as:

```text
cluster
namespace
pod
container
application
environment
```

That allowed support and engineering teams to search millions of log records centrally instead of accessing individual pods.

Another example could be a Datadog/New Relic/Dynatrace agent for infrastructure and APM monitoring.

**Important for the interview:** choose the agent you have actually deployed and then explain **why it was deployed, how you deployed it and where its data went**.

---

11. **Explain the cloud architecture you have worked on.**

Here's a strong AWS/Kubernetes example:

```text
                         Internet
                            |
                         Route 53
                            |
                       CloudFront/WAF
                            |
                            v
                     Application LB
                            |
                            v
                    Ingress Controller
                            |
                 +----------+----------+
                 |                     |
              EKS AZ-1              EKS AZ-2
                 |                     |
             App Pods              App Pods
                 |                     |
                 +----------+----------+
                            |
                     Internal Services
                            |
                  +---------+---------+
                  |                   |
                RDS                ElastiCache
            Multi-AZ DB               Redis
                  |
                 S3
```

At the infrastructure level:

```text
AWS Organization
       |
       +-- Development Account
       +-- QA Account
       +-- Production Account
       +-- Security Account
       +-- Shared Services Account
```

For networking:

```text
VPC
 |
 +-- Public Subnets
 |      |
 |     ALB / NAT Gateway
 |
 +-- Private App Subnets
 |      |
 |     EKS Worker Nodes
 |
 +-- Private DB Subnets
        |
       RDS
```

For CI/CD:

```text
Developer
   |
   v
GitHub / GitLab
   |
   v
CI Pipeline
   |
   +--> Unit tests
   +--> Security scan
   +--> Docker build
   +--> Image scan
   |
   v
ECR
   |
   v
Argo CD / Jenkins
   |
   v
EKS
```

For observability:

```text
Applications
     |
 +---+-------------------+
 |                       |
Metrics                 Logs
 |                       |
Prometheus             Fluent Bit
 |                       |
Grafana               OpenSearch/Splunk
 |
Alertmanager
 |
PagerDuty/Slack
```

And infrastructure is preferably managed through **Terraform**, with configuration/deployment handled through Helm, Argo CD or an equivalent GitOps approach.

The important part in an interview isn't naming twenty AWS services. Explain **why the architecture was designed that way: availability, scalability, security, disaster recovery, observability and cost.**

---

12. **Explain a production issue you faced.**

This is one of the most important SRE interview questions. Use the **STAR format: Situation, Task, Action, Result.**

Here's a strong example:

**Situation**

We had a production Kubernetes application where customers started reporting intermittent HTTP 500 errors shortly after a new release.

Monitoring showed:

```text
HTTP 5xx ↑
Latency ↑
Pod restarts ↑
```

**Investigation**

First I checked Kubernetes:

```bash
kubectl get pods -n production
```

Several application pods showed:

```text
CrashLoopBackOff
```

I checked:

```bash
kubectl describe pod <pod>
```

and found:

```text
Reason: OOMKilled
Exit Code: 137
```

I then correlated this with memory metrics in Grafana and observed that memory usage increased significantly after the latest deployment.

We compared the previous and current versions and identified a new application component causing increased memory consumption.

**Immediate mitigation**

Because customer impact was continuing, instead of trying to debug the application while production was degraded, we rolled back:

```bash
kubectl rollout undo deployment/<application>
```

The previous version came up successfully and the error rate immediately started declining.

We validated:

```text
5xx rate → normal
Latency → normal
Pod restart count → stable
Application health → normal
```

**Root cause**

The new application version had significantly higher memory consumption while the Kubernetes container memory limit had remained unchanged.

When the container crossed the limit, Linux cgroups terminated it, resulting in `OOMKilled`, Kubernetes restarted the container and customers temporarily received failed requests.

**Permanent corrective actions**

We corrected the application's memory issue, reviewed resource requests and limits, added memory-utilization alerts, added load/performance testing to the CI/CD process, improved deployment monitoring and introduced canary releases for high-risk services.

The strongest part of this answer is explaining both:

```text
How did you restore service quickly?
                +
How did you prevent recurrence?
```

That's what separates a senior SRE answer from simply saying, *"I restarted the pod."*

### The senior-engineer mindset interviewers are looking for

Across these questions, try to demonstrate a consistent troubleshooting approach:

```text
Observe
  ↓
Understand customer impact
  ↓
Check recent changes
  ↓
Use metrics/logs/traces
  ↓
Form a hypothesis
  ↓
Validate with evidence
  ↓
Mitigate customer impact
  ↓
Find root cause
  ↓
Implement permanent fix
  ↓
Add monitoring / automation
  ↓
Document through RCA/postmortem
```

Avoid answers such as **"I restart the pod/server and check"** as your main troubleshooting strategy. A senior DevOps/SRE engineer should show that they understand **why the failure occurred, how to minimize customer impact, and how to prevent it from happening again**.
