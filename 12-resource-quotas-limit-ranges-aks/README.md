# Resource Quotas and LimitRanges on Azure AKS: Protecting Clusters from Runaway Pods

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In a multi-tenant cloud environment, shared Kubernetes clusters are vulnerable to resource starvation if applications run without boundaries. A single unconstrained workload or memory leak can consume all CPU and RAM on a worker node, forcing the Linux kernel to terminate surrounding pods. This lab demonstrates how to architect, enforce, and verify automated **LimitRanges** and namespace **ResourceQuotas** on **Azure Kubernetes Service (AKS)** to protect your cluster infrastructure against runaway workloads.

---

## 📖 Companion Blog Guide

For complete architectural breakdowns, concept comparison tables, detailed command explanations, and live verification logs, refer to our companion publication:

* Official Guide: [Resource Quotas and LimitRanges on Azure AKS: Protecting Clusters from Runaway Pods](https://stacklayer.blogspot.com/2026/10/azure-aks-resource-quotas-limits-protect-clusters.html)

---

## 📁 Directory Structure

```text
12-resource-quotas-limit-ranges-aks/
├── README.md
└── kube-manifests/
    ├── 01-stacklayer-governance-namespace.yaml   # Isolated namespace for resource policy testing
    ├── 02-stacklayer-limit-range.yaml            # Default, minimum, and maximum container resource boundaries
    ├── 03-stacklayer-unconstrained-pod.yaml      # Pod omitting resources to verify automatic default injection
    ├── 04-stacklayer-resource-quota.yaml         # Hard ceilings for aggregate CPU, memory, and total pods
    ├── 05-stacklayer-runaway-pod.yaml            # Oversized pod to verify admission controller rejection
    └── 06-stacklayer-scaled-deployment.yaml      # Multi-replica workload demonstrating quota pool saturation
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

### Step 1: Provision the Testing Namespace

Create the isolated virtual namespace for applying governance policies:

```bash
kubectl apply -f kube-manifests/01-stacklayer-governance-namespace.yaml
```

### Step 2: Enforce Container Boundaries with LimitRange

Deploy the LimitRange to define default, minimum, and maximum container CPU and memory allocations:

```bash
kubectl apply -f kube-manifests/02-stacklayer-limit-range.yaml
```

### Step 3: Verify Automated Default Injection

Deploy an unconstrained pod and verify that the admission controller automatically injects default requests and limits:

```bash
# Deploy pod omitting resource specifications
kubectl apply -f kube-manifests/03-stacklayer-unconstrained-pod.yaml

# Verify injected compute values via jsonpath
kubectl get pod stacklayer-default-web -n stacklayer-governance -o jsonpath='{.spec.containers[0].resources}'
```

### Step 4: Enforce Namespace Capacity with ResourceQuota

Apply hard compute and pod count ceilings across the entire namespace:

```bash
# Apply namespace quota
kubectl apply -f kube-manifests/04-stacklayer-resource-quota.yaml

# Inspect current namespace quota consumption
kubectl get resourcequota -n stacklayer-governance
```

### Step 5: Test Admission Controller Rejection

Attempt to deploy an oversized container that exceeds the LimitRange maximum ceiling and verify immediate rejection:

```bash
kubectl apply -f kube-manifests/05-stacklayer-runaway-pod.yaml
```

### Step 6: Test Quota Saturation and Scaling Limits

Deploy a multi-replica deployment to demonstrate how ResourceQuotas prevent namespace exhaustion by blocking excess replicas:

```bash
# Deploy 4-replica workload against a 3-pod quota ceiling
kubectl apply -f kube-manifests/06-stacklayer-scaled-deployment.yaml

# Verify that pod count strictly honors the 3-pod ceiling
kubectl get pods -n stacklayer-governance

# Inspect replica failure events recorded by Kubernetes
kubectl describe deployment stacklayer-governed-deployment -n stacklayer-governance | grep -A 5 Conditions:
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
  <h3>🌐 Protecting Cluster Resilience with Automated Governance</h3>
  <p><i>“True cloud scalability is not merely adding compute—it is setting the intelligent boundaries that keep your platform stable and fair.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>

