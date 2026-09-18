# CGI DevOps Interview Questions and Answers

> **Experience:** 4.1 years  
> **Focus:** CI/CD, Jenkins, troubleshooting, security, CVEs, and Docker

## 1. CI/CD and Jenkins

**Continuous Integration** automatically validates frequent code changes through checkout, compilation, tests, quality checks, security scans, and artifact creation. **Continuous Delivery** keeps every accepted change ready for a controlled release. **Continuous Deployment** releases every change that passes all gates automatically.

Jenkins implements these workflows as Pipeline as Code in a `Jenkinsfile`. It supports Declarative and Scripted syntax. Declarative Pipeline uses a structured `pipeline {}` model, while Scripted Pipeline provides greater Groovy flexibility. citeturn5search26

```text
Commit -> Checkout -> Build -> Test -> Scan -> Package -> Publish
       -> Deploy Dev -> Integration Test -> Approval
       -> Deploy Production -> Verify -> Monitor/Rollback
```

Key practices include immutable artifacts, short-lived credentials, shared libraries, parallel checks, quality gates, auditability, approvals, and automatic rollback.

## 2. How did you reduce a pipeline from one hour to 20 minutes?

A strong answer is measurable:

> I enabled stage and queue-time reporting and analyzed multiple builds. Tests consumed approximately 25 minutes sequentially, dependency downloads 12 minutes, the image build 10 minutes, and scans plus agent waits the remaining time. I ran independent tests and scans in parallel, sharded the test suite, cached dependencies with lock-file keys, improved Docker layer reuse, used pre-baked autoscaled agents, avoided repeated checkouts, and skipped irrelevant stages based on changed paths. The median duration fell from about 60 to 20 minutes without removing quality gates.

Steps:

1. Measure queue time and stage execution time.
2. Identify the critical path and flaky or duplicated work.
3. Run lint, unit-test shards, SAST, SCA, and IaC scans concurrently where independent.
4. Cache Maven, Gradle, npm, pip, and BuildKit data safely.
5. Use shallow checkout and avoid repeated SCM operations.
6. Copy dependency manifests before source code in Dockerfiles.
7. Use right-sized ephemeral agents close to registries.
8. Use `when` or change-set rules for conditional stages.
9. Run fast checks first and fail early.
10. Verify that test coverage, security gates, reliability, and cost remain acceptable.

## 3. Checkout stage with Git credentials

Store the SSH key or token in Jenkins Credentials and reference its ID.

```groovy
stage('Checkout') {
    steps {
        cleanWs()
        checkout([
            $class: 'GitSCM',
            branches: [[name: '*/main']],
            userRemoteConfigs: [[
                url: 'git@github.com:example-org/payment-service.git',
                credentialsId: 'github-deploy-key'
            ]],
            extensions: [[
                $class: 'CloneOption',
                depth: 1,
                shallow: true,
                timeout: 10
            ]]
        ])
    }
}
```

In a Multibranch Pipeline, prefer the trigger revision:

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

Use a read-only deploy key, restrict credential scope, verify SSH host keys, and never print secrets.

## 4. Shared variable approach

Choose the mechanism by scope:

- Pipeline-wide values: top-level `environment`.
- Stage-only values: stage-level `environment`.
- User-selected values: typed `parameters`.
- Reusable organization defaults and functions: versioned Jenkins Shared Library.
- Secrets: Jenkins Credentials or an external secret manager.
- Runtime application configuration: ConfigMap, secret manager, or configuration service.

```groovy
@Library('company-pipeline-library@v2') _

pipeline {
    agent any
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'test', 'prod'])
    }
    environment {
        APP_NAME = 'payment-service'
        REGISTRY = 'registry.example.com'
    }
    stages {
        stage('Show Safe Values') {
            steps {
                echo "${env.APP_NAME} -> ${params.ENVIRONMENT}"
            }
        }
    }
}
```

Jenkins supports variables at pipeline and stage scope. Credentials should come from the credential store, not plaintext in a Jenkinsfile. citeturn5search28

## 5. Calling variables in Jenkins

```groovy
environment {
    APP_NAME = 'orders-api'
}
parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'dev')
}
steps {
    echo "App: ${env.APP_NAME}"
    echo "Tag: ${params.IMAGE_TAG}"
    sh 'echo "Shell expansion: $APP_NAME"'
    script {
        def sha = sh(script: 'git rev-parse --short=12 HEAD', returnStdout: true).trim()
        echo "Commit: ${sha}"
    }
}
```

Credentials:

```groovy
withCredentials([usernamePassword(
    credentialsId: 'registry-credential',
    usernameVariable: 'REGISTRY_USER',
    passwordVariable: 'REGISTRY_PASSWORD'
)]) {
    sh '''
        set +x
        printf "%s" "$REGISTRY_PASSWORD" | docker login registry.example.com \
          --username "$REGISTRY_USER" --password-stdin
    '''
}
```

Do not echo credentials. Prefer single-quoted Groovy shell blocks so the shell performs secret expansion.

## 6. What is a Jenkins agent?

A Jenkins agent is a host, VM, container, or Kubernetes Pod that executes pipeline work. The controller schedules jobs and maintains pipeline state; builds should normally execute on agents. The `agent` directive can be applied to the complete pipeline or individual stages. citeturn5search26

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux && java21' }
            steps { sh './mvnw clean verify' }
        }
    }
}
```

Prefer ephemeral, immutable, patched agents; isolate trusted release workers from pull-request workers; enforce resource limits; and do not run builds on the controller.

## 7. Sample production Dockerfile question

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk AS build
WORKDIR /workspace
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN chmod +x mvnw && ./mvnw -B dependency:go-offline
COPY src/ src/
RUN ./mvnw -B clean package -DskipTests && cp target/*.jar /workspace/app.jar

FROM eclipse-temurin:21-jre
RUN groupadd --system --gid 10001 app && \
    useradd --system --uid 10001 --gid app --home-dir /app app
WORKDIR /app
COPY --from=build --chown=app:app /workspace/app.jar /app/app.jar
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Explain build/runtime separation, cache-friendly ordering, non-root execution, exec-form entrypoint, no embedded secrets, small runtime image, and production digest pinning.

## 8. General troubleshooting

1. Confirm impact, scope, start time, and the last change.
2. Compare a failed execution with a successful one.
3. Inspect metrics, events, logs, traces, and deployment history.
4. Follow the request path through DNS, load balancer, service, Pod, dependency, and database.
5. Test one hypothesis at a time and preserve evidence.
6. Mitigate through rollback, scaling, failover, or feature disablement.
7. Fix the root cause and verify recovery.
8. Complete a blameless review with owned preventive actions.

```bash
kubectl get pods -A -o wide
kubectl describe pod POD -n NAMESPACE
kubectl logs POD -n NAMESPACE --all-containers --since=30m
kubectl logs POD -n NAMESPACE --previous
kubectl get events -n NAMESPACE --sort-by=.metadata.creationTimestamp
kubectl get svc,endpointslice,networkpolicy -n NAMESPACE
```

## 9. Database connection works for others but not from your Pod

Start with the exact error: DNS failure, timeout, refused connection, TLS failure, authentication denial, authorization denial, or connection-pool exhaustion.

1. Compare failing and healthy Pods: image digest, namespace, node, service account, ConfigMaps, Secrets, sidecars, environment, and restart history.
2. Validate DNS from the failing context:

```bash
kubectl exec -n team-a FAILING_POD -- cat /etc/resolv.conf
kubectl exec -n team-a FAILING_POD -- getent hosts database.example.internal
```

A short Kubernetes Service name resolves in the caller's namespace. Use `service.namespace` or its FQDN for another namespace. citeturn5search39

3. Test the target port:

```bash
kubectl exec -n team-a FAILING_POD -- \
  sh -c 'nc -vz -w 5 database.example.internal 5432'
```

4. Check egress NetworkPolicy, CNI enforcement, service mesh, security groups, NACLs, routes, and firewall rules. NetworkPolicy controls Pod flows at IP and port level only when implemented by the network plugin. citeturn5search38
5. Validate the Secret version, username, database name, TLS CA, certificate expiry, and clock.
6. Review database authentication logs, grants, connection limits, listener, and observed source IP.
7. If the image is distroless, use an approved ephemeral debug container rather than changing production permanently.

## 10. Database logs are not being written

Check:

- Logging is enabled and the level includes the event.
- Correct destination: file, stdout, stderr, syslog, audit log, or managed export.
- The database loaded the expected configuration.
- Directory ownership, permissions, SELinux/AppArmor, and read-only mounts.
- Disk space and inode availability.
- Rotation configuration and open deleted files.
- Sidecar/collector path, volume mount, filters, quotas, and destination permissions.
- Managed-database log export and retention settings.

```bash
ls -ld /var/log/database
namei -l /var/log/database/database.log
df -h
df -i
lsof +L1
journalctl -u database-service --since "30 minutes ago"

kubectl logs DB_POD -n database --all-containers
kubectl describe pod DB_POD -n database
kubectl get pvc,pv -n database
```

Generate one controlled test event and trace it to the destination. Never blindly delete active database files to recover space.

## 11. CI/CD security

- Protect branches and require review.
- Separate untrusted pull-request jobs from production credentials.
- Use short-lived, least-privilege cloud identities.
- Keep secrets in Jenkins Credentials or a secret manager.
- Pin and patch Jenkins, plugins, tools, dependencies, and base images.
- Run secret scanning, SAST, SCA, IaC scanning, image scanning, and license checks.
- Generate an SBOM and sign artifacts.
- Promote one immutable image digest across environments.
- Restrict script approvals, Jenkins administration, and plugin installation.
- Retain tamper-resistant audit evidence and secure artifact repositories.

## 12. What are CVEs?

**CVE** means **Common Vulnerabilities and Exposures**. A CVE identifier provides a standard reference for a publicly disclosed software or hardware vulnerability.

A CVE is not by itself a risk priority. Also evaluate CVSS vector, exploit availability, known exploitation, application reachability, internet exposure, required privileges, data sensitivity, compensating controls, and fix availability.

Process:

```text
Discover -> Validate -> Prioritize -> Remediate -> Test -> Rescan -> Document
```

## 13. Production CVE example and resolution

Do not claim a vulnerability you did not handle. A credible example is:

> I handled findings in Linux base packages, cryptographic libraries, application frameworks, logging libraries, and transitive dependencies. For a critical library finding, we used scanner results and the SBOM to locate affected repositories and image digests. We assessed whether the vulnerable feature was reachable and prioritized externally exposed services. We upgraded the dependency and base image, rebuilt the artifact, ran regression and security tests, deployed by canary, and rescanned the exact production digest. Until the permanent fix was available, we disabled the vulnerable feature, tightened egress or WAF controls, increased monitoring, and tracked a time-bound exception.

Resolution steps:

1. Validate package name, installed version, and scanner evidence.
2. Determine runtime reachability and exposure.
3. Inventory all affected images, hosts, repositories, and environments.
4. Apply the vendor-fixed version or approved mitigation.
5. Rebuild from clean, trusted inputs.
6. Run functional, regression, and security tests.
7. Deploy progressively with rollback.
8. Rescan the deployed digest and retain evidence.
9. Add automated dependency updates and prevention gates.

## 14. Five CVE tools

1. **Trivy:** Images, filesystems, repositories, IaC, and SBOMs.
2. **Snyk:** Dependencies, containers, code, and IaC with upgrade guidance.
3. **Grype:** Images, filesystems, and SBOM vulnerability matching.
4. **Dependabot:** Automated supported dependency security updates.
5. **Amazon Inspector:** Continuous vulnerability and exposure assessment for supported AWS workloads.

Other examples include Prisma Cloud, Aqua, Anchore, JFrog Xray, Mend, Sonatype Lifecycle, and Clair. A scanner identifies findings; remediation normally requires upgrade, rebuild, mitigation, testing, and rescan.

## 15. Docker topics to prepare

Be ready for image versus container, layers, build context, `.dockerignore`, `CMD` versus `ENTRYPOINT`, `COPY` versus `ADD`, `ARG` versus `ENV`, bind mounts versus volumes, networking, health checks, signal handling, BuildKit caching, registries, tags versus digests, resource controls, rootless/non-root execution, SBOMs, signing, and scanning.

```bash
docker build -t payment-api:1.0.0 .
docker image inspect payment-api:1.0.0
docker history payment-api:1.0.0
docker run --rm --read-only --user 10001:10001 payment-api:1.0.0
docker logs CONTAINER_ID
docker stats
```

## 16. Sample multi-stage Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25-alpine AS build
RUN apk add --no-cache ca-certificates git
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath -ldflags="-s -w" -o /out/service ./cmd/service

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /out/service /service
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["/service"]
```

Multi-stage builds use multiple `FROM` instructions and copy only required artifacts into the final stage, keeping compilers and build tools out of the runtime image. citeturn5search32turn5search33

## 17. Detailed Jenkins pipeline stages

```groovy
@Library('company-pipeline-library@v2') _

pipeline {
    agent none

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds(abortPrevious: true)
        buildDiscarder(logRotator(numToKeepStr: '30'))
        skipDefaultCheckout(true)
    }

    parameters {
        choice(name: 'TARGET_ENV', choices: ['dev', 'test', 'prod'])
    }

    environment {
        APP_NAME = 'payment-service'
        REGISTRY = 'registry.example.com/team'
        IMAGE_REPOSITORY = "${REGISTRY}/${APP_NAME}"
    }

    stages {
        stage('Checkout') {
            agent { label 'linux-small' }
            steps {
                cleanWs()
                checkout scm
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
                }
                stash name: 'source', includes: '**/*', useDefaultExcludes: false
            }
        }

        stage('Fast Validation') {
            parallel {
                stage('Lint') {
                    agent { label 'linux-small' }
                    steps {
                        unstash 'source'
                        sh './mvnw -B spotless:check'
                    }
                }
                stage('Secret Scan') {
                    agent { label 'security-tools' }
                    steps {
                        unstash 'source'
                        sh 'gitleaks detect --redact'
                    }
                }
                stage('IaC Validate') {
                    agent { label 'security-tools' }
                    when { changeset 'infra/**' }
                    steps {
                        unstash 'source'
                        sh 'terraform -chdir=infra fmt -check -recursive'
                        sh 'terraform -chdir=infra init -backend=false'
                        sh 'terraform -chdir=infra validate'
                    }
                }
            }
        }

        stage('Build and Unit Test') {
            agent { label 'linux-java21' }
            steps {
                unstash 'source'
                sh './mvnw -B clean verify'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml'
                }
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }

        stage('Security and Quality') {
            parallel {
                stage('SAST') {
                    agent { label 'security-tools' }
                    steps {
                        unstash 'source'
                        sh './ci/run-sast.sh'
                    }
                }
                stage('Dependency Scan') {
                    agent { label 'security-tools' }
                    steps {
                        unstash 'source'
                        sh './ci/scan-dependencies.sh'
                    }
                }
            }
        }

        stage('Build Image') {
            agent { label 'trusted-container-builder' }
            steps {
                unstash 'source'
                sh '''
                    DOCKER_BUILDKIT=1 docker build \
                      --label "org.opencontainers.image.revision=$GIT_SHA" \
                      --tag "$IMAGE_REPOSITORY:$IMAGE_TAG" .
                '''
            }
        }

        stage('Scan and Publish') {
            agent { label 'trusted-container-builder' }
            when { branch 'main' }
            steps {
                sh '''
                    trivy image --exit-code 1 --severity CRITICAL \
                      "$IMAGE_REPOSITORY:$IMAGE_TAG"
                    syft "$IMAGE_REPOSITORY:$IMAGE_TAG" \
                      -o cyclonedx-json=sbom.json
                '''
                withCredentials([usernamePassword(
                    credentialsId: 'registry-credential',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASSWORD'
                )]) {
                    sh '''
                        set +x
                        printf "%s" "$REGISTRY_PASSWORD" | docker login "$REGISTRY" \
                          --username "$REGISTRY_USER" --password-stdin
                        docker push "$IMAGE_REPOSITORY:$IMAGE_TAG"
                    '''
                }
                archiveArtifacts artifacts: 'sbom.json', fingerprint: true
            }
        }

        stage('Deploy Development') {
            agent { label 'deployment-tools' }
            when { branch 'main' }
            steps {
                sh '''
                    helm upgrade --install "$APP_NAME" ./helm/$APP_NAME \
                      --namespace development --create-namespace \
                      --set image.repository="$IMAGE_REPOSITORY" \
                      --set image.tag="$IMAGE_TAG" \
                      --atomic --timeout 5m
                '''
            }
        }

        stage('Integration Test') {
            agent { label 'linux-test' }
            when { branch 'main' }
            steps { sh './ci/run-integration-tests.sh development' }
        }

        stage('Production Approval') {
            agent none
            when {
                allOf {
                    branch 'main'
                    expression { params.TARGET_ENV == 'prod' }
                }
            }
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                    input message: "Deploy ${env.IMAGE_TAG} to production?",
                          submitter: 'production-approvers'
                }
            }
        }

        stage('Deploy Production') {
            agent { label 'production-deployer' }
            when {
                allOf {
                    branch 'main'
                    expression { params.TARGET_ENV == 'prod' }
                }
            }
            steps {
                sh '''
                    helm upgrade --install "$APP_NAME" ./helm/$APP_NAME \
                      --namespace production \
                      --set image.repository="$IMAGE_REPOSITORY" \
                      --set image.tag="$IMAGE_TAG" \
                      --atomic --timeout 10m
                    kubectl rollout status deployment/$APP_NAME \
                      -n production --timeout=5m
                '''
            }
        }

        stage('Smoke Test') {
            agent { label 'linux-test' }
            when {
                allOf {
                    branch 'main'
                    expression { params.TARGET_ENV == 'prod' }
                }
            }
            steps { sh './ci/smoke-test.sh https://payment.example.com' }
        }
    }

    post {
        success { echo "Succeeded: ${env.BUILD_URL}" }
        failure { echo "Failed: ${env.BUILD_URL}" }
        always { cleanWs(deleteDirs: true, notFailBuild: true) }
    }
}
```

Explanation:

- `agent none` prevents one worker being occupied for the entire workflow.
- Each stage uses an agent suited to its trust and tool requirements.
- Checkout happens once and a traceable commit-based image tag is created.
- Independent checks run in parallel.
- Test reports and artifacts are archived and fingerprinted.
- Credentials exist only around the steps that require them.
- The image is built once, scanned, given an SBOM, and published.
- Lower-environment deployment and integration testing precede production approval.
- `--atomic` helps Helm roll back a failed release.
- Post actions report status and clean the workspace.

Declarative Pipeline stages, directives, stage agents, steps, and parallel blocks provide this structure. citeturn5search26

## Quick revision

- Measure before optimizing a pipeline.
- Parallelize independent stages and provide enough executors.
- Use Shared Libraries for common organization logic.
- Use `env.NAME`, `params.NAME`, and scoped Groovy variables correctly.
- Use Jenkins Credentials and never echo secrets.
- Controllers coordinate; agents execute.
- Multi-stage images exclude build tools from the runtime layer.
- Debug Pod-to-database failures through DNS, TCP, policy, TLS, credentials, and database evidence.
- CVE severity alone is not business risk; assess reachability and exposure.
- Build once and promote the same immutable image.
