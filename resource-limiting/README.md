# 🧪 Kubernetes Lab: Resource Limits on Namespaces (with Minikube)

This lab teaches you how to control compute and object usage at the namespace level
using **ResourceQuota** and **LimitRange**. You’ll create quotas, deploy workloads that
consume resources, and watch Kubernetes enforce limits.

---

## 🎯 Learning Objectives

- Understand what **ResourceQuota** and **LimitRange** do.
- Apply namespace-wide quotas for CPU, memory, and pod counts.
- Verify how violations are enforced and troubleshoot errors.
- Practice monitoring quota usage with `kubectl`.

---

## 📋 Prerequisites

- **Minikube** installed and running.
- **kubectl** installed and configured to use the `minikube` context.
- A terminal (Bash, PowerShell, etc.).

---

### Step 1: Create a Namespace

```
kubectl create namespace lab-quota
kubectl get ns lab-quota
```

### Step 2: Apply a ResourceQuota

Create quota.yaml:

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-and-pods
  namespace: lab-quota
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
    pods: "3"
```

Apply it:

```
kubectl apply -n lab-quota -f quota.yaml
kubectl describe resourcequota compute-and-pods -n lab-quota
```

### Step 3: Apply a LimitRange

Create limitrange.yaml:

```
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: lab-quota
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
```

Apply it:

```
kubectl apply -n lab-quota -f limitrange.yaml
kubectl describe limitrange defaults -n lab-quota
```

### Step 4: Deploy a Sample App

Create deploy.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-quota
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

Apply and inspect:

```
kubectl apply -n lab-quota -f deploy.yaml
kubectl get pods -n lab-quota
kubectl describe pod -n lab-quota -l app=web
```

Check that requests/limits are auto‑applied from the LimitRange.

### Step 5: Scale Until You Hit the Quota

```
kubectl -n lab-quota scale deployment/web --replicas=5
kubectl get pods -n lab-quota
kubectl -n lab-quota get events --sort-by=.lastTimestamp
kubectl describe resourcequota compute-and-pods -n lab-quota
```

You should see failures like:

```
Error creating: pods "web-..." is forbidden: exceeded quota: compute-and-pods
```

### Step 6: (Optional) Pod-Count-Only Quota

Create pods-quota.yaml:

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-only
  namespace: lab-quota
spec:
  hard:
    pods: "2"
```

Apply and test:

```
kubectl apply -n lab-quota -f pods-quota.yaml
kubectl -n lab-quota scale deployment/web --replicas=3
kubectl -n lab-quota get events --sort-by=.lastTimestamp
```

Step 7: Cleanup

```
kubectl delete ns lab-quota
```

## 🛠️ Troubleshooting

Check defaults applied: kubectl describe pod <pod-name> -n lab-quota

See why a pod was blocked: kubectl get events -n lab-quota --sort-by=.lastTimestamp

Quota categories:

- Compute (CPU, memory)

- Storage (PVCs, ephemeral storage)

- Objects (pods, services, configmaps, etc.)
