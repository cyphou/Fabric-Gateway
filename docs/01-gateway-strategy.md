# Data Gateway Strategy for Microsoft Fabric & Power BI

> **Version**: 1.1  
> **Date**: 2026-03-12  
> **Owner**: Centralized IT  
> **Scope**: Enterprise-wide gateway strategy across North Europe, East US, and France Central — including multi-cloud data sources (AWS, GCP, other providers)  

---

## 1. Executive Summary

This document defines the strategic approach for deploying, managing, and governing On-premises Data Gateways (OPDG) and VNet Data Gateways for Microsoft Fabric and Power BI across a multi-region, centrally-managed enterprise environment.

The strategy addresses **six workload categories**: Power BI Semantic Models (Import & DirectQuery), Power BI Dataflows Gen1/Gen2, Fabric Data Pipelines (Data Factory), Fabric Dataflows Gen2, Fabric Mirroring, and Paginated Reports — connected to a diverse **multi-cloud and hybrid source landscape** including Oracle, PostgreSQL, SQL Server, SAP, file-based systems, and data sources hosted on **AWS** (RDS, Redshift, S3), **GCP** (BigQuery, Cloud SQL, GCS), and **other cloud providers**.

---

## 2. Gateway Types — When to Use What

### 2.1 On-premises Data Gateway (Standard Mode)

| Attribute | Detail |
|---|---|
| **Use when** | Data source is on-premises, on IaaS VMs without VNet integration, **or hosted on non-Azure clouds (AWS, GCP, etc.)** |
| **Supports** | All six workload types listed in scope |
| **Clustering** | Yes — up to 10 nodes per cluster for HA and load distribution |
| **Network** | Outbound HTTPS 443 (Service Bus, Entra ID, Graph, OneLake, Shortcut targets) + TDS 1433 (Fabric SQL/Mirroring/XMLA) + optional AMQP 5671-5672; no inbound ports |
| **Install** | Windows Server (physical or VM); requires .NET Framework 4.8+ |
| **Best for** | Oracle on-prem, SQL Server on-prem, SAP systems, file shares, **AWS RDS/Redshift, GCP Cloud SQL/BigQuery, any non-Azure cloud DB** |

### 2.2 On-premises Data Gateway (Personal Mode)

| Attribute | Detail |
|---|---|
| **Use when** | Never in enterprise — only for individual developer testing |
| **Supports** | Power BI Desktop refresh only |
| **Clustering** | No |
| **Recommendation** | **Prohibit in production** via tenant settings |

### 2.3 VNet Data Gateway

| Attribute | Detail |
|---|---|
| **Use when** | Data source is in Azure VNet (e.g., Azure PostgreSQL with private endpoint, Azure SQL MI) |
| **Supports** | Power BI Semantic Models, Dataflows Gen2, Fabric Pipelines, Fabric Mirroring |
| **Clustering** | No — managed by Microsoft, auto-scales |
| **Network** | Delegates into your VNet subnet; no VM to manage |
| **Limitations** | Does NOT support Paginated Reports or Power BI Dataflows Gen1 |
| **Best for** | Azure-hosted PostgreSQL, Azure SQL, Azure Synapse private endpoints |

### 2.4 Decision Matrix

```
┌──────────────────────────────────────────────┐
│  Where is the data source hosted?            │
└───────┬──────────────┬───────────────┬───────┘
        │ Azure VNet   │ On-premises   │ Other Cloud
        │              │               │ (AWS/GCP/etc.)
        ▼              ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌────────────────────┐
│ VNet Data GW │ │ OPDG Standard│ │ OPDG Standard      │
│ (if workload │ │ in HA Cluster│ │ in HA Cluster      │
│  supports it)│ │              │ │ (gateway in Azure  │
└──────┬───────┘ └──────────────┘ │  connects to other │
       │                          │  cloud via VPN /   │
       ▼                          │  peering / public) │
  Does workload                   └────────────────────┘
  need Paginated
  Reports or
  Dataflows Gen1?
       │ YES
       ▼
  Use OPDG Standard
  (even for Azure
   VNet sources)
```

> **Multi-cloud rule**: VNet Data Gateway **only** works with data sources inside an Azure VNet. For any source on AWS, GCP, or other cloud providers, the **On-premises Data Gateway (Standard)** is the only option.

---

## 3. Strategic Principles

### 3.1 Centralized Ownership, Federated Usage

| Principle | Implementation |
|---|---|
| **IT owns the infrastructure** | Gateway VMs, clustering, patching, monitoring |
| **BUs consume via connections** | Data source connections shared through managed gateway |
| **RBAC controls access** | Gateway admins ≠ connection creators ≠ connection users |
| **Audit trail** | All gateway operations logged to Azure Monitor / Log Analytics |

### 3.2 Minimize Gateway Sprawl

- **One logical gateway cluster per region per purpose** (not per team)
- Consolidation into shared clusters with capacity planning
- Exception process for dedicated gateways (e.g., SAP with specific driver requirements)

### 3.3 Separate by Workload Criticality, Not by Team

| Gateway Cluster | Purpose | Workloads |
|---|---|---|
| `GW-PROD-<region>-ANALYTICS` | Production BI & analytics | Semantic Models (Import+DQ), Paginated Reports |
| `GW-PROD-<region>-DATAFLOW` | Production data integration | Dataflows Gen1/Gen2, Fabric Pipelines, Fabric Mirroring |
| `GW-NONPROD-<region>` | Dev/Test/UAT | All workloads (lower SLA) |

> **Rationale**: Import refresh and DirectQuery have different load profiles. Data integration (pipelines, dataflows) can be long-running and memory-intensive. Isolating them prevents refresh storms from impacting interactive queries.

### 3.4 No Personal Gateways in Production

- Disable personal gateway installs via **Fabric Admin Portal → Tenant Settings**
- Only Standard mode gateways managed by IT

### 3.5 VNet Gateways for Azure-Native Sources

- Every Azure PaaS data source with private endpoint connectivity **must** use VNet Data Gateway
- Eliminates VM management overhead for cloud-to-cloud data movement
- Use OPDG only when VNet Gateway has a workload limitation (Paginated Reports, Dataflows Gen1)

### 3.6 OPDG as the Multi-Cloud Bridge

- **Non-Azure cloud sources always require OPDG** — VNet Data Gateway has no cross-cloud capability
- Place OPDG gateway VMs in Azure (closest region to the Fabric capacity), not inside the other cloud
- Connect to external cloud sources via:
  - **Site-to-Site VPN** (Azure VPN Gateway ↔ AWS VGW / GCP Cloud VPN) — preferred for security
  - **ExpressRoute + partner peering** (e.g., Megaport, Equinix) to AWS Direct Connect or GCP Interconnect — preferred for bandwidth/latency
  - **Public internet with TLS** — acceptable for SaaS-like endpoints (e.g., BigQuery API, Snowflake, Databricks) with IP whitelisting
- Install appropriate **ODBC/JDBC drivers** on gateway VMs for each cloud data source (e.g., Simba driver for BigQuery, Amazon Redshift ODBC)
- Treat multi-cloud connections with the **same credential rotation and monitoring** as on-prem sources

---

## 4. Regional Deployment Strategy

### 4.1 Region Map

| Region | Role | Gateway Types | Data Sources |
|---|---|---|---|
| **North Europe** | Primary | OPDG Clusters + VNet GW | Oracle on-prem, SQL Server on-prem, SAP, Azure PG, File Shares, **AWS RDS (eu-west-1), GCP BigQuery (europe-west1)** |
| **East US** | Secondary | OPDG Clusters + VNet GW | Oracle on-prem (US DC), SQL Server, Azure SQL, **AWS Redshift (us-east-1), Snowflake, Databricks** |
| **France Central** | Future | OPDG Cluster + VNet GW | TBD — planned for DR or localized compliance |

> **Multi-cloud region affinity**: Place the OPDG cluster in the Azure region closest to both the Fabric capacity **and** the external cloud region hosting the data source. For example, AWS `eu-west-1` (Ireland) maps naturally to Azure North Europe.

### 4.2 Gateway-to-Fabric Region Affinity

**Critical rule**: Gateway cluster region must match (or be closest to) the Fabric capacity region.

| Fabric Capacity Region | Gateway Cluster Region | Latency Target |
|---|---|---|
| North Europe | North Europe | < 5 ms |
| East US | East US | < 5 ms |
| France Central | France Central (or North Europe fallback) | < 10 ms |

> Cross-region gateway usage causes latency penalties and potential data residency issues. Always co-locate.

---

## 5. Governance Framework

### 5.1 Naming Convention

```
GW-{ENV}-{REGION}-{PURPOSE}[-{SUFFIX}]

ENV     = PROD | NONPROD | DR
REGION  = NEU | EUS | FRC
PURPOSE = ANALYTICS | DATAFLOW | DEDICATED
SUFFIX  = optional (e.g., SAP, ORACLE)
```

**Examples**:
- `GW-PROD-NEU-ANALYTICS` — Production analytics gateway cluster in North Europe
- `GW-PROD-EUS-DATAFLOW` — Production data integration cluster in East US
- `GW-NONPROD-NEU` — Non-production gateway in North Europe
- `GW-PROD-NEU-DEDICATED-SAP` — Dedicated SAP gateway (exception)

### 5.2 Role Assignments

| Role | Scope | Who |
|---|---|---|
| **Gateway Admin** | Install, configure, cluster management, patching | Central IT Ops |
| **Connection Creator** | Add data source connections to a gateway | Data Engineering leads |
| **Connection User** | Use existing connections in reports/dataflows | Report authors, analysts |
| **Monitoring** | Read-only dashboard access | IT Ops + BI CoE |

### 5.3 Change Management

- Gateway version upgrades: **Monthly**, aligned with Microsoft's release cadence
- New data source onboarding: Request → IT reviews → Connection Creator provisions
- Capacity changes (add/remove cluster nodes): Quarterly review or on-demand if SLA breach

### 5.4 Security

| Control | Requirement |
|---|---|
| **Service account** | Dedicated AD service account per gateway cluster (not personal accounts) |
| **Credentials** | Stored in gateway encrypted store; rotate every 90 days |
| **Network** | Outbound 443 only; no inbound. ExpressRoute preferred for on-prem |
| **Cross-cloud network** | Site-to-Site VPN or ExpressRoute Global Reach + partner peering for AWS/GCP; public TLS for SaaS endpoints |
| **Encryption** | TLS 1.2+ enforced; gateway recovery key stored in secure vault |
| **MFA** | Required for gateway admin access (Entra ID Conditional Access) |
| **Private endpoints** | VNet Gateway uses delegated subnet; OPDG uses private link where possible |
| **Multi-cloud credentials** | AWS IAM keys, GCP service account keys managed with same rotation policy (90 days); store in Azure Key Vault |

---

## 6. Capacity Planning Guidelines

### 6.1 Sizing per Gateway VM (OPDG)

| Workload Profile | CPU | RAM | Disk | Nodes in Cluster |
|---|---|---|---|---|
| **Light** (< 50 datasets, import only) | 4 vCPU | 16 GB | 128 GB SSD | 2 (HA minimum) |
| **Medium** (50-200 datasets, mixed) | 8 vCPU | 32 GB | 256 GB SSD | 3 |
| **Heavy** (200+ datasets, DirectQuery + pipelines) | 16 vCPU | 64 GB | 512 GB SSD | 4-5 |

### 6.2 Scaling Triggers

| Metric | Threshold | Action |
|---|---|---|
| CPU sustained > 80% | 15 min window | Add cluster node |
| Memory > 85% | 15 min window | Add cluster node or increase VM size |
| Concurrent queries > 30/node | Queueing observed | Add cluster node |
| Refresh failures > 5% | Per day | Investigate; likely resource or timeout |

---

## 7. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Single-node failure | Medium | High | HA clustering (min 2 nodes) |
| Gateway version drift | Medium | Medium | Automated update policy; max 1 version behind |
| Credential expiry | Medium | High | Automated rotation; monitoring alerts |
| Network partition (ExpressRoute down) | Low | Critical | VPN fallback; public internet as last resort |
| VNet Gateway feature gap | Medium | Medium | Maintain parallel OPDG for unsupported workloads |
| Cross-cloud network failure | Medium | High | Redundant VPN tunnels; multi-path routing; public internet fallback |
| Cross-cloud credential leak | Low | Critical | Store AWS/GCP keys in Azure Key Vault; auto-rotate; restrict IAM scope |
| Multi-cloud driver compatibility | Medium | Medium | Maintain driver matrix; test after each gateway update |
| Gateway sprawl (shadow gateways) | High | Medium | Tenant settings lockdown; quarterly audit |

---

## 8. Success Metrics

| KPI | Target |
|---|---|
| Gateway availability | ≥ 99.5% per cluster |
| Scheduled refresh success rate | ≥ 98% |
| Mean time to recover (MTTR) | < 30 min with cluster failover |
| Gateway sprawl ratio | < 1.5 gateways per region (vs baseline) |
| Time to onboard new data source | < 2 business days |

---

## Next Steps

→ Proceed to [02-gateway-architecture.md](./02-gateway-architecture.md) for the detailed technical architecture and deployment design.
