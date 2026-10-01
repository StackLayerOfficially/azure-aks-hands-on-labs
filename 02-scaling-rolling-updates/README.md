# Declarative Pod Scaling & Zero-Downtime Rolling Updates on Azure AKS

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In real-world Kubernetes deployments, managing workloads declaratively through separate, version-controlled YAML manifests guarantees predictability, auditability, and seamless teamwork. In this hands-on lab, you will learn how to scale application replicas and perform zero-downtime rolling updates across production container releases (`nginx-v1` ➔ `nginx-v2` ➔ `nginx-v3`) using dedicated Kubernetes configuration files.

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
    ├── 01-Deployment-v1.yaml    # Baseline: 2 Replicas, image: stacklayer/kubernetes:nginx-v1
    ├── 02-Deployment-v2.yaml    # Scaled & Updated: 5 Replicas, image: stacklayer/kubernetes:nginx-v2
    ├── 03-Deployment-v3.yaml    # Next Release: 5 Replicas, image: stacklayer/kubernetes:nginx-v3
    └── 04-Service.yaml          # Azure Load Balancer (Public IP Endpoint)
```

---

## 🎯 Learning Objectives

By completing this lab, you will:
1. Deploy a baseline web workload using `01-Deployment-v1.yaml` and `04-Service.yaml`.
2. Declaratively scale pod replicas from 2 to 5 while simultaneously executing a **Zero-Downtime Rolling Update** to `stacklayer/kubernetes:nginx-v2` using `02-Deployment-v2.yaml`.
3. Promote an updated release to `stacklayer/kubernetes:nginx-v3` with zero service interruption using `03-Deployment-v3.yaml`.
4. Verify that the front-facing Azure Load Balancer (`04-Service.yaml`) remains persistent and uninterrupted throughout all rolling updates.
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

### Step 1: Deploy Baseline Application (`v1`)

Deploy the baseline version of the application consisting of 2 pod replicas running `nginx-v1` behind an Azure Standard Load Balancer.

#### `kube-manifests/01-Deployment-v1.yaml`
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

#### `kube-manifests/04-Service.yaml`
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

Apply both manifests:
```bash
# Apply baseline deployment and load balancer service
kubectl apply -f kube-manifests/01-Deployment-v1.yaml
kubectl apply -f kube-manifests/04-Service.yaml

# Verify running pods and service IP
kubectl get pods -l app=stacklayer-web
kubectl get svc stacklayer-web-loadbalancer
```

---

### Step 2: Scale Up to 5 Replicas & Rolling Update to `nginx-v2`

To handle increased traffic and deploy software release `v2`, apply `02-Deployment-v2.yaml`. This manifest increases the desired replica count from 2 to 5 and updates the container image to `stacklayer/kubernetes:nginx-v2`.

#### `kube-manifests/02-Deployment-v2.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-web-deployment
spec:
  replicas: 5
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
          image: stacklayer/kubernetes:nginx-v2
          ports:
            - containerPort: 80
```

Apply the updated manifest:
```bash
# Apply scaled v2 deployment
kubectl apply -f kube-manifests/02-Deployment-v2.yaml

# Monitor the rollout status in real-time
kubectl rollout status deployment/stacklayer-web-deployment

# Confirm that 5 pods are running with nginx-v2
kubectl get pods -l app=stacklayer-web -o wide
```

> **Notice:** The Azure Load Balancer defined in `04-Service.yaml` requires zero updates. It automatically discovers all 5 pods and distributes traffic across them without a single millisecond of downtime.

---

### Step 3: Promote Next Production Release to `nginx-v3`

When the development team delivers release `v3`, deploy `03-Deployment-v3.yaml` to execute another staged rolling update.

#### `kube-manifests/03-Deployment-v3.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-web-deployment
spec:
  replicas: 5
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
          image: stacklayer/kubernetes:nginx-v3
          ports:
            - containerPort: 80
```

Apply the `v3` release:
```bash
# Deploy v3
kubectl apply -f kube-manifests/03-Deployment-v3.yaml

# Track rollout progress
kubectl rollout status deployment/stacklayer-web-deployment

# Inspect rollout revision history
kubectl rollout history deployment/stacklayer-web-deployment
```

---

### Step 4: Declarative Rollback to a Previous Stable Release

If an issue occurs in `v3` and you need to restore the stable `v2` environment immediately, re-apply `02-Deployment-v2.yaml`:

```bash
# Revert to stable v2 release declaratively
kubectl apply -f kube-manifests/02-Deployment-v2.yaml

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
  <br />
  <h3>🚀 Never Stop Building, Never Stop Scaling</h3>
  <p style="max-width: 580px; line-height: 1.6; font-style: italic;">
    “True cloud mastery isn’t memorizing theory—it’s the confidence earned by provisioning, breaking, troubleshooting, and orchestrating resilient systems with your own hands.”
  </p>
  <br />
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
