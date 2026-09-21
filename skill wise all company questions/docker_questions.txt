DOCKER INTERVIEW QUESTIONS
Company-wise extraction

Source: interview questions company wise.rtf
Total: 167 numbered questions, tasks, and topic prompts across
48 named companies plus the source's "Others" category.
Subquestions and requirements under one scenario count as part of that scenario.

Scope: Docker fundamentals, Dockerfiles, image builds, layers and caching,
multi-stage builds, networking, volumes, Compose, Swarm, security, registries,
container troubleshooting, and Docker-related CI/CD and deployment scenarios.

Questions are grouped under their original company names. Wording has been
lightly edited for readability. Repeated questions from separate interviews
are retained. No answers have been added. Scenario premises come from the source.

Labels:
  [Related] A container, registry, cloud, CI/CD, or orchestration question
            related to Docker, though not necessarily specific to Docker.
  [Relevant portion] Only the Docker/image-related part of a mixed prompt.
  [Unclear] Source wording or exercise requirements are incomplete or ambiguous.

COMPANY INDEX
  AMEX: 1
  Accion Labs: 1
  Akamai: 1
  Amazon: 1
  Arrise Solutions: 7
  BMW TechWorks: 1
  CGI: 2
  CTS (Cognizant): 1
  Deloitte: 7
  EPAM: 7
  E&Y: 1
  Emphasis: 3
  Flentas: 3
  HCL: 4
  Infosys: 5
  Intact Green Services: 2
  JPMorgan: 2
  Koerber Pharma: 2
  LTIMindtree: 5
  L&T: 5
  Marsh McLennan: 5
  Moodys: 1
  NPCI: 3
  NUOS INFO Systems: 2
  Nextturn: 3
  Nice: 3
  Nisum Technologies: 2
  Nitor Infotech: 1
  OPT IT: 2
  One2N: 1
  Oracle: 2
  Others: 31
  Plansource ValueLabs: 1
  Qburst: 3
  Rapidsoft: 1
  SAP: 1
  Sigmoid: 7
  Sony: 2
  Syncortex: 1
  Synechron: 5
  TCS: 5
  Techdome: 3
  Verizon: 3
  Virtusa: 3
  Volkswagen Group Digital: 1
  Wipro: 5
  ZS Associates: 3
  Zensar: 2
  ZopSmart: 4

AMEX
----

1. What is the difference between Docker and Kubernetes?

ACCION LABS
-----------

2. Can you explain your hands-on experience with Docker?

AKAMAI
------

3. What is the difference between CMD and ENTRYPOINT?

AMAZON
------

4. [Related] How do you build an image during CI, and how do you manage it?

ARRISE SOLUTIONS
----------------

5. Explain layers in Docker.

6. What is the difference between using a VM and using Docker?

7. What are the types of storage drivers in Docker?

8. What are the types of networking in Docker? Explain them in detail.

9. Does Docker have its own kernel?

10. [Unclear] Explain "C name" and namespaces in Docker. [The meaning of "C name" is unclear in the
    source.]

11. Which network would you use to isolate communication between two containers?

BMW TECHWORKS
-------------

12. [Related] Have you containerized any application?

CGI
---

13. [Unclear] Sample Dockerfile exercise. [The source does not include the exact question or
    requirements.]

14. Write a sample multi-stage Dockerfile.

CTS (COGNIZANT)
---------------

15. How did you reduce the sizes of Docker images?

DELOITTE
--------

16. [Related] [Relevant portion] What image-pull errors, such as ImagePullError, have you faced in
    Kubernetes, and how did you resolve them?

17. What are the stages in a Docker image build? Why do we use ENTRYPOINT and CMD instructions?

18. Which container registry do you use for storing Docker images?

19. Are you aware of security scanning tools? How do you scan Docker images—both during build and at
    the registry level? Are you using any extensions or tools for image scanning?

20. How do you pass environment variables during Docker build commands? What services do you use for
    storing Docker images?

21. A Dockerfile describes a Tomcat application running on port 8080. How would you build the image
    and run it as a container exposing port 9090?

22. A Dockerfile describes a Tomcat application running on port 8080. How would you build the image
    and run it as a container exposing port 9090?

EPAM
----

23. [Related] [Unclear] How does Lambda work with containers? [The original wording, "how lamda
    works containers", is abbreviated.]

24. What is the difference between COPY and ADD?

25. What is the difference between CMD and ENTRYPOINT?

26. Explain the run and exec commands.

27. [Related] What's your strategy for managing container image security across all stages of a
    DevOps pipeline?

28. Explain Docker networking.

29. What is the difference between ARG and ENV in Docker?

E&Y
---

30. What is Docker used for?

EMPHASIS
--------

31. Explain how to write a Dockerfile.

32. From where is the image pulled when you run docker pull?

33. How do you pull an image from a private repository?

FLENTAS
-------

34. [Related] What is the difference between a Pod and a Container?

35. What is Docker?

36. What is the difference between an image and a container?

HCL
---

37. What is inside a Dockerfile?

38. What is a multi-stage Docker build, and why is it used?

39. Can you write a basic Dockerfile for your application?

40. Can you explain Docker Compose and how it helps in multi-container application deployments?

INFOSYS
-------

41. Write a sample Dockerfile.

42. What is the difference between CMD and ENTRYPOINT?

43. What is Docker, and how do you use it in your project? Have you written a Dockerfile?

44. How do you reduce Docker image size?

45. What difficulties have you faced while building a Docker image?

INTACT GREEN SERVICES
---------------------

46. A Docker image is causing issues because of its size. What steps would you take to reduce its
    size?

47. You cannot push a Docker image to Docker Hub because of an access issue. Where else could you
    push the image?

JPMORGAN
--------

48. [Related] How do you provision container-based services for microservices deployment?

49. [Related] What’s your approach to disaster recovery for stateful apps running on containers?

KOERBER PHARMA
--------------

50. What is the difference between CMD and ENTRYPOINT?

51. Explain Docker volumes and docker prune.

LTIMINDTREE
-----------

52. Can Docker be installed inside a container?

53. Explain the Dockerfile.

54. [Related] Create three different container images, store them in ECR or ACR, and deploy them to
    EKS or AKS. Write the Kubernetes YAML files.

55. How do you provide security in Docker?

56. [Related] How do you implement blue-green or canary deployments using container orchestration?

L&T
---

57. What is Docker Compose?

58. What is a Dockerfile?

59. What is the difference between virtualization and containerization?

60. What is depends_on in Docker Compose?

61. [Related] Can a container restart itself? Explain how.

MARSH MCLENNAN
--------------

62. How do you reduce the size of a Docker image?

63. What is a multi-stage Docker build? How does it help reduce image size?

64. What is Docker image layer caching?

65. How do you implement Docker image layer caching?

66. Do you use any tool for Docker image layer caching? If yes, which one?

MOODYS
------

67. How do you manage and version Docker images stored in Amazon ECR?

NPCI
----

68. An application is accessible from outside its container, but there is packet loss when accessing
    it from inside. How would you troubleshoot this?

69. How do you retrieve logs at the Docker level?

70. One container runs the frontend application, and another runs the database. How would you ensure
    that the database starts before the frontend in this two-tier application?

NUOS INFO SYSTEMS
-----------------

71. Write a multi-stage Dockerfile for a Node.js application, removing secrets and unnecessary
    layers.

72. [Related] [Relevant portion] Which tools would you recommend for vulnerability scanning and a
    container registry in a hybrid on-premises and Azure setup?

NEXTTURN
--------

73. [Related] [Relevant portion] How would you troubleshoot a Kubernetes pod stuck in
    ImagePullBackOff?

74. Explain Docker layer caching. During a Docker build, if layers 1–10 are already cached and you
    modify Layer 5, what happens to Layers 6–10? Will Docker reuse the cache or rebuild them?
    Explain why.

75. [Related] In a Jenkins pipeline, at which stage would you publish or push artifacts/images to
    Nexus or Artifactory—pre-build, build, or post-build? Why?

NICE
----

76. What is the difference between CMD and ENTRYPOINT in Docker?

77. What is the difference between ADD and COPY?

78. [Related] How do you design and manage a containerized environment to ensure scalability and
    high availability?

NISUM TECHNOLOGIES
------------------

79. What is the difference between COPY and ADD commands?

80. How do you fix security issues in Docker images?

NITOR INFOTECH
--------------

81. [Related] How would you reduce pipeline runtime? Discuss multi-stage builds.

OPT IT
------

82. [Related] How do you containerize your application?

83. Write a Dockerfile for a Java application.

ONE2N
-----

84. [Related] Design a three-tier application architecture, considering Docker Swarm or Kubernetes.
    Follow-up questions:
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

ORACLE
------

85. Explain all the steps involved in building a multi-stage Docker image.

86. What layers are created when building a Docker image?

OTHERS
------

87. [Related] Explain Amazon ECR.

88. If Docker containers are consuming too much disk space, how do you fix it?

89. Do you use a Dockerfile? Do you build the image with CodeBuild?

90. How Docker is operable on a Linux machine? Explain the docker architecture components

91. How to roll back a failed deployment in Docker & K8s?

92. How do you reduce the size of a Dockerfile?

93. What is the difference between COPY and ADD command in Docker File.

94. Difference between entry point and CMD in Docker File.

95. [Related] Build a container image and push it to ACR. How would you reference that image in a
    Kubernetes YAML file to deploy a pod?

96. Write a Dockerfile.

97. Explain a multi-stage Dockerfile.

98. How do you authenticate Jenkins when pushing a Docker image to a registry?

99. Jenkins is failing to push a Docker image to the registry. How do you troubleshoot?

100. What is a Dockerfile, and what does it contain?

101. Why would you choose Kubernetes instead of Docker Swarm?

102. Explain a project in which you used Docker, Kubernetes, and CI/CD.

103. [Unclear] A Docker image is configured with port 8080, and the container/application uses
    another port. What would happen? [The application port is not specified in the source.]

104. Write a simple Dockerfile.

105. What is the difference between ADD and COPY in a Dockerfile?

106. What is the difference between CMD and ENTRYPOINT in a Dockerfile?

107. Write a Dockerfile and explain it.

108. How would you expose your application in Docker?

109. [Related] Suppose you have created a CI/CD process. After building the image, manual
    intervention is required. How would you configure it, and where?

110. How do you reduce the size of a Docker image?

111. What is a base image in a Dockerfile?

112. Can we write a Dockerfile without a base image?

113. Write the structure for building and pushing a Docker image for an application in GitHub
    Actions.

114. How do you check the integrity of a Docker image or file?

115. How do you configure a pipeline with AWS or Docker?

116. [Relevant portion] Which Docker commands do you use?

117. [Related] How do you troubleshoot an ImagePullBackOff error?

PLANSOURCE VALUELABS
--------------------

118. How would you stream logs from a specific path inside a Docker container to S3?

QBURST
------

119. What is the difference between ADD and COPY in a Dockerfile?

120. How do you reduce the Docker build size?

121. How do you pass a value while building a Docker image?

RAPIDSOFT
---------

122. Have you worked with Docker Swarm before?

SAP
---

123. [Related] How will you maintain your base image, vulnerability free?

SIGMOID
-------

124. How do you create a sub-user while writing a Dockerfile?

125. [Related] Explain how to create custom images.

126. Write a three-stage Dockerfile using the pre-built images supplied for each stage in the
    interview. [The attached text does not provide those image names.]
    Requirements:
    - Stage 1: Set and copy environment configuration files, environment variables, and shell
      profiles such as .bashrc and .bashprofile from the supplied image to the second stage.
    - Stage 2: Use the files from the previous stage, perform prerequisite tasks, and build the code
      from the current directory.
    - Stage 3: Use the output from the previous stage to build the final image.

127. Explain how image creation works, how layers are formed, and what the image produced by the
    final stage contains.

128. Which steps or commands in a Dockerfile create intermediate images? Are those intermediate
    images still used after the final image is created?

129. What is the difference between CMD and ENTRYPOINT? Give a scenario explaining how and why both
    are used together.

130. What happens if a Dockerfile contains multiple CMD and ENTRYPOINT instructions? Will the build
    fail? If it succeeds, how does Docker handle those instructions?

SONY
----

131. [Related] Why does a container sometimes exit immediately even though the application works
    perfectly in local testing? Give 3 real production causes.

132. [Related] How would you design container images for ultra-fast cold starts in serverless or
    autoscaled Kubernetes environments?

SYNCORTEX
---------

133. [Related] How would you ensure that developers use only authorized images?

SYNECHRON
---------

134. Share your screen and write a simple Dockerfile.

135. [Relevant portion] What is the difference between CMD and ENTRYPOINT in Docker?

136. What is the difference between Docker and Docker Compose?

137. Explain Docker Compose and Kubernetes.

138. Explain the ADD and COPY commands.

TCS
---

139. [Related] What is DevSecOps? Have you used tools to scan container images?

140. Difference between COPY and ADD commands in a Dockerfile.

141. If a Docker image becomes very large with many layers, what steps would you take to reduce its
    size?

142. If you have 10 layers in a Dockerfile and layer 6 fails, after fixing it, where will the
    rebuild start from and why?

143. Difference between bind mounts and volumes in Docker.

TECHDOME
--------

144. Explain Docker networking and types of network. What is the default network.

145. Docker image vs container

146. Docker bind mount vs volume

VERIZON
-------

147. How do you ensure Docker container security while writing a Dockerfile?

148. In docker run, does -p represent port or publish?

149. [Related] How do you delete old or untagged images in ECR?

VIRTUSA
-------

150. What is the difference between ENTRYPOINT and CMD in Docker?

151. What is the difference between COPY and RUN in Docker?

152. Provide the Docker commands to build an image, tag it, and push it to Docker Hub.

VOLKSWAGEN GROUP DIGITAL
------------------------

153. How do you run security checks on a Docker image?

WIPRO
-----

154. What problems arise from using a large image in a Dockerfile?

155. Explain the contents of a Dockerfile.

156. How about your experience developing CI/CD pipeline and utilizing tools such as Docker, Grafana
    and Prometheous. Share a particular project where these skills were critical.

157. [Related] How would you structure a multi-stage pipeline that builds, tests and deploys a
    containerized application to kubernetes using Github Actions.

158. Docker containers stop suddenly after starting. How would you troubleshoot this?

ZS ASSOCIATES
-------------

159. In which scenarios is a multi-stage Docker build useful? Is it suitable for compiled languages?

160. Explain layer caching with an example.

161. Explain privileged mode in Docker with an example.

ZENSAR
------

162. Explain Docker networking.

163. [Unclear] Why is Kubernetes needed if Docker volumes are available? [The author says the exact
    wording was not remembered.]

ZOPSMART
--------

164. Write a Dockerfile and explain its keywords.

165. Which commands display running and stopped containers?

166. Which commands remove Docker images and containers?

167. Write an Ansible playbook to install Docker on the nodes.

