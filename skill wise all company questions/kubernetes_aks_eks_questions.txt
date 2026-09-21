KUBERNETES, AKS, AND EKS INTERVIEW QUESTIONS
Company-wise extraction

Source: interview questions company wise.rtf
Total: 637 numbered questions, tasks, and topic prompts across
74 named companies plus the source's "Others" category.
Subquestions and requirements retained under a scenario count as part of that scenario.

Scope: Kubernetes architecture, workloads, scheduling, autoscaling, networking,
storage, security/RBAC, troubleshooting, upgrades, Helm, GitOps/Argo CD,
observability, AKS, EKS, and their cloud integrations.

Questions retain the source wording, with formatting cleaned and selected
prompts lightly edited for readability. Repeated questions from separate
interviews remain under their original company names. Embedded answer notes
are removed; no answers are added. Scenario premises come from the source.

Platform labels identify an explicit platform or a clear scenario context.
General Kubernetes concepts are also relevant to AKS and EKS. A few GKE/OpenShift
questions are retained as Kubernetes-related material and labeled separately.
[Follow-up] marks a general prompt retained from a Kubernetes scenario.
[Unclear] marks incomplete or ambiguous source wording.
[Relevant portion] marks the Kubernetes-related part of a mixed prompt.

PLATFORM COUNTS
  Kubernetes: 528
  AKS: 16
  EKS: 88
  AKS/EKS: 1
  Kubernetes/GKE: 3
  Kubernetes/OpenShift: 1

MANAGED-PLATFORM QUESTION INDEX
  AKS: 45, 46, 47, 48, 49, 53, 153, 234, 237, 276, 420, 421, 498, 499, 501, 541
  EKS: 4, 5, 6, 56, 78, 79, 80, 89, 91, 92, 97, 99, 101, 102, 103, 104, 107, 108, 109, 110, 113,
    114, 121, 122, 123, 161, 162, 163, 164, 165, 174, 180, 181, 182, 184, 213, 220, 232, 251, 318,
    337, 344, 345, 346, 348, 359, 370, 432, 457, 458, 459, 460, 461, 474, 475, 476, 477, 478, 479,
    480, 481, 482, 483, 484, 485, 514, 515, 516, 517, 518, 519, 520, 525, 526, 531, 532, 533, 568,
    609, 610, 614, 615, 619, 621, 622, 625, 626, 636
  AKS/EKS: 250
  Kubernetes/GKE: 15, 441, 442
  Kubernetes/OpenShift: 417

COMPANY INDEX
  AMEX: 3
  Accenture: 3
  Accion Labs: 2
  Accolite: 1
  Akamai: 1
  Alphadyne: 2
  Altimetrik: 6
  Amadeus Labs: 7
  Amazon: 11
  Arrise Solutions: 5
  Aspire: 3
  Blue Yonder: 9
  CGI: 1
  CMT: 2
  CTS (Cognizant): 6
  Capgemini: 24
  Cisco: 4
  Deloitte: 25
  EPAM: 21
  EXL Service: 7
  E&Y: 3
  Emphasis: 3
  Encora: 8
  F5: 3
  Flentas: 14
  Hexaware: 5
  IBM: 1
  ITC Infotech: 4
  Infinite Solutions: 3
  Infosys: 19
  HCL: 21
  Intact Green Services: 4
  JPMorgan: 13
  Koerber Pharma: 4
  LTIMindtree: 11
  L&T: 8
  Marsh McLennan: 2
  Morgan Stanley: 1
  NPCI: 5
  NUOS INFO Systems: 1
  NatWest Group: 1
  Netcracker: 19
  Nextturn: 7
  Nice: 6
  Nisum Technologies: 3
  Nitor Infotech: 6
  OPT IT: 2
  One2N: 5
  Oracle: 7
  Orion Innovation: 3
  Others: 91
  Perfios: 1
  Persistent Systems: 4
  Plansource ValueLabs: 1
  Publicis Global Delivery: 4
  Qburst: 6
  Rapidsoft: 3
  RelevantZ: 2
  SAP: 9
  Sapient: 5
  Sigmoid: 36
  Sonata Software: 4
  Sony: 12
  SquareOps: 22
  Syncortex: 5
  Synechron: 7
  TCS: 15
  Techdome: 2
  Verizon: 14
  Virtusa: 7
  Volkswagen Group Digital: 1
  Wipro: 21
  ZS Associates: 19
  Zensar: 8
  ZopSmart: 3

AMEX
----

1. [Kubernetes] Whats the difference between docker and kubernetes

2. [Kubernetes] How the end point authentication works in kubernetes

3. [Kubernetes] How did you troubleshoot the pod crashback loop

ACCENTURE
---------

4. [EKS] You have created an IAM user in AWS and configured role-based access in EKS. How do you
    bind the IAM user to the EKS role?

5. [EKS] If secrets are created in AWS Secrets Manager, how can Amazon EKS access those secrets?

6. [EKS] How do you set up RBAC in Amazon EKS?

ACCION LABS
-----------

7. [Kubernetes] How do you ensure high availability in Kubernetes?

8. [Kubernetes] How would you respond if the master node goes down? How would your response differ
    if a worker/slave node goes down?

ACCOLITE
--------

9. [Kubernetes] Explain kubernetes Architecture and component and their uses.

AKAMAI
------

10. [Kubernetes] How will you create Kustomize file

ALPHADYNE
---------

11. [Kubernetes] Describe a major production Kubernetes issue you solved. Explain the root cause
    analysis, troubleshooting approach, fix, and preventive measures.

12. [Kubernetes] Which object is used to manage a Kubernetes cluster and ensure that a pod always
    has specific resources?

ALTIMETRIK
----------

13. [Kubernetes] Explain what a Helm chart contains and how you integrate it into deployments.

14. [Kubernetes] What is defined in your values.yaml file?

15. [Kubernetes/GKE] An application or microservice is being deployed to a pod in GKE across
    different environments. The deployment fails, and the pod enters an error state and terminates.
    How would you troubleshoot it?

16. [Kubernetes] [Follow-up] What is your approach of doing a troubleshooting?

17. [Kubernetes] So how you are using terraform to deploy the cluster nodes?

18. [Kubernetes] You have optimized Kubernetes deployment configs. So can you explain me what have
    what was the role and what what have you done there?

AMADEUS LABS
------------

19. [Kubernetes] How to fix pod-level autoscaling not happening, what will be your approach .

20. [Kubernetes] Suppose we have application configured with hpa where it was running fine but
    suddenly it is not running what will be your approach?

21. [Kubernetes] How to access application if Ingress is configured but not accessible to end users,
    how you will resolve?

22. [Kubernetes] How Kubernetes handles service discovery can you explain .

23. [Kubernetes] [Follow-up] How to ensure all five application teams don't use more than a
    particular amount of space?

24. [Kubernetes] What type of autmation you have done in k8

25. [Kubernetes] Can a pod be scheduled with the following securityContext?

    securityContext:
      runAsNonRoot: true
      runAsUser: 0

AMAZON
------

26. [Kubernetes] How would you limit resource usage in Kubernetes without configuring it in
    deployment.yaml?

27. [Kubernetes] Updating worker nodes in k8s

28. [Kubernetes] Purpose of using CNI in K8s

29. [Kubernetes] How to configure Cluster auto-scaling, how to do that

30. [Kubernetes] If you’re not allowed to install Filebeat in your worker nodes for logging, then
    what will be possible option

31. [Kubernetes] If DB POD is down, will it affect the data it gets stored

32. [Kubernetes] What happens to sticky-session data if a pod goes down?

33. [Kubernetes] [Follow-up] What alternative service could you use to store sticky-session data?

34. [Kubernetes] [Follow-up] During a sticky-session problem, what could be the cause at the load
    balancer?

35. [Kubernetes] Do you create clusters in multi regions, is it possible? If yes, then how will you
    manage them

36. [Kubernetes] How will you take a back of your entire cluster regulary?

ARRISE SOLUTIONS
----------------

37. [Kubernetes] K8s architecture

38. [Kubernetes] CoreDNS in k8s

39. [Kubernetes] What’s the purpose of CNI

40. [Kubernetes] How does kube-proxy communicates with nodes

41. [Kubernetes] Purpose of scheduler in k8s

ASPIRE
------

42. [Kubernetes] What is the difference between replica sets and daemon sets?

43. [Kubernetes] What is the difference between pv and pvc in kuberenetes?

44. [Kubernetes] How to find orphan resources in kubernetes and remove it?

BLUE YONDER
-----------

45. [AKS] How do you use Azure Key Vault secrets in AKS?

46. [AKS] How will an application, container, or pod fetch the latest rotated secrets from Azure Key
    Vault?

47. [AKS] How do you secure your AKS cluster?

48. [AKS] How do you make your AKS cluster highly available?

49. [AKS] How do you perform cost optimization in AKS?

50. [Kubernetes] What types of HPA triggers have you used so far?

51. [Kubernetes] Stateful Vs Stateless

52. [Kubernetes] [Follow-up] How do you manage data of stateless application?

53. [AKS] How do you integrate Microsoft Entra ID with AKS for authentication?

CGI
---

54. [Kubernetes] Database connection from a pod is not working only for you. How will you
    troubleshoot?

CMT
---

55. [Kubernetes] Architecture of Kubernetes

56. [EKS] How would you maintain high availability in ecs + fargate or eks

CTS (COGNIZANT)
---------------

57. [Kubernetes] How will you know if a network policy is enabled or not in k8s

58. [Kubernetes] Difference between cluster role and cluster role binding

59. [Kubernetes] Difference between daemonset and state full set

60. [Kubernetes] What have you done in Kubernetes?

61. [Kubernetes] How did you write deployment files w.r.t microservices in your project and how did
    you configure services and ingress?

62. [Kubernetes] Did you configure Ingress and Egress rules?

CAPGEMINI
---------

63. [Kubernetes] How to assign memory to pod and how to make sure if pod should not get memory
    constraint. What to do if it happens.

64. [Kubernetes] What is the architecture of Kubernetes?

65. [Kubernetes] Kubectl apply command – how the services are used?

66. [Kubernetes] Explain the Deployment YAML file.

67. [Kubernetes] What are labels and annotations?

68. [Kubernetes] Traffic coming from outside to a cluster

69. [Kubernetes] What is the controller manager's task? [The source calls it the "control manager".]

70. [Kubernetes] Node affinity and anti-affinity

71. [Kubernetes] How to run 2 pods with one depending on another – how to set roles?

72. [Kubernetes] What is taint and toleration?

73. [Kubernetes] Reason for taint in worker nodes

74. [Kubernetes] How to limit resources in Kubernetes?

75. [Kubernetes] Blue-Green deployment

76. [Kubernetes] Canary deployment

77. [Kubernetes] What is CSI?

78. [EKS] How to configure a role to service accounts?

79. [EKS] [Unclear] Explain service accounts end to end. [The source wording ends with "end-to-en".]

80. [EKS] [Unclear] How do you create a "secret service account"? [The relationship between the
    Secret and service account is unclear in the source.]

81. [Kubernetes] Static vs dynamic storage provisioning

82. [Kubernetes] [Unclear] Explain errors related to pod labels. [The source only says "Type of
    error in pod label".]

83. [Kubernetes] If your pod is in Pending state, then what are your troubleshoot steps?

84. [Kubernetes] Different types of services in Kubernetes.

85. [Kubernetes] In a multi-cloud environment, if you want to block a pod to go into a particular
    node, how would you do it?

86. [Kubernetes] PVC.

CISCO
-----

87. [Kubernetes] Explain the upgrade process for a Kubernetes cluster with zero downtime.

88. [Kubernetes] [Follow-up] What key things should you verify post-upgrade?

89. [EKS] [Relevant portion] Explain the upgrade steps for EKS and on-premises Kubernetes clusters.

90. [Kubernetes] In a Kubernetes deployment managed through a Jenkins pipeline, a pod keeps
    restarting. What troubleshooting steps would you follow?

DELOITTE
--------

91. [EKS] What are the node groups you used in AWS EKS?

92. [EKS] What are the types of node groups in AWS EKS?

93. [Kubernetes] How do you create and manage Kubernetes clusters (using tools like Terraform), and
    what are the master and worker nodes?

94. [Kubernetes] What are common Kubernetes errors you’ve faced (like CrashLoopBackOff,
    ImagePullError), and how did you resolve them?

95. [Kubernetes] What is the command to access a pod and how can you define or create a Kubernetes
    class or object?

96. [Kubernetes] Explain the folder structure of a basic Helm chart. What commands do you use to
    deploy with Helm?

97. [EKS] How do you manage and connect services like DBs, EC2, EKS, or ECS? Include the command to
    connect to ECS.

98. [Kubernetes] [Follow-up] Which container registry do you use for storing Docker images?

99. [EKS] How do you handle authentication for EKS clusters and store secrets securely in your
    environment?

100. [Kubernetes] [Relevant portion] What is Helm chart signing, and which tools do you use to sign
    Helm charts?

101. [EKS] Difference between EKS and ECS?

102. [EKS] What are all the prerequisites for you to setup a EKS cluster with 2 worker nodes and
    some x no of pods?

103. [EKS] Generally when two or more pods are available how do you manage the load balancing? are
    you sure you'll be using ALB

104. [EKS] [Follow-up] When do you use ALB and When do you use NLB?

105. [Kubernetes] What is the use of Helm charts?

106. [Kubernetes] Explain me the high level architecture of Kubernetes?

107. [EKS] Difference between EKS and ECS?

108. [EKS] What are all the prerequisites for you to setup a EKS cluster with 2 worker nodes and
    some x no of pods?

109. [EKS] Generally when two or more pods are available how do you manage the load balancing? are
    you sure you'll be using ALB

110. [EKS] [Follow-up] When do you use ALB and When do you use NLB?

111. [Kubernetes] What is the use of Helm charts?

112. [Kubernetes] Explain me the high level architecture of Kubernetes?

113. [EKS] What will you do for zero-downtime when eks cluster upgrade

114. [EKS] What is your Terraform file structure for vpc, eks

115. [Kubernetes] What is the major issue that you resolved in Kubernetes

EPAM
----

116. [Kubernetes] How to Create and Use Custom Resources in Kubernetes

117. [Kubernetes] What are name spaces in k8s

118. [Kubernetes] What is difference between deployment and statefulset

119. [Kubernetes] What is role based control access

120. [Kubernetes] What is cluster auto scaler and horizontal auto scaling

121. [EKS] What is the ECS and eks

122. [EKS] Diff b/w Fargate vs EKS worker nodes

123. [EKS] Updating EKS cluster

124. [Kubernetes] How do you implement GitOps in a Kubernetes environment?

125. [Kubernetes] Can you explain how you would create a fully automated blue-green deployment in a
    Kubernetes-based microservices architecture?

126. [Kubernetes] How do you manage secrets and config securely at scale in Kubernetes without
    compromising GitOps workflows?

127. [Kubernetes] Explain the control plane components of Kubernetes and how you would harden them
    for production use.

128. [Kubernetes] How would you scale a Kubernetes cluster horizontally across multiple regions and
    still ensure zero-downtime upgrades?

129. [Kubernetes] What is a PodDisruptionBudget and how do you use it in critical workloads?

130. [Kubernetes] How do you implement and manage network policies in Kubernetes for strict inter-
    service communication?

131. [Kubernetes] How would you integrate runtime threat detection in Kubernetes using tools like
    Falco or Sysdig?

132. [Kubernetes] Pod is in crashback loop and logs are not showing anything, how will you
    troubleshoot?

133. [Kubernetes] What are the probes in kubernetes?

134. [Kubernetes] How will you deploy 10 pods in all nodes at a same time?

135. [Kubernetes] Difference between statefulset and deployment?

136. [Kubernetes] Explain RBAC in kubernetes.

EXL SERVICE
-----------

137. [Kubernetes] If Git is already the source of truth, why do we need Argo CD? Why not deploy
    directly using the CI/CD pipeline with Helm or kubectl?

138. [Kubernetes] Explain the complete request flow when a user accesses www.ingress.com until the
    request reaches the application pod.

139. [Kubernetes] You need to expose an application internally without using a LoadBalancer or
    NodePort service. How would you do it?

140. [Kubernetes] Pods in different namespaces can communicate. How would you block that
    communication? Where would you implement the NetworkPolicy?

141. [Kubernetes] [Follow-up] Suppose you are implementing a Canary deployment where only 10% of
    users receive the new version. How would you implement it through your CI/CD pipeline?

142. [Kubernetes] [Follow-up] Explain Rolling Update, Blue-Green, and Canary deployment strategies.

143. [Kubernetes] Where do you store application configuration and secrets? (ConfigMaps, Kubernetes
    Secrets, HashiCorp Vault, etc.)

E&Y
---

144. [Kubernetes] Asked about k8s ( deployment, services, and configs)

145. [Kubernetes] Service mesh

146. [Kubernetes] Pod disruption budget

EMPHASIS
--------

147. [Kubernetes] Explain Kubernetes architecture

148. [Kubernetes] [Follow-up] How is the image pulled from private repository

149. [Kubernetes] What is ingress in Kubernetes

ENCORA
------

150. [Kubernetes] What is CSI drivers

151. [Kubernetes] What is helm template and helm install

152. [Kubernetes] What is helpers files in helm

153. [AKS] How do you rotate secrets in Key Vault and implement a .pfx certificate in Application
    Gateway together with an ingress/controller in AKS?

154. [Kubernetes] Architecture of Kubernetes

155. [Kubernetes] What is daemonset and statefulset

156. [Kubernetes] Crashloopbackoff – what are the setps you will follow to troubleshoot further

157. [Kubernetes] What files are present in a Helm chart? What is a CNI plugin?

F5
--

158. [Kubernetes] What is K8. Explain the architecture.

159. [Kubernetes] Can I run POD inside master-node itself?

160. [Kubernetes] Have you deployed any security application on Kubernetes?

FLENTAS
-------

161. [EKS] Why did you not deploy the frontend on S3 and CloudFront, and instead deployed it on EKS?

162. [EKS] What is a namespace in EKS?

163. [EKS] How many namespaces do you currently have?

164. [EKS] What is a node group?

165. [EKS] A node is unable to join the cluster. What could be the reason?

166. [Kubernetes] What is the difference between the control plane and the data plane?

167. [Kubernetes] What is the difference between a Pod and a Container?

168. [Kubernetes] What are requests and limits in Kubernetes?

169. [Kubernetes] One node has 8 vCPU and 32 GB RAM. Pod autoscaling allows up to 4 replicas. Each
    pod has limits of 4 vCPU and 16 GB RAM, and requests of 2 vCPU and 10 GB RAM. How many instances
    of the pod can run?

170. [Kubernetes] What is a Helm chart?

171. [Kubernetes] During high traffic, how would you scale both cluster nodes and pods in
    Kubernetes?

172. [Kubernetes] Can we use Cluster Autoscaler?

173. [Kubernetes] When do we use VPA (Vertical Pod Autoscaler)?

174. [EKS] Have you upgraded the EKS cluster?

HEXAWARE
--------

175. [Kubernetes] What is deployment.yml?

176. [Kubernetes] What is service.yml?

177. [Kubernetes] What is replica set

178. [Kubernetes] What is the output of helm chart

179. [Kubernetes] What are the files available inside helm chart

IBM
---

180. [EKS] When deploying your application to Amazon EKS, what other services do you use along with
    it?

ITC INFOTECH
------------

181. [EKS] How will you write terraform module for EKS.

182. [EKS] If your application is on EKS how the traffic flows if user hits the URL.

183. [Kubernetes] What is difference between coreDNS and kube-proxy.

184. [EKS] [Follow-up] What is OIDC provider in AWS.

INFINITE SOLUTIONS
------------------

185. [Kubernetes] How you are mangaing the kubenretes DR

186. [Kubernetes] How you are taking backup kubernetes .

187. [Kubernetes] How you are setting up ingress controller

INFOSYS
-------

188. [Kubernetes] What will be your approach if pod.yaml failed

189. [Kubernetes] [Follow-up] Explain about blue green deployment strategy

190. [Kubernetes] Explain kubernets architecture

191. [Kubernetes] Commands used in kubernets

192. [Kubernetes] If application which you are trying to deploy with kubernets got crashed and you
    are not able to enter into pod what will be your approach

193. [Kubernetes] What are deployments, daemonset, statefulsets

194. [Kubernetes] Basic Kuberntes commands

195. [Kubernetes] How is traffic routed inside kuberntes clusters

196. [Kubernetes] ELB, Ingress questions

197. [Kubernetes] What is pod

198. [Kubernetes] What all Kuberntes issues you have worked on

199. [Kubernetes] If any pod/node goes down how do you troubleshoot/monitor that( via cluster and
    other monitoring tools)

200. [Kubernetes] What is HPA and how do you implement it

201. [Kubernetes] What is pod affinity

202. [Kubernetes] How many pods do you manage

203. [Kubernetes] Explain k8s architecture

204. [Kubernetes] What do you work on k8s

205. [Kubernetes] How were you managing your Kubernetes cluster through a helm file or the command
    line? Have you used Rancher or Argo CD?

206. [Kubernetes] Suppose you have a Kubernetes cluster running and in the cluster there is an
    issue. You see that one of the pod is in the state - crashloopbackoff. So what could be the
    possible issues with the pod?

HCL
---

207. [Kubernetes] What is diff between pv and pvc

208. [Kubernetes] Expalin kuberenets arthciecture

209. [Kubernetes] What is diff between deployment and statefullset

210. [Kubernetes] What is calico

211. [Kubernetes] What is etcd

212. [Kubernetes] How to take bakcup of kubenretes cluster

213. [EKS] How to upgrade eks clusetr

214. [Kubernetes] What is rolling update

215. [Kubernetes] [Follow-up] What is deployment statgergy you are using

216. [Kubernetes] Can we run 1 conatiner with 2 pods

217. [Kubernetes] What is statefullset

218. [Kubernetes] What is ingress controller

219. [Kubernetes] Supoose you have taked etcd backup and old vm corrupted,can we create new vm with
    backup etcd?

220. [EKS] What are steps to upgrade eks cluster

221. [Kubernetes] Suppose you have in satetfull 3 pods which having name mongo-0, mongo-1, mongo-2
    what happen if mongo-0 dies when new pod will create what will be pod new name?

222. [Kubernetes] Have you worked on argo cd, helm

223. [Kubernetes] Suppose we have pods 2 running in rolling updates some are in deployments set and
    some pods are in statefull set, how rolling updates strategy will work here?

224. [Kubernetes] What is a DaemonSet in Kubernetes, and what default daemons come with Kubernetes?

225. [Kubernetes] What are taints and tolerations in Kubernetes?

226. [Kubernetes] Explain compute reservations in Kubernetes manifest files.

227. [Kubernetes] In Kubernetes, how would you configure your deployment to double CPU allocation
    once usage crosses 70%?

INTACT GREEN SERVICES
---------------------

228. [Kubernetes] [Follow-up] What is desired state and in-desired state?

229. [Kubernetes] How do you deploy an application in Kubernetes?

230. [Kubernetes] Can we use POD as an agent? What are the drawbacks if we do so?

231. [Kubernetes] Type of services in Kubernetes, give their use case

JPMORGAN
--------

232. [EKS] What is the primary difference between ECS and EKS?

233. [Kubernetes] What is HPA

234. [AKS] An application deployed to Azure Kubernetes Service (AKS) randomly fails health checks.
    How would you debug this end to end?

235. [Kubernetes] [Follow-up] In a canary deployment to production, half the traffic returns 502,
    while others succeed. Walk us through your troubleshooting approach.

236. [Kubernetes] You see high CPU usage in one pod, but logs look clean. What next?

237. [AKS] An application running on AKS experiences 10-second delays every 15 minutes, with no code
    changes. How would you begin root cause analysis?

238. [Kubernetes] How would you set up an automated rollback strategy in Kubernetes for failed
    deployments?

239. [Kubernetes] What’s your approach to disaster recovery for stateful apps running on containers?

240. [Kubernetes] You have two running pods in a Kubernetes cluster, but they are unable to
    communicate with each other. There are no errors in the logs or events, and both pods appear
    healthy. How would you troubleshoot and restore communication between them?

241. [Kubernetes] If a pod’s liveness or readiness probe is failing, how would you troubleshoot the
    issue?

242. [Kubernetes] Apart from actuator health-check endpoints, what other checks can you perform
    using Kubernetes probes?

243. [Kubernetes] If an application has only one replica and you perform a rolling restart, will
    there be downtime? Also what events occur during the pod restart can you explain the sequence
    step by step?

244. [Kubernetes] You have an application with 2 replicas. During a rollout, the first pod is
    successfully replaced, but when the second pod is being replaced it enters a CrashLoopBackOff
    state. At that moment, which pod will the load balancer route traffic to the new pod, the old
    pod, or both? explain

KOERBER PHARMA
--------------

245. [Kubernetes] K8s architecture, services, if application pod fails how to troubleshoot

246. [Kubernetes] How to shecedlue a pod in specifc node

247. [Kubernetes] Diff between statefull set and stateless app

248. [Kubernetes] Can we delete pause conatiner

LTIMINDTREE
-----------

249. [Kubernetes] K8s architecture

250. [AKS/EKS] Create three different container images, store them in ECR or ACR, and deploy them to
    EKS or AKS. Write the Kubernetes YAML files.

251. [EKS] How do you upgrade your eks

252. [Kubernetes] How do you handle when pod dies

253. [Kubernetes] Create manifest for 2 nginx replicas

254. [Kubernetes] [Follow-up] How do you manage secrets securely in GitOps or deployment pipelines?

255. [Kubernetes] [Follow-up] How do you implement blue-green or canary deployments using container
    orchestration?

256. [Kubernetes] [Follow-up] How do you implement rollback in an automated deployment pipeline?

257. [Kubernetes] How do readiness and liveness probes work and why are they important in production
    environments?

258. [Kubernetes] How do you troubleshoot a pod that is stuck in CrashLoopBackOff?

259. [Kubernetes] How does your GitOps tool detect drift and how do you manage it?

L&T
---

260. [Kubernetes] Explain k8 archtiecure

261. [Kubernetes] What is loadbalancer in k8

262. [Kubernetes] What is ingress controller

263. [Kubernetes] [Unclear] Original fragment: "can we delete pod and multipilte container can run
    in". The question is incomplete in the source.

264. [Kubernetes] Access control in k8

265. [Kubernetes] What is kubeproxy

266. [Kubernetes] How to get static ip of k8 how to manage

267. [Kubernetes] How you are managing secrets in kubernetes

MARSH MCLENNAN
--------------

268. [Kubernetes] Walk me through the troubleshooting steps for a failed Helm deployment.

269. [Kubernetes] If a Helm release is partially deployed and some resources are updated while
    others have failed, how do you perform a rollback?

MORGAN STANLEY
--------------

270. [Kubernetes] Ingress vs Egress

NPCI
----

271. [Kubernetes] How would pod1 call pod2 without using a Service?

272. [Kubernetes] What is the Container Network Interface (CNI)?

273. [Kubernetes] What is CSI driver

274. [Kubernetes] Explain static and dynamic volume provisioning with use cases.

275. [Kubernetes] What is automatic volume expansion?

NUOS INFO SYSTEMS
-----------------

276. [AKS] Logs are incomplete. How would you troubleshoot across AKS, ingress, the application, and
    infrastructure?

NATWEST GROUP
-------------

277. [Kubernetes] How did u manage Kubernetes pods it is on Linux right?

NETCRACKER
----------

278. [Kubernetes] [Follow-up] What is meant by CPU throttling

279. [Kubernetes] Custom resource in k8s

280. [Kubernetes] What is ingress

281. [Kubernetes] Application is configured with Ingress but the webpage is not loading? What are
    the steps will be checked

282. [Kubernetes] How will you monitor the cluster through Prometheus

283. [Kubernetes] Upgrading the worker nodes in K8s

284. [Kubernetes] For junior team member, what are the roles will be provided in k8s

285. [Kubernetes] Diff between Role and Role binding

286. [Kubernetes] If I want to deploy my app in worker node2, what should I do?

287. [Kubernetes] Diff between Nodeselector, node affinity VS Taint, toleration

288. [Kubernetes] While updating your worker node, you're trying to perform drain out the PODs but
    some PODs are not removed from the node, what you will do

289. [Kubernetes] How would you grant view-only access to a Kubernetes cluster? What command or
    configuration would you use?

290. [Kubernetes] Storage classes in k8s

291. [Kubernetes] [Follow-up] NFS

292. [Kubernetes] What's the purpose of using storage class in k8s

293. [Kubernetes] I've two PODS in the same worker node, will they communicate with each other?

294. [Kubernetes] I've two PODS in diff worker nodes, can they communicate?

295. [Kubernetes] How would you restrict communication between the pods?

296. [Kubernetes] What component should be added in network policy YAML file

NEXTTURN
--------

297. [Kubernetes] Kubernetes – Troubleshoot a worker node that goes down.

298. [Kubernetes] Kubernetes – Troubleshoot a Pod stuck in Pending, CrashLoopBackOff, or
    ImagePullBackOff.

299. [Kubernetes] Helm – Upgrade failed. How do you rollback and troubleshoot?

300. [Kubernetes] GitOps – Push-based vs Pull-based deployment.

301. [Kubernetes] You have a Kubernetes cluster with 30 nodes. 29 nodes are Ready, but 1 node is
    NotReady. You have already checked kubectl logs, kubectl describe, and other basic commands. How
    will you troubleshoot the node further?

302. [Kubernetes] What are the common reasons for a Kubernetes node becoming NotReady, and how would
    you identify the root cause?

303. [Kubernetes] Describe your approach to troubleshooting Kubernetes worker node issues beyond the
    basic kubectl commands.

NICE
----

304. [Kubernetes] What is the deployment and statefullset?

305. [Kubernetes] What is the deployment?

306. [Kubernetes] How do you handle the HELM chart?

307. [Kubernetes] How you are maintaining ArgoCD for E1,E2,E3 env?

308. [Kubernetes] [Follow-up] How do you design and manage a containerized environment to ensure
    scalability and high availability?

309. [Kubernetes] In Kubernetes, how do you manage application deployment, scaling, and rollback?
    Can you walk through a specific scenario?

NISUM TECHNOLOGIES
------------------

310. [Kubernetes] Can we deploy services on master node?

311. [Kubernetes] If you pod is not running, how do you troubleshoot it?

312. [Kubernetes] Did you worked on helm charts?

NITOR INFOTECH
--------------

313. [Kubernetes] Gitops approach, ArgoCD, Flux

314. [Kubernetes] Statefulset, daemonset, Deployment

315. [Kubernetes] Networking in kubernetes, how have you implemented

316. [Kubernetes] Taints, tolerations, affinity

317. [Kubernetes] Create a Deployment named web-app using image nginx:1.25, with 3 replicas,
    container port 80, and the label app=web. Then write a NodePort Service to expose the
    application on port 8080.

318. [EKS] How have you implemented RBAC in your EKS setup

OPT IT
------

319. [Kubernetes] If dbs are in private subnets how do you deploy in kubernetes

320. [Kubernetes] HAVE YOU WRITTEN ANY KUBERNETES MANIFEST FILES,WHAT ARE THE KINDS YOU WROTE

ONE2N
-----

321. [Kubernetes] HPA implementation in detail

322. [Kubernetes] Why would you deploy rabbitmq as stateful set why not deployment

323. [Kubernetes] [Follow-up] How would you get application level metrics

324. [Kubernetes] How would HPA with stateful set work

325. [Kubernetes] Docker Swarm/Kubernetes

ORACLE
------

326. [Kubernetes] What's the purpose of using init containers in K8s

327. [Kubernetes] Stateful vs deployment in k8s

328. [Kubernetes] Configmap VS secrets

329. [Kubernetes] [Unclear] Explain a "Pod Distribution budget" in Kubernetes. [Term retained from
    the source; likely intended to refer to a Pod Disruption Budget.]

330. [Kubernetes] When you're trying to deploy a POD, it's throwing an error, how will you
    investigate.

331. [Kubernetes] How to deploy a POD into a certain NODE?

332. [Kubernetes] Explain me all the components present under deployment.yaml file

ORION INNOVATION
----------------

333. [Kubernetes] In Kuberenets there are 10 worker nodes, I have to deploy tomcat on each pods, how
    will you achieve it

334. [Kubernetes] I have an apache tomcat application running in k8 cluster, what are the manifests
    files you will be having inside in it

335. [Kubernetes] I have an local laptop, I want to access my hosted website, what service will be
    used in K8s

OTHERS
------

336. [Kubernetes] If you are implementing HPA for statefulsets if new pod comes the pvc would be
    empty? How would it be able to serve the request?

337. [EKS] When would prefer on-prem Kubernetes cluster over EKS and vice-versa

338. [Kubernetes] Kubernetes architecture in depth. Every component functioning. How would you join
    a new node to control plane?

339. [Kubernetes] Like kubelet is there any similar agent used to manage the control plane side of
    things?

340. [Kubernetes] When would you implement HPA and VPA. Give an example

341. [Kubernetes] Node selector, taints tolerations

342. [Kubernetes] Pod is in pending state. Reasons?

343. [Kubernetes] How would you implement security for Kubernetes(both on container side and the
    infra side using native Kubernetes solutions)

344. [EKS] What is the controller used to manage the self managed worker nodes

345. [EKS] What is karpenter. On which metric does it scale up and down?

346. [EKS] EKS cluster upgrade entire process

347. [Kubernetes] Helm commands, how would you deploy an application via helm. How do you integrate
    this entire process via CI/CD

348. [EKS] How many clusters are you managing currently, No of addons you have deployed

349. [Kubernetes] Why would you need an application to be deployed as stateful set

350. [Kubernetes] In Kubernetes, if a pod is in a pending state, how do you troubleshoot?

351. [Kubernetes] About K8's Architecture and tell me the workflow?

352. [Kubernetes] How many containers can run in a pod?

353. [Kubernetes] In ur projects how many containers u ran? can u give me the use case where can run
    4-5 containers in a pod?

354. [Kubernetes] About RBAC

355. [Kubernetes] There are 1 Master & 3 Worker nodes- if the master fails, what happens? Will pods
    keep running or they will crash?

356. [Kubernetes] In K8s, as etcd is a key-value store db, can write something manually on it?

357. [Kubernetes] How to roll back a failed deployment in Docker & K8s?

358. [Kubernetes] One of your worker nodes is not joining the cluster. How would you debug the
    issue?

359. [EKS] [Unclear] [Relevant portion] EKS and database automation and administration. [The source
    lists this topic without giving the individual scenario questions.]

360. [Kubernetes] I have 3 nodes (small, medium, and large), and I want only data load to go to the
    large node. How can I do that?

361. [Kubernetes] When I deploy the pods, it should be deployed on large and medium. Nodes, except
    small. How can I configure that?

362. [Kubernetes] Pods fail to schedule with the error "0/5 nodes are available: insufficient
    memory." What does the error mean, and how would you debug it?

363. [Kubernetes] What is the difference between scaling and autoscaling in Kubernetes?

364. [Kubernetes] If I don't specify TargetPort in the service object, what is it going to do?

365. [Kubernetes] What are the different types of secrets in Kubernetes?

366. [Kubernetes] I have an Ingress object that is not routing the traffic to the Kubernetes
    cluster. What are the reasons and how do you troubleshoot that?

367. [Kubernetes] I have created a service object that is not mapped to a deployment. What could be
    the reason and how do you debug it?

368. [Kubernetes] What are the different ways to specify the probes in Kubernetes?

369. [Kubernetes] What is an init container and why do we need to use it?

370. [EKS] What is the difference between EKS vs ECS vs Fargate?

371. [Kubernetes] What will happen if the k8 master node and worker node firewall gets broken? Will
    the existing deployments work or impact on any new deploymentsHow will you communicate to people

372. [Kubernetes] How to use the secrets in kubernetes? What encryption methods do you use?

373. [Kubernetes] Can you design the Istio Setup for your k8 cluster?

374. [Kubernetes] What will be the command to add the annotation and the labels for the existing
    pod?

375. [Kubernetes] Design the kubernetes cluster with Ingress.

376. [Kubernetes] Design the deployment of the pod with replica set set as 3 and having apache httpd
    image running as a container.

377. [Kubernetes] What is Taint/Tolerent.

378. [Kubernetes] What is stateful set.

379. [Kubernetes] Architecture of Kubernetes.

380. [Kubernetes] Use case of Node-Port and Cluster IP service Type in Kubernetes.

381. [Kubernetes] What is PDB in Kubernetes.

382. [Kubernetes] Difference between PV/PVC in Kubernetes.

383. [Kubernetes] Why Kube-let and Kube-proxy is used for in Kubernetes.

384. [Kubernetes] Build a container image and push it to ACR. How would you reference that image in
    a Kubernetes YAML file to deploy a pod?

385. [Kubernetes] What is POD in Kubernetes.

386. [Kubernetes] Types of Service in Kubernetes.

387. [Kubernetes] Namespaces in Kubernetes.

388. [Kubernetes] About Kubernetes architecture

389. [Kubernetes] Diff b/w Replicaset and Deployment

390. [Kubernetes] About ConfigMaps,PV,PVCs

391. [Kubernetes] Write a Deployment file

392. [Kubernetes] U handled any debug/troubleshoot for kubernetes?

393. [Kubernetes] Have you worked on the Kubernetes?So what deployment strategy are you following?

394. [Kubernetes] [Follow-up] So how are you implementing the blue green deployment?

395. [Kubernetes] Do you know what is HPA?

396. [Kubernetes] Suppose you deploy one application okay and you found some issue, you wanted to
    roll back using the kubernetes how you roll back to the particular version, what is the command?

397. [Kubernetes] What is the stateful set in the Kubernetes?

398. [Kubernetes] Why K8 instead of docker swam

399. [Kubernetes] Architecture of K8

400. [Kubernetes] [Follow-up] What is blue green deployment explain a project based on it

401. [Kubernetes] [Follow-up] Why canary and blue green differ

402. [Kubernetes] What happens if etcd stops working

403. [Kubernetes] What are the types of services

404. [Kubernetes] Explain a project in which u used Docker K8 and CICD

405. [Kubernetes] Can you explain the Kubernetes architecture and its components?

406. [Kubernetes] What is the difference between Pod and Deployment in Kubernetes?

407. [Kubernetes] What are the services in Kubernetes have?

408. [Kubernetes] What would you recommend: NodePort Service or LoadBalancer Service in Kubernetes
    and why?

409. [Kubernetes] What is the difference between Liveness and Readiness Probes in Kubernetes?

410. [Kubernetes] What is the difference between a Deployment and a StatefulSet in Kubernetes?

411. [Kubernetes] K8s node pending state how to debug

412. [Kubernetes] Pod is pending state, due to disk issue, how to resolve

413. [Kubernetes] Have u done k8s cluster upgrade

414. [Kubernetes] U r unable to evict the pods from node, how to resolve

415. [Kubernetes] [Follow-up] Tell me the flow of network packets starting from user hit the
    application url

416. [Kubernetes] A storage plugin issue prevents a cluster upgrade from completing. What would you
    do?

417. [Kubernetes/OpenShift] What is a pull secret in OpenShift?

418. [Kubernetes] What is kubernetes operator? If I need to run a shell script before any container
    to start how can i do it using operator?

419. [Kubernetes] What is the extra component/service present in Managed k8s cluster in cloud

420. [AKS] How would you allow only one pod or application to access a storage account while
    restricting all other pods in AKS?

421. [AKS] An AKS cluster has 32 GB of memory, of which 30 GB is already used. Can a new pod with a
    500 MB memory request and a 4 Gi memory limit be scheduled in the same cluster using HPA/VPA?

422. [Kubernetes] What are the alternate ingress controllers you suggest as Nginx IGC is deprecated

423. [Kubernetes] How do setup the communication between jenkins and kuberenetes

424. [Kubernetes] [Relevant portion] Which Kubernetes commands do you use?

425. [Kubernetes] Explain Kubernetes Structure, Config Map and Schedular..

426. [Kubernetes] How do you troubleshoot Imagepull backoff error?

PERFIOS
-------

427. [Kubernetes] Difference between cluster ip and node port?

PERSISTENT SYSTEMS
------------------

428. [Kubernetes] K8s Architecture.

429. [Kubernetes] How do you upgrage k8s cluster.

430. [Kubernetes] Pod Affinity and Node Affinity.

431. [Kubernetes] HPA & VPA.

PLANSOURCE VALUELABS
--------------------

432. [EKS] How you scale your EKS cluster based metrics/logs

PUBLICIS GLOBAL DELIVERY
------------------------

433. [Kubernetes] Kubernetes cluster upgrade from one version to another version? What is the
    approach?

434. [Kubernetes] [Unclear] What is "PDP" in Kubernetes? [The abbreviation is retained from the
    source and is not defined there.]

435. [Kubernetes] What are access modes in PVC?

436. [Kubernetes] You have one stateful application, that needs to be deployed specified node in
    k8s, how?

QBURST
------

437. [Kubernetes] Different types of services?

438. [Kubernetes] What is nodeport what are the cases we can use it?

439. [Kubernetes] What are loadbalancer used?

440. [Kubernetes] K8s command to list the pods with specific nodes?

441. [Kubernetes/GKE] How can you restrict public access to load balancers either standalone or gke?

442. [Kubernetes/GKE] How can you can you migrate one node pool vms to another node pool in gcp?

RAPIDSOFT
---------

443. [Kubernetes] You have a crashbackloop error. How would you fix this error?

444. [Kubernetes] Difference between deployment and stateful sets?

445. [Kubernetes] Explain terms in deployment.yml file in kubernetes

RELEVANTZ
---------

446. [Kubernetes] How do you login into pods using kubectl command

447. [Kubernetes] Tell me some commands on kubernetes – how do you troubleshoot it

SAP
---

448. [Kubernetes] A new deployment is implemented, and all pods, both new and old, suddenly crash.
    What could cause this?

449. [Kubernetes] [Follow-up] Which deployment is better cost wise.

450. [Kubernetes] I want my deployment to be implemented to specific workloads or regions, that
    update shouldn't go to other parts. There is an option in argo CD

451. [Kubernetes] How the auto scaling works, how things work in the back-end, from worker nodes to
    master nodes. Communication track behind that.

452. [Kubernetes] Can we perform Blue Green deployment under the same namespace? If yes, how will
    you manage them?

453. [Kubernetes] You did a deployment with Canary, when you will delete the old pods, what are the
    KPIs to cross check before deleting them.

454. [Kubernetes] [Follow-up] Once the blue green deployment is completed, how to check if the
    deployment is successful.

455. [Kubernetes] You’re trying to schedule a new POD but the new PODs are not deploying properly,
    what checks will be done.

456. [Kubernetes] Why do you want to use Argo CD over Jenkins?

SAPIENT
-------

457. [EKS] In which subnet are you placing your EKS cluster and which networking components have you
    used?

458. [EKS] In which way are you managing your cluster — using kubectl commands or something else?

459. [EKS] If your cluster is in a private subnet, then outside kubectl will not be working, right?
    How are you accessing that?

460. [EKS] What is EBS and EFS in Kubernetes?

461. [EKS] I have an S3 bucket, and there is some file inside it — my pod wants to access that S3
    bucket. How will it access it?

SIGMOID
-------

462. [Kubernetes] Diff between replica set vs replication controller

463. [Kubernetes] K8s backup policies

464. [Kubernetes] [Follow-up] How do you handle deployment failures

465. [Kubernetes] What happens if master node fails suddenly

466. [Kubernetes] How to limit the resource usage in K8s

467. [Kubernetes] Write a YAML file that configures resource requests and limits.

468. [Kubernetes] Write a Deployment manifest with the following requirements.
    Requirements:
    - Deployment name: space-alien-welcome-message-generator
    - Image: httpd:alpine
    - Replicas: 1
    - Readiness probe command: stat /tmp/ready; the pod should become ready when the file exists.
    - initialDelaySeconds: 10
    - periodSeconds: 5

469. [Kubernetes] If Liveness probe is healthy and readiness probe is failing, what will happen

470. [Kubernetes] Have you integrated Global LB with K8s cluster

471. [Kubernetes] How to enable RBAC to Service accounts

472. [Kubernetes] Which deployment strategy is best, assuming that you have only one POD is running

473. [Kubernetes] [Follow-up] If you're going with Blue Green deployment, how will you change your
    configuration to reroute the traffic between blue and green, exactly which configuration needs
    to changed?

474. [EKS] Design a three-tier AWS architecture with a frontend, backend, and database, considering
    security, high availability, and low latency. Justify whether the database should run as a
    StatefulSet or use RDS/another managed database, and whether the frontend should run as a pod or
    use S3 with CloudFront.

475. [EKS] [Follow-up] How can you tell a subnet is pubic or private, what things needs to be in
    place.

476. [EKS] How is an end user able to access the app which is running inside pods of private subnet
    nodes.

477. [EKS] [Follow-up] What is route53, how does the traffic actually flow in order, if a user
    requests or submits or posts anything from the UI.

478. [EKS] Explain each component in your architecture which is involved from a user requesting from
    the UI to the request reaching the backend pod, and how they are connecting with each other to
    ensure inbound and outbound flow(including the firewall, NACl, security groups, route tables
    etc).

479. [EKS] If you used a service with type: LoadBalancer and as you told its in private subnet, then
    how its able to launch a loadbalancer in public subnet, how its getting access to do it from
    within private subnet. Which component of the control plane takes care of it.

480. [EKS] Why did you choose eks cluster over ecs.

481. [EKS] How are you managing the cluster nodes.

482. [EKS] [Follow-up] For your specific architecture setup, how many ip addressess for all the
    components do you think will be sufficient enough from all the available IP's of both subnets
    within your VPC.

483. [EKS] Tell me how you handle the situation, if there is a DDoS attack in your cluster nodes or
    the cluster services, which is consuming all resources at 100% . What steps do you take in order
    to regain the condition back and what steps do you take to prevent this in future.

484. [EKS] How are you handling HA in your cluster, explain the reason why you use a specific
    feature over the other similar functionalities.

485. [EKS] What is the requirement of NAT gateway in your cluster, where should it placed to make it
    work as intended.

486. [Kubernetes] Which branching strategy are you following and how did you apply this strategy to
    integrate within your CI pipeline to deploy the pushed code to the right env cluster. Where
    exactly in your CI code you handled deploying to dev, and to QA and so on till its ready for
    production. How did you handled the PR checks and approvals in your CI pipeline.

487. [Kubernetes] What is k8s API. Can you connect to a k8s API using REST api calls directly from
    any external app, like how kubectl connects to the API endpoints exposed on the kube API server?

488. [Kubernetes] If there was an issue introduced in recent deployment, then how do we rollback the
    deployment, is it just the kubectl rollout undo command, how does it know which image it should
    revert to,does it take from the image repo or somewhere else?

489. [Kubernetes] Why do we need extra load balancing capabilities like host based, path based
    routing etc to our pods, if services are anyways handling the traffic to route to right pod,
    what actuslly is the issue where just tradional standalone service cannot handle it. For
    example, if an end user is traversing accross various product details, how is frontend able to
    fetch details from the right pod (backend which in turn gets the actual product data from db),
    explain how the api calls to backend and to db is handled.

490. [Kubernetes] [Follow-up] Write a PromQL expression to alert if CPU usage is above 80% on any
    node.

491. [Kubernetes] How will you alert if CPU usage stays above 90% for 5 minutes, but only if the
    number of running pods is below 5?

492. [Kubernetes] Custom resource vs custom resource definitions

493. [Kubernetes] Explain the end to end setup of ELK stack in the cluster that you are working on,
    in your current ongoing project.

494. [Kubernetes] [Follow-up] How is kibana connecting to Elasticsearch, write a yaml snippet where
    this connectivity is handled.

495. [Kubernetes] What are operators in kubernetes. How is the Elasticsearch working, is there some
    Elastic operator involved?

496. [Kubernetes] Consider there is a MySQl operator running in one of your pod in one of your node
    of a k8s cluster. How the Mysql database managed by this is different from the normal pod that
    is started with a mysql image from deployment/pod template? Apart from just handling the
    updates, version changes, lifecycle, vulnerabilitites, security issues, it provides lot more
    advantages, please explain the use cases by giving some scenarios.

497. [Kubernetes] If there is a requirement to run fixed 1 Mysql database accross each one of the
    nodes, how can you set this up in your cluster, with the help of custom operators.

SONATA SOFTWARE
---------------

498. [AKS] How do you protect endpoints in AKS?

499. [AKS] What networking do you use in AKS?

500. [Kubernetes] How do you monitor pods going down?

501. [AKS] Do you use Helm charts for AKS deployments?

SONY
----

502. [Kubernetes] So how do you guarantee zero downtime deployments in Kubernetes.

503. [Kubernetes] How do you start troubleshooting when a Kubernetes cluster feels slow?

504. [Kubernetes] How do you guarantee zero‑downtime deployments in Kubernetes?

505. [Kubernetes] Design a Kubernetes cluster that survives a complete availability-zone failure
    without data loss while running stateful workloads at scale. Address storage, networking,
    controllers, quorum, and recovery.

506. [Kubernetes] Explain the complete request flow when using Gateway API with multiple
    GatewayClasses across regions. How do you prevent split-brain routing?

507. [Kubernetes] How do you debug intermittent pod restarts when liveness probes pass, readiness
    passes, but the pod is still killed by the node?

508. [Kubernetes] What happens internally when etcd latency spikes above 500ms? How does it impact
    the scheduler, controllers, and API server?

509. [Kubernetes] Design a multi-tenant Kubernetes platform where teams must not affect each other’s
    resource usage, network traffic, or upgrade cycles.

510. [Kubernetes] How would you implement zero-trust networking inside Kubernetes without using a
    service mesh?

511. [Kubernetes] Describe a real production incident where a misconfigured HPA caused cascading
    failure. How would you redesign autoscaling to avoid this?

512. [Kubernetes] How would you design container images for ultra-fast cold starts in serverless or
    autoscaled Kubernetes environments?

513. [Kubernetes] How do you design GitOps for 1000+ clusters with environment drift detection,
    emergency hotfixes, and controlled manual overrides?

SQUAREOPS
---------

514. [EKS] [Relevant portion] Do you create EKS resources?

515. [EKS] Do you handle IAM + RBAC in Kubernetes?

516. [EKS] On which compute platform are the applications hosted?
    Follow-up questions:
    - Why EKS over ECS?
    - How do you manage worker nodes?
    - How many replicas do you run?
    - Do you use Cluster Autoscaler / Karpenter?

517. [EKS] Have you created an EKS cluster? Explain the process.
    Follow-up questions:
    - Did you use console, CLI, or Terraform?
    - Which VPC/subnet configuration did you use?
    - How did you configure node groups?
    - How do you bootstrap kubectl access?

518. [EKS] Have you upgraded an EKS cluster before? How?
    Follow-up questions:
    - What risks come with upgrading?
    - How do you handle node draining?
    - Do deployments get recreated?
    - What checks do you perform post-upgrade?

519. [EKS] If your teammate also wants access to the same cluster through kubectl, what steps do you
    follow?
    Follow-up questions:
    - What IAM policy do you attach?
    - What is aws-auth ConfigMap?
    - Where is aws-auth stored?
    - How do you map users and roles?
    - Do you provide RoleBinding or ClusterRoleBinding?

520. [EKS] In which Kubernetes resource do you map IAM users and roles?
    Follow-up questions:
    - What sections inside it? (mapUsers, mapRoles)
    - What mistakes can break authentication?

521. [Kubernetes] Have you worked with Helm and Helm Charts?
    Follow-up questions:
    - Why use Helm instead of plain YAML?
    - Show the folder structure of a Helm chart.
    - What is the templates folder?
    - What is values.yaml used for?
    - How do you manage multiple environments?

522. [Kubernetes] How do you securely inject sensitive data into Helm?
    Follow-up questions:
    - Do you use AWS Secrets Manager?
    - Do you avoid committing secrets in values.yaml?
    - What is Sealed Secrets?
    - What is the purpose of --set and --set-file flags?

523. [Kubernetes] Can a public Helm chart be customized?
    Follow-up questions:
    - Why is it not recommended to edit the chart directly?
    - How do you update config safely?
    - What happens during chart upgrades?
    - How do you extend a chart with new templates?

524. [Kubernetes] How do you add extra Kubernetes manifest files to a public Helm chart?
    Follow-up questions:
    - Where do you put extra YAML files?
    - How do you reference new values?
    - Can this break the original chart?

525. [EKS] How to implement shared storage across multiple pods running on multiple nodes in EKS?
    Follow-up questions:
    - Why EFS over EBS?
    - What is the EFS CSI driver?
    - What are the access modes?
    - How do you mount PVC in deployment?

526. [EKS] If there is one node but multiple pods, can we use EBS for shared storage?
    Follow-up questions:
    - What access mode does EBS support?
    - Why can't EBS work across multiple nodes?
    - When is EFS mandatory?

527. [Kubernetes] What is a Pod Disruption Budget (PDB)?
    Follow-up questions:
    - What is voluntary disruption?
    - What is involuntary disruption?
    - When do we use minAvailable vs maxUnavailable?

528. [Kubernetes] What is your approach to debug a CrashLoopBackOff?
    Follow-up questions:
    - What logs do you check?
    - How do you check events?
    - How do you inspect liveness/readiness probes?
    - How do you check resource limits?
    - How do you inspect environment variables and config maps?

529. [Kubernetes] During peak traffic, ingress controller is routing requests slowly. How do you
    debug it?
    Follow-up questions:
    - Check ingress controller logs?
    - Check CPU/memory usage?
    - Check pod replicas?
    - Do you use autoscaling (HPA)?
    - Load balancer throttling issues?
    - Endpoint misconfigurations?
    - Are readiness probes failing?
    - Is target response latency high?

530. [Kubernetes] Should we increase ingress controller replicas permanently or dynamically?
    Follow-up questions:
    - Why is static scaling bad?
    - When to use HPA?
    - When to use Cluster Autoscaler?

531. [EKS] Why choose EFS over EBS?
    Follow-up questions:
    - Which one supports multi-node?
    - Which one is cheaper?
    - Which one is faster?

532. [EKS] Can EBS be attached to multiple nodes? Why not?
    Follow-up questions:
    - Explain RWO vs RWX
    - Who enforces the mount restriction?

533. [EKS] How do you deploy to EKS through GitHub Actions?

534. [Kubernetes] How do you implement rolling deployments?
    Follow-up questions:
    - What happens to old pods?
    - What is maxSurge, maxUnavailable?

535. [Kubernetes] [Follow-up] Have you ever set up rollback in CI/CD?
    Follow-up questions:
    - How do you implement automatic rollback?
    - What triggers a rollback?
    - Is rollback handled by CI/CD or Kubernetes?

SYNCORTEX
---------

536. [Kubernetes] Taint and Tolerations

537. [Kubernetes] [Follow-up] If you want your developers to use only authorized images, what can we
    do?

538. [Kubernetes] How will you investigate POD failure

539. [Kubernetes] What are the parameters are used for HPA in K8s?

540. [Kubernetes] How will take a backup of K8s clusters regularly

SYNECHRON
---------

541. [AKS] How do you troubleshoot a failed pod in AKS? Provide the commands.

542. [Kubernetes] Tell me about Docker Compose and Kubernetes?

543. [Kubernetes] Tell me Kubernet architecture?

544. [Kubernetes] Can you explain how you would deploy a Kubernetes application using Jenkins? What
    plugins or tools would you use?

545. [Kubernetes] How do you monitor the health and performance of your Kubernetes pods in a
    production environment?

546. [Kubernetes] What is Helm, and why do you prefer to use it for managing Kubernetes applications
    instead of deploying them normally?

547. [Kubernetes] Can you describe the main components of a Kubernetes cluster and explain the role
    each one plays?

TCS
---

548. [Kubernetes] What are all the deployment startgies you use in deployments in k8s, explain
    canary and blue green strategies

549. [Kubernetes] What is the difference between PV and PVC in Kubernetes?

550. [Kubernetes] What is ConfigMap and Scheduler in Kubernetes?

551. [Kubernetes] How does Kubernetes work — how do the master and worker nodes communicate, and
    what runs inside them?

552. [Kubernetes] What is CrashLoopBackOff, and how do you troubleshoot it?

553. [Kubernetes] Why does a pod show a “Pending” status in Kubernetes?

554. [Kubernetes] [Follow-up] If a rollback fails, how will you handle it?

555. [Kubernetes] What is the command for rolling back to a specific revision in Kubernetes?

556. [Kubernetes] What is a PVC in Kubernetes?

557. [Kubernetes] Why do we need a StatefulSet when we can attach a PVC to a Deployment and make it
    stateful?

558. [Kubernetes] If a pod is created with a Deployment and another with a StatefulSet, will the
    StatefulSet pod always remain on the same node?

559. [Kubernetes] If we can run MySQL with a Deployment and PVC, why do we need a StatefulSet?

560. [Kubernetes] What happens if we scale a Deployment with one PVC from 1 to 3 replicas?

561. [Kubernetes] What types of services exist in Kubernetes, apart from ClusterIP, NodePort, and
    LoadBalancer?

562. [Kubernetes] Why does a pod created from a Deployment have two sets of random characters in its
    name?

TECHDOME
--------

563. [Kubernetes] Kubernetes architecture

564. [Kubernetes] Deployment vs stateful set

VERIZON
-------

565. [Kubernetes] What are the disadvantages of each deployment models in k8s

566. [Kubernetes] In k8s architecture which component is not running as pod

567. [Kubernetes] K8s QOS (quality of service)

568. [EKS] Disadvantage of using ebs volumes in eks

569. [Kubernetes] Any reason why we cannot place our app pods in master node by default

570. [Kubernetes] Requests and limits in k8s

571. [Kubernetes] How many clusters u have in ur project and how many pods in nodes

572. [Kubernetes] What version of k8s u used and did u perform any cluster upgrade

573. [Kubernetes] How you switch between clusters tell me the command

574. [Kubernetes] What is context in k8s

575. [Kubernetes] Etcd is sql or no sql database and reason

576. [Kubernetes] Init container and sidecar container

577. [Kubernetes] Liveness and readiness probes

578. [Kubernetes] [Follow-up] What is OOM and how to resolve OOM issue

VIRTUSA
-------

579. [Kubernetes] Difference between deployement and replicaset and daoemonset and statefulset (what
    key words you will be writing over the, for ex : deployement.yml – rolling update, canary)

580. [Kubernetes] [Unclear] What does the chart.yml file contain in Kubernetes? [Filename retained
    as written in the source.]

581. [Kubernetes] What files will be present in helm chart

582. [Kubernetes] What you will declare in values.yml etc

583. [Kubernetes] What is node affinity and pod affinity in K8

584. [Kubernetes] What is taint and tolerations in K8

585. [Kubernetes] Am having some min max pods running, suppose on festival day traffic increases at
    that time I have to increase pods, when no traffic is there I have reduce the pods, how will you
    achieve it in K8

VOLKSWAGEN GROUP DIGITAL
------------------------

586. [Kubernetes] What are all production issues that you faced in k8s?

WIPRO
-----

587. [Kubernetes] If there is file which is being used by 2 customers, and need to deploy that file
    in k8s cluster and on prem as well, how to do that?

588. [Kubernetes] How to deploy an app to k8s cluster in terms of app deploy only ( basically
    explain CD part)

589. [Kubernetes] A secret is stored in a vault inside a pod, and that pod goes down. How would you
    troubleshoot the issue?

590. [Kubernetes] Explain the contents of deployment.yaml or a Helm chart.

591. [Kubernetes] How would you structure a multi-stage pipeline that builds, tests and deploys a
    containerized application to kubernetes using Github Actions.

592. [Kubernetes] [Follow-up] How would you parameterize a workflow so that downstream jobs know
    which environment to deploy to?

593. [Kubernetes] Describe the security implications of using Kubernetes secret in etcd without
    encryption?

594. [Kubernetes] When designing a microservices-oriented infrastructure, what technologies and
    components (like load balancer, service mesh, Kubernetes) would you bring in, and how would you
    design the estate?

595. [Kubernetes] When you have many services in a service mesh, how do you decide the number of
    control planes and data planes needed?

596. [Kubernetes] What security measures and policies should be put in place when using a service
    mesh?

597. [Kubernetes] Since some features of service mesh are also available through other tools, is it
    worth adding the burden of installing Istio service mesh into the estate?

598. [Kubernetes] [Follow-up] How do you handle the service discovery phase when moving from a
    monolith to microservices?

599. [Kubernetes] If you have a Kubernetes cluster with pods running, but when you hit the URL you
    get HTTP errors (403, 404, 503), what would be your troubleshooting steps?

600. [Kubernetes] Is a service mesh always needed, or are alternative tools sometimes enough?

601. [Kubernetes] Diff between GitHub actions and Argo cd

602. [Kubernetes] In kubernetes a pod is going to crashloopbackoff, how do you troubleshoot.

603. [Kubernetes] Kubernetes deployment done successfully but unable to access the application
    externally, how do you troubleshoot.

604. [Kubernetes] When kubernetes node fails what will happen.

605. [Kubernetes] In pod configuration what we should do from Production perspective

606. [Kubernetes] What is argocd and why we are using it

607. [Kubernetes] What is Gitops

ZS ASSOCIATES
-------------

608. [Kubernetes] [Follow-up] Design an architecture for the scenario: if I type www.application.com
    it should get resolved to the backend service

609. [EKS] Calico and VPC CNI plugin difference. Why one is preferred over other. How would they
    help in setting up networking for pod.

610. [EKS] How is an ip address allocate to a pod. Does CNI plugin use same CIDR range which is
    provided by VPC or different?

611. [Kubernetes] Two pods which are part of same replica set are not able to communicate with each
    other what may be the reason

612. [Kubernetes] How to handle the extra traffic coming onto pods? Which solution you can implement

613. [Kubernetes] After implementing HPA also some pods are in pending state. What maybe the reason

614. [EKS] How would you provision karpenter. What all things are needed in configuration

615. [EKS] Can you deploy mongo db database in EKS cluster. If yes how and what all configuration
    things you would need to keep in mind

616. [Kubernetes] There are 3 backend pods in 3 different region. If one pod goes down how would the
    request be managed

617. [Kubernetes] [Follow-up] Suppose we configured a load balancer but it’s not accepting HTTPS
    request what would you do?

618. [Kubernetes] [Follow-up] Without installing certificate how would you divert the traffic coming
    from http to https

619. [EKS] How would karpenter know which node to provision. How would it get to know about resource
    constraints

620. [Kubernetes] Possible reasons for pod to be stuck in Crashloopbackoff

621. [EKS] A backend pod needs to interact with S3 and lambda. How would you achieve it

622. [EKS] How would the service account know which role to assume. What all things you would need
    to configure in the service account

623. [Kubernetes] Ingress, Gateway API

624. [Kubernetes] A replica set has 3 pods. One pod is not coming up. What maybe be the reason

625. [EKS] Argo CD. How do you manage CI/CD in your organisation. How many EKS clusters you manage,
    no of nodes

626. [EKS] Want to create a module for EKS cluster. What would be the structure

ZENSAR
------

627. [Kubernetes] Diff between replica set and deployment

628. [Kubernetes] Diff between stateless and stateful application

629. [Kubernetes] How configmap and secrets can be used in k8s

630. [Kubernetes] If control plane goes down what will happen to worker plane?

631. [Kubernetes] What is etcd and its use

632. [Kubernetes] [Unclear] Why is Kubernetes needed if Docker volumes are available? [The author
    says the exact question wording was not remembered.]

633. [Kubernetes] If 2 pod are in diff namespace then how can we make them communicate to each other
    securely?

634. [Kubernetes] What is ingress

ZOPSMART
--------

635. [Kubernetes] What are the commands do you know in k8s?

636. [EKS] You are using K8S Cluster as open source s/w tool like in aws service which service is
    available to create K8S Cluster

637. [Kubernetes] How to implement authentication in k8s cluster?

