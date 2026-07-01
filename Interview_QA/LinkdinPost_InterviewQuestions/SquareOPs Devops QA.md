These are solid mid-to-senior interview questions covering the core Kubernetes and AWS networking surface area. I'll answer each with the depth and follow-up reasoning interviewers are listening for.

---

## 1. What's the current Kubernetes version you're working with?

This is a "tell me about your environment" question — interviewers want to see you know your stack, can speak to version-specific features, and understand the upgrade cadence. Answer honestly with whatever version you actually run. As of mid-2026, Kubernetes 1.33 is the current stable release with 1.30–1.32 commonly in production (clusters typically run one or two versions behind latest for stability).

A good answer sounds like: "We're on 1.31 in production, with 1.32 in our staging clusters being validated for upgrade. We stay within the supported version window — Kubernetes supports the latest three minor versions, so we don't let prod drift more than two versions behind." Mention which distribution if relevant (EKS, GKE, AKS, vanilla, OpenShift) because the upgrade story differs significantly. If you've dealt with deprecated API migrations (e.g., the PodSecurityPolicy removal in 1.25, or batch/v1beta1 CronJob), mention that — it shows you've handled real upgrade pain.

The wrong answer is naming a very old version without context, or naming the latest without being able to discuss what's new in it. Be honest; interviewers respect "we're on 1.29 because we're conservative about upgrades" more than fabricated currency.

---

## 2. How do you perform a Kubernetes cluster upgrade?

I covered this in depth earlier (the zero-downtime upgrade question), but the concise interview answer hits the key points in order.

**Pre-upgrade prep.** Read release notes for deprecated/removed APIs — this is where upgrades break things most often (e.g., an Ingress API version removed, a CronJob beta API gone). Run `kubectl deprecations` tools or `pluto` to scan manifests for removed APIs and fix them *before* the upgrade. Back up etcd. Verify cluster health (all nodes Ready, no failing pods, no degraded components). Check that all add-ons (CNI, CSI drivers, ingress controllers, cert-manager) support the target version. Test the upgrade in a non-prod cluster first.

**Upgrade order — control plane first, then nodes.** The control plane gets upgraded first because the rule is kubelet can be the same or up to two minors *behind* the API server, never ahead. So API server → controller-manager/scheduler → then the data plane. In HA control planes, upgrade one master at a time so the API stays available.

**Respect version skew.** Upgrade *one minor version at a time* (1.29 → 1.30 → 1.31, not 1.29 → 1.31 directly). Patch versions can skip freely; minors cannot.

**Node upgrade — cordon, drain, upgrade, uncordon.** For each node: `kubectl cordon` marks it unschedulable, `kubectl drain` evicts pods respecting PodDisruptionBudgets, then upgrade the node (or replace it entirely if using immutable nodes via Karpenter / managed node groups — the modern pattern is to replace, not in-place upgrade). Uncordon and verify before moving to the next.

**Managed Kubernetes specifics.** On EKS/GKE/AKS, the control-plane upgrade is a single API call and the cloud handles it. Node groups upgrade via the cloud's rolling-update mechanism. The key managed-K8s discipline is keeping node groups within version skew — easy to forget on EKS where the control plane upgrades but node groups stay on the old version until you act.

**Post-upgrade.** Verify all workloads healthy, all add-ons reconciled, and the previously-deprecated APIs are no longer in use anywhere. Watch for slow-burn issues (a controller that's now reconciling on a new field, an admission webhook that breaks under new API behavior).

Mention `kubeadm upgrade plan`/`kubeadm upgrade apply` if you've used vanilla kubeadm clusters — that's the canonical procedure interviewers may probe on.

---

## 3. How Kubernetes Ingress routes traffic to services and pods

Ingress is a *spec* (an API object describing HTTP/HTTPS routing rules); the routing itself is done by an **ingress controller** — NGINX, Traefik, HAProxy, AWS ALB controller, etc. The Ingress object alone does nothing; the controller watches Ingress resources and programs its proxy/load balancer accordingly.

**The traffic flow, end to end.** A client resolves your hostname (e.g., `app.example.com`) to the ingress controller's external IP/LB (this part *is* DNS). The request hits the ingress controller — a pod or set of pods running NGINX/Envoy/etc. — exposed externally via a Service of type LoadBalancer or NodePort. The controller inspects the request (host header, path) and matches it against the Ingress rules (e.g., `app.example.com/api/*` → `api-service`). It then forwards the request to the matched **Service**.

**Crucially, the ingress controller usually bypasses kube-proxy and the Service's ClusterIP** — modern ingress controllers watch the Endpoints/EndpointSlices directly and load-balance to pod IPs themselves. This avoids an extra hop through iptables/IPVS and gives the controller better health-checking and load-balancing control. The Service still exists as a logical grouping (the controller learns about pods via its Endpoints), but the data path skips it.

**TLS termination** typically happens at the ingress controller — it holds the certificate (often issued automatically by cert-manager from Let's Encrypt) and terminates TLS, then forwards plaintext (or re-encrypts) to backend pods. Path-based and host-based routing, rewrites, redirects, rate limits, auth — all configured via Ingress annotations or, on newer setups, the **Gateway API** (which replaces Ingress for more expressive routing).

**Why "ephemeral pods" don't break this** (this is the standard follow-up): pods come and go, and their IPs change. But the Service's Endpoints object is maintained by the endpoints controller in real time — when a pod is created and passes its readiness probe, it's added to Endpoints; when it dies or fails readiness, it's removed. The ingress controller watches Endpoints (via the API server's watch mechanism) and updates its internal pod-IP list within seconds. So even though individual pod IPs are ephemeral, the *set* of healthy backend IPs is continuously kept current, and the controller routes only to live pods. The label selector on the Service is the glue — pods matching the selector are tracked; pods that don't are ignored.

---

## 4. How a pod gets created when one replica is in a failed/pending state

This is really asking about the controller reconciliation loop and which Kubernetes components are involved. It overlaps heavily with Question 8 below, so I'll consolidate the deep answer there and give the short version here.

A Deployment owns a ReplicaSet which owns Pods. The **ReplicaSet controller** (part of kube-controller-manager) continuously watches: "does my actual pod count match my desired replica count of healthy pods?" When a pod enters CrashLoopBackOff, the pod is *still counted* as existing — the ReplicaSet sees 3 pods, just one is unhealthy. CrashLoopBackOff is *kubelet* repeatedly restarting the container on the same node; the pod object isn't deleted, so the ReplicaSet doesn't create a replacement. This is a common source of confusion: "why isn't Kubernetes replacing my crashing pod?" Because from the ReplicaSet's view, the pod still exists.

When a pod is **deleted** (by eviction, node failure, explicit delete, or a controller like the Deployment during a rollout), the ReplicaSet's watch sees pod count drop below desired and immediately creates a new pod object. The **scheduler** assigns it to a node. The **kubelet** on that node pulls images and starts containers. The pod becomes Ready when probes pass, and the Service's Endpoints update to include it.

For **Pending** pods — the pod object exists but no node has been bound. The scheduler couldn't place it (insufficient resources, taints/affinity mismatch, volume zone conflict). The ReplicaSet considers the pod as existing toward the replica count, so it won't create another one — it's waiting for the scheduler to succeed. If the cluster autoscaler can add a node, it will; otherwise the pod stays Pending until you fix the cause.

---

## 5. How to configure Horizontal Pod Autoscaler

HPA scales pod replicas based on observed metrics. The configuration has a few layers.

**Prerequisites.** The metrics-server must be running (for CPU/memory metrics) or a custom-metrics adapter (Prometheus Adapter, KEDA) for non-resource metrics. The target workload's containers must have `resources.requests` set — HPA computes CPU utilization as `current usage / requested CPU`. Without requests, HPA can't compute utilization and silently fails.

**Basic HPA on CPU:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

This keeps average CPU utilization at 70% across all replicas, scaling between 3 and 30 pods.

**Custom and external metrics.** For request-serving workloads, scaling on RPS or queue depth is far better than CPU. Use a Prometheus Adapter to expose Prometheus metrics to the HPA API, then:
```yaml
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```
For event-driven scaling (Kafka lag, SQS depth, etc.), KEDA is the standard — it gives you scalers for dozens of event sources and works as a thin layer on top of HPA.

**Behavior tuning (autoscaling/v2 `behavior` field).** This controls how aggressively HPA scales up/down. Stabilization windows prevent flapping (e.g., `scaleDown.stabilizationWindowSeconds: 300` so it waits 5 minutes of sustained low load before scaling down). Policies cap the rate (e.g., "no more than 50% pod increase per minute"). Tune these for streaming-style spiky workloads — default scale-up is sometimes too slow.

**Operational notes.** Multiple metrics on one HPA — HPA scales to whichever metric demands the *most* replicas. Don't run two HPAs on the same target. Don't combine HPA with VPA in update mode on the same pods (they conflict). Monitor HPA conditions (`ScalingActive`, `ScalingLimited`, `AbleToScale`) and alert on them — covered in the earlier HPA monitoring question.

---

## 6. What is NLB and when to use it

**NLB (Network Load Balancer)** is AWS's Layer 4 (TCP/UDP) load balancer. It operates at the connection level — it doesn't inspect HTTP headers, paths, or content. It just forwards connections to targets based on flow hashing. Key properties:

*Extreme performance and low latency.* NLB handles millions of requests per second with single-digit-millisecond latency overhead. No HTTP parsing, no SSL termination overhead (though it can do TLS passthrough or termination as of recent versions).

*Static IPs and Elastic IP support.* You can pin Elastic IPs to NLB (one per AZ). ALB doesn't offer this. This matters when clients have firewall allowlists by IP, or when you need a stable IP for DNS.

*Preserves client source IP.* NLB by default passes the original client IP through to targets (in IP target mode). ALB requires the X-Forwarded-For header workaround.

*Handles any TCP/UDP protocol.* Not just HTTP. Database connections, custom binary protocols, MQTT, gRPC over TCP, game servers, streaming protocols — anything NLB does. ALB only handles HTTP/HTTPS/gRPC over HTTP/2.

**When to use NLB.** Non-HTTP workloads (MQTT, custom TCP services, database proxies). When you need stable IPs/EIPs for firewall allowlists. Extreme throughput / low latency where ALB's HTTP processing is overhead. As the LoadBalancer for a Kubernetes Service that exposes non-HTTP traffic, or where you want ingress controllers behind L4. UDP workloads (DNS, gaming, real-time video) — only NLB supports UDP.

**When NOT to use NLB.** When you need HTTP-aware routing (host/path-based routing, header rewrites, WAF integration, OIDC auth at the LB) — that's ALB's job. When you want simple path-based routing to multiple services, ALB is purpose-built.

---

## 7. How to expose Kubernetes services externally

Kubernetes offers several mechanisms with very different trade-offs:

**NodePort.** Service type that opens a static port (30000–32767 range) on every node. External traffic to `<node-ip>:<nodeport>` reaches the service. Simple and works anywhere but rarely used in production directly — nodes are ephemeral, ports are high-numbered, no load balancing across nodes from outside, no TLS. Mostly useful for local dev or as a building block under a load balancer.

**LoadBalancer.** Service type that provisions a cloud load balancer (ELB/NLB/ALB on AWS, equivalent on other clouds) which forwards to all nodes on the NodePort. The cloud controller manager handles provisioning. This is the standard for exposing a single service externally with cloud-grade load balancing. The downside: one LB per service gets expensive, and you lose HTTP-aware routing.

**Ingress.** Single entry point that routes to multiple services based on HTTP rules (host, path). One LB at the edge fronts an ingress controller, which routes to many backend services. The standard for HTTP/HTTPS workloads. Drastically cheaper than LB-per-service and gives you path/host routing, TLS termination via cert-manager, and rich annotations for rewrites/auth/rate-limiting. Use NGINX Ingress Controller, AWS Load Balancer Controller (which manages ALB/NLB from Ingress objects), Traefik, etc.

**Gateway API.** The successor to Ingress, more expressive and role-aware (separating infrastructure, cluster operator, and application developer concerns). Same conceptual purpose as Ingress but better designed. Production-ready and worth adopting for new clusters.

**ExternalName / headless services.** Edge cases — ExternalName creates a CNAME alias for an external DNS name. Headless (clusterIP: None) exposes pod IPs directly for clients that do their own load balancing (StatefulSets, custom protocols).

**On AWS specifically.** Most production setups use either: (a) AWS Load Balancer Controller managing ALB via Ingress objects for HTTP, plus NLB via Service type LoadBalancer for non-HTTP; or (b) NGINX Ingress Controller behind a single NLB, with NGINX doing the HTTP routing.

---

## 8. End-to-end: how Kubernetes detects and replaces a failed/stuck pod

This is the deep version of question 4 — interviewers asking this want to see you understand the controller architecture. Walk through the cast of components and the reconciliation flow.

**The cast.** *kubelet* runs on every node, manages pods on that node, runs probes, reports status to the API server. *kube-controller-manager* runs the controller loops, including the Deployment controller, ReplicaSet controller, and node lifecycle controller. *kube-scheduler* assigns Pending pods to nodes. *kube-apiserver* is the central state store interface (backed by etcd) — everyone watches and updates state through it.

**The Deployment → ReplicaSet → Pod hierarchy.** A Deployment doesn't manage pods directly. It manages a ReplicaSet, and the ReplicaSet manages pods via a label selector. Desired state lives in the Deployment spec (`replicas: 3`), which the ReplicaSet inherits.

**Scenario 1: CrashLoopBackOff.** A container exits with an error. The kubelet detects the exit, updates the pod status, and *restarts the container in place* (per the pod's `restartPolicy: Always`). After repeated failures, kubelet applies exponential backoff (10s, 20s, 40s, …, capped at 5 min) — this is what `CrashLoopBackOff` means. Critically, **the pod object still exists**; it's not deleted. So the ReplicaSet, which counts pod *existence*, sees 3 pods and doesn't create a replacement. This is why CrashLoopBackOff doesn't trigger a "replace the bad pod" cycle — it's the *same pod* being restarted by the kubelet.

If liveness probes are configured and failing, the kubelet restarts the container (same effect). If the pod becomes unready (readiness probe failing), the endpoints controller removes it from the Service's Endpoints, so it stops receiving traffic — but the pod still exists.

To get a replacement, the pod has to be *deleted*. This happens via operator intervention (`kubectl delete pod`), eviction, or a controller-driven action (Deployment rollout replacing it). On deletion, the ReplicaSet observes pod count < desired and creates a new pod object → scheduler binds it to a node → kubelet starts it.

**Scenario 2: Pending (scheduling failure).** The Deployment created the pod object, but the scheduler couldn't place it. The scheduler runs filtering (does any node satisfy resource requests, affinity, taints, volumes?) and scoring; if no node passes filtering, the pod stays Pending with an event explaining why. The ReplicaSet counts this Pending pod toward the replica count — so again, no replacement is created. The fix is either resource availability (cluster autoscaler adding nodes, or freeing resources) or correcting the constraint (taint, affinity).

**Scenario 3: Node failure.** The node-lifecycle controller in kube-controller-manager monitors node heartbeats. If a node stops reporting (kubelet death, network partition, hardware failure), after `node-monitor-grace-period` (default 40s) the node is marked NotReady. After `pod-eviction-timeout` (default 5 min on older versions; modern versions use taint-based eviction), pods on that node are marked for deletion (via NoExecute taint). The ReplicaSet then sees pod count drop and creates replacements, which the scheduler places on healthy nodes. This is the path where Kubernetes truly self-heals a "lost" pod.

**The watch-based reconciliation pattern is the underlying mechanism.** Every controller watches the API server for changes to its resources. When the ReplicaSet controller sees a pod deletion event, its reconciliation loop runs, observes "desired 3, actual 2," and creates a new pod object. The scheduler watches for pods with no `nodeName`, picks a node, writes `nodeName` (the "bind" operation). The kubelet on that node watches for pods assigned to it, pulls images, starts containers. Every action is independent, level-triggered (based on current state, not events), and idempotent. This is the genius of Kubernetes' design — many small controllers each doing one thing, coordinating only through shared state in etcd.

**The summary to leave the interviewer with.** "Kubernetes detects pod failure through kubelet probes and node heartbeats. Whether it *replaces* the pod depends on whether the pod gets deleted — in-place restart (CrashLoopBackOff) doesn't trigger replacement; only deletion (via node failure, eviction, or operator action) causes the ReplicaSet to create a new pod, which the scheduler places and kubelet starts. The system is built on watch-based reconciliation: many independent controllers each driving actual state toward desired state through the API server."

---

## 9. How ingress redirects traffic to pods (and ephemeral pods follow-up)

Already covered in depth in question 3. The follow-up about ephemeral pods is specifically asking: "if pod IPs change constantly, how does the ingress controller keep up?"

Answer: the ingress controller doesn't track *pod IPs* directly via the Ingress spec — it tracks them via the **Endpoints / EndpointSlice** object that Kubernetes maintains for each Service. The endpoints controller (running in kube-controller-manager) continuously reconciles the Endpoints object against pods matching the Service's label selector that pass readiness probes. When a pod becomes ready, its IP is added to Endpoints; when it terminates or fails readiness, its IP is removed.

The ingress controller establishes a **watch** on Endpoints via the Kubernetes API server. Any change to Endpoints is pushed to the controller within milliseconds. The controller updates its internal pod-IP list (its NGINX upstream config, or its Envoy cluster) and starts routing to new pods immediately, stopping routing to removed ones. So the ephemerality is handled by the layer below ingress — Kubernetes itself maintains the live list of healthy pod IPs, and the ingress controller subscribes to it.

This is also why **readiness probes are critical**: a pod failing readiness is removed from Endpoints, which is the mechanism by which load balancers and ingress controllers stop sending traffic to it. The pod is still running, just not receiving traffic until it recovers.

---

## 10. What problem does Ingress solve, and how

The problem: in a Kubernetes cluster running many services that need external HTTP/HTTPS access, the naive approach is one cloud LoadBalancer per Service. This is expensive (each LB costs money — at scale, hundreds of dollars per month per service), wasteful (each LB is dedicated to one service), and gives no HTTP-aware routing (each service has its own IP/hostname, no path-based or host-based routing across them).

Ingress solves this by introducing a **single entry point** for all HTTP/HTTPS traffic into the cluster, which then routes to many backend services based on rules:

*Cost.* One LB at the edge instead of N.

*HTTP-aware routing.* `example.com/api → api-service`, `example.com/web → web-service`, `admin.example.com → admin-service` — all from a single Ingress controller making intelligent L7 decisions.

*TLS termination centralized.* Cert-manager + Ingress automates Let's Encrypt certificates for all hostnames; backend services don't each need to handle TLS.

*Cross-cutting concerns at one place.* Rate limiting, auth, redirects, header manipulation, WAF — applied at the Ingress layer rather than reimplemented in every service.

*Decoupled from cloud LB type.* The Ingress controller itself is a pod; it can sit behind an NLB or ALB or whatever — and the Ingress spec is portable across clouds.

So if you mention NLB in the answer, the natural follow-up "why do you need NLB if Ingress is there?" — the Ingress controller itself is just pods inside the cluster; *something* has to bring external traffic to those pods from outside. That something is a cloud LB (NLB or ALB) of type LoadBalancer Service in front of the Ingress controller pods. Ingress doesn't replace cloud LBs — it sits behind one of them and adds HTTP-aware routing to many services.

---

## 11. If Ingress is already there, why NLB?

Because Ingress is a Kubernetes construct that lives *inside* the cluster — the Ingress controller is just a Deployment of NGINX or Envoy pods. External traffic from the internet can't reach those pods directly. You need a cloud-level entry point — a public IP, an externally-routable load balancer — to bring traffic into the cluster and forward it to the Ingress controller pods. That's NLB (or ALB).

So the layered architecture is: **internet → NLB (cloud, L4) → Ingress controller pods (in-cluster, L7) → backend services → pods**. The NLB is the cloud's externally-addressable entry point; the Ingress controller is the HTTP-aware router inside the cluster. They're complementary, not alternatives.

You *could* use ALB instead of NLB for this — and on AWS, the AWS Load Balancer Controller lets you do that natively (the ALB itself acts as both the cloud LB and the HTTP router, no in-cluster Ingress controller needed). But many teams prefer NLB + NGINX Ingress because: NGINX gives more routing power than ALB; the config is portable to other clouds; NLB is faster and cheaper than ALB; and the NGINX ingress controller has a richer ecosystem of annotations and plugins.

---

## 12. Types of AWS load balancers

AWS has four current load balancer types:

**ALB (Application Load Balancer)** — L7, HTTP/HTTPS/gRPC. Path and host-based routing, WAF integration, OIDC/Cognito auth, HTTP-aware features (redirects, header manipulation, sticky sessions via cookies). The go-to for HTTP workloads.

**NLB (Network Load Balancer)** — L4, TCP/UDP/TLS. High throughput, low latency, static IPs / EIPs, preserves client source IP, supports any TCP or UDP protocol. The go-to for non-HTTP, high-performance, or static-IP requirements.

**GLB / GWLB (Gateway Load Balancer)** — L3, used to deploy and scale third-party virtual appliances (firewalls, IDS/IPS, deep packet inspection) transparently. Niche; you use it when running security appliances in your VPC traffic path.

**CLB (Classic Load Balancer)** — Legacy. L4 and basic L7. Pre-VPC era, still around for backwards compatibility but AWS recommends migrating to ALB/NLB. Don't use it for new workloads.

There's also **ELB (Elastic Load Balancing)** as the umbrella service name covering all of the above — that's the service, not a specific type.

---

## 13. Why ALB if NLB can also handle external traffic?

Both *can* accept external traffic — the difference is **what they understand** about that traffic.

NLB sees TCP connections. It can route a connection to a target based on simple rules (target group, port), do flow hashing for load distribution, and that's about it. It doesn't know if it's HTTP, doesn't know the URL path, doesn't know the host header, doesn't know if a request is a `GET /api/users` or `POST /checkout`.

ALB sees HTTP requests. It parses the request line, headers, and (optionally) the body, and can route on host (`api.example.com` vs `web.example.com`), path (`/api/*` vs `/static/*`), HTTP method, header values, query strings, source IP, etc. It can rewrite paths, redirect URLs, terminate TLS with SNI for many certificates, integrate with WAF for L7 attack mitigation, do OIDC/Cognito authentication at the LB layer, sticky sessions via cookies, and HTTP/2 and gRPC support.

So the choice: if you need *HTTP-aware routing or features*, you need ALB. If you have non-HTTP protocols, or want minimum latency, or need static IPs, you need NLB. Many production setups use both — ALB for web/API traffic, NLB for database proxies / MQTT / non-HTTP services.

A common pattern is **NLB → ALB → targets** for cases where you want NLB's static IPs (for firewall allowlists) plus ALB's L7 features. Or **ALB → NLB → targets** is less common. Most often you pick one per use case.

---

## 14. Lost pem key — connecting to EC2

There are several recovery paths; the right one depends on what's already enabled and how urgent it is.

**SSM Session Manager (the modern best answer).** If the instance has the SSM Agent installed (default on Amazon Linux 2/2023, Ubuntu 18.04+ AMIs, Windows AMIs) AND has an IAM instance profile with the `AmazonSSMManagedInstanceCore` policy attached, you can connect via Session Manager without SSH or a key. From the AWS console: EC2 → select instance → Connect → Session Manager. Or via CLI: `aws ssm start-session --target i-xxxxx`. This is the recommended approach because it requires no inbound SSH port open, no key management, and gives audit logging via CloudTrail.

**Enabling Session Manager on an instance that doesn't have it.** This is the follow-up you flagged uncertainty on, so here's the accurate answer. Three things need to be true: (1) the SSM Agent is installed and running on the instance, (2) the instance has an IAM role attached with `AmazonSSMManagedInstanceCore` permissions, (3) the instance can reach the SSM endpoints (either via internet through a NAT gateway or via VPC endpoints for `ssm`, `ssmmessages`, `ec2messages`).

If the instance already has SSM agent but no IAM role, you can attach an IAM role to a running instance via the console or CLI (`aws ec2 associate-iam-instance-profile`) without reboot — SSM agent picks it up within a few minutes. If the SSM agent isn't installed, you can't easily install it without already having access — which is your chicken-and-egg problem. In that case, the fallback paths below.

**EC2 Instance Connect (Linux only, AL2/Ubuntu).** A browser-based SSH that uses temporary keys pushed via the EC2 Instance Connect API. Requires the `ec2-instance-connect` package on the instance (default on recent Amazon Linux 2 and Ubuntu) and your IAM user/role to have the `ec2-instance-connect:SendSSHPublicKey` permission. Works without you holding a permanent key.

**The recovery-via-stop method (most reliable if other options fail).** Stop the instance. Detach its root EBS volume. Attach the volume to a *different* instance you do have access to. Mount it, edit `/home/ec2-user/.ssh/authorized_keys` (or `/root/.ssh/authorized_keys`) to add a public key you control. Unmount, detach, reattach to the original instance as root device, start the instance. SSH in with your new key. This works no matter what — it's the universal recovery — but requires an hour of downtime.

**Replace the instance.** If it's stateless (auto-scaling group, immutable), the fastest answer might be "I terminate it and let ASG replace it with a fresh instance using a key I do have." Senior answer: if your architecture treats instances as cattle, key loss isn't an incident.

**Prevention going forward.** Don't rely on pem keys — use SSM Session Manager as the default, with IAM-controlled access and audit logging. Pem keys are a liability (lost, leaked, hard to rotate). The senior framing of this question: "I'd use Session Manager, and frankly I'd push the team to deprecate pem keys entirely as a security improvement."

---

## 15. Deploying a GitHub repo on EKS — what manifests/resources

This is asking you to lay out a full deployment topology. A complete answer walks the manifests in dependency order and shows you understand what each does.

**Prerequisites you'd set up (not manifests, but mention them).** ECR repository to hold the built container images. CI/CD pipeline (GitHub Actions, CodeBuild, Jenkins) that on push: builds the Docker image, tags it, pushes to ECR, and updates the manifests / triggers deployment. EKS cluster with the AWS Load Balancer Controller installed (for Ingress/ALB), and ideally external-dns and cert-manager for DNS and TLS automation.

**The core manifests:**

**Namespace** — isolation for the app's resources.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
```

**Deployment** — manages the application pods. Specifies image, replicas, resource requests/limits, probes, env vars.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: <account>.dkr.ecr.<region>.amazonaws.com/my-app:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: my-app-config
              key: db-host
```

**Service** — stable endpoint and load-balancing front for the pods.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

**Ingress** — exposes the service externally via ALB (using AWS Load Balancer Controller).
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

**ConfigMap and Secret** — non-sensitive and sensitive config.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: my-app
data:
  db-host: postgres.example.com
---
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: my-app
type: Opaque
stringData:
  db-password: <from-secrets-manager-in-real-life>
```
For production, sync from AWS Secrets Manager via External Secrets Operator rather than putting secrets in manifests.

**HorizontalPodAutoscaler** — scaling.
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
  namespace: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**PodDisruptionBudget** — protect availability during voluntary disruptions.
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app
  namespace: my-app
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

**ServiceAccount + IRSA (IAM Roles for Service Accounts)** — if the app needs AWS API access (S3, DynamoDB, etc.).
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/my-app-role
```
Reference this from the Deployment via `spec.template.spec.serviceAccountName: my-app`.

**Optional but production-ready additions:**

*NetworkPolicy* to restrict pod-to-pod traffic. *PodSecurityContext* with non-root user, read-only root filesystem, no privilege escalation. *Resource quotas* on the namespace. *Anti-affinity / topology spread constraints* to spread replicas across nodes and AZs. *Monitoring* — ServiceMonitor for Prometheus scraping if using Prometheus Operator.

**How you'd actually manage this.** Don't ship raw YAML to production. Wrap it in **Helm** (parameterized chart per environment) or **Kustomize** (base + overlays for dev/staging/prod). Apply via GitOps with **ArgoCD** or **Flux** — the Git repo is the source of truth, and the cluster continuously reconciles to match. The CI pipeline updates the image tag in Git; ArgoCD detects the change and rolls it out. This is the standard pattern for EKS deployments and worth mentioning to demonstrate operational maturity.

**The end-to-end story for the interviewer.** "I'd build a CI pipeline that, on a push to the GitHub repo, builds and pushes a container image to ECR. The Kubernetes manifests live in a Git repo (either the same one or a separate config repo) — Deployment, Service, Ingress, ConfigMap, HPA, PDB, plus a ServiceAccount with IRSA if AWS access is needed. ArgoCD watches the Git repo and syncs changes to EKS. The Ingress uses the AWS Load Balancer Controller to provision an ALB, with cert-manager handling TLS via ACM. External-dns updates Route 53. The Deployment has resource requests/limits, readiness/liveness probes, topology-spread constraints across AZs, and is fronted by an HPA scaling on CPU or custom metrics."

---

These all share a common pattern in interviewing: **start with the direct answer, then layer in the production-readiness context** (multi-AZ, IRSA over hardcoded credentials, GitOps over kubectl-apply, Helm/Kustomize over raw YAML, SSM over pem keys, custom metrics over CPU for HPA). That layering is what distinguishes a candidate who has built a tutorial from one who has run real systems.
