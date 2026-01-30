# CloudNativePG Operator Lab

This lab will teach you how to deploy and manage PostgreSQL clusters on Kubernetes using
the CloudNativePG operator.

## Objectives

- Install the CloudNativePG operator
- Deploy a PostgreSQL cluster
- Connect to the database
- Scale the cluster
- Perform a backup and restore

## Prerequisites

- Access to a Kubernetes cluster (e.g., Minikube, Kind, AKS, EKS, GKE)
- `kubectl` installed and configured

## Lab Steps

### 1. Install the CloudNativePG Operator

1. Add the CloudNativePG Helm repository:
   ```sh
   helm repo add cloudnative-pg https://cloudnative-pg.github.io/charts/
   helm repo update
   ```
2. Install the operator:
   ```sh
   helm install cloudnative-pg cloudnative-pg/cloudnative-pg --namespace cnpg-system --create-namespace
   ```
3. Verify the operator is running:
   ```sh
   kubectl get pods -n cnpg-system
   ```

---

### Installing psql CLI on Windows

To connect to PostgreSQL, you need the `psql` command-line client. On Windows, you can
install it as follows:

1. Download the PostgreSQL installer from the official site:
   [https://www.postgresql.org/download/windows/](https://www.postgresql.org/download/windows/)
2. Run the installer and select only the "Command Line Tools" if you do not want the
   full PostgreSQL server. (you don't want that, we're running it in the cluster)
3. After installation, add the PostgreSQL `bin` directory (e.g.,
   `C:\Program Files\PostgreSQL\16\bin`) to your system `PATH` environment variable.
4. Open a new Command Prompt and run:
   ```sh
   psql --version
   ```
   You should see the installed version.

Alternatively, you can use
[Windows Package Manager (winget)](https://learn.microsoft.com/en-us/windows/package-manager/winget/)
to install PostgreSQL tools:

```sh
winget install PostgreSQL
```

---

### 2. Deploy a PostgreSQL Cluster

1. Create a namespace for your database:
   ```sh
   kubectl create namespace pg-lab
   ```
2. Apply a sample cluster manifest:
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: pg-demo
     namespace: pg-lab
   spec:
     instances: 1
     imageName: ghcr.io/cloudnative-pg/postgresql:16
     storage:
       size: 1Gi
   ```
   Save this as `pg-cluster.yaml` and apply:
   ```sh
   kubectl apply -f pg-cluster.yaml
   ```
3. Check the cluster status:
   ```sh
   kubectl get clusters -n pg-lab
   kubectl get pods -n pg-lab
   ```

### 3. Connect to PostgreSQL

1. Forward the service port:
   ```sh
   kubectl port-forward svc/pg-demo-rw 5432:5432 -n pg-lab
   ```
2. Connect using `psql`:
   ```sh
   psql -h localhost -U postgres
   # Password is in the secret: kubectl get secret pg-demo-superuser -n pg-lab -o jsonpath='{.data.password}' | base64 -d
   ```

### 4. Scale the Cluster

1. Edit the cluster manifest to change `instances` to 3 and re-apply:
   ```sh
   kubectl apply -f pg-cluster.yaml
   kubectl get pods -n pg-lab
   ```

You now have a primary and 2 replicas running. Also have a look at the services:

```sh
kubectl get svc -n pg-lab
```

### 5. Backup and Restore

1. Create a backup resource:
   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Backup
   metadata:
     name: pg-demo-backup
     namespace: pg-lab
   spec:
     cluster:
       name: pg-demo
   ```
   Save as `pg-backup.yaml` and apply:
   ```sh
   kubectl apply -f pg-backup.yaml
   kubectl get backups -n pg-lab
   ```
2. To restore, create a `Restore` resource (see
   [CloudNativePG docs](https://cloudnative-pg.io/docs/)).

## Cleanup

```sh
kubectl delete namespace pg-lab
helm uninstall cloudnative-pg -n cnpg-system
kubectl delete namespace cnpg-system
```

## References

- [CloudNativePG Documentation](https://cloudnative-pg.io/docs/)
- [CloudNativePG GitHub](https://github.com/cloudnative-pg/cloudnative-pg)
