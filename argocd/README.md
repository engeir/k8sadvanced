# 🚀 Lab: GitOps with ArgoCD on Kubernetes

## 🎯 Learning Objectives

By the end of this lab, you will:

- Understand GitOps principles
- Install and configure ArgoCD on Kubernetes
- Deploy applications using ArgoCD from a Git repository
- Explore ArgoCD UI and CLI for app management
- Practice automated sync, rollback, and self-healing

## 🛠️ Prerequisites

- A running Kubernetes cluster (Minikube or other)
- kubectl installed and configured
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) (optional)

---

## Step 1: Install ArgoCD

Create a namespace for ArgoCD:

```sh
kubectl create namespace argocd
```

Install ArgoCD using manifests:

```sh
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check ArgoCD pods:

```sh
kubectl get pods -n argocd
```

## Step 2: Access ArgoCD UI

Expose the ArgoCD API server (choose one):

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open [http://localhost:8080](http://localhost:8080) in your browser.

Get the initial admin password (MacOS or Linux):

```sh
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

-or-

if you're on Windows, use this statement instead:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

Login with username `admin` and the password above.

## Step 3: Deploy an Application with ArgoCD

Create a new namespace for your app:

```sh
kubectl create namespace demo-app
```

Add a sample app repo (e.g., https://github.com/argoproj/argocd-example-apps): In the
ArgoCD UI, click 'New App' and fill in:

- Application Name: `demo-app`
- Project: `default`
- Sync Policy: Automatic
- Repository URL: `https://github.com/argoproj/argocd-example-apps`
- Path: `kustomize-guestbook`
- Destination: Kubernetes cluster, namespace `demo-app`

Click 'Create' and watch ArgoCD deploy the app!

## Step 4: Make changes

Open the ArgoCD UI and go to the application, find the `Details` buttons and `edit` the
definition. Now switch the path you choose for the initial deploy to one of the
`guestbook`, which is a much simpeler deployment. `Save` the changes.

See what happens. You might see that it gets deployed, but will not be completely
green/healthy and there are left overs.

Go back to the details edit-page and select 'Prune' and 'Self heal' options. Save again
and see the difference!

## Step 4: Explore GitOps Features

For this step you need to clone the repo (to be able to make changes). Or create your
own repo and add it as a new application to ArgoCD.

- Make a change in the Git repo (e.g., update image tag)
- ArgoCD detects the change and syncs automatically
- Try manual sync, rollback, and self-healing from the UI or CLI

## Step 5: Custom app

## Step 5: Clean Up

Delete the app from ArgoCD UI or run:

```sh
kubectl delete app demo-app -n argocd
kubectl delete namespace demo-app
kubectl delete namespace argocd
```

---

For more, see the [ArgoCD documentation](https://argo-cd.readthedocs.io/en/stable/).
