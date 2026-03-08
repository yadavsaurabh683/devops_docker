# Day 3: Configuration, Storage, and Access Control
## Theme: ConfigMaps, Secrets, PersistentVolumes, RBAC, and Namespaces

**Goal:** By end of Day 3, you will understand how to inject configuration into pods safely,
how Kubernetes storage works from PV to pod, how RBAC controls what workloads can do, and how
namespaces serve as isolation and quota boundaries. You will also understand the most common
security mistake engineers make with Secrets.

**Estimated time:** 5–6 hours

---

## Background: Configuration Is Not Just Environment Variables

Most engineers inject configuration through environment variables and move on. This works —
until it doesn't. Volume-mounted ConfigMaps update live. Env-var ConfigMaps require a pod
restart. Secrets are base64, not encrypted. PVCs have lifecycle semantics that determine
whether data survives a pod failure. RBAC silently denies requests with no error message
unless you know where to look.

Every concept in Day 3 has a "gotcha" that engineers discover the hard way in production.
Today you discover them the easy way, in minikube.

---

## Exercise 3.1 — ConfigMaps: Two Ways to Inject Config (and Their Gotchas)

**What you'll learn:** The two methods for using ConfigMaps (env vars and volume mounts),
and the critical difference in how updates propagate to running pods.

**Instructions:**

1. Create a namespace for today's work:
```bash
kubectl create namespace day3
kubectl config set-context --current --namespace=day3
```

2. Create a ConfigMap with app configuration:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: day3
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  APP_MODE: "production"
  config.properties: |
    log.level=info
    max.connections=100
    app.mode=production
EOF
```

3. Deploy a pod that uses the ConfigMap BOTH ways simultaneously:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
  namespace: day3
spec:
  containers:
  - name: app
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "=== Environment Variables ==="
      echo "LOG_LEVEL=$LOG_LEVEL"
      echo "MAX_CONNECTIONS=$MAX_CONNECTIONS"
      echo ""
      echo "=== Volume Mount ==="
      cat /config/config.properties
      echo ""
      echo "Watching for changes (check every 10s)..."
      while true; do
        echo "--- $(date) ---"
        echo "ENV: LOG_LEVEL=$LOG_LEVEL"
        echo "FILE: $(cat /config/config.properties | grep log.level)"
        sleep 10
      done
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: MAX_CONNECTIONS
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: MAX_CONNECTIONS
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
EOF

kubectl get pods config-demo
```

4. Observe the initial state:
```bash
kubectl logs config-demo --tail=5
```

5. Update the ConfigMap and observe the difference in reload behavior:
```bash
kubectl patch configmap app-config \
  -p '{"data":{"LOG_LEVEL":"debug","config.properties":"log.level=debug\nmax.connections=200\napp.mode=production\n"}}'

echo "ConfigMap updated. Waiting 70 seconds for kubelet to sync volume mounts..."
```

6. After ~70 seconds, check the pod logs:
```bash
kubectl logs config-demo --tail=20

# You will see:
# - ENV: LOG_LEVEL=info   ← STILL OLD VALUE (env vars don't update without pod restart)
# - FILE: log.level=debug  ← UPDATED (volume mount synced by kubelet ~60s)
```

7. Confirm the env var did not update by checking it directly:
```bash
kubectl exec config-demo -- sh -c 'echo $LOG_LEVEL'
# Output: info   ← Not updated, despite the ConfigMap change
```

8. Clean up:
```bash
kubectl delete pod config-demo
```

**Questions to answer:**
> 1. Why do env-var-based ConfigMaps require a pod restart but volume mounts do not?
>    (Answer: Environment variables are set at process start time and do not change for the
>    lifetime of the process — this is a Linux process fundamental, not a Kubernetes behavior.
>    Volume mounts are files on a filesystem; kubelet syncs ConfigMap data to the mounted
>    files every ~60 seconds, so the process can re-read the file at any time.)
> 2. What is the kubelet sync interval for ConfigMap volume mounts?
>    (Answer: Controlled by `--sync-frequency`, default is 1 minute (60s). The actual
>    propagation delay is between 0 and 2x the sync interval — up to 2 minutes in the worst
>    case.)
> 3. What happens if you reference a ConfigMap key that does not exist in the ConfigMap?
>    (Answer: The pod fails to start with status `CreateContainerConfigError`. The error
>    appears in `kubectl describe pod` Events: "couldn't find key <key> in ConfigMap".)

> **Hint:** For apps that support it, the volume-mount pattern is always preferred because
> it supports live reload without pod restarts. Design your application to re-read its config
> file periodically or on SIGHUP rather than reading once at startup.

---

## Exercise 3.2 — Secrets: What Every Engineer Gets Wrong

**What you'll learn:** Why Kubernetes Secrets are NOT encrypted by default, how to audit
exposed secrets, and what real secret security in Kubernetes looks like.

**Instructions:**

1. Create a Secret with database credentials:
```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_PASSWORD='super-secret-password-123' \
  --from-literal=DB_USER='admin' \
  -n day3
```

2. Retrieve the secret and observe that it's base64, NOT encrypted:
```bash
kubectl get secret db-credentials -o yaml
# You will see:
# data:
#   DB_PASSWORD: c3VwZXItc2VjcmV0LXBhc3N3b3JkLTEyMw==
#   DB_USER: YWRtaW4=
```

3. Decode it trivially — anyone with kubectl read access to the namespace can do this:
```bash
kubectl get secret db-credentials \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# Output: super-secret-password-123

kubectl get secret db-credentials \
  -o jsonpath='{.data.DB_USER}' | base64 -d
# Output: admin
```

4. Show what happens when a developer accidentally puts secrets in a ConfigMap:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: accidental-secret-cm
  namespace: day3
data:
  DB_PASSWORD: "super-secret-password-123"
  DB_USER: "admin"
  API_KEY: "sk-prod-key-abc123"
EOF

# ConfigMap data is completely plaintext:
kubectl get configmap accidental-secret-cm -o yaml
# No encoding at all. Anyone with read access sees everything immediately.
```

5. Deploy a pod that uses the secret correctly (as an environment variable):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
  namespace: day3
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "echo DB_USER=$DB_USER; echo DB_PASS length=$(echo $DB_PASSWORD | wc -c); sleep 3600"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_PASSWORD
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
EOF

kubectl exec secret-demo -- sh -c 'echo $DB_USER; echo $DB_PASSWORD'
# The secret is visible to the running process — this is unavoidable for env var injection
```

6. Understand the actual security model:
```bash
# Secrets vs ConfigMaps: the Kubernetes API enforces different access controls
# kubectl get configmaps  (usually broad access in most clusters)
# kubectl get secrets     (should be restricted in production via RBAC)

# Without RBAC restricting it, your "secret" is as accessible as a ConfigMap to anyone
# with kubectl access. Real security comes from:
# 1. RBAC: restrict who can 'get' or 'list' secrets in each namespace
# 2. etcd encryption at rest: kube-apiserver --encryption-provider-config
# 3. External secret stores: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager
#    combined with tools like External Secrets Operator

# Check whether etcd encryption is configured in this cluster:
kubectl get apiserver -o yaml 2>/dev/null || echo "Cannot query encryption config from kubectl"
# In minikube: etcd is NOT encrypted at rest by default
```

7. Describe what Sealed Secrets solves (conceptual):
```bash
# Problem: Secrets cannot be safely committed to git (even base64 is trivially reversible)
# Solution: Sealed Secrets (bitnami/sealed-secrets)
#   1. A controller runs in the cluster with a private key
#   2. You encrypt your secret with the controller's PUBLIC key → produces a SealedSecret
#   3. SealedSecret can be safely committed to git — it cannot be decrypted outside the cluster
#   4. The controller decrypts it and creates the actual Secret inside the cluster
#   This is the correct way to do GitOps with secrets.

# The alternative: External Secrets Operator pulls secrets from Vault/AWS at runtime
# The secret NEVER touches git
echo "Sealed Secrets: safe for git. External Secrets: fetched at runtime from Vault/cloud."
```

8. Clean up:
```bash
kubectl delete pod secret-demo
kubectl delete configmap accidental-secret-cm
```

**Questions to answer:**
> 1. If Secrets are just base64, where is the actual security?
>    (Answer: The security is in RBAC — restricting who can read secrets via the Kubernetes
>    API. In a well-configured cluster, only the specific pods that need a secret can mount
>    it, and only specific service accounts can read it via the API. The "Secret" type signals
>    intent and enables different RBAC policies — the encoding itself provides no security.)
> 2. What is etcd encryption at rest?
>    (Answer: An optional kube-apiserver feature that encrypts Secret data before writing it
>    to etcd. Without it, anyone with direct etcd access reads Secrets in plaintext. With it,
>    the data is encrypted using a key — typically AES-CBC or AES-GCM — before storage.)
> 3. What is the most common secret mistake in Kubernetes environments?
>    (Answer: Committing Secrets (even encoded) to git repositories. Also: putting secrets in
>    ConfigMaps. Also: using overly broad RBAC that allows any pod to read all secrets in a
>    namespace.)

---

## Exercise 3.3 — PersistentVolumes: The Full Lifecycle

**What you'll learn:** The PV → PVC → Pod binding chain, what happens to data when a pod
dies vs when a PVC is deleted, and how StorageClass drives dynamic provisioning.

**Instructions:**

1. Create a PersistentVolumeClaim (dynamic provisioning via StorageClass):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: day3
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
  storageClassName: standard
EOF

kubectl get pvc data-pvc
# Status shows Pending initially, then Bound once the PV is provisioned

kubectl get pv
# A PersistentVolume was automatically created by minikube's dynamic provisioner
```

2. Deploy a pod that writes data to the PVC:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: writer-pod
  namespace: day3
spec:
  containers:
  - name: writer
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "Writing data at $(date)" >> /data/log.txt
      echo "Pod name: writer-pod" >> /data/log.txt
      echo "Written successfully."
      cat /data/log.txt
      sleep 3600
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: data-pvc
EOF

kubectl get pods writer-pod -w
# Wait for Running
```

3. Verify the data was written:
```bash
kubectl exec writer-pod -- cat /data/log.txt
```

4. Delete the pod — the PVC and PV survive:
```bash
kubectl delete pod writer-pod
kubectl get pvc data-pvc
# Status: Bound  ← PVC survives pod deletion
kubectl get pv
# PV still exists and is Bound to the PVC
```

5. Recreate the pod and verify data persists:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
  namespace: day3
spec:
  containers:
  - name: reader
    image: alpine
    command:
    - sh
    - -c
    - |
      echo "=== Data from previous pod ==="
      cat /data/log.txt
      echo "New entry: $(date)" >> /data/log.txt
      echo "=== After appending ==="
      cat /data/log.txt
      sleep 3600
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: data-pvc
EOF

kubectl logs reader-pod
# You see: data from the PREVIOUS writer-pod — it persisted through pod deletion
```

6. Examine StorageClass and reclaim policies:
```bash
kubectl get storageclass
# minikube has a 'standard' StorageClass with provisioner: k8s.io/minikube-hostpath

kubectl describe storageclass standard
# Look for: ReclaimPolicy: Delete
# This means: when the PVC is deleted, the PV is automatically deleted too
```

7. Understand Retain vs Delete reclaim policy:
```bash
# Create a PVC with a Retain policy (requires creating a custom PV or StorageClass):
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: retain-pv
spec:
  capacity:
    storage: 50Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/retain-test
  storageClassName: ""
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: retain-pvc
  namespace: day3
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
  storageClassName: ""
  volumeName: retain-pv
EOF

kubectl get pvc retain-pvc
kubectl get pv retain-pv

# Now delete the PVC:
kubectl delete pvc retain-pvc

# With Retain policy, the PV is NOT deleted:
kubectl get pv retain-pv
# Status: Released  ← not Bound anymore, but NOT deleted
# The data remains. An admin must manually delete the PV when they confirm it's safe.
```

8. Clean up:
```bash
kubectl delete pod reader-pod
kubectl delete pvc data-pvc retain-pvc
kubectl delete pv retain-pv
```

**Questions to answer:**
> 1. What is the difference between a PersistentVolume and a PersistentVolumeClaim?
>    (Answer: PV is the actual storage resource — created by an admin or dynamic provisioner.
>    PVC is a request for storage by a workload — claims a PV that matches its size and
>    access mode requirements. The binding is 1:1: one PVC binds to one PV.)
> 2. What happens to data when a pod dies?
>    (Answer: Nothing. The PVC and PV remain. When a new pod mounts the same PVC, the data
>    is available. Pod death does not affect PVC lifecycle.)
> 3. What happens to data when the PVC is deleted with a Delete reclaim policy?
>    (Answer: The PV is automatically deleted. The underlying storage resource is released.
>    Data is lost. With a Retain policy, the PV stays in Released state — data is preserved
>    but the PV must be manually reclaimed before another PVC can bind to it.)

> **Hint:** `ReadWriteOnce` means the volume can be mounted read-write by a single node at a
> time. `ReadWriteMany` allows multiple nodes to mount it simultaneously (requires a
> shared filesystem like NFS or cloud NFS). For most single-node deployments and databases,
> ReadWriteOnce is correct.

---

## Exercise 3.4 — RBAC: Least Privilege in Practice

**What you'll learn:** How to create a ServiceAccount with a minimal Role, bind it with a
RoleBinding, and prove what it can and cannot do by running actual API calls inside a pod.

**Instructions:**

1. Create a ServiceAccount for a limited monitoring app:
```bash
kubectl create serviceaccount pod-reader -n day3
```

2. Create a Role that only allows listing and getting pods in the day3 namespace:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-role
  namespace: day3
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

3. Bind the Role to the ServiceAccount:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: day3
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: day3
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

4. Run a pod using the restricted ServiceAccount:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: day3
spec:
  serviceAccountName: pod-reader
  containers:
  - name: kubectl-runner
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

kubectl get pods rbac-test-pod
```

5. From inside the pod, test what the ServiceAccount can do:
```bash
# Test: can it list pods? (should work)
kubectl exec -n day3 rbac-test-pod -- kubectl get pods -n day3
# Output: list of pods in day3

# Test: can it delete a pod? (should be denied)
POD_NAME=$(kubectl get pod -n day3 -l run=rbac-test-pod 2>/dev/null | tail -1 | awk '{print $1}' || echo "rbac-test-pod")
kubectl exec -n day3 rbac-test-pod -- kubectl delete pod rbac-test-pod -n day3
# Output: Error from server (Forbidden): pods "rbac-test-pod" is forbidden:
#         User "system:serviceaccount:day3:pod-reader" cannot delete resource "pods"

# Test: can it list pods in a DIFFERENT namespace? (should be denied)
kubectl exec -n day3 rbac-test-pod -- kubectl get pods -n kube-system
# Output: Error from server (Forbidden)
```

6. Understand the default ServiceAccount and why it's dangerous:
```bash
# By default, every pod that does NOT specify a serviceAccountName
# uses the 'default' ServiceAccount in its namespace.

kubectl get serviceaccount default -n day3

# The default SA typically has NO roles bound — but it still gets a token mounted.
# In many older clusters, the default SA has cluster-wide permissions inherited from
# permissive ClusterRoleBindings (often added by Helm charts or operators).

# Verify: does the default SA in day3 have any bindings?
kubectl get rolebindings -n day3 | grep default
kubectl get clusterrolebindings | grep day3

# Best practice: Create dedicated SAs for every workload.
# Add: automountServiceAccountToken: false
# to pods that don't need API access at all.
```

7. Check current permissions using kubectl auth can-i:
```bash
# Test as the pod-reader SA:
kubectl auth can-i get pods --as=system:serviceaccount:day3:pod-reader -n day3
# Output: yes

kubectl auth can-i delete pods --as=system:serviceaccount:day3:pod-reader -n day3
# Output: no

kubectl auth can-i get pods --as=system:serviceaccount:day3:pod-reader -n kube-system
# Output: no
```

8. Clean up:
```bash
kubectl delete pod rbac-test-pod
```

**Questions to answer:**
> 1. What is the difference between a Role and a ClusterRole?
>    (Answer: A Role is namespaced — it only grants permissions within one namespace.
>    A ClusterRole is cluster-wide — it can grant permissions across all namespaces, or for
>    non-namespaced resources like nodes. Use ClusterRole + ClusterRoleBinding for cluster-wide
>    permissions. Use Role + RoleBinding for namespace-scoped permissions.)
> 2. What is the default ServiceAccount and why is it dangerous?
>    (Answer: Every namespace has a 'default' ServiceAccount. Any pod that doesn't specify
>    serviceAccountName uses it automatically. If the default SA has been given broad
>    permissions — which happens with many Helm charts — every pod in that namespace
>    inherits those permissions, even pods that don't need API access.)
> 3. What happens when an RBAC check fails?
>    (Answer: The API server returns HTTP 403 Forbidden with a message: "User
>    system:serviceaccount:<namespace>:<name> cannot <verb> resource <resource>". This is
>    logged in audit logs if audit logging is enabled. The error does NOT appear in pod logs
>    unless the application explicitly logs the API response.)

---

## Exercise 3.5 — Namespaces as Isolation Boundaries

**What you'll learn:** How ResourceQuotas limit total resource consumption in a namespace,
how LimitRange sets per-pod defaults, and what namespaces do and do NOT isolate.

**Instructions:**

1. Create production and staging namespaces:
```bash
kubectl create namespace production
kubectl create namespace staging
```

2. Apply a ResourceQuota to staging to limit total resource consumption:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    pods: "5"
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "1000m"
    limits.memory: "1Gi"
EOF

kubectl describe resourcequota staging-quota -n staging
```

3. Apply a LimitRange to enforce defaults in staging (pods without requests get defaults):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limits
  namespace: staging
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "128Mi"
    defaultRequest:
      cpu: "100m"
      memory: "64Mi"
    max:
      cpu: "500m"
      memory: "256Mi"
EOF
```

4. Deploy an identical app in both namespaces:
```bash
# Production (no quota):
kubectl create deployment nginx-prod --image=nginx:alpine --replicas=2 -n production

# Staging (quota enforced):
kubectl create deployment nginx-staging --image=nginx:alpine --replicas=2 -n staging
```

5. Observe quota enforcement — try to exceed the pod limit:
```bash
kubectl scale deployment nginx-staging --replicas=6 -n staging
kubectl get pods -n staging
# Only 5 pods will be created (quota: pods: "5")

kubectl get replicaset -n staging
# The ReplicaSet shows: DESIRED=6, CURRENT=5, READY=5
# Describe the ReplicaSet to see the quota error:
RS_NAME=$(kubectl get rs -n staging -o jsonpath='{.items[0].metadata.name}')
kubectl describe rs $RS_NAME -n staging | grep -A5 "Warning"
# Event: "exceeded quota: staging-quota, requested: pods=1, used: pods=5, limited: pods=5"
```

6. Verify the namespace does NOT provide network isolation by default:
```bash
# Run a pod in production:
kubectl run prod-curl --image=curlimages/curl --restart=Never -n production \
  -- sleep 3600

# Get a staging pod IP:
STAGING_POD_IP=$(kubectl get pod -n staging -l app=nginx-staging \
  -o jsonpath='{.items[0].status.podIP}')
echo "Staging pod IP: $STAGING_POD_IP"

# Try to reach staging from production:
kubectl exec -n production prod-curl -- curl -s --max-time 3 http://$STAGING_POD_IP
# SUCCESS — namespaces do NOT provide network isolation by default
# Network isolation requires NetworkPolicy (and a CNI that enforces it)
```

7. Clean up:
```bash
kubectl delete namespace production staging
kubectl delete pod --all -n day3
kubectl config set-context --current --namespace=default
```

**Questions to answer:**
> 1. Do namespaces provide network isolation by default?
>    (Answer: No. Namespaces are a naming and policy scope boundary, not a network boundary.
>    Traffic between pods in different namespaces flows freely unless NetworkPolicy explicitly
>    blocks it.)
> 2. What is a ResourceQuota?
>    (Answer: A quota object that limits the total amount of resources that can be consumed
>    in a namespace. It limits counts (pods, services) and resource amounts (CPU, memory).
>    When a quota is exceeded, new pod creation is rejected with a quota exceeded error.)
> 3. What is the difference between ResourceQuota and LimitRange?
>    (Answer: ResourceQuota is a namespace-wide total cap. LimitRange sets per-container
>    or per-pod defaults and bounds — it ensures all containers have requests/limits and
>    prevents individual containers from consuming excessive resources. They complement
>    each other: LimitRange fills in missing requests/limits, ResourceQuota caps the total.)

---

## Day 3 Checkpoint

Before moving to Day 4, answer these without looking at the material:

1. Why do env-var ConfigMaps require a pod restart to see changes, but volume-mounted
   ConfigMaps update automatically?
2. Kubernetes Secrets are base64 encoded. Where does actual security come from?
3. What is the relationship between PV, PVC, and StorageClass?
4. What happens to data stored in a PVC when the pod using it is deleted?
5. What is the difference between a Role and a ClusterRole?
6. What does RBAC return when a permission is denied?
7. Does a namespace provide network isolation by default?

---

## Day 3 Summary

| Concept | What You Proved |
|---|---|
| ConfigMap env vs volume | Env vars froze at pod start; file updated after ~60s without restart |
| Secrets are base64 | Decoded DB_PASSWORD with a single base64 -d command |
| RBAC enforces API access | pod-reader SA could list pods but got Forbidden on delete |
| PVC survives pod death | reader-pod found writer-pod's data intact |
| Retain vs Delete reclaim | Retain kept PV in Released state; Delete destroys the PV |
| ResourceQuota caps totals | 6th pod rejected when quota of 5 pods was hit |
| Namespace ≠ network boundary | Production pod reached staging pod IP directly |
