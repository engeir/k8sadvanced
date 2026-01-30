# Kubernetes Advanced Lab: Deploying MongoDB with Percona Operator on Minikube

## Introduction

In this lab, you'll learn how to deploy a production-grade MongoDB cluster on Kubernetes
using the Percona Operator. Operators simplify the management of complex stateful
applications by automating deployment, scaling, and maintenance tasks.

---

## Prerequisites

- Minikube running (`minikube start`)
- `kubectl` configured to use your Minikube cluster
- Internet access from your cluster nodes

---

## Step 1: Install the Percona Operator for MongoDB

1. **Create a namespace for the operator:**
   ```bash
   kubectl create namespace percona-mongodb
   ```
2. **Install the operator using official manifests:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/main/deploy/bundle.yaml -n percona-mongodb
   ```
3. **Verify the operator is running:**
   ```bash
   kubectl get pods -n percona-mongodb
   ```
   Look for a pod named `percona-server-mongodb-operator-*` in the `Running` state.

---

## Step 2: Deploy a MongoDB Cluster

1. **Apply the manifest to create the cluster:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/main/deploy/cr.yaml -n percona-mongodb
   ```
2. **Monitor pod creation:**
   ```bash
   kubectl get pods -n percona-mongodb
   ```
   You should see several pods for MongoDB and related components.

---

## Step 3: Install the MongoDB CLI Client (Windows)

If you do not have the MongoDB shell (`mongo`) installed on your Windows machine, follow
these steps:

1. **Download the MongoDB Database Tools:**
   - Go to the official MongoDB Database Tools download page:
     https://www.mongodb.com/try/download/database-tools
   - Select Windows and download the ZIP file.
2. **Extract the ZIP file:**
   - Unzip the downloaded file to a folder of your choice.
3. **Add the folder to your PATH (optional but recommended):**
   - Open System Properties > Environment Variables.
   - Edit the `Path` variable and add the path to the folder containing the `mongo.exe`.
4. **Verify installation:**
   - Open a new Command Prompt and run:
     ```cmd
     mongo --version
     ```
   - You should see the MongoDB shell version output.

---

## Step 4: Connect to MongoDB

1. **Port-forward the MongoDB service:**
   ```bash
   kubectl port-forward svc/cluster1-mongos 27017:27017 -n percona-mongodb
   ```
2. **Connect using the MongoDB shell:**
   ```bash
   mongo --host localhost --port 27017 -u userAdmin -p userAdmin123456 --authenticationDatabase admin
   ```
   Credentials are set in the manifest (`userAdmin`/`userAdmin123456`).

---

## Step 5: Scale the Cluster

1. **Edit the cluster manifest to increase replica count:**
   ```bash
   kubectl edit psmdb/cluster cluster1 -n percona-mongodb
   ```
   Change the `spec.replicas` value for the desired replica set.
2. **Observe scaling:**
   ```bash
   kubectl get pods -n percona-mongodb
   ```

---

## Step 6: Backup and Restore (Optional Advanced)

1. **Enable backup in the manifest and apply changes.**

   **!! Important: This will only work with some cloud or netwerk storage**, so might
   not be available in your lab environment. Alternatively you can have a look below or
   investigate in the documentation.

   Below is an example of how to configure a backup that stores to an AWS s3 (or
   compatible) storage

   a. Edit your cluster manifest (e.g., `cr.yaml`) and add a backup section under
   `spec`:
   `yaml       spec:          backup:             enabled: true             storages:                s3-us-west:                   type: s3                   s3:                      bucket: <your-bucket-name>                      region: <your-region>                      endpointUrl: <your-s3-endpoint>                      credentialsSecret: my-s3-secret       `

   b. Create a secret with your S3 credentials (replace values as needed):
   `bash       kubectl create secret generic my-s3-secret \          --from-literal=AWS_ACCESS_KEY_ID=<your-access-key> \          --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-key> \          -n percona-mongodb       `

   c. Apply the updated manifest:
   `bash       kubectl apply -f cr.yaml -n percona-mongodb       `

2. **Simulate a restore by deleting a pod and watching it recover.**

---

## Cleanup

1. **Delete the MongoDB cluster:**
   ```bash
   kubectl delete -f cr.yaml -n percona-mongodb
   ```
2. **Delete the operator:**
   ```bash
   kubectl delete -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/main/deploy/bundle.yaml -n percona-mongodb
   kubectl delete namespace percona-mongodb
   ```

---

## Discussion

- What are the benefits of using operators for databases?
- How does Percona Operator simplify MongoDB management?
- What challenges might you face running stateful workloads in Kubernetes?

---

## References

- [Percona Operator for MongoDB Documentation](https://docs.percona.com/percona-operator-for-mongodb/)
- [Percona Operator GitHub](https://github.com/percona/percona-server-mongodb-operator)

---

**Congratulations! You've deployed and managed MongoDB on Kubernetes using the Percona
Operator.**
