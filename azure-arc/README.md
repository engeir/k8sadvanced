# Lab: Connect a Talosctl-based Kubernetes Cluster to Azure Arc

This lab guides you through connecting a local k3d-managed Kubernetes cluster to Azure
Arc.

Initially the lab used Talos Linux, but when enabling extensions in Azure Arc you
quickly run into challenges with the read-only OS filesystem. So for sake of simplicity
we're using K3S in docker (K3D) here.

## Prerequisites

- Azure CLI installed and logged in (`az login`)
- k3d installed and configured for your cluster
- kubectl access to your k3d cluster
- Sufficient permissions in your Azure subscription

## Steps

### 0. Install k3d on Docker Desktop

1. Ensure Docker Desktop is installed and running.
2. Install k3d using Homebrew (macOS):

```sh
brew install k3d
```

Or via script (Linux/macOS):

```sh
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

On Windows you need to run the install script from WSL. Or use chocolatey (if
installed): `choco install k3d`

3. Verify installation:

```sh
k3d version
docker version
```

4. Create a test cluster:

```sh
k3d cluster create test-cluster
kubectl get nodes
```

---

### 1. Register Azure Arc Providers

```
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
```

### 2. Create a Resource Group

```
az group create --name <resource-group> --location <location>
```

### 3. Install Azure Arc CLI Extension

```
az extension add --name connectedk8s
```

### 4. Connect Your k3d Cluster to Azure Arc

Replace `<resource-group>`, `<cluster-name>`, and `<location>` as needed.

```
az connectedk8s connect --name <cluster-name> --resource-group <resource-group> --location <location>
```

This command will deploy Azure Arc agents to your cluster. You may need to provide
kubeconfig path:

```
az connectedk8s connect --name <cluster-name> --resource-group <resource-group> --location <location> --kube-config <path-to-kubeconfig>
```

Note: On k3d, privileged containers are usually allowed by default. If you encounter
issues, ensure your k3d cluster was created with proper options (e.g.,
`--kubeconfig-update-default` and `--kubeconfig-switch-context`).

If you use custom PodSecurityPolicies or restrictions, you may need to label the
namespace as shown below:

```
kubectl create ns azure-arc
kubectl label ns azure-arc \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

kubectl label namespace azure-arc app.kubernetes.io/managed-by=Helm --overwrite
kubectl annotate namespace azure-arc meta.helm.sh/release-name=azure-arc --overwrite
kubectl annotate namespace azure-arc meta.helm.sh/release-namespace=azure-arc-release --overwrite

kubectl create ns azure-arc-release
kubectl label ns azure-arc-release \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite

kubectl label namespace azure-arc-release app.kubernetes.io/managed-by=Helm --overwrite
kubectl annotate namespace azure-arc-release meta.helm.sh/release-name=azure-arc --overwrite
kubectl annotate namespace azure-arc-release meta.helm.sh/release-namespace=azure-arc-release --overwrite
```

### 5. Validate Connection

- Go to Azure Portal > Azure Arc > Kubernetes Clusters
- Confirm your cluster appears and is "Connected"

### 6. (Optional) Apply Azure Policies or GitOps Configurations

Refer to Azure Arc documentation for advanced management.

## Troubleshooting

- Ensure your kubeconfig is correct and points to the Talos cluster
- Check agent pods in `azure-arc` namespace for errors
- Use `kubectl get pods -n azure-arc` to verify agent status

## References

- [Azure Arc Kubernetes Docs](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/)
- [k3d Docs](https://k3d.io/)
