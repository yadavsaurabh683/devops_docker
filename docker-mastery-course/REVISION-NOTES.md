# Revision Notes — Docker Mastery Cheat Sheet
### Concise Reference for All 5 Days

Use this after completing the course to quickly revise concepts before interviews,
code reviews, or production incidents.

---

## Core Concepts in One Line Each

- **Container** = Linux process(es) + namespaces (isolation) + cgroups (resource limits) + OverlayFS (filesystem)
- **Image** = immutable stack of read-only filesystem layers + metadata
- **Namespace** = kernel feature that gives a process a restricted view of one system resource
- **cgroup** = kernel feature that enforces resource limits on a process group
- **OverlayFS** = union filesystem that stacks layers; writes go to a thin top layer only
- **docker0** = Linux bridge (software switch) that Docker creates; containers connect via veth pairs
- **veth pair** = virtual ethernet cable; one end in container (eth0), one end on docker0 bridge
- **iptables DNAT** = the actual mechanism behind `-p hostport:containerport`
- **Docker DNS (127.0.0.11)** = embedded resolver; works only on user-defined networks
- **Named volume** = Docker-managed persistent storage; survives container removal
- **Bind mount** = host directory mapped into container; both sides see changes immediately
- **tmpfs** = in-memory mount; ephemeral, lost on container stop
- **Multi-stage build** = multiple FROM stages in one Dockerfile; only final stage ships
- **PID 1 problem** = if your app is not PID 1, SIGTERM may not reach it; use exec form
- **Health check** = command Docker runs to determine if a service is actually functional
- **`service_healthy` depends_on** = waits for health check pass, not just container start
- **Restart policy** = `unless-stopped` for services; `on-failure` for workers; never `always` for tasks
- **Debug sidecar** = separate container sharing namespaces with target; gives you tools without modifying target image
- **nsenter** = enter container namespaces from host using host binaries; works on no-shell images
- **OOM kill** = kernel kills process when it exceeds cgroup memory limit; `OOMKilled: true` in inspect

---

## Namespace Quick Reference

```
pid  → process IDs            (container sees PID 1, host sees large PID)
net  → network stack          (container has own eth0, routing, iptables)
mnt  → filesystem mounts      (container has own root filesystem)
uts  → hostname               (container can have own hostname)
ipc  → IPC resources          (shared memory, semaphores)
user → UID/GID mapping        (UID 0 in container ≠ UID 0 on host if user ns enabled)
```

Check container's namespaces:
```bash
ls -la /proc/$(docker inspect <c> -f '{{.State.Pid}}')/ns/
```

---

## Dockerfile Instruction Reference Card

```dockerfile
FROM image:tag          # Base image — pin to digest in production
ARG NAME=default        # Build-time variable — NOT in final image, IS in docker history
ENV NAME=value          # Runtime variable — baked into image permanently
RUN cmd1 && cmd2        # New layer — combine related commands
COPY src dest           # New layer — prefer over ADD; order matters for cache
ADD src dest            # Like COPY but unpacks tars and fetches URLs
WORKDIR /path           # Create + set working directory
EXPOSE 8080             # Metadata only — does NOT publish the port
USER appuser            # Set runtime user — use non-root in production
HEALTHCHECK CMD ...     # How to test if the container is healthy
CMD ["exec", "form"]    # Default command — exec form passes signals correctly
ENTRYPOINT ["binary"]   # Fixed executable — CMD becomes its args
LABEL key=value         # Metadata — zero size impact
VOLUME /data            # Declares mount point — avoid (creates anonymous volumes)
```

---

## Dockerfile Anti-Patterns → Fixes

| Anti-pattern | Problem | Fix |
|---|---|---|
| `FROM ubuntu:latest` | Unpinned, huge attack surface | `FROM python:3.12-slim` or pin to digest |
| `ARG SECRET` + `ENV SECRET=$SECRET` | Secret in history and image | BuildKit `--mount=type=secret` |
| `ENV DB_PASSWORD=secret` | Credential baked into all layers | Inject at runtime with `-e` |
| Separate `RUN apt-get update` and `RUN apt-get install` | Two layers, cache drift | Combine: `RUN apt-get update && apt-get install -y ...` |
| No `rm -rf /var/lib/apt/lists/*` after apt | Carries apt cache in layer | Add to same RUN after install |
| `COPY . .` before `COPY requirements.txt` | Invalidates pip cache on any source change | COPY deps first, then source |
| No `.dockerignore` | Secrets and junk sent to daemon | Add `.dockerignore` with `.env`, `*.pem`, `.git`, etc. |
| `CMD python app.py` (shell form) | SIGTERM masked by shell | `CMD ["python", "app.py"]` (exec form) |
| No `USER` instruction | Runs as root | `RUN useradd -r appuser && USER appuser` |
| `EXPOSE 22` | Implies SSH into container (anti-pattern) | Remove; use `docker exec` for access |
| No `HEALTHCHECK` | Docker can't distinguish "running" from "healthy" | Add health check appropriate to app |
| Installing debug tools in prod image | Attack surface expansion, bloat | Multi-stage build; tools only in builder |

---

## Network Commands Cheat Sheet

```bash
# List all networks
docker network ls

# Inspect a network (see subnet, connected containers)
docker network inspect <network>

# Create a user-defined bridge network
docker network create mynet

# Connect a running container to a network
docker network connect mynet <container>

# Disconnect
docker network disconnect mynet <container>

# See veth pairs on host
ip link show type veth

# See bridge interfaces
ip link show type bridge

# See iptables NAT rules (Docker port publishing)
sudo iptables -t nat -L DOCKER -n

# Check DNS inside a container
docker exec <c> cat /etc/resolv.conf
docker exec <c> nslookup <service-name>

# Test TCP connectivity from inside container
docker exec <c> wget -q --spider http://<target>:<port>
```

---

## Storage Quick Reference

```bash
# Create named volume
docker volume create mydata

# List volumes
docker volume ls

# Inspect volume (see mount point on host)
docker volume inspect mydata

# Remove volume
docker volume rm mydata

# Run with named volume
docker run -v mydata:/var/lib/data myimage

# Run with bind mount (host:container)
docker run -v /host/path:/container/path myimage

# Run with read-only bind mount
docker run -v /host/config:/config:ro myimage

# Run with tmpfs
docker run --tmpfs /tmp:size=64m,mode=1777 myimage

# Remove container AND its anonymous volumes
docker rm -v <container>

# Prune unused volumes
docker volume prune
```

---

## Debugging Cheat Sheet

```bash
# Full container state
docker inspect <container>

# Just the status and OOM info
docker inspect <c> --format '{{.State.Status}} OOM={{.State.OOMKilled}}'

# Live logs
docker logs -f --timestamps <container>

# Last 50 lines
docker logs --tail 50 <container>

# Live resource usage
docker stats <container>
docker stats --no-stream     # single snapshot

# Run command in container
docker exec <container> <cmd>
docker exec -it <container> sh   # interactive shell

# Enter namespaces without shell in image
CPID=$(docker inspect <c> --format '{{.State.Pid}}')
sudo nsenter -t $CPID --mount --net --pid -- /bin/sh

# Debug sidecar
docker run -it --rm \
  --pid=container:<target> \
  --network=container:<target> \
  nicolaka/netshoot sh

# Copy file from container
docker cp <container>:/path/to/file /local/path

# Watch real-time events
docker events

# Filter events
docker events --filter event=oom --since 10m

# Check process memory from cgroup
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect <c> --format '{{.Id}}').scope/memory.current

# Check process state from /proc
sudo cat /proc/$CPID/status | grep -E 'VmRSS|VmSize|State|Threads'

# Scan image for vulnerabilities
trivy image --severity HIGH,CRITICAL <image>

# Explore image layers
dive <image>

# Check image history for secrets
docker history --no-trunc <image> | grep -i secret

# Deep scan image filesystem for strings
docker save <image> | tar -xO | strings | grep -i password
```

---

## Docker Compose Cheat Sheet

```bash
# Start all services (build if needed)
docker compose up --build

# Start in background
docker compose up -d

# Start specific service
docker compose up -d api

# Scale a service
docker compose up -d --scale api=3

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# View status and health
docker compose ps

# Follow logs (all services)
docker compose logs -f

# Follow logs (specific service)
docker compose logs -f api

# Run one-off command in a service
docker compose exec api sh
docker compose run --rm api python manage.py migrate

# Use specific compose files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up

# Reload compose file without downtime (recreates changed containers only)
docker compose up -d --no-deps api

# See resource usage across all services
docker stats $(docker compose ps -q)
```

---

## Security Hardening Reference

```bash
# Run as non-root user
docker run --user 1000:1000 <image>

# Drop all capabilities, add only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE <image>

# Read-only root filesystem with writable tmpfs
docker run --read-only --tmpfs /tmp:size=64m <image>

# No new privileges escalation
docker run --security-opt no-new-privileges <image>

# Limit memory and CPU
docker run --memory=512m --cpus=1.0 <image>

# Scan image
trivy image <image>

# Check if image runs as root
docker inspect <image> --format '{{.Config.User}}'

# Check capabilities configured on a running container
docker inspect <container> --format '{{json .HostConfig.CapAdd}}'
docker inspect <container> --format '{{json .HostConfig.CapDrop}}'

# Check if privileged
docker inspect <container> --format '{{.HostConfig.Privileged}}'
```

---

## Restart Policy Decision Tree

```
Is the container a long-running service? (web server, API, worker)
  └─ YES → restart: unless-stopped
           (restarts on crash AND on daemon restart, but NOT after manual docker stop)

Is the container a job/task? (migration, one-shot script)
  └─ YES → restart: on-failure
           (restarts on non-zero exit only; won't loop on clean completion)

Are you in development and want clean failures visible?
  └─ YES → restart: "no"
           (no automatic restart — you diagnose manually)

Do you want to restart even after a clean docker stop?
  └─ YES → restart: always
           (use rarely; typical case is a cron-like process you always want alive)
```

---

## The Production Container Readiness Checklist

Run before any container goes to production:

```
[ ] Runs as non-root user (USER in Dockerfile or --user at runtime)
[ ] CMD/ENTRYPOINT uses exec form (not shell form)
[ ] Health check defined (HEALTHCHECK in Dockerfile)
[ ] Base image pinned (to digest, not mutable tag)
[ ] Image scanned — no HIGH/CRITICAL CVEs unmitigated
[ ] No secrets in ENV, ARG, or any image layer
[ ] .dockerignore present and covers .env, *.pem, .git, node_modules
[ ] Multi-stage build used (build tools not in final image)
[ ] Capabilities dropped (--cap-drop ALL in production compose/manifests)
[ ] Read-only root filesystem where possible
[ ] Memory and CPU limits set
[ ] Log rotation configured (--log-opt max-size=10m max-file=3)
[ ] Restart policy set (unless-stopped for services)
[ ] /var/run/docker.sock NOT mounted (unless absolutely required)
[ ] Not running with --privileged
```
