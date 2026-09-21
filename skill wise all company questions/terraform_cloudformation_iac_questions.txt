TERRAFORM, CLOUDFORMATION, AND RELATED IaC INTERVIEW QUESTIONS
================================================================================================

Source: interview questions company wise.rtf
Extracted: 276 numbered prompts, with 18 nested follow-ups.
Organization: 58 named companies plus the source group "Others".

ARM templates: No explicit questions found in the supplied attachment.
Azure Bicep: No explicit questions found in the supplied attachment.

Questions are grouped by company and labeled by topic. Terraform questions
include Azure, AWS, GCP, multi-cloud, CI/CD, state, modules, security, and
provisioning scenarios. Comparisons with CloudFormation, Terragrunt, and
Ansible, along with related IaC and AWS CDK prompts, are labeled separately.

Wording and spelling have been lightly cleaned; no answers have been added.
Repeated source entries are retained. Counts refer to numbered prompts, not
unique questions or every individual subquestion. Coding requirements and
relevant scenario follow-ups remain attached to the original prompt.

[Context] identifies a tool or scenario inferred from nearby questions.
[Relevant portion] marks the requested part of a mixed source question.
[Topic-only] marks a topic recorded without a full question.
[Unclear], [Incomplete], and [Missing diagram] preserve gaps in the source.

TOPIC INDEX
------------------------------------------------------------------------------------------------
Terraform: 256 prompts
Question numbers: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24,
  25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 39, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50,
  51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74,
  75, 76, 77, 78, 79, 80, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99,
  100, 101, 102, 103, 104, 105, 106, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119,
  120, 121, 122, 125, 126, 127, 128, 129, 130, 131, 132, 134, 135, 136, 137, 138, 139, 140, 141,
  142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 156, 157, 158, 159, 160, 161,
  162, 163, 164, 165, 166, 167, 168, 169, 170, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180,
  181, 182, 183, 184, 188, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 200, 201, 202, 203,
  204, 205, 206, 207, 208, 209, 210, 211, 212, 213, 214, 215, 216, 217, 218, 220, 221, 222, 223,
  224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 243,
  244, 246, 247, 248, 250, 251, 252, 253, 254, 255, 256, 257, 258, 259, 262, 263, 264, 265, 267,
  268, 269, 270, 271, 272, 273, 274, 275, 276

Terraform / CloudFormation: 5 prompts
Question numbers: 38, 40, 185, 186, 260

CloudFormation: 4 prompts
Question numbers: 155, 187, 242, 245

Terraform / Terragrunt: 1 prompt
Question numbers: 81

Terraform / Ansible: 2 prompts
Question numbers: 133, 261

Related IaC: 7 prompts
Question numbers: 22, 107, 123, 124, 199, 219, 266

Related AWS CDK: 1 prompt
Question numbers: 249

COMPANY INDEX
------------------------------------------------------------------------------------------------
Accenture: 3 prompts (questions 1-3)
Accion Labs: 2 prompts (questions 4-5)
Alphadyne: 1 prompt (question 6)
Altimetrik: 4 prompts (questions 7-10)
Amazon: 3 prompts (questions 11-13)
Aspire: 3 prompts (questions 14-16)
Belcan: 2 prompts (questions 17-18)
CMT: 1 prompt (question 19)
CTS (Cognizant): 6 prompts (questions 20-25)
Capgemini: 5 prompts (questions 26-30)
Cisco: 5 prompts (questions 31-35)
Deloitte: 8 prompts (questions 36-43)
EPAM: 20 prompts (questions 44-63)
EXL Service: 3 prompts (questions 64-66)
E&Y: 1 prompt (question 67)
Encora: 5 prompts (questions 68-72)
F5: 3 prompts (questions 73-75)
Five9: 4 prompts (questions 76-79)
Flentas: 10 prompts (questions 80-89)
Hexaware: 2 prompts (questions 90-91)
IBM: 8 prompts (questions 92-99)
ITC Infotech: 1 prompt (question 100)
Infosys: 5 prompts (questions 101-105)
HCL: 2 prompts (questions 106-107)
Intact Green Services: 1 prompt (question 108)
JPMorgan: 4 prompts (questions 109-112)
Koerber Pharma: 7 prompts (questions 113-119)
LTIMindtree: 6 prompts (questions 120-125)
Moodys: 1 prompt (question 126)
NUOS INFO Systems: 5 prompts (questions 127-131)
Nextturn: 3 prompts (questions 132-134)
Nice: 4 prompts (questions 135-138)
Nisum Technologies: 2 prompts (questions 139-140)
Nitor Infotech: 3 prompts (questions 141-143)
OPT IT: 1 prompt (question 144)
Optum: 6 prompts (questions 145-150)
Oracle: 2 prompts (questions 151-152)
Others: 34 prompts (questions 153-186)
Persistent Systems: 1 prompt (question 187)
Plansource ValueLabs: 1 prompt (question 188)
Qburst: 3 prompts (questions 189-191)
Qentelli Solutions: 2 prompts (questions 192-193)
Rapidsoft: 1 prompt (question 194)
RelevantZ: 4 prompts (questions 195-198)
Sapient: 7 prompts (questions 199-205)
Sigmoid: 10 prompts (questions 206-215)
Sonata Software: 10 prompts (questions 216-225)
Sony: 5 prompts (questions 226-230)
SquareOps: 6 prompts (questions 231-236)
Syncortex: 2 prompts (questions 237-238)
Synechron: 4 prompts (questions 239-242)
TCS: 4 prompts (questions 243-246)
Techdome: 2 prompts (questions 247-248)
Turning: 3 prompts (questions 249-251)
UST: 5 prompts (questions 252-256)
Virtusa: 4 prompts (questions 257-260)
Wipro: 7 prompts (questions 261-267)
ZS Associates: 6 prompts (questions 268-273)
ZopSmart: 3 prompts (questions 274-276)

================================================================================================
Accenture
================================================================================================

1. [Terraform] What happens when you run terraform init?

2. [Terraform] Write Terraform code to create EC2 instances in multiple AWS Regions.

3. [Terraform] You have defined a multi-region Terraform configuration (region1, region2, region3). If
    you create an EC2 instance, in which region will it be deployed?


================================================================================================
Accion Labs
================================================================================================

4. [Terraform] Have you written Terraform code for deployments? If yes, can you explain the
    implementation?

5. [Terraform] Why do we use workspaces in Terraform?


================================================================================================
Alphadyne
================================================================================================

6. [Terraform] Write Terraform code for the following Azure infrastructure.
    Requirements:
    - Define an Azure Virtual Network (VNet) using the given IP range. The source does not specify the
      actual range.
    - Create separate subnets for the bastion, application, and database layers.
    - Configure network interfaces (NICs) and attach them to the correct subnets.
    - Provision virtual machines using those NICs, with username/password authentication.
    - Create public IP addresses where required.
    - Set up an Azure Bastion host for secure access to the VMs.
    - Configure an Azure Load Balancer to route traffic to the application VMs.


================================================================================================
Altimetrik
================================================================================================

7. [Terraform] How do you import a resource into Terraform that was created manually in AWS or GCP? What
    command would you use?

8. [Terraform] How do you use Terraform to deploy cluster nodes?

9. [Terraform] When setting up nodes with Terraform, what do you write in the provider file and main.tf?

10. [Terraform] Using the AWS provider, write Terraform code to create a VPC and a subnet associated
    with that VPC. You may choose either a public or a private subnet.


================================================================================================
Amazon
================================================================================================

11. [Terraform] Write Terraform code to create a VPC, subnet, EC2 instance, and S3 bucket.

12. [Terraform] What is the difference between locals and variables in Terraform?

13. [Terraform] You created resources using Terraform. How would you ensure that they are not modified
    through the UI, and how would you automate that check?


================================================================================================
Aspire
================================================================================================

14. [Terraform] What is terraform fmt used for?

15. [Terraform] What is terraform import used for?

16. [Terraform] What are provisioners and providers in Terraform?


================================================================================================
Belcan
================================================================================================

17. [Terraform] What is the Terraform state file?

18. [Terraform] List several Terraform commands and explain each one.


================================================================================================
CMT
================================================================================================

19. [Terraform] Explain the structure of a Terraform project.


================================================================================================
CTS (Cognizant)
================================================================================================

20. [Terraform] [Unclear] The backend.tf file is visible in the repository but not in the storage
    account. What could be the issue?

21. [Terraform] How do you manage the Terraform state file?

22. [Related IaC] What happens when a resource managed through infrastructure-as-code is modified
    manually? How would you prevent this?

23. [Terraform] What have you done in Terraform and how did you do the integration?

24. [Terraform] What is the difference between terraform destroy and terraform refresh?

25. [Terraform] How do you prevent someone from running terraform destroy or otherwise destroying the
    infrastructure?


================================================================================================
Capgemini
================================================================================================

26. [Terraform] [Context] What are the different Terraform provisioners?

27. [Terraform] [Context] How do you unlock a Terraform state file?

28. [Terraform] Suppose you have created an EC2 instance by logging into the AWS console. And now you
    would like to manage it using Terraform. How shall you do it?

29. [Terraform] [Incomplete] Resources in Terraform. (The source contains only this incomplete
    fragment.)

30. [Terraform] If we can use terraform import for existing AWS resources which are not created by
    Terraform, then what is the use of data source?


================================================================================================
Cisco
================================================================================================

31. [Terraform] How would you migrate a Terraform backend from local to a remote backend like S3 with
    DynamoDB locking?

32. [Terraform] What happens if the Terraform state becomes corrupted, and how would you recover from
    it?

33. [Terraform] Write Terraform code to provision an EC2 instance with a security group allowing only
    SSH access.

34. [Terraform] [Context] How do you move a state file from local storage to an S3 bucket, and what
    would you do if it were lost?

35. [Terraform] [Incomplete] [Relevant portion] Write Terraform scripts to create AWS services. (The
    source does not identify the services for this exercise.)


================================================================================================
Deloitte
================================================================================================

36. [Terraform] How do you create and manage Kubernetes clusters (using tools like Terraform), and what
    are the master and worker nodes?

37. [Terraform] What work have you done with Terraform on AWS?

38. [Terraform / CloudFormation] What is the difference between Terraform and CloudFormation templates?

39. [Terraform] What work have you done with Terraform on AWS?

40. [Terraform / CloudFormation] What is the difference between Terraform and CloudFormation templates?

41. [Terraform] Write Terraform configuration for an EC2 instance with an EBS volume attached.

42. [Terraform] Explain your Terraform file structure for VPC and EKS resources.

43. [Terraform] Explain terraform init and terraform refresh.


================================================================================================
EPAM
================================================================================================

44. [Terraform] [Context] What is a Terraform provider?

45. [Terraform] How do you manage state in Terraform?

46. [Terraform] [Context] Do you store the state file locally or remotely? Which configuration block do
    you use to define its storage?

47. [Terraform] What is a Terraform module?

48. [Terraform] How do you manage multiple environments in Terraform?

49. [Terraform] How would you manage cross-region deployments using Terraform in a multi-cloud setup?

50. [Terraform] How would you refactor a legacy Terraform codebase used by multiple teams to follow best
    practices like DRY and modularity?

51. [Terraform] Explain the internals of how Terraform handles dependencies and graph building during
    the planning phase.

52. [Terraform] How do you manage and isolate Terraform state files across multiple environments and
    teams?

53. [Terraform] What's your strategy to prevent and recover from a corrupted or deleted remote backend
    state file?

54. [Terraform] Have you implemented policy-as-code (e.g., Sentinel, OPA) with Terraform? Give a real
    use case.

55. [Terraform] What is Terraform taint?

56. [Terraform] What are Terraform meta-arguments, including those other than for_each and count?

57. [Terraform] [Context] [Unclear] Is it possible to use "foreach" together? (The source does not
    specify what it would be combined with.)

58. [Terraform] Write a Terraform configuration that deploys Azure storage accounts based on the
    environment: 2 storage accounts for dev and 5 storage accounts for prod.

59. [Terraform] How do you use Terraform in a pipeline?

60. [Terraform] What is Terraform taint?

61. [Terraform] If someone manually deletes resources, how do you handle the situation in Terraform?

62. [Terraform] What are sets and lists in Terraform?

63. [Terraform] What Terraform meta-arguments are available apart from for_each and count?


================================================================================================
EXL Service
================================================================================================

64. [Terraform] Do you execute Terraform locally or through a CI/CD pipeline? Explain the complete
    workflow.

65. [Terraform] Two engineers are working on the same Terraform code. How do you prevent conflicts and
    handle Terraform state locking or drift?

66. [Terraform] Draw and explain your Terraform repository structure. How do your dev, qa, and prod
    environments consume shared modules like the VPC module?


================================================================================================
E&Y
================================================================================================

67. [Terraform] [Topic-only] Explain Terraform.


================================================================================================
Encora
================================================================================================

68. [Terraform] What is Terraform lifecycle management?

69. [Terraform] How do you bring a manually created resource under Terraform management?

70. [Terraform] What is the difference between for_each and count? Give examples in Terraform.

71. [Terraform] How do you create a Terraform module and reference it?

72. [Terraform] Given the address range 10.0.0.0/16, how would you create /21 subnets using Terraform?
    Explain how you would use locals to do this.


================================================================================================
F5
================================================================================================

73. [Terraform] What happens if the tfstate file is deleted?

74. [Terraform] What is the Terraform lock HCL file?

75. [Terraform] What Terraform best practices do you follow?


================================================================================================
Five9
================================================================================================

76. [Terraform] Apart from storing Terraform log files in S3, what other options are available?

77. [Terraform] What are the different types of provisioners in Terraform?

78. [Terraform] What is the difference between local and remote provisioners in Terraform runners?

79. [Terraform] [Context] What is the difference between user data and a remote provisioner?


================================================================================================
Flentas
================================================================================================

80. [Terraform] Explain the Terraform folder structure.

81. [Terraform / Terragrunt] What is the difference between Terraform and Terragrunt?

82. [Terraform] What is the state file in Terraform?

83. [Terraform] How do you handle resource dependencies in Terraform?

84. [Terraform] When you ran terraform apply, did you encounter any unexpected changes?

85. [Terraform] If there is a problem in Terraform, how will you roll back the changes?

86. [Terraform] You created a load balancer using Terraform, and it was subsequently updated. How would
    you delete only that load balancer?

87. [Terraform] How do you handle secrets in Terraform?

88. [Terraform] [Context] How will you pass secrets from AWS Secrets Manager to the pipeline?

89. [Terraform] Do you commit the tfvars file to Git? These parameters need to be filled in while the
    pipeline runs. How do you manage this in Jenkins?


================================================================================================
Hexaware
================================================================================================

90. [Terraform] Which Terraform command option enables automatic approval?

91. [Terraform] Write the overall skeleton of a Terraform configuration.


================================================================================================
IBM
================================================================================================

92. [Terraform] What is Terraform, and how does it work?

93. [Terraform] Where do you run your Terraform code—on your local system or on a specific server in
    your organization?

94. [Terraform] What is a tfstate file?

95. [Terraform] Where do you store the tfstate file in your organization?

96. [Terraform] What does the tfstate file actually do?

97. [Terraform] Suppose another DevOps engineer on your team has made changes to an instance via the UI,
    and you then run the terraform plan command. What will be the output?

98. [Terraform] What do the "+" and "−" symbols mean in the terraform plan output?

99. [Terraform] If changes have been made to an instance via the UI and you run terraform apply without
    first running terraform plan, what will happen? Will there be an error, and will it still execute?


================================================================================================
ITC Infotech
================================================================================================

100. [Terraform] How would you write a Terraform module for EKS?


================================================================================================
Infosys
================================================================================================

101. [Terraform] Write a sample Terraform resource configuration.

102. [Terraform] What is Terraform, how do you use it in your project, and which resources have you
    provisioned with it?

103. [Terraform] Why is Terraform used?

104. [Terraform] Which Terraform modules do you use?

105. [Terraform] What kind of experience do you have with Terraform?


================================================================================================
HCL
================================================================================================

106. [Terraform] Have you provisioned a virtual machine using Terraform?

107. [Related IaC] [Context] An EC2 instance was manually changed from t3.medium to t3.large, and the
    code was updated and committed, but the pipeline has not run. What would happen when the pipeline
    runs?


================================================================================================
Intact Green Services
================================================================================================

108. [Terraform] Two instances were created with Terraform, and the state file is stored locally and in
    an S3 remote backend. If a user deletes one instance, what happens, and how would you handle it?


================================================================================================
JPMorgan
================================================================================================

109. [Terraform] Define a plan for blue/green deployment with rollback on Azure using Terraform and
    pipelines.

110. [Terraform] As with code-quality and security checks before building an image, how would you
    prevent insecure infrastructure changes from being applied through Terraform?
    - For example, if someone changes a security group in Terraform to allow 0.0.0.0/0, what mechanisms
      can stop the change from being applied?

111. [Terraform] How would you manage Terraform state for a team? Explain how you would store tfstate
    securely, allow only one person to modify state at a time, and prevent state tampering.

112. [Terraform] You’ve joined a company where a large production infrastructure was built manually and
    the previous engineers have left. How would you bring that existing infrastructure under Terraform
    management? Walk me through your plan


================================================================================================
Koerber Pharma
================================================================================================

113. [Terraform] If someone deletes a Terraform-managed resource, how do you detect the deletion and
    restore it?

114. [Terraform] If the state file is deleted, how can you recover without recreating the resources?

115. [Terraform] How do you create and manage dev, prod, and test environments in Terraform?

116. [Terraform] Explain Terraform provisioners.

117. [Terraform] How would you connect your Terraform environment to AWS and implement CI/CD?

118. [Terraform] What is a Terraform workspace, and how do you manage workspaces?

119. [Terraform] Describe recent real-world Terraform issues or errors you have encountered.


================================================================================================
LTIMindtree
================================================================================================

120. [Terraform] You created three instances using Terraform, with their names stored in a list. If you
    remove the second instance name and apply the configuration again, what happens to the three
    existing instances?

121. [Terraform] Write Terraform code to create an Azure App Service, or write Terraform code to create
    an AWS Lambda function.

122. [Terraform] [Unclear] Create an S3 bucket for Terraform state that "expires within 30 days". (The
    source does not specify whether expiration concerns the bucket or its contents.)

123. [Related IaC] How do you manage multiple environments using reusable infrastructure code?

124. [Related IaC] What is the purpose of backends in infrastructure-as-code and how do you implement
    remote state with locking?

125. [Terraform] [Context] What’s the difference between using count and for_each in infrastructure
    code, and when should you use each?


================================================================================================
Moodys
================================================================================================

126. [Terraform] How do you set up infrastructure for deploying ML models using Terraform?


================================================================================================
NUOS INFO Systems
================================================================================================

127. [Terraform] How would you scale a Terraform pipeline that takes more than 25 minutes?

128. [Terraform] What happens to the Terraform state file if someone deletes resources from Azure?

129. [Terraform] [Context] If the pipeline fails because resources already exist, how would you handle
    RIP: Remove, Import, Plan?

130. [Terraform] How do you export Azure resources into Terraform code?

131. [Terraform] How do you enforce Azure Policies, such as tag or location restrictions, using
    Terraform at scale?


================================================================================================
Nextturn
================================================================================================

132. [Terraform] Terraform – Difference between terraform refresh and terraform plan.

133. [Terraform / Ansible] Terraform vs Ansible – When to use each and how they work together.

134. [Terraform] In Terraform, how would you create multiple EC2 instances, each with different
    configurations (for example, different instance types, AMIs, tags, or volumes)?


================================================================================================
Nice
================================================================================================

135. [Terraform] What is Terraform, and how did you configure it for your project?

136. [Terraform] What are Terraform modules?

137. [Terraform] Are you familiar with Terraform?

138. [Terraform] Can you describe a real-time scenario where you used Terraform to provision a highly
    scalable infrastructure?


================================================================================================
Nisum Technologies
================================================================================================

139. [Terraform] [Unclear] What is the difference between "content" and a tuple in Terraform? (The
    source does not define what "content" means here.)

140. [Terraform] What is the difference between a list and a string in Terraform?


================================================================================================
Nitor Infotech
================================================================================================

141. [Terraform] Explain the terraform refresh command.

142. [Terraform] How do you manage infrastructure code for multiple environments using Terraform?

143. [Terraform] Explain state locking in Terraform.


================================================================================================
OPT IT
================================================================================================

144. [Terraform] [Context] Explain workspace and module concepts.


================================================================================================
Optum
================================================================================================

145. [Terraform] What are Terraform lifecycle policies?

146. [Terraform] Why do we use workspaces in Terraform?

147. [Terraform] [Unclear] What is the "Terraform external command", and when should it be used? (The
    source does not define this term.)

148. [Terraform] How do you ensure that a particular AMI is available in an AWS account using Terraform?

149. [Terraform] What are Terraform provisioners?

150. [Terraform] What are meta-arguments in Terraform?


================================================================================================
Oracle
================================================================================================

151. [Terraform] How do you recover a corrupted Terraform state file?

152. [Terraform] [Unclear] Explain Terraform lifecycle behavior for resource creation and destruction.
    (Source wording: "create / after destroy".)


================================================================================================
Others
================================================================================================

153. [Terraform] How would you set up a new environment on AWS through Terraform provisioning? Explain
    using your project.

154. [Terraform] How do you provision infrastructure using Terraform through CI/CD?

155. [CloudFormation] [Relevant portion] Explain AWS CloudFormation (CFT).

156. [Terraform] In Terraform, what is the purpose of init, plan, and apply commands?

157. [Terraform] What happens if the Terraform state file is accidentally deleted?

158. [Terraform] Can u pls write terraform file to provision the Ec2 instance in a public subnet in a
    VPC?

159. [Terraform] What would you do if a tfstate file were lost, both with and without a backup?

160. [Terraform] What are the provisioners available in Terraform and can you explain the use cases?

161. [Terraform] [Context] You have created EC2 instance A and want to create instance B without
    deleting A. How would you achieve this?

162. [Terraform] I have created an EC2 instance through Terraform. I don't have a backup of the
    Terraform state file, it is not in the remote state and locally not available. Now when I do apply,
    what can I do?

163. [Terraform] [Context] A command in a null_resource should run every time. How would this work?

164. [Terraform] [Unclear] Explain a map of objects in Terraform and write an example. (The source asks
    for a comparison but does not name the other type.)

165. [Terraform] Create and configure AWS EventBridge using Terraform.

166. [Terraform] What is a state file in Terraform?

167. [Terraform] What is a lock file in Terraform?

168. [Terraform] [Relevant portion] Explain Terraform.

169. [Terraform] Write Terraform code to create multiple S3 buckets.

170. [Terraform] [Context] How do you manage the state file?

171. [Terraform] [Context] How do you manage state-file conflicts?

172. [Terraform] Terraform script to provision an EC2 instance with a custom security group and user
    data script.

173. [Terraform] Explain Terraform state-file locking.

174. [Terraform] Which CI/CD system do you use for Terraform infrastructure in your project?

175. [Terraform] [Unclear] How do IAM users, a GitHub OIDC role, and a "terraform io role" differ in
    security? When would you use a GitHub OIDC role, and when would you use a "terraform io role"? (The
    source does not define "terraform io role".)

176. [Terraform] Write Terraform code to provision an EC2 instance.

177. [Terraform] What is the difference between terraform validate and terraform fmt?

178. [Terraform] [Context] What are provisioners, and how do you use them?

179. [Terraform] Create S3 resources using Terraform and ensure that the resource is automatically
    deleted after seven days.

180. [Terraform] [Context] What is a state file, and how do you store it?

181. [Terraform] What is Terraform lifecycle management, and what does it do?

182. [Terraform] Can the same Terraform code be used for different cloud providers?

183. [Terraform] Why are dynamic blocks used in Terraform? Write a skeleton for an Azure resource using
    a dynamic block.

184. [Terraform] [Context] How do you pass the subnet ID output from a VNet module as an input to a VM
    module?

185. [Terraform / CloudFormation] What are the differences between CloudFormation and Terraform?

186. [Terraform / CloudFormation] How do you apply least privilege to CloudFormation stacks and
    Terraform?


================================================================================================
Persistent Systems
================================================================================================

187. [CloudFormation] [Topic-only] Explain AWS CloudFormation.


================================================================================================
Plansource ValueLabs
================================================================================================

188. [Terraform] How do you manage Terraform pipeline variables for different environments, such as dev,
    live, and feature?


================================================================================================
Qburst
================================================================================================

189. [Terraform] How would you prevent a Terraform-provisioned resource from being deleted when its
    resource configuration is removed from the code?

190. [Terraform] How do you manage different environments in Terraform?

191. [Terraform] Can you move an existing Terraform state file to a remote backend?


================================================================================================
Qentelli Solutions
================================================================================================

192. [Terraform] Create an S3 bucket using Terraform.

193. [Terraform] How do you manage resources in Terraform?


================================================================================================
Rapidsoft
================================================================================================

194. [Terraform] Terraform reports errors while provisioning infrastructure. How do you investigate
    them, and how do you validate the Terraform configuration?


================================================================================================
RelevantZ
================================================================================================

195. [Terraform] You have two environments. Infrastructure should be deployed with Terraform to one
    environment, while nothing should be deployed to the other. What strategy would you use?

196. [Terraform] You have written 100 lines of Terraform code and want to avoid repeating it. How would
    you achieve this?

197. [Terraform] What problems can occur when multiple people execute Terraform commands?

198. [Terraform] What is Terraform drift?


================================================================================================
Sapient
================================================================================================

199. [Related IaC] [Context] [Unclear] What is an "iteration limit"? (The source does not identify the
    tool or situation.)

200. [Terraform] [Context] What is a data block?

201. [Terraform] What are modules in Terraform?

202. [Terraform] [Context] How do you call your modules?

203. [Terraform] [Context] Have you worked with null resources?

204. [Terraform] Explain the Terraform state file.

205. [Terraform] [Context] Which configuration file defines where Terraform state is created and
    maintained?


================================================================================================
Sigmoid
================================================================================================

206. [Terraform] Explain Terraform taint.

207. [Terraform] What happens if a resource created through Terraform fails during provisioning?

208. [Terraform] What is the purpose of null_resource in Terraform?

209. [Terraform] What would you do if a Terraform deployment failed?

210. [Terraform] How do you enable debug logs in Terraform?

211. [Terraform] How does a data block differ from a resource block in Terraform? Give a scenario that
    uses both together.

212. [Terraform] Some Terraform-managed resources must be removed from Terraform management and handled
    manually. How would you do this without modifying, disrupting, or destroying those resources?

213. [Terraform] What is Terraform taint? Give a scenario where you would use it and explain how it
    works.

214. [Terraform] What are Terraform workspaces, and what is the main reason for using them?

215. [Terraform] A configuration change to a Terraform-managed AWS resource caused brief downtime during
    apply. What could have caused it? How would you handle future changes to minimize downtime and
    maximize availability? Write Terraform code for your approach.


================================================================================================
Sonata Software
================================================================================================

216. [Terraform] How do you write a Terraform module?

217. [Terraform] How do you upgrade a Terraform module?

218. [Terraform] Share your screen and write the structure of a Terraform project.

219. [Related IaC] How long would it take you to write infrastructure-as-code and deploy an application
    to Azure App Service?

220. [Terraform] Have you used Terraform Cloud?

221. [Terraform] Have you worked on Terraform?

222. [Terraform] Say I created an S3 bucket using Terraform and want to modify the bucket name. Is it
    possible? How would you do this?

223. [Terraform] Can you tell me what is Data Block in Terraform?

224. [Terraform] How do you handle multiple environments in Terraform?

225. [Terraform] What are modules in Terraform?


================================================================================================
Sony
================================================================================================

226. [Terraform] Write Terraform code to start an EC2 instance when CPU utilization reaches 80% and also
    copy an image from S3.

227. [Terraform] How would you safely refactor a Terraform monorepo containing hundreds of state files
    into an architecture based on modules without downtime?

228. [Terraform] Explain how Terraform handles dependency graphs internally. How can circular
    dependencies still appear in real projects?

229. [Terraform] How would you manage Terraform when multiple teams deploy to the same AWS account but
    must not overwrite each other’s resources?

230. [Terraform] Describe a production failure caused by terraform apply. What guardrails would you
    implement to prevent it permanently?


================================================================================================
SquareOps
================================================================================================

231. [Terraform] [Context] Have you created an EKS cluster? Explain the process.
    - Did you use the console, CLI, or Terraform?
    - Which VPC and subnet configuration did you use?
    - How did you configure node groups?
    - How do you bootstrap kubectl access?

232. [Terraform] Have you used Terraform?
    - Show the module structure.
    - Show the provider file.
    - How do you manage a remote backend?
    - How do you manage state locking?
    - What is a data block?
    - What is a module block?

233. [Terraform] Terraform generated an RDS password, but you did not save it. Can you retrieve it?
    - Where does Terraform store generated values?
    - Are they in local state or a remote backend?
    - Why is storing secrets in plaintext dangerous?

234. [Terraform] What is a custom Terraform module?

235. [Terraform] [Context] What does a Terraform module contain?

236. [Terraform] [Context] What is in main.tf?
    - What is in variables.tf?
    - What is in outputs.tf?
    - What is in providers.tf?
    - How do you call a module from the root module?


================================================================================================
Syncortex
================================================================================================

237. [Terraform] How would you implement Terraform code for multiple Regions?

238. [Terraform] [Context] How do you ensure that an EC2 instance is not deleted when you run the
    destroy command?


================================================================================================
Synechron
================================================================================================

239. [Terraform] Which command do you use to identify Terraform drift?

240. [Terraform] Explain Terraform architecture.

241. [Terraform] [Context] What is a Terraform backend?

242. [CloudFormation] What is the difference between CloudWatch and CloudFormation?


================================================================================================
TCS
================================================================================================

243. [Terraform] [Unclear] What is a "Terraform state file interpreter"? (The source does not explain
    this term.)

244. [Terraform] What would you do if terraform apply took too long?

245. [CloudFormation] [Missing diagram] Explain the AWS architecture shown in the diagram, involving
    CodePipeline, CodeBuild, CodeDeploy, CloudFormation, and CloudWatch. [The referenced diagram is not
    present in the attached text.]

246. [Terraform] What is Terraform drift?


================================================================================================
Techdome
================================================================================================

247. [Terraform] Explain Terraform provisioners.

248. [Terraform] Explain the Terraform state file.


================================================================================================
Turning
================================================================================================

249. [Related AWS CDK] Explain AWS CDK commands.

250. [Terraform] How do you implement Terraform in a CD pipeline?

251. [Terraform] How do you resolve manual configuration changes to an EC2 instance created through
    Terraform?


================================================================================================
UST
================================================================================================

252. [Terraform] If something is created on the cloud platform and it is not present in Terraform, how
    will you achieve it?

253. [Terraform] Terraform apply is creating all the resources again. What can be the possible problem?

254. [Terraform] Write Terraform code to create an AWS EC2 instance and include variables for
    instance_type and region.

255. [Terraform] You need to create 50 instances in one go. How will you create them in Terraform?

256. [Terraform] If somebody has deleted the Terraform state file locally, what can be done?


================================================================================================
Virtusa
================================================================================================

257. [Terraform] What Terraform command unlocks a state file?

258. [Terraform] What is a Terraform module?

259. [Terraform] Several EC2 instances were created manually through the console. How would you bring
    them under Terraform management so you can update them? Which command would you use?

260. [Terraform / CloudFormation] What is the difference between CloudFormation and Terraform?


================================================================================================
Wipro
================================================================================================

261. [Terraform / Ansible] Could you elaborate your experience with automating and optimizing the
    deployment over large infrastructure using AWS and other tools like Terraform and Ansible from your
    previous roles?

262. [Terraform] Which Terraform commands do you know?

263. [Terraform] [Context] How do you manage the state file?

264. [Terraform] Have you encountered a scenario in which you used terraform destroy?

265. [Terraform] Explain your Terraform modules.

266. [Related IaC] [Context] [Unclear] Explain the command for using S3 native locking. (The source does
    not specify the locking mechanism.)

267. [Terraform] [Context] What are the different types of modules, and why are they used?


================================================================================================
ZS Associates
================================================================================================

268. [Terraform] How do you implement state locking in Terraform?

269. [Terraform] Explain for_each in Terraform with an example.

270. [Terraform] What would your Terraform code structure look like to deploy EC2 instances in three
    different Regions?

271. [Terraform] [Context] [Topic-only] Explain Terraform modules.

272. [Terraform] You changed a module for one resource. The resource should be updated without being
    destroyed and recreated by terraform apply. How would you achieve this?

273. [Terraform] [Context] What would be the structure of a module for an EKS cluster?


================================================================================================
ZopSmart
================================================================================================

274. [Terraform] What does terraform init do?

275. [Terraform] Which Terraform commands do you know?

276. [Terraform] What does terraform plan do?
