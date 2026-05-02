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
