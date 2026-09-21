AZURE CLOUD INTERVIEW QUESTIONS
Company-wise extraction

Source: interview questions company wise.rtf
Total: 157 prompts across 22 named companies plus the source's "Others" category.

Scope: Azure cloud services, AKS, Azure DevOps, Terraform on Azure, and
multi-cloud scenarios involving Azure.

Spelling and grammar have been lightly edited for readability. Questions
repeated across interviews are retained under their original company names.
No answers have been added. Source premises and references such as "recent"
are retained as interview wording.

Labels:
  [Follow-up] A general question kept from an Azure interview context.
  [Multi-cloud] A question involving Azure together with another cloud.
  [Azure portion] Only the Azure-related part of a mixed source prompt.
  [Incomplete] or [Unclear] The source wording is incomplete or ambiguous.

COMPANY INDEX
  Alphadyne: 1
  Blue Yonder: 8
  CTS (Cognizant): 2
  Capgemini: 3
  EPAM: 11
  Elixr Labs: 20
  Encora: 2
  HCL: 1
  Hexaware: 4
  JPMorgan: 10
  LTIMindtree: 4
  NUOS INFO Systems: 10
  Nextturn: 1
  Nice: 5
  Orion Innovation: 2
  Others: 23
  RelevantZ: 8
  Sonata Software: 12
  Synechron: 11
  TCS: 6
  Volkswagen Group Digital: 2
  Wipro: 2
  Zensar: 9

ALPHADYNE
---------

1. Terraform coding task on Azure: define a VNet with a given IP range; create separate subnets for
    the bastion, application, and database layers; configure NICs and attach them to the correct
    subnets; provision VMs using those NICs with username/password authentication; create public IPs
    where required; set up Azure Bastion for secure VM access; and configure an Azure Load Balancer
    to route traffic to the application VMs.

BLUE YONDER
-----------

2. How do you use Azure Key Vault secrets in AKS?

3. How will an application, container, or pod fetch the latest rotated secrets from Azure Key Vault?

4. How do you integrate SonarQube and Snyk into an Azure pipeline?

5. How do you secure your AKS cluster?

6. How do you make your AKS cluster highly available?

7. How do you perform cost optimization in AKS?

8. [Follow-up] What types of HPA triggers have you used so far?

9. How do you integrate Microsoft Entra ID with AKS for authentication?

CTS (COGNIZANT)
---------------

10. The backend.tf file is present in the repository but is not showing in the storage account. What
    could be the issue?

11. What is Azure Application Gateway, and how does it encrypt HTTP/HTTPS traffic?

CAPGEMINI
---------

12. How do you pass variables in an Azure pipeline? How do you parameterize the pipeline?

13. [Follow-up] What is an availability zone? Explain its physical layout in depth.

14. How can MySQL interact with Azure Key Vault privately, without any public connectivity?

EPAM
----

15. [Multi-cloud] How do you secure cloud-native DevOps infrastructure using identity federation,
    such as Azure AD and AWS IAM?

16. [Multi-cloud] How do you securely set up workload identity federation between GitHub Actions and
    Google Cloud or Azure?

17. What are availability sets?

18. [Follow-up] Two sites are connected through a site-to-site VPN, and the connection breaks. How
    would you troubleshoot it?

19. What are the features of Network Watcher? Can it be used globally?

20. How do you configure monitoring through Log Analytics?

21. Write a Terraform configuration that deploys Azure storage accounts based on the environment: 2
    storage accounts for dev and 5 storage accounts for prod.

22. [Follow-up] How do you write variables in a pipeline?

23. [Follow-up] How do you use Terraform in a pipeline?

24. What is the difference between classic and YAML pipelines?

25. [Incomplete] Original fragment: "whats the difference between service end". The comparison is
    incomplete in the source.

ELIXR LABS
----------

26. Which basic Azure services would you consider?

27. Explain hub-and-spoke topology.

28. [Follow-up] What deployment model do you use: IaaS or SaaS?

29. Choose one requirement and explain how you implemented it end to end using Azure services.

30. A three-tier .NET application uses Azure App Service for Web APIs, Function Apps for background
    jobs, and Service Bus for messaging. How would you configure connectivity between these
    services?

31. [Follow-up] How would an on-premises user access an application or code deployed in the cloud?
    How would you configure connectivity from on-premises to the cloud?

32. As an Azure specialist, would you recommend a site-to-site VPN or ExpressRoute?

33. A banking customer has a limited budget. How would you justify choosing between a site-to-site
    VPN and ExpressRoute when discussing cost and security?

34. In which scenarios would you use a site-to-site VPN, and in which would you use ExpressRoute?

35. [Follow-up] Which storage-related components have you worked with?

36. A storage account receives continuously recorded data, 24/7, and users also retrieve that data.
    As the data grows daily, what storage-account cost optimization techniques would you implement?

37. If a storage account was created with the hot tier, can it be changed to the cool tier?

38. Manually changing from the hot tier to the cool tier takes time when there is a large amount of
    data. How would you handle this at scale?

39. [Follow-up] What challenges or issues have you faced while taking backups?

40. Which backup services have you used? Were they Azure-specific or on-premises services?

41. [Follow-up] For which scenarios have you configured disaster recovery?

42. [Follow-up] [Incomplete] Original fragment: "how about the monitoring techniques like any alerts
    we can". The question is incomplete in the source.

43. A large production environment runs on Azure, with customer-defined alert thresholds generating
    hundreds of alerts. How would you identify and prioritize the alerts that need attention?

44. Have you performed migrations from on-premises to Azure or from Azure to Azure? What scenarios
    were involved, and were these resource migrations or data migrations?

45. [Follow-up] What prechecks should you consider before migration?

ENCORA
------

46. How do you rotate secrets in Key Vault and implement a .pfx certificate in Application Gateway
    together with an ingress/controller in AKS?

47. What is a service principal in Azure? Provide an example.

HCL
---

48. How do you integrate Azure Key Vault with Jenkins or an Azure pipeline?

HEXAWARE
--------

49. [Follow-up] Write the overall YAML structure of a CI/CD pipeline.

50. What is Storage Explorer?

51. [Follow-up] How do you set approvals in a CD pipeline?

52. What is a variable group in Azure DevOps?

JPMORGAN
--------

53. An application deployed to Azure Kubernetes Service (AKS) randomly fails health checks. How
    would you debug this end to end?

54. How do you ensure secure and dynamic secret rotation in Azure DevOps pipelines?

55. How would you use Azure Application Gateway with Web Application Firewall for a sensitive
    banking application?

56. During an Azure deployment, you encounter intermittent DNS resolution issues. What could cause
    them?

57. An application running on AKS experiences 10-second delays every 15 minutes, with no code
    changes. How would you begin root cause analysis?

58. An Azure Function is being throttled. How would you detect and fix it?

59. Define a plan for blue/green deployment with rollback on Azure using Terraform and pipelines.

60. How do scaling strategies differ for compute-intensive and I/O-intensive workloads in Azure?

61. [Multi-cloud] Are you aware of the recent AWS and Azure outages? What were your key takeaways
    from those incidents?

62. [Multi-cloud] If an entire region goes down and even a multi-cloud setup such as AWS and Azure
    experiences outages, how would you protect the data from loss? What strategies would you use for
    reliable backups and recovery?

LTIMINDTREE
-----------

63. [Azure portion] Write a Terraform script to create an App Service in Azure.

64. [Azure portion] Create three different container images, store them in Azure Container Registry
    (ACR), and deploy them to AKS. Write the Kubernetes YAML files.

65. Application Gateway returns a 404 error even though the backend services are healthy. How would
    you troubleshoot it?

66. What is the difference between a firewall and an NSG?

NUOS INFO SYSTEMS
-----------------

67. What happens to the Terraform state file if someone deletes resources from Azure?

68. [Follow-up] If the pipeline fails because resources already exist, how would you handle RIP:
    Remove, Import, Plan?

69. How do you export Azure resources into Terraform code?

70. How do you enforce Azure Policies, such as tag or location restrictions, using Terraform at
    scale?

71. Logs are incomplete. How would you troubleshoot across AKS, ingress, the application, and
    infrastructure?

72. How do you monitor Azure VM memory and alert when it exceeds 80%?

73. Which tools would you recommend for CI/CD, artifact storage, vulnerability scanning, and
    container registry in a hybrid on-premises and Azure setup?

74. How do you assess Azure DevOps migration readiness and plan the transition?

75. [Multi-cloud] How do you manage AWS and Azure through a single DevOps process, with a focus on
    security and cost?

76. How would you use the Azure DevOps REST API to apply a security policy to all repositories
    programmatically?

NEXTTURN
--------

77. Explain variable groups, environment variables, and secrets in Azure DevOps.

NICE
----

78. What is the advantage of using a YAML file over classic build pipelines in Azure DevOps?

79. [Follow-up] What additional advantages of YAML pipelines have you experienced personally?

80. Do you have experience using Azure Key Vault?

81. Have you integrated Azure Key Vault into your pipelines or branches in a project?

82. Did you create the Azure Key Vault access policies yourself, or did someone else create them?

ORION INNOVATION
----------------

83. An Application Gateway TLS certificate has expired. What steps would you follow to renew it?

84. Does Application Gateway operate at Layer 7 or Layer 4?

OTHERS
------

85. What is the difference between an NSG and a firewall?

86. Can two VMs in different VNets communicate with each other?

87. Explain private endpoints.

88. Explain ExpressRoute in Azure.

89. How do you build a container image and push it to ACR?

90. [Follow-up] How do you reference that existing image in a YAML file to deploy a pod?

91. What is the difference between an Azure managed identity and a service principal? How would you
    explain it in an interview?

92. Which metrics would you use to create alerts for high VM CPU and memory usage? What is an action
    group? Explain alert creation step by step, give basic troubleshooting KQL queries for a Log
    Analytics workspace, and describe any monitoring automation you have done with scripts.

93. What is the difference between build artifacts and pipeline artifacts? Which is better?

94. [Multi-cloud] Explain the pipeline steps to automatically move a file from Azure Blob Storage to
    Google Cloud Storage.

95. Why are dynamic blocks used in Terraform? Write the skeleton for an Azure resource using a
    dynamic block.

96. How do you use a subnet ID output from a VNet module as an input to a VM module?

97. [Follow-up] If a team member deletes a pipeline, how would you recreate it and prevent the
    scenario in the future?

98. What is the difference between a stakeholder and an administrator in Azure DevOps?

99. [Follow-up] How would you build one or a small number of reusable pipeline templates for 50
    different applications?

100. [Follow-up] What command or pipeline syntax is used to reference the output variable of a
    previous stage in the current stage?

101. [Follow-up] How is sensitive data managed in pipelines?

102. How are authentication and networking established between an Azure DevOps pipeline and Azure
    Key Vault?

103. How would you allow only one pod or application to access a storage account while restricting
    all other pods in AKS?

104. An AKS cluster has 32 GB of memory, of which 30 GB is already used. Can a new pod with a 500 MB
    memory request and a 4 Gi memory limit be scheduled in the same cluster using HPA/VPA?

105. Which Application Gateway setting is used to upload an SSL certificate, and why?

106. [Follow-up] What alternative ingress controllers would you suggest, as NGINX IGC is deprecated?
    [Deprecation premise retained from the source.]

107. How are pipeline logs stored in Azure DevOps?

RELEVANTZ
---------

108. Which Azure DevOps tools have you used, and where do you store the output of a CI pipeline,
    such as Azure Artifacts?

109. What steps do you create in an Azure pipeline?

110. How do you store secrets in Azure DevOps?

111. What is a Log Analytics workspace used for?

112. What methods are available to enable communication from one subscription to another?

113. Data in Azure Data Factory (ADF) changes dynamically. How would you retrieve, analyze, and
    present it, potentially through a pipeline?

114. What do you use an Azure Recovery Services vault for?

115. [Follow-up] How do you take backups? Explain some backup strategies you have used.

SONATA SOFTWARE
---------------

116. What is the difference between Application Gateway and Front Door?

117. How do you protect endpoints in AKS?

118. What networking do you use in AKS?

119. [Follow-up] How do you perform cost optimization in the cloud?

120. How do you block a particular domain in Application Gateway?

121. [Follow-up] How do you monitor pods going down?

122. [Follow-up] Which metrics do you use when CPU or memory usage on a VM exceeds 75%?

123. Where do you store the TLS/SSL certificate used by Application Gateway?

124. How long would you take to write infrastructure-as-code and deploy to App Service?

125. How do you build a CI/CD pipeline in Azure DevOps?

126. [Follow-up] A SQL database has CPU usage above 75%. How would you upgrade it?

127. Do you use Helm charts for AKS deployments?

SYNECHRON
---------

128. What is Azure Boards, and what does it contain?

129. Share your screen and write the structure of an Azure pipeline.

130. How do you troubleshoot a failed pod in AKS? Provide the commands.

131. What are the different types of Azure Storage?

132. How do you store credentials in Azure pipelines?

133. What are the different types of subscriptions in Azure?

134. How would you implement a DC/DR setup in Azure? Which services would you use?

135. [Unclear] Would you register App Service first or deploy it? [The source's intended meaning of
    "register" is unclear.]

136. What is Azure Artifacts?

137. What are self-hosted agents and Microsoft-hosted agents?

138. [Azure portion] How do you configure SonarQube with Azure?

TCS
---

139. What are deployment groups in Azure DevOps?

140. [Follow-up] How do you set approvals in a pipeline?

141. What is the difference between Microsoft-hosted agents and self-hosted agents?

142. [Follow-up] Explain how a pipeline is triggered across different environments.

143. What is the difference between a classic pipeline and a YAML pipeline?

144. [Follow-up] Explain the pipeline steps for an Angular, Java, or .NET application.

VOLKSWAGEN GROUP DIGITAL
------------------------

145. Which Azure services do you use?

146. Which additional services would you like to learn in Azure DevOps?

WIPRO
-----

147. How is Azure Key Vault integrated into CI/CD?

148. [Multi-cloud] What are cloud-agnostic strategies? How would you use conditionals to make an
    Ansible role cloud-agnostic across AWS, Azure, and GCP?

ZENSAR
------

149. What is a service connection or connection string?

150. How do you set alerts in Azure Monitor? Explain the steps and configuration.

151. What are the types of VNet peering?

152. [Follow-up] Which types of pipelines do you use? How many types of pipelines are there?

153. [Follow-up] What is a variable in a pipeline?

154. [Follow-up] Can multiple jobs run in parallel through a single pipeline? How?

155. [Follow-up] What is an agent?

156. How do you create an app registration?

157. [Unclear] For cost optimization, would you use Azure Application Gateway or a "network
    gateway"? [The source does not specify what it means by "network gateway".]

