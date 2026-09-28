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
| **01** | [01-aks-cluster-deploy-app](./01-aks-cluster-deploy-app) | AKS Cluster Provisioning, CLI Config & Load Balancer | [Read Guide ↗](https://stacklayer.io/deploy-first-app-aks) |
| **02** | [02-scaling-rolling-updates](./02-scaling-rolling-updates) | Horizontal Pod Scaling, Zero-Downtime Updates & Log Diagnostics | [Read Guide ↗](https://stacklayer.io/azure-aks-scaling-rolling-updates-logs) |
| **03** | [03-configmaps-secrets](./03-configmaps-secrets) | Decoupling Configurations via ConfigMaps, Secrets & Volume Mounts | [Read Guide ↗](https://stacklayer.io/azure-aks-configmaps-secrets) |
| **04** | [04-persistent-storage-disks-files](./04-persistent-storage-disks-files) | Stateful Volumes, CSI Dynamic Provisioning (Azure Disks & Azure Files) | [Read Guide ↗](https://stacklayer.io/azure-aks-persistent-storage-disks-files) |
| **05** | [05-azure-mysql-phpmyadmin](./05-azure-mysql-phpmyadmin) | PaaS Database Integration (Azure MySQL Flexible Server + phpMyAdmin) | [Read Guide ↗](https://stacklayer.io/connect-azure-mysql-flexible-server-phpmyadmin-aks) |

---

## 🚀 Quick Start

### 1. Clone This Repository
```bash
git clone https://github.com/StackLayerOfficially/azure-aks-hands-on-labs.git
cd azure-aks-hands-on-labs
