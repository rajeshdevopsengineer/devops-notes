Below are answers for both LTIMindtree sets. Use the experience-based examples as templates and replace them with details from your own projects.

**1. Can we install Docker inside a container?**

**Yes, but installing the Docker client and running a Docker daemon are different things.**

There are two common approaches:

| Approach | How it works | Important consideration |
|---|---|---|
| Docker-in-Docker, or DinD | A Docker daemon runs inside the container | The conventional setup requires elevated privileges |
| Container using the host’s Docker daemon | The container contains the Docker client and accesses the host daemon | It can control host Docker resources |

A demonstration of DinD:

```bash
docker run --privileged \
  --name dind \
  -d docker:dind

docker exec dind docker info
```

The official Docker image provides a `dind` variant. Its conventional configuration uses `--privileged`, which grants extensive access and should be restricted to appropriate, isolated environments. [Docker Hub](https://hub.docker.com/_/docker?utm_source=chatgpt.com)

Another approach mounts the host socket:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli docker ps
```

Here, the command talks to the **host daemon**. Containers it creates are managed by that daemon; they are not nested inside the client container.

Access to a rootful Docker daemon can provide powerful host-level control. For CI, I would use isolated build agents and carefully control which pipelines can access the daemon. [Docker Docs](https://docs.docker.com/engine/security/?utm_source=chatgpt.com)

---

**2. How can we create three instances from a list of names in Terraform?**

You can use either `count` or `for_each`. The choice affects how Terraform identifies the instances.

**Example using `count`:**

```hcl
variable "instance_names" {
  type    = list(string)
  default = ["app-a", "app-b", "app-c"]
}

variable "ami_id" {
  type = string
}

resource "aws_instance" "app" {
  count = length(var.instance_names)

  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = var.instance_names[count.index]
  }
}
```

The resource addresses are:

```text
aws_instance.app[0]
aws_instance.app[1]
aws_instance.app[2]
```

Terraform identifies these instances by their **numeric indexes**. The AWS `Name` tag does not determine their Terraform identity. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/count?utm_source=chatgpt.com)

For resources that have meaningful names, `for_each` often provides a more stable mapping. The next answer explains why.

---

**3. What happens if we remove the second instance name and apply again?**

**It depends on whether the configuration uses `count` or `for_each`.**

Suppose the original list is:

```hcl
["app-a", "app-b", "app-c"]
```

After removing the second name:

```hcl
["app-a", "app-c"]
```

**With the `count` example above:**

| Resource address | Before | After |
|---|---|---|
| `app[0]` | Name: `app-a` | Remains `app-a` |
| `app[1]` | Name: `app-b` | Changes to `app-c` |
| `app[2]` | Name: `app-c` | Destroyed because index 2 is no longer required |

Because only the `Name` tag changes at index 1 in this example, Terraform normally updates that tag in place.

Therefore, the original instance named `app-b` survives but gets renamed, while the original instance named `app-c` is destroyed.

If other index-based settings change—such as an attribute requiring replacement—the plan can contain additional replacements.

**With `for_each`:**

```hcl
resource "aws_instance" "app" {
  for_each = toset(var.instance_names)

  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = each.key
  }
}
```

The addresses become:

```text
aws_instance.app["app-a"]
aws_instance.app["app-b"]
aws_instance.app["app-c"]
```

Removing `"app-b"` then proposes destroying that keyed instance while preserving the other two. `for_each` identifies instances using map keys or set elements. [HashiCorp Developer](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each?utm_source=chatgpt.com)

Always review the plan before applying. Changing an existing configuration from `count` to `for_each` also requires an address-migration plan, such as appropriate `moved` blocks.

---

**4. If a Jenkins pipeline has five stages and the fifth contains a syntax error, what happens?**

**If the error is in the Jenkinsfile’s Groovy syntax or Declarative structure, the pipeline normally fails before executing its stages.**

Jenkins must parse and process the Jenkinsfile as a whole. It does not parse each stage only when that stage starts.

However, distinguish the type of error:

| Error | Expected behaviour |
|---|---|
| Missing brace or quotation mark in the Jenkinsfile | Compilation/parsing fails; stages do not execute |
| Invalid Declarative Pipeline structure | Validation fails before normal stage execution |
| Syntax error inside an external shell script | Earlier stages may complete; failure occurs when the script runs |
| Syntax error in a file loaded during execution | Failure occurs when that file is loaded; earlier work may have completed |

For example:

```groovy
stage('Deploy') {
    steps {
        sh 'bash scripts/deploy.sh'
    }
}
```

The Jenkinsfile can be valid even when `deploy.sh` contains a Bash syntax error. In that case, the failure occurs when Jenkins executes the shell step.

Jenkins provides development tools for validating Declarative Pipelines before running the full workflow. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/development/?utm_source=chatgpt.com)

A useful interview answer is:

> “A Jenkinsfile syntax error generally prevents stage execution. A syntax error in a script invoked by the fifth stage is detected when that script runs.”

---

**5. What are the differences between Scripted and Declarative Pipelines?**

Both are Jenkins Pipeline approaches, and both can be stored in a version-controlled Jenkinsfile.

| Aspect | Declarative | Scripted |
|---|---|---|
| Main structure | `pipeline { ... }` | Commonly `node { ... }` |
| Style | Structured DSL with defined sections | More direct Groovy programming |
| Validation | Enforces Declarative structure | Relies more on Groovy and Pipeline execution |
| Conditions | Usually `when` and related directives | Groovy `if`, loops, and other control flow |
| Error handling | `post` conditions and Pipeline steps | Commonly `try`, `catch`, and `finally` |
| Flexibility | Structured, with `script` blocks when needed | Convenient for complex dynamic logic |

**Declarative example:**

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building application'
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed'
        }
    }
}
```

**Scripted example:**

```groovy
node {
    try {
        stage('Build') {
            echo 'Building application'
        }
    } catch (Exception error) {
        echo 'Pipeline failed'
        throw error
    }
}
```

Jenkins documents the syntax and capabilities of both approaches. [jenkins.io](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com)

For standard team workflows, Declarative is often easier to maintain. Scripted is useful when the workflow needs substantial dynamic programming. Declarative Pipelines can still call shared libraries and execute Groovy through appropriate blocks.

---

**6. What is the difference between code quality and code coverage?**

**Code quality concerns how well the code is written. Coverage measures how much code tests execute.**

| Code quality | Code coverage |
|---|---|
| Includes reliability, security, maintainability, and readability | Measures execution of lines, branches, or other structures during tests |
| Can identify defects, insecure patterns, duplication, and complexity | Identifies code that tests do or do not exercise |
| Evaluated through analysis, reviews, and testing | Collected through coverage instrumentation |

For example, if tests execute 80 of 100 executable lines, line coverage is 80%.

However, those tests might contain weak assertions. Executing a function does not prove its output is correct.

I would assess:

- Coverage of important paths.
- Meaningful assertions.
- Error and boundary cases.
- Security and reliability findings.
- Complexity and maintainability.

SonarQube imports coverage reports generated by tools in the build pipeline. It does not generate test coverage simply by scanning the source code. The coverage tool should run before SonarScanner imports its report. [docs.sonarsource.com](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/test-coverage/overview?utm_source=chatgpt.com)

Typical tools include JaCoCo for Java, coverage.py for Python, and Istanbul-based tooling for JavaScript.

---

**7. What is the default quality gate in SonarQube?**

The built-in default is **Sonar way**, although administrators can select a different default or assign another gate to a project.

The documented Sonar way conditions focus on new code:

| Condition | Required result |
|---|---|
| New issues | No new issues introduced |
| New security hotspots | All reviewed |
| New-code coverage | At least 80% |
| New-code duplication | At most 3% |

Sonar provides this built-in gate as read-only. [docs.sonarsource.com](https://docs.sonarsource.com/sonarqube-server/quality-standards-administration/managing-quality-gates/introduction-to-quality-gates?utm_source=chatgpt.com)

Two practical details:

- Coverage and duplication conditions can be ignored for sufficiently small changes under the default small-change settings.
- Exact security conditions depend on the installed version and migration state; current documentation notes that security hotspots are being phased out. [docs.sonarsource.com](https://docs.sonarsource.com/sonarqube-server/quality-standards-administration/managing-quality-gates/introduction-to-quality-gates?utm_source=chatgpt.com)

Also distinguish:

- **Quality profile:** selects the analysis rules.
- **Quality gate:** decides whether analysis results pass the required conditions.

In CI, wait for the quality-gate result. A successful scanner command alone does not necessarily mean the gate passed.

---

**8. How do you apply a YAML manifest without creating a YAML file?**

Pass the manifest through **standard input**:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
data:
  APP_ENV: "dev"
  LOG_LEVEL: "INFO"
EOF
```

Here:

- `-f -` tells kubectl to read from standard input.
- The heredoc supplies the YAML.
- Quoting `'EOF'` prevents the shell from expanding variables inside the manifest.

Kubectl supports applying configuration from standard input. [Kubernetes](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/?utm_source=chatgpt.com)

You can also create some objects imperatively:

```bash
kubectl create deployment web \
  --image=nginx:stable \
  --replicas=2
```

Or generate YAML and immediately apply it:

```bash
kubectl create deployment web \
  --image=nginx:stable \
  --replicas=2 \
  --dry-run=client -o yaml |
kubectl apply -f -
```

For production, keep the desired configuration in version control so changes remain reviewable and reproducible.

The following answers cover the **five-year-experience set**.

---

**1. What are your day-to-day activities?**

A sample answer to adapt:

> “I start by reviewing production alerts, overnight pipeline failures, and planned changes. I support application teams with build and deployment issues, maintain infrastructure configuration, and investigate Kubernetes or cloud incidents.
>
> I also review pull requests for pipelines and Terraform, support releases across environments, and verify post-deployment health. Alongside operational work, I automate repetitive tasks, improve monitoring, and follow up on security and reliability findings.”

Be specific about your actual responsibilities:

- CI/CD maintenance.
- Infrastructure changes.
- Deployment support.
- Incident investigation.
- Observability.
- Security and patching.
- Automation and documentation.

Prepare one concrete example of an incident and one example of a task you automated. Explain your own contribution rather than presenting every team activity as personal work.

---

**2. Explain Git rebase.**

**Rebase replays commits onto a different base commit.**

Suppose your feature branch was created from `main`, and `main` has advanced. Rebasing places your feature work on top of the newer main history.

```bash
git fetch origin
git switch feature/login
git rebase origin/main
```

The replayed commits generally receive new commit IDs because their parent history changes. Git documents rebase as applying commits on top of another base. [git-rebase Documentation](https://git-scm.com/docs/git-rebase?utm_source=chatgpt.com)

**Handling conflicts**

```bash
# Resolve the affected files
git add path/to/resolved-file
git rebase --continue
```

To abandon the operation:

```bash
git rebase --abort
```

**When I would use it**

- Updating a personal feature branch.
- Keeping a linear history where the team prefers it.
- Cleaning commits before review with interactive rebase.

Rewriting a published shared branch can disrupt other contributors. For an approved update to a personal published branch, `--force-with-lease` provides a useful guard against overwriting unexpected remote changes.

---

**3. Explain Git clone.**

**`git clone` creates a local Git repository from an existing repository.**

A normal clone downloads repository content and history, configures a remote usually named `origin`, and checks out the initial branch.

```bash
git clone https://github.com/example/application.git
cd application
```

Clone into a different directory:

```bash
git clone https://github.com/example/application.git local-app
```

Clone a specific branch:

```bash
git clone \
  --branch develop \
  --single-branch \
  https://github.com/example/application.git
```

A shallow clone limits the history downloaded:

```bash
git clone --depth 1 \
  https://github.com/example/application.git
```

These behaviours and options are documented by Git. [git-clone Documentation](https://git-scm.com/docs/git-clone?utm_source=chatgpt.com)

After cloning, developers edit locally, commit changes locally, and push commits to the remote. CI agents can clone or check out the same remote repository independently.

Repository authentication depends on the hosting service and configured HTTPS or SSH access.

---

**4. Explain the AWS CodeCommit flow.**

**CodeCommit hosts Git repositories; deployment requires a separate build and delivery workflow.**

A typical process is:

1. Create a repository and configure access.
2. Clone it locally.
3. Create a feature branch.
4. Commit and push changes.
5. Open and review a pull request.
6. Merge approved changes.
7. Trigger the configured CI/CD workflow.
8. Build, test, package, and deploy the application.

For supported temporary or federated credentials, AWS recommends `git-remote-codecommit`. [AWS CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html?utm_source=chatgpt.com)

With that helper installed and an approved profile configured:

```bash
aws sso login --profile devops

AWS_PROFILE=devops \
  git clone codecommit::ap-south-1://MyApp
```

**Example delivery flow**

- CodeCommit provides the source.
- A configured event triggers CodePipeline.
- CodeBuild runs tests and creates artifacts.
- Artifacts go to S3 or images to ECR.
- Required approvals and checks complete.
- The deployment mechanism updates the target environment.

CodeDeploy supports appropriate EC2, ECS, and Lambda deployments. EKS can use a pipeline deployment step or a GitOps controller.

CodeCommit reopened new-customer sign-ups in November 2025. [AWS DevOps & Developer Productivity Blog](https://aws.amazon.com/blogs/devops/aws-codecommit-returns-to-general-availability/?utm_source=chatgpt.com)

---

**5. What are the application deployment types?**

| Strategy | How it works | Main consideration |
|---|---|---|
| Recreate | Stop the old version, then start the new version | Usually causes interruption |
| Rolling update | Replace replicas gradually | Old and new versions coexist |
| Blue-green | Prepare a second environment and switch traffic | Extra capacity and data compatibility |
| Canary | Send a small traffic share to the new version | Requires traffic control and health analysis |

For example, a canary could start with a small percentage of traffic and increase only when error rate, latency, and business transactions remain healthy.

For rolling updates, configure readiness, replacement capacity, and graceful shutdown. Kubernetes Deployments provide controls for introducing and removing replicas. [kubernetes.io](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com)

For every strategy, define rollback behaviour in advance.

Database compatibility matters: switching application traffic back does not automatically reverse incompatible schema or data changes.

---

**6. What are Lambda functions?**

**Lambda Functions run code in response to events or requests without requiring you to manage the execution servers.**

Triggers include:

- API Gateway.
- S3 events.
- SQS messages.
- EventBridge schedules.
- Other supported service integrations.

Example Python handler:

```python
import json

def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Hello"})
    }
```

For an HTTP integration, this returns a simple response. The configured handler name must match the module and function.

A deployment normally includes:

1. Code packaged as a supported ZIP artifact or container image.
2. An execution role.
3. Memory and timeout configuration.
4. A trigger or invocation mechanism.
5. Logging, monitoring, and failure handling.

Lambda manages execution environments and scaling for functions. [docs.aws.amazon.com](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html?utm_source=chatgpt.com)

Use idempotent processing when events can be delivered again. For queue-driven work, define retry and dead-letter handling deliberately.

---

**7. How do you secure Lambda?**

I would secure **invocation, execution permissions, data, code, and operations**.

**Invocation**

- Restrict who can invoke the function.
- Use suitable API Gateway authentication or IAM authentication.
- Review resource-based policies.
- Avoid unintentionally exposing an unauthenticated function URL.

Function URLs support `AWS_IAM` and `NONE` authentication types. `NONE` does not provide Lambda-managed authentication. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/urls-auth.html?utm_source=chatgpt.com)

**Execution permissions**

Give the execution role only the required actions and resources. For example, a file processor might need access to one input prefix and one output destination.

The execution role controls the function’s access to AWS resources. [AWS Lambda](https://docs.aws.amazon.com/us_en/lambda/latest/dg/lambda-intro-execution-role.html?utm_source=chatgpt.com)

**Data and configuration**

- Retrieve secrets from an approved secret store.
- Control encryption-key access.
- Avoid logging passwords, tokens, or sensitive request bodies.
- Validate event input.

**Code and operations**

- Scan dependencies and deployed artifacts.
- Update supported runtimes.
- Control who can change code and configuration.
- Set appropriate timeouts and concurrency controls.
- Monitor failures, unusual invocation patterns, and audit activity.

Connecting Lambda to a VPC enables access to relevant private resources; it does not automatically authenticate its public API endpoint.

---

**8. Explain Kubernetes architecture.**

Kubernetes uses a **control plane** to manage desired state and **worker nodes** to run workloads.

| Component | Responsibility |
|---|---|
| API server | Receives and processes Kubernetes API requests |
| etcd | Stores cluster API state |
| Scheduler | Selects nodes for unscheduled Pods |
| Controller manager | Reconciles desired and observed state |
| kubelet | Manages Pod execution on its node |
| Container runtime | Runs containers through the runtime interface |
| CNI/networking components | Provide Pod connectivity |
| Service dataplane | Implements Service traffic routing |

A cloud controller manager handles relevant cloud integration where used. [Kubernetes](https://kubernetes.io/docs/concepts/overview/components/?utm_source=chatgpt.com)

**What happens when a Deployment is created?**

1. The API server accepts and stores the desired configuration.
2. Controllers create the necessary ReplicaSet and Pods.
3. The scheduler selects nodes.
4. Kubelets request container creation through the runtime.
5. Networking and storage are configured.
6. Readiness determines normal eligibility for Service traffic.

The scheduler selects placement; the kubelet and runtime perform node-level execution.

CoreDNS commonly provides cluster DNS. kube-proxy is one Service-routing implementation; some clusters use alternative dataplanes.

---

**9. What is Git cherry-pick? Give the command.**

**Cherry-pick applies the changes introduced by a selected commit onto the current branch.**

Example: bring a specific bug fix into a release branch:

```bash
git fetch origin
git switch release/1.2
git cherry-pick -x a1b2c3d
```

The `-x` option records the source commit reference in the new commit message, which is useful for tracking backports.

Cherry-pick normally creates a new commit with a different identity on the target branch. [git-cherry-pick Documentation](https://git-scm.com/docs/git-cherry-pick?utm_source=chatgpt.com)

**Conflicts**

```bash
# Resolve the affected files
git add path/to/resolved-file
git cherry-pick --continue
```

To cancel:

```bash
git cherry-pick --abort
```

Before selecting a commit, check whether it depends on earlier changes. A commit that applies cleanly can still fail logically when a dependency is missing.

Run the appropriate tests after applying it.

---

**10. What is `appspec.yml` used for?**

**The AppSpec file tells AWS CodeDeploy how to deploy an application revision and which lifecycle hooks to execute.**

Its contents depend on the deployment platform:

| Platform | AppSpec describes |
|---|---|
| EC2/on-premises | Files, destinations, permissions, and lifecycle scripts |
| ECS | Target service, task definition, container/port information, and hooks |
| Lambda | Function version and alias transition, plus validation hooks |

AWS documents these platform-specific AppSpec formats. [AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html?utm_source=chatgpt.com)

**Example for EC2/Linux:**

```yaml
version: 0.0
os: linux

files:
  - source: app/
    destination: /opt/myapp

hooks:
  AfterInstall:
    - location: scripts/configure.sh
      timeout: 120
      runas: root

  ApplicationStart:
    - location: scripts/start.sh
      timeout: 120
      runas: root

  ValidateService:
    - location: scripts/health-check.sh
      timeout: 60
      runas: root
```

The application files and referenced scripts must be present in the deployment revision. Select hook privileges according to what the scripts require.

For EC2/on-premises deployments, CodeDeploy uses its agent to process the revision and lifecycle events. AppSpec examples show the files and hook structure. [AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-example.html?utm_source=chatgpt.com)

Also distinguish:

- `appspec.yml`: CodeDeploy instructions.
- `buildspec.yml`: CodeBuild build instructions.
- `Jenkinsfile`: Jenkins Pipeline instructions.

---

**11. Write and explain a Dockerfile.**

This example assumes:

- `app.py` exposes a Flask application named `app`.
- `requirements.txt` includes Flask and Gunicorn.
- The application listens through Gunicorn on port 8080.

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN groupadd --gid 10001 app \
    && useradd --uid 10001 --gid app --create-home app

COPY --chown=app:app app.py .

USER app

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

**Explanation**

- `FROM`: chooses the base image.
- `ENV`: configures process defaults.
- `WORKDIR`: sets the working directory.
- `COPY requirements.txt`: allows dependency-layer reuse when only application code changes.
- `RUN`: installs dependencies and creates the runtime user.
- `USER`: runs the application with the selected non-root identity.
- `EXPOSE`: documents the listening port.
- `CMD`: defines the default startup command.

Build and run:

```bash
docker build -t myapp:1.0 .
docker run --rm -p 9090:8080 myapp:1.0
```

The host uses port `9090`, while the application still listens on container port `8080`.

Use an appropriate `.dockerignore`, reviewed dependency versions, trusted updated base images, and build-secret mechanisms where needed. Docker documents these build practices. [Docker Docs](https://docs.docker.com/build/building/best-practices/?utm_source=chatgpt.com)

---

**12. How do you configure environment variables in AWS?**

The mechanism depends on the service:

| Service | Configuration method |
|---|---|
| Lambda | Function environment-variable configuration |
| ECS | Task-definition environment values or secret references |
| CodeBuild | Project or buildspec configuration |
| EC2 | Process configuration, startup scripts, or service environment files |
| EKS | Pod environment configuration, ConfigMaps, and secret integrations |

**Lambda example**

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --environment \
    'Variables={APP_ENV=dev,LOG_LEVEL=INFO}'
```

Read the value in Python:

```python
import os

environment = os.getenv("APP_ENV", "dev")
```

Updating the Lambda environment map replaces its values, so preserve any existing variables that are still required. Lambda documents configuration and retrieval of environment variables. [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html?utm_source=chatgpt.com)

**ECS**

Use `environment` for appropriate configuration values and `secrets` for supported secret references. Plain environment values in a task definition can be visible to identities permitted to read that definition. [Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/taskdef-envfiles.html?utm_source=chatgpt.com)

**EC2**

Exporting a variable in one shell affects that shell’s child processes. For a systemd-managed application, configure its service environment or an appropriate environment file.

Keep ordinary configuration separate from sensitive credentials. Use approved secret storage and role-based access for secrets.

---

**13. What is Ansible? How do you use it?**

**Ansible automates configuration, application deployment, and operational tasks.**

Its main concepts are:

- **Inventory:** target hosts and groups.
- **Playbook:** YAML defining plays and tasks.
- **Modules:** implementations of operations such as installing packages.
- **Variables:** environment-specific inputs.
- **Roles:** reusable task and configuration structure.

For typical Linux automation, the control node connects over SSH and executes the required operations. Ansible playbooks describe tasks against selected hosts. [Ansible Community Documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html?utm_source=chatgpt.com)

**Example playbook**

Assuming suitable repositories and service management are available:

```yaml
---
- name: Install and enable the web server
  hosts: webservers
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Enable and start Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

**Validate and execute**

```bash
ansible-playbook -i inventory.ini web.yml --syntax-check
ansible-playbook -i inventory.ini web.yml --check
ansible-playbook -i inventory.ini web.yml
```

Use idempotent modules where possible: running the same desired configuration again should avoid unnecessary changes.

For credentials, use Ansible Vault or an approved external secret store. Keep SSH and privilege-escalation access controlled, and avoid exposing secret values in task output.
