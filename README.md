# Project 14 — Kubernetes NetworkPolicy

## Objective

Learn how to control communication between Kubernetes Pods using a **NetworkPolicy**.

We tested:

```text
Without NetworkPolicy:

Any Pod
   ↓
Backend Service
   ↓
Backend Pod
   ✅ Communication allowed


With NetworkPolicy:

Frontend Pod
   ↓
Backend Service
   ↓
Backend Pod
   ✅ Allowed

Other Pods
   ↓
Backend Service
   ↓
Backend Pod
   ❌ Blocked
```

---

# 1. Create Namespace

```bash
kubectl create namespace project-14
```

Set it as the current namespace:

```bash
kubectl config set-context --current --namespace=project-14
```

> **Note:** In the YAML used during this project, `namespace: default` was explicitly specified. Therefore, those resources were actually created in the `default` namespace, despite creating `project-14`.

---

# 2. EKS NetworkPolicy Prerequisite

A Kubernetes `NetworkPolicy` defines network rules, but the cluster networking layer must support and enforce those rules.

Check the VPC CNI add-on:

```bash
aws eks describe-addon \
  --cluster-name my1 \
  --addon-name vpc-cni \
  --region ap-south-2
```

Check the NetworkPolicy configuration:

```bash
aws eks describe-addon \
  --cluster-name my1 \
  --addon-name vpc-cni \
  --region ap-south-2 \
  --query "addon.configurationValues" \
  --output text
```

Initially:

```text
None
```

Enable NetworkPolicy enforcement:

```bash
aws eks update-addon \
  --cluster-name my1 \
  --addon-name vpc-cni \
  --region ap-south-2 \
  --resolve-conflicts PRESERVE \
  --configuration-values '{"enableNetworkPolicy":"true"}'
```

Verify:

```bash
aws eks describe-addon \
  --cluster-name my1 \
  --addon-name vpc-cni \
  --region ap-south-2 \
  --query "addon.configurationValues" \
  --output text
```

Expected:

```text
{"enableNetworkPolicy":"true"}
```

This is a **cluster-level configuration** and does not need to be repeated for every NetworkPolicy.

---

# 3. Project Files

```text
project-14/
└── k8s/
    ├── frontend.yaml
    ├── backend.yaml
    ├── service.yaml
    └── network-policy.yaml
```

---

# 4. Frontend Pod

## `frontend.yaml`

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: frontend
  namespace: default
  labels:
    app: frontend

spec:
  containers:
    - name: frontend
      image: busybox
      command:
        - sleep
        - "3600"
```

The important label is:

```yaml
labels:
  app: frontend
```

Later, the NetworkPolicy uses this label to identify the Pod allowed to communicate with the backend.

---

# 5. Backend Pod

## `backend.yaml`

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: backend
  namespace: default
  labels:
    app: backend

spec:
  containers:
    - name: backend
      image: nginx:alpine
      ports:
        - containerPort: 80
```

The backend Pod runs Nginx on:

```text
TCP Port 80
```

Its label is:

```yaml
labels:
  app: backend
```

The NetworkPolicy uses this label to select the Pod it should protect.

---

# 6. Backend Service

## `service.yaml`

```yaml
apiVersion: v1
kind: Service

metadata:
  name: backend-service
  namespace: default

spec:
  selector:
    app: backend

  ports:
    - port: 80
      targetPort: 80
```

The Service selects the backend Pod:

```text
Service selector:

app=backend
      ↓
Matches
      ↓
Backend Pod

app=backend
```

The communication flow is:

```text
Frontend Pod
      ↓
backend-service
      ↓
Backend Pod
```

The Service provides a **stable DNS name** instead of requiring us to use the changing Pod IP address.

---

# 7. NetworkPolicy

## `network-policy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-frontend-to-backend
  namespace: default

spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend

      ports:
        - protocol: TCP
          port: 80
```

---

# 8. Understanding the NetworkPolicy

## Which Pod does the policy protect?

```yaml
podSelector:
  matchLabels:
    app: backend
```

This means:

```text
Apply this NetworkPolicy to:

app=backend
```

Therefore, the **backend Pod is protected**.

---

## Which Pod is allowed?

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

This means:

```text
Allow traffic only from Pods with:

app=frontend
```

---

## Which port is allowed?

```yaml
ports:
  - protocol: TCP
    port: 80
```

Only TCP traffic to port `80` is allowed.

---

# 9. Apply the Resources

Apply everything:

```bash
kubectl apply -f k8s/
```

Check the Pods:

```bash
kubectl get pods
```

Check the Service:

```bash
kubectl get svc
```

Check the NetworkPolicy:

```bash
kubectl get networkpolicy
```

---

# 10. Test Allowed Traffic

Test communication from the frontend Pod:

```bash
kubectl exec -it frontend -- wget -qO- --timeout=5 http://backend-service
```

The Nginx response was returned successfully.

This proves:

```text
Frontend Pod
app=frontend
      ↓
Matches NetworkPolicy
      ↓
Backend Service
      ↓
Backend Pod
      ↓
Port 80
      ✅ Allowed
```

---

# 11. Test Blocked Traffic

We changed the NetworkPolicy from:

```yaml
app: frontend
```

to a label that does not exist:

```yaml
app: allowed-pod
```

Then applied the updated policy:

```bash
kubectl apply -f k8s/network-policy.yaml
```

Test again:

```bash
kubectl exec -it frontend -- sh
```

Inside the Pod:

```bash
wget -qO- --timeout=5 http://backend-service
```

Result:

```text
wget: download timed out
```

Why?

```text
Frontend Pod:

app=frontend
```

But the NetworkPolicy allowed:

```text
app=allowed-pod
```

Therefore:

```text
app=frontend
       ↓
Does not match allowed selector
       ↓
Backend Pod
       ❌ Blocked
```

---

# 12. Restore Allowed Traffic

Change the policy back:

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

Apply it:

```bash
kubectl apply -f k8s/network-policy.yaml
```

Test again:

```bash
kubectl exec -it frontend -- wget -qO- --timeout=5 http://backend-service
```

The connection worked again.

---

# Key Understanding

## Without NetworkPolicy

```text
Frontend Pod ──→ Backend Pod ✅
Other Pod    ──→ Backend Pod ✅
Random Pod   ──→ Backend Pod ✅
```

## With NetworkPolicy

```text
Frontend Pod ──→ Backend Pod ✅
Other Pod    ──→ Backend Pod ❌
Random Pod   ──→ Backend Pod ❌
```

Only traffic matching the NetworkPolicy rules is allowed.

---

# Service vs NetworkPolicy

```text
Service
=
Provides stable networking and DNS access to Pods.
```

```text
NetworkPolicy
=
Controls which Pods are allowed to communicate with other Pods.
```

Together:

```text
Frontend Pod
      ↓
NetworkPolicy checks permission
      ↓
Backend Service
      ↓
Backend Pod
```

# Project 14 — Completed ✅
