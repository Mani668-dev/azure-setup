# Azure Kubernetes Service (AKS) Setup Guide

## 📋 Overview

Azure Kubernetes Service (AKS) is a managed Kubernetes service that simplifies deploying and managing containerized applications. This guide covers creating an AKS cluster, configuring Azure AD integration, attaching ACR, and deploying applications.

## Prerequisites

- Completed [Azure AD Setup](./01-azure-ad-setup.md)
- Completed [ACR Setup](./02-acr-setup.md)
- Azure CLI installed and logged in
- kubectl installed locally
- Service principal and ACR details from previous guides

## Table of Contents

1. [Create AKS Cluster](#1-create-aks-cluster)
2. [Configure Azure AD Integration](#2-configure-azure-ad-integration)
3. [Attach ACR to AKS](#3-attach-acr-to-aks)
4. [Configure kubectl Access](#4-configure-kubectl-access)
5. [Configure Node Pools](#5-configure-node-pools)
6. [Deploy Sample Application](#6-deploy-sample-application)
7. [Configure Networking](#7-configure-networking)
8. [Enable Monitoring and Logging](#8-enable-monitoring-and-logging)
9. [Verification](#9-verification)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Create AKS Cluster

### Step 1.1: Set Environment Variables

```bash
# Set variables
RESOURCE_GROUP="rg-aks-demo"
LOCATION="eastus"
AKS_CLUSTER_NAME="aks-demo-cluster"
ACR_NAME="acrdemoacr"
NODE_COUNT=2
VM_SIZE="Standard_D2s_v3"

# From Azure AD setup
AKS_SP_CLIENT_ID="your-service-principal-client-id"
AKS_SP_CLIENT_SECRET="your-service-principal-secret"
```

### Step 1.2: Create AKS Cluster via Azure Portal

**Via Azure Portal:**

1. Navigate to [Azure Portal](https://portal.azure.com)
2. In the search bar, type **"Kubernetes services"**
3. Click **"Kubernetes services"**
4. Click **"+ Create"** > **"Create a Kubernetes cluster"**

📸 **Screenshot: Kubernetes services page**
```
Kubernetes services > + Create > Create a Kubernetes cluster
```


#### Basics Tab

5. Fill in the **Basics** tab:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Cluster preset configuration**: Select "Dev/Test" or "Production" based on needs
   - **Kubernetes cluster name**: `aks-demo-cluster`
   - **Region**: `East US` (same as resource group)
   - **Availability zones**: Select zones 1, 2, 3 (for production)
   - **AKS pricing tier**: Free or Standard
   - **Kubernetes version**: Leave as default (latest stable)
   - **Automatic upgrade**: Select "Enabled with patch"
   - **Authentication and Authorization**: Select "Local accounts with Kubernetes RBAC"

📸 **Screenshot: Create Kubernetes cluster - Basics tab (Part 1)**
```
Basics tab - Project details
Subscription: [Your subscription]
Resource group: rg-aks-demo
Cluster preset configuration: Dev/Test (default)

Cluster details
Kubernetes cluster name: aks-demo-cluster
Region: East US
Availability zones: ☑ 1  ☑ 2  ☑ 3
AKS pricing tier: Free ▼
Kubernetes version: 1.28.5 (default) ▼
Automatic upgrade: Enabled with patch (recommended) ▼
```

📸 **Screenshot: Create Kubernetes cluster - Basics tab (Part 2)**
```
Authentication and Authorization
Authentication and Authorization: Local accounts with Kubernetes RBAC ▼

Primary node pool
Node size: Standard_D2s_v3 (2 vCPUs, 8 GB memory) - Change size
Scale method: • Autoscale  ○ Manual
Node count range: 1 to 5
```

6. Under **Primary node pool**:
   - **Node size**: Click "Change size" and select `Standard_D2s_v3` or similar
   - **Scale method**: Select "Manual" for simplicity (or Autoscale for production)
   - **Node count**: `2`

7. Click **"Next: Node pools >"**


#### Node Pools Tab

8. In the **Node pools** tab:
   - Review the default node pool (agentpool)
   - You can add additional node pools later if needed
   - Click **"Next: Networking >"**

📸 **Screenshot: Node pools tab**
```
Node pools tab
Name        | OS    | Mode    | Node size          | Node count
agentpool   | Linux | System  | Standard_D2s_v3   | 2

+ Add node pool (optional)
[Next: Networking >]
```

#### Networking Tab

9. In the **Networking** tab:
   - **Network configuration**: Select "Azure CNI" (recommended) or "kubenet" (simpler)
   - **Bring your own Azure virtual network**: Leave unchecked (creates new VNet)
   - **DNS name prefix**: Leave default or customize
   - **Load balancer**: Standard
   - **Enable HTTP application routing**: Unchecked (deprecated)
   - **Enable private cluster**: Unchecked (for now; enable for production)

📸 **Screenshot: Networking tab**
```
Networking tab
Network configuration: • Azure CNI  ○ kubenet
Bring your own Azure virtual network: ☐
Virtual network: aks-demo-cluster-vnet (new)
Cluster subnet: default (10.224.0.0/16)
Kubernetes service address range: 10.0.0.0/16
Kubernetes DNS service IP address: 10.0.0.10
DNS name prefix: aks-demo-cluster-dns

Advanced networking
Load balancer: Standard ▼
Enable private cluster: ☐
Set authorized IP ranges: ☐
Network policy: None ▼
```

10. Click **"Next: Integrations >"**


#### Integrations Tab

11. In the **Integrations** tab:
    - **Container registry**: Select your ACR (`acrdemoacr`)
    - **Azure Monitor Container insights**: Select "Enabled" (recommended)
    - **Azure Policy**: Unchecked (optional)

📸 **Screenshot: Integrations tab**
```
Integrations tab
Container registry: acrdemoacr ▼
This grants AcrPull permission to the selected registry

Azure Monitor
Container insights: • Enabled  ○ Disabled
Log Analytics workspace: DefaultWorkspace-xxx-EUS (new)

Azure Policy: ☐ Enable
```

12. Click **"Next: Advanced >"**

#### Advanced Tab

13. In the **Advanced** tab:
    - **Node resource group**: Leave default
    - **Enable cluster autoscaler**: Uncheck for now
    - **Infrastructure encryption**: Uncheck (optional security feature)

📸 **Screenshot: Advanced tab**
```
Advanced tab
Node resource group: MC_rg-aks-demo_aks-demo-cluster_eastus (default)

Agent pool settings
Enable cluster autoscaler: ☐
Max pods per node: 30 ▼

Security
Infrastructure encryption: ☐
```

14. Click **"Next: Tags >"**

#### Tags Tab

15. Add tags (optional):
    - **Name**: `Environment`, **Value**: `Development`
    - **Name**: `Project`, **Value**: `AKS-Demo`

📸 **Screenshot: Tags tab**
```
Tags tab
Name            | Value
Environment     | Development
Project         | AKS-Demo

+ Add tag
[Next: Review + create >]
```

16. Click **"Review + create"**


#### Review + Create Tab

17. Review all settings
18. Click **"Create"**

📸 **Screenshot: Review + create tab**
```
Review + create tab
✓ Validation passed

Basics
Subscription: Your subscription
Resource group: rg-aks-demo
Kubernetes cluster name: aks-demo-cluster
Region: East US
Kubernetes version: 1.28.5

Node pools
agentpool: 2 nodes, Standard_D2s_v3

Networking
Network configuration: Azure CNI
Container registry: acrdemoacr

[Create button]
```

19. Deployment will take 5-10 minutes
20. Wait for **"Your deployment is complete"** message
21. Click **"Go to resource"**

📸 **Screenshot: Deployment complete**
```
Your deployment is complete
Deployment name: aks-demo-cluster
Start time: 6/21/2026, 11:00:00 AM
Subscription: Your subscription
Resource group: rg-aks-demo

Deployment details
✓ microsoft.aks-aks-demo-cluster

[Go to resource button]
```

### Step 1.3: Create AKS Cluster via Azure CLI

**Create with ACR integration (Recommended):**

```bash
# Create AKS cluster with ACR attached
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $VM_SIZE \
  --enable-managed-identity \
  --attach-acr $ACR_NAME \
  --network-plugin azure \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --zones 1 2 3

# This command:
# - Creates AKS cluster with managed identity
# - Attaches ACR (grants AcrPull permission)
# - Uses Azure CNI networking
# - Enables monitoring
# - Generates SSH keys for nodes
# - Distributes nodes across availability zones
```


**Create with service principal (Alternative):**

```bash
# If using service principal instead of managed identity
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --node-count $NODE_COUNT \
  --node-vm-size $VM_SIZE \
  --service-principal $AKS_SP_CLIENT_ID \
  --client-secret $AKS_SP_CLIENT_SECRET \
  --attach-acr $ACR_NAME \
  --network-plugin azure \
  --enable-addons monitoring \
  --generate-ssh-keys

echo "AKS cluster creation started. This will take 5-10 minutes..."
```

**Expected Output:**

```json
{
  "aadProfile": null,
  "addonProfiles": {
    "omsagent": {
      "enabled": true
    }
  },
  "fqdn": "aks-demo-cluster-xxx.hcp.eastus.azmk8s.io",
  "kubernetesVersion": "1.28.5",
  "location": "eastus",
  "name": "aks-demo-cluster",
  "nodeResourceGroup": "MC_rg-aks-demo_aks-demo-cluster_eastus",
  "provisioningState": "Succeeded"
}
```

---

## 2. Configure Azure AD Integration

### Step 2.1: Enable Azure AD RBAC (Optional)

For Azure AD-based authentication and authorization.

**Via Azure CLI:**

```bash
# Get Azure AD tenant ID
TENANT_ID=$(az account show --query tenantId -o tsv)

# Enable Azure AD integration
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --enable-aad \
  --aad-admin-group-object-ids <your-admin-group-object-id> \
  --aad-tenant-id $TENANT_ID

# Note: You need to create an Azure AD group for AKS admins first
```

### Step 2.2: Create Azure AD Admin Group

**Via Azure Portal:**

1. Go to **Azure Active Directory**
2. Click **"Groups"** > **"+ New group"**
3. Fill in:
   - **Group type**: Security
   - **Group name**: `aks-administrators`
   - **Group description**: Administrators for AKS cluster
4. Click **"Create"**

📸 **Screenshot: Create Azure AD group**
```
New Group
Group type: Security ▼
Group name: aks-administrators
Group description: Administrators for AKS cluster
Members: [Add members]
[Create button]
```

5. Add yourself and other admins as members
6. Copy the **Object ID** of the group

📸 **Screenshot: Group overview**
```
aks-administrators | Overview
Object ID: zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
```


### Step 2.3: Assign Kubernetes Roles

**Via Azure CLI:**

```bash
# Assign Azure Kubernetes Service RBAC Cluster Admin role
CURRENT_USER=$(az account show --query user.name -o tsv)

az role assignment create \
  --assignee $CURRENT_USER \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --scope $(az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query id -o tsv)

echo "Cluster Admin role assigned"
```

---

## 3. Attach ACR to AKS

### Step 3.1: Verify ACR is Attached

If you used `--attach-acr` during cluster creation, ACR is already attached.

**Verify via Azure CLI:**

```bash
# Check if ACR is attached
az aks check-acr \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --acr ${ACR_NAME}.azurecr.io

# Expected output:
# ACR connectivity test passed
```

### Step 3.2: Attach ACR to Existing Cluster

If ACR wasn't attached during creation:

**Via Azure Portal:**

1. Go to your AKS cluster
2. Click **"Settings"** > **"Integrations"**
3. Under **"Container registry"**, click **"Select"**
4. Choose your ACR (`acrdemoacr`)
5. Click **"Save"**

📸 **Screenshot: AKS Integrations**
```
aks-demo-cluster | Integrations
Container registry: [Select] → acrdemoacr
This grants AcrPull permission to the cluster's managed identity
[Save button]
```

**Via Azure CLI:**

```bash
# Attach ACR to AKS
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --attach-acr $ACR_NAME

echo "ACR attached to AKS successfully"

# Verify
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query "servicePrincipalProfile"
```

### Step 3.3: Verify RBAC Permissions

```bash
# Get AKS managed identity
AKS_IDENTITY=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query "identityProfile.kubeletidentity.objectId" -o tsv)

# Get ACR resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

# Check role assignment
az role assignment list \
  --assignee $AKS_IDENTITY \
  --scope $ACR_ID \
  --query "[].{Role:roleDefinitionName}" -o table

# Should show:
# Role
# -------
# AcrPull
```


---

## 4. Configure kubectl Access

### Step 4.1: Install kubectl (if not installed)

**On Linux:**

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make it executable
chmod +x kubectl

# Move to bin directory
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

**On macOS:**

```bash
# Using Homebrew
brew install kubectl

# Or download directly
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**On Windows:**

```powershell
# Using Chocolatey
choco install kubernetes-cli

# Or download from https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
```

### Step 4.2: Get AKS Credentials

**Via Azure CLI:**

```bash
# Get cluster credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --overwrite-existing

# Expected output:
# Merged "aks-demo-cluster" as current context in /home/user/.kube/config
```

**For admin access (bypasses RBAC):**

```bash
# Get admin credentials (use with caution)
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --admin \
  --overwrite-existing
```

### Step 4.3: Verify kubectl Connection

```bash
# Check cluster info
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://aks-demo-cluster-xxx.hcp.eastus.azmk8s.io:443
# CoreDNS is running at https://aks-demo-cluster-xxx.hcp.eastus.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Get nodes
kubectl get nodes

# Expected output:
# NAME                                STATUS   ROLES   AGE   VERSION
# aks-agentpool-12345678-vmss000000   Ready    agent   5m    v1.28.5
# aks-agentpool-12345678-vmss000001   Ready    agent   5m    v1.28.5

# Get all namespaces
kubectl get namespaces

# Get all pods
kubectl get pods --all-namespaces
```

### Step 4.4: Configure kubectl Context

```bash
# List all contexts
kubectl config get-contexts

# Switch context (if multiple clusters)
kubectl config use-context aks-demo-cluster

# View current context
kubectl config current-context

# Set default namespace
kubectl config set-context --current --namespace=default
```


---

## 5. Configure Node Pools

### Step 5.1: View Existing Node Pools

**Via Azure Portal:**

1. Go to your AKS cluster
2. Click **"Settings"** > **"Node pools"**
3. View existing node pools

📸 **Screenshot: AKS Node pools**
```
aks-demo-cluster | Node pools
Name        | Status    | Mode    | Kubernetes version | Node count | VM size
agentpool   | Succeeded | System  | 1.28.5            | 2         | Standard_D2s_v3

+ Add node pool
```

**Via Azure CLI:**

```bash
# List node pools
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER_NAME \
  --output table

# Output:
# Name        Mode    VmSize           Count  EnableAutoScaling
# ----------  ------  ---------------  -----  -------------------
# agentpool   System  Standard_D2s_v3  2      False
```

### Step 5.2: Add User Node Pool (Optional)

Separate workloads from system pods.

**Via Azure Portal:**

1. Click **"+ Add node pool"**
2. Fill in:
   - **Node pool name**: `userpool`
   - **Mode**: User
   - **Node size**: `Standard_D2s_v3`
   - **Scale method**: Manual
   - **Node count**: 2
3. Click **"Add"**

📸 **Screenshot: Add node pool**
```
Add node pool
Node pool name: userpool
Mode: • User  ○ System
Kubernetes version: 1.28.5 (default)
Node size: Standard_D2s_v3 - Change size
Scale method: • Manual  ○ Autoscale
Node count: 2
[Add button]
```

**Via Azure CLI:**

```bash
# Add user node pool
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER_NAME \
  --name userpool \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --mode User

echo "User node pool added"

# Verify
kubectl get nodes
```

### Step 5.3: Scale Node Pool

```bash
# Scale node pool
az aks nodepool scale \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER_NAME \
  --name agentpool \
  --node-count 3

# Verify scaling
kubectl get nodes -w
```


---

## 6. Deploy Sample Application

### Step 6.1: Create Kubernetes Namespace

```bash
# Create a namespace for your application
kubectl create namespace demo-app

# Verify
kubectl get namespaces

# Set as default namespace
kubectl config set-context --current --namespace=demo-app
```

### Step 6.2: Deploy Application from ACR

Create a deployment manifest using an image from your ACR:

```bash
# Create deployment file
cat << EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: ${ACR_NAME}.azurecr.io/samples/sample-app:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  namespace: demo-app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: sample-app
EOF

# Apply the deployment
kubectl apply -f deployment.yaml

# Expected output:
# deployment.apps/sample-app created
# service/sample-app-service created
```

### Step 6.3: Monitor Deployment

```bash
# Watch pods being created
kubectl get pods -n demo-app -w

# Expected output:
# NAME                          READY   STATUS              RESTARTS   AGE
# sample-app-5d8f7b4c9d-abcde   0/1     ContainerCreating   0          5s
# sample-app-5d8f7b4c9d-fghij   0/1     ContainerCreating   0          5s
# ...
# sample-app-5d8f7b4c9d-abcde   1/1     Running             0          30s
# sample-app-5d8f7b4c9d-fghij   1/1     Running             0          30s

# Check deployment status
kubectl get deployments -n demo-app

# Check service and get external IP
kubectl get services -n demo-app

# Wait for EXTERNAL-IP (may take 2-3 minutes)
# NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
# sample-app-service    LoadBalancer   10.0.123.45    20.123.45.67     80:30123/TCP   2m
```


### Step 6.4: Test the Application

```bash
# Get the external IP
EXTERNAL_IP=$(kubectl get service sample-app-service -n demo-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Application URL: http://$EXTERNAL_IP"

# Test with curl
curl http://$EXTERNAL_IP

# Or open in browser
# http://$EXTERNAL_IP

# Expected output:
# <!DOCTYPE html>
# <html>
# <head><title>ACR Demo</title></head>
# <body>
# <h1>Hello from Azure Container Registry!</h1>
# <p>This image is hosted in ACR.</p>
# </body>
# </html>
```

### Step 6.5: View Deployment in Azure Portal

**Via Azure Portal:**

1. Go to your AKS cluster
2. Click **"Kubernetes resources"** > **"Workloads"**
3. You should see your deployment

📸 **Screenshot: AKS Workloads**
```
aks-demo-cluster | Workloads
Namespace: demo-app ▼

Deployments
Name         | Namespace | Pods  | Status
sample-app   | demo-app  | 2/2   | Ready
```

4. Click **"Services and ingresses"** to see services

📸 **Screenshot: AKS Services**
```
aks-demo-cluster | Services and ingresses
Namespace: demo-app ▼

Services
Name                  | Type           | Cluster IP    | External IP   | Port(s)
sample-app-service    | LoadBalancer   | 10.0.123.45   | 20.123.45.67  | 80:30123/TCP
```

### Step 6.6: Scale the Application

```bash
# Scale to 4 replicas
kubectl scale deployment sample-app -n demo-app --replicas=4

# Watch scaling
kubectl get pods -n demo-app -w

# Verify
kubectl get deployment sample-app -n demo-app

# Output:
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# sample-app   4/4     4            4           5m
```

### Step 6.7: View Pod Logs

```bash
# Get pod name
POD_NAME=$(kubectl get pods -n demo-app -l app=sample-app -o jsonpath='{.items[0].metadata.name}')

# View logs
kubectl logs $POD_NAME -n demo-app

# Follow logs
kubectl logs -f $POD_NAME -n demo-app

# Logs for all pods with label
kubectl logs -l app=sample-app -n demo-app --all-containers=true
```


---

## 7. Configure Networking

### Step 7.1: View Virtual Network Configuration

**Via Azure Portal:**

1. Go to **Resource groups** > **MC_rg-aks-demo_aks-demo-cluster_eastus** (node resource group)
2. Find the Virtual Network (e.g., `aks-vnet-12345678`)
3. Click on it to view details

📸 **Screenshot: AKS Virtual Network**
```
aks-vnet-12345678 | Overview
Address space: 10.224.0.0/12
Subnets:
- aks-subnet: 10.224.0.0/16
```

**Via Azure CLI:**

```bash
# Get node resource group
NODE_RG=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query nodeResourceGroup -o tsv)

# List VNets in node resource group
az network vnet list --resource-group $NODE_RG --output table

# Get VNet details
VNET_NAME=$(az network vnet list --resource-group $NODE_RG --query "[0].name" -o tsv)
az network vnet show --resource-group $NODE_RG --name $VNET_NAME
```

### Step 7.2: Configure Network Security Groups (NSG)

```bash
# List NSGs
az network nsg list --resource-group $NODE_RG --output table

# Get NSG name
NSG_NAME=$(az network nsg list --resource-group $NODE_RG --query "[0].name" -o tsv)

# View NSG rules
az network nsg rule list --resource-group $NODE_RG --nsg-name $NSG_NAME --output table

# Add custom rule (example: allow traffic on port 8080)
az network nsg rule create \
  --resource-group $NODE_RG \
  --nsg-name $NSG_NAME \
  --name allow-8080 \
  --priority 1001 \
  --source-address-prefixes '*' \
  --destination-port-ranges 8080 \
  --access Allow \
  --protocol Tcp
```

### Step 7.3: Configure Ingress Controller (Optional)

Install NGINX Ingress Controller:

```bash
# Add ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Create namespace for ingress
kubectl create namespace ingress-nginx

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

# Wait for external IP
kubectl get service -n ingress-nginx -w

# Create ingress resource
cat << EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app-ingress
  namespace: demo-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sample-app-service
            port:
              number: 80
EOF

# Apply ingress
kubectl apply -f ingress.yaml
```


---

## 8. Enable Monitoring and Logging

### Step 8.1: Verify Azure Monitor Container Insights

If enabled during cluster creation, Container Insights should be active.

**Via Azure Portal:**

1. Go to your AKS cluster
2. Click **"Monitoring"** > **"Insights"**
3. View cluster performance, nodes, and containers

📸 **Screenshot: AKS Insights**
```
aks-demo-cluster | Insights
Cluster | Nodes | Controllers | Containers | Deployments

Overview
Cluster Status: Healthy
Node Count: 2
Pod Count: 15
Container Count: 18

Performance charts showing CPU, Memory, Network usage
```

**Via Azure CLI:**

```bash
# Check if monitoring is enabled
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query "addonProfiles.omsagent.enabled" -o tsv

# Should return: true
```

### Step 8.2: Enable Monitoring (if not enabled)

```bash
# Enable Container Insights
az aks enable-addons \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --addons monitoring

echo "Monitoring enabled"
```

### Step 8.3: View Logs in Azure Monitor

**Via Azure Portal:**

1. Go to your AKS cluster
2. Click **"Monitoring"** > **"Logs"**
3. Run a sample query:

```kusto
// View container logs
ContainerLog
| where TimeGenerated > ago(1h)
| project TimeGenerated, ContainerID, LogEntry
| order by TimeGenerated desc
| take 100
```

📸 **Screenshot: Azure Monitor Logs**
```
aks-demo-cluster | Logs
[Query editor with sample query]

Results:
TimeGenerated          | ContainerID | LogEntry
2026-06-21 11:30:00   | abc123...   | Application started successfully
...
```

### Step 8.4: Create Alerts

**Via Azure Portal:**

1. In AKS cluster, click **"Monitoring"** > **"Alerts"**
2. Click **"+ Create"** > **"Alert rule"**
3. Configure alert:
   - **Condition**: CPU usage > 80%
   - **Actions**: Email notification
   - **Alert rule name**: High CPU Usage

📸 **Screenshot: Create alert rule**
```
Create an alert rule
Scope: aks-demo-cluster

Condition
Signal: Percentage CPU
Operator: Greater than
Threshold: 80

Actions
Action group: [Create or select]

Alert rule details
Alert rule name: High CPU Usage
Severity: 2 - Warning

[Create alert rule button]
```


### Step 8.5: Enable Diagnostic Settings

```bash
# Get AKS resource ID
AKS_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query id -o tsv)

# Get Log Analytics workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace list --query "[0].id" -o tsv)

# Enable diagnostic settings
az monitor diagnostic-settings create \
  --name aks-diagnostics \
  --resource $AKS_ID \
  --workspace $WORKSPACE_ID \
  --logs '[
    {
      "category": "kube-apiserver",
      "enabled": true
    },
    {
      "category": "kube-controller-manager",
      "enabled": true
    },
    {
      "category": "kube-scheduler",
      "enabled": true
    },
    {
      "category": "kube-audit",
      "enabled": true
    }
  ]' \
  --metrics '[
    {
      "category": "AllMetrics",
      "enabled": true
    }
  ]'

echo "Diagnostic settings configured"
```

---

## 9. Verification

### Step 9.1: Verify Cluster Health

**Via Azure Portal:**

1. Go to your AKS cluster
2. Check **"Overview"** page
3. Verify **"Status"** is **"Succeeded"**

📸 **Screenshot: AKS Overview - Healthy status**
```
aks-demo-cluster | Overview
Status: Succeeded ✓
Kubernetes version: 1.28.5
Location: East US
Node count: 2
FQDN: aks-demo-cluster-xxx.hcp.eastus.azmk8s.io
```

**Via Azure CLI:**

```bash
# Check cluster status
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query "{Name:name, Status:provisioningState, K8sVersion:kubernetesVersion, NodeCount:agentPoolProfiles[0].count}" \
  --output table

# Expected output:
# Name               Status     K8sVersion  NodeCount
# ----------------   ---------  ----------  ----------
# aks-demo-cluster   Succeeded  1.28.5      2
```

### Step 9.2: Verify Node Health

```bash
# Check node status
kubectl get nodes -o wide

# Expected output (all nodes should be Ready):
# NAME                                STATUS   ROLES   AGE   VERSION   INTERNAL-IP   EXTERNAL-IP
# aks-agentpool-12345678-vmss000000   Ready    agent   30m   v1.28.5   10.224.0.4    <none>
# aks-agentpool-12345678-vmss000001   Ready    agent   30m   v1.28.5   10.224.0.5    <none>

# Check node conditions
kubectl describe nodes | grep -A 5 Conditions

# View node resource usage
kubectl top nodes

# Output:
# NAME                                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# aks-agentpool-12345678-vmss000000   120m         6%     1256Mi          18%
# aks-agentpool-12345678-vmss000001   98m          5%     1102Mi          16%
```


### Step 9.3: Verify ACR Integration

```bash
# Check ACR attachment
az aks check-acr \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --acr ${ACR_NAME}.azurecr.io

# Expected output:
# [2026-06-21 11:45:00] Validating image pull from ACR
# [2026-06-21 11:45:02] ACR connectivity test passed

# Verify pod is using ACR image
kubectl describe pod -n demo-app -l app=sample-app | grep -A 3 "Image:"

# Output should show:
#     Image:          acrdemoacr.azurecr.io/samples/sample-app:v1
#     Image ID:       acrdemoacr.azurecr.io/samples/sample-app@sha256:xxxxx
#     State:          Running
```

### Step 9.4: Verify Application Deployment

```bash
# Check deployment status
kubectl get deployments -n demo-app

# Expected output:
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# sample-app   2/2     2            2           15m

# Check all resources
kubectl get all -n demo-app

# Test application endpoint
EXTERNAL_IP=$(kubectl get service sample-app-service -n demo-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -I http://$EXTERNAL_IP

# Expected output:
# HTTP/1.1 200 OK
# Server: nginx/1.25.3
# ...

echo "✓ All verifications passed"
```

### Step 9.5: Run Cluster Diagnostics

```bash
# Run AKS diagnostics
az aks kollect \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME

# This collects diagnostic information for troubleshooting

# Check cluster addons
az aks addon list --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME

# Get cluster upgrade versions
az aks get-upgrades \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --output table
```

---

## 10. Troubleshooting

### Issue 1: kubectl cannot connect to cluster

**Symptoms:**
```bash
kubectl get nodes
# Error: Unable to connect to the server: dial tcp: lookup aks-demo-cluster
```

**Solutions:**

```bash
# Solution 1: Re-fetch credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --overwrite-existing

# Solution 2: Check if cluster is running
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query provisioningState

# Solution 3: Verify network connectivity
AKS_FQDN=$(az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query fqdn -o tsv)
nslookup $AKS_FQDN

# Solution 4: Check kubeconfig
kubectl config view
kubectl config get-contexts
kubectl config use-context aks-demo-cluster
```


### Issue 2: Pods stuck in "ImagePullBackOff"

**Symptoms:**
```bash
kubectl get pods
# NAME                          READY   STATUS             RESTARTS   AGE
# sample-app-5d8f7b4c9d-abcde   0/1     ImagePullBackOff   0          2m
```

**Causes:**
- ACR not attached to AKS
- Incorrect image name
- Missing RBAC permissions

**Solutions:**

```bash
# Check pod events
kubectl describe pod <pod-name> -n demo-app

# Look for error messages like:
# Failed to pull image "acrdemoacr.azurecr.io/samples/sample-app:v1": 
# rpc error: code = Unknown desc = failed to pull and unpack image...

# Solution 1: Verify ACR attachment
az aks check-acr \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --acr ${ACR_NAME}.azurecr.io

# Solution 2: Attach ACR if not attached
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --attach-acr $ACR_NAME

# Solution 3: Verify image exists in ACR
az acr repository show-tags --name $ACR_NAME --repository samples/sample-app

# Solution 4: Check RBAC permissions
AKS_IDENTITY=$(az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query "identityProfile.kubeletidentity.objectId" -o tsv)
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

az role assignment create \
  --assignee $AKS_IDENTITY \
  --role AcrPull \
  --scope $ACR_ID

# Wait 2-3 minutes for propagation
# Then delete and recreate pod
kubectl delete pod <pod-name> -n demo-app
```

### Issue 3: Service External IP stuck in "Pending"

**Symptoms:**
```bash
kubectl get service -n demo-app
# NAME                  TYPE           EXTERNAL-IP   PORT(S)
# sample-app-service    LoadBalancer   <pending>     80:30123/TCP
```

**Solutions:**

```bash
# Check service events
kubectl describe service sample-app-service -n demo-app

# Check Azure Load Balancer status
az network lb list --resource-group $NODE_RG --output table

# Verify service configuration
kubectl get service sample-app-service -n demo-app -o yaml

# Delete and recreate service
kubectl delete service sample-app-service -n demo-app
kubectl apply -f deployment.yaml

# Check load balancer provisioning in Azure
# Go to Node Resource Group in Azure Portal
# Look for Load Balancer resource
```

### Issue 4: Insufficient CPU or Memory

**Symptoms:**
```bash
kubectl get pods
# NAME                          READY   STATUS    RESTARTS   AGE
# sample-app-5d8f7b4c9d-abcde   0/1     Pending   0          2m

kubectl describe pod <pod-name>
# Events:
# Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims
# 0/2 nodes are available: 2 Insufficient cpu.
```

**Solutions:**

```bash
# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Scale up node pool
az aks nodepool scale \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER_NAME \
  --name agentpool \
  --node-count 3

# Or add larger nodes
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $AKS_CLUSTER_NAME \
  --name largerpool \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3

# Adjust pod resource requests
# Edit deployment.yaml and reduce resource requests
kubectl apply -f deployment.yaml
```


### Issue 5: Cannot delete AKS cluster

**Symptoms:**
```bash
az aks delete --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
# Error: Cluster cannot be deleted because it has dependencies
```

**Solutions:**

```bash
# Solution 1: Force delete
az aks delete \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --yes \
  --no-wait

# Solution 2: Delete services with LoadBalancer type first
kubectl get services --all-namespaces | grep LoadBalancer
kubectl delete service <service-name> -n <namespace>

# Then delete cluster
az aks delete --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --yes

# Solution 3: Delete node resource group if cluster deletion fails
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query nodeResourceGroup -o tsv)
az group delete --name $NODE_RG --yes --no-wait
```

### Issue 6: Azure AD authentication not working

**Symptoms:**
```bash
kubectl get nodes
# To sign in, use a web browser to open the page https://microsoft.com/devicelogin...
```

**Solutions:**

```bash
# Solution 1: Use admin credentials temporarily
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --admin \
  --overwrite-existing

# Solution 2: Verify you're added to admin group
# Check in Azure Portal: Azure AD > Groups > aks-administrators

# Solution 3: Re-authenticate
az login
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --overwrite-existing

# Follow the device login flow
```

### Issue 7: Monitoring not showing data

**Symptoms:**
- Container Insights shows "No data available"

**Solutions:**

```bash
# Check if monitoring addon is enabled
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query "addonProfiles.omsagent.enabled"

# Enable if disabled
az aks enable-addons \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --addons monitoring

# Check omsagent pods
kubectl get pods -n kube-system | grep omsagent

# If pods are not running, restart them
kubectl delete pod -n kube-system -l component=oms-agent

# Wait 10-15 minutes for data to appear in Azure Monitor
```

### Issue 8: Connection timeout to pods

**Symptoms:**
```bash
kubectl logs <pod-name>
# Error: context deadline exceeded
```

**Solutions:**

```bash
# Check if API server is accessible
kubectl cluster-info

# Verify network configuration
kubectl get pods -o wide
kubectl get nodes -o wide

# Check if pods are in Running state
kubectl get pods --all-namespaces

# Increase timeout
kubectl logs <pod-name> --request-timeout=60s

# Try exec into pod
kubectl exec -it <pod-name> -- /bin/sh
```

---

## Summary Checklist

After completing this guide, you should have:

- [ ] Created AKS cluster with appropriate node size and count
- [ ] Configured Azure AD integration (optional)
- [ ] Attached ACR to AKS cluster
- [ ] Configured kubectl access and verified connection
- [ ] Created and configured node pools
- [ ] Deployed sample application from ACR
- [ ] Verified application is accessible via external IP
- [ ] Configured networking (LoadBalancer/Ingress)
- [ ] Enabled Azure Monitor Container Insights
- [ ] Configured diagnostic settings and alerts
- [ ] Verified cluster health and node status
- [ ] Tested ACR integration with image pulls

### Important Information to Save

```bash
# AKS Cluster Information
AKS_CLUSTER_NAME=aks-demo-cluster
AKS_RESOURCE_GROUP=rg-aks-demo
AKS_LOCATION=eastus
NODE_RESOURCE_GROUP=MC_rg-aks-demo_aks-demo-cluster_eastus
AKS_FQDN=aks-demo-cluster-xxx.hcp.eastus.azmk8s.io

# Application Endpoints
SAMPLE_APP_URL=http://<EXTERNAL-IP>

# ACR Integration
ACR_NAME=acrdemoacr
ACR_ATTACHED=true
```


### AKS Quick Reference Commands

```bash
# Cluster Management
az aks list --output table
az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
az aks stop --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
az aks start --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
az aks delete --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --yes

# Node Management
az aks nodepool list --resource-group $RESOURCE_GROUP --cluster-name $AKS_CLUSTER_NAME
az aks nodepool scale --resource-group $RESOURCE_GROUP --cluster-name $AKS_CLUSTER_NAME --name agentpool --node-count 3
az aks nodepool upgrade --resource-group $RESOURCE_GROUP --cluster-name $AKS_CLUSTER_NAME --name agentpool --kubernetes-version 1.28.5

# kubectl Commands
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
kubectl delete pod <pod-name>
kubectl scale deployment <deployment-name> --replicas=3

# Monitoring and Troubleshooting
kubectl top nodes
kubectl top pods
kubectl get events --sort-by='.lastTimestamp'
kubectl describe nodes
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --admin

# Cluster Upgrade
az aks get-upgrades --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
az aks upgrade --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --kubernetes-version 1.29.0

# Check cluster health
az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query provisioningState
```

---

## Best Practices

### 1. Security
- ✅ Use managed identities instead of service principals
- ✅ Enable Azure AD integration for RBAC
- ✅ Use private clusters for production
- ✅ Enable network policies
- ✅ Regularly update Kubernetes version
- ✅ Use Azure Policy for governance

### 2. Reliability
- ✅ Deploy across availability zones
- ✅ Use multiple replicas for applications
- ✅ Configure pod disruption budgets
- ✅ Implement health checks (liveness/readiness probes)
- ✅ Set resource requests and limits

### 3. Performance
- ✅ Right-size your node VMs
- ✅ Use autoscaling for node pools
- ✅ Implement horizontal pod autoscaling
- ✅ Monitor resource utilization
- ✅ Use appropriate storage classes

### 4. Cost Optimization
- ✅ Use spot instances for non-critical workloads
- ✅ Stop clusters in dev/test environments when not in use
- ✅ Right-size resources based on actual usage
- ✅ Use cluster autoscaler
- ✅ Review and delete unused resources

### 5. Operations
- ✅ Enable Container Insights
- ✅ Configure alerts and notifications
- ✅ Implement GitOps for deployments
- ✅ Use namespaces for isolation
- ✅ Regular backup of cluster configurations

---

## Next Steps

Now that AKS is configured and running, proceed to:

👉 **[04-jump-server-setup.md](./04-jump-server-setup.md)** - Set up Jump Server VM for secure AKS access

---

## Additional Resources

- [AKS Best Practices](https://docs.microsoft.com/azure/aks/best-practices)
- [AKS Production Baseline](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline-aks)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Azure Monitor for Containers](https://docs.microsoft.com/azure/azure-monitor/containers/container-insights-overview)
- [AKS Network Concepts](https://docs.microsoft.com/azure/aks/concepts-network)
- [AKS Security Concepts](https://docs.microsoft.com/azure/aks/concepts-security)
- [Troubleshoot AKS](https://docs.microsoft.com/azure/aks/troubleshooting)
