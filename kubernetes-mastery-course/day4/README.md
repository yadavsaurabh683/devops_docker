# Day 4: Production Workloads
## Theme: Deployments, Probes, StatefulSets, Scaling, and Self-Healing

**Goal:** By end of Day 4, you will know how to configure Deployments for zero-downtime
rolling updates, design probes correctly to prevent traffic from reaching unready pods,
understand StatefulSet guarantees and when to use them over Deployments, and configure HPA
to scale workloads automatically based on CPU metrics.

**Estimated time:** 5–6 hours

---

## Background: Production Is Different From Development

In development, you run one pod and it works. In production:
- Pods die during node maintenance and must be replaced without downtime
- Bad deployments must be rolled back in under 60 seconds
- Slow-starting apps must not receive traffic before they're ready
- Database pods must have stable identity and storage across restarts
- Traffic spikes must trigger automatic scale-out without manual intervention

Every exercise today simulates one of these production realities.

---

## Exercise 4.1 — Rolling Updates with Zero Downtime

**What you'll learn:** How Deployment rolling updates work, why readiness probes are
mandatory for zero-downtime, and what happens when you deploy a bad image.

**Instructions:**

1. Create a namespace:
```bash
kubectl create namespace day4
kubectl config set-context --current --namespace=day4
```

2. Deploy a v1 nginx app with a readiness probe:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: day4
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra pod above desired count during update
      maxUnavailable: 0  # Never reduce below desired count (zero downtime)
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
EOF

kubectl rollout status deployment/webapp
# Wait until: deployment "webapp" successfully rolled out
```

3. Watch the rolling update in action — update to v1.25:
```bash
# In a second terminal, watch pods:
kubectl get pods -w -n day4

# In the first terminal, trigger the update:
kubectl set image deployment/webapp nginx=nginx:1.25-alpine

# Observe in the watch terminal:
# - One new pod starts (maxSurge=1 means 4 pods temporarily)
# - New pod becomes Ready (readiness probe passes)
# - One old pod terminates
# - Repeat until all 3 are on new image
```

4. Check rollout status:
```bash
kubectl rollout status deployment/webapp
kubectl get pods -o wide
kubectl describe pods | grep -E "Image:|Ready:"
```

5. Now deploy WITHOUT a readiness probe and observe the difference:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-noprobe
  namespace: day4
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: webapp-noprobe
  template:
    metadata:
      labels:
        app: webapp-noprobe
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
        # NO readiness probe
EOF

# Now update to a slow-starting image (simulate with a sleep before nginx starts):
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-noprobe
  namespace: day4
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: webapp-noprobe
  template:
    metadata:
      labels:
        app: webapp-noprobe
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        command: ["sh", "-c", "sleep 20 && nginx -g 'daemon off;'"]
        ports:
        - containerPort: 80
        # NO readiness probe: Kubernetes marks it "Running" after container starts,
        # even though nginx hasn't started yet (it's still sleeping)
EOF

kubectl get pods -n day4 -l app=webapp-noprobe -w
# Watch: pods transition to Running while actually still sleeping
# Without a readiness probe, traffic goes to pods that aren't serving yet
```

6. Now deploy a BAD image to observe rollout failure:
```bash
kubectl set image deployment/webapp nginx=nginx:NONEXISTENT-TAG

kubectl rollout status deployment/webapp
# This will hang — the new pods fail to start (ImagePullBackOff)

kubectl get pods -n day4 -l app=webapp
# You see: old pods still running + new pod(s) in ImagePullBackOff
# Because maxUnavailable=0, the old pods stay up — zero downtime is maintained
```

7. Roll back from the bad deployment:
```bash
kubectl rollout undo deployment/webapp
kubectl rollout status deployment/webapp
# Back to the previous working image
```

**Questions to answer:**
> 1. What does maxSurge control? What does maxUnavailable control?
>    (Answer: maxSurge is the maximum number of pods above the desired count during an update.
>    maxUnavailable is the maximum number of pods below the desired count. Setting
>    maxUnavailable=0 means no traffic degradation is allowed — new pods must be Ready before
>    old ones are terminated.)
> 2. What triggers a rollout to pause or fail?
>    (Answer: If the new pods cannot start — bad image, ImagePullBackOff, or failing readiness
>    probe — the rollout hangs. It does not automatically fail; it waits. Set
>    spec.progressDeadlineSeconds to define a timeout after which the rollout is marked Failed.)
> 3. Why is a readiness probe critical for zero-downtime rolling updates?
>    (Answer: Without it, Kubernetes assumes a pod is ready the moment the container starts.
>    Traffic flows to the new pod before the app is actually serving. With a readiness probe,
>    the pod is only added to Service endpoints after the probe passes — ensuring real readiness.)

---

## Exercise 4.2 — Rollback: When You Need to Go Back Fast

**What you'll learn:** How Kubernetes tracks deployment revision history, and how to roll
back to any previous version in under 60 seconds.

**Instructions:**

1. Deploy version 1 and record the first revision:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: versioned-app
  namespace: day4
  annotations:
    kubernetes.io/change-cause: "Initial deployment: nginx 1.24"
spec:
  replicas: 2
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: versioned-app
  template:
    metadata:
      labels:
        app: versioned-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.24-alpine
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
EOF

kubectl rollout status deployment/versioned-app
```

2. Deploy version 2:
```bash
kubectl set image deployment/versioned-app nginx=nginx:1.25-alpine
kubectl annotate deployment/versioned-app kubernetes.io/change-cause="Update to nginx 1.25" --overwrite
kubectl rollout status deployment/versioned-app
```

3. Deploy a bad version 3:
```bash
kubectl set image deployment/versioned-app nginx=nginx:99.99-broken
kubectl annotate deployment/versioned-app kubernetes.io/change-cause="Broken: nginx 99.99" --overwrite
# Don't wait for this one — it will fail
sleep 15
kubectl get pods -n day4 -l app=versioned-app
# Mix of old running pods and new ImagePullBackOff pods
```

4. Check the revision history:
```bash
kubectl rollout history deployment/versioned-app
# Shows:
# REVISION  CHANGE-CAUSE
# 1         Initial deployment: nginx 1.24
# 2         Update to nginx 1.25
# 3         Broken: nginx 99.99
```

5. Roll back to the last working version (revision 2):
```bash
time kubectl rollout undo deployment/versioned-app
# Should complete in well under 60 seconds

kubectl rollout status deployment/versioned-app
kubectl get pods -n day4 -l app=versioned-app
# All pods now back on nginx:1.25-alpine
```

6. Roll back to a SPECIFIC revision (revision 1):
```bash
kubectl rollout undo deployment/versioned-app --to-revision=1
kubectl rollout status deployment/versioned-app

# Verify:
kubectl get pods -n day4 -l app=versioned-app -o jsonpath='{.items[0].spec.containers[0].image}'
# Output: nginx:1.24-alpine
```

7. Check how many revisions are kept:
```bash
kubectl get replicasets -n day4 -l app=versioned-app
# Each revision is a separate ReplicaSet
# revisionHistoryLimit=5 means at most 5 inactive ReplicaSets are kept
```

**Questions to answer:**
> 1. How many revisions does Kubernetes keep by default?
>    (Answer: 10, set by spec.revisionHistoryLimit. Each revision is stored as a ReplicaSet
>    with 0 desired replicas. Setting revisionHistoryLimit: 0 removes all history — you lose
>    the ability to roll back.)
> 2. What is stored in a revision?
>    (Answer: The complete pod template spec — image, environment, volume mounts, resources,
>    probes. When you roll back, Kubernetes re-applies the old pod template, which creates a
>    new rollout from the stored spec.)
> 3. If you run `kubectl rollout undo` twice in a row, what happens?
>    (Answer: The first undo moves to the previous revision. The second undo moves back to
>    the revision before that — not forward. If you want to "redo" a rollback, use
>    `--to-revision=<n>` to target a specific revision explicitly.)

> **Hint:** Always set `kubernetes.io/change-cause` annotations on your deployments. It makes
> rollout history readable. Without it, the CHANGE-CAUSE column shows `<none>` and you have
> to inspect each ReplicaSet to understand what changed.

---

## Exercise 4.3 — Liveness vs Readiness vs Startup Probes

**What you'll learn:** The precise behavioral difference between all three probe types. A
wrong probe configuration causes silent failures in production — pods that restart when they
shouldn't, or pods that stay in Service endpoints when they can't serve traffic.

**Instructions:**

1. Deploy a pod with all three probes configured:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  namespace: day4
spec:
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80

    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30    # Allow up to 30*3s = 90s for startup
      periodSeconds: 3
      # During startup, liveness is DISABLED until startupProbe succeeds
      # This prevents slow-starting apps from being killed by liveness

    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
      # Failure: pod is REMOVED from Service endpoints (but stays running)
      # Recovery: pod is ADDED BACK when probe starts passing again

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      failureThreshold: 3
      # Failure: pod is RESTARTED (container killed and recreated)
      # Use for detecting deadlocks or hung processes
EOF

kubectl get pods probe-demo -w
# Watch: ContainerCreating → Running 0/1 (startup probe running) → Running 1/1 (all probes passing)
```

2. Create a Service and verify the pod is an endpoint only when ready:
```bash
kubectl expose pod probe-demo --port=80 --name=probe-svc -n day4
kubectl get endpoints probe-svc
# Shows the pod's IP when Ready=True
```

3. Simulate a readiness failure (exec and break the nginx index):
```bash
# Kill nginx temporarily from inside the pod:
kubectl exec probe-demo -- sh -c 'nginx -s stop; sleep 60 &'
# nginx is stopped — readiness probe HTTP check will fail

# Watch the pod status:
kubectl get pods probe-demo -w
# READY: 1/1 → 0/1 (readiness probe failed)
# STATUS: stays Running — pod is NOT killed

# Check endpoints:
kubectl get endpoints probe-svc
# Endpoint is REMOVED when readiness fails — no traffic goes there
```

4. After 60 seconds, nginx will NOT restart (because we didn't configure it to).
   Observe that liveness fires after 3 consecutive failures (3 x 10s = 30s post-startup):
```bash
# Wait and watch:
kubectl get pods probe-demo -w
# Eventually: READY=0/1, RESTARTS increments
# The liveness probe failed (nginx not responding) and the container was restarted
# After restart, nginx comes back, probes pass, READY=1/1
```

5. Observe the probe results in describe:
```bash
kubectl describe pod probe-demo
# Look for:
# Liveness:    http-get  ... delay=15s timeout=1s period=10s #success=1 #failure=3
# Readiness:   http-get  ... delay=5s timeout=1s period=5s  #success=1 #failure=3
# Startup:     http-get  ... delay=0s timeout=1s period=3s  #success=1 #failure=30
# Conditions: Ready=True/False
```

6. Clean up:
```bash
kubectl delete pod probe-demo
kubectl delete service probe-svc
```

**Questions to answer:**
> 1. What is the critical difference between liveness and readiness failures?
>    (Answer: Readiness failure → pod is removed from Service endpoints but NOT restarted.
>    Traffic stops going there; pod keeps running and may recover. Liveness failure →
>    container is killed and restarted. Use readiness for "not ready to serve" situations
>    and liveness for "process is hung and needs a restart.")
> 2. What is the startup probe for?
>    (Answer: Slow-starting applications. Without it, if a Java app takes 60s to start,
>    the liveness probe would kill it before it ever finishes starting. The startup probe
>    disables liveness checking until the startup probe first succeeds — effectively giving
>    the app a one-time extended startup window.)
> 3. What happens if you set initialDelaySeconds too low on a liveness probe?
>    (Answer: The liveness probe fires before the app has finished starting up, fails, and
>    the container is killed. The pod enters CrashLoopBackOff even though the app itself is
>    correct. This is a very common production misconfiguration.)

---

## Exercise 4.4 — StatefulSets: Ordered, Stable, Persistent

**What you'll learn:** The guarantees that StatefulSets provide — stable network identity,
ordered startup/shutdown, per-pod PVCs — and why these matter for databases.

**Instructions:**

1. Create a headless service (required for StatefulSet DNS):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: stateful-svc
  namespace: day4
spec:
  clusterIP: None    # Headless: no ClusterIP, DNS resolves to individual pod IPs
  selector:
    app: stateful-app
  ports:
  - port: 80
    name: web
EOF
```

2. Create a StatefulSet with 3 replicas and per-pod PVCs:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
  namespace: day4
spec:
  serviceName: stateful-svc    # Must match the headless service name
  replicas: 3
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: pod-storage
          mountPath: /data
        command:
        - sh
        - -c
        - |
          echo "Pod $(hostname) started at $(date)" > /data/identity.txt
          nginx -g 'daemon off;'
  volumeClaimTemplates:
  - metadata:
      name: pod-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Mi
EOF
```

3. Observe the ordered startup (pods start one at a time in order):
```bash
kubectl get pods -n day4 -l app=stateful-app -w
# stateful-app-0 starts first, must be Running before stateful-app-1 starts
# stateful-app-1 starts next, must be Running before stateful-app-2 starts
# This guarantees: pod-0 is always the "primary" in databases that elect a leader
```

4. Verify predictable names and individual PVCs:
```bash
kubectl get pods -n day4 -l app=stateful-app
# Names: stateful-app-0, stateful-app-1, stateful-app-2  (always these, never random)

kubectl get pvc -n day4 | grep stateful
# Each pod has its own PVC: pod-storage-stateful-app-0, pod-storage-stateful-app-1, etc.
```

5. Verify each pod wrote its own identity to ITS OWN storage:
```bash
kubectl exec -n day4 stateful-app-0 -- cat /data/identity.txt
# Pod stateful-app-0 started at...

kubectl exec -n day4 stateful-app-1 -- cat /data/identity.txt
# Pod stateful-app-1 started at...

# They have separate storage — one pod cannot see the other's /data
```

6. Delete pod-1 and watch it return with the SAME identity and SAME PVC:
```bash
kubectl delete pod stateful-app-1 -n day4
kubectl get pods -n day4 -l app=stateful-app -w
# stateful-app-1 is recreated — with the SAME name

# Verify the data persists:
kubectl exec -n day4 stateful-app-1 -- cat /data/identity.txt
# Same content as before the delete — the PVC was preserved and reattached
```

7. Verify stable DNS — each pod has its own DNS entry:
```bash
kubectl run dns-test --image=busybox --restart=Never -it --rm -n day4 -- sh -c '
  nslookup stateful-app-0.stateful-svc.day4.svc.cluster.local
  nslookup stateful-app-1.stateful-svc.day4.svc.cluster.local
  nslookup stateful-app-2.stateful-svc.day4.svc.cluster.local
'
# Each pod resolves to its own IP via: <pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

**Questions to answer:**
> 1. What is the headless service used with StatefulSets?
>    (Answer: A Service with clusterIP: None. Instead of routing to a virtual ClusterIP,
>    DNS for the service resolves to the individual pod IPs. This enables stable DNS names
>    for each pod: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.)
> 2. What DNS name does pod-1 get in a StatefulSet named 'db' with headless service 'db-svc'
>    in namespace 'prod'?
>    (Answer: `db-1.db-svc.prod.svc.cluster.local`)
> 3. What is the key difference between deleting pod-1 in a Deployment vs a StatefulSet?
>    (Answer: In a Deployment, the replacement pod gets a new random name and may get a
>    different PVC if it was not specifically attached. In a StatefulSet, the replacement
>    always gets the same name (stateful-app-1), the same PVC, and the same DNS name —
>    stable identity is guaranteed.)

---

## Exercise 4.5 — Horizontal Pod Autoscaler (HPA) Under Load

**What you'll learn:** How HPA uses metrics-server to make scaling decisions, how to
configure target CPU utilization, and how to observe scale-up and scale-down behavior.

**Instructions:**

1. Verify metrics-server is running (enabled in Day 1 setup):
```bash
kubectl top nodes
kubectl top pods -n day4
# If these work, metrics-server is running
# If not: minikube addons enable metrics-server
```

2. Deploy a CPU-intensive app with resource requests:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-app
  namespace: day4
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-app
  template:
    metadata:
      labels:
        app: cpu-app
    spec:
      containers:
      - name: stress
        image: alpine
        command: ["sh", "-c", "while true; do sleep 0.01; done"]
        resources:
          requests:
            cpu: "100m"
          limits:
            cpu: "300m"
EOF

kubectl expose deployment cpu-app --port=80 --target-port=80 -n day4
kubectl rollout status deployment/cpu-app -n day4
```

3. Create an HPA targeting 50% CPU utilization:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-app-hpa
  namespace: day4
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

kubectl get hpa -n day4 -w
# Initially shows: TARGETS=<unknown>/50%  — metrics-server needs ~60s to gather data
# After a minute: TARGETS=X%/50%
```

4. Generate CPU load to trigger scale-up:
```bash
# Open a second terminal and run the load generator:
kubectl run load-gen --image=busybox --restart=Never -n day4 \
  -- sh -c 'while true; do wget -q -O- http://cpu-app.day4.svc.cluster.local; done'
```

5. Watch the HPA scale up:
```bash
# In the original terminal:
kubectl get hpa cpu-app-hpa -n day4 -w
# TARGETS will climb above 50%
# REPLICAS will increase: 1 → 2 → 3 → ... up to 5

kubectl get pods -n day4 -l app=cpu-app -w
# New pods appear
```

6. Stop the load and watch scale-down:
```bash
kubectl delete pod load-gen -n day4

# HPA has a scale-down cool-down period (default: 5 minutes)
# After 5 minutes without load, replicas decrease back to minReplicas=1
kubectl get hpa cpu-app-hpa -n day4 -w
```

7. Inspect the HPA events to understand its decisions:
```bash
kubectl describe hpa cpu-app-hpa -n day4
# Look at Events:
# SuccessfulRescale: "New size: 3; reason: cpu resource utilization (percentage of request) above target"
# SuccessfulRescale: "New size: 1; reason: All metrics below target"
```

8. Clean up:
```bash
kubectl delete namespace day4
kubectl config set-context --current --namespace=default
```

**Questions to answer:**
> 1. What does HPA use to make scaling decisions?
>    (Answer: HPA queries metrics-server for the current resource utilization of all pods
>    in the target deployment. It computes: current utilization / target utilization × current
>    replicas = desired replicas. For example, if current CPU is 80% and target is 50%,
>    it scales to ceil(80/50 × current_replicas) pods.)
> 2. What is the scale-down cool-down period and why does it exist?
>    (Answer: By default, HPA waits 5 minutes before scaling down to prevent flapping —
>    a scenario where a burst of traffic causes scale-up, which briefly reduces per-pod CPU,
>    triggering immediate scale-down, which increases per-pod CPU again. The cool-down period
>    ensures load truly subsided before removing capacity.)
> 3. Why must resource requests be set for HPA to work?
>    (Answer: HPA expresses targets as a percentage of the request. Without requests, there
>    is no denominator — you cannot compute "50% of request." Pods without resource requests
>    will show `<unknown>` in the TARGETS column and HPA will not scale them.)

---

## Day 4 Checkpoint

Before moving to Day 5, answer these without looking at the material:

1. What does maxSurge=1 and maxUnavailable=0 mean for a rolling update?
2. What is stored in a Deployment revision?
3. What is the difference between a readiness probe failure and a liveness probe failure?
4. What is the startup probe for?
5. What are the three StatefulSet guarantees (identity, ordering, storage)?
6. What is a headless service and what DNS name format does it enable for StatefulSet pods?
7. What metric does HPA use by default, and what must be set on pods for it to work?

---

## Day 4 Summary

| Concept | What You Proved |
|---|---|
| maxUnavailable=0 | Old pods kept running until new pods were Ready |
| Bad image during rollout | Old pods survived; new pods stuck in ImagePullBackOff |
| Readiness probe failure | Pod removed from endpoints but not restarted |
| Liveness probe failure | Container killed and restarted after 3 failures |
| Startup probe | Disables liveness during slow startup |
| StatefulSet ordering | pod-0 started before pod-1 started before pod-2 |
| StatefulSet identity | Deleted pod-1 came back with same name and same PVC |
| StatefulSet DNS | Each pod resolves individually via headless service |
| HPA scale-up | Replicas increased when CPU exceeded 50% of request |
| HPA cool-down | Scale-down waited 5 minutes after load dropped |
