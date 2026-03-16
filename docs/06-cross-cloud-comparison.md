<p align="center">
  <img src="https://img.shields.io/badge/Document-Cross--Cloud%20Comparison-1155CC?style=flat-square" alt="Cross-cloud Comparison"/>
  <img src="https://img.shields.io/badge/Coverage-AWS%20%7C%20GCP%20%7C%20Azure%20%7C%20SaaS-0F766E?style=flat-square" alt="Coverage"/>
  <img src="https://img.shields.io/badge/Focus-Gateway%20Decision%20Rules-F2C811?style=flat-square&logoColor=000000" alt="Focus"/>
</p>

<h1 align="center">Cross-Cloud Connectivity Comparison</h1>

<p align="center">
  <strong>A single decision page for comparing AWS, GCP, Azure, and SaaS connectivity patterns across Microsoft Fabric and Power BI.</strong>
</p>

<p align="center">
  <a href="#1-executive-grid">Executive Grid</a> •
  <a href="#2-gateway-selection-by-source-pattern">Gateway Rules</a> •
  <a href="#3-authentication-patterns">Authentication</a> •
  <a href="#4-architecture-preferences">Architecture Preferences</a>
</p>

> **Version**: 1.0  
> **Date**: 2026-03-16  
> **Companion to**: [03-aws-connectivity.md](./03-aws-connectivity.md) | [04-gcp-connectivity.md](./04-gcp-connectivity.md) | [05-azure-connectivity.md](./05-azure-connectivity.md) | [07-snowflake-connectivity.md](./07-snowflake-connectivity.md) | [08-databricks-connectivity.md](./08-databricks-connectivity.md)

---

## Executive Summary

The architecture bias across this repository is now explicit:

1. **Direct cloud connection first** when the connector is supported and the endpoint is intentionally public.
2. **VNet Data Gateway first** for Azure private-endpoint patterns.
3. **VNet Data Gateway is also feasible for selected AWS and GCP connectors** when those sources are routed through Azure networking and the workload is supported.
4. **OPDG first** for on-premises, driver- or DSN-dependent workloads, shortcut/on-prem bridging, and any source outside the VNet GW support envelope.
5. **OneLake shortcuts first** for cloud object storage when the goal is lake-centric consumption instead of connector-based refresh.

The result is not "one gateway model for everything." It is a **provider-aware model** that uses the lightest viable path for each source type.

## 1. Executive Grid

| Provider / Pattern | Typical direct path | Typical private path | Default recommendation |
|---|---|---|---|
| AWS analytics | Redshift connector, Databricks connector | VNet GW or OPDG over VPN/peering | Direct for public endpoints; VNet GW feasible for supported connectors; OPDG for heavier private/runtime needs |
| AWS storage | S3 shortcut or S3 connector | OPDG only when VPC/network restriction exists | Direct or shortcut first |
| GCP analytics | BigQuery connector, PostgreSQL connector to Cloud SQL public IP | VNet GW or OPDG over VPN to Cloud SQL private IP | Direct for BigQuery; VNet GW feasible for supported connectors; mixed for Cloud SQL |
| GCP storage | GCS shortcut | OPDG only when shortcut path is network-restricted | Shortcut first |
| Azure private PaaS | Native connector | VNet Data Gateway | VNet GW first |
| Azure storage | Native connector or shortcut | VNet GW or OPDG for firewall edge cases | Direct or shortcut first |
| SaaS analytics | Native connector | OPDG only if private networking or enterprise routing requires it | Direct first |

## 2. Gateway Selection by Source Pattern

### Use OPDG when

- The source is **on-premises**.
- The source is in **AWS or GCP private networking** and Fabric cannot reach it directly.
- The connector still depends on a **gateway runtime**, **ODBC driver**, or **DSN-based** execution path.
- You need a controlled enterprise egress point across clouds.

### Use VNet Data Gateway when

- The source is an **Azure private endpoint** service.
- The workload is supported by VNet Data Gateway.
- You want Azure-native private connectivity without managing gateway VMs.
- The source is a **supported AWS or GCP connector** and Azure networking provides the route through VPN or ExpressRoute.

### Use no gateway when

- A **cloud connection** is supported in Power BI / Power Query Online.
- The endpoint is intentionally public and protected by identity plus allowlisting.
- The source is a **shortcut target** and does not require gateway bridging.

## 3. Authentication Patterns

| Pattern | Typical examples | Design preference |
|---|---|---|
| Federated / OAuth identity | BigQuery organizational account, Snowflake Entra ID, Azure SQL Microsoft account | Prefer for user-centric SaaS and cloud analytics |
| Service principal / workload identity | Azure SQL direct cloud connection, ADLS shortcuts, Databricks client credentials | Prefer for automated refresh where supported |
| Database credentials | Redshift, PostgreSQL, Snowflake username/password | Acceptable but should be narrowed and rotated |
| Storage keys / HMAC / account keys | S3, GCS, Blob, ADLS fallback patterns | Use only where delegated identity is not practical |
| PATs | Databricks | Acceptable operationally, but client credentials or OAuth is stronger |

## 4. Architecture Preferences

### Preferred by provider

| Provider | Preferred pattern | Why |
|---|---|---|
| Azure | Direct cloud or VNet GW | Native cloud alignment and lower infrastructure overhead |
| AWS | Direct for public analytics, OPDG for private | Mixed connector landscape and common VPC isolation |
| GCP | Direct for BigQuery, shortcut for GCS, OPDG for private Cloud SQL | Clear separation between analytics, storage, and private database paths |
| SaaS | Direct cloud connector | Lowest latency and least operational burden |

### Preferred by source type

| Source type | Best pattern |
|---|---|
| Data warehouse / SQL analytics | Native connector first |
| Object storage for lake use cases | OneLake shortcut first |
| Private relational database | VNet GW for Azure, OPDG elsewhere |
| Private file or driver-based source | OPDG |

## 5. Fast Decision Table

| Question | Best answer |
|---|---|
| Public Redshift? | Direct connector |
| Private Redshift? | OPDG |
| BigQuery? | Direct connector |
| GCS lake access? | OneLake shortcut |
| Azure SQL private endpoint? | VNet Data Gateway |
| Azure Blob with same-region firewall limitation? | VNet GW or OPDG |
| Snowflake? | Direct connector |
| Databricks public SQL Warehouse? | Direct connector |
| Databricks private workspace storage dependencies? | OPDG or VNet GW depending on platform |

## 6. Companion Map

| Need | Document |
|---|---|
| AWS details | [03-aws-connectivity.md](./03-aws-connectivity.md) |
| GCP details | [04-gcp-connectivity.md](./04-gcp-connectivity.md) |
| Azure details | [05-azure-connectivity.md](./05-azure-connectivity.md) |
| Snowflake deep dive | [07-snowflake-connectivity.md](./07-snowflake-connectivity.md) |
| Databricks deep dive | [08-databricks-connectivity.md](./08-databricks-connectivity.md) |