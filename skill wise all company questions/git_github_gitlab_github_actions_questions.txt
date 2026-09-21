GIT, GITHUB, GITLAB, AND GITHUB ACTIONS INTERVIEW QUESTIONS
================================================================

Source: interview questions company wise.rtf
Extracted: 142 numbered interview prompts, with 19 nested follow-ups.
Organization: 40 named companies plus the source group "Others".

Questions are grouped by company and labeled by topic. Wording and spelling
have been lightly cleaned; no answers have been added. Repeated questions
from separate source entries are retained. Multi-part scenarios and their
follow-ups remain together. The count refers to numbered prompts, not unique
questions or every individual subquestion.

Related GitOps, Argo CD, Flux, repository management, Jenkins/source-control
integration, and CI/CD questions are included under their own topic labels.
[Context] identifies a nearby or grouped follow-up whose platform is not
explicit in that question. [Relevant portion] means only the relevant part
of a mixed question is included. [Unclear] and [Missing snippet] preserve
limitations in the source. [Follow-up] preserves a source follow-up label.

TOPIC INDEX
----------------------------------------------------------------
Git: 75 prompts
Question numbers: 1, 3, 4, 5, 6, 7, 8, 9, 10, 13, 15, 16, 17, 20, 21, 22, 25, 28, 30, 31, 32,
  33, 34, 35, 36, 37, 38, 40, 41, 43, 44, 45, 46, 49, 50, 52, 53, 56, 57, 58, 59, 60, 61, 62,
  69, 70, 71, 75, 77, 78, 79, 80, 81, 83, 86, 87, 108, 109, 112, 114, 115, 122, 123, 124, 125,
  126, 127, 128, 130, 131, 132, 133, 137, 141, 142

GitHub: 6 prompts
Question numbers: 11, 18, 23, 97, 105, 129

GitLab: 2 prompts
Question numbers: 116, 117

GitHub / GitLab: 3 prompts
Question numbers: 14, 19, 88

GitHub Actions: 26 prompts
Question numbers: 27, 63, 64, 65, 66, 72, 82, 84, 85, 89, 90, 91, 92, 93, 94, 95, 96, 98, 100,
  101, 106, 107, 121, 134, 135, 136

GitOps / Argo CD / Flux: 16 prompts
Question numbers: 24, 26, 29, 42, 47, 54, 55, 73, 74, 76, 110, 111, 119, 138, 139, 140

Related CI/CD and repositories: 14 prompts
Question numbers: 2, 12, 39, 48, 51, 67, 68, 99, 102, 103, 104, 113, 118, 120

COMPANY INDEX
----------------------------------------------------------------
Alphadyne: 2 prompts (questions 1-2)
Aspire: 2 prompts (questions 3-4)
Belcan: 3 prompts (questions 5-7)
CGI: 1 prompt (question 8)
Capgemini: 2 prompts (questions 9-10)
Cisco: 1 prompt (question 11)
Deloitte: 12 prompts (questions 12-23)
EPAM: 4 prompts (questions 24-27)
EXL Service: 4 prompts (questions 28-31)
E&Y: 2 prompts (questions 32-33)
Encora: 1 prompt (question 34)
Flentas: 1 prompt (question 35)
HCL: 7 prompts (questions 36-42)
Hexaware: 1 prompt (question 43)
Infosys: 4 prompts (questions 44-47)
Intact Green Services: 1 prompt (question 48)
LTIMindtree: 7 prompts (questions 49-55)
L&T: 5 prompts (questions 56-60)
Marsh McLennan: 5 prompts (questions 61-65)
Moodys: 1 prompt (question 66)
NUOS INFO Systems: 4 prompts (questions 67-70)
NatWest Group: 1 prompt (question 71)
Nextturn: 2 prompts (questions 72-73)
Nice: 2 prompts (questions 74-75)
Nitor Infotech: 1 prompt (question 76)
Others: 28 prompts (questions 77-104)
Perfios: 1 prompt (question 105)
Persistent Systems: 3 prompts (questions 106-108)
Publicis Global Delivery: 1 prompt (question 109)
SAP: 3 prompts (questions 110-112)
Sigmoid: 3 prompts (questions 113-115)
Sonata Software: 2 prompts (questions 116-117)
Sony: 2 prompts (questions 118-119)
SquareOps: 4 prompts (questions 120-123)
Synechron: 5 prompts (questions 124-128)
TCS: 2 prompts (questions 129-130)
Turning: 1 prompt (question 131)
Wikreate Media: 2 prompts (questions 132-133)
Wipro: 6 prompts (questions 134-139)
ZS Associates: 1 prompt (question 140)
ZopSmart: 2 prompts (questions 141-142)

================================================================================================
Alphadyne
================================================================================================

1. [Git] Explain your branching strategy in depth.
    - Why was this strategy chosen?
    - How does it support multiple environments, releases, and hotfixes?

2. [Related CI/CD and repositories] How do you configure a multibranch pipeline in Jenkins? What
    problems does it solve, and why is it preferred over a normal SCM pipeline?


================================================================================================
Aspire
================================================================================================

3. [Git] What is the difference between git pull and git fetch?

4. [Git] What are git stash and git stash pop?


================================================================================================
Belcan
================================================================================================

5. [Git] Write down the Git commands you use daily and explain them.

6. [Git] You have a local copy of a repository and have changed one file. Which Git commands would
    you run to push the changes to the remote repository?

7. [Git] Explain git stash.


================================================================================================
CGI
================================================================================================

8. [Git] Write a checkout stage with Git credentials.


================================================================================================
Capgemini
================================================================================================

9. [Git] What is the difference between git fetch and git pull? Explain in depth what happens in the
    background.

10. [Git] What happens internally when you run git add? How does Git record the added file in its
    database?


================================================================================================
Cisco
================================================================================================

11. [GitHub] Explain how you would set up a multi-branch Jenkins pipeline for a GitHub repository.


================================================================================================
Deloitte
================================================================================================

12. [Related CI/CD and repositories] What is the purpose of a webhook, and how is it used in a CI/CD
    pipeline?

13. [Git] What branching strategy do you follow, and how do you handle merges to avoid breaking the
    release branch? If a bug appears in production, what’s your approach to resolving it?

14. [GitHub / GitLab] How do you migrate a Git repository from one hosting platform to another, such
    as GitHub to GitLab, while preserving its commit history? What steps would you follow?

15. [Git] What is the difference between git fetch and git pull? When would you use each?

16. [Git] What is git cherry-pick, and how do you use it?

17. [Git] How do you handle merge conflicts? When investigating a conflict, do you check the commit
    history on the source or the target?

18. [GitHub] You have a new Jenkins installation and need to connect it to GitHub. What
    configuration steps are required, and what connection methods are available? Can webhooks
    establish communication immediately, or are prerequisite activities needed?

19. [GitHub / GitLab] How do you migrate a Git repository from one hosting platform to another, such
    as GitHub to GitLab, while preserving its commit history? What steps would you follow?

20. [Git] What is the difference between git fetch and git pull? When would you use each?

21. [Git] What is git cherry-pick, and how do you use it?

22. [Git] How do you handle merge conflicts? When investigating a conflict, do you check the commit
    history on the source or the target?

23. [GitHub] You have a new Jenkins installation and need to connect it to GitHub. What
    configuration steps are required, and what connection methods are available? Can webhooks
    establish communication immediately, or are prerequisite activities needed?


================================================================================================
EPAM
================================================================================================

24. [GitOps / Argo CD / Flux] How do you implement GitOps in a Kubernetes environment?

25. [Git] In a monorepo setup, how do you ensure that only relevant services are built and deployed
    in a CI/CD pipeline?

26. [GitOps / Argo CD / Flux] How do you manage secrets and config securely at scale in Kubernetes
    without compromising GitOps workflows?

27. [GitHub Actions] How do you set up workload identity federation between GitHub Actions and
    Google Cloud / Azure securely?


================================================================================================
EXL Service
================================================================================================

28. [Git] Explain your Git branching strategy. How do you deploy code from different branches to
    different environments?

29. [GitOps / Argo CD / Flux] If Git is already the source of truth, why do we need Argo CD? Why not
    deploy directly using the CI/CD pipeline with Helm or kubectl?

30. [Git] Explain the difference between Git Merge and Git Rebase.

31. [Git] A developer accidentally commits AWS credentials to Git. What is your complete incident
    response process?


================================================================================================
E&Y
================================================================================================

32. [Git] What is squashing in Git?

33. [Git] Explain git rebase.


================================================================================================
Encora
================================================================================================

34. [Git] What branching strategies do you use, and how do you create the associated pipelines?


================================================================================================
Flentas
================================================================================================

35. [Git] Do you commit the tfvars file to Git? These parameters need to be filled in while the
    pipeline runs. How do you manage this in Jenkins?


================================================================================================
HCL
================================================================================================

36. [Git] What Git branching strategy is used in your organization?

37. [Git] How do you deploy to different environments using a Git repository?

38. [Git] [Unclear] How and from where do you clone a repository? Do you use a local repository and
    transfer changes from local to remote? (The source author marked this question as unclear.)

39. [Related CI/CD and repositories] [Context] What is a PAT?

40. [Git] How do you handle merge conflicts in Git? If two people work on the same file, commit
    their changes, and encounter a conflict, what different approaches can resolve it?

41. [Git] [Unclear] After validating pipeline changes, how do you schedule the pipeline for the
    stage/main branch? (The source is unclear about the intended scheduling operation.)

42. [GitOps / Argo CD / Flux] [Relevant portion] Have you worked on Argo CD?


================================================================================================
Hexaware
================================================================================================

43. [Git] What branching strategies do you use?


================================================================================================
Infosys
================================================================================================

44. [Git] Which Git commands do you use in day-to-day activities?

45. [Git] What is the difference between git rebase and git merge?

46. [Git] Explain your branching strategy.

47. [GitOps / Argo CD / Flux] [Relevant portion] Have you used Argo CD to manage your Kubernetes
    cluster?


================================================================================================
Intact Green Services
================================================================================================

48. [Related CI/CD and repositories] What are the advantages of a multibranch pipeline?


================================================================================================
LTIMindtree
================================================================================================

49. [Git] Explain git rebase.

50. [Git] Explain git clone.

51. [Related CI/CD and repositories] Explain the AWS CodeCommit workflow.

52. [Git] Explain the git cherry-pick command.

53. [Git] Explain your branching strategy.

54. [GitOps / Argo CD / Flux] How do you manage secrets securely in GitOps or deployment pipelines?

55. [GitOps / Argo CD / Flux] How does your GitOps tool detect drift and how do you manage it?


================================================================================================
L&T
================================================================================================

56. [Git] What is cherry-pick in Git?

57. [Git] What is git checkout?

58. [Git] Explain your Git branching strategy.

59. [Git] Explain your Git branching strategy.

60. [Git] What is a Git merge conflict?


================================================================================================
Marsh McLennan
================================================================================================

61. [Git] What is the difference between Git Merge and Git Rebase?

62. [Git] Explain Git Merge and Git Rebase with an example using the main and feature (or master)
    branches.

63. [GitHub Actions] In GitHub Actions, if one job depends on another job, which parameter do you
    use?

64. [GitHub Actions] How do you prevent concurrent executions in GitHub Actions?

65. [GitHub Actions] What is the difference between needs and concurrency in GitHub Actions?


================================================================================================
Moodys
================================================================================================

66. [GitHub Actions] [Missing snippet] You are given a GitHub Actions workflow snippet. How would
    you identify incorrect steps and suggest improvements or missing steps for a robust CI/CD
    pipeline? (The referenced workflow snippet is not included in the attachment.)


================================================================================================
NUOS INFO Systems
================================================================================================

67. [Related CI/CD and repositories] What are the best practices for structuring repositories and
    pipelines in a large DevOps project?

68. [Related CI/CD and repositories] How would you use Azure DevOps REST API to apply a security
    policy to all repos programmatically?

69. [Git] What’s the difference between Git Merge and Rebase?

70. [Git] If someone force-pushed and lost the main branch, how do you recover it?
    - How do you push the recovered branch back to the remote repository?


================================================================================================
NatWest Group
================================================================================================

71. [Git] What types of branching strategies do you use?


================================================================================================
Nextturn
================================================================================================

72. [GitHub Actions] Compare Jenkins and GitHub Actions.

73. [GitOps / Argo CD / Flux] Compare push-based and pull-based deployment in GitOps.


================================================================================================
Nice
================================================================================================

74. [GitOps / Argo CD / Flux] How do you maintain Argo CD for the E1, E2, and E3 environments?

75. [Git] What branching strategy do you follow for source code management in a large team with a
    complex application?


================================================================================================
Nitor Infotech
================================================================================================

76. [GitOps / Argo CD / Flux] Explain the GitOps approach, Argo CD, and Flux.


================================================================================================
Others
================================================================================================

77. [Git] In Git, explain the push and pull commands.

78. [Git] What is the use of Git tags?

79. [Git] What are the different types of branches in Git?

80. [Git] What is the difference between git push --force-with-lease and git push --force?

81. [Git] How can you delete the last two Git commits?

82. [GitHub Actions] Explain a GitHub Actions workflow file.

83. [Git] What Git branching strategies do you use?

84. [GitHub Actions] [Context] Why would you use a self-hosted runner instead of the default runner?

85. [GitHub Actions] [Unclear] How do IAM users, a GitHub OIDC role, and a "terraform io role"
    differ in security? When would you use a GitHub OIDC role, and when would you use a "terraform
    io role"? (The source does not define "terraform io role".)

86. [Git] What is the difference between git fetch and git pull?

87. [Git] What is the difference between git rebase and git merge?

88. [GitHub / GitLab] Write a GitHub/GitLab pipeline to deploy a microservice with three services
    running in parallel.

89. [GitHub Actions] What is runs-on in a pipeline? Which type of runners does your organization
    use, and how do you configure self-hosted runners?

90. [GitHub Actions] [Context] How do you configure the pipeline triggers for the following cases?
    - Trigger when code is pushed to a specific branch.
    - Ignore pushes to other branches.
    - Trigger when a pull request is raised.

91. [GitHub Actions] How do you set up a manual trigger in GitHub Actions?

92. [GitHub Actions] How do you set up GitHub runners for the application environment?

93. [GitHub Actions] [Follow-up] If you use GitHub Marketplace actions, which are third-party tools,
    how do you address the security concerns associated with them?

94. [GitHub Actions] What is a matrix in GitHub Actions?

95. [GitHub Actions] What is the needs keyword in GitHub Actions?

96. [GitHub Actions] Write the structure for building and pushing a Docker image for an application
    in GitHub Actions.

97. [GitHub] What type of GitHub branching strategy are you using? Please explain.

98. [GitHub Actions] How do you run jobs in parallel in GitHub Actions?

99. [Related CI/CD and repositories] [Context] How do you handle secrets in your project?

100. [GitHub Actions] What steps are included in your GitHub Actions workflow file?

101. [GitHub Actions] [Context] How is static code analysis configured in your pipeline file?

102. [Related CI/CD and repositories] Which is the optimized method to run SonarQube: for every PR
    raised or every push?

103. [Related CI/CD and repositories] What are webhooks, and have you used them anywhere?

104. [Related CI/CD and repositories] [Context] How do you configure a pipeline with AWS or Docker?


================================================================================================
Perfios
================================================================================================

105. [GitHub] In GitHub, after a code commit, what validations do you perform? Do you validate
    before the commit or after the commit?


================================================================================================
Persistent Systems
================================================================================================

106. [GitHub Actions] What is the matrix strategy in GitHub Actions?

107. [GitHub Actions] How does caching work in GitHub Actions?

108. [Git] What is a stale branch?


================================================================================================
Publicis Global Delivery
================================================================================================

109. [Git] How do you extract all Git commits from the last three days?


================================================================================================
SAP
================================================================================================

110. [GitOps / Argo CD / Flux] How would you use Argo CD to deploy only to specific workloads or
    regions, without updating other workloads or regions?

111. [GitOps / Argo CD / Flux] Why do you want to use Argo CD over Jenkins?

112. [Git] What are Git submodules, and what is their purpose?


================================================================================================
Sigmoid
================================================================================================

113. [Related CI/CD and repositories] A developer accidentally hardcodes a password in source code
    and pushes it to a repository. How would you address this in the CI process?

114. [Git] What branching strategy does your organization follow, and why did you choose it over
    other strategies?

115. [Git] Which branching strategy do you follow, and how do you integrate it into your CI pipeline
    to deploy pushed code to the correct environment cluster? Where in the CI code do you handle
    deployment to dev, QA, and subsequent environments up to production? How do you handle PR checks
    and approvals?


================================================================================================
Sonata Software
================================================================================================

116. [GitLab] Have you worked on GitLab?

117. [GitLab] What are rules in GitLab?


================================================================================================
Sony
================================================================================================

118. [Related CI/CD and repositories] How would you configure Jenkins to run a pipeline only after N
    commits have been pushed?

119. [GitOps / Argo CD / Flux] How do you design GitOps for 1000+ clusters with environment drift
    detection, emergency hotfixes, and controlled manual overrides?


================================================================================================
SquareOps
================================================================================================

120. [Related CI/CD and repositories] Do you avoid committing secrets in values.yaml?

121. [GitHub Actions] [Context] What CI/CD tools have you worked with?
    - Explain your GitHub Actions pipeline.
    - How do you run iOS build automation?
    - How do you handle secrets in pipelines?
    - How do you deploy to EKS through GitHub Actions?

122. [Git] [Context] How do you handle pipelines for multiple environments?
    - How do you handle the Dev → QA → Prod flow?
    - What is your promotion strategy? Do you use manual approvals?
    - What is your Git branching strategy?

123. [Git] A repository has three branches: dev, staging, and prod. How do you ensure that pushing
    to staging triggers only the staging deployment?
    - Would you use separate pipelines or a single pipeline?
    - How would you use branch conditions?
    - How would you use environment variables?
    - How would you use webhooks?
    - Should the pipeline constantly check the repository?
    - What type of Jenkins job is best for this scenario?


================================================================================================
Synechron
================================================================================================

124. [Git] What is the difference between git pull and git clone?

125. [Git] What is the difference between git pull and git fetch? When would you use one instead of
    the other?

126. [Git] What is the difference between git pull and git fetch? What is git merge?

127. [Git] Suppose you have two commits. How would you check the differences between them?

128. [Git] If you want to use the feature branch instead of the main branch, how would you design
    the CI/CD pipeline?


================================================================================================
TCS
================================================================================================

129. [GitHub] What is the difference between a GitHub repository and JFrog?

130. [Git] What is a branching strategy?


================================================================================================
Turning
================================================================================================

131. [Git] Ten developers are checking code into Git. How would you remove the check-in made by
    developer 10?


================================================================================================
Wikreate Media
================================================================================================

132. [Git] Explain git stash.

133. [Git] What is the difference between git fetch and git pull?


================================================================================================
Wipro
================================================================================================

134. [GitHub Actions] How would you structure a multistage GitHub Actions pipeline that builds,
    tests, and deploys a containerized application to Kubernetes?

135. [GitHub Actions] [Context] How would you parameterize a workflow so that downstream jobs know
    which environment to deploy to?

136. [GitHub Actions] What is the difference between GitHub Actions and Argo CD?

137. [Git] What is Git, why do we use it, and what is a branching strategy?

138. [GitOps / Argo CD / Flux] What is Argo CD, and why do we use it?

139. [GitOps / Argo CD / Flux] What is GitOps?


================================================================================================
ZS Associates
================================================================================================

140. [GitOps / Argo CD / Flux] [Relevant portion] Explain Argo CD and how you manage CI/CD in your
    organization.


================================================================================================
ZopSmart
================================================================================================

141. [Git] Which Git commands do you use in daily tasks?

142. [Git] What is git stash?
