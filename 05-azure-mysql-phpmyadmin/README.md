# Connect Azure Database for MySQL Flexible Server to phpMyAdmin on Azure AKS

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In modern enterprise architectures, running production relational database engines inside container pods can introduce unnecessary operational overhead (such as manual backup schedules, storage replication management, and complex failover topologies). The industry-standard architecture connects containerized microservices running on **Azure Kubernetes Service (AKS)** to fully managed cloud database services like **Azure Database for MySQL Flexible Server**.

In this hands-on lab, you will learn how to provision a managed Azure MySQL Flexible Server, configure Azure cloud firewall rules, securely store database credentials inside Kubernetes Secrets, deploy **phpMyAdmin** as a containerized management portal on AKS, and interact with live cloud database tables directly through your web browser.

---

## 📖 Companion Blog Guide
For comprehensive architectural diagrams, deep-dive firewall rule explanations, SSL/TLS connection parameters, and GUI troubleshooting, refer to our companion publication:
* **Official Guide:** [Azure AKS: Connect Azure Database for MySQL Flexible Server to phpMyAdmin Step-by-Step](https://stacklayer.blogspot.com/2026/09/azure-aks-connect-mysql-flexible-server-phpmyadmin.html)

---

## 📁 Directory Structure

```text
05-azure-mysql-phpmyadmin/
├── README.md
└── kube-manifests/
    ├── 01-mysql-secret.yaml          # Sensitive MySQL host, username, and password
    ├── 02-phpmyadmin-deployment.yaml # phpMyAdmin workload connecting to Azure MySQL
    └── 03-phpmyadmin-service.yaml    # Public Azure Load Balancer endpoint
```

---

## 🎯 Learning Objectives

By completing this lab, you will:
1. Provision a cost-effective, managed **Azure Database for MySQL Flexible Server** (burstable `B1ms` tier) using Azure CLI.
2. Authorize inbound network traffic from your AKS cluster by configuring the **AllowAllAzureIPs** firewall rule.
3. Encapsulate database endpoints and credentials inside a secure **Kubernetes Secret** (`01-mysql-secret.yaml`).
4. Deploy a scalable **phpMyAdmin** workload on AKS configured for SSL/TLS database connectivity (`02-phpmyadmin-deployment.yaml`).
5. Expose phpMyAdmin to the internet via an automated **Azure Standard Load Balancer** (`03-phpmyadmin-service.yaml`).
6. Create database tables and rows in your browser, test pod disaster recovery, and verify persistent cloud storage.
7. Clean up cloud resources safely to avoid unexpected Azure billing.

---

## 📋 Prerequisites

* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS) Step-by-Step](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)).
* **Azure CLI (`az`)** and **Kubectl** installed and authenticated.

Connect your local terminal to your running cluster:
```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity
kubectl cluster-info
```

---

## 🚀 Step-by-Step Hands-On Guide

### Step 1: Provision Azure Database for MySQL Flexible Server

Set environment variables and provision a lightweight, burstable MySQL instance in your resource group:

```bash
# Set unique configuration variables
export RG_NAME="stacklayer-aks-rg"
export LOCATION="eastus"
export MYSQL_SERVER_NAME="stacklayer-mysql-$RANDOM"
export ADMIN_USER="stacklayeradmin"
export ADMIN_PASSWORD="VaultPassword@2026!Azure"

# Provision Azure MySQL Flexible Server (takes ~3-5 minutes)
az mysql flexible-server create \
  --resource-group $RG_NAME \
  --name $MYSQL_SERVER_NAME \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASSWORD \
  --tier Burstable \
  --sku-name Standard_B1ms \
  --storage-size 20 \
  --version 8.0.21 \
  --yes
```

---

### Step 2: Configure Azure Database Firewall for AKS Connectivity

By default, Azure MySQL blocks all public and private network traffic. Authorize connections originating from Azure services (including your AKS cluster worker nodes):

```bash
# Configure Azure firewall rule to allow AKS traffic
az mysql flexible-server firewall-rule create \
  --resource-group $RG_NAME \
  --name $MYSQL_SERVER_NAME \
  --rule-name AllowAllAzureIPs \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

Obtain the Fully Qualified Domain Name (FQDN) of your server:
```bash
# Display the server FQDN
az mysql flexible-server show \
  --resource-group $RG_NAME \
  --name $MYSQL_SERVER_NAME \
  --query "fullyQualifiedDomainName" -o tsv
```

---

### Step 3: Create the Database Secret (`01-mysql-secret.yaml`)

Store the MySQL connection parameters securely inside a Kubernetes Secret:

#### `kube-manifests/01-mysql-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: stacklayer-mysql-secret
  labels:
    app: stacklayer-phpmyadmin
type: Opaque
stringData:
  PMA_HOST: "<YOUR_MYSQL_SERVER_NAME>.mysql.database.azure.com"
  PMA_USER: "stacklayeradmin"
  PMA_PASSWORD: "VaultPassword@2026!Azure"
```
*(Make sure to replace `<YOUR_MYSQL_SERVER_NAME>` with your actual Azure server name).*

Apply the secret:
```bash
# Deploy database secret
kubectl apply -f kube-manifests/01-mysql-secret.yaml

# Verify secret creation
kubectl get secret stacklayer-mysql-secret
```

---

### Step 4: Deploy phpMyAdmin on AKS (`02-phpmyadmin-deployment.yaml`)

Deploy phpMyAdmin with environment variables mapped directly to your database secret. We enable `PMA_SSL: "true"` to meet Azure Flexible Server encryption standards.

#### `kube-manifests/02-phpmyadmin-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-phpmyadmin-deployment
  labels:
    app: stacklayer-phpmyadmin
spec:
  replicas: 2
  selector:
    matchLabels:
      app: stacklayer-phpmyadmin
  template:
    metadata:
      labels:
        app: stacklayer-phpmyadmin
    spec:
      containers:
      - name: phpmyadmin
        image: phpmyadmin/phpmyadmin:latest
        ports:
        - containerPort: 80
          name: http
        env:
        - name: PMA_HOST
          valueFrom:
            secretKeyRef:
              name: stacklayer-mysql-secret
              key: PMA_HOST
        - name: PMA_USER
          valueFrom:
            secretKeyRef:
              name: stacklayer-mysql-secret
              key: PMA_USER
        - name: PMA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: stacklayer-mysql-secret
              key: PMA_PASSWORD
        - name: PMA_SSL
          value: "true"
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

Apply the deployment:
```bash
# Deploy phpMyAdmin workload
kubectl apply -f kube-manifests/02-phpmyadmin-deployment.yaml

# Confirm pods are running
kubectl get pods -l app=stacklayer-phpmyadmin -o wide
```

---

### Step 5: Expose phpMyAdmin via Azure Load Balancer (`03-phpmyadmin-service.yaml`)

Provision an Azure Standard Load Balancer to grant public access to the phpMyAdmin web interface:

#### `kube-manifests/03-phpmyadmin-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-phpmyadmin-service
  labels:
    app: stacklayer-phpmyadmin
spec:
  type: LoadBalancer
  selector:
    app: stacklayer-phpmyadmin
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Apply the service and stream real-time IP allocation:
```bash
# Deploy Load Balancer service
kubectl apply -f kube-manifests/03-phpmyadmin-service.yaml

# Watch until EXTERNAL-IP is assigned
kubectl get svc stacklayer-phpmyadmin-service -w
```
*(Press `Ctrl + C` once the public IP address is assigned).*

---

### Step 6: Test Data Persistence via Web Browser & Pod Deletion

1. Open your browser and navigate to `http://<YOUR_EXTERNAL_IP>`.
2. Because credentials are automatically injected from your Kubernetes Secret, you will be logged directly into the phpMyAdmin dashboard!
3. **Create a Database & Table:**
   * Click **New** in the left sidebar and name the database `stacklayer_db`.
   * Create a table named `users` with columns: `id` (INT, Primary Key, Auto Increment) and `username` (VARCHAR 100).
   * Insert 2 sample user records (`alice`, `bob`).
4. **The Disaster Recovery Test:**
   * In your terminal, delete all running phpMyAdmin pods:
     ```bash
     kubectl delete pods -l app=stacklayer-phpmyadmin
     ```
   * Wait a few seconds for the Kubernetes Deployment controller to automatically provision replacement pods.
   * Refresh your web browser at `http://<YOUR_EXTERNAL_IP>`.
   * **Result:** Your database, tables, and rows remain 100% intact! Because storage and database computation are managed by Azure MySQL Flexible Server, pod lifecycles have zero impact on your data.

---

## 🧹 Cost Safeguards & Cleanup

Managed database services and load balancers incur hourly cloud consumption costs. Always execute cleanup when your testing session is complete:

### 1. Delete Kubernetes Workloads & Service
```bash
# Delete phpMyAdmin deployment and public load balancer
kubectl delete -f kube-manifests/03-phpmyadmin-service.yaml
kubectl delete -f kube-manifests/02-phpmyadmin-deployment.yaml
kubectl delete -f kube-manifests/01-mysql-secret.yaml
```

### 2. Delete Azure MySQL Flexible Server
```bash
# Permanently delete the managed MySQL server
az mysql flexible-server delete \
  --resource-group stacklayer-aks-rg \
  --name $MYSQL_SERVER_NAME \
  --yes
```

### 3. Manage AKS Cluster Lifecycles
* **Option A: Stop AKS Cluster (Halts VM compute billing, preserves cluster)**:
  ```bash
  az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
  ```
* **Option B: Delete Resource Group (Permanent Teardown)**:
  ```bash
  az group delete --name stacklayer-aks-rg --yes --no-wait
  ```

---

## 🤝 Community & Support
* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud tutorials on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <br />
  <h3>🌐 Bridge the Cloud. Architect for Resilience.</h3>
  <p style="max-width: 580px; line-height: 1.6; font-style: italic;">
    “True cloud architecture is about choosing the right tool for the job—orchestrating stateless workloads in Kubernetes while trusting enterprise data to resilient, managed cloud databases.”
  </p>
  <br />
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
