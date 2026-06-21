# 📸 Screenshot Guide for Azure Infrastructure Documentation

## Overview

This guide tells you **exactly which screenshots to take** as you follow the setup documentation. Each screenshot is numbered and corresponds to the 📸 markers in the main documentation files.

## 📋 How to Use This Guide

1. Open Azure Portal: https://portal.azure.com
2. Follow the documentation step-by-step
3. When you see a 📸 marker, take a screenshot according to this guide
4. Save screenshots with the provided filenames
5. Add screenshots to the `/screenshots` folder in the repository

## 📁 Folder Structure

Create this structure in your repository:

```
azure-setup/
├── README.md
├── docs/
│   ├── 01-azure-ad-setup.md
│   ├── 02-acr-setup.md
│   ├── ... (other docs)
└── screenshots/
    ├── 01-azure-ad/
    │   ├── screenshot-01-search-azure-ad.png
    │   ├── screenshot-02-ad-overview.png
    │   └── ...
    ├── 02-acr/
    ├── 03-aks/
    ├── 04-jump-server/
    └── 05-blob-storage/
```

## 🔧 Screenshot Settings

**Recommended Settings:**
- **Format**: PNG (for clarity)
- **Resolution**: Full HD (1920x1080) or higher
- **Browser zoom**: 100%
- **Hide sensitive data**: Blur subscription IDs, tenant IDs, personal emails
- **Highlight important elements**: Use red boxes or arrows (optional)

---

## 📸 01 - Azure AD Setup Screenshots

### Navigate to Azure Portal
**Filename**: `screenshots/01-azure-ad/screenshot-01-portal-home.png`


**What to capture**:
- Azure Portal home page
- Top search bar visible
- Your account name in top-right corner

**Instructions**:
1. Login to Azure Portal
2. Take screenshot of the main dashboard

---

### Search for Azure Active Directory
**Filename**: `screenshots/01-azure-ad/screenshot-02-search-azure-ad.png`

**What to capture**:
- Search bar with "Azure Active Directory" typed
- Dropdown showing search results
- "Azure Active Directory" option highlighted

**Instructions**:
1. Click the search bar at top
2. Type "Azure Active Directory"
3. Take screenshot showing the dropdown

---

### Azure AD Overview Page
**Filename**: `screenshots/01-azure-ad/screenshot-03-ad-overview.png`

**What to capture**:
- Azure Active Directory Overview page
- Tenant name visible
- Tenant ID visible
- Primary domain visible
- Left navigation menu visible

**Instructions**:
1. Click on "Azure Active Directory" from search results
2. You should see the Overview page
3. Take full-page screenshot

---

### App Registrations Page
**Filename**: `screenshots/01-azure-ad/screenshot-04-app-registrations.png`

**What to capture**:
- App registrations page
- "+ New registration" button visible
- List of existing app registrations (if any)

**Instructions**:
1. In Azure AD, click "App registrations" in left menu
2. Take screenshot of the page

---

### New App Registration Form
**Filename**: `screenshots/01-azure-ad/screenshot-05-new-app-registration.png`

**What to capture**:
- "Register an application" form
- Name field: "sp-aks-cluster"
- Supported account types: "Single tenant" selected
- Register button visible

**Instructions**:
1. Click "+ New registration"
2. Fill in the form as shown in documentation
3. Take screenshot BEFORE clicking Register

---

### App Registration Overview (After Creation)
**Filename**: `screenshots/01-azure-ad/screenshot-06-app-created-overview.png`


**What to capture**:
- App registration overview page
- Application (client) ID visible (blur for security)
- Directory (tenant) ID visible (blur for security)
- Display name: "sp-aks-cluster"

**Instructions**:
1. After creating the app registration
2. You'll see the overview page automatically
3. Take screenshot

---

### Certificates & Secrets Page
**Filename**: `screenshots/01-azure-ad/screenshot-07-certificates-secrets.png`

**What to capture**:
- "Certificates & secrets" page
- "+ New client secret" button visible
- Empty client secrets section (or existing secrets)

**Instructions**:
1. Click "Certificates & secrets" in left menu
2. Take screenshot of the page

---

### Add Client Secret Dialog
**Filename**: `screenshots/01-azure-ad/screenshot-08-add-client-secret.png`

**What to capture**:
- "Add a client secret" side panel
- Description field: "aks-cluster-secret"
- Expires dropdown showing options
- Add button

**Instructions**:
1. Click "+ New client secret"
2. Fill in description
3. Take screenshot BEFORE clicking Add

---

### Client Secret Created
**Filename**: `screenshots/01-azure-ad/screenshot-09-client-secret-value.png`

**What to capture**:
- Client secret listed with description
- Value field showing secret (⚠️ BLUR THIS)
- Expires date visible
- Warning message about copying value

**Instructions**:
1. After clicking Add
2. Secret is created and shown
3. Take screenshot (remember to blur the secret value)
4. Copy the secret immediately before it's hidden

---

### Access Control (IAM) Page
**Filename**: `screenshots/01-azure-ad/screenshot-10-iam-page.png`

**What to capture**:
- Access control (IAM) page
- "+ Add" button and dropdown
- "Add role assignment" option visible
- Existing role assignments list

**Instructions**:
1. Navigate to a resource (storage account, ACR, etc.)
2. Click "Access Control (IAM)"
3. Take screenshot

---

### Add Role Assignment - Role Tab
**Filename**: `screenshots/01-azure-ad/screenshot-11-add-role-assignment-role.png`


**What to capture**:
- Add role assignment wizard - Role tab
- Search box with role name
- Role selected (e.g., "Storage Blob Data Contributor")
- Role description visible
- Next button

**Instructions**:
1. Click "+ Add" > "Add role assignment"
2. Search for the role
3. Select it
4. Take screenshot

---

### Add Role Assignment - Members Tab
**Filename**: `screenshots/01-azure-ad/screenshot-12-add-role-assignment-members.png`

**What to capture**:
- Add role assignment wizard - Members tab
- "Assign access to" options
- "+ Select members" button
- Selected members shown
- Next button

**Instructions**:
1. Click Next to Members tab
2. Select members
3. Take screenshot showing selected members

---

### Review + Assign
**Filename**: `screenshots/01-azure-ad/screenshot-13-review-assign.png`

**What to capture**:
- Review + assign tab
- Role name
- Members list
- Scope information
- "Review + assign" button

**Instructions**:
1. Click Next to Review + assign
2. Take screenshot before clicking final assign button

---

### Managed Identities Page
**Filename**: `screenshots/01-azure-ad/screenshot-14-managed-identities.png`

**What to capture**:
- Managed Identities service page
- "+ Create" button visible
- List of existing managed identities (if any)

**Instructions**:
1. Search for "Managed Identities"
2. Click on the service
3. Take screenshot

---

### Create Managed Identity Form
**Filename**: `screenshots/01-azure-ad/screenshot-15-create-managed-identity.png`

**What to capture**:
- Create User Assigned Managed Identity form
- Subscription selected
- Resource group selected
- Region selected
- Name: "id-aks-managed-identity"
- Review + create button

**Instructions**:
1. Click "+ Create"
2. Fill in the form
3. Take screenshot before clicking Review + create

---

## 📸 02 - Azure Container Registry Screenshots

### Container Registries Page
**Filename**: `screenshots/02-acr/screenshot-01-acr-list.png`


**What to capture**:
- Container registries list page
- "+ Create" button visible
- Any existing registries (if any)

**Instructions**:
1. Search for "Container registries"
2. Click on the service
3. Take screenshot

---

### Create Container Registry - Basics
**Filename**: `screenshots/02-acr/screenshot-02-create-acr-basics.png`

**What to capture**:
- Create container registry form - Basics tab
- Resource group: rg-aks-demo
- Registry name: acrdemoacr (or your unique name)
- Location: East US
- SKU: Standard
- Next button

**Instructions**:
1. Click "+ Create"
2. Fill in basics tab
3. Take screenshot

---

### Create Container Registry - Networking
**Filename**: `screenshots/02-acr/screenshot-03-create-acr-networking.png`

**What to capture**:
- Networking tab
- Public access selected
- Network connectivity options visible

**Instructions**:
1. Click Next to Networking
2. Take screenshot showing network options

---

### Create Container Registry - Review
**Filename**: `screenshots/02-acr/screenshot-04-create-acr-review.png`

**What to capture**:
- Review + create tab
- Validation passed checkmark
- All settings summary
- Create button

**Instructions**:
1. Click Next through to Review + create
2. Take screenshot

---

### ACR Deployment Complete
**Filename**: `screenshots/02-acr/screenshot-05-acr-deployment-complete.png`

**What to capture**:
- Deployment complete page
- Deployment name
- Resource group
- "Go to resource" button

**Instructions**:
1. Wait for deployment
2. Take screenshot when complete

---

### ACR Overview Page
**Filename**: `screenshots/02-acr/screenshot-06-acr-overview.png`

**What to capture**:
- ACR Overview page
- Registry name
- Login server (e.g., acrdemoacr.azurecr.io)
- Location, SKU
- Status: Available

**Instructions**:
1. Click "Go to resource" or navigate to ACR
2. Take screenshot of overview

---

### ACR Access Keys
**Filename**: `screenshots/02-acr/screenshot-07-acr-access-keys.png`

**What to capture**:
- Access keys page
- Admin user toggle (enabled/disabled)
- Username visible
- Password fields (⚠️ BLUR passwords)

**Instructions**:
1. Click "Access keys" in left menu
2. Take screenshot (blur password values)

---

### ACR Repositories Page
**Filename**: `screenshots/02-acr/screenshot-08-acr-repositories.png`

**What to capture**:
- Repositories page
- List of repositories (after pushing images)
- Repository names
- Last updated times

**Instructions**:
1. Click "Repositories" in left menu
2. Take screenshot after pushing images

---

### ACR Repository Details
**Filename**: `screenshots/02-acr/screenshot-09-acr-repository-details.png`


**What to capture**:
- Repository detail page (e.g., samples/sample-app)
- Tags list
- Digest, size information

**Instructions**:
1. Click on a repository name
2. Take screenshot showing tags

---

## 📸 03 - Azure Kubernetes Service Screenshots

### Kubernetes Services Page
**Filename**: `screenshots/03-aks/screenshot-01-aks-list.png`

**What to capture**:
- Kubernetes services list page
- "+ Create" button with dropdown
- Any existing clusters

**Instructions**:
1. Search for "Kubernetes services"
2. Take screenshot of the list

---

### Create AKS - Basics Tab
**Filename**: `screenshots/03-aks/screenshot-02-create-aks-basics.png`

**What to capture**:
- Create Kubernetes cluster form - Basics tab
- Cluster preset: Dev/Test
- Cluster name: aks-demo-cluster
- Region: East US
- Kubernetes version
- Node pool configuration

**Instructions**:
1. Click "+ Create"
2. Fill in basics tab
3. Take screenshot

---

### Create AKS - Node Pools Tab
**Filename**: `screenshots/03-aks/screenshot-03-create-aks-node-pools.png`

**What to capture**:
- Node pools tab
- Default node pool details
- "+ Add node pool" button

**Instructions**:
1. Click Next to Node pools
2. Take screenshot

---

### Create AKS - Networking Tab
**Filename**: `screenshots/03-aks/screenshot-04-create-aks-networking.png`

**What to capture**:
- Networking tab
- Network configuration: Azure CNI selected
- Virtual network settings
- DNS name prefix

**Instructions**:
1. Click Next to Networking
2. Take screenshot

---

### Create AKS - Integrations Tab
**Filename**: `screenshots/03-aks/screenshot-05-create-aks-integrations.png`

**What to capture**:
- Integrations tab
- Container registry selected (acrdemoacr)
- Azure Monitor settings
- "Enabled" for Container insights

**Instructions**:
1. Click Next to Integrations
2. Take screenshot showing ACR integration

---

### Create AKS - Review + Create
**Filename**: `screenshots/03-aks/screenshot-06-create-aks-review.png`

**What to capture**:
- Review + create tab
- Validation passed
- Summary of all settings
- Create button

**Instructions**:
1. Click Next to Review + create
2. Take screenshot

---

### AKS Deployment Progress
**Filename**: `screenshots/03-aks/screenshot-07-aks-deployment-progress.png`

**What to capture**:
- Deployment in progress page
- Deployment name
- Progress indicator

**Instructions**:
1. After clicking Create
2. Take screenshot of deployment progress

---

### AKS Overview Page
**Filename**: `screenshots/03-aks/screenshot-08-aks-overview.png`


**What to capture**:
- AKS cluster overview
- Status: Succeeded
- Kubernetes version
- Node count
- FQDN
- Location

**Instructions**:
1. Navigate to AKS cluster
2. Take screenshot of overview page

---

### AKS Node Pools
**Filename**: `screenshots/03-aks/screenshot-09-aks-node-pools.png`

**What to capture**:
- Node pools page
- Node pool list with details
- Status, mode, count, VM size

**Instructions**:
1. Click "Node pools" in left menu
2. Take screenshot

---

### AKS Workloads
**Filename**: `screenshots/03-aks/screenshot-10-aks-workloads.png`

**What to capture**:
- Workloads page
- Deployed applications
- Namespace selector
- Pods status

**Instructions**:
1. Click "Workloads" under Kubernetes resources
2. Take screenshot after deploying applications

---

### AKS Services
**Filename**: `screenshots/03-aks/screenshot-11-aks-services.png`

**What to capture**:
- Services page
- Service list with type, cluster IP, external IP
- LoadBalancer service showing external IP

**Instructions**:
1. Click "Services and ingresses"
2. Take screenshot

---

### AKS Insights
**Filename**: `screenshots/03-aks/screenshot-12-aks-insights.png`

**What to capture**:
- Container Insights page
- Cluster performance metrics
- CPU/Memory graphs
- Node status

**Instructions**:
1. Click "Insights" under Monitoring
2. Take screenshot showing metrics

---

## 📸 04 - Jump Server VM Screenshots

### Virtual Machines Page
**Filename**: `screenshots/04-jump-server/screenshot-01-vm-list.png`

**What to capture**:
- Virtual machines list
- "+ Create" button with dropdown
- Azure virtual machine option

**Instructions**:
1. Search for "Virtual machines"
2. Take screenshot

---

### Create VM - Basics Tab
**Filename**: `screenshots/04-jump-server/screenshot-02-create-vm-basics.png`

**What to capture**:
- Create virtual machine form - Basics tab
- VM name: vm-jumpserver
- Region: East US
- Image: Ubuntu Server 22.04 LTS
- Size: Standard_B2s
- Authentication type: SSH public key

**Instructions**:
1. Click "+ Create" > "Azure virtual machine"
2. Fill in basics tab
3. Take screenshot

---

### Create VM - Disks Tab
**Filename**: `screenshots/04-jump-server/screenshot-03-create-vm-disks.png`

**What to capture**:
- Disks tab
- OS disk type: Standard SSD
- OS disk size

**Instructions**:
1. Click Next to Disks
2. Take screenshot

---

### Create VM - Networking Tab
**Filename**: `screenshots/04-jump-server/screenshot-04-create-vm-networking.png`


**What to capture**:
- Networking tab
- Virtual network selected
- Subnet selected
- Public IP: new
- NIC network security group: Advanced
- NSG selected

**Instructions**:
1. Click Next to Networking
2. Take screenshot

---

### Create VM - Management Tab
**Filename**: `screenshots/04-jump-server/screenshot-05-create-vm-management.png`

**What to capture**:
- Management tab
- Boot diagnostics: Enabled
- Auto-shutdown: Enabled (if configured)

**Instructions**:
1. Click Next to Management
2. Take screenshot

---

### Create VM - Review
**Filename**: `screenshots/04-jump-server/screenshot-06-create-vm-review.png`

**What to capture**:
- Review + create tab
- Validation passed
- Summary of settings
- Create button

**Instructions**:
1. Click Next to Review + create
2. Take screenshot

---

### Generate SSH Key Pair Dialog
**Filename**: `screenshots/04-jump-server/screenshot-07-ssh-key-download.png`

**What to capture**:
- Generate new key pair dialog
- Key pair name
- Download private key button

**Instructions**:
1. After clicking Create
2. Download popup appears
3. Take screenshot (before downloading)

---

### VM Overview Page
**Filename**: `screenshots/04-jump-server/screenshot-08-vm-overview.png`

**What to capture**:
- VM overview page
- Status: Running
- Public IP address visible
- Private IP address visible
- Size, OS information

**Instructions**:
1. After deployment, navigate to VM
2. Take screenshot of overview

---

### Network Security Group Rules
**Filename**: `screenshots/04-jump-server/screenshot-09-nsg-rules.png`

**What to capture**:
- NSG inbound security rules
- SSH rule (port 22)
- Priority, source, destination
- Action: Allow

**Instructions**:
1. Navigate to NSG resource
2. Click "Inbound security rules"
3. Take screenshot

---

## 📸 05 - Blob Storage Screenshots

### Storage Accounts Page
**Filename**: `screenshots/05-blob-storage/screenshot-01-storage-list.png`

**What to capture**:
- Storage accounts list
- "+ Create" button
- Existing storage accounts (if any)

**Instructions**:
1. Search for "Storage accounts"
2. Take screenshot

---

### Create Storage Account - Basics
**Filename**: `screenshots/05-blob-storage/screenshot-02-create-storage-basics.png`

**What to capture**:
- Create storage account form - Basics tab
- Storage account name: staksdemostorage
- Region: East US
- Performance: Standard
- Redundancy: LRS

**Instructions**:
1. Click "+ Create"
2. Fill in basics
3. Take screenshot

---

### Create Storage Account - Advanced
**Filename**: `screenshots/05-blob-storage/screenshot-03-create-storage-advanced.png`


**What to capture**:
- Advanced tab
- Require secure transfer: Enabled
- Enable storage account key access: Checked
- Default to Azure AD authorization: Checked
- Minimum TLS version: 1.2
- Access tier: Hot

**Instructions**:
1. Click Next to Advanced
2. Take screenshot

---

### Create Storage Account - Data Protection
**Filename**: `screenshots/05-blob-storage/screenshot-04-create-storage-data-protection.png`

**What to capture**:
- Data protection tab
- Enable soft delete for blobs: Checked
- Retention days: 7
- Enable versioning: Checked
- Enable soft delete for containers: Checked

**Instructions**:
1. Click Next to Data protection
2. Take screenshot

---

### Create Storage Account - Review
**Filename**: `screenshots/05-blob-storage/screenshot-05-create-storage-review.png`

**What to capture**:
- Review + create tab
- Validation passed
- Summary of settings
- Create button

**Instructions**:
1. Click Next to Review + create
2. Take screenshot

---

### Storage Account Overview
**Filename**: `screenshots/05-blob-storage/screenshot-06-storage-overview.png`

**What to capture**:
- Storage account overview
- Status: Available
- Primary endpoints (Blob, File, etc.)
- Performance, replication info

**Instructions**:
1. Navigate to storage account
2. Take screenshot of overview

---

### Containers Page
**Filename**: `screenshots/05-blob-storage/screenshot-07-containers-list.png`

**What to capture**:
- Containers page
- "+ Container" button
- List of containers (after creation)
- Public access level for each

**Instructions**:
1. Click "Containers" under Data storage
2. Take screenshot

---

### New Container Dialog
**Filename**: `screenshots/05-blob-storage/screenshot-08-new-container.png`

**What to capture**:
- New container side panel
- Name: demo-container
- Public access level: Private
- Create button

**Instructions**:
1. Click "+ Container"
2. Fill in name
3. Take screenshot

---

### Container Details
**Filename**: `screenshots/05-blob-storage/screenshot-09-container-details.png`

**What to capture**:
- Container detail page
- Upload button
- List of blobs (after upload)
- Using Azure AD authorization indicator

**Instructions**:
1. Click on container name
2. Take screenshot

---

### Upload Blob Dialog
**Filename**: `screenshots/05-blob-storage/screenshot-10-upload-blob.png`

**What to capture**:
- Upload blob side panel
- File browser
- Advanced options
- Upload button

**Instructions**:
1. Click "Upload" button
2. Take screenshot of upload dialog

---

### Storage Access Control (IAM)
**Filename**: `screenshots/05-blob-storage/screenshot-11-storage-iam.png`

**What to capture**:
- Storage account IAM page
- Role assignments list
- Storage Blob Data roles assigned

**Instructions**:
1. Click "Access Control (IAM)"
2. Click "Role assignments" tab
3. Take screenshot

---

### Storage Configuration
**Filename**: `screenshots/05-blob-storage/screenshot-12-storage-configuration.png`


**What to capture**:
- Configuration page
- Secure transfer required: Enabled
- Allow storage account key access toggle
- Default to Azure AD authorization: Enabled
- Minimum TLS version

**Instructions**:
1. Click "Configuration" under Settings
2. Take screenshot

---

### Storage Networking
**Filename**: `screenshots/05-blob-storage/screenshot-13-storage-networking.png`

**What to capture**:
- Networking page
- Firewalls and virtual networks settings
- Public network access options
- Selected networks/IP ranges (if configured)

**Instructions**:
1. Click "Networking" under Security + networking
2. Take screenshot

---

## 🔧 After Taking Screenshots

### 1. Organize Screenshots

```bash
# Create folder structure
mkdir -p screenshots/{01-azure-ad,02-acr,03-aks,04-jump-server,05-blob-storage}

# Move screenshots to appropriate folders
```

### 2. Edit Screenshots (Optional but Recommended)

Use tools like:
- **Windows**: Paint, Snipping Tool, Snagit
- **Mac**: Preview, Skitch
- **Linux**: GIMP, Ksnip
- **Online**: Photopea, Figma

**Edit to:**
- Blur sensitive information (subscription IDs, tenant IDs, emails)
- Add red boxes or arrows to highlight important elements
- Add step numbers in corners
- Crop to relevant area

### 3. Optimize File Size

```bash
# Using ImageMagick (if installed)
mogrify -resize 1920x1080 -quality 85 screenshots/**/*.png

# Or use online tools:
# - TinyPNG.com
# - Compressor.io
```

### 4. Update Markdown Files

In each documentation file, replace the text markers with image links:

**Before:**
```markdown
📸 **Screenshot: Azure Portal > Search "Azure Active Directory"**
```

**After:**
```markdown
![Search Azure Active Directory](../screenshots/01-azure-ad/screenshot-02-search-azure-ad.png)
```

### 5. Commit and Push to GitHub

```bash
cd azure-setup
git add screenshots/
git commit -m "Add Azure Portal screenshots for all setup guides"
git push origin main
```

---

## 📊 Screenshot Summary

### Total Screenshots Needed:

| Documentation File | Screenshot Count | Folder |
|-------------------|------------------|--------|
| 01-azure-ad-setup.md | ~15 screenshots | `01-azure-ad/` |
| 02-acr-setup.md | ~9 screenshots | `02-acr/` |
| 03-aks-setup.md | ~12 screenshots | `03-aks/` |
| 04-jump-server-setup.md | ~9 screenshots | `04-jump-server/` |
| 05-blob-storage-setup.md | ~13 screenshots | `05-blob-storage/` |
| **Total** | **~58 screenshots** | |

---

## 💡 Tips for Better Screenshots

1. **Use Full HD Resolution**: 1920x1080 or higher
2. **Clean Browser**: Close unnecessary tabs, bookmarks
3. **Consistent Zoom**: Keep browser at 100% zoom
4. **Hide Personal Info**: Blur emails, subscription IDs
5. **Highlight Important Elements**: Use red boxes/arrows
6. **Take Extras**: Better to have more than needed
7. **Name Clearly**: Use descriptive filenames
8. **Check Quality**: Ensure text is readable
9. **Compress Images**: Use PNG with reasonable compression
10. **Test Links**: Verify all image paths work in markdown

---

## 🚀 Quick Start

**Fastest way to add screenshots:**

1. Follow each documentation guide
2. When you see 📸, press `Win + Shift + S` (Windows) or `Cmd + Shift + 4` (Mac)
3. Save with the filename from this guide
4. After finishing a section, organize screenshots into folders
5. Update markdown files with image links
6. Push to GitHub

---

## ❓ Need Help?

If you need assistance with:
- Taking screenshots
- Editing images
- Updating markdown files
- Organizing the repository

Just let me know! 🎉
