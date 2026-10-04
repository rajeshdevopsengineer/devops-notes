**1. What is a Maven release?**

A Maven release produces a versioned artifact that can be traced to a specific source-code revision.

For example:

```text
Development version: 1.2.0-SNAPSHOT
Release version:     1.2.0
Next development:    1.2.1-SNAPSHOT
```

A `SNAPSHOT` represents ongoing development and can change. A published release should remain immutable.

The **Maven Release Plugin** commonly manages two steps.

**`release:prepare`**

It checks the working copy and SNAPSHOT dependencies, updates project versions, runs configured preparation checks, commits changes, creates the release tag, and advances the project to its next development version. [Maven Release plugin](https://maven.apache.org/maven-release/maven-release-plugin/usage/prepare-release.html?utm_source=chatgpt.com)

Example:

```bash
mvn -B release:prepare \
  -DreleaseVersion=1.2.0 \
  -DdevelopmentVersion=1.2.1-SNAPSHOT \
  -Dtag=payments-1.2.0
```

`-B` enables batch mode. With Git, the plugin’s `pushChanges` setting defaults to true, so prepare can push commits and tags upstream. The release process must align with branch protections and the automation account’s permissions. [Maven Release plugin](https://maven.apache.org/maven-release/maven-release-plugin/prepare-mojo.html?utm_source=chatgpt.com)

**`release:perform`**

It checks out the release tag into a separate working directory and executes the configured release goals.

```bash
mvn -B release:perform -Dgoals=deploy
```

Here, `deploy` publishes the artifact to the configured remote Maven repository. [Maven Release plugin](https://maven.apache.org/maven-release/maven-release-plugin/perform-mojo.html?utm_source=chatgpt.com)

Prerequisites usually include:

- Correct SCM configuration.
- A clean working copy.
- Approved dependency versions.
- A configured artifact repository.
- Scoped SCM and repository credentials.
- Passing release checks.

Repository credentials belong in protected Maven settings supplied through the CI secret mechanism. [Maven](https://maven.apache.org/settings.html?utm_source=chatgpt.com)

**Important distinction:** Publishing a Maven artifact and deploying an application to Kubernetes are separate pipeline operations.

**2. Explain the Maven lifecycle**

Maven has three built-in lifecycles:

| Lifecycle | Purpose |
|---|---|
| `default` | Build, test, package, verify, and publish the project |
| `clean` | Remove generated build output |
| `site` | Generate and publish project documentation |

Common phases in the default lifecycle are:

| Phase | Typical purpose |
|---|---|
| `validate` | Validate project configuration |
| `compile` | Compile application sources |
| `test` | Run unit tests |
| `package` | Produce the JAR, WAR, or other package |
| `integration-test` | Execute configured integration tests |
| `verify` | Check results and configured quality requirements |
| `install` | Place the project artifact in the local repository |
| `deploy` | Publish it to a remote repository |

Calling a phase runs the preceding phases in that lifecycle:

```bash
mvn verify
```

The actual work depends on packaging and plugin goals bound to phases; integration tests require appropriate configuration. [Maven](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html?utm_source=chatgpt.com)

A **phase** is a lifecycle stage. A **goal** is a specific plugin operation:

```bash
mvn dependency:tree
```

Combining lifecycles is common:

```bash
mvn clean install
```

This performs cleaning, then the default lifecycle through `install`.

**3. What is the purpose of `<dependencyManagement>`?**

`<dependencyManagement>` centralizes dependency versions and other supported dependency settings.

It supplies managed configuration when a dependency is used. **Declaring it there alone does not add the dependency to the application.**

For example, a parent POM can manage an illustrative internal library:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>shared-utils</artifactId>
            <version>1.4.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

A child project that inherits this parent can declare:

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>shared-utils</artifactId>
    </dependency>
</dependencies>
```

The child obtains the managed version without repeating it.

| Section | Responsibility |
|---|---|
| `<dependencies>` | Declare dependencies used by the project |
| `<dependencyManagement>` | Manage dependency configuration and versions |

This helps keep modules consistent and control versions selected for transitive dependencies. An explicit version in a direct dependency declaration can override its managed default. [Maven](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html?utm_source=chatgpt.com)

A **BOM—Bill of Materials** imports a coordinated set of managed dependencies using:

```xml
<type>pom</type>
<scope>import</scope>
```

The import belongs inside `<dependencyManagement>`. [Maven](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html?utm_source=chatgpt.com)

Useful diagnostics:

```bash
mvn dependency:tree
mvn help:effective-pom
```

Plugin configuration is managed separately through build/plugin configuration, including `<pluginManagement>`.

**4. How do you manage Kubernetes Pods? They run on Linux, right?**

Linux worker nodes are common for Linux containers. Kubernetes also supports Windows worker nodes for Windows workloads. [Kubernetes](https://kubernetes.io/docs/concepts/windows/intro/?utm_source=chatgpt.com)

Pods are scheduled onto worker nodes. The node’s **kubelet** works with a container runtime, such as containerd, to run their containers. [Kubernetes](https://kubernetes.io/docs/concepts/overview/components/?utm_source=chatgpt.com)

I would manage application workloads through the appropriate controller:

| Controller | Use |
|---|---|
| Deployment | Replicated application workloads and rolling updates |
| StatefulSet | Workloads requiring stable identities and storage relationships |
| DaemonSet | Node-level workloads |
| Job/CronJob | Finite or scheduled processing |

For a typical Java application:

1. Maven builds the artifact.
2. Docker packages it into an image.
3. The pipeline publishes the image.
4. Helm, manifests, or a GitOps controller deploys it.
5. Kubernetes reconciles the declared workload.

Useful commands:

```bash
kubectl get nodes -L kubernetes.io/os

kubectl get pods -n payments -o wide

kubectl apply -f deployment.yaml -n payments

kubectl rollout status deployment/payments -n payments

kubectl describe pod <pod-name> -n payments

kubectl logs <pod-name> -n payments --previous
```

Production management also includes probes, requests and limits, scaling, secrets, access controls, and monitoring.

If GitOps owns the workload, make lasting configuration changes in Git. An imperative change may otherwise be reverted during reconciliation.

**5. What happens when you run `mvn install`?**

Maven executes the default lifecycle through `install`, including the preceding build and configured validation phases.

The install plugin then places the project’s main artifact, POM, and any attached artifacts into the **local Maven repository**. Maven uses the artifact’s coordinates to determine its location. [Apache Maven Install Plugin](https://maven.apache.org/plugins/maven-install-plugin/?utm_source=chatgpt.com)

For:

```text
groupId:    com.example
artifactId: payments
version:    1.2.0
packaging:  jar
```

Typical locations are:

```text
Build output:
target/payments-1.2.0.jar

Installed artifact:
~/.m2/repository/com/example/payments/1.2.0/payments-1.2.0.jar
```

The default local repository is `${user.home}/.m2/repository`, but settings can change it. [Maven](https://maven.apache.org/settings.html?utm_source=chatgpt.com)

| Command | Result for the project artifact |
|---|---|
| `mvn package` | Creates the build package |
| `mvn install` | Also installs it locally |
| `mvn deploy` | Also publishes it to the configured remote repository |

On a CI agent, “local” means that agent’s configured repository.

Also, `mvn install` does not automatically clean previous output. Use:

```bash
mvn clean install
```

when cleaning is required. Application deployment and Maven release tagging remain separately configured operations.

**6. What branching strategies do you use?**

Describe the strategy your actual team follows. Common approaches are:

| Strategy | Structure | Suitable context |
|---|---|---|
| GitHub flow | Feature branches, pull requests, protected main | Frequent releases |
| Trunk-based development | Frequent integration into main/trunk, very short-lived branches | Strong automated validation and frequent delivery |
| Gitflow | Main, develop, feature, release, and hotfix branches | Explicit release cycles and stabilization |
| Release-branch approach | Main development plus maintained release branches | Supporting multiple released versions |

GitHub flow uses branches and pull requests to propose and review changes before merging. [GitHub Docs](https://docs.github.com/en/get-started/using-github/github-flow?utm_source=chatgpt.com)

Traditional Gitflow separates production history on `main` from integration on `develop`, with supporting release and hotfix branches. [nvie.com](https://nvie.com/posts/a-successful-git-branching-model/?utm_source=chatgpt.com)

For a frequent-delivery workflow, a possible process is:

1. Create a short-lived feature branch.
2. Run CI checks on the pull request.
3. Review and merge into protected main.
4. Create a traceable release artifact and tag.
5. Promote the same approved artifact through environments.

Example:

```bash
git switch main
git pull --ff-only

git switch -c feature/payment-validation

# Make and commit changes
git push -u origin feature/payment-validation
```

Microsoft’s branching guidance also emphasizes short-lived feature branches and keeping main healthy, with release branches where needed. [Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance?view=azure-devops\&utm_source=chatgpt.com)

For hotfixes, ensure the correction reaches every maintained branch that needs it. Branching alone does not control deployment: pipeline conditions, environment permissions, and approvals provide that control.

**7. Where is `pom.xml` located?**

Normally, `pom.xml` is at the **Maven project’s root directory**.

A typical layout is:

| Path | Purpose |
|---|---|
| `pom.xml` | Project configuration |
| `src/main/java/` | Application source |
| `src/main/resources/` | Application resources |
| `src/test/java/` | Test source |
| `src/test/resources/` | Test resources |
| `target/` | Generated build output |

This is Maven’s standard directory layout. [Maven](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html?utm_source=chatgpt.com)

For a multi-module repository:

| Example path | Role |
|---|---|
| `pom.xml` | Root/aggregator POM |
| `payment-api/pom.xml` | API module POM |
| `payment-worker/pom.xml` | Worker module POM |

The aggregator lists modules. Parent inheritance and aggregation are related capabilities, but a POM does not necessarily serve both roles. [Maven](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html?utm_source=chatgpt.com)

Usually run Maven from the target project directory:

```bash
cd payment-api
mvn clean verify
```

Or specify the POM:

```bash
mvn -f payment-api/pom.xml clean verify
```

In CI, the repository is checked out into the agent workspace, and Maven reads the selected POM from there.

The POM contains project coordinates, dependencies, build plugins, profiles, packaging, and potentially module declarations. Maven settings provide execution-specific configuration such as repository credentials and mirrors.
