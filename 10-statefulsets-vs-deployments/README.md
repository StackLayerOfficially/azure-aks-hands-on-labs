# StatefulSets vs. Deployments on Azure AKS: Architecting Ordered, Predictable Workloads

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In cloud engineering, choosing the right Kubernetes controller is critical for application reliability. Stateless workloads like web servers and APIs run best inside a standard **Deployment**, where pods are interchangeable and hold no permanent local data. However, databases and distributed systems (such as PostgreSQL, MySQL, Redis, and Kafka) require fixed identities, sequential startup, and dedicated storage disks that only a **StatefulSet** can provide.

This lab provides verified, declarative Kubernetes manifests to deploy, inspect, and contrast StatefulSets and Deployments on Microsoft Azure using Azure Kubernetes Service (AKS) and Azure Managed Disks.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [StatefulSets vs. Deployments on Azure AKS: Architecting Ordered, Predictable Workloads](https://stacklayer.blogspot.com/2026/10/statefulsets-vs-deployments-azure-aks-ordered-predictable-workloads.html)

---

## 📁 Directory Structure

```text
10-statefulsets-vs-deployments/
├── README.md                                         # Lab documentation and operational runbook
└── kube-manifests/
    ├── 01-stacklayer-stateful-headless-service.yaml   # Headless service for direct pod DNS routing
    ├── 02-stacklayer-statefulset.yaml                # StatefulSet with 5Gi Azure Managed Disks
    └── 03-stacklayer-stateless-deployment.yaml       # Comparison Deployment with random pod names
```

---

## 📋 Prerequisites

* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS)](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)).
* **Azure CLI** (`az`) and **Kubectl** installed and authenticated.

Connect your local terminal to your running cluster:

```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity and healthy nodes
kubectl cluster-info
kubectl get nodes
```

---

## 🚀 Hands-on Execution Runbook

### Step 1: Deploy the Headless Service

A Headless Service (`clusterIP: None`) skips allocating a single shared cluster IP. Instead, it registers individual direct DNS records for each stateful pod.

```bash
kubectl apply -f kube-manifests/01-stacklayer-stateful-headless-service.yaml
```

**Command Breakdown:**
* `kubectl apply`: Submits the declarative manifest to the cluster.
* `-f kube-manifests/01-stacklayer-stateful-headless-service.yaml`: Creates the direct internal routing service named `stacklayer-stateful-service`.

---

### Step 2: Deploy the StatefulSet with Dedicated Azure Disks

Deploy the StatefulSet. Through `volumeClaimTemplates`, Kubernetes automatically asks the Azure CSI driver to create a separate 5GB Azure Managed Disk (`managed-csi`) for each pod.

```bash
kubectl apply -f kube-manifests/02-stacklayer-statefulset.yaml
```

Watch the pods launch in strict order (pod `0` must finish before pod `1` begins):

```bash
kubectl get pods -l app=stacklayer-stateful-app -w
```

**Command Breakdown:**
* `-l app=stacklayer-stateful-app`: Filters the output to display only our stateful pods.
* `-w`: Watches status changes in real time. Press `Ctrl + C` when both pods show `Running`.

---

### Step 3: Verify Dedicated Azure Disks and Separate Storage

Verify that each pod received its own dedicated PersistentVolumeClaim (PVC):

```bash
kubectl get pvc -l app=stacklayer-stateful-app
```

Check the data stored on pod `0`:

```bash
kubectl exec stacklayer-db-0 -- curl -s http://localhost
```

Check the data stored on pod `1`:

```bash
kubectl exec stacklayer-db-1 -- curl -s http://localhost
```

Each pod writes and reads only from its own private Azure Disk.

---

### Step 4: Test Pod Recovery and Data Safety

Delete pod `stacklayer-db-0` to simulate a pod failure:

```bash
kubectl delete pod stacklayer-db-0
```

Check the pods again to observe the recovery:

```bash
kubectl get pods -l app=stacklayer-stateful-app
```

Verify that the new pod kept the exact same name (`stacklayer-db-0`) and that its stored data is safe:

```bash
kubectl exec stacklayer-db-0 -- curl -s http://localhost
```

---

### Step 5: Contrast with a Stateless Deployment

Deploy a standard stateless Deployment with 2 replicas to observe the difference in pod naming:

```bash
kubectl apply -f kube-manifests/03-stacklayer-stateless-deployment.yaml
```

List all pods in the cluster:

```bash
kubectl get pods
```

Notice the contrast:
* **StatefulSet pods** have clear, ordered names: `stacklayer-db-0`, `stacklayer-db-1`.
* **Deployment pods** have random generated hashes: `stacklayer-web-deployment-7f85d68d4f-5g9xq`.

---

### Step 6: Test Ordered Scaling (Up and Down)

Scale the StatefulSet up to 3 replicas:

```bash
kubectl scale statefulset stacklayer-db --replicas=3
kubectl get pods -l app=stacklayer-stateful-app
```

Notice that `stacklayer-db-2` is added after `stacklayer-db-1`.

Now scale down to 1 replica to observe safe reverse termination:

```bash
kubectl scale statefulset stacklayer-db --replicas=1
kubectl get pods -l app=stacklayer-stateful-app -w
```

Notice that Kubernetes terminates `stacklayer-db-2` first, then `stacklayer-db-1`, while `stacklayer-db-0` remains completely untouched. Even after scaling down, the Azure Disks remain safely preserved in your cluster.

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure subscription credits and prevent unnecessary cloud compute billing, choose one of the following teardown options:

### Option A: Pause Cluster Nodes (Recommended If Continuing Later Labs)
Suspends AKS VMSS virtual machine nodes so compute billing stops immediately while keeping all manifests, networking, and disk claims intact for future practice:

```bash
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

To resume your cluster later:
```bash
az aks start --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Complete Deletion (Zero Cost)
Removes all deployed manifests and persistent disks from your AKS cluster:

```bash
kubectl delete -f kube-manifests/03-stacklayer-stateless-deployment.yaml
kubectl delete -f kube-manifests/02-stacklayer-statefulset.yaml
kubectl delete -f kube-manifests/01-stacklayer-stateful-headless-service.yaml
kubectl delete pvc -l app=stacklayer-stateful-app
```

If you wish to delete the entire Azure Resource Group and all associated resources:

```bash
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <h3>🌐 Master Cloud-Native State on Azure AKS</h3>
  <p><i>“When applications need memory and order, StatefulSets give every pod its true home and identity.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
