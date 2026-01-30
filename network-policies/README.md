# 🧪 Kubernetes Lab: NetworkPolicies with Minikube

This lab teaches you how to control Pod-to-Pod communication using NetworkPolicies.
You’ll deploy sample apps, apply policies, and observe how traffic is allowed or
blocked.

---

## 🎯 Learning Objectives

Understand the default allow model in Kubernetes networking.

Apply a default deny policy to isolate Pods.

Create fine-grained policies to allow only specific traffic.

Test connectivity using kubectl exec and curl.

---

## 📋 Prerequisites

Minikube running with a CNI plugin that supports NetworkPolicies (e.g., --cni=calico).

```
minikube start --cni=calico
```

kubectl installed and configured.

---

### Step 1: Create a Namespace

```
kubectl create namespace lab-netpol
```

### Step 2: Deploy Sample Apps

We’ll deploy a backend (nginx) and two clients (frontend and test).

```
# save as apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab-netpol
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
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: lab-netpol
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: lab-netpol
  labels:
    role: frontend
spec:
  containers:
  - name: curl
    image: curlimages/curl:8.5.0
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: test
  namespace: lab-netpol
  labels:
    role: test
spec:
  containers:
  - name: curl
    image: curlimages/curl:8.5.0
    command: ["sleep", "3600"]
```

Apply:

```
kubectl apply -f apps.yaml
kubectl get pods -n lab-netpol
```

### Step 3: Verify Default Open Connectivity

```
kubectl exec -n lab-netpol frontend -- curl -s backend
kubectl exec -n lab-netpol test -- curl -s backend
```

✅ Both should succeed — by default, all Pods can talk to each other.

### Step 4: Apply a Default Deny Policy

```
# save as default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: lab-netpol
spec:
  podSelector: {}   # applies to all Pods
  policyTypes:
  - Ingress
```

Apply:

```
kubectl apply -f default-deny.yaml
```

Test again:

```
kubectl exec -n lab-netpol frontend -- curl -s --max-time 3 backend
kubectl exec -n lab-netpol test -- curl -s --max-time 3 backend
```

❌ Both should now fail — ingress is blocked.

### Allow Only Frontend → Backend

```
# save as allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: lab-netpol
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
          role: frontend
    ports:
    - protocol: TCP
      port: 80
```

Apply:

```
kubectl apply -f allow-frontend.yaml

```

Test:

```
kubectl exec -n lab-netpol frontend -- curl -s backend   # ✅ should work
kubectl exec -n lab-netpol test -- curl -s backend       # ❌ should fail
```

### Step 6: (Optional) Add Egress Control

Block all egress except DNS (UDP 53):

```
# save as egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: lab-netpol
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

Apply and test:

```
kubectl exec -n lab-netpol frontend -- curl -s https://example.com   # ❌ blocked
```

### Step 7: Cleanup

```
kubectl delete ns lab-netpol
```

### 🛠️ Troubleshooting Tips

Use `kubectl describe netpol -n lab-netpol` to see applied policies.

Remember: Policies are additive — multiple policies can apply to the same Pod.

If traffic is blocked unexpectedly, check both ingress and egress rules.
