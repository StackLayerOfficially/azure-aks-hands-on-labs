# 01: Deploy Your First App on Azure Kubernetes Service (AKS)

Declarative Kubernetes manifests for provisioning an AKS cluster, connecting via Azure CLI, and deploying a sample containerized web application exposed publicly via an Azure Standard Load Balancer.

---

### 📖 Complete Architecture Guide
> **Read the full step-by-step tutorial with detailed command breakdowns on Stack Layer:**  
> 👉 [**Deploy Your First Containerized App on Azure Kubernetes Service (AKS) Step-by-Step**](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)

---

## 📁 Manifests & Files in this Directory

```text
01-aks-cluster-deploy-app/
├── README.md
└── kube-manifests/
    ├── 0A-Deployment.yml   # Nginx container deployment (replicas, ports, and resource limits)
    └── 0B-Service.yml      # Azure Standard Load Balancer (public IP on port 80)
```

---

## 🚀 Quick Execution Guide

### 1. Authenticate Local CLI with Your AKS Cluster
```bash
az aks get-credentials --resource-group aks-rg --name aks-cluster --overwrite-existing
```

### 2. Verify Cluster Nodes
```bash
kubectl get nodes -o wide
```

### 3. Deploy All Manifests
Deploy both the Deployment and Load Balancer Service in a single batch:
```bash
kubectl apply -f kube-manifests/
```

### 4. Verify Rollout & Get Public External IP
```bash
# Verify pods are running
kubectl get pods

# Watch for the Azure Public IP assignment
kubectl get service web-app-service -w
```
Once the `EXTERNAL-IP` appears, open `http://<EXTERNAL-IP>` in your browser to verify access.

---

## 🧹 Teardown & Cost Cleanup

To prevent ongoing cloud charges after testing:

```bash
# 1. Delete deployed Kubernetes resources
kubectl delete -f kube-manifests/

# 2. Pause cluster nodes to halt VM compute billing (Resume anytime with 'az aks start')
az aks stop --resource-group aks-rg --name aks-cluster
```

---

<div align="center">
  <sub>Part of the <a href="https://github.com/StackLayerOfficially/azure-aks-hands-on-labs">Azure AKS Hands-on Labs Series</a> by <b>Stack Layer</b></sub>
</div>
