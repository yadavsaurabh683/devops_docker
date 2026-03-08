# Day 5: Production-Level Debugging, Security, and Incident Simulation
## Theme: The Skills That Separate Senior Engineers from Everyone Else

**Goal:** By end of Day 5, you will be able to debug any container failure without restarting
it, harden images against real security threats, and handle production incident scenarios with
confidence. You will also understand container runtime security at a practical level.

**Estimated time:** 6–7 hours

---

## Background: Why Most Engineers Fail at Container Debugging

The most common response to a broken container in the wild is `docker restart <name>`.
This destroys the evidence. A senior engineer diagnoses first, then acts.

The debugging toolkit for containers:
- `docker logs` — what the process wrote to stdout/stderr
- `docker inspect` — all metadata: state, networks, mounts, env, config
- `docker exec` — run commands inside the live container's namespaces
- `docker stats` — live resource usage
- `docker events` — real-time Docker daemon event stream
- `nsenter` — enter container namespaces from the host (works even without a shell in image)
- `strace` — trace system calls of a running process
- `/proc/<pid>/` — the filesystem interface to process state
- `docker cp` — copy files out of a container for inspection

Master these and no container problem can hide from you.

---

## Exercise 5.1 — Advanced Debugging: No Shell, No Problem

**What you'll learn:** How to debug a container that has no shell (distroless/scratch images).

**Setup — run a minimal container with no shell:**
```bash
# Use a distroless image (no shell, no package manager)
docker run -d --name noshelldemo \
  gcr.io/distroless/static-debian12 \
  /bin/true || true

# More practical: run our Go app from Day 3 (scratch-based)
# If you don't have it, create a minimal one:
docker run -d --name noshell \
  --name noshelldemo2 \
  alpine sh -c 'rm /bin/sh; sleep 600' 2>/dev/null || \
  docker run -d --name noshelldemo2 alpine sleep 600
```

**Try the naive approach — fails:**
```bash
docker exec -it noshelldemo2 sh
# Error: no such file or directory
```

**The professional approach — `nsenter`:**

```bash
# Get the container's PID on the host
CPID=$(docker inspect noshelldemo2 --format '{{.State.Pid}}')
echo "Container PID: $CPID"

# Enter the container's namespaces using the HOST's tools
# This gives you a shell INSIDE the container's namespaces
# but using the host's /bin/sh binary
sudo nsenter -t $CPID --mount --uts --ipc --net --pid \
  -- /bin/sh

# Once inside, you have full access:
ls /
cat /proc/1/cmdline | tr '\0' ' '
ss -tlnp
env
```

**The debug sidecar pattern (no root required):**
```bash
# Run a debug container sharing namespaces with the target
docker run -it --rm \
  --pid=container:noshelldemo2 \
  --network=container:noshelldemo2 \
  --volumes-from noshelldemo2 \
  nicolaka/netshoot \
  sh

# Now you have all network/process tools in the target's namespace context
ps aux          # sees the target container's processes
ss -tlnp        # target's network bindings
netstat -rn     # target's routing table
```

> **Hint:** `nicolaka/netshoot` is the standard debugging image used across the industry.
> It contains: tcpdump, curl, dig, nslookup, ss, netstat, ping, traceroute, strace,
> iperf3, and dozens more. Keep it in your mental toolkit.

---

## Exercise 5.2 — Diagnosing a Memory Leak Without Killing the Container

**What you'll learn:** How to profile memory usage of a running container.

**Setup — simulate a leaking process:**
```bash
cat > /tmp/leak.py << 'EOF'
import time
import sys

leaked = []
print("Starting memory leak simulation", flush=True)
while True:
    # Allocate 10MB every 2 seconds
    leaked.append('x' * 10 * 1024 * 1024)
    print(f"Allocated chunk #{len(leaked)}, total approx: {len(leaked)*10}MB", flush=True)
    sys.stdout.flush()
    time.sleep(2)
EOF

docker run -d --name leaky \
  --memory=150m \
  -v /tmp/leak.py:/leak.py \
  python:3.12-slim \
  python /leak.py
```

**Diagnose without killing:**
```bash
# Terminal 1: Watch memory in real time
docker stats leaky

# Terminal 2: Watch logs
docker logs -f leaky

# Get the host PID of the leaking process
CPID=$(docker inspect leaky --format '{{.State.Pid}}')

# Read memory stats directly from cgroup (bytes)
# For cgroups v2:
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect leaky --format '{{.Id}}').scope/memory.current

# For cgroups v1:
cat /sys/fs/cgroup/memory/docker/$(docker inspect leaky --format '{{.Id}}')/memory.usage_in_bytes

# Read process memory maps from /proc
sudo cat /proc/$CPID/status | grep -E 'VmRSS|VmSize|VmPeak'

# When the OOM killer fires, check:
docker inspect leaky --format '{{.State.OOMKilled}}'
```

**After OOM kill, investigate the event:**
```bash
# Check Docker event log
docker events --since 10m --filter event=oom

# Check system OOM log
sudo dmesg | grep -i "oom\|killed process" | tail -10
```

---

## Exercise 5.3 — Network Debugging: Packet Capture Inside a Container

**What you'll learn:** How to capture and inspect traffic between containers.

**Setup:**
```bash
docker network create capturetest
docker run -d --name server --network capturetest nginx:alpine
docker run -d --name client --network capturetest alpine sleep 600
```

**Method 1 — tcpdump via debug sidecar:**
```bash
# Run a sidecar that shares the server's network namespace
docker run -it --rm \
  --network=container:server \
  nicolaka/netshoot \
  tcpdump -i eth0 -n port 80

# In another terminal, generate traffic:
docker exec client wget -q -O- http://server/
```

You'll see the full HTTP request/response in the tcpdump output.

**Method 2 — tcpdump on the host veth interface:**
```bash
# Find the veth interface for the server container
SERVER_PID=$(docker inspect server --format '{{.State.Pid}}')

# Find the ifindex of eth0 inside the container
IFINDEX=$(sudo nsenter -t $SERVER_PID --net -- ip link show eth0 | head -1 | cut -d: -f1)

# Find the matching veth on the host
HOST_VETH=$(ip link | grep "if${IFINDEX}:" | grep -o 'veth[a-z0-9]*')

echo "Capturing on host interface: $HOST_VETH"
sudo tcpdump -i $HOST_VETH -n -A port 80
```

**Method 3 — Inspect DNS queries:**
```bash
docker run -it --rm \
  --network=container:client \
  nicolaka/netshoot \
  tcpdump -i eth0 -n port 53

# Trigger DNS lookup in another terminal:
docker exec client nslookup server
```

Watch the DNS query go to 127.0.0.11 (Docker's embedded DNS) and the response come back.

---

## Exercise 5.4 — Container Security Hardening

**What you'll learn:** How to audit and harden a container's security posture.

**Part A — Capability Audit:**

By default, Docker grants containers a subset of Linux capabilities. Most apps need far fewer.

```bash
# See all capabilities a default container has
docker run --rm alpine sh -c 'apk add -q libcap && capsh --print'

# Common capabilities Docker grants by default:
# CAP_NET_BIND_SERVICE  - bind ports < 1024
# CAP_CHOWN            - change file ownership
# CAP_SETUID/SETGID    - change UID/GID
# CAP_KILL             - send signals
# CAP_SYS_CHROOT       - use chroot
# ...and more (15+ capabilities by default)

# Drop ALL capabilities, add only what's needed:
docker run --rm \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx:alpine \
  nginx -v
```

**Part B — Read-Only Root Filesystem:**

```bash
# Try to write to filesystem — succeeds by default:
docker run --rm alpine sh -c 'echo "I can write" > /tmp/test && cat /tmp/test'

# With read-only root filesystem:
docker run --rm --read-only alpine sh -c 'echo "test" > /tmp/test'
# Error: Read-only file system

# Combine read-only with tmpfs for writable temp space:
docker run --rm \
  --read-only \
  --tmpfs /tmp:size=64m,mode=1777 \
  alpine sh -c 'echo "test" > /tmp/test && cat /tmp/test'
```

**Part C — Non-Root User Enforcement:**

```bash
# Check what user a container runs as by default:
docker run --rm nginx whoami        # root! this is the default
docker run --rm nginx:unprivileged whoami  # nginx user

# Force non-root at runtime (even if Dockerfile doesn't specify):
docker run --rm --user 1000:1000 nginx sh -c 'id'

# This will fail if the app tries to bind port 80 (needs root or NET_BIND_SERVICE)
docker run --rm --user 1000:1000 nginx
# Permission denied on port 80

# Correct solution: run on port > 1024 as non-root:
docker run --rm --user 1000:1000 -p 8080:8080 \
  nginx:alpine sh -c 'nginx -g "daemon off;"' 2>&1 | head -5
```

**Part D — Privileged Mode Danger Demonstration:**

```bash
# DANGER: privileged mode gives the container full host access
# Only demonstrate this — never use in production

# Show that a privileged container can see ALL host devices:
docker run --rm --privileged alpine ls /dev/

# A privileged container can load kernel modules, mount the host filesystem, etc.
# This is effectively a full host escape vector.

# Instead, use specific capabilities if you MUST:
docker run --rm --cap-add SYS_PTRACE alpine ls /dev/
# Only ptrace capability, not full privilege
```

**Part E — Security Scan and Compliance Check:**

```bash
# Check if your image runs as root (automated):
docker inspect pyapp:prod --format '{{.Config.User}}'

# Check for dangerous capabilities in running container:
docker inspect $(docker ps -q | head -1) \
  --format '{{json .HostConfig.CapAdd}}' | python3 -m json.tool

# Use Docker Bench for Security (automated CIS benchmark check):
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -v /etc:/etc:ro \
  -v /lib/systemd/system:/lib/systemd/system:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security
```

> **Hint:** A production security checklist for containers:
> - [ ] Non-root user (`USER appuser` in Dockerfile)
> - [ ] Read-only root filesystem (`--read-only`)
> - [ ] Dropped all capabilities (`--cap-drop ALL`) + only necessary added back
> - [ ] No `--privileged`
> - [ ] No bind mount of `/var/run/docker.sock` (gives container full Docker API access)
> - [ ] Image scanned with Trivy (no HIGH/CRITICAL CVEs)
> - [ ] Base image pinned to digest, not just tag (`FROM python:3.12@sha256:...`)

---

## Exercise 5.5 — Production Incident Simulation

**Scenario:** You receive an alert at 2 AM: "API service is returning 500 errors intermittently.
CPU is spiking. One container appears stuck." Your job: diagnose root cause without restarting
anything and write a 3-line incident summary.

**Setup the broken environment:**
```bash
mkdir -p ~/docker-day5/incident
cd ~/docker-day5/incident

cat > broken_app.py << 'EOF'
from flask import Flask
import threading
import time
import random

app = Flask(__name__)

# Simulate a memory leak via a background thread
leaked = []
def leak_memory():
    while True:
        leaked.append('x' * 1024 * 512)  # 512KB every second
        time.sleep(1)

threading.Thread(target=leak_memory, daemon=True).start()

@app.route('/health')
def health():
    return 'ok', 200

@app.route('/')
def index():
    # Randomly fail 30% of requests
    if random.random() < 0.3:
        raise Exception("Simulated intermittent error")
    # Simulate occasional slow responses
    if random.random() < 0.2:
        time.sleep(5)
    return 'Hello World', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > Dockerfile << 'EOF'
FROM python:3.12-slim
RUN pip install flask gunicorn
COPY broken_app.py /app/app.py
WORKDIR /app
CMD ["python", "app.py"]
EOF

docker build -t incident-app:v1 .

docker run -d --name incident \
  --memory=200m \
  -p 5000:5000 \
  incident-app:v1
```

**Your diagnostic runbook (do these in order):**

```bash
# Step 1: Is the container running?
docker ps | grep incident
docker inspect incident --format '{{.State.Status}} OOMKilled={{.State.OOMKilled}}'

# Step 2: What does the container see as its resource usage?
docker stats incident --no-stream

# Step 3: What are the recent logs saying?
docker logs --tail 50 incident 2>&1

# Step 4: Is it responding?
for i in {1..10}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 http://localhost:5000/)
  echo "Request $i: $STATUS"
done

# Step 5: What processes are running inside?
docker exec incident ps aux

# Step 6: Check memory trend (run this for 30 seconds)
for i in {1..6}; do
  docker exec incident cat /proc/1/status | grep VmRSS
  sleep 5
done

# Step 7: Check if there are threads/goroutines consuming CPU
CPID=$(docker inspect incident --format '{{.State.Pid}}')
sudo ls /proc/$CPID/task/   # each entry is a thread

# Step 8: Examine open file descriptors (file descriptor leaks look like this too)
sudo ls /proc/$CPID/fd/ | wc -l

# Step 9: Check the application's error rate
docker logs incident 2>&1 | grep -c "500\|ERROR\|Exception"

# Step 10: Write your incident summary
```

**Write your 3-line incident summary:**
```
Root cause: Memory leak in background thread allocating 512KB/s with no bound.
Impact: Container OOM-killed after ~6 minutes; 30% of requests returning 500 errors.
Fix: Add memory bound to leak thread, fix error handling, set restart policy until code fixed.
```

---

## Exercise 5.6 — Dockerfile Security: Finding and Fixing Real Anti-Patterns

**What you'll learn:** How to audit a Dockerfile and identify every security and operational flaw.

**The deliberately bad Dockerfile — find ALL issues:**

```dockerfile
FROM ubuntu:latest

# TODO: remove before production
ARG API_KEY=sk-prod-1234567890abcdef

ENV DATABASE_URL=postgresql://admin:password@db.internal/prod
ENV API_KEY=$API_KEY

RUN apt-get update
RUN apt-get install -y curl wget git vim python3 python3-pip nodejs npm

COPY . /app

WORKDIR /app

RUN pip3 install -r requirements.txt

RUN chmod 777 /app

EXPOSE 80
EXPOSE 22

CMD python3 app.py
```

**Issues to find (try yourself before reading the answers):**

1. `FROM ubuntu:latest` — unpinned tag; builds will be non-reproducible, vulnerability surface
   is large. Fix: use `python:3.12-slim` or pin to digest.

2. `ARG API_KEY` + `ENV API_KEY=$API_KEY` — secret baked into image environment permanently
   and visible in `docker history`. Fix: use BuildKit secret mounts.

3. `ENV DATABASE_URL` with hardcoded credentials — baked into every image layer and visible to
   anyone who runs `docker inspect`. Fix: inject at runtime via `-e` or secrets manager.

4. Two separate `RUN apt-get` commands — two layers instead of one. Cache-inefficient and each
   layer carries apt metadata overhead. Fix: combine with `&&`.

5. No `--no-install-recommends` and no `rm -rf /var/lib/apt/lists/*` — image carries full apt
   cache unnecessarily. Adds ~50MB.

6. Installing `vim`, `git`, `nodejs`, `npm` in a Python web app image — attack surface
   expansion, image bloat. Fix: remove build tools, use multi-stage.

7. `COPY . /app` with no `.dockerignore` — copies `.git`, `.env`, `node_modules`, etc.

8. `RUN chmod 777 /app` — grants world-write to the app directory. Fix: `chmod 755` or
   proper ownership with `chown`.

9. `EXPOSE 22` — exposes SSH port, implying SSH into containers (anti-pattern). Fix: remove.

10. `CMD python3 app.py` — shell form, signal handling broken. Fix: `CMD ["python3", "app.py"]`.

11. No `USER` instruction — runs as root. Fix: create and use non-root user.

12. No `HEALTHCHECK` — Docker has no way to know if the app is actually functional.

13. No `WORKDIR` before `COPY` — files land in root directory if WORKDIR not set early.

**Write the corrected Dockerfile:**

```dockerfile
FROM python:3.12-slim

# Non-root user setup
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser

WORKDIR /app

# Dependencies first (cache efficiency)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application code
COPY app.py .

# Fix ownership
RUN chown -R appuser:appuser /app

# Switch to non-root
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

# Exec form for correct signal handling
CMD ["python", "app.py"]
```

Runtime secrets injected via environment, never baked in:
```bash
docker run -d \
  -e DATABASE_URL="postgresql://..." \
  -e API_KEY="sk-..." \
  myapp:v1
```

---

## Exercise 5.7 — The Full Production Readiness Checklist

**What you'll learn:** How to evaluate any container for production readiness systematically.

Run this against any image (`pyapp:prod` from Day 3 is a good target):

```bash
IMAGE=pyapp:prod

echo "=== PRODUCTION READINESS CHECK: $IMAGE ==="

echo ""
echo "--- 1. User (non-root?) ---"
docker inspect $IMAGE --format 'User: {{.Config.User}}'

echo ""
echo "--- 2. Health Check defined? ---"
docker inspect $IMAGE --format 'Healthcheck: {{json .Config.Healthcheck}}'

echo ""
echo "--- 3. Exposed ports ---"
docker inspect $IMAGE --format 'Ports: {{json .Config.ExposedPorts}}'

echo ""
echo "--- 4. Environment variables (check for secrets!) ---"
docker inspect $IMAGE --format '{{range .Config.Env}}{{println .}}{{end}}'

echo ""
echo "--- 5. Image size ---"
docker images $IMAGE --format 'Size: {{.Size}}'

echo ""
echo "--- 6. Number of layers ---"
docker history $IMAGE --quiet | wc -l

echo ""
echo "--- 7. Base image ---"
docker inspect $IMAGE --format 'Base: {{index .RepoDigests 0}}'

echo ""
echo "--- 8. Vulnerability scan (HIGH/CRITICAL) ---"
trivy image --severity HIGH,CRITICAL --quiet $IMAGE 2>/dev/null | tail -10

echo ""
echo "--- 9. CMD/Entrypoint form (exec vs shell) ---"
docker inspect $IMAGE --format 'CMD: {{json .Config.Cmd}}'
docker inspect $IMAGE --format 'Entrypoint: {{json .Config.Entrypoint}}'

echo ""
echo "--- 10. Labels ---"
docker inspect $IMAGE --format '{{json .Config.Labels}}'

echo ""
echo "=== END CHECKLIST ==="
```

---

## Day 5 Checkpoint

Answer these without looking:

1. How do you enter the namespaces of a container that has no shell?
2. What is the debug sidecar pattern and when do you use it?
3. How do you capture network traffic between two containers without installing tcpdump inside them?
4. What Linux capabilities does Docker grant by default? Why should you drop them in production?
5. What does `--read-only` do and how do you allow temporary file writes with it?
6. What is the PID 1 problem and what happens when PID 1 dies in a container?
7. Why is mounting `/var/run/docker.sock` into a container a critical security risk?
8. What does `OOMKilled: true` in `docker inspect` tell you?

---

## Day 5 Summary

| Concept | What You Proved |
|---|---|
| No-shell debugging | nsenter, debug sidecar with nicolaka/netshoot |
| Memory leak diagnosis | /proc, cgroup stats, OOM event tracking |
| Packet capture in containers | tcpdump via sidecar, veth capture on host |
| Capability hardening | --cap-drop ALL, --cap-add selective |
| Read-only filesystem | --read-only + --tmpfs for temp space |
| Production incident simulation | 10-step diagnostic runbook on a broken app |
| Dockerfile security audit | Found 12 anti-patterns in a real Dockerfile |
| Production readiness checklist | Automated evaluation script for any image |
