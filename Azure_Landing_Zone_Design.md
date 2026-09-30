# Azure Landing Zone Design
## Based on GCP Infrastructure Analysis

**Design Date:** September 2026  
**Based on:** dr-highradius-prod-cloud15 GCP Export  
**Target State:** Multi-subscription Enterprise Landing Zone

---

## Executive Summary

This Azure landing zone design translates the analyzed GCP infrastructure (10 VPCs, 20 databases, 8 Kubernetes clusters, 56 firewall rules) into an enterprise-ready Azure environment following Microsoft Cloud Adoption Framework (CAF) best practices.

**Key Design Principles:**
- Hub-and-spoke network topology for centralized control
- Multi-subscription strategy for isolation and governance
- Zero-trust security architecture
- Compliance and audit readiness
- Cost optimization and resource governance

---

## 1. LANDING ZONE ARCHITECTURE OVERVIEW

### 1.1 Core Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    TENANT ROOT GROUP                        │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
    ┌───▼────────┐   ┌────────▼────────┐   ┌──────▼──────┐
    │  Platform  │   │   Workload      │   │  Sandbox    │
    │   Subs     │   │   Subscriptions │   │  (Non-Prod) │
    └────────────┘   └─────────────────┘   └─────────────┘
```

### 1.2 Management Groups Hierarchy

```
Tenant Root
├── Platform
│   ├── Management
│   ├── Connectivity
│   └── Identity
├── Landing Zones
│   ├── Production
│   │   ├── Critical Workloads
│   │   ├── Standard Workloads
│   │   └── Data & Analytics
│   ├── Non-Production
│   │   ├── Development
│   │   └── Testing
│   └── Decommissioned
└── Sandbox
    └── Innovation Lab
```

---

## 2. SUBSCRIPTION STRATEGY

### 2.1 Recommended Subscription Allocation

Based on GCP's 10 VPCs and 8 different workload clusters, recommend **12-15 subscriptions**:

| Subscription | Purpose | Workloads | GCP Equivalent |
|---|---|---|---|
| **Platform Management** | Central management, governance, monitoring | Azure Policy, Blueprints | - |
| **Connectivity Hub** | Hub network, firewalls, VPN/ExpressRoute | Network hub, routers | Cloud Router network |
| **Identity & Access** | Azure AD, KeyVault, privileged access | Service principals | Service accounts |
| **Production-Kubernetes** | AKS clusters for production | 4-5 GKE clusters | dr-core-prod, dr-rivana, etc. |
| **Production-Data** | Databases, data warehouses, backups | 20 Cloud SQL + replicas | Cloud SQL instances |
| **Production-App** | Application servers, services | Compute instances | Compute instances |
| **DR/Backup** | Disaster recovery, backup vaults | Snapshots, backups | Cloud snapshots |
| **Development** | Dev/test environment | Dev instances, test DBs | Dev GKE clusters |
| **Monitoring & Logging** | Centralized observability | Log Analytics, Application Insights | Cloud Logging/Monitoring |
| **Security & Compliance** | Security tools, policy enforcement | Sentinel, Defender | Firewall rules analysis |

### 2.2 Subscription Naming Convention

```
Format: <environment>-<workload>-<region>-<sequence>

Examples:
- prod-kubernetes-eastus-001
- prod-data-eastus-001
- prod-app-westeurope-001
- dev-general-eastus-001
- platform-management-eastus-001
- platform-connectivity-eastus-001
```

---

## 3. NETWORK ARCHITECTURE

### 3.1 Hub-and-Spoke Topology

Maps GCP's multi-VPC peering to Azure's hub-spoke model:

```
                    ┌──────────────────┐
                    │   HUB VNet       │
                    │  10.0.0.0/16     │
                    │                  │
                    │ ┌──────────────┐ │
                    │ │  Firewall    │ │
                    │ │  (Premium)   │ │
                    │ └──────────────┘ │
                    │                  │
                    │ ┌──────────────┐ │
                    │ │  Gateway Sub │ │
                    │ │  VPN/ER      │ │
                    │ └──────────────┘ │
                    └──────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼────┐       ┌────▼────┐      ┌───▼─────┐
    │ SPOKE 1 │       │ SPOKE 2 │      │ SPOKE 3 │
    │ Apps    │       │ Data    │      │ Infra   │
    │ VNet    │       │ VNet    │      │ VNet    │
    └─────────┘       └─────────┘      └─────────┘
    10.1.0.0/24    10.2.0.0/24      10.3.0.0/24
```

### 3.2 Network Design Specifications

#### Hub Virtual Network
```yaml
Name: hub-vnet-eastus
Address Space: 10.0.0.0/16
Subnets:
  - AzureFirewall: 10.0.1.0/26
  - Gateway: 10.0.2.0/26 (VPN/ExpressRoute)
  - Management: 10.0.3.0/26 (Bastion, Management tools)
  - SharedServices: 10.0.4.0/25 (DNS, monitoring)
```

#### Spoke VNets (Based on GCP Workload Separation)

**Spoke 1: Kubernetes Production (Maps to 4-5 GKE Clusters)**
```yaml
Name: prod-aks-vnet-eastus
Address Space: 10.1.0.0/16
Subnets:
  - AKS-Cluster-1: 10.1.1.0/24 (500+ node IPs)
  - AKS-Cluster-2: 10.1.2.0/24
  - AKS-Cluster-3: 10.1.3.0/24
  - AKS-Cluster-4: 10.1.4.0/24
  - Private-Endpoints: 10.1.5.0/24 (PaaS connections)
Network Policies: Enabled (equivalents to firewall rules 03a)
```

**Spoke 2: Database & Data (Maps to 20 Cloud SQL Instances)**
```yaml
Name: prod-data-vnet-eastus
Address Space: 10.2.0.0/16
Subnets:
  - SQLDatabase: 10.2.1.0/24 (Managed services)
  - DataLake: 10.2.2.0/24 (Storage endpoints)
  - Analytics: 10.2.3.0/24 (Synapse, Analytics)
  - Backup: 10.2.4.0/24 (Recovery Services Vault)
Delegation: Microsoft.Sql/servers, Microsoft.DBforPostgreSQL/servers
```

**Spoke 3: Application Services (Maps to 16 Compute Instances)**
```yaml
Name: prod-app-vnet-eastus
Address Space: 10.3.0.0/16
Subnets:
  - AppServers: 10.3.1.0/24
  - Middleware: 10.3.2.0/24
  - LoadBalancer: 10.3.3.0/26
  - Batch-Processing: 10.3.4.0/24
```

**Spoke 4: Development (Maps to Dev GKE Clusters)**
```yaml
Name: dev-workload-vnet-eastus
Address Space: 10.4.0.0/16
Subnets:
  - Dev-AKS: 10.4.1.0/24
  - Dev-Apps: 10.4.2.0/24
  - Dev-Data: 10.4.3.0/24
Lower security policies, faster change deployment
```

**Spoke 5: Hybrid Connectivity & DR (Maps to VPN/Interconnect)**
```yaml
Name: hybrid-vnet-eastus
Address Space: 10.5.0.0/16
Subnets:
  - VPN-Gateway: 10.5.1.0/27
  - ExpressRoute-Gateway: 10.5.2.0/27
  - Replication: 10.5.3.0/24 (Site Recovery)
```

### 3.3 Peering Configuration

All spokes peer to hub with:
- **Allow Gateway Transit:** Enabled on hub
- **Use Remote Gateways:** Enabled on spokes
- **Allow VNet Access:** Enabled on both
- **Allow Forwarded Traffic:** Enabled on hub

```
Hub ←→ Spoke1 (AKS)
Hub ←→ Spoke2 (Data)
Hub ←→ Spoke3 (Apps)
Hub ←→ Spoke4 (Dev)
Hub ←→ Spoke5 (Hybrid)
```

### 3.4 Subnet Allocation Summary

**Total VNet Space:** 10.0.0.0/16 - 10.5.0.0/16  
**Hub:** 10.0.0.0/16 (256 /24 networks)  
**Spoke1 (AKS):** 10.1.0.0/16 (256 /24 networks)  
**Spoke2 (Data):** 10.2.0.0/16 (256 /24 networks)  
**Spoke3 (Apps):** 10.3.0.0/16 (256 /24 networks)  
**Spoke4 (Dev):** 10.4.0.0/16 (256 /24 networks)  
**Spoke5 (Hybrid):** 10.5.0.0/16 (256 /24 networks)  

**Extensibility:** Designed for multi-region expansion

---

## 4. SECURITY & NETWORK SEGMENTATION

### 4.1 Azure Firewall Configuration

Maps GCP's 56 firewall rules to Azure Firewall Premium:

```
Azure Firewall Premium (Hub)
├── Network Rules (Maps to 03a-ingress-rules)
│   ├── Allow SSH (22) - from corp CIDR
│   ├── Allow RDP (3389) - from corp CIDR
│   ├── Allow HTTP (80) - from 0.0.0.0/0 (with TLS inspection)
│   ├── Allow HTTPS (443) - from 0.0.0.0/0
│   └── Allow custom ports (3306, 9200, 6443, 8080)
│
├── Application Rules
│   ├── SQL Database outbound
│   ├── Storage account access
│   ├── Managed identity flows
│   └── Third-party API access
│
├── NAT Rules
│   └── Maps to 6 GCP Cloud Router NAT configs
│
└── Threat Intelligence
    ├── Alert mode: High/Medium severity
    └── Deny mode: Critical threats only
```

### 4.2 Network Security Groups (NSGs)

Layered approach matching GCP's firewall rules depth:

**Layer 1: Subnet NSG (Hub)**
```
Inbound:
  - Allow VNet-to-VNet (10.0.0.0/8)
  - Deny internet by default
  - Allow specific services
  
Outbound:
  - Allow all to Azure services
  - Allow controlled internet (via firewall)
```

**Layer 2: Application NSG (Spokes)**
```
Prod-AKS NSG:
  Inbound:
    - Allow from hub firewall
    - Allow inter-pod (10.1.0.0/16)
    - Allow Health checks (168.63.129.16)
  Outbound:
    - Controlled via firewall

Data NSG:
  Inbound:
    - Allow only from app tier
    - Block cross-spoke
  Outbound:
    - Restricted to Azure services
```

### 4.3 Web Application Firewall (WAF)

For public-facing services (maps to HTTP/HTTPS rules from 03e):

```
Application Gateway + WAF Premium
├── Public IP pool
├── Backend pools
│   ├── AKS ingress
│   └── App servers
├── Rules
│   ├── OWASP Top 10
│   ├── Bot protection
│   └── DDoS protection
└── Logging to Log Analytics
```

---

## 5. SERVICE MAPPING: GCP → AZURE

### 5.1 Compute Services

| GCP Service | Azure Equivalent | Landing Zone Location |
|---|---|---|
| Compute Instances | Virtual Machines | App Spoke |
| GKE Cluster (8 clusters) | Azure Kubernetes Service (AKS) | AKS Spoke (4-5 clusters) |
| Instance Templates | VM Scale Sets / Images | Shared Image Gallery |
| Managed Instance Groups | Virtual Machine Scale Sets | AKS/App subnet |
| Cloud Functions | Azure Functions | Shared Services |

### 5.2 Database Services

| GCP Service | Azure Equivalent | Landing Zone Location |
|---|---|---|
| Cloud SQL (20 instances) | Azure SQL Database + Azure Database for PostgreSQL/MySQL | Data Spoke |
| Cloud SQL Replicas | Read Replicas / Geo-replication | Multi-region setup |
| Firestore | Azure Cosmos DB | Data Spoke |
| Cloud Datastore | Azure Cosmos DB | Data Spoke |

### 5.3 Networking Services

| GCP Service | Azure Equivalent | Landing Zone Location |
|---|---|---|
| VPC (10 networks) | Virtual Networks | Hub-Spoke model |
| VPC Peering (19) | VNet Peering | Hub to spokes |
| Cloud Router (6) | Azure Route Server | Hub gateway subnet |
| Cloud NAT | Azure NAT Gateway | Hub network |
| Cloud VPN | VPN Gateway | Hub gateway subnet |
| Cloud Interconnect | ExpressRoute | Hub gateway subnet |
| Cloud DNS | Azure DNS (Public & Private) | Hub SharedServices |

### 5.4 Security Services

| GCP Service | Azure Equivalent | Landing Zone Location |
|---|---|---|
| Firewall Rules (56) | Azure Firewall + NSGs | Hub |
| Cloud Armor | Azure DDoS Protection + WAF | Public endpoints |
| Cloud KMS | Azure Key Vault | Management subscription |
| Service Accounts (29) | Service Principals + Managed Identities | Identity subscription |
| VPC Service Controls | Azure Policy + Conditional Access | Management |

### 5.5 Monitoring & Management

| GCP Service | Azure Equivalent | Landing Zone Location |
|---|---|---|
| Cloud Logging | Azure Monitor / Log Analytics | Monitoring subscription |
| Cloud Monitoring | Azure Monitor / Application Insights | Monitoring subscription |
| Cloud Audit Logs | Azure Activity Log + Sentinel | Security subscription |

---

## 6. IDENTITY & ACCESS MANAGEMENT

### 6.1 Service Principal Strategy

Maps GCP's 29 service accounts to Azure Service Principals with Managed Identities:

**Organizational Structure:**
```
Azure AD
├── Workload Groups
│   ├── AKS Cluster Group
│   ├── Database Access Group
│   ├── App Service Group
│   ├── Automation Group
│   └── Third-party Integration Group
├── Administrative Groups
│   ├── Subscription Owners
│   ├── Network Administrators
│   ├── Security Team
│   ├── DBA Group
│   └── Kubernetes Administrators
└── Application Groups
    ├── 8 Service Groups (one per former GKE cluster)
    └── Integration Service Accounts
```

### 6.2 Managed Identity Usage

**System-Assigned Identities:**
```
- Each AKS cluster → System identity for kubelet
- Each VM → System identity for VM agent
- Each App Service → System identity
- Each Function App → System identity
```

**User-Assigned Identities:**
```
- Database access SP
- Storage account access SP
- Key Vault access SP
- Monitoring/logging SP
- Backup/DR SP
```

### 6.3 RBAC Role Assignments

**Subscription-Level Roles:**
```
Hub Connectivity Sub:
  - Network Contributors: Network team
  - Firewall Admins: Security team
  - Reader: All operations teams

Prod-AKS Sub:
  - AKS Cluster User: Dev team
  - AKS Cluster Admin: Platform team
  - Contributor: Kubernetes admins

Prod-Data Sub:
  - SQL DB Contributor: DBA team
  - Backup Operator: DBA team
  - Reader: App teams
```

---

## 7. FIREWALL RULES CONVERSION

### 7.1 Azure Firewall Rules (from 03a-ingress-rules-open-ports.json)

**High-Risk Ports Rule Set:**
```yaml
Name: Allow-High-Risk-Services
Priority: 1000
Rules:
  - Name: Allow-SSH
    Source: 10.0.0.0/8 (internal only)
    Destination: 10.1.0.0/16, 10.3.0.0/16
    Protocol: TCP:22
    Action: Allow
    
  - Name: Allow-Kubernetes-API
    Source: 10.4.0.0/16 (dev)
    Destination: 10.1.0.0/16 (AKS)
    Protocol: TCP:6443
    Action: Allow
    
  - Name: Allow-SQL
    Source: 10.1.0.0/16, 10.3.0.0/16
    Destination: 10.2.0.0/16
    Protocol: TCP:3306,5432
    Action: Allow
    
  - Name: Allow-Elasticsearch
    Source: 10.1.0.0/16, 10.3.0.0/16
    Destination: 10.2.1.0/24
    Protocol: TCP:9200,9300
    Action: Allow
```

**Internet-Facing Rules:**
```yaml
Name: Allow-Public-Web
Priority: 100
Rules:
  - Name: Allow-HTTP-HTTPS-AppGW
    Source: 0.0.0.0/0
    Destination: 10.3.3.0/26 (AppGW)
    Protocol: TCP:80,443
    Action: Allow
    
  - Name: Block-Other-Internet
    Source: 0.0.0.0/0
    Destination: All
    Protocol: All
    Action: Deny
```

**Outbound Rules:**
```yaml
Name: Controlled-Egress
Rules:
  - Name: Allow-Azure-Services
    Source: All spokes
    Destination: AzureCloud
    Protocol: TCP:443
    Action: Allow
    
  - Name: Allow-Corporate-Servers
    Source: All spokes
    Destination: 192.168.0.0/16 (on-prem)
    Protocol: TCP:443,3306,5432
    Action: Allow
    
  - Name: Deny-Unknown-Internet
    Source: All spokes
    Destination: 0.0.0.0/0
    Protocol: All
    Action: Deny
```

### 7.2 Network Security Group Rules

Applied at subnet level as secondary enforcement:

```yaml
Kubernetes-Subnet NSG:
  Inbound:
    - Allow pod-to-pod: 10.1.0.0/16
    - Allow from LoadBalancer: 10.3.3.0/26
    - Allow AKS Health probe: 168.63.129.16
    - DenyAll: 0.0.0.0/0
    
  Outbound:
    - Allow to firewall: 10.0.0.0/16
    - Allow to DNS: 168.63.129.16
    - Allow pod egress: 0.0.0.0/0 (controlled by firewall)
```

---

## 8. DISASTER RECOVERY & BACKUP STRATEGY

### 8.1 Backup Architecture

Maps GCP's 56 snapshots to Azure backup strategy:

```
Recovery Services Vault
├── Azure Backup (VMs & file shares)
│   └── Daily backups: 16 app VMs
├── Azure SQL Backup
│   ├── Automated backups: 20 SQL DBs
│   ├── Geo-replication: Primary + secondary regions
│   └── Point-in-time restore: 35 days
├── AKS Node Image Backup
│   └── VM snapshots of node pools
└── Application Data
    └── App Insights snapshots
```

### 8.2 Disaster Recovery Plan

**RTO/RPO Targets:**
```
Production-Critical (AKS + SQL):
  RTO: 1 hour
  RPO: 15 minutes
  
Production-Standard (App servers):
  RTO: 4 hours
  RPO: 1 hour
  
Development:
  RTO: 8 hours
  RPO: Best effort
```

**Multi-Region Setup:**
```
Primary Region: East US (main production)
Secondary Region: West Europe (DR site)

Failover Setup:
- SQL: Geo-replication configured
- AKS: Manual failover (cluster recreated from IaC)
- Apps: Traffic Manager for failover
- Data: Geo-redundant storage (GRS)
```

---

## 9. GOVERNANCE & POLICY

### 9.1 Azure Policy Framework

Applied at management group level:

```
Landing Zone Policies:
├── Network Security
│   ├── Require NSG on all subnets
│   ├── Require Network Watcher
│   ├── Deny public IPs except app gateways
│   └── Require private endpoints for PaaS
│
├── Encryption
│   ├── Require encryption at rest
│   ├── Require TLS in transit
│   ├── Require disk encryption
│   └── Require Key Vault for secrets
│
├── Logging & Monitoring
│   ├── Require diagnostic settings
│   ├── Require Log Analytics agent
│   ├── Require application insights
│   └── Centralize to hub Log Analytics
│
├── Identity
│   ├── Require MFA for privileged roles
│   ├── Require service principals for automation
│   ├── Require Azure AD integration
│   └── Deny legacy auth
│
└── Compliance
    ├── Allowed resource types
    ├── Allowed locations
    ├── Cost management policies
    └── Tag enforcement
```

### 9.2 Resource Tagging Strategy

```yaml
Required Tags (enforced by policy):
  Environment: prod | dev | test | dr
  CostCenter: Finance code
  Owner: Team/person responsible
  Application: Application name
  DataClassification: Public | Internal | Confidential | Restricted
  BackupPolicy: Daily | Weekly | None
  DisasterRecovery: Critical | Standard | NonCritical
  
Example VM Tags:
  Environment: prod
  CostCenter: 12345
  Owner: platform-team
  Application: kubernetes
  DataClassification: Internal
  BackupPolicy: Daily
  DisasterRecovery: Critical
```

### 9.3 Compliance & Audit

```
Azure Policy Compliance Dashboard
├── PCI-DSS (for payment systems)
├── HIPAA (if healthcare data)
├── SOC2 Type II
├── ISO 27001
└── Custom organizational policies

Audit Trail:
  - Azure Activity Log (all control plane)
  - Sentinel (SIEM integration)
  - Log Analytics (all resource logs)
  - Change Tracking (IaC-based)
```

---

## 10. MONITORING & OBSERVABILITY

### 10.1 Monitoring Stack

```
Azure Monitor Hub (Monitoring Subscription)
├── Log Analytics Workspace
│   ├── All platform logs
│   ├── All workload logs
│   ├── Network flow logs
│   ├── Firewall logs
│   └── SQL Audit logs
│
├── Application Insights
│   ├── AKS cluster monitoring
│   ├── Application performance
│   ├── Custom metrics
│   └── Dependency tracking
│
├── Azure Metrics
│   ├── Resource utilization
│   ├── Network bandwidth
│   ├── Database performance
│   └── Firewall throughput
│
└── Alerts & Actions
    ├── High CPU utilization
    ├── Network saturation
    ├── Failed backups
    ├── Security alerts
    └── Budget alerts
```

### 10.2 Logging Configuration

**Network Logs (replaces GCP Cloud Logging):**
```
Azure Firewall Logs:
  - Network rule logs
  - Application rule logs
  - NAT rule logs
  - Threat Intelligence logs
  Destination: Log Analytics → Sentinel

NSG Flow Logs:
  - All subnet NSGs
  - Enabled on all spoke subnets
  - 5-minute flow data
  - Destination: Storage account → Log Analytics

VNet Peering Logs:
  - Cross-subscription traffic
  - Diagnostic logging enabled
  - Destination: Log Analytics
```

**Application Logs:**
```
AKS Cluster Logs:
  - Control plane logs
  - Node logs
  - Container logs
  - Audit logs
  Destination: Log Analytics

SQL Database Audit:
  - Connection logs
  - Query audit
  - Security events
  Destination: Log Analytics + Storage archive
```

### 10.3 Alerting Strategy

```
Alert Rules (replaces GCP monitoring):
├── Infrastructure Alerts
│   ├── High CPU: >85% for 10 minutes
│   ├── High Memory: >90% for 5 minutes
│   ├── Disk usage: >85%
│   ├── Network latency: >100ms
│   └── Packet loss: >1%
│
├── Security Alerts
│   ├── Firewall blocks from high-risk countries
│   ├── Failed login attempts: >5 in 10min
│   ├── Privilege escalation attempts
│   └── Unusual data access patterns
│
├── Application Alerts
│   ├── Error rate: >1%
│   ├── Response time: >2s
│   ├── Failed requests: >10/min
│   └── Database connection pool exhaustion
│
└── Business Alerts
    ├── Failed backups
    ├── SLA violations
    ├── Cost threshold exceeded
    └── Capacity warnings
```

---

## 11. COST MANAGEMENT

### 11.1 Cost Allocation Structure

```
Cost Center Assignment:
├── Hub & Connectivity
│   ├── Firewall: Shared cost
│   ├── VPN/ExpressRoute: Shared cost
│   └── Log Analytics: Shared cost
│
├── Workload Subscriptions
│   ├── AKS: Charged to platform team
│   ├── SQL: Charged to DBA team
│   ├── Apps: Charged to app teams
│   └── Dev: Charged to dev team
│
└── Service Subscriptions
    ├── Backup: Shared cost
    ├── Monitoring: Shared cost
    └── Security: Shared cost
```

### 11.2 Cost Optimization

**Reserved Instances:**
```
- AKS node pools: 3-year RIs for 80% of stable capacity
- SQL Databases: RI for production databases
- VMs: RI for always-on app servers
- Estimated savings: 30-40% on compute
```

**Right-Sizing:**
```
Weekly Review:
  - VM utilization vs allocated size
  - Database compute tier vs usage
  - Storage tier analysis
  - Unused IP addresses
  
Monthly Review:
  - Cost by team/application
  - Unused resources
  - Duplicate resources
  - Service redundancy
```

**Budget Alerts:**
```
Subscriptions: Set to 80% of monthly budget
Resource Groups: Set to specific cost centers
Alerts to cost owners when:
  - Monthly spending increases 10% YoY
  - Projected monthly > budget
  - Unused resources detected
```

---

## 12. IMPLEMENTATION ROADMAP

### Phase 1: Foundation (Weeks 1-4)
```
Week 1: Planning & Approval
  - Validate design with stakeholders
  - Prepare IAM structure
  - Set up DevOps pipeline
  
Week 2-3: Infrastructure
  - Create management groups
  - Create hub + first spoke VNet
  - Deploy Azure Firewall
  - Configure peering
  
Week 4: Governance
  - Implement Azure Policy
  - Set up RBAC
  - Configure tagging
  - Enable monitoring
```

### Phase 2: Workload Migration (Weeks 5-8)
```
Week 5: AKS Migration
  - Deploy 1st AKS cluster
  - Migrate 1 GKE workload
  - Test network connectivity
  
Week 6: Database Migration
  - Deploy Azure SQL
  - Migrate 2-3 Cloud SQL instances
  - Configure replication & backup
  
Week 7: App Server Migration
  - Deploy VMs for app servers
  - Migrate 2-3 instances
  - Configure load balancing
  
Week 8: Validation
  - Performance testing
  - Security assessment
  - Failover testing
```

### Phase 3: Optimization (Weeks 9-12)
```
Week 9-10: Tuning
  - Network optimization
  - Cost optimization
  - Security hardening
  
Week 11: Documentation
  - Architecture diagrams
  - Runbooks
  - Disaster recovery procedures
  
Week 12: Knowledge Transfer
  - Team training
  - Documentation review
  - Go-live readiness
```

---

## 13. DETAILED AZURE SERVICES

### 13.1 AKS Cluster Configuration

**Cluster 1: Production General (Maps to dr-core-prod)**
```yaml
Cluster Name: prod-aks-core-eastus
Resource Group: prod-kubernetes-eastus-001
VNet/Subnet: prod-aks-vnet / AKS-Cluster-1
VM SKU: Standard_D4s_v3 (4 vCPU, 16GB RAM)
Node Count: 5 nodes (min 3, max 20)
K8s Version: 1.28.x (latest stable)

Network:
  - Network Plugin: Azure CNI
  - Service CIDR: 10.100.0.0/16
  - DNS Service IP: 10.100.0.10
  - Network Policy: Azure (replaces GCP Firewall)
  
Addons:
  - HTTP Application Routing: Enabled
  - Azure Policy: Enabled
  - Azure Monitor: Enabled
  - Secrets Store CSI: Enabled
  
RBAC:
  - Enabled: True
  - Managed identity: Enabled
  
Storage:
  - Default storage class: managed-premium (SSD)
  - Backup: Daily snapshots
```

**Cluster 2: Production Critical (Maps to dr-rivana)**
```yaml
Cluster Name: prod-aks-critical-eastus
VM SKU: Standard_E8s_v3 (8 vCPU, 64GB RAM)
Node Count: 8 nodes (min 5, max 30)
Availability Zones: All 3 zones (zonal)
SLA: 99.9% uptime
Workload: Mission-critical services
```

**Cluster 3-5: Additional Production Clusters**
```
Cluster 3: prod-aks-batch-eastus (batch processing)
  - VM SKU: Compute-optimized
  - Auto-scaling: 1-50 nodes
  
Cluster 4: prod-aks-gpu-eastus (ML/Analytics)
  - VM SKU: GPU-enabled (NC series)
  - Spot VMs: Enabled for cost optimization
  
Cluster 5: prod-aks-infra-eastus (platform services)
  - Node pools: Linux + Windows
  - Pod security policy: Restricted
```

### 13.2 Azure SQL Configuration

**Primary Database (Maps to Cloud SQL primary instances)**
```yaml
Name: prod-db-primary-eastus
Resource Group: prod-data-eastus-001
Database Type: Azure SQL Database
Tier: Premium (DTU model)
Compute: P4 (200 DTUs)
Storage: 250GB
Backup: Geo-redundant (14-day retention)
Geo-replication: West Europe (active)
High Availability: Zone-redundant deployment

Databases:
  - app_prod: 50GB
  - analytics: 100GB
  - reporting: 50GB
  - archive: 50GB
  
Networking:
  - Private Endpoint: Yes (10.2.1.0/26)
  - VNet Rules: App subnet only
  - Public Access: Disabled
  
Security:
  - Transparent Data Encryption: Enabled
  - Azure Defender: Enabled
  - Audit: All actions logged
  - Compliance: SOC2, ISO27001
```

**Secondary Databases (Replicas)**
```yaml
Name: prod-db-replica-westeu
Geo-replication: Secondary (read-only)
Auto-failover: Enabled with 2-hour RPO
Failover Group: prod-db-failover-group

Replica Databases:
  - Replica 1: Secondary for primary app DB
  - Replica 2: Secondary for analytics DB
  - Replicas 3-10: For other Cloud SQL instances
```

### 13.3 Load Balancing

**Internal Load Balancer (App-to-DB)**
```yaml
Name: ilb-app-to-db
Type: Internal
Region: East US
VNet: prod-app-vnet
Subnet: LoadBalancer (10.3.3.0/26)
SKU: Standard
Session persistence: Client IP
Health probe: TCP:3306 (SQL) or TCP:5432 (PostgreSQL)
```

**Application Gateway (Public-facing)**
```yaml
Name: appgw-web-eastus
Type: Application Gateway v2
Tier: WAF v2 (Premium)
Capacity: Auto-scaling (2-10 instances)
Public IP: Static

Backend Pools:
  - AKS Ingress: 10.1.x.x (multiple pods)
  - App Servers: 10.3.1.0/24 (IIS/Apache)

Listeners:
  - HTTPS (443): Default
  - HTTP (80): Redirects to HTTPS

Rules:
  - Path-based routing
  - Host-based routing
  - WAF rules: OWASP Top 10
```

**Traffic Manager (Multi-region)**
```yaml
Name: tm-prod-app-global
DNS Name: prodapp.trafficmanager.net
Routing Policy: Weighted (90% primary, 10% secondary)

Endpoints:
  - Primary: App GW in East US
  - Secondary: App GW in West Europe (DR)

Health Check: HTTP GET every 10 seconds
```

---

## 14. SECURITY ARCHITECTURE

### 14.1 Defense in Depth

**Layer 1: DDoS Protection**
```
Azure DDoS Protection Standard
├── Layer 3/4 (Network)
├── Layer 7 (Application via WAF)
├── Automatic attack mitigation
└── DDoS Protection alerts → Sentinel
```

**Layer 2: Azure Firewall**
```
Premium Firewall (in hub)
├── HTTPS inspection
├── URL filtering
├── Threat intelligence
├── TLS/SSL termination
└── IP reputation
```

**Layer 3: Network Security Groups**
```
Multiple NSG rules
├── Subnet-level NSGs
├── NIC-level NSGs
├── Resource-level restrictions
└── Implicit deny default
```

**Layer 4: Azure Bastion**
```
Bastion Host (in hub management subnet)
├── RDP/SSH without public IPs
├── VM access logging
├── Session recording
└── MFA enforcement
```

### 14.2 Secret Management

**Azure Key Vault (in Management subscription)**
```yaml
Vault Name: kv-prod-eastus
Tier: Premium (HSM-backed)

Secrets:
  - Database connection strings
  - API keys
  - Certificates
  - Credentials for automation
  
Access:
  - RBAC for users
  - Managed identities for services
  - Private endpoint for network access
  - Audit logging enabled
  
Rotation:
  - Key rotation: Annual
  - Certificate renewal: 30 days before expiry
  - Secret rotation: Application-dependent
```

### 14.3 Azure Sentinel (SIEM/SOAR)

```
Log Sources:
├── Azure Firewall logs
├── NSG flow logs
├── SQL Audit logs
├── Application Insights
├── Azure Activity Logs
├── Azure AD logs
└── Third-party integrations (Palo Alto, Checkpoint, etc.)

Detections:
├── Firewall anomalies
├── Brute force attacks
├── Privilege escalation
├── Data exfiltration
├── Malware signatures
└── Custom rules based on GCP patterns

Playbooks (Automated Response):
├── Block malicious IP
├── Isolate compromised VM
├── Alert security team
├── Create incident ticket
└── Initiate incident response
```

---

## 15. HYBRID CONNECTIVITY

### 15.1 VPN Configuration (Maps to GCP Cloud VPN)

**Site-to-Site VPN**
```yaml
VPN Gateway: vpngw-hub-eastus
Type: Route-based
SKU: VpnGw2 (1.25 Gbps)

Local Network Gateways:
  - On-premises HQ: 192.168.0.0/16
  - DR site: 192.168.100.0/16
  - Partner network: 172.16.0.0/12

Connections:
  - Site 1 (HQ): 1 primary + 1 redundant
  - Site 2 (DR): 1 primary connection
  - Partner: 1 connection

Throughput: 1-2 Gbps (shared)
Encryption: IPsec (AES-256)
High Availability: Active-active preferred
```

**Point-to-Site VPN (for remote users)**
```yaml
VPN Client Configuration:
  - Protocol: OpenVPN or SSTP
  - Address pool: 192.168.1.0/24
  - MFA: Required via Azure AD
  - Client certificate: Optional
  - Data encryption: AES-128
  
Access:
  - Maximum clients: 1000
  - Bandwidth: Up to 100 Mbps
  - Automatic reconnect: Enabled
```

### 15.2 ExpressRoute (Maps to GCP Cloud Interconnect)

**ExpressRoute Circuit**
```yaml
Bandwidth: 10 Gbps
Peering Location: New York / Amsterdam
Service Provider: Equinix / Megaport

Peerings:
  - Microsoft Peering: For Office 365, Azure
  - Azure Private Peering: For VNet access
  - BGP: Redundant paths, failover < 1 second

Circuit Status: 99.9% SLA
Monthly cost: ~$5,000-10,000 (10 Gbps)
```

**Failover Configuration**
```
Primary: ExpressRoute (10 Gbps)
Secondary: VPN Gateway (1.25 Gbps)
Tertiary: Internet (if all else fails)

Routing:
  - ExpressRoute: Preferred (lowest cost)
  - BGP weight: 100
  - VPN: Backup (weight 200)
  - Internet: Last resort (weight 300)
```

---

## 16. DISASTER RECOVERY

### 16.1 Backup Strategy

**VM Backup (16 app VMs)**
```yaml
Backup Vault: backup-vault-eastus
Policy: prod-daily-backup
Frequency: Daily at 2 AM UTC
Retention: 30 days
Geo-redundant: Yes (replicated to West Europe)

Restoration Time:
  - Disk restoration: 5 minutes
  - Full VM restore: 15 minutes
  - Restore to alternate location: 20 minutes
```

**Database Backup (20 SQL instances)**
```yaml
Automatic Backups:
  - Full backup: Weekly (Sunday)
  - Differential backup: Daily
  - Log backup: Every 5 minutes
  
Retention:
  - Short-term: 35 days
  - Long-term: 7 years (via policies)
  
Point-in-Time Restore:
  - RPO: 5 minutes
  - Any point within retention window
```

**AKS Persistent Data**
```yaml
Persistent Volumes:
  - Storage Class: managed-premium
  - Backup: Daily snapshots to Azure Storage
  - Retention: 30-day rolling window

State Backup:
  - etcd cluster: Backed up hourly
  - Cluster config: Version-controlled in Git
  - YAML manifests: GitOps repository
```

### 16.2 Failover Procedures

**AKS Cluster Failover**
```
RPO: 1 hour (cluster from IaC)
RTO: 30 minutes

Steps:
1. Create new AKS cluster in DR region from IaC
2. Restore persistent volumes from snapshots
3. Deploy applications from GitOps
4. Update DNS/Traffic Manager
5. Validate application health
6. Cut over traffic

Automated: 80% (IaC + Helm charts)
Manual: 20% (validation & cutover)
```

**Database Failover**
```
RPO: 15 minutes (geo-replication lag)
RTO: 5 minutes (automatic failover)

Automatic:
- Geo-replication active
- Failover group configured
- Health probe monitoring
- Automatic failover on primary failure

Manual:
- If automatic fails
- During planned maintenance
- For testing purposes
```

---

## 17. COMPLIANCE & GOVERNANCE

### 17.1 Regulatory Requirements

```
Compliance Frameworks:
├── SOC2 Type II (for cloud services)
├── ISO 27001 (information security)
├── PCI-DSS (if payment processing)
├── GDPR (if EU customer data)
├── HIPAA (if healthcare data)
└── NIST (if government contracts)

Azure Policy Initiatives:
├── Azure Security Benchmark
├── SOC2 Type II Compliance
├── ISO 27001 Compliance
├── NIST SP 800-53 Compliance
└── CIS Azure Foundations Benchmark
```

### 17.2 Data Residency & Sovereignty

```
Primary Data Residency: East US
Secondary (DR): West Europe

Data Handling:
├── Customer data: Encrypted at rest & in transit
├── Audit logs: Retained per compliance requirement
├── Backups: Geo-redundant with controlled regions
├── Metadata: May be replicated globally
└── PII: Masked in non-production environments

Compliance Certification:
- Region-specific certifications available
- Compliance reports: ISO 27001, SOC2
- Data center audits: Annual
```

---

## 18. COST ESTIMATION

### 18.1 Monthly Cost Breakdown (Estimated)

```
Azure Infrastructure (12 VMs, 5 AKS clusters, 20 SQL DBs)

Compute:
├── AKS Nodes (100 vCPUs, 500GB RAM): $12,000/month
├── VM Instances (16x D4s, 64 vCPU, 256GB RAM): $4,500/month
├── Batch compute (auto-scale): $2,000/month (variable)
└── Subtotal Compute: $18,500/month

Database:
├── SQL Database Premium (20x P4, 5TB): $8,000/month
├── PostgreSQL (3x 8vCore): $2,000/month
├── Cosmos DB (if used): $1,500/month (variable)
└── Subtotal Database: $11,500/month

Networking:
├── Azure Firewall Premium: $3,000/month
├── VPN Gateway (redundant): $1,200/month
├── NAT Gateway (6 public IPs): $900/month
├── Load Balancer (2x): $400/month
├── Application Gateway WAF: $2,000/month
├── ExpressRoute (10 Gbps): $8,000/month
├── Traffic Manager: $500/month
└── Subtotal Networking: $16,000/month

Storage:
├── Managed Disks (2TB): $1,500/month
├── Blob Storage (Backup): $2,000/month
├── Backup Vault: $500/month
└── Subtotal Storage: $4,000/month

Monitoring & Logging:
├── Log Analytics (100GB/day ingestion): $2,500/month
├── Application Insights: $500/month
├── Azure Sentinel: $2,000/month
└── Subtotal Monitoring: $5,000/month

Security & Compliance:
├── Azure Defender (multi-server): $1,500/month
├── Key Vault: $200/month
├── Policy & Governance: $200/month
└── Subtotal Security: $1,900/month

Other:
├── Azure Bastion: $200/month
├── Private Endpoint DNS: $500/month
├── Site Recovery: $1,000/month (for DR)
└── Subtotal Other: $1,700/month

TOTAL ESTIMATED: $58,600/month (~$703,200/year)

Cost Optimization Opportunities:
- Reserved Instances (compute): -$6,000/month (10% discount)
- Spot VMs (non-critical): -$2,000/month
- Right-sizing analysis: -$1,500/month
- Budget alerts & chargeback: -$500/month

Optimized Total: ~$48,600/month (~$583,200/year)
```

### 18.2 Cost Comparison: GCP vs Azure

| Component | GCP Est. | Azure Est. | Difference |
|---|---|---|---|
| Compute | $18K | $18.5K | +2.8% |
| Databases | $12K | $11.5K | -4.2% |
| Networking | $15K | $16K | +6.7% |
| Storage | $2K | $4K | +100% |
| Monitoring | $4K | $5K | +25% |
| **Total** | **$51K** | **$58.6K** | **+14.9%** |

**Note:** Azure includes higher baseline for backup/recovery and advanced security

---

## 19. MIGRATION STRATEGY

### 19.1 Workload Priority

```
Wave 1 (Weeks 1-4): Foundation & Non-Critical
├── Dev/Test environments
├── Non-critical applications
└── Testing infrastructure

Wave 2 (Weeks 5-8): Standard Production
├── Standard workloads
├── Non-critical databases
├── Some AKS clusters

Wave 3 (Weeks 9-12): Critical Production
├── Mission-critical services
├── Primary databases
├── Core AKS clusters

Wave 4 (Weeks 13+): Optimization & Decommission
├── Fine-tuning & optimization
├── GCP resource decommissioning
├── Archive & retention
```

### 19.2 Data Migration

**Database Migration (20 Cloud SQL instances)**
```
Method: Azure Database Migration Service (DMS)
Tool: Microsoft DMS Agent or native tooling

Steps:
1. Schema migration (SQL Server Migration Assistant)
2. Data sync (DMS for continuous replication)
3. Validation (row count, data integrity)
4. Cutover (application connection update)
5. Decommissioning (keep GCP backup 30 days)

Downtime: < 5 minutes (if using transactional replication)
Data loss: None (with validation)
```

**VM Migration**
```
Method: Azure Migrate (Modernize on Azure)
Tool: Azure Site Recovery for continuous replication

Steps:
1. Discovery (assess current VMs)
2. Replication (continuous sync to Azure)
3. Test failover (validate in Azure)
4. Cutover (switch to Azure)
5. Post-migration optimization

Downtime: Seconds (during network cutover)
Data loss: None (with replication)
```

**Container Migration (AKS)**
```
Method: Docker containers → AKS
Tool: Docker image registry (ACR - Azure Container Registry)

Steps:
1. Build & push images to ACR
2. Prepare Kubernetes manifests
3. Deploy to AKS
4. Validate health
5. Route traffic

Downtime: None (canary deployment possible)
Data loss: None
```

---

## 20. OPERATIONAL RUNBOOKS

### 20.1 Daily Operations

**Morning Check (8 AM UTC)**
```
1. Dashboard Review (5 min)
   - Check Azure health dashboard
   - Review Sentinel incidents
   - Check cost alerts

2. Alert Review (10 min)
   - Failed backups
   - Performance anomalies
   - Security alerts
   
3. Capacity Check (5 min)
   - Disk usage trends
   - Network utilization
   - Database connections
   
Action: Escalate any critical issues
```

**Evening Review (6 PM UTC)**
```
1. Cost Review
   - Daily spend vs budget
   - Cost anomalies
   
2. Backup Validation
   - Backup completion status
   - Recovery readiness
   
3. Compliance Check
   - Policy violations
   - Audit events
```

### 20.2 Weekly Operations

```
Monday:
- Weekly backup verification
- Cost analysis & optimization review
- Capacity planning

Wednesday:
- Security patch assessment
- Azure update tracking
- Policy compliance scan

Friday:
- Week-in-review meeting
- Incident postmortem (if any)
- Planning for next week
```

### 20.3 Monthly Operations

```
1st Week:
- Patch Tuesday deployments
- Performance baseline review
- Cost optimization report

2nd Week:
- Disaster recovery drill
- Documentation updates
- Architecture review

3rd Week:
- Scaling assessments
- Budget vs actuals
- Roadmap planning

4th Week:
- Month-end reconciliation
- Capacity forecasting
- Executive reporting
```

---

## 21. TROUBLESHOOTING GUIDE

### 21.1 Network Issues

**Symptom: Cross-spoke traffic fails**
```
Diagnosis:
1. Check hub firewall rules
2. Verify NSG rules on both subnets
3. Check VNet peering status
4. Verify routing in Network Watcher

Resolution:
- Add missing firewall rule
- Update NSG
- Recreate peering if failed
- Fix routing issue
```

**Symptom: Intermittent connectivity**
```
Diagnosis:
1. Check firewall connection tracking
2. Review network logs for drops
3. Check DNS resolution
4. Analyze packet captures

Resolution:
- Increase firewall session timeout
- Check for fragmentation issues
- Clear DNS cache
- Enable DNS logging
```

### 21.2 Database Issues

**Symptom: Connection pool exhausted**
```
Diagnosis:
1. Check active connection count
2. Review application logs
3. Check database CPU/memory
4. Verify network connectivity

Resolution:
- Increase connection pool size
- Check for connection leaks
- Scale up database tier
- Enable connection pooling
```

### 21.3 Kubernetes Issues

**Symptom: AKS node unhealthy**
```
Diagnosis:
1. Check node status: kubectl get nodes
2. Review node logs
3. Check kubelet status
4. Verify NSG rules

Resolution:
- Drain and delete node
- Let autoscaler recreate
- Or manually restart node
```

---

## APPENDIX A: Reference Architecture Diagrams

### A.1 Overall Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    MANAGEMENT GROUP                          │
│  (Governance, Policy, Monitoring, Compliance)                │
└──────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
    ┌───▼────────┐   ┌────────▼────────┐   ┌──────▼──────┐
    │ PLATFORM   │   │   LANDING ZONES │   │  SANDBOX    │
    │ (Hub/Core) │   │   (Spokes)      │   │             │
    └────────────┘   └─────────────────┘   └─────────────┘
         │                   │
    ┌────┴────┐          ┌───┴────┐
    │ Firewall│          │App VNets│
    │ NAT GW  │          │Data VNet│
    │ VPN/ER  │          │AKS VNet │
    └─────────┘          └─────────┘
```

### A.2 Detailed Network Diagram

[See section 3.1 above for detailed topology]

---

## APPENDIX B: Implementation Checklist

```
Pre-Implementation:
☐ Stakeholder approval
☐ Budget approval
☐ Resource planning
☐ Team training completed

Week 1:
☐ Management groups created
☐ Policy framework deployed
☐ RBAC configured
☐ Naming conventions ready

Week 2-3:
☐ Hub VNet created
☐ Hub subnet deployed
☐ Azure Firewall deployed
☐ Log Analytics created

Week 4:
☐ 1st spoke VNet created
☐ VNet peering configured
☐ NSG rules created
☐ DNS configured

Week 5-8:
☐ AKS deployed
☐ SQL migrated
☐ VMs deployed
☐ Load balancing configured

Post-Implementation:
☐ Documentation completed
☐ Team training completed
☐ Runbooks created
☐ DR procedures tested
☐ Go-live approval
```

---

## APPENDIX C: Useful Azure CLI Commands

```bash
# Get subscription info
az account list --output table

# Create resource group
az group create -n rg-prod-eastus -l eastus

# Create VNet with subnets
az network vnet create \
  -n hub-vnet \
  -g rg-prod-eastus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name firewall --subnet-prefix 10.0.1.0/26

# Deploy Azure Firewall
az network firewall create \
  -n fw-hub \
  -g rg-prod-eastus \
  -l eastus

# Create AKS cluster
az aks create \
  -n prod-aks-core \
  -g rg-prod-kubernetes \
  --vm-set-type VirtualMachineScaleSets \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/{subId}/resourceGroups/rg-prod-kubernetes/providers/Microsoft.Network/virtualNetworks/prod-aks-vnet/subnets/aks

# Create SQL Database
az sql server create \
  -n sqlserver-prod \
  -g rg-prod-data \
  -l eastus \
  -u sqladmin \
  -p {secure-password}

# Get resource status
az resource list -g rg-prod-eastus --output table

# Check policy compliance
az policy state summarize \
  --query "results[0].[policyCount, nonCompliantCount]" \
  -o tsv
```

---

## FINAL NOTES

This landing zone design provides:
✅ Enterprise-grade security architecture  
✅ Multi-region disaster recovery  
✅ Centralized governance and compliance  
✅ Cost management framework  
✅ Clear operational procedures  
✅ Scalable multi-workload environment  

The design can accommodate your current GCP footprint (10 VPCs, 20 databases, 8 clusters) while providing room for growth and evolution based on business needs.

---

**Document Version:** 1.0  
**Last Updated:** September 2026  
**Status:** Ready for Implementation
