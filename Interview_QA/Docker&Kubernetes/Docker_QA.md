
***

## 1) What is Docker and why use containers instead of VMs?

**Docker** is a platform for building, packaging, distributing, and running applications as **containers**. It standardizes how software is shipped and executed by bundling code with its runtime dependencies into portable, isolated units.

**Containers vs VMs:**

*   **Isolation model:**
    *   *Containers* share the host OS kernel, isolate via namespaces/cgroups.
    *   *VMs* virtualize hardware and run their own full guest OS.
*   **Footprint & density:**
    *   Containers are lightweight (MBs), high density per host.
    *   VMs are heavy (GBs), lower density.
*   **Startup time:**
    *   Containers start in milliseconds to seconds.
    *   VMs take much longer.
*   **Portability & consistency:**
    *   Container images are immutable, consistent across environments.
    *   VM images are larger and less agile.
*   **Performance:**
    *   Near-native performance for containers due to no guest OS overhead.
    *   VMs incur hypervisor + guest OS overhead.
*   **Security model:**
    *   VMs provide strong isolation via hypervisor boundaries.
    *   Containers rely on kernel isolation; hardening and least privilege are important.
*   **Use cases:**
    *   Containers excel for microservices, CI/CD, ephemeral workloads, and rapid scale.
    *   VMs fit for strong isolation needs, diverse OS kernels, or when you need full OS control.

***

## 2) What is a container image?

A **container image** is a **read‑only, immutable template** containing application binaries plus all runtime dependencies (libraries, system tools) and metadata (entrypoint, environment, exposed ports). Images are:

*   **Layered** (each instruction like `RUN`, `COPY` adds a layer).
*   **Content‑addressed** (identified by **digest** `sha256:...`), usually referenced by **tags** (`repo:tag`).
*   **Portable** across hosts supporting the same CPU architecture.

Common commands:

```bash
# List images
docker images

# Pull an image
docker pull nginx:1.25

# Inspect an image
docker inspect nginx:1.25

# Tag and push to a registry
docker tag nginx:1.25 myregistry.example.com/web/nginx:prod
docker push myregistry.example.com/web/nginx:prod
```

***

## 3) What is a Docker container?

A **container** is a **running instance** of an image plus a **writable container layer**. It has its own process namespace, filesystem view, network stack, and resource limits.

Key points:

*   Containers run a **single main process** (PID 1 in the container). When it exits, the container stops.
*   The image layers are **read‑only**; changes at runtime go into the **writable layer** (ephemeral unless you mount volumes).
*   You can map ports and mount volumes to interact with the host.

Common commands:

```bash
# Run a container (detached), map port 8080->80
docker run -d -p 8080:80 --name web nginx:1.25

# List running containers
docker ps

# Exec into a container shell
docker exec -it web /bin/sh

# View logs
docker logs -f web

# Stop & remove
docker stop web && docker rm web
```

***

## 4) What is a Dockerfile?

A **Dockerfile** is a declarative text manifest with instructions to **build an image**. Common instructions:

*   `FROM` (base image)
*   `RUN` (execute commands at build time)
*   `COPY`/`ADD` (add files)
*   `WORKDIR` (set working directory)
*   `ENV` (set environment variables)
*   `EXPOSE` (document ports)
*   `USER` (set runtime user)
*   `ENTRYPOINT` / `CMD` (default process)

Example (small web app):

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
ENV NODE_ENV=production
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

Build and run:

```bash
docker build -t myapp:1.0 .
docker run -d -p 3000:3000 --name myapp myapp:1.0
```

**Multi‑stage build** (to reduce image size):

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY . .
RUN go build -o app

FROM alpine:3.20
COPY --from=build /src/app /usr/local/bin/app
ENTRYPOINT ["app"]
```

***

## 5) Main components of Docker architecture

*   **Docker Engine**  
    The overall client–server system that builds and runs containers.

*   **Docker daemon (`dockerd`)**  
    Background service that manages images, containers, networks, volumes; exposes REST API over a local Unix socket (default `/var/run/docker.sock`) or TCP.

*   **Container runtime stack**
    *   **containerd**: High‑level container lifecycle management (pull, push, run).
    *   **runc**: Low‑level OCI runtime that creates containers using Linux namespaces/cgroups.

*   **Docker CLI (`docker`)**  
    Command‑line client that talks to the daemon via the API.

*   **Registry**  
    Remote service to store and distribute images (e.g., Docker Hub, ACR, ECR, GHCR). Images are pushed/pulled via the OCI Distribution API.

***

## 6) What is the Docker daemon (`dockerd`)?

`dockerd` is the **server‑side process** that:

*   Listens for **API requests** from the CLI or other clients.
*   Manages **builds**, **image storage**, **container lifecycle**, **networks** (bridge/overlay/macvlan), and **volumes**.
*   Integrates with **containerd**/**runc** to create and run containers.
*   Uses **storage drivers** (e.g., `overlay2`) and **network drivers**.
*   Can be configured to listen on TCP (`-H tcp://0.0.0.0:2375`) or default Unix socket.

Useful commands:

```bash
# Check daemon info & configuration
docker info
docker system df

# Manage resources
docker network ls
docker volume ls
```

***

## 7) What is the Docker CLI and how do you use it?

The **Docker CLI** (`docker`) is the **front‑end** to interact with Docker Engine. It’s organized as verbs and subcommands:

*   **Image lifecycle:** `docker build`, `pull`, `push`, `tag`, `rmi`, `inspect`
*   **Container lifecycle:** `docker run`, `start`, `stop`, `rm`, `ps`, `exec`, `logs`, `cp`, `inspect`
*   **Resources:** `docker network`, `docker volume`
*   **Maintenance:** `docker system prune`, `docker login`, `logout`

Examples:

```bash
# Build with build cache
docker build -t repo/app:1.0 .

# Run with env, volumes, resource limits
docker run -d \
  --name app \
  -p 8080:8080 \
  -e APP_ENV=prod \
  -v app-data:/var/lib/app \
  --memory=512m --cpus=1.0 \
  repo/app:1.0

# View container details (JSON)
docker inspect app

# Clean up unused images/containers/networks
docker system prune -f
```

Tip: Use **Compose** for multi‑container apps:

```bash
docker compose up -d
docker compose down
```

***

## 8) What is Docker Hub?

**Docker Hub** is the **default public registry** integrated with Docker:

*   Hosts **official images** (e.g., `nginx`, `redis`, `postgres`) and **verified publisher** images.
*   Supports **namespaces**, **teams**, and **private repositories**.
*   Offers **webhooks**, **scanning** (on certain plans), and **rate limits** for pulls (account‑dependent).

You interact via:

```bash
docker login
docker pull nginx:latest
docker push myorg/myimage:1.0
```

***

## 9) What is a Docker registry and what alternatives exist?

A **Docker (OCI) registry** is a server that **stores and serves container images**. Docker Hub is the default, but there are many alternatives:

*   **Cloud registries:**
    *   **Azure Container Registry (ACR)**
    *   **Amazon Elastic Container Registry (ECR)**
    *   **Google Artifact Registry / GCR**
    *   **GitHub Container Registry (GHCR)**
    *   **GitLab Container Registry**
*   **Enterprise/self-hosted:**
    *   **Harbor** (CNCF)
    *   **Quay** (Red Hat)
    *   **JFrog Artifactory**
    *   **Sonatype Nexus**
    *   **Docker Distribution** (open-source registry server)

Registries support features like **RBAC**, **vulnerability scanning**, **immutability**, **geo-replication**, **content trust**, and **retention policies**.

***

## 10) What is an image layer?

An **image layer** is a **snapshot of filesystem changes** created by each Dockerfile instruction (e.g., `RUN`, `COPY`). Key properties:

*   **Immutable & cached:** Layers are content-addressed; identical instructions produce reusable cache. Speeds up builds and reduces storage/network usage.
*   **Union filesystem:** Layers are stacked using a storage driver (e.g., `overlay2`). At runtime, the container adds a **writable layer** on top (**copy‑on‑write** for file modifications).
*   **Optimization best practices:**
    *   Order instructions to **maximize cache hits** (e.g., separate `COPY package*.json` from app source for Node `npm ci` caching).
    *   **Minimize layers** and **combine RUN steps** where sensible.
    *   Clean up temporary files within the same `RUN` step.
    *   Prefer slim/base images to cut size (e.g., `alpine`, `distroless`).

Visualizing:

    [ Layer 1: FROM node:20-alpine ]
    [ Layer 2: RUN apk add --no-cache curl ]
    [ Layer 3: COPY package*.json /app/ ]
    [ Layer 4: RUN npm ci ]
    [ Layer 5: COPY . /app ]
    --------------------------------
    [ Writable container layer ]

Inspect and manage:

```bash
docker history repo/app:1.0   # show layers
docker image prune -a          # remove unused image layers (careful!)
```

***

Awesome—continuing the interview‑ready series with hands‑on commands, practical tips, and gotchas. I’ll map answers directly to your questions (11–20).

***

## 11) How do Docker image layers affect builds and storage?

**Builds:**

*   Each Dockerfile instruction (e.g., `RUN`, `COPY`) creates a **layer**. Docker uses **content-addressed caching**—if a layer’s inputs haven’t changed, it reuses it, speeding up rebuilds.
*   The **order of instructions** matters: changes earlier in the Dockerfile invalidate cache for all subsequent layers.
*   Best practices:
    *   Put frequently changing files **later** (e.g., app source) and package manifests **earlier** to maximize cache.
    *   Clean up in the same `RUN` to avoid carrying temp files into layer history.

**Storage:**

*   Layers are **immutable and shared** across images. If multiple images use the same base layer, it’s stored once on the host.
*   Containers add a **writable layer** at runtime; changes there are ephemeral unless persisted via **volumes** or **bind mounts**.
*   Storage drivers (commonly `overlay2`) implement **union filesystems** with **copy‑on‑write** semantics.

Useful commands:

```bash
# See image layer history
docker history repo/app:1.0

# Storage usage by images/containers
docker system df

# Remove dangling/unused images & build cache (careful!)
docker image prune -a         # images
docker builder prune          # build cache
```

***

## 12) Difference between `docker run` and `docker create`

*   `docker create`  
    Creates a container **from an image** with the given config (env, volumes, ports) but **does not start** it. Returns a container ID.
    ```bash
    docker create --name app -p 8080:8080 -e APP_ENV=prod repo/app:1.0
    ```

*   `docker run`  
    Convenience command equivalent to **create + start** (and optionally attach). Most commonly used.
    ```bash
    docker run -d --name app -p 8080:8080 repo/app:1.0
    ```

Tip: Use `create` when you want to pre‑stage containers and control start timing or attach behavior (`docker start -ai`).

***

## 13) How do you list running containers? (`docker ps`)

```bash
docker ps
```

Common flags:

```bash
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps --filter "name=web" --filter "ancestor=nginx:1.25"
```

***

## 14) How do you list all containers including stopped ones? (`docker ps -a`)

```bash
docker ps -a
```

Useful filters:

```bash
# Show exited containers
docker ps -a --filter "status=exited"

# Show by label or name
docker ps -a --filter "label=env=prod" --filter "name=app"
```

***

## 15) How do you remove an image? (`docker rmi`)

```bash
# Remove by tag or ID
docker rmi repo/app:1.0

# Force remove (e.g., if referenced by stopped containers)
docker rmi -f repo/app:1.0

# Remove dangling images (no tag)
docker image prune
# Remove all unused images
docker image prune -a
```

> **Note:** You must remove containers based on an image before removing that image (or use `-f` with caution).

***

## 16) How do you remove a container? (`docker rm`)

```bash
# Remove a stopped container
docker rm app

# Remove running container (force: sends SIGKILL)
docker rm -f app

# Remove along with anonymous volumes
docker rm -v app

# Bulk remove exited containers
docker rm $(docker ps -aq --filter status=exited)
```

***

## 17) How do you start and stop a container? (`docker start/stop`)

```bash
# Start a stopped container
docker start app

# Start and attach to STDOUT/STDERR
docker start -ai app

# Stop (graceful SIGTERM then SIGKILL after timeout)
docker stop app

# Customize timeout (seconds before SIGKILL)
docker stop --time 20 app
```

Extra:

```bash
# Restart a running/stopped container
docker restart app

# Set restart policy at run/create time
docker run --restart=unless-stopped -d --name app repo/app:1.0
```

***

## 18) How do you attach to a running container? (`docker attach`)

```bash
docker attach app
```

**Behavior & gotchas:**

*   Attaches to the **primary process (PID 1)** stdout/stderr/STDIN. If that process doesn’t produce output or manages TTY differently, `attach` may look like it’s “stuck”.
*   **Detach keys**: `Ctrl‑P` then `Ctrl‑Q` (default) to leave the container running.
*   For interactive shells inside a container, prefer **`docker exec -it`** instead of `attach`.

Set detach keys:

```bash
docker attach --detach-keys="ctrl-c" app
```

***

## 19) How do you execute a command inside a running container? (`docker exec`)

```bash
# Run a shell inside the container
docker exec -it app /bin/sh
# or for Debian/Ubuntu-based images
docker exec -it app /bin/bash

# Execute arbitrary commands
docker exec app ls -la /var/log
docker exec -e DEBUG=true app env

# With user and working directory
docker exec -u root -w /tmp app whoami
```

**Tip:** Use `-it` (interactive + TTY) for shells; omit it for non-interactive commands in scripts.

***

## 20) How do you view container logs? (`docker logs`)

```bash
# Basic logs
docker logs app

# Follow (stream) logs
docker logs -f app

# Tail last N lines
docker logs --tail 100 app

# Since / until time window
docker logs --since 1h app
docker logs --until "2025-12-30T13:00:00" app

# With timestamps
docker logs -t app
```

**Notes:**

*   `docker logs` shows stdout/stderr of the container’s main process (PID 1). If the app logs to files, use `docker exec` to inspect them or configure log drivers.
*   Production setups often use **log drivers** (`json-file` default, `syslog`, `fluentd`, `gelf`) or sidecars/agents to ship logs to a centralized system.

***

### Quick extras tailored for you (DevOps/SRE)

*   **Inspect for deep details:**
    ```bash
    docker inspect app | jq '.[0].HostConfig | {PortBindings, Binds, LogConfig, Resources: {Memory, NanoCpus}}'
    ```
*   **Resource cleanup routine (safe):**
    ```bash
    docker ps -aq --filter status=exited | xargs -r docker rm
    docker image prune -f
    docker volume prune -f
    ```
*   **ACR push/pull (since you work a lot with Azure):**
    ```bash
    az acr login --name myacr
    docker tag repo/app:1.0 myacr.azurecr.io/repo/app:1.0
    docker push myacr.azurecr.io/repo/app:1.0
    docker pull myacr.azurecr.io/repo/app:1.0
    ```

You got it—continuing with crisp, interview‑ready answers and practical examples (21–30).

***

## 21) What is `docker pull` and `docker push`?

*   **`docker pull`**: Downloads an image from a registry (e.g., Docker Hub, ACR, ECR) to your local host.
    ```bash
    docker pull nginx:1.25
    docker pull myacr.azurecr.io/apps/web:prod
    ```
    You can reference by **tag** (`repo:tag`) or **digest** (`repo@sha256:<digest>`).

*   **`docker push`**: Uploads a local image to a registry. Requires `docker login` and that the image is **tagged** with the registry hostname.
    ```bash
    docker login
    docker tag repo/app:1.0 myacr.azurecr.io/repo/app:1.0
    docker push myacr.azurecr.io/repo/app:1.0
    ```

***

## 22) How do you build an image from a Dockerfile? (`docker build`)

```bash
# Basic build (Dockerfile in current directory)
docker build -t repo/app:1.0 .

# Specify Dockerfile and platform (BuildKit enabled)
docker build -t repo/app:1.0 -f deploy/Dockerfile --platform=linux/amd64 .

# Build with args and no cache
docker build -t repo/app:1.0 --build-arg NODE_ENV=production --no-cache .
```

**Notes:**

*   The final `.` is the **build context** (see next).
*   Use `.dockerignore` to keep your context lean.
*   BuildKit adds features like `--secret`, `--ssh`, better caching, and parallel stages.

***

## 23) What is a context in a Docker build?

The **build context** is the directory (and its contents) that Docker sends to the daemon when you run `docker build`. Paths in `Dockerfile` (e.g., `COPY . /app`) are relative to this context.

*   Keep the context **small** (large contexts slow builds and bloat cache).
*   Use **`.dockerignore`** to exclude files (e.g., `node_modules/`, `.git/`, build artifacts).
*   Typical pattern: run `docker build` from your project root **but** structure Dockerfile to copy only what’s needed.

***

## 24) What is the `FROM` instruction used for in a Dockerfile?

`FROM` sets the **base image** for your build. It’s typically the first instruction and defines the starting filesystem and default environment.

Examples:

```dockerfile
FROM node:20-alpine             # language/runtime base
FROM alpine:3.20                # minimal OS base
FROM scratch                    # empty base (for statically compiled binaries)
FROM golang:1.22-alpine AS build  # multi-stage with a named build stage
```

You can use multiple `FROM` instructions for **multi-stage builds** to compile in one stage and copy artifacts to a slim runtime stage.

***

## 25) What is the `RUN` instruction used for?

`RUN` executes commands **at build time** and creates a **new layer** with the resulting filesystem changes.

Forms:

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apk add --no-cache curl

# Exec form (no shell; better for exact arg handling)
RUN ["apk", "add", "--no-cache", "curl"]
```

**Best practices:**

*   Combine commands and **clean up** in the same `RUN` to avoid bloating layers:
    ```dockerfile
    RUN apk add --no-cache curl && rm -rf /var/cache/apk/*
    ```
*   Order your Dockerfile to **maximize cache** (install deps before copying app source).

***

## 26) What is the `CMD` instruction and how does it differ from `ENTRYPOINT`?

*   **`CMD`** sets the **default arguments or command** that runs when the container starts. It can be **overridden** by `docker run <args>`.

*   **`ENTRYPOINT`** sets the **executable** that always runs. Combine `ENTRYPOINT` + `CMD` to define an executable with default args.

Examples:

```dockerfile
# CMD alone (overridden by docker run arguments)
CMD ["npm", "start"]

# ENTRYPOINT + CMD (CMD supplies default args)
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

Behavior:

*   `docker run image --other-args` → overrides **CMD**, not **ENTRYPOINT**.
*   To override ENTRYPOINT at runtime: `docker run --entrypoint /bin/sh image`.

***

## 27) What is `ENTRYPOINT` used for?

`ENTRYPOINT` defines the **main executable** for the container. It makes your image behave like a specific command/tool.

```dockerfile
ENTRYPOINT ["/usr/local/bin/app"]
CMD ["--port=8080", "--log-level=info"]  # default args
```

**Tips:**

*   Prefer **exec form** (`["binary","arg1"]`) to avoid shell’s PID 1 issues and signal handling problems.
*   If you need pre‑start logic, use a tiny wrapper (bash/sh) but ensure proper signal forwarding.

***

## 28) How do you set environment variables in a Dockerfile? (`ENV`)

`ENV` defines environment variables **persisted** in the image and available at **runtime**.

```dockerfile
ENV NODE_ENV=production
ENV PATH="/app/bin:${PATH}"
ENV APP_NAME="web" APP_REGION="ap-south-1"
```

*   Override at runtime: `docker run -e NODE_ENV=dev repo/app:1.0`
*   For **build‑time values**, use `ARG` (not persisted):
    ```dockerfile
    ARG BUILD_DATE
    RUN echo "Build date: ${BUILD_DATE}"
    ```

***

## 29) How to add files from host to image? (`COPY` vs `ADD`)

Use `COPY` to copy files/directories from the **build context** (or from another stage) into the image. Prefer `COPY` for clarity and predictable behavior.

```dockerfile
COPY --chown=node:node package*.json /app/
COPY . /app

# Multi-stage copy (from a named stage)
COPY --from=build /src/app /usr/local/bin/app
```

`ADD` can do everything `COPY` does **plus** extra features (see next), but those extras are rarely needed and can be surprising. Most Dockerfiles should use `COPY`.

***

## 30) What is the difference between `COPY` and `ADD`?

*   **`COPY`**
    *   Copies files/directories from **build context** (or another stage) to the image.
    *   Predictable; no side effects.

*   **`ADD`**
    *   Copies from context **and** additionally:
        *   **Auto‑extracts local tar archives** (`.tar`, `.tar.gz`, `.tgz`, `.xz`) into the destination.
        *   **Can fetch remote URLs** (e.g., `ADD https://example.com/file.tar.gz /tmp/`).
            > This is often discouraged—prefer `RUN curl`/`wget` for explicit control, caching, and verification.
    *   Because of these behaviors, `ADD` can **invalidate cache unexpectedly** and introduce hidden steps.

**Guideline:** Use `COPY` by default. Use `ADD` **only** when you intentionally want tar auto‑extraction (local archive) or a one‑off URL download with awareness of caching/security implications.

***

### Mini examples tying it together

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM alpine:3.20
WORKDIR /app
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
EXPOSE 3000
ENTRYPOINT ["node"]
CMD ["./dist/server.js"]
```

Build & run:

```bash
docker build -t repo/app:1.0 .
docker run -d -p 3000:3000 --name app repo/app:1.0
```

***

Absolutely—continuing with crisp, interview‑ready answers and practical examples (31–40). I’ll keep it hands‑on and include tips you’ll actually use day‑to‑day.

***

## 31) What is `WORKDIR` in a Dockerfile?

`WORKDIR` sets the **working directory** for subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` instructions. If the directory doesn’t exist, Docker **creates it**.

```dockerfile
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]   # runs in /app
```

**Tips:**

*   Prefer `WORKDIR` over chaining `cd` in `RUN` lines—cleaner and more cache‑friendly.
*   You can set it multiple times; each changes the active directory for subsequent instructions.

***

## 32) What does `EXPOSE` do in a Dockerfile?

`EXPOSE` **documents** the port(s) your containerized application listens on. It **does not** publish ports to the host by itself.

```dockerfile
EXPOSE 8080
```

To actually publish the port when running:

```bash
docker run -p 8080:8080 repo/app:1.0
```

**Notes:**

*   `EXPOSE` pairs with `docker run -P` (capital P) to auto‑map all exposed ports to random host ports.
*   Treat `EXPOSE` as documentation; don’t rely on it for security or networking behavior.

***

## 33) How do you specify default command arguments? (`CMD ["arg1","arg2"]`)

There are two common patterns:

1.  **`CMD` alone (exec form)** — sets the full command:

```dockerfile
CMD ["node", "server.js"]
```

This runs `node server.js` by default and can be **overridden** by appending args to `docker run`.

2.  **`ENTRYPOINT` + `CMD`** — `ENTRYPOINT` sets the executable; `CMD` provides **default arguments**:

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]   # default args passed to ENTRYPOINT
```

Overriding:

```bash
# Overrides CMD only (keeps ENTRYPOINT)
docker run repo/nginx -g "daemon off;" -c /etc/nginx/nginx.conf

# Override ENTRYPOINT at runtime
docker run --entrypoint /bin/sh repo/nginx -c 'echo hi'
```

> The JSON array (`exec` form) is preferred for proper signal handling and avoiding shell parsing issues.

***

## 34) How do you tag an image? (`docker tag`)

`docker tag` creates an additional **name (repository:tag)** for an existing image ID. This doesn’t copy data; it’s just another pointer.

```bash
# Syntax: docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
docker tag repo/app:1.0 myacr.azurecr.io/repo/app:1.0
docker push myacr.azurecr.io/repo/app:1.0

# Tag latest or a digest
docker tag repo/app@sha256:<digest> repo/app:stable
```

***

## 35) How do you view image history and layers? (`docker history`)

```bash
docker history repo/app:1.0
docker history --no-trunc repo/app:1.0   # full commands and IDs
```

This shows each layer’s size and the instruction that created it. Useful to:

*   Find **large layers** causing bloat.
*   Verify that cache‑friendly ordering is working as expected.

***

## 36) What are volumes and bind mounts?

*   **Volumes**  
    Docker‑managed storage living under Docker’s area (e.g., `/var/lib/docker/volumes/...`). Decoupled from the container lifecycle and can be backed by **volume drivers** (e.g., NFS, third‑party plugins).

*   **Bind mounts**  
    Mount a **host path** (file or directory) directly into a container. Great for **development** where you need live code sync.

Mounting syntax (prefer `--mount` for clarity):

```bash
# Named volume
docker run -d --name db \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres:16

# Bind mount
docker run -d --name web \
  --mount type=bind,source="$(pwd)"/site,target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx:1.25
```

Short `-v` examples:

```bash
# Named volume
docker run -v pgdata:/var/lib/postgresql/data postgres:16
# Bind mount
docker run -v $(pwd)/site:/usr/share/nginx/html:ro nginx:1.25
```

**Extras:**

*   Add `:ro` for read‑only.
*   On SELinux hosts, consider `:z` or `:Z` for relabeling bind mounts.

***

## 37) When should you use volumes vs bind mounts?

**Use volumes when:**

*   You need **portable, Docker‑managed data** decoupled from host paths.
*   You want **easier backups**, migration, and isolation from host file changes.
*   You rely on **drivers** (e.g., NFS, cloud/third‑party plugins).
*   You’re running **production** databases or stateful services.

**Use bind mounts when:**

*   **Local development**: live‑edit code and see changes instantly.
*   You need to mount **specific host files** (e.g., config files, TLS certs) into a container.
*   You’re debugging and want quick access to logs or artifacts on the host.

Rule of thumb: **Volumes for prod/state; bind mounts for dev/debug or specific host integrations.**

***

## 38) How do you create and list Docker volumes? (`docker volume create`, `docker volume ls`)

```bash
# Create a named volume
docker volume create pgdata

# List volumes
docker volume ls

# Inspect a volume (path, mountpoint, driver)
docker volume inspect pgdata
```

You can specify a driver and options:

```bash
docker volume create --driver local --opt type=nfs \
  --opt o=addr=10.0.0.5,rw --opt device=:/export/pgdata nfs_pgdata
```

***

## 39) How do you remove unused volumes? (`docker volume prune`)

```bash
# Remove all unused (dangling) volumes
docker volume prune
# Non-interactive
docker volume prune -f

# Remove a specific volume
docker volume rm pgdata
```

> Be careful: removing a volume permanently deletes the data stored in it.

***

## 40) What is a data‑only container pattern? (concept)

The **data‑only container** is an older pattern where you create a container **whose sole purpose is to own volumes**, and other containers consume those volumes via `--volumes-from`.

Example (legacy approach):

```bash
# Create a data container that defines the volume(s)
docker create --name app-data -v /var/lib/app busybox:latest

# Run app containers that reuse those volumes
docker run --name app1 --volumes-from app-data repo/app:1.0
docker run --name app2 --volumes-from app-data repo/app:1.0
```

**Why it existed:** Before **named volumes** were first‑class, this was a way to persist and share data independently of any single app container.

**Today:** Prefer **named volumes** instead—they’re clearer, easier to manage, and don’t require a “holder” container.

```bash
docker volume create appdata
docker run -d --name app -v appdata:/var/lib/app repo/app:1.0
```

***

### Quick, practical add‑ons for you

*   **Clean up safely on a dev box:**
    ```bash
    docker ps -aq --filter status=exited | xargs -r docker rm
    docker image prune -f
    docker volume prune -f
    ```
*   **Compose volume/bind examples:**
    ```yaml
    services:
      db:
        image: postgres:16
        volumes:
          - pgdata:/var/lib/postgresql/data
      web:
        image: nginx:1.25
        volumes:
          - ./site:/usr/share/nginx/html:ro
    volumes:
      pgdata:
    ```

Great—continuing with interview‑ready, hands‑on answers for 41–50, with tips you’ll actually use in SRE/DevOps work.

***

## 41) What is copy‑on‑write filesystem in container images?

**Copy‑on‑write (CoW)** is how Docker’s storage drivers (e.g., `overlay2`) implement layered images and the container’s writable layer:

*   **Read from lower layers; write to the top**: Image layers are **read‑only**. When a file is modified, the driver **copies** the file from the lower (image) layer to the container’s **writable layer** and then writes the change there.
*   **Space & speed efficiency**: Files are shared across containers until modified; only **differences** consume extra space.
*   **Whiteouts (deletes)**: Deleting a file in an upper layer creates a **whiteout** so it appears removed even though it still exists in a lower layer.
*   **Implications**:
    *   Frequent small writes to big files can be expensive (triggers full file copy). Put write‑heavy data on **volumes**.
    *   Keep images small and stable; runtime changes should go to volumes or external storage.

Check your driver:

```bash
docker info | grep -i 'Storage Driver'
```

***

## 42) How to inspect a container or image metadata? (`docker inspect`)

`docker inspect` returns detailed JSON metadata for images, containers, networks, volumes, etc.

```bash
# Container metadata (mounts, network, env, health, resources…)
docker inspect app

# Image metadata (layers, config, entrypoint, env…)
docker inspect repo/app:1.0

# Query specific fields with --format
docker inspect app --format '{{.NetworkSettings.IPAddress}}'
docker inspect app --format '{{json .Mounts}}' | jq .
docker inspect repo/app:1.0 --format '{{.Config.Entrypoint}} {{.Config.Cmd}}'
```

***

## 43) How to monitor container resource usage? (`docker stats`)

Use `docker stats` to stream live CPU, memory, network, and I/O metrics:

```bash
# All running containers
docker stats

# Specific containers with formatted output
docker stats app db --no-stream \
  --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
```

For deeper metrics in prod, ship to **Prometheus**/Grafana via cAdvisor or node agents; or use orchestrator tooling (K8s Metrics Server + HPA, Azure Monitor Container Insights, etc.).

***

## 44) How to limit container CPU and memory resources at runtime? (`--cpus`, `--memory`)

```bash
# Limit to 1 vCPU and 512 MiB RAM
docker run -d --name app --cpus=1.0 --memory=512m repo/app:1.0

# Additional controls
docker run -d \
  --memory=512m --memory-swap=1g \        # total incl. swap
  --memory-reservation=256m \             # soft limit
  --cpuset-cpus="0,2" \                   # pin to specific CPUs
  --cpu-shares=512 \                      # relative CPU weight
  --pids-limit=200 \                      # process count
  repo/app:1.0
```

**Tips:**

*   `--memory` is a **hard limit**; the kernel OOM killer may terminate the process if exceeded.
*   `--cpus` sets a quota (e.g., `0.5` = half a CPU); `--cpu-shares` is **relative** within contention.

***

## 45) How to run a container in detached vs foreground mode? (`-d`)

*   **Foreground (attached)**: default; streams logs to your terminal. Add `-it` for interactive TTY.
    ```bash
    docker run --name web -p 8080:80 -it nginx:1.25
    ```

*   **Detached**: runs in background; view logs with `docker logs`.
    ```bash
    docker run --name web -p 8080:80 -d nginx:1.25
    ```

Useful combos:

```bash
docker logs -f web          # follow logs
docker attach web           # attach to PID 1 (detach with Ctrl-P Ctrl-Q)
```

***

## 46) What is `HEALTHCHECK` in Dockerfile and how to use it?

`HEALTHCHECK` lets Docker **probe** your container to determine if it’s healthy. The result appears in `docker ps` (`healthy` / `unhealthy`) and can be used by orchestrators.

```dockerfile
# Inside Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:8080/healthz || exit 1
```

Override or define at runtime:

```bash
docker run -d \
  --health-cmd='curl -fsS http://localhost:8080/healthz || exit 1' \
  --health-interval=30s --health-timeout=10s --health-retries=3 \
  --name app repo/app:1.0
```

Check status:

```bash
docker inspect app --format '{{json .State.Health}}' | jq .
```

**Best practice:** Make the health endpoint **fast** and **local**; avoid external dependencies to prevent false negatives.

***

## 47) How do you define port mappings between host and container? (`-p host:container`)

Publish container ports to the host with `-p`:

```bash
# hostPort:containerPort
docker run -d -p 8080:80 nginx:1.25

# hostIP:hostPort:containerPort (bind to specific interface)
docker run -d -p 127.0.0.1:8080:80 nginx:1.25

# Multiple mappings and protocols
docker run -d -p 8080:80 -p 8443:443/tcp -p 53:53/udp my/dns

# Publish all EXPOSEd ports to random host ports
docker run -d -P repo/app:1.0
```

List published ports:

```bash
docker ps --format '{{.Names}}\t{{.Ports}}'
```

***

## 48) What is the default network driver (`bridge`) and when is it used?

*   **`bridge`** is the default driver for standalone containers on a single host. Containers join a private **NATed** network (default `bridge`, via `docker0`), get a private IP, and can reach the internet via NAT through the host.
*   Containers on the same user‑defined bridge can reach each other by **name** using built‑in DNS.

Key commands:

```bash
# List & inspect networks
docker network ls
docker network inspect bridge

# Create a user-defined bridge (better isolation & DNS)
docker network create --driver bridge appnet
docker run -d --name api --network appnet repo/api:1.0
docker run -d --name web --network appnet -p 8080:80 repo/web:1.0
```

**Tip:** Prefer **user‑defined bridge networks** over the default for automatic DNS and better control.

***

## 49) What are other built‑in Docker network drivers (host, none, overlay)?

*   **host**: Container shares the host’s network stack (no port mapping). Lowest latency, but no isolation.
    ```bash
    docker run --network host repo/app:1.0
    ```
*   **none**: No networking; loopback only. Useful for offline/batch/security‑tight jobs.
    ```bash
    docker run --network none repo/job:1.0
    ```
*   **overlay**: Multi‑host virtual network (VXLAN) for **Docker Swarm** services/containers; lets containers on different hosts communicate as if on one L2 network. Requires `docker swarm init`.
    ```bash
    docker swarm init
    docker network create -d overlay appmesh
    docker service create --name web --network appmesh nginx:1.25
    ```
*   **macvlan/ipvlan** (also built‑in): Assigns containers their **own MAC/IP** on the physical network; useful when devices on the LAN must talk directly to containers.
    ```bash
    docker network create -d macvlan \
      --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
      -o parent=eth0 lan
    ```

***

## 50) What is an overlay network and when is it used?

An **overlay network** is a **multi‑host** virtual L2 network (typically VXLAN) that spans multiple Docker hosts. It’s used primarily in **Docker Swarm mode** so services/containers scheduled on different nodes can communicate with **container‑level addressing** and service discovery.

**Characteristics & use cases:**

*   **Service discovery & load balancing**: Swarm’s built‑in DNS resolves **service names** to VIPs or tasks.
*   **Secure by default**: Can enable IPsec/encapsulation encryption in some setups.
*   **Multi‑host microservices**: Ideal when you deploy services across nodes and need them to talk as if on one network.
*   **Kubernetes note**: In K8s, overlay‑like behavior is provided by **CNI plugins** (Flannel, Calico, Cilium) rather than Docker overlay networks.

Basic flow:

```bash
docker swarm init
docker network create -d overlay --attachable appmesh
docker service create --name api --network appmesh repo/api:1.0
docker service create --name web --network appmesh -p 8080:80 repo/web:1.0
```

***

### Quick extras tailored for you

*   **Debug networking quickly:**
    ```bash
    docker run --rm -it --network appnet nicolaka/netshoot sh
    # tools: dig, curl, tcpdump, iproute2, etc.
    ```
*   **Prefer volumes for DB/state**; avoid heavy write paths on CoW layers.
*   **Healthchecks** + `--restart=always` (or orchestrator probes) help self‑heal transient issues.

Love the momentum—here are crisp, interview‑ready answers for **51–70**, with hands‑on examples and real‑world tips tailored for DevOps/SRE work.

***

## 51) How do you create a user in a Dockerfile and run a container as non‑root? (`USER`)

Create a user/group during build, set file ownership, and switch to it:

```dockerfile
FROM alpine:3.20

# Create a non-root user/group (UID/GID 10001)
RUN addgroup -g 10001 appgrp && adduser -D -u 10001 -G appgrp appuser

WORKDIR /app
# Copy files owned by that user
COPY --chown=appuser:appgrp . /app

# Drop privileges
USER appuser

ENTRYPOINT ["./app"]
```

**Tips**

*   Use `--chown` on `COPY` to avoid root-owned files.
*   Ensure the runtime user has needed permissions (log dirs, sockets, etc.).
*   In Debian/Ubuntu base, use `useradd`/`groupadd`.

***

## 52) Why avoid running processes as root inside containers?

*   **Least privilege**: Reduces impact of vulnerabilities or escape attempts.
*   **Volume safety**: Root in container may map to root on host; write/permission mistakes can harm host paths.
*   **Compliance & policy**: Many orgs require `runAsNonRoot`.
*   **Accidental damage**: Scripts or tools running as root can `chown` or delete critical files.
*   **Defense in depth**: Containers rely on kernel isolation; don’t rely solely on it—reduce process privilege too.

***

## 53) How do you pass build‑time arguments to Docker build? (`ARG`)

Use `ARG` in your Dockerfile and `--build-arg` at build time. `ARG` values are **not persisted** in the final image (unlike `ENV`).

```dockerfile
ARG BUILD_DATE
ARG VCS_REF
RUN echo "Built on ${BUILD_DATE} from ${VCS_REF}"

# Default value
ARG NODE_ENV=production
RUN echo "Node env: ${NODE_ENV}"
```

Build:

```bash
docker build \
  --build-arg BUILD_DATE="$(date -Iseconds)" \
  --build-arg VCS_REF="$(git rev-parse --short HEAD)" \
  -t repo/app:1.0 .
```

> For secrets (tokens, SSH keys), prefer **BuildKit** features (`--secret`, `--ssh`) instead of `ARG`.

***

## 54) How to use multi‑stage builds in Dockerfile and why?

**Multi‑stage builds** compile or package in a “builder” stage and copy only the final artifacts into a slim runtime stage. Benefits:

*   **Smaller images**, faster pulls.
*   **Reduced attack surface** (no compilers, package managers).
*   **Cleaner SBOMs** and licensing.

Example (Go):

```dockerfile
# Builder
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY . .
RUN go build -o app

# Runtime
FROM alpine:3.20
WORKDIR /app
COPY --from=build /src/app /usr/local/bin/app
USER 10001
ENTRYPOINT ["app"]
```

***

## 55) What is `docker-compose` and basic use‑case?

**Docker Compose** defines and runs **multi‑container applications** with a YAML file (`docker-compose.yml`). Use it for local dev, integration tests, and simple multi‑service setups (web + db + cache) with networks and volumes.

> Modern Docker uses the plugin command **`docker compose`** (space). The legacy Python tool is `docker-compose`.

***

## 56) How do you define services in a `docker-compose.yml` file?

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - pgdata:/var/lib/postgresql/data

  api:
    build: ./api
    env_file: api/.env
    depends_on:
      - db
    ports:
      - "8080:8080"
    networks:
      - appnet

  web:
    image: nginx:1.25
    volumes:
      - ./site:/usr/share/nginx/html:ro
    depends_on:
      - api
    ports:
      - "80:80"
    networks:
      - appnet

networks:
  appnet:

volumes:
  pgdata:
```

***

## 57) How do you bring up services with Docker Compose? (`docker-compose up`)

```bash
# With the modern plugin
docker compose up

# Legacy command still common
docker-compose up
```

Add `--build` to force rebuilds:

```bash
docker compose up --build
```

***

## 58) How do you run Compose in detached mode? (`docker-compose up -d`)

```bash
docker compose up -d
# or
docker-compose up -d
```

View logs later:

```bash
docker compose logs -f
```

***

## 59) How do you scale a service with docker-compose? (`docker-compose up --scale`)

```bash
docker compose up -d --scale api=3
# legacy:
docker-compose up -d --scale api=3
```

**Notes**

*   If a service publishes a host port (e.g., `ports: "8080:8080"`), only **one** replica can bind that port on the host. Use a **reverse proxy/load balancer** or remove host port publishing and let another frontend expose a single port.
*   Compose balances name lookup inside the app network; clients should connect via the proxy/front-end.

***

## 60) How do you view published ports from a container? (`docker port`)

```bash
docker port web
# Example output:
# 80/tcp -> 0.0.0.0:80
# 443/tcp -> 0.0.0.0:8443
```

Or via `docker ps --format '{{.Names}}\t{{.Ports}}'`.

***

## 61) What is immutable infrastructure in context of containers?

Treat images and deployments as **immutable artifacts**:

*   **Don’t mutate containers in place** (no manual patching).
*   Make changes via **new image builds** and **redeploy**.
*   Configuration via env vars or mounted config (12‑factor).
*   Benefits: **consistency**, **reproducibility**, easy **rollback**, simpler **drift control**.

***

## 62) How to inspect running processes inside a container? (`docker top`)

```bash
docker top app
# With options (depends on platform ps)
docker top app -o pid,user,etime,cmd
```

For deeper inspection:

```bash
docker exec -it app ps aux
```

***

## 63) How to export/import an image or container?

(`docker save` / `docker load`, `docker export` / `docker import`)

*   **Images (preserve layers & metadata)**:
    ```bash
    docker save repo/app:1.0 -o app.tar
    docker load -i app.tar
    ```
*   **Containers (flatten filesystem, lose image metadata)**:
    ```bash
    docker export app -o appfs.tar
    docker import appfs.tar repo/app:flat
    ```

**Difference**: `save/load` is for **images**; `export/import` creates a **single-layer** image from a container’s filesystem (no history, no config).

***

## 64) How to perform image cleanup on a host? (`docker system prune`)

```bash
# Remove stopped containers, unused networks, dangling images, build cache
docker system prune -f

# Also remove unused images (not just dangling)
docker image prune -a -f

# Remove volumes (careful: data loss)
docker volume prune -f

# Build cache cleanup (BuildKit)
docker builder prune -f
```

**Tip**: Check usage first:

```bash
docker system df
```

***

## 65) What is `docker login` and why is it necessary?

`docker login` authenticates you against a registry so you can **pull private images** or **push** images. Credentials are stored via **credential helpers** (keychain, wincred, pass).

```bash
docker login myacr.azurecr.io
# For ACR (Azure):
az acr login --name myacr
```

Necessary because registries require authentication and authorization for non-public access and for push operations.

***

## 66) How do you sign images and what is image signing conceptually?

**Concept**: Image signing attaches **cryptographic signatures** to images (by digest) to prove **provenance**, **integrity**, and support **policy enforcement** (only run trusted images).

Common approaches:

*   **Docker Content Trust (DCT)** / Notary: enable signing & verification.
    ```bash
    export DOCKER_CONTENT_TRUST=1
    docker push repo/app:1.0   # signs
    docker pull repo/app:1.0   # verifies
    ```
*   **Cosign (sigstore)**: modern, keyless (OIDC) or key‑pair signatures.
    ```bash
    # Keyed example
    cosign generate-key-pair
    cosign sign myacr.azurecr.io/repo/app:1.0
    cosign verify myacr.azurecr.io/repo/app:1.0
    ```

**Policies**: Use admission controls (K8s, Gatekeeper/Kyverno) to enforce “signed only” images.

***

## 67) How do layered caches speed up Docker builds?

*   Each Dockerfile instruction produces a **content-addressed layer**.
*   If inputs (command + files) **haven’t changed**, Docker reuses the layer from cache → **dramatically faster builds**.
*   **Ordering matters**: Move frequently changing files (e.g., app source) **later**; keep dependency steps **earlier**.
*   With **BuildKit**, caching improves further; use **mount=cache** for tool caches.

Example:

```dockerfile
# Good cache pattern for Node
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
```

***

## 68) What is the difference between layered images and flattening an image?

*   **Layered images**
    *   Multiple immutable layers; shared across images; **cache‑friendly**; smaller network/storage due to reuse.
*   **Flattened images**
    *   Single layer (via `docker export`/`import` or squash). Loses build history and layer reuse; **not cache‑friendly**; can be slightly simpler for legacy environments.

**Trade‑off**: Prefer layered images for modern CI/CD and registries. Flatten only for edge cases (legacy systems, specific distribution constraints). Note: `docker build --squash` exists in experimental modes but is rarely needed.

***

## 69) How to tag and push images to a private registry?

```bash
# Tag for registry
docker tag repo/app:1.0 myacr.azurecr.io/repo/app:1.0

# Login and push
az acr login --name myacr          # Azure example
docker push myacr.azurecr.io/repo/app:1.0

# ECR (AWS) example
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.ap-south-1.amazonaws.com
docker tag repo/app:1.0 <acct>.dkr.ecr.ap-south-1.amazonaws.com/repo/app:1.0
docker push <acct>.dkr.ecr.ap-south-1.amazonaws.com/repo/app:1.0
```

***

## 70) How to configure Docker to use a private registry (insecure registry config)?

If your private registry uses **HTTP** or self‑signed certs (not recommended for production), configure the daemon:

**Linux (`/etc/docker/daemon.json`):**

```json
{
  "insecure-registries": [
    "myregistry.local:5000"
  ],
  "registry-mirrors": []
}
```

Apply changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

**With TLS (preferred)**: Place CA certs under:

    /etc/docker/certs.d/<registry-hostname>/ca.crt

Docker will trust the registry over HTTPS.

> **Security note**: Insecure registries send credentials and images over plain HTTP; prefer HTTPS with proper certificates.

***

### Handy extras & best practices

*   **Run as non‑root** + add a **`USER`** early to catch permission issues during build.
*   **Compose scaling**: Use a reverse proxy (Nginx, Traefik) or remove host port mapping and expose only the proxy.
*   **Cosign + policy**: Combine signing with Kubernetes admission policies for strong supply‑chain guarantees.
*   **Cleanup cadence**: Use `docker system df` before pruning; scripts can safely remove exited containers and dangling images.

***

Absolutely—here are crisp, interview‑ready answers with hands‑on tips for **71–100**.

***

## 71) How do you secure a private registry with TLS? (concept)

*   **Get a valid TLS cert** for your registry hostname (public CA or internal CA).
*   **Terminate TLS** either:
    *   **Directly in the registry** (Docker Distribution/Harbor/Quay) by configuring `tls: { certificate, key }`, or
    *   **Behind a reverse proxy** (Nginx/Traefik/HAProxy) that serves HTTPS and proxies to the registry on HTTP.
*   **Client trust**:
    *   For public CA: works out of the box.
    *   For **private/self‑signed CA**: place CA at `/etc/docker/certs.d/<registry-host>/ca.crt` on each client.
*   **Avoid “insecure registries”** (HTTP) except in isolated labs.

***

## 72) Typical image naming conventions (`registry/repo:tag`)

    [registry[:port]/]namespace/repository[:tag]

Examples:

*   `nginx:1.25` (Docker Hub library)
*   `myacr.azurecr.io/team/api:1.4.2`
*   `ghcr.io/org/tool:sha-abc123`
    **Conventions**: all lowercase, **semantic tags** (`1.4.2`, `1.4`, `1`), env tags (`-dev`, `-prod`), and optionally **arch** (`-amd64`, `-arm64`) if not using multi‑arch manifests.

***

## 73) Force rebuild all layers (`docker build --no-cache`)

```bash
docker build --no-cache -t repo/app:debug .
```

Disables layer cache; every instruction runs from scratch.

***

## 74) How to limit build context size and why?

*   **Why**: smaller builds, faster uploads to daemon/remote builder, fewer cache invalidations, avoid **leaking secrets**.
*   **How**:
    *   Use **`.dockerignore`** to exclude large/secret files (`node_modules`, `.git`, `*.log`, `.env`).
    *   Build from a **minimal subdirectory** when possible.
    *   Avoid `COPY . .` at root; **copy only what’s needed**.
    *   Use multi‑stage builds to avoid bundling build tools/artifacts.

***

## 75) What is Docker Desktop (purpose & features)?

A desktop app providing:

*   Local VM‑based Docker Engine, **Docker Compose**, and optional **Kubernetes single‑node cluster**.
*   GUI to view images/containers/volumes/networks, **Dev Environments**, extensions.
*   **Context switching** to remote engines, file sharing from host, networking integration.

***

## 76) Container runtime vs orchestrator

*   **Runtime**: creates/runs containers on a node (e.g., **containerd**, **runc**, **CRI‑O**, Docker Engine).
*   **Orchestrator**: manages **multi‑node scheduling, scaling, service discovery, networking, health, and rollouts** (e.g., **Kubernetes**, **Docker Swarm**).

***

## 77) Minimal base image (e.g., `alpine`) and trade‑offs

*   **Alpine**: tiny, uses **musl** libc; good for small images.
    *   Trade‑offs: musl vs glibc differences, some binaries expect glibc; fewer tools, potential DNS/TLS quirks.
*   **Distroless**: only app + runtime libs, **no shell**—smaller attack surface.
*   **Scratch**: truly empty; requires **statically linked** binaries.
*   Choose based on compatibility, tooling needs, and security posture.

***

## 78) Inspect image filesystem without “running” the container

*   Create but don’t start:
    ```bash
    id=$(docker create repo/app:1.0)
    docker cp "$id":/path/inside ./local
    docker rm "$id"
    ```
*   Explore layers: `docker history`, `docker image save` then untar.
*   Use **`dive`** (third‑party) to analyze layers and sizes.
*   (If acceptable) temporary shell: `docker run --rm --entrypoint sh -it image` (starts but useful for ad‑hoc).

***

## 79) What is `docker swarm` and how it differs from Kubernetes?

*   **Docker Swarm**: Docker’s built‑in orchestrator—**simple** ops, `docker swarm init`, `docker service`, overlay networks, secrets, rolling updates.
*   **Kubernetes**: richer ecosystem & features (CRDs, operators, advanced scheduling, network policies), broader adoption. More complex to operate, but more extensible/portable.

***

## 80) Redirect container logs to files or syslog (driver concept)

Docker’s **logging drivers** route stdout/stderr:

*   Default: `json-file`.
*   Others: `syslog`, `journald`, `fluentd`, `gelf`, `awslogs`, `splunk`, etc.

```bash
docker run --log-driver=syslog --log-opt syslog-address=udp://1.2.3.4:514 app
```

In Compose:

```yaml
logging:
  driver: syslog
  options:
    syslog-address: "udp://1.2.3.4:514"
```

In production, prefer **shipping logs** to a central system vs writing files inside containers.

***

## 81) Role of `dockerd` **user namespaces** (basic security)

**Userns‑remap** maps container user IDs to **unprivileged UIDs on the host**. Even if the container runs as root (UID 0), on the host it maps to, e.g., UID 100000—reducing impact of escapes/volume writes.

*   Enable in `/etc/docker/daemon.json`:

```json
{ "userns-remap": "default" }
```

Consider volume ownership mappings and plugins compatibility.

***

## 82) Mount a host device into a container (`--device`)

```bash
docker run --device /dev/video0:/dev/video0:mwr app
```

Grants the container access to the host device node. Use **minimal permissions**; avoid `--privileged`.

***

## 83) Ephemeral vs persistent storage in containers

*   **Ephemeral**: the container’s **writable layer**—deleted when the container is removed; CoW semantics; not for state.
*   **Persistent**: **volumes** (Docker‑managed) or **bind mounts** (host paths) survive container lifecycle. Use volumes for production data; bind mounts for dev or host‑integration cases.

***

## 84) Pass secrets to containers securely (runtime concept)

*   Prefer **orchestrator secrets**:
    *   Docker Swarm **`docker secret`** (mounted as files).
    *   **Kubernetes Secrets** (mounted as files/env; consider encryption at rest + CSI).
*   **External secret stores**: Azure Key Vault (CSI driver), HashiCorp Vault sidecars/agents.
*   **Avoid** baking secrets in images, passing via build args, or committing `.env` to VCS.
*   Runtime env vars are convenient but **less secure** (appear in inspect/logs); mounted **files** with tight perms are safer.

***

## 85) `docker-compose.override.yml` and usage

Compose automatically merges `docker-compose.override.yml` with the base file—great for **dev‑only overrides** (ports, bind mounts, debug env).

*   Applied by default with `docker compose up`.
*   You can specify alternates with multiple `-f` flags for different envs.

***

## 86) Display Docker version and environment info

```bash
docker version   # client & server versions, API compatibility
docker info      # runtime details, storage driver, cgroup, registries, etc.
```

***

## 87) Use `.dockerignore` and why

Excludes files from **build context**:

*   Reduces transfer size and build time.
*   Prevents cache invalidation churn.
*   Avoids leaking secrets.
    Example `.dockerignore`:

<!---->

    .git
    node_modules
    dist
    *.log
    .env
    Dockerfile*

***

## 88) Image **digest (sha256)** vs **tag**

*   **Tag**: human‑friendly alias (mutable).
*   **Digest**: **content‑addressed**, immutable identifier for an exact image.
    Use in deployments to pin:

```bash
docker pull repo/app@sha256:<digest>
```

***

## 89) Inspect labels and why use them (`LABEL` in Dockerfile)

*   Add metadata:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/acme/api" \
      org.opencontainers.image.version="1.4.2" \
      maintainer="platform@acme.com"
```

*   Inspect:

```bash
docker inspect repo/app:1.4.2 --format '{{json .Config.Labels}}' | jq .
```

**Why**: ownership, provenance, automation, policy, billing, selection (`docker ps --filter label=...`).

***

## 90) Process supervisor inside a container—what & when?

A **supervisor** (e.g., **tini**, **dumb‑init**, **s6**, **supervisord**) acts as **PID 1** to:

*   Properly **forward signals** and **reap zombies**.
*   Manage **multiple child processes**.
    Use when your app doesn’t handle PID 1 duties or you truly need multiple processes. For simple apps, `tini` is a lightweight choice.

***

## 91) Foreground PID 1 vs using a supervisor

*   **App as PID 1**: simpler; ensure it handles **SIGTERM/SIGINT** and reaps zombies (or use `--init`).
*   **Supervisor as PID 1**: handles signals/reaping and can manage multiple processes; adds complexity but improves robustness.

***

## 92) Running multiple processes in a container (best practices)

*   **Prefer one process per container**; compose services using separate containers.
*   If you must:
    *   Use a **supervisor** (s6, supervisord) or
    *   A lightweight **init** (`tini`, `--init`) and a wrapper script that starts/monitors children.
*   In Kubernetes, consider **sidecars** (e.g., log shippers, proxies) as separate containers in the same pod.

***

## 93) `--restart` policy options

*   `no` (default): don’t auto‑restart.
*   `on-failure[:N]`: restart on non‑zero exit (optional max retries).
*   `always`: always restart if it stops (including daemon restart).
*   `unless-stopped`: like `always` but not restarted if **manually** stopped.

```bash
docker run --restart=unless-stopped -d app
```

Note: Docker **doesn’t** restart on `unhealthy`—that’s for Swarm/K8s logic.

***

## 94) Set logging driver per container (`--log-driver`)

```bash
docker run --log-driver=fluentd \
  --log-opt fluentd-address=127.0.0.1:24224 \
  --log-opt tag="app.{{.Name}}" app
```

Compose:

```yaml
logging:
  driver: gelf
  options:
    gelf-address: "udp://1.2.3.4:12201"
    tag: "service.api"
```

***

## 95) Common container security principles

*   **Least privilege**: run as **non‑root**, **drop capabilities** (`--cap-drop=ALL; --cap-add=NET_BIND_SERVICE` as needed).
*   **Read‑only** root filesystem, write via volumes; avoid `--privileged`.
*   **User namespaces**, **seccomp**, **AppArmor/SELinux** profiles.
*   **Minimal images**; patch frequently; **scan** images (SCA + vuln scanning).
*   **Secrets management** (no secrets in images); avoid mounting `docker.sock`.
*   **Network segmentation**; only necessary ports exposed.
*   **Resource limits** to mitigate DoS.
*   **Image signing** (Cosign/DCT) + policy enforcement.

***

## 96) Container immutability & declarative deployment

*   Build **immutable images**; never patch running containers.
*   Deploy via **declarative manifests** (Compose, Helm/K8s YAML, Terraform).
*   Use **GitOps** (PR‑based changes, automated reconciler).
*   Enables auditability, reproducibility, and fast rollbacks.

***

## 97) What is **BuildKit** and how it improves Docker builds?

Next‑gen builder providing:

*   **Parallel** builds and better caching (including **remote cache**).
*   **`--secret`** and **`--ssh`** mounts for secure build‑time access.
*   **`RUN --mount=type=cache`** for dependency caches.
*   Sane output, improved performance, multi‑platform with **buildx**.
    Enable:

```bash
export DOCKER_BUILDKIT=1
docker build -t repo/app:1.0 .
# or use buildx for multi-arch:
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.0 --push .
```

***

## 98) Significance of the `latest` tag and why avoid in prod

*   `latest` is just another **mutable tag**; **not special**.
*   Risks: non‑deterministic deploys, surprise upgrades, hard rollbacks.
*   Prefer **pinned versions** (semver) and/or **digests** for immutability.

***

## 99) Export container environment variables to host (and why avoid)

*   You can **read** envs:
    ```bash
    docker inspect app --format '{{json .Config.Env}}' | jq -r '.[]'
    docker exec app env
    ```
*   “Exporting” to host shell typically involves unsafe hacks (`eval $(...)`)—**avoid** for security and reproducibility.
*   **Best practice**: manage env in **versioned `.env` files**, Compose/K8s manifests, or secret stores; inject at runtime, don’t scrape from containers.

***

## 100) What is OCI and how does Docker relate to the OCI image spec?

*   **OCI (Open Container Initiative)** defines open standards:
    *   **OCI Image Spec** (format/layout, manifests)
    *   **OCI Runtime Spec** (how to run a container—**runc** is a reference implementation)
    *   **OCI Distribution Spec** (registry API)
*   Docker **contributed** its image format/runtime to OCI; modern Docker images conform to **OCI**. Runtimes like **containerd**, **CRI‑O**, and orchestrators like **Kubernetes** rely on these standards.

***

Awesome—continuing with interview‑ready answers and hands‑on examples for **101–110**, tailored for day‑to‑day DevOps/SRE work.

***

## 101) Explain Docker image lifecycle (build, tag, push, pull, run)

**Lifecycle steps:**

1.  **Build** – Create an image from a Dockerfile and build context.
    ```bash
    docker build -t repo/api:1.0 .
    ```
2.  **Tag** – Add a new name (repository:tag) to the image ID.
    ```bash
    docker tag repo/api:1.0 myacr.azurecr.io/repo/api:1.0
    ```
3.  **Push** – Upload the image to a registry.
    ```bash
    az acr login --name myacr
    docker push myacr.azurecr.io/repo/api:1.0
    ```
4.  **Pull** – Download the image on another host/CI agent.
    ```bash
    docker pull myacr.azurecr.io/repo/api:1.0
    ```
5.  **Run** – Start a container from the image.
    ```bash
    docker run -d -p 8080:8080 --name api myacr.azurecr.io/repo/api:1.0
    ```

> Under the hood: the image is a stack of immutable layers; containers add a writable CoW layer at runtime.

***

## 102) How do you use `docker-compose` for multi‑container app networking?

Compose creates **user‑defined bridge networks**; services get **DNS names** equal to the service name. Containers in the same network can reach each other by `http://service-name:port`.

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    networks: [appnet]

  api:
    build: ./api
    environment:
      DB_HOST: db
    depends_on:
      - db
    networks: [appnet]

  web:
    image: nginx:1.25
    depends_on:
      - api
    ports:
      - "8080:80"    # published to host
    networks: [appnet]

networks:
  appnet:
    driver: bridge
```

*   **Service discovery**: `api` can reach `db:5432`.
*   **Isolation**: multiple networks can be defined; a service can attach to several.
*   **Aliases** (optional): add alternate names via `networks: { appnet: { aliases: ["backend"] } }`.

***

## 103) How to optimize Dockerfile for smaller images? (layer ordering, slim base)

**Key tactics:**

*   Use **slim/minimal base** (`alpine`, `distroless`, `scratch`) where compatible.
*   **Multi‑stage builds**: compile in builder, copy final artifacts only.
*   **Order for cache**: copy manifests (e.g., `package.json`) before app source to cache dependency installs.
*   Combine `RUN` steps and **clean up** in the same layer.
*   Avoid `apt-get update` without **`--no-install-recommends`** and remove package lists.

Example (Node):

```dockerfile
# Builder
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Runtime
FROM alpine:3.20
WORKDIR /app
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
USER 10001
ENTRYPOINT ["node"]
CMD ["./dist/server.js"]
```

***

## 104) What is `dive` and how can it help analyze image layers? (concept)

**`dive`** is a CLI tool that:

*   Visualizes image **layers**, sizes, and file changes per layer.
*   Shows **efficiency score** and highlights **bloat** (large files added repeatedly).
*   Helps identify **cache‑unfriendly** instruction ordering.

Usage (local):

```bash
dive repo/api:1.0
```

***

## 105) How to use multi‑stage builds to remove build‑time dependencies?

Compile/package in a builder stage; copy only the compiled artifact into a clean runtime stage—**no compilers or package managers** remain.

Example (Go):

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY . .
RUN go build -o app

FROM scratch
COPY --from=build /src/app /app
USER 10001
ENTRYPOINT ["/app"]
```

Benefits: smaller image, reduced attack surface, faster pulls, cleaner SBOMs.

***

## 106) How to cache credentials securely during builds (best practice concept)?

*   **Avoid** putting credentials into `ARG`/`ENV` (they end up in image metadata or layer history).
*   Use **private registries** with `docker login` for base images.
*   Leverage **BuildKit** secret mounts for package manager tokens (npm, pip, maven).
*   Use **SSH forwarding** for private Git (no secrets stored in layers).
*   Ensure CI injects secrets via **secure variables**; never commit `.npmrc`, `.pypirc`, or `.netrc` with tokens.

***

## 107) How to mount secrets at build‑time without leaving them in image (BuildKit secrets)?

Enable BuildKit and use `RUN --mount=type=secret`:

```bash
# Build command
docker build \
  --secret id=npm_token,env=NPM_TOKEN \
  -t repo/app:1.0 .

# Dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
# Mount secret only during RUN; not persisted
RUN --mount=type=secret,id=npm_token \
    sh -c 'echo "//registry.npmjs.org/:_authToken=$(cat /run/secrets/npm_token)" > ~/.npmrc && npm ci'
COPY . .
RUN npm run build
```

For private Git clones:

```dockerfile
# Requires: docker build --ssh default
RUN --mount=type=ssh git clone git@github.com:org/private-repo.git
```

> Secrets are available **ephemerally** during the `RUN` step and **not** committed to image layers.

***

## 108) How to use `ONBUILD` instruction and why be careful with it?

`ONBUILD` queues a trigger in a base image that runs when **another Dockerfile uses it as `FROM`**.

Example:

```dockerfile
# In a base image
ONBUILD COPY . /app
ONBUILD RUN npm ci
```

**Be careful**:

*   Hidden side effects for downstream builds—**surprising behavior**.
*   Hard to control build order/caching in child Dockerfiles.
*   Modern guidance: **avoid ONBUILD**; instead document a **base image + sample Dockerfile** or provide **multi‑stage templates**.

***

## 109) How to create a healthcheck that verifies app readiness?

Use a **local**, **fast** endpoint for readiness (e.g., `/readyz`) and a shallow liveness (`/healthz`). In Docker:

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20
RUN apk add --no-cache curl
COPY app /usr/local/bin/app
ENTRYPOINT ["/usr/local/bin/app"]

# Readiness check (fast, local)
HEALTHCHECK --interval=30s --timeout=3s --retries=3 --start-period=20s \
  CMD ["curl", "-fsS", "http://localhost:8080/readyz"]
```

In Compose, you can rely on health status:

```yaml
services:
  api:
    image: repo/api:1.0
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/readyz"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 20s
  web:
    image: nginx:1.25
    depends_on:
      api:
        condition: service_healthy
```

**Tips**:

*   Keep readiness shallow: check **listener bound** + **event loop**; avoid deep external checks.
*   On graceful shutdown, **fail readiness** immediately, keep liveness passing while draining.

***

## 110) How to use `docker network create` and custom networks?

Create **user‑defined bridge** networks (better isolation and DNS) and attach containers:

```bash
# Create a network with custom subnet
docker network create --driver bridge \
  --subnet 172.22.0.0/16 --gateway 172.22.0.1 appnet

# Run containers on it
docker run -d --name db --network appnet postgres:16
docker run -d --name api --network appnet -e DB_HOST=db repo/api:1.0
docker run -d --name web --network appnet -p 8080:80 nginx:1.25

# Connect/disconnect an existing container
docker network connect appnet some-container
docker network disconnect appnet some-container

# Inspect network (subnets, endpoints, aliases)
docker network inspect appnet
```

Other drivers:

*   `overlay` (Swarm, multi‑host networking)
*   `host` (share host stack, no port mapping)
*   `none` (no networking)
*   `macvlan` (containers get their own MAC/IP on the LAN)

***

### Quick add‑ons you might find handy

*   **BuildKit defaults**:
    ```bash
    export DOCKER_BUILDKIT=1
    docker build -t repo/app:1.0 .
    ```
*   **Compose profiles** (dev vs prod):
    ```yaml
    services:
      api:
        profiles: ["dev"]
    ```
*   **Clean up safely**:
    ```bash
    docker ps -aq --filter status=exited | xargs -r docker rm
    docker image prune -f
    docker volume prune -f
    ```

Absolutely—here are crisp, interview‑ready answers with hands‑on examples for **111–120**, tailored for DevOps/SRE work.

***

## 111) How to attach containers to multiple networks?

You can connect a running container to **additional networks** (and disconnect later):

```bash
# Create networks
docker network create appnet
docker network create observability

# Start container on one network
docker run -d --name api --network appnet repo/api:1.0

# Attach to another
docker network connect observability api

# Optional: assign alias on that network
docker network connect --alias api-metrics observability api

# Disconnect
docker network disconnect observability api
```

**Compose** (attach to multiple networks):

```yaml
services:
  api:
    image: repo/api:1.0
    networks:
      appnet:
      observability:
        aliases: ["api-metrics"]

networks:
  appnet:
  observability:
```

***

## 112) How to inspect network details (`docker network inspect`)?

Get subnets, endpoints, IP/MACs, and options:

```bash
docker network inspect appnet
# Pretty-print selected fields
docker network inspect appnet \
  --format '{{json .IPAM.Config}}' | jq .
```

***

## 113) How do containers resolve DNS by default? (embedded DNS in Docker)

*   On **user‑defined bridge networks**, Docker provides an **embedded DNS** at **`127.0.0.11`** that resolves **container names** and **network aliases** to container IPs on that network.
*   On the **default `bridge`** network, name resolution between containers is **not automatic** (historically required `--link`). Containers will still resolve **external DNS** using the host’s resolvers.
*   On **overlay networks** (Swarm), embedded DNS also resolves **service names**.

Check inside a container:

```bash
cat /etc/resolv.conf
# Expect nameserver 127.0.0.11 on user-defined networks
```

***

## 114) How to set custom DNS for containers (`--dns`)?

Per container:

```bash
docker run -d --name api \
  --dns 1.1.1.1 --dns 8.8.8.8 \
  --dns-search mycorp.local \
  repo/api:1.0
```

Global defaults in **`/etc/docker/daemon.json`**:

```json
{
  "dns": ["1.1.1.1", "8.8.8.8"]
}
```

Then reload:

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

***

## 115) How to control container restart behavior on failure?

Use **`--restart`** policy:

```bash
# Restart on non-zero exit (up to N retries)
docker run --restart=on-failure:5 -d repo/job:1.0

# Always restart (including on daemon restart)
docker run --restart=always -d repo/api:1.0

# Restart unless manually stopped
docker run --restart=unless-stopped -d repo/worker:1.0
```

> Note: Docker **does not** automatically restart containers marked `unhealthy` by healthchecks; orchestration (Swarm/K8s) handles that.

***

## 116) How to manage container log rotation at Docker daemon level? (`daemon.json` logging-opts)

Configure **log driver** and rotation globally:

`/etc/docker/daemon.json`

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "mode": "non-blocking",
    "max-buffer-size": "8m"
  }
}
```

Apply:

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

Per container override:

```bash
docker run -d --log-driver=json-file \
  --log-opt max-size=10m --log-opt max-file=5 \
  repo/api:1.0
```

For centralized logging:

```bash
docker run --log-driver=fluentd --log-opt fluentd-address=127.0.0.1:24224 repo/api:1.0
```

***

## 117) How to set resource **reservations** vs **limits** for containers? (`--memory-reservation`)

*   **Hard limit** (enforced): `--memory`
*   **Soft reservation** (best effort; throttling under pressure): `--memory-reservation`

```bash
docker run -d --name api \
  --memory=512m \
  --memory-reservation=256m \
  --cpus=1.0 \
  --cpu-shares=512 \
  repo/api:1.0
```

Other useful flags:

*   **CPU pinning**: `--cpuset-cpus="0,2"`
*   **PIDs**: `--pids-limit=200`
*   **I/O throttling**: `--device-read-bps /dev/sda:10mb` (needs correct device path)

***

## 118) How to configure cgroups and namespace limits for container isolation? (concept)

**Namespaces** isolate:

*   **PID** (process IDs), **NET** (network stack), **MNT** (filesystems), **IPC**, **UTS** (hostname), **USER** (UID/GID mapping).

**cgroups** enforce limits:

*   **CPU** (`--cpus`, `--cpu-quota/--cpu-period`, `--cpu-shares`)
*   **Memory** (`--memory`, `--memory-reservation`, `--memory-swap`)
*   **I/O** (`--device-read-bps`, `--device-write-bps`, `--blkio-weight`)
*   **Processes** (`--pids-limit`)

Examples:

```bash
docker run -d \
  --cpus=0.5 --cpuset-cpus="1-2" \
  --memory=256m --memory-swap=512m \
  --pids-limit=150 \
  repo/worker:1.0
```

Security profiles:

*   **seccomp**: `--security-opt seccomp=/path/profile.json`
*   **AppArmor/SELinux**: `--security-opt apparmor=profile` or `--security-opt label:type:...`
*   **Capabilities**: `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`

***

## 119) How to enable user namespaces to map container root to non‑root on host?

Enable **userns‑remap** so container **UID 0** maps to an **unprivileged host UID range**:

`/etc/docker/daemon.json`

```json
{
  "userns-remap": "default"
}
```

Then:

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

**Considerations:**

*   Host paths/volumes may need ownership remapping (UID/GID from `/etc/subuid`, `/etc/subgid`).
*   Some plugins/drivers and privileged operations aren’t compatible.
*   Improves defense‑in‑depth even if containers run as root inside.

***

## 120) What are rootless containers and when to use them?

**Rootless Docker** (or Podman) runs the daemon and containers **without root privileges**:

**Why/When:**

*   Multi‑tenant developer machines or hardened environments where root is prohibited.
*   Reduce risk of host compromise.

**Setup (Docker):**

*   Install rootless extras, then:

```bash
dockerd-rootless-setuptool.sh install
# Start daemon in user session
systemctl --user start docker
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

**Limitations:**

*   No `--privileged` or direct device mounts; limited capabilities.
*   Port binding <1024 needs workarounds (e.g., rootlesskit) or use higher ports.
*   Networking uses **slirp4netns** (slower NAT); some storage/network drivers unavailable.
*   Some cgroup features require cgroup v2 & user delegation.

**Alternatives:**

*   **Podman rootless** (daemonless) is commonly used for rootless workflows.

***

### Quick add‑ons (tailored for you)

*   **Best practice bundle for secure runtime**:
    ```bash
    docker run -d \
      --read-only \
      --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
      --pids-limit=200 --cpus=1 --memory=512m \
      --user 10001 \
      -v appdata:/var/lib/app \
      -p 8080:8080 \
      repo/app:1.0
    ```
*   **Network troubleshooting**:
    ```bash
    docker run --rm -it --network appnet nicolaka/netshoot sh
    # dig api, curl http://api:8080, tcpdump -i any
    ```

Awesome—continuing with interview‑ready, hands‑on answers for **121–140**, tailored for real DevOps/SRE workflows (and with a few Azure/ACR examples where useful).

***

## 121) Sign and verify images using Notary / Docker Content Trust (concept)

**Docker Content Trust (DCT)** uses **Notary v1** to sign images and verify publisher identity + integrity.

*   **Enable trust** (client-side):
    ```bash
    export DOCKER_CONTENT_TRUST=1
    ```
*   **Sign on push**:
    ```bash
    docker push myregistry/repo/app:1.0   # signs if DCT enabled
    ```
*   **Verify on pull**:
    ```bash
    docker pull myregistry/repo/app:1.0   # verifies signatures
    ```
*   **Keys**: root key (offline) + repository keys (online). Manage via `notary` CLI or Docker key stores.
*   **Modern alternatives**: Many orgs now prefer **Sigstore Cosign** or **Notation (Notary v2)** with OCI‑native signatures.

***

## 122) Image scanning and tools (concept)

Scan images for known CVEs and misconfigurations:

*   **Open-source**: **Trivy**, **Grype**, **Clair**.
*   **Commercial**: **Aqua**, **Snyk**, **Anchore**, **Qualys**, **Tenable**.
*   **Cloud**: **Azure Container Registry (ACR)** integration via **Microsoft Defender for Cloud**, **Amazon ECR** built‑in scanning, **GCR/Artifact Registry** scanning.

Usage (Trivy example):

```bash
trivy image myacr.azurecr.io/repo/app:1.0 --severity HIGH,CRITICAL
```

Integrate in CI with **quality gates** (fail builds on critical findings).

***

## 123) Handle layer caching with frequently changing files

*   **Order instructions** to maximize cache:
    ```dockerfile
    COPY package*.json ./
    RUN npm ci --only=production
    COPY . .
    ```
*   Use **`.dockerignore`** to exclude noisy/large files (e.g., `node_modules/`, `.git/`, `dist/`, `*.log`, `.env`).
*   Split steps: copy **manifests early**, app source **later**.
*   Prefer **BuildKit** cache mounts for tooling (npm/maven/pip).

***

## 124) Share volumes between containers & manage permissions

*   Use **named volumes**:
    ```bash
    docker volume create appdata
    docker run -d --name api -v appdata:/var/lib/app repo/api:1.0
    docker run -d --name worker -v appdata:/var/lib/app repo/worker:1.0
    ```
*   **Permissions**:
    *   Run containers as the **same UID/GID** (`USER 10001`) or use `--user`.
    *   Initialize volume ownership (one‑off):
        ```bash
        docker run --rm -v appdata:/var/lib/app alpine chown -R 10001:10001 /var/lib/app
        ```
*   Avoid root‑owned data; prefer non‑root runtime users.

***

## 125) Back up and restore named volumes

**Backup**:

```bash
docker run --rm \
  -v appdata:/data \
  -v "$(pwd)":/backup \
  alpine sh -c 'cd /data && tar -czf /backup/appdata-backup.tgz .'
```

**Restore**:

```bash
docker run --rm \
  -v appdata:/data \
  -v "$(pwd)":/backup \
  alpine sh -c 'cd /data && tar -xzf /backup/appdata-backup.tgz'
```

***

## 126) Use `tmpfs` mounts for ephemeral data (`--tmpfs`)

Mount an in‑memory filesystem (cleared when container stops):

```bash
docker run -d \
  --tmpfs /tmp:rw,size=64m,mode=1777 \
  --tmpfs /run \
  repo/app:1.0
```

Compose:

```yaml
services:
  app:
    image: repo/app:1.0
    tmpfs:
      - /tmp:size=64m
```

***

## 127) Use `docker context` for multiple endpoints

Switch between local/remote engines (desktop, VM, remote host):

```bash
docker context ls
docker context create my-remote --docker "host=tcp://10.0.0.5:2376"
docker context use my-remote
docker info    # against remote
```

Contexts can store **TLS** settings and credentials; great for targeting different environments cleanly.

***

## 128) Docker behind corporate HTTP proxies

Set environment variables for **client** and **daemon**:

**Client** (shell):

```bash
export HTTP_PROXY=http://proxy.mycorp:8080
export HTTPS_PROXY=http://proxy.mycorp:8080
export NO_PROXY=localhost,127.0.0.1,.mycorp.local
```

**Daemon** (systemd drop‑in `/etc/systemd/system/docker.service.d/proxy.conf`):

```ini
[Service]
Environment="HTTP_PROXY=http://proxy.mycorp:8080"
Environment="HTTPS_PROXY=http://proxy.mycorp:8080"
Environment="NO_PROXY=localhost,127.0.0.1,.mycorp.local"
```

Reload:

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

Builds may need proxy **build args**:

```bash
docker build --build-arg HTTP_PROXY --build-arg HTTPS_PROXY -t repo/app:1.0 .
```

***

## 129) Containerized CI runners with Docker‑in‑Docker (dind) caveats

*   **Security**: `dind` often requires `--privileged`; broad host access risk.
*   **Performance**: nested overlay filesystems can be slow; cleanup is critical.
*   **Alternatives**:
    *   **Docker‑out‑of‑Docker** (mount host `docker.sock`)—simpler but grants host control.
    *   **Kaniko**, **BuildKit (buildx)**, **Podman**—rootless or daemonless builders for CI.
*   **If using dind**:
    *   Pin versions; enable **BuildKit**; set log/cleanup policies; guard network egress.

***

## 130) Build multi‑arch images (amd64/arm64) with `buildx`

```bash
docker buildx create --name multi --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myacr.azurecr.io/repo/app:1.0 \
  --push .
```

Requires QEMU emulation or native builders; `buildx` handles **multi‑platform** cross‑builds.

***

## 131) Push multi‑platform manifests to registry

The `buildx build --platform ... --push` creates a **manifest list** pointing to per‑arch images. Verify:

```bash
docker buildx imagetools inspect myacr.azurecr.io/repo/app:1.0
```

This publishes a single tag that works on **both amd64 and arm64** clients.

***

## 132) Programmatic authentication to registries (CI, service principals)

*   **ACR (Azure)**:
    ```bash
    az acr login --name myacr
    # or Service Principal:
    docker login myacr.azurecr.io -u <appId> -p <password>
    ```
*   **ECR (AWS)**:
    ```bash
    aws ecr get-login-password | \
      docker login --username AWS --password-stdin <acct>.dkr.ecr.ap-south-1.amazonaws.com
    ```
*   **GHCR**:
    ```bash
    echo $GH_PAT | docker login ghcr.io -u <username> --password-stdin
    ```

Store creds as **CI secrets** (masked) and avoid echoing them in logs.

***

## 133) Image retention / lifecycle policies in private registries

*   **ACR**: retention policies for **untagged manifests**, repository‑level **content trust**, tasks to purge old tags.
*   **ECR**: **lifecycle policies** (JSON) to expire images by count/age/tag patterns.
*   **Harbor/Quay**: retention, immutability, vulnerability gates, replication rules.

Best practice: **immutable tags** with **automated cleanup** for stale or untagged images.

***

## 134) Compress images & reduce layer sizes (squashing trade‑offs)

*   **Reduce size** properly:
    *   Minimal base (alpine/distroless)
    *   Multi‑stage builds
    *   Combine `RUN` steps and **remove caches** (apt, npm, pip)
    *   `.dockerignore` to prevent copying junk
*   **Squashing** (experimental): merges layers into one; may reduce redundant copies but **breaks caching**, complicates diffs.
    *   Prefer **not** to squash unless required by legacy constraints.

***

## 135) Immutable tags & promotion across environments

*   Use immutable **semver** tags (e.g., `1.4.2`) and **digests** for deploys.
*   Promotion by **retagging/importing**:
    ```bash
    # Copy image across registries without pull/push (ACR import)
    az acr import --name prodacr \
      --source devacr.azurecr.io/repo/app:1.4.2 \
      --image repo/app:1.4.2
    ```
*   Keep `:dev`, `:staging`, `:prod` as **pointers** to immutable version tags; never mutate existing version tags.

***

## 136) Image provenance & metadata tracking (labels, SBOM)

*   **Labels**:
    ```dockerfile
    LABEL org.opencontainers.image.source="https://github.com/acme/api" \
          org.opencontainers.image.version="1.4.2" \
          org.opencontainers.image.revision="abc123"
    ```
*   **SBOM** (Software Bill of Materials):
    *   Generate with **Syft**, **Trivy**, or **buildx attestation**.
    ```bash
    syft myacr.azurecr.io/repo/app:1.4.2 -o spdx-json
    ```
*   Attach SBOM/signatures to enforce **supply‑chain** policies in admission controllers (Kyverno/Gatekeeper).

***

## 137) Run security hardening scans during CI

*   **Stages**:
    *   **SAST** (code), **SCA** (dependencies), **Image scanning** (Trivy/Grype), **Secret scanning** (git‑secrets, trufflehog), **Policy** (Open Policy Agent).
*   **Quality gates**: fail pipeline on **CRITICAL/HIGH** vulns; allow waivers with expiration.
*   **Continuous**: rescan regularly and on base image updates.

***

## 138) Detect & prevent sensitive files from being added to images

*   Use **`.dockerignore`** to exclude `.env`, keys, `.aws/`, `.kube/`, `id_rsa`, etc.
*   Add CI checks: **trufflehog**, **git‑secrets**, **detect‑secrets**.
*   Prefer **BuildKit secrets** for build‑time access; never bake secrets into `ENV` or layers.

***

## 139) Integrate runtime security protections (seccomp, AppArmor, SELinux)

*   **seccomp**: restrict syscalls (Docker has a **default** profile).
*   **AppArmor/SELinux**: confine filesystem/process access.
*   **Capabilities**: drop all, add minimal necessary (e.g., `NET_BIND_SERVICE`).
*   **User namespaces**, **rootless**, **read‑only** FS, **no privileged**, **pids/cpu/mem limits**.

Example:

```bash
docker run -d \
  --security-opt seccomp=default \
  --security-opt no-new-privileges \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --read-only --tmpfs /tmp \
  --user 10001 \
  repo/app:1.0
```

***

## 140) Configure seccomp profiles (concept)

*   A **seccomp profile** is a JSON policy defining allowed/blocked **syscalls**.
*   Start from Docker’s **default** profile; customize only as needed.
*   Apply per container:
    ```bash
    docker run --security-opt seccomp=/path/to/profile.json repo/app:1.0
    ```
*   **Process**:
    *   Identify required syscalls (test under load; use `strace`/audit logs).
    *   Block dangerous calls (`ptrace`, `keyctl`, `kexec`, `mount`, etc.) not needed by your app.
    *   Validate with integration tests; avoid `seccomp=unconfined` in production.

***

### Handy bundle (security‑tight runtime template)

```bash
docker run -d \
  --restart=unless-stopped \
  --cpus=1.0 --memory=512m --memory-reservation=256m --pids-limit=200 \
  --read-only --tmpfs /tmp:rw,size=64m \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 10001 \
  -v appdata:/var/lib/app \
  -p 8080:8080 \
  myacr.azurecr.io/repo/app:1.0
```

***

Absolutely—continuing with crisp, interview‑ready answers and hands‑on examples for **141–160**, tailored for day‑to‑day DevOps/SRE work.

***

## 141) How to configure AppArmor or SELinux policies for Docker? (concept)

**AppArmor (Ubuntu/Debian):**

*   Docker ships a **default AppArmor profile**. You can apply a custom one per container:
    ```bash
    docker run --security-opt apparmor=docker-default repo/app:1.0
    docker run --security-opt apparmor=my-custom-profile repo/app:1.0
    ```
*   Create a profile under `/etc/apparmor.d/my-custom-profile`, then load it:
    ```bash
    sudo apparmor_parser -r /etc/apparmor.d/my-custom-profile
    ```
*   Profile controls filesystem access, network, capabilities, etc. Start from `docker-default`, restrict paths (e.g., `deny /host/secrets/** r`), and allow only what the app needs.

**SELinux (RHEL/CentOS/Fedora):**

*   Run containers under `container_t` domain; volumes need labels:
    *   `:Z` (private relabel) or `:z` (shared relabel) for bind mounts:
        ```bash
        docker run -v /data/app:/var/lib/app:Z repo/app:1.0
        ```
*   Per‑container contexts/booleans via `--security-opt`:
    ```bash
    docker run --security-opt label=type:container_t repo/app:1.0
    ```
*   Avoid `--privileged`; prefer fine‑grained SELinux labels + capabilities.

***

## 142) How to manage user/group IDs when mounting host volumes (avoid permission issues)

*   **Align UID/GID**: Run the container with the same UID/GID as files on the host:
    ```bash
    docker run --user 10001:10001 -v /data/app:/var/lib/app repo/app:1.0
    ```
*   **Pre‑set ownership** of the mount:
    ```bash
    sudo chown -R 10001:10001 /data/app
    ```
*   **Initialize via helper container**:
    ```bash
    docker run --rm -v appdata:/var/lib/app alpine sh -c 'chown -R 10001:10001 /var/lib/app'
    ```
*   On SELinux hosts, add `:Z`/`:z` to bind mounts to fix labeling.
*   Prefer **numeric IDs** (portable) vs names (which may differ across images).

***

## 143) Group namespaces or `chown` in entrypoint for runtime permissions

*   Linux doesn’t have “group namespaces” like user ns; instead:
    *   Add **supplementary groups**: `--group-add 2000`
    *   Use consistent **UID/GID** across containers sharing volumes.
*   **Entrypoint `chown`** (last resort): fix ownership at startup, understanding it can be slow on large trees:
    ```bash
    # entrypoint.sh
    set -e
    chown -R 10001:10001 /var/lib/app
    exec /usr/local/bin/app
    ```
*   Prefer using **init/sidecar** to initialize volume ownership once, rather than every start. Tools like **gosu**/**su-exec** can drop privileges cleanly in scripts.

***

## 144) Running system services inside containers (best practices & alternatives)

*   **Avoid full init/systemd** in containers; run **one process per container**.
*   If you must run multiple processes, use a **lightweight init** (e.g., `tini`, `dumb-init`) or a **supervisor** (s6, supervisord) to forward signals and reap zombies:
    ```bash
    docker run --init repo/app:1.0  # adds tini
    ```
*   Don’t run **SSH** in app containers; use `docker exec` or orchestrator tooling for access.
*   For cron, use:
    *   App‑native schedulers,
    *   Orchestrator **CronJobs** (K8s),
    *   Separate **sidecar** for scheduled tasks.

***

## 145) Blue/green or canary deployments with Docker + orchestrator (concept)

*   **Blue/Green**: run **two versions** (blue and green) and switch traffic atomically.
    *   **Kubernetes**: two Deployments behind one Service; flip selector or update Ingress routing to point to the new deployment.
    *   **Swarm**: two Services; switch DNS or external LB to the new one.
*   **Canary**: gradually shift traffic to the new version.
    *   **Kubernetes**: use Ingress/Service mesh (Istio/Linkerd/NGINX Ingress) with **weighted routing**; scale canary replicas progressively.
    *   **Swarm**: use external LB with weighted backends or gradually scale tasks.
*   Always combine with **readiness probes**, **rolling updates**, and **automatic rollback** on errors.

***

## 146) Graceful shutdown of containers & handling SIGTERM

*   Ensure the **main process (PID 1)** handles `SIGTERM` and exits cleanly.
*   Docker sends `SIGTERM`, then `SIGKILL` after the stop timeout (default \~10s).
*   App example (Node.js):
    ```js
    process.on('SIGTERM', async () => {
      await server.close(); // stop accepting new connections
      process.exit(0);
    });
    ```
*   Docker run options:
    ```bash
    docker stop --time 20 app  # extend grace period
    ```
*   In Kubernetes, use a **preStop** hook for drain, and set `terminationGracePeriodSeconds`.

***

## 147) Forward signals to child processes in entrypoint

*   Use **exec** form in shell entrypoint so the child becomes **PID 1** and receives signals:
    ```bash
    #!/bin/sh
    set -e
    trap 'echo "SIGTERM"; kill -TERM "$child" 2>/dev/null' TERM INT
    /usr/local/bin/app "$@" &
    child=$!
    wait "$child"
    ```
*   Or use **tini**:
    ```dockerfile
    ENTRYPOINT ["/sbin/tini","--","/usr/local/bin/app"]
    ```
*   Avoid `bash -c "app"` without `exec`—signals won’t reach the actual app properly.

***

## 148) Create a reproducible build environment using Docker (dev containers)

*   Package dev toolchain and dependencies into a **Dev Container**:
    *   **VS Code**: `.devcontainer/devcontainer.json` + `Dockerfile`.
    *   Pin base image **digest** and versions.
*   Mount source, use **bind mounts** for live edit, and define tasks/debug configs inside the container.
*   Benefits: consistent tooling across machines/CI, fast onboarding, fewer “works‑on‑my‑machine” bugs.

***

## 149) Debug build failures locally & in CI (layer caching, verbose build)

*   Enable **BuildKit** and verbose logs:
    ```bash
    export DOCKER_BUILDKIT=1
    docker build --progress=plain -t repo/app:debug .
    ```
*   Force cold build to find cache issues:
    ```bash
    docker build --no-cache -t repo/app:nocache .
    ```
*   Check **context** and `.dockerignore` (missing/excluded files).
*   Use `docker history`, `dive`, and **split steps** to pinpoint failing instructions.
*   In CI, replicate with same Builder (`buildx create --use`), same platform, same base image **digest** (`FROM node@sha256:...`).

***

## 150) Use `docker-compose.override.yml` for dev vs prod configs

*   Compose auto‑merges `docker-compose.override.yml`—great for dev:
    *   Add **ports**, **bind mounts**, **debug env**.
*   For prod, avoid overrides or specify explicit files:
    ```bash
    docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
    ```
*   Example override for dev:
    ```yaml
    services:
      api:
        ports: ["8080:8080"]
        volumes: ["./src:/app/src:ro"]
        environment: { LOG_LEVEL: "debug" }
    ```

***

## 151) Limit image pull rate or manage pull‑through cache (registry caching)

*   Configure **registry mirrors** in the Docker daemon to cache upstream pulls:
    ```json
    // /etc/docker/daemon.json
    {
      "registry-mirrors": ["https://mirror.mycorp.local"]
    }
    ```
*   Run a **pull‑through cache** (e.g., `registry:2` configured as a proxy to Docker Hub).
*   Prefer **private registries** (ACR/ECR/Harbor) with geo‑replication and local caching to avoid Hub rate limits.

***

## 152) Operate private registries at scale (storage, GC, replication)

*   Back registry with **object storage** (S3/Azure Blob/GCS) and enable **garbage collection** (delete unreferenced blobs).
*   Use **retention policies** (expire old/untagged manifests), **immutability**, and **vulnerability scanning**.
*   **Replication** to DR/edge sites; protect with **TLS**, **RBAC**, and **WAF/CDN** as appropriate.
*   Monitor **storage usage**, **latency**, and **error rates**; run GC during **low‑traffic windows**.

***

## 153) Configure TLS & auth for Docker registry (Harbor, registry:2)

*   **registry:2**:
    *   TLS: configure `http: { tls: { certificate, key } }` in `config.yml` and run behind a secure endpoint.
    *   Auth: **basic auth** via `htpasswd`, or **token‑based** auth. Place CA under `/etc/docker/certs.d/<host>/ca.crt` on clients.
*   **Harbor**:
    *   Built‑in HTTPS, **OIDC/LDAP** auth, **RBAC**, **project‑level** permissions, **vuln scanning**, and **replication**.
    *   Use real certificates (Let’s Encrypt/internal CA), enforce **content trust** and **immutable tags**.

***

## 154) Set up role‑based access control in private registries (Harbor/ACR/ECR)

*   **Harbor**: roles (guest/developer/maintainer/project admin), per‑project permissions (pull/push/delete/scan).
*   **ACR**: Azure **RBAC** (`AcrPull`, `AcrPush`, `Owner`) bound to users/service principals.
*   **ECR**: **IAM** policies for repositories/actions (GetDownloadUrl, BatchGetImage, PutImage, etc.).
*   Principle: **least privilege**, audit logs, service accounts/tokens for CI.

***

## 155) Migrate images between registries (preserve tags/digests)

*   **Skopeo** (best for copy without re‑pulling):
    ```bash
    skopeo copy docker://devacr.azurecr.io/repo/app:1.4.2 \
                docker://prodacr.azurecr.io/repo/app:1.4.2
    ```
*   **ACR import**:
    ```bash
    az acr import --name prodacr \
      --source devacr.azurecr.io/repo/app:1.4.2 \
      --image repo/app:1.4.2
    ```
*   Avoid `docker save/load` when you need to **preserve digests/manifests** and multi‑arch lists—use registry‑native copy tools.

***

## 156) Atomic deploys with image tags & orchestration (concept)

*   Deploy **immutable** tags/digests; never mutate existing version tags.
*   Use **rolling updates** with readiness probes for zero‑downtime; ensure **maxUnavailable/ surge** settings (K8s).
*   Prefetch images (daemon or node image cache) to prevent cold start delays.
*   Rollback on probe failure; record **deploy metadata** (build ID, git SHA) for traceability.

***

## 157) Use labels & metadata for automated orchestration/discovery

*   **Dockerfile**:
    ```dockerfile
    LABEL org.opencontainers.image.source="https://github.com/acme/api" \
          org.opencontainers.image.version="1.4.2" \
          org.opencontainers.image.revision="abc123"
    ```
*   Use labels for:
    *   Ownership & cost allocation,
    *   Automated routing (e.g., Traefik uses container labels to configure routes),
    *   CI/CD selection & policies,
    *   Search/filters: `docker ps --filter "label=env=prod"`.

***

## 158) Test containerized apps locally with Compose & integration tests

*   Spin up dependencies with Compose; gate tests on **healthchecks**:
    ```yaml
    services:
      db:
        image: postgres:16
        healthcheck:
          test: ["CMD", "pg_isready","-U","postgres"]
          interval: 10s
      api:
        build: ./api
        depends_on:
          db:
            condition: service_healthy
    ```
*   Run integration tests **inside the network**:
    ```bash
    docker compose up -d
    docker run --rm --network <compose_net> repo/tests:latest
    docker compose down -v
    ```
*   Seed data via init scripts; ensure **isolated** test environments and **cleanup** (`down -v`).

***

## 159) Ensure reproducible builds across CI runners (same builder/image)

*   Pin **base images by digest** (`FROM node@sha256:...`).
*   Use **buildx** with a **named builder** across runners and **remote/registry cache**:
    ```bash
    docker buildx create --name ci --use
    docker buildx build --cache-to type=registry,ref=myacr.azurecr.io/repo/cache:build \
                        --cache-from type=registry,ref=myacr.azurecr.io/repo/cache:build \
                        -t myacr.azurecr.io/repo/app:1.0 --push .
    ```
*   **DOCKER\_BUILDKIT=1**, consistent **platform** (`--platform`), and deterministic tooling versions (lockfiles).
*   Generate and store **SBOM** + metadata (labels) for traceability.

***

## 160) Implement image signing in CI/CD and verify at deploy time (concept)

*   **Sign after build** (CI):
    *   **Cosign (Sigstore)** – keyless with OIDC or key‑pair:
        ```bash
        cosign sign myacr.azurecr.io/repo/app:1.0
        ```
    *   **Notation (Notary v2)** for OCI‑native signatures.
*   **Store signatures** in the registry (OCI artifacts).
*   **Verify at deploy**:
    *   **Admission control** (K8s: Kyverno/Gatekeeper/OPA) requiring **valid signatures** and matching **issuer/subject**.
    *   **Cluster image policy**: only allow images from trusted registries, signed by your CI.
*   Combine with **provenance/attestations** (SLSA, `buildx` attestations) and **immutable tags** to enforce supply‑chain integrity end‑to‑end.

***

### Handy bundle (secure, robust runtime template)

```bash
docker run -d \
  --restart=unless-stopped \
  --cpus=1.0 --memory=512m --memory-reservation=256m --pids-limit=200 \
  --read-only --tmpfs /tmp:rw,size=64m \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 10001 \
  -v appdata:/var/lib/app:rw \
  -p 8080:8080 \
  myacr.azurecr.io/repo/app:1.0
```

***
Love the focus—here are crisp, interview‑ready answers with practical commands and patterns for **161–180**. I’ll keep them hands‑on and tuned for a DevOps/SRE workflow.

***

## 161) Use `docker exec` to collect diagnostics and create reproducible traces

Run lightweight commands inside the running container to capture **process**, **network**, **filesystem**, and **env** state. Store outputs to a mounted path for reproducibility.

```bash
# One-off: collect key diagnostics
docker exec app sh -c '
  set -o pipefail
  echo "# date"; date
  echo "# uname -a"; uname -a
  echo "# env"; env
  echo "# ps aux"; ps aux
  echo "# open files"; (command -v lsof && lsof -p 1) || echo "lsof not installed"
  echo "# listening sockets"; (command -v ss && ss -lntp) || netstat -lntp
  echo "# routes"; ip route
  echo "# disk usage"; df -h
' > diag_app.txt

# Stream logs during incident
docker logs -t --since 10m -f app > app_logs_tail.txt
```

Tip: Use a **netshoot** toolbox container for deep network debugging:

```bash
docker run --rm -it --network appnet nicolaka/netshoot sh
# dig api, curl http://api:8080, tcpdump -i any
```

***

## 162) Configure `daemon.json` for proxy, logging, registry mirrors

**Linux:** `/etc/docker/daemon.json`

```json
{
  "dns": ["1.1.1.1", "8.8.8.8"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "5", "mode": "non-blocking", "max-buffer-size": "8m" },
  "registry-mirrors": ["https://mirror.mycorp.local"],
  "insecure-registries": ["registry.mycorp.local:5000"]
}
```

Systemd proxy for the **daemon** (if behind corporate proxy):
`/etc/systemd/system/docker.service.d/proxy.conf`

```ini
[Service]
Environment="HTTP_PROXY=http://proxy.mycorp:8080"
Environment="HTTPS_PROXY=http://proxy.mycorp:8080"
Environment="NO_PROXY=localhost,127.0.0.1,.mycorp.local"
```

Apply:

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

***

## 163) Docker Swarm secrets and configs (concept)

*   **Secrets**: small, sensitive blobs mounted **in‑memory** at `/run/secrets/<name>` to services.
*   **Configs**: non‑secret blobs (e.g., config files) mounted as read‑only files.

```bash
# Create
echo "supersecret" | docker secret create db_pass -
docker config create app_conf ./conf.yml

# Use in service
docker service create --name api \
  --secret db_pass \
  --config source=app_conf,target=/etc/app/conf.yml \
  myorg/api:1.0
```

Secrets are not exposed in image layers, `docker logs`, or environment variables (unless you explicitly export them—which you should avoid).

***

## 164) Restrict container capabilities (`--cap-drop`, `--cap-add`)

Drop **all** Linux capabilities, add back only what’s needed:

```bash
docker run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \   # bind to low ports
  --cap-add=CHOWN \              # if you need chown
  --security-opt no-new-privileges \
  myorg/web:1.0
```

Common minimal set for web apps: `NET_BIND_SERVICE` only.

***

## 165) Run privileged containers & risks

`--privileged` gives the container **full** capabilities, device access, and relaxes security (AppArmor/SELinux). Risks:

*   Potential **host compromise** (kernel interfaces, device nodes).
*   Broad **filesystem** and **proc** access.
*   Bypasses many isolation guarantees.

**Avoid** unless absolutely necessary (e.g., low‑level tooling). Prefer targeted device access (`--device`), specific capabilities, and tailored security opts.

***

## 166) Use Linux capabilities to reduce surface area

Start with none, then add only what’s required:

```bash
docker run --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=SYS_TIME \        # only if you truly must set time (rare)
  myorg/app:1.0
```

Combine with:

*   `--read-only` (root filesystem),
*   `--tmpfs /tmp`,
*   Non‑root `USER`,
*   Seccomp default profile,
*   AppArmor/SELinux confinement.

***

## 167) Audit host kernel parameters for container workload safety (concept)

Checklist:

*   **cgroups v2** enabled and configured (consistent across nodes).
*   **Kernel hardening**:
    *   `kernel.unprivileged_bpf_disabled=1`
    *   `kernel.kptr_restrict=2`
    *   `fs.protected_hardlinks=1`, `fs.protected_symlinks=1`
    *   `vm.max_map_count` tuned for Elasticsearch‑like workloads.
*   **Networking**:
    *   `net.ipv4.ip_forward=1` (if NAT/forwarding needed),
    *   `net.ipv4.conf.all.rp_filter=1` (anti‑spoof),
    *   `net.ipv4.tcp_syncookies=1` (DoS protection).
*   **Time sync**: chrony/ntpd running on host.

Use `sysctl -a | grep <param>` and manage via `/etc/sysctl.d/*.conf`.

***

## 168) Detect container breakout attempts & harden hosts (concept)

*   **Runtime detection**: Falco/Tracee (eBPF) rules for suspicious syscalls (e.g., `mount`, `ptrace`, `chmod 777`, `/proc` tampering).
*   **Hardening**:
    *   Drop caps; `no-new-privileges`; seccomp default; AppArmor/SELinux enforcing.
    *   User namespaces (`userns-remap`), **rootless** where possible.
    *   Read‑only FS; minimal images; mount only required volumes.
    *   Isolate CI builders from prod nodes; never mount `/var/run/docker.sock` into arbitrary containers.
*   **Monitoring**: centralize logs, kernel audit, eBPF sensors; alert on anomalies.

***

## 169) Host‑level isolation: VMs + containers vs bare‑metal

*   **VMs + containers**: stronger **isolation boundary** (hypervisor + container), safer multi‑tenancy, blast‑radius reduction. Slight overhead; more layers to manage. Good for **security‑sensitive** workloads.
*   **Bare‑metal containers**: max performance; fewer layers; but weaker isolation. Better for **trusted**, single‑tenant clusters.

Middle ground: **sandboxed runtimes** (gVisor, Kata Containers) add syscall interception / lightweight VMs for stronger isolation with partial performance trade‑offs.

***

## 170) Measure image pull & startup times and optimize

**Measure:**

```bash
time docker pull myorg/api:1.0
time docker run --rm myorg/api:1.0 --help
docker events --since 5m   # see pull/run events with timestamps
```

**Optimize:**

*   **Smaller images** (multi‑stage, minimal base, remove build tools).
*   **Layer reuse** across images (common base).
*   **Registry caching/mirrors** close to nodes.
*   **Warm cache**: pre‑pull on nodes ahead of deploy.
*   **Multi‑arch** manifests for native pulls; avoid emulation.

***

## 171) Incremental image builds during CI to speed iteration

*   Enable **BuildKit** and use **registry cache**:

```bash
docker buildx build \
  --cache-to type=registry,ref=myacr.azurecr.io/repo/cache:build,mode=max \
  --cache-from type=registry,ref=myacr.azurecr.io/repo/cache:build \
  -t myacr.azurecr.io/repo/app:1.0 --push .
```

*   Order Dockerfile to maximize cache hits (copy manifests first).
*   Pin base image digest to avoid unexpected cache busts.

***

## 172) Use `docker-compose` with environment variable substitution (`env_file`)

Compose supports `${VAR}` and `.env` / `env_file`:

```yaml
services:
  api:
    image: ${REGISTRY}/api:${TAG}
    env_file:
      - ./api/.env
    environment:
      LOG_LEVEL: ${LOG_LEVEL:-info}
      DB_HOST: db
```

`.env` (auto‑loaded from project root) or explicit `env_file` entries provide variables securely without hardcoding in YAML.

***

## 173) Centralize logging to ELK/EFK or Splunk (concept)

Options:

*   **Log drivers**: `fluentd`, `gelf`, `awslogs`, `splunk` to ship stdout/stderr directly.
    ```bash
    docker run --log-driver=fluentd --log-opt fluentd-address=127.0.0.1:24224 myorg/api:1.0
    ```
*   **Agent approach**: run **Fluent Bit/Fluentd/Filebeat** on nodes to collect container logs (json-file) and forward to **Elasticsearch/Kibana**, **Splunk**, or **Graylog**.
*   Use structured logs (JSON), include correlation IDs, and set sane rotation (`daemon.json`).

***

## 174) Expose metrics & scrape via Prometheus (concept)

*   **Instrument the app** (Prometheus client libraries) or run an **exporter** (Node Exporter, cAdvisor) exposing `/metrics`.
*   Prometheus scrape config (targets via static list or service discovery):

```yaml
scrape_configs:
  - job_name: 'api'
    static_configs:
      - targets: ['api:9090']
```

In Compose, run a Prometheus service on the same network and point to `api:9090`.

***

## 175) Use healthchecks with orchestrator to avoid routing to unhealthy containers

*   **Docker Engine**: `HEALTHCHECK` marks containers `healthy/unhealthy` but doesn’t stop routing by itself.
*   **Compose**: `depends_on: condition: service_healthy` gates start order only.
*   **Kubernetes**: **readinessProbe** controls **traffic routing** (Service excludes unready pods), **livenessProbe** controls restarts, **startupProbe** avoids premature kills. This is the recommended way to **avoid routing to unhealthy containers**.

***

## 176) Readiness vs liveness at container level

*   **Liveness**: “should I restart?”—very shallow check (process responsiveness/heartbeat).
*   **Readiness**: “can I serve traffic?”—listener bound, local fast checks; **fail readiness** during drain/startup.
*   Docker’s `HEALTHCHECK` is a **single probe**; treat it like **readiness**. For true separation, use **Kubernetes** with distinct probes.

***

## 177) Run multiple replicas locally with Compose

```bash
docker compose up -d --scale api=3
```

**Port publishing caveat**: if `api` publishes a host port (`8080:8080`), only one replica can bind it. Use a **reverse proxy** (Traefik/Nginx) fronting the replicas or remove direct port publishing and expose only the proxy.

***

## 178) Live reloading of code in containers during development

*   **Bind mount** your source and run a watcher:

```bash
docker run -it --rm \
  -v "$PWD"/src:/app/src \
  -p 3000:3000 \
  myorg/node-dev:latest sh -c "npm install && npx nodemon --watch src server.js"
```

Compose example:

```yaml
services:
  api:
    build: ./api
    command: npx nodemon --watch src server.js
    volumes:
      - ./api/src:/app/src
    ports:
      - "3000:3000"
```

***

## 179) CI pipelines that build, scan, sign, and push images (outline)

1.  **Checkout & metadata** (commit SHA, build date).
2.  **Build** (BuildKit, multi‑arch if needed).
3.  **Scan** image (Trivy/Grype) → **quality gates**.
4.  **Sign** (Cosign/Notation).
5.  **Push** to registry (ACR/ECR/GHCR).
6.  **Attestations/SBOM** (Syft/buildx).
7.  **Deploy** (K8s/Compose) using **immutable tags/digests** and probes.

Example (pseudo‑script):

```bash
export DOCKER_BUILDKIT=1
docker buildx build --platform linux/amd64 -t myacr.azurecr.io/repo/app:${TAG} --push .
trivy image --exit-code 1 --severity HIGH,CRITICAL myacr.azurecr.io/repo/app:${TAG}
cosign sign myacr.azurecr.io/repo/app:${TAG}
kubectl set image deploy/api api=myacr.azurecr.io/repo/app:${TAG} --record
```

***

## 180) Handle time synchronization (chrony/NTP) for containers needing correct time

*   Containers **inherit time** from the **host kernel**. Ensure host runs **chrony/ntpd** and is in sync.
*   **Timezone** inside container:
    *   Set `TZ` env: `-e TZ=Asia/Kolkata`
    *   Or bind mount localtime: `-v /etc/localtime:/etc/localtime:ro`
*   **Changing time inside container** requires elevated privileges (`SYS_TIME`)—**avoid**; fix time at the **host** level or via orchestration.

***

### Handy bundle (secure dev/prod defaults)

```bash
docker run -d \
  --restart=unless-stopped \
  --cpus=1.0 --memory=512m --memory-reservation=256m --pids-limit=200 \
  --read-only --tmpfs /tmp:rw,size=64m \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 10001 \
  -v appdata:/var/lib/app \
  -p 8080:8080 \
  myorg/app:1.0
```

Absolutely—continuing with crisp, interview‑ready answers and practical patterns for **181–200**. I’ll keep them hands‑on and tuned for real SRE/DevOps work.

***

## 181) How to implement runtime configuration reloading inside containerized apps?

**Approaches:**

*   **Signal‑based reload**: App traps `SIGHUP`/`SIGUSR1` and re‑reads config (Nginx, HAProxy, many servers).
    ```bash
    # Trigger reload from host
    docker kill -s HUP app
    ```
*   **File watch**: App uses inotify to re‑load when a mounted config file/dir changes (e.g., `fsnotify` in Go).
*   **Admin endpoint**: Expose `/reload` guarded by auth to trigger reload without restart.
*   **Mount configs**: Use bind mounts or configs/secrets (Swarm/K8s) so updates are visible in‑place.
*   **Kubernetes**: Use **ConfigMap volumes** + app file watch; for env var changes you need a restart (or reloader sidecar).

**Tips:** Make reload **idempotent**, **fast**, and **fail‑safe** (validate new config; fallback to last good).

***

## 182) How to avoid layering secrets into images (runtime secret providers)?

*   **Do NOT** bake secrets via `ARG`/`ENV`/`COPY`.
*   Use **runtime secret mounts**:
    *   **Docker Swarm**: `docker secret` → mounted at `/run/secrets/<name>`.
    *   **Kubernetes**: `Secret` → mounted files or env (files preferred).
    *   **External managers**: Azure Key Vault (CSI driver), HashiCorp Vault Agent sidecar, AWS Secrets Manager/SSM.
*   Configure apps to **read secrets from files** at known paths; combine with **`read-only`** FS and strict permissions.

***

## 183) Init containers pattern analog with Docker Compose (workarounds)

Compose has no native `initContainers`. Workarounds:

*   **One‑shot init service**:
    ```yaml
    services:
      initdb:
        image: postgres:16
        command: ["bash","-lc","psql ... < /init.sql"]
        depends_on: [db]
      api:
        depends_on:
          initdb:
            condition: service_completed_successfully
    ```
*   **Healthcheck gating**: gate dependent service start on `service_healthy`.
*   **`wait-for`** scripts in the app entrypoint to delay until deps are ready.
*   **Manual step**: `docker compose run --rm migrator` before `up`.

***

## 184) Ensure containers start in a specific order (depends\_on caveats)

*   `depends_on` controls **start order**, not **readiness** (unless you use Compose v2 conditions).
*   Prefer **healthchecks** + `condition: service_healthy`:
    ```yaml
    services:
      db:
        image: postgres:16
        healthcheck: { test: ["CMD","pg_isready","-U","postgres"], interval: 10s, retries: 6 }
      api:
        depends_on:
          db:
            condition: service_healthy
    ```
*   Even then, apps should **retry** connections (defensive design).

***

## 185) Test network partitioning locally (tc/netem concepts)

*   Apply **latency/loss/packet‑corruption** using `tc netem` **inside the container’s netns**:
    ```bash
    pid=$(docker inspect -f '{{.State.Pid}}' api)
    sudo nsenter -t "$pid" -n tc qdisc add dev eth0 root netem delay 200ms loss 2%
    # remove
    sudo nsenter -t "$pid" -n tc qdisc del dev eth0 root
    ```
*   Block routes using iptables in the same netns:
    ```bash
    sudo nsenter -t "$pid" -n iptables -A OUTPUT -d 10.0.0.5 -j DROP
    ```
*   Tools: **pumba**, **toxiproxy**, **comcast**, service‑mesh fault injection (in K8s).

***

## 186) Debug container networking issues

Checklist:

*   **Inspect network**: `docker network inspect <net>` (subnet, endpoints, aliases).
*   **Inside container**: `ip a`, `ip route`, `ss -lntup`, `dig`, `nslookup`.
*   **Host NAT** (published ports): `sudo iptables -t nat -L -n -v | grep <port>`.
*   **Logs & events**: `docker events --since 10m`.
*   **Name resolution**: check `/etc/resolv.conf` (Docker embedded DNS `127.0.0.11` on user-defined bridges).
*   **Packet capture**: `tcpdump -i any port 5432` (or use **netshoot** container).
*   **nsenter** into container netns as shown in #185.

***

## 187) Handle IPv6 in Docker networks (configuration concept)

*   Enable IPv6 in the daemon:
    ```json
    // /etc/docker/daemon.json
    { "ipv6": true, "fixed-cidr-v6": "2001:db8:1::/64" }
    ```
*   Create networks with IPv6:
    ```bash
    docker network create --ipv6 --subnet "2001:db8:2::/64" appnet6
    docker run --network appnet6 --ip6 "2001:db8:2::10" nginx
    ```
*   Ensure host kernel IPv6 enabled, firewall allows IPv6; be aware of **NAT66** limitations (often none—use routing).

***

## 188) Transparent outbound proxying for inspection

Options:

*   **Env proxy**: set `HTTP_PROXY/HTTPS_PROXY/NO_PROXY` per container or globally in daemon/systemd (simple, app‑aware).
*   **Sidecar proxy**: run Squid/mitmproxy and set containers to route via it.
*   **Transparent (app‑agnostic)**: iptables **REDIRECT/TProxy** to proxy’s port within container’s netns or host:
    ```bash
    # example: redirect all :80 to local proxy :3128 in the container netns
    nsenter -t "$pid" -n iptables -t nat -A OUTPUT -p tcp --dport 80 -j REDIRECT --to-ports 3128
    ```
*   In Kubernetes, use a **service mesh** (Istio/Linkerd) for mTLS/inspection policies.

***

## 189) `docker scan` (Snyk) concept

*   Historically `docker scan <image>` invoked Snyk to scan for CVEs and report severities/fixes.
*   In CI, prefer **Snyk CLI** (`snyk container test`) or **Trivy/Grype**.
*   Output can be used as a **quality gate** (fail on HIGH/CRITICAL).

*(Note: newer Docker tooling emphasizes **Docker Scout**; for interview context, focus on the scanning concept and CI integration.)*

***

## 190) Enforce runtime policies (block unsigned/untrusted images)

*   **Kubernetes**: Admission controllers (Kyverno, OPA Gatekeeper, Sigstore Policy Controller, Connaisseur) to **verify signatures**, registry allow‑lists, label checks, and block privileged configs.
*   **Docker/Compose**: No native admission control—enforce at **registry** (immutability, RBAC), **CI** (signing), and **host** (run as non‑root, userns, seccomp/AppArmor/SELinux).
*   **Registries**: Enable content trust/immutability; require signed images via org policy.

***

## 191) Periodic image garbage collection (registry & host)

*   **Host**:
    *   `docker system prune -f`, `docker image prune -a -f`, `docker volume prune -f` via **cron/systemd timers**.
    *   Observe `docker system df` before cleanup.
*   **Registry**:
    *   **registry:2**: delete manifests/tags, then run `garbage-collect` during maintenance.
    *   **ACR/ECR/Harbor**: use **retention/lifecycle policies** to auto‑expire old or untagged images (no downtime).

***

## 192) Minimal attack surface images (distroless) & trade‑offs

*   **Distroless**: only app + runtime libs, **no shell/package manager** → smallest surface.
    *   **Pros**: tiny, fewer CVEs, faster pull/start.
    *   **Cons**: debugging harder (no shell/tools), must ensure logging to stdout/stderr; may need a separate **debug** tag.
*   Alternatives: `alpine` (musl, tiny), `scratch` (empty, static binaries only).

***

## 193) Run GUI apps in Docker (X forwarding / VNC) + considerations

*   **X11 on Linux**:
    ```bash
    xhost +local:root
    docker run -e DISPLAY=$DISPLAY \
      -v /tmp/.X11-unix:/tmp/.X11-unix:ro guiimage
    ```
*   **VNC/noVNC/xpra**: run an X server inside container and connect via VNC/web.
*   **Considerations**: security (X11 is permissive), GPU access (`--gpus all` for NVIDIA), audio (PulseAudio/ALSA), input devices `--device`.

***

## 194) Control container startup order with systemd units

*   Wrap containers in **systemd** services:
    ```ini
    [Unit]
    Description=API Container
    After=docker.service db.service
    Requires=docker.service db.service

    [Service]
    Restart=always
    ExecStart=/usr/bin/docker start -a api
    ExecStop=/usr/bin/docker stop -t 20 api

    [Install]
    WantedBy=multi-user.target
    ```
*   Use `After=` and `Requires=` to control ordering; or run **Compose** via a unit and manage dependencies across units.

***

## 195) Run in Kubernetes vs standalone Docker (concept)

*   **Docker**: single‑node, manual lifecycle (`docker run`, volumes, bridge networks).
*   **Kubernetes**: multi‑node **orchestrator**: Pods, Deployments, Services, Ingress, StatefulSets, PVCs, ConfigMaps/Secrets, HPA, RBAC, network policy, probes, rolling/blue‑green/canary, admission controls.
*   K8s uses **CNI** for networking, **containerd/runc** as runtime; Docker CLI is not used on nodes.

***

## 196) Ephemeral containers for debugging (concept)

*   **Kubernetes feature**: add a temporary **ephemeral container** into a running Pod’s namespaces for debugging without restarts:
    ```bash
    kubectl debug -it deploy/api --image=nicolaka/netshoot --target=api
    ```
*   Great for prod: you get shell/tools in the pod’s netns/PID ns. Removed when done.
*   Docker standalone: use `docker exec` or `nsenter`.

***

## 197) Test resilience to OOM/CPU starvation in CI

*   Start app with **limits**, then stress:
    ```bash
    docker run --rm --memory=256m --cpus=0.5 myorg/app:1.0 &
    docker run --rm --cpus=2 --memory=64m --name stress --pid container:<app_pid1> alpine sh -c "apk add --no-cache stress-ng && stress-ng --vm 1 --vm-bytes 300m --timeout 30s"
    ```
*   Assert behavior: exit codes (137/139), retries, backoff, health signals, graceful degradation.
*   Record metrics: RSS growth, GC pauses, latency under load.

***

## 198) Configure CPU shares vs CFS quotas (`--cpu-shares`, `--cpu-quota`)

*   **`--cpu-shares`**: relative weight **only under contention** (default 1024). 512 gets \~½ CPU share vs 1024 siblings.
*   **CFS quotas**:
    *   `--cpu-quota`/`--cpu-period` or **`--cpus`** (e.g., `--cpus=1.5`).
    *   Hard limit of CPU time over the period; enforces fair throttling.
*   Pinning: `--cpuset-cpus="0,2"` to pin to specific cores.

Examples:

```bash
docker run --cpu-shares=512 myorg/worker
docker run --cpus=1.0 myorg/api
docker run --cpu-quota=50000 --cpu-period=100000 myorg/job  # 0.5 CPU
```

***

## 199) Handle large file uploads/downloads with limited disk

*   **Stream** data (chunked upload/download); avoid buffering whole files to disk.
*   Use **tmpfs** for temporary data:
    ```bash
    docker run --tmpfs /tmp:size=256m myorg/app
    ```
*   Store artifacts on **external object storage** (S3/Blob/GCS) via **pre‑signed URLs**; don’t spool to container FS.
*   Configure proxies/servers for size/timeouts (`client_max_body_size` in Nginx, `maxRequestBodySize` etc.).
*   Mount a **dedicated volume** for larger capacity if needed; keep the root FS **read‑only**.

***

## 200) Instrument & trace container startup to reduce cold‑start latency

*   **Measure**:
    *   Add timestamps/spans around init phases (config load, DB connect, cache warm, JIT).
    *   Use **OpenTelemetry** to emit a **“startup” trace** and **duration metrics**.
    *   Capture image pull + container start times (`docker events`, CI timers).
*   **Optimize**:
    *   **Smaller images** (fewer layers, multi‑stage) → faster pulls.
    *   **Warm caches**: lazy‑init; defer non‑critical work to post‑start.
    *   **Connection pools**: exponential backoff, timeouts tuned; avoid sync waits where possible.
    *   **Pre‑pull** images on nodes; use **registry mirrors** close to hosts.
    *   **Native arch** builds (avoid emulation); avoid heavy init scripts; consider **`--init` (tini)** for reliable signal handling.
    *   For JVM/Node: class‑data sharing (JVM), AOT/native images (GraalVM), Node’s snapshot/compile strategies; trim dependencies.

***

### Handy, secure runtime template (drop‑in)

```bash
docker run -d \
  --restart=unless-stopped \
  --cpus=1.0 --memory=512m --memory-reservation=256m --pids-limit=200 \
  --read-only --tmpfs /tmp:rw,size=64m \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 10001 \
  -v appdata:/var/lib/app \
  -p 8080:8080 \
  myorg/app:1.0
```

***



