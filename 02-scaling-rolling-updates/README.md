# Declarative Pod Scaling & Zero-Downtime Rolling Updates on Azure AKS

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In real-world Kubernetes deployments, managing workloads declaratively through YAML manifests ensures consistency, auditability, and reliable version control. In this hands-on lab, you will learn how to scale application replicas and perform seamless, zero-downtime rolling updates across container releases (`nginx-v1` ➔ `nginx-v2` ➔ `nginx-v3`) using declarative Kubernetes configuration files.

---

## 📖 Companion Blog Guide
For comprehensive step-by-step explanations, command-line equivalents, and live container log diagnostics, refer to our companion publication:
* **Official Guide:** [Zero-Downtime Rolling Updates, Pod Scaling, and Log Troubleshooting on Azure AKS](https://stacklayer.blogspot.com/2026/09/azure-aks-tutorial-scaling-rolling-updates.html)

---

## 📁 Directory Structure

```text
02-scaling-rolling-updates/
├── README.md
└── kube-manifests/
    ├── 0A-Deployment.yaml
    └── 0B-Service.yaml
```

---

## 🎯 Learning Objectives

By completing this lab, you will:
1. Declaratively scale pod replicas from 2 up to 5 by updating `0A-Deployment.yaml`.
2. Perform a **Zero-Downtime Rolling Update** by upgrading the container image from `stacklayer/kubernetes:nginx-v1` to `stacklayer/kubernetes:nginx-v2`.
3. Promote an updated release to `stacklayer/kubernetes:nginx-v3` with uninterrupted user connectivity.
4. Verify that the public Azure Load Balancer (`0B-Service.yaml`) remains active and stable while pods update in the background.
5. Track deployment rollout progress and inspect historical revisions.
6. Safely manage cluster lifecycles using Azure CLI to eliminate unnecessary cloud billing.

---

## 📋 Prerequisites

* An active **Azure Subscription** ([Azure Free Account](https://azure.microsoft.com/free/) or Student Credits).
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

### Step 1: Deploy Baseline Application Manifests

Ensure your baseline web application is deployed using the two declarative manifests:

#### `kube-manifests/0A-Deployment.yaml` (Baseline: 2 Replicas, nginx-v1)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: stacklayer-web
  template: 
    metadata: 
      name: stacklayer-web-pod
      labels: 
        app: stacklayer-web       
    spec:
      containers: 
        - name: stacklayer-web-container
          image: stacklayer/kubernetes:nginx-v1
          ports:
            - containerPort: 80
```

#### `kube-manifests/0B-Service.yaml` (Azure Load Balancer)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-web-loadbalancer
  labels: 
    app: stacklayer-web
spec:
  type: LoadBalancer 
  selector:
    app: stacklayer-web
  ports: 
    - port: 80
      targetPort: 80
```

Apply both manifests to your AKS cluster:
```bash
# Apply baseline manifests
kubectl apply -f kube-manifests/

# Verify running pods and external IP
kubectl get pods -l app=stacklayer-web
kubectl get svc stacklayer-web-loadbalancer
```

---

### Step 2: Declarative Pod Scaling (Scale Up to 5 Replicas)

When application demand increases, scale your workload declaratively by updating the desired state in your manifest:

1. Open `kube-manifests/0A-Deployment.yaml` and update `replicas` to `5`:
```yaml
spec:
  replicas: 5
```

2. Apply the updated manifest:
```bash
# Apply scaled configuration
kubectl apply -f kube-manifests/0A-Deployment.yaml

# Watch new pods spin up across cluster nodes in real time
kubectl get pods -l app=stacklayer-web -w
```
*(Press `Ctrl + C` once all 5 pods report `Running` status).*

Notice that `0B-Service.yaml` requires zero modifications—the Azure Load Balancer automatically detects the new pods and begins distributing web traffic evenly across all 5 instances.

---

### Step 3: Zero-Downtime Rolling Update to `nginx-v2`

When releasing an updated version of your application, Kubernetes uses a staged rolling update strategy: it starts a new pod with the updated image, confirms its readiness, and only then terminates an older pod.

1. Open `kube-manifests/0A-Deployment.yaml` and update the container image to `stacklayer/kubernetes:nginx-v2`:
```yaml
    spec:
      containers: 
        - name: stacklayer-web-container
          image: stacklayer/kubernetes:nginx-v2
          ports:
            - containerPort: 80
```

2. Apply the manifest to start the rollout:
```bash
# Apply the version 2 update
kubectl apply -f kube-manifests/0A-Deployment.yaml

# Monitor rollout progression in real time
kubectl rollout status deployment/stacklayer-web-deployment
```

3. Confirm that all running pods are now serving version 2:
```bash
# Check the container image on all active pods
kubectl get pods -l app=stacklayer-web -o jsonpath="{..image}"
```

---

### Step 4: Promote Release to `nginx-v3` & View Revision History

To release the next application iteration, update the image tag to `nginx-v3`:

1. Update `kube-manifests/0A-Deployment.yaml`:
```yaml
    spec:
      containers: 
        - name: stacklayer-web-container
          image: stacklayer/kubernetes:nginx-v3
          ports:
            - containerPort: 80
```

2. Apply the update:
```bash
kubectl apply -f kube-manifests/0A-Deployment.yaml
kubectl rollout status deployment/stacklayer-web-deployment
```

3. Inspect the documented deployment revision history:
```bash
# View rollout revision history
kubectl rollout history deployment/stacklayer-web-deployment
```

---

### Step 5: Declarative Rollback to a Previous Version

If a release needs to be rolled back, simply update `0A-Deployment.yaml` back to the desired image tag (e.g., `stacklayer/kubernetes:nginx-v2`) and re-apply:

```bash
# Apply previous manifest configuration
kubectl apply -f kube-manifests/0A-Deployment.yaml

# Confirm rollback completion
kubectl rollout status deployment/stacklayer-web-deployment
```

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure credits after completing the lab, choose one of the options below:

### Option A: Stop Cluster (Pauses VM compute charges, preserves configuration)
```bash
# Stop the AKS cluster
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster

# Resume whenever you want to practice again:
# az aks start --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Delete Resource Group (Permanent Teardown)
```bash
# Permanently delete the resource group and all contained resources
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support
* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud tutorials on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
