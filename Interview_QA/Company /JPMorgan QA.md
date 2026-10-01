Below are answers for all three rounds. Adapt the project and experience examples to your actual work. The Java examples use Java 21; Spring Boot version differences are called out where relevant.

**Round 1**

**1. Explain the project you are currently working on.**

Structure your answer around **business purpose, architecture, responsibilities, and results**.

An example to adapt:

> “The project is an order-processing platform with services for orders, payments, inventory, and shipping. We use Spring Boot for REST APIs and business logic, relational databases for transactional data, and messaging for asynchronous workflows.
>
> Applications run as containers on AWS. Our pipeline builds the application, runs tests and security checks, publishes an image, and deploys the same tested artifact across environments. Okta handles user authentication, while individual services enforce authorization.
>
> My responsibilities include [the services or platform components you own], implementation, automated testing, deployment support, and production troubleshooting.”

Be ready to explain:

- One complete request flow.
- Which decisions you personally made.
- How you handle failures and data consistency.
- Actual traffic, availability targets, and improvements, where known.

Avoid listing technologies without explaining their purpose.

---

**2. What Spring Boot starters have you used?**

A starter supplies a coordinated set of dependencies for a capability. Spring Boot then uses the available libraries and configuration to configure components.

Common examples:

| Capability | Dependencies |
|---|---|
| REST APIs using Spring MVC | Boot 3: `spring-boot-starter-web`; Boot 4: `spring-boot-starter-webmvc` |
| JPA/Hibernate persistence | `spring-boot-starter-data-jpa` |
| JDBC access | `spring-boot-starter-jdbc` |
| Request validation | `spring-boot-starter-validation` |
| Authentication and authorization | `spring-boot-starter-security` |
| OAuth resource server | Boot 3: `spring-boot-starter-oauth2-resource-server`; Boot 4: `spring-boot-starter-security-oauth2-resource-server` |
| Health and operational endpoints | `spring-boot-starter-actuator` |
| Testing | `spring-boot-starter-test` |
| Kafka | Boot 3 commonly uses `spring-kafka`; Boot 4 provides `spring-boot-starter-kafka` |

These names are version-sensitive: current Spring Boot documentation deprecates some older starter names in favor of the more specific names above. :chatgpt-content-reference{index="0"}

Explain which starters your application actually uses and why. For example, JPA simplifies persistence, but you still need to understand generated SQL, transactions, connection pools, and query performance.

---

**3. How have you implemented transaction management?**

For operations within one database, I would usually put the transaction boundary at the **service layer**.

```java
@Service
public class OrderService {

    // Constructor-injected repositories omitted.

    @Transactional(rollbackFor = Exception.class)
    public void placeOrder(Order order) throws BusinessException {
        inventoryRepository.reserve(
            order.getProductId(),
            order.getQuantity()
        );

        orderRepository.save(order);
    }
}
```

Assume both repositories participate in the same database transaction. If saving the order fails, the stock reservation should roll back.

Important details:

- Default propagation is `REQUIRED`: join an existing transaction or create one.
- Without customized rollback rules, Spring rolls back for `RuntimeException` and `Error`, but not ordinarily for checked exceptions.
- `rollbackFor` lets you specify checked exceptions that should trigger rollback.
- An exception must reach the transaction interceptor, or the transaction must otherwise be marked rollback-only. Catching and suppressing an exception can allow a commit. :chatgpt-content-reference{index="1"}

Common mistakes:

- Calling an annotated method from another method in the same object. Default proxy-based interception does not apply to this self-invocation. :chatgpt-content-reference{index="2"}
- Assuming a transaction automatically propagates into newly created threads.
- Holding a database transaction open during a slow remote API call.
- Assuming `readOnly = true` is a universal database-enforced prohibition on writes. :chatgpt-content-reference{index="3"}

For concurrent updates, also consider database constraints, isolation, and optimistic or pessimistic locking.

---

**4. How do you connect to two databases and ensure rollback during exceptions?**

There are two separate problems: **configuring connections** and **coordinating transactions**.

For two databases, configure:

| Component | Orders database | Payments database |
|---|---|---|
| Connection pool | `ordersDataSource` | `paymentsDataSource` |
| JDBC access | Orders `JdbcTemplate` | Payments `JdbcTemplate` |
| JPA alternative | Orders `EntityManagerFactory` | Payments `EntityManagerFactory` |
| Local transaction manager | `ordersTxManager` | `paymentsTxManager` |

With JPA, explicitly associate each repository package with its entity-manager factory and transaction manager. Use qualifiers to avoid ambiguous injection. Spring Boot documents explicit configuration for multiple data sources. :chatgpt-content-reference{index="4"}

A local transaction can select its manager:

```java
@Transactional(
    transactionManager = "ordersTxManager",
    rollbackFor = Exception.class
)
public void updateOrder(Order order) {
    orderRepository.save(order);
}
```

**This annotation does not automatically include the payments database.**

Suppose:

1. Database A commits.
2. Database B fails.

Two independent local transaction managers cannot undo A’s completed commit.

Choose the consistency model deliberately:

- **Both databases must commit atomically:** Consider XA-capable resources and a JTA transaction coordinator. This requires correct resource enlistment, recovery configuration, and support from both databases.
- **Independent microservices:** Use local transactions with a saga, durable messaging, and compensating operations.
- **Closely related data:** Consider whether the operation should live within one database transaction.

Spring supports JTA-based distributed transactions, including integration with XA resources. Simply declaring two local transaction managers is insufficient. :chatgpt-content-reference{index="5"}

Verify rollback through integration tests that deliberately fail the second operation and inspect both databases afterward.

---

**5. How are you managing code coverage?**

A common Java setup is:

- **JUnit:** Test execution.
- **Mockito:** Isolating collaborators in unit tests.
- **Testcontainers:** Integration tests with real database or broker containers.
- **JaCoCo:** Measuring executed code.
- **SonarQube:** Importing coverage reports and enforcing quality gates.

In Maven, configure JaCoCo’s agent, report generation, and optional coverage checks. The pipeline runs:

```bash
mvn clean verify
```

SonarQube imports the generated JaCoCo XML report; SonarQube does not itself execute the tests or generate the underlying coverage data. :chatgpt-content-reference{index="6"}

Focus on:

- Line and branch coverage.
- Coverage of changed code.
- Error paths, authorization failures, rollback, and retry exhaustion.
- Integration behavior across real boundaries.

An example threshold might be 80% coverage on new code, but the team should agree on the target. High coverage does not prove that assertions are meaningful or that the application is correct.

---

**6. How do you perform load testing?**

Start with the workload and performance objectives:

- Expected requests per second.
- User journeys and request mix.
- Peak load and burst behavior.
- Acceptable latency and error rates.
- Realistic data size and downstream behavior.

Use tools such as **k6, JMeter, or Gatling**.

Run different tests:

| Test | Purpose |
|---|---|
| Baseline/load | Validate normal and expected peak traffic |
| Stress | Find capacity limits and failure behavior |
| Spike | Check sudden traffic increases |
| Soak | Detect leaks and degradation over time |

These test types answer different performance questions. :chatgpt-content-reference{index="7"}

Monitor application latency—especially p95/p99—throughput, errors, CPU, heap, GC, connection-pool waits, database latency, and queue lag.

For example, rising latency with low CPU but a saturated database pool suggests connection contention rather than insufficient application CPU.

Use a representative test environment and payment-provider sandboxes or controlled substitutes. Also distinguish **virtual users** from **requests per second**; they are not interchangeable.

---

**7. How have you implemented security?**

Explain security at several layers:

- **Identity:** Okta authenticates users; services validate access tokens.
- **Authorization:** Enforce scopes/roles and object-level ownership checks.
- **Application:** Validate inputs, use parameterized database access, restrict uploads, and avoid sensitive data in logs.
- **Network:** TLS, restricted ingress/egress, private database access, and appropriate service-to-service authentication.
- **Cloud:** Least-privilege IAM roles, managed secrets, encryption, and audit trails.
- **Delivery:** Dependency scanning, secret scanning, image scanning, and controlled production deployment.

For example, possession of `orders.read` should not allow a customer to read every customer’s orders. The service must also check ownership or the appropriate business entitlement.

Separate **authentication—who is calling—from authorization—what that caller may do**. Spring Security resource-server support validates tokens, while application rules determine permitted operations. :chatgpt-content-reference{index="8"}

---

**8. Explain Okta integration for identity and access management.**

Okta can act as the identity provider and authorization server.

For a user-facing application:

1. Register the application and redirect URIs in Okta.
2. Configure the appropriate authorization server, scopes, audience, and access policies.
3. Use Authorization Code flow, with PKCE where appropriate.
4. The client obtains tokens.
5. It sends an **access token** to the API.
6. The API validates that token and enforces authorization.

An **ID token** describes the authenticated user to the client. It should not be substituted for an API access token. :chatgpt-content-reference{index="9"}

A Spring resource-server configuration can use:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OKTA_ISSUER}
          audiences: api://orders
```

The issuer must be the authorization server configured for your API. Validate:

- Signature and trusted signing keys.
- Issuer.
- Audience.
- Expiry and other applicable time claims.
- Required scopes or mapped authorities.

Spring Security supports issuer-based configuration and audience validation. :chatgpt-content-reference{index="10"}

For method-level authorization, enable method security and apply a rule:

```java
@PreAuthorize("hasAuthority('SCOPE_orders.read')")
public Order getOrder(Long orderId) {
    // Also verify the caller's entitlement to this order.
    return orderService.findAuthorizedOrder(orderId);
}
```

For machine-to-machine access, use an appropriate service application and client-credentials flow. Keep client credentials out of browser code and source control.

---

**9. How do you secure internal communication between microservices?**

A private network alone is insufficient.

Use:

1. **TLS** to protect traffic in transit.
2. **Workload authentication**, such as mTLS or service access tokens.
3. **Authorization policies** specifying which service may call which operation.
4. **Network restrictions**, such as security groups and Kubernetes NetworkPolicies.
5. **Short-lived credentials and automatic rotation.**
6. **Audit and trace correlation** without logging secrets.

A service mesh such as Istio can provide workload identity, certificate management, mTLS, and authorization policies. :chatgpt-content-reference{index="11"}

Example: the order service may call the payment service’s authorization operation, while the reporting service may only read approved payment summaries.

mTLS authenticates communicating workloads; it does not automatically establish every business permission.

---

**10. How do you implement retries for a failed API call?**

Retry **transient failures**, using:

- A small attempt limit.
- Exponential backoff and jitter.
- Per-attempt timeouts and an overall deadline.
- Idempotency for operations with side effects.
- Metrics for attempts, exhaustion, and downstream failures.

Do not blindly retry validation errors or authentication failures. For throttling, respect `Retry-After` where applicable.

**Spring Framework 7 / Spring Boot 4 example:**

```java
@Configuration
@EnableResilientMethods
class ResilienceConfig {
}
```

On a method in a Spring-managed bean:

```java
@Retryable(
    includes = TransientApiException.class,
    maxRetries = 2,
    delay = 200,
    jitter = 100,
    multiplier = 2,
    maxDelay = 1000
)
public StockResponse getStock(String sku) {
    return inventoryClient.getStock(sku);
}
```

These annotations are from `org.springframework.resilience.annotation`. Here, two retries mean **at most three total attempts**. Configure the HTTP client to classify retryable failures and enforce timeouts. :chatgpt-content-reference{index="12"}

For Boot 3 projects, use a compatible Resilience4j or existing Spring Retry configuration; annotation names and attempt-count semantics differ.

For payments, a timeout can mean “payment succeeded but the response was lost.” Reuse the same idempotency key or reconcile status before attempting another charge.

---

**11. How do you implement blue-green deployment?**

Maintain two application versions:

- **Blue:** Currently serving production.
- **Green:** New version undergoing validation.

A typical AWS container deployment:

1. Build and publish an immutable image.
2. Deploy green with sufficient capacity.
3. Run health checks, smoke tests, and dependency checks.
4. Switch production traffic to green.
5. Monitor errors, latency, and business success.
6. Keep blue available during a defined observation period.
7. Roll traffic back if necessary.

Amazon ECS supports native blue-green deployments, including validation before traffic shifting and a bake period with both revisions running. Existing CodeDeploy-based deployments are another supported implementation. :chatgpt-content-reference{index="13"}

The difficult part is often **database compatibility**:

- Use expand-and-contract schema changes.
- Keep both application versions compatible during transition.
- Do not assume switching traffic back reverses database changes.
- Coordinate background consumers separately to avoid unintended duplicate side effects.

---

**12. How do you deploy an application to AWS?**

For an illustrative Spring Boot application on **ECS Fargate**:

1. Provision networking, IAM roles, ECR, ECS, load balancing, and databases through infrastructure as code.
2. Run tests, quality checks, and security scans.
3. Build and push the image to ECR.
4. Register a task-definition revision with the image digest, resources, logging, and configuration.
5. Deploy that revision through an ECS service.
6. Validate health and monitor the rollout.

Typical supporting services:

- **ALB:** Application traffic.
- **RDS/Aurora:** Relational data.
- **Secrets Manager:** Application credentials.
- **CloudWatch:** Logs, metrics, and alarms.
- **Route 53 and ACM:** DNS and certificates.

Separate the **task execution role**, used for supported startup operations such as image pulls, from the **task role**, used by application code to access AWS services. ECS manages task placement and service operation. :chatgpt-content-reference{index="14"}

On EKS, the deployment objects instead include Deployments, Services, workload identities, and an appropriate ingress implementation.

---

**13. What is serverless in AWS, and how are you using it?**

Serverless means the provider manages the underlying server infrastructure and execution capacity. You still manage application behavior, permissions, configuration, and reliability.

A sample use case:

> “An S3 upload triggers a Lambda function that validates the file and submits processing work. Results are stored durably, and failures are retried or routed for investigation.”

Other examples include scheduled reconciliation, notifications, event processing, and APIs.

Lambda is event-driven compute without managing application servers. Serverless architecture can also use services such as API Gateway, SQS, EventBridge, and DynamoDB. :chatgpt-content-reference{index="15"}

Choose it based on workload duration, latency, concurrency, integration, and cost—not simply because it eliminates server administration.

---

**14. Payment succeeds, but order or shipping fails. What do you do?**

Treat this as a **distributed workflow**, not a local database rollback.

A robust design is:

1. **Persist an order intent before charging.**  
   Create a durable order or workflow record with a unique operation ID.

2. **Make payment requests idempotent.**  
   Repeated requests for the same logical charge use the same idempotency key.

3. **Persist confirmed payment status.**  
   Update local state and write a `PaymentConfirmed` outbox event in one database transaction.

4. **Deliver the event reliably.**  
   An outbox publisher or CDC process sends it to the broker. Consumers handle duplicate delivery safely.

5. **Retry temporary downstream failures.**  
   Shipping unavailability should ordinarily leave the workflow pending and retriable, rather than immediately triggering another charge or refund.

6. **Compensate permanent failures.**  
   If the order cannot be fulfilled, follow business policy to cancel and initiate a refund. Persist states such as `REFUND_PENDING` until completion is confirmed.

A saga coordinates these local transactions and compensating actions. Compensation is a new business operation; it does not erase the original transaction. :chatgpt-content-reference{index="16"}

**Critical failure window:** Payment can succeed externally while your local payment-status write fails. An outbox alone cannot close that gap. Use durable provider callbacks, status queries, and reconciliation keyed by the operation ID.

The outbox specifically addresses the database-write/event-publication problem. Delivery can still be duplicated, so consumers need idempotency—for example, recording a processed event ID in the same transaction as their business update. :chatgpt-content-reference{index="17"}

---

**15. Find duplicate colors and their counts using streams.**

For a non-null list of non-null color names:

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

List<String> colors = List.of(
    "Red", "Blue", "Red", "Green", "Blue", "Red"
);

Map<String, Long> counts = colors.stream()
    .collect(Collectors.groupingBy(
        Function.identity(),
        LinkedHashMap::new,
        Collectors.counting()
    ));

Map<String, Long> duplicates = counts.entrySet().stream()
    .filter(entry -> entry.getValue() > 1)
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        Map.Entry::getValue,
        (first, second) -> first,
        LinkedHashMap::new
    ));

System.out.println(counts);
System.out.println(duplicates);
```

Output:

```text
{Red=3, Blue=2, Green=1}
{Red=3, Blue=2}
```

`groupingBy` groups equal colors, while `counting()` produces a `Long` count. `LinkedHashMap` preserves first-seen order. :chatgpt-content-reference{index="18"}

This comparison is case-sensitive. Normalize values first if `"Red"` and `"red"` should be treated as the same color.

---

**16. Remove duplicate colors and return a unique list.**

```java
List<String> uniqueColors = colors.stream()
    .distinct()
    .toList();

System.out.println(uniqueColors);
```

Output:

```text
[Red, Blue, Green]
```

For an ordered stream, `distinct()` preserves encounter order. Equality is determined through `equals()`.

In Java 21, `Stream.toList()` returns an unmodifiable list. If mutation is required:

```java
List<String> mutableUniqueColors = colors.stream()
    .distinct()
    .collect(Collectors.toCollection(ArrayList::new));
```

For custom objects, implement equality consistently with the intended definition of a duplicate. :chatgpt-content-reference{index="19"}

---

**17. Use two candles to measure 45 minutes without cutting or measuring.**

The standard puzzle requires these assumptions:

- Each candle—or, more commonly, rope—takes exactly **60 minutes** to burn from one end.
- Burning can be uneven.
- You can light both ends.

Procedure:

1. At the start, light candle A at **both ends** and candle B at **one end**.
2. A finishes after **30 minutes**.
3. Immediately light the other end of B.
4. B has 30 minutes of one-ended burn time remaining. Burning the remainder from both ends takes **15 minutes**.

Elapsed time: **30 + 15 = 45 minutes**.

Without the known one-hour burn duration and the ability to burn from both ends, the question does not contain enough information. The reasoning uses the idealized puzzle model; ordinary physical candles may not behave that way.

---

**Round 2**

**1. How are you provisioning AWS services?**

A typical approach is Terraform or CloudFormation/CDK, executed through a controlled pipeline.

With Terraform:

```bash
terraform fmt -check
terraform init
terraform validate
terraform plan -out=tfplan

# Apply the reviewed plan.
terraform apply tfplan
```

Use:

- Reusable modules for networking, compute, databases, and IAM.
- Separate environment configuration and appropriately isolated state.
- Remote state with restricted access and locking.
- Pull-request review and production approval.
- Short-lived pipeline credentials.
- Scheduled drift detection.

For an S3 backend, current Terraform supports native locking through `use_lockfile = true`; DynamoDB-based locking is deprecated. :chatgpt-content-reference{index="20"}

Explain which resources you provisioned personally and how you handled changes, rollback limitations, and drift.

---

**2. How do you provision container-based services for microservices?**

For ECS, provision:

- ECR repositories.
- An ECS cluster.
- Fargate configuration or EC2 capacity.
- A task definition per workload.
- ECS services for long-running workloads.
- Load-balancer listeners and target groups.
- IAM task and execution roles.
- Security groups, logs, secrets, and scaling policies.

For EKS, provision:

- An EKS cluster and suitable compute.
- Networking and required add-ons.
- Workload IAM access.
- Registry access.
- Kubernetes Deployments, Services, configuration, and scaling resources.

ECS uses task definitions and services; EKS exposes Kubernetes APIs and workload resources. :chatgpt-content-reference{index="21"}

Build the image once and promote its digest. Keep application deployment changes separate from unrelated foundational infrastructure changes when that improves review and recovery.

---

**3. What database services did you provision with the application?**

Answer based on actual requirements. A sample architecture could include:

| Requirement | Service |
|---|---|
| Orders, payments, relational constraints | RDS PostgreSQL/MySQL or Aurora |
| Key-based access at large scale | DynamoDB |
| Caching or short-lived derived data | ElastiCache |
| Object/file storage | S3 |

RDS manages common relational database administration capabilities; ElastiCache provides managed in-memory data-store options. :chatgpt-content-reference{index="22"}

Explain the associated configuration:

- Private connectivity and restricted access.
- Backups and point-in-time recovery.
- Encryption and credential rotation.
- Monitoring and connection limits.
- Availability configuration.
- Schema migration tooling.

Distinguish **availability from read scaling**. A traditional RDS Multi-AZ DB-instance standby is not a read replica. RDS Multi-AZ DB clusters have different reader capabilities, so name the deployment type precisely. :chatgpt-content-reference{index="23"}

---

**4. What is the primary difference between ECS and EKS?**

| Aspect | ECS | EKS |
|---|---|---|
| Orchestration | AWS-native container orchestration | Managed Kubernetes |
| Workload definitions | Task definitions and services | Pods, Deployments, Services, and other Kubernetes objects |
| Ecosystem | AWS-focused integration | Kubernetes tools and APIs |
| Operational knowledge | ECS concepts | Kubernetes plus AWS integration |
| Compute | EC2 or Fargate | Supported EC2-based options and Fargate |

The main distinction is the **orchestration model**, not whether containers or serverless compute are supported.

Choose ECS when its AWS-native capabilities meet the requirements with less platform complexity. Choose EKS when Kubernetes APIs, ecosystem, or organizational standards are required. :chatgpt-content-reference{index="24"}

Fargate is a compute option; it is not a replacement name for either orchestrator.

---

**5. Where do you define autoscaling parameters?**

It depends on what is being scaled:

| Target | Configuration location |
|---|---|
| ECS task count | Application Auto Scaling target and policy |
| EC2 fleet | Auto Scaling Group and scaling policies |
| Kubernetes replicas | HPA resource |
| Kubernetes node capacity | The chosen node-autoscaling system |
| Lambda concurrency | Function concurrency and event-source settings; provisioned-concurrency scaling where used |

For ECS, define minimum/maximum capacity and policies such as target tracking using CPU, memory, or a suitable workload metric. :chatgpt-content-reference{index="25"}

Store these settings in infrastructure or deployment code.

Also define:

- Scaling responsiveness and stabilization.
- Safe downstream limits.
- Queue backlog or request-based metrics where CPU is misleading.
- Capacity required during deployments.

Increasing application replicas does not automatically increase database capacity.

---

**6. What do you know about serverless architecture?**

Serverless architecture combines managed execution and managed services, often around events.

Key design characteristics:

- Stateless or externally persisted execution state.
- Automatic capacity management within service limits.
- Event triggers and asynchronous processing.
- Fine-grained permissions.
- Explicit retry and duplicate-handling behavior.
- Managed operational interfaces.

For example, API Gateway can invoke Lambda, which writes data and publishes work to a queue. A separate consumer processes that work.

Trade-offs include cold starts, concurrency limits, execution constraints, distributed debugging, and workload-dependent cost. Serverless does not eliminate architecture, security, or operational responsibility. :chatgpt-content-reference{index="26"}

---

**7. How do you optimize Lambda cold starts?**

First measure initialization latency and its contribution to user-visible latency.

Then consider:

1. **Reduce initialization work:** Avoid loading unnecessary frameworks and dependencies.
2. **Reuse suitable clients:** Initialize reusable SDK clients outside the handler.
3. **Tune memory:** More memory can provide additional CPU; benchmark latency and cost.
4. **Use provisioned concurrency:** Maintain initialized environments for latency-sensitive traffic.
5. **Use SnapStart where supported:** Restore initialized state from snapshots.
6. **Review initialization correctness:** Refresh stale connections, credentials, or other snapshot-sensitive state.

AWS recommends execution-environment reuse and appropriate memory tuning. :chatgpt-content-reference{index="27"}

Important distinctions:

- **Reserved concurrency does not pre-warm environments.**
- Provisioned concurrency has additional cost and must be applied to the version/alias receiving traffic.
- Traffic exceeding provisioned capacity can still use on-demand environments. :chatgpt-content-reference{index="28"}
- SnapStart operates on published versions and has runtime/configuration restrictions. It cannot be combined with provisioned concurrency on the same function configuration. :chatgpt-content-reference{index="29"}

Periodic “keep warm” requests do not guarantee sufficient warm capacity during a burst.

---

**8. What is serverless deployment in AWS?**

It means deploying the application code **and its managed infrastructure configuration**, without provisioning application servers.

Using AWS SAM:

1. Define functions, APIs, events, permissions, and configuration.
2. Build the artifacts.
3. Test.
4. Package and deploy through CloudFormation.
5. Validate and monitor the release.

Typical commands:

```bash
sam validate
sam build
sam deploy --guided
```

For CI/CD, use saved configuration and an appropriately scoped deployment role instead of interactive configuration.

SAM provides an infrastructure template model and tools for building and deploying serverless applications. :chatgpt-content-reference{index="30"}

Lambda artifacts can be deployment archives or supported container images. Using a container image for Lambda still follows Lambda’s execution model; it does not create an ECS service.

---

**9. What is HPA?**

The **Horizontal Pod Autoscaler** changes a workload’s replica count based on observed metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Here, CPU utilization is measured relative to configured CPU requests. For a container requesting `500m`, 70% corresponds to approximately `350m` CPU usage.

HPA needs the relevant metrics provider. Resource-based scaling commonly uses Metrics Server; custom/external metrics require suitable adapters. :chatgpt-content-reference{index="31"}

HPA scales replicas, not per-Pod CPU or node count. If the cluster has insufficient capacity, additional Pods can remain pending until node capacity becomes available.

---

**10. What are Regions and Availability Zones?**

- An **AWS Region** is a geographic area containing multiple Availability Zones.
- An **Availability Zone** consists of one or more discrete data centers with independent infrastructure and redundant connectivity.

Availability Zones within a Region are connected through low-latency networking. :chatgpt-content-reference{index="32"}

Use multiple AZs to tolerate an AZ-level failure. Use multiple Regions when requirements justify geographic disaster recovery, global access, or other regional separation.

A multi-AZ deployment does not automatically protect against every Region-wide failure.

---

**11. How do you monitor microservice health?**

Monitor both technical behavior and business outcomes.

| Area | Examples |
|---|---|
| Traffic | Requests per second, queue arrival rate |
| Errors | HTTP failures, failed messages, exceptions |
| Latency | p50/p95/p99 response time |
| Saturation | CPU, heap, threads, connection-pool waits |
| Dependencies | Database latency, broker lag, downstream failures |
| Business | Payment success, completed orders, stuck workflows |

A typical stack combines:

- Actuator and Micrometer for application telemetry.
- Prometheus or CloudWatch for metrics.
- Grafana or CloudWatch dashboards.
- Centralized logs.
- Distributed tracing with propagated trace context.

Spring Boot integrates with Micrometer and supported monitoring registries. :chatgpt-content-reference{index="33"}

Separate liveness from readiness. A temporary database outage should not automatically cause every application instance to restart.

---

**12. How do you enable Spring Boot Actuator?**

Add:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Configure selected endpoints:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: never
      probes:
        enabled: true
```

For Prometheus output, also add `micrometer-registry-prometheus`.

Endpoints include:

```text
/actuator/health
/actuator/health/liveness
/actuator/health/readiness
/actuator/prometheus
```

Expose only what is needed and restrict operational access. :chatgpt-content-reference{index="34"}

---

**13. Are Actuator endpoints accessible without authentication?**

**It depends on exposure and security configuration.**

- By default, only health is exposed over HTTP.
- With Spring Security present and no custom security filter chain, Boot secures Actuator endpoints other than health.
- Defining a custom `SecurityFilterChain` makes you responsible for the applicable access rules.
- Without an authentication configuration, exposing an endpoint does not automatically secure it. :chatgpt-content-reference{index="35"}

A production approach is to allow minimal health status where required, authenticate operational endpoints, and restrict their network reachability.

Do not expose detailed health information, environment data, or heap dumps publicly.

---

**14. Have you worked with Kafka or another messaging service?**

Use your actual experience. An example answer could cover:

> “We used messaging to decouple order processing from notifications and fulfillment. Producers published events, and independently deployed consumers processed them. We monitored lag, handled poison messages, and made consumers idempotent.”

For Kafka, be prepared to explain:

- Topics and partitions.
- Message keys and ordering within a partition.
- Consumer groups and partition assignment.
- Offset management and rebalancing.
- Retry topics or dead-letter handling.
- Serialization and schema evolution.
- Producer and consumer security.

Spring provides Kafka integration, including producer templates and listener support. :chatgpt-content-reference{index="36"}

Avoid claiming end-to-end exactly-once behavior merely because Kafka transactions are enabled. External databases, payment providers, and other side effects require their own consistency strategy.

---

**15. What serialization/deserialization frameworks have you used?**

| Format/tool | Typical purpose |
|---|---|
| Jackson JSON | REST request/response mapping |
| Apache Avro | Schema-based messaging with evolution support |
| Protocol Buffers | Compact contracts, commonly with gRPC |
| Kafka serializers/deserializers | Convert application values to/from Kafka message bytes |

Serialization converts application data into a transferable representation. Deserialization reconstructs usable data from it.

Discuss:

- Backward/forward compatibility.
- Required fields and defaults.
- Unknown fields.
- Date/time and numeric precision.
- Payload size and performance.

For money, use an agreed decimal representation and currency information; avoid silently converting monetary amounts to binary floating-point.

---

**16. Where do you register an Avro schema?**

Typically in a **schema registry**, such as Confluent Schema Registry or AWS Glue Schema Registry.

A common process is:

1. Store the `.avsc` definition in version control.
2. Review schema changes.
3. Run compatibility checks in CI.
4. Register the approved schema.
5. Configure producers and consumers to use the registry.

In Confluent’s default topic-name strategy, a topic such as `orders` commonly uses subjects:

```text
orders-key
orders-value
```

A subject holds schema versions and compatibility rules. Messages serialized using the corresponding Confluent format identify the schema so consumers can retrieve the required definition. :chatgpt-content-reference{index="37"}

AWS Glue Schema Registry is an AWS-managed alternative, with its own integration and wire-format details. :chatgpt-content-reference{index="38"}

Do not confuse storing the schema file in Git with registering it for runtime serialization.

---

**17. Have you implemented concurrency APIs in Java?**

Examples include:

- `ExecutorService` for task execution.
- `CompletableFuture` for asynchronous composition.
- `ConcurrentHashMap` for concurrent map access.
- `AtomicInteger` and related classes for atomic operations.
- `Semaphore` for concurrency limits.
- Locks and synchronization for protected shared state.
- Java 21 virtual threads for suitable blocking I/O workloads.

A practical example is calling independent downstream services concurrently to reduce aggregate response time.

Important considerations:

- Bound concurrency to protect downstream services.
- Set deadlines and handle cancellation.
- Avoid blocking the common fork-join pool with uncontrolled I/O.
- Shut down application-owned executors appropriately.
- Propagate required tracing/security context.

CompletableFuture asynchronous methods use their configured executor, or commonly the default asynchronous execution facility when one is not supplied. :chatgpt-content-reference{index="39"}

A Spring thread-bound transaction does not automatically follow work into another thread.

---

**18. How do you combine responses from three APIs?**

If the calls are independent, run them concurrently and combine their results.

Assume the clients and an application-managed bounded executor are injected:

```java
public record Dashboard(
    Profile profile,
    List<Order> orders,
    Offers offers
) {}

public CompletableFuture<Dashboard> loadDashboard(String userId) {

    CompletableFuture<Profile> profile =
        CompletableFuture.supplyAsync(
            () -> profileClient.getProfile(userId),
            apiExecutor
        );

    CompletableFuture<List<Order>> orders =
        CompletableFuture.supplyAsync(
            () -> orderClient.getOrders(userId),
            apiExecutor
        );

    CompletableFuture<Offers> offers =
        CompletableFuture.supplyAsync(
            () -> offerClient.getOffers(userId),
            apiExecutor
        );

    return CompletableFuture.allOf(profile, orders, offers)
        .orTimeout(2, TimeUnit.SECONDS)
        .thenApply(ignored -> new Dashboard(
            profile.join(),
            orders.join(),
            offers.join()
        ));
}
```

`allOf` coordinates completion; it does not directly return the individual results. The `join()` calls inside `thenApply` read already-completed results after successful completion. :chatgpt-content-reference{index="40"}

Decide the failure contract:

- Fail the whole operation if every response is required.
- Return explicitly marked partial results if the business allows it.
- Use fallbacks only where they remain meaningful.

`orTimeout` does not necessarily terminate the underlying network calls. Configure HTTP-client timeouts and resource cleanup too.

---

**19. What is active-active?**

In active-active architecture, multiple deployments serve production traffic simultaneously.

For example:

- Region A serves some users.
- Region B serves others.
- Routing redistributes traffic when one deployment becomes unavailable.

Benefits include capacity utilization, geographic proximity, and reduced dependence on a single active site.

Challenges include:

- Cross-region data consistency.
- Write conflicts and duplicate processing.
- Routing and failure detection.
- Capacity to absorb a failed site’s traffic.
- Operational complexity.

Active-active application servers do not automatically imply that the database supports writes in every Region. Define the architecture separately for each layer. :chatgpt-content-reference{index="41"}

---

**20. How does active-passive differ from active-active?**

| Aspect | Active-active | Active-passive |
|---|---|---|
| Normal traffic | Multiple sites serve traffic | Primary serves traffic |
| Secondary environment | Already active | Standby, with varying readiness |
| Failover | Redistribute traffic and handle data routing | Promote/activate standby and redirect traffic |
| Data concerns | Concurrent access and possibly write conflicts | Replication lag, promotion, and split-brain prevention |
| Cost | Capacity active in multiple sites | Depends on cold, warm, or hot standby design |

Choose based on business recovery objectives:

- **RTO:** How long the service can take to recover.
- **RPO:** How much data loss is acceptable.

Neither pattern guarantees zero downtime or zero data loss. Test promotion, traffic redirection, failback, and database recovery. :chatgpt-content-reference{index="42"}

---

**21. Find all array pairs that sum to a target.**

Clarify whether the interviewer wants **unique value pairs** or **every matching index pair**.

This Java solution returns unique unordered value pairs and handles integer-overflow boundaries:

```java
import java.util.*;

public class PairFinder {

    public record Pair(int first, int second) {}

    public static List<Pair> findPairs(int[] numbers, int target) {
        Set<Integer> seen = new HashSet<>();
        Set<Pair> pairs = new LinkedHashSet<>();

        for (int number : numbers) {
            long complement = (long) target - number;

            if (complement >= Integer.MIN_VALUE
                    && complement <= Integer.MAX_VALUE) {

                int other = (int) complement;

                if (seen.contains(other)) {
                    pairs.add(new Pair(
                        Math.min(number, other),
                        Math.max(number, other)
                    ));
                }
            }

            seen.add(number);
        }

        return new ArrayList<>(pairs);
    }

    public static void main(String[] args) {
        int[] numbers = {1, 5, 7, -1, 5, 3, 3};

        System.out.println(findPairs(numbers, 6));
    }
}
```

Output:

```text
[Pair[first=1, second=5],
 Pair[first=-1, second=7],
 Pair[first=3, second=3]]
```

The lookup occurs before adding the current element, so one array position cannot pair with itself.

Average complexity is **O(n) time** and **O(n) auxiliary space**.

For every matching index pair, store previously seen indices per value. Output can be quadratic when many duplicates exist.

---

**Manager round**

**1. Can you share your previous experience?**

Give a concise progression rather than repeating your entire résumé.

Template:

> “I have [X] years of experience, including [Y] years focused on [relevant area]. In my previous role, I worked on [business domain] and owned [specific responsibilities]. I then moved into [expanded responsibility], where I worked on [relevant delivery or reliability work]. My strongest areas are [two or three strengths], supported by [a concrete example].”

Include your actual contribution, team interactions, and one measurable result if you have reliable numbers.

---

**2. Have you recently worked on a challenging task?**

Use **Situation, Task, Action, Result**, followed by what you learned.

A suitable example—if it reflects your experience—is diagnosing intermittent timeouts:

- **Situation:** Latency increased during peak traffic.
- **Task:** Restore service and identify the cause.
- **Action:** Correlate traces, pool metrics, database waits, and recent changes; apply a controlled mitigation; implement a lasting fix.
- **Result:** Describe the measured recovery and subsequent validation.
- **Learning:** Add prevention through capacity testing, limits, alerts, or design changes.

Be precise about what you did personally. Explain the evidence behind your decision and one alternative you considered.

---

**3. How do you handle pressure and multiple critical deadlines?**

A strong answer:

> “I first prioritize by customer impact, urgency, and dependencies. During an incident, I establish ownership and a communication rhythm, stabilize the service, and separate immediate mitigation from root-cause work. For competing delivery deadlines, I break work into milestones, identify what can run in parallel, and raise trade-offs early.
>
> If all deadlines cannot realistically be met, I propose a clear choice—scope reduction, sequencing, or additional help—rather than silently accepting an unachievable plan.”

Support this with an actual example showing how you communicated and protected delivery quality.

---

**4. What if a lead or teammate is technically weak or behaves poorly?**

Address the specific problem without labeling the person.

> “If there is a technical gap, I clarify expectations and offer pairing, examples, documentation, or review support. I use evidence and tests to discuss technical disagreements.
>
> If the issue is behavior, I raise specific examples privately and explain their impact. I agree on a constructive way of working. If the behavior continues or creates a serious delivery or workplace concern, I escalate through the appropriate manager with factual examples.”

Show that you can maintain respectful collaboration while setting boundaries and protecting team outcomes.

---

**5. What do you do in your free time or on weekends?**

Answer honestly and naturally. The interviewer is often assessing communication and personal interests, not expecting every weekend to involve technical study.

A template:

> “Outside work, I enjoy [genuine hobby or activity]. I also spend time on [another real interest]. When I explore technology, I usually choose a small topic or project that interests me, such as [something you actually did].”

Be ready for a follow-up about whichever activity you mention.
