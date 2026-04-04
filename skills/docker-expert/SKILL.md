---
name: docker-expert
description: Docker mastery: multi-stage builds, layer optimization, security scanning, Compose for dev/prod, networking, volumes, and container debugging.
---

# Docker Expert

Production-grade Docker skill covering the full container lifecycle: writing optimized Dockerfiles, composing multi-service stacks, hardening images for production, debugging failing containers, and establishing repeatable patterns across languages. Every recommendation prioritizes small images, fast builds, and secure defaults.

## Table of Contents

1. [Multi-Stage Builds](#1-multi-stage-builds)
2. [Image Optimization](#2-image-optimization)
3. [Docker Compose](#3-docker-compose)
4. [Networking](#4-networking)
5. [Volume and Storage](#5-volume-and-storage)
6. [Security Hardening](#6-security-hardening)
7. [Debugging Containers](#7-debugging-containers)
8. [Production Patterns](#8-production-patterns)

---

## 1. Multi-Stage Builds

Use multi-stage builds to separate build-time dependencies from runtime artifacts. The final image should contain only what is needed to run the application.

### Builder Pattern

```dockerfile
# Stage 1 — build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts
COPY . .
RUN npm run build

# Stage 2 — runtime
FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Build Arguments and Cache Mounts

```dockerfile
FROM golang:1.22-alpine AS builder
ARG VERSION=dev
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -ldflags="-s -w -X main.version=${VERSION}" -o /app

FROM scratch
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

### Secret Mounts (private dependencies)

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=secret,id=pip_conf,target=/etc/pip.conf \
    pip install --no-cache-dir -r requirements.txt
```

Build command:

```bash
docker build --secret id=pip_conf,src=$HOME/.pip/pip.conf -t myapp .
```

### SSH Mounts (private Git repos)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS builder
RUN apk add --no-cache openssh-client git
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
WORKDIR /app
RUN --mount=type=ssh git clone git@github.com:org/private-lib.git /app/lib
```

Build command:

```bash
docker build --ssh default -t myapp .
```

### Troubleshooting

- **Build cache not used:** Ensure `COPY` of dependency manifests comes before `COPY .` so the layer cache is valid.
- **Secret not found:** You must pass `--secret` at build time; secrets are never baked into layers.
- **SSH mount fails:** Run `ssh-add` before `docker build --ssh default`.

---

## 2. Image Optimization

### Layer Caching Order

Order instructions from least-changed to most-changed:

```dockerfile
FROM python:3.12-slim
# 1. System deps (rarely change)
RUN apt-get update && apt-get install -y --no-install-recommends libpq-dev \
    && rm -rf /var/lib/apt/lists/*
# 2. Language deps (change occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# 3. Application code (changes often)
COPY . .
```

### .dockerignore Best Practices

```
.git
node_modules
__pycache__
*.md
.env*
docker-compose*.yml
.vscode
.idea
coverage/
dist/
```

### Base Image Selection

| Base               | Size      | Use case                          |
|--------------------|-----------|-----------------------------------|
| `scratch`          | 0 MB      | Static Go/Rust binaries           |
| `gcr.io/distroless/static` | ~2 MB | Static binaries needing CA certs |
| `alpine:3.19`      | ~7 MB    | When you need a shell and apk     |
| `*-slim` variants  | ~80 MB   | Debian-based, smaller than full   |

### Analyzing Image Size with `dive`

```bash
# Install dive
brew install dive        # macOS
# or
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive myapp:latest

# Analyze layers
dive myapp:latest
```

Look for wasted space: files added then deleted in later layers, duplicate packages, leftover caches.

### Quick Size Check

```bash
docker images myapp --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
docker history myapp:latest --human --no-trunc
```

### Troubleshooting

- **Image too large:** Run `dive` to find bloated layers. Combine `RUN` commands to avoid leftover files.
- **Alpine DNS issues:** Add `RUN apk add --no-cache libc6-compat` if the app uses musl-incompatible DNS.
- **Missing CA certs in distroless:** Use `gcr.io/distroless/static` or copy `/etc/ssl/certs` from builder.

---

## 3. Docker Compose

### Dev vs Prod Configurations

**docker-compose.yml** (base):

```yaml
services:
  api:
    build: .
    environment:
      DATABASE_URL: postgres://db:5432/app
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend

volumes:
  pgdata:

networks:
  backend:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**docker-compose.override.yml** (dev only, auto-loaded):

```yaml
services:
  api:
    build:
      context: .
      target: builder
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"     # Node debugger
    environment:
      NODE_ENV: development
    command: ["npm", "run", "dev"]
```

**docker-compose.prod.yml** (production overrides):

```yaml
services:
  api:
    image: registry.example.com/myapp:${TAG:-latest}
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

Run production stack:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### Profiles

```yaml
services:
  api:
    build: .
  worker:
    build: .
    command: ["npm", "run", "worker"]
    profiles: ["worker"]
  mailhog:
    image: mailhog/mailhog
    profiles: ["dev"]
```

```bash
docker compose --profile dev up        # api + mailhog
docker compose --profile worker up     # api + worker
```

### Troubleshooting

- **depends_on ignored:** Without `condition: service_healthy`, Compose only waits for the container to start, not for the service inside.
- **Override not applied:** `docker-compose.override.yml` must be in the same directory and is loaded automatically only with `docker compose up`.
- **Env vars blank:** Use `env_file` or `.env` in the project root. Variable substitution (`${VAR}`) requires the variable to be exported in the shell or in `.env`.

---

## 4. Networking

### Network Types

| Driver    | Scope   | Use case                                    |
|-----------|---------|---------------------------------------------|
| `bridge`  | Local   | Default; container-to-container on one host |
| `host`    | Local   | No network isolation; bare-metal speed      |
| `overlay` | Swarm   | Multi-host communication                    |
| `none`    | Local   | Fully isolated (batch jobs, security)       |

### DNS Resolution

Containers on the same user-defined bridge network resolve each other by service name:

```bash
# Create a network
docker network create app-net

# Run two containers
docker run -d --name api --network app-net myapp-api
docker run -d --name worker --network app-net myapp-worker

# From worker, "api" resolves automatically
docker exec worker ping api
```

### Port Publishing

```bash
# Map host 8080 -> container 3000
docker run -p 8080:3000 myapp

# Bind to localhost only (not exposed externally)
docker run -p 127.0.0.1:8080:3000 myapp

# Publish all EXPOSE'd ports to random host ports
docker run -P myapp
```

### Network Isolation

```yaml
# docker-compose.yml
services:
  api:
    networks:
      - frontend
      - backend
  db:
    networks:
      - backend          # DB unreachable from frontend
  nginx:
    networks:
      - frontend

networks:
  frontend:
  backend:
    internal: true       # No outbound internet access
```

### Container-to-Container Communication

```bash
# Inspect network to see connected containers
docker network inspect app-net --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'

# Test connectivity
docker exec api curl -s http://worker:8080/health
```

### Troubleshooting

- **Cannot resolve service name:** Ensure both containers are on the same user-defined network. The default bridge does NOT support DNS.
- **Port already in use:** `lsof -i :8080` to find the conflicting process, or change the host port.
- **Container cannot reach internet:** Check that the network is not marked `internal: true`, and verify host DNS with `docker exec <c> nslookup google.com`.

---

## 5. Volume and Storage

### Bind Mounts vs Named Volumes vs tmpfs

| Type         | Persistence | Performance   | Use case                        |
|--------------|-------------|---------------|---------------------------------|
| Bind mount   | Host FS     | Native (Linux), slow (macOS) | Dev: live-reload source code |
| Named volume | Docker-managed | Consistent | Databases, persistent state     |
| tmpfs        | RAM only    | Fastest       | Secrets at runtime, scratch space |

```bash
# Bind mount
docker run -v $(pwd)/src:/app/src myapp

# Named volume
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# tmpfs
docker run --tmpfs /tmp:rw,noexec,nosuid,size=100m myapp
```

### Volume Drivers and Backup

```bash
# Create a named volume
docker volume create --driver local dbdata

# Backup a volume to a tar archive
docker run --rm -v dbdata:/data -v $(pwd):/backup alpine \
  tar czf /backup/dbdata-backup.tar.gz -C /data .

# Restore
docker run --rm -v dbdata:/data -v $(pwd):/backup alpine \
  tar xzf /backup/dbdata-backup.tar.gz -C /data
```

### Permission Issues

```dockerfile
# Fix: create a non-root user with matching UID
FROM node:20-alpine
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -D appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
```

```bash
# Quick fix for bind mount permissions on Linux
docker run -u "$(id -u):$(id -g)" -v $(pwd):/app myapp
```

### macOS Performance

Docker Desktop on macOS suffers from slow bind mounts. Mitigations:

```yaml
# docker-compose.yml — use delegated/cached consistency
services:
  api:
    volumes:
      - .:/app:cached
      - /app/node_modules   # Anonymous volume prevents syncing node_modules
```

For large projects, use `docker compose watch` (Compose 2.22+) or Mutagen-based sync.

### Troubleshooting

- **Permission denied inside container:** The container user UID doesn't match the host file owner. Use `--user` flag or `chown` in Dockerfile.
- **Volume not empty on first run:** Named volumes are initialized from the image only when empty. Remove and recreate to reinitialize: `docker volume rm dbdata`.
- **Slow file I/O on macOS:** Use `:cached` flag, anonymous volumes for `node_modules`, or `docker compose watch`.

---

## 6. Security Hardening

### Non-Root User

```dockerfile
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "index.js"]
```

### Read-Only Filesystem

```bash
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid myapp
```

In Compose:

```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp
```

### Capability Dropping

```bash
# Drop all capabilities, add only what's needed
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
```

### Seccomp Profiles

```bash
# Use Docker's default seccomp profile (blocks ~44 syscalls)
docker run --security-opt seccomp=default myapp

# Custom profile
docker run --security-opt seccomp=custom-seccomp.json myapp
```

### Image Scanning with Trivy

```bash
# Install Trivy
brew install trivy

# Scan an image for vulnerabilities
trivy image myapp:latest

# Scan and fail CI on HIGH/CRITICAL
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest

# Scan a Dockerfile for misconfigurations
trivy config Dockerfile
```

### Image Scanning with Grype

```bash
# Install Grype
brew install grype

# Scan an image
grype myapp:latest

# Fail on high severity
grype myapp:latest --fail-on high
```

### Supply Chain Security

```dockerfile
# Pin base image by digest
FROM node:20-alpine@sha256:abcdef1234567890...

# Verify downloaded binaries
RUN wget https://example.com/tool.tar.gz \
    && echo "expected_sha256  tool.tar.gz" | sha256sum -c - \
    && tar xzf tool.tar.gz
```

### Signing Images with cosign

```bash
# Generate a key pair
cosign generate-key-pair

# Sign
cosign sign --key cosign.key registry.example.com/myapp:latest

# Verify
cosign verify --key cosign.pub registry.example.com/myapp:latest
```

### Troubleshooting

- **Container exits with permission error:** You likely dropped a needed capability. Run with `--cap-add=SYS_PTRACE` temporarily to diagnose, then narrow down.
- **Read-only FS breaks app:** Identify writable paths the app needs and mount tmpfs there.
- **Trivy shows false positives:** Pin to a specific DB version with `--db-repository` or add a `.trivyignore` file.

---

## 7. Debugging Containers

### Essential Commands

```bash
# Execute a shell in a running container
docker exec -it <container> sh

# Follow logs with timestamps
docker logs -f --timestamps <container>

# Inspect full container config (mounts, env, network)
docker inspect <container>

# Real-time resource usage
docker stats <container>

# Show running processes inside a container
docker top <container>
```

### Debugging Crashed Containers

```bash
# See why it exited
docker inspect <container> --format='{{.State.ExitCode}} {{.State.Error}}'

# Read logs from a stopped container
docker logs <container>

# Start a shell in the failed image to investigate
docker run -it --entrypoint sh myapp:latest

# Commit a stopped container to a new image for inspection
docker commit <container> debug-image
docker run -it --entrypoint sh debug-image
```

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

```bash
# Check health status
docker inspect <container> --format='{{.State.Health.Status}}'

# View health check log
docker inspect <container> --format='{{range .State.Health.Log}}{{.Output}}{{end}}'
```

### Low-Level Debugging with nsenter

```bash
# Enter the network namespace of a container (useful when no shell is installed)
PID=$(docker inspect --format '{{.State.Pid}}' <container>)
sudo nsenter -t $PID -n ip addr
sudo nsenter -t $PID -n ss -tlnp
```

### Resource Monitoring

```bash
# One-shot stats
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# Check if a container is OOM-killed
docker inspect <container> --format='{{.State.OOMKilled}}'

# System-wide disk usage
docker system df -v
```

### Troubleshooting

- **exec fails "no such container":** The container may have crashed. Use `docker ps -a` to see stopped containers, then inspect logs.
- **Logs empty:** The app may write to a file instead of stdout. Exec in and check, or mount the log directory.
- **Container keeps restarting:** Check `docker inspect` for OOMKilled, then increase memory limits or fix the memory leak.

---

## 8. Production Patterns

### Node.js

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts
COPY . .
RUN npm run build && npm prune --production

FROM node:20-alpine
RUN apk add --no-cache tini
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

Key points: `tini` for proper signal forwarding, `USER node` (built-in non-root), `npm prune --production` to drop devDependencies.

### Python

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.12-slim
WORKDIR /app
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
COPY --from=builder /opt/venv /opt/venv
COPY --from=builder /app .
ENV PATH="/opt/venv/bin:$PATH"
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
CMD ["gunicorn", "app:create_app()", "--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "120"]
```

Key points: virtualenv copied as a whole directory, no pip in the final image, gunicorn for production WSGI.

### Go

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/server

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Key points: `scratch` base for minimal attack surface, statically linked binary, CA certs copied for HTTPS.

### Java (Spring Boot)

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

Key points: JRE (not JDK) in final image, container-aware JVM flags, Actuator health endpoint.

### Graceful Shutdown and Signal Handling

Containers receive `SIGTERM` on `docker stop`. Your app must handle it:

```javascript
// Node.js
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 10000); // Force after 10s
});
```

```python
# Python
import signal, sys

def handle_sigterm(signum, frame):
    print("SIGTERM received, shutting down")
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

Set `STOPSIGNAL` and stop grace period:

```dockerfile
STOPSIGNAL SIGTERM
```

```yaml
services:
  api:
    stop_grace_period: 30s
```

### Logging Best Practices

- Write to stdout/stderr, never to files inside the container.
- Use structured JSON logging for machine parsing.
- Set log driver and rotation in Compose:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
        tag: "{{.Name}}"
```

### Troubleshooting

- **Container ignores SIGTERM:** The app is PID 1 and doesn't handle signals. Use `tini` or `dumb-init` as the entrypoint.
- **Java OOM inside container:** JVM doesn't see container memory limits by default on older versions. Use `-XX:+UseContainerSupport` (JDK 10+).
- **Python workers not stopping:** Gunicorn master forwards SIGTERM to workers. Set `--graceful-timeout` to match `stop_grace_period`.
- **Go binary won't run on scratch:** Ensure `CGO_ENABLED=0` and static linking. If using net package, also set `GOFLAGS=-tags=netgo`.
