# Create AKS Cluster

## To-Do List
- Create Azure AKS Cluster
- Install Azure CLI, kubectl CLI on local desktop and connect to Azure AKS Cluster using Azure CLI from local desktop
- Explore Azure AKS Cluster Resources using kubectl cli and Azure Portal
- Deploy Application on AKS Cluster and test
- Delete Kubernetes resources deployed as part of this project

## Step-01: Create AKS Cluster
- Create Kubernetes Cluster
### Basics
- **Subscription:** Your-Paid-Subscription
- **Resource Group:** Creat New: aks-rg
- **Cluster preset configuration:** Dev/Test
- **Kubernetes Cluster Name:** aks-cluster  
- **Region:** Central India
- **Fleet Manager:** NONE (LEAVE TO DEFAULT)
- **Availability zones:** NONE (LEAVE TO DEFAULT)
- **AKS Pricing Tier:** Free
- **Kubernetes Version:** Select what ever is latest stable version
- **Automatic upgrade:** Enabled with patch (recommended)
- **Node security channel type:** Node Image (LEAVE TO DEFAULT)
  - **Security channel scheduler:** Every week on Sunday (recommended)
- **Authentication and Authorization:** 	Local accounts with Kubernetes RBAC    
### Node Pools
- In Nodepools, **Update node pool**
  - **Node pool name:** agentpool (LEAVE TO DEFAULT)
  - **Mode:** system (LEAVE TO DEFAULT)
  - **OS SKU:** Ubuntu Linux  (LEAVE TO DEFAULT)
  - **Availability zones:** ZONES 1,2,3 (LEAVE TO DEFAULT)
  - **Node size:** Standard_DS2_v2 (2 vcpus, 7 GiB memory)
  - **Scale method:** Autoscale
  - **Minimum node count:** 2
  - **Maximum node count:** 5
  - REST ALL LEAVE TO DEFAULTS
  - Click on **UPDATE**
- REST ALL LEAVE TO DEFAULTS
### Networking
- **Private access**
  - Enable private cluster: UNCHECKED (LEAVE TO DEFAULTS)
- **Public access**
  - Set authorized IP ranges: UNCHECKED (LEAVE TO DEFAULTS)
- **Container networking:** 
  - Network configuration: Azure CNI Overlay
- **Bring your own Azure virtual network:** CHECKED  
  - Review all the auto-populated details 
  - Virtual Network
  - Cluster Subnet
  - Kubernetes Service address range
  - Kubernetes DNS Service IP Address
  - DNS Name prefix
- **Network Policy:** None (LEAVE TO DEFAULTS)
- **Load balancer:** Standard
### Integrations
  - **Azure Container Registry:** None
  - All leave to defaults
### Monitoring
  - All leave to defaults
### Security
  - All leave to defaults  
### Advanced
  - All leave to defaults  
### Tags
  - All leave to defaults 
### Review + Create
  - Click on **Create**


## Step-02: Explore the AKS cluster on Azure Management Console
- Explore the following features 
  - Overview
  - Kubernetes Resources
  - Settings
  - Monitoring
  - Automation



## Step-03: Local Desktop - Install Azure CLI and Azure AKS CLI
```t

The easiest way to install the Azure CLI is through a script maintained by the Azure CLI team. This script runs all installation commands in one step. This script is downloaded via curl and piped directly to bash to install the CLI.

# Install Azure CLI (Ubuntu)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify AZ CLI version
az --version

# Install Azure AKS CLI
sudo az aks install-cli

# Get kubectl client version only (shows client version only (no server required))
kubectl version --client=true

# Get kubectl version (Displays both client CLI and k8s server versions)
kubectl version

# Login to Azure
az login

# Configure Cluster Creds (kube config)
az aks get-credentials --resource-group aks-rg --name aks-cluster

# List AKS Nodes
kubectl get nodes 
kubectl get nodes -o wide


# List Namespaces
kubectl get namespaces
kubectl get ns

# List Pods from all namespaces
kubectl get pods --all-namespaces

# List all k8s objects from Cluster Control plane
kubectl get all --all-namespaces
```
- **Reference Documentation Links**
  - [Azure Command-Line Interface (CLI) documentation](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)
  - [Azure Kubernetes Service Commands](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest)

## Step-04: Deploy Sample Application and Test
```t
# Deploy Application
kubectl apply -f kube-manifests/

# Verify Pods
kubectl get pods

# Verify Deployment
kubectl get deployment

# Verify Service (Make a note of external ip)
kubectl get service

# Access Application
http://<External-IP-from-get-service-output>

```

## Step-05: Delete Resources
```t
# Delete Applications
kubectl delete -f kube-manifests/
```
## Notes

- **Make sure to replace placeholders (e.g., your-Paid-Subscription, your_cluster_name, your_region, your_resource_group_name...etc) with your actual Configuration.**

- **This is a basic setup for demonstration purposes. In a production environment, you should follow best practices for security and performance.**

