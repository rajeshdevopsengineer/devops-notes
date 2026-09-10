This responsibility means owning the platform across its lifecycle: choosing an architecture that can tolerate failures, implementing it consistently, and keeping applications available during incidents, scaling, and upgrades.

For interview preparation, study it through this practical scenario: **an online ordering application runs on AKS and must continue serving customers when a pod, node, or availability zone fails.**

**1. Design the platform around business requirements**

Start by defining what “highly available” means for the application.

| Requirement       | Example                                                     | Why it matters                                 |
| ----------------- | ----------------------------------------------------------- | ---------------------------------------------- |
| Availability SLO  | 99.9% successful checkout requests over 30 days             | Establishes a measurable reliability target    |
| Latency SLO       | 99% of checkout requests complete within 800 ms             | Prevents a slow service from appearing healthy |
| Peak demand       | 3,000 requests per second                                   | Determines capacity and scaling requirements   |
| Failure tolerance | Continue operating after one zone fails                     | Determines placement and spare capacity        |
| RTO               | Restore service within 30 minutes after a regional disaster | Guides the recovery architecture               |
| RPO               | Lose no more than five minutes of recoverable data          | Guides replication and backup design           |

These are illustrative targets. Agree on them with application and business owners before selecting infrastructure.

Also identify critical dependencies. An available AKS cluster does not make checkout available if its database, DNS, identity service, or payment provider is unavailable.

**2. Choose failure boundaries deliberately**

Design for several different failure scopes:

| Failure                           | Typical design response                                      |
| --------------------------------- | ------------------------------------------------------------ |
| Application process crashes       | Restart the container and investigate the cause              |
| Pod becomes unhealthy             | Stop sending it traffic and create a replacement             |
| Worker node fails                 | Use surviving replicas and reschedule affected pods          |
| Availability zone fails           | Serve traffic from replicas and capacity in other zones      |
| Application release is defective  | Stop promotion and restore a compatible version              |
| Database becomes unavailable      | Use the database’s tested availability or recovery mechanism |
| Entire region becomes unavailable | Execute a coordinated regional recovery plan                 |

The key principle is **redundancy plus usable failover capacity**. Having replicas in three zones is insufficient if the remaining two zones cannot handle the workload.

**3. Build the Azure foundation**

For the ordering project, your infrastructure design should address:

* **Network organization:** Plan VNets, subnets, address ranges, routing, private endpoints, and DNS. Reserve room for growth and temporary upgrade capacity.
* **Traffic entry:** Choose load balancing and ingress that fit HTTP routing, TLS, WAF, and backend-reachability requirements.
* **Identity:** Separate human access, infrastructure automation, cluster operations, and application identities.
* **Dependencies:** Select appropriate availability, backup, and recovery configurations for databases, storage, caches, and messaging.
* **Governance:** Define ownership, tagging, access boundaries, policies, and operational responsibilities.

**Practical scenario:** AKS tries to add nodes during a sale, but the subnet has insufficient free addresses. The application becomes overloaded even though autoscaling is configured correctly.

Your design should therefore treat IP addresses, subscription quotas, and regional compute capacity as scaling prerequisites.

**4. Design Kubernetes workloads for availability**

Each application needs more than a Deployment with several replicas.

| Kubernetes concept | Purpose                                           | Practical example                                                      |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------------------- |
| Replicas           | Provide multiple serving instances                | Run several checkout pods so one failure does not remove the service   |
| Topology spread    | Distribute replicas across failure domains        | Avoid placing every checkout pod on one node or in one zone            |
| Readiness probe    | Decide whether an instance should receive traffic | Keep a starting or temporarily unusable pod out of normal routing      |
| Liveness probe     | Detect a condition where restarting may help      | Restart a locally stuck application process                            |
| Startup probe      | Allow bounded initialization time                 | Give a slow-starting application time to initialize                    |
| Resource requests  | Inform scheduling and resource allocation         | Reserve an appropriate amount of CPU and memory per pod                |
| Resource limits    | Bound resource consumption where configured       | Prevent uncontrolled memory consumption from affecting other workloads |
| Disruption budget  | Limit eligible voluntary evictions                | Preserve enough replicas during maintenance                            |
| Graceful shutdown  | Allow work to finish during termination           | Drain checkout requests before a pod exits                             |

A disruption budget does **not** protect against every outage. It cannot prevent an unexpected zone failure, and deployment rollouts require their own availability controls.

**Practical scenario:** A liveness probe checks database connectivity. When the database slows down, all application pods restart repeatedly.

The improvement is to distinguish local application health from dependency health. Restarting every application instance usually does not repair a shared database failure.

**5. Design scaling across the entire request path**

There are several separate scaling problems:

* **Application replicas:** Can more pods handle additional requests?
* **Worker nodes:** Is there enough capacity to schedule those pods?
* **Dependencies:** Can the database, cache, broker, and external APIs handle additional concurrency?
* **Scaling delay:** Can existing capacity absorb demand while new instances become ready?

Consider a measured example:

* One checkout pod supports 150 requests per second at acceptable latency.
* Peak traffic is 3,000 requests per second.
* The simple minimum is `3,000 ÷ 150 = 20 pods`.

That minimum provides no throughput headroom at the measured operating point.

If you run 30 pods evenly across three zones, losing one zone leaves 20. You can theoretically handle the peak, but only if the measured throughput remains valid and shared dependencies continue working.

**Practical trap:** Each of those 30 pods allows 50 database connections. Together they can attempt 1,500 connections. If the database’s safe connection budget is 600, adding pods can make the incident worse.

Explain scaling as a system-wide capacity decision, not just an HPA setting.

**6. Build repeatably with infrastructure and delivery automation**

Use a controlled implementation process:

1. Define the Azure infrastructure in Terraform.
2. Review plans for replacements, permissions, exposure, and capacity changes.
3. Build and test application images through CI.
4. Promote an immutable artifact through environments.
5. Deploy the declared application configuration through the selected deployment controller.
6. Verify infrastructure behavior and the customer journey after changes.

**Practical scenario:** Staging works, but production fails because an engineer manually added a DNS configuration only in staging.

The durable improvement is to represent the required configuration in the owning infrastructure code, verify environment differences, and detect unexplained drift.

A successful Terraform apply or Kubernetes rollout is one verification step. Neither proves that a customer can place an order.

**7. Support the platform through observability and incident response**

Monitor customer outcomes and the underlying components.

| Layer                | Useful evidence                                             |
| -------------------- | ----------------------------------------------------------- |
| Customer journey     | Checkout success, latency, synthetic transactions           |
| Gateway and ingress  | Request failures, backend health, TLS and routing errors    |
| Application          | Exceptions, dependency calls, queueing, version changes     |
| Kubernetes           | Readiness, restarts, scheduling failures, node conditions   |
| Azure infrastructure | Quotas, resource health, network behavior, scaling failures |
| Dependencies         | Database connections, query latency, queue lag, throttling  |

During an incident, use a consistent sequence:

1. Confirm customer impact and scope.
2. Identify recent changes and affected dependencies.
3. Collect evidence that distinguishes competing hypotheses.
4. Apply the safest useful mitigation.
5. Verify recovery through the customer journey.
6. Record contributing causes and assign preventive improvements.

**Worked scenario:** Checkout returns intermittent errors after a deployment. Pods are Running, but some new replicas fail readiness.

Investigate the new version’s configuration, startup behavior, dependency access, and resource usage. Compare healthy and unhealthy replicas. If evidence points to the release, restore a compatible version through the owning deployment process and verify checkout success.

Restarting the entire cluster would introduce additional disruption without addressing the evidence.

**8. Maintain availability during upgrades and recovery**

Maintenance is part of availability engineering.

Before a cluster or node-pool upgrade, verify:

* Workload and API compatibility.
* Healthy replicas and appropriate placement.
* Disruption and rollout settings.
* Temporary node capacity, quota, and available IP addresses.
* Application startup and graceful shutdown behavior.
* A supported recovery approach.
* Customer-level monitoring during the operation.

For disaster recovery, test the complete service: infrastructure, application configuration, images, secrets, identity, data, dependencies, and routing.

**Practical scenario:** The recovery cluster starts successfully, but checkout fails because it cannot access the payment credential. This demonstrates that cluster recovery and service recovery are different milestones.

**How to explain this responsibility in an interview**

> “I start by defining availability, latency, capacity, and recovery objectives for the customer journey. I then identify failure domains and critical dependencies, design redundant Azure and AKS capacity, and configure workloads for healthy routing, graceful termination, and controlled scaling. I build the platform through reviewed infrastructure and deployment automation. Operationally, I use customer-level SLOs and component telemetry to detect failures, lead mitigation, and verify recovery. I also test upgrades, failure scenarios, and restores so availability is supported by evidence.”

Support that explanation with one real project: the architecture you chose, a failure you handled, the tradeoffs you made, and the measured outcome.
