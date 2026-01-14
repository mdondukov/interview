# 01. Containers & Docker

[← Назад к списку тем](README.md)

---

## Container Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Container vs VM                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Virtual Machine:                Container:                         │
│  ┌─────────────┐                 ┌─────────────┐                    │
│  │   App A     │                 │   App A     │                    │
│  ├─────────────┤                 ├─────────────┤                    │
│  │  Guest OS   │                 │   Libs      │                    │
│  ├─────────────┤                 └─────────────┘                    │
│  │ Hypervisor  │                       │                            │
│  ├─────────────┤                 ┌─────────────┐                    │
│  │   Host OS   │                 │Container Eng│                    │
│  └─────────────┘                 ├─────────────┤                    │
│                                  │   Host OS   │                    │
│                                  └─────────────┘                    │
│                                                                     │
│  Containers share host kernel                                       │
│  - Faster startup (seconds vs minutes)                              │
│  - Lower overhead (MB vs GB)                                        │
│  - Less isolation than VMs                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Linux Building Blocks

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Container Building Blocks                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Namespaces — Isolation                                             │
│  - pid:    Process IDs                                              │
│  - net:    Network interfaces, routes                               │
│  - mnt:    Mount points                                             │
│  - uts:    Hostname                                                 │
│  - ipc:    Inter-process communication                              │
│  - user:   User/group IDs                                           │
│                                                                     │
│  cgroups — Resource limits                                          │
│  - CPU shares, quotas                                               │
│  - Memory limits                                                    │
│  - I/O limits                                                       │
│  - Process counts                                                   │
│                                                                     │
│  Union Filesystem — Layered images                                  │
│  - Overlay2 (most common)                                           │
│  - Copy-on-write for efficiency                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Docker Basics

### Commands

```bash
# Images
docker pull nginx:latest            # Download image
docker images                       # List images
docker rmi nginx                    # Remove image
docker build -t myapp:v1 .          # Build from Dockerfile
docker tag myapp:v1 repo/myapp:v1   # Tag image
docker push repo/myapp:v1           # Push to registry

# Containers
docker run -d --name web nginx      # Run detached
docker run -it ubuntu bash          # Interactive terminal
docker ps                           # Running containers
docker ps -a                        # All containers
docker stop web                     # Stop container
docker rm web                       # Remove container
docker logs web                     # View logs
docker logs -f web                  # Follow logs
docker exec -it web bash            # Shell into container

# Common run options
# -d: Detached
# --name: Container name
# -p: Port mapping host:container
# -v: Volume mount
# -e: Environment variable
# --restart: Restart policy
# --memory: Memory limit
# --cpus: CPU limit
docker run \
  -d \
  --name web \
  -p 8080:80 \
  -v /host/path:/container/path \
  -e ENV_VAR=value \
  --restart unless-stopped \
  --memory 512m \
  --cpus 0.5 \
  nginx:latest

# Inspect
docker inspect web                  # Container details
docker stats                        # Resource usage
docker top web                      # Processes in container
```

---

## Dockerfile

### Basic Structure

```dockerfile
# Base image
FROM python:3.11-slim

# Metadata
LABEL maintainer="dev@example.com"

# Set working directory
WORKDIR /app

# Install dependencies (separate step for caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Set environment variables
ENV PYTHONUNBUFFERED=1

# Expose port (documentation)
EXPOSE 8000

# Run as non-root user
RUN useradd -m appuser
USER appuser

# Default command
CMD ["python", "app.py"]
```

### Best Practices

```dockerfile
# Bad: Multiple RUN commands (multiple layers)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN rm -rf /var/lib/apt/lists/*

# Good: Single RUN with cleanup
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        vim && \
    rm -rf /var/lib/apt/lists/*

# Bad: COPY everything then install deps
COPY . .
RUN pip install -r requirements.txt

# Good: Copy deps first (better caching)
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### Multi-Stage Builds

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# Runtime stage
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/server .
CMD ["./server"]
```

---

## Image Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Docker Image Layers                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  FROM python:3.11-slim    ─┬─ Base image layers (read-only)         │
│                            │                                        │
│  RUN pip install...       ─┼─ Dependencies layer (read-only)        │
│                            │                                        │
│  COPY . .                 ─┴─ Application layer (read-only)         │
│                            │                                        │
│  [Container runs]         ─── Container layer (read-write)          │
│                                                                     │
│  Layer caching:                                                     │
│  - Each instruction creates a layer                                 │
│  - Layers are cached if instruction hasn't changed                  │
│  - Change in one layer invalidates all subsequent layers            │
│                                                                     │
│  View layers:                                                       │
│  docker history image:tag                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Networking

```bash
# Network types
docker network ls                   # List networks

# Bridge (default)
docker network create mynet         # Create network
docker run --network mynet nginx    # Join network
# Containers on same bridge can communicate by name

# Host
docker run --network host nginx     # Use host network directly

# None
docker run --network none nginx     # No networking

# Connect containers
docker network connect mynet container1
docker network disconnect mynet container1
```

### Network Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Docker Networking                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Host Machine                                                       │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                                                          │      │
│  │  Bridge Network (docker0)                                │      │
│  │  ┌─────────────┐    ┌─────────────┐                     │      │
│  │  │ Container A │    │ Container B │                     │      │
│  │  │ 172.17.0.2  │    │ 172.17.0.3  │                     │      │
│  │  └──────┬──────┘    └──────┬──────┘                     │      │
│  │         └───────┬──────────┘                            │      │
│  │                 │                                       │      │
│  │         ┌───────┴───────┐                               │      │
│  │         │  docker0      │                               │      │
│  │         │  172.17.0.1   │                               │      │
│  │         └───────┬───────┘                               │      │
│  │                 │                                       │      │
│  │         ┌───────┴───────┐                               │      │
│  │         │     NAT       │ -p 8080:80                    │      │
│  │         └───────────────┘                               │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Volumes & Storage

```bash
# Volume types
# 1. Named volumes (managed by Docker)
docker volume create mydata
docker run -v mydata:/app/data nginx

# 2. Bind mounts (host path)
docker run -v /host/path:/container/path nginx

# 3. tmpfs (in-memory)
docker run --tmpfs /app/tmp nginx

# Volume commands
docker volume ls                    # List volumes
docker volume inspect mydata        # Volume details
docker volume rm mydata             # Remove volume
docker volume prune                 # Remove unused volumes

# Permissions
# Volume is owned by root by default
# Use --user or chown in Dockerfile
docker run --user 1000:1000 -v mydata:/data nginx
```

---

## Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    environment:
      - DB_HOST=db
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      - db
    volumes:
      - ./app:/app
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend

volumes:
  postgres_data:

networks:
  backend:
```

```bash
# Compose commands
docker-compose up -d                # Start all services
docker-compose down                 # Stop and remove
docker-compose logs -f              # Follow logs
docker-compose ps                   # Service status
docker-compose exec web bash        # Shell into service
docker-compose build                # Build images
docker-compose pull                 # Pull images
```

---

## Security

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Container Security                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Don't run as root                                               │
│     USER nonroot                                                    │
│                                                                     │
│  2. Use minimal base images                                         │
│     FROM alpine or distroless                                       │
│                                                                     │
│  3. Don't expose secrets in images                                  │
│     Use env vars, secrets managers                                  │
│                                                                     │
│  4. Scan images for vulnerabilities                                 │
│     docker scan image:tag                                           │
│     trivy image:tag                                                 │
│                                                                     │
│  5. Read-only filesystem where possible                             │
│     docker run --read-only nginx                                    │
│                                                                     │
│  6. Limit capabilities                                              │
│     docker run --cap-drop ALL nginx                                 │
│                                                                     │
│  7. Use resource limits                                             │
│     --memory, --cpus                                                │
│                                                                     │
│  8. Keep base images updated                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Flags

```bash
# --read-only: Read-only filesystem
# --cap-drop ALL: Drop all capabilities
# --cap-add NET_BIND_SERVICE: Add specific capability
# --security-opt no-new-privileges: No privilege escalation
# --user 1000:1000: Non-root user
# --memory 256m: Memory limit
# --cpus 0.5: CPU limit
docker run \
  --read-only \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  --memory 256m \
  --cpus 0.5 \
  nginx
```

---

## Debugging

```bash
# Container won't start
docker logs container               # Check logs
docker inspect container            # Check config
docker events                       # Docker daemon events

# Shell into running container
docker exec -it container sh
docker exec -it container bash

# Shell into failed container (override entrypoint)
docker run -it --entrypoint sh image

# Copy files
docker cp container:/path/file ./local
docker cp ./local container:/path/file

# Resource usage
docker stats                        # Live stats
docker system df                    # Disk usage

# Cleanup
docker system prune                 # Remove unused data
docker system prune -a              # Remove all unused images
docker volume prune                 # Remove unused volumes
```

---

## См. также

- [Kubernetes](./02-kubernetes.md) — оркестрация контейнеров в production-среде
- [Python Packaging & Deployment](../02-python/11-packaging-deployment.md) — упаковка Python-приложений в контейнеры

---

## На интервью

### Типичные вопросы

1. **Container vs VM?**
   - Containers share kernel, less isolation, faster startup
   - VMs have full isolation, own kernel, more overhead
   - Use VMs for different OS, security boundaries
   - Use containers for app isolation, microservices

2. **How do you minimize image size?**
   - Use minimal base images (alpine, distroless)
   - Multi-stage builds
   - Combine RUN commands
   - Clean up in same layer
   - Use .dockerignore

3. **How does layer caching work?**
   - Each instruction creates a layer
   - Unchanged layers are cached
   - Change invalidates subsequent layers
   - Put frequently changing things last

4. **Container security best practices?**
   - Non-root user
   - Minimal base images
   - Scan for vulnerabilities
   - Limit capabilities
   - Resource limits
   - Read-only filesystem

5. **Container networking?**
   - Bridge: default, isolated network
   - Host: share host network
   - Overlay: multi-host networking
   - Containers on same network resolve by name

---

[← Назад к списку тем](README.md)
