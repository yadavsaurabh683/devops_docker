# Day 2: Kubernetes Networking
## Theme: Services, DNS, kube-proxy, and Network Policies

**Goal:** By end of Day 2, you will understand how traffic flows through a Kubernetes cluster
from one pod to another. You will understand what a Service actually does at the iptables level,
how CoreDNS resolves names, and how NetworkPolicy enforces isolation. You will debug a broken
Service using nothing but kubectl.

**Estimated time:** 5–6 hours

---

## Background: Kubernetes Has No Routing Magic

Traffic between pods works because of three cooperating systems:
1. **kube-proxy** — writes iptables (or IPVS) rules on every node that redirect Service IP
   traffic to actual pod IPs. It is not a proxy in the traditional sense; it programs the
   kernel's packet routing.
2. **CoreDNS** — a DNS server running in the kube-system namespace that resolves Service names
   to ClusterIPs. Every pod in the cluster is configured to use it.
3. **Container Network Interface (CNI) plugin** — sets up pod networking so that every pod
   gets a routable IP and can reach other pods directly. minikube uses kindnet by default.

Understanding these three systems lets you debug any networking issue in Kubernetes.

---

## Exercise 2.1 — ClusterIP: How a Service Actually Routes Traffic

**What you'll learn:** What a ClusterIP Service is, how kube-proxy makes it work via iptables,
and how traffic load-balances across multiple pod endpoints.

**Instructions:**

1. Create a namespace to keep today's work clean:
```bash
kubectl create namespace day2
kubectl config set-context --current --namespace=day2
# All kubectl commands now default to day2 namespace
```

2. Deploy 3 nginx replicas:
```bash
kubectl create deployment nginx-cluster --image=nginx:alpine --replicas=3
kubectl get pods -l app=nginx-cluster -o wide
# Note the pod IPs — each pod has a unique IP
```

3. Create a ClusterIP Service for the deployment:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: day2
spec:
  selector:
    app: nginx-cluster
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

kubectl get service nginx-svc
# Note the CLUSTER-IP — this is a virtual IP that doesn't belong to any single pod
```

4. Inspect the service endpoints — the actual pod IPs the service routes to:
```bash
kubectl get endpoints nginx-svc
# You will see 3 IP:port pairs — one for each pod
kubectl describe service nginx-svc
# Read the Endpoints line carefully
```

5. Run a debug pod and curl the service from inside the cluster:
```bash
kubectl run debug-pod --image=curlimages/curl --restart=Never -it --rm \
  -- sh -c '
    echo "=== Curl by service name ==="
    for i in 1 2 3 4 5 6; do curl -s nginx-svc | grep "Welcome to nginx" | head -1; done
  '
```

The service name resolves and load-balances. Each curl may hit a different pod (you'll see
different server headers with different pod responses if you add pod hostnames to nginx).

6. Look at the iptables rules kube-proxy created for the service:
```bash
minikube ssh

# Get the service ClusterIP:
# (Run this in a separate terminal: kubectl get svc nginx-svc -o jsonpath='{.spec.clusterIP}')

# On the minikube node, list the NAT rules:
sudo iptables -t nat -L KUBE-SERVICES --line-numbers | grep nginx
sudo iptables -t nat -L KUBE-SVC-<tab-complete-or-find-by-grep> 2>/dev/null | head -20
# You will see rules that randomly select one of the pod IPs (statistic mode nth)

# Each pod gets a KUBE-SEP (Service Endpoint) chain:
sudo iptables -t nat -L -n | grep -A3 "KUBE-SEP"

exit
```

**Questions to answer:**
> 1. What creates the iptables rules for a Service? What component is responsible?
>    (Answer: kube-proxy. It watches the API server for Service and Endpoint changes, then
>    programs corresponding iptables rules on every node. When a pod is added or removed,
>    kube-proxy updates the rules.)
> 2. If you scale the nginx deployment to 5 replicas, what changes in iptables?
>    (Answer: kube-proxy detects the new Endpoints objects and adds two new KUBE-SEP chains
>    to the KUBE-SVC chain. The probability weights in the statistic rules are updated so
>    each pod gets an equal share of traffic.)
> 3. What is the ClusterIP? Can you ping it from outside the cluster?
>    (Answer: ClusterIP is a virtual IP address that exists only within the cluster's iptables
>    rules — it is not assigned to any network interface. You cannot reach it from outside
>    the cluster, and you cannot ping it because ICMP is not handled by kube-proxy's iptables
>    rules — only TCP/UDP for the configured ports.)

> **Hint:** If iptables commands feel overwhelming, `kubectl describe endpoints nginx-svc` gives
> you the same information at the Kubernetes layer. The iptables commands are for building the
> mental model of what kube-proxy actually does — not for daily operations.

---

## Exercise 2.2 — DNS Deep Dive: The Full Resolution Chain

**What you'll learn:** How CoreDNS works, what the full FQDN of a Service is, and how the
DNS search domain list in a pod's `/etc/resolv.conf` enables short-name resolution.

**Instructions:**

1. Start an interactive debug pod:
```bash
kubectl run dns-debug --image=busybox --restart=Never -it --rm -- sh
```

Inside the pod:

2. Read the DNS configuration:
```bash
cat /etc/resolv.conf
# Expected output:
# nameserver 10.96.0.10          ← CoreDNS ClusterIP
# search day2.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

3. Resolve the nginx service using just the short name:
```bash
nslookup nginx-svc
# CoreDNS resolves this because 'day2.svc.cluster.local' is in the search list
# It tries: nginx-svc.day2.svc.cluster.local → match found
```

4. Resolve using progressively longer names:
```bash
nslookup nginx-svc.day2
nslookup nginx-svc.day2.svc
nslookup nginx-svc.day2.svc.cluster.local
# All resolve to the same ClusterIP
```

5. Resolve the Kubernetes API server (built-in service):
```bash
nslookup kubernetes.default
nslookup kubernetes.default.svc.cluster.local
# Both resolve to the API server ClusterIP (typically 10.96.0.1)
```

6. Exit the debug pod:
```bash
exit
```

7. From outside the pod, verify CoreDNS is running in kube-system:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get service -n kube-system kube-dns
# The kube-dns service has ClusterIP 10.96.0.10 — matches nameserver in resolv.conf
```

8. Look at the CoreDNS ConfigMap to understand its configuration:
```bash
kubectl describe configmap coredns -n kube-system
# Read the Corefile — it shows which zones CoreDNS handles and forwards externals to
```

**Questions to answer:**
> 1. What is the full FQDN of a service named nginx-svc in namespace day2?
>    (Answer: `nginx-svc.day2.svc.cluster.local`)
> 2. When would you need the full FQDN vs the short name?
>    (Answer: You need the full FQDN when accessing a service from a DIFFERENT namespace —
>    the search domains only include your own namespace. Short names work within the same
>    namespace. Use `<service>.<namespace>` as the minimal cross-namespace form.)
> 3. What is ndots:5 in resolv.conf and why does it matter?
>    (Answer: ndots:5 means if a hostname has fewer than 5 dots, the resolver tries adding
>    the search domains before trying it as an absolute name. This makes `nginx-svc` resolve
>    via the search list. It also means external FQDNs like `api.example.com` (2 dots)
>    generate multiple DNS queries before resolving — a known latency issue in high-throughput
>    pods.)

> **Hint:** To debug DNS from inside any pod that has `curl` or `wget`: run
> `kubectl exec -it <pod> -- cat /etc/resolv.conf` without needing to exec into a shell
> interactively. For production debugging, always have a debug image like `busybox` or
> `nicolaka/netshoot` available to pull into any namespace.

---

## Exercise 2.3 — Debug a Broken Service (Label Selector Mismatch)

**What you'll learn:** How to diagnose a Service that appears correct but routes no traffic.
The cause: a label selector mismatch — the most common Service misconfiguration.

**Instructions:**

1. Deploy an app with specific labels:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: day2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      version: v1
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```

2. Create a Service with a DELIBERATELY WRONG selector:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: day2
spec:
  selector:
    app: webapp
    version: v2        # WRONG: pods have version: v1
  ports:
  - port: 80
    targetPort: 80
EOF
```

3. Observe the broken state:
```bash
kubectl get endpoints webapp-svc
# Output: webapp-svc   <none>   ...
# <none> means zero endpoints — the selector matched nothing
```

4. Diagnose using describe:
```bash
kubectl describe service webapp-svc
# Read: Selector: app=webapp,version=v2
# Read: Endpoints: <none>

# Now check what labels the pods actually have:
kubectl get pods -l app=webapp --show-labels
# Output shows: version=v1
```

5. The mismatch is clear: service wants v2, pods have v1. Fix it:
```bash
kubectl patch service webapp-svc \
  -p '{"spec":{"selector":{"app":"webapp","version":"v1"}}}'

kubectl get endpoints webapp-svc
# Now shows 2 IP:port pairs — the 2 webapp pods
```

6. Verify the fix works:
```bash
kubectl run verify --image=curlimages/curl --restart=Never -it --rm \
  -- curl -s webapp-svc | grep -o "Welcome to nginx"
# Output: Welcome to nginx
```

**Questions to answer:**
> 1. What does `Endpoints: <none>` mean in `kubectl describe service`?
>    (Answer: The service's label selector matched zero pods. The service exists but has no
>    backends to route traffic to. All requests to the ClusterIP will be silently dropped.)
> 2. What is the label selector chain from Service to pod?
>    (Answer: Service.spec.selector → pod.metadata.labels. The Service looks for pods whose
>    labels contain ALL key-value pairs in the selector. Pods with extra labels still match.
>    Pods missing even one required label do not match.)
> 3. If you add a new pod manually (not via the Deployment) with the correct labels, does it
>    automatically become an endpoint?
>    (Answer: Yes. The Endpoints controller watches for pods matching a service's selector and
>    automatically adds or removes them from the Endpoints object. Label selectors are
>    evaluated continuously, not just at creation time.)

> **Hint:** Endpoints are automatically managed by the Kubernetes Endpoints controller.
> You should never need to create Endpoint objects manually. If Endpoints shows <none>,
> the answer is always: check the service selector vs the pod labels. They must match exactly.

---

## Exercise 2.4 — NodePort and LoadBalancer Exposure

**What you'll learn:** How NodePort exposes a service outside the cluster, the NodePort range
limitation, and how to use minikube tunnel to simulate a LoadBalancer service.

**Instructions:**

1. Create a NodePort service:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: day2
spec:
  selector:
    app: nginx-cluster
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080    # must be in range 30000-32767
EOF

kubectl get service nginx-nodeport
# TYPE column shows NodePort, PORT(S) shows 80:30080/TCP
```

2. Access it via the minikube node IP:
```bash
MINIKUBE_IP=$(minikube ip)
echo "Accessing: http://$MINIKUBE_IP:30080"
curl http://$MINIKUBE_IP:30080
# Output: Welcome to nginx!
```

3. Understand the NodePort range:
```bash
# NodePort range: 30000-32767 (only 2768 ports available cluster-wide)
# This is hardcoded in the kube-apiserver as --service-node-port-range
# Reason: ports below 30000 are in the range of common services, protecting the host

kubectl explain service.spec.ports.nodePort
```

4. Create a LoadBalancer service (minikube has no cloud LB, so we use tunnel):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  namespace: day2
spec:
  selector:
    app: nginx-cluster
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl get service nginx-lb
# EXTERNAL-IP shows <pending> — no cloud LB controller exists
```

5. In a SEPARATE terminal, run minikube tunnel (it needs to stay running):
```bash
# Run this in a new terminal window/tab:
minikube tunnel
# Enter your sudo password if prompted — it needs to create a route
```

6. Back in your original terminal:
```bash
kubectl get service nginx-lb
# EXTERNAL-IP now shows an IP (typically 127.0.0.1 or a minikube IP)

LB_IP=$(kubectl get service nginx-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$LB_IP
```

7. Clean up:
```bash
kubectl delete service nginx-nodeport nginx-lb
# Stop the tunnel in the other terminal with Ctrl+C
```

**Questions to answer:**
> 1. What is the NodePort range and why is it restricted?
>    (Answer: 30000-32767. Ports below 30000 include well-known service ports (80, 443, 22,
>    etc.) that may be in use on the host node. The range protects node services from conflict.)
> 2. Why shouldn't you use NodePort in production?
>    (Answer: NodePort exposes a port on EVERY node in the cluster, even nodes not running
>    the pod. There is no health-aware routing at the node level. You also quickly exhaust
>    the 2768-port range in large clusters. LoadBalancer with a cloud LB controller or an
>    Ingress controller is the production pattern.)
> 3. What does `minikube tunnel` simulate?
>    (Answer: It simulates a cloud LoadBalancer controller. In production (GKE, EKS, AKS),
>    a controller watches for LoadBalancer services and provisions an actual cloud load
>    balancer, then updates the service's EXTERNAL-IP. minikube tunnel creates a local
>    route to serve the same function without a cloud provider.)

---

## Exercise 2.5 — Network Policies: Isolate Services

**What you'll learn:** How NetworkPolicy enforces traffic rules at the pod level, what the
default behavior is without any policy, and how to design a three-tier isolation strategy.

**Instructions:**

1. Set up three applications in separate namespaces to simulate a 3-tier app:
```bash
kubectl create namespace netpol-demo
kubectl config set-context --current --namespace=netpol-demo

# Frontend
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: netpol-demo
spec:
  replicas: 1
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
        image: curlimages/curl
        command: ["sh", "-c", "while true; do sleep 30; done"]
EOF

# Backend API
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: netpol-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# Database (simplified — just an nginx to represent a port)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: netpol-demo
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
      - name: database
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# Create services for backend and database:
kubectl expose deployment backend --port=80 --name=backend-svc -n netpol-demo
kubectl expose deployment database --port=80 --name=database-svc -n netpol-demo

kubectl get pods -n netpol-demo
```

2. Without any NetworkPolicy — verify ALL traffic is allowed:
```bash
FRONTEND_POD=$(kubectl get pod -n netpol-demo -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Frontend can reach backend:
kubectl exec -n netpol-demo $FRONTEND_POD -- curl -s --max-time 3 backend-svc
# Should respond with nginx page

# Frontend can reach database DIRECTLY (this is the problem):
kubectl exec -n netpol-demo $FRONTEND_POD -- curl -s --max-time 3 database-svc
# Should also respond — no isolation yet
```

3. Apply a NetworkPolicy that:
   - Allows frontend → backend
   - Allows backend → database
   - BLOCKS frontend → database
```bash
# First, deny all ingress to the database by default:
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-database
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress: []
  # Empty ingress list = deny ALL ingress to database pods
EOF

# Allow ONLY the backend to reach the database:
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
EOF
```

4. Test the isolation:
```bash
FRONTEND_POD=$(kubectl get pod -n netpol-demo -l app=frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(kubectl get pod -n netpol-demo -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Frontend → backend: should still work (no policy on backend yet)
kubectl exec -n netpol-demo $FRONTEND_POD -- curl -s --max-time 3 backend-svc
echo "Frontend→Backend exit code: $?"

# Frontend → database: should FAIL (policy blocks it)
kubectl exec -n netpol-demo $FRONTEND_POD -- curl -s --max-time 3 database-svc
echo "Frontend→Database exit code: $?"    # Should be non-zero (connection timeout)

# Backend → database: should work (policy allows it)
kubectl exec -n netpol-demo $BACKEND_POD -- curl -s --max-time 3 database-svc
echo "Backend→Database exit code: $?"
```

5. Examine the applied policies:
```bash
kubectl get networkpolicies -n netpol-demo
kubectl describe networkpolicy deny-all-database -n netpol-demo
kubectl describe networkpolicy allow-backend-to-database -n netpol-demo
```

6. Clean up:
```bash
kubectl delete namespace netpol-demo
kubectl config set-context --current --namespace=day2
```

**Questions to answer:**
> 1. What is the default network behavior in Kubernetes WITHOUT any NetworkPolicy?
>    (Answer: All traffic is allowed. Every pod can reach every other pod in the cluster,
>    regardless of namespace. This is the "allow all" default.)
> 2. What changes the moment you apply ANY NetworkPolicy to a pod?
>    (Answer: Only traffic matching that policy is allowed. Everything else is denied.
>    NetworkPolicy is additive for allow rules — each policy adds more allowed traffic.
>    But the presence of a policy changes the default from "allow all" to "deny all except
>    what is explicitly permitted.")
> 3. What does NetworkPolicy actually enforce at the kernel level?
>    (Answer: NetworkPolicy is enforced by the CNI plugin, not by Kubernetes core. The CNI
>    plugin — such as Calico, Cilium, or Weave — programs kernel-level rules (iptables or
>    eBPF) on each node to enforce the policies. If your CNI plugin does not support
>    NetworkPolicy, the policies are stored in etcd but silently not enforced. minikube's
>    default kindnet CNI does NOT enforce NetworkPolicy — use Calico or Cilium for real
>    enforcement.)

> **Hint:** To test whether your CNI actually enforces NetworkPolicy, after applying a deny
> policy, run a curl that should be blocked. If it still succeeds, your CNI plugin does not
> support NetworkPolicy. For minikube with real enforcement:
> `minikube start --cni=calico`

---

## Day 2 Checkpoint

Before moving to Day 3, answer these without looking at the material:

1. What is kube-proxy and what does it actually modify on the node?
2. What is the ClusterIP? Can you ping a ClusterIP?
3. What does `Endpoints: <none>` mean in `kubectl describe service`?
4. What is the full FQDN of a service named `api` in namespace `production`?
5. What component resolves Kubernetes service names in DNS?
6. What is the default network behavior in Kubernetes without any NetworkPolicy?
7. What does applying a NetworkPolicy change about the default traffic behavior for the
   selected pods?

---

## Day 2 Summary

| Concept | What You Proved |
|---|---|
| ClusterIP is virtual | iptables KUBE-SVC rules route traffic; ClusterIP has no real interface |
| kube-proxy writes iptables | Saw KUBE-SEP chains on the minikube node |
| CoreDNS resolves names | Short names work via /etc/resolv.conf search domains |
| Label selector mismatch | Endpoints: none → selector mismatch was the cause |
| NodePort exposes externally | Accessed service via minikube IP on nodePort |
| LoadBalancer needs controller | minikube tunnel simulates cloud LB |
| NetworkPolicy is additive | No policy = allow all; first policy = deny all except explicit allows |
| CNI enforces NetworkPolicy | Kubernetes stores the policy; the CNI plugin enforces it |
