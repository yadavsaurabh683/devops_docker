# Docker Mastery Course
### A 5-Day Hands-On Program for DevOps Engineers

---

## What This Course Is

This is not a tutorial. It is a **problem-first, exercise-driven program** designed to develop
the deep container intuition that separates senior platform engineers from average DevOps
engineers. Every exercise simulates a real DevOps or platform engineering scenario.

By the end of Day 5, you will:
- Understand what a container **actually is** at the Linux kernel level
- Debug any container failure without restarting it
- Build minimal, secure, production-grade images
- Design multi-service topologies with proper networking and health checks
- Speak about container concepts with expert-level confidence

---

## Prerequisites

You do not need advanced Docker knowledge. You need:

- A Linux machine or Linux VM (WSL2 on Windows works fine)
- Basic Linux command line comfort (cd, ls, cat, grep, ps)
- Docker Desktop or Docker Engine installed
- Git installed
- An internet connection for pulling images

> **Windows users:** All exercises are written for Linux/macOS shell.
> Use **WSL2** (Windows Subsystem for Linux) with Docker Desktop's WSL2 backend.
> Open a WSL2 terminal and run everything from there.

---

## Tool Installation

Install all tools before starting Day 1. Run these from your Linux/WSL2 terminal:

### 1. Docker
```bash
# Verify Docker is installed and running:
docker version
docker info

# If not installed:
# https://docs.docker.com/engine/install/
```

### 2. Docker Compose (v2)
```bash
docker compose version
# Should show: Docker Compose version v2.x.x
# If missing, install Docker Desktop or docker-compose-plugin
```

### 3. Trivy (image vulnerability scanner) — needed for Day 3
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sh -s -- -b /usr/local/bin
trivy version
```

### 4. Dive (image layer explorer) — needed for Day 3
```bash
# Grab latest release:
DIVE_VER=$(curl -s https://api.github.com/repos/wagoodman/dive/releases/latest \
  | grep tag_name | cut -d '"' -f4 | tr -d v)

wget -q https://github.com/wagoodman/dive/releases/download/v${DIVE_VER}/dive_${DIVE_VER}_linux_amd64.tar.gz \
  -O /tmp/dive.tar.gz

tar -xf /tmp/dive.tar.gz -C /tmp
sudo mv /tmp/dive /usr/local/bin/dive
dive version
```

### 5. Network debugging tools (most are already in any Linux distro)
```bash
# These should already be available:
ip --version
ss --version
tcpdump --version

# Install if missing (Ubuntu/Debian):
sudo apt-get install -y iproute2 iputils-ping netcat-openbsd tcpdump

# Install if missing (Alpine inside containers):
# apk add --no-cache iproute2 iputils bind-tools
```

### 6. nicolaka/netshoot (debug container image) — pull in advance
```bash
docker pull nicolaka/netshoot
```

### 7. Verify your setup is complete
```bash
echo "=== Docker ===" && docker version --format '{{.Client.Version}}'
echo "=== Compose ===" && docker compose version --short
echo "=== Trivy ===" && trivy version | head -1
echo "=== Dive ===" && dive version
echo "=== Netshoot ===" && docker run --rm nicolaka/netshoot echo "netshoot ok"
echo ""
echo "All tools ready. You can start Day 1."
```

---

## Course Structure

```
docker-mastery-course/
├── README.md              ← You are here
├── SUMMARY.md             ← End-of-course concept summary (read on Day 5 completion)
├── REVISION-NOTES.md      ← Quick revision cheat sheet for all 5 days
├── FINAL-CHALLENGES.md    ← Complex questions to test your knowledge
│
├── day1/
│   └── README.md          ← Containers as processes, namespaces, cgroups, Dockerfile basics
├── day2/
│   └── README.md          ← Docker networking internals, volumes, storage lifecycle
├── day3/
│   └── README.md          ← Image engineering, multi-stage builds, secrets, security scanning
├── day4/
│   └── README.md          ← Docker Compose, multi-service topology, failure simulation
└── day5/
    └── README.md          ← Production debugging, security hardening, incident simulation
```

---

## How to Go Through This Course

### Rule 1: Do Every Exercise — No Skipping
Reading exercises without running them will not give you the intuition this course is designed
to build. The "aha" moment only comes when you see the output yourself.

### Rule 2: Read the Questions at the End of Each Exercise
Every exercise has **"Questions to answer"** sections. Stop. Think. Write down your answer
before moving on. If you cannot answer it, re-read the exercise output and trace back why.

### Rule 3: Before Looking at Hints — Try First
Every exercise has `> Hint:` callouts. These are for when you are genuinely stuck for more
than 10 minutes. Try the exercise on your own first. Being stuck and figuring it out is where
learning happens.

### Rule 4: Do the Day Checkpoint at the End of Each Day
Each day ends with a **Checkpoint** — a list of questions you must answer without looking.
If you cannot answer them, do not proceed to the next day. Revisit the exercise that covers
the concept.

### Rule 5: Use `docker inspect` and `docker events` Constantly
Whenever something happens — a container starts, a network is created, a port is published —
run `docker inspect` and `docker events`. Every answer about Docker's behavior is in there.

---

## Day-by-Day Overview

| Day | Theme | Key Skills |
|-----|-------|------------|
| **Day 1** | What is a container, really? | Namespaces, cgroups, OverlayFS, Dockerfile fundamentals, live debugging |
| **Day 2** | Networking and Storage | veth pairs, iptables NAT, DNS, bridge networks, volumes, segmentation |
| **Day 3** | Image Engineering | Multi-stage builds, secrets hygiene, .dockerignore, scanning with Trivy |
| **Day 4** | Docker Compose | Health-based startup, multi-service topology, restart policies, failure simulation |
| **Day 5** | Production Debugging & Security | nsenter, sidecar debugging, capability hardening, incident simulation |

---

## Estimated Time Per Day

Each day is designed for **5–6 focused hours**. If you rush, you will memorize commands.
If you go deep, you will build intuition.

Suggested schedule:
- Morning (2–3 hours): Complete exercises up to the midpoint of the day
- Afternoon (2–3 hours): Complete the remaining exercises + Day Checkpoint
- Evening (30 min): Review the Day Summary and answer checkpoint questions from memory

---

## Common Mistakes to Avoid

- **Do not** run `docker restart` when something breaks. Diagnose first.
- **Do not** skip the networking exercises on Day 2. They underpin everything else.
- **Do not** treat Compose as just a dev tool — design it like it's production.
- **Do not** copy-paste Dockerfiles without understanding each instruction.
- **Do not** ignore security exercises. Security is operational hygiene, not a specialty track.

---

## After Completing This Course

Once you finish all 5 days and can answer the Final Challenges without hesitation, you will:

1. Be able to onboard to any project that uses Docker or containers
2. Diagnose container failures in production without guessing or restarting
3. Review and critique Dockerfiles as part of code review
4. Design container architectures that account for networking, storage, and failure modes
5. Understand what Kubernetes is managing under the hood (containers, volumes, networks)
   — making your Kubernetes learning dramatically faster

---

## Quick Reference: Most Useful Commands

```bash
# Inspect EVERYTHING about a container:
docker inspect <container>

# See live resource usage:
docker stats

# Follow logs with timestamps:
docker logs -f --timestamps <container>

# Enter a running container:
docker exec -it <container> sh

# Enter a containernamespace from host (no shell needed):
sudo nsenter -t $(docker inspect <container> -f '{{.State.Pid}}') --mount --net --pid -- sh

# Capture traffic on a container's network interface:
docker run -it --rm --network=container:<name> nicolaka/netshoot tcpdump -i eth0

# Scan image for vulnerabilities:
trivy image --severity HIGH,CRITICAL <image>

# Show image layers and sizes:
docker history <image>
dive <image>

# Watch Docker daemon events in real time:
docker events

# Check which process is PID 1 inside a container:
docker exec <container> cat /proc/1/cmdline | tr '\0' ' '

# Check OOM kill status:
docker inspect <container> --format '{{.State.OOMKilled}}'
```

---

## Getting Help

If an exercise output does not match what you expect, check in this order:
1. `docker inspect <container>` — are the config/state values what you expect?
2. `docker logs <container>` — is the process reporting an error?
3. `docker events --since 5m` — did something unexpected happen?
4. The `> Hint:` block in the exercise — is there a platform-specific note?

If you are on cgroups v1 vs v2, some paths differ:
- cgroups v1: `/sys/fs/cgroup/memory/docker/<full-container-id>/`
- cgroups v2: `/sys/fs/cgroup/system.slice/docker-<full-container-id>.scope/`

Check which version your system uses:
```bash
stat -f --format="%T" /sys/fs/cgroup
# Returns: tmpfs = v1, cgroup2fs = v2
```

---

*This course uses only open-source tools. No paid services, no cloud accounts required.*
