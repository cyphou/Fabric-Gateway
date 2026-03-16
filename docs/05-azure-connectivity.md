<p align="center">
  <img src="https://img.shields.io/badge/Document-Azure%20Connectivity-1155CC?style=flat-square" alt="Azure Connectivity"/>
    <img src="https://img.shields.io/badge/Services-Azure%20SQL%20%7C%20PostgreSQL%20%7C%20ADLS%20%7C%20Blob%20%7C%20Databricks%20%7C%20ADX%20%7C%20Cosmos-0F766E?style=flat-square" alt="Azure Services"/>
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
> **Scope**: All connectivity paths from Microsoft Fabric / Power BI to **Azure SQL Database**, **Azure Database for PostgreSQL**, **Azure Data Lake Storage Gen2**, **Azure Blob Storage**, **Azure Databricks**, **Azure Data Explorer**, and **Azure Cosmos DB**.  

---

## Azure Quick Answers

| Service | Public endpoint supported without gateway? | Preferred private pattern | Is OPDG still an option? |
|---|---|---|---|
| Azure SQL Database | Yes | VNet Data Gateway | Yes, as fallback |
| Azure Database for PostgreSQL | Yes | VNet Data Gateway | Yes, as fallback |
| ADLS Gen2 | Yes | Trusted workspace access or VNet-aware path | Yes, for specific storage firewall cases |
| Azure Blob Storage | Yes | VNet Data Gateway for same-region firewall scenarios | Yes |
| Azure Databricks | Yes | VNet Data Gateway or OPDG when workspace storage is private | Yes |
| Azure Data Explorer | Yes | VNet Data Gateway | Yes |
| Azure Cosmos DB | Yes, for connector and mirroring setup | Native Fabric mirroring or direct connector; private network bypass for mirroring | Limited fallback, but mirroring is preferred |

> [!TIP]
> Azure is where **VNet Data Gateway** should do most of the private-network heavy lifting. Use **OPDG** only when the workload is not supported by VNet Data Gateway or when you intentionally standardize on gateway-hosted runtime.

## Executive Summary

Azure connectivity is different from AWS and GCP because Fabric already lives in Microsoft cloud infrastructure and many Azure services expose first-class connector and shortcut paths.

The practical rules are:

1. Use **direct cloud connectivity** for public Azure SQL, public PostgreSQL, and standard storage access.
2. Use **VNet Data Gateway** when the Azure service is exposed only through a **private endpoint** or when Power Query Online requires a private route.
3. Use **OPDG** only as a workload fallback, especially for patterns not covered by VNet Data Gateway or where the execution engine still depends on gateway runtime semantics.

This same VNet GW model can also be extended to **supported non-Azure connectors** when those sources are routed into Azure networking, but Azure remains the place where VNet GW is the strongest default.

That makes Azure the least infrastructure-heavy cloud in this repository, but it still has important exceptions around storage firewalls, service principal behavior through gateways, Azure Databricks private storage access, and Azure Cosmos DB analytics patterns where mirroring is now the better design than legacy connector-first ingestion.

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
| **Azure Databricks** | Semantic Model (Import / DirectQuery) | Azure Databricks connector | No for public SQL warehouse; VNet GW or OPDG if private storage path must be reached | HTTPS to SQL warehouse plus storage path for CloudFetch | Azure AD, PAT, Username / Password |
| **Azure Databricks** | Dataflow Gen2 | Azure Databricks connector | No for public SQL warehouse; VNet GW or OPDG for private storage access | HTTPS plus gateway path when storage is firewalled | Azure AD, PAT, Username / Password |
| **Azure Data Explorer** | Semantic Model (Import / DirectQuery) | Azure Data Explorer connector | No for public cluster; VNet GW for private route | HTTPS to Kusto cluster | Organizational account |
| **Azure Data Explorer** | Dataflow Gen2 | Azure Data Explorer connector | No for public cluster; VNet GW for private route | HTTPS to Kusto cluster | Organizational account |
| **Azure Cosmos DB** | Semantic Model (Import / DirectQuery) | Azure Cosmos DB v2 connector | Not required for public endpoint, but not preferred for new projects | HTTPS to Cosmos endpoint | Feed key or Organizational account |
| **Azure Cosmos DB** | Fabric analytics landing zone | Fabric Mirroring for Azure Cosmos DB | No gateway; use network ACL bypass for private account | Fabric-managed near real-time replication into OneLake | Entra ID with RBAC or account keys |

### 1.1 Where VNet Data Gateway Fits in Azure

For Azure, VNet Data Gateway is the primary private-network option and should be assumed first for:

- **Azure SQL Database** private endpoints,
- **Azure Database for PostgreSQL** private endpoints,
- **Azure Data Lake Storage Gen2** private connectivity scenarios that fall inside the supported workload model, and
- **Azure Blob Storage** when firewall behavior or same-region Power Query constraints make direct access unsuitable,
- **Azure Databricks** when the SQL warehouse is usable but private workspace storage requires a governed path for CloudFetch, and
- **Azure Data Explorer** when the Kusto cluster is reachable only through private routing.

For **Azure Cosmos DB**, prefer **Fabric mirroring** before considering the legacy Power Query connector.

OPDG remains a fallback, not the default.

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
        DBR["Azure Databricks"]
        ADX["Azure Data Explorer"]
        COSMOS["Azure Cosmos DB"]
    end

    FAB -->|Direct cloud path| SQL
    FAB -->|Direct cloud path| PG
    FAB -->|Native connector| DBR
    FAB -->|Native connector| ADX
    SC -->|Shortcut path| ADLS
    SC -->|Shortcut path| BLOB
    PQO -->|Import path| BLOB
    FAB -->|Mirror or connector| COSMOS

    VNET -.->|Private endpoint access| SQL
    VNET -.->|Private endpoint access| PG
    VNET -.->|Firewall workaround| BLOB
    VNET -.->|Private storage or cluster path| DBR
    VNET -.->|Private cluster path| ADX
    OPDG -.->|Fallback path| SQL
    OPDG -.->|Fallback path| PG
    OPDG -.->|Fallback path| BLOB
    OPDG -.->|Fallback path| DBR
    OPDG -.->|Fallback path| ADX
```

### 2.3 Network Notes

| Pattern | Use it for | Why |
|---|---|---|
| **Direct cloud path** | Public Azure SQL, public Azure PostgreSQL, standard storage access | Lowest complexity |
| **VNet Data Gateway** | Private endpoints and Azure-native private connectivity | Best-aligned Azure design |
| **Fabric-native replication** | Azure Cosmos DB analytical landing into OneLake | Best analytical pattern for near real-time HTAP-style reporting |
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

### 3.5 Azure Databricks

```mermaid
graph LR
    FAB["Fabric or Power BI"] -->|Azure AD / PAT| SQLW["Databricks SQL Warehouse"]
    SQLW -->|CloudFetch| STG["Workspace Storage"]
    VNET["VNet Data Gateway"] -.->|Private storage access| STG
    OPDG["OPDG"] -.->|Fallback private access| STG
```

#### Key points

- The **Azure Databricks** connector supports **Import** and **DirectQuery**.
- Supported authentication includes **Azure Active Directory**, **Personal Access Token**, and **Username / Password** depending on host and product surface.
- Microsoft guidance is explicit that if the **workspace storage account firewall is enabled**, use **VNet Data Gateway** or **OPDG** so CloudFetch can still reach storage, or disable CloudFetch.
- The connector now supports **ADBC implementation 2.0** in preview.

#### Recommendation

- Public SQL warehouse with standard storage access: connect directly.
- Private storage account behind firewall: prefer **VNet Data Gateway**.
- If the workload or runtime model does not fit VNet GW, use **OPDG** fallback.

### 3.6 Azure Data Explorer

```mermaid
graph LR
    PQ["ADX Connector"] -->|Organizational account| ENTRA["Microsoft Entra ID"]
    ENTRA --> ADX["Azure Data Explorer Cluster"]
```

#### Key points

- The **Azure Data Explorer (Kusto)** connector supports **Import**, **DirectQuery**, and **Dataflow Gen2**.
- Authentication is **Organizational account**.
- Power Query Online supports the connector directly and can use a gateway when a private route is needed.
- For complex logic, Microsoft recommends using **Kusto functions** instead of embedding complex KQL in Power Query.

#### Recommendation

- Public cluster: direct connector.
- Private cluster: **VNet Data Gateway** first.
- Use **OPDG** only when the workload cannot use VNet GW.

### 3.7 Azure Cosmos DB

```mermaid
graph LR
    OLTP["Azure Cosmos DB"] -->|Near real-time replication| MIRROR["Fabric Mirroring"]
    MIRROR --> ONELAKE["OneLake Delta Tables"]
    ONELAKE --> SQL["SQL Analytics Endpoint"]
    ONELAKE --> DL["Direct Lake / Power BI"]
```

#### Key points

- The legacy **Azure Cosmos DB v2** connector supports **Import** and **DirectQuery**, with **Feed key** or **Organizational account** authentication.
- Microsoft now explicitly advises **not to use the v2 connector for new projects** and to prefer **Azure Cosmos DB Mirroring for Microsoft Fabric** instead.
- Mirroring provides **near real-time replication into OneLake** without consuming transactional **RUs** for replication and without a gateway.
- Mirroring supports **private networks** by using **Network ACL Bypass** for authorized Fabric workspaces.
- Continuous backup is required to enable mirroring.

#### Recommendation

- New analytical pattern: **Fabric Mirroring** first.
- Use the **v2 connector** only for existing/reporting scenarios that specifically need connector-based access.
- For AI or nested JSON workloads, mirroring is materially stronger than the legacy connector because it lands the data in OneLake and supports downstream SQL, Spark, and Direct Lake patterns.

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

### 4.4 Near Real-Time Analytics from Azure Cosmos DB

```mermaid
sequenceDiagram
    participant Cosmos as Azure Cosmos DB
    participant Mirror as Fabric Mirroring
    participant OneLake as OneLake
    participant BI as Power BI or SQL Endpoint

    Cosmos->>Mirror: Replicate inserts, updates, deletes
    Mirror->>OneLake: Materialize Delta tables
    OneLake-->>BI: Expose analytics-ready data
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
- For **Azure Databricks**, validate both the SQL warehouse path and the backing storage access path.
- For **Azure Cosmos DB**, prefer **Entra ID plus RBAC** for mirroring and enable only the minimum required network bypass settings.

---

## 6. Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Azure SQL service principal works direct but not through gateway | Unsupported auth through OPDG or VNet GW | Switch to Microsoft account or Basic for gateway path |
| Azure PostgreSQL private server not reachable | No delegated VNet or no OPDG fallback | Use VNet GW or route OPDG appropriately |
| Blob connector fails in same region with firewall enabled | Known Power Query Online storage limitation | Use VNet GW or OPDG |
| ADLS shortcut auth fails with Entra-based identity | Missing RBAC, ACLs, or delegation key rights | Validate RBAC, ACLs, and delegated key permissions |
| Cross-tenant ADLS shortcut fails with organizational account | Unsupported cross-tenant auth mode | Use service principal or SAS |
| Azure Databricks query succeeds but large results fail | CloudFetch cannot reach private workspace storage | Use VNet GW or OPDG, or disable CloudFetch |
| Azure Data Explorer DirectQuery query is brittle | Complex KQL embedded in Power Query | Move logic into Kusto functions |
| Azure Cosmos DB analytics design is slow or schema-fragile | Legacy v2 connector used for evolving or nested JSON workloads | Prefer Fabric mirroring |

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

### ADR-04 — Azure Cosmos DB analytics should prefer mirroring over the legacy connector

**Decision**: Use Fabric Mirroring as the default analytics pattern for Azure Cosmos DB.  
**Why**: Microsoft no longer recommends the v2 connector for new projects, and mirroring gives near real-time OneLake replication without gateway infrastructure.  
**Consequence**: Azure Cosmos DB joins the Azure-native path set where the best design is gateway-free by default.