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

This repository contains verified, battle-tested Kubernetes manifests, declarative YAML deployment templates, and configuration runbooks designed to help cloud architects, systems administrators, and DevOps practitioners master day-to-day Kubernetes operations on Microsoft Azure.

Every directory in this repository corresponds directly to a complete, in-depth architectural guide on our blog.

---

## 📚 Labs & Workload Index

| # | Hands-on Lab Topic | Focus Area | Step-by-Step Architecture Guide |
| :-: | :--- | :--- | :--- |
| **01** | [01-aks-cluster-deploy-app](./01-aks-cluster-deploy-app) | AKS Cluster Provisioning, CLI Config & Load Balancer | [Read Guide ↗](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html) |
| **02** | [02-scaling-rolling-updates](./02-scaling-rolling-updates) | Horizontal Pod Scaling, Zero-Downtime Updates & Log Diagnostics | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/azure-aks-tutorial-scaling-rolling-updates.html) |
| **03** | [03-configmaps-secrets](./03-configmaps-secrets) | Decoupling Configurations via ConfigMaps, Secrets & Volume Mounts | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/azure-aks-tutorial-configmaps-secrets.html) |
| **04** | [04-persistent-storage-disks-files](./04-persistent-storage-disks-files) | Stateful Volumes, CSI Dynamic Provisioning (Azure Disks & Azure Files) | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/azure-aks-persistent-storage-disks-files.html) |
| **05** | [05-azure-mysql-phpmyadmin](./05-azure-mysql-phpmyadmin) | PaaS Database Integration (Azure MySQL Flexible Server + phpMyAdmin) | [Read Guide ↗](https://stacklayer.blogspot.com/2026/09/connect-azure-mysql-flexible-server-phpmyadmin-aks.html) |

---

## 🚀 Quick Start

### 1. Clone This Repository
```bash
git clone https://github.com/StackLayerOfficially/azure-aks-hands-on-labs.git
cd azure-aks-hands-on-labs
```

### 2. Connect Your Local Terminal to Your AKS Cluster
```bash
az aks get-credentials --resource-group aks-rg --name aks-cluster --overwrite-existing
```

### 3. Deploy Any Lab in Seconds
Navigate to any lab directory and declaratively apply its manifests:
```bash
# Example: Deploying the stateful storage lab
cd 04-persistent-storage-disks-files/kube-manifests
kubectl apply -f .
```

---

## ⚠️ Educational Lab Disclaimer

> **Important Notice:** The manifests, configurations, and scripts provided in this repository are created strictly for **hands-on educational, demonstration, and learning lab purposes**. 
>
> While these configurations follow industry patterns, they are simplified demonstrations and should not be used directly in live production or regulated enterprise environments without independent security, compliance, and architectural reviews. 
> 
> *Stack Layer and its technical contributors assume no liability for any unexpected cloud charges, service disruptions, or data loss resulting from the deployment of these resources.*

---

## 🤝 Community & Support

* 📖 **Official Technical Articles:** [Stack Layer](https://stacklayer.blogspot.com/)
* 💬 **Discussions & Feedback:** Join our [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions) to ask questions or share your lab progress.
* ⭐ **Support the Project:** If you found these labs helpful, please consider giving this repository a star!

---

<div align="center">
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
