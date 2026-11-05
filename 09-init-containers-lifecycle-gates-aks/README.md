# Kubernetes Init Containers on Azure Kubernetes Service (AKS)

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In microservice architectures, application containers frequently boot faster than their backing stateful dependencies. When a web service pod initializes before its PostgreSQL database, cache, or external messaging broker is ready to accept socket connections, the application immediately throws unhandled connection exceptions, floods logs with HTTP 500 errors, and enters an aggressive `CrashLoopBackOff` restart cycle.

This lab demonstrates how to implement deterministic pod startup gates using Kubernetes `initContainers` on Azure Kubernetes Service (AKS). By decoupling network readiness probing from primary application containers, your workloads remain blocked in an `Init:0/1` state until upstream TCP endpoints pass handshakes, guaranteeing zero failed socket attempts and reliable application startup.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Architecting Resilient Pod Startup Order on Azure AKS with Kubernetes Init Containers](https://stacklayer.blogspot.com/2026/10/azure-aks-kubernetes-init-containers-pod-startup-order.html)

---

## 📁 Directory Structure

```text
09-kubernetes-init-containers-on-aks/
├── README.md                                  # Complete hands-on lab documentation and architecture walk
└── kube-manifests/
    ├── 01-stacklayer-web-app.yaml             # Web deployment with init container TCP readiness check
    ├── 02-stacklayer-postgres-service.yaml     # Internal ClusterIP service for PostgreSQL DNS resolution
    └── 03-stacklayer-postgres-deployment.yaml  # Hardened enterprise PostgreSQL database workload
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

## 🚀 Hands-on Deployment & Verification Workflow

### Step 1: Deploy Web Workload with Init Container Probe

Deploy the web application deployment. The pod specification contains an `initContainers` block configured to probe `stacklayer-postgres-service:5432` using `nc -z -v -w3` before starting the primary application container:

```bash
# Deploy the web application workload
kubectl apply -f kube-manifests/01-stacklayer-web-app.yaml
```

### Step 2: Observe Blocking State and Inspect Init Container Logs

Verify that the web pods remain halted in an `Init:0/1` state while awaiting the database:

```bash
# Verify pod lifecycle status
kubectl get pods -l app=stacklayer-web
```

```text
NAME                                  READY   STATUS     RESTARTS   AGE
stacklayer-web-app-79888d8b67-42v6r   0/1     Init:0/1   0          25s
stacklayer-web-app-79888d8b67-mff2x   0/1     Init:0/1   0          25s
```

Inspect the init container stream to confirm active TCP probing:

```bash
# Stream init container logs
kubectl logs -l app=stacklayer-web -c wait-for-database -f
```

```text
Starting database readiness check...
Waiting for stacklayer-postgres-service:5432 to accept TCP connections...
nc: bad address 'stacklayer-postgres-service'
Waiting for stacklayer-postgres-service:5432 to accept TCP connections...
```

### Step 3: Deploy PostgreSQL Database ClusterIP Service

Create the internal ClusterIP service that establishes the stable cluster-internal DNS record (`stacklayer-postgres-service.default.svc.cluster.local`):

```bash
# Deploy internal ClusterIP service
kubectl apply -f kube-manifests/02-stacklayer-postgres-service.yaml

# Confirm service creation and check endpoints
kubectl get svc stacklayer-postgres-service
kubectl get endpoints stacklayer-postgres-service
```

```text
NAME                          ENDPOINTS           AGE
stacklayer-postgres-service   <none>              6s
```

### Step 4: Deploy PostgreSQL Database Workload

Launch the backend database container. Once the database process initializes and begins listening on port `5432`, the endpoint controller attaches its pod IP to the service:

```bash
# Deploy the PostgreSQL database deployment
kubectl apply -f kube-manifests/03-stacklayer-postgres-deployment.yaml
```

### Step 5: Verify Successful Initialization & Pod Transition

Monitor the live state transition across the namespace:

```bash
# Watch pod transitions in real time
kubectl get pods -w
```

```text
NAME                                              READY   STATUS            RESTARTS   AGE
stacklayer-postgres-deployment-64d84f88d7-k8qpl   1/1     Running           0          19s
stacklayer-web-app-79888d8b67-42v6r               0/1     PodInitializing   0          84s
stacklayer-web-app-79888d8b67-mff2x               0/1     PodInitializing   0          84s
stacklayer-web-app-79888d8b67-42v6r               1/1     Running           0          86s
stacklayer-web-app-79888d8b67-mff2x               1/1     Running           0          86s
```

Inspect chronological Kubelet events to confirm the primary container started only after the init container completed:

```bash
# Inspect pod lifecycle events
kubectl describe pod -l app=stacklayer-web
```

```text
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  92s   default-scheduler  Successfully assigned default/stacklayer-web-app-79888d8b67-42v6r to aks-nodepool1-vmss000000
  Normal  Pulled     90s   kubelet            Container image "stacklayer/init-container:v1.0" already present on machine
  Normal  Created    90s   kubelet            Created container wait-for-database
  Normal  Started    89s   kubelet            Started container wait-for-database
  Normal  Pulled     24s   kubelet            Container image "stacklayer/kubernetes:nginx-v1" already present on machine
  Normal  Created    23s   kubelet            Created container stacklayer-web-container
  Normal  Started    22s   kubelet            Started container stacklayer-web-container
```

### Step 6: End-to-End Verification & Health Inspection

Port-forward to the web application pod to verify end-to-end functionality:

```bash
# Port-forward the web application to localhost:8080
kubectl port-forward deployment/stacklayer-web-app 8080:80
```

Open another terminal or test with curl:

```bash
curl -I http://127.0.0.1:8080
```

```text
HTTP/1.1 200 OK
Server: nginx/1.25.4
Content-Type: text/html
```

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure subscription credits and prevent unnecessary cloud compute billing, select one of the following teardown options:

### Option A: Pause Cluster Nodes (Recommended for Ongoing Labs)

Suspends node pool VMs to cease compute billing immediately while preserving your control plane, network configuration, and manifests:

```bash
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Complete Deletion (Zero Cloud Cost)

Permanently removes the Resource Group, cluster, Virtual Network, and all underlying cloud resources:

```bash
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

<div align="center">
  <h3>🌐 Deterministic Pod Lifecycle & Dependency Orchestration</h3>
  <p><i>“In distributed systems, true resilience is not hoping dependencies are ready—it is architecting self-healing pods that guarantee prerequisites before executing a single line of business logic.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
