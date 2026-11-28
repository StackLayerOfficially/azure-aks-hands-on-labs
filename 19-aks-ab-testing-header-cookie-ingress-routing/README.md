# Azure AKS Dark Launches & Beta Testing: Header and Cookie Routing with NGINX

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

Random weighted canary splits still expose live customers to unverified releases and cause inconsistent page refreshes. In this lab, you lock public production users onto **Genesis Core (`v1`)** with `canary-weight: "0"` while deterministically routing internal QA teams and opted-in beta cohorts to **Nexus Edition (`v3`)** on `web.stacklayer.cyou` via custom HTTP request headers (`X-Release-Stage: beta`) and sticky session cookies (`stacklayer_cohort=always`), proving NGINX Ingress's exact **Header > Cookie > Weight** evaluation order.

> **Architecture Lifecycle Note:** Upstream `kubernetes/ingress-nginx` reached end-of-life in **March 2026** (with Microsoft's AKS managed NGINX support window concluding in **November 2026**). Engineering teams continue maintaining existing `kind: Ingress` dark-launch pipelines while migrating greenfield workloads to **Kubernetes Gateway API (`HTTPRoute` `matches.headers`)**.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Azure AKS Dark Launches & Beta Testing: Header and Cookie Routing with NGINX](https://stacklayer.blogspot.com/2026/11/azure-aks-dark-launches-beta-testing-header-cookie-nginx-routing.html)

---

## 📁 Directory Structure

```text
19-aks-ab-testing-header-cookie-ingress-routing/
├── README.md
└── kube-manifests/
    ├── 01-stacklayer-v1-prod-deployment.yaml  # Deploys stable Genesis Core (v1) production Pods (3 replicas)
    ├── 02-stacklayer-v1-prod-service.yaml     # Internal ClusterIP Service targeting Production (v1) Pods
    ├── 03-stacklayer-v3-beta-deployment.yaml  # Deploys dark-launch Nexus Edition (v3) beta cohort Pods (2 replicas)
    ├── 04-stacklayer-v3-beta-service.yaml     # Internal ClusterIP Service targeting Beta Cohort (v3) Pods
    ├── 05-stacklayer-prod-ingress.yaml        # Primary Production Ingress routing default live traffic on web.stacklayer.cyou to v1
    └── 06-stacklayer-beta-ingress.yaml        # Zero-weight Canary Ingress matching X-Release-Stage header and stacklayer_cohort cookie to v3
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

### Step-02: Deploy Production (`v1`) & Beta Cohort (`v3`) Workloads

```bash
kubectl apply \
  -f kube-manifests/01-stacklayer-v1-prod-deployment.yaml \
  -f kube-manifests/02-stacklayer-v1-prod-service.yaml \
  -f kube-manifests/03-stacklayer-v3-beta-deployment.yaml \
  -f kube-manifests/04-stacklayer-v3-beta-service.yaml

kubectl get pods,svc -l "app in (stacklayer-prod,stacklayer-beta)"
```

### Step-03: Apply Primary Production Ingress (`100% v1` Baseline)

```bash
kubectl apply -f kube-manifests/05-stacklayer-prod-ingress.yaml

sleep 5
curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
```

### Step-04: Configure Header & Sticky Cookie Beta Ingress (`canary-weight: "0"`)

```bash
kubectl apply -f kube-manifests/06-stacklayer-beta-ingress.yaml

kubectl get ingress
```

### Step-05: Test Deterministic Cohort Routing & Precedence (`Header > Cookie > Weight`)

```bash
# 1. Everyday Public User (no header, no cookie -> canary-weight: "0" keeps user on v1)
echo "Test 1 - Public User Baseline:"
curl -s -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1

# 2. Internal QA via Custom HTTP Header (X-Release-Stage: beta -> routes to v3)
echo "Test 2 - QA Header Routing (X-Release-Stage: beta):"
curl -s -H "Host: web.stacklayer.cyou" -H "X-Release-Stage: beta" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1

# 3. Beta User via Sticky Cookie (stacklayer_cohort=always -> routes to v3)
echo "Test 3 - Sticky Cookie Cohort (stacklayer_cohort=always):"
curl -s -H "Host: web.stacklayer.cyou" --cookie "stacklayer_cohort=always" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1

# 4. Cookie Opt-Out (stacklayer_cohort=never -> stays locked on v1)
echo "Test 4 - Cookie Opt-Out (stacklayer_cohort=never):"
curl -s -H "Host: web.stacklayer.cyou" --cookie "stacklayer_cohort=never" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1

# 5. Precedence Proof (Beta Header overrides Cookie Opt-Out: Header > Cookie > Weight)
echo "Test 5 - Precedence Proof (Beta Header + Cookie never):"
curl -s -H "Host: web.stacklayer.cyou" -H "X-Release-Stage: beta" --cookie "stacklayer_cohort=never" "http://$INGRESS_IP/" | grep -oi "Web Ver:[0-9]" | head -n 1
```

### Step-06: Verify HTTP Health & Cohort Latency

```bash
curl -s -o /dev/null -w "Public v1 Baseline: HTTP %{http_code} | Latency: %{time_total}s\n" \
  -H "Host: web.stacklayer.cyou" "http://$INGRESS_IP/"

curl -s -o /dev/null -w "Beta Header v3:     HTTP %{http_code} | Latency: %{time_total}s\n" \
  -H "Host: web.stacklayer.cyou" -H "X-Release-Stage: beta" "http://$INGRESS_IP/"

curl -s -o /dev/null -w "Beta Cookie v3:     HTTP %{http_code} | Latency: %{time_total}s\n" \
  -H "Host: web.stacklayer.cyou" --cookie "stacklayer_cohort=always" "http://$INGRESS_IP/"
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
  <h3>🌐 Targeted Dark Launches With Zero Blast Radius</h3>
  <p><i>“Test live in production for chosen cohorts—never guess with public traffic when precision routing leads the way.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
