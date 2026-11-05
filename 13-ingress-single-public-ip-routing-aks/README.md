# Route Multiple Microservices Through a Single Public IP on Azure AKS with NGINX Ingress

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In cloud-native architectures, exposing every microservice using a dedicated cloud `LoadBalancer` service quickly becomes cost-prohibitive and operationally fragmented. In Microsoft Azure, each LoadBalancer service provisions an independent Azure Public IP address and cloud load balancer resource, creating recurring hourly compute costs and dispersed network endpoints.

In contrast, a **Kubernetes Ingress** acts as a centralized Layer 7 reverse proxy gateway. Operating under a **single public IP address**, it intelligently routes incoming HTTP and HTTPS traffic across dozens of internal microservices based on declarative URL paths or domain hostnames.

In this hands-on lab, you will deploy the official NGINX Ingress Controller on Azure Kubernetes Service (AKS), isolate two distinct corporate microservices—**Genesis Core** (`Web Ver:1`) and **Velocity Edition** (`Web Ver:2`)—behind internal ClusterIP services, author declarative path-based routing rules, and perform Dual-Layer end-to-end verification.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Route Multiple Microservices Through a Single Public IP on Azure AKS with NGINX Ingress](https://stacklayer.blogspot.com/2026/11/route-microservices-single-public-ip-azure-aks-nginx-ingress.html)

---

## 📁 Directory Structure

```text
13-ingress-single-public-ip-routing-aks/
├── README.md                                # Comprehensive hands-on documentation
└── kube-manifests/
    ├── 01-stacklayer-app1-deployment.yaml   # Genesis Core microservice deployment (Web Ver:1)
    ├── 02-stacklayer-app1-service.yaml      # Genesis Core private internal ClusterIP service
    ├── 03-stacklayer-app2-deployment.yaml   # Velocity Edition microservice deployment (Web Ver:2)
    ├── 04-stacklayer-app2-service.yaml      # Velocity Edition private internal ClusterIP service
    └── 05-stacklayer-ingress-rules.yaml     # Declarative Ingress Layer 7 path routing rules
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

*(If not already installed, deploy the current official release directly from the official Kubernetes SIG Network repository:)*

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

> 💡 **Official Documentation Reference:** This deployment manifest is maintained directly in the official [Kubernetes Ingress-NGINX Azure Deployment Documentation](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#azure).

---

### Step 2: Deploy Genesis Core (App 1) & Private ClusterIP Service

Deploy the first corporate microservice rendering **Genesis Core** (`Web Ver:1`):

```bash
# Deploy Genesis Core pod replicas
kubectl apply -f kube-manifests/01-stacklayer-app1-deployment.yaml

# Expose Genesis Core via private ClusterIP
kubectl apply -f kube-manifests/02-stacklayer-app1-service.yaml
```

---

### Step 3: Deploy Velocity Edition (App 2) & Private ClusterIP Service

Deploy the second corporate microservice rendering **Velocity Edition** (`Web Ver:2`):

```bash
# Deploy Velocity Edition pod replicas
kubectl apply -f kube-manifests/03-stacklayer-app2-deployment.yaml

# Expose Velocity Edition via private ClusterIP
kubectl apply -f kube-manifests/04-stacklayer-app2-service.yaml
```

Verify that both microservice deployments and internal endpoints are healthy:

```bash
kubectl get pods -l app=stacklayer-app1
kubectl get pods -l app=stacklayer-app2
kubectl get svc stacklayer-app1-service stacklayer-app2-service
```

---

### Step 4: Author and Apply Ingress Path-Based Routing Rules

Apply the declarative Ingress manifest mapping incoming public paths (`/app1` and `/app2`) to their corresponding internal ClusterIP services with automatic URI rewriting:

```bash
kubectl apply -f kube-manifests/05-stacklayer-ingress-rules.yaml
```

#### Routing Mechanics:
* Incoming requests targeting `http://<INGRESS-IP>/app1` route internally to `stacklayer-app1-service:80`.
* Incoming requests targeting `http://<INGRESS-IP>/app2` route internally to `stacklayer-app2-service:80`.
* The `rewrite-target: /` annotation ensures backend NGINX pods receive requests at root `/` without path pollution.

---

### Step 5: Inspect Ingress Status and Address Binding

Confirm that Kubernetes has processed the routing manifest and assigned the Azure Public IP:

```bash
kubectl get ingress stacklayer-ingress-routing
kubectl describe ingress stacklayer-ingress-routing
```

---

### Step 6: Dual-Layer End-to-End Verification

#### Layer 1: CLI Terminal Smoke Tests
Capture your cluster's public Ingress IP dynamically and execute automated smoke tests:

```bash
# Capture the active public Ingress IP
INGRESS_IP=$(kubectl get ingress stacklayer-ingress-routing -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Testing Ingress on Public IP: $INGRESS_IP"

# Test App 1 Routing (/app1)
curl -s http://$INGRESS_IP/app1 | grep -i "Web Ver"

# Test App 2 Routing (/app2)
curl -s http://$INGRESS_IP/app2 | grep -i "Web Ver"
```

Expected Terminal Output:
```text
Testing Ingress on Public IP: 20.124.88.192
        <span>Web Ver:1</span>
        <span class="telemetry-value" style="color: #fda4af;">Web Ver:1</span>
        <span>Web Ver:2</span>
        <span class="telemetry-value" style="color: #6ee7b7;">Web Ver:2</span>
```

#### Layer 2: Browser Visual Verification
Open your browser and navigate directly to your cluster's public entry points:
* `http://<INGRESS-IP>/app1` &rarr; Renders **Genesis Core** corporate portal (Neon Coral theme, active `Web Ver:1` indicator).
* `http://<INGRESS-IP>/app2` &rarr; Renders **Velocity Edition** enterprise portal (Electric Neon Mint theme, active `Web Ver:2` indicator).

Both independent microservices serve traffic flawlessly through a single Azure Public IP address with zero external load balancer duplication!

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure subscription balance and prevent unnecessary compute charges, select one of the following teardown options:

### Option A: Pause Cluster Nodes (Recommended If Continuing Later Labs)
Suspends worker node Virtual Machine Scale Sets (VMSS) so compute billing stops immediately while preserving cluster state and Ingress configuration:

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
  <h3>🌐 Consolidated Layer 7 Routing on Azure AKS</h3>
  <p><i>“Consolidate your cloud perimeter: one public entrance, infinite microservice pathways, zero wasted cloud expenditure.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>


