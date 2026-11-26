# Azure AKS Canary Deployments: Progressive Traffic Splitting with NGINX Ingress

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

Deploying a new application release to 100% of live users at once exposes production traffic to unexpected regressions, memory spikes, or latency degradation—even when Kubernetes readiness probes pass. In this lab, you decouple live traffic distribution from Pod replica counts using **Dual-Ingress Weighted Canary Routing** on **Azure Kubernetes Service (AKS)**, shifting live requests progressively (`20%` → `50%` → `100%`) across **Genesis Core (`v1`)** and **Velocity Edition (`v2`)** on `web.stacklayer.cyou` with instant zero-downtime `0%` rollback capability.

> **Architecture Lifecycle Note:** Upstream `kubernetes/ingress-nginx` reached end-of-life in **March 2026** (with Microsoft's AKS managed NGINX support window concluding in **November 2026**). Cloud teams continue operating and migrating legacy `kind: Ingress` canary pipelines in production while adopting **Kubernetes Gateway API (`HTTPRoute` `backendRefs.weight`)** for greenfield AKS clusters.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Azure AKS Canary Deployments: Progressive Traffic Splitting with NGINX Ingress](https://stacklayer.blogspot.com/2026/11/azure-aks-canary-deployments-progressive-traffic-splitting-nginx-ingress.html)

---

## 📁 Directory Structure

```text
18-kubernetes-ingress-canary-deployments-aks/
├── README.md
└── kube-manifests/
    ├── 01-stacklayer-v1-prod-deployment.yaml    # Deploys stable Genesis Core (v1) production Pods (3 replicas)
    ├── 02-stacklayer-v1-prod-service.yaml       # Internal ClusterIP Service targeting Production (v1) Pods
    ├── 03-stacklayer-v2-canary-deployment.yaml  # Deploys canary candidate Velocity Edition (v2) Pods (2 replicas)
    ├── 04-stacklayer-v2-canary-service.yaml     # Internal ClusterIP Service targeting Canary (v2) Pods
    ├── 05-stacklayer-prod-ingress.yaml          # Primary Production Ingress routing 100% baseline traffic to v1
    └── 06-stacklayer-canary-ingress.yaml        # Companion Canary Ingress splitting weighted traffic to v2 on web.stacklayer.cyou
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

## 🚀 Hands-On Execution Steps

### Step-01: Install or Verify Official NGINX Ingress Controller

> **Official Source Attribution:** Uses the official **`controller-v1.15.1`** Cloud Provider manifest from the [kubernetes/ingress-nginx Release Page](https://github.com/kubernetes/ingress-nginx/releases/tag/controller-v1.15.1). Always inspect the [NGINX Ingress Azure Deployment Docs](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#azure) and [Canary Annotations Reference](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary) for version updates and Helm deployment options.

```bash
# 1. Apply the official NGINX Ingress Controller manifest
kubectl apply \
  -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml

# 2. Wait for the controller deployment and validating webhook to become Ready
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx

# 3. Capture the Azure LoadBalancer External Public IP
kubectl get svc ingress-nginx-controller -n ingress-nginx
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Active NGINX Ingress Public IP: $INGRESS_IP"
```

### Step-02: Deploy Production (`v1`) & Canary (`v2`) Workloads

```bash
kubectl apply \
  -f kube-manifests/01-stacklayer-v1-prod-deployment.yaml \
  -f kube-manifests/02-stacklayer-v1-prod-service.yaml \
  -f kube-manifests/03-stacklayer-v2-canary-deployment.yaml \
  -f kube-manifests/04-stacklayer-v2-canary-service.yaml

kubectl get pods,svc -l "app in (stacklayer-prod,stacklayer-canary)"
```

### Step-03: Apply Primary Production Ingress (`100% v1` Baseline)

```bash
kubectl apply -f kube-manifests/05-stacklayer-prod-ingress.yaml

sleep 5
for i in {1..5}; do
  curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
done
```

### Step-04: Activate `80/20` Weighted Canary Traffic Split

```bash
kubectl apply -f kube-manifests/06-stacklayer-canary-ingress.yaml

kubectl get ingress
```

### Step-05: Progressive Rollout (`20%` → `50%` → `100%`) & Instant `0%` Rollback

```bash
# Phase 1: Sample 20 live requests at 20% Canary Weight (~16 Web Ver:1 / ~4 Web Ver:2)
echo "--- Phase 1: 20% Canary Weight Distribution (20 requests) ---"
for i in {1..20}; do
  curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
done | sort | uniq -c

# Phase 2: Promote Canary Weight dynamically to 50% (~10 / ~10)
kubectl annotate ingress stacklayer-canary-ingress \
  nginx.ingress.kubernetes.io/canary-weight="50" --overwrite

sleep 3
echo "--- Phase 2: 50% Canary Weight Distribution (20 requests) ---"
for i in {1..20}; do
  curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
done | sort | uniq -c

# Phase 3: Promote Canary Weight to 100% full cutover (100% Web Ver:2)
kubectl annotate ingress stacklayer-canary-ingress \
  nginx.ingress.kubernetes.io/canary-weight="100" --overwrite

sleep 3
echo "--- Phase 3: 100% Full Canary Cutover (20 requests) ---"
for i in {1..20}; do
  curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
done | sort | uniq -c

# Phase 4: Instant Emergency Rollback back to 0% Weight (100% Web Ver:1)
kubectl annotate ingress stacklayer-canary-ingress \
  nginx.ingress.kubernetes.io/canary-weight="0" --overwrite

sleep 3
echo "--- Phase 4: Emergency Rollback at 0% Weight (20 requests) ---"
for i in {1..20}; do
  curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
done | sort | uniq -c

# Restore 80/20 split for latency and browser checks
kubectl annotate ingress stacklayer-canary-ingress \
  nginx.ingress.kubernetes.io/canary-weight="20" --overwrite
```

### Step-06: Verify HTTP Health & Response Latency

```bash
for i in {1..5}; do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code} | Latency: %{time_total}s\n" \
    -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/"
done
```

---

## 🧹 Cost Safeguards & Cleanup

Remove all lab manifests cleanly from the cluster first:

```bash
kubectl delete -f kube-manifests/
```

### Option A: Pause Cluster Nodes (Recommended If Practicing Later Labs)

Deallocates AKS worker node VMs to halt hourly VM compute charges while preserving your cluster state:

```bash
az aks stop \
  --resource-group stacklayer-aks-rg \
  --name stacklayer-aks-cluster
```

### Option B: Complete Deletion (Zero Cost)

Permanently removes the AKS cluster, LoadBalancer Public IP, and all resources inside `stacklayer-aks-rg`:

```bash
az group delete \
  --name stacklayer-aks-rg \
  --yes \
  --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <h3>🌐 Progressive Delivery Without Production Risk</h3>
  <p><i>“Decouple traffic exposure from pod replicas—validate in small slices, promote with confidence, and roll back in seconds.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
