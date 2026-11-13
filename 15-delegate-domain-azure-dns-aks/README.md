# Delegate Custom Domain Names to Azure DNS for Kubernetes Workloads

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

Raw public IPv4 addresses are brittle for production traffic and incompatible with clean microservice branding. In this operational runbook, you will delegate a custom domain (`stacklayer.cyou`) from an external registrar to **Azure DNS**, map DNS `A` records (`@`, `app1`, `app2`) to your **Azure Kubernetes Service (AKS)** NGINX Ingress Public IP (`135.235.181.105`), and route host-based traffic to **Nexus Edition** (`stacklayer/kubernetes:nginx-v3`) and **Prism Edition** (`stacklayer/kubernetes:nginx-v4`) workloads.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Delegate Custom Domain Names to Azure DNS for Kubernetes Workloads](https://stacklayer.blogspot.com/2026/11/delegate-custom-domain-names-azure-dns-kubernetes-workloads.html)

---

## 📁 Directory Structure

```text
15-delegate-domain-azure-dns-aks/
├── README.md
└── kube-manifests/
    ├── 01-stacklayer-app1-deployment.yaml   # App 1 Deployment running Nexus Edition (stacklayer/kubernetes:nginx-v3)
    ├── 02-stacklayer-app1-service.yaml      # Internal ClusterIP Service for App 1 workload
    ├── 03-stacklayer-app2-deployment.yaml   # App 2 Deployment running Prism Edition (stacklayer/kubernetes:nginx-v4)
    ├── 04-stacklayer-app2-service.yaml      # Internal ClusterIP Service for App 2 workload
    └── 05-stacklayer-domain-ingress.yaml    # Host-based NGINX Ingress routing rules for stacklayer.cyou domains
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

## 🚀 Step-by-Step Deployment & DNS Delegation Runbook

### 1. Create the Public Azure DNS Zone & Retrieve Name Servers

Provision a public DNS Zone for your custom domain (`stacklayer.cyou`) inside `stacklayer-aks-rg`:

```bash
# Create the public Azure DNS Zone
az network dns zone create \
  --resource-group stacklayer-aks-rg \
  --name stacklayer.cyou

# Retrieve the 4 authoritative Azure Name Servers
az network dns zone show \
  --resource-group stacklayer-aks-rg \
  --name stacklayer.cyou \
  --query "nameServers" \
  -o tsv
```

> **Azure Portal Check:** Open `portal.azure.com` → search **DNS zones** → select **`stacklayer.cyou`** → inspect the **Overview** blade to view the four assigned Azure Name Servers and default `NS` / `SOA` records.

---

### 2. Delegate Domain at Your Registrar & Verify NS Propagation

Update your domain registrar's **Nameservers** setting to **Custom DNS** using the 4 Azure Name Servers returned above, then verify global delegation:

```bash
nslookup -type=NS stacklayer.cyou
```

---

### 3. Install or Verify the Official NGINX Ingress Controller

This lab uses the active official NGINX Ingress Controller release (**`controller-v1.15.1`**). Verify newer tags or Helm options in the official [kubernetes/ingress-nginx Azure Deployment Docs](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#azure).

```bash
# Install NGINX Ingress Controller (skip if already running on your cluster)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml

# Verify pods and capture LoadBalancer External IP
kubectl get pods -n ingress-nginx
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "NGINX Ingress Public IP: $INGRESS_IP"
```

---

### 4. Deploy Branded Stack Layer Microservices

Deploy **Nexus Edition** (`stacklayer/kubernetes:nginx-v3`) and **Prism Edition** (`stacklayer/kubernetes:nginx-v4`) with internal `ClusterIP` Services:

```bash
kubectl apply -f kube-manifests/01-stacklayer-app1-deployment.yaml \
  -f kube-manifests/02-stacklayer-app1-service.yaml \
  -f kube-manifests/03-stacklayer-app2-deployment.yaml \
  -f kube-manifests/04-stacklayer-app2-service.yaml

kubectl get pods,svc
```

---

### 5. Map Azure DNS `A` Records to the NGINX Ingress Public IP

Bind the apex (`@`), `app1`, and `app2` records to your LoadBalancer IP (`135.235.181.105`):

```bash
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

az network dns record-set a add-record --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou --record-set-name "@" --ipv4-address $INGRESS_IP
az network dns record-set a add-record --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou --record-set-name "app1" --ipv4-address $INGRESS_IP
az network dns record-set a add-record --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou --record-set-name "app2" --ipv4-address $INGRESS_IP

# Verify all A records in CLI
az network dns record-set a list --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou -o table
```

> **Azure Portal Check:** Open **DNS zones → `stacklayer.cyou` → DNS management → Recordsets** to confirm `@`, `app1`, and `app2` Type `A` records point to `135.235.181.105` with TTL `3600`.

---

### 6. Apply Host-Based NGINX Ingress Rules

```bash
kubectl apply -f kube-manifests/05-stacklayer-domain-ingress.yaml
kubectl get ingress stacklayer-domain-ingress
```

---

## ✅ End-to-End Verification

Validate live DNS resolution and HTTP host-based microservice routing:

```bash
# 1. Confirm A-record resolution
nslookup app1.stacklayer.cyou
nslookup app2.stacklayer.cyou

# 2. Smoke-test HTTP host routing cleanly
curl -s http://app1.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1
curl -s http://app2.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1
curl -s http://stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1
```

Expected Terminal Output:

```text
Name:    app1.stacklayer.cyou
Address: 135.235.181.105

Name:    app2.stacklayer.cyou
Address: 135.235.181.105

Web Ver:3
Web Ver:4
Web Ver:3
```

Open `http://app1.stacklayer.cyou` (**Nexus Edition** — `Web Ver:3`) and `http://app2.stacklayer.cyou` (**Prism Edition** — `Web Ver:4`) in your browser to confirm live rendering.

---

## 🧹 Cost Safeguards & Cleanup

### Option A: Pause Cluster Nodes (Keep DNS Zone Delegation Intact)

Halts AKS worker node VM billing while preserving your cluster state and Azure DNS zone (~$0.50/month for the DNS zone if retained):

```bash
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

*(Tip: To reduce AKS cost to $0.00 while still preserving your Azure DNS Zone and Name Servers, run `az aks delete --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --yes --no-wait` instead of deleting the entire resource group.)*

### Option B: Complete Deletion (Zero Ongoing Cost)

Permanently deletes the entire resource group—including AKS, LoadBalancer IP, and the Azure DNS Zone:

```bash
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

<div align="center">
  <h3>🌐 Own Your DNS, Command Your Edge</h3>
  <p><i>“Resilient cloud architectures turn ephemeral cluster IPs into timeless, human-friendly digital gateways.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
