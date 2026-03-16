<p align="center">
  <img src="https://img.shields.io/badge/Document-Azure%20Connectivity-1155CC?style=flat-square" alt="Azure Connectivity"/>
  <img src="https://img.shields.io/badge/Services-Azure%20SQL%20%7C%20PostgreSQL%20%7C%20ADLS%20%7C%20Blob-0F766E?style=flat-square" alt="Azure Services"/>
  <img src="https://img.shields.io/badge/Focus-Network%20%7C%20Identity%20%7C%20Connector%20Behavior-F2C811?style=flat-square&logoColor=000000" alt="Focus"/>
</p>

<h1 align="center">Azure Data Source Connectivity</h1>

<p align="center">
  <strong>Connector-by-connector guidance for deciding when Microsoft Fabric and Power BI can connect directly to Azure services, when a VNet Data Gateway is the right design, and when OPDG remains the fallback.</strong>
</p>

<p align="center">
  <a href="#1-connectivity-summary-matrix">Summary Matrix</a> •
  <a href="#2-network-architecture">Network</a> •
  <a href="#3-identity--authentication--deep-dive-per-service">Identity</a> •
  <a href="#4-end-to-end-flow-by-fabric-feature">Flows</a> •
  <a href="#5-security-hardening-checklist">Security</a>
</p>

> **Version**: 1.0  
> **Date**: 2026-03-16  
> **Companion to**: [01-gateway-strategy.md](./01-gateway-strategy.md) | [02-gateway-architecture.md](./02-gateway-architecture.md)  
> **Scope**: All connectivity paths from Microsoft Fabric / Power BI to **Azure SQL Database**, **Azure Database for PostgreSQL**, **Azure Data Lake Storage Gen2**, and **Azure Blob Storage**.  

---

## Azure Quick Answers

| Service | Public endpoint supported without gateway? | Preferred private pattern | Is OPDG still an option? |
|---|---|---|---|
| Azure SQL Database | Yes | VNet Data Gateway | Yes, as fallback |
| Azure Database for PostgreSQL | Yes | VNet Data Gateway | Yes, as fallback |
| ADLS Gen2 | Yes | Trusted workspace access or VNet-aware path | Yes, for specific storage firewall cases |
| Azure Blob Storage | Yes | VNet Data Gateway for same-region firewall scenarios | Yes |

> [!TIP]
> Azure is where **VNet Data Gateway** should do most of the private-network heavy lifting. Use **OPDG** only when the workload is not supported by VNet Data Gateway or when you intentionally standardize on gateway-hosted runtime.

## Executive Summary

Azure connectivity is different from AWS and GCP because Fabric already lives in Microsoft cloud infrastructure and many Azure services expose first-class connector and shortcut paths.

The practical rules are:

1. Use **direct cloud connectivity** for public Azure SQL, public PostgreSQL, and standard storage access.
2. Use **VNet Data Gateway** when the Azure service is exposed only through a **private endpoint** or when Power Query Online requires a private route.
3. Use **OPDG** only as a workload fallback, especially for patterns not covered by VNet Data Gateway or where the execution engine still depends on gateway runtime semantics.

That makes Azure the least infrastructure-heavy cloud in this repository, but it still has important exceptions around storage firewalls, service principal behavior through gateways, and unsupported workload combinations.

## 1. Connectivity Summary Matrix

| Azure Service | Fabric Feature | Connector / Method | Gateway Required? | Network Path | Identity Model |
|---|---|---|---|---|---|
| **Azure SQL Database** | Semantic Model (Import) | Azure SQL database connector | No for public endpoint; VNet GW for private endpoint | Public HTTPS / TDS or delegated VNet route | Microsoft account, Basic, Service Principal |
| **Azure SQL Database** | Semantic Model (DirectQuery) | Azure SQL database connector | No for public endpoint; VNet GW for private endpoint | Public HTTPS / TDS or delegated VNet route | Microsoft account, Basic, Service Principal |
| **Azure SQL Database** | Dataflow Gen2 | Azure SQL database connector | No for public endpoint; VNet GW for private endpoint | Public HTTPS / TDS or delegated VNet route | Microsoft account or Basic |
| **Azure SQL Database** | Workload not supported by VNet GW | Azure SQL database via OPDG | Optional fallback | OPDG outbound TLS | Microsoft account or Basic |
| **Azure Database for PostgreSQL** | Semantic Model (Import) | PostgreSQL connector | No for public endpoint; VNet GW for private endpoint | Public TCP with SSL or delegated VNet route | Database credentials or Microsoft account |
| **Azure Database for PostgreSQL** | Semantic Model (DirectQuery) | PostgreSQL connector | No for public endpoint; VNet GW for private endpoint | Public TCP with SSL or delegated VNet route | Database credentials or Microsoft account |
| **Azure Database for PostgreSQL** | Dataflow Gen2 | PostgreSQL connector | No for public endpoint; VNet GW for private endpoint | Public TCP with SSL or delegated VNet route | Database credentials |
| **ADLS Gen2** | OneLake Shortcut | Shortcut → ADLS Gen2 | No | DFS endpoint over HTTPS | Organizational account, Service principal, Workspace identity, SAS, Account key |
| **ADLS Gen2** | Lakehouse / Direct Lake | Shortcut-backed lakehouse | No | DFS endpoint over HTTPS | Connection bound to shortcut |
| **Azure Blob Storage** | Semantic Model / Dataflow Import | Azure Blob Storage connector | No in standard case; VNet GW or OPDG for same-region firewall edge case | Blob endpoint over HTTPS | Anonymous, Account key, Organizational account, SAS, Service principal |
| **Azure Blob Storage** | OneLake Shortcut | Shortcut → Azure Blob Storage | No | Blob endpoint over HTTPS | Organizational account, Service principal, Workspace identity, SAS, Account key |

---

## 2. Network Architecture

### 2.1 Decision Tree

```text
Is the Azure service private-endpoint only?
|
+-- YES
|   -> Use VNet Data Gateway by default
|   -> If workload is not supported on VNet GW, use OPDG fallback
|
+-- NO
    +-- Public connector or shortcut exists
    |   -> Use direct cloud connection
    +-- Storage firewall or same-region Power Query edge case applies
        -> Use VNet GW or OPDG
```

### 2.2 Reference Topology

```mermaid
graph TB
    subgraph Fabric
        FAB["Fabric Service"]
        PQO["Power Query Online"]
        SC["OneLake Shortcut Service"]
    end

    subgraph AzureConnectivity
        VNET["VNet Data Gateway"]
        OPDG["OPDG Fallback"]
    end

    subgraph AzureServices
        SQL["Azure SQL Database"]
        PG["Azure PostgreSQL"]
        ADLS["ADLS Gen2"]
        BLOB["Azure Blob Storage"]
    end

    FAB -->|Direct cloud path| SQL
    FAB -->|Direct cloud path| PG
    SC -->|Shortcut path| ADLS
    SC -->|Shortcut path| BLOB
    PQO -->|Import path| BLOB

    VNET -.->|Private endpoint access| SQL
    VNET -.->|Private endpoint access| PG
    VNET -.->|Firewall workaround| BLOB
    OPDG -.->|Fallback path| SQL
    OPDG -.->|Fallback path| PG
    OPDG -.->|Fallback path| BLOB
```

### 2.3 Network Notes

| Pattern | Use it for | Why |
|---|---|---|
| **Direct cloud path** | Public Azure SQL, public Azure PostgreSQL, standard storage access | Lowest complexity |
| **VNet Data Gateway** | Private endpoints and Azure-native private connectivity | Best-aligned Azure design |
| **OPDG fallback** | Unsupported VNet GW workloads or explicit gateway runtime standardization | Keeps a single escape hatch for edge cases |

---

## 3. Identity & Authentication — Deep Dive per Service

### 3.1 Azure SQL Database

```mermaid
graph LR
    CONN["Azure SQL Connection"] -->|Microsoft account| ENTRA["Microsoft Entra ID"]
    CONN -->|Basic auth| SQLAUTH["SQL Login"]
    CONN -->|Service principal| APP["App Registration"]
    ENTRA --> SQL["Azure SQL"]
    SQLAUTH --> SQL
    APP --> SQL
```

#### Key points

- The Azure SQL connector supports **Import** and **DirectQuery**.
- Supported authentication types include **Microsoft account**, **Basic**, and **Service Principal** in cloud-connected scenarios.
- **Service principal authentication is not supported through OPDG or VNet Data Gateway**.
- The connector supports advanced options such as **native SQL**, **command timeout**, and **failover support**.

#### Recommendation

- Use **direct cloud connection** whenever Azure SQL is publicly reachable and allowed by policy.
- Use **VNet Data Gateway** when the database sits behind a private endpoint.
- Use **OPDG** only when a specific workload or operational standard blocks VNet GW.

### 3.2 Azure Database for PostgreSQL

```mermaid
graph LR
    PQ["PostgreSQL Connector"] -->|Database auth| DBUSER["DB User"]
    PQ -->|Microsoft account| ENTRA["Microsoft Entra ID"]
    DBUSER --> PG["Azure PostgreSQL"]
    ENTRA --> PG
```

#### Key points

- Azure Database for PostgreSQL is consumed through the **PostgreSQL connector**.
- The connector supports **cloud connection**, **VNet Data Gateway**, and **OPDG**.
- Supported authentication types are **Database** and **Microsoft account**.
- DirectQuery is supported for Power BI semantic models.

#### Recommendation

- Public flexible server: direct connector with SSL.
- Private endpoint: VNet Data Gateway.
- Unsupported workload on VNet GW: OPDG fallback.

### 3.3 ADLS Gen2

```mermaid
graph LR
    SC["OneLake Shortcut"] -->|Cloud connection| ADLS["ADLS Gen2"]
    SC -->|Delegated auth| RBAC["RBAC and ACLs"]
```

#### Key points

- OneLake supports **ADLS Gen2 shortcuts** natively.
- ADLS shortcut auth supports **Organizational account**, **Service principal**, **Workspace identity**, **SAS**, and **Account key**.
- The target must use the **DFS endpoint**.
- If a storage firewall protects the account, you can use **trusted workspace access**.

#### Important caveats

- Cross-tenant ADLS shortcuts do not support organizational account or workspace identity.
- ADLS shortcuts do not support storage accounts using **managed private endpoints**.
- For Microsoft Entra delegated auth, plan for **Generate a user delegation key** rights.

### 3.4 Azure Blob Storage

```mermaid
graph LR
    PQ["Blob Connector"] -->|HTTPS| BLOB["Blob Storage"]
    SH["Blob Shortcut"] -->|Cloud connection| BLOB
```

#### Key points

- Azure Blob Storage connector supports **Import** scenarios.
- Authentication supports **Anonymous**, **Account key**, **Organizational account**, **SAS**, and **Service principal**.
- **Service principal auth is not supported when using OPDG or VNet Data Gateway**.
- Blob shortcuts are available in **preview** and support **Organizational account**, **Service principal**, **Workspace identity**, **SAS**, and **Account key**.

#### Same-region firewall caveat

Power Query Online cannot directly access an Azure Storage account with firewall enabled when both are in the same region. In that case, use:

- **On-premises data gateway**, or
- **VNet Data Gateway**

---

## 4. End-to-End Flow by Fabric Feature

### 4.1 DirectQuery to Azure SQL Public Endpoint

```mermaid
sequenceDiagram
    participant User
    participant PBI as Power BI Service
    participant SQL as Azure SQL

    User->>PBI: Open report
    PBI->>SQL: Run query via native connector
    SQL-->>PBI: Return rows
    PBI-->>User: Render visuals
```

### 4.2 Import Refresh to Azure PostgreSQL Private Endpoint

```mermaid
sequenceDiagram
    participant Fabric
    participant VNetGW as VNet Data Gateway
    participant PG as Azure PostgreSQL

    Fabric->>VNetGW: Start refresh
    VNetGW->>PG: Open private routed session
    PG-->>VNetGW: Return rows
    VNetGW-->>Fabric: Send imported data
```

### 4.3 Lakehouse Shortcut to ADLS Gen2

```mermaid
sequenceDiagram
    participant LH as Lakehouse
    participant SC as OneLake Shortcut
    participant ADLS as ADLS Gen2

    LH->>SC: Resolve shortcut path
    SC->>ADLS: Read via DFS endpoint
    ADLS-->>SC: Return files
    SC-->>LH: Expose files to Spark or Direct Lake
```

---

## 5. Security Hardening Checklist

- Prefer **VNet Data Gateway** over OPDG for Azure private endpoints.
- Use **Service Principal** only on direct cloud connections where the connector supports it end to end.
- Require **SSL or encrypted connections** for Azure SQL and PostgreSQL.
- Prefer **RBAC plus ACLs** over account keys for ADLS Gen2 where possible.
- Use **SAS** only with narrow scope and short lifetime.
- Avoid broad storage account keys for production shortcuts unless no delegated auth model is viable.
- Enable **trusted workspace access** for ADLS instead of broad firewall exceptions when supported.

---

## 6. Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Azure SQL service principal works direct but not through gateway | Unsupported auth through OPDG or VNet GW | Switch to Microsoft account or Basic for gateway path |
| Azure PostgreSQL private server not reachable | No delegated VNet or no OPDG fallback | Use VNet GW or route OPDG appropriately |
| Blob connector fails in same region with firewall enabled | Known Power Query Online storage limitation | Use VNet GW or OPDG |
| ADLS shortcut auth fails with Entra-based identity | Missing RBAC, ACLs, or delegation key rights | Validate RBAC, ACLs, and delegated key permissions |
| Cross-tenant ADLS shortcut fails with organizational account | Unsupported cross-tenant auth mode | Use service principal or SAS |

---

## 7. Architecture Decision Records

### ADR-01 — Azure private endpoints prefer VNet Data Gateway

**Decision**: Use VNet Data Gateway as the default private-connectivity model for Azure PaaS sources.  
**Why**: It aligns with Azure-native networking and avoids VM-based gateway management.  
**Consequence**: OPDG becomes an exception path, not the default.

### ADR-02 — OPDG remains the Azure fallback, not the first choice

**Decision**: Retain OPDG only for workloads unsupported by VNet Data Gateway or where gateway runtime behavior is required.  
**Why**: It preserves compatibility without forcing VM-hosted gateways onto Azure-native scenarios.  
**Consequence**: Lower operational overhead for most Azure data sources.

### ADR-03 — OneLake shortcuts are the preferred storage entry point

**Decision**: Prefer ADLS Gen2 and Blob shortcuts for lake-centric consumption patterns.  
**Why**: Shortcuts integrate directly with Spark, SQL, and Direct Lake patterns while centralizing storage authorization through managed connections.  
**Consequence**: Storage access becomes more standardized and less dependent on duplicated ingestion logic.