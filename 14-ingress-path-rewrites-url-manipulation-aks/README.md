# Strip and Rewrite Ingress URL Paths on Azure AKS with NGINX

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In modern cloud-native architectures, containerized web applications and microservices are built to serve HTTP traffic directly from the root path (`/`). Hardcoding path prefixes like `/app1` or `/service-b` into application code creates tight coupling, breaks local development, and complicates multi-tenant deployments.

However, when consolidating multiple microservices behind a single Azure Public IP using a Kubernetes Ingress controller, traffic must be routed using path prefixes. Without URL manipulation, forwarding requests like `/app1/index.html` causes backend containers to throw **HTTP 404 Not Found** errors because the container web root has no `/app1` directory.

This hands-on operations lab demonstrates how to resolve the 404 trap using **NGINX Ingress regex capture groups and URL rewriting annotations**. You will route live traffic across two independent corporate microservices—**Genesis Core** (`Web Ver:1`) and **Velocity Edition** (`Web Ver:2`)—stripping public prefixes at the ingress boundary while preserving dynamic sub-paths transparently to the backend pods.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Strip and Rewrite Ingress URL Paths on Azure AKS with NGINX](https://stacklayer.blogspot.com/2026/11/strip-rewrite-ingress-url-paths-azure-aks-nginx.html)

---

## 📁 Directory Structure

```text
14-ingress-path-rewrites-url-manipulation-aks/
├── README.md                                    # Comprehensive hands-on documentation
└── kube-manifests/
    ├── 01-genesis-core-app1-deployment.yaml     # Genesis Core microservice deployment (Web Ver:1)
    ├── 02-genesis-core-app1-service.yaml        # Genesis Core private ClusterIP service
    ├── 03-velocity-edition-app2-deployment.yaml   # Velocity Edition microservice deployment (Web Ver:2)
    ├── 04-velocity-edition-app2-service.yaml      # Velocity Edition private ClusterIP service
    └── 05-stacklayer-ingress-rewrite-rules.yaml # NGINX Ingress rules with regex path rewrite ($2)
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

## 🚀 Hands-on Lab Execution

### Step 1: Confirm or Install NGINX Ingress Controller

Verify that the NGINX Ingress Controller is running on your AKS cluster and has been allocated an Azure Public External IP:

```bash
# Verify running NGINX Ingress Controller
kubectl get service ingress-nginx-controller -n ingress-nginx
```

*(If not already installed, deploy the current official release directly from the official Kubernetes repository:)*

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

> 💡 **Official Documentation Reference:** This deployment manifest is maintained directly by the official [Kubernetes Ingress-NGINX project (Azure Deployment Guide)](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#azure). You can safely deploy this verified release, or visit the [Official Ingress-NGINX Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/#azure) to explore newer release tags or Helm deployment options.


---

### Step 2: Deploy Genesis Core (App 1) & Private ClusterIP Service

Deploy the first corporate microservice rendering **Genesis Core** (`Web Ver:1`):

```bash
# Deploy Genesis Core pod replicas
kubectl apply -f kube-manifests/01-genesis-core-app1-deployment.yaml

# Expose Genesis Core via private ClusterIP
kubectl apply -f kube-manifests/02-genesis-core-app1-service.yaml
```

---

### Step 3: Deploy Velocity Edition (App 2) & Private ClusterIP Service

Deploy the second corporate microservice rendering **Velocity Edition** (`Web Ver:2`):

```bash
# Deploy Velocity Edition pod replicas
kubectl apply -f kube-manifests/03-velocity-edition-app2-deployment.yaml

# Expose Velocity Edition via private ClusterIP
kubectl apply -f kube-manifests/04-velocity-edition-app2-service.yaml
```

Verify that all backend microservice pods and internal endpoints are healthy:

```bash
kubectl get pods -l app=stacklayer-app1
kubectl get pods -l app=stacklayer-app2
kubectl get svc stacklayer-app1-service stacklayer-app2-service
```

---

### Step 4: Deploy Ingress Rules with Regex URL Rewriting

Apply the ingress routing manifest configured with `use-regex: "true"` and `rewrite-target: /$2`:

```bash
kubectl apply -f kube-manifests/05-stacklayer-ingress-rewrite-rules.yaml
```

#### How the Regex Capture Group Operates:
* `path: /app1(/|$)(.*)`:
  * `/app1` matches the public entry prefix.
  * Capture Group 1 `(/|$)` matches the boundary (trailing slash or end of line).
  * Capture Group 2 `(.*)` captures all subsequent characters and query parameters.
* `nginx.ingress.kubernetes.io/rewrite-target: /$2`:
  * Replaces the URI with root `/` followed by group 2, stripping `/app1` entirely before forwarding the packet to the pod.

---

### Step 5: Inspect Ingress Status and Address Binding

Confirm that the Ingress controller has synchronized the rules and bound your Azure Public IP:

```bash
kubectl get ingress stacklayer-ingress-rewrite
kubectl describe ingress stacklayer-ingress-rewrite
```

---

### Step 6: Dual-Layer End-to-End Verification

#### Layer 1: CLI Terminal Smoke Tests
Dynamically extract your public load balancer IP and run automated smoke tests to confirm URL prefix stripping:

```bash
# Capture the active public Ingress IP
INGRESS_IP=$(kubectl get ingress stacklayer-ingress-rewrite -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Testing Ingress on Public IP: $INGRESS_IP"

# Test App 1 with URL prefix stripping (/app1)
curl -s http://$INGRESS_IP/app1 | grep -oi "Web Ver:[0-9]" | head -n 1

# Test App 2 with URL prefix stripping (/app2)
curl -s http://$INGRESS_IP/app2 | grep -oi "Web Ver:[0-9]" | head -n 1
```

Expected Terminal Output:
```text
Testing Ingress on Public IP: 20.219.229.217
Web Ver:1
Web Ver:2
```

#### Layer 2: Browser Web UI Verification
Open your browser and navigate directly to your cluster's public entry points:
* `http://<INGRESS-IP>/app1` &rarr; Renders **Genesis Core** corporate portal with `Web Ver:1` badge.
* `http://<INGRESS-IP>/app2` &rarr; Renders **Velocity Edition** enterprise portal with `Web Ver:2` badge.

Both microservices serve requests seamlessly from a single Azure Public IP address with zero application code modifications.

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure subscription balance and prevent unnecessary compute charges, select one of the following teardown options:

### Option A: Pause Cluster Nodes (Recommended If Continuing Later Labs)
Suspends worker node Virtual Machine Scale Sets (VMSS) to stop compute billing while retaining cluster configuration and Ingress state:

```bash
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Complete Deletion (Zero Ongoing Cost)
Permanently deletes the entire resource group, including all clusters, public IPs, and load balancers:

```bash
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <h3>🌐 Strip and Rewrite Ingress URL Paths on Azure AKS</h3>
  <p><i>“True microservice decoupling begins at the ingress boundary: clean public URLs, zero backend code compromises.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>

