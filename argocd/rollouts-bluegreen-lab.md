# 🚦 Lab: Blue/Green Deployment with Argo Rollouts

## 🎯 Learning Objectives

- Understand Argo Rollouts and progressive delivery
- Deploy and manage a blue/green rollout in Kubernetes
- Observe traffic shifting and rollback using Argo Rollouts

## 🛠️ Prerequisites

- Kubernetes cluster (Minikube or other)
- kubectl installed and configured
- ArgoCD and Argo Rollouts installed
- [Argo Rollouts kubectl plugin](https://argoproj.github.io/rollouts/installation/#kubectl-plugin)

---

## Step 1: Install Argo Rollouts

Create the CRDs and controller:

```sh
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Check Rollouts controller:

```sh
kubectl get pods -n argo-rollouts
```

## Step 2: Deploy a Blue/Green Rollout

Create a file named `rollout-bluegreen.yaml` with the following content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: bluegreen-demo
  namespace: argo-rollouts
spec:
  replicas: 2
  strategy:
    blueGreen:
      activeService: bluegreen-active
      previewService: bluegreen-preview
      autoPromotionEnabled: false
  selector:
    matchLabels:
      app: bluegreen-demo
  template:
    metadata:
      labels:
        app: bluegreen-demo
    spec:
      containers:
        - name: demo
          image: nginx:1.21
          ports:
            - containerPort: 80
```

Apply the rollout and services:

```sh
kubectl apply -f rollout-bluegreen.yaml
kubectl apply -n argo-rollouts -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: bluegreen-active
  namespace: argo-rollouts
spec:
  selector:
    app: bluegreen-demo
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: bluegreen-preview
  namespace: argo-rollouts
spec:
  selector:
    app: bluegreen-demo
  ports:
  - port: 80
    targetPort: 80
EOF
```

## Step 3: Observe the Rollout

Install the Argo Rollouts plugin (optional):

```sh
kubectl argo rollouts get rollout bluegreen-demo -n argo-rollouts
```

## Step 4: Update to a New Version (Green)

Edit the rollout to use a new image (e.g., `nginx:1.25`):

```sh
kubectl argo rollouts -n argo-rollouts set image bluegreen-demo demo=nginx:1.25
```

Check rollout status:

```sh
kubectl argo rollouts get rollout bluegreen-demo -n argo-rollouts
```

## Step 5: Promote or Abort

Promote the new version to active:

```sh
kubectl argo rollouts promote bluegreen-demo -n argo-rollouts
```

Or abort and rollback:

```sh
kubectl argo rollouts abort bluegreen-demo -n argo-rollouts
```

## Step 6: Use the dashboard

Start the dashboard

```sh
kubectl argo rollouts dashboard
```

Open the presented url in your browser, select the namespace 'argo-rollouts' and look at
the visual representation, check the options to promote/rollback and more...

## Step 6: Clean Up

```sh
kubectl delete namespace argo-rollouts
```

---

For more, see the [Argo Rollouts documentation](https://argoproj.github.io/rollouts/).
