### Scaling

Sometimes running one instance of a container is not enough to handle all the requests coming in in a timely manner.

Luckily Kubernetes allows us to spin up more instances when we need to.

To work with scaling we are going to change something to the application we build previously in the chapter about [Ingress](../ingress/README.md)

Open `server.js` and change `app.get('/' ... )` to the following:
```
app.get('/', (req, res) => {
  const app = `${process.env.APPNAME}`;
  const instance = `${process.env.HOSTNAME}`;
  res.send(`<html><head><style>body { background-color: ${process.env.COLOR}</style></head><body><h3>APP: ${app}</h3><h4>${instance}</h4></body></html>\n`);
});
```

Now build a new image version with `docker build . -t webapp:1.0.1` followed by publishing to minikube with `minikube image load webapp:1.0.1`

If you still have the ingress and containers running from the previous lab, you can upgrade the containers to run the new application version. Otherwise run the deployment and ingress.yaml again.

Upgrade with the following commands:
```
kubectl set image -n default deployment/demo-app-1 demo-app-1=webapp:1.0.1
kubectl set image -n default deployment/demo-app-2 demo-app-2=webapp:1.0.1
```

`kubectl scale --replicas=3 -n default deployment/demo-app-1`

Inspect with
```
kubectl get deployment/demo-app-1 -n default
kubectl get pods -n default
```

If you open `http://localhost/app1` in the browser now and refresh the page, you should see responses from the 3 different instances of the application. (remember to open the minikube tunnel first!)

---

## Autoscaling (Horizontal & Vertical)

This section introduces the **Horizontal Pod Autoscaler (HPA)** and **Vertical Pod Autoscaler (VPA)** and shows how to experiment with them using Minikube.

> Prerequisites: make sure metrics are available for HPA. Enable the metrics-server in Minikube if it's not already enabled:

```
minikube addons enable metrics-server
kubectl get deployment -n kube-system metrics-server
```

### Deployment & Service for this demo
Create a small example deployment and service (included in `demo-deployment.yaml` and `demo-service.yaml`) which sets small CPU requests so HPA can work:

```
kubectl apply -f demo-deployment.yaml
kubectl apply -f demo-service.yaml
kubectl get pods -l app=demo-app-autoscale
kubectl describe deployment demo-app-autoscale
```

### Horizontal Pod Autoscaler (HPA)
Apply the HPA for the deployment:

```
kubectl apply -f hpa.yaml
kubectl get hpa demo-app-hpa -w
```

To generate load and observe scaling, run the load generator (this is a simple busybox Pod that continuously hits the service):

```
kubectl apply -f load-generator-pod.yaml
kubectl get pods -w
kubectl top pods
kubectl get deployment demo-app-autoscale
kubectl describe hpa demo-app-hpa
```

You should see the HPA increase the number of replicas when CPU usage climbs above the configured target. Terminate the load generator when done:

```
kubectl delete pod load-generator
```

### Vertical Pod Autoscaler (VPA)
VPA runs as separate components (CRDs + controller). On Minikube you can install VPA using the upstream manifests. Example (run only if you want to experiment with VPA):

```
# Install VPA components (follow upstream docs / pick a release):
# kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/<vpa-release>/vpa-release.yaml
```

After the VPA components are installed, apply the VPA object in this folder (`vpa.yaml`):

```
kubectl apply -f vpa.yaml
kubectl describe vpa demo-app-vpa
```

- `updateMode: Auto` will allow the VPA to evict and recreate pods with new resource requests.
- Note: running HPA & VPA together can lead to conflicting behaviors. For lab experimentation, try `updateMode: Off` or use VPA for memory recommendations while HPA manages replicas.

---

### Cleanup

```
kubectl delete -f hpa.yaml
kubectl delete -f vpa.yaml || true
kubectl delete -f load-generator-pod.yaml
kubectl delete -f demo-deployment.yaml
kubectl delete -f demo-service.yaml
```


