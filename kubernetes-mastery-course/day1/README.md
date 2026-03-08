# Day 1: What Kubernetes Actually Is
## Theme: Architecture, Pods, and the Control Loop

**Goal:** By end of Day 1, you will understand what Kubernetes actually does at the systems
level. You will understand the control loop, the role of kubelet, what a pod really is, and
why CrashLoopBackOff happens and how to diagnose it. You will never again treat Kubernetes
as a black box that "runs containers."

**Estimated time:** 5–6 hours

---

## Background: The Mental Model That Changes Everything

Kubernetes is not a container runner. It is a **distributed reconciliation system**.

Every component in Kubernetes — the scheduler, the controller-manager, the kubelet — is
running the same basic loop:
1. Read the desired state from etcd (via the API server)
2. Observe the current state of the world
3. Take the minimum action needed to make current state match desired state
4. Repeat

This loop never stops. It runs while you sleep. This is why Kubernetes self-heals: it is
constantly comparing what you asked for with what it sees, and acting on the difference.

Every exercise today is about observing this loop in action.

---

## Exercise 1.1 — Prove Kubernetes Is Just Running Containers

**What you'll learn:** The relationship between a Kubernetes Pod and the underlying container
runtime. Show that a pod IS a container, just with Kubernetes metadata wrapped around it.

**Instructions:**

1. Start minikube if not already running:
```bash
minikube start --cpus=2 --memory=4096
kubectl get nodes
# Expected: minikube   Ready   control-plane   ...
```

2. Deploy a simple pod:
```bash
kubectl run nginx-test --image=nginx:alpine --restart=Never
kubectl get pods
kubectl get pods -o wide
# Note the NODE column — it's the minikube node
```

3. Describe the pod to see its full state:
```bash
kubectl describe pod nginx-test
# Read every section: Node, Containers, Conditions, Events
```

4. SSH into the minikube node and find the actual container:
```bash
minikube ssh

# Inside the minikube node, list containers via crictl (the container runtime interface):
sudo crictl ps | grep nginx

# You will see the nginx container running. Note the container ID.
sudo crictl inspect <container-id> | grep -A5 '"name"'

# Also check the pause container (the "sandbox" that holds the pod's network namespace):
sudo crictl ps -a | grep pause
```

5. Compare: the container you see in crictl is the SAME process that kubectl describes as
   a pod. Exit the minikube SSH session:
```bash
exit
```

6. Verify from outside:
```bash
kubectl get pod nginx-test -o jsonpath='{.status.containerStatuses[0].containerID}'
# Output: containerd://abc123...  — this ID matches what crictl showed
```

**Questions to answer before moving on:**
> 1. What is the difference between a Pod and a container?
>    (Answer: A pod is a Kubernetes abstraction. It contains one or more containers that share
>    the same network namespace and storage volumes. The container is the actual running process.)
> 2. What does kubelet actually do?
>    (Answer: kubelet is the agent on each node. It watches for pods scheduled to its node via
>    the API server, then uses the container runtime — via CRI — to start, stop, and monitor
>    the containers. It also reports pod status back to the API server.)
> 3. What is the pause container and why does every pod have one?
>    (Answer: The pause container holds the pod's network namespace. Other containers in the pod
>    join this namespace. This is how all containers in a pod share the same IP address.)

> **Hint:** If crictl is not found, try `docker ps` inside the minikube node — some minikube
> drivers use docker as the container runtime instead of containerd.

---

## Exercise 1.2 — The Control Loop: Break It and Watch It Heal

**What you'll learn:** What "desired state" and "current state" mean in practice. How the
reconciliation loop responds to state changes.

**Instructions:**

1. Create a Deployment with 3 replicas:
```bash
kubectl create deployment nginx-demo --image=nginx:alpine --replicas=3
kubectl get pods -w
# Watch pods come up. Press Ctrl+C when all 3 are Running.
```

2. Delete one pod directly — do NOT delete the Deployment:
```bash
# Get a pod name:
kubectl get pods -l app=nginx-demo

# Delete one pod:
kubectl delete pod <one-pod-name>

# Immediately watch what happens:
kubectl get pods -w
```

You will see the deleted pod terminate, and within seconds a new pod appears. The Deployment
controller detected that current replicas (2) did not match desired replicas (3) and created
a new pod.

3. Scale down to 0 replicas, then back up:
```bash
kubectl scale deployment nginx-demo --replicas=0
kubectl get pods -w
# All pods terminate

kubectl scale deployment nginx-demo --replicas=3
kubectl get pods -w
# 3 new pods appear
```

4. Now kill a container from INSIDE the minikube node directly — bypass kubectl entirely:
```bash
minikube ssh

# Find the nginx container ID:
sudo crictl ps | grep nginx

# Kill one of the containers using crictl:
sudo crictl stop <container-id>

exit
```

5. Watch Kubernetes respond:
```bash
kubectl get pods -w
# You will see a pod's restart count increment, or a new pod replace it.
# The kubelet detected the container died and either restarted it or reported failure to
# the controller, which created a replacement pod.
```

6. Observe the restart count:
```bash
kubectl get pods -l app=nginx-demo
# RESTARTS column shows how many times containers in that pod have been restarted
```

**Questions to answer:**
> 1. What is a reconciliation loop? What triggers an action in it?
>    (Answer: A loop that continuously compares desired state — stored in etcd — with current
>    state — observed from the cluster. Any deviation triggers corrective action.)
> 2. What is the difference between "desired state" and "current state"?
>    (Answer: Desired state is what you declared — 3 replicas. Current state is what actually
>    exists — 2 pods after one was deleted. The gap between them drives the controller's action.)
> 3. When you delete a pod that belongs to a Deployment, why does it come back?
>    (Answer: The Deployment controller's reconciliation loop sees that the current number of
>    pods is below the desired replica count and creates a new pod to compensate.)

---

## Exercise 1.3 — CrashLoopBackOff: Your First Debug

**What you'll learn:** How to diagnose a crashing pod without guessing. What CrashLoopBackOff
means mechanically, and how to use kubectl logs and describe to find the cause.

**Instructions:**

1. Create a pod designed to crash immediately (bad command):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
  - name: crasher
    image: alpine
    command: ["sh", "-c", "echo 'starting...'; exit 1"]
EOF
```

2. Watch the pod lifecycle:
```bash
kubectl get pods -w
# You will see: Pending → ContainerCreating → Error → CrashLoopBackOff
# Press Ctrl+C after you see CrashLoopBackOff
```

3. Read the current logs (what the container printed before dying):
```bash
kubectl logs crash-pod
# Output: starting...
```

4. Read the PREVIOUS container's logs (essential when the pod keeps restarting):
```bash
kubectl logs --previous crash-pod
# This reads logs from the LAST terminated container instance
```

5. Describe the pod — read the Events section carefully:
```bash
kubectl describe pod crash-pod
# Look at:
# - Last State: Terminated, reason: Error, exit code: 1
# - Restart Count: (incrementing)
# - Events: BackOff restarting failed container
```

6. Understand the back-off timer:
```bash
# The first restart happens immediately.
# Second restart: 10s delay
# Third: 20s, then 40s, 80s, 160s, maxes out at 300s (5 minutes)
# This prevents a crashing pod from hammering the container runtime.
kubectl get pods crash-pod -w
# Watch: READY column stays 0/1, RESTARTS increments, STATUS alternates
```

7. Now fix it WITHOUT deleting the pod (edit the running pod spec):
```bash
kubectl edit pod crash-pod
# Change: command: ["sh", "-c", "echo 'starting...'; exit 1"]
# To:     command: ["sh", "-c", "while true; do echo running; sleep 5; done"]
```

Save and exit the editor. Observe:
```bash
kubectl get pods crash-pod -w
```

> **Note:** For most pod spec fields, you cannot edit a running pod. Kubernetes will reject
> the change with "field is immutable." The command field is one of the few you can change.
> In practice, you fix crashes by updating the Deployment spec — which triggers a rolling
> update and replaces the pods. Editing pods directly is a diagnostic tool, not a workflow.

8. Clean up:
```bash
kubectl delete pod crash-pod
```

**Questions to answer:**
> 1. What is the back-off timer in CrashLoopBackOff? Why does Kubernetes not restart immediately?
>    (Answer: After each crash, Kubernetes waits 2x longer before the next restart, starting at
>    10s and maxing at 300s. This prevents a tight crash loop from wasting resources and allows
>    time for diagnosis.)
> 2. What is the difference between `kubectl logs` and `kubectl logs --previous`?
>    (Answer: `kubectl logs` reads logs from the currently running container. `--previous` reads
>    logs from the last terminated container instance — essential when the pod keeps restarting
>    too fast for you to capture current logs.)
> 3. What does exit code 1 tell you? What about exit code 137?
>    (Answer: Exit code 1 means the process exited with a generic error — usually an application
>    error. Exit code 137 means the process was killed by signal 9 (SIGKILL) — usually an OOM
>    kill or a forceful termination.)

> **Hint:** `kubectl describe pod` always has the answer. The Events section at the bottom
> shows the exact sequence: when the container started, when it died, what the exit code was,
> and when Kubernetes decided to back off. Read it top to bottom.

---

## Exercise 1.4 — Multi-Container Pods: Init Containers and Sidecars

**What you'll learn:** How init containers work, how sidecar containers share a pod's
resources, and when to use each pattern.

**Instructions:**

1. Create a pod with an init container that writes a config file to a shared volume:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
  - name: shared-config
    emptyDir: {}

  initContainers:
  - name: config-writer
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "APP_MODE=production" > /config/app.env
      echo "LOG_LEVEL=info" >> /config/app.env
      echo "Init container done. Config written."
    volumeMounts:
    - name: shared-config
      mountPath: /config

  containers:
  - name: main-app
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "Main container starting..."
      cat /config/app.env
      echo "Config loaded. Running..."
      while true; do sleep 10; done
    volumeMounts:
    - name: shared-config
      mountPath: /config

  - name: log-sidecar
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "Sidecar: waiting for config..."
      sleep 3
      while true; do
        echo "Sidecar heartbeat: $(date)"
        sleep 15
      done
EOF
```

2. Watch the pod initialization sequence:
```bash
kubectl get pods init-demo -w
# You will see: Init:0/1 → PodInitializing → Running
# Init containers run FIRST, sequentially, before any main containers start.
```

3. Observe that init containers must succeed before main containers start:
```bash
kubectl describe pod init-demo
# Look at Init Containers section: State = Terminated, Reason = Completed
# Main containers only appear AFTER init containers complete
```

4. Read logs from the init container:
```bash
kubectl logs init-demo -c config-writer
# Shows: "Init container done. Config written."
```

5. Read logs from the main container and verify it read the config:
```bash
kubectl logs init-demo -c main-app
# Shows: APP_MODE=production, LOG_LEVEL=info
```

6. Read sidecar logs:
```bash
kubectl logs init-demo -c log-sidecar
# Shows heartbeat messages — running alongside the main container
```

7. Now deliberately break the init container and observe the effect:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-broken
spec:
  volumes:
  - name: shared-config
    emptyDir: {}
  initContainers:
  - name: config-writer
    image: alpine
    command: ["sh", "-c", "echo 'Init failing'; exit 1"]
    volumeMounts:
    - name: shared-config
      mountPath: /config
  containers:
  - name: main-app
    image: alpine
    command: ["sh", "-c", "while true; do echo running; sleep 5; done"]
    volumeMounts:
    - name: shared-config
      mountPath: /config
EOF

kubectl get pods init-broken -w
# The main container NEVER starts. Init:CrashLoopBackOff
```

8. Clean up:
```bash
kubectl delete pod init-demo init-broken
```

**Questions to answer:**
> 1. When would you use an init container vs a sidecar?
>    (Answer: Init containers run and complete BEFORE the main container starts — use them for
>    setup tasks: writing config, waiting for dependencies, running migrations. Sidecars run
>    alongside the main container for the pod's entire lifetime — use them for log shipping,
>    proxies, metrics collection.)
> 2. What happens if an init container fails?
>    (Answer: The pod enters Init:CrashLoopBackOff. None of the main containers start. The
>    init container is restarted according to the pod's restart policy until it succeeds.)
> 3. What is an emptyDir volume and what happens to it when the pod is deleted?
>    (Answer: emptyDir is a temporary volume that is created when the pod starts and deleted
>    when the pod is removed. All containers in the pod can share it, but the data does not
>    persist across pod restarts. Restart within the same pod — the volume survives. Pod
>    deletion — the volume is gone.)

> **Hint:** When a pod has multiple containers, most kubectl commands require you to specify
> which container with `-c <container-name>`. Without it, kubectl defaults to the first
> container defined in the spec.

---

## Exercise 1.5 — Resource Requests vs Limits: The Scheduler Sees Requests, the Kernel Sees Limits

**What you'll learn:** The critical distinction between resource requests (scheduling) and
limits (enforcement). The three QoS classes and why they matter for eviction.

**Instructions:**

1. Inspect the current node capacity before scheduling anything:
```bash
kubectl describe node minikube | grep -A20 "Allocated resources"
# You will see: CPU requests vs allocatable, Memory requests vs allocatable
```

2. Create pods with different request/limit combinations:
```bash
# Guaranteed QoS: requests == limits
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "while true; do sleep 10; done"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF

# Burstable QoS: requests < limits (or only limits set)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "while true; do sleep 10; done"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

# BestEffort QoS: no requests, no limits (dangerous in production)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "while true; do sleep 10; done"]
EOF
```

3. Verify the QoS class assigned to each pod:
```bash
kubectl get pod qos-guaranteed -o jsonpath='{.status.qosClass}'
# Output: Guaranteed

kubectl get pod qos-burstable -o jsonpath='{.status.qosClass}'
# Output: Burstable

kubectl get pod qos-besteffort -o jsonpath='{.status.qosClass}'
# Output: BestEffort
```

4. Check how resource requests affect what the scheduler "sees" as consumed:
```bash
kubectl describe node minikube | grep -A30 "Allocated resources"
# The Requests column shows the SCHEDULING pressure.
# Limits are NOT shown here — they don't affect scheduling placement.
```

5. Now simulate OOMKilled by exceeding the memory limit:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: mem-hog
    image: alpine
    command:
    - sh
    - -c
    - |
      apk add --no-cache stress-ng 2>/dev/null
      echo "Starting memory stress..."
      stress-ng --vm 1 --vm-bytes 200M --timeout 30s
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "100Mi"
EOF

kubectl get pods oom-test -w
# Wait for OOMKilled status
```

6. Examine the OOMKilled state:
```bash
kubectl describe pod oom-test
# Look at: Last State: Terminated, Reason: OOMKilled
# Exit code: 137 (killed by SIGKILL from the kernel OOM mechanism)

kubectl get pod oom-test -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled
```

7. Clean up:
```bash
kubectl delete pod qos-guaranteed qos-burstable qos-besteffort oom-test
kubectl delete deployment nginx-demo
```

**Questions to answer:**
> 1. What happens if a pod's resource requests exceed what is allocatable on any node?
>    (Answer: The pod stays in Pending state permanently. The scheduler cannot find a valid
>    node. `kubectl describe pod` will show: "0/1 nodes are available: Insufficient memory"
>    or similar.)
> 2. What is the difference between BestEffort, Burstable, and Guaranteed QoS?
>    (Answer: Guaranteed (requests == limits) — last to be evicted under node pressure.
>    Burstable (requests < limits, or only one set) — evicted after BestEffort. BestEffort
>    (no requests or limits) — evicted first. This order is enforced by kubelet during
>    resource pressure events.)
> 3. Why does setting only limits (without requests) matter?
>    (Answer: When only limits are set, Kubernetes sets requests equal to limits. This means
>    the scheduler treats the pod as consuming its full limit for placement purposes — it may
>    prevent the pod from being scheduled even if it rarely uses that much memory.)

> **Hint:** `100m` CPU means 100 millicores = 0.1 of one CPU core. `64Mi` means 64 mebibytes.
> These units are not typos — they are standard Kubernetes resource notation.

---

## Day 1 Checkpoint

Before moving to Day 2, answer these without looking at the material:

1. What is the Kubernetes control loop? Name the three steps it repeats.
2. What does kubelet do? Where does it run?
3. What is the difference between a Pod and a container?
4. What does CrashLoopBackOff mean, and why does Kubernetes add a delay between restarts?
5. What is the difference between an init container and a sidecar container?
6. What does "resource requests" tell the scheduler? What do "limits" tell the kernel?
7. What are the three QoS classes, and what determines which class a pod is assigned?

---

## Day 1 Summary

| Concept | What You Proved |
|---|---|
| Pod = container(s) with metadata | Found actual container via crictl on the minikube node |
| Reconciliation loop | Deleted a pod, watched Deployment controller recreate it |
| CrashLoopBackOff | Created a crashing pod, read logs --previous, traced back-off timer |
| Init containers | Init failure blocks main container entirely |
| emptyDir shared volume | Init container wrote config, main container read it |
| Resource requests vs limits | Requests → scheduler placement, limits → kernel enforcement |
| QoS classes | Guaranteed pod is last evicted; BestEffort pod is first |
| OOMKilled | Triggered and confirmed via exit code 137 and describe output |
