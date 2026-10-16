# Kubernetes Service Networking on Azure AKS: ClusterIP, NodePort & LoadBalancer

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In modern cloud-native architectures, container pods are ephemeral and mortal. Pods frequently restart, scale out, or reschedule onto different cluster nodes, receiving brand-new dynamic IP addresses every time. If external clients or internal microservices tried communicating directly with these volatile pod IPs, applications would suffer severe traffic breaks.

A **Kubernetes Service** solves this challenge by providing an enduring, stable virtual IP address, an internal CoreDNS hostname, and intelligent layer-4 load balancing across healthy container replicas. 

In this hands-on lab, you will master the three foundational Kubernetes service types on **Azure Kubernetes Service (AKS)**: creating private internal microservices with **ClusterIP**, exposing node-level testing gateways with **NodePort**, and provisioning production-grade internet ingress via an automated **Azure Standard Load Balancer**.

---

## 📖 Companion Blog Guide
For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:
* **Official Guide:** [Architecting Kubernetes Service Networking on Azure AKS: ClusterIP, NodePort & LoadBalancer](https://stacklayer.blogspot.com/2026/10/azure-aks-kubernetes-service-types-clusterip-nodeport-loadbalancer.html)

---

## 📁 Directory Structure

```text
07-kubernetes-service-types-clusterip-nodeport-loadbalancer/
├── README.md
└── kube-manifests/
    ├── 01-backend-deployment.yaml          # 2-replica NGINX backend workload
    ├── 02-backend-clusterip-service.yaml   # Private internal service with CoreDNS
    ├── 03-backend-nodeport-service.yaml    # NodePort access on port 31080 across all nodes
    └── 04-backend-loadbalancer-service.yaml # Public Azure Standard Load Balancer endpoint
```

---

## 🎯 Learning Objectives

By completing this lab, you will:
1. Understand how Kubernetes Services decouple client traffic from volatile, ephemeral pod IP addresses.
2. Deploy a redundant 2-replica backend deployment and isolate it internally using a **ClusterIP** service.
3. Verify cluster-internal service discovery and DNS resolution using built-in **CoreDNS** and an ephemeral client pod.
4. Expose workloads across all physical AKS worker nodes using a dedicated static **NodePort** (port `31080`).
5. Trigger automated cloud provisioning of a public IP and an **Azure Standard Load Balancer** using `type: LoadBalancer`.
6. Inspect live endpoint pools (`kubectl get endpoints`) and verify layer-4 traffic routing via browser and curl.
7. Clean up cloud resources and halt Azure billing using automated teardown and pause commands.

---

## 📋 Prerequisites

* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS)](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)).
* **Azure CLI (`az`)** and **Kubectl** installed and authenticated.

Connect your local terminal to your running cluster:
```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity and healthy nodes
kubectl cluster-info
kubectl get nodes
```

---

## 🚀 Step-by-Step Hands-On Guide

### Step 1: Deploy Backend Workload & Private ClusterIP Service

Deploy an NGINX microservice with two replicas, then expose it privately inside the cluster using `type: ClusterIP`.

#### `kube-manifests/01-backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-backend-deployment
  labels:
    app: stacklayer-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: stacklayer-backend
  template:
    metadata:
      labels:
        app: stacklayer-backend
    spec:
      containers:
      - name: backend-api
        image: stacklayer/kubernetes:nginx-v1
        ports:
        - containerPort: 80
```

#### `kube-manifests/02-backend-clusterip-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-backend-service
  labels:
    app: stacklayer-backend
spec:
  type: ClusterIP
  selector:
    app: stacklayer-backend
  ports:
  - port: 80
    targetPort: 80
    name: http
```

Apply both manifests to AKS:
```bash
# Deploy the backend deployment and internal ClusterIP service
kubectl apply -f kube-manifests/01-backend-deployment.yaml
kubectl apply -f kube-manifests/02-backend-clusterip-service.yaml

# Confirm the ClusterIP service is running with no external IP
kubectl get svc stacklayer-backend-service
```

---

### Step 2: Verify Internal CoreDNS Service Discovery

Test that pods inside the cluster can reach the backend service using its clean hostname rather than an IP address:

```bash
# Spin up a temporary interactive curl diagnostic container
kubectl run curl-client --image=curlimages/curl -it --rm -- restart=Never -- sh

# Inside the curl container prompt, test DNS resolution and HTTP response:
curl http://stacklayer-backend-service
nslookup stacklayer-backend-service

# Type 'exit' to terminate the diagnostic container
exit
```

Inspect the live pod endpoint pool dynamically maintained by Kubernetes:
```bash
# View active pod target IPs bound to the service
kubectl get endpoints stacklayer-backend-service
```

---

### Step 3: Expose Workloads Across Worker Nodes with NodePort

Create a `NodePort` service that opens static port `31080` across every physical Azure VM in your cluster node pool:

#### `kube-manifests/03-backend-nodeport-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-nodeport-service
  labels:
    app: stacklayer-backend
spec:
  type: NodePort
  selector:
    app: stacklayer-backend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31080
    name: http
```

Apply the NodePort service and test connectivity across nodes:
```bash
# Apply NodePort service
kubectl apply -f kube-manifests/03-backend-nodeport-service.yaml

# Retrieve internal worker node IP addresses
kubectl get nodes -o wide

# Test internal node routing using a temporary test pod
kubectl run test-nodeport --image=curlimages/curl -it --rm -- restart=Never -- curl http://<NODE_INTERNAL_IP>:31080
```

---

### Step 4: Expose Workloads Publicly via Azure Standard Load Balancer

Trigger the Azure cloud controller to provision an external public IP and configure an Azure Standard Load Balancer:

#### `kube-manifests/04-backend-loadbalancer-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-public-service
  labels:
    app: stacklayer-backend
spec:
  type: LoadBalancer
  selector:
    app: stacklayer-backend
  ports:
  - port: 80
    targetPort: 80
    name: http
```

Deploy the service and stream real-time public IP allocation:
```bash
# Deploy public Load Balancer service
kubectl apply -f kube-manifests/04-backend-loadbalancer-service.yaml

# Watch until EXTERNAL-IP changes from <pending> to a live public IP
kubectl get svc stacklayer-public-service -w
```
*(Press `Ctrl + C` once the public IP address is assigned).*

---

### Step 5: Live Ingress Verification & Azure Portal Inspection

1. Query your public endpoint directly from your local workstation:
   ```bash
   PUBLIC_IP=$(kubectl get svc stacklayer-public-service -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
   curl http://$PUBLIC_IP
   ```
2. Open your web browser and navigate to `http://<YOUR_PUBLIC_IP>` to see the active web application.
3. In the [Azure Portal](https://portal.azure.com/), open your cluster node resource group (`MC_stacklayer-aks-rg_stacklayer-aks-cluster_<region>`), locate the **kubernetes** Standard Load Balancer, and inspect the automated **Frontend IP configurations** and **Load balancing rules**.

---

## 🧹 Cost Safeguards & Cleanup

Public IP addresses and Azure Load Balancers incur continuous consumption charges while provisioned. Always tear down resources when your lab session is complete:

### 1. Delete Kubernetes Services & Workloads
```bash
# Release the public IP and delete all service abstractions
kubectl delete -f kube-manifests/04-backend-loadbalancer-service.yaml
kubectl delete -f kube-manifests/03-backend-nodeport-service.yaml
kubectl delete -f kube-manifests/02-backend-clusterip-service.yaml

# Delete backend deployment
kubectl delete -f kube-manifests/01-backend-deployment.yaml

# Confirm cluster services are clear
kubectl get svc
```

### 2. Manage AKS Cluster Lifecycles
* **Option A: Stop AKS Cluster (Halts VM compute billing, preserves cluster)**:
  ```bash
  az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
  ```
* **Option B: Delete Resource Group (Permanent Zero-Cost Teardown)**:
  ```bash
  az group delete --name stacklayer-aks-rg --yes --no-wait
  ```

---

## 🤝 Community & Support
* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <br />
  <h3>🌐 Connect the Nodes. Architect for Scale.</h3>
  <p style="max-width: 580px; line-height: 1.6; font-style: italic;">
    “Ephemeral pods come and go, but robust services keep the traffic flowing. True cloud resilience lives in the network abstraction.”
  </p>
  <br />
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
