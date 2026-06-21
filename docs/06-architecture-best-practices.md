# Architecture & Best Practices Guide

## 📋 Overview

This guide provides a comprehensive overview of the complete Azure infrastructure architecture, visual diagrams, security best practices, cost optimization strategies, and operational recommendations for the AKS, ACR, Jump Server, and Blob Storage setup.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Component Interaction Diagram](#2-component-interaction-diagram)
3. [Network Architecture](#3-network-architecture)
4. [Security Best Practices](#4-security-best-practices)
5. [Cost Optimization](#5-cost-optimization)
6. [Reliability & High Availability](#6-reliability--high-availability)
7. [Monitoring & Observability](#7-monitoring--observability)
8. [Disaster Recovery](#8-disaster-recovery)
9. [DevOps & CI/CD Integration](#9-devops--cicd-integration)
10. [Common Scenarios & Solutions](#10-common-scenarios--solutions)

---

## 1. Architecture Overview

### 1.1: Complete Infrastructure Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Azure Active Directory                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│  │ Service      │  │ App          │  │ Managed Identities      │  │
│  │ Principals   │  │ Registrations│  │ (System & User-Assigned)│  │
│  └──────────────┘  └──────────────┘  └─────────────────────────┘  │
│                    ↓ Authentication & Authorization ↓               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                     Azure Subscription (RBAC)                        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              Resource Group: rg-aks-demo                    │   │
│  │                                                             │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │  Azure Container Registry (ACR)                    │   │   │
│  │  │  - Name: acrdemoacr                                │   │   │
│  │  │  - SKU: Standard                                   │   │   │
│  │  │  - Azure AD Integration                            │   │   │
│  │  │  - Images: samples/nginx, samples/app              │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │           ↓ Image Pull (AcrPull Role)                     │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │  Azure Kubernetes Service (AKS)                    │   │   │
│  │  │  - Name: aks-demo-cluster                          │   │   │
│  │  │  - Nodes: 2x Standard_D2s_v3                       │   │   │
│  │  │  - Network: Azure CNI                              │   │   │
│  │  │  - Monitoring: Container Insights                  │   │   │
│  │  │  ┌──────────────────────────────────┐              │   │   │
│  │  │  │  Pods & Services                 │              │   │   │
│  │  │  │  - sample-app (2 replicas)       │              │   │   │
│  │  │  │  - LoadBalancer Service          │              │   │   │
│  │  │  └──────────────────────────────────┘              │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │                                                             │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │  Blob Storage Account                              │   │   │
│  │  │  - Name: staksdemostorage                          │   │   │
│  │  │  - Azure AD Auth Enabled                           │   │   │
│  │  │  - Containers: app-data, backup, logs              │   │   │
│  │  │  - Encryption: Enabled                             │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │           ↑ Azure AD Access (RBAC)                         │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │  Jump Server VM                                    │   │   │
│  │  │  - Name: vm-jumpserver                             │   │   │
│  │  │  - Size: Standard_B2s                              │   │   │
│  │  │  - OS: Ubuntu 22.04 LTS                            │   │   │
│  │  │  - Tools: Azure CLI, kubectl, Helm                 │   │   │
│  │  │  - SSH Key Authentication                          │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │           ↓ kubectl commands                               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │     Node Resource Group: MC_rg-aks-demo_...               │   │
│  │  - Virtual Network (AKS)                                   │   │
│  │  - Load Balancer                                           │   │
│  │  - Network Security Groups                                 │   │
│  │  - Public IPs                                              │   │
│  └────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────────────────┐
                    │  External Users/    │
                    │  Applications       │
                    └─────────────────────┘
```

### 1.2: Component Relationships

| Component | Depends On | Purpose | Azure AD Integration |
|-----------|-----------|---------|---------------------|
| **Azure AD** | - | Identity provider for all resources | Core authentication |
| **ACR** | Azure AD | Store container images | Service principals, managed identities |
| **AKS** | ACR, Azure AD | Run containerized applications | Managed identity for ACR pull |
| **Jump Server** | Azure AD | Secure access to AKS | User authentication |
| **Blob Storage** | Azure AD | Object storage | RBAC roles for access control |


---

## 2. Component Interaction Diagram

### 2.1: Authentication Flow

```
┌──────────────┐
│   User/App   │
└──────┬───────┘
       │ 1. Authenticate
       ↓
┌──────────────────────────┐
│  Azure Active Directory  │
└──────┬───────────────────┘
       │ 2. Return Token
       ↓
┌──────────────────────────┐
│   Azure Resource         │
│  (ACR/AKS/Blob/VM)       │
│                          │
│  3. Validate Token       │
│  4. Check RBAC Roles     │
│  5. Grant/Deny Access    │
└──────────────────────────┘
```

### 2.2: Container Deployment Flow

```
Developer Workstation
       │
       │ 1. docker build
       │ 2. docker tag
       ↓
┌──────────────────┐
│       ACR        │
│  (Login via      │
│   Azure AD)      │
└────────┬─────────┘
         │ 3. docker push
         │
         │ 4. kubectl apply
         ↓
┌──────────────────────────┐
│         AKS              │
│                          │
│  ┌────────────────────┐  │
│  │  kubelet pulls     │  │
│  │  image from ACR    │  │
│  │  (managed identity)│  │
│  └────────────────────┘  │
│                          │
│  ┌────────────────────┐  │
│  │  Pod starts with   │  │
│  │  container image   │  │
│  └────────────────────┘  │
└──────────────────────────┘
         │
         │ 5. Access via LoadBalancer
         ↓
┌──────────────────┐
│   End Users      │
└──────────────────┘
```

### 2.3: Data Access Flow

```
┌──────────────────┐
│  Application     │
│  (AKS Pod)       │
└────────┬─────────┘
         │ 1. Request with Managed Identity
         ↓
┌──────────────────────────┐
│   Azure AD               │
│   (Token Exchange)       │
└────────┬─────────────────┘
         │ 2. Access Token
         ↓
┌──────────────────────────┐
│   Blob Storage           │
│                          │
│  3. Validate Token       │
│  4. Check RBAC           │
│     (Storage Blob Data   │
│      Contributor)        │
│  5. Return Data          │
└──────────────────────────┘
```

### 2.4: Jump Server Access Pattern

```
┌──────────────────┐
│  Administrator   │
│  (Local Machine) │
└────────┬─────────┘
         │ 1. SSH (Key-based auth)
         ↓
┌──────────────────────────┐
│   Jump Server VM         │
│   - Public IP            │
│   - NSG (Port 22)        │
│                          │
│  ┌────────────────────┐  │
│  │  az login          │  │
│  │  (Device code)     │  │
│  └────────────────────┘  │
│           ↓              │
│  ┌────────────────────┐  │
│  │  kubectl commands  │  │
│  │  to AKS API        │  │
│  └────────────────────┘  │
└────────┬─────────────────┘
         │ 2. kubectl via Azure AD
         ↓
┌──────────────────────────┐
│   AKS API Server         │
│   (Private/Public)       │
└──────────────────────────┘
```

---

## 3. Network Architecture

### 3.1: Network Diagram

```
Internet
    │
    │ HTTPS (443)
    │ SSH (22) - Restricted IPs
    ↓
┌─────────────────────────────────────────────────────────────┐
│                   Azure Public Network                       │
│                                                              │
│  ┌────────────────┐        ┌──────────────────┐            │
│  │ Jump Server    │        │  AKS LoadBalancer │            │
│  │ Public IP      │        │  Public IP        │            │
│  │ 20.123.45.67   │        │  20.234.56.78     │            │
│  └────────┬───────┘        └──────────┬────────┘            │
└───────────┼───────────────────────────┼─────────────────────┘
            │                           │
┌───────────┼───────────────────────────┼─────────────────────┐
│           │     Azure Virtual Network │                     │
│           │                           │                     │
│  ┌────────┴─────────────┐    ┌───────┴──────────────┐     │
│  │  VNet: Jump Server   │    │  VNet: AKS           │     │
│  │  10.1.0.0/16         │    │  10.224.0.0/12       │     │
│  │                      │    │                      │     │
│  │  ┌────────────────┐ │    │  ┌────────────────┐ │     │
│  │  │ Subnet         │ │    │  │ AKS Subnet     │ │     │
│  │  │ 10.1.1.0/24    │ │    │  │ 10.224.0.0/16  │ │     │
│  │  │                │ │    │  │                │ │     │
│  │  │ Jump Server VM │ │    │  │ AKS Nodes (2)  │ │     │
│  │  │ 10.1.1.4       │ │    │  │ 10.224.0.4-5   │ │     │
│  │  └────────────────┘ │    │  └────────────────┘ │     │
│  └──────────────────────┘    └─────────────────────┘     │
│           │                           │                   │
│           └─────VNet Peering (opt)────┘                   │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │  Storage Account (Private Endpoints optional)    │     │
│  │  staksdemostorage.blob.core.windows.net          │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │  Container Registry                              │     │
│  │  acrdemoacr.azurecr.io                           │     │
│  └──────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────┘
```

### 3.2: Network Security Groups (NSG) Rules

**Jump Server NSG:**

| Priority | Name | Port | Protocol | Source | Destination | Action |
|----------|------|------|----------|--------|-------------|--------|
| 100 | Allow-SSH-Inbound | 22 | TCP | Your-IP/32 | Any | Allow |
| 110 | Allow-HTTPS-Outbound | 443 | TCP | Any | Internet | Allow |
| 65500 | DenyAllInbound | Any | Any | Any | Any | Deny |

**AKS NSG (Auto-managed):**

| Priority | Name | Port | Protocol | Source | Destination | Action |
|----------|------|------|----------|--------|-------------|--------|
| 100 | AllowVnetInBound | Any | Any | VirtualNetwork | VirtualNetwork | Allow |
| 101 | AllowAzureLoadBalancer | Any | Any | AzureLoadBalancer | Any | Allow |
| 200 | AllowInternetOutBound | Any | Any | Any | Internet | Allow |

### 3.3: Traffic Flow Patterns

**Inbound Traffic:**
1. External User → Azure LoadBalancer → AKS Service → Pod
2. Administrator → Jump Server (SSH) → AKS API Server (kubectl)
3. CI/CD Pipeline → ACR (image push)

**Outbound Traffic:**
1. AKS Nodes → ACR (image pull)
2. AKS Pods → Blob Storage (data access)
3. AKS Pods → External APIs (if needed)
4. Jump Server → AKS API Server
5. Jump Server → Blob Storage

### 3.4: DNS Configuration

```
External DNS:
- *.azurecr.io          → ACR endpoint
- *.hcp.eastus.azmk8s.io → AKS API server
- *.blob.core.windows.net → Blob storage

Internal DNS (AKS):
- *.svc.cluster.local   → Kubernetes services
- CoreDNS for internal resolution
```


---

## 4. Security Best Practices

### 4.1: Identity & Access Management

**Principle of Least Privilege:**

```yaml
# Recommended RBAC Assignments

Users:
  Developers:
    - Azure Kubernetes Service Cluster User Role
    - Storage Blob Data Contributor (specific containers)
  
  DevOps Engineers:
    - Azure Kubernetes Service Cluster Admin Role
    - AcrPush (ACR)
    - Storage Blob Data Contributor
  
  Readers/Auditors:
    - Reader (resource groups)
    - Storage Blob Data Reader
    - AcrPull (ACR)

Service Principals:
  CI/CD Pipeline:
    - AcrPush (ACR)
    - Azure Kubernetes Service Cluster User Role
  
  AKS Cluster:
    - AcrPull (ACR) - via managed identity
    - Storage Blob Data Reader/Contributor

Managed Identities:
  AKS Pod Identity:
    - Storage Blob Data Contributor (specific containers)
    - Other Azure service access as needed
```

**Security Checklist:**

- [ ] Enable Azure AD integration for all services
- [ ] Use managed identities where possible
- [ ] Disable storage account key access
- [ ] Rotate service principal secrets every 90 days
- [ ] Implement conditional access policies
- [ ] Enable MFA for user accounts
- [ ] Use Azure AD PIM for just-in-time admin access
- [ ] Regular access reviews (quarterly)
- [ ] Enable audit logging for all resources

### 4.2: Network Security

**Defense in Depth Strategy:**

```
Layer 1: Perimeter
├── Azure DDoS Protection
├── Azure Firewall (optional)
└── NSG rules (restrictive)

Layer 2: Network
├── Private endpoints for services
├── VNet peering for internal communication
├── No public IPs for worker nodes
└── Network policies in AKS

Layer 3: Application
├── Ingress controller with WAF
├── TLS/SSL for all traffic
├── API authentication tokens
└── Rate limiting

Layer 4: Data
├── Encryption at rest (storage)
├── Encryption in transit (TLS 1.2+)
├── Customer-managed keys (optional)
└── Data classification and tagging
```

**Implementation:**

```bash
# Enable private endpoint for storage account
az network private-endpoint create \
  --name storage-pe \
  --resource-group rg-aks-demo \
  --vnet-name vnet-jumpserver \
  --subnet subnet-jumpserver \
  --private-connection-resource-id $STORAGE_ID \
  --group-id blob \
  --connection-name storage-connection

# Enable network policy in AKS
az aks update \
  --resource-group rg-aks-demo \
  --name aks-demo-cluster \
  --network-policy azure

# Create network policy to restrict pod communication
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

### 4.3: Container Security

**Best Practices:**

1. **Image Security:**
   - Scan images for vulnerabilities (Microsoft Defender for ACR)
   - Use minimal base images (Alpine, distroless)
   - Don't run containers as root
   - Use specific image tags (not `latest`)
   - Sign images with Docker Content Trust

2. **Pod Security:**
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: secure-pod
   spec:
     securityContext:
       runAsNonRoot: true
       runAsUser: 1000
       fsGroup: 2000
       seccompProfile:
         type: RuntimeDefault
     containers:
     - name: app
       image: acrdemoacr.azurecr.io/app:v1.0
       securityContext:
         allowPrivilegeEscalation: false
         readOnlyRootFilesystem: true
         capabilities:
           drop:
           - ALL
       resources:
         limits:
           cpu: 500m
           memory: 512Mi
         requests:
           cpu: 250m
           memory: 256Mi
   ```

3. **Secret Management:**
   - Use Azure Key Vault for secrets
   - Enable Key Vault CSI driver in AKS
   - Rotate secrets regularly
   - Never hardcode secrets in images

   ```bash
   # Install Key Vault CSI driver
   az aks enable-addons \
     --addons azure-keyvault-secrets-provider \
     --name aks-demo-cluster \
     --resource-group rg-aks-demo
   ```

### 4.4: Compliance & Auditing

**Enable Audit Logging:**

```bash
# Enable AKS diagnostic settings
az monitor diagnostic-settings create \
  --name aks-diagnostics \
  --resource $AKS_RESOURCE_ID \
  --logs '[
    {"category": "kube-apiserver", "enabled": true},
    {"category": "kube-audit", "enabled": true},
    {"category": "kube-controller-manager", "enabled": true},
    {"category": "cluster-autoscaler", "enabled": true}
  ]' \
  --workspace $LOG_ANALYTICS_WORKSPACE_ID

# Enable Storage diagnostic settings
az monitor diagnostic-settings create \
  --name storage-diagnostics \
  --resource $STORAGE_ID \
  --logs '[
    {"category": "StorageRead", "enabled": true},
    {"category": "StorageWrite", "enabled": true},
    {"category": "StorageDelete", "enabled": true}
  ]' \
  --workspace $LOG_ANALYTICS_WORKSPACE_ID
```

**Compliance Standards:**

- Azure Policy for governance
- Azure Security Center recommendations
- CIS Benchmarks for Kubernetes
- HIPAA/PCI-DSS compliance (if applicable)
- Regular vulnerability assessments


---

## 5. Cost Optimization

### 5.1: Cost Analysis by Component

**Monthly Cost Estimate (East US, as of 2026):**

| Component | Configuration | Estimated Monthly Cost |
|-----------|--------------|----------------------|
| **AKS Cluster** | 2 x Standard_D2s_v3 (8GB RAM, 2 vCPU) | $140 |
| **ACR** | Standard tier, 100GB storage | $20 |
| **Jump Server** | Standard_B2s (4GB RAM, 2 vCPU) | $30 |
| **Blob Storage** | 100GB LRS, Hot tier | $2 |
| **Networking** | LoadBalancer, egress | $25 |
| **Monitoring** | Log Analytics, Container Insights | $10 |
| **Total** | | **~$227/month** |

*Note: Costs vary by region, usage patterns, and Azure pricing changes.*

### 5.2: Cost Optimization Strategies

**1. Right-Size Resources:**

```bash
# Analyze AKS node utilization
kubectl top nodes

# If nodes are under 50% utilized, consider smaller VM sizes
az aks nodepool update \
  --resource-group rg-aks-demo \
  --cluster-name aks-demo-cluster \
  --name agentpool \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3

# Use spot instances for non-critical workloads
az aks nodepool add \
  --resource-group rg-aks-demo \
  --cluster-name aks-demo-cluster \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3
```

**2. Storage Cost Optimization:**

```bash
# Implement lifecycle management
cat << 'EOF' > lifecycle-policy.json
{
  "rules": [{
    "enabled": true,
    "name": "cost-optimization",
    "type": "Lifecycle",
    "definition": {
      "actions": {
        "baseBlob": {
          "tierToCool": {"daysAfterModificationGreaterThan": 30},
          "tierToArchive": {"daysAfterModificationGreaterThan": 90},
          "delete": {"daysAfterModificationGreaterThan": 365}
        }
      },
      "filters": {
        "blobTypes": ["blockBlob"]
      }
    }
  }]
}
EOF

az storage account management-policy create \
  --account-name staksdemostorage \
  --policy @lifecycle-policy.json
```

**3. Shutdown Non-Production Resources:**

```bash
# Stop dev/test resources during off-hours
# Create automation runbook or use Azure Automation

# Stop AKS cluster (saves compute costs)
az aks stop --name aks-demo-cluster --resource-group rg-aks-demo

# Deallocate Jump Server
az vm deallocate --name vm-jumpserver --resource-group rg-aks-demo

# Start resources when needed
az aks start --name aks-demo-cluster --resource-group rg-aks-demo
az vm start --name vm-jumpserver --resource-group rg-aks-demo
```

**4. Use Reserved Instances:**

For production workloads running 24/7, purchase reserved instances:
- 1-year or 3-year commitment
- Up to 72% savings on VMs
- Apply to AKS nodes

**5. Optimize Data Transfer:**

```bash
# Minimize data egress costs
# - Keep resources in same region
# - Use Azure CDN for static content
# - Compress data before transfer
# - Use ExpressRoute for large data transfers

# Monitor data transfer costs
az monitor metrics list \
  --resource $STORAGE_ID \
  --metric "Egress" \
  --start-time $(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ')
```

### 5.3: Cost Monitoring & Alerts

```bash
# Set up cost alerts
az consumption budget create \
  --resource-group rg-aks-demo \
  --budget-name monthly-budget \
  --amount 300 \
  --time-grain Monthly \
  --time-period '{
    "start-date": "2026-06-01T00:00:00Z"
  }' \
  --notifications '{
    "actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contact-emails": ["admin@example.com"],
      "contact-roles": ["Owner", "Contributor"]
    }
  }'

# Review cost analysis in Azure Portal
# Cost Management + Billing > Cost Analysis
# Filter by resource group: rg-aks-demo
# Group by: Service name, Resource
```

### 5.4: Cost Optimization Checklist

- [ ] Enable cluster autoscaler for AKS
- [ ] Use spot instances for dev/test
- [ ] Implement storage lifecycle policies
- [ ] Right-size VM instances based on actual usage
- [ ] Delete unused resources (old images, snapshots)
- [ ] Use Azure Hybrid Benefit (if applicable)
- [ ] Set up auto-shutdown for non-production VMs
- [ ] Monitor and set cost alerts
- [ ] Review Azure Advisor cost recommendations weekly
- [ ] Use Azure Cost Management for regular reviews
- [ ] Tag all resources for cost allocation
- [ ] Consider reserved instances for production

---

## 6. Reliability & High Availability

### 6.1: High Availability Architecture

**Production-Ready Configuration:**

```bash
# Multi-zone AKS cluster
az aks create \
  --resource-group rg-aks-prod \
  --name aks-prod-cluster \
  --node-count 3 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --node-vm-size Standard_D4s_v3 \
  --load-balancer-sku standard \
  --enable-managed-identity \
  --attach-acr acrdemoacr \
  --network-plugin azure

# Geo-redundant storage
az storage account create \
  --name staksprodstorge \
  --resource-group rg-aks-prod \
  --location eastus \
  --sku Standard_GRS \
  --kind StorageV2 \
  --access-tier Hot

# Premium ACR with geo-replication
az acr create \
  --resource-group rg-aks-prod \
  --name acrprodacr \
  --sku Premium \
  --location eastus

az acr replication create \
  --registry acrprodacr \
  --location westus
```

### 6.2: Application Reliability Patterns

**1. Pod Disruption Budgets:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

**2. Health Checks:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: healthy-pod
spec:
  containers:
  - name: app
    image: acrdemoacr.azurecr.io/app:v1
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2
```

**3. Horizontal Pod Autoscaling:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**4. Multi-Region Deployment (for critical apps):**

```
Region: East US (Primary)
├── AKS Cluster (active)
├── ACR with geo-replication
└── Storage (GRS)
    ↓ Replicated to
Region: West US (Secondary)
├── AKS Cluster (standby)
├── ACR replica
└── Storage (GRS read access)

Traffic Manager or Front Door
└── Routes traffic based on health/proximity
```

### 6.3: SLA Targets

| Component | Configuration | Azure SLA | Target Uptime |
|-----------|--------------|-----------|---------------|
| AKS | Multi-zone, 3+ nodes | 99.95% | 99.9% (43m/month) |
| ACR | Premium with geo-replication | 99.9% | 99.9% |
| Storage | GRS | 99.9% (RA-GRS: 99.99% read) | 99.9% |
| Overall System | All components | - | 99.9% |

**Calculating composite SLA:**
```
SLA_composite = SLA_aks × SLA_acr × SLA_storage
              = 0.9995 × 0.999 × 0.999
              = 0.9975 (99.75%)
```


---

## 7. Monitoring & Observability

### 7.1: Monitoring Strategy

**Three Pillars of Observability:**

```
1. Metrics (What's happening?)
   └── Azure Monitor Metrics
       ├── Node CPU/Memory/Disk
       ├── Pod resource usage
       ├── API server latency
       └── Storage IOPS/throughput

2. Logs (Why is it happening?)
   └── Azure Monitor Logs
       ├── Application logs
       ├── Kubernetes events
       ├── Audit logs
       └── Diagnostic logs

3. Traces (How did we get here?)
   └── Application Insights
       ├── Distributed tracing
       ├── Dependency maps
       └── Performance profiling
```

### 7.2: Monitoring Implementation

**1. Enable Container Insights:**

```bash
# Already enabled during AKS creation
# To enable on existing cluster:
az aks enable-addons \
  --resource-group rg-aks-demo \
  --name aks-demo-cluster \
  --addons monitoring

# View in Azure Portal:
# AKS Cluster > Insights > Cluster/Nodes/Controllers/Containers
```

**2. Custom Metrics and Alerts:**

```bash
# Create metric alert for high CPU
az monitor metrics alert create \
  --name high-cpu-alert \
  --resource-group rg-aks-demo \
  --scopes $AKS_RESOURCE_ID \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action email admin@example.com \
  --description "Alert when cluster CPU > 80%"

# Create alert for pod restarts
az monitor metrics alert create \
  --name pod-restart-alert \
  --resource-group rg-aks-demo \
  --scopes $AKS_RESOURCE_ID \
  --condition "total restartingContainerCount > 5" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --action email devops@example.com
```

**3. Log Analytics Queries:**

```kusto
// Top 10 error logs in last 24 hours
ContainerLog
| where TimeGenerated > ago(24h)
| where LogEntry contains "error" or LogEntry contains "ERROR"
| summarize Count=count() by ContainerID, Computer
| top 10 by Count desc

// Pod restart events
KubePodInventory
| where TimeGenerated > ago(24h)
| where PodStatus == "Failed" or RestartCount > 0
| summarize RestartCount=sum(RestartCount) by Name, Namespace, Computer
| order by RestartCount desc

// Node resource utilization
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsageNanoCores"
| summarize AvgCPU=avg(CounterValue) by Computer
| where AvgCPU > 80

// Storage access patterns
StorageBlobLogs
| where TimeGenerated > ago(24h)
| summarize Count=count() by OperationName, StatusCode, AccountName
| order by Count desc
```

**4. Application Insights Integration:**

```yaml
# Add Application Insights to your app
apiVersion: v1
kind: ConfigMap
metadata:
  name: appinsights-config
data:
  APPINSIGHTS_INSTRUMENTATIONKEY: "your-instrumentation-key"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitored-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: acrdemoacr.azurecr.io/app:v1
        env:
        - name: APPINSIGHTS_INSTRUMENTATIONKEY
          valueFrom:
            configMapKeyRef:
              name: appinsights-config
              key: APPINSIGHTS_INSTRUMENTATIONKEY
```

### 7.3: Dashboards

**Azure Dashboard Configuration:**

```json
{
  "tiles": [
    {
      "name": "AKS Cluster Health",
      "type": "Extension/Microsoft_Azure_Monitoring/PartType/MetricsChartPart",
      "settings": {
        "metrics": ["node_cpu_usage_percentage", "node_memory_working_set_percentage"]
      }
    },
    {
      "name": "Pod Status",
      "type": "Extension/Microsoft_Azure_Monitoring/PartType/LogsTablePart",
      "settings": {
        "query": "KubePodInventory | summarize by PodStatus"
      }
    },
    {
      "name": "Storage Operations",
      "type": "Extension/Microsoft_Azure_Storage/PartType/StorageMetricsPart",
      "settings": {
        "metrics": ["Transactions", "Ingress", "Egress"]
      }
    }
  ]
}
```

**Grafana Integration (Optional):**

```bash
# Install Grafana in AKS
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set persistence.enabled=true \
  --set adminPassword=admin

# Access Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80
# Open http://localhost:3000
```

### 7.4: Alerting Strategy

**Alert Severity Levels:**

| Severity | Description | Response Time | Examples |
|----------|-------------|---------------|----------|
| **Critical (Sev 0)** | Service down | Immediate (24/7) | Cluster unavailable, data loss |
| **High (Sev 1)** | Degraded service | 15 minutes | High error rate, slow response |
| **Medium (Sev 2)** | Warning signs | 1 hour | High CPU, disk space low |
| **Low (Sev 3)** | Informational | Next business day | Certificate expiring soon |

**Recommended Alerts:**

```bash
# Alert templates
alerts=(
  "Node CPU > 80% for 5 minutes"
  "Node Memory > 85% for 5 minutes"
  "Pod crash looping"
  "Persistent Volume usage > 80%"
  "API server error rate > 1%"
  "Container restart count > 5 in 15 minutes"
  "Storage account throttling"
  "ACR push/pull failures"
  "Certificate expiring in 30 days"
  "Failed authentication attempts > 10"
)
```

---

## 8. Disaster Recovery

### 8.1: Backup Strategy

**What to Backup:**

```
1. AKS Configuration
   ├── Kubernetes manifests (YAML files)
   ├── Helm charts and values
   ├── ConfigMaps and Secrets
   └── RBAC policies

2. Application Data
   ├── Persistent Volume data
   ├── Database backups
   └── Blob storage data

3. Container Images
   ├── ACR images (geo-replicated)
   └── Image tags and manifests

4. Configuration
   ├── Infrastructure as Code (Terraform/ARM)
   ├── Azure AD configuration
   └── Network configuration
```

**Backup Implementation:**

```bash
# 1. Backup AKS resources
kubectl get all --all-namespaces -o yaml > aks-backup-$(date +%Y%m%d).yaml
kubectl get configmaps --all-namespaces -o yaml > configmaps-backup-$(date +%Y%m%d).yaml
kubectl get secrets --all-namespaces -o yaml > secrets-backup-$(date +%Y%m%d).yaml

# 2. Use Velero for AKS backups
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=azure \
  --set configuration.backupStorageLocation.bucket=velero-backups \
  --set configuration.backupStorageLocation.config.resourceGroup=rg-aks-demo \
  --set configuration.backupStorageLocation.config.storageAccount=staksdemostorage

# Create backup schedule
velero schedule create daily-backup --schedule="0 2 * * *"

# 3. Enable blob storage soft delete (already configured)
# 4. Enable point-in-time restore for containers
az storage account blob-service-properties update \
  --account-name staksdemostorage \
  --enable-restore-policy true \
  --restore-days 7
```

### 8.2: Disaster Recovery Plan

**Recovery Time Objective (RTO) and Recovery Point Objective (RPO):**

| Component | RTO | RPO | Recovery Method |
|-----------|-----|-----|----------------|
| AKS Cluster | 30 minutes | 5 minutes | Re-create from IaC, restore from Velero |
| ACR | 15 minutes | Real-time | Geo-replication (automatic) |
| Blob Storage | 10 minutes | 1 hour | GRS replication or restore from backup |
| Application Data | 1 hour | 15 minutes | Restore from backup |

**DR Runbook:**

```markdown
# Disaster Recovery Procedure

## Scenario 1: AKS Cluster Failure

1. Assess Impact
   - Check Azure Service Health
   - Verify extent of failure
   - Estimate recovery time

2. Activate DR
   - Notify stakeholders
   - Switch DNS to secondary region (if multi-region)
   - Start recovery procedures

3. Recover AKS Cluster
   ```bash
   # Recreate cluster from Terraform/ARM
   terraform apply -target=azurerm_kubernetes_cluster.aks
   
   # Or create new cluster
   az aks create --resource-group rg-aks-dr ...
   
   # Restore from Velero
   velero restore create --from-backup daily-backup-20260621
   ```

4. Verify Services
   - Check all pods are running
   - Test application endpoints
   - Verify data integrity

5. Post-Recovery
   - Document incident
   - Update runbook
   - Conduct postmortem

## Scenario 2: Data Corruption/Loss

1. Stop writes to affected storage
2. Assess corruption extent
3. Restore from backup:
   ```bash
   # Point-in-time restore for blob storage
   az storage blob restore \
     --account-name staksdemostorage \
     --time-to-restore 2026-06-21T10:00:00Z
   ```
4. Validate restored data
5. Resume operations

## Scenario 3: Region Failure

1. Activate secondary region
2. Update DNS/Traffic Manager
3. Scale up secondary AKS cluster
4. Point applications to geo-replicated ACR
5. Use RA-GRS storage endpoint
```

### 8.3: Testing DR Procedures

```bash
# Quarterly DR test schedule
# Q1: Test AKS cluster recovery
# Q2: Test storage restore
# Q3: Test full region failover
# Q4: Test data corruption recovery

# DR Test Script
#!/bin/bash
echo "Starting DR Test - $(date)"

# 1. Create test namespace
kubectl create namespace dr-test

# 2. Deploy test application
kubectl apply -f test-app.yaml -n dr-test

# 3. Simulate failure
kubectl delete namespace dr-test --wait=false

# 4. Restore from backup
velero restore create dr-test-restore \
  --from-backup latest \
  --include-namespaces dr-test

# 5. Verify recovery
kubectl wait --for=condition=ready pod -l app=test-app -n dr-test --timeout=300s

# 6. Cleanup
kubectl delete namespace dr-test

echo "DR Test Complete - $(date)"
```


---

## 9. DevOps & CI/CD Integration

### 9.1: CI/CD Pipeline Architecture

```
┌─────────────────┐
│  Developer      │
│  (Git Push)     │
└────────┬────────┘
         │
         ↓
┌─────────────────────────────────┐
│  Source Control (GitHub/Azure   │
│  DevOps Repos)                  │
└────────┬────────────────────────┘
         │ Webhook/Trigger
         ↓
┌─────────────────────────────────┐
│  CI Pipeline                    │
│  ┌───────────────────────────┐  │
│  │ 1. Build Docker Image     │  │
│  │ 2. Run Tests              │  │
│  │ 3. Security Scan          │  │
│  │ 4. Push to ACR            │  │
│  └───────────────────────────┘  │
└────────┬────────────────────────┘
         │
         ↓
┌─────────────────────────────────┐
│  Azure Container Registry       │
│  (Tagged Image: v1.2.3)         │
└────────┬────────────────────────┘
         │
         ↓
┌─────────────────────────────────┐
│  CD Pipeline                    │
│  ┌───────────────────────────┐  │
│  │ 1. Update K8s manifests   │  │
│  │ 2. Apply to AKS (Dev)     │  │
│  │ 3. Run smoke tests        │  │
│  │ 4. Promote to Staging     │  │
│  │ 5. Approval gate          │  │
│  │ 6. Deploy to Production   │  │
│  └───────────────────────────┘  │
└────────┬────────────────────────┘
         │
         ↓
┌─────────────────────────────────┐
│  AKS Cluster (Production)       │
│  - Rolling update               │
│  - Health checks                │
│  - Automatic rollback on error  │
└─────────────────────────────────┘
```

### 9.2: GitHub Actions Pipeline Example

**.github/workflows/ci-cd.yml:**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  ACR_NAME: acrdemoacr
  AKS_CLUSTER: aks-demo-cluster
  RESOURCE_GROUP: rg-aks-demo
  IMAGE_NAME: myapp

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Build Docker image
      run: |
        docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
        docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:latest .

    - name: Run tests
      run: |
        docker run --rm ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} npm test

    - name: Scan image for vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Login to ACR
      run: |
        az acr login --name ${{ env.ACR_NAME }}

    - name: Push to ACR
      run: |
        docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
        docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:latest

  deploy-to-dev:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: development
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Get AKS credentials
      run: |
        az aks get-credentials \
          --resource-group ${{ env.RESOURCE_GROUP }} \
          --name ${{ env.AKS_CLUSTER }} \
          --overwrite-existing

    - name: Deploy to AKS
      run: |
        kubectl set image deployment/myapp \
          myapp=${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n development

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/myapp -n development --timeout=300s

    - name: Run smoke tests
      run: |
        # Add your smoke tests here
        curl -f http://myapp-dev.example.com/health || exit 1

  deploy-to-production:
    needs: deploy-to-dev
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Get AKS credentials
      run: |
        az aks get-credentials \
          --resource-group ${{ env.RESOURCE_GROUP }} \
          --name ${{ env.AKS_CLUSTER }} \
          --overwrite-existing

    - name: Deploy to Production
      run: |
        kubectl set image deployment/myapp \
          myapp=${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n production

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/myapp -n production --timeout=600s

    - name: Smoke test production
      run: |
        curl -f https://myapp.example.com/health || kubectl rollout undo deployment/myapp -n production
```

### 9.3: Azure DevOps Pipeline Example

**azure-pipelines.yml:**

```yaml
trigger:
  branches:
    include:
    - main
    - develop

variables:
  acrName: 'acrdemoacr'
  aksCluster: 'aks-demo-cluster'
  resourceGroup: 'rg-aks-demo'
  imageName: 'myapp'
  dockerfilePath: 'Dockerfile'

stages:
- stage: Build
  displayName: 'Build and Push'
  jobs:
  - job: BuildJob
    displayName: 'Build Docker Image'
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Docker@2
      displayName: 'Build Docker image'
      inputs:
        containerRegistry: 'ACR-ServiceConnection'
        repository: '$(imageName)'
        command: 'build'
        Dockerfile: '$(dockerfilePath)'
        tags: |
          $(Build.BuildId)
          latest

    - task: AzureCLI@2
      displayName: 'Scan image with Trivy'
      inputs:
        azureSubscription: 'Azure-ServiceConnection'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy image \
            $(acrName).azurecr.io/$(imageName):$(Build.BuildId)

    - task: Docker@2
      displayName: 'Push to ACR'
      inputs:
        containerRegistry: 'ACR-ServiceConnection'
        repository: '$(imageName)'
        command: 'push'
        tags: |
          $(Build.BuildId)
          latest

- stage: DeployDev
  displayName: 'Deploy to Dev'
  dependsOn: Build
  jobs:
  - deployment: DeployJob
    displayName: 'Deploy to AKS Dev'
    environment: 'development'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to AKS'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'AKS-ServiceConnection'
              namespace: 'development'
              manifests: '$(Pipeline.Workspace)/manifests/*.yaml'
              containers: '$(acrName).azurecr.io/$(imageName):$(Build.BuildId)'

- stage: DeployProd
  displayName: 'Deploy to Production'
  dependsOn: DeployDev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployJob
    displayName: 'Deploy to AKS Production'
    environment: 'production'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to AKS Production'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'AKS-ServiceConnection'
              namespace: 'production'
              manifests: '$(Pipeline.Workspace)/manifests/*.yaml'
              containers: '$(acrName).azurecr.io/$(imageName):$(Build.BuildId)'
```

### 9.4: GitOps with ArgoCD (Recommended)

```bash
# Install ArgoCD in AKS
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Create application
cat << EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/k8s-manifests
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

### 9.5: Infrastructure as Code

**Terraform Example:**

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "rg-aks-demo"
  location = "East US"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-demo-cluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aksdemodns"

  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_D2s_v3"
    zones      = ["1", "2", "3"]
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "azure"
  }
}

resource "azurerm_container_registry" "acr" {
  name                = "acrdemoacr"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  sku                 = "Standard"
  admin_enabled       = false
}

resource "azurerm_role_assignment" "aks_acr" {
  principal_id                     = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.acr.id
  skip_service_principal_aad_check = true
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}
```


---

## 10. Common Scenarios & Solutions

### 10.1: Scaling Scenarios

**Scenario 1: Traffic Spike Expected**

```bash
# Proactive scaling before expected traffic

# Scale AKS nodes
az aks nodepool scale \
  --resource-group rg-aks-demo \
  --cluster-name aks-demo-cluster \
  --name agentpool \
  --node-count 5

# Scale application pods
kubectl scale deployment myapp --replicas=10 -n production

# Verify scaling
kubectl get nodes
kubectl get pods -n production
```

**Scenario 2: Cost Reduction During Off-Hours**

```bash
# Automation script for off-hours scaling
#!/bin/bash

HOUR=$(date +%H)

if [ $HOUR -ge 18 ] || [ $HOUR -lt 6 ]; then
  # Off-hours: scale down
  echo "Scaling down for off-hours"
  az aks nodepool scale --name agentpool --node-count 1 \
    --resource-group rg-aks-demo --cluster-name aks-demo-cluster
  kubectl scale deployment myapp --replicas=2 -n development
else
  # Business hours: scale up
  echo "Scaling up for business hours"
  az aks nodepool scale --name agentpool --node-count 3 \
    --resource-group rg-aks-demo --cluster-name aks-demo-cluster
  kubectl scale deployment myapp --replicas=5 -n development
fi
```

### 10.2: Troubleshooting Scenarios

**Scenario 1: Application Not Accessible**

```bash
# Diagnosis workflow
echo "Step 1: Check pod status"
kubectl get pods -n production
kubectl describe pod <pod-name> -n production

echo "Step 2: Check service"
kubectl get svc -n production
kubectl describe svc myapp-service -n production

echo "Step 3: Check endpoints"
kubectl get endpoints myapp-service -n production

echo "Step 4: Check ingress/load balancer"
kubectl get ingress -n production
az network lb list --resource-group MC_rg-aks-demo_aks-demo-cluster_eastus

echo "Step 5: Check NSG rules"
az network nsg rule list \
  --resource-group MC_rg-aks-demo_aks-demo-cluster_eastus \
  --nsg-name <nsg-name> \
  --output table

echo "Step 6: Test from within cluster"
kubectl run curl-test --image=curlimages/curl -i --rm --restart=Never -- \
  curl -v http://myapp-service.production.svc.cluster.local
```

**Scenario 2: Pod CrashLoopBackOff**

```bash
# Diagnosis and resolution
echo "Check pod logs"
kubectl logs <pod-name> -n production
kubectl logs <pod-name> -n production --previous  # Previous container logs

echo "Check pod events"
kubectl describe pod <pod-name> -n production

echo "Common causes and fixes:"
echo "1. Application error - check logs"
echo "2. Missing ConfigMap/Secret - verify resources exist"
echo "3. Wrong image - check image tag in deployment"
echo "4. Resource limits too low - increase limits"
echo "5. Health check failing - adjust probe settings"

# Example fix: Increase memory limit
kubectl set resources deployment myapp \
  --limits=memory=512Mi \
  --requests=memory=256Mi \
  -n production
```

**Scenario 3: High Memory/CPU Usage**

```bash
# Identify resource hogs
echo "Check node resources"
kubectl top nodes

echo "Check pod resources"
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu

echo "Get detailed pod metrics"
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Solutions:
# 1. Horizontal scaling
kubectl scale deployment myapp --replicas=5 -n production

# 2. Vertical scaling (increase resources)
kubectl set resources deployment myapp \
  --limits=cpu=1000m,memory=1Gi \
  --requests=cpu=500m,memory=512Mi \
  -n production

# 3. Add node pool or scale existing
az aks nodepool scale \
  --resource-group rg-aks-demo \
  --cluster-name aks-demo-cluster \
  --name agentpool \
  --node-count 4

# 4. Enable cluster autoscaler
az aks update \
  --resource-group rg-aks-demo \
  --name aks-demo-cluster \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5
```

### 10.3: Security Incident Response

**Scenario: Suspected Security Breach**

```bash
# Immediate actions
echo "1. Isolate affected resources"
kubectl cordon <node-name>  # Prevent new pods
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

echo "2. Collect evidence"
# Get pod logs
kubectl logs <suspicious-pod> -n production > incident-logs-$(date +%Y%m%d-%H%M).txt

# Get events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' > incident-events-$(date +%Y%m%d-%H%M).txt

# Get current state
kubectl get all --all-namespaces -o yaml > incident-state-$(date +%Y%m%d-%H%M).yaml

echo "3. Check audit logs"
# Query Azure Monitor Logs
# KubeAuditReceivedLogs | where TimeGenerated > ago(24h)

echo "4. Rotate credentials"
# Rotate service principal secret
az ad sp credential reset --id <sp-id>

# Update Kubernetes secrets
kubectl create secret generic new-secret --from-literal=key=newvalue -n production --dry-run=client -o yaml | kubectl apply -f -

echo "5. Review access"
az role assignment list --all --output table > access-review-$(date +%Y%m%d).txt

echo "6. Patch vulnerabilities"
# Update base images
# Scan and update dependencies
# Apply security patches

echo "7. Restore from backup if necessary"
velero restore create --from-backup pre-incident-backup
```

### 10.4: Migration Scenarios

**Scenario: Migrate Application to New AKS Cluster**

```bash
# Step 1: Backup current cluster
echo "Creating backup..."
velero backup create migration-backup --wait

# Step 2: Create new AKS cluster
az aks create \
  --resource-group rg-aks-new \
  --name aks-new-cluster \
  --node-count 3 \
  --zones 1 2 3 \
  --attach-acr acrdemoacr

# Step 3: Set up Velero in new cluster
kubectl config use-context aks-new-cluster
# Install Velero (same as original cluster)

# Step 4: Restore to new cluster
velero restore create migration-restore \
  --from-backup migration-backup \
  --wait

# Step 5: Verify migration
kubectl get all --all-namespaces
kubectl get pv
kubectl get pvc

# Step 6: Update DNS/Load Balancer
# Point traffic to new cluster

# Step 7: Cutover
# 1. Set old cluster to read-only
# 2. Sync final data
# 3. Switch traffic
# 4. Monitor new cluster
# 5. Decommission old cluster after validation period
```

### 10.5: Compliance & Auditing

**Scenario: Prepare for Security Audit**

```bash
# Generate compliance report
echo "=== Azure Security Posture ==="

echo "1. Check Azure Policy compliance"
az policy state list \
  --resource-group rg-aks-demo \
  --query "[?complianceState=='NonCompliant'].{Policy:policyDefinitionName, Resource:resourceId}" \
  --output table

echo "2. Check Security Center recommendations"
az security assessment list \
  --query "[?status.code=='Unhealthy'].{Name:displayName, Severity:status.severity}" \
  --output table

echo "3. Review RBAC assignments"
az role assignment list \
  --resource-group rg-aks-demo \
  --include-inherited \
  --output table > rbac-audit-$(date +%Y%m%d).txt

echo "4. Check storage encryption"
az storage account show \
  --name staksdemostorage \
  --query "{Name:name, EncryptionServices:encryption.services}" \
  --output table

echo "5. Review NSG rules"
az network nsg rule list \
  --resource-group rg-aks-demo \
  --nsg-name nsg-jumpserver \
  --output table > nsg-audit-$(date +%Y%m%d).txt

echo "6. AKS security features"
az aks show \
  --resource-group rg-aks-demo \
  --name aks-demo-cluster \
  --query "{AADProfile:aadProfile.managed, NetworkPolicy:networkProfile.networkPolicy, PrivateCluster:apiServerAccessProfile.enablePrivateCluster}" \
  --output table

echo "7. Review diagnostic settings"
az monitor diagnostic-settings list \
  --resource $(az aks show -g rg-aks-demo -n aks-demo-cluster --query id -o tsv) \
  --output table

echo "=== Generate Compliance Report ==="
cat << EOF > compliance-report-$(date +%Y%m%d).md
# Security Compliance Report

## Generated: $(date)

## Infrastructure Security
- [x] Azure AD integration enabled
- [x] RBAC configured with least privilege
- [x] Network security groups configured
- [x] Encryption at rest enabled
- [x] TLS 1.2+ enforced
- [x] Audit logging enabled

## Container Security
- [x] ACR vulnerability scanning enabled
- [x] Non-root containers enforced
- [x] Resource limits configured
- [x] Network policies implemented

## Data Security
- [x] Storage account key access disabled
- [x] Azure AD authentication for blob storage
- [x] Soft delete enabled
- [x] Versioning enabled

## Monitoring & Auditing
- [x] Container Insights enabled
- [x] Diagnostic logging configured
- [x] Alert rules defined
- [x] Log retention configured (90 days)

## Recommendations
1. Enable private endpoints for production
2. Implement pod security policies
3. Regular security scans (weekly)
4. Quarterly DR testing
5. Annual access review

EOF

cat compliance-report-$(date +%Y%m%d).md
```

---

## Summary

This architecture guide provides:

✅ **Comprehensive Architecture**: Visual diagrams showing component interactions and data flows

✅ **Security Best Practices**: Multi-layered security approach with Azure AD, network security, and encryption

✅ **Cost Optimization**: Strategies to reduce costs while maintaining performance and reliability

✅ **High Availability**: Multi-zone deployment, geo-replication, and redundancy configurations

✅ **Monitoring**: Complete observability stack with metrics, logs, traces, and alerting

✅ **Disaster Recovery**: Backup strategies, recovery procedures, and RTO/RPO targets

✅ **DevOps Integration**: CI/CD pipelines, GitOps, and infrastructure as code examples

✅ **Troubleshooting Guides**: Common scenarios with step-by-step resolution procedures

### Next Actions

1. Review the architecture with your team
2. Customize configurations for your specific needs
3. Implement monitoring and alerting
4. Set up CI/CD pipelines
5. Test disaster recovery procedures
6. Document any custom configurations
7. Schedule regular security and compliance reviews

---

## Additional Resources

- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)
- [AKS Best Practices](https://docs.microsoft.com/azure/aks/best-practices)
- [Azure Security Benchmark](https://docs.microsoft.com/security/benchmark/azure/)
- [Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Azure Cost Management](https://docs.microsoft.com/azure/cost-management-billing/)
- [Azure Monitor Documentation](https://docs.microsoft.com/azure/azure-monitor/)

---

**Documentation Version**: 1.0  
**Last Updated**: June 21, 2026  
**Maintained By**: DevOps Team

For questions or updates to this documentation, please contact the DevOps team or submit a pull request.
