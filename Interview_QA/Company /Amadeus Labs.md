# Amadeus Labs: 5-Year SRE Interview Questions and Detailed Answers

> **Role focus:** Site Reliability Engineering with approximately five years of experience
>
> **Topics:** Kubernetes HPA, Ingress troubleshooting, service discovery, namespace storage governance, Linux leap seconds, Kubernetes automation, time-saving automation, and security contexts
>
> **Interview guidance:** These answers are written in first-person interview style. Replace examples, tools, incident details, and measurements with your actual project experience. If you have not implemented something directly, explain your understanding and proposed approach honestly.

---

## Table of Contents

1. [Pod-level autoscaling is not happening](#1-how-would-you-fix-pod-level-autoscaling-not-happening)
2. [HPA was working but suddenly stopped](#2-an-hpa-was-working-but-suddenly-stopped-what-is-your-approach)
3. [Ingress is configured but inaccessible](#3-ingress-is-configured-but-the-application-is-not-accessible-to-end-users)
4. [Kubernetes service discovery](#4-how-does-kubernetes-handle-service-discovery)
5. [Restricting space for five application teams](#5-how-do-you-ensure-five-application-teams-do-not-use-more-than-a-defined-amount-of-space)
6. [Leap seconds in Linux](#6-what-is-a-leap-second-in-linux)
7. [Kubernetes automation examples](#7-what-types-of-kubernetes-automation-have-you-implemented)
8. [Automation that saved project time](#8-describe-an-automation-that-saved-time-in-your-project)
9. [Conflicting security context](#9-can-a-pod-run-with-runasnonroot-true-and-runasuser-0)

---

## 1. How would you fix pod-level autoscaling not happening?

### Interview-ready answer

When pod-level autoscaling is not happening, I first determine whether the problem is with the HPA configuration, the metrics pipeline, the target workload, or cluster capacity. I do not immediately change the HPA threshold because the controller may be making the correct calculation but new replicas may be blocked elsewhere.

I troubleshoot the flow in this order:

```text
Application load
    -> metric is produced
       -> metrics API exposes it
          -> HPA reads it
             -> HPA calculates desired replicas
                -> Deployment/StatefulSet increases replicas
                   -> scheduler places new Pods
                      -> nodes have sufficient capacity
```

### Step 1: Inspect the HPA

```bash
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>
kubectl get hpa <hpa-name> -n <namespace> -o yaml
```

I check:

- `scaleTargetRef` points to the correct Deployment or StatefulSet
- Current and target metric values
- Current and desired replica counts
- `minReplicas` and `maxReplicas`
- HPA conditions such as `AbleToScale`, `ScalingActive`, and `ScalingLimited`
- Events such as `FailedGetResourceMetric`, `FailedComputeMetricsReplicas`, or `FailedGetScale`
- Scale-up and scale-down behavior policies

A useful status section may look like:

```yaml
status:
  currentReplicas: 3
  desiredReplicas: 6
  conditions:
    - type: AbleToScale
      status: "True"
    - type: ScalingActive
      status: "True"
    - type: ScalingLimited
      status: "False"
```

If `desiredReplicas` remains equal to `currentReplicas`, I focus on the metric and target calculation. If `desiredReplicas` is higher but pods are not becoming available, I investigate the Deployment, ReplicaSet, scheduler, quotas, and nodes.

### Step 2: Verify the target workload

```bash
kubectl get deployment <deployment-name> -n <namespace>
kubectl describe deployment <deployment-name> -n <namespace>
kubectl get replicasets -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

I confirm that:

- The target exists in the same namespace as the HPA
- The object is scalable
- Another tool is not repeatedly setting `.spec.replicas`
- The Deployment is not paused
- New ReplicaSet creation is successful
- Admission policies or quotas are not rejecting new pods

A common GitOps mistake is continuously reconciling a fixed replica count while the HPA is trying to change it. In that design, the desired replica count should be managed carefully so the GitOps controller and HPA do not fight each other.

### Step 3: Verify the metrics pipeline

For CPU and memory metrics:

```bash
kubectl top nodes
kubectl top pods -n <namespace>
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl describe apiservice v1beta1.metrics.k8s.io
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```

If metrics are shown as `<unknown>`, likely causes include:

- Metrics Server or the cloud metrics adapter is unavailable
- APIService registration is unhealthy
- Metrics are too new or missing
- Network or TLS communication is failing
- RBAC prevents metric retrieval
- The custom or external metric name is incorrect
- The adapter query returns no series

For custom or external metrics, I inspect the corresponding API:

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1
```

The exact API version depends on the installed adapter.

### Step 4: Check CPU or memory requests

For utilization-based resource metrics, HPA calculates utilization relative to resource requests. If relevant containers do not have requests, the HPA may not be able to calculate utilization correctly.

```bash
kubectl get deployment <deployment-name> -n <namespace> -o yaml
```

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

I also check sidecars. Missing requests in one participating container can affect resource-metric calculation depending on the HPA metric type and pod composition.

### Step 5: Verify that real load crosses the target

```bash
kubectl top pods -n <namespace> --containers
```

For a CPU utilization target, the idea is:

```text
Current utilization = Current CPU usage / Requested CPU

Desired replicas approximately equals:
ceil(Current replicas x Current metric / Target metric)
```

Example:

```text
Current replicas = 3
Current CPU utilization = 120%
Target CPU utilization = 60%

Desired replicas = ceil(3 x 120 / 60) = 6
```

If an application is I/O-bound and not CPU-bound, CPU-based HPA may never trigger even when customers see high latency. In that case, I evaluate a more meaningful metric such as request rate, queue depth, concurrency, or a service-level metric.

### Step 6: Check HPA limits and stabilization

```yaml
spec:
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
```

Potential blockers include:

- `maxReplicas` already reached
- Conservative scale-up policies
- Stabilization behavior
- Tolerance around the target
- Pods not yet contributing stable metrics
- Rapidly changing or low-volume metrics

I avoid reducing stabilization values until I understand the workload, because aggressive scaling can create flapping.

### Step 7: Check whether replicas are created but remain Pending

```bash
kubectl get pods -n <namespace> -o wide
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl describe pod <pending-pod> -n <namespace>
kubectl top nodes
```

Possible issues include:

- Insufficient CPU or memory
- ResourceQuota exhausted
- Node selector or affinity mismatch
- Taints without matching tolerations
- Pod topology constraints
- PVC unavailable
- Cluster autoscaler at its maximum
- Cloud quota exhausted
- Node pool scale-up failure

This is the key distinction:

```text
HPA scales workloads.
Cluster Autoscaler or node auto-provisioning scales node capacity.
```

An HPA can request ten replicas correctly while all new pods remain Pending because no node capacity is available.

### Step 8: Fix, test, and monitor

After correcting the issue, I generate controlled load and observe the full path:

```bash
watch kubectl get hpa,pods -n <namespace>
```

I verify:

- Metrics are available
- HPA desired replicas increase
- Deployment replicas increase
- New pods schedule and become Ready
- Traffic distributes to the new pods
- Latency or queue depth returns to normal
- Scale-down is stable after load ends

### Strong closing statement

> I separate HPA decision failure from pod-capacity failure. First, I verify metrics and HPA conditions. Then I confirm the target workload accepted the replica change. Finally, I confirm the scheduler and node autoscaler can provide capacity. This avoids changing HPA settings when the real problem is metrics, quota, or node provisioning.

---

## 2. An HPA was working but suddenly stopped. What is your approach?

### Interview-ready answer

If an HPA worked earlier and suddenly stopped, I treat it as a regression. I compare the last known good state with the current state and look for changes in the metrics pipeline, application deployment, resource requests, HPA object, permissions, adapter, traffic pattern, or cluster capacity.

### Phase 1: Establish timeline and impact

I ask:

- When was the last successful scale event?
- Did application latency or errors increase?
- Is the problem isolated to one HPA or all HPAs?
- Was there a deployment, cluster upgrade, adapter upgrade, or policy change?
- Did the workload, namespace, metric labels, or metric name change?

```bash
kubectl describe hpa <hpa-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl rollout history deployment/<deployment-name> -n <namespace>
```

### Phase 2: Check blast radius

```bash
kubectl get hpa -A
kubectl top pods -A
kubectl get apiservice | grep metrics
```

- If all HPAs show `<unknown>`, I suspect a cluster-wide metrics pipeline issue.
- If only one HPA fails, I suspect its configuration, target workload, labels, or metric series.
- If HPA calculates desired replicas but pods remain Pending, I suspect scheduling or capacity.

### Phase 3: Compare configuration changes

```bash
kubectl get hpa <hpa-name> -n <namespace> -o yaml
kubectl get deployment <deployment-name> -n <namespace> -o yaml
helm history <release-name> -n <namespace>
helm get values <release-name> -n <namespace> --all
```

Typical regression causes include:

1. A deployment removed CPU or memory requests.
2. Custom metric labels changed in a new application release.
3. The HPA target name changed but the HPA did not.
4. Metrics Server or an adapter became unhealthy after an upgrade.
5. RBAC or APIService configuration changed.
6. `maxReplicas` was reduced or reached.
7. A GitOps controller started enforcing a replica count.
8. Traffic changed and the chosen metric no longer represents demand.
9. New pods cannot schedule because of quota or cluster limits.
10. The application has many unready pods, making metric calculation conservative.

### Phase 4: Restore safely

Depending on impact, mitigation can include:

- Temporarily scaling the Deployment manually while preserving incident evidence
- Restoring the last known good HPA or application configuration
- Repairing or rolling back the metrics adapter
- Restoring resource requests
- Increasing node-pool capacity within approved limits
- Rolling back the application if it stopped exporting the required metric

Example temporary mitigation:

```bash
kubectl scale deployment/<deployment-name> \
  --replicas=10 \
  -n <namespace>
```

This is a controlled incident mitigation, not the permanent fix. I document it because the HPA may overwrite the replica count when it becomes healthy.

### Phase 5: Prevent recurrence

I add monitoring for:

- HPA unable to calculate metrics
- HPA at maximum replicas for a sustained period
- Desired replicas greater than available replicas
- Metrics API or adapter unavailable
- Pending pods after scale-up
- Node autoscaler failures
- HPA not scaling during synthetic or load tests

I also add deployment checks that reject changes removing required resource requests or metric annotations.

---

## 3. Ingress is configured, but the application is not accessible to end users

### Interview-ready answer

I troubleshoot Ingress from the outside toward the pod. I verify DNS, load balancer, firewall, Ingress controller, Ingress rule, Service, EndpointSlice, pod readiness, application port, and TLS. This identifies the failing layer instead of assuming the Ingress resource itself is the problem.

```text
End user
  -> DNS
     -> external load balancer/IP
        -> firewall/WAF
           -> Ingress controller
              -> Ingress host/path rule
                 -> Kubernetes Service
                    -> EndpointSlice
                       -> Ready Pod
                          -> application port
```

### Step 1: Identify the user-visible symptom

I record the exact behavior:

- DNS resolution failure
- Connection timeout
- Connection refused
- HTTP 404
- HTTP 502 or 503
- TLS certificate error
- Redirect loop
- Works internally but not externally
- Works using IP but not hostname

The symptom helps identify the layer. For example, an Ingress-controller 404 often indicates a host or path mismatch, while 503 frequently indicates unavailable backends.

### Step 2: Check DNS

```bash
dig +short app.example.com
nslookup app.example.com
```

I compare the DNS answer with the Ingress or load-balancer address:

```bash
kubectl get ingress <ingress-name> -n <namespace> -o wide
kubectl describe ingress <ingress-name> -n <namespace>
```

I verify:

- Correct A, AAAA, or CNAME record
- Correct public or private DNS zone
- Expected load-balancer IP or hostname
- No stale record after recreation
- Appropriate TTL and propagation
- Split-horizon DNS behavior, if used

### Step 3: Test the load balancer independently of DNS

```bash
curl -vk --resolve app.example.com:443:<load-balancer-ip> \
  https://app.example.com/health
```

This preserves the expected TLS host and HTTP `Host` header while forcing the request to a specific IP. It helps separate DNS failure from routing or backend failure.

### Step 4: Verify an Ingress controller exists and is healthy

Creating only an Ingress resource is not enough. A matching Ingress controller must watch and implement it.

```bash
kubectl get ingressclass
kubectl get pods -A | grep -i ingress
kubectl get services -A | grep -i ingress
```

I verify that `spec.ingressClassName` matches an installed controller:

```yaml
spec:
  ingressClassName: nginx
```

Then I inspect controller logs and events:

```bash
kubectl logs -n <controller-namespace> \
  deployment/<ingress-controller> \
  --tail=200

kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

### Step 5: Validate host, path, and backend references

```bash
kubectl get ingress <ingress-name> -n <namespace> -o yaml
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service
spec:
  ingressClassName: nginx
  rules:
    - host: orders.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

I check:

- Hostname exactly matches the request
- Path and `pathType` are correct
- Referenced Service exists in the same namespace
- Service port name or number matches
- Controller-specific annotations are valid
- Rewrite rules are intentional

### Step 6: Verify Service and backend endpoints

```bash
kubectl get service order-service -n <namespace> -o yaml
kubectl get endpointslice -n <namespace> \
  -l kubernetes.io/service-name=order-service
kubectl get pods -n <namespace> --show-labels
```

If the EndpointSlice is empty, I compare the Service selector with pod labels and confirm pods are Ready.

```yaml
# Service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
```

The application must actually listen on `targetPort`, and readiness must succeed before the pod is considered a ready endpoint.

### Step 7: Test inside the cluster

```bash
kubectl run network-debug \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -n <namespace> \
  -- sh
```

From the debug pod:

```bash
nslookup order-service
curl -v http://order-service:80/health
curl -v http://<pod-ip>:8080/health
```

Interpretation:

- Pod IP fails: application, pod network, port, or policy problem
- Pod IP works but Service fails: selector, port mapping, EndpointSlice, or service-networking problem
- Service works but Ingress fails: controller, rule, firewall, load balancer, or TLS problem
- Ingress works by forced IP but hostname fails: DNS problem

### Step 8: Verify TLS

```bash
kubectl get secret <tls-secret> -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>
openssl s_client -connect app.example.com:443 \
  -servername app.example.com </dev/null
```

I check certificate validity, SAN hostname, expiry, secret type, certificate chain, and whether the TLS secret is in the same namespace as the Ingress.

### Step 9: Check network and cloud controls

I verify:

- Load-balancer health checks
- Firewall rules or security groups
- WAF policy
- NetworkPolicy
- Private versus public load-balancer configuration
- Proxy-only or health-check network requirements where relevant
- Cloud quota and provisioning events

### Strong closing statement

> I troubleshoot Ingress as an end-to-end request path. DNS must point to a healthy load balancer, the controller must accept the Ingress class and rule, the Service must have ready endpoints, and the application must listen on the target port. Testing each hop quickly isolates the problem.

---

## 4. How does Kubernetes handle service discovery?

### Interview-ready answer

Kubernetes normally provides service discovery through Services and cluster DNS. Pods are ephemeral and their IP addresses change, so clients do not connect to a fixed pod IP. They connect to a stable Service name. Cluster DNS resolves that name, and the Service routes traffic to ready backend endpoints.

### Core components

```text
Client Pod
   -> DNS query for order-service
      -> Cluster DNS
         -> Service ClusterIP
            -> Service proxy/data plane
               -> Ready Pod endpoint
```

### Service name formats

For a Service named `orders` in namespace `production`:

```text
orders
orders.production
orders.production.svc
orders.production.svc.cluster.local
```

A pod in the same namespace can usually use the short name:

```text
http://orders:8080
```

A pod in another namespace should include the namespace:

```text
http://orders.production:8080
```

The fully qualified service name is normally:

```text
orders.production.svc.cluster.local
```

The actual cluster domain can be customized.

### How resolution works

1. The Service is created with a selector.
2. The control plane tracks matching and ready pods through EndpointSlices.
3. Cluster DNS publishes records for the Service.
4. The kubelet configures each pod's `/etc/resolv.conf`.
5. A client resolves the Service name.
6. For a normal ClusterIP Service, DNS returns the stable ClusterIP.
7. Cluster networking forwards traffic to one of the ready endpoints.

Example pod resolver configuration:

```text
nameserver <cluster-dns-ip>
search production.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Normal Service versus headless Service

#### Normal ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  selector:
    app: orders
  ports:
    - port: 80
      targetPort: 8080
```

DNS resolves the name to the Service ClusterIP.

#### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - name: db
      port: 5432
```

For a headless Service, DNS returns the selected pod IPs rather than a virtual ClusterIP. This is useful for StatefulSets and applications that need direct endpoint discovery.

### DNS troubleshooting

```bash
kubectl get service -n <namespace>
kubectl get endpointslice -n <namespace>
kubectl get pods -n kube-system | grep -E 'coredns|kube-dns'
kubectl logs -n kube-system deployment/coredns
```

From a debug pod:

```bash
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
nslookup orders.production.svc.cluster.local
```

I also check NetworkPolicy, DNS pod health, upstream DNS, and whether the Service selector produces endpoints.

---

## 5. How do you ensure five application teams do not use more than a defined amount of space?

### Interview-ready answer

I isolate each team in a separate namespace and apply a `ResourceQuota` to each namespace. For storage, I limit total PVC requests and, if necessary, storage consumption by StorageClass. I use a `LimitRange` or policy engine to enforce sensible defaults and per-PVC boundaries. RBAC prevents application teams from modifying or deleting the quota.

### Namespace model

```text
team-a namespace -> ResourceQuota + LimitRange + RBAC
team-b namespace -> ResourceQuota + LimitRange + RBAC
team-c namespace -> ResourceQuota + LimitRange + RBAC
team-d namespace -> ResourceQuota + LimitRange + RBAC
team-e namespace -> ResourceQuota + LimitRange + RBAC
```

### Storage quota example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-storage-quota
  namespace: team-a
spec:
  hard:
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
    requests.ephemeral-storage: 100Gi
    limits.ephemeral-storage: 200Gi
```

This can limit aggregate requested persistent storage, PVC count, and pod ephemeral-storage requests or limits in that namespace.

### StorageClass-specific quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-class-quota
  namespace: team-a
spec:
  hard:
    fast.storageclass.storage.k8s.io/requests.storage: 100Gi
    standard.storageclass.storage.k8s.io/requests.storage: 400Gi
```

This prevents a team from placing all storage on an expensive class.

### LimitRange for individual PVCs

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-size-policy
  namespace: team-a
spec:
  limits:
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 100Gi
```

A namespace may have 500 GiB total quota, while the LimitRange prevents one PVC from requesting more than 100 GiB.

### Important distinction

`requests.storage` governs requested persistent volume capacity. It does not automatically limit an application's actual filesystem usage below the provisioned volume size. Filesystem enforcement depends on the storage system and volume capacity.

For container writable-layer and `emptyDir` consumption, I define ephemeral-storage requests and limits:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "5Gi"
```

For memory-backed `emptyDir`, I set `sizeLimit` and account for memory behavior:

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 2Gi
```

### Enforcement and monitoring

```bash
kubectl get resourcequota -n team-a
kubectl describe resourcequota team-storage-quota -n team-a
kubectl get limitrange -n team-a
```

I also:

- Restrict quota modification through RBAC
- Protect policies with admission controls
- Create alerts at 70%, 85%, and 95% usage where suitable
- Tag ownership and cost center
- Review unused PVCs and snapshots
- Apply retention and cleanup automation
- Include storage growth in capacity planning

### If the question means CPU and memory rather than disk space

I use the same namespace model with compute quotas:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-compute-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
```

---

## 6. What is a leap second in Linux?

### Interview-ready answer

A leap second is a one-second correction applied to Coordinated Universal Time, or UTC, to keep it close to time based on the Earth's rotation. Because atomic time is extremely stable and the Earth's rotation is not perfectly uniform, UTC occasionally needs adjustment.

A positive leap second can appear conceptually as:

```text
23:59:58
23:59:59
23:59:60
00:00:00
```

### How Linux receives leap-second information

Linux systems normally receive time synchronization and pending leap-second information through NTP or another time synchronization service. Common implementations include:

- `chronyd`
- `ntpd`
- `systemd-timesyncd`, depending on platform and requirements
- Cloud-provider time synchronization services

An administrator normally should not set the leap second manually. The time-synchronization infrastructure distributes the correction.

### Step versus smear

There are two broad operating approaches:

#### Leap-second step

The system applies the leap-second behavior at the defined UTC boundary. Software that assumes wall-clock time always moves forward may behave unexpectedly.

#### Leap smear

The time service spreads the one-second adjustment gradually across a larger interval. This avoids an abrupt repeated or inserted second, but all communicating systems must use a compatible time source. Mixing smeared and non-smeared time can temporarily produce clock differences.

### Why SRE teams care

Time anomalies can affect:

- Distributed transactions
- Database consistency assumptions
- Log ordering
- Authentication-token validation
- Scheduled jobs
- Monitoring windows and rate calculations
- Message ordering
- Timeout logic
- JVM or runtime behavior

Applications should use the correct clock:

- Use a monotonic clock for elapsed time, duration, retry, and timeout calculations.
- Use wall-clock time for timestamps and business dates.
- Do not assume wall-clock time can never jump backward or forward.

### Operational checks

For chrony:

```bash
chronyc tracking
chronyc sources -v
chronyc sourcestats -v
```

For timedatectl:

```bash
timedatectl status
```

For kernel and service messages:

```bash
journalctl -u chronyd
journalctl -u systemd-timesyncd
dmesg | grep -i -E 'clock|time|leap'
```

### SRE preparation approach

1. Use one approved time architecture across the environment.
2. Avoid mixing leap-smear and step-based sources.
3. Keep the kernel, runtime, time daemon, and database supported and patched.
4. Monitor clock offset and synchronization state.
5. Test time-sensitive applications and scheduled processing.
6. Use monotonic time APIs for durations.
7. Confirm cloud VMs, Kubernetes nodes, and external dependencies use compatible time behavior.
8. Have a response plan for excessive offset or synchronization loss.

### Strong closing statement

> A leap second is mainly a system-design and time-synchronization concern. My focus is to use a consistent authoritative time source, monitor offset, keep the time stack patched, avoid mixed smear policies, and ensure applications use monotonic time for durations and timeouts.

---

## 7. What types of Kubernetes automation have you implemented?

### Interview-ready answer

I focus Kubernetes automation on deployment consistency, policy enforcement, observability, incident response, upgrades, and cost control. I automate repeatable and testable tasks while keeping production changes auditable and reversible.

### 1. Automated application deployment

A CI/CD pipeline can:

1. Build the image
2. Run unit and integration tests
3. Scan the image
4. Push it with an immutable tag or digest
5. Lint and render the Helm chart
6. Deploy to the target namespace
7. Wait for rollout
8. Run smoke tests
9. Roll back on failure

```bash
helm lint ./charts/order-service

helm upgrade --install order-service ./charts/order-service \
  -n production \
  -f environments/prod.yaml \
  --set image.tag="$IMAGE_TAG" \
  --atomic \
  --wait \
  --timeout 10m

kubectl rollout status deployment/order-service \
  -n production \
  --timeout 5m
```

### 2. Namespace onboarding

I can automate creation of a new team namespace with:

- Namespace labels
- ResourceQuota
- LimitRange
- RBAC
- ServiceAccount
- NetworkPolicy
- Pod security labels
- Logging and monitoring configuration
- Cost-allocation metadata

This converts a manual ticket workflow into a controlled self-service process.

### 3. Policy enforcement

Admission policies or a policy engine can enforce:

- Approved registries
- Non-root containers
- Resource requests and limits
- Required labels
- Prohibition of privileged containers
- Read-only root filesystem where compatible
- Dropped Linux capabilities
- Approved Ingress classes
- Restricted host networking and host paths

### 4. Autoscaling automation

I automate standard HPA patterns with configurable thresholds and safeguards. I also monitor HPA health, maximum replica saturation, and pending pods after scale-up.

### 5. Certificate and secret lifecycle

Depending on the platform, automation can handle:

- Certificate issuance and renewal
- Expiry alerts
- External secret synchronization
- Secret rotation workflows
- Alerting on failed synchronization

### 6. Backup and recovery validation

I automate:

- Kubernetes resource backups
- Persistent-data backup schedules
- Backup success alerts
- Periodic restore testing
- Recovery evidence and reporting

A successful backup job is not enough. Restore validation proves recoverability.

### 7. Cluster and node maintenance

Automation can perform:

- Controlled node drain
- PodDisruptionBudget validation
- Node-pool upgrades
- Add-on compatibility checks
- Post-upgrade smoke tests
- Node health verification

```bash
kubectl cordon <node>
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

These commands should be wrapped in guardrails and maintenance procedures rather than run blindly.

### 8. Incident diagnostics

An automated diagnostic package can collect:

- Pod status and events
- Current and previous logs
- Deployment and ReplicaSet state
- HPA status
- Service and EndpointSlice state
- Node assignment and capacity
- Recent rollout history

The output is attached to the incident after redacting sensitive values.

### 9. Cost and hygiene automation

Examples include:

- Detecting unused namespaces
- Finding orphaned PVCs
- Identifying overprovisioned requests
- Cleaning expired preview environments
- Scheduling non-production scaling
- Reporting idle load balancers or services

### 10. Observability as code

I version-control:

- Alert rules
- Dashboard definitions
- SLO definitions
- Recording rules
- Runbook links
- Notification routing

This makes monitoring changes reviewable and reproducible.

---

## 8. Describe an automation that saved time in your project

### Interview-ready answer using STAR

The best answer should explain one real case using Situation, Task, Action, and Result. The example below is a template and should be adjusted to your actual project.

### Situation

> During Kubernetes incidents, the on-call engineer manually ran many commands to find the affected deployment, pod restart reason, previous logs, node placement, recent rollout, HPA status, Service endpoints, and namespace events. The investigation steps were repetitive and inconsistent, especially during high-severity incidents.

### Task

> My goal was to reduce initial diagnostic time and ensure every engineer collected the same minimum evidence without requiring broad cluster-admin access.

### Action

> I created a controlled diagnostic automation triggered by the service name, namespace, and environment. It validated the user's access, identified the owning Deployment and pods, collected current and previous logs, Kubernetes events, resource utilization, rollout history, HPA status, Service and EndpointSlice details, and recent deployment metadata. It packaged the results and attached them to the incident ticket or collaboration channel. Sensitive environment variables and secrets were excluded or redacted.

Conceptual command flow:

```bash
kubectl get deployment "$APP" -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -l "app=$APP" -o wide
kubectl describe deployment "$APP" -n "$NAMESPACE"
kubectl rollout history deployment/"$APP" -n "$NAMESPACE"
kubectl get hpa -n "$NAMESPACE"
kubectl get service,endpointslice -n "$NAMESPACE"
kubectl get events -n "$NAMESPACE" --sort-by=.lastTimestamp
kubectl top pods -n "$NAMESPACE" --containers
```

For each relevant pod:

```bash
kubectl describe pod "$POD" -n "$NAMESPACE"
kubectl logs "$POD" -n "$NAMESPACE" --all-containers=true
kubectl logs "$POD" -n "$NAMESPACE" --all-containers=true --previous
```

### Result

> The automation standardized triage and allowed on-call engineers to begin hypothesis testing immediately instead of spending the first part of the incident locating information. It also improved postmortem evidence because the initial state was captured before pods were restarted or replaced.

Use a verified result if available:

```text
Before: Median initial evidence collection took X minutes.
After:  Median initial evidence collection took Y minutes.
Impact: Saved approximately Z engineer-hours per month.
```

Do not invent values. Explain how the measurement was obtained.

### Alternative time-saving example: Namespace onboarding

> Application onboarding originally required separate tickets for namespace creation, RBAC, quotas, network policy, service accounts, monitoring, and secrets integration. I created a reusable Terraform or GitOps template that accepted team name, environment, resource allocation, and owner as inputs. A pull request generated the standard resources and policy checks. This reduced manual handoffs and ensured every namespace followed the same baseline.

### What makes this a strong SRE answer?

It demonstrates:

- Toil identification
- Safe automation
- Standardization
- Access control
- Auditability
- Measurable time saving
- Improved incident response or developer experience

---

## 9. Can a pod run with `runAsNonRoot: true` and `runAsUser: 0`?

Given:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 0
```

### Direct answer

No, the container should not start successfully with that effective configuration. `runAsNonRoot: true` requires a non-root UID, while `runAsUser: 0` explicitly requests UID 0, which is the root user. These settings conflict.

The pod object may be accepted by the API depending on the admission configuration, but the container will fail during startup enforcement. You may see a status such as `CreateContainerConfigError` and an event indicating that the container is configured to run as root while non-root execution is required.

### Why the settings conflict

```text
runAsNonRoot: true -> Effective UID must not be 0
runAsUser: 0       -> Effective UID must be exactly 0
```

Both conditions cannot be true at the same time.

### Correct configuration

Use a non-zero numeric UID supported by the image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-application
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
  containers:
    - name: application
      image: registry.example.com/application:1.0.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

### Image requirements

The image must support running as that user. It must have correct permissions for:

- Application files
- Working directories
- Writable temporary directories
- Mounted volumes
- Ports
- Runtime caches and logs

Example Dockerfile:

```dockerfile
FROM alpine:3.22

RUN addgroup -g 10001 appgroup \
    && adduser -D -u 10001 -G appgroup appuser

WORKDIR /app
COPY --chown=10001:10001 . /app

USER 10001:10001
CMD ["/app/application"]
```

A non-root process generally cannot bind to privileged ports below 1024 unless additional mechanisms are used. Prefer an unprivileged container port such as 8080 and map the Kubernetes Service port to it.

```yaml
ports:
  - containerPort: 8080
```

```yaml
# Service mapping
ports:
  - port: 80
    targetPort: 8080
```

### Pod-level versus container-level precedence

Pod-level security context supplies defaults to containers. A container-level security context can override overlapping fields for that container. Therefore, I inspect both levels when troubleshooting the effective identity.

```yaml
spec:
  securityContext:
    runAsUser: 10001
  containers:
    - name: app
      securityContext:
        runAsUser: 20001
```

Here, the container-level UID is the effective value for that container.

### Troubleshooting commands

```bash
kubectl get pod secure-application -o wide
kubectl describe pod secure-application
kubectl get pod secure-application -o yaml
```

If the container runs, verify its identity:

```bash
kubectl exec secure-application -- id
```

Expected output should show a non-zero UID.

### Strong interview closing statement

> The configuration is contradictory. `runAsNonRoot: true` is an enforcement guardrail, while `runAsUser` selects the UID. I would change UID 0 to a verified non-zero UID, ensure the image and mounted volumes support it, and combine it with no privilege escalation, dropped capabilities, and other compatible hardening controls.

---

# Rapid Revision Cheat Sheet

## HPA does not scale

```bash
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa> -n <namespace>
kubectl top pods -n <namespace>
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl describe deployment <deployment> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

```text
Check: metric -> HPA decision -> replica update -> pod scheduling -> node capacity
```

## Ingress inaccessible

```bash
dig +short app.example.com
kubectl describe ingress <ingress> -n <namespace>
kubectl get ingressclass
kubectl get service,endpointslice -n <namespace>
kubectl logs -n <controller-namespace> deployment/<controller>
curl -vk --resolve app.example.com:443:<ip> https://app.example.com/
```

```text
DNS -> Load Balancer -> Controller -> Ingress -> Service -> EndpointSlice -> Pod
```

## Service discovery

```text
<service>.<namespace>.svc.cluster.local
```

## Team storage governance

```text
Namespace + ResourceQuota + LimitRange + RBAC + Monitoring
```

## Conflicting security context

```yaml
# Invalid effective intent
runAsNonRoot: true
runAsUser: 0
```

```yaml
# Correct pattern
runAsNonRoot: true
runAsUser: 10001
```

---

# Reference Documentation

- Kubernetes Horizontal Pod Autoscaling: <https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/>
- GKE HPA troubleshooting: <https://cloud.google.com/kubernetes-engine/docs/troubleshooting/horizontal-pod-autoscaling>
- Kubernetes Ingress: <https://kubernetes.io/docs/concepts/services-networking/ingress/>
- Kubernetes DNS for Services and Pods: <https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>
- Kubernetes Services: <https://kubernetes.io/docs/concepts/services-networking/service/>
- Kubernetes ResourceQuota: <https://kubernetes.io/docs/concepts/policy/resource-quotas/>
- Kubernetes security context: <https://kubernetes.io/docs/tasks/configure-pod-container/security-context/>
- Kubernetes Pod Security Standards: <https://kubernetes.io/docs/concepts/security/pod-security-standards/>
- Red Hat leap-second overview: <https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/deployment_guide/s1-understanding_leap_seconds>

---

> **Final note:** For a five-year SRE interview, do not stop at commands. Explain the failure domain, how you narrowed it, what mitigation you used, how you validated recovery, and what automation or monitoring you introduced to prevent recurrence.
