## 1. What are stages in Docker images?

### Meaning of Stages in Docker

In Docker, **stages** usually refer to **multi-stage builds**, where a single Dockerfile contains multiple `FROM` statements. Each `FROM` starts a new build stage.

Stages help separate:

1. **Build Stage** – compile code, install dependencies, run tests
2. **Runtime Stage** – copy only required output into lightweight final image

This reduces image size, improves security, and speeds deployments.

---

### Why Use Multi-Stage Builds?

Without multi-stage builds:

* Build tools remain in final image
* Larger image size
* More vulnerabilities
* Slower pull/push times

With multi-stage builds:

* Final image contains only runtime files
* Smaller and cleaner images
* Better for production

---

### Example Flow

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

---

### Common Stage Names

| Stage Name | Purpose           |
| ---------- | ----------------- |
| builder    | Compile/build app |
| test       | Run unit tests    |
| production | Final runtime     |
| debug      | Debug tools       |

---

### Interview Answer

> Docker stages are separate build phases defined using multiple FROM instructions in a Dockerfile. They are mainly used in multi-stage builds where one stage builds the application and another stage creates the final lightweight runtime image. This reduces image size and improves security.

---

---

## 2. Write a Dockerfile for a Node.js application with multi-stage builds

### Example Node.js Multi-Stage Dockerfile

```dockerfile
# Stage 1 - Build Stage
FROM node:20 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

RUN npm run build

# Stage 2 - Production Stage
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY --from=builder /app/dist ./dist

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

---

### Explanation

### Stage 1: Builder

* Uses full Node image
* Installs all packages
* Builds app

### Stage 2: Production

* Uses smaller Alpine image
* Installs only production dependencies
* Copies built files from builder stage

---

### Benefits

* Smaller image
* Faster deployments
* No source code leaks
* No dev dependencies in production

---

### Build Command

```bash
docker build -t my-node-app .
```

### Run Command

```bash
docker run -p 3000:3000 my-node-app
```

---

---

## 3. ENTRYPOINT vs CMD in Docker

### CMD

CMD provides **default command** to run container.

Example:

```dockerfile
CMD ["node", "app.js"]
```

If user provides another command during `docker run`, CMD gets replaced.

---

### ENTRYPOINT

ENTRYPOINT makes container behave like executable.

Example:

```dockerfile
ENTRYPOINT ["node"]
CMD ["app.js"]
```

This runs:

```bash
node app.js
```

If user runs:

```bash
docker run image server.js
```

Then output becomes:

```bash
node server.js
```

---

### Difference Table

| Feature           | CMD                 | ENTRYPOINT       |
| ----------------- | ------------------- | ---------------- |
| Purpose           | Default arguments   | Main executable  |
| Can be overridden | Yes                 | Harder           |
| Best for          | Flexible containers | Fixed executable |

---

### Best Practice

Use both together:

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

---

### Interview Answer

> CMD defines default command arguments, while ENTRYPOINT defines the main executable of container. CMD can be overridden at runtime, but ENTRYPOINT is usually fixed.

---

---

## 4. Which registry is used for storing Docker images?

Docker images are stored in **Container Registries**.

### Popular Registries

| Registry                                | Description                          |
| --------------------------------------- | ------------------------------------ |
| Docker Hub                              | Public/private Docker image registry |
| Amazon Elastic Container Registry (ECR) | AWS registry                         |
| Azure Container Registry (ACR)          | Azure registry                       |
| Google Artifact Registry                | GCP registry                         |
| GitHub Container Registry               | GitHub images                        |
| Harbor                                  | Enterprise private registry          |

---

### Most Common

### Public:

* Docker Hub

### Enterprise / Cloud:

* ACR
* ECR
* Harbor
* GitHub Registry

---

### Example Push to Docker Hub

```bash
docker login
docker tag myapp username/myapp:v1
docker push username/myapp:v1
```

---

### Interview Answer

> Docker images are stored in image registries such as Docker Hub, Azure Container Registry, AWS ECR, Google Artifact Registry, GitHub Container Registry, or private registries like Harbor.

---

---

## 5. How do you pass environment variables on docker build command?

During build time, variables are passed using **ARG**.

---

## Dockerfile Example

```dockerfile
FROM node:20

ARG APP_ENV=dev

ENV NODE_ENV=$APP_ENV

RUN echo $APP_ENV
```

---

## Build Command

```bash
docker build --build-arg APP_ENV=production -t myapp .
```

---

### Explanation

* `ARG` = available during build only
* `ENV` = persists inside container runtime

---

### Runtime Variable Example

```bash
docker run -e DB_HOST=mysql -e PORT=3000 myapp
```

---

### Difference Table

| Variable Type | Used In    | Command                  |
| ------------- | ---------- | ------------------------ |
| ARG           | Build time | docker build --build-arg |
| ENV           | Runtime    | docker run -e            |

---

### Best Practice

Never pass secrets using build args.

Use:

* Docker secrets
* Kubernetes secrets
* Vault

---

### Interview Answer

> Environment variables during Docker build are passed using `--build-arg` and declared in Dockerfile using `ARG`. Runtime variables are passed using `docker run -e`.

Example:

```bash
docker build --build-arg VERSION=1.0 .
```

## 6. What services do you use for storing Docker images?

Docker images are stored in **container registries**. These can be public cloud-managed services or private self-hosted registries.

---

## Common Services for Storing Images

| Service                                 | Use Case                                              |
| --------------------------------------- | ----------------------------------------------------- |
| Docker Hub                              | Public/private image hosting, common default registry |
| Azure Container Registry (ACR)          | Best for Azure + AKS environments                     |
| Amazon Elastic Container Registry (ECR) | Best for ECS/EKS on AWS                               |
| Google Artifact Registry                | GCP workloads                                         |
| GitHub Container Registry               | CI/CD integrated with GitHub Actions                  |
| Harbor                                  | On-prem private registry with scanning & RBAC         |
| JFrog Artifactory                       | Enterprise artifact management                        |

---

## Real-World Usage

### Azure DevOps Project

* Build image in pipeline
* Push to **ACR**
* Deploy to **AKS**

### On-Prem Enterprise

* Use **Harbor** for private registry
* Integrated with LDAP/AD

---

## Interview Answer

> Docker images are commonly stored in registries such as Docker Hub, Azure Container Registry, AWS ECR, Google Artifact Registry, GitHub Container Registry, Harbor, or Artifactory depending on cloud strategy, security, and enterprise needs.

---

# 7. How do you establish the connection?

This usually means **how Docker client / CI pipeline / Kubernetes cluster connects to container registry**.

---

# A. Docker CLI to Registry

Use login:

```bash
docker login myregistry.azurecr.io
```

Provide:

* Username / Password
* Token
* Service principal
* Access key

---

# B. Azure ACR Connection

```bash
az acr login --name myacr
```

Then push:

```bash
docker push myacr.azurecr.io/app:v1
```

---

# C. Kubernetes to Registry

Use imagePullSecret:

```bash
kubectl create secret docker-registry regcred \
--docker-server=myregistry.io \
--docker-username=user \
--docker-password=pass
```

Then in deployment:

```yaml
imagePullSecrets:
- name: regcred
```

---

# D. CI/CD Pipeline Connection

Use:

* Service Connection in Azure DevOps
* IAM Role for AWS ECR
* GitHub OIDC auth
* Secrets manager credentials

---

## Interview Answer

> Registry connection is established using authentication such as docker login, cloud CLI login, service principals, IAM roles, tokens, or Kubernetes imagePullSecrets depending on environment.

---

# 8. Write a Docker Compose file for a multi-container setup

Example: Node.js App + MySQL Database

```yaml
version: '3.8'

services:
  app:
    image: mynodeapp:latest
    container_name: app
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: password
      DB_NAME: appdb

  db:
    image: mysql:8
    container_name: mysql-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: appdb
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

---

## Run Compose

```bash
docker compose up -d
```

---

## Explanation

### app service

* Runs Node.js application
* Connects to DB using service name `db`

### db service

* Runs MySQL
* Stores data in volume

---

## Interview Answer

> Docker Compose is used to define multi-container applications using YAML. It helps manage app, database, cache, networking, and volumes in one command.

---

# 9. How do you scan the images at the registry level?

Image scanning checks for:

* OS vulnerabilities
* Package CVEs
* Secrets
* Malware
* Misconfigurations

---

# Registry-Level Scanning Services

| Registry                                | Native Scanning                          |
| --------------------------------------- | ---------------------------------------- |
| Azure Container Registry (ACR)          | Microsoft Defender for Cloud integration |
| Amazon Elastic Container Registry (ECR) | Inspector scanning                       |
| Harbor                                  | Trivy / Clair                            |
| Docker Hub                              | Vulnerability scan tiers                 |

---

# Example – ACR Scan

Push image to ACR, then:

* Defender scans automatically
* Detects CVEs
* Severity levels
* Recommendations

---

# CLI Tools Used

| Tool  | Purpose                  |
| ----- | ------------------------ |
| Trivy | Fast open-source scanner |
| Clair | Registry scanning        |
| Snyk  | App + container scan     |

---

## Best Practice

Block deployment if:

* Critical CVEs found
* Secrets detected
* Unsigned image
* Non-compliant base image

---

## Interview Answer

> Images are scanned at registry level using native registry security tools such as ACR + Defender, AWS ECR + Inspector, Harbor + Trivy/Clair. CI/CD pipelines often enforce security gates before deployment.

---

# 10. What are the stages in a Docker image build? Why do we use ENTRYPOINT and CMD?

---

# Docker Image Build Stages

In multi-stage Docker build, each `FROM` starts a stage.

### Common Stages

| Stage   | Purpose                    |
| ------- | -------------------------- |
| Build   | Compile source code        |
| Test    | Run unit/integration tests |
| Package | Create binaries/artifacts  |
| Runtime | Final lightweight image    |

---

## Example

```dockerfile
# Stage 1
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2
FROM node:20-alpine
COPY --from=build /app/dist /app
CMD ["node","/app/index.js"]
```

---

# Why Use Multi-Stage Builds?

* Smaller final image
* Remove compilers/tools
* Faster pull time
* Better security

---

# ENTRYPOINT vs CMD

---

## CMD

Provides default command.

```dockerfile
CMD ["node","app.js"]
```

Can be overridden.

---

## ENTRYPOINT

Defines fixed executable.

```dockerfile
ENTRYPOINT ["node"]
CMD ["app.js"]
```

This becomes:

```bash
node app.js
```

If user runs:

```bash
docker run image server.js
```

Then:

```bash
node server.js
```

---

# Why Use Both?

| Instruction | Purpose            |
| ----------- | ------------------ |
| ENTRYPOINT  | Main executable    |
| CMD         | Default parameters |

---

# Best Example

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

---

## Interview Answer

> Docker image builds commonly use multi-stage builds with build, test, and runtime stages. ENTRYPOINT defines the main executable for the container, while CMD provides default arguments or command values. Together they create flexible and reusable containers.

## 11. Which container registry do you use for storing Docker images?

In real-world environments, the container registry depends on the cloud platform, security requirements, and CI/CD ecosystem.

---

## Registries I Commonly Use

| Registry                                | Best Use Case                        |
| --------------------------------------- | ------------------------------------ |
| Azure Container Registry (ACR)          | Azure workloads, AKS, Azure DevOps   |
| Amazon Elastic Container Registry (ECR) | AWS ECS/EKS workloads                |
| Docker Hub                              | Public/open-source images            |
| GitHub Container Registry               | GitHub Actions integration           |
| Harbor                                  | On-prem private registry             |
| JFrog Artifactory                       | Large enterprise artifact governance |

---

## My Preferred Choice (Based on Your Azure Background)

Since many enterprises use Azure:

> I typically use **Azure Container Registry (ACR)** because it integrates well with Azure Kubernetes Service (AKS), Azure DevOps, RBAC, managed identities, private endpoints, and Defender scanning.

---

## Interview Answer

> I have worked with Azure Container Registry, Docker Hub, and private registries like Harbor. For enterprise Azure environments, I prefer ACR because of seamless AKS integration, private networking, RBAC, and built-in security controls.

---

# 12. How do you pass environment variables during Docker build commands? What services do you use for storing Docker images?

# Part A – Passing Environment Variables During Build

Use **ARG** in Dockerfile and `--build-arg` during build.

---

## Dockerfile Example

```dockerfile id="z0q0zw"
FROM node:20

ARG APP_ENV=dev
ARG VERSION=1.0

ENV NODE_ENV=$APP_ENV

RUN echo "Environment: $APP_ENV"
```

---

## Build Command

```bash id="zkuboi"
docker build \
--build-arg APP_ENV=production \
--build-arg VERSION=2.0 \
-t myapp:v2 .
```

---

## Difference Between ARG and ENV

| Type | Scope                          |
| ---- | ------------------------------ |
| ARG  | Build time only                |
| ENV  | Available in running container |

---

# Part B – Services for Storing Images

* Azure Container Registry (ACR)
* Amazon Elastic Container Registry (ECR)
* Docker Hub
* Harbor
* GitHub Container Registry

---

## Interview Answer

> Build-time variables are passed using `--build-arg` and declared with ARG in Dockerfile. For storing images, I use ACR, ECR, Docker Hub, or Harbor depending on cloud and security needs.

---

# 13. Are you aware of security scanning tools? How do you scan Docker images—both during build and at the registry level? Are you using any extensions or tools for image scanning?

## Yes — Container Security Scanning is Critical

Scanning is done in two phases:

1. **During Build Pipeline**
2. **At Registry Level**

---

# A. During Build Pipeline

CI/CD scans image before push.

### Popular Tools

| Tool         | Purpose                     |
| ------------ | --------------------------- |
| Trivy        | OS packages + language deps |
| Snyk         | App + image scan            |
| Anchore      | Policy enforcement          |
| Docker Scout | Supply chain + CVEs         |

---

## Example – Trivy in Pipeline

```bash id="w52fws"
trivy image myapp:v1
```

Fail build if Critical vulnerabilities found.

---

# B. Registry Level Scanning

After image push:

| Registry                                | Security Scan                |
| --------------------------------------- | ---------------------------- |
| Azure Container Registry (ACR)          | Microsoft Defender for Cloud |
| Amazon Elastic Container Registry (ECR) | Inspector                    |
| Harbor                                  | Trivy / Clair                |

---

# What I Check

* Critical CVEs
* Outdated base images
* Exposed secrets
* Malware
* License risks
* Root user containers

---

# Extensions / Integrations

* Jenkins plugin for Trivy
* Azure DevOps extension for Snyk
* GitHub Actions Trivy scan
* Defender for Cloud alerts

---

## Interview Answer

> Yes, I use tools like Trivy, Snyk, Docker Scout, and Defender for Cloud. I scan images in CI/CD before pushing and also enable registry-level continuous scanning in ACR/ECR/Harbor.

---

# 14. What is multi-stage Docker build?

A **multi-stage build** uses multiple `FROM` statements in one Dockerfile.

Each `FROM` creates a separate stage.

Used to keep final image small and secure.

---

## Example

```dockerfile id="ep9n8s"
# Build stage
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Runtime stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

---

## Benefits

* Smaller images
* No build tools in final image
* Faster deployments
* Better security
* Lower storage cost

---

## Typical Stages

| Stage   | Purpose          |
| ------- | ---------------- |
| builder | Compile app      |
| test    | Run tests        |
| package | Prepare artifact |
| runtime | Production image |

---

## Interview Answer

> Multi-stage build allows us to use one stage for building code and another lightweight stage for runtime. It removes unnecessary build dependencies from final image.

---

# 15. What are manifest files?

In Docker/container registries, a **manifest** is a JSON document that describes an image.

It tells Docker:

* Which layers belong to image
* Tags
* OS / architecture
* Digest
* Config file
* Multi-platform references

---

# Example Concept

When you run:

```bash id="zlr8ww"
docker pull nginx:latest
```

Docker first pulls the **manifest**, then downloads required layers.

---

# Types of Manifests

| Type                      | Purpose                       |
| ------------------------- | ----------------------------- |
| Image Manifest            | Single image metadata         |
| Manifest List / OCI Index | Multi-architecture image list |

---

# Multi-Arch Example

One tag can support:

* linux/amd64
* linux/arm64
* windows/amd64

Docker chooses matching platform automatically.

---

# Useful Commands

```bash id="4k0iq0"
docker manifest inspect nginx:latest
```

---

# Why Important?

* Cross-platform support
* Layer tracking
* Faster pulls
* Image integrity using digest

---

## Interview Answer

> Manifest files are metadata documents stored in registries that define image layers, config, digests, OS, and architecture. Docker reads the manifest first before downloading image layers. They are essential for multi-architecture images.
