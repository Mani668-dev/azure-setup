# Azure Active Directory (Azure AD) Setup Guide

## 📋 Overview

Azure Active Directory (Azure AD, now Microsoft Entra ID) provides identity and access management for all Azure resources. This guide covers setting up Azure AD for authenticating and authorizing access to AKS, ACR, and Blob Storage.

## Prerequisites

- Azure subscription with Owner or User Access Administrator role
- Azure CLI installed
- Azure Portal access

## Table of Contents

1. [Verify Azure AD Tenant](#1-verify-azure-ad-tenant)
2. [Create Service Principals](#2-create-service-principals)
3. [Create App Registrations](#3-create-app-registrations)
4. [Configure RBAC Roles](#4-configure-rbac-roles)
5. [Create Managed Identities](#5-create-managed-identities)
6. [Verification](#6-verification)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Verify Azure AD Tenant

### Step 1.1: Check Your Azure AD Tenant

**Via Azure Portal:**

1. Navigate to [Azure Portal](https://portal.azure.com)
2. In the search bar at the top, type **"Azure Active Directory"**
3. Click on **"Azure Active Directory"** from the results

📸 **Screenshot: Azure Portal > Search "Azure Active Directory"**
```
[Top Search Bar] → Type "Azure Active Directory" → Select from dropdown
```

4. You should see your Azure AD Overview page showing:
   - Tenant name
   - Tenant ID
   - Primary domain

📸 **Screenshot: Azure AD Overview Page**
```
Azure Active Directory > Overview
- Tenant information
- Tenant ID (copy this for later use)
- Primary domain
```

**Via Azure CLI:**

```bash
# Login to Azure
az login

# List your Azure AD tenant information
az account show --query "{TenantId:tenantId, SubscriptionName:name, SubscriptionId:id}"

# Get detailed tenant info
az account tenant list
```

**Expected Output:**
```json
{
  "TenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "SubscriptionName": "Your Subscription Name",
  "SubscriptionId": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
}
```

### Step 1.2: Note Down Important Information

Create a text file to store these values (keep it secure):

```
TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
SUBSCRIPTION_ID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
SUBSCRIPTION_NAME=Your-Subscription-Name
```

---

## 2. Create Service Principals

Service principals are used for application authentication and automation.

### Step 2.1: Create Service Principal for AKS

**Via Azure Portal:**

1. Go to **Azure Active Directory**
2. Click **"App registrations"** in the left menu
3. Click **"+ New registration"**

📸 **Screenshot: Azure AD > App registrations > New registration**
```
Azure Active Directory > App registrations > + New registration
```

4. Fill in the registration form:
   - **Name**: `sp-aks-cluster`
   - **Supported account types**: Select "Accounts in this organizational directory only"
   - **Redirect URI**: Leave blank
5. Click **"Register"**

📸 **Screenshot: Register an application form**
```
Name: sp-aks-cluster
Supported account types: [Single tenant selected]
Redirect URI: (Optional) [blank]
[Register button]
```

6. After creation, you'll see the application overview page
7. **Copy and save these values**:
   - **Application (client) ID**
   - **Directory (tenant) ID**

📸 **Screenshot: App registration overview**
```
sp-aks-cluster
Application (client) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Directory (tenant) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### Step 2.2: Create Client Secret

1. In the same app registration page, click **"Certificates & secrets"** in the left menu
2. Click **"+ New client secret"**

📸 **Screenshot: Certificates & secrets page**
```
Certificates & secrets > Client secrets > + New client secret
```

3. Fill in the details:
   - **Description**: `aks-cluster-secret`
   - **Expires**: Select your preferred expiration (e.g., 12 months, 24 months)
4. Click **"Add"**

📸 **Screenshot: Add a client secret dialog**
```
Description: aks-cluster-secret
Expires: [24 months selected]
[Add button]
```

5. **IMPORTANT**: Copy the **Value** immediately (it won't be shown again)

📸 **Screenshot: Client secret created**
```
Client secrets
Description          | Value                    | Expires
aks-cluster-secret   | xxxXXXxxxXXXxxx...       | 6/21/2028
                       ⚠️ Copy now - won't show again
```

**Save these values:**
```
AKS_SP_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AKS_SP_CLIENT_SECRET=xxxXXXxxxXXXxxx...
```

**Via Azure CLI:**

```bash
# Create service principal for AKS
az ad sp create-for-rbac --name sp-aks-cluster --skip-assignment

# Output will show:
# {
#   "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "displayName": "sp-aks-cluster",
#   "password": "xxxXXXxxxXXXxxx...",
#   "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# }

# Save the appId as AKS_SP_CLIENT_ID
# Save the password as AKS_SP_CLIENT_SECRET
```

### Step 2.3: Create Service Principal for ACR (Optional)

If you need separate authentication for ACR:

**Via Azure CLI:**

```bash
# Create service principal for ACR
az ad sp create-for-rbac --name sp-acr-access --skip-assignment

# Save the output
ACR_SP_CLIENT_ID=<appId from output>
ACR_SP_CLIENT_SECRET=<password from output>
```

---

## 3. Create App Registrations

App registrations are used for user authentication and application access.

### Step 3.1: Create App Registration for Blob Storage Access

**Via Azure Portal:**

1. Go to **Azure Active Directory** > **App registrations**
2. Click **"+ New registration"**
3. Fill in:
   - **Name**: `app-blob-storage-access`
   - **Supported account types**: "Accounts in this organizational directory only"
   - **Redirect URI (optional)**: 
     - Platform: Web
     - URI: `http://localhost:8080/auth/callback` (for testing)
4. Click **"Register"**

📸 **Screenshot: App registration for Blob Storage**
```
Name: app-blob-storage-access
Supported account types: [Single tenant]
Redirect URI: Web | http://localhost:8080/auth/callback
[Register button]
```

### Step 3.2: Configure API Permissions

1. After registration, click **"API permissions"** in the left menu
2. You should see "Microsoft Graph > User.Read" by default
3. Click **"+ Add a permission"**

📸 **Screenshot: API permissions page**
```
API permissions > + Add a permission
Configured permissions:
- Microsoft Graph | User.Read | Delegated
```

4. Select **"Azure Storage"**
5. Select **"Delegated permissions"**
6. Check **"user_impersonation"**
7. Click **"Add permissions"**

📸 **Screenshot: Request API permissions**
```
Microsoft APIs > Azure Storage
Delegated permissions: ☑ user_impersonation
[Add permissions button]
```

8. Click **"Grant admin consent for [Your Tenant]"** button
9. Confirm by clicking **"Yes"**

📸 **Screenshot: Grant admin consent**
```
⚠️ Grant admin consent for Default Directory?
This will grant consent on behalf of all users in your organization.
[Yes] [No]
```

### Step 3.3: Create Client Secret for App

1. Click **"Certificates & secrets"**
2. Click **"+ New client secret"**
3. Description: `blob-access-secret`
4. Expires: 12 months
5. Click **"Add"**
6. **Copy the secret value immediately**

Save these values:
```
BLOB_APP_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
BLOB_APP_CLIENT_SECRET=xxxXXXxxxXXXxxx...
```

---

## 4. Configure RBAC Roles

Role-Based Access Control (RBAC) defines who can do what with Azure resources.

### Step 4.1: Understand Common RBAC Roles

**For ACR:**
- `AcrPull` - Pull images from registry
- `AcrPush` - Push and pull images
- `AcrDelete` - Delete images
- `Contributor` - Full access to ACR

**For AKS:**
- `Azure Kubernetes Service Cluster User Role` - Get cluster credentials
- `Azure Kubernetes Service Cluster Admin Role` - Full admin access

**For Blob Storage:**
- `Storage Blob Data Reader` - Read blob data
- `Storage Blob Data Contributor` - Read, write, delete blobs
- `Storage Blob Data Owner` - Full access including ACL management

### Step 4.2: Assign Roles via Azure Portal

**Example: Assign Storage Blob Data Contributor to a User**

1. Navigate to your **Subscription** or **Resource Group**
2. Click **"Access control (IAM)"** in the left menu

📸 **Screenshot: Access control (IAM)**
```
Resource Group > Access control (IAM)
- Check access
- Role assignments
- Grant access
```

3. Click **"+ Add"** > **"Add role assignment"**

📸 **Screenshot: Add role assignment button**
```
Access control (IAM) > + Add > Add role assignment
```

4. In the **Role** tab:
   - Search for **"Storage Blob Data Contributor"**
   - Select it
   - Click **"Next"**

📸 **Screenshot: Add role assignment - Role tab**
```
Role tab
Search: Storage Blob Data Contributor
☑ Storage Blob Data Contributor
Description: Allows for read, write and delete access to Azure Storage blob containers and data
[Next button]
```

5. In the **Members** tab:
   - **Assign access to**: Select "User, group, or service principal"
   - Click **"+ Select members"**
   - Search for your service principal or user
   - Click on it to select
   - Click **"Select"**

📸 **Screenshot: Add role assignment - Members tab**
```
Members tab
Assign access to: • User, group, or service principal
                   ○ Managed identity

+ Select members
Search: [user or service principal name]
Selected members: 
- sp-aks-cluster
[Next button]
```

6. Review the **Review + assign** tab
7. Click **"Review + assign"**

📸 **Screenshot: Review + assign tab**
```
Review + assign tab
Role: Storage Blob Data Contributor
Members: sp-aks-cluster
Scope: /subscriptions/xxx/resourceGroups/rg-aks-demo
[Review + assign button]
```

### Step 4.3: Assign Roles via Azure CLI

**Assign AcrPull role to AKS service principal:**

```bash
# Set variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
ACR_NAME="myacrregistry"  # Replace with your ACR name
SP_OBJECT_ID=$(az ad sp show --id $AKS_SP_CLIENT_ID --query id -o tsv)

# Get ACR resource ID
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

# Assign AcrPull role
az role assignment create \
  --assignee $AKS_SP_CLIENT_ID \
  --role AcrPull \
  --scope $ACR_ID

echo "Role assigned successfully"
```

**Assign Storage Blob Data Contributor:**

```bash
# Set variables
STORAGE_ACCOUNT="mystorageaccount"  # Replace with your storage account name
RESOURCE_GROUP="rg-aks-demo"

# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Assign role to service principal
az role assignment create \
  --assignee $AKS_SP_CLIENT_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

echo "Storage role assigned successfully"
```

**Assign AKS Cluster User role:**

```bash
# Get current user
CURRENT_USER=$(az account show --query user.name -o tsv)

# Assign AKS Cluster User role
az role assignment create \
  --assignee $CURRENT_USER \
  --role "Azure Kubernetes Service Cluster User Role" \
  --resource-group $RESOURCE_GROUP

echo "AKS Cluster User role assigned"
```

---

## 5. Create Managed Identities

Managed identities provide automatic credential management for Azure resources.

### Step 5.1: Understand Managed Identity Types

- **System-assigned**: Tied to the lifecycle of the Azure resource
- **User-assigned**: Independent identity that can be shared across resources

### Step 5.2: Create User-Assigned Managed Identity

**Via Azure Portal:**

1. In the search bar, type **"Managed Identities"**
2. Click on **"Managed Identities"** service

📸 **Screenshot: Search for Managed Identities**
```
[Top Search Bar] → "Managed Identities" → Select Managed Identities
```

3. Click **"+ Create"**

📸 **Screenshot: Managed Identities page**
```
Managed Identities > + Create
```

4. Fill in the details:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo` (or your resource group)
   - **Region**: Same as your AKS cluster (e.g., East US)
   - **Name**: `id-aks-managed-identity`
5. Click **"Review + create"**
6. Click **"Create"**

📸 **Screenshot: Create User Assigned Managed Identity**
```
Basics tab
Subscription: [Your subscription]
Resource group: rg-aks-demo
Region: East US
Name: id-aks-managed-identity
[Review + create button]
```

7. Wait for deployment to complete
8. Click **"Go to resource"**
9. Copy the **Client ID** and **Object (principal) ID**

📸 **Screenshot: Managed Identity Overview**
```
id-aks-managed-identity | Overview
Client ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Object (principal) ID: yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

**Via Azure CLI:**

```bash
# Create managed identity
az identity create \
  --name id-aks-managed-identity \
  --resource-group rg-aks-demo

# Get the details
az identity show \
  --name id-aks-managed-identity \
  --resource-group rg-aks-demo \
  --query "{ClientId:clientId, PrincipalId:principalId, Id:id}"

# Save these values
MANAGED_IDENTITY_CLIENT_ID=<ClientId from output>
MANAGED_IDENTITY_PRINCIPAL_ID=<PrincipalId from output>
MANAGED_IDENTITY_ID=<Id from output>
```

### Step 5.3: Assign Roles to Managed Identity

**Assign AcrPull role:**

**Via Azure Portal:**

1. Navigate to your **Container Registry** (ACR)
2. Click **"Access control (IAM)"**
3. Click **"+ Add"** > **"Add role assignment"**
4. Select **"AcrPull"** role
5. In Members tab, select **"Managed identity"**
6. Click **"+ Select members"**
7. Select **"User-assigned managed identity"**
8. Choose your managed identity: `id-aks-managed-identity`
9. Click **"Select"** > **"Review + assign"**

📸 **Screenshot: Assign role to Managed Identity**
```
Add role assignment > Members tab
Assign access to: • Managed identity
Select members:
- User-assigned managed identity
- id-aks-managed-identity
[Select button]
```

**Via Azure CLI:**

```bash
# Get ACR ID
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)

# Assign AcrPull role to managed identity
az role assignment create \
  --assignee $MANAGED_IDENTITY_PRINCIPAL_ID \
  --role AcrPull \
  --scope $ACR_ID

echo "AcrPull role assigned to managed identity"
```

---

## 6. Verification

### Step 6.1: Verify Service Principals

**Via Azure Portal:**

1. Go to **Azure Active Directory** > **App registrations**
2. Click **"All applications"** tab
3. Verify you see:
   - `sp-aks-cluster`
   - `app-blob-storage-access`

📸 **Screenshot: App registrations list**
```
App registrations > All applications
Display name              | Application ID
sp-aks-cluster           | xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
app-blob-storage-access  | yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

**Via Azure CLI:**

```bash
# List all service principals you created
az ad sp list --show-mine --query "[].{Name:displayName, AppId:appId}" -o table

# Expected output:
# Name                        AppId
# --------------------------  ------------------------------------
# sp-aks-cluster             xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# app-blob-storage-access    yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

### Step 6.2: Verify Role Assignments

**Via Azure Portal:**

1. Go to your **Resource Group** > **Access control (IAM)**
2. Click **"Role assignments"** tab
3. Review assigned roles

📸 **Screenshot: Role assignments**
```
Access control (IAM) > Role assignments
Role                              | Members
Storage Blob Data Contributor     | sp-aks-cluster
AcrPull                          | id-aks-managed-identity
```

**Via Azure CLI:**

```bash
# List role assignments for resource group
az role assignment list \
  --resource-group rg-aks-demo \
  --query "[].{Principal:principalName, Role:roleDefinitionName}" \
  -o table

# Check specific service principal roles
az role assignment list \
  --assignee $AKS_SP_CLIENT_ID \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  -o table
```

### Step 6.3: Verify Managed Identity

**Via Azure CLI:**

```bash
# List managed identities
az identity list \
  --resource-group rg-aks-demo \
  --query "[].{Name:name, ClientId:clientId, PrincipalId:principalId}" \
  -o table
```

### Step 6.4: Test Service Principal Authentication

```bash
# Test login with service principal
az login --service-principal \
  --username $AKS_SP_CLIENT_ID \
  --password $AKS_SP_CLIENT_SECRET \
  --tenant $TENANT_ID

# If successful, you should see subscription information
az account show

# Logout and login back with your user account
az logout
az login
```

---

## 7. Troubleshooting

### Issue 1: "Insufficient privileges to complete the operation"

**Cause**: You don't have permission to create service principals or assign roles.

**Solution**:
1. Contact your Azure AD administrator
2. Required roles:
   - Application Administrator (to create app registrations)
   - User Access Administrator or Owner (to assign roles)

### Issue 2: "Client secret has expired"

**Cause**: The client secret you created has expired.

**Solution**:
```bash
# Via Azure Portal:
# 1. Go to App registrations > Your app > Certificates & secrets
# 2. Delete expired secret
# 3. Create new client secret
# 4. Update your applications with the new secret

# Via Azure CLI:
# Create new credential for service principal
az ad sp credential reset --id $AKS_SP_CLIENT_ID
```

### Issue 3: Role assignment not working

**Cause**: Role assignments can take up to 5 minutes to propagate.

**Solution**:
```bash
# Wait a few minutes and verify again
sleep 300

# Check role assignment
az role assignment list --assignee $AKS_SP_CLIENT_ID --all

# If still not working, try reassigning
az role assignment delete --assignee $AKS_SP_CLIENT_ID --role "AcrPull" --scope $ACR_ID
az role assignment create --assignee $AKS_SP_CLIENT_ID --role "AcrPull" --scope $ACR_ID
```

### Issue 4: "Application with identifier was not found"

**Cause**: Service principal doesn't exist or wrong client ID.

**Solution**:
```bash
# Verify service principal exists
az ad sp show --id $AKS_SP_CLIENT_ID

# If not found, recreate it
az ad sp create-for-rbac --name sp-aks-cluster --skip-assignment
```

### Issue 5: Cannot see managed identity in role assignment

**Cause**: Managed identity not fully provisioned.

**Solution**:
```bash
# Check managed identity status
az identity show --name id-aks-managed-identity --resource-group rg-aks-demo

# Verify principalId exists
# Wait a few minutes if just created
# Then retry role assignment
```

---

## Summary Checklist

After completing this guide, you should have:

- [ ] Verified Azure AD tenant information
- [ ] Created service principal for AKS with client secret
- [ ] Created app registration for Blob Storage access
- [ ] Configured API permissions and granted admin consent
- [ ] Created user-assigned managed identity
- [ ] Assigned appropriate RBAC roles (AcrPull, Storage Blob Data Contributor, etc.)
- [ ] Verified all role assignments are working
- [ ] Saved all important IDs and secrets securely

### Important Values to Save

Keep these values in a secure location (e.g., Azure Key Vault):

```bash
# Tenant and Subscription
TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
SUBSCRIPTION_ID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy

# AKS Service Principal
AKS_SP_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AKS_SP_CLIENT_SECRET=xxxXXXxxxXXXxxx...

# Blob Storage App Registration
BLOB_APP_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
BLOB_APP_CLIENT_SECRET=xxxXXXxxxXXXxxx...

# Managed Identity
MANAGED_IDENTITY_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
MANAGED_IDENTITY_PRINCIPAL_ID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
MANAGED_IDENTITY_ID=/subscriptions/.../resourceGroups/.../providers/Microsoft.ManagedIdentity/userAssignedIdentities/id-aks-managed-identity
```

---

## Next Steps

Now that Azure AD is configured, proceed to:

👉 **[02-acr-setup.md](./02-acr-setup.md)** - Set up Azure Container Registry with Azure AD authentication

---

## Additional Resources

- [Azure AD Service Principals Documentation](https://docs.microsoft.com/azure/active-directory/develop/app-objects-and-service-principals)
- [Azure RBAC Documentation](https://docs.microsoft.com/azure/role-based-access-control/overview)
- [Managed Identities Documentation](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
- [Azure AD App Registrations](https://docs.microsoft.com/azure/active-directory/develop/quickstart-register-app)
