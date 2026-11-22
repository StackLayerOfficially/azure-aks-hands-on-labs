# Secure Kubernetes Ingress with Automated Let's Encrypt SSL/TLS on Azure AKS

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/). Manually issuing TLS keys, updating NGINX secrets, and tracking 90-day certificate expiration windows creates recurring operational toil and avoidable downtime risks. This lab turns your **Azure Kubernetes Service (AKS)** ingress layer into a self-renewing HTTPS edge using **Cert-Manager (`v1.21.3`)**, **Let's Encrypt ACME HTTP-01 automation** (Staging dry run + Production promotion), **NGINX Ingress Controller (`controller-v1.15.1`)**, and automated `308` HTTPS redirection across **Genesis Core (`v1`)** and **Velocity Edition (`v2`)** on `stacklayer.cyou`.

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Secure Kubernetes Ingress with Automated Let's Encrypt SSL/TLS on Azure AKS](https://stacklayer.blogspot.com/2026/11/secure-kubernetes-ingress-automated-lets-encrypt-ssl-tls-azure-aks.html)

---

## 📁 Directory Structure

```text
17-letsencrypt-ssl-tls-cert-manager-aks/
├── README.md                                     # Lab execution runbook, ACME HTTP-01 validation & TLS verification
└── kube-manifests/
    ├── 01-stacklayer-v1-app1-deployment.yaml     # App 1: Genesis Core (nginx-v1) backend pod deployment
    ├── 02-stacklayer-v1-app1-service.yaml        # Internal ClusterIP Service routing traffic to App 1 pods
    ├── 03-stacklayer-v2-app2-deployment.yaml     # App 2: Velocity Edition (nginx-v2) backend pod deployment
    ├── 04-stacklayer-v2-app2-service.yaml        # Internal ClusterIP Service routing traffic to App 2 pods
    ├── 05-certmanager-staging-clusterissuer.yaml # Let's Encrypt Staging HTTP-01 ClusterIssuer for safe dry runs
    ├── 06-certmanager-prod-clusterissuer.yaml    # Let's Encrypt Production HTTP-01 ClusterIssuer for trusted certs
    └── 07-stacklayer-tls-ingress.yaml            # Multi-host NGINX Ingress with Cert-Manager TLS & 308 HTTPS redirect
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

> **Lab Dependency Note:** Ensure your delegated Azure Public DNS Zone (`stacklayer.cyou`) is active in `stacklayer-aks-rg`, `curl` and `openssl` are available in your terminal, and you have a valid reachable email address for Let's Encrypt ACME registration (Let's Encrypt rejects placeholder domains such as `@example.com`).

---

## 🚀 Step-by-Step Execution Runbook

### Step-01: Verify NGINX Ingress External IP & Azure DNS Routing

Confirm the official [NGINX Ingress Controller (`controller-v1.15.1`)](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#azure) exposes an external Azure LoadBalancer IP (`4.213.194.46` in our live cluster) on ports `80` and `443`, and verify whether ExternalDNS is active before starting ACME validation:

```bash
# 1. Apply or verify the official NGINX Ingress Controller release
kubectl apply \
  -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml

kubectl get svc ingress-nginx-controller -n ingress-nginx

# 2. Check whether ExternalDNS controller is active in the cluster
kubectl get pods -n external-dns --ignore-not-found

# 3. Inspect active A records in your Azure DNS zone
az network dns record-set a list \
  --resource-group stacklayer-aks-rg \
  --zone-name stacklayer.cyou \
  -o table
```

> **DNS Note:** If `external-dns` is not running in your cluster, manually create Azure DNS `A` records for `app1` and `app2` pointing to your NGINX Ingress External IP (`4.213.194.46`) before proceeding to Step-04 so ACME HTTP-01 probes do not fail with `NXDOMAIN`.

---

### Step-02: Install Cert-Manager (`v1.21.3`) on AKS

Deploy the official CNCF [Cert-Manager `v1.21.3`](https://github.com/cert-manager/cert-manager/releases/tag/v1.21.3) release ([kubectl installation docs](https://cert-manager.io/docs/installation/kubectl/)) and wait for the controller, CA injector, and webhook pods to become ready:

```bash
# 1. Apply the official Cert-Manager v1.21.3 manifest (includes CRDs and RBAC)
kubectl apply \
  -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.3/cert-manager.yaml

# 2. Wait until all three Cert-Manager Deployments report Ready
kubectl rollout status deployment/cert-manager -n cert-manager
kubectl rollout status deployment/cert-manager-cainjector -n cert-manager
kubectl rollout status deployment/cert-manager-webhook -n cert-manager

# 3. Verify running pods and registered Cert-Manager CRDs
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

---

### Step-03: Deploy Web Workloads & Register Staging/Production ClusterIssuers

Deploy **Genesis Core (`v1`)** and **Velocity Edition (`v2`)** behind internal `ClusterIP` Services, customize `ACME_EMAIL`, and stream both Staging and Production `ClusterIssuer` manifests into AKS without committing personal email addresses into Git:

```bash
# 1. Deploy App 1 (Genesis Core) and App 2 (Velocity Edition) workloads
kubectl apply \
  -f kube-manifests/01-stacklayer-v1-app1-deployment.yaml \
  -f kube-manifests/02-stacklayer-v1-app1-service.yaml \
  -f kube-manifests/03-stacklayer-v2-app2-deployment.yaml \
  -f kube-manifests/04-stacklayer-v2-app2-service.yaml

# 2. Export a valid reachable email before registering ClusterIssuers
export ACME_EMAIL="youremail@yourdomain.com"

# 3. Safety guard: prevent applying placeholder ACME emails
if [ -z "$ACME_EMAIL" ] || [ "$ACME_EMAIL" = "youremail@yourdomain.com" ]; then
  echo "WARNING: Update ACME_EMAIL with a real email address before applying ClusterIssuers!"
else
  sed "s/REPLACE_WITH_YOUR_EMAIL/$ACME_EMAIL/g" kube-manifests/05-certmanager-staging-clusterissuer.yaml | kubectl apply -f -
  sed "s/REPLACE_WITH_YOUR_EMAIL/$ACME_EMAIL/g" kube-manifests/06-certmanager-prod-clusterissuer.yaml | kubectl apply -f -
  kubectl get clusterissuer
fi
```

---

### Step-04: Dry-Run SSL/TLS Issuance via Let's Encrypt Staging

Validate DNS propagation and port `80` ACME HTTP-01 reachability against `letsencrypt-staging` first so you never risk exhausting Let's Encrypt Production weekly rate limits:

```bash
# 1. Stream the multi-host Ingress targeting letsencrypt-staging first
sed "s/letsencrypt-prod/letsencrypt-staging/g" kube-manifests/07-stacklayer-tls-ingress.yaml | kubectl apply -f -

# 2. Take a single snapshot of Certificate, CertificateRequest, Order, and Challenge states
kubectl get certificate,certificaterequest,order,challenge

# 3. Watch Certificate readiness live (press Ctrl+C once READY shows True)
kubectl get certificate -w
```

> **Note:** Allow 30–60 seconds on initial apply for ExternalDNS (if enabled) and ACME HTTP-01 solver validation to complete.

---

### Step-05: Promote to Trusted Production SSL/TLS Certificate

Once the Staging certificate reaches `READY: True`, apply `07-stacklayer-tls-ingress.yaml` **first** so the `Certificate` object switches to `letsencrypt-prod`, then delete the Staging secret so Cert-Manager immediately reconciles a browser-trusted Production certificate inside `stacklayer-tls-secret`:

```bash
# 1. Apply the Production TLS Ingress FIRST so Certificate switches to letsencrypt-prod
kubectl apply -f kube-manifests/07-stacklayer-tls-ingress.yaml
sleep 3

# 2. Delete the cached Staging Secret so Cert-Manager replaces it with Production TLS
kubectl delete secret stacklayer-tls-secret --ignore-not-found

# 3. Wait until the Production Certificate reports Ready=True (~30-60 seconds)
kubectl wait \
  --for=condition=Ready certificate/stacklayer-tls-secret \
  --timeout=180s

# 4. Confirm CertificateRequest issuer is letsencrypt-prod and inspect renewal metadata
kubectl get certificaterequest
kubectl describe certificate stacklayer-tls-secret | grep -A 8 "Status:"
kubectl get secret stacklayer-tls-secret
```

---

### Step-06: Live X.509 Certificate Chain, 308 Redirect & HTTPS Verification

Inspect the live SAN certificate served by NGINX Ingress over SNI, verify `308 Permanent Redirect` upgrades from HTTP to HTTPS, and validate both endpoints return clean edition headers (`Web Ver:1` and `Web Ver:2`):

```bash
# 1. Inspect live Let's Encrypt Issuer, Subject, Validity dates, and SAN hosts via OpenSSL
echo | openssl s_client \
  -connect app1.stacklayer.cyou:443 \
  -servername app1.stacklayer.cyou | openssl x509 -noout -issuer -subject -dates -ext subjectAltName

# 2. Verify automatic HTTP to HTTPS 308 Permanent Redirect
curl -I http://app1.stacklayer.cyou

# 3. Smoke-test HTTPS App 1 (Genesis Core - v1)
curl -s https://app1.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1

# 4. Smoke-test HTTPS App 2 (Velocity Edition - v2)
curl -s https://app2.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1
```

---

## 🧹 Cost Safeguards & Cleanup

Delete the TLS Ingress **before** deleting `stacklayer-tls-secret` (so Cert-Manager releases ownership of the Secret and ExternalDNS removes `app1` and `app2` records cleanly):

```bash
kubectl delete \
  -f kube-manifests/07-stacklayer-tls-ingress.yaml \
  -f kube-manifests/04-stacklayer-v2-app2-service.yaml \
  -f kube-manifests/03-stacklayer-v2-app2-deployment.yaml \
  -f kube-manifests/02-stacklayer-v1-app1-service.yaml \
  -f kube-manifests/01-stacklayer-v1-app1-deployment.yaml

kubectl delete secret stacklayer-tls-secret --ignore-not-found
```

Then select your preferred Azure teardown option:

* **Option A: Stop AKS Compute Nodes (Preserves Cert-Manager, ClusterIssuers & DNS Zone)**
  ```bash
  az aks stop \
    --resource-group stacklayer-aks-rg \
    --name stacklayer-aks-cluster
  ```
  > *Note: Pausing AKS halts VM compute charges, while keeping your Static Public IP (`4.213.194.46`) and Azure DNS Zone incurs minor ongoing Azure fees.*

* **Option B: Delete Resource Group (Zero Ongoing Cost)**
  ```bash
  az group delete \
    --name stacklayer-aks-rg \
    --yes \
    --no-wait
  ```
  > *Note: Permanently removes the AKS cluster, Cert-Manager resources, Public IPs, and Azure DNS Zone inside `stacklayer-aks-rg`.*

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <h3>🌐 Automated Zero-Touch HTTPS at the Kubernetes Edge</h3>
  <p><i>“When TLS certificates are declared as Kubernetes state, every route stays encrypted by design and expiration outages become history.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
