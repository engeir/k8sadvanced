# ⚡ KEDA Lab: Event-driven scaling with Minikube

This lab demonstrates how to use **KEDA** (Kubernetes Event-driven Autoscaling) in
Minikube to scale workloads based on external events. We'll use **Redis list length** as
a scaler example to scale a simple worker deployment.

---

## 🎯 Learning objectives

- Install KEDA on Minikube (Helm or manifest).
- Deploy a Redis queue and a worker Deployment.
- Create a `ScaledObject` to scale the worker based on Redis list length.
- Observe scaling behavior and generate load with a producer pod.

---

## 📋 Prerequisites

- **Minikube** running (single-node is fine for this lab).
- **kubectl** configured to use the `minikube` context.
- Optional: **helm** installed (makes KEDA install easier).

---

## Step 0: Install KEDA

Two ways to install KEDA. Use either Helm (recommended) or manifest apply.

Helm (recommended):

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
kubectl get pods -n keda
```

kubectl apply (manifest):

```
# Use the KEDA release manifest (check upstream for latest release)
kubectl apply -f https://github.com/kedacore/keda/releases/latest/download/keda-operator.yaml
kubectl get pods -n keda
```

Wait until the KEDA pods are running.

---

## Step 1: Deploy Redis (queue)

Apply the Redis Deployment + Service in this folder:

```
kubectl apply -f redis-deployment.yaml
kubectl get pods -l app=redis
kubectl get svc redis
```

---

## Step 2: Deploy the worker (initial replicas = 0)

The worker will `LPOP` items from Redis key `keda-queue` and print processed messages.

```
kubectl apply -f worker-deployment.yaml
kubectl get deployment redis-worker
kubectl get pods -l app=redis-worker
```

Initially replicas are `0` (no work).

---

## Step 3: Create `ScaledObject` for Redis list length

Apply the `ScaledObject`:

```
kubectl apply -f redis-scaledobject.yaml
kubectl get scaledobject
kubectl describe scaledobject redis-scaledobject
```

This ScaledObject will scale `redis-worker` between `minReplicaCount: 0` and
`maxReplicaCount: 5` when the Redis list length exceeds the threshold.

---

## Step 4: Generate load (producer)

Push messages into the Redis list to trigger scaling. Use the `producer` pod in this
folder:

```
kubectl apply -f producer-pod.yaml
kubectl logs -f producer -c producer
```

The producer will LPUSH a burst of messages to the list. Monitor worker scaling:

```
kubectl get pods -l app=redis-worker -w
kubectl get deployment redis-worker
kubectl get hpa -n default  # KEDA may create an HPA under the hood
kubectl describe scaledobject redis-scaledobject
```

You should see `redis-worker` replicas grow in response to backlog and shrink after the
queue is drained.

---

## Cleanup

```
kubectl delete -f redis-scaledobject.yaml
kubectl delete -f worker-deployment.yaml
kubectl delete -f redis-deployment.yaml
kubectl delete -f producer-pod.yaml
# uninstall KEDA if you used Helm
helm uninstall keda -n keda || true
kubectl delete ns keda || true
```

---

## Troubleshooting & tips

- Check KEDA operator logs: `kubectl -n keda logs deploy/keda-operator -c keda-operator`
- Inspect ScaledObject status and scaler metrics:
  `kubectl describe scaledobject redis-scaledobject`
- If using Minikube single node, scaling still works — replicas are regular pods on that
  node.

---

## Files in this folder

- `redis-deployment.yaml` — Redis Deployment + Service
- `worker-deployment.yaml` — Worker Deployment (replicas start at 0)
- `redis-scaledobject.yaml` — KEDA ScaledObject that monitors Redis list length
- `producer-pod.yaml` — Pod that pushes messages to the Redis list for testing

Enjoy the event-driven scaling lab! ⚡
