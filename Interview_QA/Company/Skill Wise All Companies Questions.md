# Skill-Wise Interview Questions Across Companies

> A structured collection of DevOps, cloud, infrastructure, automation, CI/CD, scripting, and observability interview questions.

## Table of Contents

- [Kubernetes Interview Questions](#kubernetes-interview-questions)
- [Ansible Interview Questions](#ansible-interview-questions)
- [GitHub Actions Interview Questions](#github-actions-interview-questions)
- [AWS Cloud Services & Infrastructure Engineer – Interview Questions](#aws-cloud-services-infrastructure-engineer-interview-questions)
- [Terraform Interview Questions](#terraform-interview-questions)
- [Azure Cloud Infrastructure Engineer – Interview Questions](#azure-cloud-infrastructure-engineer-interview-questions)
- [Git / GitHub / GitLab / Azure Repos Interview Questions](#git-github-gitlab-azure-repos-interview-questions)
- [CI/CD, Build & Deployment Interview Questions](#cicd-build-deployment-interview-questions)
- [Shell / Bash / PowerShell / Python Scripting Interview Questions](#shell-bash-powershell-python-scripting-interview-questions)
- [Monitoring & Observability Interview Questions](#monitoring-observability-interview-questions)

## Kubernetes Interview Questions

1. What is Kubernetes, and how is it different from Docker?
2. Explain Kubernetes architecture and its components.
3. What are the control plane components in Kubernetes?
4. What is the role of the Kubernetes Controller Manager?
5. What is the purpose of the Kubernetes Scheduler?
6. What is the difference between a master node and a worker node?
7. How do master and worker nodes communicate?
8. What is the purpose of kubelet and kube-proxy?
9. What is etcd, and can we manually write data into it?
10. What is a Pod in Kubernetes?
11. What is the difference between a Pod and a container?
12. How many containers can run inside a single Pod?
13. Can one container run inside two different Pods?
14. Can we schedule Pods on a master node?
15. Why are application Pods not scheduled on master nodes by default?
16. What is the difference between a Deployment and a ReplicaSet?
17. What is the difference between a Deployment and a StatefulSet?
18. What is the difference between a ReplicaSet and a DaemonSet?
19. What is the difference between a DaemonSet and a StatefulSet?
20. What are the different Kubernetes workload resources?
21. What is the purpose of a DaemonSet?
22. Which default DaemonSets are present in Kubernetes?
23. What is the purpose of an init container?
24. How do you execute a shell script before an application container starts?
25. What is a Kubernetes Operator?
26. How do you create and use Custom Resources in Kubernetes?
27. What are labels and annotations in Kubernetes?
28. How do you add labels and annotations to an existing Pod?
29. What is a namespace in Kubernetes?
30. What are the different types of Services in Kubernetes?
31. What is the difference between ClusterIP and NodePort?
32. What is the difference between NodePort and LoadBalancer Services?
33. When would you recommend NodePort instead of LoadBalancer?
34. What are the different Kubernetes Service types apart from ClusterIP, NodePort, and LoadBalancer?
35. How does Kubernetes service discovery work?
36. What is CoreDNS in Kubernetes?
37. What is the purpose of CNI in Kubernetes?
38. What is the difference between Calico and the AWS VPC CNI plugin?
39. How is an IP address assigned to a Pod?
40. Does the CNI plugin use the VPC CIDR range or a different CIDR range?
41. What is the purpose of CSI drivers in Kubernetes?
42. What is a Kubernetes StorageClass?
43. What are PersistentVolumes and PersistentVolumeClaims?
44. What is the difference between PV and PVC?
45. What are the different PVC access modes?
46. What is the difference between static and dynamic volume provisioning?
47. How do you mount a PVC into a Deployment?
48. What is the difference between EBS and EFS storage in Kubernetes?
49. What is the purpose of the EFS CSI driver?
50. What is the difference between a StatefulSet and a Deployment when using persistent storage?
51. Why do we need StatefulSets when a Deployment can also use PVCs?
52. What happens when a Deployment sharing one PVC is scaled from one replica to three replicas?
53. Can multiple Pods on the same node share an EBS volume?
54. How would you implement shared storage across multiple Pods running on different EKS nodes?
55. What happens to data when a database Pod goes down?
56. What happens to sticky session data when a Pod goes down?
57. What is the difference between stateful and stateless applications?
58. How do you manage data for stateless applications?
59. How do you deploy MongoDB in an EKS cluster?
60. What storage limitations exist when using EBS volumes in EKS?
61. How do you deploy an application in Kubernetes?
62. How do you write Kubernetes Deployment YAML files?
63. Explain the different fields in a Kubernetes Deployment YAML file.
64. How do you deploy an Apache HTTPD container with three replicas?
65. How do you deploy a Tomcat application on every Kubernetes worker node?
66. How do you deploy multiple Pods across all worker nodes?
67. What happens internally when you run kubectl apply?
68. What are the commonly used kubectl commands?
69. How do you list Pods running on specific nodes?
70. How do you log in to a Pod using kubectl?
71. How do you switch between Kubernetes clusters using kubectl?
72. What is a Kubernetes context?
73. How do you access a private Kubernetes cluster using kubectl?
74. How do you configure Services and Ingress for microservices?
75. What is an Ingress in Kubernetes?
76. What is an Ingress Controller?
77. How do you configure an Ingress Controller?
78. What is the difference between Ingress and Egress?
79. What is the difference between Kubernetes Ingress and Gateway API?
80. Explain the complete request flow from a user accessing a URL until the request reaches an application Pod.
81. How does traffic flow inside a Kubernetes cluster?
82. How do you expose a Kubernetes application to external users?
83. How do you expose an application internally without using LoadBalancer or NodePort?
84. How do two Pods communicate without using a Service?
85. Can two Pods on the same worker node communicate?
86. Can Pods running on different worker nodes communicate?
87. How do Pods communicate across different namespaces?
88. How do you securely allow two Pods in different namespaces to communicate?
89. How do you restrict communication between Pods using NetworkPolicies?
90. How do you check whether NetworkPolicy enforcement is enabled?
91. What components do you configure in a NetworkPolicy YAML file?
92. How do you implement zero-trust networking inside Kubernetes without a service mesh?
93. How do you implement and manage NetworkPolicies for strict inter-service communication?
94. What is a service mesh?
95. How do you design an Istio service mesh for Kubernetes?
96. What security policies would you implement in a service mesh?
97. Is a service mesh always necessary, or can alternative tools provide similar capabilities?
98. What is the difference between service mesh control planes and data planes?
99. What are the alternative Ingress Controllers available for Kubernetes?
100. How do you restrict public access to Kubernetes LoadBalancers?
101. What is Horizontal Pod Autoscaler (HPA)?
102. What is Vertical Pod Autoscaler (VPA)?
103. What is Cluster Autoscaler?
104. What is the difference between HPA and VPA?
105. What is the difference between Cluster Autoscaler and HPA?
106. What is the difference between scaling and autoscaling in Kubernetes?
107. What metrics can trigger HPA?
108. How do you configure HPA using CPU utilization?
109. How would you configure a Deployment to increase CPU allocation when utilization crosses 70%?
110. How do you configure HPA for an application with changing traffic?
111. How do you scale both Pods and cluster nodes during high traffic?
112. When would you use HPA versus VPA?
113. How does HPA work with StatefulSets?
114. What happens to PVC data when StatefulSet Pods are created during HPA scaling?
115. How do you troubleshoot HPA when an application suddenly stops scaling?
116. Why might Pods remain Pending even after HPA is configured?
117. What are the differences between Kubernetes Cluster Autoscaler and Karpenter?
118. How do you optimize autoscaling costs in Kubernetes?
119. How do you configure resource requests and limits for Pods?
120. What happens when a Pod exceeds its memory limit?
121. How do you troubleshoot a Pod experiencing memory constraints?
122. How do you limit resource consumption at the namespace level?
123. How do you ensure different application teams do not exceed their resource quotas?
124. What are the Kubernetes Quality of Service (QoS) classes?
125. How does Kubernetes decide whether a Pod can be scheduled based on requests and limits?
126. A node has 8 vCPUs and 32 GB RAM. If each Pod requests 2 vCPUs and 10 GB RAM, how many Pods can be scheduled?
127. If an AKS cluster has 32 GB of memory with 30 GB utilized, can a Pod requesting 500 MB with a 4 GiB limit be scheduled?
128. How do you prevent critical workloads from being affected by resource-intensive applications?
129. How do you optimize Kubernetes deployment configurations?
130. What is a PodDisruptionBudget (PDB)?
131. How do you configure PDBs for critical applications?
132. What is node affinity and anti-affinity?
133. What is Pod affinity and anti-affinity?
134. What is the difference between nodeSelector, node affinity, and taints and tolerations?
135. What are taints and tolerations in Kubernetes?
136. Why are taints applied to worker nodes?
137. How do you schedule Pods on a specific node?
138. How do you prevent a Pod from being scheduled on a particular node?
139. How do you schedule an application only on large and medium-sized nodes, excluding small nodes?
140. How do you schedule a stateful application on a particular worker node?
141. How do you configure dependencies between two Pods?
142. What is the role of Kubernetes scheduling constraints?
143. What happens if a Pod's securityContext specifies runAsNonRoot: true and runAsUser: 0?
144. How do you create a Pod with a readiness probe that checks for the existence of /tmp/ready?
145. What is the difference between liveness, readiness, and startup probes?
146. What types of checks can Kubernetes probes perform?
147. What is the difference between liveness and readiness probes?
148. How do you configure Kubernetes probes?
149. How do you troubleshoot a failing liveness or readiness probe?
150. What happens when a Pod's liveness probe fails?
151. How do you troubleshoot a Pod stuck in Pending state?
152. How do you troubleshoot a Pod stuck in CrashLoopBackOff?
153. What are the possible causes of CrashLoopBackOff?
154. How do you troubleshoot CrashLoopBackOff when Pod logs show no errors?
155. How do you troubleshoot an ImagePullBackOff error?
156. What are common Kubernetes deployment errors, and how do you resolve them?
157. How do you troubleshoot a Pod that continuously restarts?
158. How do you troubleshoot a failed Pod YAML deployment?
159. How do you troubleshoot a Pod that crashes before you can execute commands inside it?
160. How do you investigate a Pod failure caused by insufficient resources?
161. How do you troubleshoot a Pod stuck in Pending because of disk issues?
162. How do you identify and remove orphaned Kubernetes resources?
163. How do you troubleshoot a worker node that is NotReady?
164. A cluster has 30 nodes, and one node is NotReady. How would you troubleshoot beyond basic kubectl commands?
165. What are the common reasons Kubernetes worker nodes become NotReady?
166. How do you troubleshoot a worker node that cannot join the cluster?
167. What happens when the Kubernetes master node goes down?
168. Will existing Pods continue running if the control plane fails?
169. What happens when a Kubernetes worker node fails?
170. What happens when a Pod is unexpectedly terminated?
171. How do you replace or recover Pods running on a failed node?
172. What happens when you delete a Pod managed by a Deployment?
173. How do you troubleshoot Pods that cannot be evicted during node draining?
174. How do you handle worker node updates when Pods refuse to terminate?
175. How do you troubleshoot network communication failures between two healthy Pods?
176. Two Pods in the same ReplicaSet cannot communicate. What could be the cause?
177. How do you troubleshoot database connectivity issues from a Kubernetes Pod?
178. How do you troubleshoot an Ingress that is not routing traffic?
179. How do you troubleshoot an application that is inaccessible even though its Kubernetes Deployment succeeded?
180. How do you troubleshoot HTTP 403, 404, and 503 errors in an application running on Kubernetes?
181. How do you troubleshoot an AKS application with intermittent health-check failures?
182. How do you investigate high CPU utilization in a Pod when logs show no errors?
183. How do you debug intermittent Pod restarts when liveness and readiness probes pass?
184. How do you troubleshoot periodic application latency in AKS when no code changes have occurred?
185. How do you troubleshoot incomplete logs across AKS, Ingress, application, and infrastructure components?
186. How do you troubleshoot an Ingress Controller that becomes slow during peak traffic?
187. Should Ingress Controller replicas be increased permanently or dynamically?
188. How do you troubleshoot a Deployment where all old and new Pods crash after a rollout?
189. How do you troubleshoot a ReplicaSet with three replicas when one Pod fails to start?
190. How do you debug a production Kubernetes issue and prevent it from recurring?
191. What are the deployment strategies supported in Kubernetes?
192. What is the difference between Rolling Update, Blue-Green, and Canary deployments?
193. How do you implement Blue-Green deployment in Kubernetes?
194. How do you implement Canary deployment in Kubernetes?
195. How do you implement automated Blue-Green deployments for Kubernetes microservices?
196. How do you route 10% of traffic to a new application version during a Canary deployment?
197. What metrics would you monitor before increasing Canary traffic from 10% to 100%?
198. How do you implement automated rollback for failed Canary deployments?
199. Can you perform Blue-Green deployments within the same Kubernetes namespace?
200. How do you implement zero-downtime deployments in Kubernetes?
201. What happens when a single-replica application undergoes a rolling restart?
202. During a rolling restart, what is the sequence of Pod termination and recreation?
203. During a two-replica rollout, if the new Pod enters CrashLoopBackOff, where will the load balancer route traffic?
204. How does a rolling update differ between Deployments and StatefulSets?
205. How do you roll back a Kubernetes Deployment to a specific revision?
206. What kubectl commands are used for Deployment rollback?
207. How do you manage Kubernetes application deployment, scaling, and rollback?
208. What happens to old Pods during a Deployment rollout?
209. For a mission-critical application, which deployment strategy would you choose and why?
210. What are the limitations of different Kubernetes deployment strategies?
211. How do you upgrade a Kubernetes cluster with zero downtime?
212. What are the steps involved in upgrading an EKS cluster?
213. What checks should you perform before and after a Kubernetes cluster upgrade?
214. How do you upgrade Kubernetes worker nodes?
215. How do you handle Pod eviction during a cluster upgrade?
216. How would you scale Kubernetes across multiple regions while ensuring zero-downtime upgrades?
217. How do you ensure high availability in Kubernetes?
218. How do you make an AKS cluster highly available?
219. How do you design a Kubernetes cluster that can survive an Availability Zone failure without data loss?
220. How do you design a multi-tenant Kubernetes platform with workload isolation?
221. How do you manage Kubernetes clusters across multiple regions?
222. How do you implement disaster recovery for a Kubernetes cluster?
223. How do you perform regular Kubernetes cluster backups?
224. What backup policies would you use for Kubernetes?
225. How do you restore a Kubernetes cluster after a failure?
226. What is Helm, and why is it used in Kubernetes?
227. What is a Helm chart?
228. What is the folder structure of a Helm chart?
229. What files are present inside a Helm chart?
230. What is the purpose of values.yaml?
231. What are Helm templates?
232. What is the purpose of _helpers.tpl in Helm?
233. What is the difference between helm template and helm install?
234. What are the commonly used Helm commands?
235. How do you deploy an application using Helm?
236. How do you integrate Helm deployments into a CI/CD pipeline?
237. What is the advantage of Helm over plain Kubernetes YAML files?
238. Can you customize a public Helm chart?
239. How do you add Kubernetes manifest files to an existing Helm chart?
240. How do you securely inject sensitive data into Helm charts?
241. What is Helm chart signing, and which tools are used for it?
242. How do you troubleshoot a failed Helm deployment?
243. How do you roll back a partially failed Helm release?
244. What is the difference between Helm-managed deployments and kubectl-managed deployments?
245. How do you manage Kubernetes applications using Helm, Rancher, or Argo CD?
246. What is Argo CD, and why is it used?
247. Why use Argo CD instead of Jenkins for Kubernetes deployments?
248. Why use Argo CD when Git is already the source of truth?
249. What is GitOps in Kubernetes?
250. How do you implement GitOps using Argo CD or Flux?
251. How do you manage Argo CD across development, QA, and production environments?
252. How do you manage Kubernetes configuration drift using GitOps?
253. How do you implement GitOps for more than 1,000 Kubernetes clusters?
254. How do you safely handle emergency hotfixes in a GitOps environment?
255. How do you restrict Argo CD deployments to specific workloads or regions?
256. How do you manage secrets and configurations in Kubernetes without compromising GitOps workflows?
257. How would you structure a GitHub Actions pipeline to build, test, and deploy an application to Kubernetes?
258. How do you deploy Kubernetes applications using Jenkins?
259. How do you deploy an application to EKS using GitHub Actions?
260. How do you create Kubernetes manifest files for multiple container images deployed to EKS or AKS?
261. What is RBAC in Kubernetes?
262. What is the difference between Role and ClusterRole?
263. What is the difference between RoleBinding and ClusterRoleBinding?
264. How do you configure RBAC in Amazon EKS?
265. How do you give a user read-only access to a Kubernetes cluster?
266. How do you give a junior engineer restricted Kubernetes permissions?
267. How do you implement Kubernetes authentication?
268. How does endpoint authentication work in Kubernetes?
269. How do you connect AWS IAM users or roles to Kubernetes RBAC?
270. How do you grant another engineer kubectl access to an EKS cluster?
271. What is the purpose of the aws-auth ConfigMap?
272. Which Kubernetes resource maps IAM users and roles to access permissions?
273. How do you handle authentication for EKS and securely store application secrets?
274. How do you integrate Microsoft Entra ID with AKS for authentication?
275. What are Kubernetes ServiceAccounts?
276. How do you associate an AWS IAM role with a Kubernetes ServiceAccount?
277. How do you securely allow EKS Pods to access AWS Secrets Manager?
278. How do you retrieve Azure Key Vault secrets from AKS?
279. How can an AKS Pod retrieve the latest rotated secrets from Azure Key Vault?
280. How do you restrict Azure Storage Account access to one specific Pod in AKS?
281. How does an EKS Pod securely access an S3 bucket?
282. How does an EKS backend Pod access S3 and AWS Lambda?
283. What are ConfigMaps and Secrets in Kubernetes?
284. What is the difference between a ConfigMap and a Secret?
285. What are the different types of Kubernetes Secrets?
286. How do you encrypt and manage Kubernetes Secrets?
287. What are the security implications of storing Kubernetes Secrets in etcd without encryption?
288. How do you rotate Kubernetes application secrets?
289. How do you securely inject configuration and secret values into Pods?
290. How do you secure an AKS cluster?
291. What are Kubernetes security best practices?
292. How do you harden the Kubernetes control plane for production?
293. How do you secure Kubernetes workloads at both container and infrastructure levels?
294. What should be included in a production-ready Pod configuration?
295. Have you deployed security applications in Kubernetes?
296. How do you implement runtime threat detection using Falco or Sysdig?
297. How do you configure certificate rotation for applications using AKS Ingress Controllers?
298. How do you implement workload isolation using Kubernetes security policies?
299. What security considerations apply when deploying a database in Kubernetes?
300. How do you enforce namespace-level and network-level security boundaries?
301. How do you monitor Kubernetes clusters using Prometheus?
302. How do you configure Prometheus and Grafana?
303. How do you integrate Grafana with Prometheus?
304. How does Prometheus collect Kubernetes metrics?
305. What are the key Prometheus components used for Kubernetes monitoring?
306. What Kubernetes metrics should be monitored in production?
307. How do you monitor Pod CPU and memory utilization?
308. How do you monitor Pod and node health in production?
309. How do you configure Grafana dashboards for Kubernetes?
310. How do you configure alerts for Kubernetes application failures?
311. How do you troubleshoot Pod and node failures using monitoring tools?
312. How do you collect application logs from Kubernetes Pods?
313. How do you collect Kubernetes logs without installing logging agents directly on worker nodes?
314. What are Prometheus, Grafana, and Loki, and how are they used together?
315. How do you implement centralized logging for Kubernetes across multiple cloud environments?
316. How do you configure Kubernetes monitoring to detect application performance degradation?
317. How do you use monitoring metrics to decide when to scale workloads?
318. How do you monitor the health of a Canary deployment?
319. How do you use observability data to troubleshoot Kubernetes production incidents?
320. How do you monitor and optimize resource utilization across Kubernetes clusters?
321. What is Amazon EKS, and how is it different from Amazon ECS?
322. What is the difference between EKS, ECS, and AWS Fargate?
323. What is the difference between AWS Fargate and EKS worker nodes?
324. Why would you choose EKS over ECS?
325. What are the prerequisites for creating an EKS cluster with two worker nodes?
326. How do you create an EKS cluster from scratch?
327. What are the different types of EKS managed node groups?
328. What node groups have you used in EKS?
329. How do you bootstrap kubectl access to EKS?
330. How do you create an EKS cluster using Terraform?
331. How do you structure Terraform modules for EKS?
332. How do you use Terraform to deploy Kubernetes cluster nodes?
333. What networking components are required for an EKS cluster?
334. Which subnets should be used for EKS nodes, and why?
335. How does an external request reach an application hosted in EKS?
336. How do you deploy an application in EKS with multiple replicas and load balancing?
337. How do you integrate AWS ALB or NLB with Kubernetes?
338. How do you configure EKS autoscaling based on workload metrics?
339. What are the differences between managed Kubernetes and on-premises Kubernetes?
340. When would you prefer on-premises Kubernetes over EKS?
341. What additional components or services are provided by managed Kubernetes platforms?
342. How do you create and manage multiple EKS clusters?
343. How do you perform cost optimization in AKS?
344. How do you integrate Azure Key Vault, Ingress Controllers, and application certificates in AKS?
345. How do you troubleshoot an application deployed in Google Kubernetes Engine (GKE)?
346. How do you secure a GKE cluster's external LoadBalancer?
347. How do you manage Kubernetes clusters across AWS, Azure, and Google Cloud?
348. How do you deploy applications across different Kubernetes environments?
349. How do you handle environment-specific Kubernetes configurations?
350. How do you automate provisioning, deployment, monitoring, and upgrades across managed Kubernetes clusters?

## Ansible Interview Questions

1. What is Ansible?
2. How does Ansible work?
3. Have you worked with Ansible? In what context?
4. What is your hands-on experience with Ansible?
5. Explain a real project where you used Ansible.
6. What tasks have you automated using Ansible?
7. What is an Ansible playbook?
8. How do you write and execute an Ansible playbook?
9. Write an Ansible playbook and explain it.
10. How do you write an Ansible playbook based on client requirements?
11. How do you install Ansible on Ubuntu and Red Hat?
12. How do you install and start services using Ansible?
13. How do you run an Ansible playbook, and what execution options are available?
14. What are Ansible modules?
15. Which Ansible modules have you used in your projects?
16. Which Ansible module is used to install an Nginx web server?
17. Which Ansible module ensures that a specific package is installed on a server?
18. What is an Ansible role?
19. How do you create an Ansible role?
20. What is the difference between Ansible roles and templates?
21. What is Ansible Galaxy?
22. How do you initialize an Ansible role and publish it to Ansible Galaxy for public use?
23. What is an Ansible inventory file?
24. What is the difference between static and dynamic inventory in Ansible?
25. How do you group hosts in an Ansible inventory file?
26. What are the best practices for naming groups and hosts in large Ansible inventories?
27. How do you manage Ansible variables for different environments?
28. How do you store variables in Ansible?
29. How do you pass input parameters to an Ansible playbook?
30. What is an Ansible Vault?
31. How do you encrypt data in Ansible?
32. How do you encrypt an Ansible playbook?
33. How do you execute an Ansible Vault file?
34. How do you store credentials securely in Ansible?
35. What alternative approaches can you use to store secret credentials securely?
36. How do you use environment variables to manage passwords in Ansible?
37. What is Ansible Tower?
38. Have you worked with Ansible Tower?
39. What is idempotency in Ansible?
40. What is the difference between Ansible and Salt?
41. What is the difference between Terraform and Ansible?
42. When would you use Terraform versus Ansible, and how do they work together?
43. What is the difference between import and include in Ansible?
44. What is a Jinja2 template in Ansible?
45. What is the purpose of templates in Ansible?
46. What is an Ansible ad-hoc command?
47. How do you execute a single command in Ansible without writing a playbook?
48. How do you shut down all servers using an Ansible ad-hoc command?
49. How do you run an Ansible playbook against all database servers except one?
50. How do you execute Ansible tasks as the root user?
51. How do you debug errors in an Ansible playbook?
52. What command do you use to debug an Ansible playbook?
53. How do you increase the Ansible debug log level?
54. How do you view Ansible logs during playbook execution?
55. What do you do if an Ansible playbook continues running for two to three hours?
56. How do you troubleshoot an Ansible playbook that times out on one host out of twenty?
57. How do you reduce Ansible execution time on an RDBMS server?
58. How do you optimize Ansible execution time using playbook configurations?
59. Why might you receive a "command not found" error when running modules such as yum or apt?
60. How do you pass the output of one Ansible task to another task?
61. How do you pass task results between different blocks or stages of an Ansible playbook?
62. How do you use output generated in one block in a later block?
63. How do you create three users and assign them to production, task, and QA groups in a single Ansible task?
64. Write an Ansible playbook to install Nginx and ensure the service is started and enabled at boot.
65. How do you securely manage secrets in an Nginx deployment playbook?
66. Write an Ansible playbook to install Apache on a virtual machine.
67. Write an Ansible playbook to install Docker on multiple nodes.
68. How do you create a MySQL dump using encrypted data in an Ansible playbook?
69. How would you deploy an application to 100 servers using Ansible?
70. How do you modify an Ansible playbook to meet different requirements on two virtual machines?
71. What are Ansible blocks?
72. What is the purpose of rescue and always in Ansible blocks?
73. How do you structure an Ansible role for a three-tier web application?
74. What are the advantages of using Ansible roles for managing a three-tier application?
75. How do you manage custom Ansible plugins shared by multiple roles?
76. How do you use Ansible conditionals to create cloud-agnostic roles?
77. How do you design an Ansible role that works across AWS, Azure, and GCP?
78. How have you used Ansible to automate and optimize deployments across large-scale infrastructure?

## GitHub Actions Interview Questions

1. Explain your GitHub Actions pipeline.
2. Explain the GitHub Actions workflow file.
3. What steps are included in your GitHub Actions workflow file?
4. How would you structure a multi-stage pipeline that builds, tests, and deploys a containerized application to Kubernetes using GitHub Actions?
5. How do you deploy an application to Amazon EKS through GitHub Actions?
6. Write a GitHub Actions workflow to build and push a Docker image.
7. How do you configure a GitHub Actions pipeline with AWS or Docker?
8. How do you configure static code analysis in a GitHub Actions pipeline?
9. Should SonarQube run on every pull request or every push?
10. How do you configure a manual trigger in GitHub Actions?
11. How do you configure GitHub Actions runners for an application environment?
12. How do you secure third-party GitHub Marketplace actions?
13. What is a matrix strategy in GitHub Actions?
14. How does caching work in GitHub Actions?
15. What is the needs keyword in GitHub Actions?
16. If one job depends on another job, how do you configure that dependency?
17. How do you run multiple jobs in parallel in GitHub Actions?
18. How do you prevent concurrent executions in GitHub Actions?
19. What is the difference between needs and concurrency in GitHub Actions?
20. How do you manage secrets in GitHub Actions pipelines?
21. How do you parameterize a workflow so downstream jobs know which environment to deploy to?
22. How do you manage multi-environment GitHub Actions pipelines?
23. How do you securely configure workload identity federation between GitHub Actions and Azure or Google Cloud?
24. Given a GitHub Actions workflow with incorrect steps, how would you troubleshoot and improve it?
25. How do you automate iOS builds using GitHub Actions?
26. What GitHub branching strategy do you use?
27. What is a stale branch in GitHub?
28. How do you use webhooks to trigger CI/CD pipelines?
29. How do you check the integrity of a Docker image or file in a CI/CD pipeline?
30. How do you configure security policies in a CI/CD pipeline?
31. How do you manage application credentials in CI/CD pipelines?
32. How do you implement GitHub Actions workflows for multiple deployment environments?
33. What is the difference between Jenkins and GitHub Actions?
34. What is the difference between GitHub Actions and Argo CD?

## AWS Cloud Services & Infrastructure Engineer – Interview Questions

### AWS Cloud Fundamentals and Architecture

1. What AWS services have you worked with in your projects?
2. Explain the AWS cloud architecture you have worked on.
3. What are AWS Regions and Availability Zones?
4. What is the difference between an AWS Region and an Availability Zone?
5. How do you design a highly available and fault-tolerant architecture in AWS?
6. Explain the architecture of a three-tier application in AWS.
7. What AWS services are required to deploy a three-tier application?
8. How would you design a secure, scalable and highly available application on AWS?
9. How do you design a system that can survive an entire AWS Region failure?
10. How do you ensure application availability, scalability and security?
11. How do you design a disaster recovery strategy for AWS infrastructure?
12. What AWS services do you use to deploy an application?
13. How do you provision AWS resources?
14. What infrastructure changes have you implemented in AWS?
15. What are AWS security best practices?
16. How do you design an AWS architecture for a high-traffic application?
17. How do you implement infrastructure automation in AWS?
18. What is serverless computing in AWS?
19. Which AWS services are used for serverless applications?
20. How do you manage infrastructure across multiple AWS accounts?
21. How do you manage infrastructure across multiple AWS Regions?
22. How would you design AWS infrastructure for 5 million or more users?
23. How do you design a highly available logging system for more than 100 microservices across three Regions?
24. What are the best practices for deploying highly available applications in AWS?
25. How do you troubleshoot a production AWS infrastructure issue?

### AWS EC2 – Elastic Compute Cloud

26. What is Amazon EC2?
27. What are the different types of EC2 instances?
28. How do you select an EC2 instance type for an application?
29. What is the difference between On-Demand, Reserved and Spot Instances?
30. What is the difference between an AMI and an EBS snapshot?
31. What is an Amazon Machine Image (AMI)?
32. How do you create an AMI from an existing EC2 instance?
33. Can you retrieve AMI details from an EBS snapshot?
34. How do you share an AMI between AWS accounts?
35. How do you share a KMS-encrypted AMI between two AWS accounts?
36. What is EC2 user data?
37. How do you use EC2 user data to automate software installation?
38. What is the difference between EC2 user data and a Terraform remote provisioner?
39. What is an EC2 instance profile?
40. What is the difference between an IAM role and an instance profile?
41. How do you attach an IAM role to an EC2 instance?
42. How do you connect to an EC2 instance without a password?
43. How do you connect to an EC2 instance without a PEM key?
44. How do you troubleshoot an EC2 instance that is running but cannot be accessed through SSH?
45. How do you access an EC2 instance running in a private subnet?
46. How do you provide internet connectivity to an EC2 instance in a private subnet?
47. How do you troubleshoot an EC2 instance without internet connectivity?
48. How do you securely connect two EC2 instances?
49. How do you attach an additional network interface to an EC2 instance?
50. What is the difference between a public IP and an Elastic IP?
51. How do you migrate an EC2 application from a public subnet to a private subnet without downtime?
52. How do you provide HTTPS access to an application running on EC2 in a private subnet?
53. How do you patch EC2 instances?
54. How do you identify and fix vulnerabilities in EC2 instances?
55. How do you separate application and monitoring storage on an EC2 instance?
56. How do you automate EC2 instance replacement when an instance fails?
57. How do you upgrade an EC2 instance from one instance type to another?
58. How do you deploy application updates to EC2 instances with minimum downtime?
59. How do you create multiple EC2 instances using Terraform?
60. How do you provision EC2 instances across multiple AWS Regions?
61. How do you prevent an EC2 instance from being deleted during Terraform destroy?
62. How do you import manually created EC2 instances into Terraform?
63. How do you ensure a particular AMI exists in an AWS account using Terraform?
64. How do you provision EC2 instances with different AMIs, instance types, volumes and tags?
65. How do you troubleshoot an EC2 application behind a load balancer that suddenly becomes unavailable?

### AWS VPC and Networking

66. What is an Amazon Virtual Private Cloud (VPC)?
67. What components are created by default when you create a VPC?
68. What is the difference between a public subnet and a private subnet?
69. What is the difference between a public IP and a private IP?
70. What is a subnet CIDR block?
71. Can you extend a subnet CIDR block after creating the subnet?
72. Can you create multiple CIDR ranges in the same VPC?
73. How do you calculate subnet ranges from a given VPC CIDR?
74. What is a route table in AWS?
75. How do you associate a route table with a subnet?
76. What is an Internet Gateway?
77. What is a NAT Gateway?
78. What is the difference between an Internet Gateway and a NAT Gateway?
79. What is the difference between a NAT Gateway and a NAT Instance?
80. Should a NAT Gateway be deployed in a public or private subnet?
81. How does a private subnet access the internet through a NAT Gateway?
82. Can a NAT Gateway accept inbound connections from the internet?
83. How does a NAT Gateway protect private instances?
84. How do you provide internet access to private EC2 instances without exposing them publicly?
85. What is a VPC Endpoint?
86. What is the difference between an Interface Endpoint and a Gateway Endpoint?
87. When would you use a VPC Endpoint?
88. How do you access Amazon S3 from a private EC2 instance without using the internet?
89. How do you connect to an RDS database in a private subnet without a NAT Gateway or bastion host?
90. What is VPC Peering?
91. How does VPC Peering work?
92. What are the limitations of VPC Peering?
93. What is an AWS Transit Gateway?
94. What is the difference between VPC Peering and Transit Gateway?
95. How do you connect multiple VPCs using Transit Gateway?
96. What route table changes are required after connecting VPCs to Transit Gateway?
97. How do you restrict communication between selected VPCs connected through Transit Gateway?
98. Can two VPCs communicate if their CIDR ranges overlap?
99. How do you connect two VPCs so that only one-way communication is permitted?
100. What is AWS Direct Connect?
101. What is AWS Storage Gateway?
102. What is a Virtual Private Gateway?
103. What is the difference between a Virtual Private Gateway and a Transit Gateway?
104. How do you connect an on-premises network to AWS?
105. How do you securely transfer files between on-premises infrastructure and AWS?
106. How do you connect AWS infrastructure to another cloud provider?
107. How do you troubleshoot VPC network connectivity issues?
108. What are VPC Flow Logs?
109. How do you identify traffic from a specific IP address using VPC Flow Logs?
110. How do you analyze VPC Flow Logs stored in Amazon S3?

### AWS IAM and Access Management

111. What is AWS Identity and Access Management (IAM)?
112. What is the difference between an IAM user and an IAM role?
113. What is an IAM policy?
114. What is the difference between an inline policy and a managed policy?
115. What is the difference between attaching policies to individual IAM users and IAM groups?
116. What is the principle of least privilege?
117. How do you implement least-privilege access in AWS?
118. How do you provide a user access to start and stop EC2 instances only?
119. How do you provide a user access to EC2 and RDS while restricting other AWS services?
120. How do you create an EC2 IAM role with access only to S3 and DynamoDB?
121. How do you restrict access to AWS resources for a particular user?
122. How do you securely provide administrator access to a new team member?
123. How do you manage authentication across ten AWS accounts without access keys?
124. What are the different ways to authenticate to AWS?
125. How do you access resources across multiple AWS accounts?
126. How do you configure cross-account IAM roles?
127. How do you configure IAM access when users belong to one AWS account and resources belong to another?
128. What is an IAM permission boundary?
129. Why might an IAM administrator be unable to access an S3 bucket?
130. Why might a user with an S3 access policy still receive AccessDenied?
131. What happens when an IAM allow policy conflicts with an explicit deny?
132. How do you attach IAM roles to EC2 instances?
133. What permissions can an EC2 instance obtain from its IAM role?
134. What is the difference between resource-based and identity-based policies?
135. How do you secure AWS root account access?

### Amazon S3 – Simple Storage Service

136. What is Amazon S3?
137. Does Amazon S3 require a VPC?
138. What are the different Amazon S3 storage classes?
139. What is the difference between Amazon S3 and Amazon EBS?
140. What is the difference between Amazon S3 and Amazon EFS?
141. How do you secure an S3 bucket?
142. How do you restrict public access to an S3 bucket?
143. What is an S3 bucket policy?
144. What is S3 versioning?
145. What are S3 lifecycle rules?
146. What are the advantages of S3 lifecycle policies?
147. How do you automatically delete S3 objects after a specified number of days?
148. How do you configure S3 lifecycle policies using Terraform?
149. What is Cross-Region Replication in S3?
150. How do you access an S3 bucket from another AWS Region?
151. How do you configure cross-account S3 access?
152. How do you connect EC2 to S3?
153. How do you automatically upload daily files generated on EC2 to S3?
154. How do you transfer EC2 log files to S3?
155. How do you automate S3 file uploads using Python?
156. How do you access S3 privately using a Gateway Endpoint?
157. How do you prevent unauthorized access to S3 objects?
158. How do you enable encryption for S3 data?
159. How do you manage S3 bucket lifecycle policies across different environments?
160. How do you troubleshoot an S3 AccessDenied error?

### Amazon EBS, EFS and Storage

161. What is Amazon EBS?
162. What is Amazon EFS?
163. What is the difference between EBS and EFS?
164. When would you use EBS instead of EFS?
165. What are the different EBS volume types?
166. How do you create an EBS snapshot?
167. How do you restore an EBS volume from a snapshot?
168. How do you attach an EBS volume to another EC2 instance?
169. Can an EBS volume be attached to multiple EC2 instances?
170. How do you perform backups of EBS volumes?
171. What is the difference between EBS snapshots and AMIs?
172. What file system would you use when migrating a monolithic application from on-premises to AWS?
173. How do you implement shared storage between multiple EC2 instances?
174. What is the purpose of the Amazon EFS CSI driver in EKS?
175. Why would you choose EFS over EBS for Kubernetes workloads?

### AWS Load Balancing and Traffic Management

176. What are the different types of AWS load balancers?
177. What is an Application Load Balancer (ALB)?
178. What is a Network Load Balancer (NLB)?
179. What is the difference between ALB and NLB?
180. When would you use ALB instead of NLB?
181. What is the difference between public-facing and internal load balancers?
182. How do you configure an internal ALB?
183. How do you route requests between multiple applications using an ALB?
184. What is a target group in AWS?
185. How do load balancer health checks work?
186. How do you troubleshoot unhealthy targets in an ALB target group?
187. How do you monitor load balancer health using AWS services?
188. What is connection draining?
189. What are sticky sessions in an ALB?
190. How do you troubleshoot sticky session issues?
191. How do you configure HTTPS on an AWS load balancer?
192. How do you troubleshoot a load balancer that is not accepting HTTPS requests?
193. What is AWS Certificate Manager (ACM)?
194. How do you manage SSL/TLS certificates in AWS?
195. How do you handle an expired certificate in AWS?
196. How do you troubleshoot HTTP 502, 503 and 504 errors from an AWS load balancer?
197. How does traffic flow between two applications in the same VPC, each behind a public ALB?
198. How do you prevent traffic between internal applications from unnecessarily leaving the VPC?
199. What is the difference in cost between a public and an internal ALB?
200. How do you protect an AWS load balancer from malicious traffic?

### Amazon Route 53, DNS and CloudFront

201. What is Amazon Route 53?
202. What are public and private hosted zones in Route 53?
203. What is the difference between public and private hosted zones?
204. How do you configure a domain purchased from GoDaddy or another registrar in Route 53?
205. How do you configure subdomains in AWS?
206. How does DNS resolution work in AWS?
207. What is DNS failover?
208. How do you configure DNS failover when one endpoint becomes unavailable?
209. How do you configure Route 53 routing for a highly available application?
210. What is Amazon CloudFront?
211. How do you configure CloudFront with Amazon S3?
212. How do you configure a CloudFront distribution?
213. How do you secure an application using CloudFront?
214. Why would you host a frontend application on S3 and CloudFront instead of EKS?
215. How do you configure DNS for an application deployed on ECS or Fargate?

### Amazon RDS and Database Infrastructure

216. What is Amazon RDS?
217. What is the difference between Amazon RDS and Amazon Aurora?
218. What is the difference between RDS Multi-AZ and Read Replicas?
219. What is the difference between a standby RDS instance and a read replica?
220. How do you configure RDS Multi-AZ?
221. How do you configure RDS read replicas?
222. How do you replicate an RDS database between two AWS Regions?
223. How do you perform horizontal and vertical scaling of an RDS database?
224. How do you upgrade an RDS MySQL database from version 7.0 to 8.0 or later?
225. How do you restrict RDS access to a single user at a time?
226. How do you securely access an RDS database in a private subnet?
227. How do you troubleshoot application slowness caused by RDS?
228. What RDS metrics do you monitor when investigating database performance?
229. How do you troubleshoot high memory utilization on an RDS instance?
230. What immediate actions can you take if RDS has memory pressure and resizing is not allowed?
231. What is Amazon RDS Proxy?
232. Why would you use RDS Proxy?
233. How do you perform database backups in AWS?
234. How do you implement database disaster recovery?
235. How do you securely manage RDS credentials?

### AWS Lambda, API Gateway and Serverless

236. What is AWS Lambda?
237. What is serverless computing in AWS?
238. What are the different AWS Lambda triggers?
239. What are the limitations of AWS Lambda?
240. Explain a real-world use case of AWS Lambda.
241. How do you create and deploy a Lambda function?
242. How do you manage Lambda deployment artifacts?
243. How do you secure AWS Lambda functions?
244. How do you configure environment variables in Lambda?
245. How do you transfer data between Lambda functions in different AWS accounts?
246. How do you troubleshoot a Lambda function that cannot connect to an external API?
247. How do you optimize AWS Lambda cold starts?
248. How do you run container images in AWS Lambda?
249. What is Amazon API Gateway?
250. How do you configure Amazon API Gateway?
251. How do you securely expose APIs using API Gateway?
252. How do you troubleshoot API Gateway timeout errors?
253. How do Lambda and API Gateway work together?
254. What is AWS Step Functions?
255. How do you use AWS Lambda with Step Functions?

### Amazon ECS, EKS and Fargate

256. What is Amazon ECS?
257. What is Amazon EKS?
258. What is AWS Fargate?
259. What is the difference between ECS and EKS?
260. What is the difference between ECS and Fargate?
261. What is the difference between EKS worker nodes and Fargate?
262. Why would you choose EKS over ECS?
263. How do you deploy ECS using EC2 instances?
264. What is an ECS task definition?
265. How does traffic flow from an external user to an ECS application running on Fargate?
266. How do you configure DNS for an ECS or Fargate application?
267. How do you perform cost optimization for ECS workloads?
268. How do you create an EKS cluster?
269. What are the prerequisites for creating an EKS cluster?
270. What are the different types of EKS node groups?
271. How do you create EKS worker nodes using Terraform?
272. How do you configure RBAC in EKS?
273. How do you map IAM users and roles to EKS permissions?
274. What is the aws-auth ConfigMap?
275. What is an OIDC provider in AWS?
276. How do EKS Pods securely access S3 and Lambda?
277. How do you retrieve AWS Secrets Manager secrets inside EKS?
278. How do you configure the AWS VPC CNI plugin?
279. What is the difference between Calico and AWS VPC CNI?
280. How do you assign IP addresses to EKS Pods?
281. How does an external request reach an application Pod in EKS?
282. How do you configure cluster autoscaling in EKS?
283. What is the difference between Cluster Autoscaler and Karpenter?
284. How do you upgrade an EKS cluster without downtime?
285. How do you update EKS worker nodes?
286. How do you deploy applications to EKS using GitHub Actions?
287. How do you deploy MongoDB in an EKS cluster?
288. How do you configure shared storage across multiple EKS worker nodes?
289. What are the limitations of using EBS volumes in EKS?
290. How do you manage multiple EKS clusters?

### AWS Auto Scaling and High Availability

291. What is Amazon EC2 Auto Scaling?
292. What are the different EC2 Auto Scaling policies?
293. What is an Auto Scaling Group (ASG)?
294. How do you configure an Auto Scaling Group?
295. What is the difference between horizontal scaling and vertical scaling?
296. What is target tracking scaling?
297. How do you configure scaling based on CPU utilization?
298. How do you automatically launch an EC2 instance when CPU utilization reaches 80%?
299. How do you configure scheduled scaling for traffic that increases daily between 5 PM and 8 PM?
300. How do you ensure EC2 instances scale before predictable traffic spikes?
301. What is an Auto Scaling launch template?
302. How do you update an ASG when a new AMI version is released?
303. How do you verify that EC2 instances launched from a new launch template are healthy?
304. How do you reduce delays when Auto Scaling launches new EC2 instances?
305. How do you automatically replace unhealthy EC2 instances?
306. How do you deploy EC2 application updates with minimum downtime?
307. How do you maintain high availability across multiple Availability Zones?
308. How do you monitor Auto Scaling Groups?
309. How do you troubleshoot Auto Scaling instances that repeatedly terminate during provisioning?
310. How do you optimize costs while maintaining application availability?

### Amazon CloudWatch, CloudTrail and Monitoring

311. What is Amazon CloudWatch?
312. What are the different types of CloudWatch metrics?
313. What is the purpose of the CloudWatch Agent?
314. How do you monitor EC2 CPU and memory utilization using CloudWatch?
315. How do you configure CloudWatch alarms?
316. Where do you create CloudWatch alarms?
317. How do you configure an alert when CPU utilization exceeds a threshold?
318. How do you send CloudWatch alerts to users?
319. How do you collect EC2 application logs using CloudWatch?
320. How do you filter a particular IP address from CloudWatch Logs?
321. How do you troubleshoot an application performance issue using CloudWatch?
322. How do you monitor AWS load balancer performance?
323. What is AWS CloudTrail?
324. What is the difference between AWS Config and AWS CloudTrail?
325. How do you track infrastructure changes in AWS?
326. How do you monitor unauthorized access to AWS resources?
327. What metrics do you use to monitor application availability?
328. How do you configure automated remediation based on monitoring alerts?
329. How do you automate log collection, CPU checks and alarms in CloudWatch?
330. How do you identify the root cause of an application outage using AWS monitoring services?

### AWS Secrets Manager, KMS and Security

331. What is AWS Secrets Manager?
332. What is the difference between AWS Secrets Manager and Systems Manager Parameter Store?
333. What is AWS Key Management Service (KMS)?
334. How do you encrypt secrets using AWS KMS?
335. How do you configure automatic secret rotation?
336. How do you retrieve secrets from AWS Secrets Manager using Python?
337. How do you securely pass AWS Secrets Manager values into a CI/CD pipeline?
338. How can an EKS application access AWS Secrets Manager?
339. How do you secure credentials in AWS infrastructure?
340. How do you manage SSL/TLS certificates in AWS Certificate Manager?
341. What is AWS WAF?
342. How do you protect an AWS web application against common OWASP attacks?
343. Does enabling Security Groups, WAF and DDoS protection prevent bot attacks?
344. How do you configure AWS WAF rules?
345. How do you implement security best practices in a three-tier AWS architecture?

### AWS Backup, Disaster Recovery and Migration

346. How do you back up AWS services?
347. What is AWS Backup?
348. Can AWS backups be automated using shell scripting?
349. Where do you store AWS backup logs?
350. How do you back up an entire Kubernetes cluster running on AWS?
351. How do you configure scheduled EBS snapshots?
352. How do you restore EC2 instances from backups?
353. How do you restore an EBS volume and attach it to another instance?
354. How do you implement disaster recovery across AWS Regions?
355. How do you protect application data during a complete AWS Region outage?
356. How do you design a disaster recovery strategy using RTO and RPO?
357. How do you implement automatic failover in AWS?
358. How do you migrate an application from on-premises infrastructure to AWS?
359. What challenges have you faced during on-premises-to-AWS migrations?
360. How do you migrate an EC2 instance between AWS Regions?
361. How do you migrate an application from a public subnet to a private subnet?
362. How do you handle data synchronization during database migration?
363. How do you ensure availability during AWS infrastructure migration?
364. What prechecks do you perform before migrating production infrastructure?
365. How do you validate and roll back an unsuccessful cloud migration?

### AWS Cost Optimization and Billing

366. How do you perform AWS cost optimization?
367. What AWS cost optimization techniques have you implemented?
368. How do you investigate a sudden increase in AWS billing?
369. How do you identify the services responsible for increased AWS costs?
370. How do you reduce AWS infrastructure costs by 20%?
371. How do you optimize EC2 instance costs?
372. How do you reduce unused EBS volume costs?
373. How do you use S3 lifecycle policies to reduce storage costs?
374. How do you optimize RDS costs?
375. How do you optimize ECS costs?
376. How do you optimize Amazon ElastiCache costs?
377. How do you optimize EKS cluster costs?
378. How do you optimize AWS costs for machine learning workloads?
379. How do you control unauthorized provisioning of expensive AWS resources?
380. How do you balance cost optimization with high availability and reliability?

### AWS Infrastructure Automation and CI/CD

381. How do you provision AWS infrastructure using Terraform?
382. How do you create a VPC, subnet, EC2 instance and S3 bucket using Terraform?
383. How do you deploy EC2 instances across three AWS Regions using Terraform?
384. How do you create reusable Terraform modules for AWS resources?
385. How do you create an EKS Terraform module?
386. How do you configure Terraform remote state storage in S3?
387. How do you enable Terraform state locking using S3?
388. How do you import existing AWS resources into Terraform?
389. How do you detect Terraform drift caused by manual AWS Console changes?
390. How do you prevent manual changes to Terraform-managed AWS resources?
391. What is the difference between AWS CloudFormation and Terraform?
392. What AWS service can be used instead of Jenkins for CI/CD?
393. What is AWS CodePipeline?
394. How is AWS CodePipeline different from Jenkins?
395. What is AWS CodeBuild?
396. What is AWS CodeDeploy?
397. What is AWS CodeCommit?
398. How do CodePipeline, CodeBuild, CodeDeploy, CloudFormation and CloudWatch work together?
399. How do you securely connect a CI/CD pipeline to AWS?
400. How do you automate application deployment to AWS using Jenkins?
401. How do you deploy a Python application to AWS through a Jenkins pipeline?
402. How do you troubleshoot a slow AWS Jenkins deployment pipeline?
403. How do you use AWS CodeBuild build specifications?
404. What AWS CDK commands have you used?
405. How do you integrate Terraform and Ansible to automate AWS deployments?

### AWS Scenario-Based Troubleshooting

406. An application hosted on EC2 suddenly becomes unavailable even though the instance is running. How do you troubleshoot?
407. An EC2 instance is running, but SSH access fails. What would you check?
408. An EC2 instance in a private subnet cannot access the internet. How would you investigate?
409. A user has an IAM role allowing S3 access but still receives AccessDenied. How do you troubleshoot?
410. A public EC2 application needs to move to a private subnet without downtime. What approach would you follow?
411. An ALB health check is failing, but the application is running. How would you troubleshoot?
412. An application behind a load balancer returns HTTP 503. What steps would you take?
413. An application behind a load balancer returns HTTP 504 Gateway Timeout. How would you investigate?
414. A load balancer is configured but cannot accept HTTPS connections. What would you check?
415. An RDS-backed application becomes slow. How would you identify whether the database is responsible?
416. An RDS database experiences high memory utilization, and instance resizing is not allowed. What immediate actions would you take?
417. A Lambda function intermittently fails to connect to an external payment API. How would you investigate?
418. An Auto Scaling Group launches instances, but the instances terminate repeatedly. How would you troubleshoot?
419. A developer updates an AMI in an Auto Scaling launch template. How do you ensure the new configuration is deployed?
420. An AWS bill suddenly increases threefold. How would you identify and resolve the cause?
421. An S3 bucket is accessible from one AWS account but not another. How would you troubleshoot?
422. Two VPCs need to communicate, but their CIDR ranges overlap. What solution would you propose?
423. An EC2 instance requires access to S3 without internet connectivity. How would you configure it?
424. A production resource is manually deleted even though it is managed by Terraform. How would you recover it?
425. An entire AWS Region fails. How would you restore application availability while minimizing data loss?
426. An application deployed on EKS is inaccessible even though the Pods are healthy. How would you troubleshoot?
427. An EKS worker node becomes NotReady during production traffic. How would you handle it?
428. A backend Pod needs secure access to S3 and Lambda. How would you configure permissions and connectivity?
429. A developer requires an EC2 instance for application testing. How would you provision the right instance securely and economically?
430. You need to provision 200 EC2 instances across ten AWS accounts. How would you manage connectivity, security and automation?

## Terraform Interview Questions

### Terraform Basics and Architecture

1. What is Terraform, and how does it work?
2. Why is Terraform used in DevOps projects?
3. Explain Terraform architecture.
4. How have you used Terraform in your current project?
5. What kind of infrastructure have you provisioned using Terraform?
6. What are the advantages of Terraform?
7. What is Infrastructure as Code (IaC)?
8. What is the difference between Terraform and AWS CloudFormation?
9. What is the difference between Terraform and Ansible?
10. What is the difference between Terraform and Terragrunt?
11. Can the same Terraform code be used for different cloud providers?
12. What is a Terraform provider?
13. What is the difference between a provider and a provisioner?
14. What is a Terraform resource?
15. What is a Terraform data source?
16. What is the difference between a resource block and a data block?
17. Explain a real-time scenario where resource blocks and data blocks are used together.
18. What is a Terraform module?
19. What is a Terraform workspace?
20. What is the Terraform lifecycle?
21. What are Terraform meta-arguments?
22. What are Terraform provisioners?
23. What is a null resource in Terraform?
24. What is Terraform Cloud?
25. What are Terraform best practices?
26. Explain the Terraform folder structure.
27. What files are typically present in a Terraform project?
28. What do you define in provider.tf, main.tf, and variables.tf?
29. How do you write a basic Terraform configuration?
30. Where do you execute Terraform code: locally, on a server, or through a CI/CD pipeline?

### Terraform Commands

31. What happens internally when you run terraform init?
32. What is the purpose of terraform plan?
33. What happens when you execute terraform apply?
34. What is the difference between terraform plan and terraform apply?
35. What is the purpose of terraform refresh?
36. What is the difference between terraform refresh and terraform plan?
37. What is the difference between terraform destroy and terraform refresh?
38. What is the purpose of terraform fmt?
39. How do you validate a Terraform configuration?
40. What is the purpose of terraform import?
41. What is the Terraform command for automatic approval?
42. What is the purpose of terraform taint?
43. In which real-world scenario would you use Terraform tainting?
44. What is the Terraform command to unlock a state file?
45. How do you enable debug logs in Terraform?
46. What is the purpose of Terraform external commands?
47. What commands do you use regularly in Terraform?
48. What happens when you run terraform apply without running terraform plan first?
49. What do the + and - symbols represent in Terraform plan output?
50. How do you prevent someone from executing terraform destroy?
51. Have you used terraform destroy in a real project, and in what scenario?
52. How do you investigate a Terraform apply operation that takes too much time?
53. What happens when Terraform apply fails midway?
54. How do you roll back changes when a Terraform deployment fails?
55. What steps do you follow when Terraform reports provisioning errors?

### Terraform State Management

56. What is a Terraform state file?
57. What is the purpose of terraform.tfstate?
58. Where do you store Terraform state files in your organization?
59. What is the difference between local and remote state?
60. Why should Terraform state be stored remotely?
61. How do you configure a remote backend in Terraform?
62. Which configuration block specifies the Terraform backend?
63. How do you store a Terraform state file in Amazon S3?
64. How do you migrate Terraform state from a local backend to S3?
65. Can an existing Terraform state file be moved to remote storage?
66. What is Terraform state locking?
67. Why is state locking required?
68. How do you implement Terraform state locking?
69. How do you handle Terraform state conflicts when multiple engineers run deployments?
70. How do you manually unlock a Terraform state file?
71. What do you do when a Terraform state file remains locked?
72. What happens when the Terraform state file is accidentally deleted?
73. How do you recover a deleted state file when a backup exists?
74. How do you recover a deleted state file without a backup?
75. How do you recover Terraform state without recreating existing cloud resources?
76. What happens if a Terraform state file becomes corrupted?
77. How do you prevent state file corruption?
78. How do you securely manage Terraform state files across multiple teams?
79. How do you isolate state files for development, testing, and production?
80. What is the difference between a Terraform state file and a lock file?
81. What is .terraform.lock.hcl?
82. What information is stored in a Terraform state file?
83. How do you protect sensitive information in Terraform state?
84. How do you prevent unauthorized modification of a remote state file?
85. What happens if both local and remote state files exist?
86. What happens when one of two Terraform-managed EC2 instances is manually deleted?
87. How does Terraform identify resources that are missing from the actual infrastructure?
88. What happens if Terraform starts recreating all resources unexpectedly?
89. How do you manage hundreds of Terraform state files without downtime?
90. How do you ensure that different teams deploying to the same AWS account do not overwrite each other's resources?

### Terraform Drift and Resource Import

91. What is Terraform drift?
92. How do you detect Terraform drift?
93. What happens when a Terraform-managed resource is modified manually through the AWS Console?
94. If an engineer modifies an EC2 instance through the AWS Console, what will terraform plan show?
95. What happens if you run terraform apply after someone manually modifies a resource?
96. How do you prevent manual infrastructure changes from causing Terraform drift?
97. How do you automate Terraform drift detection?
98. How do you handle a resource that was manually deleted outside Terraform?
99. How do you identify and recover a Terraform-managed resource deleted manually?
100. How do you import an existing AWS resource into Terraform?
101. How do you import multiple manually created EC2 instances into Terraform?
102. How do you bring manually created production infrastructure under Terraform management?
103. How do you import existing Azure or Google Cloud resources into Terraform?
104. What is the difference between Terraform import and a data source?
105. How do you export existing Azure resources into Terraform code?
106. How do you remove a resource from Terraform management without deleting it?
107. How do you prevent certain resources from being destroyed when their Terraform configuration is removed?
108. How would you manage resources manually while excluding them from Terraform's lifecycle?
109. What happens if an S3 bucket created by Terraform is renamed?
110. How do you prevent Terraform from destroying and recreating resources when their configurations change?

### Terraform Variables and Data Types

111. What are Terraform variables?
112. What is the difference between input variables and local values?
113. What are the different Terraform variable types?
114. What is the difference between a list and a string in Terraform?
115. What is the difference between a list and a set in Terraform?
116. What is a tuple in Terraform?
117. What is the difference between a map and an object in Terraform?
118. How do you define and use a map of objects in Terraform?
119. How do you pass variables into Terraform modules?
120. How do you manage Terraform variables for different environments?
121. How do you pass Terraform variables through a CI/CD pipeline?
122. How do you securely handle passwords and secrets in Terraform?
123. Where does Terraform store dynamically generated passwords?
124. Can you retrieve an RDS password generated through Terraform if it was not saved separately?
125. How do you use Terraform locals to reduce repeated configuration?

### Terraform Modules and Workspaces

126. How do you create a custom Terraform module?
127. What files should be included inside a Terraform module?
128. How do you call a child module from the root module?
129. What is the difference between root modules and child modules?
130. What are the different types of Terraform modules?
131. How do you reuse Terraform modules across projects?
132. How do you upgrade an existing Terraform module?
133. How do you manage versioning of Terraform modules?
134. How do you write a Terraform module for an EKS cluster?
135. How do you design a reusable module for VPC and EC2 resources?
136. How do you organize shared modules across development, QA, and production?
137. How would you refactor repeated Terraform configurations using modules?
138. How do you refactor a legacy Terraform codebase to follow DRY principles?
139. What are the benefits of Terraform workspaces?
140. How do you create and manage different Terraform workspaces?
141. How do you manage development, testing, and production environments using Terraform?
142. What is the difference between Terraform workspaces and separate environment configurations?
143. How do you ensure infrastructure is deployed in one environment but not another?
144. How do you manage Terraform configurations across multiple teams?
145. How would you refactor a large Terraform monorepo into reusable modules without downtime?

### Terraform Meta-Arguments and Lifecycle

146. What is the difference between count and for_each?
147. When should you use count instead of for_each?
148. Explain for_each with a real-time example.
149. What are the Terraform meta-arguments apart from count and for_each?
150. What is the purpose of depends_on?
151. How does Terraform automatically determine resource dependencies?
152. How do you handle explicit dependencies between Terraform resources?
153. Explain how Terraform builds its dependency graph during planning.
154. How can circular dependencies occur in Terraform?
155. What is the purpose of the lifecycle block?
156. What are the different Terraform lifecycle arguments?
157. How does create_before_destroy work?
158. What is the purpose of prevent_destroy?
159. What is the purpose of ignore_changes?
160. How do you prevent Terraform from deleting a resource during an update?
161. How do you modify a Terraform-managed resource without downtime?
162. How do you handle configuration changes that require resource replacement?
163. What are Terraform provisioners, and what are their use cases?
164. What is the difference between local and remote provisioners?
165. What is the difference between EC2 user data and a remote provisioner?
166. What happens if a resource fails during provisioner execution?
167. How do you use a null resource in Terraform?
168. How do you create 50 EC2 instances using a single Terraform configuration?
169. How do you create multiple EC2 instances with different instance types, AMIs, tags, and volumes?
170. How do you create multiple resources from a list of instance names?

### Terraform Coding and Scenario-Based Questions

171. Write Terraform code to provision an EC2 instance.
172. Write Terraform code to provision an EC2 instance using variables for instance type and region.
173. Write Terraform code to create an EC2 instance with an attached EBS volume.
174. Write Terraform code to create an EC2 instance with a custom security group and user data.
175. Write Terraform code to create an EC2 instance that permits only SSH access.
176. Write Terraform code to provision an EC2 instance in a public subnet.
177. Write Terraform code to create an EC2 instance in multiple AWS regions.
178. How would you structure Terraform code to deploy EC2 instances in three different regions?
179. If multiple AWS provider regions are configured, which region will Terraform use to provision an EC2 instance?
180. How do you implement multi-region Terraform deployments?
181. How do you manage cross-region deployments in a multi-cloud setup?
182. Write Terraform code to create an AWS VPC, subnet, EC2 instance, and S3 bucket.
183. Write Terraform code to create a VPC with public and private subnets.
184. Write Terraform code to create multiple S3 buckets.
185. Write Terraform code to create an S3 bucket.
186. How do you configure an S3 lifecycle policy to delete objects automatically after seven days?
187. Write Terraform code to configure an S3 bucket with a 30-day expiration policy.
188. How do you create an AWS EventBridge resource using Terraform?
189. Write Terraform code to provision AWS Lambda.
190. Write Terraform code to provision an Azure App Service.
191. Write Terraform code to provision an Azure Virtual Network and multiple subnets.
192. Write Terraform code to create Azure network interfaces and attach them to virtual machines.
193. Write Terraform code to provision Azure virtual machines with username and password authentication.
194. Write Terraform code to configure an Azure Bastion Host.
195. Write Terraform code to configure an Azure Load Balancer.
196. Write Terraform code to deploy two Azure Storage Accounts in development and five in production.
197. How do you use Terraform locals to create multiple /21 subnets from a 10.0.0.0/16 network?
198. How do you use Terraform to provision an EKS cluster and its worker nodes?
199. How do you manage Terraform configurations for VPC and EKS resources?
200. How do you use Terraform to provision infrastructure for machine learning workloads?
201. How do you ensure a specific AMI exists in an AWS account using Terraform?
202. How do you configure infrastructure to launch an EC2 instance when CPU utilization reaches 80% and copy an image from S3?
203. How do you create multiple AWS resources using a reusable Terraform module?
204. How do you manage infrastructure resource dependencies while provisioning several cloud services?
205. Describe a Terraform configuration you wrote to provision highly scalable infrastructure.

### Terraform CI/CD Integration

206. How do you integrate Terraform into a CI/CD pipeline?
207. How do you execute Terraform through Jenkins?
208. How do you use Terraform in an Azure DevOps pipeline?
209. How do you manage Terraform variables across CI/CD environments?
210. What are the stages of a Terraform deployment pipeline?
211. How do you connect a Terraform pipeline securely to AWS?
212. How do you automate Terraform infrastructure provisioning through CI/CD?
213. How do you handle Terraform approval before applying production changes?
214. What security checks do you implement before running terraform apply?
215. How do you prevent a Terraform security group change that opens access to 0.0.0.0/0?
216. How do you implement policy-as-code with Terraform using Sentinel or OPA?
217. How do you enforce Azure policies such as mandatory tags and location restrictions using Terraform?
218. How do you reduce the execution time of a Terraform pipeline taking more than 25 minutes?
219. How do you implement Blue-Green deployments with rollback using Terraform and Azure pipelines?
220. How do you safely manage Terraform credentials in GitHub Actions?
221. What is the difference between GitHub OIDC authentication and Terraform Cloud authentication?
222. How do you prevent concurrent Terraform deployments from causing conflicts?
223. How do you validate Terraform code before it reaches production?
224. How do you prevent unauthorized infrastructure changes through Terraform pipelines?
225. What guardrails would you implement after a production failure caused by terraform apply?

### Advanced Terraform and Production Troubleshooting

226. What Terraform production issues have you faced, and how did you resolve them?
227. Describe a Terraform deployment failure you encountered in a real project.
228. What unexpected infrastructure changes have you observed during terraform apply?
229. How do you troubleshoot Terraform resource creation failures?
230. How do you troubleshoot Terraform errors caused by resource dependencies?
231. How do you investigate unexpected changes shown in a Terraform plan?
232. How do you prevent resource downtime caused by Terraform configuration updates?
233. How do you handle a Terraform resource that was provisioned partially before a failure?
234. How do you manage changes to Terraform modules without destroying existing resources?
235. How do you prevent accidental production infrastructure deletion?
236. How do you recover infrastructure after someone deletes resources directly from Azure?
237. How do you design disaster recovery for Terraform-managed infrastructure?
238. How do you manage Terraform state during disaster recovery?
239. How do you maintain availability during Terraform infrastructure changes?
240. How do you safely deploy changes when several engineers are modifying the same Terraform repository?
241. How do you manage Terraform for large-scale infrastructure across multiple AWS accounts?
242. How do you automate Terraform deployments across multiple cloud regions?
243. How do you structure Terraform repositories for multiple application teams?
244. How do you manage infrastructure changes while preventing configuration drift?
245. How do you ensure Terraform changes comply with security and governance policies?
246. How do you perform infrastructure automation and optimization using Terraform and Ansible?
247. How do you troubleshoot Terraform when the resource configuration and actual cloud infrastructure do not match?
248. How do you recover when Terraform cannot find previously managed resources?
249. How do you handle a Terraform-managed load balancer that needs to be removed after several configuration updates?
250. How would you design a secure, reusable, scalable Terraform setup for an enterprise production environment?

## Azure Cloud Infrastructure Engineer – Interview Questions

### Azure Cloud Fundamentals

1. What are the basic Azure services you have worked with?
2. What are the different types of Azure subscriptions?
3. What is the difference between IaaS and SaaS deployment models?
4. How would you design and deploy an end-to-end infrastructure solution using Azure services?
5. What are the different Azure services required for a three-tier application?
6. How do you design a highly scalable infrastructure in Azure?
7. What Azure infrastructure changes have you implemented in your projects?
8. What types of infrastructure optimizations have you performed?
9. How would you design a cost-optimized architecture for an internal reporting application that runs nightly and stores logs for three years?
10. What are the differences between scaling strategies for compute-intensive and I/O-intensive workloads in Azure?

### Azure Networking and Connectivity

11. What is an Azure Virtual Network (VNet)?
12. How do you create an Azure VNet with a specified IP address range?
13. How do you create separate subnets for bastion, application, and database layers?
14. What is Hub-and-Spoke network topology in Azure?
15. What are the different types of VNet peering?
16. Can two Azure virtual machines in different VNets communicate with each other?
17. How do you establish connectivity between two Azure subscriptions?
18. What is an Azure Private Endpoint?
19. What is the difference between an Azure Network Security Group (NSG) and Azure Firewall?
20. What is ExpressRoute in Azure?
21. What is the difference between Site-to-Site VPN and ExpressRoute?
22. In which scenarios would you choose Site-to-Site VPN over ExpressRoute?
23. How would you justify choosing ExpressRoute over a Site-to-Site VPN for a banking customer with budget constraints?
24. How do you configure connectivity between an on-premises network and applications deployed in Azure?
25. How would an on-premises user securely access applications hosted in Azure?
26. How do you provide restricted access to a database running in a private subnet?
27. How would you establish private connectivity between MySQL and Azure Key Vault without exposing traffic to the public internet?
28. How do you troubleshoot intermittent DNS resolution issues during an Azure deployment?
29. How do you use an Azure Bastion Host to securely access virtual machines?
30. How do you configure Azure network interfaces and attach them to virtual machines?

### Azure Load Balancing, Application Gateway and Front Door

31. How do you configure an Azure Load Balancer to route traffic to application virtual machines?
32. What is the difference between Azure Application Gateway and Azure Front Door?
33. Is Azure Application Gateway a Layer 4 or Layer 7 load balancer?
34. How would you use Azure Application Gateway with Web Application Firewall (WAF) for a banking application?
35. What is the difference between Azure Firewall and NSG?
36. How do you troubleshoot an Application Gateway returning HTTP 404 when the backend services are healthy?
37. How do you troubleshoot an application returning HTTP 403 for external users while internal users can access it?
38. How do you block a particular domain using Azure Application Gateway?
39. Where do you store TLS/SSL certificates used by Azure Application Gateway?
40. How do you renew an expired TLS certificate on Azure Application Gateway?
41. How do you rotate certificates in Azure Key Vault and configure a .pfx certificate with Application Gateway?
42. How would you troubleshoot HTTP 502 errors affecting only some users during a Canary deployment?
43. In terms of cost optimization, how would you compare Azure Application Gateway with other networking gateway options?

### Azure Virtual Machines and Compute

44. How do you provision an Azure virtual machine using a network interface?
45. How do you create an Azure virtual machine using username and password authentication?
46. How do you create public IP addresses for Azure virtual machines?
47. How do you monitor CPU and memory usage on an Azure VM?
48. How do you configure an alert when Azure VM memory utilization exceeds 80%?
49. Which metrics would you monitor when VM CPU or memory utilization exceeds 75%?
50. What is Azure App Service?
51. How would you deploy an application to Azure App Service using Infrastructure as Code?
52. Should an Azure App Service be registered or provisioned before deploying the application?
53. How would you connect Azure App Service, Azure Functions, and Azure Service Bus in a three-tier .NET application?
54. How do you troubleshoot an Azure Function that is being throttled?
55. How do you scale Azure compute resources for high workloads?
56. How do you investigate performance degradation in an Azure-hosted application?

### Azure Storage Infrastructure

57. What are the different types of Azure Storage?
58. Which Azure Storage services have you used in your projects?
59. What are the different Azure Blob Storage access tiers?
60. Can an Azure Storage Account be changed from the Hot tier to the Cool tier?
61. How do you move large amounts of data from Hot storage to Cool storage efficiently?
62. How would you optimize Azure Storage costs when data is continuously increasing?
63. How do you design cost optimization for a storage account holding continuous 24/7 recording data?
64. How do you provide secure access to data stored in Azure Storage?
65. How do you restrict Azure Storage Account access to a specific application or Pod?
66. How do you automate file transfers from Azure Blob Storage to Google Cloud Storage?
67. How would you create multiple Azure Storage Accounts using Terraform based on the environment?
68. Why might a Terraform backend configuration exist in the repository while the state file is not visible in the Azure Storage Account?

### Azure Identity, Security and Key Vault

69. What is the difference between Azure Managed Identity and a Service Principal?
70. What is a Service Principal in Azure? Explain with an example.
71. How do you create an App Registration in Azure?
72. How do you integrate Azure Key Vault with an application?
73. How do you use Azure Key Vault secrets in AKS?
74. How does an application Pod retrieve the latest rotated secrets from Azure Key Vault?
75. How do you configure Azure Key Vault access policies?
76. How do you integrate Azure Key Vault into an Azure DevOps pipeline?
77. How is authentication and networking established between Azure DevOps and Azure Key Vault?
78. How do you implement dynamic secret rotation in Azure DevOps pipelines?
79. How do you integrate Microsoft Entra ID with AKS for authentication?
80. How do you securely configure workload identity federation between GitHub Actions and Azure?
81. How would you secure infrastructure across both AWS and Azure using identity federation?
82. How would you enforce Azure Policies for mandatory tags and location restrictions using Terraform?
83. How do you provide access to only one AKS Pod or application for a Storage Account while restricting all other Pods?

### Azure Kubernetes Service (AKS) Infrastructure

84. How do you secure an AKS cluster?
85. How do you make an AKS cluster highly available?
86. How do you optimize AKS infrastructure costs?
87. What networking configuration have you used in AKS?
88. How do you protect application endpoints exposed through AKS?
89. How do you integrate Azure Key Vault with AKS?
90. How do you integrate Azure Application Gateway and an Ingress Controller with AKS?
91. How do you troubleshoot a failed Pod in AKS, and which commands would you use?
92. How do you investigate random health-check failures in an application deployed to AKS?
93. How do you investigate periodic application latency in AKS when no code changes have occurred?
94. How do you troubleshoot AKS issues using Ingress, application and infrastructure logs?
95. How do you monitor AKS Pods and receive alerts when they go down?
96. How do you use Helm charts for AKS deployments?
97. How do you build container images, push them to Azure Container Registry, and deploy them to AKS?
98. How do you configure resource requests and limits for Pods in AKS?
99. Can a Pod requesting 500 MB memory with a 4 GiB limit be scheduled in an AKS cluster with 32 GB total memory and 30 GB already utilized?
100. How do you implement secure Pod-to-Pod communication across Kubernetes namespaces?

### Azure Monitoring, Alerts and Log Analytics

101. What is Azure Monitor?
102. How do you configure alerts in Azure Monitor?
103. What are the steps involved in creating a CPU or memory alert for an Azure VM?
104. What is an Azure Monitor Action Group?
105. How do you configure an Action Group for infrastructure alerts?
106. What is the purpose of a Log Analytics Workspace?
107. How do you configure monitoring using Azure Log Analytics?
108. What KQL queries have you used for infrastructure troubleshooting?
109. How do you use KQL to troubleshoot Azure VM performance?
110. How do you prioritize critical alerts when an Azure environment generates hundreds of alerts?
111. How do you configure alert thresholds across a large production infrastructure?
112. What monitoring tools have you used for Azure infrastructure?
113. What automation have you implemented for infrastructure monitoring?
114. How do you troubleshoot incomplete logs across AKS, Ingress, applications and infrastructure?
115. How do you monitor end-to-end SLAs for a distributed application hosted in Azure?
116. How would you design a highly available logging system for more than 100 microservices across three regions?

### Azure Backup, Disaster Recovery and Migration

117. What is Azure Recovery Services Vault?
118. How do you use Azure Recovery Services Vault for backups?
119. What Azure services have you used for backup?
120. What backup strategies have you implemented in Azure?
121. What challenges have you faced while taking infrastructure backups?
122. How do you design a disaster recovery strategy in Azure?
123. Which Azure services would you use to implement a DR solution?
124. What DR scenarios have you configured in your projects?
125. How do you ensure data is protected during a complete regional outage?
126. How do you protect data if both AWS and Azure experience outages?
127. How do you implement disaster recovery for stateful containerized applications?
128. How do you perform zero-downtime database migrations?
129. Have you migrated infrastructure from on-premises to Azure?
130. How do you plan and execute an Azure-to-Azure resource migration?
131. What is the difference between resource migration and data migration?
132. What prechecks do you perform before migrating infrastructure to Azure?
133. What production issues have you encountered during cloud migrations, and how did you resolve them?

### Azure Infrastructure as Code and Automation

134. How do you provision Azure infrastructure using Terraform?
135. How do you write Terraform code to create an Azure VNet and multiple subnets?
136. How do you provision Azure VMs and networking resources using Terraform?
137. How do you configure Azure Bastion and Load Balancer resources using Terraform?
138. How do you write Terraform code to deploy Azure App Service?
139. How do you create two Azure Storage Accounts in development and five in production using Terraform?
140. How do you use a Terraform dynamic block when creating Azure resources?
141. How do you pass a subnet ID output from a VNet Terraform module to a VM module?
142. How do you manage Terraform state for Azure infrastructure?
143. What happens to Terraform state if someone deletes resources manually from Azure?
144. How do you export existing Azure resources into Terraform code?
145. How do you prevent Terraform from destroying and recreating existing Azure resources during configuration changes?
146. How do you manage Terraform across multiple Azure environments?
147. How do you implement Azure Policies at scale through Terraform?
148. How do you automate Azure infrastructure deployments through CI/CD pipelines?
149. How would you implement Blue-Green deployment with rollback on Azure using Terraform and pipelines?
150. How do you manage AWS and Azure infrastructure using a common DevOps process while maintaining security and cost efficiency?

### Azure DevOps Infrastructure Operations

151. What are the different types of Azure DevOps pipelines?
152. What is the difference between YAML pipelines and Classic pipelines in Azure DevOps?
153. How do you create an Azure DevOps pipeline?
154. How do you pass variables and parameters into Azure pipelines?
155. What is a Variable Group in Azure DevOps?
156. How do you store credentials and secrets securely in Azure pipelines?
157. What is an Azure DevOps Service Connection?
158. What is the difference between Microsoft-hosted and self-hosted agents?
159. What are Deployment Groups in Azure DevOps?
160. How do you execute multiple pipeline jobs in parallel?
161. Where are Azure pipeline logs stored?
162. What is Azure Artifacts, and how is it used?
163. Where do you store build artifacts generated by Azure pipelines?
164. How do you integrate SonarQube and Snyk into Azure DevOps?
165. How do you integrate Azure Key Vault into a CI/CD pipeline?
166. How do you assess Azure DevOps migration readiness?
167. How do you apply repository security policies programmatically using the Azure DevOps REST API?
168. What is the difference between Stakeholder and Administrator access in Azure DevOps?
169. What is Azure Boards?
170. How do you troubleshoot and optimize a slow infrastructure deployment pipeline?

## Git / GitHub / GitLab / Azure Repos Interview Questions

### Git Basics and Commands

1. What is Git, and why is it used?
2. What Git commands do you use in your daily activities?
3. What is the difference between git pull and git fetch?
4. What happens internally when you run git fetch and git pull?
5. What is the difference between git pull and git clone?
6. What is the purpose of git clone?
7. How do you clone a repository from a remote Git server?
8. What is the difference between a local repository and a remote repository?
9. How do you clone a repository, make changes locally, and push those changes to the remote repository?
10. Explain the Git push and pull commands.
11. What happens internally when you execute git add?
12. How does Git store a file in its internal database after git add?
13. What is git checkout, and how do you use it?
14. What is git stash?
15. What is the purpose of git stash pop?
16. What is Git cherry-pick, and how do you use it?
17. What is the purpose of Git tags?
18. How do you compare two Git commits and identify their differences?
19. How do you extract all Git commits made in the last three days?
20. How do you delete the last two Git commits?
21. How do you remove changes introduced by a particular developer's commit without affecting other developers' changes?
22. What is the purpose of Git submodules?
23. What is the difference between git push --force and git push --force-with-lease?
24. What validations do you perform before or after committing code to GitHub?
25. What is Git squash, and how is it used?

### Git Branching Strategies

26. What are the different types of branches in Git?
27. What Git branching strategy do you follow in your organization?
28. Why did you choose your current branching strategy over other strategies?
29. What are the different Git branching strategies you have worked with?
30. Explain your branching strategy with an example.
31. How does your Git branching strategy support multiple environments, releases, and hotfixes?
32. How do you deploy code from different Git branches to development, QA, staging, and production?
33. How do you integrate Git branching strategies into a CI/CD pipeline?
34. How do you configure CI/CD to deploy code from a feature branch instead of the main branch?
35. How do you ensure that pushing code to a staging branch triggers only the staging deployment?
36. How do you handle merges without breaking the release branch?
37. How do you manage production hotfixes using your branching strategy?
38. What is a stale branch in GitHub?
39. How do you manage source code branching in a large team working on a complex application?
40. How do you design branch-specific pipelines for multiple environments?

### Git Merge, Rebase and Conflict Resolution

41. What is the difference between Git merge and Git rebase?
42. Explain Git merge and Git rebase with an example using main and feature branches.
43. What is Git merge?
44. What is Git rebase?
45. What is a Git merge conflict?
46. How do you resolve merge conflicts in Git?
47. If two developers modify the same file and encounter a merge conflict, how can the conflict be resolved?
48. During merge-conflict resolution, do you check the commit history of the source branch or target branch?
49. How do you handle merge conflicts while protecting the release branch?
50. What is Git cherry-pick, and in which scenarios would you use it?
51. How do you revert a specific developer's changes when ten developers have committed code?
52. If someone force-pushes and causes the main branch history to be lost, how do you recover it?
53. How do you push a recovered Git branch back to the remote repository?

### GitHub and Repository Management

54. How do you migrate a Git repository from GitHub to GitLab while preserving the complete commit history?
55. What steps do you follow when migrating a repository from one Git hosting platform to another?
56. How do you establish connectivity between a new Jenkins installation and a GitHub repository?
57. What configuration steps are required to integrate GitHub with Jenkins?
58. What are the different ways to establish communication between Jenkins and GitHub?
59. Can a newly installed Jenkins server communicate with GitHub directly using webhooks, or are additional configurations required?
60. How do you configure a Jenkins multibranch pipeline for a GitHub repository?
61. What are the advantages of multibranch pipelines?
62. Why would you choose a multibranch pipeline over a normal SCM pipeline?
63. How do you configure a Git checkout stage with repository credentials in Jenkins?
64. How do you structure repositories and pipelines in a large DevOps project?
65. In a monorepo, how do you ensure only the services affected by code changes are built and deployed?
66. How would you use the Azure DevOps REST API to apply security policies to all repositories programmatically?
67. What happens if a developer updates infrastructure code locally but does not push it to the remote repository?
68. If credentials are accidentally committed to Git, what incident response steps would you follow?
69. How do you remove hardcoded passwords accidentally pushed to a repository?
70. Should Terraform variable files containing sensitive information be committed to Git?
71. How do you prevent application secrets from being committed to Git repositories?

### GitHub Actions

72. What is GitHub Actions?
73. Explain the structure of a GitHub Actions workflow file.
74. What are the different steps included in your GitHub Actions workflow?
75. Explain a GitHub Actions pipeline you have implemented.
76. How do you build a CI/CD pipeline using GitHub Actions?
77. How do you configure a GitHub Actions workflow to build, test, and deploy a containerized application to Kubernetes?
78. How do you deploy applications to Amazon EKS using GitHub Actions?
79. How do you write a GitHub Actions workflow to build and push a Docker image?
80. How do you configure jobs to run in parallel in GitHub Actions?
81. If one GitHub Actions job depends on another, which keyword do you use?
82. What is the needs keyword in GitHub Actions?
83. What is the difference between needs and concurrency in GitHub Actions?
84. How do you prevent concurrent executions in GitHub Actions?
85. What is the matrix strategy in GitHub Actions?
86. How does caching work in GitHub Actions?
87. How do you configure manual triggers in GitHub Actions?
88. How do you configure GitHub Actions to run only when code is pushed to a particular branch?
89. How do you prevent a GitHub Actions workflow from running when code is pushed to other branches?
90. How do you trigger a workflow when a pull request is raised?
91. What is runs-on in a GitHub Actions workflow?
92. What types of GitHub Actions runners have you used?
93. How do you configure self-hosted GitHub Actions runners?
94. How do you set up GitHub runners for your application environment?
95. How do you ensure the security of third-party GitHub Marketplace actions?
96. How do you identify errors and missing steps in an existing GitHub Actions workflow?
97. How do you improve a GitHub Actions workflow to make the CI/CD pipeline more robust?
98. How do you securely configure workload identity federation between GitHub Actions and Azure or Google Cloud?
99. What is the difference between GitHub OIDC-based authentication and Terraform Cloud authentication?
100. How do you design a GitHub Actions workflow with pull-request validations and approval requirements?

### GitLab and GitLab CI/CD

101. Have you worked with GitLab?
102. What are GitLab CI/CD rules?
103. How do you use GitLab rules to control pipeline execution?
104. How do you create a GitLab CI/CD pipeline to deploy microservices?
105. How do you configure three microservices to run in parallel in a GitLab pipeline?
106. How do you migrate a repository from GitHub to GitLab while preserving commit history?

### Azure Repos and Azure DevOps Repository Management

107. How do you apply security policies across all Azure DevOps repositories using the REST API?
108. How do you structure repositories and pipelines in a large Azure DevOps project?
109. How do you assess Azure DevOps migration readiness and plan the transition?
110. How do you integrate repository branching strategies with Azure DevOps deployment pipelines?
111. How do you configure branch conditions so that only the intended deployment environment is triggered?
112. How do you integrate pull-request checks and approvals into a CI/CD pipeline?

### Git Integration with CI/CD and GitOps

113. What is the role of Git in a CI/CD pipeline?
114. How do you trigger CI/CD pipelines when code is pushed to a particular branch?
115. How do you use webhooks to trigger pipeline execution?
116. Should a CI/CD pipeline continuously poll the repository for changes?
117. How do you schedule a pipeline against a staging or main branch?
118. How do you configure Jenkins to execute a pipeline after a specified number of commits are pushed?
119. How do you integrate Git branching strategies with application deployment to different Kubernetes clusters?
120. Where do you configure branch-based environment selection in your CI/CD pipeline?
121. How do you implement pull-request checks and approval gates before production deployment?
122. How do you decide whether SonarQube should run on every pull request or every push?
123. How do you configure a CI/CD pipeline to build and deploy only relevant microservices in a monorepo?
124. How do you deploy applications from different branches using your organization's repository strategy?
125. What is GitOps?
126. What is the difference between push-based and pull-based GitOps deployments?
127. How do you implement GitOps using Argo CD or Flux?
128. Why use Argo CD when Git is already the source of truth?
129. Why use Argo CD instead of deploying directly through a CI/CD pipeline using Helm or kubectl?
130. How does a GitOps tool detect configuration drift?
131. How do you manage configuration drift in GitOps deployments?
132. How do you manage secrets securely in GitOps workflows?
133. How do you design GitOps for more than 1,000 Kubernetes clusters with drift detection, hotfixes, and controlled overrides?
134. What is the difference between GitHub Actions and Argo CD?

## CI/CD, Build & Deployment Interview Questions

### CI/CD Fundamentals and Workflow

1. What is CI/CD?
2. Explain the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment.
3. Explain the complete CI/CD pipeline used in your project.
4. What are the different stages of a CI/CD pipeline?
5. Which CI/CD tools have you worked with?
6. Have you created a CI/CD pipeline from scratch?
7. How do you design a CI/CD pipeline for a new application when the developer provides only source code?
8. Explain the complete CI/CD process from code commit to production deployment.
9. How do you configure a CI/CD pipeline for development, QA, UAT, and production environments?
10. How do you onboard a new application from source code to release management?
11. What types of applications have you deployed through CI/CD pipelines?
12. What are the steps involved in building and deploying a Java application?
13. What are the steps involved in building and deploying an Angular application?
14. What are the steps involved in building and deploying a .NET application?
15. How do you deploy microservices using CI/CD?
16. What are the different types of CI/CD pipelines?
17. How do you integrate source code repositories with CI/CD tools?
18. What is the purpose of a webhook in CI/CD?
19. How do you trigger a pipeline automatically after a code commit?
20. What is the difference between a build pipeline and a release pipeline?
21. What are the different deployment environments in your project?
22. How do you manage application configurations across multiple environments?
23. What are environment variables in a CI/CD pipeline?
24. How do you pass environment variables between pipeline stages?
25. How do you manage sensitive information in CI/CD pipelines?
26. How do you perform quality checks during a CI/CD pipeline?
27. What post-deployment checks do you perform after releasing an application?
28. What challenges have you faced while implementing CI/CD?
29. How do you troubleshoot failures in a CI/CD pipeline?
30. How do you design a CI/CD pipeline for a hybrid environment with cloud and on-premises servers?

### Jenkins Fundamentals and Architecture

31. What is Jenkins?
32. Explain Jenkins architecture.
33. What is the difference between a Jenkins controller and an agent?
34. What is a Jenkins agent?
35. What are Jenkins nodes?
36. What are Jenkins executors, and how do they work internally?
37. How do Jenkins controllers communicate with agents?
38. What types of Jenkins agents have you used?
39. How do you configure Jenkins agents?
40. How do you configure a Jenkins agent to run a specific job?
41. How do you configure Jenkins in a high-availability setup?
42. How do you manage multiple Jenkins servers?
43. How do you perform a complete Jenkins backup, including jobs, configurations, and authentication settings?
44. How do you restore Jenkins from a backup?
45. What are the different types of Jenkins jobs?
46. What is the difference between a freestyle job and a pipeline job?
47. What are the different ways to trigger a Jenkins pipeline?
48. How do you configure scheduled Jenkins jobs?
49. How do you configure Jenkins jobs to run on a specific node?
50. How do you provide view-only access to Jenkins jobs for selected users?
51. How do you manage Jenkins permissions?
52. How do you secure a Jenkins server?
53. How do you secure Jenkins pipelines?
54. What Jenkins plugins have you used?
55. How do you troubleshoot Jenkins plugin issues?

### Jenkins Pipeline and Jenkinsfile

56. Write a Jenkins declarative pipeline.
57. Explain the structure of a Jenkinsfile.
58. What is the difference between declarative and scripted pipelines?
59. Have you written Jenkins pipelines using Groovy?
60. What is the purpose of the pipeline block?
61. What is the purpose of the agent block?
62. What is the purpose of the stages block?
63. What is the purpose of the steps block?
64. What is the purpose of the environment block?
65. What is the purpose of the post block?
66. What are post-conditions in a Jenkins pipeline?
67. How do you define variables in Jenkins?
68. How do you access variables in a Jenkinsfile?
69. How do you pass parameters into a Jenkins pipeline?
70. How do you dynamically create pipeline stages based on environment variables?
71. How do you use conditional execution in Jenkins pipelines?
72. Can you execute multiple Jenkins stages in parallel?
73. Can more than two stages run simultaneously?
74. How do you define dependencies between Jenkins pipeline stages?
75. Write a Jenkins pipeline with build, test, scan, and deployment stages.
76. Write a Jenkins pipeline that checks out source code using Git credentials.
77. How do you integrate a Jenkins pipeline with GitHub?
78. How do you configure a new Jenkins installation to communicate with GitHub?
79. Can Jenkins communicate with GitHub through webhooks without additional configuration?
80. How do you configure a Jenkins multibranch pipeline?
81. What are the benefits of a multibranch pipeline?
82. Why would you choose a multibranch pipeline over a normal SCM pipeline?
83. How do you configure branch-based deployments in Jenkins?
84. How do you ensure a staging branch triggers only the staging deployment?
85. How do you trigger one Jenkins pipeline automatically after another pipeline completes?
86. How do you configure Jenkins to execute a pipeline after a specified number of commits?
87. How do you configure manual approvals in a Jenkins pipeline?
88. How do you implement deployment rollback in a Jenkinsfile?
89. What are Jenkins Shared Libraries?
90. How do you create and configure Jenkins Shared Libraries?
91. What is the folder structure of a Jenkins Shared Library?
92. How do you integrate Shared Libraries into Jenkinsfiles?
93. How do you reuse Jenkins pipeline code across multiple projects?
94. How do you manage credentials in a Jenkins pipeline?
95. How do you pass credentials securely to Jenkins build and deployment stages?

### Jenkins Troubleshooting and Optimization

96. A Jenkins pipeline starts, but the build does not execute. What could be the reason?
97. How do you troubleshoot a Jenkins pipeline that is stuck?
98. How do you troubleshoot a failed Jenkins job?
99. How do you identify which pipeline stage caused the failure?
100. A Jenkins pipeline works in development but fails in production. How do you troubleshoot it?
101. What would you do if a Jenkins agent becomes unavailable during a build?
102. How do you troubleshoot Jenkins controller-to-agent connectivity?
103. How do you handle a pipeline that runs successfully but does not deploy the application?
104. How do you troubleshoot missing environment variables in a Jenkins pipeline?
105. How do you troubleshoot Jenkins Git checkout failures?
106. How do you troubleshoot authentication failures between Jenkins and GitHub?
107. How do you troubleshoot a failed Docker image build in Jenkins?
108. How do you troubleshoot a failed Kubernetes deployment from Jenkins?
109. How do you reduce Jenkins pipeline execution time from one hour to 20 minutes?
110. How do you optimize a CI/CD pipeline that currently takes 45 minutes?
111. How do you optimize parallel build and test execution in Jenkins?
112. How do you reduce build time using caching?
113. What steps would you take if Jenkins builds are running slowly?
114. How do you identify resource bottlenecks in Jenkins?
115. How do you troubleshoot a Jenkins pipeline that fails intermittently?

### Azure DevOps and Azure Pipelines

116. What is Azure DevOps?
117. What are the different services available in Azure DevOps?
118. What is Azure Pipelines?
119. What are the different types of pipelines in Azure DevOps?
120. What is the difference between Classic and YAML pipelines?
121. Explain the structure of an Azure Pipelines YAML file.
122. Write an Azure Pipelines YAML file from scratch.
123. How do you create a CI/CD pipeline in Azure DevOps?
124. How do you configure a multistage Azure pipeline?
125. How do you configure CI and CD in a single Azure pipeline?
126. How do you configure a build pipeline for an application in Azure DevOps?
127. How do you configure a release pipeline in Azure DevOps?
128. How do you deploy an application from development to UAT using an Azure YAML pipeline?
129. How do you configure pipelines for development, QA, UAT, and production?
130. How do you configure triggers in Azure Pipelines?
131. How do you trigger different environments based on Git branches?
132. How do you configure approvals in Azure DevOps pipelines?
133. How do you configure deployment gates in Azure Pipelines?
134. What are Deployment Groups in Azure DevOps?
135. What is an Azure DevOps agent?
136. What is the difference between Microsoft-hosted and self-hosted agents?
137. How do you configure a self-hosted Azure DevOps agent?
138. How do you execute multiple jobs in parallel in Azure Pipelines?
139. How do you pass variables in Azure Pipelines?
140. How do you parameterize an Azure YAML pipeline?
141. What is a Variable Group in Azure DevOps?
142. How do you securely store credentials in Azure Pipelines?
143. How do you integrate Azure Key Vault with an Azure pipeline?
144. How is authentication configured between Azure DevOps and Azure Key Vault?
145. What is an Azure DevOps Service Connection?
146. How do you create and configure Service Connections?
147. How do you troubleshoot Service Connection authentication failures?
148. How do you integrate SonarQube and Snyk into Azure Pipelines?
149. How do you configure SonarQube code quality analysis in Azure Pipelines?
150. How do you build Docker images using Azure Pipelines?
151. How do you push Docker images to Azure Container Registry?
152. How do you deploy applications to AKS from Azure Pipelines?
153. How do you deploy a .NET application to Azure App Service?
154. How do you deploy Angular or Java applications through Azure Pipelines?
155. How do you use Helm charts in Azure Pipelines for AKS deployments?
156. What is Azure Artifacts?
157. Where are build artifacts stored in Azure DevOps?
158. How do you publish and consume artifacts across pipeline stages?
159. How do you configure rollback in an Azure deployment pipeline?
160. How do you troubleshoot a failed Azure DevOps deployment?

### AWS CodePipeline, CodeBuild and CodeDeploy

161. What is AWS CodePipeline?
162. How is AWS CodePipeline different from Jenkins?
163. When would you choose AWS CodePipeline over Jenkins?
164. What AWS services can be used to implement CI/CD instead of Jenkins?
165. What is AWS CodeBuild?
166. What is the purpose of a buildspec file in AWS CodeBuild?
167. Explain the pre-build, build, and post-build stages in a CI/CD pipeline.
168. Can you reuse the same buildspec file across different CodeBuild projects?
169. How do you build a Docker image using AWS CodeBuild?
170. What is AWS CodeDeploy?
171. How do you deploy applications to AWS using CI/CD?
172. Explain an AWS architecture using CodePipeline, CodeBuild, CodeDeploy, CloudFormation, and CloudWatch.
173. How do you configure CodePipeline to deploy applications to multiple environments?
174. How do you manage deployment artifacts for AWS Lambda?
175. What are the options for publishing artifacts to AWS Lambda?
176. How do you deploy a Python application to AWS using a Jenkins pipeline?
177. How do you connect Terraform to AWS and implement CI/CD?
178. How do you provision AWS infrastructure through a CI/CD pipeline?
179. What AWS CI/CD plugins have you used with Jenkins?
180. How do you troubleshoot a slow AWS deployment pipeline?

### Build Process, Docker Images and Artifact Management

181. How do you build a Docker image during the CI stage?
182. How do you manage Docker images after a successful build?
183. What are the different stages involved in building a Docker image?
184. What is a multistage Docker build?
185. Why do we use multistage Docker builds?
186. Write a multistage Dockerfile for a Java application.
187. Write a multistage Dockerfile for a Node.js application.
188. How do you optimize Docker image size during the build process?
189. How do you remove secrets and unnecessary layers from a Docker image?
190. If a Dockerfile has ten layers and layer six fails, where will the rebuild start after fixing it?
191. How do you pass environment variables during a Docker build?
192. How do you build, tag, and push a Docker image to Docker Hub?
193. How do you authenticate Jenkins to push images to a container registry?
194. How do you troubleshoot Jenkins failing to push a Docker image to a registry?
195. What alternatives can you use if you cannot push an image to Docker Hub?
196. Which container registries have you used to store Docker images?
197. What is JFrog Artifactory?
198. What is Nexus Repository?
199. What is the purpose of artifact repositories in CI/CD?
200. At which CI/CD stage should artifacts or images be published to Nexus or Artifactory?
201. How do you troubleshoot Jenkins jobs that fail during artifact uploads?
202. How do you manage different application artifact versions?
203. How do you verify the integrity of a Docker image or build artifact?
204. How do you scan Docker images during the build process?
205. How do you scan container images at the registry level?

### SonarQube, Code Quality and Security Scanning

206. What is SonarQube?
207. What is the purpose of SonarQube in a CI/CD pipeline?
208. How do you install and configure SonarQube?
209. How do you integrate SonarQube with Jenkins?
210. How do you integrate SonarQube into Azure Pipelines?
211. How do you integrate SonarQube and Snyk into the same pipeline?
212. What is the difference between code quality and code coverage?
213. What is a SonarQube Quality Gate?
214. How do you configure Quality Gates in SonarQube?
215. How do you set code coverage thresholds using test reports?
216. What vulnerability reports are generated by SonarQube?
217. How do you identify and fix code smells reported by SonarQube?
218. What happens when a SonarQube Quality Gate fails?
219. How do you prevent deployment if security scanning fails?
220. Should SonarQube run on every pull request or every push?
221. How do you integrate automated testing into a CI/CD pipeline?
222. What security tools have you integrated into your pipelines?
223. How do you enforce security policies in CI/CD?
224. How do you design a DevSecOps pipeline for a fintech application with PCI-DSS requirements?
225. How do you ensure compliance and auditability in CI/CD across multiple regions?

### Deployment Strategies and Rollback

226. What deployment strategies have you used in your projects?
227. Explain the Rolling Update deployment strategy.
228. Explain the Blue-Green deployment strategy.
229. Explain the Canary deployment strategy.
230. What is the difference between Blue-Green and Canary deployments?
231. What are the disadvantages of different Kubernetes deployment strategies?
232. How do you implement Blue-Green deployment in your project?
233. How do you automatically switch traffic between Blue and Green environments?
234. How do you roll back a Blue-Green deployment?
235. How do you implement a fully automated Blue-Green deployment for Kubernetes microservices?
236. How do you implement Blue-Green deployment with rollback on Azure using Terraform and pipelines?
237. How do you configure a Canary deployment that sends 10% of traffic to the new application version?
238. How do you verify whether a Canary deployment is healthy?
239. What metrics would you monitor before shifting Canary traffic from 10% to 100%?
240. How do you implement Canary deployments with real-time monitoring and rollback?
241. What is the difference between Rolling Update and Canary deployment?
242. How do rolling updates behave differently for Deployments and StatefulSets?
243. How do you perform zero-downtime deployments in Kubernetes?
244. If an application has only one replica, will a rolling restart cause downtime?
245. During a two-replica rollout, if a new Pod enters CrashLoopBackOff, where will the load balancer route traffic?
246. How do you deploy application updates to EC2 instances with minimal downtime?
247. How do you implement automated rollback in a CI/CD pipeline?
248. Is application rollback handled by the CI/CD tool or Kubernetes?
249. How do you roll back a partially failed Helm release?
250. For a mission-critical production application, which deployment strategy would you choose and why?
251. How do you perform production deployments without interrupting running batch jobs?
252. How do you implement feature toggles in deployment pipelines?

### Kubernetes, Helm, Argo CD and GitOps Deployments

253. How do you deploy Kubernetes applications using Jenkins?
254. Which Jenkins plugins or tools are required for Kubernetes deployments?
255. Can you use a Kubernetes Pod as a Jenkins agent?
256. What are the drawbacks of using Kubernetes Pods as Jenkins agents?
257. How do you deploy applications to EKS using GitHub Actions?
258. How do you integrate Helm deployments into CI/CD?
259. What Helm commands are used for application deployment?
260. Why would you use Helm instead of deploying Kubernetes YAML files directly?
261. How do you deploy applications to different Kubernetes clusters using CI/CD?
262. How do you manage Kubernetes deployment configurations for multiple environments?
263. How do you manage Argo CD for development, QA, and production environments?
264. What is the difference between Argo CD and Jenkins?
265. What is the difference between Argo CD and GitHub Actions?
266. Why use Argo CD when Git is already the source of truth?
267. Why use Argo CD instead of deploying directly using Helm or kubectl?
268. What is GitOps?
269. What is the difference between push-based and pull-based GitOps deployments?
270. How do you implement GitOps using Argo CD or Flux?
271. How do you manage secrets securely in GitOps pipelines?
272. How do you detect environment configuration drift using GitOps?
273. How do you manage GitOps deployments across more than 1,000 clusters?
274. How do you handle emergency hotfixes and controlled manual overrides in GitOps?
275. How do you troubleshoot a Kubernetes deployment that succeeds but the application is inaccessible externally?

### GitHub Actions and Pipeline Integration

276. Explain the GitHub Actions pipeline used in your project.
277. What are the steps in a GitHub Actions workflow?
278. How do you structure a multistage workflow to build, test, and deploy an application to Kubernetes?
279. How do you configure GitHub Actions to deploy applications to EKS?
280. How do you configure GitHub Actions runners?
281. Why would you use a self-hosted runner instead of a default hosted runner?
282. What is a matrix strategy in GitHub Actions?
283. How is caching implemented in GitHub Actions?
284. How do you run jobs in parallel in GitHub Actions?
285. How do you configure job dependencies in GitHub Actions?
286. What is the purpose of the needs keyword?
287. What is the difference between needs and concurrency?
288. How do you prevent concurrent workflow executions?
289. How do you configure manual workflow triggers?
290. How do you manage secrets in GitHub Actions?
291. How do you pass deployment environment information to downstream jobs?
292. How do you configure workflows for multiple environments?
293. How do you secure third-party GitHub Marketplace actions?
294. How do you troubleshoot incorrect steps in an existing GitHub Actions workflow?
295. How do you automate iOS application builds using GitHub Actions?

### Terraform and Infrastructure Deployment Pipelines

296. How do you execute Terraform through a CI/CD pipeline?
297. Do you execute Terraform locally or through a pipeline?
298. How do you integrate Terraform with Jenkins?
299. How do you integrate Terraform into an Azure DevOps pipeline?
300. How do you configure Terraform deployment stages in CI/CD?
301. How do you pass Terraform variables securely through Jenkins?
302. Should Terraform .tfvars files be committed to Git?
303. How do you retrieve AWS Secrets Manager secrets during a Terraform pipeline?
304. How do you configure approval before a Terraform production deployment?
305. How do you troubleshoot a Terraform pipeline that fails because resources already exist?
306. How do you handle Terraform state conflicts when multiple pipelines run?
307. How do you optimize a Terraform pipeline taking more than 25 minutes?
308. How do you provision infrastructure in multiple AWS Regions through CI/CD?
309. How do you implement reusable infrastructure code across development and production pipelines?
310. How do you prevent Terraform pipelines from accidentally modifying resources belonging to another team?

### Production CI/CD Scenario-Based Questions

311. Your CI/CD pipeline takes 40 minutes to deploy a small code change. How would you optimize it?
312. Your pipeline fails only on Tuesdays, even though no code changes have occurred. How would you investigate?
313. A production pipeline is blocked by missing approvals, and stakeholders are unavailable. What would you do?
314. A Jenkins pipeline has five stages, but the fifth stage contains a syntax error. What will happen during execution?
315. A Jenkins pipeline is experiencing memory issues. How would you troubleshoot?
316. A Jenkins pipeline is stuck and not progressing. What steps would you take?
317. A Jenkins controller goes down. How would you troubleshoot and restore it?
318. A Jenkins job randomly fails while uploading artifacts. What would you investigate?
319. Jenkins cannot push a Docker image to the registry. How would you identify the cause?
320. A pipeline works in development but fails in production. How would you troubleshoot the differences?
321. Credentials are accidentally exposed in CI/CD logs. What immediate actions would you take?
322. How would you design secure and dynamic secret rotation in Azure DevOps pipelines?
323. Your CI/CD pipeline needs automatic rollback. How would you implement it?
324. A Canary deployment returns HTTP 502 for half the traffic. How would you troubleshoot?
325. A developer commits infrastructure changes locally but does not push them to the repository. What happens when the pipeline runs?
326. A new feature must deploy from a feature branch instead of the main branch. How would you configure the pipeline?
327. Three microservices need to build and deploy in parallel. How would you design the pipeline?
328. In a monorepo, how would you build and deploy only the microservices affected by a commit?
329. How would you deploy the same application to Kubernetes and on-premises servers through one pipeline?
330. How would you design a pipeline to deploy an application to 100 servers using Ansible?
331. How do you design a scalable and highly available CI/CD platform for multiple teams?
332. How do you enforce secure pipeline-as-code practices across large distributed teams?
333. How do you dynamically provision temporary development and test environments using pipelines?
334. How do you implement cost-efficient infrastructure autoscaling for high CI/CD workloads?
335. How do you design and implement an end-to-end CI/CD pipeline for machine learning models?
336. How do you handle pipeline failures during production release activities?
337. How do you identify whether a failed deployment is caused by build artifacts, infrastructure, application configuration, or deployment scripts?
338. How do you ensure release quality through automated testing, security scanning, approvals, and post-deployment validation?

## Shell / Bash / PowerShell / Python Scripting Interview Questions

### Shell and Bash Scripting

1. Have you worked with shell scripting, and what have you automated using it?
2. Which scripting languages have you used in your projects?
3. What was the most recent automation script you wrote?
4. Write a shell script to monitor a directory and store the names of newly created files in another file.
5. Write a shell script to monitor a directory and automatically copy newly created files to a remote server using SCP.
6. Write a shell script to securely copy the /nobackup directory from one Ubuntu VM to another using an SSH private key.
7. Write a shell script to find and delete files older than 30 days.
8. Write a shell script to back up log files from the last seven days and delete older logs.
9. Write a shell script to compress logs older than 30 days and delete logs older than 90 days, scheduled daily using cron.
10. Write a shell script to check disk usage and send an email alert when utilization exceeds 80%.
11. Write a shell script to monitor a service, restart it if it stops, and log the event.
12. Write a shell script to rename all .txt files in a directory by appending the current date.
13. Write a shell script to find all files larger than 1 GB.
14. Write a shell script to search for the keyword ERROR in a text file, ignoring case.
15. Write a shell script to find ERROR and WARNING entries in test.log and save them into separate files, taking the log filename as an argument.
16. Write a shell script to extract log entries containing ERROR for the current date.
17. Write a Bash script for log analysis.
18. Write a script to detect when the same error occurs more than three times and generate an alert for that error.
19. Write a script to find a specific word in a file and replace it with another word.
20. Write a script to find the first occurrence of a pattern in a file and print the complete matching line.
21. Write a shell script to find files in the current directory and all subdirectories that were modified more than five hours ago, but still within the current day.
22. Write a shell script to calculate the sum of numbers from 1 to 100.
23. Write a shell script that accepts an integer N and prints numbers in a triangular pattern.
24. Write a shell script to print numbers divisible by 3 or 5, but not by 15, within a given range.
25. Write a shell script to count the number of running processes belonging to the ubuntu user.
26. Write a Bash script or command to count the total number of lines in a file.
27. Write a script to validate whether a given IP address is a valid IPv4 address.
28. Write a script to check whether an external API is reachable before making a request.
29. How can AWS backups be automated using shell scripting?
30. What is the difference between & and && in shell scripting?
31. How do you assign and print a variable in Bash?
32. How do you use sed to remove the first and last lines of a file?
33. What is the purpose of the sed command?
34. What is the difference between find and sed?
35. How do you find the tenth word in a file?
36. How do you find a running Linux process and terminate it using commands?
37. How do you troubleshoot a script that fails with the error Too many open files?
38. How do you schedule a shell script using cron?
39. How do you configure a cron expression to schedule a Linux job?
40. How would you structure pre-task checks, main tasks, and post-task validation in a Red Hat patch automation script?
41. Explain the use of the Linux commands finger, comm, netstat, jq, yq, at, atq, shuf, lsblk, less, last, nc, mtr, iftop, lsof, blkid, mkfs, and nice.

### PowerShell Scripting

### How would you schedule a PowerShell task to run every 15 minutes on Windows, and how would you schedule the equivalent task using cron on Linux?

### Python Scripting – Fundamentals

43. What experience do you have with Python scripting?
44. Where have you used Python in your projects, and what tasks have you automated?
45. What are Python libraries?
46. What is the difference between a Python list and a tuple?
47. What is the difference between a Python list and a set?
48. What is slicing in Python, and how do you slice a string from index 0 to 4?
49. What is the difference between shallow copy and deep copy in Python?
50. What are Python decorators?
51. What is inheritance in Python?
52. Have you created a custom Python module, and can you demonstrate one that prints Hello, World?
53. What is the purpose of searching for a keyword in Python?
54. How does Python's Global Interpreter Lock (GIL) affect multithreaded web service performance, and what alternatives can overcome its limitations?
55. What can cause memory issues in a Python program, and how do you troubleshoot them?

### Python Scripting – String and List Coding

56. Write a Python program to reverse a string.
57. Write a Python program to reverse a string without using loops, slicing shortcuts, built-in functions, or external libraries.
58. Write a Python program to reverse words from a given list.
59. Write a Python program to remove the first duplicate element from a list.
60. Write a Python program to separate elements of a list based on whether each element starts with a or b, using ['abc', 'vca', 'abc', 'bca'].
61. Write a Python program to find the length of the longest substring without repeating characters.

### Write a Python program that takes an integer of at least two digits and determines whether it is divisible by 3, by 5, or by both. If divisible by neither, check whether it is prime; if it is not prime, return all divisors excluding 1 and itself.

### Python Scripting – Files, Logs and Monitoring

63. Write a Python script to monitor a directory every minute and print the names of newly added files.
64. Write a Python function that takes a list of dictionaries representing job logs and returns the job IDs whose status is FAILED.
65. Write a Python script to find a missing file.
66. Write a Python script to check disk usage and send an alert when utilization exceeds a threshold.
67. Write a Python script to extract all unique IP addresses from a continuously updated application log and print the total count.
68. Write a Python script that accepts a single uppercase alphabetic character, finds the matching numeric values in a file, calculates their minimum, maximum, and sum, and then deletes the matching lines from the file.

### Python Scripting – Cloud Automation

69. Write a Python script to list all running EC2 instances tagged PROD.
70. How would you use Python to automate infrastructure monitoring tasks?
71. How do you retrieve secrets from AWS Secrets Manager using Python?
72. How would you automate the daily transfer of files generated on an EC2 instance to Amazon S3 using a script?
73. How would you write an automation script to reduce cloud infrastructure costs?
74. What production automation have you implemented using Python, and what was the outcome?

## Monitoring & Observability Interview Questions

### Monitoring and Observability Fundamentals

1. What is monitoring?
2. What is observability?
3. What is the difference between monitoring and observability?
4. Explain the architecture of an observability system.
5. What are the three pillars of observability?
6. Explain metrics, logs, and traces, including the tools used for each.
7. Why do we need distributed tracing when application logs are already available?
8. How does observability help maintain site reliability?
9. What monitoring and observability tools have you used in your projects?
10. What monitoring solutions have you implemented from scratch?
11. What are your day-to-day monitoring and observability responsibilities?
12. What types of monitoring agents have you installed in your environment?
13. How do you design a monitoring solution for a production application?
14. What observability components are required for an application, including monitoring, alerting, logging, and remediation?
15. How do you monitor system performance?
16. What proactive monitoring solutions have you implemented?
17. How do you monitor the health of microservices?
18. How do you collect application-level metrics?
19. Which metrics indicate whether an application is up or down?
20. How do you identify how many users are affected by an application outage?

### SRE, SLA, SLO, SLI and Error Budgets

21. What are SLA, SLO, and SLI?
22. What is the difference between SLA, SLO, and SLI?
23. Why are SLA, SLO, and SLI important?
24. How do you define SLAs and SLOs for an application from a business perspective?
25. How do you select appropriate SLIs for an application?
26. What is an error budget?
27. How do you use error budgets in reliability management?
28. What are the four golden signals of monitoring?
29. How have you used SRE golden signals to configure dashboards and alerts?
30. What is SLO-based alerting?
31. Have you implemented SLO-based alerts in your project?
32. What is a burn rate in SRE?
33. What is an ideal error-budget burn rate?
34. How do you monitor end-to-end SLA compliance for a payment-processing pipeline?
35. How do you use observability to reduce Mean Time to Recovery (MTTR)?

### Prometheus Monitoring

36. What is Prometheus?
37. Why do we use Prometheus?
38. Where have you deployed Prometheus in your infrastructure?
39. Explain the architecture of Prometheus.
40. How does Prometheus collect metrics?
41. How do you install and configure Prometheus?
42. What components and agents are installed along with Prometheus, and what does each component do?
43. How do you retrieve application metrics using Prometheus?
44. How do you monitor Kubernetes clusters using Prometheus?
45. What metrics do you monitor using Prometheus?
46. How do you integrate Prometheus with Grafana?
47. How do you write PromQL queries for monitoring and alerting?
48. Write a PromQL expression to alert when CPU usage exceeds 80% on any node.
49. How do you configure an alert when CPU usage remains above 90% for five minutes, but the number of running Pods is below five?
50. How do you monitor Kubernetes Pod and node health using Prometheus?

### Grafana Dashboards and Alerting

51. What is Grafana?
52. Explain how Prometheus and Grafana work together.
53. How do you install and configure Grafana?
54. How do you set up a Grafana dashboard?
55. What Grafana dashboards have you created in your project?
56. What types of alerts have you configured in Grafana?
57. How do you configure custom alerts in Grafana?
58. What data sources can be integrated with Grafana?
59. How do you monitor CPU metrics using Grafana?
60. How do you configure a Prometheus-based dashboard in Grafana?
61. How do you create or configure Grafana extensions?
62. How do you use Grafana dashboards for day-to-day incident monitoring?

### ELK, EFK, Elasticsearch, Kibana and Loki

63. What is the ELK stack?
64. What is the EFK stack?
65. Explain the end-to-end architecture and setup of an ELK stack.
66. What is the role of each component in an ELK or EFK stack?
67. What is Elasticsearch?
68. How does Elasticsearch perform filtering and indexing?
69. What is Kibana?
70. How do you install and configure Kibana?
71. What are indices in Kibana?
72. What data sources are used by Kibana?
73. How does Kibana connect to Elasticsearch?
74. Write a YAML configuration snippet for connectivity between Kibana and Elasticsearch.
75. How are application logs segregated in an ELK stack?
76. What logging agents have you used with ELK or EFK?
77. What is Loki?
78. What are Prometheus, Grafana, and Loki, and how are they used together?
79. What is a log rotation job, and how does it work?
80. How would you design a highly available logging system for more than 100 microservices across three regions?
81. How would you implement centralized logging across multiple cloud providers and environments?
82. How do you collect Kubernetes logs when installing Filebeat directly on worker nodes is not permitted?
83. How do you troubleshoot missing or incomplete logs across AKS, Ingress, applications, and infrastructure?
84. How do you collect logs directly from Docker containers?
85. How do you stream logs from a specific path inside a Docker container to Amazon S3?
86. How do you troubleshoot a database when logs are not being written?
87. How do you troubleshoot CrashLoopBackOff when the Pod logs do not show any useful information?

### AWS CloudWatch and Cloud Monitoring

88. What is Amazon CloudWatch?
89. How do you monitor application CPU utilization using AWS CloudWatch?
90. Which CloudWatch metrics are important for infrastructure monitoring?
91. How do you create a custom metric in CloudWatch?
92. What is the purpose of CloudWatch Log Groups?
93. How do you create CloudWatch alarms?
94. Where do you configure CloudWatch alarms?
95. Are memory and disk metrics available by default in CloudWatch?
96. Why do we need the CloudWatch Agent?
97. How do you install and configure the CloudWatch Agent?
98. How do you configure Auto Scaling based on memory and disk usage metrics?
99. Do you need to modify an Auto Scaling launch template when configuring CloudWatch agents?
100. How do you monitor EC2 CPU and memory utilization?
101. Which metrics do you use to monitor high availability on web and application servers?
102. How do you monitor the health of AWS load balancers?
103. How do you identify a sudden spike in application CPU usage on AWS?
104. How do you monitor and alert on AWS cloud resources?
105. How do you troubleshoot Lambda logs that are not appearing in CloudWatch Log Groups even though IAM permissions appear correct?
106. How do you send EC2 logs to S3 while monitoring CPU metrics and generating CloudWatch alarms?
107. How do you filter logs for a particular IP address in a CloudWatch Log Group?
108. How do you extract critical information from VPC Flow Logs?
109. How do you analyze VPC Flow Logs stored in S3?
110. How do you use monitoring and observability tools to optimize infrastructure costs?

### Azure Monitor and Log Analytics

111. What is Azure Monitor?
112. How do you configure monitoring using Azure Log Analytics?
113. How do you create alerts in Azure Monitor?
114. Explain the steps required to configure Azure Monitor alerts.
115. How do you create custom alerts in Azure?
116. How do you monitor Azure VM memory utilization and generate an alert when it exceeds 80%?
117. How do you configure CPU and memory utilization thresholds for Azure VMs?
118. Which metrics are used when VM CPU or memory utilization exceeds 75%?
119. What is an Azure Monitor Action Group?
120. How do you create and configure an Action Group?
121. What KQL queries have you used for troubleshooting through a Log Analytics Workspace?
122. How do you automate Azure infrastructure monitoring using scripts?
123. How do you prioritize critical alerts when hundreds of alerts are generated by production infrastructure?
124. How do you investigate application latency in AKS when no code changes have occurred?

### Kubernetes Monitoring and Troubleshooting

125. How do you monitor Kubernetes Pod health and performance in production?
126. How do you monitor Kubernetes nodes using Prometheus?
127. How do you detect and troubleshoot a Pod going down?
128. How do you detect and troubleshoot a Kubernetes node going down?
129. How do you investigate high CPU utilization in a Pod when its application logs show no errors?
130. How do you troubleshoot an application when Kubernetes monitoring reports normal metrics but users are experiencing slowness?
131. How do you investigate a Kubernetes cluster that suddenly becomes slow?
132. How do you debug an AKS application that randomly fails health checks?
133. How do you troubleshoot two healthy Kubernetes Pods that cannot communicate when logs and events show no errors?
134. How do you troubleshoot Kubernetes Ingress Controller slowness during peak traffic?
135. What application and infrastructure metrics would you monitor during a Canary deployment before increasing traffic from 10% to 100%?
136. How do you implement a Canary deployment with real-time monitoring and automatic rollback?
137. How do you use metrics and logs to determine whether an EKS cluster needs to scale?
138. How do you monitor an EKS cluster during high-traffic periods?
139. How do you investigate a Kubernetes node that remains NotReady after reviewing kubectl logs and events?
140. How do you detect and investigate unusual restarts or failures in Kubernetes Pods?

### Alerts, Incident Management and Reliability

141. How do you configure alerts for a newly deployed application?
142. How do you receive alerts in your project?
143. How do you handle CPU and disk utilization alerts?
144. How do you design alerts to avoid false positives and unnecessary notifications?
145. How do you prevent alert fatigue?
146. How do you decide which monitoring alerts require immediate attention?
147. How do you troubleshoot an application that has suddenly developed high response latency?
148. How can monitoring tools help identify the root cause of an application latency problem?
149. How do you investigate a production application's response time increasing from 5 milliseconds to five minutes?
150. How do you identify the root cause when an application becomes unavailable in the cloud?
151. Describe a critical production incident you handled and how you resolved it.
152. How do you respond to a production outage during peak business hours?
153. What performance issues have you resolved in production?
154. How do you use observability to identify and remediate recurring production incidents?
155. How do you implement a blameless postmortem process to prevent recurring incidents?
156. How do you automate incident remediation to reduce MTTR?
157. How do you investigate latency in a distributed application when individual component metrics appear healthy?
158. What monitoring information would you gather before deciding to roll back or apply a hotfix?
159. How do you monitor service availability and identify the users affected by a failure?

### Monitoring Automation and Scenario-Based Questions

160. Write a shell script to monitor server disk usage, log the details, and send an email alert when usage exceeds 80%.
161. Write a shell script that checks whether a service is running, restarts it if necessary, and logs the event.
162. Write a Python script to monitor disk usage and send an alert when it exceeds a configured threshold.
163. Write a script to detect when the same application error occurs more than three times and generate an alert.
164. How do you automate log monitoring and error detection?
165. How do you automate infrastructure monitoring using Python or shell scripts?
166. How do you investigate application errors using centralized log analysis?
167. How do you monitor a production application's response time and identify sudden degradation?
168. How do you monitor machine learning models in production?
169. How can AI assist with cloud infrastructure monitoring?
170. What types of alerts have you configured in Dynatrace?
171. Have you installed Dynatrace OneAgent on applications?
172. How do you monitor application performance using Dynatrace?
173. How do you configure custom alerting rules for production applications?
174. How do you combine application logs, infrastructure metrics, and traces during incident investigation?
175. How do you determine whether an application performance issue originates from the application, database, network, or underlying infrastructure?
