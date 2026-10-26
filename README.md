<div align="center">
  <img src="https://raw.githubusercontent.com/StackLayerOfficially/.github/main/profile/banner.jpg" alt="Stack Layer" width="100%" />
</div>

<br>

<div align="center">

# ☁️ Azure AKS Hands-on Labs

**Practical, Step-by-Step Kubernetes Blueprints & Workloads on Microsoft Azure**

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.28+-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Azure AKS](https://img.shields.io/badge/Azure-AKS-0078D4?style=flat-square&logo=microsoftazure&logoColor=white)](https://azure.microsoft.com/services/kubernetes-service/)
[![Stack Layer](https://img.shields.io/badge/Official_Blog-Stack_Layer-be123c?style=flat-square)](https://stacklayer.blogspot.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

*Curated and maintained by the **Stack Layer** engineering team.*

</div>

---

## 📌 Overview

Welcome to the official companion code repository for the **Stack Layer Azure Kubernetes Service (AKS) Hands-on Series**. 

This repository contains verified, practical Kubernetes manifests, declarative YAML deployment templates, and configuration runbooks designed to help cloud architects, systems administrators, and DevOps learners master day-to-day Kubernetes operations on Microsoft Azure.

Every directory in this repository corresponds directly to a complete, in-depth architectural guide published on the [Stack Layer Technical Blog](https://stacklayer.blogspot.com/).

---

## 🗺️ Learning Path & Architecture Roadmap

The hands-on labs in this repository follow a progressive learning path from initial cluster setup to stateful storage and external PaaS integrations:

```text
  [ Phase 01: Foundations ]
  ├── Lab 01: Provision AKS Cluster & Deploy Sample App (Azure Load Balancer)
  ├── Lab 02: Horizontal Pod Autoscaling (HPA) & Zero-Downtime Rolling Updates
  └── Lab 03: Configuration Decoupling (ConfigMaps, Secrets & Volume Mounts)
            │
            ▼
  [ Phase 02: Storage & Stateful Workloads ]
  ├── Lab 04: Persistent Storage with Azure Disks & Azure Files (CSI Drivers)
  └── Lab 05: Stateful Integration (Azure MySQL Flexible Server + phpMyAdmin)
            │
            ▼
  [ Phase 03: Advanced Day-2 Operations (Upcoming) ]
  └── Ingress Controllers (NGINX/App Gateway), TLS Automation & Monitoring
```

---

## 🛠️ Prerequisites & Local Tooling

Before deploying any of the manifests in this repository, ensure your local workstation has the following tools installed and authenticated:

| Tool | Recommended Version | Purpose | Verification Command |
| :--- | :---: | :--- | :--- |
| **Azure CLI (`az`)** | `v2.50+` | Provisioning and managing Azure cloud resources | `az version` |
| **Kubernetes CLI (`kubectl`)** | `v1.28+` | Deploying manifests and managing AKS workloads | `kubectl version --client` |
| **Git** | `v2.30+` | Cloning code and navigating lab branches | `git --version` |
| **Azure Subscription** | Active | Dedicated resource group with standard VM core quota | `az account show` |

---

## 📚 Labs & Workload Index

Labs are organized into modular, topic-focused phases. Click on any lab directory to access its source YAML manifests and step-by-step deployment guide.

<details open>
<summary><b>📦 Module 1: Cluster Foundations, Scaling & Config Management</b></summary>
<br>

| # | Hands-on Lab Directory | Key Capabilities Explored | Step-by-Step Architecture Guide |
| :-: | :--- | :--- | :--- |
| **01** | [01-aks-cluster-deploy-app](./01-aks-cluster-deploy-app) | AKS Cluster Provisioning, CLI Context & Public Load Balancer | [Read Guide ↗](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html) |
| **02** | [02-scaling-rolling-updates](./02-scaling-rolling-updates) | Horizontal Pod Autoscaling (HPA), Rolling Upgrades & Logging | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/azure-aks-tutorial-scaling-rolling-updates.html) |
| **03** | [03-configmaps-secrets](./03-configmaps-secrets) | Decoupling Settings with ConfigMaps, Secret Objects & Volume Mounts | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/azure-aks-tutorial-configmaps-secrets.html) |

</details>

<details open>
<summary><b>💾 Module 2: Stateful Workloads & Managed Azure Storage</b></summary>
<br>

| # | Hands-on Lab Directory | Key Capabilities Explored | Step-by-Step Architecture Guide |
| :-: | :--- | :--- | :--- |
| **04** | [04-persistent-storage-disks-files](./04-persistent-storage-disks-files) | Stateful Volumes, CSI Dynamic Provisioning (Azure Disks & Azure Files) | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/azure-aks-persistent-storage-disks-files.html) |
| **05** | [05-azure-mysql-phpmyadmin](./05-azure-mysql-phpmyadmin) | PaaS Database Integration (Azure MySQL Flexible Server + phpMyAdmin) | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/connect-azure-mysql-flexible-server-phpmyadmin-aks.html) |

</details>

> 💡 *New hands-on labs are continuously added as new engineering guides publish on [Stack Layer](https://stacklayer.blogspot.com/).*

---

## 📂 Repository Layout

Each lab directory follows an isolated, predictable folder hierarchy for quick navigation:

```text
azure-aks-hands-on-labs/
├── 01-aks-cluster-deploy-app/
│   ├── kube-manifests/          # Declarative YAML manifests (Deployments, Services)
│   └── README.md                # Lab instructions & CLI execution steps
├── 02-scaling-rolling-updates/
│   ├── kube-manifests/          # HPA, Deployment v1/v2, Service manifests
│   └── README.md
├── 03-configmaps-secrets/
│   ├── kube-manifests/          # ConfigMaps, Secrets, Pod volume mounts
│   └── README.md
├── 04-persistent-storage-disks-files/
│   ├── kube-manifests/          # PVCs, StorageClasses, StatefulSets
│   └── README.md
├── 05-azure-mysql-phpmyadmin/
│   ├── kube-manifests/          # External services, endpoints, credentials
│   └── README.md
└── LICENSE                      # MIT Open Source License
```

---

## 🚀 Quick Start Runbook

### 1. Clone This Repository
```bash
git clone https://github.com/StackLayerOfficially/azure-aks-hands-on-labs.git
cd azure-aks-hands-on-labs
```

### 2. Connect Your Terminal to Your AKS Cluster
```bash
az aks get-credentials --resource-group aks-rg --name aks-cluster --overwrite-existing
```

### 3. Deploy Any Lab Manifests
Navigate to your chosen lab directory and declaratively apply its manifests:
```bash
# Example: Deploying the stateful storage lab
cd 04-persistent-storage-disks-files/kube-manifests
kubectl apply -f .
```

### 4. Verify Workload Health
```bash
kubectl get pods -o wide
kubectl get svc
kubectl get pvc,pv
```

---

## ⚠️ Educational Lab Disclaimer

> **Important Notice:** The manifests, configurations, and scripts provided in this repository are created strictly for **hands-on educational, demonstration, and learning lab purposes**. 
>
> While these configurations follow modern cloud practices, they are structured for training and conceptual clarity. They should not be used directly in live, mission-critical, or regulated production environments without independent security hardening, resource quota planning, and architectural compliance reviews.
> 
> *Stack Layer and its technical contributors assume no liability for unexpected cloud billing, resource deletion, or service disruptions incurred while running these labs.*

---

## 🤝 Community & Support

* 📖 **Official Technical Articles:** [Stack Layer Blog](https://stacklayer.blogspot.com/)
* 💬 **Discussions & Feedback:** Join our [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions) to ask questions, share progress, or suggest future lab topics.
* ⭐ **Support the Project:** If these hands-on guides assist your cloud learning journey, please consider starring this repository!

---

<div align="center">
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
