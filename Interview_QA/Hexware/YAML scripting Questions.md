# 1,000 Interview Questions — YAML Scripting for DevOps Tools

Below are **1,000 interview questions** divided into five sections (200 questions each): **Basic (Q1–Q200)**, **Intermediate (Q201–Q400)**, **Advanced (Q401–Q600)**, **Scenario-Based (Q601–Q800)**, and **Project-Level Hands-On & Troubleshooting (Q801–Q1000)**.
They cover YAML fundamentals and practical YAML use for **Ansible (playbooks & inventory), Kubernetes manifests, Docker Compose, GitHub Actions workflows, and Azure DevOps YAML pipelines**.

---

# Section 1 — Basic (Q1–Q200)

1. What is YAML and why is it used in DevOps?
2. What file extensions are commonly used for YAML files?
3. Explain the difference between YAML and JSON.
4. What are YAML scalars, mappings, and sequences?
5. How is indentation used in YAML?
6. Can YAML use tabs for indentation? Why or why not?
7. What is a flow style vs block style in YAML?
8. How do you represent a list in YAML? Give an example.
9. How do you represent a map (dictionary) in YAML? Give an example.
10. What is a YAML anchor and alias? Give an example.
11. How do merge keys work in YAML (`<<`) ?
12. What is the literal block scalar `|` and when do you use it?
13. What is the folded block scalar `>` and when is it useful?
14. How are comments represented in YAML?
15. How does YAML handle multi-document files?
16. What is a YAML tag and what are local/global tags?
17. How are boolean values written in YAML? (true/false variations)
18. How are null values represented in YAML?
19. How are numbers and strings distinguished in YAML?
20. How do you force a scalar to be a string if it looks like a number?
21. Why is quoting sometimes necessary in YAML keys or values?
22. What are common YAML parsing libraries for Python? (e.g., PyYAML)
23. What is `yamllint` and why use it?
24. What is `kubeval` and what does it validate?
25. Why should you validate YAML before applying it?
26. How does whitespace sensitivity influence YAML parsing?
27. What problems arise from mixing tabs and spaces in YAML?
28. How do you include special characters like `:` in a YAML string?
29. How do you represent an empty list and empty map in YAML?
30. How do you represent a list of maps in YAML?
31. What is the YAML header `%YAML 1.2` used for?
32. What is a YAML alias cycle and how to avoid it?
33. How do YAML anchors help in DRY (Don't Repeat Yourself) for manifests?
34. How do you write comments that describe blocks in YAML best practice?
35. How to convert JSON to YAML and vice versa?
36. What is meant by YAML “explicit typing”?
37. How do you disable YAML implicit typing when parsing?
38. What is a safe loader vs full loader in PyYAML?
39. What issues can arise from using `yaml.load()` unsafely?
40. How do you write a YAML list where items span multiple lines?
41. How to escape a leading `#` in a YAML string?
42. How to include a block of shell script in YAML for a CI step?
43. What is YAML’s `!!map` or `!!str` tag usage?
44. How do you embed a literal `|` or `>` character in YAML value?
45. What is canonical YAML representation?
46. How to represent date/time values in YAML?
47. How to ensure date strings are not parsed as timestamps?
48. How to represent nested structures in YAML?
49. How to document YAML files for team members? (README + comments)
50. What is YAML folding behavior for trailing newlines?
51. How to use environment variables inside YAML (conceptually)?
52. Why YAML is commonly used for Kubernetes manifests?
53. How to write a minimal Kubernetes Pod manifest in YAML?
54. What are the mandatory top-level fields in a K8s manifest? (`apiVersion`, `kind`, `metadata`)
55. What is `metadata.name` in Kubernetes YAML?
56. What is `metadata.namespace` and when to use it?
57. How to define container `image` in a Pod spec?
58. How to set container `ports` in Kubernetes YAML?
59. How to specify resource requests and limits in YAML?
60. How to add environment variables to a container in YAML?
61. How to mount a ConfigMap into a Pod via YAML?
62. How to mount a Secret into a Pod via YAML?
63. What is a Kubernetes Service manifest and basic fields?
64. How to write a Deployment manifest with `replicas`?
65. What is `selector` in Deployment/Service YAML?
66. How to define a `livenessProbe` in a container spec?
67. How to define a `readinessProbe` in a container spec?
68. What is a `volume` and how to define it in YAML?
69. How to use `hostPath` volume in a manifest (and when to avoid)?
70. How to define a PersistentVolumeClaim in YAML?
71. How to reference a PVC from a Pod spec?
72. How to create a Kubernetes Secret from YAML (base64 encoding)?
73. How to specify `imagePullSecrets` in Pod YAML?
74. What is a `DaemonSet` and what YAML fields does it use?
75. How to write a `StatefulSet` manifest basics?
76. What is `serviceAccountName` in Pod YAML?
77. How to define a `CronJob` manifest?
78. What is a Kubernetes `Ingress` manifest minimal structure?
79. How to define container `args` vs `command` in YAML?
80. How to specify `nodeSelector` in Pod YAML?
81. What is `tolerations` in YAML for scheduling?
82. How to define `affinity` rules in YAML?
83. How to use `topologySpreadConstraints` in YAML?
84. How to specify `dnsPolicy` in Pod YAML?
85. How to create a ConfigMap manifest with data keys?
86. How to write a simple Docker Compose file (service + image)?
87. What is the `version` field in docker-compose.yml (v2 vs v3)?
88. How to define multiple services in docker-compose YAML?
89. How to declare volumes in Docker Compose?
90. How to define networks in docker-compose YAML?
91. How to expose ports in docker-compose YAML?
92. How to pass environment variables to services in Compose YAML?
93. How to use `env_file` in docker-compose YAML?
94. How to set `depends_on` in Compose and what it implies?
95. What is `restart` policy in docker-compose YAML?
96. How to define a `build` context in Compose YAML?
97. How to mount host paths as volumes in Compose YAML?
98. How to define `healthcheck` in docker-compose YAML?
99. How to scale a service in Compose using `docker-compose up --scale`?
100. How to define a multi-container pod pattern in Compose vs Kubernetes?
101. What is a GitHub Actions workflow YAML file name and location?
102. What is the basic structure of a GitHub Actions workflow? (`name`, `on`, `jobs`)
103. How to define a `job` in a GitHub Actions workflow YAML?
104. How to specify a `runs-on` runner in workflow YAML?
105. How to write a `step` that uses `uses: actions/checkout@vX`?
106. How to run shell commands in a GitHub Actions step using `run`?
107. How to use `env:` in a job or step in Actions YAML?
108. How to define a matrix build in GitHub Actions YAML?
109. How to use `secrets` in GitHub Actions YAML?
110. How to trigger workflows on `push` and `pull_request` in YAML?
111. How to define scheduled runs in GitHub Actions (`schedule: cron`) ?
112. What is `workflow_dispatch` and how to use it?
113. What is `needs` for job dependency in Actions YAML?
114. How to upload artifacts in GitHub Actions YAML? (`actions/upload-artifact`)
115. How to cache dependencies in GitHub Actions YAML? (`actions/cache`)
116. How to define reusable workflow inputs and outputs?
117. Where are Azure DevOps YAML pipeline files stored by convention?
118. What are basic Azure Pipelines YAML top-level keys? (`trigger`, `pool`, `steps`)
119. How to define a simple build job in Azure DevOps YAML?
120. How to specify an agent VM image in pipeline YAML? (`vmImage`)
121. How to checkout source in an Azure pipeline YAML step? (`checkout: self`)
122. How to publish build artifacts in Azure Pipelines YAML? (`publish: artifact`)
123. How to define variables in Azure DevOps YAML? (`variables:`)
124. How to use templates in Azure DevOps YAML? (`template:`)
125. How to trigger pipeline on changes to specific paths in Azure YAML? (`paths`)
126. How to use conditionals (`condition:`) in Azure Pipelines YAML?
127. How to use stages in Azure DevOps YAML? (`stages:`)
128. How to declare deployment jobs to environments in Azure YAML? (`deployment:`)
129. How to use `resources` in Azure Pipelines YAML to reference other pipelines?
130. How to reference secure files or service connections in Azure YAML?
131. What is the YAML anchor best practice to reuse repetitive env blocks?
132. How to format multiline shell scripts in YAML steps?
133. How to pass secrets safely to YAML-based CI/CD tools? (built-in secret stores)
134. How to include external files or templates in Ansible playbook YAML? (`include_tasks`, `import_playbook`)
135. How to declare `hosts:` in an Ansible playbook YAML?
136. How to write a simple Ansible task in YAML to install a package?
137. How to register variables in Ansible playbook YAML? (`register:`)
138. How to use `when:` conditional in Ansible YAML tasks?
139. How to define `roles:` in an Ansible playbook YAML?
140. How to use `vars_files:` in an Ansible playbook?
141. What is the inventory file format for Ansible YAML inventory?
142. How to define group variables in group_vars YAML files?
143. How to write host_vars YAML files for per-host variables?
144. How to use `become: yes` in Ansible YAML playbooks?
145. How to tag tasks in Ansible YAML with `tags:`?
146. How to use `loop:` and `with_items` in Ansible YAML?
147. How to write a handler in an Ansible playbook YAML?
148. How to set Ansible fact with `set_fact` in YAML?
149. How to use `ansible.builtin.copy` module in YAML?
150. What is `ansible.cfg` and how does it relate to YAML playbooks?
151. How to use Jinja2 templating within YAML values for Ansible?
152. How to create a dynamic inventory script for Ansible that outputs YAML?
153. How to run `ansible-playbook` with `--check` using YAML playbooks?
154. What is the role of `defaults/main.yml` in an Ansible role?
155. How to define dependencies in `meta/main.yml` role YAML?
156. How to use vault-encrypted YAML vars in Ansible?
157. How to reference Ansible variables inside YAML tasks safely? (`{{ var }}`)
158. How to write an Ansible `block` with `rescue` and `always` in YAML?
159. How to use `delegate_to:` in an Ansible YAML task?
160. What is YAML folding vs literal in the context of long command scripts in playbooks?
161. How to write a K8s `ServiceAccount` YAML manifest?
162. How to create an RBAC `Role` and `RoleBinding` in YAML?
163. How to specify `imagePullPolicy` in container YAML?
164. How to add labels to K8s objects in YAML?
165. How to add annotations to K8s objects in YAML?
166. What is the `spec.selector.matchLabels` pattern in Deployment YAML?
167. How to set `strategy` type in Deployment YAML (RollingUpdate vs Recreate)?
168. How to configure `rollingUpdate` parameters in YAML?
169. How to create a headless Service in YAML and when to use it?
170. How to create a ConfigMap from a YAML file manifest?
171. How to use `kubectl apply -f` with a YAML manifest?
172. How to do a `kubectl create -f` vs `apply -f` with YAML differences?
173. How to split large YAML manifests into multiple files for K8s?
174. How to use `kustomize` concepts in YAML (patches) at a high level?
175. How to write health checks (`httpGet`, `exec`, `tcpSocket`) in YAML probes?
176. How to define container lifecycle hooks (`postStart`, `preStop`) in YAML?
177. How to set `terminationGracePeriodSeconds` in Pod YAML?
178. How to annotate pod disruption budgets in YAML?
179. How to define `PersistentVolume` manifest basics in YAML?
180. How to declare `storageClassName` in PVC YAML?
181. How to set `readOnly` mount in volumeMounts YAML?
182. How to reference a Secret as env var in Pod YAML?
183. How to write a Service of type `LoadBalancer` in YAML?
184. How to use `hostNetwork: true` in Pod YAML and implications?
185. How to configure `dnsConfig` in Pod YAML?
186. How to define `imagePullPolicy: IfNotPresent` in YAML and why?
187. How to create a `NetworkPolicy` manifest YAML sample?
188. How to define `podAntiAffinity` in YAML to spread pods?
189. How to include config files from `ConfigMap` as files into containers?
190. How to use `command:` vs `args:` in container specs in YAML?
191. How to use manifests to create `ReplicaSet` directly in YAML?
192. How to define `service.spec.ports.targetPort` vs `port` in YAML?
193. How to use `ingress.kubernetes.io` annotations in YAML?
194. How to define `tls` blocks in Ingress YAML?
195. How to declare `networking.k8s.io/v1` vs older Ingress apiVersions in YAML?
196. How to write a simple Docker Compose override YAML for dev vs prod?
197. How to define `profiles` in Docker Compose YAML v3.9?
198. How to use `configs` and `secrets` in Compose YAML with Docker Swarm?
199. How to declare `healthcheck` `interval`, `timeout`, `retries` in Compose YAML?
200. How to write a simple GitHub Actions `on: push` YAML trigger for CI?

---

# Section 2 — Intermediate (Q201–Q400)

201. Explain YAML anchors and aliases with an example used across multiple services.
202. How to use merge (`<<`) to combine common env vars into multiple job definitions?
203. How to prevent implicit typing (e.g., `yes`, `on`) from converting to booleans in YAML?
204. How to programmatically validate YAML schema using JSON Schema?
205. How to create and validate Kubernetes CRD YAMLs?
206. How to use `kubectl apply -k` with kustomize overlays and YAML structure?
207. How to use `kubectl diff -f` for YAML change previews?
208. How to write a Deployment YAML that sets `progressDeadlineSeconds`?
209. How to use `envFrom` to load all keys from a ConfigMap into a container?
210. How to use `configMapKeyRef` vs `secretKeyRef` in YAML env entries?
211. How to write YAML that uses `fieldRef` to reference downward API fields?
212. How to set `terminationGracePeriodSeconds` and `preStop` for graceful shutdown?
213. How to create an init container in Pod YAML and its common uses?
214. How to define `volumeClaimTemplates` in StatefulSet YAML?
215. How to use `service.spec.externalName` in Service YAML?
216. How to set up rolling updates with `maxUnavailable`/`maxSurge` in YAML?
217. How to define `podPriorityClassName` in YAML?
218. How to define `imagePullSecrets` for pulling private images in YAML?
219. How to configure `readinessGate` in Pod YAML?
220. How to create `Role` with specific verbs and resources in YAML?
221. How to write `ClusterRole` and `ClusterRoleBinding` YAML?
222. How to restrict ServiceAccount permissions via YAML?
223. How to create a PodSecurityPolicy (deprecated in new versions) — historical YAML considerations?
224. How to write YAML for a HorizontalPodAutoscaler (HPA)?
225. How to set metrics for HPA (CPU, custom metrics) in YAML?
226. How to define `PodTemplate` and reuse it across objects via YAML?
227. How to write YAML that uses `configMapGenerator` in kustomize?
228. How to handle immutable fields errors when applying YAML changes?
229. How to use `kubectl apply --prune` with labels and YAML management?
230. How to write a Job YAML for batch jobs?
231. How to configure `backoffLimit` in Job YAML?
232. How to write a CronJob YAML with concurrencyPolicy and successfulJobsHistoryLimit?
233. How to use `emptyDir` volumes in YAML and lifetime semantics?
234. How to use `projected` volumes in YAML to combine multiple sources?
235. How to configure `securityContext` at Pod and container level in YAML?
236. How to set `runAsUser`/`fsGroup` in YAML securityContext?
237. How to define `capabilities` add/drop in YAML securityContext?
238. How to write `readOnlyRootFilesystem: true` in YAML and implications?
239. How to define `lifecycle` hooks in Kubernetes YAML?
240. How to declare `startupProbe` and why use it?
241. How to create a StatefulSet with stable network IDs in YAML?
242. How to write PodDisruptionBudget (PDB) YAML and set `minAvailable`?
243. How to use `service.spec.sessionAffinity` in YAML for sticky sessions?
244. How to define `LoadBalancer` service with annotations for cloud provider in YAML?
245. How to write a Docker Compose service that builds from a local `Dockerfile` and passes build args?
246. How to use `profiles` in Docker Compose to toggle services?
247. How to include extension fields in Compose YAML for CI pipelines?
248. How to define healthchecks and use them for `depends_on.condition` in Compose v2?
249. How to write GitHub Actions YAML that checks out multiple repositories?
250. How to use `actions/checkout` `fetch-depth` option in YAML and why it matters?
251. How to define matrix variations for Node versions and OS in Actions YAML?
252. How to use `concurrency` in Actions YAML to prevent parallel runs?
253. How to use `permissions:` in Actions YAML to limit token scopes?
254. How to use `if:` expressions in GitHub Actions YAML for conditional steps?
255. How to pass outputs from one job to another in Actions YAML?
256. How to define composite actions with their own action.yml (YAML-based)?
257. How to use `uses: ./.github/actions/my-action` for local reusable action in YAML?
258. How to set `env` at workflow, job, and step levels in Actions YAML precedence?
259. How to use `workflow_call` reusable workflows in GitHub Actions YAML?
260. How to mask secrets using `::add-mask::` via YAML step run?
261. How to define YAML pipeline templates in Azure DevOps and pass parameters?
262. How to use `extends` in Azure YAML to inherit templates?
263. How to create job dependencies across stages in Azure YAML?
264. How to use deployment jobs with `environment` and `resource` in Azure YAML?
265. How to manage variable groups and secure variables in Azure YAML?
266. How to implement CI/CD gating with approvals in Azure YAML?
267. How to use `container` jobs in Azure Pipelines YAML to run steps inside a container?
268. How to use `checkout: none` in Azure YAML when you don’t need source?
269. How to run a script step in Azure YAML using `script:` vs `bash:` vs `pwsh:`?
270. How to use pipeline caching in Azure YAML to speed builds? (`cache` task)
271. How to use `dependsOn` for stages in Azure YAML?
272. How to define templates with conditional inclusion in Azure YAML?
273. How to use YAML anchors for repeated `env` blocks across multiple Azure pipeline jobs?
274. How to structure Ansible playbook YAML for multiple plays with different hosts?
275. How to use `serial:` in Ansible YAML to perform rolling updates?
276. How to write a handler with `listen:` in Ansible YAML to reuse handlers?
277. How to use `check_mode` in Ansible YAML tasks and which modules support it?
278. How to write an Ansible `template` task referencing a Jinja2 file in YAML?
279. How to use `with_items` and `loop_control` in Ansible YAML for complex loops?
280. How to write a dynamic include file with `include_vars` and YAML?
281. How to structure inventory in YAML vs INI for Ansible?
282. How to use `groups` and `children` in YAML inventory?
283. How to write a Host inventory variable like `ansible_host` in YAML inventory?
284. How to use `host_patterns` in Ansible YAML playbooks?
285. How to use `become_method` and `become_user` in Ansible YAML playbooks?
286. How to use `environment:` at task level in Ansible YAML?
287. How to use `delegate_facts: true` in an Ansible YAML task?
288. How to render complex JSON via Jinja2 in YAML values for Ansible?
289. How to avoid YAML parsing pitfalls when using `{{ }}` Jinja expressions?
290. How to write Kubernetes YAML with `ephemeralContainers` for debugging pods (concept)?
291. How to generate a Secret from a YAML manifest safely without exposing plain text?
292. How to use `kubectl apply -f` with a directory of YAML files and ordering concerns?
293. How to use YAML to define `initContainers` that prepare volumes?
294. How to configure `service.spec.externalTrafficPolicy` in Service YAML?
295. How to write a YAML manifest for `ClusterIssuer` or `Certificate` (cert-manager)?
296. How to declare `securityContext` for container user namespaces with `runAsNonRoot`?
297. How to mount ConfigMap items as subPaths in YAML volumeMounts?
298. How to write YAML for `PodPreset` (deprecated) or use alternatives?
299. How to write a Compose YAML that uses external named volumes?
300. How to migrate a version 2 Compose file to version 3 YAML considerations?
301. How to use `labels` in docker-compose YAML for orchestration/monitoring integration?
302. How to use `profiles` to enable services only in certain environments in Compose YAML?
303. How to deploy Docker Compose YAML to production with Docker Swarm stack?
304. How to use GitHub Actions YAML to build and publish Docker images to a registry?
305. How to use `permissions: contents: read` minimal token requirement for actions YAML?
306. How to use `outputs` in Azure YAML to pass values between stages?
307. How to write an Azure Pipeline that deploys to an `environment` with approval gates?
308. How to use `azure/service-connection` in YAML to authenticate Azure tasks?
309. How to define multi-stage Azure YAML with artifact handoffs?
310. How to use `trigger: none` and `pr: none` in Azure YAML when manual only?
311. How to use YAML expression syntax `${{ }}` in GitHub Actions for runtime interpolation?
312. How to use `fromJSON()` and other expression functions in Actions YAML?
313. How to use `strategy.matrix` combined with `fail-fast` in GitHub Actions YAML?
314. How to write reusable GitHub Actions workflow call signatures in YAML?
315. How to securely pass secrets from Azure Key Vault into Azure Pipelines YAML jobs?
316. How to manage environment-specific YAML overlays for Kubernetes via Kustomize?
317. How to use `helm template` with values YAML to generate manifests?
318. How to structure `values.yaml` for Helm charts (best practices)?
319. How to pass values via `--set` vs `values.yaml` in Helm when rendering YAML?
320. How to write `postStart` hook scripts in YAML that run quick checks?
321. How to define `emptyDir.medium: Memory` in YAML for tmpfs-backed volume?
322. How to configure `priorityClassName` for pods in YAML?
323. How to use `service.beta.kubernetes.io` annotations in YAML for cloud features?
324. How to define `pod-template-hash` and what it means in ReplicaSet YAML?
325. How to use `imagePullPolicy` `Always` during CI deployments?
326. How to write `Dockerfile` build args into Compose YAML and pass from environment?
327. How to compose multi-stage Docker builds and reflect tags in Compose YAML?
328. How to use conditional steps in Azure YAML based on branch name?
329. How to use `matrix` expansion in Azure Pipelines YAML? (strategy.matrix)
330. How to use YAML to define a GitHub Actions `service` (e.g., a test DB container) for tests?
331. How to declare volume mounts for Windows containers in Compose YAML?
332. How to write `hostAliases` in Pod YAML to inject /etc/hosts entries?
333. How to configure `podSecurity` admission with YAML in newer K8s (PodSecurity admission)?
334. How to create a ConfigMap from literal keys in YAML vs from files?
335. How to use `kubectl create secret generic` vs secret YAML manifest differences?
336. How to set `affinity.nodeAffinity` node selector terms in YAML syntax?
337. How to use `named ports` in Service YAML for targetPort reference?
338. How to use `kubectl apply --server-side` with declarative YAML?
339. How to write `kustomization.yaml` to patch resources created from YAML?
340. How to manage large YAML templates with `yq` for editing in pipelines?
341. How to use `ytt` (Carvel) templating vs plain YAML for overlays?
342. How to write a GitHub Actions YAML to comment test results on PRs?
343. How to use `jobs.<id>.if` for runtime conditions in Actions YAML?
344. How to store complex objects in Azure Pipeline YAML variables (objects vs strings)?
345. How to use `expression` evaluation in Azure YAML with `eq()`, `and()`, `or()`?
346. How to write an Ansible playbook YAML that uses `block/rescue` to recover?
347. How to use `ansible.builtin.wait_for` in YAML to wait for ports or files?
348. How to use `blockinfile` in Ansible YAML to insert large config fragments?
349. How to use `assert` module in Ansible YAML for precondition checks?
350. How to use `meta: flush_handlers` in Ansible YAML to immediately run handlers?
351. How to create a Kubernetes `Service` with `externalIPs` in YAML?
352. How to write `PodPreset` alternatives for injecting env vars when deprecated?
353. How to define `hostAliases` vs `hostNetwork` difference in YAML?
354. How to write `NetworkPolicy` YAML for egress-only or ingress-only restrictions?
355. How to configure Ingress `pathType` in YAML (`Prefix`, `Exact`)?
356. How to set `metadata.finalizers` in YAML and purpose?
357. How to write YAML for `PersistentVolume` with `accessModes` `ReadWriteOnce`?
358. How to set `volumeMount.mountPath: /etc/config` with `subPath` in YAML?
359. How to write a Compose YAML with service `deploy` constraints for Swarm?
360. How to use `healthcheck` `start_period` in Compose YAML?
361. How to configure a GitHub Actions YAML to use self-hosted runners group?
362. How to write a reusable Azure DevOps template YAML with parameters and default values?
363. How to use `dependsOn` and `condition` together in Azure YAML for complex flows?
364. How to write an Ansible YAML that uses `async:`, `poll: 0` and later checks `async_status`?
365. How to implement `notify` and `handlers` best practices in Ansible YAML?
366. How to use `ansible.builtin.raw` in YAML to bootstrap python on target host?
367. How to use `become: true` and `become_method: sudo` in Ansible YAML for privilege escalation?
368. How to write a K8s `Service` YAML with `externalName` to external services?
369. How to implement `initContainers` that pull secrets via sidecar pattern referenced in YAML?
370. How to incorporate `readinessProbe` that checks an application-specific endpoint with YAML?
371. How to write a Docker Compose YAML that uses `tmpfs` mounts?
372. How to set `ulimits` in docker-compose YAML for resource constraints?
373. How to write YAML to define GitHub Actions environment protection rules (environment: name)?
374. How to use `environment.url` in GitHub Actions to set environment URL after deployment?
375. How to configure `environments` and required reviewers in GitHub Actions YAML?
376. How to pass pipeline artifacts between Azure pipeline stages using YAML?
377. How to create parameterized templates in Azure YAML to reuse common steps?
378. How to write YAML that specifies container probes with `initialDelaySeconds` and `periodSeconds`?
379. How to write YAML to configure `serviceAccount` secrets mount for tokens?
380. How to create docker-compose YAML that references `.env` file for variable interpolation?
381. How to use YAML to specify `runAsGroup` in Kubernetes securityContext?
382. How to use `lifecycle` policy in Compose YAML for `stop_grace_period`?
383. How to declare `imagePullPolicy` in Helm `values.yaml` for chart flexibility?
384. How to use `kubeval` in CI to validate Kubernetes YAML against API versions?
385. How to use `kubectl set image` vs editing YAML directly for deployments?
386. How to write YAML that includes conditional template values in Helm (not plain YAML)?
387. How to structure a GitHub Actions YAML to upload coverage reports as artifacts?
388. How to write YAML to run integration tests against a Compose stack spun up as services?
389. How to configure Azure DevOps YAML to use a service connection for Azure CLI tasks?
390. How to create YAML-based Ansible inventory for dynamic cloud inventories plugin?
391. How to write K8s YAML manifests that avoid hard-coded image tags for immutability?
392. How to define `resources.limits.cpu` with millicores in YAML?
393. How to use `kubectl explain` to learn required YAML fields for resource types?
394. How to write a Compose YAML with multiple networks and connect services to specific networks?
395. How to use `condition` expressions in GitHub Actions YAML to skip steps on tag builds?
396. How to write Azure YAML that deploys ARM or Bicep templates as part of pipeline?
397. How to use `outputs` in reusable GitHub Actions workflows and consume them?
398. How to structure a complex Ansible role and call it from playbook YAML with `role:` syntax?
399. How to manage secrets in Ansible inventory YAML using `ansible-vault` encrypted vars?
400. How to write a K8s YAML that uses `pod.spec.affinity` to control pod placement?

---

# Section 3 — Advanced (Q401–Q600)

401. How to design YAML structures for multi-environment support (dev/stage/prod) using anchors?
402. How to implement secure templating patterns to avoid secret leakage in YAML?
403. How to write a Kubernetes `CustomResource` YAML instance for a CRD?
404. How to write a K8s `Admission` webhook configuration manifest in YAML?
405. How to represent complex Helm `values.yaml` patterns for arrays and nested objects?
406. How to use YAML anchors and aliases across files (tooling considerations)?
407. How to write a GitHub Actions YAML that uses `id` for step outputs and references them later?
408. How to write a multi-stage Azure DevOps YAML with conditional stage deployment strategies?
409. How to write Ansible playbook YAML that uses `strategy: free` vs `linear` and implications?
410. How to write a Kubernetes `PodPreset`-style injection using mutating webhook YAML?
411. How to design YAML templates to produce minimal diffs when regenerating manifests?
412. How to write manifests for CRI-O or containerd-specific features via YAML annotations?
413. How to use YAML to configure sidecars, e.g., service mesh proxies (Envoy) injection?
414. How to write a K8s manifest YAML for `CSI` volume driver `VolumeSnapshot` objects?
415. How to write YAML for `PodSecurity` admission labels in namespaces and enforce policies?
416. How to author complex `NetworkPolicy` YAML allowing only certain namespace-to-namespace traffic?
417. How to write a Compose YAML optimized for production (swarm mode) with `deploy.resources`?
418. How to implement blue/green or canary deployments using YAML manifests and labels?
419. How to write a GitHub Actions YAML that runs a dynamic matrix built from a script?
420. How to create an Azure YAML pipeline that deploys to multiple subscriptions using service connections?
421. How to parameterize Ansible roles using `defaults/main.yml` and `vars/main.yml` strategy?
422. How to write YAML for complex `affinity` rules combining node and pod affinities?
423. How to design YAML to create ephemeral namespaces per PR via GitHub Actions?
424. How to write a Helm chart `values.yaml` that supports per-environment overrides cleanly?
425. How to write a Kubernetes `Pod` YAML that uses `ephemeral Containers` for debug injection?
426. How to use `kustomize` transformers YAML to mutate resources (e.g., add labels) in overlays?
427. How to write Ansible playbooks that call roles with `vars:` and `when:` to toggle behaviors?
428. How to use YAML to define admission control exceptions for certain namespaces?
429. How to write a complex GitHub Actions workflow that triggers on multiple events with different jobs?
430. How to create an Azure DevOps YAML that uses deployment jobs with `deployment.strategy` options?
431. How to write YAML to define multi-arch Docker builds in GitHub Actions and push manifest lists?
432. How to write `PodSecurityPolicy` replacements using OPA Gatekeeper constraints expressed in YAML?
433. How to write YAML for `HorizontalPodAutoscaler` using custom metrics API in K8s?
434. How to create YAML that references `ConfigMap` and `Secret` generators in Helm chart templates?
435. How to design YAML for GitHub Actions caching with key/versioning to avoid cache poisoning?
436. How to craft Azure YAML pipeline templates to centralize security-sensitive steps?
437. How to write Ansible YAML that uses `host_vars` discovery from cloud metadata dynamically?
438. How to write YAML that instructs Kubernetes to perform `preStop` draining with hooks?
439. How to build advanced `Ingress` YAML with path rewrites and custom annotations for ingress controller?
440. How to write Docker Compose YAML that uses build secrets securely with BuildKit?
441. How to write GitHub Actions YAML that uploads code coverage and uses a coverage reporter action?
442. How to create an Azure Pipeline YAML that uses service principals stored as secrets to deploy?
443. How to use `imagePullSecrets` and Azure ACR integration in YAML pipelines for K8s deployments?
444. How to write Ansible YAML that handles idempotent DB migrations with locking?
445. How to write YAML for flaky test retry strategy in GitHub Actions using `continue-on-error` vs rerun?
446. How to write an Azure DevOps YAML that integrates with Terraform steps and state locking?
447. How to use YAML anchors to build environment-specific override blocks in `values.yaml`?
448. How to use YAML to define `Pod` `affinity` with `preferredDuringSchedulingIgnoredDuringExecution`?
449. How to write a `NetworkPolicy` YAML that denies all ingress by default and allows only certain peers?
450. How to use `topologyKeys` (older K8s) or modern spread constraints expressed in YAML?
451. How to write a GitHub Actions workflow YAML that uses `permissions` least privilege for `GITHUB_TOKEN`?
452. How to write YAML to define accelerated compute node selection (`nodeSelector` for GPU) and tolerations?
453. How to create complex Azure YAML that deploys AKS and then applies K8s YAMLs through pipeline tasks?
454. How to write Ansible YAML that integrates with dynamic inventory and caching strategies?
455. How to structure `values.yaml` to allow boolean toggles, lists, and nested maps without ambiguity?
456. How to write YAML that uses `kubectl.kubernetes.io/last-applied-configuration` considerations?
457. How to use `yq` to transform and merge YAML in CI pipelines safely?
458. How to design GitHub Actions YAML for monorepos to run only relevant jobs per path?
459. How to write Azure DevOps YAML that leverages `resources.repositories` for multi-repo pipelines?
460. How to write YAML for Kubernetes `Job` with `parallelism` and `completions` for batch workloads?
461. How to write `StatefulSet` YAML for postgres cluster with anti-affinity and stable PVC names?
462. How to write `Deployment` YAML with `revisionHistoryLimit` and `rollout` semantics?
463. How to write `PodDisruptionBudget` YAML that ensures a minimum number of replicas remain?
464. How to express `nodeAffinity` requiredDuringScheduling with matchExpressions in YAML?
465. How to write GitHub Actions YAML that supports reusable workflows across multiple repos?
466. How to write Azure YAML to run database migration scripts as deployment jobs with checks?
467. How to write Ansible YAML that uses `raw` module when Python not present on remote host?
468. How to write complex `ingress` YAML for gRPC services with path and host routing?
469. How to write `daemonset` YAML that runs a logging or monitoring agent on all nodes?
470. How to use YAML to configure init containers that wait for volume population?
471. How to design Compose YAML for local development that mirrors production service contracts?
472. How to write GitHub Actions YAML that builds and publishes Helm charts via CI?
473. How to design Azure DevOps YAML to use environments with `resourceName` and approvals?
474. How to handle secret rotation implications in YAML manifests for K8s and CI systems?
475. How to write YAML for a multi-container Pod that shares volumes and coordinates startup?
476. How to write a GitHub Actions YAML that creates release artifacts and tags upon success?
477. How to write an Azure DevOps pipeline YAML that triggers downstream pipelines using `resources.pipelines`?
478. How to write Ansible YAML that uses `notify` to trigger handler chains across roles?
479. How to write YAML for Helm `Chart.yaml` metadata and dependencies?
480. How to write YAML for `PodPreset` alternatives for injecting sidecars and env vars programmatically?
481. How to use YAML to define `deployment.kubernetes.io/revision` implications (rollbacks)?
482. How to express complex conditional logic in Azure YAML using templates and runtime expressions?
483. How to implement multi-stage GitHub Actions YAML with environment approvals and deployment protection?
484. How to write YAML for `Service` with `externalTrafficPolicy: Local` and its effect?
485. How to write Ansible YAML for Windows hosts with `win_*` modules and differences in path separators?
486. How to write YAML that uses `hostNetwork` and ensures port collisions are handled?
487. How to write K8s YAML that leverages `volumeClaimTemplates` in StatefulSet for dynamic PVCs?
488. How to use YAML to configure `pod.spec.automountServiceAccountToken: false` for security?
489. How to write Compose YAML using `deploy.placement.constraints` to pin services to nodes?
490. How to write Actions YAML to securely use third-party actions and pin to digest or tag?
491. How to use Azure YAML `resources.repositories` to fetch templates from external repos?
492. How to write Ansible YAML to perform transactional changes with `block/rescue/always` patterns?
493. How to author complex `values.yaml` that supports nested overrides via `helmfile` or `helm --values`?
494. How to create `NetworkPolicy` YAML to allow egress only to specific CIDRs?
495. How to create a GitHub Actions YAML that publishes multi-OS artifacts?
496. How to write Azure YAML job that runs inside Kubernetes (Kubernetes job) as part of pipeline?
497. How to write Ansible YAML to orchestrate blue/green deployments via inventory and tags?
498. How to write YAML that converts Docker Compose services into K8s manifests (manual considerations)?
499. How to write YAML to manage CRDs lifecycle with helm hooks or separate apply step?
500. How to write GitHub Actions YAML to implement rate-limited API calls using concurrency groups?

---

# Section 4 — Scenario-Based (Q601–Q800)

601. YAML indentation error causes parser failure — how would you locate and fix it?
602. A Kubernetes manifest with wrong `apiVersion` fails apply — how to debug and correct YAML?
603. GitHub Actions workflow contains `on: push` but doesn't trigger — what YAML issues could cause this?
604. Azure pipeline YAML triggers on `master` only but you expect `main` — what YAML change fixes it?
605. Ansible playbook YAML lints but fails at runtime due to Jinja syntax — how to spot the YAML vs Jinja problem?
606. A Docker Compose YAML has `depends_on` but startup ordering still causes errors — how to adjust YAML?
607. A K8s `Secret` YAML created with plain text was rejected — how to reformat correctly (base64) and why?
608. GitHub Actions step uses `${{ secrets.MY_SECRET }}` but logs show masked value — how to prevent leakage in YAML step `run`?
609. Azure YAML pipeline fails with `resource not found` when referencing service connection — what YAML fields to inspect?
610. Ansible YAML contains `when: var == "false"` but behaves unexpectedly — why and how to fix boolean typing?
611. Kubernetes `readinessProbe` YAML incorrectly formatted and always fails — how to reformat YAML for `httpGet`?
612. Docker Compose YAML mount path contains `~` and fails — how to correct YAML expression for hostPath?
613. GitHub Actions workflow YAML has `jobs.build.runs-on` mismatch — how to fix failing runner selection?
614. Azure YAML contains tabs in indentation causing parsing errors — how to detect and fix across many files?
615. Ansible inventory YAML uses duplicate keys — how to find and fix duplicates that cause variable override issues?
616. K8s YAML uses `string` for CPU request instead of number format causing HPA mismatch — how to correct YAML?
617. Docker Compose YAML service name includes uppercase letters and breaks networking — why and how to change?
618. GitHub Actions YAML matrix expands unexpectedly because of nested YAML quoting — how to fix definitions?
619. Azure YAML pipeline `condition` expression evaluates false — how to test and fix the expression syntax?
620. Ansible YAML role import failed due to missing `meta/main.yml` — how to fix role packaging for YAML usage?
621. K8s `ingress` YAML path type mismatch leads to 404 — how to adjust YAML `pathType` to `Prefix`?
622. Compose YAML `build.args` not passed — how to ensure `args` formatted correctly in YAML?
623. GitHub Actions YAML uses `uses: actions/setup-node@vX` but cache miss occurs — how to tweak YAML keys for cache?
624. Azure YAML pipeline fails with `permission denied` while accessing secure files — what YAML settings to inspect?
625. Ansible `when` conditional refers to undefined variable causing crash — how to safeguard in YAML?
626. K8s deployment YAML rolling update is stuck in crashloop — which YAML fields to inspect first?
627. Docker Compose YAML `healthcheck` causes restarts — how to tune `interval`/`timeout`/`retries` in YAML?
628. GitHub Actions YAML step `uses:` pinned to `@master` and breaks due to action change — how to update YAML to pin to a tag/sha?
629. Azure YAML uses a template parameter name mismatch causing failure — how to debug parameter passing in YAML templates?
630. Ansible playbook YAML produces `changed` every run — which YAML patterns cause non-idempotent tasks?
631. Kubernetes YAML `volumeMount.subPath` broken when used with `ConfigMap` keys — how to change YAML to use projected volumes?
632. Docker Compose YAML `env_file` contains invalid formatting and environment variables not loaded — how to fix?
633. GitHub Actions YAML `checkout` fetch-depth too shallow causing tags not available — how to modify YAML?
634. Azure YAML pipeline job times out due to no `pool` specified — how to correct YAML to select agent?
635. Ansible YAML `become` not working because `become_user` not permitted — how to resolve via YAML or sudoers?
636. K8s `Service` YAML selector mismatch due to label typo — how to find and fix via YAML diff?
637. Compose YAML named volume not created in target environment — how to check and correct `volumes:` YAML?
638. GitHub Actions YAML uses `actions/cache` but key collisions occur across jobs — how to design YAML cache keys?
639. Azure YAML fails to pick up variable group due to `variables` scoping — how to correct YAML scope usage?
640. Ansible YAML uses `with_items` with complex objects and Jinja expansion fails — how to refactor YAML to `loop:`?
641. K8s Secret YAML created via `kubectl apply` but not mounted in Pod — which YAML fields to inspect?
642. Docker Compose YAML uses `depends_on.condition` but service still fails because healthchecks not defined — how to update YAML?
643. GitHub Actions YAML `env` interpolations produce unexpected results due to quoting — how to fix YAML quoting?
644. Azure YAML pipeline resource limits hit when building many jobs — how to rework YAML with concurrency controls?
645. Ansible YAML `ansible_host` wrong IP in inventory causing remote failures — how to correct inventory YAML mapping?
646. K8s `PersistentVolume` YAML fails to bind PVC due to storage class mismatch — how to fix YAML storageClassName?
647. Compose YAML `restart: unless-stopped` applied but containers not reopening — how to check runtime vs YAML?
648. GitHub Actions YAML `schedule` cron syntax misfires due to timezone assumptions — how to adjust YAML?
649. Azure YAML `checkout` step uses shallow submodules causing missing files — how to configure submodules in YAML?
650. Ansible playbook YAML includes `loop_control.label` but labels not showing — how to structure YAML tasks properly?
651. Kubernetes YAML with `hostPort` causes scheduling failures on multi-node cluster — how to alter YAML to use Service instead?
652. Compose YAML with `network_mode: host` causes port conflicts — how to redesign YAML to use bridge network?
653. GitHub Actions YAML `workflow_dispatch` inputs not passed — how to define `inputs:` correctly in YAML?
654. Azure YAML pipeline stage skipped due to `condition: succeeded()` semantics — how to modify YAML to include `always()`?
655. Ansible YAML `vars_prompt` blocks non-interactive runs — how to guard YAML for CI environment?
656. K8s YAML `imagePullPolicy: Always` causes excessive pulls in dev — how to change YAML or use image tags?
657. Compose YAML `depends_on` does not wait for DB to be ready — how to add healthchecks in YAML?
658. GitHub Actions YAML step fails due to missing permissions for `GITHUB_TOKEN` — which YAML `permissions:` setting to change?
659. Azure YAML pipeline secrets not accessible because `allowAccess` not set — how to reference key vault secrets in YAML?
660. Ansible YAML contains `gather_facts: no` and tasks rely on facts — how to fix by updating YAML?
661. K8s YAML that sets `service.spec.type: NodePort` but port out-of-range — how to choose valid port ranges in YAML?
662. Compose YAML uses `build: context: .` but Dockerfile path different — how to change YAML `dockerfile:` key?
663. GitHub Actions YAML step times out because `run` is multline and missing `|` or `>` block scalar — how to fix YAML?
664. Azure YAML pipeline triggers infinite runs due to branch filter misconfiguration — how to correct `trigger`/`pr` paths in YAML?
665. Ansible YAML `slurp` used incorrectly on binary resulting in corrupted output — how to rewrite YAML to use `fetch`?
666. K8s deployment YAML update fails due to immutable field change (e.g., `spec.selector`) — how to change YAML safely?
667. Compose YAML uses secrets in plain text under `environment:` — how to refactor YAML to use `secrets:`?
668. GitHub Actions YAML forks sometimes run with different secrets restrictions — how to design YAML to avoid secrets in forked runs?
669. Azure YAML `resources` reference a pipeline in another project and fails — how to correct YAML cross-project resource references?
670. Ansible YAML task uses `notify` but handler not running because of tag mismatch — how to fix YAML tags?
671. K8s YAML with `readOnlyRootFilesystem: true` causes app to fail writing to `/tmp` — how to modify YAML to mount an `emptyDir`?
672. Compose YAML `ports` mapping reversed (host:container vs container:host) causing wrong connections — how to correct YAML?
673. GitHub Actions YAML `matrix` includes list of objects and fails due to serialization — how to flatten YAML matrix entries?
674. Azure YAML `pool` uses private agent queue but agents unavailable — how to adjust YAML fallback `vmImage`?
675. Ansible YAML `when` comparing strings with booleans fails — how to coerce types or adjust YAML?
676. K8s YAML `ingress` TLS secret missing leading to 404 on HTTPS — how to ensure YAML references correct secret?
677. Compose YAML `volumes` declared as short syntax and not working on Windows — how to use long syntax in YAML?
678. GitHub Actions YAML `uses` points to a local action but path incorrect—how to fix `uses: ./.github/actions/my` in YAML?
679. Azure YAML pipeline fails to expand template parameters due to indentation or missing `parameters:` — how to debug YAML templates?
680. Ansible YAML user creation fails due to home dir path with tilde `~` — how to correct YAML to use absolute path?
681. K8s YAML `cronjob` not scheduling due to timezone or invalid cron expression — how to fix YAML `schedule:`?
682. Compose YAML `environment` variables overridden by host env in CI — how to enforce YAML env values?
683. GitHub Actions YAML `artifact` expired before download in later job — how to set retention via YAML?
684. Azure YAML pipeline `checkout` fails due to LFS files — how to enable LFS in YAML?
685. Ansible YAML `copy` module encodes files in base64 unexpectedly — which YAML params cause this?
686. K8s YAML `serviceAccount` missing necessary RoleBinding causing unauthorized errors — how to add YAML `RoleBinding`?
687. Compose YAML `healthcheck` command fails due to missing `sh -c` wrapper — how to format YAML `test:` array vs string?
688. GitHub Actions YAML step uses `run:` with Windows commands on Linux runner — how to fix YAML for correct `runs-on`?
689. Azure YAML pipeline `publish` artifact fails due to wrong path — how to correct `path` in YAML step?
690. Ansible YAML `register` stores complex nested output, how to access nested keys in later YAML tasks?
691. K8s YAML `service` is `ClusterIP` but external traffic needed — how to modify YAML to `LoadBalancer` with cloud annotations?
692. Compose YAML `command` overwritten by `entrypoint` causing unexpected behavior — how to reorder or remove in YAML?
693. GitHub Actions YAML step uses `shell: bash` but `runs-on` is windows-latest — how to correct YAML?
694. Azure YAML pipeline uses `cache` but cache key formula wrong causing constant misses — how to compute YAML cache keys?
695. Ansible YAML `slurp` vs `fetch` confusion causes binary corruption — how to rework YAML to use `fetch` for files?
696. K8s YAML tries to set `hostNetwork: true` and collides with other pods — how to rewrite YAML to avoid host ports?
697. Compose YAML has `depends_on` circular dependency between services — how to detect and fix YAML architecture?
698. GitHub Actions YAML `permissions` missing `contents` read causing checkout to fail in composite workflows — how to update YAML?
699. Azure YAML pipeline `variables` mis-scoped between stages causing runtime errors — how to define `variables` at correct YAML level?
700. Ansible YAML task uses `with_dict` on an ordered list and breaks ordering expectations — how to use `loop` with `| dict2items`?
701. K8s YAML `Ingress` rules evaluated in wrong order due to host/path matches; how to reorder or use specific `pathType` via YAML?
702. Compose YAML runs `build` but CI runner lacks docker daemon; how to change YAML to use prebuilt images instead?
703. GitHub Actions YAML `jobs.<id>.needs` omitted leading to race errors — how to add YAML `needs:` dependencies?
704. Azure YAML `deployment` job fails due to missing `environment` name — how to define `environment` block in YAML?
705. Ansible YAML `become_user` set to `root` but sudoers denies NOPASSWD — how to modify YAML or sudoers to permit?
706. K8s YAML using `secretKeyRef` but secret in different namespace — how to reference across namespaces or fix YAML?
707. Compose YAML `scale` is not a v3 key — how to scale services via `docker-compose up --scale` vs YAML?
708. GitHub Actions YAML `action` version pinned to tag that no longer exists — how to update YAML to use commit SHA?
709. Azure YAML pipeline uses `task: AzureCLI@2` but `scriptType` mismatch — how to set correct YAML `scriptType: bash` vs `ps`?
710. Ansible YAML uses `with_items` and registers the loop result incorrectly — how to access `item` in registered results?
711. K8s YAML `rollingUpdate.maxUnavailable` set incorrectly causing zero available pods — how to tune YAML for availability?
712. Compose YAML `environment` values not substituting from `.env` due to syntax — how to reference variables correctly in YAML?
713. GitHub Actions YAML `actions/cache` fails due to socket path limit — how to restructure YAML to cache per job id?
714. Azure YAML pipeline `tal` missing indentation in large template causing confusing error — how to validate and reindent YAML?
715. Ansible YAML `delegate_to` to `localhost` causes path issues for copied files — how to adjust YAML target and paths?
716. K8s YAML `readinessProbe` using `exec` fails due to PATH differences — how to provide full path in YAML command?
717. Compose YAML uses `healthcheck.retries` too low and restarts keep failing — how to tune YAML `retries`/`start_period`?
718. GitHub Actions YAML uses `secrets` with colon `:` characters and interpolation breaks — how to quote or encode in YAML?
719. Azure YAML pipeline step attempts to use variable undefined at runtime — how to define default parameter in YAML template?
720. Ansible YAML `copy` fails to preserve file permissions — which YAML options to set (`mode:`) to fix?
721. K8s YAML `imagePullPolicy` `IfNotPresent` leads to stale images after CI pushes latest tag — how to update YAML strategy?
722. Compose YAML `build` context large causes slow CI; how to change YAML to use `context: ./build` and `.dockerignore`?
723. GitHub Actions YAML step writes secrets to logs due to `echo $SECRET` — how to change YAML so the step avoids printing?
724. Azure YAML pipeline uses `checkout` with `submodules: true` but submodule URL private — how to provide credentials in YAML?
725. Ansible YAML role dependency uses relative path and fails on controller; how to set correct role path in YAML `roles_path`?
726. K8s YAML `secret` data not base64 encoded, causing errors — how to convert to base64 and fix YAML?
727. Compose YAML uses `networks: default: external: true` but network missing — how to create network or fix YAML?
728. GitHub Actions YAML `paths-ignore` used but still triggers on specific file — what YAML patterns cause false negatives/positives?
729. Azure YAML pipeline `publish` artifact uses incorrect artifact name and downstream fails — how to ensure YAML artifact naming consistent?
730. Ansible YAML tries to run `become` on Windows host — how to fix YAML to use `winrm` modules and proper privilege escalation?
731. K8s YAML defines `resources.requests` but not `limits`, causing HPA unpredictable behavior — how to add YAML limits?
732. Compose YAML `secrets` used in local dev but not available in Docker Desktop; how to adapt YAML for dev vs prod?
733. GitHub Actions YAML `concurrency` group causes unrelated runs to cancel; how to scope concurrency in YAML?
734. Azure YAML pipeline uses `queue` name deprecated; how to update YAML to `pool: name: Hosted` or specify `vmImage`?
735. Ansible YAML `ansible.builtin.shell` used where `command` safer — how to rewrite YAML to avoid shell injection?
736. K8s YAML `livenessProbe` exec returning non-zero because `sh` not present in container image — how to use `httpGet` or include shell?
737. Compose YAML `restart: on-failure` keeps restarting in crashloop due to missing env var; how to add YAML pre-checks?
738. GitHub Actions YAML workflow tries to read artifact from previous workflow but cross-workflow permissions missing — how to configure YAML `permissions`?
739. Azure YAML pipeline step uses older Node version that response mismatch — how to set `node-version` in YAML `task` or `setup-node`?
740. Ansible YAML `fail` module used without clear message causing bad diagnosis—how to improve YAML to provide context?
741. K8s YAML `pod.spec.terminationGracePeriodSeconds` too small causing abrupt kills — how to increase in YAML?
742. Compose YAML uses windows named volumes with UNIX path causing failure — how to write platform-specific YAML overrides?
743. GitHub Actions YAML `uses` with `@master` breaks reproducibility — how to change YAML to `@v1` or commit SHA?
744. Azure YAML pipeline fails because `service connection` permissions revoked — which YAML references to change or update?
745. Ansible YAML `vars_files` referencing absent file causing playbook failure — how to guard YAML with `optional: true`?
746. K8s YAML includes `volumeMount.subPathExpr` with wrong expression causing mount failure—how to fix YAML?
747. Compose YAML `ports` published to privileged port <1024 and causes permission denied on host — how to avoid by remapping in YAML?
748. GitHub Actions YAML `matrix` job unexpectedly uses too many tokens and hits rate limits — how to reduce concurrency in YAML?
749. Azure YAML pipeline runs in a PR from fork and fails on secrets usage—how should YAML be written to avoid exposing secrets?
750. Ansible YAML includes `lookup('file', ...)` but file not in project path when running in AWX—how to alter YAML to use project-relative paths?
751. K8s YAML `service.spec.clusterIP` set to `None` and unintentionally headless — how to update YAML to a specific cluster IP or remove it?
752. Compose YAML `depends_on.condition: service_healthy` requires Compose v2.2; what YAML alternatives when using older versions?
753. GitHub Actions YAML `outputs` not available to dependent workflow because not persisted as artifacts — how to persist outputs using YAML?
754. Azure YAML `condition` uses `eq(variables['Build.SourceBranchName'], 'main')` but fails due to variable naming — how to correct YAML?
755. Ansible YAML `async` tasks leave orphaned processes on failure — how to handle cleanup in YAML using `failed_when` and `async_status`?
756. K8s YAML `affinity` rules prevent scheduling causing pending pods — how to simplify or relax YAML affinity rules?
757. Compose YAML `environment` referencing `${VAR}` but variable not defined on CI, causing empty values — how to set defaults in YAML?
758. GitHub Actions YAML `permissions` set to `read` prevents writing release asset — how to change YAML to include `contents: write`?
759. Azure YAML uses `runtimeParameters` not supported; how to switch YAML to `parameters` with proper syntax?
760. Ansible YAML `include_vars` loads secret file but not decrypted in AWX — how to configure YAML/run-time to use vault-id?
761. K8s YAML `Service` with `type: LoadBalancer` created but external IP stuck pending — which YAML annotations or cloud provider settings need correction?
762. Compose YAML `command` incorrectly overrides healthcheck script causing failed state — how to merge YAML to preserve both?
763. GitHub Actions YAML `schedule` cron incorrectly triggers due to misunderstanding of UTC — how to fix YAML to use correct cron?
764. Azure YAML pipeline uses `dependsOn` referencing a job that is templated and not exists at the time — how to correct template YAML ordering?
765. Ansible YAML `become` asks for password in CI because `become_method` default mismatched — how to fix YAML to use vault-stored sudo creds?
766. K8s YAML `readinessProbe` uses `exec` that needs `sh -c` for complex commands — how to format YAML properly for exec arrays vs strings?
767. Compose YAML `volumes` declared but Docker Desktop not mounting due to file sharing settings — how to alter YAML to use named volumes?
768. GitHub Actions YAML contains `@latest` in `uses` and action upstream introduces breaking change — how to secure YAML by pinning?
769. Azure YAML pipeline `resources.repositories` referencing `self` with wrong ref — how to correctly reference repo and ref in YAML?
770. Ansible YAML `set_fact` used to store large binary in memory causes controller memory pressure — how to refactor YAML to write to files?
771. K8s YAML `ingress` path collision due to wildcard host — how to adjust YAML host/path rules to ensure correct routing?
772. Compose YAML `build` uses `args` referencing secrets causing Git history leak — how to use BuildKit secrets and avoid YAML leak?
773. GitHub Actions YAML `needs` and `strategy.fail-fast` combined cause cancelled dependent jobs — how to adjust YAML to keep useful runs?
774. Azure YAML pipeline with template expansion fails due to wrong indentation in included YAML — how to enforce YAML linting in pipeline?
775. Ansible YAML `copy` vs `template` misuse leads to missing variable substitution — how to update YAML to use `template` module?
776. K8s YAML `pod.spec.securityContext` leads to RBAC denial because service account lacks permission — how to fix YAML RBAC binding?
777. Compose YAML multiple `env_file` entries conflict; which YAML precedence applies and how to fix?
778. GitHub Actions YAML `uses: actions/upload-artifact` fails on large files due to memory; how to chunk or stream in YAML steps?
779. Azure YAML pipeline `deployment` to environment triggers approval but approver missing; how to add fallback approver in YAML?
780. Ansible YAML `async` job fails silently because `poll` set to 0 and no followup check — how to modify YAML to check status?
781. K8s YAML for `StatefulSet` changed PVC name and caused data loss — how to avoid destructive YAML edits for StatefulSets?
782. Compose YAML `depends_on` left stale data volumes when tearing down with `down` — how to use `docker-compose rm -v` or YAML `volumes` configuration?
783. GitHub Actions YAML `secrets` set in repo but not in environment causing CI failure; how to scope secrets properly in YAML?
784. Azure YAML pipeline `variables` set at runtime via `--variables` not being seen in nested template — how to pass parameters in YAML?
785. Ansible YAML role with `defaults` overshadowed by `vars` — how to adjust YAML variable precedence for expected behavior?
786. K8s YAML `imagePullSecrets` specified but not created — how to create secret YAML and reference properly?
787. Compose YAML `healthcheck` using `curl` but `curl` not installed in image — how to adjust YAML to use `CMD-SHELL` or add `wget`?
788. GitHub Actions YAML step uses `env` that contains `:` char and breaks YAML parse — how to quote or escape in YAML?
789. Azure YAML pipeline `trigger` includes tags, but YAML wrong syntax causes unexpected triggers — how to correctly specify `trigger: tags:`?
790. Ansible YAML `lineinfile` regex uses `\` sequences and fails across platforms — how to escape YAML strings properly?
791. K8s YAML `Pod` fails because secret key has `/` in name leading to invalid YAML key — how to sanitize keys in YAML?
792. Compose YAML `networks` custom driver not available on host, how to fallback in YAML?
793. GitHub Actions YAML `checkout` fails due to submodule auth; how to add SSH key via secrets and YAML steps?
794. Azure YAML pipeline `container` job uses private registry, failing to pull — how to set imagePullSecrets or `container.registry` in YAML?
795. Ansible YAML `when` comparing lists fails due to quoting — how to format YAML comparisons correctly?
796. K8s YAML `HPA` reads CPU metrics but container doesn't expose metrics; how to adjust YAML for custom metrics?
797. Compose YAML `depends_on.condition` uses `service_started` but service crashes later — how to adjust YAML and rely on healthchecks?
798. GitHub Actions YAML `workflow_run` triggers but includes duplicate workflow names leading to wrong trigger — how to disambiguate YAML configuration?
799. Azure YAML pipeline `pool.vmImage` uses deprecated image name causing agent selection errors — how to update YAML to newer `vmImage`?
800. Ansible YAML includes dynamic includes that break in AWX when filesystem path differs — how to convert includes to role-based YAML for portability?

---

# Section 5 — Project-Level Hands-On & Troubleshooting (Q801–Q1000)

801. Create a YAML-based GitHub Actions workflow that builds, tests, and publishes a Docker image to Docker Hub — what YAML structure is needed?
802. Write an Azure DevOps multi-stage YAML pipeline that builds, runs tests, and deploys to AKS — what YAML pieces are required?
803. Design an Ansible playbook YAML that provisions VMs, installs Docker, and deploys a Compose stack — how to structure roles and inventory YAML?
804. Create a Helm `values.yaml` and corresponding Kubernetes manifests YAML to enable feature toggles per environment — how to design and test?
805. Convert a Docker Compose YAML for a 3-service app into Kubernetes manifests YAML using manual best-practices — what transforms?
806. Create a reusable GitHub Actions YAML workflow with inputs for image tags, environment, and release notes — how to accept and validate inputs?
807. Build an Azure DevOps YAML template repository with reusable stages and parameterized `environment` names — how to architect YAML templates?
808. Create an Ansible YAML role and playbook to manage TLS certificates from Key Vault and restart services safely — how to handle vault secrets in YAML?
809. Write a Kubernetes YAML manifest to deploy a StatefulSet-backed database with backups and retention policy — how to structure YAML for PVCs and cron backup jobs?
810. Compose a multi-stage CI GitHub Actions YAML pipeline that runs unit, integration, and end-to-end tests with artifact passing — what YAML artifacts & outputs?
811. Implement a GitHub Actions YAML that builds multi-arch images and pushes manifest lists — which YAML steps and actions needed?
812. Design Azure DevOps YAML to trigger deployment only after manual approval and add approval YAML configuration — how to implement `environment` checks?
813. Create an Ansible YAML playbook to perform rolling restarts with `serial:` and health checks — how to ensure YAML idempotence?
814. Write a YAML-based GitHub Actions workflow that automatically bumps semantic versioning and creates a release — how to structure YAML for tagging and changelog?
815. Create Kubernetes YAML manifests for an app that requires migrating config from ConfigMap to Secret during rollout — how to write migration YAML and rollout strategy?
816. Write a docker-compose YAML for a local dev environment that mirrors production service contracts and uses `profiles` — how to structure for developer ergonomics?
817. Build an Azure DevOps YAML that integrates security scanning (SAST) and fails the pipeline on high-severity issues — how to include scanning tasks in YAML?
818. Create an Ansible YAML to bootstrap a Kubernetes cluster nodes and apply network plugin manifests — what YAML tasks and templates?
819. Write a GitHub Actions YAML to deploy Helm charts to a private chart repository and update index — what YAML steps, secrets, and actions required?
820. Design a YAML manifest for K8s to implement auto-scaling using HPA and external metric adapter — what YAML resources are needed?
821. Create a set of YAML templates for Azure Pipelines to support blue/green deployments with swap slots for App Service — how to implement stage YAML?
822. Write an Ansible YAML playbook to rotate SSH keys across fleet atomically using `serial:` and notify handlers — how to coordinate YAML steps?
823. Build a GitHub Actions YAML that performs PR labeling, runs relevant test subsets, and comments results on PR — which YAML hooks and actions required?
824. Create a Kubernetes YAML for zero-downtime DB schema migrations using `preStop` hooks and migration jobs — how to orchestrate via YAML?
825. Implement a docker-compose YAML with `build` caching strategy for CI to create reproducible local images — what YAML keys to use?
826. Author an Azure DevOps YAML that deploys Terraform changes and locks state with remote backend — what YAML tasks and env vars?
827. Create Ansible YAML to provision an ACR, push docker images, and create RBAC via Azure modules — how to structure secure YAML roles?
828. Write GitHub Actions YAML that orchestrates Canary deployments with traffic shifts controlled via YAML steps and external APIs — how to structure workflow YAML?
829. Build Kubernetes YAML that implements network policies per namespace and service-level egress rules — how to validate YAML against policy?
830. Create an Azure DevOps YAML job to run integration tests inside a Kubernetes namespace spun up during the pipeline — how to write YAML for create/test/destroy lifecycle?
831. Write an Ansible YAML playbook that integrates with native cloud provider SDKs to tag resources for cost center — how to structure YAML tasks and idempotence?
832. Implement a GitHub Actions YAML that uses `workflow_call` to create reusable deploy steps shared across repos — how to design and reference YAML?
833. Create a Kubernetes YAML manifest for a StatefulSet with initialization job to create DB users — how to sequence YAML resources?
834. Write a docker-compose YAML for a microservices stack that uses user-defined networks and DNS names — how to ensure YAML service discovery?
835. Design an Azure DevOps YAML that uses job templates to run matrix tests across multiple language versions — how to parameterize YAML?
836. Create an Ansible YAML playbook to perform canary configuration changes and roll back automatically on failures — what YAML constructs to use?
837. Write GitHub Actions YAML that builds artifacts, runs SBOM generation, and fails on non-compliance — how to integrate SBOM steps in YAML?
838. Build Kubernetes YAML and Helm values.yaml to support feature flags toggled at runtime via ConfigMaps — how to design YAML values and deployments?
839. Implement a docker-compose YAML that uses healthchecks and automated restart policies to ensure dev environment resilience — which YAML parameters?
840. Author an Azure DevOps YAML that publishes and consumes artifacts between stages securely and with retention policy — how to implement YAML artifact tasks?
841. Create an Ansible YAML to create and rotate Azure Key Vault secrets and update application deployment manifests — how to propagate YAML changes?
842. Write a GitHub Actions YAML that consumes outputs from `actions/cache` and uses them in subsequent jobs — how to reference cache outputs in YAML?
843. Build Kubernetes YAML for multi-tenant deployment with namespace resource quotas and limit ranges — how to create YAML manifests per tenant?
844. Design an Azure DevOps YAML pipeline to deploy containerized workloads to AKS with security scanning and policy validation — what YAML steps and checks?
845. Create Ansible YAML that uses `block/rescue` patterns to perform safe upgrades to an application stack and rollback on errors — show YAML flow.
846. Write Compose YAML to orchestrate a local test environment for integration tests and tear down after tests — what YAML and CI integration needed?
847. Implement GitHub Actions YAML that runs dynamic test selection based on changed paths using `paths` filter logic — how to write YAML for efficiency?
848. Create Kubernetes YAML manifest for a web app requiring sticky sessions using annotation and session affinity — how to set YAML correctly?
849. Author an Azure DevOps YAML that orchestrates database backups to blob storage and rotation with retention defined in YAML — what tasks?
850. Build Ansible YAML to manage container images lifecycle: pull, tag, push, and cleanup in remote registries — what YAML modules and flows?
851. Write a GitHub Actions YAML to automate semantic-release and publish packages to npm with `GITHUB_TOKEN` — how to secure YAML?
852. Create Kubernetes YAML with sidecar container pattern for log forwarding and include proper volume mounts in YAML — how to set up?
853. Design a docker-compose YAML for local dev that includes a mailcatcher service and conditional profiles to enable it — how to implement YAML profiles?
854. Implement Azure DevOps YAML for multi-stage deployment with Canary analysis using Application Insights telemetry — how to wire YAML checks?
855. Create Ansible YAML to orchestrate multi-region VM deployments with inventory segregation and error handling — how to write role and inventory YAML?
856. Build GitHub Actions YAML that uses a self-hosted runner fleet and handles runner labels in YAML job selectors — how to configure `runs-on`?
857. Write Kubernetes YAML for an operator-managed CRD including RBAC and controller reference — what YAML resources must be created?
858. Create docker-compose YAML that uses `configs` and `secrets` for Swarm and explain conversion to K8s YAML — how to map YAML concepts?
859. Author Azure DevOps YAML to run performance tests at scale using VM scale sets as agents—how to configure `pool` and YAML job parallelism?
860. Implement Ansible YAML for zero-downtime app upgrades using load balancer state toggles via cloud modules — what YAML sequence?
861. Build GitHub Actions YAML for automated canary rollback using metrics and job outputs — how to design YAML decision gates?
862. Create Kubernetes YAML to support blue/green deployment using labels and Services with traffic shifting implemented by YAML changes — how to do this?
863. Design docker-compose YAML that facilitates local debugging by exposing container logs and mounts for in-container editing — which YAML keys to include?
864. Implement an Azure DevOps YAML that triggers on tag pushes, builds images, and uses tags in pipeline YAML to version artifacts — what fields in YAML?
865. Create Ansible YAML to manage database connection strings encrypted in vault and templated into application manifests — how to pass secrets safely in YAML?
866. Write a GitHub Actions YAML that validates K8s YAML with `kubeval` and `kube-linter` as part of PR checks — how to structure YAML jobs?
867. Build Kubernetes YAML to perform rolling secret rotations via new Secret creation and rolling restart triggered by annotation change — how to implement YAML?
868. Design docker-compose YAML to bootstrap local dev with mock services and override YAML for CI tests — what layering approach in YAML?
869. Create Azure DevOps YAML to deploy an ARM/Bicep template and populate parameters from variable groups — how to structure template `parameters:` in YAML?
870. Implement Ansible YAML to perform file system snapshots and verify integrity via checksums before destructive operations — what YAML tasks?
871. Write GitHub Actions YAML to orchestrate multi-repo releases by calling reusable workflows and passing inputs via YAML — how to configure calls?
872. Build Kubernetes YAML for a canary release that uses weighted traffic via an Ingress controller annotation in YAML — how to implement weights in YAML?
873. Design docker-compose YAML to use remote image cache and accelerate CI builds — which YAML options and Docker daemon config?
874. Create an Azure DevOps YAML that automates environment provisioning and tear-down for ephemeral test environments — how to implement `environment` and `deployment` YAML blocks?
875. Implement Ansible YAML to collect diagnostics from failing nodes and upload artifacts to a central server — how to write YAML for diagnostics collection?
876. Write GitHub Actions YAML that runs security scans, uploads SARIF results, and comments on PRs via YAML steps — what actions and YAML config?
877. Build Kubernetes YAML for an operator that requires leader election and CRD RBAC rules — what YAML manifests to author?
878. Design docker-compose YAML with multi-stage builds for fast local iterated development while preserving production image correctness — how to structure YAML?
879. Create Azure DevOps YAML that uses YAML templates to centralize deployment steps for multiple microservices — how to reference templates in YAML?
880. Implement Ansible YAML for gradual config rollouts across data centers with circuit-breaker logic in playbooks — what YAML constructs?
881. Write GitHub Actions YAML that deploys to Azure using `azure/login` and `azure/aks-set-context` actions — how to bind credentials in YAML?
882. Build Kubernetes YAML and Helm values to enable feature-flag toggles with ConfigMap hot reload — how to structure YAML values for runtime toggles?
883. Design docker-compose YAML for CI that uses named volumes persisted between runs for caching dependencies — how to declare `volumes:` YAML?
884. Create Azure DevOps YAML to deploy containerized function apps with slot swap using YAML stages — how to script slot swap in YAML?
885. Implement Ansible YAML to orchestrate service restarts with pre-checks and post-validation for SLA adherence — how to write robust YAML flows?
886. Write GitHub Actions YAML to publish artifacts to GitHub Releases and create release notes automatically — what YAML steps and secret usage?
887. Build Kubernetes YAML for an app requiring secret injection from external secret store via `ExternalSecret` CRD — how to declare YAML for external secret binding?
888. Design docker-compose YAML to run a full local stack including databases, cache, and queuing systems — what YAML best practices for resource limits?
889. Create Azure DevOps YAML that integrates with Azure Policy compliance checks before production deployment — how to invoke policy check in YAML?
890. Implement Ansible YAML that automatically repairs failed service instances using health-check driven logic — how to write self-healing YAML tasks?
891. Write GitHub Actions YAML to run integration tests against ephemeral clusters created on-the-fly from YAML — how to orchestrate create/test/destroy?
892. Build Kubernetes YAML for a highly-available service with PDBs, anti-affinity, and resource limits — show YAML considerations.
893. Design docker-compose YAML to enable developer-specific overrides using `docker-compose -f base.yaml -f override.yaml` — how to structure YAML base + overlay?
894. Create Azure DevOps YAML to manage database migrations as part of release stage with rollback conditions — what YAML steps and checks?
895. Implement Ansible YAML to automate secrets rotation and propagate to dependent services with minimal downtime — how to sequence YAML tasks?
896. Write GitHub Actions YAML for automatic image scanning on push and fail if CVEs found above threshold — what YAML integration with scanning actions?
897. Build Kubernetes YAML and Helm `values.yaml` that supports canary image tags and progressive promotion — how to parameterize YAML values?
898. Design docker-compose YAML for a CI environment that supports parallel test runners and shared test DB — how to configure YAML networks and volumes?
899. Create Azure DevOps YAML to run chaos experiments in staging using Azure CLI tasks — how to write YAML tasks that are reversible and safe?
900. Implement Ansible YAML to perform cluster-wide configuration remediation and report diffs to a central dashboard — what YAML reporting steps?
901. Write GitHub Actions YAML to publish a Helm chart to a chartmuseum or GitHub Pages index via YAML steps — how to implement authentication in YAML?
902. Build Kubernetes YAML to implement autoscaling using KEDA (event-driven) — what YAML CRDs and config needed?
903. Design docker-compose YAML for local development that supports hot-reloading of code in mounted volumes — how to avoid permission issues in YAML?
904. Create an Azure DevOps YAML that builds and signs artifacts, storing signatures with artifacts using YAML tasks — how to implement sign-and-verify in YAML?
905. Implement Ansible YAML to orchestrate multi-cloud failover with DNS update tasks — how to manage YAML inventory and cloud provider modules?
906. Write GitHub Actions YAML that integrates with SonarQube and posts quality gate status to PRs — what YAML steps are needed?
907. Build Kubernetes YAML to run ephemeral sidecar helper pods for data migration — how to helmify YAML to template parameters?
908. Design docker-compose YAML to spawn per-developer ephemeral stacks based on YAML templates with parameter substitution — how to implement?
909. Create Azure DevOps YAML to run security compliance scans and automatically open incidents via API — how to wire YAML pipeline tasks?
910. Implement Ansible YAML to perform canary DB schema changes with verification queries and rollback tasks — how to design YAML playbook steps?
911. Write GitHub Actions YAML to perform A/B testing deployment by generating two environment deployments via YAML — what YAML logic required?
912. Build Kubernetes YAML manifests to set up a multi-tenant ingress with host-based routing and cert management — how to create YAML for host rules and TLS?
913. Design docker-compose YAML for integration tests that require precise ordering and teardown, including logs aggregation — how to structure YAML?
914. Create Azure DevOps YAML that runs parallel build matrices for microservices and aggregates test reports — which YAML features to use?
915. Implement Ansible YAML to provision and bootstrap monitoring agents across heterogeneous OS types — how to structure cross-platform YAML roles?
916. Write GitHub Actions YAML to create feature-branch preview environments using YAML inputs and ephemeral namespaces — how to orchestrate via YAML?
917. Build Kubernetes YAML for operator-based upgrades and CRD-driven lifecycle management — what YAML artifacts are needed?
918. Design docker-compose YAML that integrates with host-side debugging tools (e.g., `pdb`, `delve`) exposing proper ports — how to set YAML?
919. Create Azure DevOps YAML that dynamically generates pipeline stages based on changed folders using template insertion — how to implement dynamic YAML?
920. Implement Ansible YAML to validate application health via HTTP checks and rollback by re-applying previous manifests — what YAML patterns to follow?
921. Write GitHub Actions YAML that automates DB seed data population and teardown for integration tests — how to secure secrets and clean up via YAML?
922. Build Kubernetes YAML for a multi-cloud control plane that uses federated config via YAML — what are the YAML components?
923. Design docker-compose YAML with guaranteed deterministic networking for tests requiring hostnames — how to declare YAML networks and DNS names?
924. Create Azure DevOps YAML that integrates with Key Vault to fetch secrets at job runtime securely — how to configure YAML `azureKeyVault` task?
925. Implement Ansible YAML that performs canary config rollout across Kubernetes namespaces using label toggles — how to orchestrate YAML steps?
926. Write GitHub Actions YAML to manage progressive rollout using feature flags backed by YAML-driven toggles — how to connect YAML steps to flag APIs?
927. Build Kubernetes YAML to create multi-zone Deployments using topologyConstraints and YAML `topologySpreadConstraints` — how to write YAML for distribution?
928. Design docker-compose YAML for workload that requires UID/GID mapping to host for file permissions — how to expose YAML `user:` settings?
929. Create Azure DevOps YAML pipeline to automate creation and rotation of service principals with YAML-defined schedules — how to script YAML tasks?
930. Implement Ansible YAML to capture and store inventory snapshots in Git as YAML files for audit — how to make YAML diffs human-friendly?
931. Write GitHub Actions YAML that uses reusable workflows for multiple projects with different parameter sets in YAML — how to validate YAML contract?
932. Build Kubernetes YAML to orchestrate canary analysis using K8s Job and custom metrics hooked into YAML-based jobs — how to integrate YAML with metrics systems?
933. Design docker-compose YAML for CI caching of dependency volumes across runs to reduce network usage — what YAML `volumes:` configuration?
934. Create Azure DevOps YAML that orchestrates end-to-end smoke tests in a staging slot before production swap — how to write YAML stage gates?
935. Implement Ansible YAML to coordinate blue/green migration including DNS cutover, cert swap, and verification — how to sequence YAML tasks and handlers?
936. Write GitHub Actions YAML that runs infrastructure linting, `terraform fmt`/`validate`, and reports via YAML steps — how to arrange tasks?
937. Build Kubernetes YAML with `Pod` and `InitContainer` to migrate legacy data format during first start — how to write YAML to ensure idempotent migration?
938. Design docker-compose YAML to manage multiple app versions side-by-side for compatibility testing — how to parameterize YAML with env files?
939. Create Azure DevOps YAML to orchestrate cross-subscription deployments with conditional logic based on YAML variables — how to set up YAML resource access?
940. Implement Ansible YAML for safe rollback of configuration changes with tracked backups created prior to modification — how to implement YAML backup tasks?
941. Write GitHub Actions YAML that sets up full CI for a monorepo with per-package workflows extracted via YAML templates — how to structure YAML templates and includes?
942. Build Kubernetes YAML to support zero-downtime schema change for stateful workloads using sidecar approach — how to author YAML?
943. Design docker-compose YAML for multi-service integration tests where one service must be seeded with test data via YAML-driven initialization — how to implement?
944. Create Azure DevOps YAML to run end-to-end security audit including scanning images and infrastructure manifests expressed in YAML — how to chain tasks?
945. Implement Ansible YAML to manage container runtime updates across a cluster with pre-checks and staged rollouts — how to structure YAML roles and handlers?
946. Write GitHub Actions YAML to enable self-hosted runner autoscaling based on YAML-defined queue length thresholds — how to implement YAML logic?
947. Build Kubernetes YAML with PodPreset-like injection using sidecar injector webhook configs in YAML — how to author the webhook YAML?
948. Design docker-compose YAML to run smoke-tests where services are named deterministically in YAML for test harnesses — how to set names in YAML?
949. Create Azure DevOps YAML to automate license compliance checks via YAML steps that scan dependencies and fail on violations — how to implement?
950. Implement Ansible YAML for dynamic inventory refresh, apply config changes, and report drift in YAML outputs for dashboards — how to emit YAML-friendly reports?
951. Write GitHub Actions YAML to implement cross-repo promotion of artifacts using YAML `repository_dispatch` or reusable workflows — how to structure YAML triggers?
952. Build Kubernetes YAML to orchestrate backups into object storage with retention rules expressed in YAML CronJob and Job specs — how to parameterize YAML schedule and storage secrets?
953. Design docker-compose YAML that supports conditional mounts for CI vs developer machines using YAML variable interpolation — how to write robust YAML?
954. Create Azure DevOps YAML to run chaos experiments in isolated YAML-defined environments and revert via YAML rollback steps — how to implement safe YAML tasks?
955. Implement Ansible YAML to audit and remediate insecure SSH configurations across fleet (CIS compliance) with YAML checks and fixes — how to present YAML remediation steps?
956. Write GitHub Actions YAML that generates environment-based deployment previews using YAML inputs and provides a URL output — how to build the YAML flow?
957. Build Kubernetes YAML to orchestrate external DNS changes during deployment using YAML `ExternalDNS` CRD configs — how to author YAML?
958. Design docker-compose YAML for local multi-tenant testing where each tenant runs in its own isolated network and volume — how to write YAML for tenant instantiation?
959. Create Azure DevOps YAML that automatically updates `values.yaml` and `Chart.yaml` for Helm releases and triggers CD — what YAML tasks and safeguards?
960. Implement Ansible YAML to run long-running tasks asynchronously and collect results later in YAML artifacts — how to orchestrate with `async_status` in YAML?
961. Write GitHub Actions YAML to perform dependency graph analysis and run only affected tests using YAML-driven path filters — how to implement detection in YAML?
962. Build Kubernetes YAML to roll out a new API version while keeping the old version for backward compatibility, using YAML services and selectors — how to define YAML objects?
963. Design docker-compose YAML to support reproducible developer environments with pinned image digests expressed in YAML — how to pin digests in YAML?
964. Create Azure DevOps YAML to orchestrate multi-tenant deployment pipelines with YAML templates per tenant and parameter injection — how to maintain YAML templates?
965. Implement Ansible YAML to integrate with incident response playbooks triggered by monitoring alerts and produce YAML event logs — how to structure playbooks?
966. Write GitHub Actions YAML to enable automatic rollback on failed deployments by capturing previous artifact references in YAML — how to record and use previous refs?
967. Build Kubernetes YAML to migrate from NodePort to LoadBalancer with minimal downtime and YAML steps for verification — how to sequence YAML changes?
968. Design docker-compose YAML to use buildkit secrets for credentials and avoid exposing them in YAML `args` — how to configure YAML and build pipeline?
969. Create Azure DevOps YAML to encrypt pipeline variables using secure files and pass them into tasks as YAML-secured inputs — how to declare in YAML?
970. Implement Ansible YAML that performs a safe migration of application config formats and validates using `assert` in YAML tasks — how to design testable YAML playbooks?
971. Write GitHub Actions YAML to orchestrate multi-stage release workflows with environment protections and approval rules defined in YAML — how to wire approvals?
972. Build Kubernetes YAML to support scheduled scaling events defined by YAML CronJob that calls K8s APIs — how to author YAML jobs and RBAC?
973. Design docker-compose YAML that supports replication of stateful services for local test clusters via YAML `replicas` (Swarm) — how to adapt for K8s?
974. Create Azure DevOps YAML to automate cross-tenant role provisioning (ARM role assignments) and log changes in YAML artifacts — what YAML tasks and security models?
975. Implement Ansible YAML to produce idempotent infrastructure changes with pre- and post-plan checks recorded as YAML artifacts — how to integrate plan output into YAML playbooks?
976. Write GitHub Actions YAML to detect infra drift by rendering Helm charts and diffing against cluster YAML using `kubectl diff` in YAML steps — how to implement safe checks?
977. Build Kubernetes YAML to support ephemeral test clusters spun per PR with YAML-driven naming and TTL controllers — how to implement YAML for lifecycle?
978. Design docker-compose YAML for integration tests that run in parallel with isolated networks and teardown using YAML-defined commands — how to handle concurrency in YAML?
979. Create Azure DevOps YAML to orchestrate disaster recovery drills: spin-up from backups via YAML-defined stages and verify applications — what YAML flows are needed?
980. Implement Ansible YAML to coordinate cross-team deploys with gate checks, manual approvals, and logging of decisions in YAML output — how to build governance into YAML playbooks?
981. Write GitHub Actions YAML that integrates code scanning, SBOM generation, and signs artifacts with cosign in YAML steps — what secrets and YAML steps are required?
982. Build Kubernetes YAML to support a multi-version rollout with canary analysis and automated rollbacks via YAML-driven metrics — how to author YAML and integrate with metrics systems?
983. Design docker-compose YAML that abstracts environment differences with `override.yaml` files and provides standard developer scripts to spawn environments — how to structure YAML?
984. Create Azure DevOps YAML pipeline that implements policy-as-code checks using YAML tasks that validate ARM/Bicep and Terraform plans — how to embed policy checks in YAML?
985. Implement Ansible YAML for zero-touch provisioning of bare-metal nodes including PXE boot orchestration and YAML-driven inventory registration — how to structure YAML role flows?
986. Write GitHub Actions YAML to orchestrate cross-repo dependency updates automatically and create PRs via YAML steps — how to compose YAML workflows?
987. Build Kubernetes YAML to integrate external authn/authz (OIDC providers) requiring YAML `ConfigMap` and `Deployment` changes — what YAML resources must be updated?
988. Design docker-compose YAML for CI that optimizes caching layers, uses parallel build stages, and keeps YAML maintainable — what YAML best practices?
989. Create Azure DevOps YAML to perform nightly security baseline checks and report results as YAML artifacts — which tasks and YAML structure are needed?
990. Implement Ansible YAML to orchestrate large-scale OS upgrades with canary nodes and health gating using YAML `serial` and checks — how to plan YAML playbooks?
991. Write GitHub Actions YAML to coordinate multi-cloud deployments by calling cloud CLIs in YAML steps and validating outputs — how to secure cloud creds in YAML?
992. Build Kubernetes YAML to support sidecar injection via an admission webhook and author the YAML for webhook config and CA bundle — how to create YAML for webhook?
993. Design docker-compose YAML for local test harness that extracts logs to host paths and rotates them via YAML-defined cron containers — how to implement log management in YAML?
994. Create Azure DevOps YAML to perform release approvals with conditional gates and automated post-approval verification steps — how to implement using YAML `approvals`?
995. Implement Ansible YAML to run preflight checks across networks and produce structured YAML reports to be consumed by dashboards — how to format YAML outputs?
996. Write GitHub Actions YAML to orchestrate multi-stage canary promotion using feature flags controlled by an external service and YAML-based gates — what YAML pattern to use?
997. Build Kubernetes YAML to perform rolling re-encryption of secrets (KMS change) and coordinate restarts via YAML jobs — how to write the YAML orchestration?
998. Design docker-compose YAML for a multi-service acceptance testing pipeline that must run within a single CI job and report results as YAML artifacts — how to implement?
999. Create Azure DevOps YAML to automate environment seeding, run integration tests, and tear down while publishing test results in YAML — how to structure pipeline YAML?
1000. Implement a governance model defined in YAML templates for CI/CD (reusable YAML templates, approval gates, secrets usage policies) — how to design and enforce YAML governance at scale?

---



Tell me which range(s) or output format(s) you want next and I’ll generate them.
