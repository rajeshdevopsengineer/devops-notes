# Senior DevOps, Cloud and SRE Interview Handbook

**Target profile:** Senior Cloud Administrator / AWS Cloud Engineer / DevOps / SRE  
**Experience framing:** 6 years  
**Prepared for:** Rajesh Singh  
**Edition:** Expanded question-by-question answer bank

> This edition expands every consolidated interview question from the earlier handbook. Repeated or near-identical questions from the original list are intentionally merged into one canonical answer, but every distinct technical theme is covered. Project-specific figures such as cluster count, RDS size, incident duration and cost savings are left as placeholders or answer frameworks because they must match your real experience.

## How to answer at senior level

Use this sequence in most technical answers:

1. **Define the concept precisely.**
2. **Explain the internal workflow or architecture.**
3. **State when you would and would not use it.**
4. **Show an implementation, command, manifest or design.**
5. **Explain validation, monitoring and rollback.**
6. **Call out security, availability and cost trade-offs.**
7. **Close with a short project example using only facts you can defend.**

---
A consolidated answer bank with architectures, troubleshooting flows, Kubernetes YAML, Terraform, CI/CD, Bash, Python, Ansible, AWS and Azure examples. Repeated questions are merged into stronger canonical answers so the handbook remains usable during interview preparation.


# How to use this handbook

- Use the first 30-60 seconds of each answer for the definition and decision criteria. Add the implementation detail only when the interviewer probes.
- Replace sample scale, counts and outcomes with facts from your own project. Never invent production numbers.
- Use STAR for behavioral questions: Situation, Task, Action, Result, then Learning.
- Commands are examples. Validate versions, permissions, maintenance windows and rollback steps before production use.

## Fast answer pattern

Context -> design decision -> implementation -> validation/monitoring -> failure handling -> security/cost trade-off.


## Architecture at a glance


```bash
Users -> Route 53 / DNS -> CloudFront + WAF -> ALB / Ingress
                         |                  |
                         v                  v
                    S3 static         EKS/AKS workloads
                                         |
                         +---------------+---------------+
                         |                               |
                  RDS Multi-AZ / cache             S3 / object store
                         |
         Metrics/Logs/Traces -> Prometheus, Grafana, CloudWatch, ELK
         Delivery -> Git -> CI -> scan/test -> registry -> GitOps/Helm -> cluster
```


# 1. Kubernetes architecture, scheduling and workloads


## Q1. Explain Kubernetes architecture and the request workflow.

### Direct interview answer

Kubernetes is a declarative control system. The API server is the front door. Desired state is persisted in etcd. Controllers continuously compare desired and observed state. The scheduler binds unscheduled Pods to nodes. On each node, kubelet reconciles PodSpecs through the container runtime, while kube-proxy or an eBPF data plane implements Service connectivity.

- kubectl sends an authenticated and authorized API request; admission controllers validate or mutate it.
- The API server writes the object to etcd and exposes it through watches.
- A Deployment controller creates a ReplicaSet; the ReplicaSet controller creates Pods.
- The scheduler filters and scores feasible nodes, then writes a binding.
- The target kubelet pulls images, mounts volumes and asks containerd/CRI-O to start containers.
- CNI configures Pod networking; CSI provisions or attaches storage; endpoints are published when readiness succeeds.
Interview close: I describe Kubernetes as multiple reconciliation loops, not a sequence of imperative scripts.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q2. What does every major component do?

### Direct interview answer

Control-plane components manage cluster state; node components execute workloads.

- kube-apiserver: REST API, authentication, authorization, admission and validation.
- etcd: strongly consistent key-value store for cluster state. Back it up and protect access.
- kube-scheduler: places Pods based on requests, constraints, affinity, taints and scoring.
- kube-controller-manager: runs Deployment, ReplicaSet, Node, Job, EndpointSlice and other controllers.
- cloud-controller-manager: integrates cloud nodes, routes, load balancers and volumes where applicable.
- kubelet: node agent that reconciles assigned Pods and reports node status.
- container runtime: runs containers through CRI.
- kube-proxy/data plane: implements Service routing; CoreDNS provides service discovery.


### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q3. Like kubelet, is there one control-plane agent?

### Direct interview answer

No single control-plane equivalent manages everything. The API server, scheduler and controller managers are independent processes. They use leader election where multiple replicas run. On managed services such as EKS, the provider operates the control plane; on self-managed clusters, systemd or static Pods commonly keep control-plane components running.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q4. How do you join a new worker or control-plane node?

### Direct interview answer

For kubeadm, first satisfy version, runtime, networking, swap and firewall prerequisites. A worker joins using the API endpoint, bootstrap token and CA hash. A new control-plane node additionally receives the certificate key and --control-plane flag. Validate node Ready state, CNI, DNS and labels after joining.


```bash

### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# Generate on an existing control-plane node
kubeadm token create --print-join-command

# Worker example
sudo kubeadm join api.internal:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Additional control-plane example
sudo kubeadm join api.internal:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>

kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
```


## Q5. What if one master/control-plane node fails?

### Direct interview answer

Existing containers normally continue running because kubelet and the runtime do not require continuous API availability to keep already-started containers alive. However, scheduling, controller reconciliation, endpoint updates and cluster changes are impaired. A single control-plane design is a single point of failure. Production uses multiple control-plane nodes and an odd-numbered etcd quorum behind a stable API endpoint.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q6. Can you write directly to etcd?

### Direct interview answer

Technically etcdctl can write keys, but do not manually edit Kubernetes keys. It bypasses API validation, admission, authorization and object invariants and can corrupt state. Use the Kubernetes API. Use etcdctl for supported health checks, snapshots and carefully controlled restore procedures.


```bash
ETCDCTL_API=3 etcdctl endpoint health
ETCDCTL_API=3 etcdctl snapshot save /secure/etcd-$(date +%F).db
ETCDCTL_API=3 etcdctl snapshot status /secure/etcd-2026-10-08.db
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q7. Why StatefulSet, and what happens to PVCs when HPA adds a Pod?

### Direct interview answer

A StatefulSet is for stable identity, ordered lifecycle and per-replica persistent storage. With volumeClaimTemplates, scaling from app-2 to app-3 creates a new PVC for ordinal 3. That PVC is initially empty unless the storage system or application bootstraps it. The new replica must not receive traffic until it has joined the database/cluster, restored a snapshot, replayed logs or synchronized data. Readiness gates, init containers and application-specific membership automation are essential.

- Use a headless Service for stable DNS such as db-2.db.default.svc.
- Use one PVC per ordinal; never mount one ReadWriteOnce volume to multiple nodes.
- For databases, use the database operator or native replication rather than blind CPU-based HPA.
- Keep the new Pod unready during recovery, then let the Service add it to endpoints.
- Scaling down retains PVCs by default, which protects data but requires lifecycle governance.

```bash
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: gp3
    resources:
      requests:
        storage: 100Gi
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q8. Deployment vs ReplicaSet vs StatefulSet vs DaemonSet vs Job?

### Direct interview answer

A Deployment manages stateless rolling releases through ReplicaSets. A ReplicaSet only maintains a count and is rarely managed directly. StatefulSet adds stable ordinal identity and storage. DaemonSet runs one Pod per eligible node. Job runs work to completion; CronJob schedules Jobs.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q9. When do you use HPA and VPA?

### Direct interview answer

Use HPA when additional replicas increase throughput, for example a stateless API scaling on CPU, request rate or queue depth. Use VPA when a workload needs right-sized CPU/memory requests and adding replicas does not solve the bottleneck. Avoid letting HPA and VPA both change CPU requests for the same workload because HPA utilization uses requests as the denominator. A common pattern is HPA on external metrics plus VPA in recommendation mode.


```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: api-hpa}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: api}
  minReplicas: 3
  maxReplicas: 30
  behavior:
    scaleDown: {stabilizationWindowSeconds: 300}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 65}
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q10. Scaling vs autoscaling?

### Direct interview answer

Scaling is any deliberate capacity change, manual or automated. Autoscaling is a feedback loop that adjusts capacity from observed metrics. HPA changes replica count, VPA changes resource requests, and a node autoscaler changes compute capacity.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q11. Node selector, affinity, taints and tolerations?

### Direct interview answer

nodeSelector is a simple hard label match. Node affinity supports required and preferred rules. A taint repels Pods; a toleration permits but does not force placement. Combine a taint with node affinity for dedicated nodes.


```bash

### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# Label and taint the large node
kubectl label node node-large workload=data
kubectl taint node node-large dedicated=data:NoSchedule

# Pod fragment
nodeSelector:
  workload: data
tolerations:
- key: dedicated
  operator: Equal
  value: data
  effect: NoSchedule
```


## Q12. Schedule only on medium and large, never small.

### Direct interview answer

Label eligible nodes and use required node affinity with In values. This is clearer than using negative hostname logic.


```bash
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-size
          operator: In
          values: [medium, large]
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q13. What is PDB?

### Direct interview answer

A PodDisruptionBudget limits voluntary disruption by requiring minAvailable or maxUnavailable during drains, upgrades and autoscaler evictions. It does not protect against involuntary node failure and cannot create capacity. Configure it with enough replicas and topology spread.


```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: api-pdb}
spec:
  minAvailable: 2
  selector:
    matchLabels: {app: api}
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q14. PV vs PVC and storage classes?

### Direct interview answer

A PV represents storage capacity; a PVC is a workload request for capacity and access mode. StorageClass defines dynamic provisioning and parameters. CSI drivers implement provisioning, attachment, mounting and snapshots. Check access mode, zone topology, reclaim policy and expansion support.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q15. ConfigMap and Secrets?

### Direct interview answer

ConfigMap stores non-sensitive configuration. Secret stores sensitive bytes but is only base64-encoded by default, not inherently encrypted. Enable encryption at rest with a KMS provider, restrict RBAC, avoid environment variables for highly sensitive values when possible, mount read-only volumes, rotate and audit. External Secrets or CSI Secret Store can synchronize cloud secret managers.

- Common secret types: Opaque, kubernetes.io/tls, dockerconfigjson, service-account-token and bootstrap token.
- Use immutable Secrets/ConfigMaps for stable release artifacts where appropriate.
- Do not commit decoded secret values or kubectl output to tickets and logs.


### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q16. Init containers and restartPolicy Never?

### Direct interview answer

Init containers run sequentially before app containers and are useful for schema checks, config generation, dependency gates and permissions. restartPolicy: Never means kubelet does not restart containers in that Pod after termination. A Job controller can still create replacement Pods according to backoff policy.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q17. How many containers can run in a Pod? Give a 4-5 container case.

### Direct interview answer

Kubernetes does not define a small fixed business limit, but every container consumes resources and increases coupling. Use multiple containers only when they share lifecycle, network namespace and volumes. Example: application, service-mesh proxy, log/telemetry sidecar, configuration reloader and security agent. Prefer node-level DaemonSets for generic agents to avoid repeating them in every Pod.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q18. Service types and targetPort default.

### Direct interview answer

ClusterIP exposes a stable in-cluster virtual IP. NodePort opens a port on every node. LoadBalancer requests an external load balancer. ExternalName returns a DNS CNAME. A headless Service uses clusterIP: None. If targetPort is omitted it defaults to the value of port; confirm that the container actually listens there.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q19. Ingress design and packet flow.

### Direct interview answer

DNS resolves the application name to an external load balancer. The load balancer forwards to the ingress controller Service. The controller matches host/path/TLS rules and proxies to a Service, which selects ready Pod endpoints. Security controls include WAF, TLS, security groups/NSGs, NetworkPolicy and authentication.


```bash
Client -> DNS -> WAF/LB -> Ingress Controller -> Service -> Ready Pod
                         TLS       host/path          EndpointSlice
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q20. Ingress is not routing. How do you troubleshoot?

### Direct interview answer

Validate from outside inward, then from Pod outward.

- Confirm DNS, certificate, LB listener, health checks and firewall rules.
- kubectl describe ingress; verify ingressClassName and controller logs.
- Check that the Service port/targetPort is correct and EndpointSlices contain ready Pod IPs.
- Test kubectl port-forward to the Pod, then Service DNS from a debug Pod.
- Check readiness probes, application bind address, NetworkPolicies and path rewrite/host header behavior.


### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q21. Service is not mapped to a Deployment.

### Direct interview answer

Services do not map to Deployments directly. The selector must match Pod labels. Compare kubectl get svc -o yaml with kubectl get pods --show-labels and kubectl get endpointslice. Also check namespace, readiness and targetPort names.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q22. Probes: liveness, readiness and startup.

### Direct interview answer

Readiness controls traffic eligibility. Liveness restarts a hung container. Startup protects slow starts by delaying liveness/readiness behavior. Probes can use HTTP, TCP, gRPC or exec. Keep liveness independent from fragile downstream dependencies to avoid cascading restarts.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q23. Create a three-replica Apache deployment.

### Direct interview answer

Use a Deployment so rollout and rollback are managed.


```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: apache}
spec:
  replicas: 3
  selector: {matchLabels: {app: apache}}
  template:
    metadata: {labels: {app: apache}}
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports: [{containerPort: 80}]
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits: {cpu: 500m, memory: 256Mi}
        readinessProbe:
          httpGet: {path: /, port: 80}
          initialDelaySeconds: 5
          periodSeconds: 10
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q24. Add labels and annotations to an existing Pod.

### Direct interview answer

Use overwrite when a key already exists. Direct changes to a controller-owned Pod are ephemeral; update the Deployment/StatefulSet template for permanence.


```bash
kubectl label pod api-123 environment=prod --overwrite
kubectl annotate pod api-123 owner=platform --overwrite
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
# 2. Kubernetes troubleshooting and security


## Q25. Pod is Pending. What are the reasons and workflow?

### Direct interview answer

Pending means the Pod has been accepted but one or more containers are not running. Start with events, scheduler messages and PVC state.

- Insufficient CPU/memory/ephemeral storage or max Pods/IP exhaustion.
- Node selector/affinity mismatch, untolerated taints or topology constraints.
- Unbound PVC, zone mismatch, missing StorageClass or CSI failure.
- ResourceQuota/LimitRange/admission policy blocks or image pull secret delays.
- Node NotReady, cordoned nodes, hostPort conflict or scheduling gates.

```bash
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe node <node>
kubectl get pvc,pv -n <ns>
kubectl top nodes
kubectl get pod <pod> -o jsonpath='{.status.conditions}' | jq
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q26. 0/5 nodes available: insufficient memory.

### Direct interview answer

The scheduler compares Pod memory requests, not current free memory. Inspect requests on nodes and the Pod. Correct unrealistic requests, scale the node group/Karpenter, add a suitable node, or remove safe non-production capacity. Do not simply increase limits. HPA cannot solve an unschedulable first Pod; node autoscaling must add capacity.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q27. Pending due to disk pressure.

### Direct interview answer

Determine whether the issue is node ephemeral storage, inode exhaustion, PV provisioning or a full application volume. Check DiskPressure, df -h, df -i, container runtime usage and kubelet eviction events. Safely prune unused images/logs, fix log rotation, expand storage if supported and prevent recurrence with ephemeral-storage requests/limits.



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
## Q28. Worker node is not joining.

### Direct interview answer

Check bootstrap and network in layers.

- Validate time sync, DNS, hostname uniqueness, supported OS/kernel, swap and container runtime.
- Verify TCP reachability to the API endpoint, security groups/firewall and proxy/NO_PROXY.
- Inspect kubelet logs and bootstrap kubeconfig/certificates.
- Validate token expiry and CA hash; for EKS validate IAM role and aws-auth/access entry.
- Confirm CNI prerequisites and node IAM permissions after registration.

```bash
systemctl status kubelet
journalctl -u kubelet -n 200 --no-pager
crictl info
curl -k https://api.internal:6443/livez
kubeadm reset -f   # only after confirming rebuild is intended
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q29. Unable to evict Pods during drain.

### Direct interview answer

Check PDBs, finalizers, local storage, unmanaged Pods and disruption budgets. Use kubectl drain without unsafe flags first. Coordinate stateful quorum and approve exceptions. --disable-eviction or --force can cause outages and should be a controlled last resort.


```bash
kubectl get pdb -A
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=20m
kubectl get pods -A --field-selector spec.nodeName=<node>
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q30. ImagePullBackOff.

### Direct interview answer

Describe the Pod and check event text. Verify image name/tag/digest, registry DNS/network, credentials, rate limits, architecture compatibility and node disk. For private registries validate imagePullSecrets or workload identity. BackOff means kubelet is retrying with increasing delay.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q31. If control-plane-to-worker firewall breaks, what happens?

### Direct interview answer

Already-running containers can continue, but control-plane reconciliation, logs/exec, health status, rescheduling and changes are affected depending on direction and blocked ports. Treat it as control-plane degradation: freeze risky changes, page the platform/network team, identify affected flows, restore validated rules, then reconcile node and workload state. Communicate impact, mitigation, owner and next update cadence without guessing.



### Senior-level depth
- Separate Azure control-plane RBAC, Kubernetes API authorization, cluster managed identity and pod workload identity.
- Prefer private endpoints, private DNS, managed identities/federation and scoped data-plane roles instead of account keys.
- Trace traffic through DNS, UDRs, NSGs, firewalls and service endpoints/private links.

### Validation checklist
1. Check the signed-in principal and role assignment scope.
2. Verify private DNS resolution and effective routes.
3. Inspect NSG flow logs and application connection errors.
4. Confirm token audience, issuer and federated credential for workload identity.
5. Validate access from the intended pod and denial from an unintended pod.

### Interview example framing
“I use identity rather than stored keys, restrict the network path with Private Link, and validate both the intended access and the expected denial path.”

---
## Q32. Kubernetes security: container and infrastructure sides.

### Direct interview answer

Apply defense in depth from supply chain to runtime.

- Identity: SSO/OIDC, least-privilege RBAC, short-lived credentials, disable anonymous access.
- Admission: Pod Security Admission restricted profile, policy-as-code for images, capabilities and host access.
- Workload: non-root, read-only root filesystem, drop capabilities, seccomp, AppArmor/SELinux, resource limits.
- Network: private API endpoint, NetworkPolicies, segmented subnets, controlled egress and mTLS where justified.
- Secrets: KMS encryption, external secret store, rotation, no plaintext in Git.
- Supply chain: signed images, SBOM, vulnerability and IaC scans, digest pinning, trusted registries.
- Infra: hardened images, patching, minimal node access, audit/control-plane logs, runtime detection and backups.

```bash
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q33. RBAC example.

### Direct interview answer

Grant the smallest verbs on the smallest resources in one namespace; bind groups rather than individuals.


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: deploy-reader, namespace: prod}
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: platform-read, namespace: prod}
subjects:
- kind: Group
  name: platform-readers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deploy-reader
  apiGroup: rbac.authorization.k8s.io
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q34. Design Istio for a cluster.

### Direct interview answer

Run highly available ingress gateways at the edge and sidecar or ambient data plane for selected namespaces. Use mTLS STRICT after migration validation, AuthorizationPolicy for service-to-service access, DestinationRule for traffic policy, VirtualService for routing and telemetry export to Prometheus/tracing. Keep the control plane out of the request path, size gateways separately and define an emergency bypass/rollback plan.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# 3. EKS, Helm and cluster lifecycle


## Q35. On-prem Kubernetes vs EKS.

### Direct interview answer

Prefer on-prem when data residency, deterministic hardware, disconnected operation, specialized appliances or existing data-center economics dominate and the organization can operate control planes, etcd, upgrades and hardware. Prefer EKS when speed, managed control-plane availability, AWS integration, elastic capacity and reduced undifferentiated operations matter. Evaluate total cost, skills, latency, compliance, lock-in, egress and disaster recovery, not just node price.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q36. What manages self-managed worker nodes?

### Direct interview answer

Self-managed EKS nodes are normally EC2 Auto Scaling Groups with launch templates. Cluster Autoscaler can adjust ASG desired capacity based on pending Pods. Managed Node Groups let EKS orchestrate node lifecycle. Karpenter provisions EC2 capacity directly from Pod scheduling requirements.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q37. What is Karpenter and which metric scales it?

### Direct interview answer

Karpenter is a node lifecycle controller. Scale-up is primarily triggered by unschedulable Pods and their aggregate resource requests plus scheduling constraints, not by an average CPU threshold. It evaluates CPU, memory, GPU, topology, taints, affinity and storage constraints. Scale-down is driven by emptiness/underutilization and consolidation/disruption policy while respecting scheduling feasibility and PDBs.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q38. EKS cluster upgrade process.

### Direct interview answer

Upgrade one minor version at a time through a tested, observable sequence.

- Inventory cluster version, node versions, APIs, add-ons, CRDs, controllers and clients. Review deprecations.
- Back up manifests and data; verify restore procedures. Enable control-plane logs and define success/rollback criteria.
- Test in a representative non-production cluster and run conformance/application smoke tests.
- Upgrade the control plane. Validate API, DNS and controllers.
- Upgrade EKS add-ons such as VPC CNI, CoreDNS and kube-proxy to compatible versions.
- Roll managed node groups or new launch-template/AMI nodes; cordon and drain old nodes within PDBs. For Karpenter, update NodeClass/AMI and use controlled drift.
- Upgrade autoscalers, CSI drivers, ingress, service mesh and observability components.
- Run functional, load and resilience tests; monitor errors, latency, pending Pods and node health; retain a documented recovery plan.


### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q39. Storage plugin issue blocks an upgrade.

### Direct interview answer

Stop the rollout, preserve healthy old nodes, inspect CSI controller/node Pods and CSINode/VolumeAttachment objects, and compare driver compatibility with both versions. Test attach/mount/snapshot in non-production. Roll the CSI driver forward or back, then resume with a canary node group. Never drain the last node hosting a critical volume without confirming rescheduling and attachment.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q40. ECR.

### Direct interview answer

Amazon ECR is a managed OCI container registry. Use immutable tags, scan on push/continuous scanning as configured, lifecycle policies, KMS encryption, private networking endpoints, cross-region/account replication and least-privilege repository policies. In CI authenticate with short-lived IAM/OIDC credentials, build once, push by digest and deploy that digest.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q41. Helm deployment and CI/CD integration.

### Direct interview answer

A chart packages templates, values and dependencies. CI should lint, render, validate, scan and publish the chart. CD uses environment-specific values, short-lived cluster identity, atomic upgrades and smoke tests. GitOps is preferred for auditability: CI updates the desired image digest in Git; Argo CD/Flux reconciles.


```bash
helm lint ./chart
helm dependency update ./chart
helm template api ./chart -f values-prod.yaml | kubeconform -strict
helm upgrade --install api ./chart \
  --namespace prod --create-namespace \
  -f values-prod.yaml --set image.digest=$DIGEST \
  --atomic --wait --timeout 10m
helm history api -n prod
helm rollback api <revision> -n prod
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q42. How many clusters and add-ons are you managing?

### Direct interview answer

This is experience-specific. Give your real current number, split by production/non-production and regions, then name only add-ons you actually operate. A strong structure is: “I manage X clusters across Y environments. Standard add-ons include CNI, CoreDNS, kube-proxy, CSI, ingress, metrics-server, autoscaling, ExternalDNS, certificate management, observability, policy and secret integration. We manage versions through IaC/GitOps and test upgrades in lower environments.”



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# 4. AWS architecture, networking, security and operations


## Q43. Explain the OSI model.

### Direct interview answer

OSI has seven conceptual layers: 1 Physical, 2 Data Link, 3 Network, 4 Transport, 5 Session, 6 Presentation and 7 Application. In troubleshooting, map symptoms to layers: link/NIC, ARP/VLAN, IP/routes, TCP/UDP/ports, session state, TLS/encoding and HTTP/DNS/application. Real systems often use the TCP/IP model, but OSI gives a disciplined diagnostic path.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q44. VPC, public/private subnets and NAT Gateway min/max.

### Direct interview answer

A subnet is public when its route table sends internet-bound traffic to an Internet Gateway and resources have suitable public addressing. Private subnets do not route directly to the IGW. For two private subnets across two AZs, the minimum is one NAT Gateway total, but that creates cross-AZ dependency and charges. A common resilient design is one NAT Gateway per AZ, so two. There is no architecture-driven “maximum” of two; AWS quotas and design choices apply. Route each private subnet to the NAT in its own AZ.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q45. Security Group vs NACL vs firewall/NSG.

### Direct interview answer

AWS Security Groups are stateful allow-list controls attached to ENIs; return traffic is automatically allowed. Network ACLs are stateless subnet controls with ordered allow and deny rules; return paths need explicit rules. Azure NSGs are stateful allow/deny controls for subnet/NIC traffic. A managed firewall adds centralized L3-L7 inspection, threat intelligence, egress filtering and policy governance. Use SG/NSG for workload micro-boundaries, NACLs for coarse subnet guardrails and firewalls for centralized inspection.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q46. Can a Security Group explicitly deny a port?

### Direct interview answer

No. AWS Security Groups contain allow rules only. Remove the allow rule or narrow its source/destination. For explicit deny, use NACLs, AWS Network Firewall, WAF for HTTP-layer controls or an application policy, chosen according to layer and scope.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q47. VPC peering and Transit Gateway.

### Direct interview answer

VPC peering provides private point-to-point routing between two VPCs and is non-transitive. Transit Gateway is a hub that connects many VPCs and on-prem networks with centralized route domains. Use peering for a small number of simple relationships; use TGW for scaled hub-and-spoke connectivity, segmentation and centralized inspection.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q48. Connect AWS to on-prem.

### Direct interview answer

Use site-to-site VPN for encrypted internet connectivity, Direct Connect for dedicated private connectivity, or both with VPN as backup. Terminate through Transit Gateway or a virtual private gateway, design BGP routes and failover, avoid overlapping CIDRs, centralize DNS resolution and test asymmetric routing, MTU and failover.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q49. TTL in DNS and flow.

### Direct interview answer

TTL tells recursive resolvers how long they may cache a record. A lower TTL can shorten convergence during planned changes but increases query load and does not flush already-cached entries. Flow: client asks local resolver, resolver follows root/TLD/authoritative delegation if cache misses, caches the answer for TTL, then returns it. Plan TTL reduction before migration, verify health, change records and later restore an appropriate TTL.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q50. Weighted routing.

### Direct interview answer

Route 53 weighted records distribute DNS answers proportionally among healthy records with the same name/type; they are not per-request load-balancer weights and are affected by resolver caching. At an ALB, weighted target groups can split actual requests between target groups, useful for canary or blue-green releases. Always define health checks and rollback thresholds.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q51. Design secure, highly available three-tier AWS architecture.

### Direct interview answer

Use Route 53, CloudFront and WAF at the edge; an internet-facing ALB in public subnets across at least two AZs; stateless application capacity in private subnets using EKS/ECS/ASG; and RDS Multi-AZ/ElastiCache in isolated data subnets. Use one NAT per AZ or VPC endpoints, least-privilege IAM, KMS encryption, Secrets Manager, backups, centralized logs, GuardDuty/Security Hub/Config and tested DR. Security groups allow only edge-to-ALB, ALB-to-app and app-to-database flows.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q52. EKS vs ECS vs Fargate.

### Direct interview answer

EKS is managed Kubernetes when portability/ecosystem and Kubernetes APIs matter. ECS is AWS-native container orchestration with lower Kubernetes operational complexity. Fargate is serverless compute capacity for EKS or ECS: no worker-node management, but with workload/feature/cost constraints. Choose from team skills, portability, control, workload shape and operational burden.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q53. EC2 vs Lambda.

### Direct interview answer

Use EC2 for long-running services, custom OS/runtime, steady high utilization, specialized networking or accelerators. Use Lambda for event-driven, short-lived functions with bursty traffic and minimal server management. Example: EC2/EKS hosts a continuously running API; Lambda reacts to S3 uploads to validate metadata. Consider duration, concurrency, cold starts, package/runtime limits, state and cost profile.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q54. Lambda logs missing though role exists.

### Direct interview answer

Check that the function is invoked and inspect invocation metrics/errors. Verify the execution role trust policy and CloudWatch Logs permissions, region/log group, VPC egress if the function calls external services, application logger configuration and reserved concurrency. Lambda normally creates /aws/lambda/<function-name> on first invocation if permitted. Test with a minimal handler and inspect CloudTrail for denied APIs.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q55. Simple Lambda file.

### Direct interview answer

This Python handler validates an EventBridge-style event and emits structured logs.


```python
import json, logging, os
log = logging.getLogger()
log.setLevel(os.getenv("LOG_LEVEL", "INFO"))

def lambda_handler(event, context):
    log.info(json.dumps({"request_id": context.aws_request_id,
                         "event": event}, default=str))
    return {"statusCode": 200,
            "body": json.dumps({"message": "processed"})}
```



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q56. IAM users vs roles; policy types.

### Direct interview answer

An IAM user is a long-lived identity and should be exceptional for workforce access. A role is assumed to obtain temporary credentials and is preferred for workloads, federation and cross-account access. Policies include identity-based, resource-based, permissions boundaries, Organizations SCPs, session policies and ACLs in services that support them. Effective permission is the intersection of applicable guardrails, with explicit deny winning.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q57. Two accounts: EC2 in A needs a token/secret in B.

### Direct interview answer

Attach an instance role in Account A that may call sts:AssumeRole on a tightly scoped role in Account B. The B role trust policy trusts the A role and its permissions allow only the specific secret action/resource. The application assumes B, receives temporary credentials and reads the token. Prefer Secrets Manager resource policy or cross-account role according to service support; encrypt with a KMS key policy that permits the role. Do not copy long-lived keys to EC2.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q58. Secure S3 and public-bucket incident response.

### Direct interview answer

Enable Block Public Access at account and bucket level, remove public ACL/policy grants, validate access points, and use least-privilege bucket policy with TLS and expected principals. Preserve CloudTrail/S3 data events and access logs, identify who changed the policy, assess exposure and objects accessed, rotate exposed credentials, notify incident response and add AWS Config/Security Hub controls. For private website delivery, keep S3 private and use CloudFront Origin Access Control.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q59. S3 policy, lifecycle and ACL difference.

### Direct interview answer

A bucket policy is a resource-based JSON policy. ACLs are legacy, coarse grants and should usually be disabled with Bucket owner enforced Object Ownership. Lifecycle rules transition or expire object versions and abort incomplete multipart uploads. Test prefixes/tags, versioning and retention/legal requirements before expiration.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q60. RDS design and migration with minimal downtime.

### Direct interview answer

Size from CPU, memory, IOPS, storage growth, connections, engine features, RTO/RPO and licensing. Use Multi-AZ for HA, read replicas for read scaling, encryption, private subnets, backups and monitoring. For minimal-downtime migration, use schema conversion if required, initial full load, continuous CDC with AWS DMS/native replication, validation, application freeze or dual-write only if designed, final lag drain, controlled cutover, monitoring and rollback window. State actual database size and replica count only from your project.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q61. Cost optimization in AWS.

### Direct interview answer

Start with allocation and baselines, then remove waste and change architecture.

- Tag resources; use Cost Explorer, CUR, budgets and anomaly detection.
- Delete idle EBS, EIPs, snapshots, old load balancers and non-production schedules.
- Rightsize EC2/RDS from utilization and memory/IO data; adopt Graviton where tested.
- Use Savings Plans/Reserved Instances for steady baselines and Spot for interruptible capacity.
- Use S3 lifecycle/tiering, VPC endpoints and CDN/cache to control storage and transfer.
- Tune autoscaling, requests/limits and data retention; review unit cost per request/customer.
- Enforce cost checks in IaC and review savings against SLOs so performance is not sacrificed.


### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q62. SSL/TLS flow with Let’s Encrypt/Certbot and AWS.

### Direct interview answer

The server presents a certificate chain; the client validates hostname, validity, trust and revocation signals, then performs a key exchange to derive symmetric session keys. Let’s Encrypt uses ACME challenges; Certbot automates issuance and renewal. In AWS, ACM can issue and renew certificates attached to ALB, CloudFront and API Gateway, while private CA serves internal PKI. Redirect HTTP to HTTPS, enforce modern policies, monitor expiry and protect private keys.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q63. Top infrastructure attacks and mitigations.

### Direct interview answer

Common categories are credential theft, exposed services/misconfiguration, vulnerable software/supply-chain compromise, DDoS and ransomware/data exfiltration. Mitigate with phishing-resistant MFA and temporary credentials; least privilege and continuous configuration checks; patching, SBOM/signing/scanning; WAF/Shield/rate limiting/autoscaling; segmentation, immutable backups, KMS, detection and tested incident response.



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q64. Vulnerability scanning for AWS instances.

### Direct interview answer

Use Amazon Inspector for continuous EC2/ECR/Lambda findings where applicable, Systems Manager inventory/patching, image pipeline scans before release and authenticated host scans when required. Prioritize by exploitability, internet exposure and business criticality; patch through immutable AMI replacement or controlled SSM maintenance windows; verify closure and keep exception expiry dates.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q65. EC2 production security checklist.

### Direct interview answer

Use approved hardened AMI, IMDSv2, encrypted EBS with customer-managed keys when required, least-privilege instance profile, no public IP unless justified, restricted SG, SSM Session Manager instead of inbound SSH, patching, endpoint protection, centralized logs, backups, deletion protection/termination safeguards where suitable and tagging/config rules. Never put credentials in user data.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q66. EBS-backed vs instance-store-backed.

### Direct interview answer

EBS volumes are network-attached and persist independently according to delete-on-termination settings; they support snapshots and stop/start for EBS-root instances. Instance store is physically attached ephemeral storage: data is lost on stop/termination or host failure. Use it only for caches, scratch or replicated data.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q67. How do you access a private instance when SSH fails?

### Direct interview answer

Prefer SSM Session Manager with a working agent, instance role and network path through NAT or VPC endpoints. Alternatives include EC2 Instance Connect Endpoint or a controlled bastion. For recovery, use serial console if supported, or stop and attach the root EBS volume to a rescue instance to fix sshd, permissions or disk issues, then reattach. Preserve evidence if compromise is suspected.



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
## Q68. What is DRS failover/failback?

### Direct interview answer

AWS Elastic Disaster Recovery continuously replicates source block data to a low-cost staging area. During recovery it launches recovery instances in the target AWS region/account using launch settings. Failover is a tested cutover to those instances; failback reverses replication and returns service after the source is ready. Define RPO/RTO, run drills, validate DNS, identity, dependencies and data consistency.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q69. EventBridge via Terraform.

### Direct interview answer

Create a rule with an event pattern, target and permission allowing EventBridge to invoke the target.


```hcl
resource "aws_cloudwatch_event_rule" "ec2_state" {
  name = "ec2-state-change"
  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
  })
}
resource "aws_cloudwatch_event_target" "lambda" {
  rule = aws_cloudwatch_event_rule.ec2_state.name
  arn  = aws_lambda_function.handler.arn
}
resource "aws_lambda_permission" "events" {
  statement_id  = "AllowEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.handler.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.ec2_state.arn
}
```



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
# 5. Terraform and infrastructure delivery


## Q70. Terraform init, fmt, validate, plan and apply.

### Direct interview answer

init downloads providers/modules and configures the backend. fmt canonicalizes HCL formatting. validate checks configuration syntax and internal consistency. plan compares configuration, state and refreshed remote objects to propose changes. apply executes an approved plan. In CI, persist the binary plan and apply exactly that reviewed artifact.



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q71. What are state and lock files?

### Direct interview answer

State maps Terraform resource addresses to real objects and stores attributes/dependencies. Store it remotely, encrypt it, restrict access, enable versioning and use supported state locking. .terraform.lock.hcl pins provider selections/checksums for reproducible initialization and should normally be committed. It is not the state-lock mechanism.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q72. What if tfstate is deleted?

### Direct interview answer

Stop applies immediately. If remote backend versioning/backup exists, restore the latest known-good version and run plan-only validation. Without backup, recreate matching configuration and import every existing resource, or use configuration-driven import, then reconcile drift carefully. Never apply an empty state against existing production because Terraform may attempt duplicate creation. Some relationships and sensitive historical attributes may require manual reconstruction.



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q73. Create B without deleting A.

### Direct interview answer

Keep A in configuration/state and add B using a new resource address, for_each key or module instance. Review the plan. If renaming an address, use a moved block or terraform state mv so Terraform does not interpret it as destroy/create.


```hcl
resource "aws_instance" "app" {
  for_each = {
    A = { size = "t3.small" }
    B = { size = "t3.small" }
  }
  ami           = var.ami_id
  instance_type = each.value.size
  tags = { Name = each.key }
}
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q74. Terraform provisioners and null_resource behavior.

### Direct interview answer

file, local-exec and remote-exec exist, but provisioners are last resort because they are imperative, hard to model and can leave partial state. Prefer cloud-init/user_data, image baking, configuration management or provider resources. A null_resource runs when created or replaced. triggers = { always = timestamp() } forces replacement and execution on every apply, making plans perpetually changing; use it only with explicit idempotency and safer alternatives considered.



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q75. Map of objects example.

### Direct interview answer

A map of objects gives stable keys plus typed structured values, ideal with for_each.


```hcl
variable "instances" {
  type = map(object({
    instance_type = string
    subnet_id     = string
    private_ip    = optional(string)
  }))
}
resource "aws_instance" "this" {
  for_each      = var.instances
  ami           = var.ami_id
  instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_id
  private_ip    = try(each.value.private_ip, null)
  tags = { Name = each.key }
}
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q76. Dynamic blocks.

### Direct interview answer

A dynamic block generates repeatable nested blocks when the provider schema expects blocks rather than a list argument. Use it sparingly; overuse harms readability.


```bash
dynamic "ip_restriction" {
  for_each = var.ip_restrictions
  content {
    ip_address = ip_restriction.value.ip_address
    name       = ip_restriction.value.name
  }
}
```



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q77. Terraform EC2 in public subnet with custom SG and user data.

### Direct interview answer

Below is an interview-sized example. Production should add remote state, modules, encryption, IMDSv2, logging and policy checks.


```bash
terraform {
  required_providers { aws = { source = "hashicorp/aws" } }
}
provider "aws" { region = var.region }
resource "aws_vpc" "main" { cidr_block = "10.20.0.0/16" }
resource "aws_internet_gateway" "igw" { vpc_id = aws_vpc.main.id }
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.20.1.0/24"
  map_public_ip_on_launch = true
}
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.igw.id }
}
resource "aws_route_table_association" "public" {
  subnet_id = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
  ingress { from_port=443; to_port=443; protocol="tcp"; cidr_blocks=var.allowed_cidrs }
  egress  { from_port=0; to_port=0; protocol="-1"; cidr_blocks=["0.0.0.0/0"] }
}
resource "aws_instance" "web" {
  ami=var.ami_id; instance_type="t3.micro"; subnet_id=aws_subnet.public.id
  vpc_security_group_ids=[aws_security_group.web.id]
  user_data = <<-EOF
    #!/bin/bash
    dnf -y install nginx
    systemctl enable --now nginx
  EOF
  metadata_options { http_tokens = "required" }
  tags={Name="interview-web"}
}
```



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q78. Multiple S3 buckets and 7-day deletion.

### Direct interview answer

S3 lifecycle expiration deletes objects, not the Terraform bucket resource itself after seven days. Terraform does not have a native time-to-live for arbitrary resources. Use a lifecycle rule for objects and a controlled external cleanup workflow for ephemeral buckets.


```hcl
variable "buckets" { type = set(string) }
resource "aws_s3_bucket" "this" { for_each = var.buckets; bucket = each.value }
resource "aws_s3_bucket_lifecycle_configuration" "this" {
  for_each = aws_s3_bucket.this
  bucket   = each.value.id
  rule {
    id = "expire-7-days"; status = "Enabled"
    expiration { days = 7 }
  }
}
```



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q79. Terraform through CI/CD.

### Direct interview answer

Use workload identity/OIDC rather than static keys. PR pipeline runs fmt, validate, lint, security/policy scans and plan using a read/write remote backend lock. Store plan as a protected artifact and post a summary. Require review for production. Apply from the protected main branch/environment using the reviewed commit and plan. Use separate state per environment, least-privilege roles, drift detection, audit logs and an emergency break-glass process.



### Senior-level depth
- Treat Terraform configuration, state, provider schema and live infrastructure as four separate things. A safe plan reconciles all four.
- Production design should include a remote encrypted backend, state versioning, locking, separate environment state, OIDC-based CI identity, reviewed plans and drift detection.
- Explain replacement risk. Attributes marked ForceNew can turn a harmless-looking edit into destroy/create, so saved-plan review and lifecycle design matter.

### Validation and recovery checklist
1. Freeze applies when state integrity is uncertain.
2. Back up or version the state before state operations.
3. Run `terraform plan` and inspect replacements and destroys explicitly.
4. Use import or moved blocks to repair resource addressing.
5. Apply the reviewed saved plan, then run a second plan expecting no unintended changes.

### Interview example framing
“I do not use `apply` as a discovery tool in production. I establish backend integrity, produce a reviewed plan, verify replacement behavior, and only then execute with a short-lived environment role.”

---
## Q80. New AWS environment project answer.

### Direct interview answer

I start with requirements and guardrails, then build reusable Terraform modules for organization/account baseline, VPC/subnets/routes/endpoints, security, KMS, logs, EKS/ECS/EC2, databases and observability. Environments consume versioned modules with separate state and variables. CI validates and plans on PR; CD assumes an environment role and applies after approval. Post-deploy smoke, security and cost tests prove readiness. DNS/certificates, backups, dashboards, alerts, runbooks and DR are part of definition of done.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
# 6. CI/CD, Git, Helm, Jenkins and GitHub Actions


## Q81. Describe your current CI/CD process and tools.

### Direct interview answer

Use this adaptable truthful structure: GitHub/GitLab for source and PRs; Jenkins/GitHub Actions/Azure DevOps for orchestration; Maven/npm/pip for build; unit/integration tests; SonarQube and SAST; dependency, secret, IaC and container scans; artifact registry/ECR/ACR; Helm/Kustomize; and Argo CD/Flux or controlled deployment jobs. Production requires protected branches, OIDC, approvals, immutable artifacts, progressive delivery, automated verification and rollback. Replace tool names with your real stack.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q82. Declarative vs scripted Jenkins pipelines.

### Direct interview answer

Declarative syntax is structured, validates early and provides standard sections such as agent, stages, post and options. Scripted syntax is Groovy code with maximum flexibility but more complexity. Prefer Declarative and use small script blocks only when necessary.


```bash
pipeline {
  agent any
  options { disableConcurrentBuilds() }
  stages {
    stage('Test') { steps { sh 'make test' } }
    stage('Build') { steps { sh 'docker build -t app:${BUILD_NUMBER} .' } }
    stage('Approve') {
      when { branch 'main' }
      steps { input message: 'Deploy production?' }
    }
  }
  post { always { junit 'reports/*.xml' } }
}

node {
  stage('Test') { sh 'make test' }
  stage('Build') { sh 'make build' }
}
```



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q83. GitHub Actions workflow: build, scan and push.

### Direct interview answer

Pin third-party actions to commit SHAs, use least permissions, OIDC for cloud, protected environments and provenance. A simplified structure is:


```bash
name: container
on:
  push: {branches: [main]}
  pull_request: {branches: [main]}
  workflow_dispatch:
permissions:
  contents: read
  id-token: write
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@<pinned-commit-sha>
    - run: make test
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@<pinned-commit-sha>
    - run: docker build -t "$IMAGE:$GITHUB_SHA" .
    - run: trivy image --exit-code 1 --severity HIGH,CRITICAL "$IMAGE:$GITHUB_SHA"
    - run: echo "authenticate with OIDC, then push by immutable SHA/digest"
```



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q84. jobs in parallel, matrix, needs and runs-on.

### Direct interview answer

Jobs without dependencies run in parallel. needs establishes a dependency and allows outputs from prerequisite jobs. strategy.matrix expands one job over multiple dimensions. runs-on selects the runner label. Use GitHub-hosted runners for convenience and clean ephemeral environments; self-hosted runners when private network access, specialized hardware, caching or compliance requires it. Harden and isolate self-hosted runners and prefer ephemeral instances.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q85. Pipeline triggers for branch, ignore and PR.

### Direct interview answer

Use on.push.branches and on.pull_request.branches. Branch filters refer to the target branch for pull requests. paths/paths-ignore can restrict file changes, but understand required-check behavior before combining filters.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q86. Manual trigger.

### Direct interview answer

Add workflow_dispatch under on. Inputs can collect environment or version. Protect production with a GitHub Environment and required reviewers rather than trusting an input alone.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q87. Secure third-party marketplace actions.

### Direct interview answer

Pin to a full commit SHA, review source and permissions, allow-list actions at organization level, minimize GITHUB_TOKEN permissions, avoid secrets on untrusted forks, use dependency update tooling, scan workflow changes, and isolate self-hosted runners. Treat workflow code as production code.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q88. Build artifacts vs pipeline artifacts.

### Direct interview answer

In Azure DevOps, modern Pipeline Artifacts are optimized for Azure Pipelines and are generally preferred for pipeline-to-pipeline/stage sharing. Build Artifacts are the older mechanism and can fit compatibility scenarios. Regardless of type, use immutable names, checksums, retention and promotion of the same artifact between environments.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q89. Reusable templates for 50 applications.

### Direct interview answer

Create a small versioned template contract: language/build type, test command, artifact path, image metadata, deployment target and policy profile. Central templates implement standard stages and security controls; application repositories supply parameters. Release template versions semantically, test them with representative apps, pin consumers, and provide escape hatches with governance rather than copying YAML.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q90. What if a pipeline is deleted?

### Direct interview answer

Recreate it from version-controlled YAML and infrastructure/API configuration. Prevent recurrence with least-privilege project roles, protected pipeline settings, change approval, audit logs, backups/export of non-YAML metadata and policy that production pipelines are code, not click-built.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q91. Pipeline secrets and CyberArk/Key Vault.

### Direct interview answer

Pipelines should retrieve short-lived secrets just in time from a managed vault using workload identity. CyberArk can broker privileged credentials or rotate service accounts; Azure Key Vault/AWS Secrets Manager can supply application secrets. Mask logs, restrict scopes, avoid command-line echo, rotate automatically and record access. Prefer OIDC federation so the pipeline receives temporary cloud credentials without stored keys.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q92. SonarQube integration and optimal trigger.

### Direct interview answer

Run fast lint/unit checks on every push, SonarQube PR analysis with quality gate on every pull request, and full main-branch analysis after merge. Jenkins uses scanner/plugin credentials from its credential store, runs analysis, waits for the webhook-driven quality gate and blocks promotion on failure. Avoid exposing tokens or making production deployment depend on an unbounded polling loop.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q93. Git push, pull, fetch, tags, branches.

### Direct interview answer

git push publishes local refs. git fetch updates remote-tracking refs without changing the working branch. git pull performs fetch then merge or rebase according to configuration. Tags name specific commits, commonly releases; annotated signed tags are preferable for releases. Branch types are a team convention: main, feature, release, hotfix and maintenance. Keep them short-lived where possible.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q94. merge vs rebase; force-with-lease vs force.

### Direct interview answer

Merge preserves parallel history and adds a merge commit. Rebase rewrites commits onto a new base for linear history, so avoid rebasing shared public history. --force overwrites the remote ref unconditionally. --force-with-lease refuses if the remote advanced beyond what you last observed, so it is safer but still requires coordination.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q95. Delete the last two commits.

### Direct interview answer

If commits are local, git reset --hard HEAD~2 deletes them and working changes. If shared, prefer git revert for each commit to create auditable inverse commits. Rewriting shared history requires coordination and protected-branch exceptions.


```bash

### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# Local/unpublished
 git reset --hard HEAD~2
# Shared
 git revert HEAD
 git revert HEAD~1
```


## Q96. Blue-green vs canary and rollback.

### Direct interview answer

Blue-green runs old and new full environments and switches traffic, enabling fast rollback at higher cost. Canary shifts a small percentage first and increases based on metrics, reducing blast radius but requiring strong observability and traffic control. Kubernetes rollback uses kubectl rollout history/status and kubectl rollout undo deployment/app --to-revision=N. Database changes must be backward compatible because application rollback cannot undo destructive schema changes.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
# 7. Docker, Linux, Ansible and Python


## Q97. Docker architecture on Linux.

### Direct interview answer

The Docker CLI calls dockerd through a Unix socket/API. dockerd manages images, networks, volumes and build requests and delegates container execution through containerd and an OCI runtime such as runc. Linux namespaces isolate processes/network/mounts; cgroups govern resources; layered filesystems implement images. Protect the Docker socket because access is effectively root-equivalent.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q98. Dockerfile, COPY vs ADD, CMD vs ENTRYPOINT.

### Direct interview answer

A Dockerfile is an ordered build recipe. COPY copies local build-context files and is preferred. ADD also supports archive extraction and remote URL behavior, which can be surprising. ENTRYPOINT defines the executable; CMD supplies default arguments or, without ENTRYPOINT, the default command. Runtime arguments replace CMD but append to exec-form ENTRYPOINT.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q99. Simple secure multistage Dockerfile.

### Direct interview answer

Build dependencies in one stage and copy only the runtime artifact into a minimal non-root image.


```bash
FROM python:3.13-slim AS build
WORKDIR /src
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM python:3.13-slim
RUN useradd --system --uid 10001 app
WORKDIR /app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels
COPY --chown=app:app . .
USER 10001
EXPOSE 8080
ENTRYPOINT ["python", "app.py"]
```



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q100. How do you reduce image size?

### Direct interview answer

Use small trusted bases, multistage builds, .dockerignore, pinned dependencies, one package-manager layer with cache cleanup, no compilers/debug tools in runtime, and copy only required artifacts. Do not optimize solely for size: patchability, provenance and CVE posture matter. Inspect layers with docker history and verify the app after changes.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q101. Image EXPOSE 8080 but app listens elsewhere.

### Direct interview answer

EXPOSE is metadata and does not reconfigure the application. Container port mappings and Kubernetes targetPort must reach the actual listening port and interface. Check ss -lntp inside the container, application config and Service targetPort. A process bound to 127.0.0.1 may not be reachable through the Pod IP.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q102. Docker disk usage too high.

### Direct interview answer

Measure before deleting: docker system df -v, du under Docker root, container logs and volumes. Rotate json-file logs, prune unused stopped containers/images/build cache with change control, clean registry caches, and move persistent data to managed volumes. Never blindly delete /var/lib/docker on a running host.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q103. Linux load average.

### Direct interview answer

Load average is the average number of tasks runnable or in uninterruptible sleep over 1, 5 and 15 minutes. Compare it with logical CPU count, CPU utilization, run queue and I/O wait. A load of 8 on 8 CPUs differs from 8 on 2 CPUs, and high load can be disk I/O rather than CPU.



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
## Q104. /var 90% full: immediate action.

### Direct interview answer

Confirm filesystem and inode usage, identify top consumers, protect service availability, then clean safely.


```bash
df -hT /var; df -ih /var
sudo du -xhd1 /var | sort -h
sudo journalctl --disk-usage
sudo lsof +L1   # deleted but still-open files

### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# rotate/compress via supported service policy; do not rm active DB files
```


## Q105. df -h shows space but df -i is full.

### Direct interview answer

The filesystem has exhausted inodes, usually because of many small files. Find high file-count directories, stop the producer if needed, archive/delete safely, then correct retention. Increasing capacity may not help unless the filesystem gains more inodes; a rebuild with suitable inode density may be required.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q106. Attach/detach filesystem and add 50 GB to /opt via LVM.

### Direct interview answer

Identify the correct device and backups first. Add/expand a disk, create PV, extend VG/LV, then grow the filesystem online. Commands differ by filesystem.


```bash
lsblk -f; pvs; vgs; lvs
sudo pvcreate /dev/nvme1n1
sudo vgextend vg_app /dev/nvme1n1
sudo lvextend -L +50G /dev/vg_app/lv_opt
sudo xfs_growfs /opt          # XFS

### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
# or: sudo resize2fs /dev/vg_app/lv_opt  # ext4
findmnt /opt; df -hT /opt
# detach: stop users, umount /opt; update /etc/fstab carefully
```


## Q107. SSH locked out with no root.

### Direct interview answer

Use an approved out-of-band method: cloud serial console, SSM Session Manager, console rescue mode or attach the root disk to a rescue VM. Fix network, disk, sshd config, authorized_keys ownership/permissions or expired credentials; validate before reboot. If this may be compromise, preserve evidence and follow incident response rather than modifying the host first.



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
## Q108. Server performance troubleshooting.

### Direct interview answer

Start with scope and time: what changed, which requests and whether all hosts are affected. Check uptime/load, CPU, memory/swap, disk latency/capacity/inodes, network errors/connections, process/thread and application metrics, then dependencies. Correlate with deployments and logs. Form one hypothesis at a time and validate after mitigation.


```bash
uptime; top; vmstat 1; pidstat 1
free -m; sar -r 1
 iostat -xz 1; df -hT; df -ih
ss -s; ip -s link; sar -n DEV 1
journalctl -p warning --since '-30 min'
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q109. Network details and traffic flow commands.

### Direct interview answer

Use ip addr/route/neigh, ss, ping where allowed, dig, traceroute/mtr, curl -v, tcpdump and cloud flow logs. Capture minimally and protect packet data because it may contain sensitive content.


```bash
ip -br addr; ip route
ss -tulpn
dig +trace app.example.com
curl -vk https://app.example.com/health
sudo tcpdump -ni any host 10.0.1.10 and port 443
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q110. Zombie process.

### Direct interview answer

A zombie has exited but its parent has not called wait(), so only a process-table entry remains. It consumes negligible memory but many zombies can exhaust PIDs. Identify PPID, fix/restart the parent gracefully; killing the zombie itself has no effect.



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
## Q111. Last 15 lines, line count and ERR count.

### Direct interview answer

Use tail -n 15 file, wc -l < file and grep -c "ERR" file. For exact token matching use grep -cw and define case sensitivity explicitly.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q112. Passwordless server authentication.

### Direct interview answer

Use SSH public keys or certificates, not shared passwords. Generate an Ed25519 key, install the public key using ssh-copy-id through an approved channel, verify host keys, protect the private key and disable password/root login only after testing and preserving break-glass access.


```bash
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519
ssh-copy-id user@server
ssh -o IdentitiesOnly=yes user@server
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q113. Ansible playbook and client requirements.

### Direct interview answer

Managed Linux hosts generally need SSH access and a suitable Python interpreter for most modules; network appliances may use APIs/CLI. Inventory, privilege escalation, idempotent modules, variables, secrets in Vault, check mode and handlers are core considerations.


```bash
- name: Configure web servers
  hosts: web
  become: true
  serial: 2
  vars:
    package_name: nginx
  tasks:
    - name: Install package
      ansible.builtin.package:
        name: "{{ package_name }}"
        state: present
    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: Restart nginx
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```



### Senior-level depth
- Emphasize idempotency, deterministic inventory, variable precedence, secrets management, check mode, controlled concurrency and handlers.
- Prefer purpose-built modules over shell commands. A playbook should converge state and remain safe when run repeatedly.
- For production rollout, use `serial`, health checks and clear failure thresholds.

### Validation checklist
1. Run syntax check, lint and check mode where meaningful.
2. Test against a small inventory subset.
3. Use `--diff` carefully because output may contain sensitive data.
4. Confirm handlers fire only on change.
5. Re-run and expect no unintended changes.

### Interview example framing
“I design Ansible automation to be idempotent and observable. I canary on a subset, limit concurrency, use handlers for controlled restart, and prove the second run is clean.”

---
## Q114. Ansible import vs include; roles, inventory and debugging.

### Direct interview answer

Imports are static and expanded during playbook parsing; includes are dynamic at runtime and can depend on variables/conditions. Roles package tasks, handlers, templates, files, vars and defaults. Inventory groups hosts and variables. Debug with --syntax-check, --check --diff, -vvv, --limit, list-tasks/hosts and module-level connectivity tests. If one of twenty hosts times out, compare DNS, route, SSH, firewall, Python, sudo and host vars against a working host.



### Senior-level depth
- Emphasize idempotency, deterministic inventory, variable precedence, secrets management, check mode, controlled concurrency and handlers.
- Prefer purpose-built modules over shell commands. A playbook should converge state and remain safe when run repeatedly.
- For production rollout, use `serial`, health checks and clear failure thresholds.

### Validation checklist
1. Run syntax check, lint and check mode where meaningful.
2. Test against a small inventory subset.
3. Use `--diff` carefully because output may contain sensitive data.
4. Confirm handlers fire only on change.
5. Re-run and expect no unintended changes.

### Interview example framing
“I design Ansible automation to be idempotent and observable. I canary on a subset, limit concurrency, use handlers for controlled restart, and prove the second run is clean.”

---
## Q115. Python list vs tuple, slicing, inheritance, decorators and module.

### Direct interview answer

A list is mutable; a tuple is immutable and can be hashable if its elements are hashable. s[0:4] returns indexes 0 through 3. Inheritance lets a class reuse/override behavior. A decorator wraps a function/class to add behavior. A module is a .py file imported by name.


```bash

### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
# mymodule.py
def hello():
    print("Hello, World")

# reverse and slice
text = "DevOps"
print(text[::-1])
print(text[0:4])
```


## Q116. Python reverse words and remove first duplicate.

### Direct interview answer

Clarify whether “reverse words” means order or characters. This example reverses order and removes only the first repeated occurrence while retaining later data.


```bash
words = ["cloud", "devops", "sre"]
print(words[::-1])

def remove_first_duplicate(items):
    seen = set()
    for i, item in enumerate(items):
        if item in seen:
            return items[:i] + items[i+1:]
        seen.add(item)
    return items[:]
```



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
# 8. Monitoring, SRE and incident response


## Q117. SLI, SLO and SLA.

### Direct interview answer

An SLI is a measured reliability signal, such as successful requests / valid requests or latency under 300 ms. An SLO is the internal target for that SLI over a window, such as 99.9% monthly availability. An SLA is an external commitment with defined remedies. Use error budgets to balance release velocity and reliability.



### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
## Q118. Prometheus and Grafana installation.

### Direct interview answer

For Kubernetes, use a maintained Helm chart such as kube-prometheus-stack in a dedicated monitoring namespace. Configure persistent storage, HA where justified, retention, remote write, ServiceMonitors/PodMonitors, Alertmanager routes, RBAC and ingress/authentication. Treat dashboards and alerts as code and test them. Verify scrape targets and cardinality before broad rollout.



### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
## Q119. CPU and memory dashboard metrics.

### Direct interview answer

For CPU utilization, graph rate(container_cpu_usage_seconds_total[5m]) and compare with requests/limits; for nodes compare non-idle CPU to cores. For memory use working set and compare with requests/limits, while showing OOM kills. Add saturation, throttling, restarts and request latency so CPU alone does not mislead. Exclude infrastructure/Pause containers and aggregate by cluster, namespace, workload and Pod.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q120. Grafana extensions.

### Direct interview answer

Install signed data-source, panel or app plugins through supported provisioning/container configuration, pin versions and review permissions. In managed Grafana use the service’s plugin catalog. For custom extensions, follow Grafana plugin tooling, sign the plugin, test compatibility and manage it as a versioned artifact.



### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
## Q121. ELK architecture and collection agents.

### Direct interview answer

Beats or Elastic Agent/Fluent Bit/Fluentd collect host/container logs, enrich with Kubernetes metadata and send directly or through Logstash. Elasticsearch indexes and retains data; Kibana provides search and dashboards. Use structured JSON, index lifecycle management, backpressure/buffering, TLS, authentication, field governance and careful cardinality. Avoid logging secrets.



### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
## Q122. CloudWatch log groups vs CloudTrail.

### Direct interview answer

A CloudWatch Logs log group is a container for log streams with retention, encryption, subscriptions and metric filters. CloudTrail records AWS API activity and account events for audit. “Log trail” is often an imprecise phrase. A CloudTrail trail delivers events to S3 and optionally CloudWatch Logs; the services solve different problems.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q123. Airflow job failure debugging.

### Direct interview answer

Check DAG parse errors, task instance logs, retries, upstream/downstream state, scheduler health, executor/worker capacity, pools/queues, connection/secret resolution and external dependency status. Reproduce with airflow tasks test where appropriate. Make tasks idempotent before clearing/retrying, and distinguish code failure from orchestration/resource failure.



### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
## Q124. JSON ingestion into key fields.

### Direct interview answer

Define a schema, parse JSON at the collector or stream processor, normalize timestamps/severity/service/correlation IDs, preserve the raw event for reprocessing, reject/quarantine malformed messages and enforce size/cardinality limits. Examples include Fluent Bit parsers to Elasticsearch, CloudWatch Logs subscription to Lambda/Firehose, or Logstash json filter. Never flatten unbounded user keys into index fields.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q125. Sudden traffic surge makes app unresponsive.

### Direct interview answer

Declare and triage the incident, protect the system, then recover deliberately.

- Confirm user impact and bottleneck using RED metrics: rate, errors, duration; check saturation and dependencies.
- Apply WAF/rate limits, cache/CDN, queueing and load shedding; prioritize critical endpoints.
- Scale application replicas and nodes only after confirming DB/cache/network headroom.
- Rollback recent changes if correlated; disable expensive features with tested flags.
- Protect the database with connection limits, read replicas/cache and circuit breakers.
- Communicate impact, mitigation and next update; preserve timeline. After recovery run RCA, capacity tests and alert/runbook improvements.


### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q126. SEV-1 and failed production deployment communication.

### Direct interview answer

Assign incident commander, operations lead, communications lead and scribe. Stop further changes, assess blast radius, roll back or fail over using the safest known path, and update stakeholders with facts: impact, start time, affected services, mitigation, owner and next checkpoint. Avoid blame and speculative ETAs. After restoration validate data, close monitoring gaps and run a blameless RCA with owned actions.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q127. Linux RCA example.

### Direct interview answer

Example structure: intermittent API latency mapped to high iowait. vmstat/iostat showed a saturated volume; lsof and application logs traced it to an unbounded debug log plus synchronous compression. Immediate mitigation rotated logs and moved compression off peak. Permanent fixes added retention, async shipping, disk-latency alerts, ephemeral-storage limits and a load test. State only outcomes you can substantiate from your own experience.



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
# 9. Azure and cross-cloud topics


## Q128. Managed identity vs service principal.

### Direct interview answer

A service principal is an application identity in Microsoft Entra ID and may use a secret, certificate or federated credential. A managed identity is an Azure-managed service principal whose credential lifecycle is handled by Azure and is exposed to the attached resource. Use managed identity for Azure-hosted workloads when supported; use a federated service principal for external CI or multi-cloud identities. Avoid long-lived client secrets.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q129. Private Endpoint and ExpressRoute.

### Direct interview answer

A Private Endpoint gives an Azure PaaS resource a private IP in your VNet through Private Link; private DNS must resolve the service name to that IP. ExpressRoute provides private dedicated connectivity from on-premises to Microsoft networks through a connectivity provider. They solve different layers and are often combined.



### Senior-level depth
- Separate Azure control-plane RBAC, Kubernetes API authorization, cluster managed identity and pod workload identity.
- Prefer private endpoints, private DNS, managed identities/federation and scoped data-plane roles instead of account keys.
- Trace traffic through DNS, UDRs, NSGs, firewalls and service endpoints/private links.

### Validation checklist
1. Check the signed-in principal and role assignment scope.
2. Verify private DNS resolution and effective routes.
3. Inspect NSG flow logs and application connection errors.
4. Confirm token audience, issuer and federated credential for workload identity.
5. Validate access from the intended pod and denial from an unintended pod.

### Interview example framing
“I use identity rather than stored keys, restrict the network path with Private Link, and validate both the intended access and the expected denial path.”

---
## Q130. Can VMs in different VNets communicate?

### Direct interview answer

Yes, using VNet peering, Virtual WAN/hub routing, VPN gateways or ExpressRoute, provided CIDRs do not overlap and NSGs, UDRs, firewalls and DNS allow the flow. Peering is non-transitive unless a hub design and gateway transit/routing are configured appropriately.



### Senior-level depth
- Separate Azure control-plane RBAC, Kubernetes API authorization, cluster managed identity and pod workload identity.
- Prefer private endpoints, private DNS, managed identities/federation and scoped data-plane roles instead of account keys.
- Trace traffic through DNS, UDRs, NSGs, firewalls and service endpoints/private links.

### Validation checklist
1. Check the signed-in principal and role assignment scope.
2. Verify private DNS resolution and effective routes.
3. Inspect NSG flow logs and application connection errors.
4. Confirm token audience, issuer and federated credential for workload identity.
5. Validate access from the intended pod and denial from an unintended pod.

### Interview example framing
“I use identity rather than stored keys, restrict the network path with Private Link, and validate both the intended access and the expected denial path.”

---
## Q131. Build and push to ACR, then deploy.

### Direct interview answer

Authenticate with workload identity/service connection, build, scan and push an immutable tag/digest. Grant AKS pull permission through managed identity or workload identity integration. Reference the ACR image in the Pod/Deployment and avoid admin credentials.


```bash
az acr login --name myregistry
az acr build --registry myregistry --image api:${GIT_SHA} .
kubectl set image deployment/api api=myregistry.azurecr.io/api@sha256:<digest>
```



### Senior-level depth
- Separate Azure control-plane RBAC, Kubernetes API authorization, cluster managed identity and pod workload identity.
- Prefer private endpoints, private DNS, managed identities/federation and scoped data-plane roles instead of account keys.
- Trace traffic through DNS, UDRs, NSGs, firewalls and service endpoints/private links.

### Validation checklist
1. Check the signed-in principal and role assignment scope.
2. Verify private DNS resolution and effective routes.
3. Inspect NSG flow logs and application connection errors.
4. Confirm token audience, issuer and federated credential for workload identity.
5. Validate access from the intended pod and denial from an unintended pod.

### Interview example framing
“I use identity rather than stored keys, restrict the network path with Private Link, and validate both the intended access and the expected denial path.”

---
## Q132. Azure VM CPU/memory alert and action group.

### Direct interview answer

Create an Azure Monitor metric alert for Percentage CPU. Guest memory generally requires Azure Monitor Agent and a Data Collection Rule so guest metrics/logs reach the workspace. Define scope, condition, aggregation/window, threshold or dynamic logic, severity and an Action Group containing email/SMS/webhook/ITSM/automation actions. Test the action, document ownership and tune to avoid flapping.



### Senior-level depth
- Separate Azure control-plane RBAC, Kubernetes API authorization, cluster managed identity and pod workload identity.
- Prefer private endpoints, private DNS, managed identities/federation and scoped data-plane roles instead of account keys.
- Trace traffic through DNS, UDRs, NSGs, firewalls and service endpoints/private links.

### Validation checklist
1. Check the signed-in principal and role assignment scope.
2. Verify private DNS resolution and effective routes.
3. Inspect NSG flow logs and application connection errors.
4. Confirm token audience, issuer and federated credential for workload identity.
5. Validate access from the intended pod and denial from an unintended pod.

### Interview example framing
“I use identity rather than stored keys, restrict the network path with Private Link, and validate both the intended access and the expected denial path.”

---
## Q133. Useful KQL troubleshooting patterns.

### Direct interview answer

Adjust table names to your configured data source.


```bash
// Heartbeat gaps
Heartbeat
| summarize LastSeen=max(TimeGenerated) by Computer
| where LastSeen < ago(10m)

// High CPU from performance counters
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvgCPU > 80

// Error events
Syslog
| where SeverityLevel <= 3
| summarize Count=count() by Computer, ProcessName, bin(TimeGenerated, 15m)
```



### Senior-level depth
- Separate Azure control-plane RBAC, Kubernetes API authorization, cluster managed identity and pod workload identity.
- Prefer private endpoints, private DNS, managed identities/federation and scoped data-plane roles instead of account keys.
- Trace traffic through DNS, UDRs, NSGs, firewalls and service endpoints/private links.

### Validation checklist
1. Check the signed-in principal and role assignment scope.
2. Verify private DNS resolution and effective routes.
3. Inspect NSG flow logs and application connection errors.
4. Confirm token audience, issuer and federated credential for workload identity.
5. Validate access from the intended pod and denial from an unintended pod.

### Interview example framing
“I use identity rather than stored keys, restrict the network path with Private Link, and validate both the intended access and the expected denial path.”

---
## Q134. AKS pod-only access to Storage Account.

### Direct interview answer

Use Microsoft Entra Workload ID with a dedicated Kubernetes service account federated to a narrowly scoped managed identity. Grant that identity only the required data-plane role on the storage account/container. Use a private endpoint and private DNS, disable public network access where feasible, and apply NetworkPolicy so only the workload can reach the endpoint. Do not put storage keys in a cluster-wide Secret.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q135. 32 GB cluster, 30 GB used; Pod request 500 Mi and limit 4 Gi.

### Direct interview answer

Scheduling is based primarily on requests and allocatable resources, not current usage or limits. If an eligible node has at least 500 Mi allocatable unrequested memory and other constraints fit, it may schedule even though the 4 Gi limit could later cause node pressure. HPA adds Pods and may worsen capacity; VPA may raise requests and make scheduling harder. Add node capacity, right-size requests from observed working set, and preserve headroom for system and failure scenarios.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q136. Nginx ingress alternatives.

### Direct interview answer

Choose based on support and requirements: cloud-native Application Gateway/ALB controllers, Kubernetes Gateway API implementations, Envoy Gateway, HAProxy, Traefik, Kong or a service-mesh gateway. Validate project status, Gateway API maturity, WAF/TLS, TCP/UDP, observability, migration path and operational ownership rather than switching only because of a rumor about deprecation.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
# 10. Behavioral and project answers


## Q137. Tell me about yourself.

### Direct interview answer

“I am a Senior DevOps/Cloud engineer with six years of experience automating infrastructure, CI/CD, Kubernetes platforms and production reliability. My core strengths are AWS, Terraform, containers, Linux and observability. I focus on secure, repeatable delivery: infrastructure as code, immutable artifacts, least-privilege identity, progressive deployment and measurable SLOs. In my recent work I have partnered with developers and security teams to improve release reliability, troubleshoot incidents and optimize cloud cost. I am now looking for a role where I can own platform outcomes and mentor engineers while remaining hands-on.”



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q138. Why this role / motivation?

### Direct interview answer

Connect three points: the role’s problems, your evidence and growth. Example: “I am motivated by roles where platform engineering directly improves developer speed and service reliability. This role combines cloud architecture, Kubernetes operations and automation, which matches my hands-on experience. I also value the opportunity to deepen SRE practices such as SLOs, error budgets and resilience testing while helping the team standardize secure delivery.”



### Senior-level depth
- Emphasize idempotency, deterministic inventory, variable precedence, secrets management, check mode, controlled concurrency and handlers.
- Prefer purpose-built modules over shell commands. A playbook should converge state and remain safe when run repeatedly.
- For production rollout, use `serial`, health checks and clear failure thresholds.

### Validation checklist
1. Run syntax check, lint and check mode where meaningful.
2. Test against a small inventory subset.
3. Use `--diff` carefully because output may contain sensitive data.
4. Confirm handlers fire only on change.
5. Re-run and expect no unintended changes.

### Interview example framing
“I design Ansible automation to be idempotent and observable. I canary on a subset, limit concurrency, use handlers for controlled restart, and prove the second run is clean.”

---
## Q139. Rate your AWS skill out of five.

### Direct interview answer

Use calibrated confidence: “I would rate myself 4/5 for the services I operate daily: VPC, IAM, EC2, EKS, S3, RDS, CloudWatch and Terraform. I can design, automate and troubleshoot them independently. I keep one point open because AWS is broad and I validate unfamiliar services against documentation and test environments rather than bluffing.”



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q140. Failed deployment story.

### Direct interview answer

STAR template: Situation: a release increased 5xx and latency. Task: restore service while preserving evidence and stakeholder trust. Action: froze changes, declared incident roles, compared deployment and service metrics, shifted traffic/rolled back the immutable release, validated database compatibility, and sent factual updates. Result: service restored within the measured incident window, no data loss confirmed. Learning: added automated canary gates, backward-compatible schema checks and a tested rollback runbook. Replace timing/results with your true facts.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q141. Introduce a new DevOps tool and gain buy-in.

### Direct interview answer

Start with a shared pain and baseline. Run a small reversible pilot with one willing team, define success metrics, include security/operations early, document migration and support, and show measured outcomes. Offer training and office hours. Do not impose the tool solely because it is popular; publish trade-offs and an exit plan.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q142. Conflict and collaboration under pressure.

### Direct interview answer

Separate facts from positions. Re-state the shared objective, assign incident roles, use evidence from logs/metrics, time-box experiments and record decisions. In retrospectives, focus on system conditions and actions, not personalities. Escalate only when risk, ethics or decision rights require it.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q143. Reduce deployment time.

### Direct interview answer

Map the value stream, identify wait time, parallelize independent tests, cache dependencies safely, use incremental builds, create ephemeral test environments and remove duplicate scans while preserving quality gates. Promote one immutable artifact instead of rebuilding. Measure lead time, deployment frequency, change failure rate and recovery time before and after.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q144. Pipeline security vulnerability response.

### Direct interview answer

Disable or isolate the affected path, rotate exposed credentials, identify affected builds/artifacts, preserve logs, patch the dependency/action, rebuild from clean trusted sources and revoke compromised artifacts. Add secret scanning, dependency pinning, provenance/signing, OIDC, least permissions and policy gates. Communicate scope and residual risk transparently.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q145. Automation caused a problem.

### Direct interview answer

Own the impact. Stop the automation, restore known-good state, determine why safeguards failed, and add dry-run/plan, canary scope, concurrency control, idempotency, approvals for destructive actions and automated rollback. Test failure modes, not just the happy path.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q146. Mentor juniors while delivering.

### Direct interview answer

Give bounded ownership with clear outcomes, pair on risky changes, use design reviews and runbooks, and rotate on-call shadowing. Set checkpoints based on risk, not micromanagement. Protect delivery by keeping critical-path work with appropriate support and turning incidents into learning reviews.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q147. Successful DevOps transformation.

### Direct interview answer

Structure the answer around baseline, intervention and measured outcome. Example interventions: standard pipeline templates, IaC modules, GitOps, environment parity, SLO dashboards and blameless incident review. State your personal decisions and collaboration, not only “we”. Use only real metrics and explain trade-offs and adoption challenges.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# 11. Ready-to-use scripts and command appendix


## Q148. Delete files older than 30 days safely.

### Direct interview answer

Use a fixed directory, log candidates, avoid symlink traversal and separate dry run from deletion.


```bash
#!/usr/bin/env bash
set -euo pipefail
DIR=${1:-/var/tmp/app}
[[ -d "$DIR" && "$DIR" != "/" ]] || { echo "Unsafe directory" >&2; exit 1; }
find -P "$DIR" -xdev -type f -mtime +30 -print

### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
# After review, enable deletion:
find -P "$DIR" -xdev -type f -mtime +30 -print -delete
```


## Q149. Compress logs older than 30 days, delete older than 90, and run daily.

### Direct interview answer

Avoid recompressing .gz files and use null-delimited paths.


```bash
#!/usr/bin/env bash
set -euo pipefail
DIR=/var/log/myapp
find -P "$DIR" -xdev -type f ! -name '*.gz' -mtime +30 -mtime -90 -print0 \
 | xargs -0 -r gzip --
find -P "$DIR" -xdev -type f -mtime +90 -print -delete

### Senior-level depth
- Monitor user outcomes first, then service and infrastructure causes. Use RED for request-driven services and USE for resources.
- Define alert ownership, severity, runbook, suppression and recovery conditions. A dashboard without an operational decision is not enough.
- Control metric cardinality, log retention and trace sampling to keep the observability system reliable and affordable.

### Validation checklist
1. Confirm data is actually being scraped/ingested.
2. Verify labels, units, query window and aggregation.
3. Test alert delivery and recovery notification.
4. Link alerts to a runbook and service owner.
5. Review false positives, missed incidents and storage cost.

### Interview example framing
“I alert on symptoms that threaten the SLO, use infrastructure metrics for diagnosis, and tune the signal from incident outcomes. Every production alert has an owner and an executable runbook.”

---
# /etc/cron.d/myapp-retention
15 2 * * * root /usr/local/sbin/myapp-retention >>/var/log/myapp-retention.log 2>&1
```


## Q150. Disk monitor over 80% with email alert.

### Direct interview answer

In production, prefer monitoring agents and deduplication. This script demonstrates the interview logic.


```bash
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD=80
LOG=/var/log/disk-alert.log
HOST=$(hostname -f)
df -P -x tmpfs -x devtmpfs | awk 'NR>1 {gsub("%","",$5); print $5, $6}' |
while read -r used mount; do
  if (( used > THRESHOLD )); then
    msg="$(date -Is) host=$HOST mount=$mount usage=${used}%"
    echo "$msg" | tee -a "$LOG" | mail -s "Disk alert: $HOST $mount" ops@example.com
  fi
done
```



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
## Q151. Rename .txt files with current date.

### Direct interview answer

Handle spaces and preserve paths.


```bash
#!/usr/bin/env bash
set -euo pipefail
DIR=${1:-.}; DATE=$(date +%Y%m%d)
find "$DIR" -maxdepth 1 -type f -name '*.txt' -print0 |
while IFS= read -r -d '' f; do
  mv -- "$f" "${f%.txt}_${DATE}.txt"
done
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q152. Monitor and restart a service.

### Direct interview answer

Use systemctl is-active, log the action and return failure if restart fails.


```bash
#!/usr/bin/env bash
set -euo pipefail
SERVICE=${1:-nginx}; LOG=/var/log/service-watch.log
if ! systemctl is-active --quiet "$SERVICE"; then
  echo "$(date -Is) $SERVICE inactive; restarting" >> "$LOG"
  if systemctl restart "$SERVICE"; then
    echo "$(date -Is) $SERVICE restarted" >> "$LOG"
  else
    echo "$(date -Is) $SERVICE restart FAILED" >> "$LOG"
    exit 1
  fi
fi
```



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q153. Capture names of newly created files.

### Direct interview answer

For continuous events, inotifywait is appropriate if installed. For a portable snapshot, compare directory listings. Example using inotify-tools:


```bash
#!/usr/bin/env bash
set -euo pipefail
DIR=${1:-/data/incoming}; OUT=${2:-/var/log/new-files.log}
inotifywait -m -e create -e moved_to --format '%T %w%f' --timefmt '%FT%T%z' "$DIR" |
while IFS= read -r line; do printf '%s
' "$line" >> "$OUT"; done
```



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q154. Bash log analysis.

### Direct interview answer

Count levels and top error patterns while avoiding assumptions about the log schema.


```bash
#!/usr/bin/env bash
set -euo pipefail
FILE=${1:?usage: $0 logfile}
printf 'ERROR=%s WARN=%s
'   "$(grep -c 'ERROR' "$FILE" || true)"   "$(grep -c 'WARN' "$FILE" || true)"
grep 'ERROR' "$FILE" | sed -E 's/[0-9]+/<N>/g' | sort | uniq -c | sort -nr | head -20
```



### Senior-level depth
- Troubleshoot from evidence: symptoms and scope, recent change, resource saturation, process behavior, dependency health and kernel/system logs.
- Distinguish capacity from saturation: free disk percentage is different from inode exhaustion or latency; CPU usage is different from run queue or I/O wait.
- Make the smallest reversible change, preserve evidence, and validate at application level.

### Diagnostic checklist
1. Establish time window, blast radius and change history.
2. Check CPU/run queue, memory/swap, disk latency/capacity/inodes and network errors.
3. Identify the responsible process and open files/sockets.
4. Review service and kernel logs.
5. Mitigate, observe, and create a prevention action such as retention, alerting or capacity policy.

### Interview example framing
“I avoid jumping directly to restart. I first capture resource and process evidence, apply a reversible mitigation, verify user impact is gone, and then fix the underlying retention, capacity or configuration defect.”

---
# 12. Quick-fire answers checklist


## Q155. What happens if etcd stops?

### Direct interview answer

Reads/writes and control-plane progress fail depending on quorum; existing containers can continue. Restore quorum, never create a split brain.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q156. Kubernetes Operator?

### Direct interview answer

A controller plus CRDs that encodes domain-specific lifecycle. For a script before containers, normally use an init container; do not create an Operator for a one-off script.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q157. OpenShift pull secret?

### Direct interview answer

A cluster/global or namespace-level registry credential used to pull Red Hat and private registry images; restrict and rotate it.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q158. Maven lifecycle?

### Direct interview answer

validate, compile, test, package, verify, install, deploy; clean and site are separate lifecycles.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q159. MongoDB HA?

### Direct interview answer

A replica set elects one primary for writes and secondaries replicate the oplog. Majority write concern/read preference and odd voting members shape consistency/availability.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q160. Artifact management?

### Direct interview answer

Versioned, immutable storage of build outputs with metadata, checksums, retention and promotion, using tools such as Nexus, Artifactory, ECR or ACR.



### Senior-level depth
- Separate CI from CD: CI proves a commit and produces one immutable artifact; CD promotes that same artifact through environments.
- Use protected branches/environments, short-lived workload identity, pinned dependencies/actions, security scanning, approvals based on risk, and auditable rollback.
- Do not rebuild for production. Promote the previously tested digest and keep database changes backward compatible.

### Pipeline validation checklist
1. Reproduce the failing stage with the exact commit and artifact.
2. Verify credentials, network reachability and registry permissions without printing secrets.
3. Confirm artifact digest and provenance.
4. Validate deployment health using readiness plus service-level metrics.
5. Roll back by version/digest and preserve logs for RCA.

### Interview example framing
“My pipeline builds once, tests and scans in CI, publishes an immutable artifact, and promotes it through policy-controlled CD. Production rollout is progressive and automatically stops or rolls back when predefined SLO signals regress.”

---
## Q161. GSLB?

### Direct interview answer

Global server load balancing uses DNS/anycast and health or policy signals to steer clients across regions. Account for DNS caching and failover TTL.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q162. Password practices?

### Direct interview answer

SSO, phishing-resistant MFA, password manager, long unique passwords, no sharing, breached-password checks, rotation on compromise, break-glass controls and audit.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q163. High availability principle?

### Direct interview answer

Remove single points of failure across AZs, decouple state, automate health-based failover, test capacity and recovery, and monitor user-facing SLOs.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q164. Container rollback?

### Direct interview answer

Containers are immutable artifacts; redeploy the previous signed digest. In Kubernetes use rollout undo or GitOps revert and verify health.



### Senior-level depth
- Distinguish image build, container runtime, host kernel isolation, storage and networking.
- Secure the supply chain with trusted minimal bases, digest pinning, SBOMs, vulnerability scanning, signatures, non-root execution and restricted capabilities.
- Optimize layer caching and runtime size without sacrificing patchability or troubleshooting requirements.

### Validation checklist
1. Inspect image history, digest, architecture and vulnerability report.
2. Run as the intended non-root user with a read-only filesystem where possible.
3. Verify the actual listening interface and port.
4. Apply CPU, memory, PID and storage controls.
5. Test shutdown signals, health checks and log collection.

### Interview example framing
“I treat the image as an immutable release artifact. I keep build tools out of the runtime stage, scan and sign the digest, and deploy with restricted runtime settings rather than relying on Dockerfile metadata alone.”

---
## Q165. Static website without public S3?

### Direct interview answer

Private S3 origin behind CloudFront with Origin Access Control; block public access and grant only the CloudFront distribution.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q166. Secrets Manager vs Parameter Store?

### Direct interview answer

Secrets Manager focuses on secrets and rotation; Parameter Store handles hierarchical configuration and secure strings. Choose by rotation, integration, size, throughput and cost needs.



### Senior-level depth
- Explain the design across identity, network, data protection, availability, observability and cost rather than discussing one AWS service in isolation.
- Prefer temporary credentials, private networking, encryption, multi-AZ designs, managed backups, centralized audit logs and policy guardrails.
- Separate control-plane configuration from data-plane traffic. A resource can be “healthy” in the console while the actual end-to-end request path is broken.

### Validation and operations checklist
1. Confirm caller identity and effective authorization.
2. Trace DNS, route tables, gateways, security groups, NACLs and service health checks.
3. Validate encryption keys, backups, restore tests and retention.
4. Check CloudTrail, CloudWatch, Config and service-specific metrics.
5. Test failover and rollback, then compare cost against the required SLO.

### Interview example framing
“I make the decision from RTO/RPO, traffic pattern, compliance boundaries and operating model. I then automate it through Terraform, validate it with synthetic checks, and continuously monitor both reliability and unit cost.”

---
## Q167. What is Fargate?

### Direct interview answer

Serverless container compute for ECS/EKS that removes node management; evaluate workload constraints, observability and economics.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q168. What is CloudFormation?

### Direct interview answer

AWS-native declarative IaC using stacks and change sets; Terraform is multi-provider with its own state and ecosystem.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q169. Least privilege for IaC?

### Direct interview answer

Separate plan/read and apply roles where possible; scope by environment/resources, use permission boundaries/SCPs, short-lived OIDC and reviewed policy changes.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q170. What is a Pod?

### Direct interview answer

The smallest Kubernetes scheduling unit: one or more tightly coupled containers sharing network namespace and volumes.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q171. Namespace?

### Direct interview answer

A logical API scope for names, RBAC, quotas and policies; it is not by itself a hard security boundary.



### Senior-level depth
- State assumptions before choosing a solution.
- Explain the end-to-end workflow, operational ownership and failure modes.
- Include security, availability, observability, rollback and cost implications.

### Validation checklist
1. Establish the expected behavior and measurable success criteria.
2. Collect evidence before changing production.
3. Apply the smallest reversible action.
4. Validate from the user or consumer perspective.
5. Record the permanent corrective action and owner.

### Interview example framing
“I would first establish scope and success criteria, choose the lowest-risk reversible action, validate with observable evidence, and then document the long-term prevention.”

---
## Q172. Kube-proxy?

### Direct interview answer

Programs Service forwarding rules; some clusters replace it with an eBPF data plane.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q173. Kubelet?

### Direct interview answer

Node agent that reconciles assigned PodSpecs, uses CRI/CSI/CNI integrations and reports status.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
## Q174. Deployment vs Service?

### Direct interview answer

Deployment manages Pod replicas/releases. Service provides stable discovery and load distribution to selected ready Pods.



### Senior-level depth
- Start from the Kubernetes reconciliation model: desired state is stored through the API, controllers converge actual state, and node agents execute assigned work.
- Separate **control-plane failure**, **scheduling failure**, **container startup failure**, **networking failure**, and **storage failure**. Each has a different evidence source.
- For every proposed change, mention PodDisruptionBudgets, readiness, topology spread, resource requests, rollback, and observability.

### Validation and troubleshooting checklist
1. Read the object status and conditions.
2. Inspect namespace events in timestamp order.
3. Verify selectors, labels, requests, affinities, taints and quotas.
4. Trace dependencies: API -> scheduler/controller -> kubelet/runtime -> CNI/CSI -> application.
5. Confirm the user-facing path with a synthetic request, not only a green Pod status.

### Interview example framing
“First I classify whether this is an API, scheduling, node, network, storage or application issue. I collect events and component evidence before changing anything, apply the smallest reversible mitigation, then validate service-level metrics and record the permanent corrective action.”

---
# Sources and verification notes

This handbook uses official vendor documentation as the primary factual baseline. Product behavior changes over time; verify commands and version compatibility before production use.

- Kubernetes documentation: Autoscaling Workloads and StatefulSets - https://kubernetes.io/docs/concepts/workloads/autoscaling/ and https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- AWS EKS Best Practices: Cluster Upgrades and Karpenter - https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html and https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html
- HashiCorp Terraform documentation: lifecycle and importing state - https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle and https://developer.hashicorp.com/terraform/language/state/import
- Microsoft Learn: Private AKS clusters and access/identity - https://learn.microsoft.com/en-us/azure/aks/private-clusters and https://learn.microsoft.com/en-us/azure/aks/concepts-identity

# Official reference set

Use these sources to verify behavior against the versions used in the target environment:

- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Security](https://kubernetes.io/docs/concepts/security/)
- [Amazon EKS cluster upgrade best practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html)
- [Amazon EKS Karpenter best practices](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html)
- [HashiCorp Terraform import documentation](https://developer.hashicorp.com/terraform/language/import/single-resource)
- [HashiCorp Terraform provisioners](https://developer.hashicorp.com/terraform/language/provisioners)
- [Microsoft Entra Workload ID on AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)

## Final interview reminder

A senior answer is not only a definition. It demonstrates decision quality, safe execution, production validation, communication and learning. Replace all sample metrics, environment counts and project outcomes with defensible facts from your own work.
