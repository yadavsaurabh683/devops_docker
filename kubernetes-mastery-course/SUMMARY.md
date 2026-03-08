# End-of-Course Summary
## Kubernetes Mastery — What You Learned Across 5 Days

This document consolidates every major concept from the course. Use it after completing Day 5
to verify your understanding is complete and connected — not just a list of commands you ran.

---

## Day 1 — Architecture, Control Loop, and Pod Lifecycle

### The Control Loop: The Single Most Important Concept

Every Kubernetes component is a reconciliation loop:
1. **Read** desired state from etcd (via API server)
2. **Observe** current state of the cluster
3. **Act** on the difference: create, update, or delete resources
4. **Repeat** forever

When you understand this, all Kubernetes behavior becomes predictable. A pod that
disappeared and came back is not magic — the Deployment controller saw `current < desired`
and acted. A pod that won't start is the scheduler seeing `current_placement != valid_node`
and waiting.

### What Pods Actually Are

A pod is a Kubernetes scheduling unit, not a container. It contains one or more containers
that share:
- The same network namespace (same IP, same port space)
- The same storage volumes (emptyDir, ConfigMap, PVC mounts)

The actual container is managed by the container runtime (containerd) via the kubelet.
The kubelet finds the container on the node using CRI (Container Runtime Interface).

### Pod Lifecycle States

| State | Meaning |
|-------|---------|
| `Pending` | Accepted by API, but not scheduled to a node yet (or downloading image) |
| `ContainerCreating` | Scheduled to a node, kubelet is starting the containers |
| `Running` | At least one container is running. Does NOT mean the app is healthy |
| `CrashLoopBackOff` | Container has crashed and Kubernetes is backing off on restarts |
| `OOMKilled` | Container exceeded memory limit — kernel killed it |
| `Completed` | All containers exited with code 0 (typical for Jobs) |
| `Evicted` | Pod removed from node due to resource pressure (disk/memory) |
| `Terminating` | Pod is being deleted — containers receiving SIGTERM |

### CrashLoopBackOff

The container crashed. The back-off timer grows exponentially: 10s, 20s, 40s, 80s, 160s,
capping at 300s (5 minutes). This prevents a tight crash loop from hammering the runtime.

Diagnostic sequence:
1. `kubectl describe pod <name>` → Last State, Exit Code, Events
2. `kubectl logs --previous <name>` → last crashed container's stdout
3. Fix the Deployment spec, not the pod directly

### Init Containers

Run to completion before any main container starts. Used for:
- Setup tasks (write config, wait for dependency, run schema migration)
- Ordering guarantees (database must be reachable before the API starts)

If an init container fails, the main containers never start. The init container enters
CrashLoopBackOff independently.

### Resource Requests vs Limits

| | Requests | Limits |
|--|----------|--------|
| **Who sees it** | Scheduler | Linux kernel (cgroup) |
| **What happens if exceeded** | Pod is Pending (no room) | OOMKill (memory) or throttled (CPU) |
| **Effect on QoS** | Determines QoS class | Determines QoS class |

### QoS Classes

| Class | Condition | Eviction order |
|-------|-----------|----------------|
| `Guaranteed` | All containers have requests == limits | Last evicted |
| `Burstable` | At least one container has requests or limits | Evicted second |
| `BestEffort` | No requests and no limits on any container | Evicted first |

Production workloads must be `Guaranteed` to avoid unexpected eviction under node pressure.

---

## Day 2 — Services, CoreDNS, kube-proxy, and Network Policies

### The Four Service Types

| Type | Accessible from | Use case |
|------|-----------------|----------|
| `ClusterIP` | Inside cluster only | Internal service-to-service |
| `NodePort` | Node IP + port (30000-32767) | Dev, simple external access |
| `LoadBalancer` | Cloud LB external IP | Production external access |
| `Headless` (ClusterIP: None) | DNS → individual pod IPs | StatefulSets, service discovery |

### How kube-proxy Makes ClusterIP Work

1. You create a Service with a ClusterIP (virtual IP, no real interface)
2. kube-proxy watches for Service + Endpoints changes via the API server
3. kube-proxy writes iptables (KUBE-SERVICES → KUBE-SVC-xxx → KUBE-SEP-xxx) on every node
4. When a pod sends traffic to the ClusterIP, iptables DNAT rewrites the destination
   to a random pod IP from the Endpoints list
5. The packet goes to the actual pod — the ClusterIP never touched any real interface

### The Full DNS Resolution Chain

```
Pod sends: curl http://nginx-svc

1. Pod looks at /etc/resolv.conf:
   nameserver 10.96.0.10          ← CoreDNS ClusterIP
   search day2.svc.cluster.local svc.cluster.local cluster.local

2. Resolver tries: nginx-svc.day2.svc.cluster.local → DNS query to 10.96.0.10

3. CoreDNS looks up: nginx-svc in namespace day2 → returns ClusterIP (e.g. 10.96.100.5)

4. Pod sends packet to 10.96.100.5

5. iptables DNAT: 10.96.100.5:80 → 10.244.0.7:80 (actual pod IP)

6. Packet reaches nginx pod
```

Full FQDN format: `<service>.<namespace>.svc.cluster.local`

### Service Debugging Checklist

When a Service doesn't route traffic:
1. `kubectl get endpoints <svc>` — is it `<none>`?
2. `kubectl describe service <svc>` — what is the Selector?
3. `kubectl get pods --show-labels` — do pod labels match the Selector exactly?
4. `kubectl run debug --image=busybox --rm -it -- wget -O- http://<svc>`
5. Check if the target port number matches the container port

### NetworkPolicy Semantics

- **No policy on a pod** = accept all traffic from all sources
- **Any policy targeting a pod** = deny all EXCEPT what the policy explicitly allows
- Policies are additive — multiple policies OR together (more policies = more allowed traffic)
- Enforcement is by the CNI plugin, not Kubernetes core
- Kubernetes stores the policy in etcd; the CNI programs iptables/eBPF on each node
- If your CNI does not support NetworkPolicy (e.g., kindnet), policies are stored but not enforced

---

## Day 3 — Configuration, Storage, and Access Control

### ConfigMap: Env Var vs Volume Mount

| Method | Updates propagate? | Requires restart? |
|--------|-------------------|-------------------|
| `env.valueFrom.configMapKeyRef` | No — frozen at pod start | Yes — pod must restart |
| `volume mount` | Yes — kubelet syncs ~60s | No — app re-reads file |

Design apps to read config from files at startup AND on reload (SIGHUP) for live config updates.

### Secrets Are Not Encrypted

- Secret data is base64 encoded, not encrypted
- Anyone with `kubectl get secret -o yaml` access can decode immediately
- Real security layers:
  1. **RBAC**: restrict who can `get` or `list` secrets
  2. **etcd encryption at rest**: kube-apiserver `--encryption-provider-config`
  3. **External secret stores**: Vault, AWS Secrets Manager via External Secrets Operator
  4. **Sealed Secrets**: encrypted for git storage, decrypted only inside the cluster

Never put secrets in ConfigMaps. Never commit even base64-encoded secrets to git.

### PV / PVC / StorageClass Relationship

```
StorageClass (defines: provisioner, parameters, reclaimPolicy)
     │
     │ Dynamic provisioning (automatic)
     ▼
PersistentVolume (actual storage: hostPath, NFS, cloud disk)
     │
     │ 1:1 binding
     ▼
PersistentVolumeClaim (namespace-scoped request for storage)
     │
     │ volumeMounts
     ▼
Pod (mounts the PVC at a path inside the container)
```

Reclaim policies:
- `Delete`: PV is deleted when PVC is deleted (default in cloud StorageClasses)
- `Retain`: PV stays in `Released` state — admin must manually reclaim

PVC lifecycle is independent of pod lifecycle. Pod death does not affect PVC.

### RBAC Full Chain

```
Pod
 └── ServiceAccount (identity of the pod)
      └── RoleBinding or ClusterRoleBinding (binds identity to a role)
           └── Role or ClusterRole (set of {apiGroup, resource, verb} rules)
```

Key distinctions:
- `Role` + `RoleBinding`: namespaced (applies only within one namespace)
- `ClusterRole` + `ClusterRoleBinding`: cluster-wide (applies everywhere, or for non-namespaced resources)
- `ClusterRole` + `RoleBinding`: grants ClusterRole permissions scoped to one namespace

Testing permissions: `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa>`

### Namespaces

What namespaces DO provide:
- Resource naming scope (two services can have the same name in different namespaces)
- RBAC scope boundary (Roles apply within a namespace)
- ResourceQuota scope (quotas are per-namespace)
- LimitRange scope (default limits are per-namespace)

What namespaces do NOT provide:
- Network isolation (pods in any namespace can reach pods in any other namespace by default)
- Security boundary (not sufficient alone — needs RBAC + NetworkPolicy + admission control)

---

## Day 4 — Production Workloads

### Deployment Rolling Update Mechanics

```
spec.strategy.rollingUpdate:
  maxSurge: 1         # Up to desired+1 pods temporarily
  maxUnavailable: 0   # Zero traffic reduction during update

Update steps (desired=3, maxSurge=1, maxUnavailable=0):
1. Create new pod → total=4 (3 old + 1 new)
2. Wait for new pod readiness probe to pass
3. Terminate 1 old pod → total=3 (2 old + 1 new)
4. Repeat until all 3 are new version
```

Without a readiness probe, Kubernetes assumes ready at container start. Traffic goes to
pods that haven't finished initializing. This causes dropped requests during rolling updates.

### Probe Types

| Probe | Failure action | Recovery action | Use for |
|-------|---------------|-----------------|---------|
| `readiness` | Removed from Service endpoints | Re-added to endpoints | "Not ready to serve" |
| `liveness` | Container killed and restarted | Normal pod lifecycle | "Process is hung/deadlocked" |
| `startup` | Container killed (if failureThreshold exceeded) | Enables liveness/readiness | Slow-starting apps |

**Never configure liveness with a short initialDelaySeconds.** If liveness fires before the
app finishes starting, it kills the container in a restart loop that looks like CrashLoopBackOff.

### StatefulSet Guarantees

1. **Stable network identity**: pod-0, pod-1, pod-2 — always these names, never random
2. **Ordered deployment and scaling**: pod-0 must be Running before pod-1 starts
3. **Ordered deletion**: pod-2 deleted before pod-1, pod-1 before pod-0
4. **Stable storage**: each pod gets its own PVC that is not reassigned on pod death
5. **Stable DNS**: `<pod-name>.<headless-service>.<namespace>.svc.cluster.local`

Headless service (`clusterIP: None`) enables individual pod DNS resolution.
Without it, DNS only resolves to the ClusterIP — no way to address individual pods.

### HPA Mechanics

```
desired_replicas = ceil(current_replicas × (current_metric / target_metric))

Example: current=2, current_cpu=80%, target=50%
desired = ceil(2 × (80/50)) = ceil(3.2) = 4
```

Requirements:
- metrics-server must be running
- All pods must have resource requests set (denominator for % calculation)

Scale-up: immediate (next evaluation cycle ~15s after threshold exceeded)
Scale-down: 5-minute cool-down (configurable via `--horizontal-pod-autoscaler-downscale-stabilization`)

---

## Day 5 — Debugging and Production Operations

### The Complete kubectl Debug Toolkit

| Command | When to use |
|---------|-------------|
| `kubectl get events --sort-by=.lastTimestamp` | First command in any incident |
| `kubectl describe pod <name>` | Scheduling, probe, events, last state |
| `kubectl logs --previous <name>` | Logs from the last crashed container |
| `kubectl top pods` | Current CPU/memory (requires metrics-server) |
| `kubectl debug -it <pod> --image=busybox` | Debug without modifying the pod |
| `kubectl debug --copy-to=<name>` | Debug a CrashLoopBackOff pod |
| `kubectl port-forward svc/<name>` | Test an internal service from localhost |
| `kubectl exec -it <pod> -- curl <svc>` | Test service-to-service connectivity |
| `kubectl cp <pod>:/path ./local` | Extract files from a container |
| `kubectl get endpoints <svc>` | Verify service routing is configured |
| `kubectl auth can-i` | Verify RBAC permissions |
| `kubectl cordon / uncordon` | Manage node scheduling status |
| `kubectl drain` | Evict pods from a node for maintenance |

### Ephemeral Containers

Added to a running pod without restart via `kubectl debug`:
- Share the pod's network, PID, and storage namespaces
- Cannot be removed once added
- Not restarted if they exit
- Require Kubernetes 1.23+

Use case: debug a distroless or scratch container that has no shell by injecting a
debug image alongside it.

### Eviction Order Under Node Pressure

When a node runs low on memory:
1. BestEffort pods evicted first (no resource reservations — easiest to sacrifice)
2. Burstable pods using more than their requests evicted next
3. Burstable pods within their requests evicted after that
4. Guaranteed pods evicted last (only under critical node pressure)

Eviction is triggered by kubelet when memory falls below `evictionHard.memory.available`
(default: 100Mi). Pods with lower QoS class are evicted first to reclaim memory.

### Incident Response Framework

When paged about a Kubernetes production incident:

```
Step 1: kubectl get events --sort-by=.lastTimestamp -n <namespace>
        → What happened? When did it start?

Step 2: kubectl get pods -n <namespace> -o wide
        → Which pods are failing? Which node are they on?

Step 3: kubectl describe pod <failing-pod> -n <namespace>
        → Scheduling, probe results, last state, events

Step 4: kubectl logs --previous <pod> -n <namespace>
        → What did the process print before crashing?

Step 5: kubectl get endpoints -n <namespace>
        → Is traffic actually routed to working pods?

Step 6: kubectl top pods -n <namespace>
        → Is any pod resource-constrained right now?

Fix from the data layer up:
  Database (foundation) → API (business logic) → Frontend (presentation)
```

---

## Full Kubernetes Mental Model Map

```
KUBERNETES CLUSTER
│
├── CONTROL PLANE (runs on master node)
│   ├── API Server         ← Single entry point for all kubectl commands
│   │                         Reads/writes to etcd. Authenticates via RBAC.
│   ├── etcd               ← Distributed key-value store. Single source of truth.
│   │                         All desired state lives here.
│   ├── Scheduler          ← Watches for unscheduled pods. Selects a node based on
│   │                         requests, taints, tolerations, affinity.
│   └── Controller Manager ← Runs all reconciliation loops:
│                             Deployment controller, ReplicaSet controller,
│                             Endpoint controller, Job controller, etc.
│
├── WORKER NODES (where workloads run)
│   ├── kubelet            ← Agent on each node. Watches API for pods assigned to its
│   │                         node. Uses CRI to manage containers. Reports pod status.
│   ├── kube-proxy         ← Programs iptables for Service routing. Watches API for
│   │                         Service and Endpoints changes.
│   └── CNI plugin         ← Sets up pod networking. Enforces NetworkPolicy.
│                             (kindnet, calico, cilium, weave, flannel)
│
├── PODS
│   ├── Containers          ← Managed by container runtime (containerd, cri-o)
│   ├── Shared network      ← All containers in a pod share one IP address
│   ├── Shared volumes      ← emptyDir, ConfigMap, PVC mounts shared between containers
│   └── Init containers     ← Run and complete before main containers start
│
├── WORKLOAD CONTROLLERS
│   ├── Deployment          ← Stateless apps. Rolling updates. Rollback. No identity.
│   ├── StatefulSet         ← Stateful apps. Stable name/DNS/PVC. Ordered ops.
│   ├── DaemonSet           ← One pod per node. Log collectors, monitoring agents.
│   └── Job / CronJob       ← One-time or scheduled batch tasks.
│
├── NETWORKING
│   ├── ClusterIP Service   ← Virtual IP. kube-proxy iptables. Internal only.
│   ├── NodePort Service    ← NodeIP:port. External access (dev only).
│   ├── LoadBalancer Svc    ← Cloud LB. Production external access.
│   ├── Headless Service    ← DNS → pod IPs. StatefulSets. Service mesh.
│   ├── CoreDNS             ← Resolves <service>.<ns>.svc.cluster.local
│   └── NetworkPolicy       ← Firewall rules at pod level. CNI enforced.
│
├── CONFIGURATION
│   ├── ConfigMap           ← Non-sensitive config. Env or volume mount.
│   └── Secret              ← Sensitive data. Base64. RBAC-controlled.
│
├── STORAGE
│   ├── PersistentVolume    ← Actual storage resource (hostPath, NFS, cloud disk)
│   ├── PersistentVolumeClaim ← Namespace-scoped storage request. 1:1 binds to PV.
│   └── StorageClass        ← Dynamic provisioner policy. Reclaim policy.
│
└── ACCESS CONTROL
    ├── ServiceAccount      ← Identity of a pod when calling the Kubernetes API
    ├── Role / ClusterRole  ← Set of permitted {apiGroup, resource, verb} operations
    └── RoleBinding / ClusterRoleBinding ← Binds a ServiceAccount to a Role
```

---

## What You Can Now Do That Most DevOps Engineers Cannot

1. Explain what Kubernetes IS — a distributed reconciliation system — without saying
   "it manages containers"

2. Debug any failing workload using only kubectl: CrashLoopBackOff, OOMKilled, Pending,
   Evicted, CreateContainerConfigError — without guessing, without restarting

3. Trace a Service routing failure from the Service selector to the iptables DNAT rules
   and back to CoreDNS — finding the exact break point

4. Design production-grade Deployments with correct rolling update strategy, readiness
   probes, liveness probes, and rollback capability

5. Architect RBAC from scratch: identify what API calls each workload needs, create
   minimum-privilege Roles, bind them to dedicated ServiceAccounts

6. Choose between Deployment and StatefulSet based on actual guarantees (identity,
   storage, ordering) — not based on "databases use StatefulSets"

7. Configure HPA with correct resource requests so it scales based on real load signals

8. Run a structured incident response using `kubectl get events`, `describe pod`, `logs
   --previous`, and `port-forward` — from first alert to root cause in under 10 minutes

9. Explain why your Secrets aren't actually secret without RBAC + etcd encryption, and
   design a real secrets strategy using Sealed Secrets or External Secrets Operator

10. Debug any pod — including distroless and scratch containers — using ephemeral containers
    without modifying the running workload or taking downtime
