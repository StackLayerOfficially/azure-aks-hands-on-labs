# Configuration Management & Secret Injection on Azure AKS

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/).

In production Kubernetes deployments, hardcoding sensitive credentials or environment-specific properties inside container images is a critical anti-pattern. Decoupling configuration from application code ensures security, portability, and streamlined updates across development, staging, and production environments. 

In this hands-on lab, you will learn how to declare, manage, and inject non-sensitive configuration settings with **ConfigMaps** and securely deliver sensitive database credentials and API tokens with **Kubernetes Secrets** using environment variables and volume mounts.

---

## 📖 Companion Blog Guide
For in-depth architectural breakdowns, security best practices, and Azure CLI imperative workflows, refer to our companion publication:
* **Official Guide:** [Configuration Management on Azure AKS: Decoupling ConfigMaps and Secrets Step-by-Step](https://stacklayer.blogspot.com/2026/09/azure-aks-tutorial-configmaps-secrets.html)

---

## 📁 Directory Structure

```text
03-configmaps-secrets/
├── README.md
└── kube-manifests/
    ├── 01-configmap.yaml     # Non-sensitive application configuration
    ├── 02-secret.yaml        # Base64-encoded confidential credentials
    ├── 03-deployment.yaml    # Workload injecting config & secrets (env & volumes)
    └── 04-service.yaml       # Public Azure Load Balancer endpoint
```

---

## 🎯 Learning Objectives

By completing this lab, you will:
1. Define and deploy non-confidential runtime settings using a declarative **ConfigMap**.
2. Securely store and inject sensitive database passwords and API tokens using a **Kubernetes Secret**.
3. Inject configurations directly into container processes as **Environment Variables** (`envFrom`).
4. Mount configuration files and secrets dynamically into container filesystems using **Volume Mounts**.
5. Log into running pods using `kubectl exec` to verify environment injection and inspect file permissions.
6. Safely manage cluster lifecycles using Azure CLI to eliminate unnecessary cloud billing.

---

## 📋 Prerequisites

* An active **Azure Subscription** ([Azure Free Account](https://azure.microsoft.com/free/) or Student Credits).
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

## 🚀 Declarative Step-by-Step Hands-On Guide

### Step 1: Declare the Application ConfigMap

ConfigMaps store non-sensitive configuration as key-value pairs. 

#### `kube-manifests/01-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: stacklayer-app-config
  labels:
    app: stacklayer-secure-app
data:
  APP_ENV: "production"
  APP_PORT: "80"
  LOG_LEVEL: "debug"
  FEATURE_FLAG_ANALYTICS: "true"
```

Apply the ConfigMap manifest:
```bash
# Deploy ConfigMap
kubectl apply -f kube-manifests/01-configmap.yaml

# Inspect ConfigMap details and stored keys
kubectl describe configmap stacklayer-app-config
```

---

### Step 2: Declare Sensitive Credentials with Kubernetes Secrets

Secrets store confidential data (such as database passwords, TLS certificates, and API tokens) encoded in Base64 or plain text via `stringData`.

#### `kube-manifests/02-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: stacklayer-app-secret
  labels:
    app: stacklayer-secure-app
type: Opaque
stringData:
  DB_USERNAME: "stacklayer_admin"
  DB_PASSWORD: "VaultPassword@2026!Azure"
  API_AUTH_TOKEN: "sec_live_948271039472619"
```

Apply the Secret manifest:
```bash
# Deploy Secret
kubectl apply -f kube-manifests/02-secret.yaml

# Verify Secret existence (values remain masked for security)
kubectl get secrets stacklayer-app-secret
```

---

### Step 3: Deploy the Workload with Injected Config & Secrets

This deployment demonstrates both industry-standard injection methods:
1. **Environment Variables:** Injecting whole ConfigMaps and Secrets as environment variables (`envFrom`).
2. **Volume Mounts:** Projecting secret data directly as read-only files in `/etc/secrets`.

#### `kube-manifests/03-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stacklayer-secure-deployment
  labels:
    app: stacklayer-secure-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: stacklayer-secure-app
  template:
    metadata:
      labels:
        app: stacklayer-secure-app
    spec:
      containers:
      - name: secure-app
        image: stacklayer/kubernetes:nginx-v1
        ports:
        - containerPort: 80
        # Method 1: Inject as Environment Variables
        envFrom:
        - configMapRef:
            name: stacklayer-app-config
        - secretRef:
            name: stacklayer-app-secret
        # Method 2: Mount as read-only files inside the pod
        volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secrets"
          readOnly: true
      volumes:
      - name: secret-volume
        secret:
          secretName: stacklayer-app-secret
```

#### `kube-manifests/04-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: stacklayer-secure-service
  labels:
    app: stacklayer-secure-app
spec:
  type: LoadBalancer
  selector:
    app: stacklayer-secure-app
  ports:
  - port: 80
    targetPort: 80
```

Apply both manifests to deploy the application:
```bash
# Apply deployment and load balancer service
kubectl apply -f kube-manifests/03-deployment.yaml
kubectl apply -f kube-manifests/04-service.yaml

# Confirm pods are running
kubectl get pods -l app=stacklayer-secure-app -o wide
```

---

### Step 4: Verify Environment Injection & File Mounts Inside Pod

Validate that your application container successfully received the configuration and secret values.

1. Retrieve one of your running pod names:
```bash
export POD_NAME=$(kubectl get pods -l app=stacklayer-secure-app -o jsonpath="{.items[0].metadata.name}")
echo "Testing Pod: $POD_NAME"
```

2. Verify injected environment variables:
```bash
# Query the container environment directly
kubectl exec $POD_NAME -- printenv | grep -E "APP_|DB_|API_"
```
*Expected output shows `APP_ENV=production`, `DB_USERNAME=stacklayer_admin`, and `FEATURE_FLAG_ANALYTICS=true` loaded directly into memory.*

3. Inspect the mounted secret files:
```bash
# List files inside the mounted secrets directory
kubectl exec $POD_NAME -- ls -la /etc/secrets

# Read secret file contents directly
kubectl exec $POD_NAME -- cat /etc/secrets/DB_USERNAME
```

---

## 🧹 Cost Safeguards & Cleanup

To protect your Azure credits after completing the lab, choose one of the options below:

### Option A: Stop Cluster (Pauses VM compute charges, preserves configuration)
```bash
# Stop the AKS cluster
az aks stop --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster

# Resume whenever you want to practice again:
# az aks start --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster
```

### Option B: Delete Resource Group (Permanent Teardown)
```bash
# Permanently delete the resource group and all contained resources
az group delete --name stacklayer-aks-rg --yes --no-wait
```

---

## 🤝 Community & Support
* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud tutorials on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <br />
  <h3>🔐 Secure by Design. Built for Production.</h3>
  <p style="max-width: 580px; line-height: 1.6; font-style: italic;">
    “Security is not an afterthought added before launch—it is an architectural discipline mastered one secret, one configuration, and one decoupled workload at a time.”
  </p>
  <br />
  <sub>Built with ❤️ by the <b>Stack Layer Engineering Team</b></sub>
</div>
