CI/CD, BUILD, AND DEPLOYMENT INTERVIEW QUESTIONS
================================================================================================

Source: interview questions company wise.rtf
Extracted: 400 numbered prompts, with 53 nested follow-ups.
Organization: 62 named companies plus the source group "Others".

Includes general delivery workflows, build tools and artifacts, code quality,
GitHub Actions/GitLab CI, GitOps, Helm, application deployment, verification,
and rollback. Questions specific to Jenkins, Azure Pipelines, and AWS CI/CD
services are in jenkins_azure_aws_pipeline_questions.txt.

Questions are grouped by company and labeled by topic. Wording and spelling
have been lightly cleaned; no answers have been added. Repeated source
entries are retained. Counts refer to numbered prompts, not unique questions
or every individual subquestion. Scenario follow-ups and coding requirements
remain attached to their original question.

[Context] marks a platform or scenario inferred from nearby questions.
[Relevant portion] marks the relevant part of a mixed source question.
[Unclear] and [Missing material] mark gaps in the supplied text.

TOPIC INDEX
------------------------------------------------------------------------------------------------
Application deployment: 109 prompts
Question numbers: 1, 2, 3, 7, 8, 10, 11, 19, 20, 21, 22, 23, 24, 25, 42, 45, 47, 59, 62, 64, 65, 68,
  69, 77, 78, 79, 80, 82, 87, 88, 90, 93, 95, 99, 100, 102, 103, 105, 106, 113, 118, 119, 120, 121,
  122, 123, 125, 126, 129, 130, 131, 134, 137, 142, 158, 159, 174, 178, 187, 190, 196, 203, 204,
  212, 221, 224, 225, 226, 229, 230, 243, 244, 267, 269, 271, 273, 283, 284, 285, 286, 288, 289,
  290, 292, 293, 295, 296, 297, 298, 301, 310, 312, 315, 316, 318, 327, 332, 339, 348, 350, 351,
  356, 359, 360, 370, 376, 384, 387, 392

Build tools and artifacts: 90 prompts
Question numbers: 15, 16, 17, 28, 29, 31, 32, 33, 36, 39, 43, 71, 86, 92, 94, 101, 107, 108, 109,
  114, 116, 132, 138, 139, 146, 147, 148, 149, 150, 151, 166, 169, 170, 171, 172, 173, 179, 195,
  197, 200, 201, 202, 211, 213, 214, 215, 217, 218, 222, 223, 228, 237, 238, 239, 241, 245, 246,
  247, 251, 252, 279, 280, 281, 291, 299, 305, 306, 307, 308, 309, 330, 331, 343, 344, 345, 346,
  347, 349, 352, 353, 354, 361, 362, 364, 371, 374, 386, 390, 399, 400

Code quality and testing: 13 prompts
Question numbers: 41, 84, 85, 97, 117, 135, 136, 185, 186, 253, 264, 319, 329

General CI/CD: 121 prompts
Question numbers: 4, 5, 6, 12, 13, 14, 18, 26, 30, 35, 38, 44, 48, 49, 50, 51, 52, 55, 56, 57, 58,
  61, 63, 66, 67, 70, 72, 75, 83, 91, 104, 110, 111, 115, 124, 127, 128, 133, 140, 143, 145, 157,
  160, 162, 163, 164, 165, 167, 168, 175, 180, 182, 184, 188, 189, 192, 194, 198, 199, 206, 207,
  208, 209, 210, 219, 220, 227, 231, 232, 233, 235, 240, 242, 261, 265, 266, 268, 270, 272, 274,
  275, 278, 282, 294, 300, 302, 303, 304, 317, 325, 326, 328, 333, 334, 335, 336, 337, 338, 341,
  342, 355, 357, 358, 365, 366, 367, 368, 369, 372, 373, 377, 380, 381, 382, 385, 391, 394, 395,
  396, 397, 398

GitHub Actions / GitLab CI: 27 prompts
Question numbers: 54, 152, 153, 154, 161, 216, 234, 236, 248, 249, 250, 254, 255, 256, 257, 258,
  259, 260, 262, 263, 276, 277, 313, 314, 378, 379, 383

GitOps / Argo CD / Flux: 15 prompts
Question numbers: 46, 53, 60, 89, 112, 141, 144, 177, 183, 193, 287, 320, 388, 389, 393

Helm and deployment manifests: 25 prompts
Question numbers: 9, 27, 34, 37, 40, 73, 74, 76, 81, 96, 98, 155, 156, 176, 181, 191, 205, 311, 321,
  322, 323, 324, 340, 363, 375

COMPANY INDEX
------------------------------------------------------------------------------------------------
AMEX: 1 prompt (question 1)
Accion Labs: 2 prompts (questions 2-3)
Accolite: 1 prompt (question 4)
Alphadyne: 4 prompts (questions 5-8)
Altimetrik: 3 prompts (questions 9-11)
Amazon: 2 prompts (questions 12-13)
BMW TechWorks: 1 prompt (question 14)
CGI: 2 prompts (questions 15-16)
CTS (Cognizant): 3 prompts (questions 17-19)
Capgemini: 3 prompts (questions 20-22)
Cisco: 3 prompts (questions 23-25)
Deloitte: 16 prompts (questions 26-41)
EPAM: 16 prompts (questions 42-57)
EXL Service: 9 prompts (questions 58-66)
Elixr Labs: 4 prompts (questions 67-70)
Emphasis: 2 prompts (questions 71-72)
Encora: 4 prompts (questions 73-76)
F5: 1 prompt (question 77)
Flentas: 4 prompts (questions 78-81)
HCL: 13 prompts (questions 82-94)
Hexaware: 4 prompts (questions 95-98)
IBM: 2 prompts (questions 99-100)
Infosys: 12 prompts (questions 101-112)
Intact Green Services: 4 prompts (questions 113-116)
JPMorgan: 16 prompts (questions 117-132)
Koerber Pharma: 2 prompts (questions 133-134)
LTIMindtree: 11 prompts (questions 135-145)
L&T: 1 prompt (question 146)
Marsh McLennan: 11 prompts (questions 147-157)
Moodys: 4 prompts (questions 158-161)
NUOS INFO Systems: 7 prompts (questions 162-168)
NatWest Group: 5 prompts (questions 169-173)
Nextturn: 7 prompts (questions 174-180)
Nice: 8 prompts (questions 181-188)
Nisum Technologies: 3 prompts (questions 189-191)
Nitor Infotech: 4 prompts (questions 192-195)
OPT IT: 4 prompts (questions 196-199)
One2N: 1 prompt (question 200)
Oracle: 4 prompts (questions 201-204)
Others: 68 prompts (questions 205-272)
Perfios: 3 prompts (questions 273-275)
Persistent Systems: 2 prompts (questions 276-277)
Plansource ValueLabs: 1 prompt (question 278)
Qburst: 3 prompts (questions 279-281)
Rapidsoft: 2 prompts (questions 282-283)
RelevantZ: 1 prompt (question 284)
SAP: 7 prompts (questions 285-291)
Sigmoid: 18 prompts (questions 292-309)
Sonata Software: 5 prompts (questions 310-314)
Sony: 6 prompts (questions 315-320)
SquareOps: 9 prompts (questions 321-329)
Synechron: 13 prompts (questions 330-342)
TCS: 12 prompts (questions 343-354)
Turning: 2 prompts (questions 355-356)
UST: 3 prompts (questions 357-359)
Verizon: 3 prompts (questions 360-362)
Virtusa: 2 prompts (questions 363-364)
Volkswagen Group Digital: 4 prompts (questions 365-368)
Wikreate Media: 1 prompt (question 369)
Wipro: 20 prompts (questions 370-389)
ZS Associates: 4 prompts (questions 390-393)
Zensar: 4 prompts (questions 394-397)
ZopSmart: 3 prompts (questions 398-400)

================================================================================================
AMEX
================================================================================================

1. [Application deployment] How do you reduce downtime during deployments?


================================================================================================
Accion Labs
================================================================================================

2. [Application deployment] What is the deployment setup in your organization?

3. [Application deployment] Have you written Terraform code for deployments? If yes, can you explain the
    implementation?


================================================================================================
Accolite
================================================================================================

4. [General CI/CD] What is CI/CD? Explain it briefly.


================================================================================================
Alphadyne
================================================================================================

5. [General CI/CD] Explain your branching strategy in depth.
    - Why was this strategy chosen?
    - How does it support multiple environments, releases, and hotfixes?

6. [General CI/CD] A developer has written only the source code for a new application. As a DevOps
    engineer, how would you design CI/CD and deploy to DEV, QA, and PROD using best practices?

7. [Application deployment] Which tool should a company use to automate deployment and configuration
    management for virtual machines and containers?

8. [Application deployment] Which Ansible module would you use to install the NGINX web server on a new
    server?


================================================================================================
Altimetrik
================================================================================================

9. [Helm and deployment manifests] Walk through the contents of your Helm charts and explain how they
    are integrated.

10. [Application deployment] An application or microservice is being deployed to a pod in GKE across
    different environments. The deployment fails, and the pod enters an error state and terminates. How
    would you troubleshoot it?

11. [Application deployment] You have optimized Kubernetes deployment configurations. What was your
    role, and what changes did you make?


================================================================================================
Amazon
================================================================================================

12. [General CI/CD] What are the stages of CI/CD?

13. [General CI/CD] How do you build an image during CI, and how do you manage it?


================================================================================================
BMW TechWorks
================================================================================================

14. [General CI/CD] Have you set up any CI/CD pipelines on your own?


================================================================================================
CGI
================================================================================================

15. [Build tools and artifacts] [Unclear] [Missing material] A sample Dockerfile exercise was mentioned
    in the interview. (The source does not provide the exact exercise.)

16. [Build tools and artifacts] Write a sample multi-stage Dockerfile.


================================================================================================
CTS (Cognizant)
================================================================================================

17. [Build tools and artifacts] How did you reduce the sizes of Docker images?

18. [General CI/CD] Explain the CI/CD pipeline you use.

19. [Application deployment] How did you write deployment files for the microservices in your project,
    and how did you configure Services and Ingress?


================================================================================================
Capgemini
================================================================================================

20. [Application deployment] Explain a Kubernetes Deployment YAML file.

21. [Application deployment] Explain blue-green deployment.

22. [Application deployment] Explain canary deployment.


================================================================================================
Cisco
================================================================================================

23. [Application deployment] Write a playbook to deploy an Nginx server and ensure the service is
    started and enabled on boot. How would you manage secrets in Ansible?

24. [Application deployment] Write an Ansible playbook to install Apache on a virtual machine.

25. [Application deployment] [Unclear] If a deployment times out, what kind of API gateway have you
    used? (The relationship between the timeout and API gateway is unclear in the source.)


================================================================================================
Deloitte
================================================================================================

26. [General CI/CD] What is the purpose of a webhook, and how is it used in a CI/CD pipeline?

27. [Helm and deployment manifests] Explain the folder structure of a basic Helm chart. What commands do
    you use to deploy with Helm?

28. [Build tools and artifacts] What are the stages in a Docker image build? Why do we use ENTRYPOINT
    and CMD instructions?

29. [Build tools and artifacts] Which container registry do you use for storing Docker images?

30. [General CI/CD] What branching strategy do you follow, and how do you handle merges to avoid
    breaking the release branch? If a bug appears in production, what’s your approach to resolving it?

31. [Build tools and artifacts] Are you aware of security scanning tools? How do you scan Docker
    images—both during build and at the registry level? Are you using any extensions or tools for image
    scanning?

32. [Build tools and artifacts] How do you pass environment variables during Docker build commands? What
    services do you use for storing Docker images?

33. [Build tools and artifacts] How do you create AWS Lambda functions and manage the artifacts for
    deployment? What options do you use to push artifacts to Lambda?

34. [Helm and deployment manifests] [Relevant portion] What is Helm chart signing? Which tools do you
    use to sign Helm charts?

35. [General CI/CD] Which CI/CD tools do you use?

36. [Build tools and artifacts] A Dockerfile describes a Tomcat application running on port 8080. How
    would you build the image and run it as a container exposing port 9090?

37. [Helm and deployment manifests] What is the use of Helm charts?

38. [General CI/CD] Which CI/CD tools do you use?

39. [Build tools and artifacts] A Dockerfile describes a Tomcat application running on port 8080. How
    would you build the image and run it as a container exposing port 9090?

40. [Helm and deployment manifests] What is the use of Helm charts?

41. [Code quality and testing] How do you integrate SonarQube into your pipeline?


================================================================================================
EPAM
================================================================================================

42. [Application deployment] The development team changed the AMI configuration in an Auto Scaling Group
    launch template. How would you ensure the new version is deployed correctly?

43. [Build tools and artifacts] Explain AWS Image Builder.

44. [General CI/CD] How would you design a scalable, highly available CI/CD system for microservices
    across multiple teams?

45. [Application deployment] How would you manage cross-region deployments using Terraform in a
    multi-cloud setup?

46. [GitOps / Argo CD / Flux] How do you implement GitOps in a Kubernetes environment?

47. [Application deployment] Can you explain how you would create a fully automated blue-green
    deployment in a Kubernetes-based microservices architecture?

48. [General CI/CD] How do you design an end-to-end DevSecOps pipeline for a fintech application with
    strict compliance requirements (e.g., PCI-DSS)?

49. [General CI/CD] What are some best practices for managing pipeline as code in large, distributed
    teams?

50. [General CI/CD] How would you dynamically provision ephemeral environments (dev/test) using
    pipelines?

51. [General CI/CD] In a monorepo setup, how do you ensure that only relevant services are built and
    deployed in a CI/CD pipeline?

52. [General CI/CD] How do you implement a canary deployment strategy with real-time monitoring rollback
    in a CI/CD system?

53. [GitOps / Argo CD / Flux] How do you manage secrets and config securely at scale in Kubernetes
    without compromising GitOps workflows?

54. [GitHub Actions / GitLab CI] How do you set up workload identity federation between GitHub Actions
    and Google Cloud / Azure securely?

55. [General CI/CD] How do you ensure cost-efficient auto-scaling of infrastructure in cloud when
    managing high workloads in CI/CD?

56. [General CI/CD] How do you enforce compliance and auditability in your CI/CD processes across global
    regions (e.g., GDPR, HIPAA)?

57. [General CI/CD] What's your strategy for managing container image security across all stages of a
    DevOps pipeline?


================================================================================================
EXL Service
================================================================================================

58. [General CI/CD] Explain your complete CI/CD pipeline from code commit to production deployment.

59. [Application deployment] Explain your Git branching strategy. How do you deploy code from different
    branches to different environments?

60. [GitOps / Argo CD / Flux] If Git is already the source of truth, why do we need Argo CD? Why not
    deploy directly using the CI/CD pipeline with Helm or kubectl?

61. [General CI/CD] Suppose you are implementing a Canary deployment where only 10% of users receive the
    new version. How would you implement it through your CI/CD pipeline?

62. [Application deployment] During a Canary deployment, how would you verify that the 10% deployment is
    healthy? What metrics would you monitor before proceeding to 100%?

63. [General CI/CD] Do you execute Terraform locally or through a CI/CD pipeline? Explain the complete
    workflow.

64. [Application deployment] Explain Rolling Update, Blue-Green, and Canary deployment strategies.

65. [Application deployment] For a mission-critical production application, which deployment strategy
    would you choose and why?

66. [General CI/CD] Where do you store CI/CD secrets such as pipeline credentials?


================================================================================================
Elixr Labs
================================================================================================

67. [General CI/CD] Which types of pipelines do you handle, for example QA and UAT?

68. [Application deployment] What deployment model do you use: IaaS or SaaS?

69. [Application deployment] Choose one requirement and explain how you deployed it end to end using
    Azure services.

70. [General CI/CD] Describe a critical issue you encountered with a pipeline or production deployment.
    How did you resolve it?


================================================================================================
Emphasis
================================================================================================

71. [Build tools and artifacts] Explain how to write a Dockerfile.

72. [General CI/CD] Explain a CI/CD pipeline and its stages.


================================================================================================
Encora
================================================================================================

73. [Helm and deployment manifests] What are helm template and helm install?

74. [Helm and deployment manifests] What are helper files in Helm?

75. [General CI/CD] What branching strategies do you use, and how do you create the associated
    pipelines?

76. [Helm and deployment manifests] [Relevant portion] What files are present inside a Helm chart?


================================================================================================
F5
================================================================================================

77. [Application deployment] Have you deployed any security application on Kubernetes?


================================================================================================
Flentas
================================================================================================

78. [Application deployment] What applications are deployed in the frontend and backend?

79. [Application deployment] Why did you not deploy the frontend on S3 and CloudFront, and instead
    deployed it on EKS?

80. [Application deployment] If there is a problem in Terraform, how will you roll back the changes?

81. [Helm and deployment manifests] What is a Helm chart?


================================================================================================
HCL
================================================================================================

82. [Application deployment] How do you deploy to different environments using a Git repository?

83. [General CI/CD] [Context] What is a PAT?

84. [Code quality and testing] How do you configure SonarQube?

85. [Code quality and testing] What output does SonarQube produce, and how do you address code smells or
    vulnerabilities that it finds?

86. [Build tools and artifacts] What is inside a Dockerfile?

87. [Application deployment] What is a rolling update?

88. [Application deployment] Which deployment strategy do you use?

89. [GitOps / Argo CD / Flux] Have you worked with Argo CD and Helm?

90. [Application deployment] Suppose we have pods 2 running in rolling updates some are in deployments
    set and some pods are in statefull set, how rolling updates strategy will work here?

91. [General CI/CD] An EC2 instance was manually changed from t3.medium to t3.large, and the code was
    updated and committed, but the pipeline has not run. What would happen when the pipeline runs?

92. [Build tools and artifacts] Can you write a basic Dockerfile for your application?

93. [Application deployment] If you have an on-prem application, how would you migrate and deploy it in
    a cloud-native environment?

94. [Build tools and artifacts] Can you explain Docker Compose and how it helps in multi-container
    application deployments?


================================================================================================
Hexaware
================================================================================================

95. [Application deployment] What is deployment.yml?

96. [Helm and deployment manifests] What is the output of a Helm chart?

97. [Code quality and testing] What is SonarQube, and what is it used for?

98. [Helm and deployment manifests] What files are available inside a Helm chart?


================================================================================================
IBM
================================================================================================

99. [Application deployment] How do you deploy your application on AWS? What services do you use?

100. [Application deployment] When deploying your application to Amazon EKS, what other services do you
    use along with it?


================================================================================================
Infosys
================================================================================================

101. [Build tools and artifacts] Write a sample Dockerfile.

102. [Application deployment] Explain the blue-green deployment strategy.

103. [Application deployment] An application you are deploying to Kubernetes crashes, and you cannot
    enter the pod. How would you investigate?

104. [General CI/CD] Explain your project pipeline.

105. [Application deployment] Have you worked on production deployments?

106. [Application deployment] How frequently do you deploy to production in your current project?

107. [Build tools and artifacts] What is Docker, and how do you use it in your project? Have you written
    a Dockerfile?

108. [Build tools and artifacts] How do you reduce Docker image size?

109. [Build tools and artifacts] What difficulties have you faced while building a Docker image?

110. [General CI/CD] Which tools have you used for CI and CD pipelines?

111. [General CI/CD] Have you created and set up CI/CD pipelines? Explain a CD pipeline you created,
    including all stages and integrated tools.

112. [GitOps / Argo CD / Flux] How were you managing your Kubernetes cluster through a helm file or the
    command line? Have you used Rancher or Argo CD?


================================================================================================
Intact Green Services
================================================================================================

113. [Application deployment] How do you deploy an application in Kubernetes?

114. [Build tools and artifacts] A Docker image is causing issues because of its size. What steps would
    you take to reduce its size?

115. [General CI/CD] What are the tools you have used for CI/CD pipeline?

116. [Build tools and artifacts] You cannot push a Docker image to Docker Hub because of an access
    issue. Where else could you push the image?


================================================================================================
JPMorgan
================================================================================================

117. [Code quality and testing] How are you managing code coverage in your project?

118. [Application deployment] How do you implement blue-green deployment in your project?

119. [Application deployment] How do you deploy an application to AWS?

120. [Application deployment] How do you provision container-based services for microservices
    deployment?

121. [Application deployment] What is Serverless deployment in AWS?

122. [Application deployment] An application deployed to Azure Kubernetes Service (AKS) randomly fails
    health checks. How would you debug this end to end?

123. [Application deployment] In a canary deployment to production, half the traffic returns 502, while
    others succeed. Walk us through your troubleshooting approach.

124. [General CI/CD] CI/CD pipeline takes 40 mins to deploy a small change. What would you do to
    optimize it?

125. [Application deployment] During an Azure deployment, you encounter intermittent DNS resolution
    issues. What could cause them?

126. [Application deployment] How would you set up an automated rollback strategy in Kubernetes for
    failed deployments?

127. [General CI/CD] Define a plan for blue/green deployment with rollback on Azure using Terraform and
    pipelines.

128. [General CI/CD] Suppose your production pipeline is blocked due to missing approvals and
    stakeholders are unreachable. What will you do?

129. [Application deployment] Your application runs on EC2 instances in a public subnet. How would you
    migrate it to a private subnet without downtime? Explain the complete approach and how you would
    roll back if the migration fails.

130. [Application deployment] If an application has only one replica and you perform a rolling restart,
    will there be downtime? Also what events occur during the pod restart can you explain the sequence
    step by step?

131. [Application deployment] You have an application with 2 replicas. During a rollout, the first pod
    is successfully replaced, but when the second pod is being replaced it enters a CrashLoopBackOff
    state. At that moment, which pod will the load balancer route traffic to the new pod, the old pod,
    or both? explain

132. [Build tools and artifacts] Just like we use code-quality and security checks (quality gates,
    OWASP) before building an image, how can we prevent insecure infrastructure changes from being
    pushed using Terraform?


================================================================================================
Koerber Pharma
================================================================================================

133. [General CI/CD] How would you connect your Terraform environment to AWS and implement CI/CD?

134. [Application deployment] What strategies can you use to deploy an application?


================================================================================================
LTIMindtree
================================================================================================

135. [Code quality and testing] What is the difference between code quality and code coverage?

136. [Code quality and testing] What is the default quality gate in SonarQube?

137. [Application deployment] Explain deployment types.

138. [Build tools and artifacts] Explain the Dockerfile.

139. [Build tools and artifacts] Create three different container images, store them in ECR or ACR, and
    deploy them to EKS or AKS. Write the Kubernetes YAML files.

140. [General CI/CD] Create a manifest for two NGINX replicas.

141. [GitOps / Argo CD / Flux] How do you manage secrets securely in GitOps or deployment pipelines?

142. [Application deployment] How do you implement blue-green or canary deployments using container
    orchestration?

143. [General CI/CD] How do you implement rollback in an automated deployment pipeline?

144. [GitOps / Argo CD / Flux] How does your GitOps tool detect drift and how do you manage it?

145. [General CI/CD] How do you handle parallel execution in CI/CD workflows?


================================================================================================
L&T
================================================================================================

146. [Build tools and artifacts] What is a Dockerfile?


================================================================================================
Marsh McLennan
================================================================================================

147. [Build tools and artifacts] How do you reduce the size of a Docker image?

148. [Build tools and artifacts] What is a multi-stage Docker build? How does it help reduce image size?

149. [Build tools and artifacts] What is Docker image layer caching?

150. [Build tools and artifacts] How do you implement Docker image layer caching?

151. [Build tools and artifacts] Do you use any tool for Docker image layer caching? If yes, which one?

152. [GitHub Actions / GitLab CI] In GitHub Actions, if one job depends on another job, which parameter
    do you use?

153. [GitHub Actions / GitLab CI] How do you prevent concurrent executions in GitHub Actions?

154. [GitHub Actions / GitLab CI] What is the difference between needs and concurrency in GitHub
    Actions?

155. [Helm and deployment manifests] Walk me through the troubleshooting steps for a failed Helm
    deployment.

156. [Helm and deployment manifests] If a Helm release is partially deployed and some resources are
    updated while others have failed, how do you perform a rollback?

157. [General CI/CD] Where do you store application credentials in your CI/CD pipeline?


================================================================================================
Moodys
================================================================================================

158. [Application deployment] How do you set up infrastructure for deploying ML models using Terraform?

159. [Application deployment] If batch jobs are running for ML workloads, how do you handle deployments
    without impacting ongoing processing?

160. [General CI/CD] How do you design and implement a complete CI/CD pipeline for ML models?

161. [GitHub Actions / GitLab CI] [Missing material] You are given a GitHub Actions workflow snippet.
    How would you identify incorrect steps and suggest improvements or missing steps for a robust CI/CD
    pipeline? (The referenced workflow snippet is not included in the attachment.)


================================================================================================
NUOS INFO Systems
================================================================================================

162. [General CI/CD] How would you scale a Terraform pipeline that takes more than 25 minutes?

163. [General CI/CD] If the pipeline fails because resources already exist, how would you handle RIP:
    Remove, Import, Plan?

164. [General CI/CD] What are the best practices for structuring repositories and pipelines in a large
    DevOps project?

165. [General CI/CD] Pipeline fails only on Tuesdays, no code changes — how do you debug?

166. [Build tools and artifacts] Write a multi-stage Dockerfile for a Node.js application, removing
    secrets and unnecessary layers.

167. [General CI/CD] Which tools would you recommend for CI/CD, artifact storage, vulnerability
    scanning, and container registry in a hybrid on-premises and Azure setup?

168. [General CI/CD] How do you manage AWS and Azure through a single DevOps process, with a focus on
    security and cost?


================================================================================================
NatWest Group
================================================================================================

169. [Build tools and artifacts] Explain Maven release.

170. [Build tools and artifacts] Explain the Maven lifecycle.

171. [Build tools and artifacts] Explain the dependency-management tag in Maven.

172. [Build tools and artifacts] What happens when you run maven install?

173. [Build tools and artifacts] Where is your pom.xml file located?


================================================================================================
Nextturn
================================================================================================

174. [Application deployment] Explain Continuous Integration, Continuous Delivery, and Continuous
    Deployment.

175. [General CI/CD] Explain an end-to-end CI/CD pipeline.

176. [Helm and deployment manifests] A Helm upgrade fails. How would you roll it back and troubleshoot
    it?

177. [GitOps / Argo CD / Flux] Compare push-based and pull-based deployment in GitOps.

178. [Application deployment] Difference between Continuous Delivery and Continuous Deployment.

179. [Build tools and artifacts] Explain Docker layer caching. During a Docker build, if layers 1–10 are
    already cached and you modify Layer 5, what happens to Layers 6–10? Will Docker reuse the cache or
    rebuild them? Explain why.

180. [General CI/CD] Explain the pre-build, build, and post-build stages in a CI/CD pipeline.


================================================================================================
Nice
================================================================================================

181. [Helm and deployment manifests] How do you manage Helm charts?

182. [General CI/CD] Explain CI/CD.

183. [GitOps / Argo CD / Flux] How do you maintain Argo CD for the E1, E2, and E3 environments?

184. [General CI/CD] You mentioned improving CI/CD efficiency by 60%. Can you explain the specific
    optimizations you made for this?

185. [Code quality and testing] What is your approach to integrating automated testing in pipelines to
    ensure high code quality?

186. [Code quality and testing] How do you integrate tools like SonarQube into your pipelines?

187. [Application deployment] In Kubernetes, how do you manage application deployment, scaling, and
    rollback? Can you walk through a specific scenario?

188. [General CI/CD] What is a recent challenge you faced while implementing a DevOps practice or
    pipeline in your team or organization?


================================================================================================
Nisum Technologies
================================================================================================

189. [General CI/CD] Explain your CI/CD pipeline.

190. [Application deployment] Which deployment strategy do you follow?

191. [Helm and deployment manifests] Have you worked with Helm charts?


================================================================================================
Nitor Infotech
================================================================================================

192. [General CI/CD] Explain the CI/CD setup and flow in your current project.

193. [GitOps / Argo CD / Flux] Explain the GitOps approach, Argo CD, and Flux.

194. [General CI/CD] How would you reduce pipeline runtime? Discuss multistage builds.

195. [Build tools and artifacts] Create a Deployment named web-app using image nginx:1.25, with 3
    replicas, container port 80, and the label app=web. Then write a NodePort Service to expose the
    application on port 8080.


================================================================================================
OPT IT
================================================================================================

196. [Application deployment] If databases are in private subnets, how do you deploy the application in
    Kubernetes?

197. [Build tools and artifacts] Write a Dockerfile for a Java application.

198. [General CI/CD] Which CI/CD tools have you worked with?

199. [General CI/CD] Which types of pipelines have you worked with?


================================================================================================
One2N
================================================================================================

200. [Build tools and artifacts] Design a three-tier application architecture, considering Docker Swarm
    or Kubernetes.
    - What approach and set of tools would you choose?
    - What kind of database would you use?
    - How would you manage the microservices?
    - How would you expose the application?
    - Is a load balancer required in this setup? Why?
    - How would a reverse proxy work in this setup?
    - How would you set up DNS?
    - How would you set up CI/CD for the application?
    - Is NGINX required in this setup?
    - How would you manage SSL and TLS?
    - How would you update the image and deploy it?
    - How would you set up alerting?
    - How would you determine how many users are affected?
    - Which metric would tell you whether the application is up or down?


================================================================================================
Oracle
================================================================================================

201. [Build tools and artifacts] Explain all the steps involved in building a multi-stage Docker image.

202. [Build tools and artifacts] What layers are created when building a Docker image?

203. [Application deployment] A pod deployment fails with an error. How would you investigate?

204. [Application deployment] Explain all the components in a deployment.yaml file.


================================================================================================
Others
================================================================================================

205. [Helm and deployment manifests] Which Helm commands do you use to deploy an application, and how do
    you integrate the process into CI/CD?

206. [General CI/CD] How do you provision infrastructure using Terraform through CI/CD?

207. [General CI/CD] What kind of CI/CD pipelines are you familiar with?

208. [General CI/CD] What are your organization's current CI/CD process and tools?

209. [General CI/CD] About K8's Architecture and tell me the workflow?

210. [General CI/CD] Which security measures or tools have you included in your CI/CD pipeline?

211. [Build tools and artifacts] How do you roll back a failed deployment in Docker and Kubernetes?

212. [Application deployment] I have created a service object that is not mapped to a deployment. What
    could be the reason and how do you debug it?

213. [Build tools and artifacts] Write a deployment for three replicas running the Apache httpd
    container image.

214. [Build tools and artifacts] How do you reduce the size of a Dockerfile?

215. [Build tools and artifacts] What is the difference between COPY and ADD command in Docker File.

216. [GitHub Actions / GitLab CI] Explain a GitHub Actions workflow file.

217. [Build tools and artifacts] Difference between entry point and CMD in Docker File.

218. [Build tools and artifacts] Build a container image and push it to ACR. How would you reference
    that image in a Kubernetes YAML file to deploy a pod?

219. [General CI/CD] What are your organization's current CI/CD process and tools?

220. [General CI/CD] What do you know about CyberArk, and how do you use it in your pipeline?

221. [Application deployment] Write a Deployment manifest.

222. [Build tools and artifacts] Write a Dockerfile.

223. [Build tools and artifacts] Explain a multi-stage Dockerfile.

224. [Application deployment] Have you worked with Kubernetes? Which deployment strategy do you follow?

225. [Application deployment] How do you implement blue-green deployment?

226. [Application deployment] After deploying an application, you discover an issue. How do you roll
    back to a particular version in Kubernetes, and what command do you use?

227. [General CI/CD] CI/CD pipeline needs rollback capability. How would you implement it?

228. [Build tools and artifacts] What is a Dockerfile, and what does it contain?

229. [Application deployment] Explain blue-green deployment using an example from your project.

230. [Application deployment] How do canary and blue-green deployments differ?

231. [General CI/CD] Explain a project in which you used Docker, Kubernetes, and CI/CD.

232. [General CI/CD] Describe the CI/CD tool or process you use in detail.

233. [General CI/CD] Explain a CI/CD pipeline in detail.

234. [GitHub Actions / GitLab CI] [Context] Why would you use a self-hosted runner instead of the
    default runner?

235. [General CI/CD] Which CI/CD system do you use for Terraform infrastructure in your project?

236. [GitHub Actions / GitLab CI] [Unclear] [Context] How do IAM users, a GitHub OIDC role, and a
    "terraform io role" differ in security? When would you use a GitHub OIDC role, and when would you
    use a "terraform io role"? (The source does not define "terraform io role".)

237. [Build tools and artifacts] Write a simple Dockerfile.

238. [Build tools and artifacts] What is the difference between ADD and COPY in a Dockerfile?

239. [Build tools and artifacts] What is the difference between CMD and ENTRYPOINT in a Dockerfile?

240. [General CI/CD] How do you secure secrets and credentials in your CI/CD process?

241. [Build tools and artifacts] Write a Dockerfile and explain it.

242. [General CI/CD] Suppose you have created a CI/CD process. After building the image, manual
    intervention is required. How would you configure it, and where?

243. [Application deployment] What is Blue-Green Deployment, and what is Canary Deployment? Can you
    explain the difference between them?

244. [Application deployment] Let's say, A critical production deployment failed and caused downtime.
    How would you handle the situation?

245. [Build tools and artifacts] What is artifact management, and which tool do you use in your
    organization?

246. [Build tools and artifacts] How do you reduce the size of a Docker image?

247. [Build tools and artifacts] Explain the Maven lifecycle.

248. [GitHub Actions / GitLab CI] Write a GitHub/GitLab pipeline to deploy a microservice with three
    services running in parallel.

249. [GitHub Actions / GitLab CI] [Context] What is runs-on in a pipeline? Which type of runners does
    your organization use, and how do you configure self-hosted runners?

250. [GitHub Actions / GitLab CI] [Context] How do you configure the pipeline triggers for the following
    cases?
    - Trigger when code is pushed to a specific branch.
    - Ignore pushes to other branches.
    - Trigger when a pull request is raised.

251. [Build tools and artifacts] What is a base image in a Dockerfile?

252. [Build tools and artifacts] Can we write a Dockerfile without a base image?

253. [Code quality and testing] What is SonarQube, and why is it used?

254. [GitHub Actions / GitLab CI] How do you set up a manual trigger in GitHub Actions?

255. [GitHub Actions / GitLab CI] How do you set up GitHub runners for the application environment?

256. [GitHub Actions / GitLab CI] [Follow-up] If you use GitHub Marketplace actions, which are
    third-party tools, how do you address the security concerns associated with them?

257. [GitHub Actions / GitLab CI] What is a matrix in GitHub Actions?

258. [GitHub Actions / GitLab CI] What is the needs keyword in GitHub Actions?

259. [GitHub Actions / GitLab CI] Write the structure for building and pushing a Docker image for an
    application in GitHub Actions.

260. [GitHub Actions / GitLab CI] How do you run jobs in parallel in GitHub Actions?

261. [General CI/CD] How do you handle secrets in your project?

262. [GitHub Actions / GitLab CI] What steps are included in your GitHub Actions workflow file?

263. [GitHub Actions / GitLab CI] [Context] How is static code analysis configured in your pipeline
    file?

264. [Code quality and testing] Which is the optimized method to run SonarQube: for every PR raised or
    every push?

265. [General CI/CD] What are webhooks, and have you used them anywhere?

266. [General CI/CD] How do you configure a pipeline with AWS or Docker?

267. [Application deployment] Tell me about a time you handled a failed deployment in production. How
    did you manage the team and stakeholders?

268. [General CI/CD] How do you prioritize and manage multiple critical issues in a CI/CD pipeline
    failure?

269. [Application deployment] Explain a situation where you were responsible for reducing deployment
    time. What approach did you take?

270. [General CI/CD] Have you ever dealt with a security vulnerability in your DevOps pipeline? How did
    you detect and respond to it?

271. [Application deployment] Describe how you handled a rollback situation during a major release. What
    went wrong, and what did you learn?

272. [General CI/CD] Explain how you build a CI/CD pipeline.


================================================================================================
Perfios
================================================================================================

273. [Application deployment] Compare canary and blue-green deployments.

274. [General CI/CD] When you deploy from CI, you build a package and then need a platform to deploy the
    application. How do you build that platform, and if it requires human intervention, how do you
    eliminate that dependency?

275. [General CI/CD] In GitHub, after a code commit, what validations do you perform? Do you validate
    before the commit or after the commit?


================================================================================================
Persistent Systems
================================================================================================

276. [GitHub Actions / GitLab CI] What is the matrix strategy in GitHub Actions?

277. [GitHub Actions / GitLab CI] How does caching work in GitHub Actions?


================================================================================================
Plansource ValueLabs
================================================================================================

278. [General CI/CD] How do you manage Terraform pipeline variables for different environments, such as
    dev, live, and feature?


================================================================================================
Qburst
================================================================================================

279. [Build tools and artifacts] What is the difference between ADD and COPY in a Dockerfile?

280. [Build tools and artifacts] How do you reduce the Docker build size?

281. [Build tools and artifacts] How do you pass a value while building a Docker image?


================================================================================================
Rapidsoft
================================================================================================

282. [General CI/CD] You have a multi-cloud environment. How do you manage pipelines for all those cloud
    environments?

283. [Application deployment] Explain the fields in a Kubernetes deployment.yml file.


================================================================================================
RelevantZ
================================================================================================

284. [Application deployment] You have two environments. Infrastructure should be deployed with
    Terraform to one environment, while nothing should be deployed to the other. What strategy would you
    use?


================================================================================================
SAP
================================================================================================

285. [Application deployment] A new deployment is performed, and suddenly both old and new pods crash.
    What could cause this?

286. [Application deployment] Which deployment strategy is better from a cost perspective?

287. [GitOps / Argo CD / Flux] How would you use Argo CD to deploy only to specific workloads or
    regions, without updating other workloads or regions?

288. [Application deployment] Can you perform a blue-green deployment within the same namespace? If so,
    how would you manage it?

289. [Application deployment] After a canary deployment, when would you delete the old pods? Which KPIs
    would you check first?

290. [Application deployment] After completing a blue-green deployment, how do you verify that it
    succeeded?

291. [Build tools and artifacts] How will you maintain your base image, vulnerability free?


================================================================================================
Sigmoid
================================================================================================

292. [Application deployment] How do you handle deployment failures?

293. [Application deployment] Write a Deployment manifest with the following requirements.
    Requirements:
    - Deployment name: space-alien-welcome-message-generator
    - Image: httpd:alpine
    - Replicas: 1
    - Readiness probe command: stat /tmp/ready; the pod should become ready when the file exists.
    - initialDelaySeconds: 10
    - periodSeconds: 5

294. [General CI/CD] A developer accidentally hardcodes a password in source code and pushes it to a
    repository. How would you address this in the CI process?

295. [Application deployment] What would you do if a Terraform deployment failed?

296. [Application deployment] Which deployment strategy would you choose when only one pod is running?

297. [Application deployment] During a blue-green deployment, which exact configuration would you change
    to reroute traffic between blue and green?

298. [Application deployment] A new web deployment was completed today. How would you verify application
    behavior and detect issues before users notice them?

299. [Build tools and artifacts] How do you create a sub-user while writing a Dockerfile?

300. [General CI/CD] Explain the flow from dev to production and the security checks in your current
    project's CI pipeline.

301. [Application deployment] [Relevant portion] Design an AWS three-tier application with frontend,
    backend, and database, following security best practices and providing high availability and low
    latency.
    - How would you choose between a StatefulSet and RDS or another managed database?
    - How would you decide whether to deploy the frontend as a pod or use S3 with CloudFront?

302. [General CI/CD] How do you handle CI integration for dev, test, QA, staging, and production
    environments in your current project?

303. [General CI/CD] Which branching strategy do you follow, and how do you integrate it into your CI
    pipeline to deploy pushed code to the correct environment cluster? Where in the CI code do you
    handle deployment to dev, QA, and subsequent environments up to production? How do you handle PR
    checks and approvals?

304. [General CI/CD] What activities occur after deployment to each environment and before the
    application is ready for the next environment?

305. [Build tools and artifacts] If there was an issue introduced in recent deployment, then how do we
    rollback the deployment, is it just the kubectl rollout undo command, how does it know which image
    it should revert to,does it take from the image repo or somewhere else?

306. [Build tools and artifacts] [Missing material] Write a three-stage Dockerfile using the pre-built
    images supplied for each stage in the interview. [The attached text does not provide those image
    names.]
    Requirements:
    - Stage 1: Set and copy environment configuration files, environment variables, and shell profiles
      such as .bashrc and .bashprofile from the supplied image to the second stage.
    - Stage 2: Use the files from the previous stage, perform prerequisite tasks, and build the code
      from the current directory.
    - Stage 3: Use the output from the previous stage to build the final image.

307. [Build tools and artifacts] Explain how image creation works, how layers are formed, and what the
    image produced by the final stage contains.

308. [Build tools and artifacts] Which steps or commands in a Dockerfile create intermediate images? Are
    those intermediate images still used after the final image is created?

309. [Build tools and artifacts] What happens if a Dockerfile contains multiple CMD and ENTRYPOINT
    instructions? Will the build fail? If it succeeds, how does Docker handle those instructions?


================================================================================================
Sonata Software
================================================================================================

310. [Application deployment] How long would it take you to write infrastructure-as-code and deploy an
    application to Azure App Service?

311. [Helm and deployment manifests] Do you use Helm charts for AKS deployments?

312. [Application deployment] Say you have EC2 instances running web servers and need deployment with
    minimal downtime during updates. How do you approach this?

313. [GitHub Actions / GitLab CI] Have you worked on GitLab?

314. [GitHub Actions / GitLab CI] What are rules in GitLab?


================================================================================================
Sony
================================================================================================

315. [Application deployment] How do you guarantee zero-downtime deployments in Kubernetes?

316. [Application deployment] How do you decide to roll back or apply a hot‑fix when a production issue
    occurs?

317. [General CI/CD] How can you shorten a CI/CD pipeline that currently takes 45 minutes?

318. [Application deployment] How do you guarantee zero-downtime deployments in Kubernetes?

319. [Code quality and testing] What vulnerability reports are available in SonarQube?

320. [GitOps / Argo CD / Flux] How do you design GitOps for 1000+ clusters with environment drift
    detection, emergency hotfixes, and controlled manual overrides?


================================================================================================
SquareOps
================================================================================================

321. [Helm and deployment manifests] Have you worked with Helm and Helm Charts?
    - Why use Helm instead of plain YAML?
    - Show the folder structure of a Helm chart.
    - What is the templates folder?
    - What is values.yaml used for?
    - How do you manage multiple environments?

322. [Helm and deployment manifests] How do you securely inject sensitive data into Helm?
    - Do you use AWS Secrets Manager?
    - Do you avoid committing secrets in values.yaml?
    - What is Sealed Secrets?
    - What is the purpose of --set and --set-file flags?

323. [Helm and deployment manifests] Can a public Helm chart be customized?
    - Why is it not recommended to edit the chart directly?
    - How do you update config safely?
    - What happens during chart upgrades?
    - How do you extend a chart with new templates?

324. [Helm and deployment manifests] How do you add extra Kubernetes manifest files to a public Helm
    chart?
    - Where do you put extra YAML files?
    - How do you reference new values?
    - Can this break the original chart?

325. [General CI/CD] What CI/CD tools have you worked with?
    - Explain your GitHub Actions pipeline.
    - How do you run iOS build automation?
    - How do you handle secrets in pipelines?
    - How do you deploy to EKS through GitHub Actions?

326. [General CI/CD] How do you handle pipelines for multiple environments?
    - How do you handle the Dev → QA → Prod flow?
    - What is your promotion strategy? Do you use manual approvals?
    - What is your Git branching strategy?

327. [Application deployment] How do you implement rolling deployments?
    - What happens to old pods?
    - What is maxSurge, maxUnavailable?

328. [General CI/CD] Have you ever set up rollback in CI/CD?
    - How do you implement automatic rollback?
    - What triggers a rollback?
    - Is rollback handled by CI/CD or Kubernetes?

329. [Code quality and testing] Have you integrated SonarQube in your CI/CD pipeline?
    - How to get Sonar token?
    - Where to store token?
    - How to insert Sonar scanner stage?
    - What is quality gate?


================================================================================================
Synechron
================================================================================================

330. [Build tools and artifacts] What is pom.xml in Maven?

331. [Build tools and artifacts] Share your screen and write a simple Dockerfile.

332. [Application deployment] [Unclear] Would you register App Service first or deploy it? (The source
    does not specify what registration means in this scenario.)

333. [General CI/CD] What is CI/CD?

334. [General CI/CD] What CI/CD challenges has your team faced, and how did you overcome them?

335. [General CI/CD] Can you explain what CI/CD is and describe how you have implemented CI/CD pipelines
    in one of your projects?

336. [General CI/CD] How do you configure your CI/CD pipelines? Can you walk me through the steps you
    followed to set up a pipeline in a recent project?

337. [General CI/CD] If you need to deploy an application to both cloud environments and on-premises
    servers (hybrid environment), how would you design and configure your pipeline to handle this?

338. [General CI/CD] Where and how do you manage environment variables for your applications in a CI/CD
    setup? Can you explain how you write and use them securely?

339. [Application deployment] After an application is deployed, what post-deployment steps do you
    typically perform to ensure everything is running smoothly?

340. [Helm and deployment manifests] What is Helm, and why do you prefer to use it for managing
    Kubernetes applications instead of deploying them normally?

341. [General CI/CD] If you want to use the feature branch instead of the main branch, how would you
    design the CI/CD pipeline?

342. [General CI/CD] Who handles pipeline failures, and how are they troubleshot?


================================================================================================
TCS
================================================================================================

343. [Build tools and artifacts] What is JFrog Artifactory?

344. [Build tools and artifacts] What is JFrog Xray?

345. [Build tools and artifacts] What are the use cases for JFrog?

346. [Build tools and artifacts] What is the difference between a GitHub repository and JFrog?

347. [Build tools and artifacts] What are the different repository types in JFrog?

348. [Application deployment] Which Kubernetes deployment strategies do you use? Explain canary and
    blue-green strategies.

349. [Build tools and artifacts] What is DevSecOps? Have you used tools to scan container images?

350. [Application deployment] If a rollback fails, how will you handle it?

351. [Application deployment] What is the command for rolling back to a specific revision in Kubernetes?

352. [Build tools and artifacts] Difference between COPY and ADD commands in a Dockerfile.

353. [Build tools and artifacts] If a Docker image becomes very large with many layers, what steps would
    you take to reduce its size?

354. [Build tools and artifacts] If you have 10 layers in a Dockerfile and layer 6 fails, after fixing
    it, where will the rebuild start from and why?


================================================================================================
Turning
================================================================================================

355. [General CI/CD] How do you implement Terraform in a CD pipeline?

356. [Application deployment] What is the purpose of blue-green deployment, and how do you switch back
    between deployments?


================================================================================================
UST
================================================================================================

357. [General CI/CD] If a CI pipeline is taking 45 minutes to run, how can you optimize it?

358. [General CI/CD] If credentials are visible in CI/CD pipeline logs, what will you do?

359. [Application deployment] Explain Blue-Green deployment.


================================================================================================
Verizon
================================================================================================

360. [Application deployment] What are the disadvantages of the different Kubernetes deployment models?

361. [Build tools and artifacts] How do you ensure Docker container security while writing a Dockerfile?

362. [Build tools and artifacts] How do you delete old or untagged images in ECR?


================================================================================================
Virtusa
================================================================================================

363. [Helm and deployment manifests] What files are present in a Helm chart?

364. [Build tools and artifacts] Provide the Docker commands to build an image, tag it, and push it to
    Docker Hub.


================================================================================================
Volkswagen Group Digital
================================================================================================

365. [General CI/CD] How do you onboard a new project from source code through release management across
    environments such as dev, QA, and production?

366. [General CI/CD] How do you automate all the CI and CD steps using multiple tools?

367. [General CI/CD] How do you write CI/CD pipelines?

368. [General CI/CD] Explain a CI/CD process you have worked on, including all its steps.


================================================================================================
Wikreate Media
================================================================================================

369. [General CI/CD] What is the role of continuous integration?


================================================================================================
Wipro
================================================================================================

370. [Application deployment] A file used by two customers needs to be deployed to both a Kubernetes
    cluster and an on-premises environment. How would you do this?

371. [Build tools and artifacts] What problems arise from using a large image in a Dockerfile?

372. [General CI/CD] How do you deploy an application to a Kubernetes cluster? Explain the CD part of
    the process.

373. [General CI/CD] How is Azure Key Vault integrated into CI/CD?

374. [Build tools and artifacts] Explain the contents of a Dockerfile.

375. [Helm and deployment manifests] Explain the contents of a deployment.yaml file or Helm chart.

376. [Application deployment] Could you elaborate your experience with automating and optimizing the
    deployment over large infrastructure using AWS and other tools like Terraform and Ansible from your
    previous roles?

377. [General CI/CD] How about your experience developing CI/CD pipeline and utilizing tools such as
    Docker, Grafana and Prometheous. Share a particular project where these skills were critical.

378. [GitHub Actions / GitLab CI] How would you structure a multistage GitHub Actions pipeline that
    builds, tests, and deploys a containerized application to Kubernetes?

379. [GitHub Actions / GitLab CI] [Context] How would you parameterize a workflow so that downstream
    jobs know which environment to deploy to?

380. [General CI/CD] How would you implement feature toggles in Deployment pipelines?

381. [General CI/CD] Explain your CI/CD pipeline.

382. [General CI/CD] Which policies have you used in your CI/CD pipeline?

383. [GitHub Actions / GitLab CI] What is the difference between GitHub Actions and Argo CD?

384. [Application deployment] A Kubernetes deployment succeeds, but the application is inaccessible
    externally. How would you troubleshoot it?

385. [General CI/CD] Write a YAML CI/CD pipeline from scratch to test and deploy an application from dev
    to UAT.

386. [Build tools and artifacts] What is Maven? Explain its repositories.

387. [Application deployment] How would you deploy an application to 100 servers using Ansible?

388. [GitOps / Argo CD / Flux] What is Argo CD, and why do we use it?

389. [GitOps / Argo CD / Flux] What is GitOps?


================================================================================================
ZS Associates
================================================================================================

390. [Build tools and artifacts] In which scenarios is a multi-stage Docker build useful? Is it suitable
    for compiled languages?

391. [General CI/CD] Explain layer caching with an example.

392. [Application deployment] Can you deploy mongo db database in EKS cluster. If yes how and what all
    configuration things you would need to keep in mind

393. [GitOps / Argo CD / Flux] [Relevant portion] Explain Argo CD and how you manage CI/CD in your
    organization.


================================================================================================
Zensar
================================================================================================

394. [General CI/CD] Which pipeline types do you use, and what types are available?

395. [General CI/CD] What is a variable in a pipeline?

396. [General CI/CD] Can you run multiple jobs in parallel within a single pipeline? How?

397. [General CI/CD] [Context] What is an agent in the pipeline context?


================================================================================================
ZopSmart
================================================================================================

398. [General CI/CD] Explain a CI/CD pipeline.

399. [Build tools and artifacts] Write a Dockerfile and explain its keywords.

400. [Build tools and artifacts] What is the Maven lifecycle?
