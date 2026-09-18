# Belcan DevOps Interview Questions and Answers

**Company:** Belcan  
**Experience:** 9 years  
**Focus:** Jenkins, AWS, Git, and Terraform

> Senior interview approach: explain the implementation, security model, failure handling, recovery, and trade-offs. Replace sample names and values with your actual project details.

---

## 1. Write a Jenkins pipeline

A production pipeline should be stored as a `Jenkinsfile`, reviewed in source control, use controlled credentials, build immutable artifacts, enforce quality gates, verify deployment, and support rollback.

```groovy
pipeline {
    agent none

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
        timestamps()
        skipDefaultCheckout(true)
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'stage', 'prod'])
        booleanParam(name: 'RUN_SCAN', defaultValue: true)
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '123456789012'
        ECR_REPOSITORY = 'payment-api'
        APP_NAME       = 'payment-api'
    }

    stages {
        stage('Checkout') {
            agent { label 'linux-docker' }
            steps {
                deleteDir()
                checkout scm
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${GIT_SHA}"
                }
            }
        }

        stage('Build and Test') {
            agent { label 'linux-docker' }
            steps {
                sh './mvnw -B clean verify'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }

        stage('Quality Gates') {
            parallel {
                stage('Static Analysis') {
                    agent { label 'linux-docker' }
                    steps {
                        sh './mvnw -B sonar:sonar'
                    }
                }
                stage('Dependency Scan') {
                    when { expression { params.RUN_SCAN } }
                    agent { label 'linux-docker' }
                    steps {
                        sh 'dependency-check.sh --project payment-api --scan . --format XML'
                    }
                }
            }
        }

        stage('Build and Scan Image') {
            agent { label 'linux-docker' }
            steps {
                sh 'docker build --pull --tag "$IMAGE_URI" .'
                sh 'if [ "$RUN_SCAN" = "true" ]; then trivy image --exit-code 1 --severity CRITICAL "$IMAGE_URI"; fi'
            }
        }

        stage('Push Image') {
            agent { label 'linux-docker' }
            steps {
                sh 'aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"'
                sh 'docker push "$IMAGE_URI"'
            }
        }

        stage('Production Approval') {
            when { expression { params.DEPLOY_ENV == 'prod' } }
            agent none
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Deploy ${IMAGE_URI} to production?", ok: 'Deploy'
                }
            }
        }

        stage('Deploy and Verify') {
            agent { label 'kubernetes-tools' }
            environment {
                K8S_NAMESPACE = "${params.DEPLOY_ENV}"
            }
            steps {
                withCredentials([file(credentialsId: "kubeconfig-${DEPLOY_ENV}", variable: 'KUBECONFIG')]) {
                    sh 'kubectl -n "$K8S_NAMESPACE" set image deployment/"$APP_NAME" "$APP_NAME"="$IMAGE_URI"'
                    sh 'kubectl -n "$K8S_NAMESPACE" rollout status deployment/"$APP_NAME" --timeout=5m'
                    sh './scripts/smoke-test.sh "$DEPLOY_ENV"'
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${env.IMAGE_URI}"
        }
        failure {
            echo 'Build failed. Notify the owner and follow the approved rollback procedure.'
        }
        always {
            cleanWs(deleteDirs: true, notFailBuild: true)
        }
    }
}
```

### Senior-level design points

- Use `agent none` globally and fit-for-purpose agents per stage.
- Prefer ephemeral agents and never run routine builds on the controller.
- Use agent IAM roles, workload identity, or short-lived credentials instead of static cloud keys.
- Build once and promote the same artifact or image digest through environments.
- Parallelize independent tests and scans.
- Store reusable logic in a versioned Jenkins Shared Library.
- Add timeouts, concurrency control, retention, evidence, approvals, smoke tests, and rollback.

---

## 2. Purpose of `agent`, `post`, and `environment`

### `agent`

The `agent` directive specifies where the Pipeline or an individual stage runs. It allocates an executor and workspace on a matching Jenkins node, container, or Kubernetes Pod.

```groovy
agent any
agent { label 'linux-docker' }
```

With `agent none` at Pipeline level, every stage that performs work declares its own agent. This avoids holding one executor for the entire Pipeline, especially during manual approvals.

### `post`

The `post` block runs actions according to completion state. Common conditions are `always`, `success`, `failure`, `unstable`, `unsuccessful`, `changed`, `aborted`, and `cleanup`.

```groovy
post {
    always {
        junit '**/test-results/*.xml'
        cleanWs()
    }
    failure {
        echo 'Notify support and preserve diagnostics'
    }
}
```

It is typically used for reports, notifications, cleanup, audit evidence, and recovery actions.

### `environment`

The `environment` block defines Pipeline-level or stage-level environment variables.

```groovy
environment {
    APP_NAME = 'payment-api'
    REGION   = 'ap-south-1'
}
```

For secrets, use Jenkins Credentials rather than plaintext. Prefer a narrowly scoped `withCredentials` block so the secret is available only to the required steps.

---

## 3. Complete Jenkins backup, including jobs, configurations, and authentication

A complete recovery design must protect Jenkins state, encryption key material, software versions, and external configuration. The backup is only proven after a successful restore test.

### Back up required `JENKINS_HOME` content

- Global XML configurations
- `jobs/`, including job/folder configuration and required build metadata
- `users/` and user configuration
- `nodes/` for static agent definitions
- Credentials files and domains
- `secrets/`, because encrypted credentials require matching key material
- Plugin files, or preferably a pinned and reproducible plugin catalog
- Script approvals, fingerprints, and other required installation state
- Jenkins Configuration as Code files and Shared Libraries if they are not already protected in Git

Also record or back up configuration outside `JENKINS_HOME`, including reverse proxy, TLS, DNS, IAM, storage, Java, Jenkins core, and infrastructure definitions.

### Backup procedure

1. Put Jenkins into quiet-down mode or stop Jenkins for an application-consistent copy.
2. Prefer a filesystem/storage snapshot where possible.
3. Copy `JENKINS_HOME` while preserving ownership, permissions, ACLs, links, and timestamps.
4. Encrypt the backup and copy it to a remote, access-controlled location.
5. Protect the Jenkins controller key separately from the general backup archive, following separation-of-duties requirements.
6. Export Jenkins core, Java, and plugin versions.
7. Apply retention, immutability, integrity checks, and access logging.
8. Perform scheduled recovery tests on an isolated controller.

Example inventory:

```bash
java -jar jenkins-cli.jar -s https://jenkins.example.com/ version
java -jar jenkins-cli.jar -s https://jenkins.example.com/ list-plugins > plugins.txt
sudo du -sh /var/lib/jenkins
```

Example offline copy:

```bash
sudo systemctl stop jenkins
sudo rsync -aHAX --numeric-ids /var/lib/jenkins/ /backup/jenkins/JENKINS_HOME/
sudo systemctl start jenkins
```

### Restore procedure

1. Provision a compatible OS, Java, Jenkins core, storage, and network configuration.
2. Stop Jenkins.
3. Restore `JENKINS_HOME` with correct ownership and permissions.
4. Restore separately protected key material through the approved secure procedure.
5. Start Jenkins and inspect logs.
6. Validate SSO/login, authorization, credentials, plugins, agents, jobs, libraries, webhooks, and representative builds.
7. Rotate credentials if compromise is suspected.

Do not treat agent workspaces as the system of record. Publish artifacts to an external repository such as ECR, Artifactory, Nexus, or S3.

---

## 4. Ways to trigger a Jenkins pipeline

1. Manual **Build Now** or **Build with Parameters**
2. SCM webhook on push, pull request, tag, or repository event
3. SCM polling with `pollSCM`
4. Scheduled execution with Jenkins cron
5. Upstream/downstream job completion
6. Authenticated REST API or remote build token
7. Multibranch Pipeline or Organization Folder branch scan
8. External events through plugins, such as artifact repositories, queues, issue trackers, or generic webhooks
9. Authorized replay or rebuild, where policy permits

```groovy
triggers {
    cron('H 2 * * 1-5')
    pollSCM('H/10 * * * *')
    upstream(upstreamProjects: 'artifact-publish', threshold: hudson.model.Result.SUCCESS)
}
```

`H` distributes scheduled execution to reduce all jobs starting simultaneously. A webhook is normally preferable to frequent polling because it is event-driven. Remote triggers must be authenticated, authorized, restricted, and audited.

---

## 5. Give view-only access to five Jenkins jobs

A Jenkins **View** organizes items but should not be treated as the authorization boundary. Use authentication and item/folder-level authorization.

### Recommended approach

1. Put the five jobs into a dedicated folder.
2. Integrate Jenkins with LDAP, Active Directory, SAML, OIDC, or the approved identity provider.
3. Create a group such as `jenkins-belcan-viewers`.
4. Configure Matrix Authorization Strategy at appropriate global and folder/item scopes, or use a governed role-based authorization plugin.
5. Grant only the read permissions necessary.

Typical read-only baseline:

```text
Overall/Read
Job/Read
View/Read
```

Optional only when users need artifacts:

```text
Job/Artifacts
```

Do not grant:

```text
Job/Build
Job/Cancel
Job/Configure
Job/Delete
Job/Workspace
Credentials/View
Overall/Administer
```

Test with a non-admin user. Review whether console logs, artifacts, test reports, parameters, SCM metadata, or Pipeline scripts expose sensitive data. Authorization cannot compensate for secrets printed into logs.

---

## 6. AWS load balancer types

AWS Elastic Load Balancing has **three current-generation load-balancer types**, plus the previous-generation Classic Load Balancer.

### Application Load Balancer

- Layer 7, primarily HTTP and HTTPS
- Content-aware routing by host, path, headers, method, query string, and other supported conditions
- Suitable for web applications, REST APIs, microservices, containers, WebSockets, HTTP/2, and gRPC use cases
- Supports TLS termination, target groups, redirects, authentication integrations, AWS WAF, and weighted routing

### Network Load Balancer

- Layer 4 for TCP, TLS, UDP, and TCP_UDP
- Very high throughput and low latency
- Suitable for non-HTTP protocols, static-IP requirements, source-IP preservation scenarios, and TLS pass-through or termination
- Can use Elastic IP addresses in supported internet-facing IPv4 designs

### Gateway Load Balancer

- Deploys, scales, and distributes traffic across virtual network appliances
- Used for firewalls, IDS/IPS, deep packet inspection, and centralized traffic inspection
- Integrates through Gateway Load Balancer endpoints and transparent service insertion

### Classic Load Balancer

- Previous generation
- Basic Layer 4 and Layer 7 functionality for legacy EC2 workloads
- AWS recommends migration to a current-generation load balancer

```text
HTTP content routing      -> ALB
TCP/UDP and performance   -> NLB
Security appliances       -> GWLB
Legacy existing workload  -> CLB with migration plan
```

---

## 7. Elastic IP vs public IP in AWS

### Auto-assigned public IPv4

- Assigned from Amazon's pool when launch and subnet settings permit
- Associated with the primary network interface
- Not permanently allocated to the account
- Can change after stop/start
- Released when it is no longer assigned

### Elastic IP

- Static public IPv4 explicitly allocated to the AWS account
- Remains allocated until released
- Can be associated and reassociated with supported resources
- Useful for stable allow-listing, NAT gateways, recovery through remapping, and selected NLB designs
- Regional and subject to quotas and public IPv4 charges
- Not available for IPv6

Prefer private instances behind load balancers or managed endpoints rather than exposing individual EC2 instances. AWS charges for public IPv4 addresses, including Elastic IP addresses and public IPv4 addresses assigned to running instances.

---

## 8. Daily Git commands

### Inspect

```bash
git status
git log --oneline --graph --decorate --all -20
git branch -vv
git remote -v
git diff
git diff --staged
```

These inspect working-tree status, history, branch tracking, remotes, unstaged changes, and staged changes.

### Clone and synchronize

```bash
git clone <repository-url>
git fetch --prune origin
git pull --ff-only
```

`fetch` updates remote-tracking references without integrating them. `pull --ff-only` fetches and updates only when a fast-forward is possible.

### Branch

```bash
git switch main
git switch -c feature/observability
git branch -d feature/observability
```

Switch, create, and safely delete a merged local branch.

### Stage and commit

```bash
git add path/to/file
git add -p
git commit -m "Add filesystem alarm"
```

`git add -p` supports selective hunk staging and cleaner commits.

### Integrate

```bash
git rebase origin/main
git merge --no-ff feature/observability
git rebase --continue
git rebase --abort
```

Use the team's agreed merge strategy. Avoid rebasing published shared branches.

### Push

```bash
git push -u origin feature/observability
git push
```

The first command pushes the branch and configures its upstream.

### Undo and recover

```bash
git restore path/to/file
git restore --staged path/to/file
git revert <commit>
git reflog
```

Use `revert` for shared history because it creates a new reversing commit. `reflog` helps recover previous local reference positions.

### Stash and tag

```bash
git stash push -u -m "WIP before hotfix"
git stash list
git stash pop
git tag -a v2.4.0 -m "Release 2.4.0"
git push origin v2.4.0
```

---

## 9. Push a modified local file to the remote repository

```bash
# Check branch and status
git status
git branch --show-current

# Review the file change
git diff -- path/to/file

# Refresh remote references
git fetch origin

# Rebase according to team policy
git rebase origin/main

# Stage only the intended file
git add path/to/file

# Review staged content
git diff --staged

# Commit
git commit -m "Fix health-check timeout"

# Push the current feature branch and set upstream
git push -u origin HEAD
```

Normally open a pull request rather than pushing directly to protected `main`.

If rebase conflicts occur:

```bash
git status
# Resolve files
git add <resolved-file>
git rebase --continue
```

To cancel:

```bash
git rebase --abort
```

If an already-published personal feature branch was intentionally rebased, use `git push --force-with-lease`, not plain `--force`, and only under team policy.

---

## 10. Git stash

Git stash temporarily records local modifications and returns the working tree toward `HEAD`. It is useful when unfinished work is not ready to commit but you need to switch context or update the branch.

```bash
git stash push -u -m "WIP: listener update"
git stash list
git stash show -p stash@{0}
git stash apply stash@{0}
git stash pop stash@{0}
git stash drop stash@{0}
```

- `apply` restores changes but keeps the stash entry.
- `pop` restores changes and removes the entry if successful.
- `-u` includes untracked files.
- `-a` also includes ignored files and should be used carefully.
- `-p` lets you select hunks interactively.
- Conflicts can occur when applying a stash onto changed code.

A stash is not a long-term branch, collaboration mechanism, or remote backup.

---

## 11. Terraform state file

Terraform state stores bindings between Terraform resource addresses and real infrastructure objects. Terraform uses those bindings and metadata to calculate create, update, and destroy actions.

Default local files:

```text
terraform.tfstate
terraform.tfstate.backup
```

Conceptually:

```text
aws_instance.web -> state binding -> i-0123456789abcdef0
```

### Production practices

- Use a remote backend or HCP Terraform.
- Enable locking where supported.
- Encrypt state at rest and in transit.
- Use least-privilege access and audit logging.
- Enable versioning and recovery mechanisms.
- Never commit state to Git.
- Treat state as sensitive because provider data can include sensitive values.
- Use separate state boundaries to reduce blast radius.
- Do not manually edit state JSON. Use `terraform state`, import, and `moved` blocks.

```bash
terraform state list
terraform state show aws_instance.web
terraform state pull > state-backup.json
terraform show
```

---

## 12. Terraform commands

### Format and validate

```bash
terraform fmt -check -recursive
terraform validate
```

Checks canonical formatting and configuration validity.

### Initialize

```bash
terraform init
terraform init -upgrade
terraform init -reconfigure
```

Initializes the backend, providers, and modules. `-upgrade` considers newer allowed dependencies; `-reconfigure` reinitializes backend configuration.

### Plan

```bash
terraform plan -out=tfplan
terraform show tfplan
terraform show -json tfplan > tfplan.json
```

Creates and inspects a saved execution plan.

### Apply and destroy

```bash
terraform apply tfplan
terraform destroy
```

`apply` executes the saved plan. `destroy` removes managed infrastructure and requires strict controls.

### Output and workspace

```bash
terraform output
terraform output -json
terraform workspace list
terraform workspace new dev
terraform workspace select dev
```

Workspaces separate state for the same configuration, but they do not automatically create strong security or blast-radius isolation.

### Import

```bash
terraform import aws_s3_bucket.logs existing-company-logs
```

Associates an existing remote object with a Terraform address. Reconcile configuration and run `plan` afterward. Declarative import blocks are another option.

### State operations

```bash
terraform state list
terraform state show aws_s3_bucket.logs
terraform state mv aws_instance.old module.compute.aws_instance.web
terraform state rm aws_s3_bucket.logs
```

- `list` displays tracked resources.
- `show` inspects one instance.
- `mv` changes a binding during refactoring.
- `rm` makes Terraform forget an object without deleting it. Use cautiously.

A reviewable refactor can use:

```hcl
moved {
  from = aws_instance.old
  to   = module.compute.aws_instance.web
}
```

### Providers, version, and locks

```bash
terraform providers
terraform providers lock
terraform version
terraform force-unlock <LOCK_ID>
```

Only force-unlock after proving no active process owns the lock.

### Recommended CI sequence

```bash
terraform fmt -check -recursive
terraform init -input=false
terraform validate
terraform plan -input=false -out=tfplan
terraform show -json tfplan > tfplan.json
# Security/policy checks and approval
tfsec .
terraform apply -input=false tfplan
```

Use short-lived cloud credentials, protected state, policy checks, environment approvals, and retained plan/apply evidence.

---

## Rapid revision

- Store Jenkins Pipelines as code and place reusable logic in Shared Libraries.
- `agent` selects execution capacity; `environment` supplies scoped variables; `post` handles result-dependent activities.
- Back up required Jenkins home state, separately protect key material, record plugin/core versions, and test restoration.
- Jenkins Pipelines can be triggered manually, by webhook, polling, cron, an upstream job, API, branch scan, or external event.
- A Jenkins View is not sufficient authorization. Use folders and matrix/role controls.
- AWS offers ALB, NLB, and GWLB as current-generation options; CLB is previous-generation.
- An auto-assigned public IPv4 can change; an Elastic IP remains allocated until released.
- Review with `status` and `diff` before `add`, `commit`, and `push`.
- Stash is temporary local context management, not durable backup.
- Terraform state maps configuration addresses to remote objects and must be secured.
- Use saved plans, locking, remote state, short-lived credentials, and approval gates.

---

## Official references

### Jenkins

- Pipeline syntax: https://www.jenkins.io/doc/book/pipeline/syntax/
- Jenkins Pipeline: https://www.jenkins.io/doc/book/pipeline/
- Pipeline as Code: https://www.jenkins.io/doc/book/pipeline/pipeline-as-code/
- Using a Jenkinsfile: https://www.jenkins.io/doc/book/pipeline/jenkinsfile/
- Pipeline triggers: https://www.jenkins.io/doc/pipeline/steps/params/pipelinetriggers/
- Backup and restore: https://www.jenkins.io/doc/book/system-administration/backing-up/
- Access control: https://www.jenkins.io/doc/book/security/access-control/
- Permissions: https://www.jenkins.io/doc/book/security/access-control/permissions/

### AWS

- Elastic Load Balancing: https://docs.aws.amazon.com/elasticloadbalancing/
- How ELB works: https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html
- Elastic IP addresses: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html
- EC2 IPv4 management: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-ip-addresses.html

### Git

- Everyday Git: https://git-scm.com/docs/giteveryday
- Git push: https://git-scm.com/docs/git-push
- Git rebase: https://git-scm.com/docs/git-rebase
- Git stash: https://git-scm.com/docs/git-stash

### Terraform

- Terraform CLI: https://developer.hashicorp.com/terraform/cli/commands
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- State commands: https://developer.hashicorp.com/terraform/cli/commands/state
- State list: https://developer.hashicorp.com/terraform/cli/commands/state/list
- State show: https://developer.hashicorp.com/terraform/cli/commands/state/show
- State move: https://developer.hashicorp.com/terraform/cli/commands/state/mv
