# 🧪 Lab: Deploying and Exploring Istio on Minikube

## 🎯 Learning Objectives

By the end of this lab, participants will be able to:

- Install and configure Istio on a Minikube cluster
- Deploy the Bookinfo sample application
- Use Istio features: traffic management, observability, and security
- Experiment with canary deployments and request routing

## 🛠️ Prerequisites

- Minikube installed (v1.24+), with at least 4 CPUs and 8–16 GB RAM
- kubectl installed and configured
- [istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/) CLI installed
  (download from Istio releases)
- Optional: [k9s](https://k9scli.io/) for easier cluster exploration

## Step 1: Start Minikube

You need to **stop and delete** your existing Minikube first!

```sh
minikube start --cpus=4 --memory=8192mb
```

Open `minikube tunnel` in a separate window. Make sure to run it as administrator (sudo)
to allow for opening ports < 1024

---

## Step 2: Install Istio

Download and install istioctl:

```sh
curl -L https://istio.io/downloadIstio | sh -

export PATH=$HOME/.istioctl/bin:$PATH
```

Install gateway api in Kubernetes

```sh
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

Install Istio

```sh
istioctl experimental precheck
istioctl install --set profile=demo -y
```

## Step 3: Enable Automatic Sidecar Injection

Label the default namespace:

```sh
kubectl label namespace default istio-injection=enabled
```

## Step 4: Deploy a Sample Application

You can use the Bookinfo app or a simple webapp. Example with Bookinfo:

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/bookinfo/platform/kube/bookinfo.yaml
```

## Step 5: Verify Sidecar Injection

Check pods for Istio sidecars (might fail on Window due to missing jq, install via
[https://jqlang.org/download/](https://jqlang.org/download/)):

```sh
kubectl get pods -o json | jq '.items[].spec.containers[].name'
```

## Step 6: Expose the Application via Istio Ingress

Apply the Gateway and VirtualService:

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/bookinfo/networking/bookinfo-gateway.yaml
```

You can check if the bookstore is running:

```sh
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

it should come back with something like `<title>Simple Bookstore App</title>`

Get the ingress IP and port:

```sh
kubectl get svc istio-ingressgateway -n istio-system
```

Access the app in your browser using the EXTERNAL-IP and PORT. With minikube this would
be `http://localhost/productpage`

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.27/samples/bookinfo/networking/destination-rule-all.yaml
```

## Step 7: Observability

Enable Prometheus, Grafana, Jaeger, and Kiali (optional):

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/kiali.yaml
```

Access dashboards using port-forwarding:

```sh
kubectl port-forward svc/grafana -n istio-system 3000:3000
```

and view

Use Kiali dashboard:

```
istioctl dashboard kiali
```

## Step 8: Generate some traffic

```sh
for i in $(seq 1 100); do curl -s -o /dev/null "http://127.0.0.1/productpage"; done
```

---

For more advanced scenarios, see the
[Istio documentation](https://istio.io/latest/docs/).
