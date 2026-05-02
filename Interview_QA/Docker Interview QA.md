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


## 16. What is the difference between Virtual Machines and Containers?

Virtual Machines and Containers are both used to run applications in isolated environments, but they work differently.

---

# Architecture Difference

## Virtual Machine

A VM virtualizes **hardware**. Each VM contains:

* Full Guest OS
* Application
* Libraries
* Virtual hardware

Runs on a hypervisor like VMware, Microsoft Hyper-V, KVM.

---

## Container

A container virtualizes the **OS layer**.

Contains:

* Application
* Dependencies
* Runtime libraries

Shares host OS kernel.

Common runtime: Docker, containerd.

---

# Comparison Table

| Feature     | Virtual Machine | Container     |
| ----------- | --------------- | ------------- |
| Boot Time   | Minutes         | Seconds       |
| Size        | GBs             | MBs           |
| OS Included | Yes             | No            |
| Performance | Heavier         | Lightweight   |
| Density     | Fewer per host  | Many per host |
| Isolation   | Stronger        | Good          |
| Portability | Moderate        | Excellent     |

---

# Real Example

If you deploy 10 Java apps:

* VM model = 10 OS installations
* Container model = 10 containers sharing host kernel

Much more efficient.

---

# Interview Answer

> Virtual machines virtualize hardware and run separate guest operating systems, while containers virtualize the OS layer and share the host kernel. Containers are lighter, faster, and more portable, whereas VMs provide stronger isolation.

---

# 17. Explain the Docker lifecycle

Docker lifecycle means the stages a container/image goes through from creation to deletion.

---

# Main Lifecycle Flow

```text
Build Image → Store Image → Pull Image → Run Container → Stop → Restart → Remove
```

---

# Detailed Stages

## 1. Write Dockerfile

Define how image should be built.

```dockerfile id="u1y5cn"
FROM node:20
COPY . .
CMD ["node","app.js"]
```

---

## 2. Build Image

```bash id="8fz4n3"
docker build -t myapp:v1 .
```

Creates reusable image.

---

## 3. Push to Registry

Push to Docker Hub or Azure Container Registry (ACR).

```bash id="qynk6o"
docker push myrepo/myapp:v1
```

---

## 4. Pull Image

```bash id="ehg5ih"
docker pull myrepo/myapp:v1
```

---

## 5. Run Container

```bash id="g53i2r"
docker run -d -p 3000:3000 myrepo/myapp:v1
```

---

## 6. Manage Running Container

```bash id="4w7iyv"
docker ps
docker logs app
docker exec -it app sh
```

---

## 7. Stop / Start

```bash id="0ukj1r"
docker stop app
docker start app
docker restart app
```

---

## 8. Remove

```bash id="7f6i9j"
docker rm app
docker rmi myrepo/myapp:v1
```

---

# Container States

| State   | Meaning               |
| ------- | --------------------- |
| Created | Initialized           |
| Running | Active                |
| Paused  | Temporarily suspended |
| Exited  | Stopped               |
| Dead    | Failed state          |

---

# Interview Answer

> Docker lifecycle includes writing Dockerfile, building image, storing in registry, running container, monitoring it, stopping/restarting, and finally removing unused containers and images.

---

# 18. Write some Docker commands

Here are common interview commands.

---

# Image Commands

```bash id="kv3j8w"
docker build -t myapp .
docker images
docker pull nginx
docker push repo/app:v1
docker rmi myapp
```

---

# Container Commands

```bash id="g8g1cn"
docker run -d -p 80:80 nginx
docker ps
docker ps -a
docker stop container_id
docker start container_id
docker restart container_id
docker rm container_id
```

---

# Troubleshooting Commands

```bash id="d6d4i2"
docker logs container_id
docker exec -it container_id bash
docker inspect container_id
docker stats
```

---

# Network Commands

```bash id="v5vhk9"
docker network ls
docker network create mynet
docker network inspect mynet
```

---

# Volume Commands

```bash id="mifv5d"
docker volume ls
docker volume create mydata
docker volume inspect mydata
```

---

# Cleanup Commands

```bash id="gj7r8p"
docker system prune -a
docker container prune
docker image prune
```

---

# Interview Answer

> Frequently used commands are docker build, run, ps, logs, exec, stop, rm, images, pull, push, network ls, volume ls, and system prune.

---

# 19. Write a Dockerfile for one application. Explain each layer in it.

Example: Node.js Application

```dockerfile id="tz6e5x"
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["node","server.js"]
```

---

# Explain Each Layer

## Layer 1

```dockerfile id="7f98n9"
FROM node:20-alpine
```

Base image with Node.js runtime. Alpine is lightweight.

---

## Layer 2

```dockerfile id="y0f5w0"
WORKDIR /app
```

Sets working directory inside container.

---

## Layer 3

```dockerfile id="rq6n8w"
COPY package*.json ./
```

Copies dependency files first.

This improves build caching.

---

## Layer 4

```dockerfile id="r0c9j5"
RUN npm install
```

Installs dependencies.

---

## Layer 5

```dockerfile id="l5a3cv"
COPY . .
```

Copies application source code.

---

## Layer 6

```dockerfile id="1wo4lg"
EXPOSE 3000
```

Documents listening port.

---

## Layer 7

```dockerfile id="gmcn0r"
CMD ["node","server.js"]
```

Starts application.

---

# Why This Order?

If code changes frequently but dependencies don’t:

* `npm install` layer is cached
* Faster rebuilds

---

# Interview Answer

> Each Dockerfile instruction creates a layer. We optimize ordering so dependency installation is cached, reducing rebuild time.

---

# 20. What is a docker-compose file? Explain what it does. Write one sample file if you can.

A **docker-compose.yml** file defines and runs **multi-container applications**.

Instead of running many `docker run` commands manually, Compose manages:

* Multiple services
* Networks
* Volumes
* Environment variables
* Dependencies

---

# Example Use Case

Application needs:

* Web app
* Database
* Redis cache

Compose runs all together.

---

# Sample File

```yaml id="8mpc9l"
version: "3.8"

services:
  app:
    image: mynodeapp:latest
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      DB_HOST: db

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: appdb
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
```

---

# Commands

```bash id="3p6xw8"
docker compose up -d
docker compose ps
docker compose down
```

---

# What It Does

| Section     | Purpose            |
| ----------- | ------------------ |
| services    | Defines containers |
| ports       | Port mapping       |
| environment | Variables          |
| depends_on  | Startup dependency |
| volumes     | Persistent data    |

---

# Interview Answer

> A docker-compose file is a YAML configuration used to define and run multi-container Docker applications. It allows us to start app, database, cache, networking, and volumes together using one command.

## 21. By default, which Docker network is present?

When Docker is installed, it creates default networks automatically.

## Default Docker Networks

```bash id="g5v3a9"
docker network ls
```

Usually shows:

| Network    | Driver | Purpose                                   |
| ---------- | ------ | ----------------------------------------- |
| **bridge** | bridge | Default network for standalone containers |
| **host**   | host   | Uses host machine network directly        |
| **none**   | null   | No networking                             |

---

## Most Common Default Network = bridge

If you run:

```bash id="q4m4q6"
docker run -d nginx
```

The container is attached to the **bridge** network unless another network is specified.

---

## How bridge Works

* Docker creates internal virtual network
* Containers get private IPs
* NAT used for internet access
* Port mapping publishes services

Example:

```bash id="r26smr"
docker run -d -p 80:80 nginx
```

---

## Interview Answer

> By default, Docker uses the **bridge network** for standalone containers. Docker also creates **host** and **none** default networks during installation.

---

# 22. What is the purpose of a multi-stage Dockerfile? How does it reduce image size?

## Purpose of Multi-Stage Dockerfile

A multi-stage Dockerfile uses multiple `FROM` statements to separate:

1. Build environment
2. Final runtime environment

This allows us to keep only required runtime files in final image.

---

## Why Needed

Traditional Docker image may include:

* Build tools
* Compilers
* Source code
* Dev dependencies
* Temporary files

All these increase size and security risk.

---

## How It Reduces Size

### Stage 1: Build Stage

Uses heavy image like Node, Java, Maven, Go SDK etc.

### Stage 2: Runtime Stage

Uses slim image like Alpine, Distroless, Nginx, JRE.

Only copies:

* compiled binaries
* build artifacts
* static files

---

## Example

Build stage size:

```text id="vx91dj"
node image = 1 GB
```

Runtime stage:

```text id="0ob4oq"
alpine image = 50 MB
```

Final image becomes much smaller.

---

## Benefits

* Smaller images
* Faster pull/push
* Better security
* Faster deployments
* Less storage cost

---

## Interview Answer

> Multi-stage Dockerfiles separate build and runtime stages. Build tools remain in earlier stages, while only final artifacts are copied to the runtime image, significantly reducing image size.

---

# 23. Write the multi-stage Dockerfile for the same

Example for Node.js application:

```dockerfile id="3rmg5r"
# Stage 1 - Build
FROM node:20 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

RUN npm run build

# Stage 2 - Runtime
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY --from=builder /app/dist ./dist

EXPOSE 3000

CMD ["node","dist/server.js"]
```

---

## Explanation

### Builder Stage

* Installs all dependencies
* Builds project

### Runtime Stage

* Uses lightweight Alpine image
* Installs production packages only
* Copies build output only

---

## Interview Answer

> This multi-stage Dockerfile uses one stage to build the Node.js application and another lightweight stage to run it in production.

---

# 24. How does container-to-container communication happen? Explain it.

Containers communicate through Docker networks.

---

# Same Network Communication

If two containers are attached to same user-defined bridge network, they can communicate using:

* Container name
* Service name (Compose)
* Internal IP

---

## Example

Create network:

```bash id="ppgnn1"
docker network create appnet
```

Run DB:

```bash id="83owep"
docker run -d --name db --network appnet mysql:8
```

Run App:

```bash id="9xajy7"
docker run -d --name web --network appnet myapp
```

App connects using:

```text id="c32j9f"
Host = db
Port = 3306
```

---

# Why Container Name Works?

Docker has built-in DNS service for user-defined networks.

It resolves:

```text id="8ltpkh"
db -> container IP
web -> container IP
```

---

# Docker Compose Example

```yaml id="rzklqj"
services:
  app:
  db:
```

Then app reaches DB via hostname:

```text id="qv2trj"
db
```

---

# Best Practice

Use user-defined bridge network instead of default bridge.

---

## Interview Answer

> Container-to-container communication happens through Docker networks. If containers are on the same network, Docker DNS resolves container names, allowing one container to connect to another using service names like db or redis.

---

# 25. Mention some Docker network types and explain their real-world use cases.

## Main Docker Network Types

---

# 1. Bridge Network

Default for standalone containers.

## Use Case

* Web app + DB on same host
* Local development

```bash id="zj3w7g"
docker network create mynet
```

---

# 2. Host Network

Container uses host machine’s network directly.

No NAT.

## Use Case

* High-performance apps
* Monitoring agents
* Low latency workloads

```bash id="ey7f8o"
docker run --network host nginx
```

---

# 3. None Network

No network access.

## Use Case

* Highly secure batch jobs
* Offline processing

```bash id="5j1znv"
docker run --network none busybox
```

---

# 4. Overlay Network

Connects containers across multiple Docker hosts.

Used in Docker Swarm / cluster.

## Use Case

* Multi-host microservices
* Distributed applications

---

# 5. Macvlan Network

Container gets its own IP on physical LAN.

## Use Case

* Legacy apps needing direct network presence
* Appliances / monitoring tools

---

# Summary Table

| Network | Real Use Case        |
| ------- | -------------------- |
| Bridge  | Single host apps     |
| Host    | High performance     |
| None    | Secure isolated jobs |
| Overlay | Multi-host cluster   |
| Macvlan | Direct LAN IP        |

---

## Interview Answer

> Common Docker networks are bridge, host, none, overlay, and macvlan. Bridge is used for standard containers, host for performance workloads, none for isolated workloads, overlay for clustered multi-host communication, and macvlan when containers need their own LAN IP.

## 26. What is the difference between CMD and ENTRYPOINT?

Both `CMD` and `ENTRYPOINT` define what runs when a container starts, but they behave differently.

---

# CMD

`CMD` provides the **default command or arguments**.

If another command is supplied during `docker run`, CMD is replaced.

```dockerfile id="k4d1sa"
CMD ["node","app.js"]
```

Run:

```bash id="q0n2u9"
docker run myapp
```

Executes:

```bash id="r2h4zp"
node app.js
```

If user runs:

```bash id="p1x5el"
docker run myapp node test.js
```

CMD is overridden.

---

# ENTRYPOINT

`ENTRYPOINT` defines the **main executable** that always runs.

```dockerfile id="v6j8mk"
ENTRYPOINT ["node"]
CMD ["app.js"]
```

Default run:

```bash id="f9w3ba"
node app.js
```

If user runs:

```bash id="g4m7zs"
docker run myapp test.js
```

Executes:

```bash id="n8u1tw"
node test.js
```

---

# Key Difference Table

| Feature           | CMD                  | ENTRYPOINT                 |
| ----------------- | -------------------- | -------------------------- |
| Purpose           | Default command/args | Main executable            |
| Easily overridden | Yes                  | No (unless `--entrypoint`) |
| Best use          | Flexible defaults    | Fixed runtime behavior     |

---

# Real-World Example

```dockerfile id="x7e2cr"
ENTRYPOINT ["python"]
CMD ["app.py"]
```

Useful because users can pass another script name.

---

# Interview Answer

> CMD sets the default command or parameters, while ENTRYPOINT sets the main executable of the container. CMD can be overridden easily, but ENTRYPOINT is typically fixed unless explicitly changed.

---

# 27. Where are Docker volumes stored?

Docker volumes are stored on the host machine in Docker-managed storage.

---

# Default Linux Path

```text id="y7t4vk"
/var/lib/docker/volumes/
```

Each volume gets its own folder.

Example:

```text id="r5m1qd"
/var/lib/docker/volumes/mysql_data/_data
```

---

# Windows / Mac

Docker Desktop stores volumes inside its managed VM/backend storage.

---

# Check Existing Volumes

```bash id="j6s8po"
docker volume ls
```

Inspect location:

```bash id="u4q2dn"
docker volume inspect mydata
```

Shows mountpoint path.

---

# Why Use Volumes?

* Persist data after container deletion
* Database storage
* Shared data between containers
* Backup and migration

---

# Interview Answer

> Docker volumes are stored on the host under Docker’s managed storage, commonly `/var/lib/docker/volumes/` on Linux. They are used for persistent data outside the container filesystem.

---

# 28. What is the difference between COPY and ADD?

Both instructions copy files into Docker images, but `ADD` has extra features.

---

# COPY

Copies local files/directories from build context into image.

```dockerfile id="n3z8ah"
COPY app/ /app/
```

Recommended for normal file copying.

---

# ADD

Can do everything COPY does, plus:

1. Extract local tar archives automatically
2. Download remote URLs (legacy capability)

```dockerfile id="w8r2cv"
ADD app.tar.gz /app/
```

Automatically extracts archive.

---

# Comparison Table

| Feature               | COPY | ADD              |
| --------------------- | ---- | ---------------- |
| Copy local files      | Yes  | Yes              |
| Auto extract tar      | No   | Yes              |
| URL download          | No   | Yes              |
| Simpler / predictable | Yes  | Less predictable |

---

# Best Practice

Use `COPY` unless you specifically need tar extraction.

---

# Interview Answer

> COPY is used for straightforward file copying into the image. ADD has additional features like extracting tar archives and fetching URLs. In most cases, COPY is preferred because it is simpler and more predictable.

---

# 29. How many containers can we run in Docker exactly?

There is **no fixed Docker limit** on number of containers.

The actual number depends on host resources and workload.

---

# Depends On

* CPU cores
* RAM
* Disk I/O
* Network bandwidth
* Container resource limits
* App type (lightweight vs heavy)

---

# Example

On a powerful server:

* Hundreds of lightweight containers possible
* Dozens of database containers
* Thousands of tiny idle containers possible

---

# Real Constraints

If each container needs:

* 512 MB RAM on 16 GB server
  Then practical limit is far lower than tiny Alpine containers.

---

# Manage with Limits

```bash id="z2v9pk"
docker run --memory=512m --cpus=1 nginx
```

---

# Enterprise Scale

Use Kubernetes or Docker Swarm to spread containers across nodes.

---

# Interview Answer

> Docker has no exact hard limit on container count. The number depends on CPU, memory, storage, networking, and workload size. A host may run from dozens to hundreds of containers depending on resources.

---

# 30. What happens to the data inside a container when you delete the running container?

It depends where the data is stored.

---

# Case 1: Data Inside Container Filesystem

If data is stored only inside container writable layer:

```text id="c9k4qt"
/app/data
/tmp/files
```

When container is deleted:

```bash id="v1d5mb"
docker rm mycontainer
```

That data is lost.

---

# Case 2: Data in Volume

If mounted volume is used:

```bash id="p7m2sd"
docker run -v mydata:/var/lib/mysql mysql
```

Deleting container does **not** delete volume by default.

Data remains.

---

# Case 3: Bind Mount

```bash id="f8t1zn"
docker run -v /host/data:/app/data myapp
```

Host data remains after container deletion.

---

# Important Command

Delete container + anonymous volumes:

```bash id="h3r6xa"
docker rm -v mycontainer
```

---

# Best Practice

Use volumes for:

* Databases
* Uploads
* Logs
* Persistent application data

---

# Interview Answer

> If data is stored inside the container filesystem, it is lost when the container is deleted. If data is stored in Docker volumes or bind mounts, it persists even after container removal.

## 31. You’re running an app using docker-compose with low traffic. As traffic grows, how do you scale the application in AWS? What services will you choose — EKS, ECS or EC2? Why?

## Initial State

Low traffic app using `docker-compose` usually runs on one VM/EC2.

As traffic grows, single-host compose becomes limited for:

* High availability
* Auto scaling
* Self healing
* Rolling deployments
* Load balancing
* Multi-AZ resilience

---

# My Migration Approach

## Phase 1 – Low Traffic

Use Amazon EC2 + Docker Compose

Good for:

* MVP
* Small traffic
* Low cost

---

## Phase 2 – Medium Growth

Move to Amazon ECS with AWS Fargate

Why:

* No server management
* Easy scaling
* ALB integration
* Simpler than Kubernetes

Best for teams wanting managed containers without K8s complexity.

---

## Phase 3 – Large Scale / Complex Microservices

Move to Amazon EKS

Why:

* Kubernetes ecosystem
* Advanced autoscaling
* Service mesh
* Helm
* GitOps
* Portability across clouds

---

# My Choice Summary

| Scenario                 | Best Choice   |
| ------------------------ | ------------- |
| Small app                | EC2 + Compose |
| Growing web app          | ECS + Fargate |
| Enterprise microservices | EKS           |

---

# Interview Answer

> If traffic grows from docker-compose on a single host, I would first move to ECS with Fargate because it simplifies scaling, load balancing, and HA. If the platform becomes microservice-heavy and needs Kubernetes capabilities, I would move to EKS. EC2 + Compose is fine only in early stages.

---

# 32. What’s your approach to disaster recovery for stateful apps running on containers?

Stateful apps include:

* MySQL
* PostgreSQL
* MongoDB
* Redis
* File storage

Containers are ephemeral, so state must live outside containers.

---

# DR Strategy

## 1. Separate Compute from Data

Use managed storage:

* Amazon RDS
* Amazon EFS
* Amazon EBS
* Amazon S3

---

## 2. Multi-AZ Deployment

Deploy workloads across multiple AZs.

---

## 3. Automated Backups

* DB snapshots
* Point-in-time restore
* Volume snapshots
* Config backups

---

## 4. Cross-Region DR

Replicate backups to second region.

---

## 5. IaC Rebuild

Use:

* Terraform
* AWS CloudFormation

Recreate environment quickly.

---

## 6. DR Testing

Regular restore drills.

---

# Interview Answer

> For stateful container apps, I keep state outside containers using managed databases or persistent volumes, enable backups and snapshots, deploy multi-AZ, replicate to DR region, and automate rebuild using Terraform. Recovery is tested regularly.

---

# 33. How do you debug a container that has exited?

## Step-by-Step

### 1. Check Exit Status

```bash id="j2s4pf"
docker ps -a
```

---

### 2. View Logs

```bash id="m6x1qd"
docker logs container_name
```

Most common issue source.

---

### 3. Inspect Container

```bash id="k8n2wr"
docker inspect container_name
```

Check:

* ExitCode
* CMD
* Env vars
* Mounts

---

### 4. Run Interactive Shell

```bash id="r1v8sn"
docker run -it image_name sh
```

or bash.

---

### 5. Override Entrypoint

```bash id="y3u7fa"
docker run -it --entrypoint sh image_name
```

---

### 6. Check Common Causes

* Wrong CMD
* Missing dependency
* Port conflict
* Bad permissions
* Crash on startup
* Env vars missing

---

# Interview Answer

> I check `docker ps -a`, review logs, inspect exit code, then run the image interactively or override entrypoint to troubleshoot startup scripts, permissions, environment variables, or dependency issues.

---

# 34. What's the difference between COPY and ADD commands in Dockerfile?

| Feature             | COPY | ADD |
| ------------------- | ---- | --- |
| Copy local files    | Yes  | Yes |
| Extract tar archive | No   | Yes |
| Fetch URL           | No   | Yes |
| Recommended default | Yes  | No  |

---

## Best Practice

Use `COPY` for normal file transfer.

Use `ADD` only when you need archive extraction.

---

## Example

```dockerfile id="j7e9la"
COPY . /app
ADD app.tar.gz /app
```

---

# Interview Answer

> COPY is the preferred simple command for copying files. ADD includes extra features like extracting archives and URL fetches, so COPY is safer and more predictable for most builds.

---

# 35. How would you handle secrets in a Docker container for a PHP application connecting to MySQL?

Never hardcode credentials in Dockerfile or source code.

---

# Secure Methods

## Best Options

* AWS Secrets Manager
* AWS Systems Manager Parameter Store
* HashiCorp Vault
* Docker Swarm secrets
* Kubernetes secrets

---

## PHP Example

Use environment variables:

```php
$dbhost = getenv("DB_HOST");
$dbuser = getenv("DB_USER");
$dbpass = getenv("DB_PASS");
```

---

## Runtime Injection

```bash id="n4z6qx"
docker run -e DB_USER=app -e DB_PASS=secret phpapp
```

Better: inject from secret manager in orchestration platform.

---

# Interview Answer

> I never store DB passwords in images. I use AWS Secrets Manager or Vault and inject secrets at runtime as environment variables or mounted secret files. PHP reads them securely using getenv().

---

# 36. What is a multi-stage Dockerfile? Can we use the second stage as the first?

## Multi-Stage Dockerfile

Uses multiple `FROM` statements.

```dockerfile id="q1f5mc"
FROM node:20 AS build
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

---

## Can We Use Second Stage as First?

Execution order stays top to bottom, but later stages can reference earlier stages.

You cannot logically use an undefined future stage before it exists.

This is invalid:

```dockerfile id="n5x8pa"
COPY --from=stage2 ...
FROM node AS stage2
```

---

## But You Can Build a Specific Stage

```bash id="v8m1kr"
docker build --target build .
```

---

# Interview Answer

> A multi-stage Dockerfile separates build and runtime stages to reduce image size. Later stages can copy artifacts from earlier stages. You cannot reference a later stage before it is defined.

---

# 37. How does an application communicate with the outside world?

Applications inside containers communicate externally through networking.

---

# Methods

## 1. Port Mapping

```bash id="b2s7rm"
docker run -p 80:3000 myapp
```

Host port 80 forwards to container port 3000.

---

## 2. Outbound Internet Access

Containers use NAT through host bridge network to call APIs, DBs, websites.

---

## 3. Reverse Proxy / Load Balancer

Use:

* Nginx
* HAProxy
* Application Load Balancer

---

## 4. DNS

Containers resolve external services using DNS.

---

# Interview Answer

> Containerized applications communicate externally through exposed ports, outbound NAT networking, DNS resolution, and load balancers or reverse proxies.

---

# 38. Write a Dockerfile for a ReactJS application

```dockerfile id="g4x2pz"
# Build stage
FROM node:20 AS build

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Runtime stage
FROM nginx:alpine

COPY --from=build /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx","-g","daemon off;"]
```

---

# Explanation

* Node builds React static files
* Nginx serves production content
* Multi-stage keeps image small

---

# Interview Answer

> React apps are commonly built using Node.js and served using Nginx in a multi-stage Dockerfile.

---

# 39. What is the difference between an image and a container? How to persist data across container restarts?

# Image vs Container

| Image              | Container                  |
| ------------------ | -------------------------- |
| Read-only template | Running instance of image  |
| Blueprint          | Live process               |
| Reusable           | Temporary unless persisted |

---

## Example

```bash id="x4r1qc"
docker run nginx
```

* `nginx` = image
* running instance = container

---

# Persist Data Across Restarts

Use volumes or bind mounts.

## Named Volume

```bash id="m3u6sv"
docker run -v mysql_data:/var/lib/mysql mysql
```

## Bind Mount

```bash id="f7n2lp"
docker run -v /host/data:/app/data myapp
```

---

# Why Important

Without volumes, container filesystem data may be lost when container is removed.

---

# Interview Answer

> An image is a packaged template containing application code and dependencies, while a container is the running instance of that image. To persist data across restarts, I use Docker volumes or bind mounts.

## 41. What is the use of docker-compose?

## What is Docker Compose?

**Docker Compose** is a tool used to define and run **multi-container Docker applications** using a YAML file (`docker-compose.yml` or `compose.yaml`).

Instead of starting containers one by one, Compose starts all related services together.

---

## Main Uses of Docker Compose

### 1. Multi-Container Application Management

Example application:

* Frontend (React)
* Backend (PHP/Node/Java)
* Database (MySQL/PostgreSQL)
* Cache (Redis)

Compose manages all together.

---

### 2. Single Command Startup

```bash id="r4j8xm"
docker compose up -d
```

Starts full stack.

---

### 3. Networking Between Containers

Services communicate by name:

```text id="u2w9pa"
app connects to db using hostname: db
```

---

### 4. Environment Variables

Pass config using `.env`

---

### 5. Persistent Volumes

Store database data.

---

## Example

```yaml id="g8m4la"
services:
  app:
    image: myapp
  db:
    image: mysql
```

---

## Interview Answer

> Docker Compose is used to manage multi-container applications using a YAML file. It helps define services, networks, volumes, and start or stop the entire stack with one command.

---

# 42. How do you check logs of a specific container?

Use the `docker logs` command.

---

## Basic Command

```bash id="s8x2fr"
docker logs container_name
```

Example:

```bash id="d5v1pl"
docker logs myapp
```

---

## Follow Live Logs

```bash id="j7n3cw"
docker logs -f myapp
```

Like tail -f.

---

## Last 100 Lines

```bash id="m4r8yt"
docker logs --tail 100 myapp
```

---

## With Timestamps

```bash id="v9q1db"
docker logs -t myapp
```

---

## Compose Logs

```bash id="h2x6se"
docker compose logs app
docker compose logs -f
```

---

## Interview Answer

> I use `docker logs <container_name>` to check logs. For real-time troubleshooting I use `docker logs -f`, and for compose environments I use `docker compose logs`.

---

# 43. How to expose a container to the outside world?

Use **port mapping**.

---

## Command

```bash id="p6w3qa"
docker run -p 8080:80 nginx
```

Meaning:

* Host Port = 8080
* Container Port = 80

Access from browser:

```text id="k3f9oe"
http://server-ip:8080
```

---

## Dockerfile Port Documentation

```dockerfile id="u1r8cw"
EXPOSE 80
```

This only documents port; actual exposure still needs `-p`.

---

## Compose Example

```yaml id="b7z4sn"
services:
  web:
    ports:
      - "8080:80"
```

---

## Production Exposure

Often done through:

* Nginx
* HAProxy
* Application Load Balancer

---

## Interview Answer

> A container is exposed externally using port mapping such as `docker run -p 8080:80 image`. This maps host traffic to the container port.

---

# 44. What is Docker?

Docker is a containerization platform used to build, package, ship, and run applications in lightweight containers.

---

## What Docker Provides

* Consistent runtime environment
* Fast deployment
* Isolation
* Portability
* Efficient resource usage

---

## Example

A Node.js app runs same way on:

* Laptop
* Test server
* Cloud VM
* Kubernetes cluster

Because it runs inside container.

---

## Core Components

| Component     | Purpose              |
| ------------- | -------------------- |
| Docker Engine | Runtime              |
| Docker Image  | Blueprint            |
| Container     | Running instance     |
| Registry      | Stores images        |
| Compose       | Multi-container apps |

---

## Interview Answer

> Docker is a platform that packages applications and dependencies into containers so they can run consistently across environments.

---

# 45. Difference between Containers & VMs?

| Feature      | Container     | Virtual Machine |
| ------------ | ------------- | --------------- |
| OS Included  | No            | Yes             |
| Startup Time | Seconds       | Minutes         |
| Size         | MBs           | GBs             |
| Performance  | Lightweight   | Heavier         |
| Isolation    | Process level | Full OS level   |

---

## Container

Shares host kernel.

## VM

Runs full guest OS on hypervisor.

---

## Interview Answer

> Containers share the host OS kernel and are lightweight, while VMs run separate guest operating systems and consume more resources.

---

# 46. Difference between Docker & Virtualization?

## Virtualization

A technology that creates virtual machines using hypervisors.

Examples:

* VMware
* Microsoft Hyper-V

---

## Docker

Docker uses **containerization**, not full hardware virtualization.

It isolates applications using OS kernel features like namespaces and cgroups.

---

## Comparison

| Feature         | Docker           | Traditional Virtualization |
| --------------- | ---------------- | -------------------------- |
| Model           | Containerization | Hardware virtualization    |
| OS per workload | No               | Yes                        |
| Speed           | Fast             | Slower                     |
| Resource Use    | Low              | Higher                     |

---

## Interview Answer

> Docker uses containerization and shares the host kernel, whereas virtualization creates full virtual machines with separate operating systems.

---

# 47. Difference between container and image?

| Image                     | Container          |
| ------------------------- | ------------------ |
| Read-only template        | Running instance   |
| Static package            | Live process       |
| Used to create containers | Created from image |

---

## Example

```bash id="e9q3ld"
docker run nginx
```

* `nginx` = image
* Running nginx instance = container

---

## Interview Answer

> An image is a packaged template containing code and dependencies. A container is the running instance created from that image.

---

# 48. How image builds?

Docker images are built using a **Dockerfile**.

---

## Steps

1. Read Dockerfile
2. Execute instructions top to bottom
3. Create layers
4. Save final image

---

## Example Dockerfile

```dockerfile id="w2m7ps"
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node","app.js"]
```

Build:

```bash id="x6r1bt"
docker build -t myapp:v1 .
```

---

## Interview Answer

> Docker builds images by reading Dockerfile instructions and creating layers for each step such as FROM, COPY, RUN, and CMD.

---

# 49. What are image layers? How image layers work?

## What are Layers?

Each major Dockerfile instruction creates a filesystem layer.

Example:

```dockerfile id="n8u4cx"
FROM ubuntu
COPY . /app
RUN apt install curl
```

Creates multiple layers.

---

## How Layers Work

Layers are:

* Read-only
* Cached
* Reusable
* Stacked together

When container starts, Docker adds a thin writable layer on top.

---

## Layer Example

| Instruction     | Layer           |
| --------------- | --------------- |
| FROM ubuntu     | Base layer      |
| COPY app        | App files layer |
| RUN apt install | Package layer   |

---

## Build Cache Benefit

If only source code changes:

* COPY layer rebuilds
* Earlier layers reused

This speeds builds.

---

## Why Important?

* Faster builds
* Smaller downloads (shared layers)
* Efficient storage

---

## Check Layers

```bash id="t1v5zo"
docker history nginx
```

---

## Interview Answer

> Docker image layers are read-only filesystem layers created by Dockerfile instructions. Layers are cached and reused, making builds faster and image storage more efficient.

