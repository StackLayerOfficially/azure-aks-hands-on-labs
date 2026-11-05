# DaemonSets on Azure AKS: Running an Agent on Every Worker Node

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In cloud-native operations, production clusters require system-level agents to run continuously across every physical and virtual machine. Whether collecting host log streams, gathering hardware metrics, or enforcing network security rules, you cannot rely on manual replica counts. This lab demonstrates how to architect, deploy, and verify Kubernetes **DaemonSets** on **Azure Kubernetes Service (AKS)** to guarantee that exactly one telemetry agent pod runs on every active worker node in your cluster.

---

## 📖 Companion Blog Guide

For complete architectural breakdowns, component comparison tables, detailed command explanations, and live verification logs, refer to our companion publication:

* Official Guide: [DaemonSets on Azure AKS: Running an Agent on Every Worker Node](https://stacklayer.blogspot.com/2026/10/azure-aks-daemonsets-running-agent-on-every-worker-node.html)

---

## 📁 Directory Structure

```text
11-kubernetes-daemonsets-on-aks/
├── README.md
└── kube-manifests/
    └── 01-stacklayer-node-agent.yaml   # DaemonSet manifest with hostPath volume mount and Downward API
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

## 🚀 Hands-on Execution Workflow

### Step 1: Inspect the Active Cluster Worker Nodes

Query the cluster to identify your baseline worker node count:

```bash
kubectl get nodes -o wide
```

### Step 2: Deploy the Node Agent DaemonSet

Apply the declarative YAML manifest defining the node telemetry agent:

```bash
kubectl apply -f kube-manifests/01-stacklayer-node-agent.yaml
```

### Step 3: Verify Controller Scheduling and Node Placement

Inspect the DaemonSet controller status to verify that desired pods match your total active worker nodes:

```bash
kubectl get daemonsets
```

Verify that each pod is running on a distinct worker node:

```bash
kubectl get pods -o wide -l app=stacklayer-agent
```

### Step 4: Stream Live Agent Telemetry and Host Log Access

View aggregated logs across all node agents to verify host filesystem access and Downward API node name injection:

```bash
kubectl logs -l app=stacklayer-agent --tail=10 --prefix=true
```

### Step 5: Test Dynamic Scaling Behavior

Scale the AKS node pool to 3 nodes and observe Kubernetes automatically scheduling a new agent pod without manual intervention:

```bash
# Scale cluster worker nodes to 3
az aks scale --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --node-count 3

# Verify that the DaemonSet automatically scaled its desired count to 3
kubectl get daemonsets
kubectl get pods -o wide -l app=stacklayer-agent
```

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure subscription credits and prevent unnecessary cloud compute billing, execute one of the following teardown options:

### Option A: Pause Cluster Nodes (Recommended If Continuing Later Labs)

Suspends virtual machines in the node pool to stop compute billing while retaining cluster configurations:

```bash
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Complete Deletion (Zero Cost)

Permanently removes the entire resource group and all associated Azure cloud resources:

```bash
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <h3>🌐 Architecting Resilient Node-Level Operations on AKS</h3>
  <p><i>“True cloud resilience begins at the foundation: when every node is monitored, every workload runs with confidence.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
