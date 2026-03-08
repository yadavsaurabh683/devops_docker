# Day 1: What Is a Container, Really?
## Theme: Foundations, Mental Models, and Debugging from the Start

**Goal:** By end of Day 1, you will understand what a container actually IS at the Linux kernel
level, and you will never again confuse a container with a VM. You will also write your first
non-trivial Dockerfile and debug a broken one.

**Estimated time:** 5–6 hours

---

## Background: The Mental Model That Changes Everything

A container is NOT a mini VM. It is a **group of Linux processes** that are isolated using
kernel features called **namespaces** and **constrained** using **cgroups**.

That's it. No hypervisor. No guest OS. Just your process, with a restricted view of the system.

Before every exercise, ask yourself: "What Linux feature makes this work?"

---

## Exercise 1.1 — Prove That a Container Is Just a Process

**What you'll learn:** The relationship between container processes and the host kernel.

**Instructions:**

1. Start a long-running container:
```bash
docker run -d --name mybox alpine sleep 600
```

2. On your **host machine**, list all running processes:
```bash
ps aux | grep sleep
```

3. Note the PID you see on the host. Now enter the container:
```bash
docker exec -it mybox sh
ps aux
```

4. Inside the container, `sleep` appears as PID 1 (or small number). On the host it has a
   large PID. Same process — different views.

5. Now, from your **host**, kill the sleep process using the HOST PID:
```bash
kill <host-pid-of-sleep>
```

6. Run `docker ps -a` — the container is dead. You killed the container from outside by killing
   its process.

**Question to answer before moving on:**
> If a container is just a process, what stops that process from seeing and killing OTHER
> processes on the host? (Answer: PID namespaces — the container has its own PID space.)

---

## Exercise 1.2 — Namespaces: Seeing the Isolation Layers

**What you'll learn:** What namespaces are and which ones Docker uses.

**Instructions:**

1. Start a container:
```bash
docker run -d --name nstest nginx:alpine
```

2. Find the container's main process PID on the host:
```bash
docker inspect nstest --format '{{.State.Pid}}'
```
Save this as `CPID`.

3. View the namespaces of that process:
```bash
ls -la /proc/$CPID/ns/
```

You'll see entries like `net`, `pid`, `mnt`, `uts`, `ipc`, `user`. Each file is a handle to
an isolated namespace.

4. Compare with a host process PID (e.g., PID 1):
```bash
ls -la /proc/1/ns/
```

Different inode numbers = different namespaces. Same inode = shared namespace.

5. Now run a container with `--network host` and repeat the comparison for the `net` namespace:
```bash
docker run -d --name nstest2 --network host nginx:alpine
CPID2=$(docker inspect nstest2 --format '{{.State.Pid}}')
ls -la /proc/$CPID2/ns/net
ls -la /proc/1/ns/net
```

**Observation:** With `--network host`, the net namespace inode matches the host. That's why
host-network containers can bind to host ports directly — they ARE the host network.

> **Hint:** On Linux, `lsns` command (from util-linux) gives you a clean view of all namespaces.
> Try: `lsns -p $CPID`

---

## Exercise 1.3 — cgroups: Resource Limits Are Real

**What you'll learn:** How Docker enforces CPU/memory limits using cgroups.

**Instructions:**

1. Run a container with memory limit:
```bash
docker run -d --name memlimit --memory=50m alpine sleep 600
```

2. Find its cgroup on the host:
```bash
# For cgroups v2 (modern Linux):
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect memlimit --format '{{.Id}}').scope/memory.max

# For cgroups v1:
cat /sys/fs/cgroup/memory/docker/$(docker inspect memlimit --format '{{.Id}}')/memory.limit_in_bytes
```

3. The number you see (52428800 = 50 MB in bytes) is the kernel-enforced limit. If the
   process exceeds this, the OOM killer fires.

4. Simulate OOM — run a stress test:
```bash
docker run --rm --memory=30m --name oomtest alpine sh -c '
  apk add --no-cache stress-ng 2>/dev/null
  stress-ng --vm 1 --vm-bytes 100M --timeout 10s
'
```

5. Check what happened:
```bash
docker inspect oomtest 2>/dev/null || docker events --filter event=oom --since 5m
```

**Expected:** Container is killed. Check with `docker inspect --format '{{.State.OOMKilled}}'`
if the container still exists.

> **Hint:** `docker stats` (run in a second terminal while the stress test runs) shows live
> memory usage and lets you watch the OOM event happen in real time.

---

## Exercise 1.4 — Your First Real Dockerfile (Not a Tutorial Dockerfile)

**What you'll learn:** Dockerfile fundamentals, image layers, cache behavior.

**Scenario:** A developer hands you this Dockerfile for a Python web app and says "it works but
the image is 800 MB and builds take forever." Your job: understand why, then fix it.

**Setup — create the broken app:**
```bash
mkdir -p ~/docker-day1/app
cd ~/docker-day1/app
```

Create `app.py`:
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Container World!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Create `requirements.txt`:
```
flask==3.0.0
```

Create the BAD Dockerfile (`Dockerfile.bad`):
```dockerfile
FROM python:3.12
RUN apt-get update
RUN apt-get install -y curl vim git wget
COPY . .
RUN pip install -r requirements.txt
ENV FLASK_APP=app.py
CMD python app.py
```

**Step 1 — Build and observe:**
```bash
docker build -f Dockerfile.bad -t badapp:v1 .
docker images badapp
```

Note the image size.

**Step 2 — Understand the layer problem:**
```bash
docker history badapp:v1
```

Each `RUN` is a layer. The apt installs, even if removed later, are baked in.

**Step 3 — Now change ONE line of app.py (anything tiny) and rebuild:**
```bash
docker build -f Dockerfile.bad -t badapp:v2 .
```

Notice: `pip install` runs again from scratch because `COPY . .` copied everything before it,
invalidating the cache.

**Step 4 — Write the GOOD Dockerfile (`Dockerfile.good`):**
```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Copy dependencies FIRST — this layer is cached unless requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code AFTER deps — cache-friendly
COPY app.py .

ENV FLASK_APP=app.py
EXPOSE 5000

# Use exec form (not shell form) for correct signal handling
CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

**Step 5 — Build and compare:**
```bash
docker build -f Dockerfile.good -t goodapp:v1 .
docker images | grep -E 'badapp|goodapp'
```

**Questions to answer:**
> 1. Why does `COPY requirements.txt .` before `COPY app.py .` matter for build speed?
> 2. What's the difference between `CMD python app.py` and `CMD ["python", "app.py"]`?
>    (Hint: check signal handling — shell form wraps in `/bin/sh -c`, masking SIGTERM)
> 3. Why is `python:3.12-slim` significantly smaller than `python:3.12`?

---

## Exercise 1.5 — Debugging a Broken Container Without Restarting It

**What you'll learn:** Debugging containers using `docker exec`, `docker logs`, `docker inspect`.

**Scenario:** You receive a report that a container is "running but not responding." Fix it
without stopping it.

**Setup — create a broken service:**
```bash
docker run -d --name brokenapp \
  -e PORT=8080 \
  -p 8080:8080 \
  nginx:alpine \
  sh -c 'nginx -g "daemon off;" 2>&1 | tee /tmp/app.log &
         sleep 5
         pkill nginx
         sleep 9999'
```

Wait 8 seconds, then:
```bash
curl http://localhost:8080
```

It fails. Now investigate **without killing the container**:

**Step 1 — Check logs:**
```bash
docker logs brokenapp
docker logs --tail 20 brokenapp
```

**Step 2 — Check what's running inside:**
```bash
docker exec brokenapp ps aux
```

**Step 3 — Inspect the container metadata:**
```bash
docker inspect brokenapp | python3 -m json.tool | less
```

Look at: `State`, `NetworkSettings.Ports`, `Mounts`, `Config.Env`.

**Step 4 — Check network bindings from inside:**
```bash
docker exec brokenapp ss -tlnp 2>/dev/null || docker exec brokenapp netstat -tlnp 2>/dev/null
```

**Step 5 — Read the log file inside the container:**
```bash
docker exec brokenapp cat /tmp/app.log
```

**What you should find:** nginx was started, then killed by the script. Port 8080 is not bound.
The process is just sleeping. Classic case of a container "running" but the actual service
having crashed internally.

> **Key insight:** Docker only monitors PID 1 (the shell in this case). If your app crashes
> but PID 1 survives, Docker reports the container as "running." This is why health checks
> are critical in production.

---

## Day 1 Checkpoint

Before moving to Day 2, you must be able to answer these without looking:

1. What kernel feature provides process isolation in containers?
2. What kernel feature provides resource limits in containers?
3. What does it mean when two processes share a namespace inode?
4. Why does layer order in a Dockerfile matter for build performance?
5. What is the PID 1 problem in containers?
6. What's the difference between `docker logs` and `docker exec cat /some/log`?
7. Why can `--network host` be a security concern?

---

## Day 1 Summary

| Concept | What You Proved |
|---|---|
| Containers are processes | Killed container by killing host PID |
| Namespaces provide isolation | Saw different PID/net namespace inodes |
| cgroups enforce limits | Triggered OOM killer with memory cap |
| Dockerfile layer caching | Measured cache hit vs miss on rebuild |
| Debugging live containers | Diagnosed broken service without restart |
