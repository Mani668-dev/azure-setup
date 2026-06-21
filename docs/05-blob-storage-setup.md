# Azure Blob Storage Setup Guide

## 📋 Overview

Azure Blob Storage is a scalable object storage solution for unstructured data. This guide covers creating a Storage Account, configuring Blob containers, implementing Azure AD authentication, and setting up appropriate RBAC access for secure blob access.

## Prerequisites

- Completed [Azure AD Setup](./01-azure-ad-setup.md)
- Azure subscription with Contributor role
- Azure CLI installed
- Service principal credentials from Azure AD setup

## Table of Contents

1. [Create Storage Account](#1-create-storage-account)
2. [Create Blob Container](#2-create-blob-container)
3. [Configure Azure AD Authentication](#3-configure-azure-ad-authentication)
4. [Set Up RBAC Roles](#4-set-up-rbac-roles)
5. [Test Access with Azure AD](#5-test-access-with-azure-ad)
6. [Configure Storage Account Security](#6-configure-storage-account-security)
7. [Access from Applications](#7-access-from-applications)
8. [Enable Advanced Features](#8-enable-advanced-features)
9. [Verification](#9-verification)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Create Storage Account

### Step 1.1: Create Storage Account via Azure Portal

1. Navigate to [Azure Portal](https://portal.azure.com)
2. In the search bar, type **"Storage accounts"**
3. Click **"Storage accounts"**
4. Click **"+ Create"**

📸 **Screenshot: Storage accounts page**
```
Storage accounts > + Create
```

#### Basics Tab

5. Fill in the **Basics** tab:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Storage account name**: `staksdemostorage` (must be globally unique, 3-24 lowercase letters/numbers)
   - **Region**: `East US` (same as other resources)
   - **Performance**: `Standard` (or Premium for high-performance needs)
   - **Redundancy**: `Locally-redundant storage (LRS)` (or GRS/ZRS for production)

📸 **Screenshot: Create storage account - Basics**
```
Basics tab - Project details
Subscription: [Your subscription]
Resource group: rg-aks-demo

Instance details
Storage account name: staksdemostorage
Region: East US
Performance: • Standard  ○ Premium
Redundancy: Locally-redundant storage (LRS) ▼

[Next: Advanced >]
```

6. Click **"Next: Advanced >"**


#### Advanced Tab

7. In the **Advanced** tab:
   - **Require secure transfer for REST API operations**: Check ✓
   - **Enable storage account key access**: Check ✓ (can disable after Azure AD setup)
   - **Default to Azure Active Directory authorization in the Azure portal**: Check ✓
   - **Minimum TLS version**: `Version 1.2`
   - **Permitted scope for copy operations**: `From any storage account`
   - **Enable hierarchical namespace**: Uncheck (unless using Data Lake Gen2)
   - **Access tier**: `Hot` (for frequently accessed data)

📸 **Screenshot: Create storage account - Advanced**
```
Advanced tab
Security
☑ Require secure transfer for REST API operations
☑ Enable storage account key access
☑ Default to Azure Active Directory authorization in the Azure portal
☑ Enable infrastructure encryption
Minimum TLS version: Version 1.2 ▼

Blob storage
Access tier (default): • Hot  ○ Cool

Azure Files
☐ Enable large file shares

Data Lake Storage Gen2
☐ Enable hierarchical namespace

[Next: Networking >]
```

8. Click **"Next: Networking >"**

#### Networking Tab

9. In the **Networking** tab:
   - **Network access**: Select `Enable public access from all networks`
   - For production, consider `Enable public access from selected virtual networks and IP addresses`
   - **Routing preference**: `Microsoft network routing`

📸 **Screenshot: Create storage account - Networking**
```
Networking tab
Network connectivity
Network access:
• Enable public access from all networks
○ Enable public access from selected virtual networks and IP addresses
○ Disable public access and use private access

Routing preference
• Microsoft network routing
○ Internet routing

[Next: Data protection >]
```

10. Click **"Next: Data protection >"**

#### Data Protection Tab

11. In the **Data protection** tab:
    - **Enable point-in-time restore for containers**: Check ✓ (recommended)
    - **Days to retain deleted blobs**: `7` days
    - **Enable soft delete for blobs**: Check ✓
    - **Days to retain deleted blob versions**: `7` days
    - **Enable versioning for blobs**: Check ✓
    - **Enable soft delete for containers**: Check ✓

📸 **Screenshot: Create storage account - Data protection**
```
Data protection tab
Recovery
☑ Enable point-in-time restore for containers
Days to retain deleted blobs: 7

☑ Enable soft delete for blobs
Days to retain deleted blob versions: 7

☑ Enable versioning for blobs

☑ Enable soft delete for containers
Days to retain deleted containers: 7

Tracking
☐ Enable blob change feed

[Next: Encryption >]
```

12. Click **"Next: Encryption >"**


#### Encryption Tab

13. In the **Encryption** tab:
    - **Encryption type**: `Microsoft-managed keys (MMK)` (or customer-managed for more control)
    - **Enable support for customer-managed keys**: Select `Blobs and files only`
    - **Enable infrastructure encryption**: Check ✓

📸 **Screenshot: Create storage account - Encryption**
```
Encryption tab
Encryption type: • Microsoft-managed keys (MMK)
                 ○ Customer-managed keys (CMK)

Enable support for customer-managed keys: Blobs and files only ▼
☑ Enable infrastructure encryption

[Next: Tags >]
```

14. Click **"Next: Tags >"**

#### Tags Tab

15. Add tags (optional):
    - **Name**: `Environment`, **Value**: `Development`
    - **Name**: `Purpose`, **Value**: `Demo`

📸 **Screenshot: Tags tab**
```
Tags tab
Name          | Value
Environment   | Development
Purpose       | Demo

[Next: Review + create >]
```

16. Click **"Review + create"**
17. Review all settings
18. Click **"Create"**

📸 **Screenshot: Review + create**
```
Review + create tab
✓ Validation passed

Basics
Storage account name: staksdemostorage
Location: East US
Performance: Standard
Replication: Locally-redundant storage (LRS)

Advanced
Secure transfer: Enabled
Azure AD authorization default: Enabled

Networking
Public network access: Enabled from all networks

[Create button]
```

19. Wait for deployment (1-2 minutes)
20. Click **"Go to resource"**

📸 **Screenshot: Deployment complete**
```
Your deployment is complete
Deployment name: staksdemostorage
Resource group: rg-aks-demo
[Go to resource button]
```

### Step 1.2: Create Storage Account via Azure CLI

```bash
# Set variables
RESOURCE_GROUP="rg-aks-demo"
LOCATION="eastus"
STORAGE_ACCOUNT_NAME="staksdemostorage"  # Must be globally unique
SKU="Standard_LRS"  # Options: Standard_LRS, Standard_GRS, Standard_ZRS, Premium_LRS

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku $SKU \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --default-action Allow

echo "Storage account created successfully"

# Get storage account details
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{Name:name, Location:location, SKU:sku.name, PrimaryEndpoints:primaryEndpoints.blob}" \
  --output table
```

### Step 1.3: Note Down Storage Account Information

**From Azure Portal:**

📸 **Screenshot: Storage account Overview**
```
staksdemostorage | Overview
Status: Available
Location: East US
Subscription: Your subscription
Resource group: rg-aks-demo
Performance: Standard
Replication: Locally-redundant storage (LRS)

Primary endpoints
Blob service: https://staksdemostorage.blob.core.windows.net/
```

**Save these values:**

```bash
STORAGE_ACCOUNT_NAME=staksdemostorage
STORAGE_ACCOUNT_RESOURCE_ID=/subscriptions/.../resourceGroups/rg-aks-demo/providers/Microsoft.Storage/storageAccounts/staksdemostorage
BLOB_ENDPOINT=https://staksdemostorage.blob.core.windows.net/
```


---

## 2. Create Blob Container

### Step 2.1: Create Container via Azure Portal

1. In your storage account, click **"Containers"** in the left menu (under Data storage)
2. Click **"+ Container"**

📸 **Screenshot: Containers page**
```
staksdemostorage | Containers
+ Container | Refresh | Change access level

[Empty - no containers yet]
```

3. Fill in the container details:
   - **Name**: `demo-container`
   - **Public access level**: `Private (no anonymous access)`

📸 **Screenshot: New container dialog**
```
New container
Name: demo-container
Public access level: Private (no anonymous access) ▼

Note: Public access level can be changed after the container is created.
[Create button]
```

4. Click **"Create"**

5. Container appears in the list

📸 **Screenshot: Container created**
```
staksdemostorage | Containers
Name              | Public access level | Last modified
demo-container    | Private            | 6/21/2026, 12:00 PM
```

### Step 2.2: Create Additional Containers

Repeat the process to create more containers for different purposes:

- `app-data` - For application data
- `backup` - For backups
- `logs` - For log files

📸 **Screenshot: Multiple containers**
```
staksdemostorage | Containers
Name              | Public access level | Last modified
app-data          | Private            | 6/21/2026, 12:01 PM
backup            | Private            | 6/21/2026, 12:01 PM
demo-container    | Private            | 6/21/2026, 12:00 PM
logs              | Private            | 6/21/2026, 12:02 PM
```

### Step 2.3: Create Container via Azure CLI

```bash
# Create container using Azure AD authentication
az storage container create \
  --name demo-container \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login

# Create multiple containers
for container in app-data backup logs; do
  az storage container create \
    --name $container \
    --account-name $STORAGE_ACCOUNT_NAME \
    --auth-mode login
  echo "Container '$container' created"
done

# List containers
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --output table

# Expected output:
# Name            Lease Status    Last Modified
# --------------  --------------  -------------------------
# app-data        unlocked        2026-06-21T12:01:00+00:00
# backup          unlocked        2026-06-21T12:01:00+00:00
# demo-container  unlocked        2026-06-21T12:00:00+00:00
# logs            unlocked        2026-06-21T12:02:00+00:00
```

### Step 2.4: Upload Sample Blob

**Via Azure Portal:**

1. Click on a container (e.g., `demo-container`)
2. Click **"Upload"**
3. Click **"Browse for files"** or drag and drop a file
4. Select a file from your computer
5. Click **"Upload"**

📸 **Screenshot: Upload blob**
```
Upload blob
Select files: sample-file.txt [Browse]

Advanced
☐ Overwrite if files already exist
Blob type: Block blob ▼
Block size: 4 MiB ▼
Access tier: Hot (inferred) ▼

[Upload button]
```

**Via Azure CLI:**

```bash
# Create a sample file
echo "Hello from Azure Blob Storage!" > sample-file.txt

# Upload blob
az storage blob upload \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --name sample-file.txt \
  --file sample-file.txt \
  --auth-mode login

echo "Blob uploaded successfully"

# List blobs in container
az storage blob list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --auth-mode login \
  --output table

# Expected output:
# Name              Blob Type    Length    Content Type              Last Modified
# ----------------  -----------  --------  ------------------------  -------------------------
# sample-file.txt   BlockBlob    31        application/octet-stream  2026-06-21T12:05:00+00:00
```


---

## 3. Configure Azure AD Authentication

### Step 3.1: Understand Azure AD Authentication for Blob Storage

Azure Storage supports three authorization methods:
1. **Azure AD (Recommended)**: Use Azure AD identities with RBAC
2. **Shared Key**: Storage account keys (full access)
3. **Shared Access Signature (SAS)**: Time-limited, scoped access

We'll focus on Azure AD authentication.

### Step 3.2: Disable Shared Key Access (Optional, Recommended for Production)

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Configuration"** under Settings
3. Scroll to **"Allow storage account key access"**
4. Toggle to **"Disabled"**
5. Click **"Save"**

📸 **Screenshot: Storage account Configuration**
```
staksdemostorage | Configuration

Security
Secure transfer required: Enabled ▼
Minimum TLS version: Version 1.2 ▼
Permitted scope for copy operations: From any storage account ▼
Allow storage account key access: Enabled → [Toggle to Disabled]

⚠️ Disabling key access means only Azure AD authentication will work

[Save button]
```

**Via Azure CLI:**

```bash
# Disable shared key access
az storage account update \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --allow-shared-key-access false

echo "Shared key access disabled - Azure AD authentication required"

# To re-enable (if needed for testing):
# az storage account update \
#   --name $STORAGE_ACCOUNT_NAME \
#   --resource-group $RESOURCE_GROUP \
#   --allow-shared-key-access true
```

### Step 3.3: Configure Default Authentication Method

**Via Azure Portal:**

1. In storage account, go to **"Configuration"**
2. Under **"Default to Azure Active Directory authorization"**
3. Ensure it's **"Enabled"**

📸 **Screenshot: Azure AD authorization default**
```
staksdemostorage | Configuration

Azure Active Directory
Default to Azure Active Directory authorization in the Azure portal: Enabled ▼

When enabled, the Azure portal will use Azure AD authorization by default 
when you access blob and queue data.
```

This ensures the Azure Portal uses Azure AD instead of access keys by default.

---

## 4. Set Up RBAC Roles

### Step 4.1: Understand Storage RBAC Roles

**Common roles for Blob Storage:**

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Storage Blob Data Owner** | Full access including ACL management | Admins, full control |
| **Storage Blob Data Contributor** | Read, write, delete blobs | Applications that need write access |
| **Storage Blob Data Reader** | Read and list blobs | Read-only applications |
| **Storage Blob Delegator** | Get user delegation key for SAS | Creating user delegation SAS tokens |

### Step 4.2: Assign Role to Current User

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Access Control (IAM)"** in the left menu
3. Click **"+ Add"** > **"Add role assignment"**

📸 **Screenshot: Access control (IAM)**
```
staksdemostorage | Access control (IAM)
Check access | Role assignments | + Add

+ Add > Add role assignment
```

4. In the **Role** tab:
   - Search for **"Storage Blob Data Contributor"**
   - Select it
   - Click **"Next"**

📸 **Screenshot: Add role assignment - Role tab**
```
Add role assignment - Role tab

Search: Storage Blob Data Contributor

☑ Storage Blob Data Contributor
Grants read, write, and delete access to Azure Storage blob containers and data

Members | Conditions | Review + assign

[Next button]
```


5. In the **Members** tab:
   - **Assign access to**: Select `User, group, or service principal`
   - Click **"+ Select members"**
   - Search for your user account
   - Select it
   - Click **"Select"**
   - Click **"Next"**

📸 **Screenshot: Add role assignment - Members tab**
```
Add role assignment - Members tab

Assign access to: • User, group, or service principal
                   ○ Managed identity

+ Select members
Search: [your-email@domain.com]

Selected members:
- Your Name (your-email@domain.com)

[Next button]
```

6. Review and click **"Review + assign"**

📸 **Screenshot: Review + assign**
```
Review + assign tab

Role: Storage Blob Data Contributor
Scope: staksdemostorage
Members: Your Name (your-email@domain.com)

[Review + assign button]
```

**Via Azure CLI:**

```bash
# Get current user's object ID
CURRENT_USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Assign Storage Blob Data Contributor role to current user
az role assignment create \
  --assignee $CURRENT_USER_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

echo "Role assigned to current user"

# Verify role assignment
az role assignment list \
  --assignee $CURRENT_USER_OBJECT_ID \
  --scope $STORAGE_ID \
  --output table
```

### Step 4.3: Assign Role to Service Principal

**From Azure AD setup, you should have:**
- `AKS_SP_CLIENT_ID` - Service principal client ID

```bash
# Set service principal ID (from Azure AD setup)
AKS_SP_CLIENT_ID="your-service-principal-client-id"

# Assign Storage Blob Data Contributor role to service principal
az role assignment create \
  --assignee $AKS_SP_CLIENT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

echo "Role assigned to service principal"

# Verify
az role assignment list \
  --assignee $AKS_SP_CLIENT_ID \
  --scope $STORAGE_ID \
  --output table
```

### Step 4.4: Assign Role to Managed Identity

If you're using managed identities with AKS or VMs:

```bash
# Get managed identity principal ID (from Azure AD setup)
MANAGED_IDENTITY_PRINCIPAL_ID="your-managed-identity-principal-id"

# Assign role to managed identity
az role assignment create \
  --assignee $MANAGED_IDENTITY_PRINCIPAL_ID \
  --role "Storage Blob Data Reader" \
  --scope $STORAGE_ID

echo "Role assigned to managed identity"
```

### Step 4.5: Assign Role at Container Level (Fine-grained Access)

For more granular control, assign roles at the container level:

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Containers"**
3. Click on a container (e.g., `demo-container`)
4. Click **"Access Control (IAM)"**
5. Click **"+ Add"** > **"Add role assignment"**
6. Select role and assign to user/service principal/managed identity

📸 **Screenshot: Container-level IAM**
```
demo-container | Access Control (IAM)
+ Add > Add role assignment

Assign "Storage Blob Data Reader" to specific users
Assign "Storage Blob Data Contributor" to app service principals
```

**Via Azure CLI:**

```bash
# Get container resource ID
CONTAINER_ID="$STORAGE_ID/blobServices/default/containers/demo-container"

# Assign role at container level
az role assignment create \
  --assignee $AKS_SP_CLIENT_ID \
  --role "Storage Blob Data Reader" \
  --scope $CONTAINER_ID

echo "Container-level role assigned"
```


---

## 5. Test Access with Azure AD

### Step 5.1: Test Access via Azure Portal

1. Go to your storage account
2. Click **"Containers"**
3. Click on `demo-container`
4. You should see the files (if you have appropriate role assigned)

📸 **Screenshot: Accessing container with Azure AD**
```
demo-container | Overview

Using Azure AD authorization ✓

Name              | Type       | Size  | Last modified
sample-file.txt   | BlockBlob  | 31 B  | 6/21/2026, 12:05 PM
```

If you see an error, check your role assignments.

### Step 5.2: Test Access via Azure CLI

```bash
# Ensure you're logged in
az login

# List containers using Azure AD authentication
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --output table

# List blobs in a container
az storage blob list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --auth-mode login \
  --output table

# Download a blob
az storage blob download \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --name sample-file.txt \
  --file ./downloaded-file.txt \
  --auth-mode login

echo "File downloaded successfully"
cat ./downloaded-file.txt

# Upload a new blob
echo "Test blob content" > test-upload.txt
az storage blob upload \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --name test-upload.txt \
  --file test-upload.txt \
  --auth-mode login

echo "Blob uploaded successfully"

# Verify upload
az storage blob list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --auth-mode login \
  --query "[].name" \
  --output tsv
```

### Step 5.3: Test Access with Service Principal

```bash
# Set service principal credentials (from Azure AD setup)
AKS_SP_CLIENT_ID="your-service-principal-client-id"
AKS_SP_CLIENT_SECRET="your-service-principal-secret"
TENANT_ID=$(az account show --query tenantId -o tsv)

# Login with service principal
az login --service-principal \
  --username $AKS_SP_CLIENT_ID \
  --password $AKS_SP_CLIENT_SECRET \
  --tenant $TENANT_ID

# Try to list containers
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --output table

# If successful, service principal has access
echo "✓ Service principal can access blob storage"

# Logout and login back with your user
az logout
az login
```

### Step 5.4: Test Access with Python (Azure SDK)

Create a Python script to test programmatic access:

```bash
# Install Azure SDK
pip install azure-identity azure-storage-blob

# Create Python test script
cat << 'EOF' > test-blob-access.py
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

# Storage account details
account_name = "staksdemostorage"
account_url = f"https://{account_name}.blob.core.windows.net"

# Authenticate with Azure AD
credential = DefaultAzureCredential()

# Create blob service client
blob_service_client = BlobServiceClient(account_url, credential=credential)

# List containers
print("Containers:")
containers = blob_service_client.list_containers()
for container in containers:
    print(f"  - {container.name}")

# List blobs in demo-container
print("\nBlobs in demo-container:")
container_client = blob_service_client.get_container_client("demo-container")
blobs = container_client.list_blobs()
for blob in blobs:
    print(f"  - {blob.name} ({blob.size} bytes)")

# Upload a test blob
print("\nUploading test blob...")
blob_client = container_client.get_blob_client("test-from-python.txt")
blob_client.upload_blob("Hello from Python with Azure AD!", overwrite=True)
print("✓ Upload successful")

# Download the blob
print("\nDownloading test blob...")
download_stream = blob_client.download_blob()
content = download_stream.readall().decode('utf-8')
print(f"Downloaded content: {content}")

print("\n✓ All tests passed!")
EOF

# Run the script
python test-blob-access.py
```


---

## 6. Configure Storage Account Security

### Step 6.1: Configure Firewall and Virtual Networks

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Networking"** under Security + networking
3. Under **"Firewalls and virtual networks"**:
   - Select **"Enabled from selected virtual networks and IP addresses"**
   - Click **"+ Add existing virtual network"**
   - Select your VNet and subnet (e.g., AKS VNet)
   - Add your client IP address
4. Click **"Save"**

📸 **Screenshot: Networking - Firewalls and virtual networks**
```
staksdemostorage | Networking

Firewalls and virtual networks
Public network access: • Enabled from selected virtual networks and IP addresses
                       ○ Enabled from all networks
                       ○ Disabled

Virtual networks
+ Add existing virtual network
Name                    | Subnet
vnet-jumpserver        | subnet-jumpserver

Firewall
+ Add your client IP address (123.45.67.89)
Address range: 123.45.67.89

Exceptions
☑ Allow Azure services on the trusted services list to access this storage account
☑ Allow read access to storage logging from any network
☑ Allow read access to storage metrics from any network

[Save button]
```

**Via Azure CLI:**

```bash
# Get your public IP
MY_IP=$(curl -s ifconfig.me)

# Update network rules
az storage account update \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --default-action Deny

# Add your IP to firewall
az storage account network-rule add \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --ip-address $MY_IP

# Add virtual network rule (if using VNet)
VNET_NAME="vnet-jumpserver"
SUBNET_NAME="subnet-jumpserver"

# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --query id -o tsv)

# Add VNet rule
az storage account network-rule add \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --subnet $SUBNET_ID

# Allow trusted Azure services
az storage account update \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --bypass AzureServices

echo "Network rules configured"
```

### Step 6.2: Enable Advanced Threat Protection

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Microsoft Defender for Cloud"** under Security + networking
3. Click **"Enable Microsoft Defender for Storage"**

📸 **Screenshot: Microsoft Defender for Cloud**
```
staksdemostorage | Microsoft Defender for Cloud

Microsoft Defender for Storage
Status: Not enabled

Microsoft Defender for Storage detects unusual and potentially harmful 
attempts to access or exploit your storage accounts.

[Enable Microsoft Defender for Storage button]
```

**Via Azure CLI:**

```bash
# Enable Advanced Threat Protection
az security atp storage update \
  --resource-group $RESOURCE_GROUP \
  --storage-account $STORAGE_ACCOUNT_NAME \
  --is-enabled true

echo "Advanced Threat Protection enabled"
```

### Step 6.3: Configure Blob Versioning and Soft Delete

Already configured during creation, but can be modified:

```bash
# Enable blob versioning
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-versioning true

# Configure soft delete
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-delete-retention true \
  --delete-retention-days 7

# Configure container soft delete
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-container-delete-retention true \
  --container-delete-retention-days 7

echo "Soft delete and versioning configured"
```

### Step 6.4: Enable Blob Encryption

```bash
# Encryption is enabled by default with Microsoft-managed keys
# Verify encryption settings
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "encryption" \
  --output json

# Enable infrastructure encryption (if not already enabled)
# Note: Can only be set during account creation
# If you need to enable it, must create a new storage account
```


---

## 7. Access from Applications

### Step 7.1: Access from AKS Pod

Create a Kubernetes deployment that accesses blob storage using Azure AD.

**Option 1: Using Pod Identity (Recommended)**

First, install Azure AD Pod Identity on your AKS cluster:

```bash
# Install AAD Pod Identity
kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml

# Create Azure Identity
IDENTITY_NAME="id-blob-access"
az identity create \
  --name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP

# Get identity details
IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv)
IDENTITY_RESOURCE_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)
IDENTITY_PRINCIPAL_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query principalId -o tsv)

# Assign Storage Blob Data Contributor role to identity
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Create AzureIdentity in Kubernetes
cat << EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: blob-identity
  namespace: default
spec:
  type: 0
  resourceID: $IDENTITY_RESOURCE_ID
  clientID: $IDENTITY_CLIENT_ID
EOF

# Create AzureIdentityBinding
cat << EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: blob-identity-binding
  namespace: default
spec:
  azureIdentity: blob-identity
  selector: blob-app
EOF
```

**Option 2: Using Workload Identity (Newer, Recommended for new clusters)**

```bash
# Enable workload identity on AKS
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get OIDC issuer URL
OIDC_ISSUER=$(az aks show --name $AKS_CLUSTER_NAME --resource-group $RESOURCE_GROUP --query oidcIssuerProfile.issuerUrl -o tsv)

# Create service account
kubectl create serviceaccount blob-sa --namespace default

# Create federated credential
az identity federated-credential create \
  --name blob-federated-credential \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $OIDC_ISSUER \
  --subject system:serviceaccount:default:blob-sa

# Annotate service account
kubectl annotate serviceaccount blob-sa \
  azure.workload.identity/client-id=$IDENTITY_CLIENT_ID \
  --namespace default
```

### Step 7.2: Deploy Application Pod

Create a sample application that accesses blob storage:

```bash
cat << EOF > blob-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blob-app
      aadpodidbinding: blob-app  # For Pod Identity
  template:
    metadata:
      labels:
        app: blob-app
        aadpodidbinding: blob-app  # For Pod Identity
        azure.workload.identity/use: "true"  # For Workload Identity
    spec:
      serviceAccountName: blob-sa  # For Workload Identity
      containers:
      - name: blob-app
        image: python:3.9-slim
        command: ["/bin/sh"]
        args:
          - -c
          - |
            pip install azure-identity azure-storage-blob
            python3 << 'SCRIPT'
            from azure.identity import DefaultAzureCredential
            from azure.storage.blob import BlobServiceClient
            import time
            
            account_name = "$STORAGE_ACCOUNT_NAME"
            account_url = f"https://{account_name}.blob.core.windows.net"
            
            while True:
                try:
                    credential = DefaultAzureCredential()
                    blob_service_client = BlobServiceClient(account_url, credential=credential)
                    
                    containers = blob_service_client.list_containers()
                    print("Containers:")
                    for container in containers:
                        print(f"  - {container.name}")
                    
                    print("✓ Successfully accessed blob storage with Azure AD")
                    time.sleep(60)
                except Exception as e:
                    print(f"Error: {e}")
                    time.sleep(10)
            SCRIPT
        env:
        - name: AZURE_CLIENT_ID
          value: "$IDENTITY_CLIENT_ID"
EOF

# Apply deployment
kubectl apply -f blob-app-deployment.yaml

# Check logs
kubectl logs -f deployment/blob-app
```


### Step 7.3: Access from Jump Server

SSH into your Jump Server and test blob access:

```bash
# Install Azure Storage Blob library
pip3 install azure-identity azure-storage-blob

# Create test script
cat << 'EOF' > blob-test.py
from azure.identity import AzureCliCredential
from azure.storage.blob import BlobServiceClient
import sys

try:
    account_name = "staksdemostorage"
    account_url = f"https://{account_name}.blob.core.windows.net"
    
    # Use Azure CLI credential (since we're logged in via az login)
    credential = AzureCliCredential()
    blob_service_client = BlobServiceClient(account_url, credential=credential)
    
    # List containers
    print("Containers in storage account:")
    containers = blob_service_client.list_containers()
    for container in containers:
        print(f"  - {container.name}")
    
    # List blobs in demo-container
    print("\nBlobs in demo-container:")
    container_client = blob_service_client.get_container_client("demo-container")
    blobs = container_client.list_blobs()
    for blob in blobs:
        print(f"  - {blob.name} ({blob.size} bytes)")
    
    print("\n✓ Successfully accessed blob storage from Jump Server!")
    
except Exception as e:
    print(f"Error: {e}")
    sys.exit(1)
EOF

# Run the script
python3 blob-test.py
```

### Step 7.4: Access from Azure Function or App Service

For Azure Functions or App Service, enable managed identity:

```bash
# Example: Enable managed identity for App Service
APP_NAME="myapp"

# Enable system-assigned managed identity
az webapp identity assign \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP

# Get identity principal ID
APP_IDENTITY=$(az webapp identity show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

# Assign Storage role to app identity
az role assignment create \
  --assignee $APP_IDENTITY \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

echo "App Service can now access blob storage with managed identity"
```

**Sample code for Azure Function (Python):**

```python
import azure.functions as func
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

def main(req: func.HttpRequest) -> func.HttpResponse:
    try:
        # Storage account details
        account_name = "staksdemostorage"
        account_url = f"https://{account_name}.blob.core.windows.net"
        
        # Use managed identity
        credential = DefaultAzureCredential()
        blob_service_client = BlobServiceClient(account_url, credential=credential)
        
        # List containers
        containers = [c.name for c in blob_service_client.list_containers()]
        
        return func.HttpResponse(
            f"Containers: {', '.join(containers)}",
            status_code=200
        )
    except Exception as e:
        return func.HttpResponse(
            f"Error: {str(e)}",
            status_code=500
        )
```

---

## 8. Enable Advanced Features

### Step 8.1: Configure Lifecycle Management

Automatically move or delete old blobs to save costs.

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Lifecycle management"** under Data management
3. Click **"+ Add a rule"**
4. Configure rule:
   - **Rule name**: `move-old-logs-to-cool`
   - **Rule scope**: Limit blobs with filters
   - **Blob type**: Block blobs
5. Define conditions:
   - If blobs were last modified more than 30 days ago → Move to cool tier
   - If blobs were last modified more than 90 days ago → Delete
6. Click **"Add"**

📸 **Screenshot: Lifecycle management rule**
```
Lifecycle management | Add a rule

Details
Rule name: move-old-logs-to-cool
☑ Enable rule
Rule scope: • Limit blobs with filters

Filters
Blob types: ☑ Block blobs
Blob subtype: Base blobs ▼
Prefix match: logs/

Conditions
Last modified: More than (days ago) 30 → Move to cool storage
Last modified: More than (days ago) 90 → Delete the blob

[Add button]
```

**Via Azure CLI:**

```bash
# Create lifecycle policy
cat << 'EOF' > lifecycle-policy.json
{
  "rules": [
    {
      "enabled": true,
      "name": "move-old-logs-to-cool",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "delete": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    }
  ]
}
EOF

# Apply lifecycle policy
az storage account management-policy create \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --policy @lifecycle-policy.json

echo "Lifecycle management policy configured"
```


### Step 8.2: Enable Static Website Hosting (Optional)

Host static websites directly from blob storage.

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Static website"** under Data management
3. Toggle **"Enabled"**
4. Set **Index document name**: `index.html`
5. Set **Error document path**: `404.html`
6. Click **"Save"**

📸 **Screenshot: Static website**
```
staksdemostorage | Static website

Static website: [Disabled] → Toggle to [Enabled]
Index document name: index.html
Error document path: 404.html

Primary endpoint: https://staksdemostorage.z13.web.core.windows.net/
[Save button]
```

**Via Azure CLI:**

```bash
# Enable static website
az storage blob service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --static-website \
  --index-document index.html \
  --404-document 404.html

# Get website URL
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "primaryEndpoints.web" -o tsv
```

### Step 8.3: Configure CORS (Cross-Origin Resource Sharing)

Allow web applications to access blob storage from different domains.

**Via Azure CLI:**

```bash
# Set CORS rules
az storage cors add \
  --services b \
  --methods GET POST PUT \
  --origins "https://yourdomain.com" "http://localhost:3000" \
  --allowed-headers "*" \
  --exposed-headers "*" \
  --max-age 3600 \
  --account-name $STORAGE_ACCOUNT_NAME

echo "CORS configured"
```

### Step 8.4: Enable Blob Change Feed (for event processing)

```bash
# Enable change feed
az storage account blob-service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-change-feed true

echo "Change feed enabled"
```

### Step 8.5: Configure Blob Inventory

Generate reports of blob properties and metadata.

**Via Azure Portal:**

1. Go to your storage account
2. Click **"Blob inventory"** under Data management
3. Click **"+ Add rule"**
4. Configure inventory rule
5. Save

📸 **Screenshot: Blob inventory**
```
staksdemostorage | Blob inventory
+ Add rule

Rule name: daily-inventory
Destination container: inventory-reports
Schedule: Daily
Report format: CSV
Include blob snapshots: Yes
Include blob versions: Yes
```

---

## 9. Verification

### Step 9.1: Verify Storage Account Configuration

**Via Azure Portal:**

1. Go to your storage account
2. Check **Overview** page for status

📸 **Screenshot: Storage account healthy status**
```
staksdemostorage | Overview
Status: Available ✓
Location: East US
Performance: Standard
Replication: Locally-redundant storage (LRS)

Storage usage
Used capacity: 1.2 KB

Data services
Blob containers: 4
File shares: 0
Tables: 0
Queues: 0
```

**Via Azure CLI:**

```bash
# Check storage account status
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{Name:name, Status:statusOfPrimary, Location:location, SKU:sku.name, AADEnabled:azureFilesIdentityBasedAuthentication.directoryServiceOptions}" \
  --output table
```

### Step 9.2: Verify RBAC Configuration

```bash
# List all role assignments on storage account
az role assignment list \
  --scope $STORAGE_ID \
  --output table

# Should show assignments like:
# Principal                                    Role                            Scope
# ------------------------------------------   ------------------------------  ------
# your-email@domain.com                        Storage Blob Data Contributor   /subscriptions/.../staksdemostorage
# sp-aks-cluster                              Storage Blob Data Contributor   /subscriptions/.../staksdemostorage
```

### Step 9.3: Verify Blob Access

```bash
# List containers
az storage container list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --auth-mode login \
  --output table

# Upload a test file
echo "Verification test" > verify-test.txt
az storage blob upload \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --name verify-test.txt \
  --file verify-test.txt \
  --auth-mode login

# Download the file
az storage blob download \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --name verify-test.txt \
  --file verify-downloaded.txt \
  --auth-mode login

# Verify content
cat verify-downloaded.txt

# Clean up
rm verify-test.txt verify-downloaded.txt

echo "✓ Blob access verification successful"
```


### Step 9.4: Verify Security Settings

```bash
# Check network rules
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "networkRuleSet" \
  --output json

# Check encryption settings
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "encryption" \
  --output json

# Check if shared key access is disabled
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "allowSharedKeyAccess" \
  --output tsv
# Should return: false (if disabled for security)

# Check HTTPS requirement
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "enableHttpsTrafficOnly" \
  --output tsv
# Should return: true

echo "✓ Security verification complete"
```

---

## 10. Troubleshooting

### Issue 1: "AuthorizationPermissionMismatch" Error

**Symptoms:**
```bash
az storage container list --account-name staksdemostorage --auth-mode login
# AuthorizationPermissionMismatch: This request is not authorized to perform this operation using this permission.
```

**Cause:** User doesn't have appropriate RBAC role assigned.

**Solutions:**

```bash
# Solution 1: Check current role assignments
CURRENT_USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment list \
  --assignee $CURRENT_USER_OBJECT_ID \
  --scope $STORAGE_ID

# Solution 2: Assign appropriate role
az role assignment create \
  --assignee $CURRENT_USER_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Wait 2-3 minutes for propagation
sleep 180

# Solution 3: Try again
az storage container list --account-name $STORAGE_ACCOUNT_NAME --auth-mode login
```

### Issue 2: "The specified resource does not exist"

**Symptoms:**
```bash
az storage blob list --account-name staksdemostorage --container-name demo-container --auth-mode login
# ResourceNotFound: The specified resource does not exist
```

**Cause:** Container doesn't exist or incorrect container name.

**Solutions:**

```bash
# Solution 1: List all containers
az storage container list --account-name $STORAGE_ACCOUNT_NAME --auth-mode login --output table

# Solution 2: Create container if it doesn't exist
az storage container create \
  --account-name $STORAGE_ACCOUNT_NAME \
  --name demo-container \
  --auth-mode login

# Solution 3: Check for typos in container name (case-sensitive)
```

### Issue 3: Cannot access storage from specific IP

**Symptoms:**
- Can access from Azure Portal but not from local machine

**Cause:** Firewall rules blocking your IP

**Solutions:**

```bash
# Check current network rules
az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --query "networkRuleSet" \
  --output json

# Add your current IP
MY_IP=$(curl -s ifconfig.me)
az storage account network-rule add \
  --account-name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --ip-address $MY_IP

echo "IP $MY_IP added to firewall rules"

# Wait a minute and try again
sleep 60
```

### Issue 4: "Forbidden - Public access is not permitted"

**Symptoms:**
```bash
# Trying to access blob via URL
curl https://staksdemostorage.blob.core.windows.net/demo-container/file.txt
# <Error><Code>PublicAccessNotPermitted</Code></Error>
```

**Cause:** Public access is disabled (which is good for security!).

**Solutions:**

```bash
# Option 1: Use Azure AD authentication (recommended)
# Access via Azure CLI with --auth-mode login

# Option 2: Generate SAS token for temporary access
END_DATE=$(date -u -d "1 hour" '+%Y-%m-%dT%H:%M:%SZ')
SAS_TOKEN=$(az storage container generate-sas \
  --account-name $STORAGE_ACCOUNT_NAME \
  --name demo-container \
  --permissions r \
  --expiry $END_DATE \
  --auth-mode login \
  --as-user \
  --output tsv)

# Access with SAS token
curl "https://staksdemostorage.blob.core.windows.net/demo-container/file.txt?$SAS_TOKEN"

# Option 3: Enable public access (NOT recommended for production)
# az storage container set-permission \
#   --name demo-container \
#   --account-name $STORAGE_ACCOUNT_NAME \
#   --public-access blob \
#   --auth-mode login
```


### Issue 5: Service principal cannot access storage

**Symptoms:**
```bash
# Using service principal
az login --service-principal --username $SP_ID --password $SP_SECRET --tenant $TENANT_ID
az storage container list --account-name staksdemostorage --auth-mode login
# Error: AuthorizationPermissionMismatch
```

**Solutions:**

```bash
# Check if service principal has role assigned
az role assignment list \
  --assignee $SP_ID \
  --scope $STORAGE_ID

# Assign role if missing
az role assignment create \
  --assignee $SP_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Wait for propagation (5-10 minutes for service principals)
sleep 600

# Try again
```

### Issue 6: Soft-deleted blobs taking up space

**Symptoms:**
- Storage usage higher than expected
- Deleted blobs still counting towards quota

**Solutions:**

```bash
# List soft-deleted blobs
az storage blob list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --include d \
  --auth-mode login

# Permanently delete soft-deleted blobs
az storage blob list \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --include d \
  --auth-mode login \
  --query "[?properties.deleted].name" -o tsv | while read blob; do
    az storage blob undelete \
      --account-name $STORAGE_ACCOUNT_NAME \
      --container-name demo-container \
      --name "$blob" \
      --auth-mode login
    
    az storage blob delete \
      --account-name $STORAGE_ACCOUNT_NAME \
      --container-name demo-container \
      --name "$blob" \
      --auth-mode login \
      --delete-snapshots include
done

echo "Soft-deleted blobs permanently removed"
```

### Issue 7: Python SDK authentication fails

**Symptoms:**
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
# ClientAuthenticationError: DefaultAzureCredential failed to retrieve a token
```

**Solutions:**

```bash
# Solution 1: Ensure Azure CLI is logged in
az login
az account show

# Solution 2: Set environment variables for service principal
export AZURE_CLIENT_ID="your-sp-client-id"
export AZURE_CLIENT_SECRET="your-sp-client-secret"
export AZURE_TENANT_ID="your-tenant-id"

# Solution 3: Use AzureCliCredential explicitly
# In Python code:
# from azure.identity import AzureCliCredential
# credential = AzureCliCredential()

# Solution 4: Install/update Azure libraries
pip install --upgrade azure-identity azure-storage-blob
```

### Issue 8: Large file upload fails

**Symptoms:**
- Upload times out for files > 256 MB

**Solutions:**

```bash
# Use block blob upload with chunking
az storage blob upload \
  --account-name $STORAGE_ACCOUNT_NAME \
  --container-name demo-container \
  --name large-file.zip \
  --file ./large-file.zip \
  --max-connections 4 \
  --auth-mode login

# Or use Python SDK with custom chunk size
cat << 'EOF' > upload-large-file.py
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

account_url = "https://staksdemostorage.blob.core.windows.net"
credential = DefaultAzureCredential()
blob_service_client = BlobServiceClient(account_url, credential=credential)

container_client = blob_service_client.get_container_client("demo-container")
blob_client = container_client.get_blob_client("large-file.zip")

# Upload with custom settings
with open("large-file.zip", "rb") as data:
    blob_client.upload_blob(
        data,
        overwrite=True,
        max_concurrency=4,
        blob_type="BlockBlob"
    )
    
print("Large file uploaded successfully")
EOF

python3 upload-large-file.py
```

---

## Summary Checklist

After completing this guide, you should have:

- [ ] Created Azure Storage Account with appropriate settings
- [ ] Created Blob containers for different purposes
- [ ] Configured Azure AD authentication as default
- [ ] Disabled shared key access (optional, for production)
- [ ] Assigned RBAC roles to users, service principals, and managed identities
- [ ] Tested blob access using Azure AD with Azure CLI
- [ ] Configured storage account firewall and network rules
- [ ] Enabled security features (encryption, soft delete, versioning)
- [ ] Configured lifecycle management policies
- [ ] Tested programmatic access from applications
- [ ] Verified all security settings

### Important Information to Save

```bash
# Storage Account Information
STORAGE_ACCOUNT_NAME=staksdemostorage
STORAGE_RESOURCE_GROUP=rg-aks-demo
STORAGE_LOCATION=eastus
BLOB_ENDPOINT=https://staksdemostorage.blob.core.windows.net/

# Containers
# - demo-container (testing)
# - app-data (application data)
# - backup (backups)
# - logs (log files)

# RBAC Roles Assigned
# - Current User: Storage Blob Data Contributor
# - Service Principal (sp-aks-cluster): Storage Blob Data Contributor
# - Managed Identity (id-aks-managed-identity): Storage Blob Data Reader

# Security Settings
# - Shared key access: Disabled
# - HTTPS only: Enabled
# - Minimum TLS version: 1.2
# - Soft delete: Enabled (7 days retention)
# - Versioning: Enabled
# - Network access: Restricted to selected IPs/VNets
```


### Blob Storage Quick Reference Commands

**Container Operations:**

```bash
# List containers
az storage container list --account-name $STORAGE_ACCOUNT_NAME --auth-mode login --output table

# Create container
az storage container create --name mycontainer --account-name $STORAGE_ACCOUNT_NAME --auth-mode login

# Delete container
az storage container delete --name mycontainer --account-name $STORAGE_ACCOUNT_NAME --auth-mode login

# Check if container exists
az storage container exists --name mycontainer --account-name $STORAGE_ACCOUNT_NAME --auth-mode login
```

**Blob Operations:**

```bash
# Upload blob
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name demo-container --name file.txt --file ./file.txt --auth-mode login

# Download blob
az storage blob download --account-name $STORAGE_ACCOUNT_NAME --container-name demo-container --name file.txt --file ./downloaded.txt --auth-mode login

# List blobs
az storage blob list --account-name $STORAGE_ACCOUNT_NAME --container-name demo-container --auth-mode login --output table

# Delete blob
az storage blob delete --account-name $STORAGE_ACCOUNT_NAME --container-name demo-container --name file.txt --auth-mode login

# Copy blob
az storage blob copy start --account-name $STORAGE_ACCOUNT_NAME --destination-container demo-container --destination-blob newfile.txt --source-container demo-container --source-blob file.txt --auth-mode login
```

**RBAC Operations:**

```bash
# Assign Storage Blob Data Reader
az role assignment create --assignee user@domain.com --role "Storage Blob Data Reader" --scope $STORAGE_ID

# Assign Storage Blob Data Contributor
az role assignment create --assignee user@domain.com --role "Storage Blob Data Contributor" --scope $STORAGE_ID

# Assign Storage Blob Data Owner
az role assignment create --assignee user@domain.com --role "Storage Blob Data Owner" --scope $STORAGE_ID

# List role assignments
az role assignment list --scope $STORAGE_ID --output table

# Remove role assignment
az role assignment delete --assignee user@domain.com --role "Storage Blob Data Reader" --scope $STORAGE_ID
```

**Monitoring:**

```bash
# Show storage account usage
az storage account show-usage --location eastus

# Get storage account properties
az storage account show --name $STORAGE_ACCOUNT_NAME --resource-group $RESOURCE_GROUP

# View metrics
az monitor metrics list --resource $STORAGE_ID --metric-names "Transactions" "Ingress" "Egress" --output table
```

---

## Best Practices

### 1. Security
- ✅ Use Azure AD authentication instead of storage account keys
- ✅ Disable shared key access in production
- ✅ Enable HTTPS-only access
- ✅ Use minimum TLS version 1.2
- ✅ Implement network restrictions (firewall rules)
- ✅ Enable Microsoft Defender for Storage
- ✅ Use private endpoints for private network access
- ✅ Regularly rotate SAS tokens if used
- ✅ Enable soft delete and versioning
- ✅ Use customer-managed keys for sensitive data

### 2. Cost Optimization
- ✅ Implement lifecycle management policies
- ✅ Move infrequently accessed data to Cool or Archive tiers
- ✅ Delete old or unused blobs automatically
- ✅ Use appropriate redundancy level (LRS vs GRS)
- ✅ Monitor storage usage and set alerts
- ✅ Compress data before uploading
- ✅ Use reserved capacity for predictable workloads

### 3. Performance
- ✅ Use appropriate blob type (Block, Page, or Append)
- ✅ Upload large files in chunks
- ✅ Use multiple concurrent connections
- ✅ Enable CDN for frequently accessed content
- ✅ Co-locate storage and compute in same region
- ✅ Use Premium performance tier for latency-sensitive workloads
- ✅ Implement retry logic in applications

### 4. Reliability
- ✅ Use geo-redundant storage (GRS/GZRS) for critical data
- ✅ Enable versioning for data protection
- ✅ Implement point-in-time restore
- ✅ Regular backups to separate storage account
- ✅ Test disaster recovery procedures
- ✅ Monitor storage health and availability

### 5. Operations
- ✅ Use descriptive container and blob names
- ✅ Implement consistent naming conventions
- ✅ Tag resources for organization and cost tracking
- ✅ Set up monitoring and alerting
- ✅ Document access patterns and requirements
- ✅ Use Azure Storage Explorer for management
- ✅ Automate common tasks with scripts

---

## Next Steps

Now that Blob Storage is configured with Azure AD authentication, proceed to:

👉 **[06-architecture-best-practices.md](./06-architecture-best-practices.md)** - Review architecture diagrams and best practices

---

## Additional Resources

- [Azure Blob Storage Documentation](https://docs.microsoft.com/azure/storage/blobs/)
- [Azure AD Authentication for Storage](https://docs.microsoft.com/azure/storage/common/storage-auth-aad)
- [Storage Security Best Practices](https://docs.microsoft.com/azure/storage/blobs/security-recommendations)
- [Azure Storage RBAC Roles](https://docs.microsoft.com/azure/role-based-access-control/built-in-roles#storage)
- [Lifecycle Management Policies](https://docs.microsoft.com/azure/storage/blobs/storage-lifecycle-management-concepts)
- [Azure Storage SDKs](https://docs.microsoft.com/azure/storage/common/storage-samples)
- [Optimize Costs in Blob Storage](https://docs.microsoft.com/azure/storage/blobs/storage-blob-storage-tiers)
- [Monitor Azure Storage](https://docs.microsoft.com/azure/storage/common/monitor-storage)
