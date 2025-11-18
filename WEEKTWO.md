# Docker Learning Guide — Week Two

A structured reference for Docker fundamentals, images, container internals, volumes, and networking.

---

## Table of Contents

1. [Part 1: Docker Fundamentals & Images](#part-1-docker-fundamentals--images)
2. [Part 2: Building Images (Dockerfile)](#part-2-building-images-dockerfile)
3. [Part 3: Container Internals](#part-3-container-internals)
4. [Part 4: Volumes & Persistent Storage](#part-4-volumes--persistent-storage)
5. [Part 5: Docker Networking](#part-5-docker-networking)

---

# Part 1: Docker Fundamentals & Images

## What Docker Does

Docker is a platform for developing, packaging, and running applications in **containers** — lightweight, isolated units that bundle:

- Application code
- Runtime and libraries
- Config and env

Same container runs consistently on dev, QA, and production.

---

## Why Containerize?

| Without containers | With Docker |
|--------------------|-------------|
| Apps installed directly on host; dependency conflicts | Isolated containers; each has its own stack |
| Manual, repeated setup on new machines | Build once, run anywhere |
| “Works on my machine” | Consistent environment |

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Image** | Read-only blueprint; built from a Dockerfile |
| **Container** | Running instance of an image; create, start, stop, delete |
| **Dockerfile** | Recipe: base OS, dependencies, commands |
| **Docker Engine** | Daemon that builds and runs containers |
| **Docker Hub** | Public image registry (like GitHub for images) |
| **Docker Compose** | Multi-container apps via `docker-compose.yml` |

---

## Architecture

```
Docker CLI (client)  →  Docker Daemon (server)
                              ↓
                    Container runtime (images, containers)
                              ↓
                    Host OS
```

- CLI (`docker`) talks to the daemon; daemon does build, pull, run.

---

## Install Docker

**Convenience script (Linux):**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**Ubuntu (apt):**

```bash
sudo apt update && sudo apt install docker.io -y
sudo systemctl enable docker && sudo systemctl start docker
```

**Verify:**

```bash
docker --version
docker run hello-world
```

---

## Basic Commands

| Task | Command |
|------|---------|
| Version | `docker --version` |
| Run container | `docker run hello-world` |
| List running | `docker ps` |
| List all | `docker ps -a` |
| List images | `docker images` |
| Pull image | `docker pull nginx` |
| Run interactive | `docker run -it ubuntu bash` |
| Stop | `docker stop <id>` |
| Remove container | `docker rm <id>` |
| Remove image | `docker rmi <id>` |

---

## Image Layers

Images are **read-only, layered** (union filesystem: OverlayFS, AUFS, Btrfs). Each Dockerfile instruction can add a layer.

| Instruction | Layer |
|--------------|--------|
| `FROM ubuntu:22.04` | Base OS |
| `RUN apt-get install nginx` | New layer (files changed) |
| `COPY . /var/www/html` | New layer |
| `CMD ["nginx", "-g", "daemon off;"]` | Metadata |

**When you run a container:** Docker adds a **writable layer** on top. Changes in the container live only in that layer; when the container is removed, that layer is discarded.

**Storage (Linux, overlay2):** `/var/lib/docker/overlay2/`

**Inspect layers:**

```bash
docker history <image>
docker image inspect <image> --format='{{json .RootFS.Layers}}' | jq
```

**Caching:** Layers are content-addressable (SHA256). Same base layers = shared, not duplicated → faster builds, less disk.

---

# Part 2: Building Images (Dockerfile)

## Dockerfile Instructions

### FROM — Base image

```dockerfile
FROM <image>[:<tag>]
# e.g. FROM ubuntu:22.04  FROM python:3.12-slim
```

- Every Dockerfile starts with `FROM` (except scratch).
- Prefer minimal bases: `-alpine`, `-slim`; pin tags for reproducibility.

---

### RUN — Commands during build

```dockerfile
RUN apt-get update && apt-get install -y curl
RUN pip install flask
```

- Each `RUN` adds a layer. Combine related commands and clean caches:

```dockerfile
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

---

### CMD — Default command when container starts

```dockerfile
CMD ["executable", "param1", "param2"]   # exec form (preferred)
CMD command param1 param2               # shell form
```

- Overridable: `docker run <image> <your_command>`.
- Prefer JSON (exec) form; only one `CMD` (last wins).

---

### ENTRYPOINT — Fixed main executable

```dockerfile
ENTRYPOINT ["python", "app.py"]
```

- Always runs; `CMD` often supplies default args to `ENTRYPOINT`.
- `docker run myapp arg1` → `python app.py arg1`.

**Pattern:** `ENTRYPOINT` = app, `CMD` = default args.

```dockerfile
ENTRYPOINT ["echo"]
CMD ["Hello, World!"]
# docker run myimage       → Hello, World!
# docker run myimage Bye   → Bye
```

---

### COPY vs ADD

| | COPY | ADD |
|--|------|-----|
| Use | Copy files/dirs from host | Same + URLs + auto tar extract |
| When | Default choice | Only when you need URL or tar |

Prefer `COPY` for normal file copies.

---

### ENV — Environment variables

```dockerfile
ENV APP_ENV=production
ENV PORT=8080
```

- Use `ENV` for build/runtime config; override at run with `docker run -e KEY=value`.

---

### Minimal Flask example

**Layout:** `flask-app/` with `app.py`, `requirements.txt`, `Dockerfile`.

**Dockerfile:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["python"]
CMD ["app.py"]
```

**Build & run:**

```bash
docker build -t flask-app:latest .
docker run -p 8080:8080 flask-app
```

---

### Image Best Practices

| Area | Practice |
|------|----------|
| Base | Minimal images; pin versions |
| Layers | Combine RUNs; copy deps first for cache |
| Cleanup | Remove apt caches, temp files |
| Security | Non-root user (`USER appuser`); no secrets in image |
| Scanning | `docker scan` or Trivy |
| Versioning | Pin base and dependencies |
| Multi-stage | Use for smaller final images |
| Health | Add `HEALTHCHECK` when useful |

**Inspect / scan:**

```bash
docker history flask-app
docker inspect flask-app
docker scan flask-app
# or: trivy image flask-app
```

---

# Part 3: Container Internals

## Containers ≠ VMs

A container is **a normal Linux process** with **kernel-level isolation** (namespaces + cgroups). It shares the host kernel; no separate OS.

- **Namespaces** — isolate view (PID, network, mount, UTS, IPC, user).
- **Cgroups** — limit CPU, memory, I/O, PIDs.

So: **Containers = processes with isolation**, not lightweight VMs.

---

## Process chain

```
docker (CLI) → Docker daemon → containerd → containerd-shim → runc → your process
```

| Component | Role |
|-----------|------|
| **containerd** | Manages containers, images, snapshots, networking |
| **containerd-shim** | One per container; parent of container process; allows daemon restart without killing containers |
| **runc** | Low-level runtime; sets up namespaces, cgroups, starts process |

**Process tree (host):** `containerd → containerd-shim → nginx` (and workers). Inside the container, the main process is **PID 1**; on the host it’s just another PID.

---

## Why the main process matters

**Container lifecycle = lifecycle of PID 1.** When PID 1 exits:

1. Shim detects exit → containerd marks container stopped.
2. Writable layer discarded, network removed, cgroups cleaned.
3. Container is gone.

So: nginx exits → container exits; `sleep 5` finishes → container exits. For multiple processes you need a process manager (e.g. supervisord) or design the app accordingly.

---

## Namespaces (short)

| Namespace | Effect |
|-----------|--------|
| **PID** | Container sees its own process tree; main process = PID 1 |
| **Network** | Own interfaces, IP, routing, firewall |
| **Mount** | Own root filesystem (OverlayFS + chroot) |
| **UTS** | Own hostname |
| **IPC** | Own semaphores / shared memory |
| **User** | Root inside can map to non-root on host |

---

## Cgroups

Limit resources per container (CPU, memory, disk I/O, PIDs). Example: cap container at 256MB RAM, 0.5 CPU. Prevents one container from starving others.

---

## Ephemeral storage

- Container = read-only image layers + **one writable layer**.
- Writable layer is **temporary**; removed when container is removed.
- Data that must persist → use **volumes**, **bind mounts**, or external storage (DB, S3).

---

## Summary (containers)

1. Container = **process + isolation** (namespaces + cgroups).
2. Runs on **host kernel**, not a VM.
3. **PID 1** in container = main process; if it dies, container stops.
4. Writable layer is **ephemeral**.
5. Isolation is process-level, not as strong as a VM.
6. Container processes are visible on host (`ps -ef`).

---

# Part 4: Volumes & Persistent Storage

## The problem

Containers are ephemeral. Data written to paths like `/var/lib/mysql`, `/app/logs` inside the container is **lost** when the container is removed (not when merely stopped). To keep data, use **volumes** or **bind mounts**.

---

## Storage types

| Type | Persistent? | Where | Use |
|------|-------------|--------|-----|
| **Anonymous volume** | Yes* | `/var/lib/docker/volumes/...` | Short-lived; Docker-generated name |
| **Named volume** | Yes | `/var/lib/docker/volumes/<name>/` | DBs, persistent app data |
| **Bind mount** | Host dir | Host path | Dev (live code), configs, logs |
| **tmpfs** | No (RAM) | RAM | Sensitive temp data, caches |

*Anonymous volumes are removed when container is removed (e.g. with `--rm`).

---

## Anonymous volume

```bash
docker run -d -v /data --name app1 nginx
```

Docker creates a volume with a random ID. List with `docker volume ls`.

---

## Named volume

```bash
docker volume create mydata
docker run -d -v mydata:/var/lib/mysql --name mysql mysql:8
```

Data survives `docker rm`. Reuse: `docker run -d -v mydata:/var/lib/mysql mysql:8`.

**Good for:** MySQL, PostgreSQL, Redis, uploads, any persistent app data.

---

## Bind mount

```bash
docker run -d -v $(pwd)/website:/usr/share/nginx/html -p 8080:80 nginx
```

Host directory is mounted into the container; changes on host appear in container. **Good for:** development, configs, logs. Use absolute paths.

---

## tmpfs

```bash
docker run -d --tmpfs /cache nginx
# or: --mount type=tmpfs,target=/cache
```

Stored in RAM only; gone when container stops. **Good for:** secrets, caches, temporary data.

---

## Comparison

| Feature | Anonymous | Named | Bind | tmpfs |
|---------|-----------|--------|------|--------|
| Persists | ✔ | ✔ | On host | No |
| Managed by Docker | ✔ | ✔ | No | ✔ |
| Best for DB | No | ✔ | No | No |
| Best for dev | No | No | ✔ | No |
| Best for temp/secure | No | No | No | ✔ |

**Volume location (Linux):** `/var/lib/docker/volumes/`. Bind mounts are not under this path; they reference host paths directly.

---

# Part 5: Docker Networking

## Networking basics (recap)

- **IP** — Routable address for a device.
- **NIC** — Physical/virtual interface (Ethernet, Wi-Fi); MAC addresses, frames.
- **Switch** — Layer 2; forwards by MAC; MAC table.
- **ARP** — Maps IP → MAC on the LAN.
- **Gateway** — Router for traffic leaving the subnet.
- **NAT** — SNAT/MASQUERADE (private → public); DNAT (port forwarding / load balancing).
- **Subnetting** — Split network into smaller subnets (e.g. /24 → /26).

Docker uses: **network namespaces**, **veth pairs**, **Linux bridges**, **iptables/NAT**.

---

## Container networking

Each container has a **network namespace**: own interfaces, routing, firewall. Docker connects containers via **veth pairs** (one end in container as `eth0`, other on a **bridge**). Default bridge: `docker0` (e.g. 172.17.0.1/16). Docker adds **iptables** rules for NAT so containers can reach the internet and for port publishing.

---

## Docker network types

| Type | Use |
|------|-----|
| **bridge** | Default; containers on same bridge can talk by IP/name (custom networks only) |
| **host** | Container uses host network stack; no isolation, best performance |
| **none** | No networking |
| **overlay** | Multi-host (Swarm, K8s) |
| **macvlan** | Container gets own MAC; appears as real host on LAN |

---

## Bridge (default and custom)

**Default bridge:** Containers get IPs on `docker0`; no DNS by name. **Custom bridge:** You get DNS resolution by container name.

```bash
docker network create mynet
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet busybox
# From app2: ping app1
```

---

## Host and none

```bash
docker run --network host nginx    # Uses host’s network
docker run --network none alpine  # No network
```

---

## Port mapping

```bash
docker run -p 8080:80 nginx
```

Host port 8080 → container port 80. Implemented with iptables DNAT.

---

## How containers reach the internet

Docker adds NAT (e.g. MASQUERADE) so traffic from container (e.g. 172.17.0.x) goes out with host’s IP. Reply traffic is DNAT’d back to the container.

---

## Inspect networks

```bash
docker network ls
docker network inspect bridge
docker inspect <container>
```

---

## Example: app + DB

```bash
docker network create backend
docker run -d --name db --network backend mysql:8
docker run -d --name app --network backend myapp
```

App connects to `db:3306` (Docker resolves `db` on the same network).

---

## Architecture (conceptual)

```
Internet ↔ Host eth0 ↔ NAT/iptables ↔ docker0 bridge
                                        ↔ veth ↔ Container 1 (e.g. Nginx)
                                        ↔ veth ↔ Container 2 (e.g. API)
                                        ↔ veth ↔ Container 3 (e.g. MySQL)
```

---

*End of Docker notes — Week Two.*
