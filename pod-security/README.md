# 🔐 Kubernetes Lab: Pod Security (PSA), Capabilities & Admission

This lab explores Pod security configuration using Kubernetes *Pod Security Admission (PSA)*, container securityContext fields (capabilities, runAsNonRoot, seccomp, readOnlyRootFilesystem), and how admission controls behave in Minikube.

---

## 🎯 Learning objectives
- Learn the Pod Security Admission (PSA) enforcement model: **enforce**, **audit**, **warn** and *standards*: **restricted**, **baseline**, **privileged**.
- Use `securityContext` to set `runAsNonRoot`, `capabilities`, `allowPrivilegeEscalation`, `readOnlyRootFilesystem` and `seccomp` profiles.
- Observe how admission rejects or warns on non-compliant pods and how to diagnose failures with `kubectl describe` and events.
- Understand the role of admission controllers and how to extend them (brief overview of validating/mutating webhooks / Kyverno / OPA).

---

## 📋 Prerequisites
- Minikube running with `kubectl` configured.
- Kubernetes version that supports Pod Security Admission (>= v1.22 is recommended).

---

## Lab layout
Files in this folder:
- `ns-restricted.yaml` — namespace labeled to **enforce restricted** policy
- `ns-warn.yaml` — namespace labeled to **warn** about restricted violations
- `privileged-pod.yaml` — demonstrates a privileged pod (should be rejected under restricted)
- `cap-netadmin-pod.yaml` — pod that requests `CAP_NET_ADMIN` (disallowed by restricted)
- `seccomp-unconfined-pod.yaml` — pod with `seccompProfile: Unconfined` (disallowed by restricted)
- `allowed-pod.yaml` — pod that should pass the restricted policy

---

### Step 1 — Create a restricted namespace

This labels the namespace so the Pod Security Admission controller enforces the `restricted` policy.

```
kubectl apply -f ns-restricted.yaml
kubectl get ns lab-podsec -o jsonpath='{.metadata.labels}'
```

If PSA is enabled it will start enforcing restrictions on pods in this namespace right away.

---

### Step 2 — Try creating non-compliant pods

Apply a privileged pod and watch it be rejected:

```
kubectl -n lab-podsec apply -f privileged-pod.yaml
kubectl -n lab-podsec describe pod privileged
kubectl -n lab-podsec get events --sort-by=.lastTimestamp
```

You should see an admission error explaining which rule was violated.

Apply a pod that adds capabilities:

```
kubectl -n lab-podsec apply -f cap-netadmin-pod.yaml
kubectl -n lab-podsec describe pod cap-netadmin
```

Apply a pod with seccomp `Unconfined`:

```
kubectl -n lab-podsec apply -f seccomp-unconfined-pod.yaml
kubectl -n lab-podsec describe pod seccomp-unconfined
```

---

### Step 3 — Use `warn` mode to see warnings instead of rejections

Create a namespace that only warns about violations. This is useful when rolling out policy changes.

```
kubectl apply -f ns-warn.yaml
kubectl -n lab-podsec-warn apply -f privileged-pod.yaml
kubectl -n lab-podsec-warn describe pod privileged
kubectl -n lab-podsec-warn get events --sort-by=.lastTimestamp
```

Under `warn`, the pod may be created but a warning event will be recorded.

---

### Step 4 — Create a pod that complies with `restricted`

Apply the `allowed-pod.yaml` which sets secure defaults (non-root user, seccomp runtime/default, readOnlyRootFilesystem, dropped capabilities):

```
kubectl -n lab-podsec apply -f allowed-pod.yaml
kubectl -n lab-podsec get pods -o wide
kubectl -n lab-podsec describe pod allowed
```

---

### Admission controllers & AdmissionConfiguration (overview)
- Pod Security Admission is an **admission controller** that enforces policy at pod creation time via namespace labels (recommended).
- Cluster admins can configure admission controllers at the API server level. For more advanced admission control (policy-as-code) use tools like **Kyverno** or **OPA Gatekeeper** (they work via validating/mutating webhooks).
- PodSecurityPolicy (PSP) is deprecated; prefer Pod Security Admission + Kyverno/OPA if you need more granularity.

---

### Cleanup

```
kubectl delete -f ns-restricted.yaml
kubectl delete -f ns-warn.yaml
# the namespaces deletion will remove the test pods
```

---

## 🛠️ Troubleshooting
- If a pod is rejected, `kubectl describe pod <name> -n <ns>` and check events for the admission failure reason.
- `kubectl get events -n <ns> --sort-by=.lastTimestamp` shows recent warnings/errors including PSA warnings.

---

Good luck and stay secure! 🔒