# Stateful Workloads on Azure AKS: Persistent Volumes with Azure Disks & Azure Files  (PV & PVCs)

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

By default, containers in Kubernetes are stateless and ephemeral—when a pod crashes, restarts, or reschedules to another worker node, all data written inside the container filesystem is permanently lost. To run stateful enterprise workloads (such as relational databases, content management systems, and shared media repositories), Kubernetes utilizes the **Container Storage Interface (CSI)** to dynamically provision and attach durable cloud storage.

In this hands-on lab, you will master dynamic persistent storage provisioning on **Azure Kubernetes Service (AKS)** using **StorageClasses**, **PersistentVolumeClaims (PVCs)**, and **PersistentVolumes (PVs)** across two major cloud storage tiers:
1. **Azure Managed Disks (`ReadWriteOnce`):** High-performance block storage dedicated to single-pod workloads (ideal for databases like MySQL and PostgreSQL).
2. **Azure Files (`ReadWriteMany`):** Distributed cloud file shares mounted concurrently across multiple pods across different nodes (ideal for shared media, web assets, and WordPress).

---

## 📖 Companion Blog Guide
For in-depth architectural diagrams, access mode deep dives, and live troubleshooting checks, refer to our companion publication:
* **Official Guide:** [Stateful Workloads on Azure AKS: Persistent Volumes with Azure Disks & Azure Files with PV and PVCs](https://stacklayer.blogspot.com/2026/09/azure-aks-persistent-storage-disks-files.html)

---

## 📁 Directory Structure

```text
04-persistent-storage-disks-files/
├── README.md
└── kube-manifests/
    ├── 01-azure-disk-pvc.yaml           # Azure Managed Disk PVC (RWO, managed-csi)
    ├── 02-azure-disk-deployment.yaml    # Workload mounting Azure Disk volume
    ├── 03-azure-files-pvc.yaml          # Azure Files PVC (RWX, azurefile-csi)
    └── 04-azure-files-deployment.yaml   # Multi-replica workload sharing Azure Files
```

---

## 🎯 Learning Objectives

By completing this lab, you will:
1. Query and inspect built-in AKS **StorageClasses** (`managed-csi` and `azurefile-csi`).
2. Declaratively provision high-speed **Azure Managed Disks** using PersistentVolumeClaims with `ReadWriteOnce` access mode.
3. Attach persistent block storage to a container and prove **Data Survivability** across pod crashes and restarts.
4. Declaratively provision an **Azure Files** SMB share using `ReadWriteMany` access mode.
5. Scale a multi-replica application that reads and writes concurrently to the exact same shared filesystem.
6. Safely manage cluster lifecycles using Azure CLI to eliminate unnecessary cloud billing.

---

## 📋 Prerequisites

* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS) Step-by-Step](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)).
* **Azure CLI (`az`)** and **Kubectl** installed and authenticated.

Connect your local terminal to your running cluster:
```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity
kubectl cluster-info
```

---

## 🚀 Declarative Step-by-Step Hands-On Guide

### Step 1: Inspect Pre-Configured AKS StorageClasses

AKS includes built-in StorageClasses powered by Azure CSI drivers:

```bash
# List all pre-installed StorageClasses on your AKS cluster
kubectl get storageclass
```

You will see:
* `managed-csi` (Default): Dynamic Azure Managed Disk (Standard SSD).
* `managed-csi-premium`: Ultra-fast Premium SSD Azure Disk.
* `azurefile-csi`: Dynamic Azure Files standard storage account (SMB/NFS).
* `azurefile-csi-premium`: High-throughput Azure Files premium storage.

---

### Step 2: Provision Azure Managed Disk (`ReadWriteOnce`)

Azure Disks attach as raw virtual hard disks to a single worker node at a time (`ReadWriteOnce`).

#### `kube-manifests/01-azure-disk-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stacklayer-disk-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 5Gi
```

Apply the claim:
```bash
# Deploy Azure Disk PVC
kubectl apply -f kube-manifests/01-azure-disk-pvc.yaml

# Check PVC status
kubectl get pvc stacklayer-disk-pvc
```
*(Note: With `managed-csi`, the PVC status will remain `Pending` until a pod actually requests it—this is standard Kubernetes `WaitForFirstConsumer` binding).*

---

### Step 3: Attach Azure Disk to Workload & Test Data Survivability

Now, mount the disk to `/mnt/azuredisk` inside a deployment.

#### `kube-manifests/02-azure-disk-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-disk-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stacklayer-disk-app
  template:
    metadata:
      labels:
        app: stacklayer-disk-app
    spec:
      containers:
      - name: storage-tester
        image: stacklayer/kubernetes:nginx-v1
        ports:
        - containerPort: 80
        volumeMounts:
        - name: azure-disk-storage
          mountPath: /mnt/azuredisk
      volumes:
      - name: azure-disk-storage
        persistentVolumeClaim:
          claimName: stacklayer-disk-pvc
```

Apply the deployment:
```bash
# Deploy workload
kubectl apply -f kube-manifests/02-azure-disk-deployment.yaml

# Confirm PVC is now Bound and a real Azure VHD disk is provisioned
kubectl get pvc stacklayer-disk-pvc
kubectl get pv
```

#### Prove Data Survivability (The Persistence Test):

1. Write a timestamped file into the mounted disk:
```bash
export DISK_POD=$(kubectl get pods -l app=stacklayer-disk-app -o jsonpath="{.items[0].metadata.name}")
kubectl exec $DISK_POD -- sh -c "echo 'Stack Layer Persistence Test: Data Saved at $(date)' > /mnt/azuredisk/status.txt"
```

2. Confirm the file exists on the disk:
```bash
kubectl exec $DISK_POD -- cat /mnt/azuredisk/status.txt
```

3. Delete the pod to simulate a catastrophic hardware crash:
```bash
kubectl delete pod $DISK_POD
```

4. Verify that the new pod automatically re-attached the Azure Disk and recovered the data:
```bash
export NEW_DISK_POD=$(kubectl get pods -l app=stacklayer-disk-app -o jsonpath="{.items[0].metadata.name}")
kubectl exec $NEW_DISK_POD -- cat /mnt/azuredisk/status.txt
```
*Your file is completely intact! The data survived the pod destruction.*

---

### Step 4: Provision Azure Files Shared Storage (`ReadWriteMany`)

When multiple pods running on different cluster nodes must read and write to the exact same filesystem simultaneously, use Azure Files with `ReadWriteMany`.

#### `kube-manifests/03-azure-files-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stacklayer-files-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 5Gi
```

Apply the Azure Files claim:
```bash
# Deploy Azure Files PVC
kubectl apply -f kube-manifests/03-azure-files-pvc.yaml
kubectl get pvc stacklayer-files-pvc
```

---

### Step 5: Mount Azure Files Across Multiple Concurrent Pods

Deploy a 3-replica application where all 3 pods mount `/mnt/azurefiles`.

#### `kube-manifests/04-azure-files-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-files-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stacklayer-files-app
  template:
    metadata:
      labels:
        app: stacklayer-files-app
    spec:
      containers:
      - name: shared-worker
        image: stacklayer/kubernetes:nginx-v1
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-azurefile
          mountPath: /mnt/azurefiles
      volumes:
      - name: shared-azurefile
        persistentVolumeClaim:
          claimName: stacklayer-files-pvc
```

Apply the deployment:
```bash
# Deploy shared workload
kubectl apply -f kube-manifests/04-azure-files-deployment.yaml

# Confirm all 3 replicas are running
kubectl get pods -l app=stacklayer-files-app -o wide
```

#### Test Multi-Pod File Sharing:

1. Have Pod 1 write a file to the shared file share:
```bash
export POD_1=$(kubectl get pods -l app=stacklayer-files-app -o jsonpath="{.items[0].metadata.name}")
kubectl exec $POD_1 -- sh -c "echo 'Hello from Pod 1 via Shared Azure Files!' > /mnt/azurefiles/shared.txt"
```

2. Have Pod 2 and Pod 3 read that exact file immediately:
```bash
export POD_2=$(kubectl get pods -l app=stacklayer-files-app -o jsonpath="{.items[1].metadata.name}")
export POD_3=$(kubectl get pods -l app=stacklayer-files-app -o jsonpath="{.items[2].metadata.name}")

kubectl exec $POD_2 -- cat /mnt/azurefiles/shared.txt
kubectl exec $POD_3 -- cat /mnt/azurefiles/shared.txt
```
*Both pods read the shared file in real time across node boundaries!*

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure credits after completing the lab, choose one of the options below:

### First: Delete Persistent Claims & Workloads
```bash
# Delete workloads and PVC claims
kubectl delete -f kube-manifests/02-azure-disk-deployment.yaml
kubectl delete -f kube-manifests/04-azure-files-deployment.yaml
kubectl delete -f kube-manifests/01-azure-disk-pvc.yaml
kubectl delete -f kube-manifests/03-azure-files-pvc.yaml
```

### Option A: Stop Cluster (Pauses VM compute charges, preserves configuration)
```bash
# Stop the AKS cluster
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster

# Resume whenever you want to practice again:
# az aks start --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Delete Resource Group (Permanent Teardown)
```bash
# Permanently delete the resource group and all contained storage resources
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support
* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud tutorials on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <br />
  <h3>💾 Resilient by Design. Durable by Architecture.</h3>
  <p style="max-width: 580px; line-height: 1.6; font-style: italic;">
    “In cloud computing, compute is transient, but data is forever. Mastering Persistent Volumes is the true threshold between deploying toys and engineering mission-critical enterprise systems.”
  </p>
  <br />
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
