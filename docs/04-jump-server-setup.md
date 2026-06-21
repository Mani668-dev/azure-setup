# Jump Server (VM) Setup Guide

## 📋 Overview

A Jump Server (also known as a bastion host or jump box) provides secure access to your AKS cluster from outside the Azure network. This guide covers creating a Jump Server VM, configuring network security, installing required tools, and establishing secure access to AKS.

## Prerequisites

- Completed [Azure AD Setup](./01-azure-ad-setup.md)
- Completed [AKS Setup](./03-aks-setup.md)
- Azure CLI installed
- SSH client installed locally
- AKS cluster running

## Table of Contents

1. [Create Virtual Network (if needed)](#1-create-virtual-network-if-needed)
2. [Create Network Security Group](#2-create-network-security-group)
3. [Create Jump Server VM](#3-create-jump-server-vm)
4. [Configure Network Security Rules](#4-configure-network-security-rules)
5. [Connect to Jump Server](#5-connect-to-jump-server)
6. [Install Required Tools](#6-install-required-tools)
7. [Configure kubectl Access to AKS](#7-configure-kubectl-access-to-aks)
8. [Set Up VNet Peering (Optional)](#8-set-up-vnet-peering-optional)
9. [Security Hardening](#9-security-hardening)
10. [Verification](#10-verification)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Create Virtual Network (if needed)

### Step 1.1: Check Existing Virtual Networks

**Via Azure Portal:**

1. Navigate to [Azure Portal](https://portal.azure.com)
2. Search for **"Virtual networks"**
3. Click **"Virtual networks"**
4. Check if you have an existing VNet in your resource group

📸 **Screenshot: Virtual networks page**
```
Virtual networks
+ Create

Name                | Resource group | Location | Subscription
aks-vnet-12345678  | MC_rg-aks...  | East US  | Your subscription
```

**Via Azure CLI:**

```bash
# Set variables
RESOURCE_GROUP="rg-aks-demo"
LOCATION="eastus"
VNET_NAME="vnet-jumpserver"
SUBNET_NAME="subnet-jumpserver"

# Check existing VNets
az network vnet list --resource-group $RESOURCE_GROUP --output table

# If you want to use the AKS VNet, get its name
AKS_CLUSTER_NAME="aks-demo-cluster"
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query nodeResourceGroup -o tsv)
AKS_VNET=$(az network vnet list --resource-group $NODE_RG --query "[0].name" -o tsv)

echo "AKS VNet: $AKS_VNET"
```


### Step 1.2: Create New Virtual Network for Jump Server

**Via Azure Portal:**

1. Search for **"Virtual networks"**
2. Click **"+ Create"**
3. Fill in the **Basics** tab:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Name**: `vnet-jumpserver`
   - **Region**: `East US` (same as AKS)

📸 **Screenshot: Create virtual network - Basics**
```
Basics tab
Subscription: [Your subscription]
Resource group: rg-aks-demo
Virtual network name: vnet-jumpserver
Region: East US
[Next: IP Addresses >]
```

4. Click **"Next: IP Addresses >"**
5. In the **IP Addresses** tab:
   - **IPv4 address space**: `10.1.0.0/16`
   - Click **"+ Add subnet"**
   - **Subnet name**: `subnet-jumpserver`
   - **Subnet address range**: `10.1.1.0/24`
   - Click **"Add"**

📸 **Screenshot: Create virtual network - IP Addresses**
```
IP Addresses tab
IPv4 address space: 10.1.0.0/16

Subnets
+ Add subnet | + Add gateway subnet

Name               | Address range
subnet-jumpserver  | 10.1.1.0/24

[Next: Security >]
```

6. Click **"Next: Security >"**
7. Leave security defaults (can be configured later)
8. Click **"Review + create"**
9. Click **"Create"**

📸 **Screenshot: Review + create**
```
Review + create tab
✓ Validation passed

Basics
Virtual network name: vnet-jumpserver
Region: East US

IP Addresses
IPv4 address space: 10.1.0.0/16
Subnets: subnet-jumpserver (10.1.1.0/24)

[Create button]
```

**Via Azure CLI:**

```bash
# Create Virtual Network
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --address-prefix 10.1.0.0/16 \
  --subnet-name $SUBNET_NAME \
  --subnet-prefix 10.1.1.0/24 \
  --location $LOCATION

echo "Virtual Network created successfully"

# Verify
az network vnet show \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --query "{Name:name, AddressSpace:addressSpace.addressPrefixes, Location:location}" \
  --output table
```


---

## 2. Create Network Security Group

Network Security Groups (NSG) control inbound and outbound traffic to your VM.

### Step 2.1: Create NSG via Azure Portal

1. Search for **"Network security groups"**
2. Click **"+ Create"**
3. Fill in the details:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Name**: `nsg-jumpserver`
   - **Region**: `East US`

📸 **Screenshot: Create network security group**
```
Basics tab
Subscription: [Your subscription]
Resource group: rg-aks-demo
Name: nsg-jumpserver
Region: East US
[Review + create button]
```

4. Click **"Review + create"**
5. Click **"Create"**

### Step 2.2: Create NSG via Azure CLI

```bash
# Set variables
NSG_NAME="nsg-jumpserver"

# Create NSG
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name $NSG_NAME \
  --location $LOCATION

echo "NSG created successfully"
```

### Step 2.3: Add SSH Rule to NSG

**Via Azure Portal:**

1. Go to your NSG (`nsg-jumpserver`)
2. Click **"Inbound security rules"**
3. Click **"+ Add"**
4. Fill in:
   - **Source**: `IP Addresses`
   - **Source IP addresses/CIDR ranges**: Your public IP (get from https://whatismyip.com)
   - **Source port ranges**: `*`
   - **Destination**: `Any`
   - **Service**: `SSH`
   - **Destination port ranges**: `22` (auto-filled)
   - **Protocol**: `TCP` (auto-filled)
   - **Action**: `Allow`
   - **Priority**: `100`
   - **Name**: `Allow-SSH-Inbound`

📸 **Screenshot: Add inbound security rule**
```
Add inbound security rule

Source: IP Addresses ▼
Source IP addresses/CIDR ranges: 123.45.67.89/32
Source port ranges: *

Destination: Any ▼
Service: SSH ▼
Destination port ranges: 22
Protocol: TCP

Action: • Allow  ○ Deny
Priority: 100
Name: Allow-SSH-Inbound

[Add button]
```

5. Click **"Add"**

**Via Azure CLI:**

```bash
# Get your public IP
MY_IP=$(curl -s ifconfig.me)
echo "Your public IP: $MY_IP"

# Add SSH rule
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-Inbound \
  --priority 100 \
  --source-address-prefixes "${MY_IP}/32" \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp \
  --description "Allow SSH from my IP"

echo "SSH rule added to NSG"
```


---

## 3. Create Jump Server VM

### Step 3.1: Create VM via Azure Portal

1. Search for **"Virtual machines"**
2. Click **"+ Create"** > **"Azure virtual machine"**

📸 **Screenshot: Virtual machines page**
```
Virtual machines > + Create > Azure virtual machine
```

#### Basics Tab

3. Fill in the **Basics** tab:
   - **Subscription**: Your subscription
   - **Resource group**: `rg-aks-demo`
   - **Virtual machine name**: `vm-jumpserver`
   - **Region**: `East US`
   - **Availability options**: `No infrastructure redundancy required`
   - **Security type**: `Standard`
   - **Image**: `Ubuntu Server 22.04 LTS - x64 Gen2`
   - **VM architecture**: `x64`
   - **Size**: `Standard_B2s` (2 vCPUs, 4 GiB memory) - Cost-effective for jump server

📸 **Screenshot: Create a virtual machine - Basics (Part 1)**
```
Basics tab - Project details
Subscription: [Your subscription]
Resource group: rg-aks-demo

Instance details
Virtual machine name: vm-jumpserver
Region: East US
Availability options: No infrastructure redundancy required ▼
Security type: Standard ▼
Image: Ubuntu Server 22.04 LTS - x64 Gen2 ▼
VM architecture: x64 ▼
Size: Standard_B2s - 2 vcpus, 4 GiB memory  [See all sizes]
```

4. Under **Administrator account**:
   - **Authentication type**: Select `SSH public key`
   - **Username**: `azureuser`
   - **SSH public key source**: `Generate new key pair`
   - **Key pair name**: `vm-jumpserver_key`

📸 **Screenshot: Create a virtual machine - Basics (Part 2)**
```
Administrator account
Authentication type: • SSH public key  ○ Password
Username: azureuser
SSH public key source: Generate new key pair ▼
Key pair name: vm-jumpserver_key
```

5. Under **Inbound port rules**:
   - **Public inbound ports**: Select `Allow selected ports`
   - **Select inbound ports**: Select `SSH (22)`

📸 **Screenshot: Inbound port rules**
```
Inbound port rules
Public inbound ports: • Allow selected ports
Select inbound ports: SSH (22) ☑
```

6. Click **"Next: Disks >"**


#### Disks Tab

7. In the **Disks** tab:
   - **OS disk type**: `Standard SSD (locally-redundant storage)`
   - Leave other settings as default

📸 **Screenshot: Disks tab**
```
Disks tab
OS disk
OS disk size: Image default (30 GiB)
OS disk type: Standard SSD (locally-redundant storage) ▼
Delete with VM: ☑
Encryption type: (Default) Encryption at-rest with a platform-managed key
[Next: Networking >]
```

8. Click **"Next: Networking >"**

#### Networking Tab

9. In the **Networking** tab:
   - **Virtual network**: Select `vnet-jumpserver`
   - **Subnet**: Select `subnet-jumpserver`
   - **Public IP**: `(new) vm-jumpserver-ip`
   - **NIC network security group**: Select `Advanced`
   - **Configure network security group**: Select `nsg-jumpserver`
   - **Delete public IP and NIC when VM is deleted**: Check this box

📸 **Screenshot: Networking tab**
```
Networking tab
Network interface
Virtual network: vnet-jumpserver ▼
Subnet: subnet-jumpserver (10.1.1.0/24) ▼
Public IP: (new) vm-jumpserver-ip ▼
NIC network security group: Advanced ▼
Configure network security group: nsg-jumpserver ▼

☑ Delete public IP and NIC when VM is deleted
☐ Enable accelerated networking

Load balancing
☐ Place this virtual machine behind an existing load balancing solution

[Next: Management >]
```

10. Click **"Next: Management >"**

#### Management Tab

11. In the **Management** tab:
    - **Enable auto-shutdown**: Check this box (optional, for cost savings)
    - **Shutdown time**: Set to 7:00 PM or your preferred time
    - Leave other defaults

📸 **Screenshot: Management tab**
```
Management tab
Azure Security Center
☐ Enable basic plan for free (Deprecated)

Monitoring
Boot diagnostics: Enable with managed storage account (recommended) ▼

Identity
System assigned managed identity: Off ▼

Auto-shutdown
☑ Enable auto-shutdown
Shutdown time: 19:00:00 (7:00 PM) | Time zone: (UTC) Coordinated Universal Time
[Next: Advanced >]
```

12. Click **"Next: Advanced >"**
13. Leave Advanced settings as default
14. Click **"Next: Tags >"**


#### Tags Tab

15. Add tags (optional):
    - **Name**: `Purpose`, **Value**: `Jump-Server`
    - **Name**: `Environment`, **Value**: `Development`

📸 **Screenshot: Tags tab**
```
Tags tab
Name        | Value
Purpose     | Jump-Server
Environment | Development

[Next: Review + create >]
```

16. Click **"Review + create"**
17. Review settings
18. Click **"Create"**

📸 **Screenshot: Review + create**
```
Review + create tab
✓ Validation passed

Basics
Virtual machine name: vm-jumpserver
Region: East US
Image: Ubuntu Server 22.04 LTS
Size: Standard_B2s
Authentication type: SSH public key
Username: azureuser

Networking
Virtual network: vnet-jumpserver
Subnet: subnet-jumpserver
Public IP: vm-jumpserver-ip
NSG: nsg-jumpserver

[Create button]
```

19. **Download private key** popup will appear
20. Click **"Download private key and create resource"**
21. Save the `.pem` file securely (e.g., `vm-jumpserver_key.pem`)

📸 **Screenshot: Generate new key pair**
```
Generate new key pair
Download private key: vm-jumpserver_key.pem
[Download private key and create resource button]
```

22. Wait for deployment (2-3 minutes)
23. Click **"Go to resource"**

📸 **Screenshot: Deployment complete**
```
Your deployment is complete
Deployment name: vm-jumpserver
Resource group: rg-aks-demo
[Go to resource button]
```

### Step 3.2: Create VM via Azure CLI

```bash
# Set variables
VM_NAME="vm-jumpserver"
VM_SIZE="Standard_B2s"
VM_IMAGE="Ubuntu2204"
ADMIN_USERNAME="azureuser"

# Generate SSH key if you don't have one
ssh-keygen -t rsa -b 4096 -f ~/.ssh/vm-jumpserver_key -N ""

# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --query id -o tsv)

# Create public IP
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name "${VM_NAME}-ip" \
  --sku Standard \
  --allocation-method Static

# Create VM
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --location $LOCATION \
  --size $VM_SIZE \
  --image $VM_IMAGE \
  --admin-username $ADMIN_USERNAME \
  --ssh-key-values ~/.ssh/vm-jumpserver_key.pub \
  --public-ip-address "${VM_NAME}-ip" \
  --subnet $SUBNET_ID \
  --nsg $NSG_NAME \
  --storage-sku StandardSSD_LRS

echo "VM created successfully"

# Get VM public IP
VM_PUBLIC_IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query publicIps -o tsv)

echo "VM Public IP: $VM_PUBLIC_IP"
echo "SSH Command: ssh -i ~/.ssh/vm-jumpserver_key $ADMIN_USERNAME@$VM_PUBLIC_IP"
```


---

## 4. Configure Network Security Rules

### Step 4.1: View Current NSG Rules

**Via Azure Portal:**

1. Go to your NSG (`nsg-jumpserver`)
2. Click **"Inbound security rules"**
3. Review existing rules

📸 **Screenshot: NSG Inbound security rules**
```
nsg-jumpserver | Inbound security rules
+ Add | Refresh

Priority | Name              | Port | Protocol | Source      | Destination | Action
100      | Allow-SSH-Inbound | 22   | TCP      | 123.45.../32| Any        | Allow
65000    | AllowVnetInBound  | Any  | Any      | VirtualNet..| VirtualNet..| Allow
65001    | AllowAzureLBInB.. | Any  | Any      | AzureLoadB..| Any        | Allow
65500    | DenyAllInBound    | Any  | Any      | Any         | Any        | Deny
```

**Via Azure CLI:**

```bash
# List NSG rules
az network nsg rule list \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --output table

# Show specific rule
az network nsg rule show \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-Inbound
```

### Step 4.2: Add Outbound Rules (if needed)

Allow outbound traffic to AKS API server and internet:

```bash
# Allow outbound HTTPS (for package downloads, Azure CLI, etc.)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-HTTPS-Outbound \
  --priority 100 \
  --direction Outbound \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes 'Internet' \
  --destination-port-ranges 443 \
  --access Allow \
  --protocol Tcp \
  --description "Allow HTTPS to internet"

# Allow outbound HTTP (optional)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-HTTP-Outbound \
  --priority 110 \
  --direction Outbound \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes 'Internet' \
  --destination-port-ranges 80 \
  --access Allow \
  --protocol Tcp \
  --description "Allow HTTP to internet"

echo "Outbound rules configured"
```

### Step 4.3: Restrict SSH Access to Specific IPs (Optional)

**Update SSH rule to allow only specific IPs:**

```bash
# Update existing rule with multiple IPs
az network nsg rule update \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-Inbound \
  --source-address-prefixes "123.45.67.89/32" "98.76.54.32/32"

# Or create separate rules for different locations
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-Office \
  --priority 101 \
  --source-address-prefixes "203.0.113.0/24" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp \
  --description "Allow SSH from office network"
```


---

## 5. Connect to Jump Server

### Step 5.1: Get VM Public IP

**Via Azure Portal:**

1. Go to your VM (`vm-jumpserver`)
2. On the **Overview** page, find the **Public IP address**

📸 **Screenshot: VM Overview with Public IP**
```
vm-jumpserver | Overview
Status: Running ⚫

Essentials
Resource group: rg-aks-demo
Location: East US
Subscription: Your subscription
Public IP address: 20.123.45.67
Private IP address: 10.1.1.4
Size: Standard_B2s
Operating system: Linux (ubuntu 22.04)
```

**Via Azure CLI:**

```bash
# Get public IP
VM_PUBLIC_IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query publicIps -o tsv)

echo "Jump Server Public IP: $VM_PUBLIC_IP"
```

### Step 5.2: Set SSH Key Permissions (Linux/macOS)

```bash
# Set correct permissions on private key
chmod 400 ~/Downloads/vm-jumpserver_key.pem

# Or if you generated it with Azure CLI
chmod 400 ~/.ssh/vm-jumpserver_key
```

### Step 5.3: Connect via SSH

**On Linux/macOS:**

```bash
# Connect to Jump Server
ssh -i ~/Downloads/vm-jumpserver_key.pem azureuser@$VM_PUBLIC_IP

# Or with generated key
ssh -i ~/.ssh/vm-jumpserver_key azureuser@$VM_PUBLIC_IP

# Expected output:
# Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.2.0-1018-azure x86_64)
# ...
# azureuser@vm-jumpserver:~$
```

**On Windows (using PowerShell):**

```powershell
# Using OpenSSH (built into Windows 10/11)
ssh -i C:\Users\YourUser\Downloads\vm-jumpserver_key.pem azureuser@<VM_PUBLIC_IP>

# Or use PuTTY (need to convert .pem to .ppk first with PuTTYgen)
```

### Step 5.4: Verify Connection

Once connected to the Jump Server:

```bash
# Check system information
uname -a

# Check Azure CLI is not installed yet
az version || echo "Azure CLI not installed"

# Check internet connectivity
ping -c 3 google.com

# Check you can reach Azure
ping -c 3 portal.azure.com
```

---

## 6. Install Required Tools

Once connected to the Jump Server, install necessary tools.

### Step 6.1: Update System Packages

```bash
# Update package lists
sudo apt-get update

# Upgrade installed packages (optional)
sudo apt-get upgrade -y

echo "System updated"
```


### Step 6.2: Install Azure CLI

```bash
# Install prerequisites
sudo apt-get install -y ca-certificates curl apt-transport-https lsb-release gnupg

# Download and install Microsoft signing key
curl -sL https://packages.microsoft.com/keys/microsoft.asc | \
    gpg --dearmor | \
    sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null

# Add Azure CLI repository
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | \
    sudo tee /etc/apt/sources.list.d/azure-cli.list

# Update package list
sudo apt-get update

# Install Azure CLI
sudo apt-get install -y azure-cli

# Verify installation
az version

# Expected output:
# {
#   "azure-cli": "2.57.0",
#   "azure-cli-core": "2.57.0",
#   ...
# }

echo "Azure CLI installed successfully"
```

### Step 6.3: Install kubectl

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Verify the binary (optional)
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client

# Expected output:
# Client Version: v1.29.0
# Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

echo "kubectl installed successfully"

# Clean up
rm kubectl kubectl.sha256
```

### Step 6.4: Install Helm (Optional)

```bash
# Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version

# Expected output:
# version.BuildInfo{Version:"v3.14.0", ...}

echo "Helm installed successfully"
```

### Step 6.5: Install Docker (Optional)

```bash
# Install Docker
sudo apt-get install -y docker.io

# Add current user to docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation (need to logout and login again for group changes)
sudo docker version

echo "Docker installed successfully"
echo "Note: Logout and login again to use Docker without sudo"
```

### Step 6.6: Install Additional Useful Tools

```bash
# Install git
sudo apt-get install -y git

# Install jq (JSON processor)
sudo apt-get install -y jq

# Install htop (system monitor)
sudo apt-get install -y htop

# Install vim
sudo apt-get install -y vim

# Verify installations
git --version
jq --version
htop --version
vim --version | head -1

echo "All tools installed successfully"
```


---

## 7. Configure kubectl Access to AKS

### Step 7.1: Login to Azure from Jump Server

```bash
# Login to Azure (this will open a browser-based login)
az login

# If the jump server doesn't have a browser, use device code flow
az login --use-device-code

# Follow the instructions:
# 1. Open https://microsoft.com/devicelogin in your browser
# 2. Enter the code displayed
# 3. Sign in with your Azure credentials

# Verify login
az account show

# Set the correct subscription if needed
az account set --subscription "Your-Subscription-Name"

# Verify
az account show --query "{Name:name, SubscriptionId:id}" -o table
```

### Step 7.2: Get AKS Credentials

```bash
# Set variables
RESOURCE_GROUP="rg-aks-demo"
AKS_CLUSTER_NAME="aks-demo-cluster"

# Get AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --overwrite-existing

# Expected output:
# Merged "aks-demo-cluster" as current context in /home/azureuser/.kube/config

echo "AKS credentials configured"
```

### Step 7.3: Verify kubectl Access

```bash
# Check cluster connection
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://aks-demo-cluster-xxx.hcp.eastus.azmk8s.io:443
# CoreDNS is running at https://aks-demo-cluster-xxx.hcp.eastus.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Get nodes
kubectl get nodes

# Expected output:
# NAME                                STATUS   ROLES   AGE   VERSION
# aks-agentpool-12345678-vmss000000   Ready    agent   1h    v1.28.5
# aks-agentpool-12345678-vmss000001   Ready    agent   1h    v1.28.5

# Get all namespaces
kubectl get namespaces

# Get pods in all namespaces
kubectl get pods --all-namespaces

# Get services
kubectl get services --all-namespaces

echo "✓ Successfully connected to AKS from Jump Server"
```

### Step 7.4: Test Deploying an Application

```bash
# Create a test deployment
kubectl create deployment nginx-test --image=nginx:latest

# Expose the deployment
kubectl expose deployment nginx-test --port=80 --type=ClusterIP

# Check deployment
kubectl get deployments
kubectl get pods
kubectl get services

# Test accessing the service
kubectl run curl-test --image=curlimages/curl -i --rm --restart=Never -- curl http://nginx-test

# Clean up test resources
kubectl delete deployment nginx-test
kubectl delete service nginx-test

echo "✓ Deployment test successful"
```

### Step 7.5: Configure Bash Completion (Optional)

```bash
# Enable kubectl bash completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

# Enable Azure CLI bash completion
echo 'source /etc/bash_completion.d/azure-cli' >> ~/.bashrc

# Reload bashrc
source ~/.bashrc

# Now you can use tab completion
# k get po<TAB>  → k get pods
# az aks <TAB>   → shows available commands
```


---

## 8. Set Up VNet Peering (Optional)

VNet peering connects your Jump Server VNet with the AKS VNet for direct network access.

### Step 8.1: Get VNet Information

```bash
# Get Jump Server VNet ID
JUMPSERVER_VNET_ID=$(az network vnet show \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --query id -o tsv)

# Get AKS VNet information
NODE_RG=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query nodeResourceGroup -o tsv)

AKS_VNET_NAME=$(az network vnet list \
  --resource-group $NODE_RG \
  --query "[0].name" -o tsv)

AKS_VNET_ID=$(az network vnet show \
  --resource-group $NODE_RG \
  --name $AKS_VNET_NAME \
  --query id -o tsv)

echo "Jump Server VNet ID: $JUMPSERVER_VNET_ID"
echo "AKS VNet ID: $AKS_VNET_ID"
```

### Step 8.2: Create VNet Peering

**Via Azure Portal:**

1. Go to your Jump Server VNet (`vnet-jumpserver`)
2. Click **"Peerings"** in the left menu
3. Click **"+ Add"**
4. Fill in:
   - **This virtual network - Peering link name**: `jumpserver-to-aks`
   - **Remote virtual network - Peering link name**: `aks-to-jumpserver`
   - **Virtual network**: Select the AKS VNet
   - Leave other defaults
5. Click **"Add"**

📸 **Screenshot: Add peering**
```
Add peering

This virtual network
Peering link name: jumpserver-to-aks
☑ Allow 'vnet-jumpserver' to access the peered virtual network
☐ Allow 'vnet-jumpserver' to receive forwarded traffic from the peered virtual network

Remote virtual network
Peering link name: aks-to-jumpserver
Virtual network deployment model: Resource manager ▼
Subscription: [Your subscription]
Virtual network: aks-vnet-12345678 ▼

[Add button]
```

**Via Azure CLI:**

```bash
# Create peering from Jump Server VNet to AKS VNet
az network vnet peering create \
  --name jumpserver-to-aks \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --remote-vnet $AKS_VNET_ID \
  --allow-vnet-access

# Create peering from AKS VNet to Jump Server VNet
az network vnet peering create \
  --name aks-to-jumpserver \
  --resource-group $NODE_RG \
  --vnet-name $AKS_VNET_NAME \
  --remote-vnet $JUMPSERVER_VNET_ID \
  --allow-vnet-access

echo "VNet peering configured"
```

### Step 8.3: Verify VNet Peering

```bash
# Check peering status
az network vnet peering list \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --output table

# Expected output:
# Name                PeeringState    ProvisioningState
# ------------------  --------------  -------------------
# jumpserver-to-aks   Connected       Succeeded

# Test connectivity from Jump Server to AKS node (if you have node IP)
# Get AKS node private IP
kubectl get nodes -o wide

# From Jump Server, ping the node (if ICMP is allowed)
# ping <node-private-ip>
```


---

## 9. Security Hardening

### Step 9.1: Disable Password Authentication

```bash
# On Jump Server, edit SSH config
sudo vim /etc/ssh/sshd_config

# Find and set these values:
# PasswordAuthentication no
# ChallengeResponseAuthentication no
# UsePAM no

# Or use sed to update
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Restart SSH service
sudo systemctl restart sshd

echo "Password authentication disabled"
```

### Step 9.2: Configure Firewall (UFW)

```bash
# Install and enable UFW
sudo apt-get install -y ufw

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw --force enable

# Check status
sudo ufw status verbose

# Expected output:
# Status: active
# To                         Action      From
# --                         ------      ----
# 22/tcp                     ALLOW       Anywhere
```

### Step 9.3: Set Up Automatic Security Updates

```bash
# Install unattended-upgrades
sudo apt-get install -y unattended-upgrades

# Enable automatic updates
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Configure to automatically reboot if needed (optional)
echo 'Unattended-Upgrade::Automatic-Reboot "true";' | sudo tee -a /etc/apt/apt.conf.d/50unattended-upgrades
echo 'Unattended-Upgrade::Automatic-Reboot-Time "03:00";' | sudo tee -a /etc/apt/apt.conf.d/50unattended-upgrades

echo "Automatic security updates enabled"
```

### Step 9.4: Configure Fail2Ban (Optional)

```bash
# Install fail2ban
sudo apt-get install -y fail2ban

# Copy default config
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Edit configuration
sudo vim /etc/fail2ban/jail.local

# Find [sshd] section and ensure it's enabled:
# [sshd]
# enabled = true
# port    = ssh
# logpath = %(sshd_log)s
# maxretry = 3
# bantime = 3600

# Or use sed to enable
sudo sed -i '/\[sshd\]/a enabled = true' /etc/fail2ban/jail.local

# Start and enable fail2ban
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Check status
sudo fail2ban-client status sshd

echo "Fail2ban configured"
```

### Step 9.5: Enable Azure Disk Encryption (Optional)

**Via Azure CLI (from local machine, not Jump Server):**

```bash
# Create Key Vault for encryption keys
KEYVAULT_NAME="kv-jumpserver-$RANDOM"

az keyvault create \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enabled-for-disk-encryption

# Enable disk encryption on VM
az vm encryption enable \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --disk-encryption-keyvault $KEYVAULT_NAME \
  --volume-type all

# Check encryption status
az vm encryption show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

echo "Disk encryption enabled"
```

### Step 9.6: Implement IP Whitelisting

**Update NSG to allow only specific IP ranges:**

```bash
# Remove broad SSH rule and add specific ones
az network nsg rule delete \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-Inbound

# Add rules for specific IP ranges
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-Office \
  --priority 100 \
  --source-address-prefixes "203.0.113.0/24" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-VPN \
  --priority 101 \
  --source-address-prefixes "198.51.100.0/24" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

echo "IP whitelisting configured"
```


---

## 10. Verification

### Step 10.1: Verify Jump Server is Running

**Via Azure Portal:**

1. Go to your VM (`vm-jumpserver`)
2. Check **Status** is **"Running"**

📸 **Screenshot: VM Status**
```
vm-jumpserver | Overview
Status: Running ⚫
Computer name: vm-jumpserver
Operating system: Linux (ubuntu 22.04)
Size: Standard_B2s (2 vcpus, 4 GiB memory)
Public IP address: 20.123.45.67
Private IP address: 10.1.1.4
```

**Via Azure CLI:**

```bash
# Check VM status
az vm get-instance-view \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query instanceView.statuses[1] \
  --output table

# Expected output:
# Code               DisplayStatus
# -----------------  -------------
# PowerState/running VM running
```

### Step 10.2: Verify Network Connectivity

**From your local machine:**

```bash
# Test SSH connection
ssh -i ~/.ssh/vm-jumpserver_key azureuser@$VM_PUBLIC_IP "echo 'Connection successful'"

# Expected output:
# Connection successful
```

**From Jump Server:**

```bash
# Test Azure connectivity
az account show --query name -o tsv

# Test AKS connectivity
kubectl get nodes

# Test internet connectivity
curl -I https://google.com

echo "✓ All connectivity tests passed"
```

### Step 10.3: Verify Installed Tools

**On Jump Server:**

```bash
# Check versions
echo "=== Tool Versions ==="
echo "Azure CLI: $(az version --query '\"azure-cli\"' -o tsv)"
echo "kubectl: $(kubectl version --client --short 2>/dev/null || kubectl version --client)"
echo "Helm: $(helm version --short)"
echo "Docker: $(docker --version 2>/dev/null || echo 'Not installed')"
echo "Git: $(git --version)"
echo "jq: $(jq --version)"
echo ""

# Check Azure login status
echo "=== Azure Login Status ==="
az account show --query "{SubscriptionName:name, User:user.name}" -o table
echo ""

# Check kubectl context
echo "=== kubectl Context ==="
kubectl config current-context
echo ""

# Check AKS access
echo "=== AKS Cluster Info ==="
kubectl cluster-info
echo ""

echo "✓ All verifications passed"
```

### Step 10.4: Verify Security Configuration

```bash
# Check SSH config
sudo grep "PasswordAuthentication" /etc/ssh/sshd_config

# Check firewall status
sudo ufw status

# Check fail2ban status
sudo systemctl status fail2ban --no-pager

# Check NSG rules
az network nsg rule list \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --query "[].{Name:name, Priority:priority, Access:access, Protocol:protocol, Direction:direction}" \
  --output table

echo "✓ Security verification complete"
```


---

## 11. Troubleshooting

### Issue 1: Cannot SSH to Jump Server

**Symptoms:**
```bash
ssh -i vm-jumpserver_key.pem azureuser@20.123.45.67
# Connection timed out
```

**Solutions:**

```bash
# Solution 1: Check VM is running
az vm get-instance-view \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query instanceView.statuses[1]

# If not running, start it
az vm start --resource-group $RESOURCE_GROUP --name $VM_NAME

# Solution 2: Verify NSG rules allow your IP
MY_IP=$(curl -s ifconfig.me)
echo "Your IP: $MY_IP"

az network nsg rule list \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --output table

# Add your IP if not present
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name Allow-SSH-MyIP \
  --priority 102 \
  --source-address-prefixes "${MY_IP}/32" \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

# Solution 3: Check SSH key permissions
chmod 400 vm-jumpserver_key.pem

# Solution 4: Verify public IP
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query publicIps -o tsv
```

### Issue 2: "Permission denied (publickey)"

**Symptoms:**
```bash
ssh -i vm-jumpserver_key.pem azureuser@20.123.45.67
# Permission denied (publickey)
```

**Solutions:**

```bash
# Solution 1: Verify you're using the correct key
ssh -i vm-jumpserver_key.pem -v azureuser@$VM_PUBLIC_IP
# Look for which keys are being tried

# Solution 2: Verify key permissions
ls -l vm-jumpserver_key.pem
# Should show -r-------- (400)

chmod 400 vm-jumpserver_key.pem

# Solution 3: Try with correct username
ssh -i vm-jumpserver_key.pem azureuser@$VM_PUBLIC_IP
# Default username is 'azureuser'

# Solution 4: Reset SSH key via Azure CLI
az vm user update \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --username azureuser \
  --ssh-key-value "$(cat ~/.ssh/vm-jumpserver_key.pub)"
```

### Issue 3: Cannot access AKS from Jump Server

**Symptoms:**
```bash
kubectl get nodes
# Unable to connect to the server: dial tcp: lookup aks-demo-cluster
```

**Solutions:**

```bash
# Solution 1: Verify Azure login
az account show
# If not logged in:
az login --use-device-code

# Solution 2: Re-fetch AKS credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --overwrite-existing

# Solution 3: Check network connectivity to AKS
AKS_FQDN=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER_NAME \
  --query fqdn -o tsv)

nslookup $AKS_FQDN
ping -c 3 $AKS_FQDN

# Solution 4: Check kubectl config
kubectl config view
kubectl config get-contexts
kubectl config use-context $AKS_CLUSTER_NAME
```

### Issue 4: Out of disk space on Jump Server

**Symptoms:**
```bash
df -h
# /dev/sda1  30G  29G  0  100% /
```

**Solutions:**

```bash
# Check disk usage
df -h
du -sh /* 2>/dev/null | sort -h

# Clean up package cache
sudo apt-get clean
sudo apt-get autoremove -y

# Clean up old logs
sudo journalctl --vacuum-time=7d

# If using Docker, clean up
docker system prune -a -f

# Resize disk via Azure Portal:
# 1. Stop VM
# 2. Go to Disks
# 3. Click on OS disk
# 4. Increase size
# 5. Start VM
# 6. Resize partition:
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
```


### Issue 5: Jump Server stopped automatically

**Symptoms:**
- Cannot connect to Jump Server at expected time
- VM shows as "Stopped (deallocated)" in portal

**Cause:** Auto-shutdown feature is enabled

**Solutions:**

```bash
# Check auto-shutdown status
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "shutdownTimeUtc"

# Disable auto-shutdown
az vm auto-shutdown \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --off

# Or configure different shutdown time via Portal:
# VM > Operations > Auto-shutdown > Modify time or disable
```

### Issue 6: High CPU usage on Jump Server

**Symptoms:**
```bash
top
# Shows high CPU usage
```

**Solutions:**

```bash
# Check what's using CPU
top
htop  # If installed

# Check running processes
ps aux --sort=-%cpu | head -20

# If it's a legitimate load, consider upgrading VM size
az vm resize \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --size Standard_B4ms

# Current size has 2 vCPUs, 4 GB RAM
# Standard_B4ms has 4 vCPUs, 16 GB RAM
```

### Issue 7: Cannot install packages (permission denied)

**Symptoms:**
```bash
apt-get install package-name
# E: Could not open lock file - open (13: Permission denied)
```

**Solutions:**

```bash
# Use sudo
sudo apt-get install package-name

# If sudo doesn't work, check if user is in sudo group
groups
# Should show: azureuser sudo ...

# Add user to sudo group if not present
sudo usermod -aG sudo azureuser

# Logout and login again
exit
ssh -i vm-jumpserver_key.pem azureuser@$VM_PUBLIC_IP
```

### Issue 8: VNet peering not working

**Symptoms:**
- Cannot ping AKS nodes from Jump Server
- Peering shows "Disconnected"

**Solutions:**

```bash
# Check peering status
az network vnet peering list \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --output table

# If showing Disconnected, delete and recreate
az network vnet peering delete \
  --name jumpserver-to-aks \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME

# Recreate peering
az network vnet peering create \
  --name jumpserver-to-aks \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --remote-vnet $AKS_VNET_ID \
  --allow-vnet-access

# Verify both directions are connected
az network vnet peering show \
  --name jumpserver-to-aks \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --query peeringState
```

---

## Summary Checklist

After completing this guide, you should have:

- [ ] Created Virtual Network for Jump Server
- [ ] Created and configured Network Security Group
- [ ] Created Jump Server VM with Ubuntu
- [ ] Configured SSH access with key-based authentication
- [ ] Installed Azure CLI, kubectl, and other tools
- [ ] Configured kubectl access to AKS cluster
- [ ] Set up VNet peering (optional)
- [ ] Hardened security (disabled password auth, enabled firewall)
- [ ] Verified connectivity to AKS and deployed test application
- [ ] Documented connection details and credentials

### Important Information to Save

```bash
# Jump Server Information
VM_NAME=vm-jumpserver
VM_RESOURCE_GROUP=rg-aks-demo
VM_PUBLIC_IP=20.123.45.67
VM_PRIVATE_IP=10.1.1.4
VM_SIZE=Standard_B2s
VM_USERNAME=azureuser
SSH_KEY_PATH=~/.ssh/vm-jumpserver_key.pem

# Network Information
VNET_NAME=vnet-jumpserver
SUBNET_NAME=subnet-jumpserver
NSG_NAME=nsg-jumpserver
VNET_ADDRESS_SPACE=10.1.0.0/16
SUBNET_ADDRESS_RANGE=10.1.1.0/24

# Connection Command
SSH_COMMAND="ssh -i ~/.ssh/vm-jumpserver_key.pem azureuser@20.123.45.67"
```


### Jump Server Quick Reference Commands

**From Local Machine:**

```bash
# Connect to Jump Server
ssh -i vm-jumpserver_key.pem azureuser@<VM_PUBLIC_IP>

# Copy files to Jump Server
scp -i vm-jumpserver_key.pem file.txt azureuser@<VM_PUBLIC_IP>:~/

# Copy files from Jump Server
scp -i vm-jumpserver_key.pem azureuser@<VM_PUBLIC_IP>:~/file.txt ./

# Port forwarding (access AKS service locally)
ssh -i vm-jumpserver_key.pem -L 8080:localhost:80 azureuser@<VM_PUBLIC_IP>

# Start Jump Server
az vm start --resource-group rg-aks-demo --name vm-jumpserver

# Stop Jump Server (to save costs)
az vm stop --resource-group rg-aks-demo --name vm-jumpserver

# Deallocate Jump Server (stop and release resources)
az vm deallocate --resource-group rg-aks-demo --name vm-jumpserver
```

**From Jump Server:**

```bash
# Azure operations
az login --use-device-code
az account show
az aks get-credentials --resource-group rg-aks-demo --name aks-demo-cluster

# Kubernetes operations
kubectl get nodes
kubectl get pods --all-namespaces
kubectl apply -f deployment.yaml
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# System maintenance
sudo apt-get update
sudo apt-get upgrade -y
df -h  # Check disk space
free -m  # Check memory
top  # Check processes
```

---

## Best Practices

### 1. Security
- ✅ Use key-based authentication only (disable passwords)
- ✅ Restrict SSH access to specific IPs via NSG
- ✅ Enable automatic security updates
- ✅ Implement fail2ban for intrusion prevention
- ✅ Use Azure Bastion instead of public IP (production)
- ✅ Enable disk encryption
- ✅ Regularly rotate SSH keys

### 2. Cost Optimization
- ✅ Enable auto-shutdown for non-production Jump Servers
- ✅ Use appropriate VM size (B-series for burstable workloads)
- ✅ Deallocate VM when not in use
- ✅ Use spot instances for dev/test environments
- ✅ Clean up unused resources regularly

### 3. Reliability
- ✅ Create VM snapshots before major changes
- ✅ Document all configuration changes
- ✅ Keep tools and OS updated
- ✅ Monitor disk space usage
- ✅ Set up backup scripts for important configurations

### 4. Operations
- ✅ Use VNet peering for direct AKS access
- ✅ Install bash completion for easier command usage
- ✅ Create aliases for frequently used commands
- ✅ Keep SSH keys secure and backed up
- ✅ Document Jump Server usage procedures

### 5. Alternative: Azure Bastion

For production environments, consider using **Azure Bastion** instead of a Jump Server VM:

**Advantages:**
- Fully managed PaaS service
- No public IP needed on VMs
- RDP/SSH access through Azure Portal
- Better security and compliance
- No VM maintenance required

**Create Azure Bastion:**

```bash
# Create Bastion subnet
az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name AzureBastionSubnet \
  --address-prefixes 10.1.2.0/27

# Create Bastion public IP
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name bastion-ip \
  --sku Standard \
  --location $LOCATION

# Create Bastion host
az network bastion create \
  --resource-group $RESOURCE_GROUP \
  --name bastion-host \
  --public-ip-address bastion-ip \
  --vnet-name $VNET_NAME \
  --location $LOCATION
```

---

## Next Steps

Now that your Jump Server is configured and running, proceed to:

👉 **[05-blob-storage-setup.md](./05-blob-storage-setup.md)** - Set up Azure Blob Storage with Azure AD authentication

---

## Additional Resources

- [Azure Bastion Documentation](https://docs.microsoft.com/azure/bastion/bastion-overview)
- [Secure AKS Access](https://docs.microsoft.com/azure/aks/operator-best-practices-network)
- [VNet Peering](https://docs.microsoft.com/azure/virtual-network/virtual-network-peering-overview)
- [Azure VM Security](https://docs.microsoft.com/azure/security/fundamentals/virtual-machines-overview)
- [SSH Best Practices](https://docs.microsoft.com/azure/virtual-machines/linux/create-ssh-keys-detailed)
- [Network Security Groups](https://docs.microsoft.com/azure/virtual-network/network-security-groups-overview)
