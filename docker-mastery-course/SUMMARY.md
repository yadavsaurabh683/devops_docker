# End-of-Course Summary
## Docker Mastery — What You Learned Across 5 Days

This document consolidates every major concept you encountered during the course. Use it after
completing Day 5 to verify your understanding is complete and connected.

---

## Day 1 — What a Container Actually Is

### The Core Truth
A container is a **Linux process (or group of processes)** that operates with:
- A restricted view of the system (via **namespaces**)
- Enforced resource limits (via **cgroups**)
- An isolated root filesystem (via **union filesystems** like OverlayFS)

There is no hypervisor. No guest OS kernel. The host kernel runs everything.

### Namespaces
Namespaces provide **isolation**. Each namespace type restricts a different view:

| Namespace | Isolates | Example effect |
|-----------|---------|----------------|
| `pid` | Process IDs | Container sees its processes starting at PID 1 |
| `net` | Network stack | Container has its own interfaces, routing table, iptables |
| `mnt` | Filesystem mounts | Container has its own root filesystem |
| `uts` | Hostname | Container can have its own hostname |
| `ipc` | IPC resources | Shared memory, semaphores isolated |
| `user` | User/Group IDs | UID 0 inside maps to different UID on host |

`--network host` skips the `net` namespace — the container shares the host's network stack
directly. This is why host-network containers can bind to host ports without port publishing.

### Control Groups (cgroups)
cgroups provide **resource enforcement**. When a container exceeds its memory limit, the Linux
kernel OOM killer terminates the process. `docker inspect <container> --format '{{.State.OOMKilled}}'`
tells you if this happened.

cgroup paths:
- v2: `/sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max`
- v1: `/sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes`

### Union Filesystem (OverlayFS)
Images are stacks of **read-only layers**. When a container runs, a thin **read-write layer**
is added on top (the container layer). All writes go there. When the container is removed,
this layer is discarded. Image layers are shared across containers using the same base.

### Dockerfile Layer Mechanics
- Every `RUN`, `COPY`, `ADD` creates a new layer
- Layers are cached by Docker — a cache miss on one instruction invalidates all subsequent
- Copy dependencies before code to maximize cache hits across rebuilds
- `CMD ["exec", "form"]` passes signals correctly. `CMD shell form` wraps in `/bin/sh -c`,
  which masks `SIGTERM` — your app never gets the shutdown signal

### PID 1 Problem
Docker monitors PID 1. If PID 1 dies, the container stops. If your app is NOT PID 1
(wrapped in a shell script), shell exit may not propagate correctly. Use `exec` in scripts
to replace the shell with your app, or use exec form of CMD/ENTRYPOINT.

---

## Day 2 — Networking and Storage

### The Docker Network Stack (per container)
1. Docker creates a **veth pair** — a virtual cable with two ends
2. One end goes inside the container as `eth0`
3. The other end connects to the **docker0 bridge** (a software switch)
4. Docker assigns an IP from its subnet (default: 172.17.0.0/16)
5. Traffic to/from external world goes through **iptables NAT rules**

### Network Types
| Network | Description | Use case |
|---------|-------------|----------|
| `bridge` (default) | Shared docker0 bridge, no DNS by name | Quick single-container tests |
| User-defined bridge | Isolated bridge, DNS by container name | All multi-container apps |
| `host` | No network namespace — shares host stack | Performance-critical, no isolation |
| `none` | No networking at all | Fully isolated batch jobs |
| `overlay` | Multi-host networking (Swarm/Kubernetes) | Distributed systems |

### Docker DNS
On user-defined networks, Docker runs an embedded DNS server at **127.0.0.11** inside every
container. Container names resolve to their IPs automatically. This does NOT work on the
default bridge.

### Port Publishing (iptables DNAT)
`-p 8080:80` creates an iptables DNAT rule: traffic hitting the host on port 8080 is
redirected to the container's IP on port 80. Docker manages these rules dynamically.
Docker modifies iptables in the `DOCKER` chain — this runs **before** UFW rules, which
is a common source of unexpected firewall bypass in production.

### Storage Types

| Type | Managed by | Persists after `docker rm`? | Use case |
|------|------------|------------------------------|----------|
| Anonymous volume | Docker | Yes (but unmanageable hash name) | Avoid |
| Named volume | Docker | Yes | Persistent data (DBs) |
| Bind mount | Host filesystem | Yes (it's a host directory) | Config files, dev code reload |
| tmpfs | Memory | No | Ephemeral secrets, scratch space |

`docker rm` removes the container. `docker rm -v` removes container AND its anonymous volumes.
Named volumes survive both.

---

## Day 3 — Image Engineering

### Dockerfile Instruction Reference

| Instruction | Layer? | Notes |
|-------------|--------|-------|
| `FROM` | Yes | Defines base image. Pin to digest for reproducibility |
| `RUN` | Yes | Each RUN is a layer. Combine related commands with `&&` |
| `COPY` | Yes | Prefer over `ADD` (explicit, no magic). Order matters for cache |
| `ADD` | Yes | Adds tar extraction and URL fetching. Use COPY unless you need these |
| `ENV` | No (metadata) | Baked into image permanently. Visible in `docker inspect` |
| `ARG` | No (build-time only) | NOT in final image but visible in `docker history` |
| `WORKDIR` | Yes (if new dir) | Creates and sets working directory. Prefer over `RUN mkdir && cd` |
| `EXPOSE` | No (metadata) | Documents intent only. Does NOT publish the port |
| `USER` | No (metadata) | Sets the user for subsequent instructions and the container |
| `HEALTHCHECK` | No (metadata) | Defines how to test if the app is healthy |
| `CMD` | No (metadata) | Default command. Overridable at `docker run`. Use exec form |
| `ENTRYPOINT` | No (metadata) | Defines the executable. CMD becomes its arguments |
| `LABEL` | No (metadata) | Image metadata. Zero byte layer |
| `VOLUME` | No (metadata) | Declares a mount point. Creates anonymous volume at runtime |

### Secrets — The Cardinal Rules
- Never use `ARG` for secrets — visible in `docker history`
- Never use `ENV` for secrets — baked into every image layer permanently
- Use **BuildKit `--mount=type=secret`** for secrets needed during build
- Inject runtime secrets via `-e` flag or a secrets manager (Vault, AWS Secrets Manager)
- To audit: `docker history --no-trunc <image> | grep -i secret`
- To deep scan: `docker save <image> | tar -xO | strings | grep <keyword>`

### Multi-Stage Builds
Separate build environment from runtime environment in one Dockerfile:
```dockerfile
FROM golang:1.22 AS builder  # Heavy build stage
RUN go build -o app main.go

FROM scratch                  # Minimal runtime stage
COPY --from=builder /app /app
CMD ["/app"]
```

Result: 800 MB build image → 8 MB runtime image. Only the runtime image is shipped.

### Base Image Selection (smallest to largest attack surface)
`scratch` < `distroless` < `alpine` < `*-slim` < full base image

### .dockerignore
Controls what the Docker client sends to the daemon as the build context. Missing this means:
- Slow builds (large context transfer)
- Risk of secrets (`.env`, `*.pem`) being sent to the daemon and potentially baked in layers
- Even if you delete the file in a later `RUN`, it's already in the previous layer

### Image Security Scanning
`trivy image <name>` — checks CVEs in OS packages and application dependencies.
Run in CI/CD to gate deployment on vulnerability severity thresholds.

---

## Day 4 — Docker Compose

### `depends_on` — The Critical Nuance
```yaml
depends_on:
  db:
    condition: service_healthy  # wait for health check, not just container start
```
`condition: service_started` (default) only waits for the container to start. The service
inside may not be ready. Always use `service_healthy` with a proper `healthcheck`.

### Healthcheck Anatomy
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 5s        # how often to check
  timeout: 5s         # how long to wait for result
  retries: 10         # failures before marking unhealthy
  start_period: 15s   # grace period before first check counts
```

### Restart Policies
| Policy | Restarts on crash? | Restarts on clean exit? | Restarts after `docker stop`? |
|--------|-------------------|------------------------|-------------------------------|
| `no` | No | No | No |
| `on-failure` | Yes | No | No |
| `always` | Yes | Yes | Yes |
| `unless-stopped` | Yes | Yes | No |

Use `unless-stopped` for long-running services. Use `on-failure` for workers/processors.
Never use `always` for one-shot tasks.

### Environment Variable Resolution Order (highest to lowest priority)
1. Shell environment at `docker compose` runtime
2. `.env` file
3. `environment:` key in compose file
4. `env_file:` referenced file
5. `ENV` in the image's Dockerfile

### Compose Override Pattern
- `docker-compose.yml` — base config (commit this)
- `docker-compose.override.yml` — auto-loaded locally for dev overrides (gitignore this)
- `docker-compose.prod.yml` — explicit production additions (use with `-f`)

### Compose Service Scaling
```bash
docker compose up --scale api=3
```
Compose round-robins DNS for the service name across all instances. Do not add `ports:` to
scaled services (multiple instances can't bind the same host port). Use a reverse proxy.

---

## Day 5 — Production Debugging and Security

### The Debugging Toolkit

| Tool | When to use |
|------|-------------|
| `docker logs` | First stop. What did the process print? |
| `docker inspect` | Full metadata: state, network, mounts, env, health |
| `docker stats` | Live CPU, memory, I/O |
| `docker exec` | Run commands inside a live container |
| `docker events` | Real-time stream of daemon events (start, stop, OOM, health) |
| `docker cp` | Extract files from container for analysis |
| `nsenter` | Enter namespaces from host — works without shell in image |
| `nicolaka/netshoot` | Debug sidecar: all network tools, shares target's namespaces |
| `/proc/<pid>/` | Direct kernel interface to process state, memory, FDs, namespaces |
| `tcpdump` via sidecar | Capture traffic in the container's network namespace |

### The Debug Sidecar Pattern
```bash
docker run -it --rm \
  --pid=container:<target>      \  # shares PID namespace
  --network=container:<target>  \  # shares network namespace
  --volumes-from <target>       \  # shares volumes
  nicolaka/netshoot sh
```
Gives you a full tool suite in the target's namespace without needing a shell inside it.

### nsenter
```bash
CPID=$(docker inspect <container> --format '{{.State.Pid}}')
sudo nsenter -t $CPID --mount --uts --ipc --net --pid -- /bin/sh
```
Uses the host's `/bin/sh` binary but inside the container's namespaces. Works even for
scratch-based or distroless containers.

### Container Security Hardening Summary

| Hardening measure | How to apply | What it prevents |
|---|---|---|
| Non-root user | `USER appuser` in Dockerfile | Privilege escalation inside container |
| Drop all capabilities | `--cap-drop ALL` | Unnecessary kernel privileges |
| Read-only root FS | `--read-only` | Malware writing to container filesystem |
| No privileged mode | Never use `--privileged` in prod | Full host escape |
| No Docker socket mount | Never `-v /var/run/docker.sock:/var/run/docker.sock` | Container escape to host Docker |
| Pinned base images | `FROM python:3.12@sha256:...` | Supply chain attacks via tag mutation |
| Scanned images | Trivy in CI/CD | Known CVE exploitation |
| tmpfs for secrets | `--tmpfs /tmp` | Secrets persisting to disk |

### OOM Kill Chain
1. Process exceeds cgroup memory limit
2. Kernel OOM killer selects and kills the process
3. Container PID 1 dies (or the process that exceeded the limit)
4. Docker detects PID 1 exit, marks container as stopped
5. `docker inspect --format '{{.State.OOMKilled}}'` returns `true`
6. `docker events` shows the `oom` event
7. `dmesg | grep oom` shows kernel-level OOM report

---

## The Mental Model Map

```
CONTAINER
    │
    ├── Process (one or more Linux processes)
    │       └── PID Namespace (isolated PID space, PID 1 = your app)
    │
    ├── Network (isolated network stack)
    │       ├── eth0 (veth pair end, inside container)
    │       ├── 127.0.0.11 (Docker embedded DNS — user-defined networks only)
    │       └── Routing table, iptables (within net namespace)
    │
    ├── Filesystem (layered union filesystem)
    │       ├── Read-only image layers (shared, from registry)
    │       └── Read-write container layer (ephemeral, lost on rm)
    │
    ├── Resources (enforced by cgroups)
    │       ├── Memory limit → OOM kill on breach
    │       └── CPU shares/quota
    │
    └── Identity
            ├── UID/GID (user namespace, or host UID if no user ns)
            └── Linux capabilities (subset of root powers)

IMAGE
    │
    ├── Stack of read-only layers (OverlayFS)
    ├── Metadata (CMD, ENV, EXPOSE, HEALTHCHECK, USER, LABEL)
    └── Build cache (layer digest = cache key)

DOCKER DAEMON
    │
    ├── Manages container lifecycle (create, start, stop, rm)
    ├── Manages images (pull, build, push)
    ├── Manages networks (bridge creation, veth pairs, iptables rules)
    └── Manages volumes (named volume lifecycle)
```

---

## What You Can Now Do That Most DevOps Engineers Cannot

1. Explain what a container IS without saying "like a lightweight VM"
2. Debug a broken container using `/proc`, `nsenter`, and `docker inspect` — not by restarting
3. Write a Dockerfile that is cache-efficient, non-root, minimal, and signal-safe
4. Build a multi-service Compose topology with proper health-based startup ordering
5. Capture network traffic between containers without touching the container image
6. Audit any container for production security readiness in under 5 minutes
7. Explain why the Docker socket mount is dangerous, why `--privileged` is dangerous,
   and why ENV is not for secrets
8. Trace an OOM kill event from the Docker event stream to the kernel dmesg log
