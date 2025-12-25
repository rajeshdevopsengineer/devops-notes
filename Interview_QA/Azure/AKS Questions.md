# 🚀 ADVANCED AKS HANDS-ON LABS (INTERVIEW-GRADE)

Each lab includes:

* 🎯 **Objective**
* 🏗 **Architecture**
* 🧩 **Step-by-step execution**
* 🔍 **Validation & troubleshooting**
* 💬 **How to explain in interviews**

---

## 🔵 LAB 1: Enterprise-Grade AKS Cluster (Private + Secure)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/private-clusters/aks-private-hub-spoke.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/concepts-clusters-workloads/control-plane-and-nodes.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/azure-cni-overlay/azure-cni-overlay.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/operator-best-practices-identity/pod-identities.png)

### 🎯 Objective

Deploy a **private AKS cluster** with:

* Private API server
* System + User node pools
* Azure CNI
* Managed identities
* No public exposure

### 🏗 Architecture

* VNet (AKS subnet + private endpoint)
* Private AKS
* Azure CNI
* System & User node pools

### 🧩 Steps

1. Create **VNet (10.20.0.0/16)**

   * AKS subnet: `10.20.1.0/24`
2. Create **Private AKS Cluster**

   * Enable private cluster
   * Network plugin: Azure CNI
   * Disable public API
3. Configure **System Node Pool**

   * VM size: Standard_D2s_v5
   * Taints for system workloads
4. Add **User Node Pool**

   * Autoscaling enabled
5. Enable **Managed Identity**
6. Connect via **Azure Bastion + Jump VM**
7. Install kubectl + az CLI

### 🔍 Validation

* API server private IP only
* Pods receive VNet IPs
* Nodes separated by role

### 💬 Interview Talking Points

* Why private AKS
* System vs User node pools
* Azure CNI vs Kubenet

---

## 🔵 LAB 2: AKS Ingress with Application Gateway (WAF)

![Image](https://learn.microsoft.com/en-us/azure/application-gateway/media/application-gateway-ingress-controller-overview/architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/aks-agic/media/aks-agic.svg)

![Image](https://learn.microsoft.com/en-us/azure/application-gateway/media/application-gateway-autoscaling-zone-redundant/application-gateway-autoscaling-zone-redundant.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/concepts-network/aks-ingress.png)

### 🎯 Objective

Expose AKS apps securely using **Application Gateway Ingress Controller (AGIC)** with **WAF**.

### 🧩 Steps

1. Deploy **Application Gateway (WAF v2)**
2. Enable **WAF (OWASP ruleset)**
3. Enable **AGIC add-on** on AKS
4. Deploy sample app (nginx)
5. Create **Ingress YAML**
6. Validate routing via App Gateway public IP
7. Trigger WAF rule (SQLi test)

### 🔍 Validation

* Traffic flows via App Gateway
* WAF blocks malicious requests
* No service public IPs

### 💬 Interview Talking Points

* AGIC vs NGINX Ingress
* L7 security in AKS
* WAF false-positive tuning

---

## 🔵 LAB 3: AKS Networking – Zero Trust (NSG + Azure Firewall)

![Image](https://imesh.ai/blog/wp-content/uploads/2023/03/Zero-Trust-Network.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/limit-egress-traffic/aks-azure-firewall-egress.png)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/tunnel-connectivity-issues/kubernetes-tunnel-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/private-clusters/aks-private-hub-spoke.png)

### 🎯 Objective

Implement **Zero Trust AKS networking** with:

* Forced egress via Azure Firewall
* NSG + UDR control
* Private endpoints only

### 🧩 Steps

1. Deploy **Azure Firewall** in hub VNet
2. Create **UDR**

   * `0.0.0.0/0 → Firewall`
3. Associate UDR with AKS subnet
4. Allow required FQDNs (AKS, ACR)
5. Block all other outbound traffic
6. Deploy test pod and validate access

### 🔍 Validation

* Internet blocked without firewall rule
* AKS system traffic allowed
* Firewall logs visible

### 💬 Interview Talking Points

* AKS outbound traffic control
* FQDN tags
* SNAT exhaustion risks

---

## 🔵 LAB 4: AKS Security – Workload Identity + Key Vault

![Image](https://learn.microsoft.com/en-us/azure/aks/media/workload-identity-overview/workload-id-model.png)

![Image](https://dewolfs.github.io/assets/img/2020-12-12-msi-design.png)

![Image](https://res.cloudinary.com/samcogan/image/upload/v1644592102/CSI_driver_Interface_aimaap.png)

![Image](https://raw.githubusercontent.com/ams0/ams0/deddfb117c95f739fda88c00a963604b9df3dd59/blog/dev.to/posts/stateless-monitoring-with-aks-thanos-prometheus-grafana/assets/images/stateless_thanos.png)

### 🎯 Objective

Build **secretless AKS workloads** using **Workload Identity** + **Key Vault CSI Driver**.

### 🧩 Steps

1. Enable **AKS Workload Identity**
2. Create **User-Assigned Managed Identity**
3. Grant Key Vault access (RBAC)
4. Install **Secrets Store CSI Driver**
5. Create `SecretProviderClass`
6. Deploy pod consuming secret
7. Validate no secrets stored in YAML

### 🔍 Validation

* Pod accesses secret securely
* No Kubernetes secrets used
* Identity-based access

### 💬 Interview Talking Points

* Why secretless architecture
* CSI Driver vs Kubernetes secrets
* Identity security boundaries

---

## 🔵 LAB 5: AKS Autoscaling (HPA + Cluster Autoscaler)

![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-autoscaling-hpa-1.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/cluster-autoscaler/cluster-autoscaler.png)

![Image](https://2.bp.blogspot.com/-AAPYcBKrAPw/XAUMSHTPlZI/AAAAAAACAeE/3dsZXLrbh2Yf1h-CStDQLpeq52RaSiyTgCLcBGAs/s1600/p1.PNG)

![Image](https://miro.medium.com/1%2AXXUG8rnUC188QYmbHJR4bA.png)

### 🎯 Objective

Implement **dynamic AKS scaling** at:

* Pod level (HPA)
* Node level (Cluster Autoscaler)

### 🧩 Steps

1. Enable **Cluster Autoscaler**
2. Deploy CPU-based app
3. Configure **HPA**
4. Generate load
5. Observe pod scaling
6. Observe node pool scale-out

### 🔍 Validation

* Pods scale first
* Nodes scale when needed
* Scale-in without disruption

### 💬 Interview Talking Points

* HPA vs Cluster Autoscaler
* Scaling pitfalls
* DB overload risk

---

## 🔵 LAB 6: AKS Observability (Prometheus + Azure Monitor)

![Image](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/media/kubernetes-monitoring-overview/layers.png)

![Image](https://learn.microsoft.com/en-us/samples/azure-samples/aks-managed-prometheus-and-grafana-bicep/aks-managed-prometheus-and-grafana-bicep/media/architecture.png)

![Image](https://intercept.cloud/media/sutdkita/aks_monitoring.webp?height=434\&rmode=max\&width=772)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/advanced-container-networking-services/flow-log-stats.png)

### 🎯 Objective

Implement **full observability** for AKS.

### 🧩 Steps

1. Enable **Container Insights**
2. Enable **Managed Prometheus**
3. Deploy **Grafana**
4. Create dashboards (CPU, memory, latency)
5. Create alerts (pod crash, node pressure)

### 🔍 Validation

* Metrics visible
* Logs searchable
* Alerts trigger correctly

### 💬 Interview Talking Points

* Metrics vs logs vs traces
* Alert fatigue
* SRE golden signals

---

## 🔵 LAB 7: AKS CI/CD with GitOps (Argo CD)

![Image](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gitops-aks/media/gitops-ci-cd-argo-cd.svg)

![Image](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/gitops-aks/media/gitops-ci-cd-flux.svg)

![Image](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)

![Image](https://media.licdn.com/dms/image/v2/D5612AQE50_aRo4yIHw/article-cover_image-shrink_600_2000/B56ZnYqY3XJkAU-/0/1760276615076?e=2147483647\&t=gQ6UvMJEndFzRq65un5Qsds6p-iUVlbHkE_VTFZaEv8\&v=beta)

### 🎯 Objective

Implement **GitOps-based AKS deployments**.

### 🧩 Steps

1. Install **Argo CD** on AKS
2. Connect Git repository
3. Deploy app via Git commit
4. Observe auto-sync
5. Rollback via Git revert

### 🔍 Validation

* Git is source of truth
* No kubectl apply in prod
* Rollback is instant

### 💬 Interview Talking Points

* GitOps benefits
* Drift detection
* ArgoCD vs Flux

---

## 🔵 LAB 8: AKS Upgrade, DR & Troubleshooting

![Image](https://miro.medium.com/0%2ADg1Gd0UVtsWseELT)

![Image](https://learn.microsoft.com/en-us/azure/architecture/guide/aks/media/blue-green-aks-deployment-diagram-public-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-multi-region/images/aks-multi-cluster.svg)

![Image](https://cdn.prod.website-files.com/681e366f54a6e3ce87159ca4/68757f9e61edb27b795c9588_What-is-Crashloopbackoff-02.png)

### 🎯 Objective

Practice **real AKS failure & recovery**.

### 🧩 Steps

1. Perform **node image upgrade**
2. Upgrade AKS minor version
3. Simulate pod failure (`CrashLoopBackOff`)
4. Debug using logs & describe
5. Design **multi-region AKS DR**
6. Test failover manually

### 🔍 Validation

* No downtime during upgrade
* Failures diagnosed quickly
* DR design documented

### 💬 Interview Talking Points

* AKS upgrade strategies
* DR limitations
* Kubernetes failure modes

---

## 🧠 How to Explain in Interviews (Golden Rule)

Use this format:

> **Problem → Design Choice → Trade-off → Result**

Example:

> “We used **private AKS with Azure CNI** to meet compliance.
> Trade-off was IP planning complexity, but we achieved Zero Trust networking.”

---

Excellent choice 🔥
These are **REAL FAILURE–DRIVEN AKS TROUBLESHOOTING LABS** — exactly what **senior Azure / AKS interviews** simulate.

You will **break AKS on purpose**, diagnose it like a production incident, and fix it **step-by-step**.

---

# 🧨 AKS TROUBLESHOOTING LABS (REAL PRODUCTION FAILURES)

Each lab includes:

* 💥 **Failure injected**
* 🔍 **Symptoms**
* 🧠 **Root cause analysis**
* 🛠 **Step-by-step fix**
* 🎤 **How to explain in interviews**

---

## 🔴 LAB 1: Pods Stuck in `CrashLoopBackOff`

![Image](https://cdn.prod.website-files.com/681e366f54a6e3ce87159ca4/68757f9e61edb27b795c9588_What-is-Crashloopbackoff-02.png)

![Image](https://cdn.prod.website-files.com/626a25d633b1b99aa0e1afa7/66c75301ed5f21b4964b2226_66605d5facc1cdd6828aa251_image2.png)

![Image](https://miro.medium.com/v2/da%3Atrue/resize%3Afit%3A800/1%2AYLqEOnyKyX_s4giE6QkF0g.gif)

![Image](https://cdn.prod.website-files.com/681e366f54a6e3ce87159ca4/688696e07ad4f20cb5fb61b8_What-is-Crashloopbackoff-01.png)

### 💥 Failure

Deploy an app with an invalid startup command.

### 🔍 Symptoms

```bash
kubectl get pods
kubectl describe pod <pod>
```

* Pod restarts repeatedly
* `CrashLoopBackOff`

### 🧠 Root Cause

* App crashes immediately
* Kubernetes restarts container

### 🛠 Fix

```bash
kubectl logs <pod> --previous
```

* Fix entrypoint / command
* Re-deploy workload

### 🎤 Interview Explanation

> “CrashLoopBackOff usually indicates application-level failure, not AKS failure.
> I inspect logs, container command, env vars, and probes.”

---

## 🔴 LAB 2: Pods Stuck in `ImagePullBackOff`

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/extensions/media/cannot-pull-image-from-acr-to-aks-cluster/add-aks-load-balancer-public-ip.png)

![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-and-acr.png)

![Image](https://anupams.net/wp-content/uploads/2021/08/AKS-ACR-LinkedIn-1-1024x674.jpg)

![Image](https://cdn.prod.website-files.com/626a25d633b1b99aa0e1afa7/64e359df88cf4cc6bf6f3af7_tA78IlN9-Sl9ZxX4dFxpd0IUV0nGPwucDdBU_Lcmb88TE0YOKPisQiT5WayEFB87_YDv0tteBCA4IDRMF-Q4B4WajGh9Y-xNRT3MQ-U2coGtCXeNf-mLpDSZWufkHpX9LttuBeYiEu3GRgO8L3Jsjb_HOfq8VH-U.png)

### 💥 Failure

Deploy image from **private ACR** without permissions.

### 🔍 Symptoms

* `ImagePullBackOff`
* `401 Unauthorized`

### 🧠 Root Cause

* AKS not authorized to pull image

### 🛠 Fix (choose one)

```bash
az aks update --attach-acr <acr-name>
```

OR

* Use imagePullSecret

### 🎤 Interview Explanation

> “AKS pulls images using managed identity.
> Missing ACR role assignment causes ImagePullBackOff.”

---

## 🔴 LAB 3: Pods Pending – No Nodes Available

![Image](https://techcommunity.microsoft.com/t5/s/gxcuf89792/images/bS00MDM2MjU0LTU0NDQ0NGk1MTU5QUZFOEYwNkM2Mjk0?image-dimensions=410x660\&revision=7)

![Image](https://user-images.githubusercontent.com/8063058/115822055-3c864d80-a447-11eb-8ad9-0e446402a45c.png)

![Image](https://img.site24x7static.com/images/without-set-requests-limits-consume-memory-resulting-failure-pod.png)

![Image](https://us1.discourse-cdn.com/flex016/uploads/kubernetes/optimized/2X/c/c596c5a3c4d72cb63a571b2635e5112c5e72d199_2_690x388.png)

### 💥 Failure

Deploy pods requesting more CPU than available.

### 🔍 Symptoms

```bash
kubectl describe pod <pod>
```

* `Insufficient cpu`

### 🧠 Root Cause

* Node pool capacity exhausted
* Autoscaler disabled or max reached

### 🛠 Fix

* Enable cluster autoscaler
* Increase node pool max
* Reduce resource requests

### 🎤 Interview Explanation

> “Pending pods indicate scheduling failure, not app failure.
> I check resource requests vs node capacity.”

---

## 🔴 LAB 4: Service Not Reachable (Network Failure)

![Image](https://opengraph.githubassets.com/c46667ef10a07acec6c06852158aae0859cf0cff466c6d729b9c0069083a9878/Azure/AKS/issues/2899)

![Image](https://i.sstatic.net/LiUCz.png)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/basic-troubleshooting-outbound-connections/external-traffic-load-balancer.svg)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/connection-issues-application-hosted-aks-cluster/aks-cluster-app-request-flow.svg)

### 💥 Failure

Expose app via Service but traffic fails.

### 🔍 Symptoms

* Pod running
* Service exists
* No response

### 🧠 Root Cause

* Selector mismatch
* Wrong port/targetPort

### 🛠 Fix

```bash
kubectl describe svc <svc>
kubectl get endpoints
```

* Fix labels
* Fix ports

### 🎤 Interview Explanation

> “Service issues are usually selector or port mismatch, not networking.”

---

## 🔴 LAB 5: AKS API Server Not Reachable (Private Cluster)

![Image](https://res.cloudinary.com/samcogan/image/upload/v1641644799/akspe_doxl8q.png)

![Image](https://user-images.githubusercontent.com/1821568/41420138-71835d54-6fb9-11e8-9571-50be7a92cf95.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/private-clusters/aks-private-hub-spoke.png)

### 💥 Failure

Private AKS without DNS or connectivity.

### 🔍 Symptoms

* `kubectl get nodes` times out

### 🧠 Root Cause

* Missing private DNS resolution
* No jump host / VPN

### 🛠 Fix

* Use Bastion / Jump VM
* Verify private DNS zone linkage

### 🎤 Interview Explanation

> “Private AKS requires private DNS and controlled access paths.”

---

## 🔴 LAB 6: AKS Nodes `NotReady`

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/media/node-not-ready-basic-troubleshooting/aks-cluster-node-pools.png)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/connection-issues-application-hosted-aks-cluster/aks-cluster-app-request-flow.svg)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/basic-troubleshooting-outbound-connections/external-traffic-load-balancer.svg)

![Image](https://i.sstatic.net/jDHoR.png)

### 💥 Failure

Block outbound traffic using UDR/Firewall.

### 🔍 Symptoms

* Nodes `NotReady`
* Pods evicted

### 🧠 Root Cause

* AKS nodes cannot reach control plane or required endpoints

### 🛠 Fix

* Allow AKS required FQDNs
* Fix UDR / Firewall rules

### 🎤 Interview Explanation

> “AKS depends on outbound connectivity even in private clusters.”

---

## 🔴 LAB 7: Ingress Works Intermittently (AGIC)

![Image](https://learn.microsoft.com/en-us/azure/application-gateway/media/application-gateway-ingress-controller-overview/architecture.png)

![Image](https://opengraph.githubassets.com/b45fb541490bc70d38c478eef2d8881854192140c4ff5d1dccc0ef17348b0b32/Azure/application-gateway-kubernetes-ingress/issues/535)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/connection-issues-application-hosted-aks-cluster/ingress-resource-app-traffic-flow.svg)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/connection-issues-application-hosted-aks-cluster/aks-cluster-app-request-flow.svg)

### 💥 Failure

Incorrect health probe path.

### 🔍 Symptoms

* 502 / 503 errors
* Backend unhealthy

### 🧠 Root Cause

* App doesn’t respond on probe path

### 🛠 Fix

* Fix probe path
* Align readinessProbe

### 🎤 Interview Explanation

> “Ingress health is tied to both Kubernetes and App Gateway probes.”

---

## 🔴 LAB 8: HPA Not Scaling Pods

![Image](https://www.apptio.com/wp-content/uploads/hpa-autoscaling.png)

![Image](https://miro.medium.com/1%2Ag_CCHx8E6ZNlRXaTBtfuuQ.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/cluster-autoscaler/cluster-autoscaler.png)

![Image](https://i.sstatic.net/jntrX.png)

### 💥 Failure

HPA configured but no scaling.

### 🔍 Symptoms

* CPU load high
* Pods not scaling

### 🧠 Root Cause

* Metrics server missing
* Requests not defined

### 🛠 Fix

```bash
kubectl top pods
```

* Install metrics server
* Define CPU requests

### 🎤 Interview Explanation

> “HPA scales on requested resources, not actual usage.”

---

## 🔴 LAB 9: Cluster Autoscaler Not Scaling Nodes

![Image](https://i0.wp.com/rajanieshkaushikk.com/wp-content/uploads/2023/01/AKS-autoscaler.png?fit=888%2C468\&ssl=1)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/cluster-autoscaler/cluster-autoscaler.png)

![Image](https://cdn.prod.website-files.com/622926e1a85e0cb10d8f5d5d/6717d149a8ee8ccd81c70d6f_AD_4nXegac-V69egxMmWfuCtUoRPwCid9MjmRed7Owo0zKAohib8S1_pZ-icKY3jIeuNbwKkfiQ40t7PQF5xXOOC5hHaj7SEJG3sRV7qItrIU4Z0Whch-vMOmNtSMJYo3R7-ZEtkzlQRAJRkKEDCfzt5wf19nPk.webp)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2AB9BpicfTrwCQJ03paqcIiw.png)

### 💥 Failure

Max node count reached.

### 🔍 Symptoms

* Pods pending
* No new nodes

### 🧠 Root Cause

* Autoscaler max limit
* Quota exhaustion

### 🛠 Fix

* Increase max nodes
* Request quota

### 🎤 Interview Explanation

> “Autoscaler obeys limits — it won’t violate safety constraints.”

---

## 🔴 LAB 10: Secrets Not Available in Pods (Key Vault CSI)

![Image](https://res.cloudinary.com/samcogan/image/upload/v1644592102/CSI_driver_Interface_aimaap.png)

![Image](https://i0.wp.com/baeke.info/wp-content/uploads/2022/11/image-8.png?fit=1200%2C650\&ssl=1)

![Image](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/aks-front-door/media/aks-front-door.svg)

![Image](https://learn-attachment.microsoft.com/api/attachments/a085d0c9-5a0e-4b71-9108-bbff8a6be260?platform=QnA)

### 💥 Failure

Wrong identity binding.

### 🔍 Symptoms

* Pod starts
* Secret mount fails

### 🧠 Root Cause

* Missing federated identity
* RBAC misconfiguration

### 🛠 Fix

* Verify identity mapping
* Fix Key Vault RBAC

### 🎤 Interview Explanation

> “Workload Identity failures are identity or RBAC related, not AKS.”

---

## 🔴 LAB 11: Pods Evicted Randomly

![Image](https://cdn.prod.website-files.com/681e366f54a6e3ce87159ca4/6877c46ca4e4034b4436661f_kubernetes-resources-cheatsheet-1170x585.png)

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/media/identify-memory-saturation-aks/nodes-containerinsights-memorypressure.png)

![Image](https://mihai-albert.com/wp-content/uploads/2022/02/image-1.png?w=1024)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/concepts-scale/cluster-autoscaler.png)

### 💥 Failure

Memory pressure on nodes.

### 🔍 Symptoms

* `Evicted` pods

### 🧠 Root Cause

* No resource limits
* Overcommitment

### 🛠 Fix

* Set limits
* Increase node size

### 🎤 Interview Explanation

> “Evictions protect node stability — it’s a capacity planning issue.”

---

## 🔴 LAB 12: AKS Upgrade Causes Downtime

![Image](https://miro.medium.com/0%2ADg1Gd0UVtsWseELT)

![Image](https://opengraph.githubassets.com/afdbe9dd0e1ee7558efa380351a2d8461ec8018975bf7e441a01a0652f776966/Azure/AKS/issues/5129)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2ACT2KDHWSCI26QCTi6vrogw.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/node-updates-kured/node-reboot-process.png)

### 💥 Failure

Upgrade without PDBs.

### 🔍 Symptoms

* Pods killed
* App unavailable

### 🧠 Root Cause

* No Pod Disruption Budget
* Single replica workloads

### 🛠 Fix

* Add PDB
* Increase replicas

### 🎤 Interview Explanation

> “AKS upgrades are safe only if workloads are disruption-aware.”

---

Perfect. This is the **AKS FAILURE MATRIX** senior interviewers *love* because it proves **pattern recognition + production experience**, not memorization.

Below is a **clear, interview-ready matrix**:
**SYMPTOM → ROOT CAUSE → FIX → PREVENTION (Architect View)**

I’ll reference **Azure Kubernetes Service (AKS)** once and keep this practical.

---

![Image](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/media/basic-troubleshooting-outbound-connections/external-traffic-load-balancer.svg)

![Image](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/images/aks-baseline-network-topology.svg)

![Image](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/eks-to-aks/media/control-plane-and-nodes.svg)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/what-is-aks/what-is-aks.png)

---

# 🧠 AKS FAILURE MATRIX (Production-Grade)

---

## 🔴 POD-LEVEL FAILURES

| Symptom                            | Root Cause                              | Immediate Fix                                    | Long-Term Prevention                         |
| ---------------------------------- | --------------------------------------- | ------------------------------------------------ | -------------------------------------------- |
| `CrashLoopBackOff`                 | App crash, bad command, missing env var | `kubectl logs --previous`, fix container command | Health checks, CI validation, startup probes |
| `ImagePullBackOff`                 | ACR auth missing, wrong image tag       | Attach ACR / fix imagePullSecret                 | Managed identity + ACR role assignment       |
| `Pending` pods                     | Insufficient CPU/memory                 | Scale node pool or reduce requests               | Capacity planning + autoscaler               |
| Pods restart randomly              | OOMKilled                               | Increase memory limits                           | Right-size requests/limits                   |
| Pods evicted                       | Node pressure                           | Add limits / bigger nodes                        | Resource governance                          |
| App works locally but fails in AKS | Missing config/secret                   | Validate ConfigMaps/Secrets                      | Config-as-code                               |
| Pods stuck `ContainerCreating`     | CSI / CNI issue                         | Check events + CSI logs                          | Version compatibility checks                 |
| Init container fails               | Dependency unreachable                  | Fix network/DNS                                  | Dependency health checks                     |
| Readiness probe fails              | Wrong path/port                         | Fix probe                                        | Align probes with app                        |
| Liveness kills healthy pod         | Aggressive probe                        | Increase delay/timeout                           | Tune probes per app                          |

---

## 🔴 SERVICE & INGRESS FAILURES

| Symptom                       | Root Cause                     | Immediate Fix          | Long-Term Prevention       |
| ----------------------------- | ------------------------------ | ---------------------- | -------------------------- |
| Service unreachable           | Selector mismatch              | Fix labels             | Standard labeling strategy |
| `kubectl get endpoints` empty | Pod labels don’t match         | Correct selectors      | CI validation              |
| LoadBalancer IP pending       | Quota / subnet issue           | Check limits           | Pre-allocate IP strategy   |
| Ingress returns 502/503       | Backend unhealthy              | Fix probe path         | Unified probe design       |
| Ingress works intermittently  | Readiness vs LB probe mismatch | Align probes           | Single source of truth     |
| AGIC backend unhealthy        | App path mismatch              | Fix ingress annotation | Ingress standards          |
| WAF blocks valid traffic      | False positives                | Exclude rule           | WAF tuning process         |
| TLS fails on ingress          | Cert mismatch                  | Fix secret/cert        | Cert automation            |
| NodePort works, LB doesn’t    | NSG blocks probe               | Allow probe IP         | NSG templates              |
| App Gateway routing wrong     | Host/path rule error           | Fix rule priority      | Routing design review      |

---

## 🔴 NODE & CLUSTER FAILURES

| Symptom                          | Root Cause           | Immediate Fix    | Long-Term Prevention    |
| -------------------------------- | -------------------- | ---------------- | ----------------------- |
| Nodes `NotReady`                 | Outbound blocked     | Allow AKS FQDNs  | Firewall allowlist      |
| Kubelet stopped                  | Disk full / OS issue | Restart node     | Disk monitoring         |
| Node pool stuck upgrading        | PDB blocks drain     | Fix PDB          | PDB standards           |
| Pods stuck `Terminating`         | Finalizers           | Remove finalizer | Resource hygiene        |
| Node scale-out fails             | Quota exhausted      | Request quota    | Capacity buffers        |
| Cluster autoscaler not scaling   | Max reached          | Increase max     | Scale limits policy     |
| Node joins but no pods scheduled | Taints               | Add tolerations  | Node pool strategy      |
| Node OS upgrade causes outage    | Single replica apps  | Add replicas     | HA by default           |
| Frequent node churn              | Spot evictions       | Use regular pool | Spot only for stateless |
| Disk pressure warnings           | Log growth           | Clean logs       | Log rotation            |

---

## 🔴 NETWORKING FAILURES (MOST COMMON IN INTERVIEWS)

| Symptom                             | Root Cause             | Immediate Fix    | Long-Term Prevention   |
| ----------------------------------- | ---------------------- | ---------------- | ---------------------- |
| Pod cannot reach internet           | UDR/firewall           | Allow outbound   | Egress design          |
| Pod cannot reach service            | NetworkPolicy          | Update policy    | Zero Trust design      |
| DNS resolution fails                | CoreDNS blocked        | Fix firewall     | DNS dependency mapping |
| Private endpoint unreachable        | DNS misconfig          | Link Private DNS | DNS architecture       |
| Intermittent network issues         | SNAT exhaustion        | Add NAT Gateway  | Outbound scaling       |
| On-prem cannot reach AKS            | Route missing          | Fix BGP/UDR      | Hybrid routing review  |
| AKS API unreachable                 | Private cluster access | Use Bastion/VPN  | Access patterns        |
| East-west traffic bypasses firewall | Missing UDR            | Add route        | Mandatory inspection   |
| Packet loss                         | Node NIC saturation    | Resize nodes     | Network sizing         |
| Slow service-to-service calls       | DNS latency            | Cache tuning     | DNS optimization       |

---

## 🔴 AUTOSCALING FAILURES

| Symptom                   | Root Cause        | Immediate Fix   | Long-Term Prevention |
| ------------------------- | ----------------- | --------------- | -------------------- |
| HPA not scaling           | No metrics server | Install metrics | Managed metrics      |
| HPA scales too late       | High thresholds   | Tune thresholds | Load testing         |
| HPA causes DB overload    | No rate limiting  | Add limits      | App-aware scaling    |
| Cluster autoscaler slow   | Scale-down delay  | Tune CA         | Autoscale tuning     |
| Pods pending during scale | Node startup lag  | Warm pools      | Pre-scaling          |
| Autoscaler loops          | Conflicting HPAs  | Simplify logic  | Scaling governance   |
| Scale-in kills sessions   | Stateful app      | Session store   | Stateless design     |
| CPU low but app slow      | Memory bound      | Use memory HPA  | Multi-metric HPA     |
| Burst traffic drops       | Cold start        | Pre-scale       | Predictive scaling   |
| Cost spike from scaling   | No max limits     | Set caps        | Cost guardrails      |

---

## 🔴 SECURITY & IDENTITY FAILURES

| Symptom                      | Root Cause             | Immediate Fix      | Long-Term Prevention |
| ---------------------------- | ---------------------- | ------------------ | -------------------- |
| Pod cannot access Key Vault  | Identity binding wrong | Fix federation     | Identity-as-code     |
| Secret mount fails           | RBAC missing           | Add role           | RBAC templates       |
| App uses secrets in YAML     | Bad practice           | Move to CSI        | Secretless standard  |
| ACR pull fails suddenly      | Role removed           | Reassign role      | Drift detection      |
| RBAC denies valid user       | Scope mismatch         | Fix scope          | RBAC hierarchy       |
| Excessive permissions        | Role sprawl            | Audit roles        | PIM + reviews        |
| Identity outage impacts apps | Central auth           | Cache tokens       | Resilience design    |
| Token expiration errors      | Short TTL              | Increase TTL       | Token policy         |
| Compromised pod access       | No network policy      | Add policy         | Zero Trust           |
| Security audit failure       | Missing logs           | Enable diagnostics | Compliance baseline  |

---

## 🔴 UPGRADE & MAINTENANCE FAILURES

| Symptom                   | Root Cause       | Immediate Fix     | Long-Term Prevention  |
| ------------------------- | ---------------- | ----------------- | --------------------- |
| Upgrade causes downtime   | No PDB           | Add PDB           | Upgrade readiness     |
| Node drain hangs          | Pod stuck        | Force drain       | Graceful shutdown     |
| App breaks after upgrade  | Deprecated API   | Fix manifests     | API deprecation scans |
| CSI breaks after upgrade  | Version mismatch | Upgrade driver    | Compatibility matrix  |
| Metrics disappear         | Agent restart    | Redeploy agent    | Managed add-ons       |
| Helm upgrade fails        | Immutable fields | Recreate resource | Helm practices        |
| Config drift post-upgrade | Manual changes   | Reconcile GitOps  | Git as source         |
| Rollback impossible       | Stateful change  | Restore backup    | Rollback plan         |
| Upgrade too slow          | Large nodes      | Parallel pools    | Rolling strategy      |
| Repeated failures         | No test env      | Stage upgrades    | Canary clusters       |

---

Excellent. We’ll run this exactly like a **real senior/principal AKS incident interview** — pressure, ambiguity, follow-ups, and decision trade-offs.

You are the **on-call AKS Lead Engineer**.
I am the **incident panel** (Platform + Security + Networking).

We’re operating on **Azure Kubernetes Service (AKS)**.

---

![Image](https://azure.github.io/AKS-DevSecOps-Workshop/assets/images/module0/devsecops-lifycycle-phases.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/concepts-clusters-workloads/control-plane-and-nodes.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/concepts-scale/burst-scaling.png)

![Image](https://learn.microsoft.com/en-us/azure/aks/media/limit-egress-traffic/aks-azure-firewall-egress.png)

---

# 🚨 MOCK AKS INCIDENT INTERVIEW (LIVE)

## ⏱ Incident Context (Read Carefully)

> **Time:** 10:17 AM (Peak business hours)
> **Environment:** Production
> **Cluster:** Private AKS, Azure CNI, App Gateway Ingress (WAF), Azure Firewall egress
> **Impact:** ~35% of users report intermittent failures
> **Symptoms:**
>
> * Some requests succeed, others return **502 / timeout**
> * No recent application deployment
> * Issue started **15 minutes ago**
> * Azure Portal shows cluster as *Healthy*

You are paged.

---

## 🎤 QUESTION 1 — FIRST 5 MINUTES (CRITICAL)

**What are the *first 5 things* you check — in order — and *why*?**

> ⚠️ Interviewer is testing:

* Incident prioritization
* Blast-radius thinking
* AKS mental model (Ingress → Service → Pod → Node → Network)

👉 **Your answer should be structured**, for example:

* Step 1: What
* Step 2: Why
* Step 3: Tool / command

✍️ **Reply with your answer to Question 1 only.**
I will then **challenge your assumptions and escalate the incident**.

---

### 💡 Tip (Do NOT say this out loud)

Strong candidates say things like:

* “I classify this as ingress-path degradation”
* “Intermittent 502 usually means backend health or networking”
* “Portal health ≠ workload health”

---

When you’re ready, answer **Question 1**.
I will immediately continue the interview based on *your response*.


Got it — I’ll **continue the mock AKS incident interview** and drive it forward like a real panel would.
Answer **mentally** as you read; I’ll also show you what a **strong senior-level response** looks like so you can self-calibrate.

---

# 🚨 MOCK AKS INCIDENT INTERVIEW (CONTINUED)

---

## 🎤 QUESTION 2 — INGRESS DEEP DIVE (10–15 MIN)

> You checked Application Gateway and notice:

* Backend pool shows **some pods healthy, some unhealthy**
* 502 errors correlate with unhealthy backends
* No recent deployments

### ❓ Question

**What are the top 5 reasons this can happen in AKS with App Gateway Ingress, and how do you validate each?**

### ✅ Strong Senior Answer (What interviewers want)

1. **Readiness probe failures**

   * Check: `kubectl describe pod`
   * Why: App Gateway respects readiness, not liveness

2. **Probe path mismatch**

   * App Gateway probe ≠ Kubernetes readiness probe
   * Validate ingress annotations + app endpoint

3. **Pod resource pressure**

   * Pods OOM / CPU throttled
   * Check: `kubectl top pods`

4. **Uneven traffic due to pod churn**

   * Pods restarting / evicted
   * Check events + restart count

5. **Network latency / SNAT exhaustion**

   * Especially with Azure Firewall in path
   * Check firewall logs + metrics

> 🎯 Key signal: *“Some pods healthy” = infra is partially working*

---

## 🎤 QUESTION 3 — ESCALATION (NETWORKING)

> New info:

* App pods show **sporadic readiness probe failures**
* Node metrics show **normal CPU/memory**
* Firewall metrics show **high SNAT port usage**

### ❓ Question

**Explain what SNAT exhaustion is in AKS, why it causes intermittent failures, and how you fix it immediately vs permanently.**

### ✅ Strong Answer

**What’s happening**

* Outbound connections exceed SNAT port limits
* Some requests succeed, others timeout → intermittent failures

**Immediate mitigation**

* Scale out NAT Gateway / Firewall
* Reduce connection churn (timeouts, keep-alive)

**Permanent fix**

* Attach **NAT Gateway** to AKS subnet
* Reduce egress via service endpoints / private endpoints
* Tune app connection pooling

> 🚩 Red flag if candidate says: “Increase node size” (wrong layer)

---

## 🎤 QUESTION 4 — SECURITY CHECK (PARALLEL TRACK)

> Security team asks:
> “Did our firewall change cause this?”

You find:

* A firewall policy update was applied **20 minutes ago**
* No rule rollback yet

### ❓ Question

**How do you prove (quickly) whether the firewall change is causal or coincidental?**

### ✅ Strong Answer

1. Correlate timestamps:

   * Incident start vs firewall policy commit
2. Check firewall logs:

   * Look for denies to AKS egress FQDNs
3. Bypass test:

   * Temporarily route traffic around firewall (test node pool)
4. Rollback policy safely:

   * Use versioned firewall policy

> 🎯 Key phrase interviewers love:
> **“Correlation is not causation — logs decide.”**

---

## 🎤 QUESTION 5 — DECISION TIME (BUSINESS PRESSURE)

> Leadership asks:
> “Can we stabilize in 5 minutes, even if it’s not perfect?”

### ❓ Question

**What is your fastest *safe* stabilization action, and what risk are you accepting?**

### ✅ Strong Answer

**Action**

* Temporarily relax firewall egress or attach NAT Gateway
* Scale out node pool to reduce SNAT pressure

**Risk accepted**

* Reduced egress control temporarily
* Slightly higher cost

**Why acceptable**

* Availability > perfect security during incident
* Changes are reversible and logged

> ❌ Weak answer: “Wait and investigate more”

---

## 🎤 QUESTION 6 — ROOT CAUSE STATEMENT (RCA TEST)

> Incident resolved after NAT Gateway attachment.
> Users stable.

### ❓ Question

**Give a 3-line executive RCA. No jargon.**

### ✅ Model Answer

> “A firewall policy change increased outbound connection load, causing SNAT exhaustion in our AKS cluster.
> This led to intermittent request failures.
> We mitigated by introducing a NAT Gateway and will permanently redesign egress capacity.”

---

## 🎤 QUESTION 7 — PREVENTION (ARCHITECT LEVEL)

**What 5 guardrails will you implement so this never happens again?**

✅ Expected points:

* NAT Gateway mandatory for AKS
* Egress load testing
* Firewall policy CI/CD with canary
* Alerts on SNAT utilization
* Private endpoints to reduce egress

---

## 🎤 QUESTION 8 — AKS MATURITY SIGNAL (FINAL)

**If this incident repeats in 6 months, what does that say about the platform team?**

✅ Strong answer:

> “It means we fixed symptoms, not systems.
> A mature platform learns once and encodes prevention.”

---

# 🏆 HOW YOU SCORE THIS INTERVIEW

If you:

* Classified the issue correctly → **Senior**
* Considered networking + ingress + app → **Lead**
* Spoke about guardrails + prevention → **Principal**
* Balanced security vs availability → **Architect**

---

## 🚀 Next Options

Reply with one of these:

* **HARDER INCIDENT** (multi-region AKS outage)
* **WHITEBOARD AKS DESIGN INTERVIEW**
* **AKS INCIDENT ANSWER KEY (PDF)**
* **REAL INTERVIEW CHEAT PHRASES**
* **AKS PRINCIPAL-LEVEL QUESTIONS**

You’re operating at **real production AKS interview level now** 💪
