# Azure Infrastructure Setup Guide

Complete step-by-step documentation for setting up Azure Kubernetes Service (AKS), Azure Container Registry (ACR), Jump Server VM, Blob Storage, and Azure AD authentication.

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture](#architecture)
4. [Setup Guides](#setup-guides)
5. [Best Practices](#best-practices)

## Overview

This documentation provides comprehensive guides for setting up a complete Azure infrastructure including:

- **Azure Active Directory (Azure AD)**: Identity and access management
- **Azure Container Registry (ACR)**: Private Docker container registry
- **Azure Kubernetes Service (AKS)**: Managed Kubernetes cluster
- **Jump Server (VM)**: Secure access point to AKS cluster
- **Blob Storage**: Object storage with Azure AD authentication

## Prerequisites

Before starting, ensure you have:

- [ ] Active Azure subscription
- [ ] Azure CLI installed locally
- [ ] Owner or Contributor role on the subscription
- [ ] kubectl installed (for Kubernetes management)
- [ ] Basic understanding of Kubernetes concepts
- [ ] SSH client for Jump Server access

## Architecture

The infrastructure components work together as follows:

```
Azure AD (Identity Provider)
    ↓
    ├─→ ACR (Container Registry)
    ├─→ AKS (Kubernetes Cluster)
    ├─→ Jump Server VM (Bastion/Gateway)
    └─→ Blob Storage (Object Storage)
```

**Access Flow:**
1. Users authenticate via Azure AD
2. Jump Server VM provides secure access to AKS cluster
3. AKS pulls container images from ACR using managed identity
4. Applications access Blob Storage using Azure AD authentication

## Setup Guides

Follow these guides in order for a complete setup:

### 1. [Azure AD Setup](./docs/01-azure-ad-setup.md)
- Create Azure AD tenant (if needed)
- Set up service principals
- Configure App registrations
- Assign roles and permissions

### 2. [Azure Container Registry (ACR) Setup](./docs/02-acr-setup.md)
- Create ACR instance
- Configure Azure AD authentication
- Enable admin user (optional)
- Push sample container images
- Set up RBAC for ACR access

### 3. [Azure Kubernetes Service (AKS) Setup](./docs/03-aks-setup.md)
- Create AKS cluster
- Configure Azure AD integration
- Attach ACR to AKS
- Configure kubectl access
- Deploy sample application

### 4. [Jump Server VM Setup](./docs/04-jump-server-setup.md)
- Create Virtual Network (VNet)
- Create Jump Server VM
- Configure Network Security Groups (NSG)
- Install kubectl and Azure CLI
- Set up SSH access
- Connect to AKS from Jump Server

### 5. [Blob Storage Setup](./docs/05-blob-storage-setup.md)
- Create Storage Account
- Create Blob Container
- Configure Azure AD authentication
- Set up RBAC roles
- Test access from applications
- Enable encryption and security features

### 6. [Architecture & Best Practices](./docs/06-architecture-best-practices.md)
- Architecture diagram
- Security best practices
- Cost optimization tips
- Monitoring and logging
- Troubleshooting guide

## Quick Start

If you want to set up everything quickly:

```bash
# Login to Azure
az login

# Set your subscription
az account set --subscription "Your-Subscription-Name"

# Create resource group
az group create --name rg-aks-demo --location eastus

# Follow the individual setup guides
```

## Support and Troubleshooting

Each guide includes a troubleshooting section. Common issues:

- **Authentication failures**: Check Azure AD permissions
- **Network connectivity**: Verify NSG rules and VNet configuration
- **ACR access denied**: Ensure managed identity has AcrPull role
- **Blob storage access**: Verify RBAC role assignments

## Additional Resources

- [Azure AKS Documentation](https://docs.microsoft.com/azure/aks/)
- [Azure ACR Documentation](https://docs.microsoft.com/azure/container-registry/)
- [Azure Storage Documentation](https://docs.microsoft.com/azure/storage/)
- [Azure AD Documentation](https://docs.microsoft.com/azure/active-directory/)

## Version History

- **v1.0** (June 2026) - Initial documentation created

---

**Note**: Screenshots are referenced in each guide. Look for `📸 Screenshot:` markers indicating where to take screenshots in the Azure Portal.
