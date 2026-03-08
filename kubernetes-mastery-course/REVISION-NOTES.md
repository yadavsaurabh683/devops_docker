# Revision Notes
## Kubernetes Mastery — Complete Cheat Sheet

Use this document for quick review before interviews, on-call rotations, or returning to
Kubernetes after time away. Every entry is a distillation of something you proved empirically
during the course.

---

## Essential kubectl Commands

### Getting Information
```bash
# Get all resources in current namespace with wide output:
kubectl get all -o wide

# Get pods across all namespaces:
kubectl get pods -A

# Get pods with labels shown:
kubectl get pods --show-labels

# Watch pods update in real time:
kubectl get pods -w

# Get a resource in YAML format:
kubectl get pod <name> -o yaml
kubectl get deployment <name> -o yaml

# Extract a specific field with jsonpath:
kubectl get pod <name> -o jsonpath='{.status.phase}'
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# Get events sorted by time (first command in any incident):
kubectl get events --sort-by=.lastTimestamp
kubectl get events --sort-by=.lastTimestamp -n <namespace>

# Get events for a specific resource:
kubectl get events --field-selector involvedObject.name=<pod-name>
```

### Describing Resources
```bash
# Full diagnostic dump — always read the Events section at the bottom:
kubectl describe pod <name>
kubectl describe node <name>
kubectl describe service <name>
kubectl describe deployment <name>
kubectl describe hpa <name>
kubectl describe pvc <name>
kubectl describe networkpolicy <name>

# Check resource allocation on a node:
kubectl describe node <name> | grep -A30 "Allocated resources"
```

### Logs
```bash
# Current logs:
kubectl logs <pod>
kubectl logs <pod> -c <container>      # specific container in multi-container pod

# Follow logs:
kubectl logs -f <pod>

# Previous container logs (most important for CrashLoopBackOff):
kubectl logs --previous <pod>
kubectl logs --previous <pod> -c <container>

# Last N lines:
kubectl logs --tail=50 <pod>

# Logs with timestamps:
kubectl logs --timestamps <pod>

# Multi-pod log tailing (requires stern):
stern <deployment-name>
stern <partial-name> -n <namespace>
```

### Executing Commands
```bash
# Interactive shell:
kubectl exec -it <pod> -- bash
kubectl exec -it <pod> -- sh
kubectl exec -it <pod> -c <container> -- sh

# Run a one-off command:
kubectl exec <pod> -- cat /etc/hosts
kubectl exec <pod> -- env
kubectl exec <pod> -- curl http://localhost:8080/healthz
```

### Applying and Deleting
```bash
# Apply a manifest:
kubectl apply -f manifest.yaml
kubectl apply -f ./directory/

# Delete:
kubectl delete -f manifest.yaml
kubectl delete pod <name>
kubectl delete pod <name> --grace-period=0 --force   # immediate (use sparingly)

# Dry run (server-side — validates against cluster):
kubectl apply -f manifest.yaml --dry-run=server

# Generate YAML without applying:
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

### Rollouts
```bash
# Check rollout status (blocks until complete or failed):
kubectl rollout status deployment/<name>

# View revision history:
kubectl rollout history deployment/<name>

# Roll back to previous version:
kubectl rollout undo deployment/<name>

# Roll back to a specific revision:
kubectl rollout undo deployment/<name> --to-revision=2

# Pause/resume a rollout:
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
```

### Scaling
```bash
kubectl scale deployment/<name> --replicas=5
kubectl autoscale deployment/<name> --min=1 --max=10 --cpu-percent=50
```

### Resource Usage (requires metrics-server)
```bash
kubectl top nodes
kubectl top pods
kubectl top pods -n <namespace>
kubectl top pods --sort-by=memory
```

### Debugging
```bash
# Inject ephemeral debug container into running pod:
kubectl debug -it <pod> --image=busybox --target=<container>

# Create debug copy of a pod (good for CrashLoopBackOff):
kubectl debug <pod> --copy-to=<debug-pod-name> --image=busybox -- sh

# Port-forward to test a service locally:
kubectl port-forward svc/<name> <local-port>:<service-port>
kubectl port-forward pod/<name> <local-port>:<container-port>

# Copy files from/to a pod:
kubectl cp <namespace>/<pod>:/remote/path ./local/path
kubectl cp ./local/file <namespace>/<pod>:/remote/path

# Check RBAC permissions:
kubectl auth can-i get pods -n <namespace>
kubectl auth can-i delete pods --as=system:serviceaccount:<ns>:<sa>
```

### Namespace Management
```bash
# Set default namespace for current context:
kubectl config set-context --current --namespace=<namespace>

# View current context:
kubectl config get-contexts

# Create namespace:
kubectl create namespace <name>
```

### Node Management
```bash
# Mark node unschedulable (no new pods):
kubectl cordon <node>

# Restore node to schedulable:
kubectl uncordon <node>

# Drain node (evict all pods, then cordon):
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

---

## Pod Lifecycle States and What They Mean

| Status | Meaning | First diagnostic step |
|--------|---------|----------------------|
| `Pending` | Not scheduled to any node | `kubectl describe pod` → Events for scheduling failure |
| `ContainerCreating` | Scheduled, kubelet starting containers | `kubectl describe pod` → image pull, volume mount issues |
| `Init:0/1` | Init container running | `kubectl logs <pod> -c <init-container>` |
| `Init:CrashLoopBackOff` | Init container crashing | `kubectl logs --previous <pod> -c <init-container>` |
| `Running` | Containers started. NOT guaranteed to be serving traffic | Check readiness probe: `kubectl get pods` READY column |
| `CrashLoopBackOff` | Container crashing repeatedly | `kubectl logs --previous <pod>` |
| `OOMKilled` | Memory limit exceeded, kernel killed it | `kubectl describe pod` → Last State: OOMKilled, exit code 137 |
| `Evicted` | Removed by kubelet under resource pressure | `kubectl describe pod` → Message shows eviction reason |
| `CreateContainerConfigError` | Bad ConfigMap/Secret reference | `kubectl describe pod` → Events: couldn't find key |
| `ImagePullBackOff` | Cannot pull the image | `kubectl describe pod` → Events: image pull error, check image name/tag |
| `Terminating` | Being deleted — waiting for graceful shutdown | SIGTERM sent; waits `terminationGracePeriodSeconds` (default 30s) |
| `Completed` | All containers exited 0 (Jobs) | Normal for Jobs/CronJobs |
| `Error` | Container exited non-zero | `kubectl logs --previous <pod>` |

---

## Service Types Quick Reference

| Type | clusterIP | External access | DNS resolves to | Use case |
|------|-----------|-----------------|-----------------|----------|
| `ClusterIP` | Assigned | No | ClusterIP → pod IPs via iptables | Internal service calls |
| `NodePort` | Assigned | NodeIP:30000-32767 | ClusterIP | Dev/testing external access |
| `LoadBalancer` | Assigned | Cloud LB IP | ClusterIP | Production external access |
| `Headless` | `None` | No | Pod IPs directly | StatefulSets, service mesh, DNS-based discovery |
| `ExternalName` | None | N/A | CNAME to external DNS | Map external service into cluster DNS |

Service DNS name: `<service>.<namespace>.svc.cluster.local`

Short forms (within same namespace): `<service>` or `<service>.<namespace>`

---

## RBAC Quick Reference

### Resource Types

| Resource | Scope | Bound with |
|----------|-------|-----------|
| `Role` | Namespaced | `RoleBinding` |
| `ClusterRole` | Cluster-wide | `ClusterRoleBinding` (cluster-wide) or `RoleBinding` (namespace-scoped) |

### Binding Combinations

| Subject | Role type | Binding type | Effective scope |
|---------|-----------|--------------|-----------------|
| ServiceAccount | Role | RoleBinding | One namespace |
| ServiceAccount | ClusterRole | RoleBinding | One namespace only |
| ServiceAccount | ClusterRole | ClusterRoleBinding | All namespaces |

### ServiceAccount Identity String
`system:serviceaccount:<namespace>:<serviceaccount-name>`

### Checking Permissions
```bash
# Can the current user:
kubectl auth can-i list pods

# Can a specific ServiceAccount:
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa -n production

# List all ClusterRoleBindings for a ServiceAccount:
kubectl get clusterrolebindings -o wide | grep <sa-name>
```

### Common Built-in ClusterRoles
| ClusterRole | What it allows |
|-------------|----------------|
| `cluster-admin` | Everything. Full cluster control. |
| `admin` | Full namespace control (no cluster-wide resources) |
| `edit` | Read/write most namespace resources |
| `view` | Read-only most namespace resources |

---

## Probe Types Quick Reference

| Probe | Fires | Failure result | Recovery result |
|-------|-------|----------------|-----------------|
| `startupProbe` | Once at container start | Container killed if failureThreshold exceeded | Enables liveness and readiness |
| `readinessProbe` | Continuously | Pod removed from Service endpoints | Pod re-added to endpoints |
| `livenessProbe` | Continuously (after startup succeeds) | Container killed and restarted | Normal pod operation |

### Probe Handler Types
- `httpGet`: HTTP GET to a path/port. Success = 200-399.
- `tcpSocket`: TCP connection attempt to a port. Success = connection established.
- `exec`: Run a command inside the container. Success = exit code 0.
- `grpc`: gRPC health check (Kubernetes 1.24+).

### Probe Timing Fields
```yaml
startupProbe:
  failureThreshold: 30    # attempts before container is killed
  periodSeconds: 3        # time between attempts
  # Max startup window = failureThreshold × periodSeconds = 90s

readinessProbe:
  initialDelaySeconds: 5  # wait before first check
  periodSeconds: 5        # check every 5s
  failureThreshold: 3     # 3 consecutive failures → remove from endpoints
  successThreshold: 1     # 1 success → re-add to endpoints

livenessProbe:
  initialDelaySeconds: 15 # must be long enough for app to start
  periodSeconds: 10
  failureThreshold: 3     # 3 × 10s = 30s of failure before restart
```

---

## StorageClass / PVC / PV Relationship

```
StorageClass
  provisioner: k8s.io/minikube-hostpath  ← who creates the actual storage
  reclaimPolicy: Delete                   ← what happens to PV when PVC is deleted
  volumeBindingMode: Immediate            ← when to provision (Immediate or WaitForFirstConsumer)

PersistentVolume (cluster-scoped)
  capacity.storage: 5Gi
  accessModes: [ReadWriteOnce]
  storageClassName: standard
  persistentVolumeReclaimPolicy: Delete
  status: Bound / Available / Released

PersistentVolumeClaim (namespace-scoped)
  storageClassName: standard  ← matches StorageClass
  accessModes: [ReadWriteOnce]
  resources.requests.storage: 1Gi
  status: Bound / Pending
  volumeName: pvc-abc123      ← bound PV name
```

### Access Modes
| Mode | Short | Meaning |
|------|-------|---------|
| `ReadWriteOnce` | RWO | Read-write by ONE node at a time |
| `ReadOnlyMany` | ROX | Read-only by MANY nodes simultaneously |
| `ReadWriteMany` | RWX | Read-write by MANY nodes simultaneously (needs NFS/cloud) |
| `ReadWriteOncePod` | RWOP | Read-write by ONE pod (Kubernetes 1.22+) |

---

## Resource QoS Classes

| Class | Condition | Eviction priority |
|-------|-----------|-------------------|
| `Guaranteed` | Every container: `requests.cpu == limits.cpu` AND `requests.memory == limits.memory` | Evicted last |
| `Burstable` | At least one container has requests or limits (but not equal for all) | Evicted second |
| `BestEffort` | No container has any requests or limits | Evicted first |

Check a pod's QoS class:
```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

---

## Deployment Strategy Quick Reference

### RollingUpdate (default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1         # Extra pods above desired (absolute or %)
    maxUnavailable: 0   # Pods below desired (0 = zero-downtime update)
```
- Use for stateless apps where old and new versions can run simultaneously
- Requires readiness probes for true zero-downtime

### Recreate
```yaml
strategy:
  type: Recreate
  # All old pods terminated before new pods start — causes downtime
```
- Use for apps where old and new versions cannot coexist (schema changes, singleton processes)

---

## NetworkPolicy Default Behavior

| Situation | Default behavior |
|-----------|-----------------|
| No NetworkPolicy exists for a pod | All ingress and egress allowed |
| Any NetworkPolicy exists targeting a pod | Only explicitly allowed traffic passes; everything else denied |
| Multiple NetworkPolicies target the same pod | Policies are ORed — the union of all allowed traffic |

### Deny-All Ingress Template
```yaml
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress: []         # Empty list = allow nothing
```

### Allow Specific Source Template
```yaml
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: allowed-source
    ports:
    - protocol: TCP
      port: 8080
```

---

## Common Error States and What They Mean

| Error | Cause | Diagnostic command |
|-------|-------|-------------------|
| `CrashLoopBackOff` | Container exits non-zero repeatedly | `kubectl logs --previous <pod>` |
| `ImagePullBackOff` | Cannot pull the container image | `kubectl describe pod` → Events: image pull error |
| `Pending` | No node can satisfy scheduling requirements | `kubectl describe pod` → Events: scheduling failure reason |
| `Evicted` | kubelet removed pod under resource pressure | `kubectl describe pod` → Message: eviction reason |
| `OOMKilled` | Memory limit exceeded, kernel SIGKILL | `kubectl describe pod` → Last State: OOMKilled, ExitCode: 137 |
| `CreateContainerConfigError` | ConfigMap or Secret key reference doesn't exist | `kubectl describe pod` → Events: couldn't find key |
| `ContainerCannotRun` | Container's command/entrypoint failed to start | `kubectl describe pod` → Events: exec format error, or bad entrypoint |
| `Terminating (stuck)` | Finalizers blocking deletion, or pod ignoring SIGTERM | `kubectl describe pod` → Finalizers; `kubectl delete pod --grace-period=0 --force` |
| `0/N nodes available` (Pending) | No nodes meet the pod's scheduling requirements | `kubectl describe pod` → Events: Insufficient memory/cpu, node selector mismatch, taint |

---

## CoreDNS Naming Convention

Full FQDN: `<service-name>.<namespace>.svc.cluster.local`

Examples:
```
nginx-svc.default.svc.cluster.local
postgres.production.svc.cluster.local
api-gateway.microservices.svc.cluster.local
```

Short forms (within same namespace as the pod):
```
nginx-svc                          ← works within the same namespace
nginx-svc.production               ← works from any namespace
nginx-svc.production.svc           ← works from any namespace
nginx-svc.production.svc.cluster.local  ← fully explicit, works anywhere
```

StatefulSet pod DNS (via headless service):
`<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local`
Example: `postgres-0.postgres-headless.production.svc.cluster.local`

DNS configuration in every pod (`/etc/resolv.conf`):
```
nameserver 10.96.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

---

## HPA Quick Reference

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # % of request
```

Scaling formula: `desired = ceil(current × (actual_metric / target_metric))`

Requirements: metrics-server running + resource requests set on all pods

Scale-down stabilization: 5 minutes (default). Set with
`--horizontal-pod-autoscaler-downscale-stabilization` or per-HPA behavior block.

---

## StatefulSet vs Deployment Quick Reference

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| Pod names | Random suffix (myapp-abc12) | Stable ordinal (myapp-0) |
| Pod DNS | Cluster IP via Service | Individual: `<pod>.<headless-svc>.<ns>.svc.cluster.local` |
| Storage | Shared PVC or ephemeral | Each pod gets its own PVC |
| Scale-up order | Parallel (all at once) | Sequential: pod-0 ready before pod-1 starts |
| Scale-down order | Parallel | Reverse ordinal: pod-2 deleted before pod-1 |
| Pod replacement | New random name | Same ordinal name |
| Use case | Stateless web apps, APIs | Databases, queues, any app needing stable identity |

---

## PodDisruptionBudget Quick Reference

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-pdb
spec:
  minAvailable: 2          # OR: maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

- Protects against: `kubectl drain`, rolling updates, cluster upgrades (voluntary disruptions)
- Does NOT protect against: node crashes, OOM kills, network partitions (involuntary)
- `minAvailable`: minimum pods that must remain available
- `maxUnavailable`: maximum pods that can be simultaneously unavailable
