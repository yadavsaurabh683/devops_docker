# Final Challenges
## Complex Questions to Test Your Kubernetes Mastery

These questions are designed to be answered **without running kubectl** — they test conceptual
depth, diagnostic reasoning, and production-level thinking. If you can answer all of them
confidently, you have genuinely mastered Kubernetes fundamentals.

Some questions require multi-part answers. Some require you to trace through multiple system
layers (pod → API server → etcd → controller → kubelet → container runtime). That is
intentional.

---

## Category 1: Core Concepts & Architecture

**Q1.**
A junior engineer says: "Kubernetes crashed — my pod is gone. I'll just re-apply the YAML."

What is wrong with this mental model? Explain what actually happens when a pod "disappears"
in a healthy cluster, what component is responsible for detecting and responding to that
disappearance, and under what circumstances re-applying YAML would be the correct action
vs the wrong action.

---

**Q2.**
You have a Deployment with 3 replicas. You run `kubectl delete pod <pod-name>` for one of
the pods. Forty seconds later, `kubectl get pods` shows 3 pods, but one has a different
name and a RESTARTS count of 0.

Trace the exact sequence of events that occurred from the moment you ran `kubectl delete pod`
to the moment the new pod appeared. Name every component that acted and in what order.

---

**Q3.**
A pod has been in `Pending` state for 20 minutes. No events show an error. You run
`kubectl describe pod` and see:

```
Events:
  Warning  FailedScheduling  ... 0/1 nodes are available:
  1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }
```

Explain: What is a taint? What is a toleration? Why does a single-node minikube cluster
work despite this taint? What change would make this pod schedule on that node?

---

**Q4.**
Explain etcd's role in Kubernetes by answering:
- What is stored in etcd?
- When you run `kubectl apply -f deployment.yaml`, which component writes to etcd?
- When the Deployment controller creates a pod, does it write to etcd directly?
- If etcd becomes unavailable, can existing running pods continue to run? Why or why not?
- If you restore etcd from a backup taken 1 hour ago, what happens to pods that were
  created in the last hour?

---

## Category 2: Networking

**Q5.**
You create a Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: production
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 3000
  type: ClusterIP
```

Trace the complete network path when a pod in the `staging` namespace makes a request to
`api-svc.production.svc.cluster.local:8080` and it reaches the `api` pod's port 3000.

Name every component involved: DNS query path, CoreDNS response, iptables chains, packet
rewriting, final delivery.

---

**Q6.**
You have 5 pods behind a ClusterIP Service. Pod-3 is experiencing a memory leak and is
responding slowly (2-second responses). The other 4 pods respond in 50ms.

The Service has no session affinity configured. Clients are sending requests every 100ms.

Explain:
- Is pod-3 receiving any traffic? How is kube-proxy distributing traffic?
- What would you do to temporarily remove pod-3 from the Service without deleting it or
  the Deployment?
- After you fix the memory leak and redeploy, how does pod-3 re-enter the Service?
- What probe would have automatically removed pod-3 without your manual intervention?

---

**Q7.**
You apply this NetworkPolicy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

Answer each:
1. Can the `frontend` pod reach `api` on port 8080? On port 9090?
2. Can the `api` pod reach `database` on port 5432? On port 3306?
3. Can the `api` pod reach external DNS (8.8.8.8:53)?
4. Can a pod with labels `app: monitoring` reach `api` on port 8080?
5. What single addition to the Egress rules would allow the API pod to perform DNS lookups?

---

**Q8.**
A team member applies a NetworkPolicy to the `database` pods that blocks all ingress.
Five minutes later, all the API pods start returning 500 errors. The API pods themselves
are healthy — readiness probes pass. The database pod is running.

What is happening? Walk through the exact sequence of events from "NetworkPolicy applied"
to "API pods return 500". What is the fastest way to confirm your diagnosis without modifying
any running pods?

---

## Category 3: Configuration & Storage

**Q9.**
You have a deployment that reads its configuration from a ConfigMap mounted as a volume.
At 2 AM, you push an updated ConfigMap with a corrected connection string.

Answer:
- Does the running pod automatically use the new value?
- How long until the file inside the pod reflects the change?
- If your application only reads the config file at startup, does it benefit from the
  automatic volume sync?
- What is the complete approach to ensure a config change takes effect without downtime?

---

**Q10.**
A developer argues: "I store my database password in a Kubernetes Secret. It's secure because
the Secret is base64 encoded and stored separately from the application config."

Identify every false assumption in this statement. For each false assumption, explain the
real security property (or lack thereof) and what the correct approach would be.

Then describe the complete secrets security model you would implement for a production cluster.

---

**Q11.**
You have a StatefulSet running a database with 3 replicas. Each replica has a PVC with
`persistentVolumeReclaimPolicy: Retain`.

Answer:
- You delete pod `db-1`. What happens to the PVC `data-db-1`?
- You scale the StatefulSet from 3 to 1. What happens to PVCs `data-db-1` and `data-db-2`?
- You delete the StatefulSet entirely. What happens to ALL three PVCs?
- You create a new StatefulSet with the same name and 3 replicas. Will the pods bind to
  the original PVCs, or will new PVCs be created?
- What is the operational implication of `Retain` vs `Delete` for a production database?

---

**Q12.**
A pod in namespace `app` with ServiceAccount `app-runner` tries to list pods in namespace
`monitoring` and gets:

```
Error from server (Forbidden): pods is forbidden: User
"system:serviceaccount:app:app-runner" cannot list resource "pods" in API group ""
in the namespace "monitoring"
```

Write the complete RBAC configuration (Role or ClusterRole, and Binding) that would grant
`app-runner` in namespace `app` the ability to list and get pods in namespace `monitoring`
only — with minimum required privilege. Explain each field.

---

## Category 4: Workloads & Scheduling

**Q13.**
You have a Deployment with:
```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 2
```

No readiness probe is configured. You push a new image version.

Walk through:
- How many pods are running at the peak of the update?
- How many pods could receive traffic from a Service during the update?
- What is the minimum number of pods serving traffic at any point?
- If the new image has a 45-second startup time before it's actually serving, what happens
  to traffic during those 45 seconds?
- What two changes would you make to this Deployment spec to prevent traffic loss?

---

**Q14.**
You have a StatefulSet `kafka` with 3 replicas and a headless service `kafka-headless`.

A producer application needs to connect to a specific Kafka broker by its stable address —
not load-balanced. The producer is in namespace `streaming`, the StatefulSet is in namespace
`data`.

Write the exact hostname the producer must use to reach broker `kafka-1`. Explain why a
regular ClusterIP Service would not work for this use case.

---

**Q15.**
Your HPA is configured with `minReplicas: 2`, `maxReplicas: 20`, and `averageUtilization: 50`
for CPU.

At 9 AM, traffic spikes. CPU utilization hits 95%. The HPA fires and scales to 10 replicas.
By 9:30 AM, traffic drops. CPU falls to 5%. You expect scale-down to start, but at 9:35 AM
there are still 10 replicas.

Explain why. What is the mechanism that prevents immediate scale-down? What would the
replicas be at 9:35 AM if you had set `stabilizationWindowSeconds: 60` in the behavior
section instead of using the default?

---

**Q16.**
You are doing a rolling update of a Deployment. After 3 minutes, `kubectl rollout status`
has not returned. `kubectl get pods` shows:

```
NAME                    READY   STATUS    RESTARTS   AGE
myapp-abc123-1          1/1     Running   0          45m
myapp-abc123-2          1/1     Running   0          45m
myapp-abc123-3          1/1     Running   0          45m
myapp-def456-1          0/1     Running   0          3m
```

The new pod (def456-1) shows `Running` but `0/1` in the READY column. Its exit status is
`Running`. No errors in `kubectl describe pod`.

What is causing the rollout to hang? What is the pod doing? What must happen for the rollout
to continue? If this is a legitimate slow-start, what probe should be configured and how?

---

## Category 5: Debugging & Production Incidents

**Q17.**
A pod has been running for 6 hours. It suddenly enters `CrashLoopBackOff`. RESTARTS shows 8.
The current container is running but crashing within 2 seconds of start.

List every diagnostic step in exact order, with the kubectl command for each step, and what
you are specifically looking for at each step. What is the one command that most engineers
skip but is often the fastest path to root cause?

---

**Q18.**
You receive an alert: "Pod `payment-processor-xyz123` OOMKilled. Restarted automatically."
This is the 4th time this week.

Design your complete investigation and remediation plan:
1. How do you confirm OOMKill is the cause? (command + what you look for)
2. How do you determine the memory usage pattern before the kill? (what tool, what limitation)
3. How do you determine WHERE in the code memory is growing? (what type of tool)
4. What is the correct immediate mitigation? What are the risks of that mitigation?
5. What monitoring change would alert you BEFORE the OOMKill occurs next time?

---

**Q19.**
You are draining a node for maintenance. It has 15 pods running. The drain command hangs:

```
kubectl drain node-2 --ignore-daemonsets --delete-emptydir-data
evicting pod production/api-7d9c5b-1
error when evicting pods/"api-7d9c5b-1" -n "production"
(will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
```

Answer:
- What is a PodDisruptionBudget and what is it protecting in this scenario?
- How do you inspect the PDB to understand what constraint it's enforcing?
- The PDB says `minAvailable: 2` and there are currently 2 replicas of `api`. How do you
  resolve this to allow the drain to proceed safely?
- What would happen to the `api` service if you forced the eviction anyway?

---

**Q20.**
A cluster was working fine yesterday. Today, all new pods are stuck in `Pending` state.
Existing running pods are unaffected.

`kubectl describe pod <any-new-pod>` shows:
```
Events:
  Warning  FailedScheduling  ... 0/3 nodes are available:
  3 node(s) didn't have free ports for the requested pod ports.
```

Explain what this error means. What is the most likely cause? How do you find which
port is conflicting and on which node? How do you fix it without deleting any running pods?

---

## Synthesis Challenge (Final Boss)

**Q21 — The 2 AM Incident**

You are on call. At 2:07 AM you receive:

> "CRITICAL: Production API response time spiked from 80ms to 12s. Error rate 22%.
> Database team says Postgres is healthy. No deployments in 18 hours.
> 3 API pods are running. 1 API pod shows OOMKilled in the last 10 minutes."

You have kubectl access to the cluster. The API runs in namespace `production`. The API
deployment has 3 replicas, each with 512Mi memory limit. The stack is: nginx ingress →
api deployment → postgres StatefulSet.

Write your complete incident response:

1. **First 60 seconds**: Write the exact sequence of 5 kubectl commands you run, in order,
   and what specific information you are looking for from each.

2. **Diagnosis**: The API pod that OOMKilled was pod `api-7f9d4c-2`. It has restarted and
   is now Running but READY shows 0/1. Explain what this means and why it matters for traffic.

3. **Traffic isolation**: How do you remove `api-7f9d4c-2` from receiving any new requests
   without deleting it or modifying the Deployment? Write the exact command.

4. **Root cause identification**: You find in the pod's `--previous` logs:
   ```
   ERROR: Cache fill: loaded 847,293 objects into in-memory cache. Size: 489MB
   ```
   What is the root cause? What Kubernetes configuration was insufficient?

5. **Immediate fix**: Write the exact kubectl command to adjust the memory limit for the
   Deployment to 1Gi without taking the other 2 pods offline.

6. **Rollback decision**: After changing the memory limit, the pods restart (rolling update).
   During the rollout, error rate jumps to 45%. Do you continue the rollout or roll back?
   What is your decision criterion?

7. **Post-incident improvements**: Write 4 specific Kubernetes configuration changes (with
   the YAML field names) that would prevent this exact incident from happening again.
   For each, explain what it would have changed.

---

**Q22 — The Architecture Review**

A team is proposing this Kubernetes architecture for a new microservices application:

```yaml
# All 8 microservices in a single namespace: 'myapp'
# Each microservice:
#   - No resource requests or limits
#   - No readiness or liveness probes
#   - Using the 'default' ServiceAccount with cluster-admin ClusterRoleBinding
#   - ConfigMaps used for database passwords and API keys
#   - All inter-service communication uses NodePort services
#   - Single Deployment for database (postgres), not StatefulSet, no PVC
```

Identify every problem with this architecture. For each problem:
- Name the specific issue
- Explain the production consequence
- Provide the corrected configuration approach (be specific — name the YAML fields)

There are at least 8 distinct problems. Find them all.

---

## Scoring Guide

| Questions answered correctly | Assessment |
|---|---|
| 20–22 | Expert-level Kubernetes engineer. Ready to lead platform architecture. |
| 16–19 | Strong practitioner. Can operate production clusters and mentor others. Minor gaps. |
| 12–15 | Solid intermediate. Can handle most production scenarios. Re-do Day 3 and Day 4. |
| 8–11 | Foundational knowledge but significant gaps in advanced operations. Re-do Days 2–5. |
| Below 8 | Return to Day 1. Build the mental model before advancing. |

---

## Answer Approach Guide

For every question you cannot answer:

1. Identify which Day's content covers it:
   - Q1-Q4: Day 1 (architecture, control loop, kubelet)
   - Q5-Q8: Day 2 (networking, services, NetworkPolicy)
   - Q9-Q12: Day 3 (config, secrets, storage, RBAC)
   - Q13-Q16: Day 4 (deployments, probes, StatefulSets, HPA)
   - Q17-Q22: Day 5 (debugging, incidents)

2. Return to that day's exercises — not to read, but to RE-RUN with specific attention
   to the mechanism behind the answer

3. Write your answer in your own words before checking anything. The act of writing exposes
   exactly where the gap is.

The goal is not a score. The goal is that when your production cluster breaks at 3 AM, you
have a systematic mental model that tells you WHAT to look at and WHY — not just a list of
commands to try in random order.
