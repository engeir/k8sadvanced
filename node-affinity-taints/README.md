# 🧭 Kubernetes Lab: Node Affinity, Taints & Tolerations (with Minikube)

This lab demonstrates how to control where pods run using **Node Affinity** and how to prevent or allow pods to run on specific nodes using **Taints & Tolerations**. You'll run the lab on Minikube (multi-node) and experiment with required vs preferred affinity and NoSchedule taints.

---

## 🎯 Learning objectives
- Understand the difference between **nodeSelector**, **required** and **preferred** Node Affinity.
- Learn how **taints** can repel pods and **tolerations** allow pods to tolerate taints.
- Observe scheduling behavior and debug pending pods using `kubectl describe` and events.

---

## 📋 Prerequisites
- **Minikube** installed and running (we'll use 2 nodes in the examples).
- **kubectl** installed and set to the `minikube` context.
- A terminal (PowerShell or Bash).

---

> Note: Minikube defaults to a single node. For this lab we recommend starting Minikube with two nodes so you can see scheduling behavior clearly.

### Start Minikube with 2 nodes (if needed)

On a fresh environment you can run:

```
minikube delete --all
minikube start --nodes=2
```

If you already have Minikube running you can add a node:

```
minikube node add
kubectl get nodes -o wide
```

---

### Step 1: Create a namespace for the lab

```
kubectl create namespace lab-affinity
kubectl get ns lab-affinity
```

### Step 2: Label one node (target node)

Pick one node name from `kubectl get nodes` and label it. We'll use `disktype=ssd` as an example label.

```
NODE=$(kubectl get nodes -o name | sed -n '1p' | cut -d/ -f2)
# or on PowerShell: $NODE = (kubectl get nodes -o name)[0] -replace 'node/', ''
kubectl label node $NODE disktype=ssd
kubectl get node $NODE --show-labels
```

> You should now have a node labeled `disktype=ssd`.

---

### Step 3: Required node affinity (hard requirement)

Apply `required-affinity-deployment.yaml` to create a Deployment that only schedules on nodes labeled `disktype=ssd`.

```
kubectl apply -n lab-affinity -f required-affinity-deployment.yaml
kubectl get pods -n lab-affinity -o wide
kubectl describe pod -n lab-affinity -l app=nginx-req
```

If there's a node with the label the pod will be scheduled there. If not, the pod will remain `Pending`.

---

### Step 4: Preferred node affinity (soft preference)

Apply `preferred-affinity-deployment.yaml`. This deployment prefers nodes with `disktype=ssd` but may schedule elsewhere if none are available.

```
kubectl apply -n lab-affinity -f preferred-affinity-deployment.yaml
kubectl get pods -n lab-affinity -o wide
kubectl describe pod -n lab-affinity -l app=nginx-pref
```

---

### Step 5: Taints and Tolerations

Pick the other node and taint it so pods without the matching toleration cannot be scheduled there.

```
# get second node name
NODE2=$(kubectl get nodes -o name | sed -n '2p' | cut -d/ -f2)
# PowerShell analog available above
kubectl taint nodes $NODE2 key1=value1:NoSchedule
kubectl describe node $NODE2 | sed -n '/Taints/,$p'
```

Apply a pod that does NOT include tolerations (`taint-pod-no-toleration.yaml`) and one that DOES include the appropriate toleration (`taint-pod-with-toleration.yaml`):

```
kubectl apply -n lab-affinity -f taint-pod-no-toleration.yaml
kubectl apply -n lab-affinity -f taint-pod-with-toleration.yaml
kubectl get pods -n lab-affinity -o wide
kubectl describe pod -n lab-affinity taint-demo-no-tol
kubectl describe pod -n lab-affinity taint-demo-with-tol
```

Observe that the pod without toleration will not be scheduled on the tainted node (it might schedule on the other node if available), while the one with toleration may run on the tainted node.

---

### Step 6: Forcing a pod to need the tainted node

To see a pod remain `Pending` because the only node that matches its affinity is tainted, combine a required nodeAffinity to the tainted node: label that node (for example, `purpose=tainted`) and use a deployment that requires `purpose=tainted` without tolerations. It will remain Pending because of NoSchedule.

```
# label the node and taint it
kubectl label node <tainted-node> purpose=tainted
kubectl taint node <tainted-node> key1=value1:NoSchedule

# apply the deployment that requires the tainted node
kubectl apply -n lab-affinity -f required-affinity-tainted-node.yaml
kubectl get pods -n lab-affinity -o wide
kubectl describe pod -n lab-affinity -l app=nginx-req-tainted
```

---

### Cleanup

```
kubectl delete ns lab-affinity
kubectl label node --all disktype- || true
kubectl taint nodes --all key1:NoSchedule- || true
```

---

## 🛠️ Troubleshooting & Tips
- Use `kubectl describe pod <pod>` and `kubectl get events -n lab-affinity --sort-by=.lastTimestamp` to see scheduling failures and reasons.
- `requiredDuringSchedulingIgnoredDuringExecution` is a hard rule at scheduling time.
- `preferredDuringSchedulingIgnoredDuringExecution` is a best effort.
- Taints use three effects: `NoSchedule`, `PreferNoSchedule`, and `NoExecute`.

---

## Files in this folder
- `required-affinity-deployment.yaml` — Deployment with a `required` nodeAffinity
- `preferred-affinity-deployment.yaml` — Deployment with a `preferred` nodeAffinity
- `taint-pod-no-toleration.yaml` — Pod to show behavior without toleration
- `taint-pod-with-toleration.yaml` — Pod with toleration to run on tainted node
- `required-affinity-tainted-node.yaml` — Deployment that requires a label on a tainted node (shows Pending)

---

Good luck! 🚀