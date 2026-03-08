# Kubernetes Mastery Course
### A 5-Day Hands-On Program for DevOps and Platform Engineers

---

## Deep Reasoning and Analysis

Before you read the course plan, read this section carefully. It explains why this course
is structured the way it is, and what separates engineers who truly understand Kubernetes
from those who merely know how to run `kubectl apply`.

### What Skills Distinguish a Senior Platform Engineer from Average DevOps

The average DevOps engineer who has "worked with Kubernetes" can:
- Run `kubectl apply -f` on a manifest someone else wrote
- Restart a pod when something breaks
- Copy-paste a Helm chart and deploy it

The senior platform engineer can:
- Look at any broken cluster state and reason about what the control loop is doing
- Design workload scheduling that accounts for node pressure, QoS, and topology
- Build a network policy strategy from scratch without breaking existing traffic
- Tell you exactly why a pod is Pending without guessing
- Tune HPA, resource requests, and limits based on observed workload behavior — not defaults
- Debug a CrashLoopBackOff that started at 3AM using only `kubectl` and `kubectl logs --previous`

The difference is not experience in years. It is **depth of mental model**.

### What Real-World Kubernetes Knowledge Most Engineers Lack

The topics that break in production but are rarely covered in courses:

1. **The reconciliation loop** — most engineers treat Kubernetes as a command runner. It is
   not. It is a continuous loop comparing desired state to current state. Everything flows
   from this.

2. **Service routing internals** — most engineers know Services exist. Almost none can explain
   what kube-proxy does to iptables to make them work, or why a Service with zero Endpoints
   silently drops traffic.

3. **Resource requests vs limits** — the distinction between what the scheduler sees
   (requests) and what the kernel enforces (limits) is one of the most consequential gaps
   in production Kubernetes knowledge. Getting this wrong causes mysterious evictions.

4. **RBAC at the chain level** — engineers create ServiceAccounts and Roles but often cannot
   trace the full permission chain from a pod's identity to a specific API call.

5. **StatefulSet guarantees** — engineers use StatefulSets for databases but cannot explain
   why pod-0 always starts before pod-1, what the headless service provides, or what happens
   to the PVC when the pod is deleted.

6. **Probe design** — the critical distinction between liveness, readiness, and startup
   probes. Misconfigured probes cause subtle production failures that look like random crashes.

### What Concepts Are Critical for Mastering Kubernetes

These are not optional depth. Each one is a load-bearing concept:

- **Control loop / reconciliation** — every Kubernetes component (scheduler, controller-manager,
  kubelet) is a reconciliation loop. Internalizing this makes all behavior predictable.
- **Pod as the atomic unit** — not containers. Pods share network and storage namespaces.
- **Labels and selectors** — the connective tissue of Kubernetes. Services, Deployments, HPA,
  NetworkPolicies all work through label selectors. A one-character typo silently breaks routing.
- **etcd as the source of truth** — all objects live in etcd. What you see in kubectl is
  always a read from etcd. Understand that writes to kubectl go to the API server → etcd.
- **Scheduler** — binds pods to nodes based on requests (not limits), taints, tolerations,
  affinity rules. A pod that is Pending forever is telling you the scheduler cannot find a
  valid node.
- **QoS classes** — BestEffort, Burstable, Guaranteed. This determines eviction order under
  node pressure. Production workloads must be Guaranteed.

### Mistakes Most Engineers Make When Learning Kubernetes

1. **Learning kubectl before understanding the control loop.** Commands without a model
   produce cargo-cult operations — doing things without understanding why.

2. **Treating YAML as configuration and not as API calls.** Every kubectl apply is an API
   call to the API server. The YAML is the payload. Understanding this changes how you debug.

3. **Ignoring resource requests.** Pods without requests are BestEffort — they are the first
   to be evicted when a node is under pressure. This is not obvious until it happens at 2 AM.

4. **Using kubectl delete pod to "fix" problems.** If a pod is crashing, deleting it does not
   fix the crash — the Deployment controller recreates it immediately from the same spec.
   Fix the spec, not the pod.

5. **Copying NetworkPolicy YAML without understanding what "deny all" means.** Without any
   NetworkPolicy, all traffic is allowed. The moment you add one policy to a pod, only traffic
   matching that policy is allowed. This causes silent breakage.

6. **Not reading `kubectl describe`.** This is the most informative command in Kubernetes.
   Engineers who skip straight to logs miss half the diagnostic information.

### How Exercises Develop Intuition vs Memorizing Commands

Every exercise in this course follows a pattern:
1. Set up a working state
2. Break something deliberately or observe a constraint
3. Diagnose using kubectl describe, logs, events, exec
4. Fix it by changing the underlying cause (not restarting)
5. Verify the fix by observing the control loop converge

This forces you to build the mental model, not memorize outputs. When you face an unknown
failure in production, the diagnostic process — not the specific command — is what saves you.

### Why This Approach Produces Better Engineers

Typical Kubernetes courses show you a Deployment YAML, run it, confirm it works, move on.
This builds familiarity, not competence.

This course shows you what happens when Kubernetes does NOT work — and forces you to reason
from first principles about why. After five days of breaking and fixing real systems, your
mental model of Kubernetes is built from evidence, not documentation.

---

## Skill Gaps in Typical DevOps Engineers

| Skill Area | What Most Engineers Can Do | What This Course Adds |
|---|---|---|
| Deployments | Apply, delete, check status | Rolling update mechanics, maxSurge, rollback |
| Services | Create ClusterIP, access pods | iptables routing, endpoints, DNS chain |
| Config | kubectl create configmap | Volume vs env reload behavior, secret encoding |
| Storage | Create PVC and attach | PV lifecycle, StorageClass, Retain vs Delete |
| RBAC | Create Role and bind it | Full identity chain from pod SA to API call |
| Debugging | kubectl logs, describe | Ephemeral containers, OOM forensics, eviction |
| Networking | Basic service access | NetworkPolicy semantics, CoreDNS, kube-proxy |
| Scaling | kubectl scale | HPA mechanics, cool-down, metrics-server dependency |

---

## Learning Design Philosophy

- **Problem-first.** You encounter the failure before you encounter the solution.
- **Evidence-based.** Every claim is verified with an actual kubectl command.
- **Production-relevant.** Every exercise is a scenario you will encounter in real work.
- **Open-source only.** No cloud accounts. Runs on a laptop with minikube.
- **No fluff.** No motivational sections. Only technical content and actionable steps.

---

## Prerequisites

You need:
- Completion of the Docker Mastery Course (or equivalent: you understand what a container is)
- A laptop with at least 8 GB RAM (16 GB preferred for Day 5 exercises)
- Linux, macOS, or Windows with WSL2
- Basic Linux comfort: bash, file paths, curl, ps, grep

> **Windows users:** Use WSL2 for all exercises. All commands are written for Linux/macOS shell.
> Open a WSL2 terminal and run everything from there. minikube runs inside WSL2 perfectly.

---

## Tool Installation

Install all tools before starting Day 1. Run these from your Linux/WSL2 terminal.

### 1. minikube (primary local Kubernetes cluster)
```bash
# Linux / WSL2:
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version

# macOS (with Homebrew):
brew install minikube

# Verify:
minikube version
```

> **Hint:** minikube requires a container or VM driver. On WSL2, use the `docker` driver.
> On macOS, use the `docker` or `hyperkit` driver. Set it once:
> `minikube config set driver docker`

### 2. kubectl
```bash
# Linux / WSL2:
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# macOS:
brew install kubectl
```

### 3. helm (needed for Day 5)
```bash
# Linux / WSL2:
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# macOS:
brew install helm
```

### 4. k9s (optional — terminal UI for Kubernetes)
```bash
# Linux / WSL2:
K9S_VER=$(curl -s https://api.github.com/repos/derailed/k9s/releases/latest \
  | grep tag_name | cut -d '"' -f4)
curl -LO "https://github.com/derailed/k9s/releases/download/${K9S_VER}/k9s_Linux_amd64.tar.gz"
tar -xf k9s_Linux_amd64.tar.gz k9s
sudo mv k9s /usr/local/bin/
k9s version

# macOS:
brew install k9s
```

### 5. stern (optional — multi-pod log tailing)
```bash
# Linux / WSL2:
STERN_VER=$(curl -s https://api.github.com/repos/stern/stern/releases/latest \
  | grep tag_name | cut -d '"' -f4 | tr -d v)
curl -LO "https://github.com/stern/stern/releases/download/v${STERN_VER}/stern_${STERN_VER}_linux_amd64.tar.gz"
tar -xf stern_${STERN_VER}_linux_amd64.tar.gz stern
sudo mv stern /usr/local/bin/
stern --version

# macOS:
brew install stern
```

### 6. Start minikube and enable required addons
```bash
# Start with enough resources:
minikube start --cpus=2 --memory=4096 --driver=docker

# Enable metrics-server (required for Day 4 HPA exercise):
minikube addons enable metrics-server

# Enable ingress (useful for Day 2 exploration):
minikube addons enable ingress

# Verify everything is running:
kubectl get nodes
kubectl get pods -n kube-system
```

### 7. Verify your full setup
```bash
echo "=== minikube ===" && minikube version | head -1
echo "=== kubectl ===" && kubectl version --client --short 2>/dev/null || kubectl version --client
echo "=== helm ===" && helm version --short
echo "=== Node ready ===" && kubectl get nodes
echo "=== metrics-server ===" && kubectl get pods -n kube-system -l k8s-app=metrics-server
echo ""
echo "All tools ready. You can start Day 1."
```

> **kind as an alternative:** If minikube does not work in your environment, use kind
> (Kubernetes IN Docker): `brew install kind` or download from `kind.sigs.k8s.io`.
> Create a cluster: `kind create cluster --name mastery`. All kubectl commands work identically.

---

## Course Structure

```
kubernetes-mastery-course/
├── README.md                  ← You are here
├── SUMMARY.md                 ← End-of-course concept summary (read after Day 5)
├── REVISION-NOTES.md          ← Complete cheat sheet for all concepts
├── FINAL-CHALLENGES.md        ← 22 challenging questions to test mastery
│
├── day1/
│   └── README.md              ← Architecture, control loop, pods, CrashLoopBackOff, resources
├── day2/
│   └── README.md              ← Services, DNS, kube-proxy, NetworkPolicy
├── day3/
│   └── README.md              ← ConfigMaps, Secrets, PVCs, RBAC, Namespaces
├── day4/
│   └── README.md              ← Deployments, probes, StatefulSets, HPA
└── day5/
    └── README.md              ← Production debugging, eviction, incident simulation, Helm
```

---

## How to Go Through This Course

### Rule 1: Do Every Exercise — No Skipping
Reading without running will not build diagnostic intuition. You must see the output, observe
the state transitions, and trace the cause yourself. The learning is in the doing.

### Rule 2: Read the Questions at the End of Each Exercise
Every exercise ends with **"Questions to answer."** Stop. Write down your answers before
moving to the next exercise. If you cannot answer, re-trace the exercise. The question is
telling you exactly what the exercise was teaching.

### Rule 3: Before Looking at Hints — Try First
`> Hint:` callouts exist for things engineers genuinely get stuck on. Spend 10 minutes
trying first. Getting stuck and breaking through is where lasting understanding forms.

### Rule 4: Complete the Day Checkpoint Before Moving On
Each day ends with a checkpoint of 7 questions. Answer them without looking at the material.
If you cannot answer one, find the exercise that covers it and re-run it. Do not proceed to
the next day with unresolved gaps.

### Rule 5: Use `kubectl describe` Constantly
In Docker, the habit is `docker inspect`. In Kubernetes, the equivalent is `kubectl describe`.
Every time something does not behave as expected, run `kubectl describe <resource> <name>`
before anything else. Events at the bottom of the output are almost always the answer.

---

## Day-by-Day Overview

| Day | Theme | Key Skills |
|-----|-------|------------|
| **Day 1** | Architecture, Pods, and the Control Loop | Control loop, pod lifecycle, CrashLoopBackOff, init containers, QoS |
| **Day 2** | Kubernetes Networking | ClusterIP, DNS, kube-proxy, NodePort, NetworkPolicy |
| **Day 3** | Configuration, Storage, and Access Control | ConfigMaps, Secrets, PVCs, RBAC, Namespaces, ResourceQuota |
| **Day 4** | Production Workloads | Rolling updates, probes, rollback, StatefulSets, HPA |
| **Day 5** | Debugging and Production Operations | kubectl debug, OOM forensics, eviction, incident simulation, Helm |

---

## Estimated Time Per Day

Each day is designed for **5–6 focused hours**.

Suggested schedule:
- Morning (2–3 hours): Complete exercises up to the midpoint of the day
- Afternoon (2–3 hours): Complete remaining exercises + Day Checkpoint
- Evening (30 min): Review Day Summary table and answer checkpoint questions from memory

---

## Common Mistakes to Avoid

- **Do not** `kubectl delete pod` to fix crashes — the controller recreates it from the same
  broken spec. Fix the Deployment spec instead.
- **Do not** skip reading `kubectl describe` output. The Events section at the bottom tells
  you exactly what Kubernetes has tried and why it failed.
- **Do not** set resource limits without also setting requests. Limits without requests
  default requests to the same value — understand what that does to scheduling.
- **Do not** copy NetworkPolicy YAML without understanding default behavior. Adding any policy
  to a pod changes the default from "allow all" to "deny all except this policy."
- **Do not** treat Secrets as encrypted. They are base64 encoded. Plan for etcd encryption
  at rest and access controls separately.
- **Do not** ignore probe configuration. A deployment without readiness probes will send
  traffic to unready pods during rolling updates.

---

## Quick Reference: Most Useful kubectl Commands

```bash
# Get all resources with wide output (shows node assignment):
kubectl get pods -o wide

# Watch pod status in real time:
kubectl get pods -w

# Full diagnostic dump for any resource:
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe service <svc-name>

# Follow logs for a crashing container:
kubectl logs -f <pod-name>
kubectl logs --previous <pod-name>   # logs from the PREVIOUS crashed container

# Execute commands inside a running pod:
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -c <container-name> -- sh   # specific container in multi-container pod

# Apply / delete manifests:
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml

# Rolling update operations:
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Scale a deployment:
kubectl scale deployment/<name> --replicas=5

# Watch events sorted by time (the most useful debugging command):
kubectl get events --sort-by=.lastTimestamp

# Check resource usage (requires metrics-server):
kubectl top pods
kubectl top nodes

# Port-forward to debug a service locally:
kubectl port-forward svc/<name> 8080:80

# Debug with an ephemeral container:
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# Copy files from a pod:
kubectl cp <pod-name>:/path/to/file ./local-file

# Apply to a specific namespace:
kubectl apply -f manifest.yaml -n <namespace>
kubectl get pods -n <namespace>
kubectl get pods -A   # all namespaces
```

---

## After Completing This Course

Once you finish all 5 days and can answer the Final Challenges without hesitation, you will:

1. Understand what Kubernetes actually is — a distributed reconciliation system — not a
   container runner
2. Debug any Kubernetes failure using kubectl alone, without guessing or restarting pods
3. Design workload configurations that are production-safe: correct probes, resource
   requests, QoS class, and rollout strategy
4. Build and enforce RBAC policies that follow least-privilege from the start
5. Explain to any engineer why their pod is Pending, Evicted, CrashLoopBackOff, or OOMKilled
   — and what to do about each
6. Operate confidently in any Kubernetes environment, on any cloud provider, because you
   understand the abstractions beneath the provider-specific tooling

---

*This course uses only open-source tools. No cloud accounts required. Runs entirely on a laptop.*
