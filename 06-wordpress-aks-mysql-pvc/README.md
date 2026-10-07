# Production 2-Tier WordPress on Azure AKS with Persistent Storage (PVC) & Azure MySQL

Welcome to the hands-on cloud engineering lab by [Stack Layer](https://stacklayer.blogspot.com/)!

In modern enterprise architectures, production workloads are never bundled into a single fragile container where everything is lost if something crashes. Instead, systems use a **2-tier decoupled architecture**:
* **Frontend Web Tier:** WordPress container pods running on Azure Kubernetes Service (AKS), configured with a Persistent Volume Claim (PVC) so your themes, plugins, and media uploads are safely preserved.
* **Backend Database Tier:** An enterprise-managed **Azure Database for MySQL Flexible Server**, offloading database backups, automated patching, and high availability directly to Azure.

In this lab, you will build, connect, and chaos-test these two tiers using native Kubernetes Secrets, declarative YAML manifests, and Azure Managed Disks.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, access mode deep dives, and live troubleshooting checks, refer to our companion publication:
* **Official Guide:** [Deploy a Production 2-Tier WordPress Application on Azure AKS with Persistent Storage (PVC) & Azure MySQL](https://stacklayer.blogspot.com/)

---

## 🎯 Learning Objectives

By completing this hands-on lab, you will:
1. Architect a production-grade **2-tier decoupled stateful application** on Microsoft Azure.
2. Provision an enterprise-managed **Azure Database for MySQL Flexible Server** using Azure CLI and Azure Portal GUI.
3. Securely decouple database connection credentials using Kubernetes **Secrets** without exposing plain text in manifests.
4. Dynamically provision cloud block storage using Azure Managed Disks via Kubernetes **PersistentVolumeClaims (PVC)**.
5. Deploy the **`stacklayer/wordpress:latest`** web application and mount durable storage to `/var/www/html/wp-content`.
6. Expose the web tier to the internet via an **Azure Public Load Balancer Service**.
7. Execute a **live chaos resilience test** by intentionally terminating running pods and proving 100% data survivability.
8. Safely manage cluster lifecycles and cost optimization to eliminate unnecessary cloud consumption bills.

---

## 📋 Prerequisites

Before beginning this lab, ensure you have:
* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS) Step-by-Step](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)).
* **Azure CLI (`az`)** and **Kubectl** installed and authenticated.

Connect your local terminal to your running cluster:

```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity
kubectl cluster-info
kubectl get nodes
```

---

## 📁 Repository Directory Structure

```text
06-wordpress-aks-mysql-pvc/
├── README.md
└── kube-manifests/
    ├── 01-mysql-secret.yaml          # Database credentials & host configuration
    ├── 02-wordpress-pvc.yaml         # Persistent Volume Claim for wp-content
    ├── 03-wordpress-deployment.yaml  # WordPress application deployment
    └── 04-wordpress-service.yaml     # Public LoadBalancer service
```

---

## 🏗️ Architecture Overview

```text
                      +---------------------------------------------------+
                      |             Azure Kubernetes Service              |
                      |            (stacklayer-aks-cluster)               |
                      |                                                   |
Internet Traffic ---> |  [Service: LoadBalancer]                          |
                      |             |                                     |
                      |      [WordPress Pods] <---> [Azure Managed Disk]  |
                      |      (stacklayer/     |     (Mount: wp-content)   |
                      |       wordpress:latest)                           |
                      +-------------|-------------------------------------+
                                    |
                                    v (Encrypted MySQL SSL Connection)
                      +---------------------------------------------------+
                      |      Azure Database for MySQL Flexible Server     |
                      |           (stacklayer-mysql-server)               |
                      +---------------------------------------------------+
```

---

## Step 1: Provision Azure Database for MySQL Flexible Server

We provide two deployment workflows so you can choose the path you prefer:

### Path A: Fast Automated Deployment via Azure CLI

Run these commands in your terminal. Replace `UNIQUE_SUFFIX` with 3 random digits or your initials to make your database server name globally unique:

```bash
# 1. Define configuration variables
RESOURCE_GROUP="stacklayer-aks-rg"
LOCATION="eastus"
MYSQL_SERVER="stacklayer-mysql-UNIQUE_SUFFIX"
ADMIN_USER="stackadmin"
ADMIN_PASSWORD="StackLayer2026!Secure"

# 2. Provision the burstable MySQL Flexible Server (B1ms tier for maximum cost savings)
az mysql flexible-server create \
  --resource-group $RESOURCE_GROUP \
  --name $MYSQL_SERVER \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASSWORD \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --storage-size 32 \
  --version 8.0.21 \
  --yes

# 3. Create the dedicated WordPress database
az mysql flexible-server db create \
  --resource-group $RESOURCE_GROUP \
  --server-name $MYSQL_SERVER \
  --database-name stacklayer_wordpressdb

# 4. Configure firewall to allow AKS pods to reach MySQL
az mysql flexible-server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --name $MYSQL_SERVER \
  --rule-name AllowAllAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

---

### Path B: Visual Deployment via Azure Portal GUI

1. Log in to the [Azure Portal](https://portal.azure.com/) and search for **Azure Database for MySQL flexible servers**.
2. Click **Create**, select subscription, and choose resource group `stacklayer-aks-rg`.
3. Set server name to `stacklayer-mysql-UNIQUE_SUFFIX` and region to your AKS region (e.g., *East US*).
4. Under Compute + Storage, select **Burstable (Standard_B1ms)** with **32 GiB** storage.
5. Set administrator username to `stackadmin` and configure your password.
6. Under the **Networking** tab, check: **Allow public access from any Azure service within Azure to this server**.
7. Click **Review + create**, then **Create**.
8. Once created, open the resource, click **Databases** from the left blade, click **+ Add**, and create database `stacklayer_wordpressdb`.

---

## Step 2: Store Database Credentials in a Kubernetes Secret

Create `kube-manifests/01-mysql-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: stacklayer-mysql-secret
  labels:
    app: stacklayer-wordpress
type: Opaque
stringData:
  # Replace with your actual MySQL server FQDN:
  WORDPRESS_DB_HOST: "stacklayer-mysql-UNIQUE_SUFFIX.mysql.database.azure.com"
  WORDPRESS_DB_NAME: "stacklayer_wordpressdb"
  WORDPRESS_DB_USER: "stackadmin"
  WORDPRESS_DB_PASSWORD: "StackLayer2026!Secure"
```

Apply the secret:
```bash
kubectl apply -f kube-manifests/01-mysql-secret.yaml
```

---

## Step 3: Claim Persistent Storage (PVC) for WordPress Uploads

Create `kube-manifests/02-wordpress-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stacklayer-wordpress-pvc
  labels:
    app: stacklayer-wordpress
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: default
  resources:
    requests:
      storage: 10Gi
```

Apply the persistent volume claim:
```bash
kubectl apply -f kube-manifests/02-wordpress-pvc.yaml
```

Verify storage registration:
```bash
kubectl get pvc
```
*(Note: Under Azure default storage classes, the PVC status changes from `Pending` to `Bound` as soon as the pod is scheduled in Step 4).*

---

## Step 4: Deploy the WordPress Web Tier

Create `kube-manifests/03-wordpress-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-wordpress-deployment
  labels:
    app: stacklayer-wordpress
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stacklayer-wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: stacklayer-wordpress
        tier: frontend
    spec:
      containers:
        - name: wordpress
          image: stacklayer/wordpress:latest
          env:
            - name: WORDPRESS_DB_HOST
              valueFrom:
                secretKeyRef:
                  name: stacklayer-mysql-secret
                  key: WORDPRESS_DB_HOST
            - name: WORDPRESS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: stacklayer-mysql-secret
                  key: WORDPRESS_DB_NAME
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: stacklayer-mysql-secret
                  key: WORDPRESS_DB_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: stacklayer-mysql-secret
                  key: WORDPRESS_DB_PASSWORD
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - name: wordpress-storage
              mountPath: /var/www/html/wp-content
      volumes:
        - name: wordpress-storage
          persistentVolumeClaim:
            claimName: stacklayer-wordpress-pvc
```

Apply the deployment:
```bash
kubectl apply -f kube-manifests/03-wordpress-deployment.yaml
```

---

## Step 5: Expose WordPress via Public Load Balancer

Create `kube-manifests/04-wordpress-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-wordpress-service
  labels:
    app: stacklayer-wordpress
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      name: http
  selector:
    app: stacklayer-wordpress
    tier: frontend
```

Apply the service and monitor for external IP provisioning:
```bash
kubectl apply -f kube-manifests/04-wordpress-service.yaml

# Monitor external IP creation
kubectl get service stacklayer-wordpress-service --watch
```

Once the `EXTERNAL-IP` appears, copy it into your web browser!

---

## Step 6: Live Verification & Cloud Resilience Chaos Testing

1. Open `http://<EXTERNAL-IP>` in your browser.
2. Complete the **WordPress 5-Minute Installation** by setting your site title and administrator credentials.
3. Log in to the WordPress dashboard, publish a test blog post, and upload an image into your Media Library.
4. **The Ultimate Cloud Resilience Test:** Intentionally delete your running WordPress pod to simulate a sudden hardware crash:
   ```bash
   kubectl delete pod -l app=stacklayer-wordpress
   ```
5. Kubernetes will immediately detect the missing pod and spin up a replacement in seconds:
   ```bash
   kubectl get pods -l app=stacklayer-wordpress
   ```
6. Refresh your web browser: **Your blog post, settings, and uploaded image are 100% intact!** The post was preserved in Azure MySQL, and the media file was saved on the Azure Managed Disk PVC.

---

## Step 7: Cost Optimization & Lab Teardown

When you finish practicing, clean up your resources to avoid unwanted cloud charges:

```bash
# Option A: Pause cluster and database overnight (preserves your work)
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
az mysql flexible-server stop --resource-group stacklayer-aks-rg --name stacklayer-mysql-UNIQUE_SUFFIX

# Option B: Complete lab deletion
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud tutorials on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <br />
  <h3>🛡️ State Decoupling is the Foundation of True Cloud Resilience</h3>
  <p style="max-width: 580px; line-height: 1.6; font-style: italic;">
    “A fragile system fears container restarts; a production-grade system treats them as routine. By separating your presentation layer from persistent disks and managed databases, you have engineered a system that bends but never breaks.”
  </p>
  <br />
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
