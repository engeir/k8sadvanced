# Solution to README

## Prerequisites

```bash
kubectl config current-context
# minikube
kubectl config set-context minikube
```

Then deleting and creating 2 nodes.

- Issue when starting:

  ```bash
  $ minikube start --nodes=2
  😄  minikube v1.38.0 on Redhat 9.7
  ✨  Automatically selected the kvm2 driver
  ❗  Starting v1.39.0, minikube will default to "containerd" container runtime. See #21973 for more info.
  👍  Starting "minikube" primary control-plane node in "minikube" cluster
  🔥  Creating kvm2 VM (CPUs=2, Memory=3900MB, Disk=20000MB) ...
  ❗  Failing to connect to https://registry.k8s.io/ from inside the minikube VM
  💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
  🐳  Preparing Kubernetes v1.35.0 on Docker 28.5.2 ...
  🔗  Configuring CNI (Container Networking Interface) ...
  🔎  Verifying Kubernetes components...
      ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
  🌟  Enabled addons: storage-provisioner, default-storageclass

  👍  Starting "minikube-m02" worker node in "minikube" cluster
  🔥  Creating kvm2 VM (CPUs=2, Memory=3900MB, Disk=20000MB) ...
  🌐  Found network options:
      ▪ NO_PROXY=192.168.39.178
  ❗  Failing to connect to https://registry.k8s.io/ from inside the minikube VM
  💡  To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
  🐳  Preparing Kubernetes v1.35.0 on Docker 28.5.2 ...
      ▪ env NO_PROXY=192.168.39.178
  🔎  Verifying Kubernetes components...
  🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
  ```

We verify by running

```bash
kubectl get nodes # Both nodes show NotReady
kubectl get pods -n kube-system -o wide # kindnet-lscww showing ErrImagePull
# kindnet-n5lwx stuck in ContainerCreating
# coredns and storage-provisioner in Pending
```

Lets revert, use podman and set it to rootless:

```bash
minikube config set rootless true
minikube start --nodes=2 --driver=podman
```

This also did not work since my system needed CPU controllers or something.

> As we'll se below, minikube was abandoned in the end, and this file deleted to go back
> to the original state.

```conf
# /etc/systemd/system/user@.service.d/delegate.conf
[System]
Delegate=cpu cpuset io memory pids
```

Reboot the system and try again with the start command. I had to also remove the
`~/.minikube/` directory and re-set the rootless option:

```bash
rm -rf ~/.minikube
minikube config set rootless true
minikube start --nodes=2 --driver=podman
```

### Still issues

We move over to using kind.

```bash
kind create cluster --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF
```

Verifying success with:

```bash
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                 CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   2m22s   v1.35.0   10.89.2.2     <none>        Debian GNU/Linux 12 (bookworm)   5.14.0-611.24.1.el9_7.x86_64   containerd://2.2.0
kind-worker          Ready    <none>          2m14s   v1.35.0   10.89.2.3     <none>        Debian GNU/Linux 12 (bookworm)   5.14.0-611.24.1.el9_7.x86_64   containerd://2.2.0
```

## Step 1

Ok.

## Step 2

The command now show the node as "Ready".

```bash
kubectl get node $NODE --show-labels
```

## Step 3

```bash
kubectl apply -n lab-affinity -f required-affinity-deployment.yaml
kubectl get pods -n lab-affinity -o wide
kubectl describe pod -n lab-affinity -l app=nginx-req
```

These three didn't work initially, since I labelled the control-plane node instead of
the worker node.

### Why labelling the control-plane didn't work

When I first labelled `kind-control-plane` with `disktype=ssd`, the pod remained
**Pending**. This was a different scenario than what the README describes.

**README's "Pending" scenario:**

- "If not, the pod will remain Pending" means: **No nodes have the `disktype=ssd` label
  at all**
- Pod can't match the required affinity
- Events would show: `"0/2 nodes matched Pod's node affinity/selector"`

**My "Pending" scenario (with control-plane labelled):**

- **A node HAD the `disktype=ssd` label** (kind-control-plane) ✅
- **BUT that node had a taint** the pod couldn't tolerate ❌
- Events showed: `"1 node(s) didn't match affinity, 1 node(s) had untolerated taints"`

The control-plane node has this taint:

```txt
Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

This is a standard Kubernetes best practice - control plane nodes are tainted to keep
them free for system components and prevent user workloads from interfering.

**The fix:** Label the worker node instead:

```bash
kubectl label node kind-control-plane disktype-  # Remove label
kubectl label node kind-worker disktype=ssd      # Add to worker
```

**So was labelling the control-plane "wrong"?**

Not wrong, just **ineffective**. The label was there, but the taint blocked scheduling.

**Parking lot analogy:**

> We are VIP

- README scenario: "No parking spots are marked 'Reserved for VIP'" → Can't park
- My scenario: "One spot is marked 'Reserved for VIP' but it ALSO says 'No Entry without
  permit'" → Still can't park

**Key lesson:** For a pod to be scheduled on a node, **BOTH** conditions must be
satisfied:

1. ✅ **Node Affinity**: Node must have the required label
2. ✅ **Taints/Tolerations**: Pod must tolerate ANY taints on the node

Both result in Pending, but for different reasons:

- README scenario: Pending due to **affinity mismatch**
- My case: Pending due to **taint** (affinity matched, but taint blocked it)

## Step 4

This worked now as expected after step 3 was fixed. However, this would also have
started running even if the label was placed on the control-plane, since the pod only
have a preference for nodes labelled `disktype=ssd`, not a requirement for it.

## Step 5

Since we chose to label the _second_ node above, we now focus on the first node when we
now play around with tolerating taints. To make out scenario identical in outcome as the
README specify, we add a toleration to the `with-toleration` YAML file.

```yaml
# ...
# under spec, tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
# ...
```

Then

```bash
NODE=$(kubectl get nodes -o name | sed -n '1p' | cut -d/ -f2)
kubectl taint nodes $NODE key1=value1:NoSchedule
kubectl describe node $NODE | sed -n '/Taints/,$p'
```

```bash
kubectl apply -n lab-affinity -f taint-pod-no-toleration.yaml
kubectl apply -n lab-affinity -f taint-pod-with-toleration.yaml
kubectl get pods -n lab-affinity -o wide
kubectl describe pod -n lab-affinity taint-demo-no-tol
kubectl describe pod -n lab-affinity taint-demo-with-tol
```

### How to check where a pod can possibly run

```bash
# Check pod tolerations
kubectl get pod -n lab-affinity taint-demo-with-tol -o jsonpath='{.spec.tolerations}' | jq

# Check node taints
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.spec.taints // "none")"'

# Compare: pod can run where all node taints are tolerated
```

## Step 6

This step demonstrates a **deadlock scenario** where a pod can't schedule anywhere.

**Setup:**

1. Pod has **required** affinity for `purpose=tainted` label (MUST run on that node)
2. That node has a taint
3. Pod has NO toleration for the taint

**Result:** Pod stays **Pending** because:

- Affinity says: "I MUST run on the node with `purpose=tainted`"
- Taint says: "You CANNOT run here without toleration"
- **Deadlock!** → Pod can't schedule anywhere

**Parking lot analogy:** "I MUST park in spot #5" but spot #5 says "No parking without
permit" and you don't have a permit → you can't park anywhere!

**Commands:**

```bash
# Label the control-plane node
kubectl label node kind-control-plane purpose=tainted

# Already has taint from Step 5: key1=value1:NoSchedule
# Verify:
kubectl describe node kind-control-plane | grep Taints

# Apply the deployment that requires the tainted node
kubectl apply -n lab-affinity -f required-affinity-tainted-node.yaml
kubectl get pods -n lab-affinity -o wide
# NAME                                        READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
# nginx-preferred-affinity-6c5b64965f-9qzvr   1/1     Running   0          45m   10.244.1.3   kind-worker   <none>           <none>
# nginx-required-affinity-78d87d44bf-ntwst    1/1     Running   0          63m   10.244.1.2   kind-worker   <none>           <none>
# nginx-required-tainted-788b44f7b5-mrqph     0/1     Pending   0          29s   <none>       <none>        <none>           <none>
# taint-demo-no-tol                           1/1     Running   0          19m   10.244.1.4   kind-worker   <none>           <none>
# taint-demo-with-tol                         1/1     Running   0          14m   10.244.1.6   kind-worker   <none>           <none>
kubectl describe pod -n lab-affinity -l app=nginx-req-tainted
# ...
# Events:
#   Type     Reason            Age   From               Message
#   ----     ------            ----  ----               -------
#   Warning  FailedScheduling  55s   default-scheduler  0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint(s). no new claims to deallocate, preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```

The pod will remain **Pending** because it requires a node it cannot tolerate.
