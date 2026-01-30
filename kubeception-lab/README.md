# Kubeception Lab: Running K3s (Kubernetes-in-Kubernetes) on Docker Desktop

## Overview

This lab demonstrates how to run a lightweight Kubernetes cluster (K3s) inside a pod on
your existing Kubernetes cluster (Docker Desktop) or Minikube. This technique, known as
"kubeception" or "cluster-in-cluster," is useful for testing, CI/CD, multi-tenancy, and
learning advanced Kubernetes concepts.

## Why K3s?

- Lightweight and fast to start
- Low resource usage
- Official Docker image available
- Ideal for nested cluster scenarios

## Benefits of Kubeception

- Isolated test environments
- Simulate multi-cluster setups
- Experiment with cluster upgrades and networking
- Useful for CI/CD pipelines and operator development

## Lab Steps

1. Deploy a privileged pod running K3s using the official Docker image.
2. Access the nested K3s cluster using `kubectl exec` or port-forwarding.
3. Apply resource limits and security best practices.
4. Clean up resources after the lab.

## Prerequisites

- Docker Desktop with Kubernetes enabled **or** Minikube running locally
- kubectl installed and configured

## Setup Instructions

### Option 1: Docker Desktop

1. Make sure Docker Desktop is running and Kubernetes is enabled.
2. Apply the provided manifest to deploy the K3s pod:
   ```sh
   kubectl apply -f k3s-pod.yaml
   ```
3. Wait for the pod to be ready:
   ```sh
   kubectl get pods -l app=k3s-nested
   ```
4. Access the nested K3s cluster:
   ```sh
   kubectl exec -it <k3s-pod-name> -- /bin/sh
   # Inside the pod, use: kubectl get nodes --kubeconfig /etc/rancher/k3s/k3s.yaml
   ```
   Or port-forward the K3s API server to your local machine:
   ```sh
   kubectl port-forward <k3s-pod-name> 6443:6443
   # Then use the kubeconfig from the pod to connect locally
   ```
5. Clean up:
   ```sh
   kubectl delete -f k3s-pod.yaml
   ```

### Option 2: Minikube

1. Start Minikube locally (stop and delete existing minikube cluster if you have that
   running):
   ```sh
   minikube start --cpus=4 --memory=4096
   ```
   > Adjust resources as needed for your system.
2. Set your kubectl context to Minikube:
   ```sh
   kubectl config use-context minikube
   ```
3. Apply the K3s pod manifest:
   ```sh
   kubectl apply -f k3s-pod.yaml
   ```
4. Wait for the pod to be ready:
   ```sh
   kubectl get pods -l app=k3s-nested
   ```
5. Access the nested K3s cluster (same as above):
   ```sh
   kubectl exec -it <k3s-pod-name> -- /bin/sh
   # Inside the pod, use: kubectl get nodes --kubeconfig /etc/rancher/k3s/k3s.yaml
   ```
   Or port-forward:
   ```sh
   kubectl port-forward <k3s-pod-name> 6443:6443
   # Use the kubeconfig from the pod to connect locally
   ```
6. Clean up:
   ```sh
   kubectl delete -f k3s-pod.yaml
   ```

## Best Practices

- Use resource limits to avoid overloading the host cluster (Minikube or Docker Desktop)
- Run the pod in a dedicated namespace
- Restrict privileges as much as possible
- Always clean up after testing

## Troubleshooting

- Ensure Docker Desktop or Minikube has enough resources allocated
- Check pod logs for K3s startup issues
- Verify network policies if you cannot access the nested cluster

---

Next: See `k3s-pod.yaml` for the deployment manifest.
