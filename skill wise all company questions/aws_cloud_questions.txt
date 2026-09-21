AWS CLOUD INTERVIEW QUESTIONS
Company-wise extraction

Source: interview questions company wise.rtf
Total: 552 numbered questions, tasks, and topic prompts across
59 named companies plus the source's "Others" category.
Subquestions kept under a numbered scenario are included in that scenario count.

Scope: AWS services, EC2, EKS/ECS/Fargate, IAM, VPC networking, S3/storage,
RDS/databases, Lambda/serverless, monitoring, security, cost optimization,
AWS infrastructure automation, and multi-cloud scenarios involving AWS.

Source wording is retained with formatting cleaned and selected questions
lightly edited for readability. Repeated questions from different interview
sets are retained under their original company names. No answers are added;
embedded answer notes are removed. Scenario assumptions, abbreviations, and
references such as "recent" come from the source.

Labels:
  [Follow-up] A general question retained from an AWS interview scenario.
  [Multi-cloud] A question involving AWS together with another cloud.
  [AWS portion] Only the AWS-related portion of a mixed source prompt.
  [Unclear] Source wording contains an ambiguous term or requirement.

COMPANY INDEX
  Accenture: 9
  Accion Labs: 1
  Akamai: 1
  Alphadyne: 3
  Altimetrik: 2
  Amazon: 13
  Arrise Solutions: 7
  Aspire: 5
  BMW TechWorks: 4
  Belcan: 2
  CMT: 8
  CTS (Cognizant): 5
  Capgemini: 16
  Cisco: 7
  Deloitte: 48
  EPAM: 38
  EXL Service: 5
  Emphasis: 4
  F5: 3
  Five9: 2
  Flentas: 16
  IBM: 2
  ITC Infotech: 5
  Infosys: 6
  HCL: 19
  Intact Green Services: 2
  JPMorgan: 14
  Koerber Pharma: 18
  LTIMindtree: 11
  Moodys: 4
  NUOS INFO Systems: 1
  Nextturn: 2
  Nisum Technologies: 1
  Nitor Infotech: 5
  OPT IT: 1
  Optum: 4
  Orion Innovation: 2
  Others: 86
  Perfios: 2
  Persistent Systems: 7
  Plansource ValueLabs: 10
  Publicis Global Delivery: 1
  Qburst: 1
  Qentelli Solutions: 10
  Sapient: 14
  Sigmoid: 20
  Sonata Software: 20
  Sony: 4
  SquareOps: 27
  Syncortex: 1
  Synechron: 4
  TCS: 6
  Turning: 4
  UST: 2
  Verizon: 6
  Virtusa: 5
  Wikreate Media: 2
  Wipro: 8
  ZS Associates: 11
  ZopSmart: 5

ACCENTURE
---------

1. You have created an IAM user in AWS and configured role-based access in EKS. How do you bind the
    IAM user to the EKS role?

2. Assume you have 10 AWS accounts. How will you securely log in to them, considering access keys
    are not used for security reasons?

3. What are the ways to log in to an AWS account?

4. Does Amazon S3 require a VPC?

5. Write a Terraform script to create an EC2 instance in multiple regions.

6. You have defined a multi-region Terraform configuration (region1, region2, region3). If you
    create an EC2 instance, in which region will it be deployed?

7. [Follow-up] If the frontend, backend, and database are all deployed in private subnets, how can
    an end user access the application?

8. If secrets are created in AWS Secrets Manager, how can Amazon EKS access those secrets?

9. How do you set up RBAC in Amazon EKS?

ACCION LABS
-----------

10. How do you configure a VPC for high availability?

AKAMAI
------

11. What is the difference between public and private hosted zones?

ALPHADYNE
---------

12. An application is hosted on a public EC2 instance. How would you migrate it to a private subnet
    following AWS best practices for security, networking, and high availability?

13. [Follow-up] How to provide HTTPS access to an application hosted in a private subnet.

14. Discuss ALB/NLB, ACM certificates, routing, and security groups.

ALTIMETRIK
----------

15. [Multi-cloud] How do you import a resource into Terraform that was created manually in AWS or
    GCP? What command would you use?

16. Using the AWS provider, write Terraform code to create a VPC and a subnet, then attach the
    subnet to the VPC. The subnet may be public or private.

AMAZON
------

17. Sending log files from EC2 to S3, what are the steps?

18. You are an administrator but cannot access an S3 bucket. What could prevent access?

19. You have an S3 bucket at us-south-1, is it possible to access that bucket from us-east-1?

20. Write a Terraform code to create VPC, subnet, EC2, S3 bucket.

21. Automate sending log files from EC2 to S3, checking CPU metrics, and sending an alarm through
    CloudWatch.

22. Is it possible to create NAT gateway in private subnet?

23. [Follow-up] How to configure Cluster auto-scaling, how to do that

24. You are migrating a monolithic application from on-premises to the cloud. It currently uses a
    local file system. Which file system would you use in AWS?

25. [Follow-up] How will you store all the configurations related to your monolithic app in Cloud

26. A trading application is being onboarded to AWS. How would you ensure availability, scalability,
    and security?

27. Write a Python script to list the EC2 instances running in your cloud that have the PROD tag.

28. You must create 20 EC2 instances in each of 10 AWS accounts, for a total of 200 instances. How
    would you connect all these instances, and which service would you use?

29. How would you connect to a database in a private subnet without using a NAT gateway, NAT
    instance, or bastion host? What other options are available?

ARRISE SOLUTIONS
----------------

30. [Follow-up] Explain me about three tier architecture

31. Diff b/w ALB and NLB in depth

32. [Multi-cloud] How to connect a VPC in AWS to VPC in IBM Cloud

33. [Follow-up] Diff b/w Public and Private subnet

34. [Follow-up] How to connect your private subnet with Internet

35. Does NAT gateway will run in public or private subnet

36. CloudFront

ASPIRE
------

37. What is the difference between NACL and Security groups?

38. What is mean by CRR in s3?

39. What are the different types of triggers in lambda aws?

40. What is mean by Nat Gateway and Nat instance?

41. What is mean by Sticky session in ALB?

BMW TECHWORKS
-------------

42. If an EC2 instance has a vulnerability, how would you identify and fix it?

43. How do you patch EC2 instances?

44. Can we separate disk space in an EC2 instance and run the application on one partition and the
    observability stack on another?

45. How do you perform this disk separation, and which AWS services/tools do you use?

BELCAN
------

46. How many load balancer types are available in AWS? Explain each one.

47. What is the difference between an Elastic IP address and a public IP address in AWS?

CMT
---

48. Which are all the services you used in AWS

49. Design an high availability, fault tolerance system in aws

50. How would you maintain high availability in ecs + fargate or eks

51. What is the difference between alb and nlb, in which scenario you use alb and nlb

52. [Follow-up] In webserver/app server which metrics is used to monitor the high availability

53. What metrics is used to monitor ec2 instance cpu, memory in aws

54. If there is slowness issue in decouple (SQS), how would you handle it

55. [Follow-up] What are the best practices can be used to keep the systems highly available

CTS (COGNIZANT)
---------------

56. [Follow-up] How would you design an architecture for a 2 tier application

57. Difference between subnet and nacl

58. Difference between nat gateway and internet gateway

59. How would you set up networking in vpc

60. [Follow-up] How you will direct traffic to and from a instance in private subnet

CAPGEMINI
---------

61. Explain AWS Lambda functions and Step Functions.

62. What are autoscaling policies and their uses?

63. [Unclear] What is a "target probe" in autoscaling? [Term retained from the source; its intended
    meaning is unclear.]

64. Which role to give to access services in AWS?

65. [Follow-up] How to configure a role to service accounts?

66. What is the difference between NAT Gateway and IGW?

67. Can you tell me the difference between secondary RDS and read-only RDS?

68. So have you built the RDS of your own or you have managed it only?

69. You have RDS, and the client wants only one user to access it at a time. How would you configure
    this, and where would you make the configuration?

70. To upgrade the version of DB in RDS, suppose you have 7.0 MySQL installed, and you want to
    upgrade it into 8.0 and above. What is the process?

71. In a multi-account environment, resources reside in one account and users in different accounts.
    How would you let the users access the resources? Address both IAM and VPC networking.

72. You have an EC2 instance and you would like to migrate it from one region to another. How will
    you do it?

73. You want to create an EC2, and while creating the instance, you are getting an error like IP
    address exceeded. How will you troubleshoot and fix it?

74. [Follow-up] That we can extend the subnet CIDR once it is created?

75. Suppose you have created an EC2 instance by logging into the AWS console. And now you would like
    to manage it using Terraform. How shall you do it?

76. If we can use terraform import for existing AWS resources which are not created by Terraform,
    then what is the use of data source?

CISCO
-----

77. How would you migrate a Terraform backend from local to a remote backend like S3 with DynamoDB
    locking?

78. [Follow-up] What happens if the Terraform state becomes corrupted, and how would you recover
    from it?

79. Write Terraform code to provision an EC2 instance with a security group allowing only SSH
    access.

80. How do you update the statefile from local to S3 bucket,what will you do if it gets lost.

81. [AWS portion] Write Terraform scripts to create AWS services and explain the steps to upgrade an
    EKS cluster.

82. For deployment if you get timeout issue, what kind of api gateway you used?

83. [Follow-up] And how did you managed security for application level?

DELOITTE
--------

84. What types of nodes did you deploy in AWS?

85. What is the difference between Interface Endpoint and Gateway Endpoint?

86. How did you set up ECS using EC2 instances?

87. Can't we configure Route 53?

88. If we want to configure third-party domains like GoDaddy in Route 53, how do we do that?

89. What is the difference between AWS Config and AWS CloudTrail?

90. What are the node groups you used in AWS EKS?

91. What are the types of node groups in AWS EKS?

92. If a user wants to access the S3 bucket, what are the processes?

93. How does VPC Peering work?

94. How does Transit Gateway work and how did you configure it?

95. If we connect VPCs to the Transit Gateway, what will you update in the VPC Route Table?

96. For all VPCs, will you configure the Transit Gateway attachment with CIDR range?

97. What is Route 53?

98. [Unclear] What are WAF (Web Application Firewall) and "AAF (Application Access Firewall)"? [The
    AAF name is retained from the source.]

99. What is VPC Flow Logs and how will you track the IPs hitting the VPC?

100. How to filter a particular IP from AWS CloudWatch Log Group?

101. If you are storing logs in S3 Bucket, how will you track that particular IP?

102. How do you take the backup of AWS Services?

103. Can we create AWS backup using Shell Scripting?

104. [Follow-up] Once the backup is created, where will you store the log files?

105. How do you manage and connect services like DBs, EC2, EKS, or ECS? Include the command to
    connect to ECS.

106. [Follow-up] Which container registry do you use for storing Docker images?

107. How do you handle authentication for EKS clusters and store secrets securely in your
    environment?

108. How do you create AWS Lambda functions and manage the artifacts for deployment? What options do
    you use to push artifacts to Lambda?

109. Difference between EKS and ECS?

110. What are all the prerequisites for you to setup a EKS cluster with 2 worker nodes and some x no
    of pods?

111. Generally when two or more pods are available how do you manage the load balancing? are you
    sure you'll be using ALB

112. When do you use ALB and When do you use NLB?

113. What have you done with terraform in AWS space?

114. Difference between terraform and Cloud formation templates?

115. An AWS account is given to you and if I ask you to create a VPC, all the services required for
    a service in EC2 to be exposes to internet, what all services come into picture?

116. What is transit gateway?

117. Difference between EKS and ECS?

118. What are all the prerequisites for you to setup a EKS cluster with 2 worker nodes and some x no
    of pods?

119. Generally when two or more pods are available how do you manage the load balancing? are you
    sure you'll be using ALB

120. When do you use ALB and When do you use NLB?

121. What have you done with terraform in AWS space?

122. Difference between terraform and Cloud formation templates?

123. An AWS account is given to you and if I ask you to create a VPC, all the services required for
    a service in EC2 to be exposes to internet, what all services come into picture?

124. What is transit gateway?

125. Web application is not accessible but ec2 is running fine.what are the major reasons

126. [Follow-up] What measures you will take to reduce the infra cost by 20%

127. Write a Terraform configuration file for ec2 with EBS volume attached

128. What will you do for zero-downtime when eks cluster upgrade

129. [Follow-up] Did you write any automation script that will use for cost optimization.

130. What is your Terraform file structure for vpc, eks

131. I want to take data of how many ec2 running and how many EBS are attached from 50 or 60 aws
    accounts without logging into individual account . How can we do that

EPAM
----

132. What is cloud watch uses cases.

133. What is the ECS and eks

134. What is fargate

135. What are the limitations of AWS Lambda?

136. [Unclear] How does Lambda work with containers? [The source wording, "how lamda works
    containers", is abbreviated.]

137. What is ec2 instances

138. Direct connect in aws

139. Storage gateway in AWS

140. VPC, NAT gateway,s3, route53, vpc peering,transit gateway, autoscaling group

141. Difference between sG and NACL

142. Diff b/w ALB and NLB

143. Purpose of using VPC Endpoint with use case

144. Is it possible to take AMI details from Snapshots

145. How to check LB health details (monitoring) through AWS service

146. What are the different types of instance profiles?

147. AMI vs Snapshots

148. The development team changed the AMI configuration in an Auto Scaling group launch template.
    How would you ensure that the new version is deployed correctly?

149. Explain Transit Gateway (TGW) in AWS.

150. After connecting different VPCs through a Transit Gateway, how would you block traffic from A
    to B and from B to C?

151. An EC2 instance in a private subnet must receive inbound traffic. How would you enable it
    without using a NAT gateway?

152. [Follow-up] Enabling tight security to my LB

153. [Follow-up] Is it possible to add multiple LBs to my sub web pages

154. Userdata in EC2

155. How to segregate the critical details from VPC flow logs

156. Diff b/w Fargate vs EKS worker nodes

157. Updating EKS cluster

158. An Auto Scaling group launches two EC2 instances during heavy load, but provisioning takes 2 to
    3 minutes and the group terminates the instances. How would you prevent this?

159. Traffic is high between 5 PM and 8 PM every day. How would you configure the Auto Scaling
    group?

160. Can one VPC have two different CIDR blocks, one in the 172.* range and one in the 192.* range?

161. Customizing WAF

162. Cloud Front configuration

163. AWS Image builder

164. [Follow-up] Diff type of instances

165. If you're storing all your VPC Flow logs in S3 bucket, how to see that

166. API Gateway configuration

167. [Follow-up] Diff between Private and Public IPs

168. Diff between Spot VS reserved instances

169. [Multi-cloud] What's your approach to securing cloud-native DevOps infrastructure with Identity
    Federation (e.g., Azure AD + AWS IAM)?

EXL SERVICE
-----------

170. Draw and explain your Terraform repository structure. How do your dev, qa, and prod
    environments consume shared modules like the VPC module?

171. Two VPCs need to communicate, but their CIDR ranges overlap. Transit Gateway is not allowed.
    What alternative solution would you recommend?

172. Which AWS EC2 instance types have you used, and why did you choose them?

173. Give a real-world use case of AWS Lambda.

174. A developer accidentally commits AWS credentials to Git. What is your complete incident
    response process?

EMPHASIS
--------

175. [Follow-up] Explain components in 3-tier architecture

176. [Follow-up] How private subnet connect with outside world

177. What is difference between NACL & security groups

178. What is the purpose of NAT gateway

F5
--

179. If we have security group configured in the instance do we really need nacl.

180. Difference between Transit gateway and VPC

181. If thre is an instance we have security group and web application firewall enabled, DDOS attack
    enabled will it protect from Bot attack.

FIVE9
-----

182. Apart from storing TF log files in S3, do we have any other options

183. Apart from using password, how to login to EC2

FLENTAS
-------

184. Why did you not deploy the frontend on S3 and CloudFront, and instead deployed it on EKS?

185. What is a namespace in EKS?

186. [Follow-up] How many namespaces do you currently have?

187. What is a node group?

188. How do you perform cost optimization on ECS, RDS, and ElastiCache?

189. Why have you used RDS Proxy?

190. [Follow-up] The client application did not have connection pooling — how did you handle it?

191. [Follow-up] A node is unable to join the cluster. What could be the reason?

192. [Follow-up] One node has 8 vCPU and 32 GB RAM. Pod autoscaling allows up to 4 replicas. Each
    pod has limits of 4 vCPU and 16 GB RAM, and requests of 2 vCPU and 10 GB RAM. How many instances
    of the pod can run?

193. What did you implement in Lambda and API Gateway?

194. How will you pass secrets from AWS Secrets Manager to the pipeline?

195. [Follow-up] During high traffic, how would you enable Kubernetes autoscaling for both cluster
    nodes and pods?

196. [Follow-up] Can we use Cluster Autoscaler?

197. How do you scale EC2 instances?

198. [Follow-up] When do we use VPA (Vertical Pod Autoscaler)?

199. Have you upgraded the EKS cluster?

IBM
---

200. How do you deploy your application on AWS? What services do you use?

201. When deploying your application to Amazon EKS, what other services do you use along with it?

ITC INFOTECH
------------

202. How to make connections between on prem to AWS suppose if we want to share files from on prem.

203. How to connect S3 with ec2 if a script generates files daily that need to be pushed S3.

204. How will you write terraform module for EKS.

205. If your application is on EKS how the traffic flows if user hits the URL.

206. What is OIDC provider in AWS.

INFOSYS
-------

207. ELB, Ingress questions

208. What is security group and what is the default traffic rule in sg

209. So what all services do you have used in AWS?

210. [Follow-up] How do you make your cloud infrastructure more secure?

211. How to secure the S3 bucket?

212. Suppose you have a VPC, and in your VPC, you have 2 subnets. One is a private subnet, and
    another one is a public subnet. And in these subnets you have 2 - 3 instances. And for security
    purposes we need to keep those instances updated right on a regular basis. The instances in the
    public subnet are okay. They have internet connectivity. They can get updated. How will you
    update the instances which are in the private subnet?

HCL
---

213. How to upgrade eks clusetr

214. What are steps to upgrade eks cluster

215. [Unclear] Explain the following source topics: landing zone, "guard trail", SCP, and "control
    tar". [The terms "guard trail" and "control tar" are unclear in the source.]

216. How would you share a KMS-encrypted AMI from Account 1 to Account 2?

217. An EC2 instance has IAM roles. What can you access or explore through them?

218. The source describes EC2 and S3 as being in the same subnet and region. How would you access
    the bucket from the instance?

219. An EC2 instance has no internet connectivity. How would you troubleshoot it?

220. What is the best way for two instances to communicate? Would you create a new NIC or attach
    one?

221. For multiple IAM users, would you attach a policy to each user individually, or add the users
    to a group and attach the policy to the group?

222. Would you use an inline policy or attach a policy? Which approach would you choose?

223. Have you used permissions boundaries?

224. What are the advantages of S3 lifecycle rules?

225. Have you worked with Transit Gateway?

226. Have you used ALB and NLB?

227. A public and a private subnet exist, and the source describes a NAT gateway as attached to the
    private subnet. What is its purpose?

228. How does a NAT gateway protect a private subnet? What concept does it use?

229. Have you worked with KMS and Secrets Manager?

230. An EC2 instance was manually changed from t3.medium to t3.large. The code was also updated and
    committed, but the pipeline has not run. What happens when the pipeline runs?

231. Can you reuse the same build specification used in AWS CodeBuild?

INTACT GREEN SERVICES
---------------------

232. There are 2 VPCs A & B. Give me options so that A communicate to B and vice versa. Also, option
    where only A has to communicate to B.

233. 2 Instances are created using terraform. Statefile is located locally and also in remote
    backend(S3). If a user deletes 1 instance what would happen? How would you handle this?

JPMORGAN
--------

234. How do you deploy an application to AWS?

235. What is serverless in AWS, and how are you using it?

236. How are you provisioning your AWS services?

237. [Follow-up] How do you provision container-based services for microservices deployment?

238. [Follow-up] What database services did you provision along with your application?

239. What is the primary difference between ECS and EKS?

240. [Follow-up] Where do you define auto-scaling parameters?

241. [Follow-up] What do you know about Serverless architecture?

242. How do you optimize cold starts in AWS Lambda?

243. What is Serverless deployment in AWS?

244. Your application is currently running on EC2 instances in a public subnet. How would you
    migrate it to a private subnet without any downtime? Explain the complete approach. if it doent
    work how would u rollback?

245. How would you prevent insecure infrastructure changes from being applied through Terraform? For
    example, if someone opens a security group to 0.0.0.0/0, what mechanisms would stop that change?

246. [Multi-cloud] Are you aware of the recent AWS and Azure outages? What were your key takeaways
    from those incidents?

247. [Multi-cloud] If an entire region goes down and even a multi-cloud setup (like AWS + Azure)
    experiences outages how would you ensure that your data is still safe and not lost? What
    strategies would you use to guarantee reliable backups and recovery?

KOERBER PHARMA
--------------

248. How to handle cost optimization in aws…how can we plan for cost optimization.

249. How are you connecting client's environment from your AWS environment.

250. Types of ec2 instance.

251. What are Spot, Reserved, and On-Demand Instances?

252. How will you connect your terraform environment from aws and implement CI/CD.

253. [Unclear] Explain CloudWatch, CloudTrail, and "cloud matrix". [The meaning of "cloud matrix" is
    unclear in the source.]

254. [Follow-up] If some resource is deleted how can you identify which resource is deleted.

255. Security best pracices in aws

256. How to manages certificate in aws. If the certificate expires how are you managing it and
    what's the action you are taking over here.

257. Difference between NAT gateway and NAT instance.

258. Difference between transit gateway and vpc peering.

259. Suppose you joined to a organisation, how the access would be given to you.how to secure aws
    account as admin.

260. Difference between roles and policies

261. Types of storages…difference between s3 and EBS.

262. Aws lambda where to use use case

263. Different plugins for ci/cd in jenkins using aws platform

264. Service to monitor spike in aws .application cpu usuage in cloud.

265. How to do backup from ebs volume and attach in another server

LTIMINDTREE
-----------

266. Aws code commit flow

267. Lambda functions

268. How u secure Lambda

269. How u do environmental variable in aws

270. [AWS portion] Write a Terraform script to create an AWS Lambda function.

271. [AWS portion] Create three different container images, store them in Amazon ECR, and deploy
    them to EKS. Write the Kubernetes YAML files.

272. How do you deploy python application on aws using jenkins pipeline

273. How do you upgrade your eks

274. [Follow-up] How do you handle when pod dies

275. Your aws jenkins pipeline takes high time, how will you troubleshoot

276. [Unclear] Create a Terraform state-file S3 bucket that expires within 30 days. [The source does
    not clarify whether it means the bucket, its objects, or the state file.]

MOODYS
------

277. How do you manage and version Docker images stored in Amazon ECR?

278. Apart from SageMaker, which AWS or open-source services have you used or are aware of for
    training ML models?

279. How do you prevent misuse or unauthorized usage if someone attempts to spin up ML services in
    AWS?

280. What strategies do you use to optimize and control AWS costs for ML workloads?

NUOS INFO SYSTEMS
-----------------

281. [Multi-cloud] How do you manage AWS + Azure using a single DevOps process with focus on
    security & cost?

NEXTTURN
--------

282. In Terraform, how would you create multiple EC2 instances, each with different configurations
    (for example, different instance types, AMIs, tags, or volumes)?

283. You are able to launch an EC2 instance from the AWS Console, but you cannot SSH into the
    instance. How would you install tree package

NISUM TECHNOLOGIES
------------------

284. Expalin about fargate?

NITOR INFOTECH
--------------

285. Explain SCPs.

286. VPC Endpoint, IGW, Transit Gateway, Virtual Gateway

287. Need to provide user only EC2 start and stop access how would you do it

288. User has the role with policy to access S3 bucket, but it still not able to access what may be
    the reason

289. How have you implemented RBAC in your EKS setup

OPT IT
------

290. [Unclear] What best practices do you follow when creating resources such as EC2, RDS, and
    "MANGODB"? [The final term is retained from the source.]

OPTUM
-----

291. How do I transfer payloads between lambda function in 2 different AWS account

292. How do you ensure the least privilege access to the IAM users

293. How do you ensure particular AMI image is present in AWS account using terraform

294. What are s3 bucket lifecycle policies?

ORION INNOVATION
----------------

295. You have multiple VPC, how will you connect them in AWS

296. An RDS is there in India region, I want to do read sync with RDS in London region, how will you
    implement it

OTHERS
------

297. When would prefer on-prem Kubernetes cluster over EKS and vice-versa

298. ECR

299. [Follow-up] What is the controller used to manage the self managed worker nodes

300. What is karpenter. On which metric does it scale up and down?

301. EKS cluster upgrade entire process

302. How would you set up a new environment on AWS and provision it with Terraform code?

303. How do you did cost optimization in AWS?

304. Explain on Lambda, CFT, Data Storage, S3?

305. How do you implement best security policies on AWS?

306. [Follow-up] Explain how you did your cloud migration

307. [Unclear] In CloudWatch, what is the use of log groups and "log trails"? [The term "log trails"
    is retained from the source.]

308. What is the purpose of creating S3 bucket policies?

309. How do you maintain the lifecycle of an S3 bucket?

310. What are Network ACLs and Security Groups, and how do they differ?

311. Explain EC2 instances and handling multiple VPCs.

312. How do you configure AWS RDS, and what factors do you consider (size, requirements, etc.)?

313. How much data is stored in your RDS MySQL?

314. How many masters and slaves are in RDS?

315. How comfortable with AWS and how much rate urself out of 5?

316. About IAM/Fargate/EC2/Lambda?

317. Can u pls write a lambda file?

318. Can u pls write terraform file to provision the Ec2 instance in a public subnet in a VPC?

319. R u using Dockerfile? u r build the dockerfile by codebuild?

320. How many NAT Gateways are needed for two public & two private subnets n a single VPC? Min &
    max?

321. How does SSL work (Certbot, Let's Encrypt, AWS)? Explain the Flow.

322. Scenario prompts on load balancers, Route 53, EKS, and database automation and administration.
    [The source lists these topics without the individual scenario questions.]

323. [Follow-up] How you ensure the best possible security for high availability architectures for 3
    tier applications.

324. Diff b/w SGs and NACLs.

325. What is VPC peering.

326. How do you scan the vulnerabilities specially for AWS instances.

327. Diff between IAM Users and Roles

328. Can you avoid the specific port traffic using SGs?

329. [Follow-up] How you connect to private instances when the SSH connection is not working?

330. Where do you use firewalls, SGs and NACLs

331. What are the security parameters we must consider while we are creating an EC2 instance for
    production?

332. How can you protect the data in an AWS instance?

333. How can you connect from AWS to on-prem servers?

334. Explain about the transit gateway and why do we use this?

335. I have created an EC2 instance named A, and I want to create another instance B. It should
    create an instance without deleting instance A. What can I do during this?

336. I have created an EC2 instance through Terraform. I don't have a backup of the Terraform state
    file, it is not in the remote state and locally not available. Now when I do apply, what can I
    do?

337. Suppose in your DevOps team, new team members are added to your team. How can you provide AWS
    access to your new users, what is the behavior of login to the console?

338. What is the difference between an EBS-backed instance and a non-EBS-backed instance?

339. What is the difference between EKS vs ECS vs Fargate?

340. AWS event bridge creation and setup via terraform

341. What are the services u were used in AWS

342. [Follow-up] If U want to design a infra for high scalablity, how did u do that?

343. What are NACLs,SecurityGroups,NAT Gateway

344. About Fargate

345. What is lambda functions? did u used any Lambda functions? what did u acheived from that?

346. Write a Terraform code to create multiple S3 buckets

347. [Follow-up] How you managed statefile

348. Have you worked on the AWS, right?

349. So how many types of policy, IAM policy are there? IAM policies?

350. So what is the difference between the S3 bucket policies and acls?

351. What is the dynamic auto scaling?

352. What is the difference between Security groups and NACL?

353. Two AWS accounts are in the same organization. Account A has an EC2 instance, and Account B has
    tokens. How would the EC2 instance access those tokens?

354. EC2 instance is unreachable, and it’s not a security group issue. What’s your next step?

355. An S3 bucket was made public by mistake. How do you secure and audit it?

356. RDS migration with minimal downtime – how would you approach it?

357. Terraform script to provision an EC2 instance with a custom security group and user data
    script.

358. Design a highly available backend on AWS – what services and architecture would you use?

359. VPC

360. How to host an S3 static website without enabling public Access

361. Difference between secret manager and parameter store

362. Difference between Iam users.. GitHub Oidc role and terraform io role.. which is secured and
    when to use use GitHub Oidc and when to use terraform io role

363. Write a terraform code to provision an Ec2

364. You are having lambda function and role everything setup perfectly but logs are not coming up
    in the cw group how to troubleshoot?

365. When to use Ec2 and when to use Lambda.. give scenario based answers

366. How DRS works.. explain the architecture

367. How failover and failback happens in DRS

368. How would you optimize AWS resource costs? Can you explain the methods you would use?

369. Create Terraform S3 resources, and ensure that the resource is deleted automatically after 7
    days?

370. What is IAM and how it works?

371. How did you migrate an application from an on-premise server to AWS?, Can you explain the
    process and the method you followed?

372. You have a microservices application that needs to scale dynamically based on traffic. How
    would you design an architecture for this using AWS services?

373. How do you configure a pipeline with AWS or Docker?

374. List the running instances across five AWS accounts.

375. What would you do if an EC2 instance were compromised?

376. What are the different types of IAM policies?

377. [Follow-up] How is the connectiivy from on prem to cloud

378. How to access s3 from vpc securely

379. Then cloudformation and terraform differences

380. How would you implement least privilege for CloudFormation stacks and Terraform?

381. List IAM users whose access keys have not been used for more than 90 days.

382. Delete inline policies for IAM users where "*" is specified in the policy.

PERFIOS
-------

383. Explain Amazon traffic architecture: how does traffic reach a private subnet? Draw a diagram
    and explain it.

384. What are ALB and NLB, and when should you use each one?

PERSISTENT SYSTEMS
------------------

385. ACM

386. CloudTrail

387. CloudFront

388. CloudFormation.

389. Cloud watch & how do you create a custom metric in AWS CloudWatch.

390. How do you login to the ec2 instance if you've lost the .pem key?

391. [Follow-up] Public and Private Subnet, what makes it public and private??

PLANSOURCE VALUELABS
--------------------

392. How many subnets you can add to a VPC

393. How to stream logs from docker conatiner to s3 from specific path within conatiner

394. Difference between Service and Task in ecs

395. How Autocaling happens in aws ECS

396. How you scale your EKS cluster based metrics/logs

397. Difference between ALB and ELB and comes under which layer, when to choose and why

398. How to spead up s3 upload with files in large size, and client uploaded 10 Gb file but failed
    after uploading 5 gb how you confirm that 5 gb is uploaded to s3

399. How do you optimize S3 cost

400. How do you make s3 secure which is have client sensitive data

401. How autoscaing happens with ALB

PUBLICIS GLOBAL DELIVERY
------------------------

402. [Unclear] How does authentication work in a Jenkins pipeline when using AWS with a particular
    login? [The source adds "if you have 1 logout?", whose meaning is unclear.]

QBURST
------

403. [Multi-cloud] What is major vpc difference between aws & gcp vpcs?

QENTELLI SOLUTIONS
------------------

404. Create s3 bucket with terraform

405. [Follow-up] If developer sets private subnet to public, what should you do?

406. KMS

407. ELB inflow and outflow

408. SG and NACL

409. How do you secure your environments in aws

410. [Follow-up] User not able to get ssh, what are troubleshooting steps to perfrom

411. What are metrics in cloudwatch you should focus on.

412. Your aws billing spikes, what should you check

413. What are security options in aws

SAPIENT
-------

414. In AWS, what all the services you have used?

415. In Route 53, what is the difference between an A record and a CNAME record?

416. [Follow-up] Purpose of DNS in your project.

417. [Follow-up] Have you created DNS record?

418. Can you tell about your VPC structure and networking architecture used in your project?

419. [Follow-up] How many subnets are you having?

420. In which subnet are you placing your EKS cluster and which networking components have you used?

421. [Follow-up] Why are you keeping your web application in a public subnet?

422. [Follow-up] Where Load Balancer will be there?

423. How many AWS storage services does it have?

424. EBS: Suppose you are maintaining data in EBS (sensitive data) — how are you securing those
    data?

425. [Follow-up] If your cluster is in a private subnet, then outside kubectl will not be working,
    right? How are you accessing that?

426. What is EBS and EFS in Kubernetes?

427. I have an S3 bucket, and there is some file inside it — my pod wants to access that S3 bucket.
    How will it access it?

SIGMOID
-------

428. [Follow-up] If LB is created in us-south and that region is down what will happen?

429. If RDS is hosted in us-west region and has read replicas spread across other region, due to
    some issue us-west region is down, how will you redirect your traffic. But having one more DB is
    expensive to the client

430. [Follow-up] Failover mechanism in LB

431. Is it possible to have ASG and LB in different regions?

432. Design a three-tier AWS architecture with a frontend, backend, and database, considering
    security, high availability, and low latency. Justify the optimized choice for each component,
    including a database in a StatefulSet versus RDS or another managed database, and a frontend in
    pods versus S3 with CloudFront.

433. [Follow-up] How can you tell a subnet is pubic or private, what things needs to be in place.

434. [Follow-up] How is an end user able to access the app which is running inside pods of private
    subnet nodes.

435. What is route53, how does the traffic actually flow in order, if a user requests or submits or
    posts anything from the UI.

436. Explain each component in your architecture which is involved from a user requesting from the
    UI to the request reaching the backend pod, and how they are connecting with each other to
    ensure inbound and outbound flow(including the firewall, NACl, security groups, route tables
    etc).

437. [Follow-up] If you used a service with type: LoadBalancer and as you told its in private
    subnet, then how its able to launch a loadbalancer in public subnet, how its getting access to
    do it from within private subnet. Which component of the control plane takes care of it.

438. Why did you choose eks cluster over ecs.

439. [Follow-up] How are you managing the cluster nodes.

440. For your specific architecture setup, how many ip addressess for all the components do you
    think will be sufficient enough from all the available IP's of both subnets within your VPC.

441. [Follow-up] Tell me how you handle the situation, if there is a DDoS attack in your cluster
    nodes or the cluster services, which is consuming all resources at 100% . What steps do you take
    in order to regain the condition back and what steps do you take to prevent this in future.

442. [Follow-up] How are you handling HA in your cluster, explain the reason why you use a specific
    feature over the other similar functionalities.

443. What is the requirement of NAT gateway in your cluster, where should it placed to make it work
    as intended.

444. Did you use AWS accounts? How did you handle the segregation of various environments within
    AWS.

445. There was an issue last time with some config changes in TF resource which had lead to slight
    downtime of the AWS resource during apply. What might have caused this downtime, how do you
    handle this in terraform for futurue config changes, for ensuring least possible downtime and
    maximum possible availability. Write a terraform code to handle this situation.

446. Explain the S3 lifecycle. What are classess in S3.

447. Did you work on cloudfront. How did you leveraged this for exposing frontend static code to
    serve the UI of the app.

SONATA SOFTWARE
---------------

448. What are the AWS services that you have worked on?

449. Can you tell me what is cold start in Lambda?

450. Have you worked on API Gateway?

451. Can you tell me the difference between REST APIs and WebSocket APIs in API Gateway?

452. How do you protect an API Gateway?

453. Can you tell me the difference between NAT Gateway and Internet Gateway?

454. Say if you have both explicit deny and allow policy to a user in AWS IAM, what happens?

455. Can you tell me the difference between managed policy and inline policy in IAM?

456. How do you retrieve secrets from AWS Secrets Manager using Python?

457. How do you protect the secrets in Secrets Manager?

458. How do you encrypt secrets in AWS Secrets Manager?

459. Say you need to configure EC2 instances automatically or replace themselves automatically when
    they fail. How do you implement this?

460. Have you worked on AWS Auto Scaling?

461. What is the use case of Auto Scaling?

462. Will Auto Scaling automatically scale up? How do you set that up?

463. Say you have EC2 instances running web servers and need deployment with minimal downtime during
    updates. How do you approach this?

464. Can you tell me the difference between traditional RDS and Aurora?

465. How does Aurora handle automatic backups?

466. Have you worked on AppConfig?

467. Say I created an S3 bucket using Terraform and want to modify the bucket name. Is it possible?
    How would you do this?

SONY
----

468. How would you design a system to survive if one AWS region goes down?

469. How would you investigate a sudden three‑times increase in the AWS bill?

470. If I want to start an ec2 instance once the cpu is utilised 80% what terraform code will you
    write and it also should copy the image from S3.

471. How would you manage Terraform when multiple teams deploy to the same AWS account but must not
    overwrite each other’s resources?

SQUAREOPS
---------

472. [Multi-cloud] Is everything on AWS or multi-cloud?

473. What exactly do you do in AWS Cloud in this project?
    Follow-up questions:
    - Do you manage VPC/subnets/security groups?
    - Do you create EC2/EKS/ECS resources?
    - Do you manage S3 lifecycle policies?
    - Do you handle IAM + RBAC in Kubernetes?
    - Do you participate in cost-optimization?

474. On which compute platform are the applications hosted?
    Follow-up questions:
    - Why EKS over ECS?
    - How do you manage worker nodes?
    - How many replicas do you run?
    - Do you use Cluster Autoscaler / Karpenter?

475. Have you created an EKS cluster? Explain the process.
    Follow-up questions:
    - Did you use console, CLI, or Terraform?
    - Which VPC/subnet configuration did you use?
    - How did you configure node groups?
    - How do you bootstrap kubectl access?

476. Have you upgraded an EKS cluster before? How?
    Follow-up questions:
    - What risks come with upgrading?
    - How do you handle node draining?
    - Do deployments get recreated?
    - What checks do you perform post-upgrade?

477. If your teammate also wants access to the same cluster through kubectl, what steps do you
    follow?
    Follow-up questions:
    - What IAM policy do you attach?
    - What is aws-auth ConfigMap?
    - Where is aws-auth stored?
    - How do you map users and roles?
    - Do you provide RoleBinding or ClusterRoleBinding?

478. In which Kubernetes resource do you map IAM users/roles?
    Follow-up questions:
    - What sections inside it? (mapUsers, mapRoles)
    - What mistakes can break authentication?

479. How do you securely inject sensitive data into Helm?
    Follow-up questions:
    - Do you use AWS Secrets Manager?
    - Do you avoid committing secrets in values.yaml?
    - What is Sealed Secrets?
    - What is the purpose of --set and --set-file flags?

480. How to implement shared storage across multiple pods running on multiple nodes in EKS?
    Follow-up questions:
    - Why EFS over EBS?
    - What is the EFS CSI driver?
    - What are the access modes?
    - How do you mount PVC in deployment?

481. If there is one node but multiple pods, can we use EBS for shared storage?
    Follow-up questions:
    - What access mode does EBS support?
    - Why can't EBS work across multiple nodes?
    - When is EFS mandatory?

482. Why choose EFS over EBS?
    Follow-up questions:
    - Which one supports multi-node?
    - Which one is cheaper?
    - Which one is faster?

483. Can EBS be attached to multiple nodes? Why not?
    Follow-up questions:
    - Explain RWO vs RWX
    - Who enforces the mount restriction?

484. How do you deploy to EKS through GitHub Actions?

485. Which AWS services do you have the most hands-on experience with?
    Follow-up questions:
    - EC2?
    - IAM?
    - VPC?
    - S3?
    - RDS?
    - CloudWatch?
    - Have you worked on cost optimization?

486. Create an EC2 IAM role that allows access only to S3 + DynamoDB and denies access to all other
    services.
    Follow-up questions:
    - What is the logic behind a custom policy?
    - What is "Explicit Deny"?
    - How would you restrict everything except two services?
    - Which policy pattern do you use (Allow + NotAction Deny)?

487. What exact cost optimization steps have you implemented?
    Follow-up questions:
    - What was the % saving?
    - Have you used Reserved Instances or Savings Plans?
    - Do you know cost of ALB vs private ALB?
    - Which scenario costs more?

488. Two apps in same VPC → each behind a public ALB → if App A calls App B, how does traffic flow?
    Follow-up questions:
    - Does it go out to internet?
    - Does it come back through IGW?
    - Which option is costlier, public or private ALB?
    - What stays inside VPC vs what leaves?

489. How do you create auto scaling policies based on memory & disk usage?
    Follow-up questions:
    - Are memory and disk metrics available by default?
    - Why do we need CloudWatch Agent?
    - How do you configure the agent?
    - Where do you create CloudWatch alarms?
    - Do you need to update Launch Template?

490. In a versioned bucket, how do you delete objects + all older versions after 10 days?
    Follow-up questions:
    - What options appear in lifecycle rules?
    - Do we explicitly delete previous versions?
    - What is the difference between Current vs Previous versions?

491. App is slow → you suspect RDS. What do you check?
    Follow-up questions:
    - CPU? Memory? Latency? IOPS? Connections? Disk queue depth?
    - Slow query logs? Error logs? Performance Insights?

492. You found memory pressure on RDS. You cannot resize. What immediate action can you take without
    downtime?
    Follow-up questions:
    - Can you kill heavy queries?
    - Remove idle connections?
    - Create a read replica?
    - Which action applies instantly?
    - Which action causes 0 downtime?

493. Request comes from Internet → enters VPC through IGW → what is the first security layer? NACL
    or SG?
    Follow-up questions:
    - Why NACL first?
    - Which one is stateless?
    - Which one is stateful?
    - Which takes precedence if conflict?

494. If NACL denies a CIDR, but SG allows same IP, can the IP access LB?
    Follow-up questions:
    - Why not?
    - Which one checks traffic first?
    - How many IPs does /32 allow?
    - Does IP X fall in CIDR Y?

495. [Follow-up] Does IP 10.11.7.44 fall under 10.11.0.0/16?

496. [Follow-up] Does IP 10.11.44.76 fall under 10.1.0.0/16?

497. [Follow-up] What does /32 represent?
    Follow-up questions:
    - How many IPs in /32?
    - Which exact IP?
    - How to calculate whether an IP is inside a CIDR block?

498. Terraform generated RDS password, you didn’t save it. Can you retrieve it?
    Follow-up questions:
    - Where does Terraform store generated values?
    - Local state or remote backend?
    - Why is storing secrets in plaintext dangerous?

SYNCORTEX
---------

499. How will you make sure EC2 is not deleted while running destroy command.

SYNECHRON
---------

500. Tell me the difference betweeen Cloud watch and CloudFormation?

501. [Follow-up] How will you create the Custom alerts, tell me the procedure.

502. What is ACM and S3?

503. How does an AWS CodePipeline differ from a Jenkins pipeline? Can you give an example of when
    you would choose one over the other?

TCS
---

504. What types of nodes did you deploy on AWS?

505. What is the difference between Interface Endpoint and Gateway Endpoint in AWS?

506. How do you restrict access to AWS resources for a specific user?

507. How do you restrict a user to only EC2 and RDS access?

508. When you create a VPC, what default components are added?

509. Explain the AWS architecture shown in the diagram, involving CodePipeline, CodeBuild,
    CodeDeploy, CloudFormation, and CloudWatch. [The referenced diagram is not present in the
    attached text.]

TURNING
-------

510. Explain Lambda cold starts.

511. AWS CDK commands

512. If someone manually changed the EC2 config, which was created through TF, how to fix it?

513. I've payment gateway app running in Lambda, sometimes there was an issue with connecting to
    external API, how to cross check and fix it while performing the payment

UST
---

514. Write Terraform code to create an AWS EC2 instance and include variables for instance_type and
    region.

515. [Follow-up] You need to create 50 instances in one go. How will you create them in Terraform?

VERIZON
-------

516. Disadvantage of using ebs volumes in eks

517. AWS secret manager vs parameter store

518. Other ways to connect EC2 without pem key

519. Horizontal and vertical scaling RDS db steps

520. Db RDS multiavailability vs read replicas

521. How to delete old/untaged images in ECR

VIRTUSA
-------

522. Suppose there are multiple ec2 instances manually created via console, I have to update those
    ec2 instances via terraform, what is the command

523. Suppose you are having an ecs task definition, I have an website, the developer is hitting the
    url or website, what will be the flow of traffic, fargate is a serverless – how do you achieve
    configuration DNS or website over there

524. What is task definition in ecs

525. Difference between cloud formation and terraform

526. What is the difference between ecs and fargate

WIKREATE MEDIA
--------------

527. Use of Route53

528. Difference between S3 and EBS

WIPRO
-----

529. Could you elaborate your experience with automating and optimizing the deployment over large
    infrastructure using AWS and other tools like Terraform and Ansible from your previous roles?

530. [Multi-cloud] What is Cloud-agnostic strategies? how do you leverage conditionals to make a
    role cloud-agnostic, particularly for environments like AWS, Azure and GCP.

531. Diff bet Nat gateway and igw

532. Explain command to use s3 native lock

533. Application on AWS EC2 behind the loadbalancer suddenly unavailable, how do you troubleshoot

534. AWS billing increased suddenly, how do you identify the costs

535. A developer asking Ec2 instance for his local deployment, how you will achieve and what type of
    instance you create and give it to them.

536. In AWS how do you configure subdomains (like godady/bigrock)

ZS ASSOCIATES
-------------

537. Calico and VPC CNI plugin difference. Why one is preferred over other. How would they help in
    setting up networking for pod.

538. How is an ip address allocate to a pod. Does CNI plugin use same CIDR range which is provided
    by VPC or different?

539. How would you provision karpenter. What all things are needed in configuration

540. Can you deploy mongo db database in EKS cluster. If yes how and what all configuration things
    you would need to keep in mind

541. [Follow-up] There are 3 backend pods in 3 different region. If one pod goes down how would the
    request be managed

542. How would karpenter know which node to provision. How would it get to know about resource
    constraints

543. A backend pod needs to interact with S3 and lambda. How would you achieve it

544. [Follow-up] How would the service account know which role to assume. What all things you would
    need to configure in the service account

545. Argo CD. How do you manage CI/CD in your organisation. How many EKS clusters you manage, no of
    nodes

546. If you want to deploy EC2 instances in 3 different region what would the terraform code
    structure look like

547. Want to create a module for EKS cluster. What would be the structure

ZOPSMART
--------

548. What is IAM Role

549. What is VPC Peering?

550. You are using jenkins server as open source s/w tool like in aws service which service is
    available to implement CICD Pipeline

551. What is the differance between EBS and EFS

552. You are using K8S Cluster as open source s/w tool like in aws service which service is
    available to create K8S Cluster

