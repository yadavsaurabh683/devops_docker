# Day 5: Production Debugging, Advanced Operations, and Incident Simulation
## Theme: kubectl debug, OOM Forensics, Eviction, and Full Incident Response

**Goal:** By end of Day 5, you will have a complete production debugging toolkit for
Kubernetes. You will know how to debug pods without modifying them, investigate OOMKilled
containers, understand the eviction order under node pressure, and simulate a multi-failure
production incident and diagnose all failures simultaneously using only kubectl.

**Estimated time:** 6–7 hours (the incident simulation takes significant time — do not rush it)

---

## Background: Production Kubernetes Is Not Day 2 of the Tutorial

Production failures are never single-issue. They are compound. A database pod is OOMKilling,
which makes the API return 500s, which makes the frontend show errors. Diagnosing in the
wrong order wastes time. The engineers who are effective at 3 AM are the ones who have a
repeatable diagnostic framework, not the ones who know the most kubectl flags.

This day builds that framework.

---

## Exercise 5.1 — kubectl debug: Debug Without Modifying the Running Pod

**What you'll learn:** Ephemeral containers (Kubernetes 1.23+) and how to use `kubectl debug`
to inspect a running pod that has no shell or debugging tools.

**Instructions:**

1. Create a namespace:
```bash
kubectl create namespace day5
kubectl config set-context --current --namespace=day5
```

2. Deploy a distroless-style pod (minimal image, no shell):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: minimal-app
  namespace: day5
spec:
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF

kubectl get pods minimal-app -w
```

3. Try the normal debug approach — it fails for minimal images:
```bash
kubectl exec -it minimal-app -- sh
# This works for alpine, but for distroless images (like gcr.io/distroless/base):
# OCI runtime exec failed: exec failed: ... no such file or directory: unknown
```

4. Use kubectl debug with an ephemeral container — inject a debug shell into the running pod:
```bash
kubectl debug -it minimal-app --image=busybox --target=app -- sh
```

Inside the ephemeral container:
```bash
# You share the same network namespace as the app:
wget -O- http://localhost:80
# Nginx responds — we're in the app's network

# View the processes (requires shared PID namespace):
ps aux
# Lists processes in the pod

# Check the filesystem (shared process namespace gives access to /proc):
ls /proc/1/root    # View the app container's filesystem
cat /proc/1/cmdline | tr '\0' '\n'
```

Exit:
```bash
exit
```

5. Use `--copy-to` to debug a COPY of a failing pod (non-destructive debugging):
```bash
# Create a broken pod:
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: failing-app
  namespace: day5
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "echo 'App started'; exit 1"]
EOF

kubectl get pods failing-app -w
# Enters CrashLoopBackOff
```

6. Create a debugging copy — same spec but with your shell image replacing the container:
```bash
kubectl debug failing-app --copy-to=failing-app-debug --image=busybox \
  -it -- sh

# Inside: you have the same pod configuration but with a working shell
# You can examine environment variables, try to run the original command manually, etc.
env | sort
# See what environment the original app had

exit

# The copy stays running — clean it up when done:
kubectl delete pod failing-app-debug
```

7. Describe an ephemeral container:
```bash
kubectl describe pod minimal-app
# Look for "Ephemeral Containers" section — shows the debug container you attached
```

**Questions to answer:**
> 1. What is an ephemeral container?
>    (Answer: A special type of container that can be added to a running pod without restarting
>    it. It shares the pod's namespaces. Unlike regular containers, ephemeral containers cannot
>    be removed once added, and they are not automatically restarted. They are designed purely
>    for debugging.)
> 2. When would you use `kubectl debug --copy-to` vs an ephemeral container?
>    (Answer: Use ephemeral containers when you need to inspect the LIVE running pod in its
>    exact current state. Use `--copy-to` when the pod is in CrashLoopBackOff (exits before
>    you can attach) — the copy runs your debug shell instead of the crashing command, so you
>    can inspect the environment the app would run in.)
> 3. Why might `kubectl exec` fail on a distroless container?
>    (Answer: `kubectl exec` requires an executable to run — typically `/bin/sh`. Distroless
>    images contain only the application binary and its runtime dependencies — no shell, no
>    package manager, no utilities. The exec fails because the shell binary simply does not
>    exist in the container's filesystem.)

> **Hint:** Verify your cluster version supports ephemeral containers:
> `kubectl version --short | grep Server`
> Requires 1.23+. minikube with a recent start will be on a supported version.
> If not available, use the `--copy-to` approach instead.

---

## Exercise 5.2 — Investigating OOMKilled Pods

**What you'll learn:** How to find and diagnose an OOMKilled pod, read the kill reason from
kubectl, and trace the OOM event through the Kubernetes layers.

**Instructions:**

1. Create a pod that will be OOMKilled:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-victim
  namespace: day5
spec:
  containers:
  - name: mem-hog
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "Starting. Will allocate memory past the limit..."
      apk add --no-cache stress-ng 2>/dev/null
      stress-ng --vm 1 --vm-bytes 150M --timeout 60s
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "80Mi"
EOF

kubectl get pods oom-victim -w
# Watch: Running → OOMKilled → CrashLoopBackOff
```

2. Confirm the OOMKill reason:
```bash
kubectl describe pod oom-victim
# Look for:
# Last State:  Terminated
#   Reason:    OOMKilled
#   Exit Code: 137
#   Started:   ...
#   Finished:  ...

kubectl get pod oom-victim -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled

kubectl get pod oom-victim -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
# Output: 137  (128 + 9 = SIGKILL = kernel OOM kill)
```

3. Check memory usage with kubectl top:
```bash
kubectl top pods -n day5
# Shows current memory usage — pod may have restarted by now

# To see usage BEFORE the OOM, you would need external monitoring (Prometheus)
# kubectl top only shows current, not historical
```

4. Look for the OOM event on the minikube node:
```bash
minikube ssh

# Check kernel OOM kill log:
dmesg | grep -i "oom\|kill" | tail -20
# Shows: kernel: oom-kill event in cgroupv2... stress-ng was killed

# Check cgroup memory events:
PID=$(systemctl show -p MainPID --value minikube 2>/dev/null || echo "")
cat /sys/fs/cgroup/kubepods/burstable/*/memory.events 2>/dev/null | grep oom_kill
# Shows: oom_kill 1 (or higher if multiple kills)

exit
```

5. Read OOM details from events:
```bash
kubectl get events -n day5 --sort-by=.lastTimestamp | grep -i oom
# Shows: OOMKilling event with details

kubectl get events -n day5 -o wide | grep oom-victim
```

6. Clean up:
```bash
kubectl delete pod oom-victim
```

**Questions to answer:**
> 1. How do you confirm a pod was OOMKilled vs just crashed?
>    (Answer: Check `kubectl describe pod` for `Reason: OOMKilled` in the Last State section.
>    Or check the exit code: 137 = killed by SIGKILL = OOMKill. A process that exits with code
>    1 or any non-137 non-143 code crashed itself; 137 means the kernel killed it.)
> 2. What is the difference between memory limit exceeded and OOMKill?
>    (Answer: They are the same event triggered by different semantics. When a container's
>    memory usage exceeds its cgroup limit, the Linux kernel OOM killer selects and kills the
>    process. Kubernetes reports this as OOMKilled. You cannot "almost OOMKill" — the kernel
>    kills immediately when the limit is breached.)
> 3. What is the long-term fix for a pod that keeps getting OOMKilled?
>    (Answer: Either increase the memory limit (if the workload genuinely needs more memory),
>    or fix the memory leak in the application. Increasing limits without understanding the
>    root cause is only a temporary mitigation. Use tools like pprof, Pyspy, or Java heap
>    dumps to identify the allocation pattern causing the leak.)

---

## Exercise 5.3 — Node Pressure and Pod Eviction

**What you'll learn:** How kubelet evicts pods under resource pressure, the order eviction
follows (based on QoS class), and how PodDisruptionBudgets prevent too many pods from being
evicted simultaneously.

**Instructions:**

1. Create pods with different QoS classes to observe eviction ordering:
```bash
# BestEffort — evicted FIRST
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: evict-besteffort
  namespace: day5
  labels:
    qos: besteffort
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "while true; do sleep 10; done"]
    # No requests or limits = BestEffort
EOF

# Burstable — evicted SECOND
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: evict-burstable
  namespace: day5
  labels:
    qos: burstable
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "while true; do sleep 10; done"]
    resources:
      requests:
        memory: "32Mi"
      limits:
        memory: "128Mi"
EOF

# Guaranteed — evicted LAST
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: evict-guaranteed
  namespace: day5
  labels:
    qos: guaranteed
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "while true; do sleep 10; done"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "32Mi"
        cpu: "50m"
EOF

kubectl get pods -n day5
```

2. Verify the QoS class of each pod:
```bash
kubectl get pod evict-besteffort -n day5 -o jsonpath='{.status.qosClass}'
# BestEffort

kubectl get pod evict-burstable -n day5 -o jsonpath='{.status.qosClass}'
# Burstable

kubectl get pod evict-guaranteed -n day5 -o jsonpath='{.status.qosClass}'
# Guaranteed
```

3. Understand the eviction threshold — where kubelet draws the line:
```bash
# kubelet has configurable eviction thresholds. In minikube:
minikube ssh
cat /var/lib/kubelet/config.yaml | grep -A10 eviction
# Shows: evictionHard thresholds for memory.available, nodefs.available, etc.
exit

# When memory.available drops below threshold (default: 100Mi):
# 1. kubelet evicts BestEffort pods first (no reserved resources, so they're cheapest to kill)
# 2. Then Burstable pods that are using more than their requests
# 3. Then Guaranteed pods (only if node is critically pressured)
```

4. Create a PodDisruptionBudget to protect a deployment during voluntary disruptions:
```bash
# First create a deployment to protect:
kubectl create deployment protected-app --image=nginx:alpine --replicas=3 -n day5

cat <<'EOF' | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: protected-app-pdb
  namespace: day5
spec:
  minAvailable: 2       # At least 2 pods must be available at all times
  selector:
    matchLabels:
      app: protected-app
EOF

kubectl get pdb -n day5
# Shows: NAME, MIN AVAILABLE, MAX UNAVAILABLE, ALLOWED DISRUPTIONS, AGE
```

5. Test the PDB by trying to drain the node (cordon + evict):
```bash
# Cordon: mark the node as unschedulable (no new pods will be placed here)
kubectl cordon minikube

# Try to evict pods (drain):
kubectl drain minikube --ignore-daemonsets --delete-emptydir-data
# The PDB prevents draining past 2 replicas remaining
# You'll see: "Cannot evict pod as it would violate the pod's disruption budget."

# Restore the node:
kubectl uncordon minikube
```

6. Clean up:
```bash
kubectl delete pod evict-besteffort evict-burstable evict-guaranteed -n day5
kubectl delete pdb protected-app-pdb -n day5
kubectl delete deployment protected-app -n day5
```

**Questions to answer:**
> 1. What is the eviction order by QoS class?
>    (Answer: BestEffort pods are evicted first (no resource reservations),
>    then Burstable pods that are using more memory than their requests,
>    then Burstable pods within their requests,
>    then Guaranteed pods — only if the node is critically short of memory.)
> 2. What is a PodDisruptionBudget?
>    (Answer: A PDB defines the minimum number of pods that must remain available during
>    voluntary disruptions — node drains, rolling updates, cluster upgrades. It prevents
>    tools like kubectl drain from evicting so many pods that the service becomes unavailable.
>    It has no effect on involuntary disruptions like node crashes.)
> 3. What is the difference between voluntary and involuntary disruption in Kubernetes?
>    (Answer: Voluntary: planned operations like node drain, rolling updates, or manual pod
>    deletion. PDBs protect against these. Involuntary: node crash, kernel panic, OOM kill.
>    PDBs do not protect against these — there is no time to consult the PDB when hardware
>    fails.)

---

## Exercise 5.4 — Advanced kubectl Debugging Toolkit

**What you'll learn:** A systematic set of kubectl commands for every production scenario.
This exercise is a guided drill — run each command and understand what it tells you.

**Instructions:**

1. Find what happened and when — events sorted by time:
```bash
# Deploy something first:
kubectl create deployment debug-demo --image=nginx:alpine -n day5
kubectl expose deployment debug-demo --port=80 -n day5

# Get events sorted by timestamp (most useful first diagnostic):
kubectl get events -n day5 --sort-by=.lastTimestamp

# Filter events for a specific resource:
kubectl get events -n day5 --field-selector involvedObject.name=debug-demo
```

2. Find which node a pod is on and why it was placed there:
```bash
kubectl get pods -n day5 -o wide
# NODE column shows the node

# Describe the pod to see scheduling reason:
POD_NAME=$(kubectl get pods -n day5 -l app=debug-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME -n day5
# Look at: Node, Events (Scheduled)

# Inspect the node for capacity:
kubectl describe node minikube | grep -A30 "Allocated resources"
```

3. Port-forward to debug a service not exposed externally:
```bash
# Forward local port 8888 to the service port 80:
kubectl port-forward svc/debug-demo 8888:80 -n day5 &
PF_PID=$!

# Test from your local machine:
curl http://localhost:8888
# Nginx responds — you're directly proxied to the pod via kubectl

# Stop port-forward:
kill $PF_PID 2>/dev/null
```

4. Test service-to-service connectivity from inside a pod:
```bash
# This is the MOST USEFUL debugging technique for service mesh problems:
kubectl run curl-test --image=curlimages/curl --restart=Never -it --rm -n day5 \
  -- sh -c '
    echo "=== Testing by service name ==="
    curl -v -s --max-time 5 debug-demo.day5.svc.cluster.local 2>&1 | head -20

    echo "=== Testing by ClusterIP ==="
    SVC_IP=$(nslookup debug-demo.day5.svc.cluster.local | grep Address | tail -1 | awk "{print \$2}")
    curl -s --max-time 5 http://$SVC_IP

    echo "=== DNS resolution check ==="
    cat /etc/resolv.conf
  '
```

5. Copy files from a crashing pod for offline analysis:
```bash
POD_NAME=$(kubectl get pods -n day5 -l app=debug-demo -o jsonpath='{.items[0].metadata.name}')

# Copy a file out of the container:
kubectl cp day5/$POD_NAME:/etc/nginx/nginx.conf ./nginx.conf.backup
cat ./nginx.conf.backup | head -20

# Copy a directory out:
kubectl cp day5/$POD_NAME:/etc/nginx ./nginx-config-backup/

# Copy a file INTO the container:
echo "custom" > /tmp/custom.txt
kubectl cp /tmp/custom.txt day5/$POD_NAME:/tmp/custom.txt
kubectl exec -n day5 $POD_NAME -- cat /tmp/custom.txt
```

6. Debug a pod that restarts too fast to exec into it:
```bash
# Create a fast-crashing pod:
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fast-crash
  namespace: day5
spec:
  containers:
  - name: crasher
    image: alpine
    command: ["sh", "-c", "echo 'crash' && exit 1"]
EOF

# Too fast to exec — use logs --previous instead:
kubectl logs fast-crash --previous -n day5

# Or use kubectl debug --copy-to to get a stable shell:
kubectl debug fast-crash --copy-to=fast-crash-debug -n day5 \
  --image=alpine -- sh -c "env; ls /; sleep 3600" &
sleep 5
kubectl get pod fast-crash-debug -n day5
kubectl exec -n day5 fast-crash-debug -- env | grep -i app

kubectl delete pod fast-crash-debug -n day5
```

**Questions to answer:**
> 1. How do you find which node a pod is on, and how do you get node-level details?
>    (Answer: `kubectl get pods -o wide` shows the NODE column. `kubectl describe node <name>`
>    shows the node's capacity, allocatable resources, running pods, and events.)
> 2. How do you debug a pod that keeps restarting before you can exec into it?
>    (Answer: Three options: (1) `kubectl logs --previous` to read the last crashed container's
>    logs. (2) `kubectl debug --copy-to` to create a copy with your debug shell replacing the
>    crashing command. (3) `kubectl describe pod` to read the Events and Last State fields —
>    often enough to determine the cause without needing a shell.)
> 3. When would you use `kubectl cp` vs `kubectl exec cat <file>`?
>    (Answer: `kubectl cp` is for binary files, large files, or directories where piping would
>    corrupt the content. `kubectl exec cat` is fine for small plaintext files you want to
>    read in the terminal. For structured analysis of logs or configs, `kubectl cp` to a local
>    file and use local tools.)

---

## Exercise 5.5 — Full Production Incident Simulation

**What you'll learn:** Multi-failure diagnosis and remediation. This simulates a real
production scenario with three simultaneous failures. You must find and fix all three
in the correct order.

**Instructions — Setup (Read This First, Then Run):**

You will deploy a 3-tier app with three hidden defects already baked in:
1. The database pod has a memory limit that's too low — it will OOMKill
2. The frontend's Service selector is wrong — no endpoints
3. The API has a wrong secret reference — it cannot start

Do not look ahead at the defects. Set up the environment, then diagnose from scratch.

**Setup — Deploy the broken 3-tier app:**
```bash
kubectl create namespace incident

# 1. Database (Postgres simulation — nginx pretending to be a DB)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: incident
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: db
        image: alpine
        command:
        - sh
        - -c
        - |
          apk add --no-cache stress-ng 2>/dev/null
          # Simulate a DB that uses more memory than its limit allows:
          stress-ng --vm 1 --vm-bytes 200M --timeout 9999s
        resources:
          requests:
            memory: "64Mi"
          limits:
            memory: "90Mi"   # Too low — will OOMKill
---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: incident
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
EOF

# 2. Secret that the API needs (intentionally wrong key name):
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: incident
stringData:
  DATABASE_URL: "postgres://admin:password@database-svc:5432/appdb"
  # Note: the key is DATABASE_URL, but the API will look for DB_CONNECTION_STRING
EOF

# 3. API (references a wrong secret key — will fail to start):
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: incident
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: alpine
        command: ["sh", "-c", "echo DB=$DB_CONNECTION_STRING; sleep 3600"]
        env:
        - name: DB_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: DB_CONNECTION_STRING   # WRONG KEY — secret has DATABASE_URL, not this
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: incident
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
EOF

# 4. Frontend (correct app, wrong service selector):
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: incident
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: incident
spec:
  selector:
    app: frontend-web   # WRONG: pods have app=frontend, not app=frontend-web
  ports:
  - port: 80
    targetPort: 80
EOF

echo "Broken 3-tier app deployed. Begin diagnosis."
```

**Now diagnose — do not look at the setup YAML above. Start from scratch:**

```bash
# Step 1: Get the overall state:
kubectl get all -n incident

# Step 2: Find events (what has happened so far):
kubectl get events -n incident --sort-by=.lastTimestamp

# Step 3: Diagnose each component:
kubectl describe deployment database -n incident
kubectl describe deployment api -n incident
kubectl describe deployment frontend -n incident

kubectl describe pods -n incident
kubectl get endpoints -n incident
```

**Guided diagnosis and fix — work through this yourself, then compare:**

**Problem 1: Database OOMKilling**
```bash
kubectl get pods -n incident -l app=database
# Status: OOMKilled / CrashLoopBackOff

kubectl describe pod -n incident -l app=database | grep -A5 "Last State"
# Reason: OOMKilled, ExitCode: 137

# Fix: increase memory limit
kubectl set resources deployment/database -n incident \
  --limits=memory=300Mi --requests=memory=128Mi

kubectl get pods -n incident -l app=database -w
# Watch it stabilize (no more OOMKill)
```

**Problem 2: API pods failing to start (wrong secret key)**
```bash
kubectl get pods -n incident -l app=api
# Status: CreateContainerConfigError

kubectl describe pod -n incident -l app=api | grep -A10 "Events:"
# Event: Error: couldn't find key DB_CONNECTION_STRING in Secret incident/api-secrets

# Check what key the secret actually has:
kubectl get secret api-secrets -n incident -o jsonpath='{.data}' | python3 -c "import json,sys; print(list(json.load(sys.stdin).keys()))"
# ['DATABASE_URL']

# Fix: update the env var to use the correct key name
kubectl set env deployment/api -n incident \
  DB_CONNECTION_STRING-  # remove old

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: incident
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: alpine
        command: ["sh", "-c", "echo DB=$DB_CONNECTION_STRING; sleep 3600"]
        env:
        - name: DB_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: DATABASE_URL    # corrected key name
EOF

kubectl rollout status deployment/api -n incident
```

**Problem 3: Frontend service has no endpoints**
```bash
kubectl get endpoints frontend-svc -n incident
# Endpoints: <none>

kubectl describe service frontend-svc -n incident
# Selector: app=frontend-web

kubectl get pods -n incident -l app=frontend --show-labels
# Labels: app=frontend (not frontend-web)

# Fix: patch the service selector
kubectl patch service frontend-svc -n incident \
  -p '{"spec":{"selector":{"app":"frontend"}}}'

kubectl get endpoints frontend-svc -n incident
# Now shows pod IPs
```

**Verify all three are fixed:**
```bash
kubectl get all -n incident
kubectl get endpoints -n incident
kubectl get events -n incident --sort-by=.lastTimestamp | tail -20
```

**Clean up:**
```bash
kubectl delete namespace incident
kubectl delete namespace day5
kubectl config set-context --current --namespace=default
```

**Questions to answer:**
> 1. What was the fastest single command to identify all problems?
>    (Answer: `kubectl get events -n incident --sort-by=.lastTimestamp` shows the OOMKill
>    events, the CreateContainerConfigError from the secret key mismatch, and the scheduling
>    events all in chronological order. This is always the first command in a multi-failure
>    incident.)
> 2. In what order should you fix a database + API + frontend failure?
>    (Answer: Database first — the database is the dependency for everything else. Fixing the
>    API before the database is stable just moves the failure. Then fix the API (which depends
>    on the database). Then fix the frontend (which depends on the API). Always fix from
>    the data layer up to the presentation layer.)
> 3. What would you add to prevent these three failures in the future?
>    (Answer: (1) Database: set proper memory limits based on measured usage, add a readiness
>    probe so the database only accepts traffic when actually ready. (2) API: use `kubectl auth
>    can-i` checks in CI to validate secret key references, or use external secret validation.
>    (3) Frontend: validate service selectors in CI with a dry-run — `kubectl apply --dry-run=server`
>    will catch label mismatches before deployment.)

---

## Day 5 Checkpoint

Before completing the course, answer these without looking at the material:

1. What is an ephemeral container and how is it different from a regular container?
2. How do you confirm a pod was OOMKilled vs exiting with a bug? What exit code distinguishes them?
3. What is the eviction order under node memory pressure?
4. What is a PodDisruptionBudget and what type of disruptions does it protect against?
5. If a pod keeps crashing before you can exec into it, what are your three diagnostic options?
6. What does `kubectl get events --sort-by=.lastTimestamp` show you?
7. What single command would you run FIRST when you get paged about a production incident?

---

## Day 5 Summary

| Concept | What You Proved |
|---|---|
| Ephemeral containers | Injected a debug shell into a running pod without restarting it |
| --copy-to debugging | Created a stable debug copy of a CrashLoopBackOff pod |
| OOMKilled forensics | Found OOMKill via Reason: OOMKilled, exit code 137, and dmesg |
| Eviction order | BestEffort → Burstable → Guaranteed under memory pressure |
| PodDisruptionBudget | Prevented kubectl drain from removing too many pods |
| Port-forward debugging | Reached an internal service from localhost without NodePort |
| kubectl cp | Extracted container files for offline analysis |
| Incident simulation | Three simultaneous failures: OOM, secret key mismatch, label selector |
| Fix order matters | Database layer fixed first, then API, then frontend |
