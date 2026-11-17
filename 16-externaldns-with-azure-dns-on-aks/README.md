# Automate Kubernetes DNS Records with ExternalDNS and Azure DNS on AKS

Welcome to the hands-on cloud operations lab by [Stack Layer](https://stacklayer.blogspot.com/). In dynamic Kubernetes platforms, manually updating Azure DNS tables whenever LoadBalancer IPs change or new host-based Ingress rules deploy introduces operational drift and slows down releases. This lab turns your Azure Kubernetes Service (AKS) cluster into a passwordless, self-healing DNS control plane using **ExternalDNS (`v0.21.0`)**, **Microsoft Entra Workload Identity (OIDC)**, **Azure Public DNS**, and the **NGINX Ingress Controller (`controller-v1.15.1`)** across three Stack Layer application editions (`v5`, `v1`, and `v2`).

---

## 📖 Companion Blog Guide

For in-depth architectural diagrams, CoreDNS resolution packet walks, comparison tables, and Azure Portal visual verification, refer to our companion publication:

* Official Guide: [Automate Kubernetes DNS Records with ExternalDNS and Azure DNS on AKS](https://stacklayer.blogspot.com/2026/11/externaldns-azure-dns-automation-kubernetes-aks.html)

---

## 📁 Directory Structure

```text
16-externaldns-with-azure-dns-on-aks/
├── README.md                                  # Lab execution runbook, OIDC federation & verification guide
└── kube-manifests/
    ├── 01-externaldns-serviceaccount.yaml     # Workload Identity ServiceAccount mapped via client-id annotation
    ├── 02-externaldns-clusterrole.yaml        # Read-only RBAC ClusterRole with EndpointSlices & Ingress watch access
    ├── 03-externaldns-clusterrolebinding.yaml # Binds external-dns ClusterRole to the external-dns ServiceAccount
    ├── 04-externaldns-deployment.yaml         # ExternalDNS v0.21.0 controller configured with OIDC & --policy=sync
    ├── 05-stacklayer-v5-direct-deployment.yaml# Scenario A: Apex Edition (nginx-v5) pod deployment
    ├── 06-stacklayer-v5-direct-service.yaml   # Scenario A: Standalone LoadBalancer Service annotated for direct.stacklayer.cyou
    ├── 07-stacklayer-v1-app1-deployment.yaml  # Scenario B: Genesis Core (nginx-v1) pod deployment
    ├── 08-stacklayer-v1-app1-service.yaml     # Scenario B: Internal ClusterIP Service backing App 1 pods
    ├── 09-stacklayer-v2-app2-deployment.yaml  # Scenario B: Velocity Edition (nginx-v2) pod deployment
    ├── 10-stacklayer-v2-app2-service.yaml     # Scenario B: Internal ClusterIP Service backing App 2 pods
    └── 11-stacklayer-auto-ingress.yaml        # Scenario B: Multi-host NGINX Ingress triggering auto-synced DNS records
```

---

## 📋 Prerequisites

* An active **Azure Subscription** (Free Trial, Student Credits, or Pay-As-You-Go).
* A running **Azure AKS Cluster** (provisioned as shown in [Deploy Your First Containerized App on Azure Kubernetes Service (AKS)](https://stacklayer.blogspot.com/2026/08/azure-aks-tutorial-deploy-first-app.html)
.
Connect your local terminal to your running cluster:

```bash
# Connect Azure CLI to your running AKS cluster
az aks get-credentials --resource-group stacklayer-aks-rg --name stacklayer-aks-cluster --overwrite-existing

# Verify cluster connectivity and healthy nodes
kubectl cluster-info
kubectl get nodes
```

> **Lab Dependency Note:** Ensure your delegated Azure Public DNS Zone (`stacklayer.cyou`) already exists inside resource group `stacklayer-aks-rg` and that `nslookup` is available in your terminal.

---

## 🚀 Step-by-Step Execution Runbook

### Step-01: Verify Delegated Azure DNS Zone & NGINX Ingress Controller

Confirm your delegated `stacklayer.cyou` DNS Zone is active, remove any leftover manual `A` records so ExternalDNS establishes clean `TXT` ownership, and verify or deploy the official [NGINX Ingress Controller (`controller-v1.15.1`)](https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/index.md#azure) (`20.244.59.188` in our reference run):

```bash
# 1. Confirm the Azure DNS Zone exists and inspect existing A records
az network dns zone show \
  --resource-group stacklayer-aks-rg \
  --name stacklayer.cyou \
  --query "nameServers" \
  -o tsv

az network dns record-set a list \
  --resource-group stacklayer-aks-rg \
  --zone-name stacklayer.cyou \
  -o table

# 2. (Optional) Remove old manual app1/app2/@ A records so ExternalDNS owns them cleanly
az network dns record-set a delete --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou --name "app1" --yes 2>/dev/null || true
az network dns record-set a delete --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou --name "app2" --yes 2>/dev/null || true
az network dns record-set a delete --resource-group stacklayer-aks-rg --zone-name stacklayer.cyou --name "@" --yes 2>/dev/null || true

# 3. Install or verify official NGINX Ingress Controller and capture its External IP
kubectl apply \
  -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml

kubectl get svc ingress-nginx-controller -n ingress-nginx
```

---

### Step-02: Enable OIDC Issuer & Workload Identity on AKS

Enable the [AKS OIDC Issuer](https://learn.microsoft.com/en-us/azure/aks/use-oidc-issuer) and [Microsoft Entra Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) webhook so pods authenticate without stored client secrets:

```bash
# 1. Enable OIDC issuer and Workload Identity on your running AKS cluster
az aks update \
  --resource-group stacklayer-aks-rg \
  --name stacklayer-aks-cluster \
  --enable-oidc-issuer \
  --enable-workload-identity

# 2. Retrieve and export your cluster's unique OIDC Issuer URL
AKS_OIDC_ISSUER=$(az aks show \
  --resource-group stacklayer-aks-rg \
  --name stacklayer-aks-cluster \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv)

echo "AKS OIDC Issuer URL: $AKS_OIDC_ISSUER"
```

---

### Step-03: Create Managed Identity, Grant Scoped RBAC & Federate OIDC Trust

Create User-Assigned Managed Identity `external-dns-uami`, assign least-privilege `DNS Zone Contributor` strictly scoped to `stacklayer.cyou` (passing `--assignee-principal-type ServicePrincipal` per [Azure RBAC CLI guidance](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-cli)), and federate it to ServiceAccount `external-dns/external-dns`:

```bash
# 1. Create the User-Assigned Managed Identity
az identity create \
  --resource-group stacklayer-aks-rg \
  --name external-dns-uami

# 2. Capture all required Azure and OIDC variables
AKS_OIDC_ISSUER=$(az aks show \
  --resource-group stacklayer-aks-rg \
  --name stacklayer-aks-cluster \
  --query "oidcIssuerProfile.issuerUrl" \
  -o tsv)

IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group stacklayer-aks-rg \
  --name external-dns-uami \
  --query "clientId" \
  -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group stacklayer-aks-rg \
  --name external-dns-uami \
  --query "principalId" \
  -o tsv)

SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)
TENANT_ID=$(az account show --query "tenantId" -o tsv)

echo "Client ID: $IDENTITY_CLIENT_ID"
echo "OIDC Issuer: $AKS_OIDC_ISSUER"

# 3. Assign "DNS Zone Contributor" RBAC role scoped strictly to stacklayer.cyou
az role assignment create \
  --role "DNS Zone Contributor" \
  --assignee-object-id "$IDENTITY_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/stacklayer-aks-rg/providers/Microsoft.Network/dnsZones/stacklayer.cyou"

# 4. Create Federated Identity Credential for Kubernetes ServiceAccount external-dns/external-dns
az identity federated-credential create \
  --name "external-dns-federated-cred" \
  --resource-group stacklayer-aks-rg \
  --identity-name external-dns-uami \
  --issuer "$AKS_OIDC_ISSUER" \
  --subject "system:serviceaccount:external-dns:external-dns" \
  --audience "api://AzureADTokenExchange"
```

---

### Step-04: Configure `azure.json` Secret & Deploy ExternalDNS (`v0.21.0`)

Build the passwordless `azure-config-file` Secret in memory (`useWorkloadIdentityExtension: true`), inject `$IDENTITY_CLIENT_ID` on the fly without modifying `kube-manifests/01-externaldns-serviceaccount.yaml` on disk, and deploy [ExternalDNS `v0.21.0`](https://github.com/kubernetes-sigs/external-dns/releases/tag/v0.21.0) with `--policy=sync`:

```bash
# 1. Refresh and validate shell variables
export SUBSCRIPTION_ID=$(az account show --query "id" -o tsv)
export TENANT_ID=$(az account show --query "tenantId" -o tsv)
export IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group stacklayer-aks-rg \
  --name external-dns-uami \
  --query "clientId" \
  -o tsv)

if [ -z "$TENANT_ID" ] || [ -z "$SUBSCRIPTION_ID" ] || [ -z "$IDENTITY_CLIENT_ID" ]; then
  echo "WARNING: Missing Azure IDs. Verify az login and Step-03 before continuing."
else
  echo "Verified Tenant: $TENANT_ID | Subscription: $SUBSCRIPTION_ID | Client: $IDENTITY_CLIENT_ID"
fi

# 2. Create dedicated external-dns namespace idempotently
kubectl create namespace external-dns \
  --dry-run=client \
  -o yaml | kubectl apply -f -

# 3. Store passwordless azure.json Secret cleanly in memory
AZURE_JSON=$(printf '{"tenantId":"%s","subscriptionId":"%s","resourceGroup":"stacklayer-aks-rg","useWorkloadIdentityExtension":true}' "$TENANT_ID" "$SUBSCRIPTION_ID")

kubectl create secret generic azure-config-file \
  --namespace external-dns \
  --from-literal=azure.json="$AZURE_JSON" \
  --dry-run=client \
  -o yaml | kubectl apply -f -

# 4. Stream ServiceAccount with IDENTITY_CLIENT_ID injected (keeps local YAML template clean)
sed "s/REPLACE_WITH_IDENTITY_CLIENT_ID/$IDENTITY_CLIENT_ID/g" kube-manifests/01-externaldns-serviceaccount.yaml | kubectl apply -f -

# 5. Apply RBAC and ExternalDNS Deployment manifests
kubectl apply \
  -f kube-manifests/02-externaldns-clusterrole.yaml \
  -f kube-manifests/03-externaldns-clusterrolebinding.yaml \
  -f kube-manifests/04-externaldns-deployment.yaml

# 6. Verify pod readiness and synchronization logs
kubectl rollout status deployment/external-dns -n external-dns
kubectl get pods -n external-dns
kubectl logs -n external-dns -l app=external-dns --tail=15
```

---

### Step-05: Scenario A — Automate LoadBalancer Service DNS (`direct.stacklayer.cyou`)

Deploy **Apex Edition** (`stacklayer/kubernetes:nginx-v5`, reporting `Web Ver:5`) with an annotated `LoadBalancer` Service (`4.187.186.61` in our reference run) so ExternalDNS creates `direct.stacklayer.cyou` automatically:

```bash
kubectl apply \
  -f kube-manifests/05-stacklayer-v5-direct-deployment.yaml \
  -f kube-manifests/06-stacklayer-v5-direct-service.yaml

# Wait ~45 seconds for Azure LB IP assignment and ExternalDNS sync cycle
kubectl get svc stacklayer-direct-v5-lb-service
kubectl logs -n external-dns -l app=external-dns --tail=10
```

---

### Step-06: Scenario B — Automate Multi-Host NGINX Ingress Routing (`v1` & `v2`)

Deploy **Genesis Core** (`stacklayer/kubernetes:nginx-v1`, `Web Ver:1`) and **Velocity Edition** (`stacklayer/kubernetes:nginx-v2`, `Web Ver:2`) behind internal `ClusterIP` Services, then apply the multi-host Ingress binding `app1.stacklayer.cyou` and `app2.stacklayer.cyou` to your single NGINX Ingress Public IP (`20.244.59.188`):

```bash
kubectl apply \
  -f kube-manifests/07-stacklayer-v1-app1-deployment.yaml \
  -f kube-manifests/08-stacklayer-v1-app1-service.yaml \
  -f kube-manifests/09-stacklayer-v2-app2-deployment.yaml \
  -f kube-manifests/10-stacklayer-v2-app2-service.yaml \
  -f kube-manifests/11-stacklayer-auto-ingress.yaml

# Wait ~45 seconds for Ingress ADDRESS population and ExternalDNS sync
kubectl get ingress stacklayer-auto-ingress

# Inspect all auto-created A and companion TXT ownership records in Azure DNS
az network dns record-set list \
  --resource-group stacklayer-aks-rg \
  --zone-name stacklayer.cyou \
  -o table
```

---

### Step-07: Live Verification & `--policy=sync` Deletion Proof

Validate DNS resolution and live HTTP version output across all three automated endpoints, then delete the standalone Service to verify self-healing DNS record removal:

```bash
# 1. Verify public DNS resolution
nslookup direct.stacklayer.cyou
nslookup app1.stacklayer.cyou
nslookup app2.stacklayer.cyou

# 2. Smoke-test HTTP edition headers (expected: Web Ver:5, Web Ver:1, Web Ver:2)
curl -s http://direct.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1
curl -s http://app1.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1
curl -s http://app2.stacklayer.cyou | grep -oi "Web Ver:[0-9]" | head -n 1

# 3. Delete standalone LoadBalancer Service to prove --policy=sync purges direct & a-direct automatically
kubectl delete -f kube-manifests/06-stacklayer-v5-direct-service.yaml

# Wait ~35 seconds, then inspect logs and Azure DNS records
kubectl logs -n external-dns -l app=external-dns --tail=8
az network dns record-set list \
  --resource-group stacklayer-aks-rg \
  --zone-name stacklayer.cyou \
  -o table
```

---

## 🧹 Cost Safeguards & Cleanup

First, flush remaining test workloads while `external-dns` is still running so ExternalDNS cleanly purges `app1` and `app2` records from Azure DNS before node shutdown:

```bash
kubectl delete \
  -f kube-manifests/11-stacklayer-auto-ingress.yaml \
  -f kube-manifests/10-stacklayer-v2-app2-service.yaml \
  -f kube-manifests/09-stacklayer-v2-app2-deployment.yaml \
  -f kube-manifests/08-stacklayer-v1-app1-service.yaml \
  -f kube-manifests/07-stacklayer-v1-app1-deployment.yaml \
  -f kube-manifests/05-stacklayer-v5-direct-deployment.yaml
```

Then select your preferred Azure teardown option:

* **Option A: Stop AKS Compute Nodes (Preserves Identity, ExternalDNS & DNS Zone)**
  ```bash
  az aks stop \
    --resource-group stacklayer-aks-rg \
    --name stacklayer-aks-cluster
  ```
  > *Note: Pausing AKS halts VM compute charges, while keeping your Static Public IP (`20.244.59.188`) and Azure DNS Zone incurs minor ongoing Azure fees.*

* **Option B: Delete Resource Group (Zero Ongoing Cost)**
  ```bash
  az group delete \
    --name stacklayer-aks-rg \
    --yes \
    --no-wait
  ```
  > *Note: Removes AKS, Managed Identity, Public IPs, and Azure DNS Zone cleanly. Restore default registrar nameservers later if reassigning `stacklayer.cyou`.*

---

## 🤝 Community & Support

* 💬 Have questions or run into an issue? Join the conversation in [GitHub Discussions](https://github.com/StackLayerOfficially/azure-aks-hands-on-labs/discussions).
* 🌐 Explore more hands-on cloud operations guides on [Stack Layer](https://stacklayer.blogspot.com/).

---

<div align="center">
  <h3>🌐 Declarative DNS Without Static Secrets</h3>
  <p><i>“When Kubernetes manifests become the single source of truth for both workloads and global DNS, identity remains passwordless and infrastructure heals itself.”</i></p>
  <br>
  <p>Built with ❤️ by the <b>Stack Layer Engineering Team</b></p>
</div>
