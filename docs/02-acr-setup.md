# Azure Container Registry (ACR) Setup Guide

## 📋 Overview

Azure Container Registry (ACR) is a private Docker registry service for storing and managing container images. This guide covers creating an ACR instance, configuring Azure AD authentication, and integrating with AKS.

## Prerequisites

- Completed [Azure AD Setup](./01-azure-ad-setup.md)
- Azure subscription with Contributor role
- Azure CLI installed
- Docker installed locally (for pushing images)
- Service principal credentials from Azure AD setup

## Table of Contents

1. [Create Azure Container Registry](#1-create-azure-container-registry)
2. [Configure Azure AD Authentication](#2-configure-azure-ad-authentication)
3. [Enable Admin User (Optional)](#3-enable-admin-user-optional)
4. [Push Sample Container Images](#4-push-sample-container-images)
5. [Set Up RBAC for ACR Access](#5-set-up-rbac-for-acr-access)
6. [Configure ACR for AKS Integration](#6-configure-acr-for-aks-integration)
7. [Enable Security Features](#7-enable-security-features)
8. [Verification](#8-verification)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Create Azure Container Registry

### Step 1.1: Create Resource Group (if not exists)

**Via Azure Portal:**

1. Navigate to [Azure Portal](https://portal.azure.com)
2. In the search bar, type **"Resource groups"**
3. Click **"Resource groups"** from the results
4. Click **"+ Create"**

📸 **Screenshot: Resource groups page**
```
Resource groups > + Create
```

5. Fill in the details:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Region**: `East US` (or your preferred region)
6. Click **"Review + create"**
7. Click **"Create"**

📸 **Screenshot: Create a resource group**
```
Basics tab
Subscription: [Your subscription]
Resource group: rg-aks-demo
Region: East US
[Review + create button]
```

**Via Azure CLI:**

```bash
# Set variables
RESOURCE_GROUP="rg-aks-demo"
LOCATION="eastus"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Output:
# {
#   "id": "/subscriptions/.../resourceGroups/rg-aks-demo",
#   "location": "eastus",
#   "name": "rg-aks-demo",
#   "properties": {
#     "provisioningState": "Succeeded"
#   }
# }
```


### Step 1.2: Create Azure Container Registry

**Via Azure Portal:**

1. In the search bar, type **"Container registries"**
2. Click **"Container registries"** from the results
3. Click **"+ Create"**

📸 **Screenshot: Container registries page**
```
Container registries > + Create
```

4. Fill in the **Basics** tab:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Registry name**: `acrdemoacr` (must be globally unique, lowercase alphanumeric)
   - **Location**: `East US` (same as resource group)
   - **SKU**: `Standard` (or Premium for production)

📸 **Screenshot: Create container registry - Basics tab**
```
Basics tab
Subscription: [Your subscription]
Resource group: rg-aks-demo
Registry name: acrdemoacr
Location: East US
SKU: Standard ▼
[Next: Networking >]
```

> **Note**: Registry name must be:
> - Globally unique across Azure
> - 5-50 characters
> - Alphanumeric only (no hyphens or special characters)
> - Lowercase letters and numbers

5. Click **"Next: Networking >"**

6. In the **Networking** tab:
   - **Network connectivity**: Select "Public access from all networks" (for now)
   - For production, consider "Private endpoint" or "Selected networks"

📸 **Screenshot: Create container registry - Networking tab**
```
Networking tab
Network connectivity:
• Public access from all networks
○ Private endpoint
○ Disable public access
[Next: Encryption >]
```

7. Click **"Next: Encryption >"**


8. In the **Encryption** tab:
   - **Encryption**: Keep default "Microsoft-managed keys"
   - For production, you can use "Customer-managed keys" for additional security

📸 **Screenshot: Create container registry - Encryption tab**
```
Encryption tab
Encryption:
• Microsoft-managed keys (default)
○ Customer-managed keys
[Review + create button]
```

9. Click **"Review + create"**
10. Review your settings
11. Click **"Create"**

📸 **Screenshot: Review + create**
```
Review + create tab
Registry name: acrdemoacr
Resource group: rg-aks-demo
Location: East US
SKU: Standard
Validation passed ✓
[Create button]
```

12. Wait for deployment (usually 1-2 minutes)
13. Click **"Go to resource"**

📸 **Screenshot: Deployment complete**
```
Your deployment is complete
Deployment name: microsoft.containerregistry-xxx
Start time: 6/21/2026, 10:30:00 AM
Subscription: Your subscription
Resource group: rg-aks-demo
[Go to resource button]
```

**Via Azure CLI:**

```bash
# Set variables
ACR_NAME="acrdemoacr"  # Must be globally unique
RESOURCE_GROUP="rg-aks-demo"
SKU="Standard"  # Options: Basic, Standard, Premium

# Create ACR
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku $SKU \
  --location eastus

# Output will show the ACR details
echo "ACR created successfully"

# Get ACR login server
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
echo "Login Server: $ACR_LOGIN_SERVER"
# Output: acrdemoacr.azurecr.io
```


### Step 1.3: Note Down ACR Information

From the ACR Overview page, save these values:

📸 **Screenshot: ACR Overview page**
```
acrdemoacr | Overview
Login server: acrdemoacr.azurecr.io
Resource group: rg-aks-demo
Location: East US
Subscription: Your subscription
SKU: Standard
```

```bash
ACR_NAME=acrdemoacr
ACR_LOGIN_SERVER=acrdemoacr.azurecr.io
ACR_RESOURCE_ID=/subscriptions/.../resourceGroups/rg-aks-demo/providers/Microsoft.ContainerRegistry/registries/acrdemoacr
```

---

## 2. Configure Azure AD Authentication

ACR supports multiple authentication methods. Azure AD authentication is recommended for production.

### Step 2.1: Understand ACR Authentication Methods

1. **Azure AD Authentication** (Recommended)
   - Service principals
   - Managed identities
   - Individual Azure AD identities

2. **Admin User** (For development/testing only)
   - Username/password authentication
   - Not recommended for production

3. **Repository-scoped Access Token**
   - Fine-grained access control

### Step 2.2: Configure Service Principal Access

We'll use the service principal created in the Azure AD setup.

**Via Azure Portal:**

1. In your ACR resource, click **"Access control (IAM)"**
2. Click **"+ Add"** > **"Add role assignment"**

📸 **Screenshot: ACR Access control (IAM)**
```
acrdemoacr | Access control (IAM)
+ Add > Add role assignment
```

3. In the **Role** tab:
   - Search for **"AcrPull"**
   - Select **"AcrPull"**
   - Click **"Next"**

📸 **Screenshot: Add role assignment - Role tab**
```
Role tab
Search: AcrPull
☑ AcrPull
Description: Allows pulling artifacts from a container registry
[Next button]
```


4. In the **Members** tab:
   - **Assign access to**: "User, group, or service principal"
   - Click **"+ Select members"**
   - Search for your service principal: `sp-aks-cluster`
   - Select it
   - Click **"Select"**
   - Click **"Next"**

📸 **Screenshot: Add role assignment - Members tab**
```
Members tab
Assign access to: • User, group, or service principal

+ Select members
Search: sp-aks-cluster
Selected members:
- sp-aks-cluster
[Next button]
```

5. In the **Review + assign** tab, click **"Review + assign"**

📸 **Screenshot: Review + assign**
```
Review + assign tab
Role: AcrPull
Members: sp-aks-cluster
Scope: /subscriptions/.../registries/acrdemoacr
[Review + assign button]
```

**Via Azure CLI:**

```bash
# Set variables (from Azure AD setup)
AKS_SP_CLIENT_ID="your-service-principal-client-id"
ACR_NAME="acrdemoacr"

# Get ACR resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

# Assign AcrPull role
az role assignment create \
  --assignee $AKS_SP_CLIENT_ID \
  --role AcrPull \
  --scope $ACR_ID

echo "AcrPull role assigned to service principal"

# If you need push access too, assign AcrPush role
az role assignment create \
  --assignee $AKS_SP_CLIENT_ID \
  --role AcrPush \
  --scope $ACR_ID

echo "AcrPush role assigned to service principal"
```

### Step 2.3: Verify Service Principal Access

```bash
# Login to ACR using service principal
az acr login \
  --name $ACR_NAME \
  --service-principal $AKS_SP_CLIENT_ID \
  --password $AKS_SP_CLIENT_SECRET

# Expected output:
# Login Succeeded
```


---

## 3. Enable Admin User (Optional)

> ⚠️ **Warning**: Admin user authentication is convenient for development but NOT recommended for production. Use Azure AD authentication instead.

### Step 3.1: Enable Admin User

**Via Azure Portal:**

1. In your ACR resource, click **"Access keys"** in the left menu
2. Toggle **"Admin user"** to **"Enabled"**

📸 **Screenshot: ACR Access keys**
```
acrdemoacr | Access keys
Admin user: [Disabled] → Toggle to [Enabled]

Login server: acrdemoacr.azurecr.io
Username: acrdemoacr
password: [show] xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
password2: [show] xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

3. Copy the username and password

**Via Azure CLI:**

```bash
# Enable admin user
az acr update --name $ACR_NAME --admin-enabled true

# Get admin credentials
az acr credential show --name $ACR_NAME

# Output:
# {
#   "passwords": [
#     {
#       "name": "password",
#       "value": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
#     },
#     {
#       "name": "password2",
#       "value": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
#     }
#   ],
#   "username": "acrdemoacr"
# }
```

### Step 3.2: Login with Admin Credentials (Testing)

```bash
# Get admin password
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

# Login to ACR with admin credentials
echo $ACR_PASSWORD | docker login $ACR_LOGIN_SERVER --username $ACR_NAME --password-stdin

# Expected output:
# Login Succeeded
```

---

## 4. Push Sample Container Images

### Step 4.1: Prepare a Sample Docker Image

**Option 1: Use an existing public image**

```bash
# Pull a public image from Docker Hub
docker pull nginx:latest

# Tag it for your ACR
docker tag nginx:latest $ACR_LOGIN_SERVER/samples/nginx:v1

# Example:
# docker tag nginx:latest acrdemoacr.azurecr.io/samples/nginx:v1
```


**Option 2: Build a custom image**

```bash
# Create a directory for your app
mkdir sample-app && cd sample-app

# Create a simple Dockerfile
cat << 'EOF' > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# Create a sample HTML file
cat << 'EOF' > index.html
<!DOCTYPE html>
<html>
<head><title>ACR Demo</title></head>
<body>
<h1>Hello from Azure Container Registry!</h1>
<p>This image is hosted in ACR.</p>
</body>
</html>
EOF

# Build the image
docker build -t sample-app:v1 .
```

### Step 4.2: Login to ACR

**Method 1: Using Azure CLI (Recommended)**

```bash
# Login to ACR
az acr login --name $ACR_NAME

# This uses your Azure AD credentials
# Expected output: Login Succeeded
```

**Method 2: Using Docker directly with admin credentials**

```bash
# Get admin credentials
ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

# Login
echo $ACR_PASSWORD | docker login $ACR_LOGIN_SERVER --username $ACR_USERNAME --password-stdin
```

### Step 4.3: Tag and Push Images

```bash
# Tag the image for ACR
docker tag sample-app:v1 $ACR_LOGIN_SERVER/samples/sample-app:v1

# Push the image
docker push $ACR_LOGIN_SERVER/samples/sample-app:v1

# Expected output:
# The push refers to repository [acrdemoacr.azurecr.io/samples/sample-app]
# v1: digest: sha256:xxxxx size: 1234
```

**Push multiple images:**

```bash
# Pull and push nginx
docker pull nginx:latest
docker tag nginx:latest $ACR_LOGIN_SERVER/samples/nginx:v1
docker push $ACR_LOGIN_SERVER/samples/nginx:v1

# Pull and push redis
docker pull redis:alpine
docker tag redis:alpine $ACR_LOGIN_SERVER/samples/redis:alpine
docker push $ACR_LOGIN_SERVER/samples/redis:alpine

echo "All images pushed successfully"
```


### Step 4.4: Verify Images in ACR

**Via Azure Portal:**

1. In your ACR resource, click **"Repositories"** in the left menu
2. You should see your pushed repositories

📸 **Screenshot: ACR Repositories**
```
acrdemoacr | Repositories
Repository               | Last updated         | Images
samples/sample-app       | 6/21/2026, 10:45 AM | 1
samples/nginx           | 6/21/2026, 10:46 AM | 1
samples/redis           | 6/21/2026, 10:47 AM | 1
```

3. Click on a repository to see its tags

📸 **Screenshot: Repository details**
```
samples/sample-app
Tag    | Last updated         | Digest
v1     | 6/21/2026, 10:45 AM | sha256:xxxxx...
```

**Via Azure CLI:**

```bash
# List repositories
az acr repository list --name $ACR_NAME --output table

# Output:
# Result
# ------------------
# samples/sample-app
# samples/nginx
# samples/redis

# List tags for a specific repository
az acr repository show-tags --name $ACR_NAME --repository samples/sample-app --output table

# Output:
# Result
# ------
# v1

# Get image details
az acr repository show --name $ACR_NAME --repository samples/sample-app --output table
```

---

## 5. Set Up RBAC for ACR Access

### Step 5.1: Assign Roles to Managed Identity

If you created a managed identity in the Azure AD setup, assign it ACR roles.

**Via Azure CLI:**

```bash
# Get managed identity principal ID (from Azure AD setup)
MANAGED_IDENTITY_PRINCIPAL_ID="your-managed-identity-principal-id"

# Get ACR resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

# Assign AcrPull role to managed identity
az role assignment create \
  --assignee $MANAGED_IDENTITY_PRINCIPAL_ID \
  --role AcrPull \
  --scope $ACR_ID

echo "Managed identity can now pull images from ACR"
```


### Step 5.2: Assign Roles to Users/Groups

**Via Azure Portal:**

1. In ACR, go to **"Access control (IAM)"**
2. Click **"+ Add"** > **"Add role assignment"**
3. Select role based on needs:
   - **AcrPull**: Read-only access (pull images)
   - **AcrPush**: Push and pull images
   - **AcrDelete**: Delete images (use with caution)
   - **Contributor**: Full management access
4. Select users or groups
5. Click **"Review + assign"**

📸 **Screenshot: Role assignments summary**
```
acrdemoacr | Access control (IAM) | Role assignments
Name                        | Role          | Type
sp-aks-cluster             | AcrPull       | Service Principal
id-aks-managed-identity    | AcrPull       | Managed Identity
user@company.com           | AcrPush       | User
```

### Step 5.3: Verify Role Assignments

```bash
# List all role assignments for ACR
az role assignment list --scope $ACR_ID --output table

# Output shows:
# Principal                    Role       Scope
# --------------------------   ---------  --------
# sp-aks-cluster              AcrPull    /subscriptions/.../registries/acrdemoacr
# id-aks-managed-identity     AcrPull    /subscriptions/.../registries/acrdemoacr
```

---

## 6. Configure ACR for AKS Integration

### Step 6.1: Understand ACR-AKS Integration

AKS can pull images from ACR using:
1. **Managed Identity** (Recommended for new clusters)
2. **Service Principal** (Legacy method)
3. **Admin credentials** (Not recommended)

### Step 6.2: Prepare for AKS Attachment

When creating AKS in the next guide, you'll attach ACR using one of these methods:

**Method 1: Attach during AKS creation (Recommended)**
```bash
# This will be used in the AKS setup
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name myAKSCluster \
  --attach-acr $ACR_NAME
```

**Method 2: Attach to existing AKS cluster**
```bash
# If AKS already exists
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name myAKSCluster \
  --attach-acr $ACR_NAME
```

**What happens behind the scenes:**
- Azure assigns AcrPull role to AKS kubelet managed identity
- AKS can now pull images from ACR without additional configuration
- No image pull secrets needed in Kubernetes

### Step 6.3: Create Image Pull Secret (Alternative Method)

If not using managed identity attachment, create a Kubernetes secret:

```bash
# Get ACR credentials
ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

# Create Kubernetes secret (run this after AKS is created)
kubectl create secret docker-registry acr-secret \
  --docker-server=$ACR_LOGIN_SERVER \
  --docker-username=$ACR_USERNAME \
  --docker-password=$ACR_PASSWORD \
  --docker-email=your-email@example.com

# Use in pod spec:
# imagePullSecrets:
# - name: acr-secret
```


---

## 7. Enable Security Features

### Step 7.1: Enable Content Trust (Optional)

Content trust ensures image integrity.

**Via Azure CLI:**

```bash
# Enable content trust
az acr config content-trust update --name $ACR_NAME --status enabled

# Verify
az acr config content-trust show --name $ACR_NAME
```

**Via Azure Portal:**

1. In ACR, go to **"Settings"** > **"Content trust"**
2. Toggle **"Content trust"** to **"Enabled"**

📸 **Screenshot: Content trust settings**
```
acrdemoacr | Content trust
Content trust: [Disabled] → Toggle to [Enabled]
Description: Docker Content Trust provides the ability to verify image integrity
```

### Step 7.2: Configure Network Access

**Limit access to specific networks:**

**Via Azure Portal:**

1. In ACR, click **"Networking"** in the left menu
2. Under **"Public access"**, select **"Selected networks"**
3. Click **"+ Add your client IP address"** to add your current IP
4. Add other allowed IP ranges or Virtual Networks
5. Click **"Save"**

📸 **Screenshot: ACR Networking settings**
```
acrdemoacr | Networking
Public access:
○ All networks
• Selected networks
○ Disabled

Firewall:
+ Add your client IP address (123.45.67.89)
+ Add IP range

Virtual networks:
+ Add existing virtual network
```

**Via Azure CLI:**

```bash
# Allow specific IP
az acr network-rule add \
  --name $ACR_NAME \
  --ip-address 123.45.67.89/32

# List network rules
az acr network-rule list --name $ACR_NAME
```

### Step 7.3: Enable Vulnerability Scanning (Premium SKU)

Available with Premium SKU only.

**Via Azure Portal:**

1. Upgrade to Premium if needed: ACR > **"Overview"** > **"Upgrade"**
2. Go to **"Security"** > **"Microsoft Defender for Cloud"**
3. Enable **"Microsoft Defender for container registries"**

📸 **Screenshot: Security scanning**
```
acrdemoacr | Microsoft Defender for Cloud
Microsoft Defender for container registries: [Enable]
- Vulnerability assessment for images
- Continuous scanning
- Integration with Microsoft Defender
```


### Step 7.4: Enable Geo-replication (Premium SKU)

For high availability and reduced latency across regions.

**Via Azure Portal:**

1. In ACR, click **"Replications"** in the left menu
2. Click **"+ Add"**
3. Select a location (e.g., West US)
4. Click **"Create"**

📸 **Screenshot: Geo-replication**
```
acrdemoacr | Replications
+ Add
Location              | Status
East US (primary)     | Ready
West US              | Provisioning...
```

**Via Azure CLI:**

```bash
# Add replication
az acr replication create \
  --registry $ACR_NAME \
  --location westus

# List replications
az acr replication list --registry $ACR_NAME --output table
```

### Step 7.5: Configure Retention Policy

Automatically delete untagged manifests.

**Via Azure CLI:**

```bash
# Enable retention policy (Premium SKU)
az acr config retention update \
  --registry $ACR_NAME \
  --status enabled \
  --days 7 \
  --type UntaggedManifests

# Verify
az acr config retention show --registry $ACR_NAME
```

---

## 8. Verification

### Step 8.1: Verify ACR is Accessible

**Via Azure Portal:**

1. Navigate to your ACR
2. Check that **"Status"** shows as **"Ready"**
3. Verify **"Login server"** is displayed

📸 **Screenshot: ACR Overview - Ready status**
```
acrdemoacr | Overview
Status: Ready ✓
Login server: acrdemoacr.azurecr.io
Location: East US
Subscription: Your subscription
SKU: Standard
```

**Via Azure CLI:**

```bash
# Check ACR status
az acr show --name $ACR_NAME --query "{Name:name, LoginServer:loginServer, ProvisioningState:provisioningState}" --output table

# Expected output:
# Name         LoginServer              ProvisioningState
# -----------  -----------------------  ------------------
# acrdemoacr   acrdemoacr.azurecr.io   Succeeded
```

### Step 8.2: Verify Images are Stored

```bash
# List all repositories
az acr repository list --name $ACR_NAME --output table

# Should show:
# Result
# ------------------
# samples/sample-app
# samples/nginx
# samples/redis

# Count total images
IMAGE_COUNT=$(az acr repository list --name $ACR_NAME --query "length(@)")
echo "Total repositories: $IMAGE_COUNT"
```


### Step 8.3: Verify RBAC Permissions

```bash
# Check role assignments
az role assignment list --scope $ACR_ID --query "[].{Principal:principalName, Role:roleDefinitionName}" --output table

# Test pull access with service principal
az acr login --name $ACR_NAME --service-principal $AKS_SP_CLIENT_ID --password $AKS_SP_CLIENT_SECRET

# Pull a test image
docker pull $ACR_LOGIN_SERVER/samples/nginx:v1

# If successful, ACR is properly configured
echo "✓ ACR verification complete"
```

### Step 8.4: Test Image Pull and Push

```bash
# Pull an image
docker pull $ACR_LOGIN_SERVER/samples/sample-app:v1

# Tag it with a new version
docker tag $ACR_LOGIN_SERVER/samples/sample-app:v1 $ACR_LOGIN_SERVER/samples/sample-app:v2

# Push new version
docker push $ACR_LOGIN_SERVER/samples/sample-app:v2

# Verify new tag exists
az acr repository show-tags --name $ACR_NAME --repository samples/sample-app --output table

# Expected output:
# Result
# ------
# v1
# v2
```

---

## 9. Troubleshooting

### Issue 1: "unauthorized: authentication required"

**Symptoms:**
```bash
docker push acrdemoacr.azurecr.io/samples/app:v1
unauthorized: authentication required
```

**Solutions:**

```bash
# Solution 1: Login to ACR
az acr login --name $ACR_NAME

# Solution 2: Check if admin user is enabled (if using admin credentials)
az acr update --name $ACR_NAME --admin-enabled true

# Solution 3: Verify service principal has correct role
az role assignment list --assignee $AKS_SP_CLIENT_ID --scope $ACR_ID

# Solution 4: Check if credentials are expired
az acr credential renew --name $ACR_NAME --password-name password
```

### Issue 2: "denied: requested access to the resource is denied"

**Cause:** Insufficient permissions

**Solution:**

```bash
# Check current user's roles
az role assignment list --assignee $(az account show --query user.name -o tsv) --scope $ACR_ID

# Assign AcrPush role if needed
az role assignment create \
  --assignee $(az account show --query user.name -o tsv) \
  --role AcrPush \
  --scope $ACR_ID

# Wait 2-3 minutes for propagation
sleep 180
```


### Issue 3: "Error response from daemon: Get https://acrdemoacr.azurecr.io/v2/: dial tcp: lookup acrdemoacr.azurecr.io: no such host"

**Cause:** DNS resolution issue or ACR name is incorrect

**Solution:**

```bash
# Verify ACR exists and get correct login server
az acr show --name $ACR_NAME --query loginServer -o tsv

# Test DNS resolution
nslookup $ACR_LOGIN_SERVER

# If DNS fails, check network connectivity
ping $ACR_LOGIN_SERVER

# Check if firewall/network rules are blocking access
az acr show-usage --name $ACR_NAME
```

### Issue 4: Image push is slow or timing out

**Cause:** Network configuration or large image size

**Solution:**

```bash
# Check ACR network settings
az acr show --name $ACR_NAME --query networkRuleSet

# If using private endpoint or firewall, verify your IP is allowed
az acr network-rule list --name $ACR_NAME

# Add your IP if needed
MY_IP=$(curl -s ifconfig.me)
az acr network-rule add --name $ACR_NAME --ip-address $MY_IP

# Optimize image size using multi-stage builds
# Example Dockerfile:
# FROM node:16 AS builder
# WORKDIR /app
# COPY package*.json ./
# RUN npm install
# COPY . .
# RUN npm run build
# 
# FROM node:16-alpine
# WORKDIR /app
# COPY --from=builder /app/dist ./dist
# CMD ["node", "dist/index.js"]
```

### Issue 5: "The registry name is invalid"

**Cause:** ACR name doesn't meet requirements

**Requirements:**
- 5-50 characters
- Alphanumeric only (no hyphens, underscores, or special characters)
- Lowercase only
- Globally unique

**Solution:**

```bash
# Check if name is available
az acr check-name --name mynewacr

# Output will indicate if name is available:
# {
#   "message": null,
#   "nameAvailable": true,
#   "reason": null
# }

# If not available, try variations:
az acr check-name --name mynewacr2026
az acr check-name --name mycompanyacr
```

### Issue 6: Cannot see pushed images in Azure Portal

**Cause:** Images pushed but not appearing in portal

**Solution:**

```bash
# Refresh the portal page (Ctrl+F5)

# Verify images via CLI
az acr repository list --name $ACR_NAME

# Check specific repository
az acr repository show --name $ACR_NAME --repository samples/sample-app

# If images are there but portal is slow, it's a UI caching issue
# Images are safe and accessible to AKS
```

### Issue 7: "Operation returned an invalid status code 'Forbidden'"

**Cause:** Role assignment propagation delay or insufficient permissions

**Solution:**

```bash
# Wait for role assignment propagation (can take 5-10 minutes)
echo "Waiting for role assignment propagation..."
sleep 300

# Verify role assignments
az role assignment list --assignee $AKS_SP_CLIENT_ID --all --query "[].{Role:roleDefinitionName, Scope:scope}" -o table

# If still failing, reassign the role
az role assignment delete --assignee $AKS_SP_CLIENT_ID --role AcrPull --scope $ACR_ID
az role assignment create --assignee $AKS_SP_CLIENT_ID --role AcrPull --scope $ACR_ID

# Try again after 2-3 minutes
```


---

## Summary Checklist

After completing this guide, you should have:

- [ ] Created Azure Container Registry (ACR)
- [ ] Configured Azure AD authentication with service principal
- [ ] Assigned AcrPull/AcrPush roles to appropriate identities
- [ ] Pushed sample container images to ACR
- [ ] Verified images are accessible
- [ ] Configured security features (content trust, network access)
- [ ] Prepared ACR for AKS integration
- [ ] Tested image pull and push operations
- [ ] Documented ACR login server and credentials

### Important Values to Save

Keep these values for AKS integration:

```bash
# ACR Information
ACR_NAME=acrdemoacr
ACR_LOGIN_SERVER=acrdemoacr.azurecr.io
ACR_RESOURCE_ID=/subscriptions/.../resourceGroups/rg-aks-demo/providers/Microsoft.ContainerRegistry/registries/acrdemoacr

# Sample Images Available
# - acrdemoacr.azurecr.io/samples/sample-app:v1
# - acrdemoacr.azurecr.io/samples/nginx:v1
# - acrdemoacr.azurecr.io/samples/redis:alpine

# Service Principal (from Azure AD setup)
AKS_SP_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AKS_SP_CLIENT_SECRET=xxxXXXxxxXXXxxx...

# Managed Identity (if created)
MANAGED_IDENTITY_PRINCIPAL_ID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

### ACR Quick Reference Commands

```bash
# Login to ACR
az acr login --name $ACR_NAME

# List repositories
az acr repository list --name $ACR_NAME

# List tags for a repository
az acr repository show-tags --name $ACR_NAME --repository samples/sample-app

# Delete an image
az acr repository delete --name $ACR_NAME --image samples/sample-app:v1

# Show ACR usage
az acr show-usage --name $ACR_NAME

# Get ACR credentials (if admin enabled)
az acr credential show --name $ACR_NAME

# Update ACR SKU
az acr update --name $ACR_NAME --sku Premium

# Check ACR health
az acr check-health --name $ACR_NAME
```

---

## Next Steps

Now that ACR is configured and ready, proceed to:

👉 **[03-aks-setup.md](./03-aks-setup.md)** - Set up Azure Kubernetes Service and attach ACR

---

## Additional Resources

- [ACR Best Practices](https://docs.microsoft.com/azure/container-registry/container-registry-best-practices)
- [ACR Authentication Options](https://docs.microsoft.com/azure/container-registry/container-registry-authentication)
- [ACR Service Tiers](https://docs.microsoft.com/azure/container-registry/container-registry-skus)
- [ACR Tasks for CI/CD](https://docs.microsoft.com/azure/container-registry/container-registry-tasks-overview)
- [Geo-replication in ACR](https://docs.microsoft.com/azure/container-registry/container-registry-geo-replication)
- [ACR Content Trust](https://docs.microsoft.com/azure/container-registry/container-registry-content-trust)
