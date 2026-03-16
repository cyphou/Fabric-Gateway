<p align="center">
  <img src="https://img.shields.io/badge/Document-AWS%20Connectivity-1155CC?style=flat-square" alt="AWS Connectivity"/>
  <img src="https://img.shields.io/badge/Services-S3%20%7C%20Redshift%20%7C%20Databricks%20%7C%20SageMaker-0F766E?style=flat-square" alt="AWS Services"/>
  <img src="https://img.shields.io/badge/Focus-Network%20%7C%20Identity%20%7C%20Connector%20Behavior-F2C811?style=flat-square&logoColor=000000" alt="Focus"/>
</p>

<h1 align="center">AWS Data Source Connectivity</h1>

<p align="center">
  <strong>Connector-by-connector guidance for deciding when Fabric and Power BI can connect directly to AWS and when a gateway, VPN, or driver-based path is still required.</strong>
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
> **Scope**: All connectivity paths from Microsoft Fabric / Power BI to **AWS S3**, **AWS Databricks (Databricks on AWS)**, **AWS Redshift**, and **AWS SageMaker Unified Studio** — covering network topology, identity / authentication models, and Fabric connector mapping.  

---

## AWS Quick Answers

| Service | Public endpoint supported without gateway? | Is OPDG still an option? | Best starting point |
|---|---|---|---|
| S3 | Yes | Only for VPC endpoint or on-prem shortcut scenarios | Shortcut or native S3 connector |
| Redshift | Yes | Yes, but it is not mandatory | Native Redshift connector for public endpoints |
| Databricks on AWS | Yes | Yes | Native Databricks connector for public SQL Warehouse |
| SageMaker Unified Studio | No native connector | Yes for Athena ODBC path | Athena via OPDG or direct S3-based access |

> [!TIP]
> This document is the operational answer key for AWS connectivity decisions. Use it when the strategy says "AWS" and you need the exact connector, gateway, identity, and network path.

## Executive Summary

This document separates AWS connectivity into two broad patterns:

1. **Native cloud connectivity** for public endpoints where Fabric or Power BI has a supported connector.
2. **Gateway-mediated connectivity** for private endpoints, VPC-only access, or services that still depend on ODBC / DSN-based runtime execution.

In practice, the current guidance is:

- **S3**: usually direct, especially for Shortcuts and public connector paths; use OPDG only for VPC endpoint or on-prem shortcut scenarios.
- **Redshift**: direct is the default pattern; OPDG remains optional when you deliberately choose a private-routing or driver-based path.
- **Databricks on AWS**: direct for public SQL Warehouse endpoints; OPDG for Private Link or other private workspace patterns.
- **SageMaker Unified Studio**: no native Power Query connector; use Athena via OPDG or direct S3-based access depending on the use case.

The rest of the document provides the implementation detail behind those four rules: connectors, identity models, network topology, security controls, and end-to-end flow patterns.

## 1. Connectivity Summary Matrix

| AWS Service | Fabric Feature | Connector / Method | Gateway Required? | Network Path | Identity Model |
|---|---|---|---|---|---|
| **S3** | OneLake Shortcut | Shortcut → S3 | No | Public HTTPS | IAM Access Key *or* IAM Role (cross-account) |
| **S3** | Fabric Pipeline (Copy Activity) | Amazon S3 connector | No (cloud) / OPDG (VPC endpoint) | Public HTTPS or S2S VPN | IAM Access Key (stored in Fabric connection) |
| **S3** | Dataflow Gen2 | Amazon S3 connector | No (cloud) / OPDG (VPC endpoint) | Public HTTPS or S2S VPN | IAM Access Key |
| **S3** | Notebook (Spark) | `boto3` / `s3a://` protocol | No | Public HTTPS | IAM Access Key or STS Assume-Role (env vars / Fabric secret) |
| **Redshift** | Semantic Model (Import) | Amazon Redshift connector | No; OPDG optional for private-routing patterns | Public + TLS by default; optional S2S VPN / OPDG path | Redshift DB credentials, Microsoft account, Organizational account (Entra ID SSO) |
| **Redshift** | Semantic Model (DirectQuery) | Amazon Redshift connector | No; OPDG optional for private-routing patterns | Public + TLS by default; optional S2S VPN / OPDG path | Redshift DB credentials, Microsoft account, Organizational account (Entra ID SSO) |
| **Redshift** | Dataflow Gen2 | Amazon Redshift connector | No; OPDG optional for private-routing patterns | Public + TLS by default; optional S2S VPN / OPDG path | Redshift DB credentials, Microsoft account, Organizational account |
| **Redshift** | Fabric Pipeline (Copy Activity) | Amazon Redshift connector | No; OPDG optional for private-routing patterns | Public + TLS by default; optional S2S VPN / OPDG path | Redshift DB credentials |
| **Redshift** | Notebook (Spark) | `redshift-connector` / JDBC | No | Public + TLS | Redshift DB credentials or IAM-based auth |
| **Redshift Serverless** | Same as above | Same connectors | No; OPDG optional for private-routing patterns | Public + TLS by default; optional S2S VPN / OPDG path | IAM or DB credentials, Entra ID SSO |
| **Databricks on AWS** | Semantic Model (Import/DQ) | Databricks connector | No (public endpoint) / OPDG (private workspace) | Public HTTPS or S2S VPN | Username/Password, PAT, OAuth (OIDC) |
| **Databricks on AWS** | Dataflow Gen2 | Databricks connector | No (public endpoint) / OPDG (private workspace) | Public HTTPS or S2S VPN | Username/Password, PAT, OAuth (OIDC) |
| **Databricks on AWS** | Fabric Pipeline (Copy Activity) | Databricks connector | No (public endpoint) / OPDG (private workspace) | Public HTTPS or S2S VPN | Username/Password, PAT, OAuth (OIDC) |
| **Databricks on AWS** | Notebook (Spark) | Databricks Connect v2 or REST API | No | Public HTTPS | Databricks PAT or OAuth (M2M) |
| **Databricks on AWS** | OneLake Shortcut | Shortcut → ADLS/S3 (Databricks-managed) | No | Public HTTPS | See S3 row above |
| **SageMaker Unified Studio** | Semantic Model (Import/DQ) | No native connector — use Amazon Athena or ODBC | OPDG (Athena requires ODBC driver) | Public HTTPS or S2S VPN | DSN configuration (Athena ODBC), Organizational account |
| **SageMaker Unified Studio** | Dataflow Gen2 | Amazon Athena connector or ODBC | OPDG | Public HTTPS or S2S VPN | DSN configuration, Organizational account |
| **SageMaker Unified Studio** | Fabric Pipeline (Copy Activity) | No native connector — use S3 connector (for underlying storage) | No (public endpoint) / OPDG (VPC endpoint) | Public HTTPS or S2S VPN | IAM Access Key (for S3 storage layer) |
| **SageMaker Unified Studio** | Notebook (Spark) | `boto3` (S3), `pyathena`, or JDBC | No | Public HTTPS | IAM Access Key, STS AssumeRole, or Athena JDBC |

---

## 2. Network Architecture

### 2.1 Decision Tree — Choosing the Right Network Path

```
Is the AWS resource publicly accessible?
│
├── YES ──────────────────────────────────────────────┐
│   Is an OPDG required for this connector?           │
│   ├── YES → OPDG makes outbound HTTPS/TDS call     │
│   │         over the internet (TLS 1.2+)            │
│   │         ► Whitelist OPDG public IPs on AWS SG   │
│   └── NO  → Fabric cloud service connects directly  │
│             ► Whitelist Fabric service IPs or use    │
│               Fabric Trusted Service                 │
│                                                      │
└── NO (Private VPC only) ────────────────────────────┘
    OPDG is ALWAYS required.
    Choose connectivity method:
    ├── Option A: Site-to-Site VPN (Azure VPN GW ↔ AWS VGW)
    ├── Option B: ExpressRoute + Partner Peering ↔ AWS Direct Connect
    └── Option C: AWS PrivateLink (service-specific, limited use)
```

### 2.2 Network Topology — Core AWS Services

```mermaid
graph TB
    subgraph "Microsoft Fabric"
        FAB["Fabric Service\n(Cloud)"]
        SPARK["Fabric Spark\nRuntime"]
        PIPE["Data Pipeline\nCopy Activity"]
        SC["OneLake\nShortcut Service"]
    end

    subgraph "Azure (Gateway VNet)"
        VPNGW["Azure VPN GW\nor ER Circuit"]
        OPDG["OPDG Cluster\n(3 nodes)"]
    end

    subgraph "AWS Account"
        subgraph "VPC (Private)"
            RS["Redshift\nCluster"]
            DBR_PRIV["Databricks\nWorkspace\n(Private Link)"]
            S3_EP["S3 VPC\nEndpoint"]
        end
        subgraph "Public Endpoints"
            S3_PUB["S3 Bucket\n(Public)"]
            DBR_PUB["Databricks\nWorkspace\n(Public)"]
            RS_PUB["Redshift\nPublic Endpoint"]
        end
        IAM["AWS IAM\n(Identity)"]
    end

    %% Private paths
    OPDG <-->|IPsec Tunnel| VPNGW
    VPNGW <-.->|Optional private route| RS
    VPNGW <-->|S2S VPN| DBR_PRIV
    VPNGW <-->|S2S VPN| S3_EP

    %% Public paths
    FAB -->|HTTPS 443| RS_PUB
    OPDG -.->|Optional routed access| RS_PUB
    OPDG -->|HTTPS 443| DBR_PUB
    FAB -->|HTTPS 443| S3_PUB
    SC -->|HTTPS 443| S3_PUB
    PIPE -->|HTTPS 443| S3_PUB
    SPARK -->|HTTPS 443| S3_PUB

    %% IAM for all
    IAM -.->|Authorises| S3_PUB
    IAM -.->|Authorises| S3_EP
    IAM -.->|Authorises| RS
    IAM -.->|Authorises| RS_PUB
```

### 2.3 Network Option Details

#### Option A — Site-to-Site VPN (Recommended for most cases)

| Property | Value |
|---|---|
| **Azure side** | Azure VPN Gateway (VpnGw2 or higher), BGP enabled |
| **AWS side** | AWS Virtual Private Gateway (VGW) attached to target VPC |
| **Tunnels** | 2 × IPsec tunnels (active/active) for HA |
| **Encryption** | IKEv2, AES-256-GCM |
| **Bandwidth** | Up to 1.25 Gbps per tunnel (VpnGw2); aggregated with 2 tunnels |
| **Latency** | 10-30 ms same-geo (e.g., Azure North Europe ↔ AWS eu-west-1) |
| **Routing** | BGP dynamic routes; AWS VPC CIDR propagated to Azure route table |
| **Cost** | ~$0.05/hr Azure VPN GW + AWS VGW hourly + data transfer |

**Setup summary**:
1. Create Azure VPN Gateway in the Gateway VNet
2. Create AWS Virtual Private Gateway, attach to VPC
3. Create AWS Site-to-Site VPN connection → download configuration for Azure
4. Create Local Network Gateway in Azure with AWS VPC CIDR
5. Create Connection (IKEv2) between Azure VPN GW and Local Network GW
6. Verify BGP peering and route propagation
7. Update AWS Security Groups to allow OPDG subnet CIDRs

#### Option B — ExpressRoute + AWS Direct Connect (High-bandwidth / low-latency)

| Property | Value |
|---|---|
| **Azure side** | ExpressRoute Circuit (Standard or Premium SKU) |
| **AWS side** | AWS Direct Connect (Dedicated or Hosted) |
| **Bridge** | Colocation partner (Megaport, Equinix Fabric, or Console Connect) |
| **Bandwidth** | 50 Mbps – 100 Gbps depending on Direct Connect type |
| **Latency** | 3-10 ms same-geo |
| **Routing** | BGP between Azure MSEE ↔ Megaport VXC ↔ AWS DX Router |
| **Cost** | Higher than VPN — circuit fees + colocation port + data egress |

**When to use**: Large-volume Redshift extracts (> 50 GB/day), latency-sensitive DirectQuery workloads against Databricks SQL Warehouse, or when VPN throughput is insufficient.

#### Option C — Public Endpoint + TLS (Simplest)

| Property | Value |
|---|---|
| **Encryption** | TLS 1.2+ end-to-end |
| **Authentication** | IAM keys (S3), DB credentials (Redshift), PAT/OAuth (Databricks) |
| **IP restriction** | AWS Security Group or S3 bucket policy whitelisting OPDG NAT IPs / Fabric service IPs |
| **Bandwidth** | Internet throughput; no SLA |
| **Latency** | Variable (20-80 ms typical) |

**When to use**: S3 buckets that are already public/semi-public, Databricks workspaces with public networking, Redshift Serverless with public endpoints, or when VPN setup is not feasible.

---

## 3. Identity & Authentication — Deep Dive per Service

### 3.1 AWS S3

```mermaid
graph LR
    subgraph "Fabric Side"
        CONN_S3["Fabric Connection\n(S3)"]
        KV["Azure Key Vault\n(optional secret store)"]
    end

    subgraph "AWS Side"
        IAM_USER["IAM User\n(Access Key + Secret)"]
        IAM_ROLE["IAM Role\n(Cross-Account)"]
        S3B["S3 Bucket\n+ Bucket Policy"]
    end

    CONN_S3 -->|Access Key ID + Secret| IAM_USER
    IAM_USER -->|s3:GetObject, s3:ListBucket| S3B
    CONN_S3 -.->|STS AssumeRole| IAM_ROLE
    IAM_ROLE -.->|Temporary credentials| S3B
    KV -.->|Secret reference| CONN_S3
```

#### Authentication Options

| Method | How it works | Pros | Cons | Recommended for |
|---|---|---|---|---|
| **IAM Access Key** (Key ID + Secret) | Static long-lived credentials stored in Fabric connection | Simple setup; works with all Fabric connectors | Keys don't expire unless rotated; risk of leakage | OneLake Shortcut, Pipeline, Dataflow — when STS is not supported |
| **IAM Role (STS AssumeRole)** | Fabric or gateway calls `sts:AssumeRole` to get temporary creds | Short-lived tokens (1-12 hrs); no static secret stored | Requires trust policy on AWS side; not all Fabric connectors support it | Notebooks (via `boto3`); Pipeline if using custom activity |
| **S3 Bucket Policy (IP-based)** | Bucket policy restricts `s3:*` to specific source IPs + IAM principal | Defence-in-depth; limits blast radius | Must maintain IP list; doesn't replace authentication | Combined with IAM key — always add as secondary control |

#### IAM Policy — Minimum Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FabricS3ReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-fabric-bucket",
        "arn:aws:s3:::my-fabric-bucket/*"
      ]
    }
  ]
}
```

> Add `s3:PutObject` only if Fabric needs to write back (e.g., Pipeline sink). Follow least-privilege.

#### S3 Bucket Policy — IP Restriction

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFabricAndGateway",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-fabric-bucket",
        "arn:aws:s3:::my-fabric-bucket/*"
      ],
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": [
            "20.50.0.0/16",
            "52.236.0.0/16"
          ]
        }
      }
    }
  ]
}
```

> Replace IP ranges with your actual OPDG NAT gateway IPs and/or [Fabric Service Tags](https://learn.microsoft.com/en-us/fabric/security/security-managed-vnets-fabric-service-tags).

#### Credential Rotation Policy

| Item | Policy |
|---|---|
| IAM Access Key rotation | Every 90 days (AWS Security Hub benchmark) |
| Store location | Fabric Connection credential store *or* Azure Key Vault referenced by Pipeline |
| Monitoring | Enable AWS CloudTrail `s3:GetObject` logging; alert on calls from unknown IPs |
| Break-glass | Deactivate key immediately in IAM Console; update Fabric connection |

---

### 3.2 AWS Redshift

```mermaid
graph LR
    subgraph "Fabric Side"
    FAB_RS["Fabric / Power BI\nService (Cloud)"]
    CONN_RS["Fabric Connection\n(Redshift)"]
    OPDG_RS["OPDG\n(optional Redshift\nODBC path)"]
    end

    subgraph "AWS Side"
        RS_CLUSTER["Redshift Cluster\nor Serverless"]
        IAM_RS["IAM Role\n(GetClusterCredentials)"]
        SECRETS["AWS Secrets Manager\n(optional)"]
    end

    CONN_RS -->|DB user + password or Entra ID SSO| FAB_RS
    FAB_RS -->|Cloud connection| RS_CLUSTER
    CONN_RS -.->|Optional gateway path| OPDG_RS
    OPDG_RS -.->|ODBC 5439| RS_CLUSTER
    IAM_RS -.->|Temporary DB creds| RS_CLUSTER
    SECRETS -.->|Auto-rotation| RS_CLUSTER
```

  > **Gateway requirement**: The Amazon Redshift connector supports direct cloud connectivity and does **not** require OPDG. OPDG remains an **optional** pattern when teams choose a private routed path, reuse an existing gateway estate, or need an installed-driver execution model.

#### Authentication Options

| Method | How it works | Pros | Cons | Recommended for |
|---|---|---|---|---|
| **Database user + password** | Traditional Redshift superuser or scoped user | Universal; works with all Fabric connectors; **no gateway needed** for public endpoints | Static password; must rotate manually | Semantic Model, Dataflow Gen2, Pipeline — default method |
| **Microsoft Entra ID SSO** | Entra ID identity federated to Redshift via IAM IdP; user's token passed through | True SSO; no stored DB password; works **without gateway** (cloud connection) or with OPDG | Requires Redshift IAM federation with Entra ID; tenant admin must enable Redshift SSO setting | Semantic Model (DirectQuery) — recommended for per-user RLS |
| **IAM-based temporary credentials** | Call `redshift:GetClusterCredentials` API to get a temp DB user/password (15 min – 1 hr) | No static password; auto-expires | Requires custom script or Pipeline Web Activity to fetch creds before query | Notebooks; Pipeline (advanced) |
| **Redshift Serverless + IAM** | Workgroup attached to IAM role; federated access | Fully IAM-native; no DB passwords | Only Redshift Serverless; limited connector support | Spark notebooks via JDBC with IAM plugin |
| **AWS Secrets Manager rotation** | Secrets Manager auto-rotates Redshift password on a schedule | Creds always fresh; no manual rotation | Fabric connection must be updated after each rotation (or use Pipeline secret lookup) | Environments with strict compliance |

#### Redshift Network — Firewall Rules

| Direction | Source | Destination | Port | Protocol |
|---|---|---|---|---|
| Outbound | Fabric cloud service | Redshift public endpoint | 5439 | TCP / TLS |
| Outbound | OPDG subnet (optional path) | Redshift cluster endpoint | 5439 | TCP (Postgres wire protocol) |
| Outbound | OPDG subnet (optional path) | Redshift Serverless endpoint | 5439 | TCP |
| Inbound (AWS SG) | Fabric service IPs or OPDG NAT IP / VPN CIDR | Redshift cluster SG | 5439 | TCP |

#### ODBC Driver Configuration (on OPDG VM — optional path only)

| Setting | Value |
|---|---|
| Driver | `Amazon Redshift ODBC Driver (x64)` v2.x |
| Server | `my-cluster.xxxx.region.redshift.amazonaws.com` |
| Port | 5439 |
| Database | `analytics_db` |
| SSL Mode | `verify-full` (recommended) |
| SSL CA | Amazon root CA bundle (pre-installed with driver) |

> Install the latest **Amazon Redshift ODBC driver** only when you intentionally use the optional OPDG path. Keep versions identical across gateway nodes if that path is enabled.

#### Redshift User — Minimum Permissions

```sql
-- Create a read-only Fabric user
CREATE USER fabric_reader PASSWORD 'RotateMe!90days' NOCREATEDB NOCREATEUSER;

-- Grant access to target schema
GRANT USAGE ON SCHEMA analytics TO fabric_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO fabric_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA analytics GRANT SELECT ON TABLES TO fabric_reader;
```

---

### 3.3 Databricks on AWS

```mermaid
graph LR
    subgraph "Fabric Side"
        FAB_DB["Fabric / Power BI\nService (Cloud)"]
        CONN_DB["Fabric Connection\n(Databricks)"]
        OPDG_DB["OPDG\n(only if private\nworkspace)"]
    end

    subgraph "Databricks on AWS"
        DBRWS["Databricks\nWorkspace"]
        SQLWH["SQL Warehouse\n(Serverless / Classic)"]
        UC["Unity Catalog"]
        SCIM["SCIM / Entra ID\nSync"]
    end

    subgraph "AWS Infra"
        DBRAWS["Databricks\nControl Plane\n(*.cloud.databricks.com)"]
        S3_DBR["S3 Bucket\n(managed storage)"]
    end

    CONN_DB -->|PAT or OAuth| FAB_DB
    FAB_DB -->|HTTPS 443 cloud connection| SQLWH
    CONN_DB -.->|PAT or OAuth| OPDG_DB
    OPDG_DB -.->|HTTPS 443 via VPN| SQLWH
    SQLWH --> UC
    UC -->|fine-grained ACLs| S3_DBR
    DBRWS --- DBRAWS
    SCIM -.->|Sync groups from Entra ID| DBRWS
```

> **Gateway requirement**: The Databricks connector supports **cloud connections** (no gateway) when the Databricks workspace has a **public endpoint**. OPDG is only required when the workspace is behind **Private Link** with no public access.

#### Authentication Options

| Method | How it works | Pros | Cons | Recommended for |
|---|---|---|---|---|
| **Personal Access Token (PAT)** | Long-lived token scoped to a Databricks user | Simple; supported by all Fabric connectors; **no gateway needed** for public workspaces | User-scoped; no auto-expiry (must set manually); tied to individual | Semantic Model, Dataflow Gen2 — quick start |
| **OAuth (OIDC) / Machine-to-Machine (M2M)** | Databricks service principal + OAuth client credentials flow | No human identity; auto-token refresh; audit-friendly; **no gateway needed** for public workspaces | Requires Databricks account-level SP; setup more complex | Production Pipelines, Semantic Models — recommended for prod |
| **Username / Password** | Basic auth with Databricks user credentials | Simple; works with cloud connection (no gateway) | Less secure than OAuth; must rotate manually | Quick PoC; not recommended for prod |
| **OAuth User-Delegated (U2DP)** | End-user signs in via Entra ID → SSO into Databricks | True SSO; respects user-level Unity Catalog ACLs | Requires Entra ID federation with Databricks; not all connectors support it | Interactive / DirectQuery when per-user RLS is needed |
| **Entra ID Federated Identity** | Databricks workspace linked to Entra ID via SCIM + SSO | Groups synced automatically; unified identity plane | Initial setup effort; requires Databricks Premium | Enterprise environments with centralised identity |

#### Databricks Network — Firewall Rules

| Direction | Source | Destination | Port | Protocol | Purpose |
|---|---|---|---|---|---|
| Outbound | Fabric cloud service | `*.cloud.databricks.com` | 443 | HTTPS | Cloud connection (no gateway) to Databricks SQL Warehouse |
| Outbound | OPDG subnet (if private) | `*.cloud.databricks.com` | 443 | HTTPS | Databricks REST API + SQL Warehouse via gateway |
| Outbound | OPDG subnet (if private) | Databricks Private Link (if enabled) | 443 | HTTPS (via S2S VPN) | Private connectivity to workspace |
| AWS SG Inbound | Fabric service IPs / OPDG NAT IP / VPN CIDR | Databricks SG (data plane) | 443 | HTTPS | Allow Fabric / OPDG traffic |

#### Connection String / ODBC (on OPDG VM — only needed for private workspaces)

| Setting | Value |
|---|---|
| Driver | `Simba Spark ODBC Driver` or `Databricks ODBC Driver` v2.x |
| Host | `dbc-xxxxxxx.cloud.databricks.com` |
| Port | 443 |
| HTTP Path | `/sql/1.0/warehouses/<warehouse_id>` |
| Auth | `AuthMech=11` (OAuth M2M) or `AuthMech=3` (PAT) |
| SSL | Enabled (always) |
| Thrift Transport | `HTTP` |

> Use a **Databricks SQL Warehouse** (Serverless preferred) as the compute for Fabric connectivity. Avoid connecting to all-purpose clusters for BI workloads — they are more expensive and less optimised for concurrent queries.

#### Databricks Service Principal — Setup Checklist

1. **Create a Databricks Service Principal** at the account level (Account Console → Service Principals)
2. **Generate OAuth secret** (Client ID + Client Secret) — store Client Secret in Azure Key Vault
3. **Assign workspace access** — add the SP to the target workspace
4. **Grant Unity Catalog privileges**:
   ```sql
   GRANT USE CATALOG ON CATALOG analytics_catalog TO `fabric-sp`;
   GRANT USE SCHEMA ON SCHEMA analytics_catalog.gold TO `fabric-sp`;
   GRANT SELECT ON SCHEMA analytics_catalog.gold TO `fabric-sp`;
   ```
5. **Create or use a SQL Warehouse** — ensure the SP has CAN USE permission
6. **Configure Fabric connection** with Client ID + Client Secret + Workspace URL

#### Identity Federation — Entra ID ↔ Databricks (Optional but recommended)

| Step | Detail |
|---|---|
| 1. Enable SCIM | Databricks workspace → Admin Settings → Identity Providers → Enable SCIM provisioning |
| 2. Configure Entra ID Enterprise App | Azure Portal → Enterprise Applications → Databricks → Provisioning → Automatic |
| 3. Map attributes | `userPrincipalName` → `userName`, `displayName` → `displayName`, Security Groups → Databricks Groups |
| 4. Enable SSO | SAML 2.0 between Entra ID and Databricks (ACS URL from Databricks admin) |
| 5. Verify | Users/groups from Entra ID appear in Databricks; SSO login works; Unity Catalog ACLs reference synced groups |

> With Entra ID federation, your Databricks access model mirrors your Azure access model. Security groups used for Fabric workspace RBAC can also govern Unity Catalog permissions — single pane of glass for identity governance.

---

### 3.4 AWS SageMaker Unified Studio

> **No native Power Query connector exists** for SageMaker Unified Studio (as of March 2026). Data must be accessed through indirect paths.

```mermaid
graph LR
    subgraph "Fabric Side"
        FAB_SM["Fabric / Power BI"]
        CONN_ATH["Fabric Connection\n(Amazon Athena)"]
        CONN_S3B["Fabric Connection\n(S3 — underlying storage)"]
        OPDG_ATH["OPDG\n(Athena ODBC)"]
        NB_SM["Fabric Notebook\n(Spark)"]
    end

    subgraph "AWS SageMaker Unified Studio"
        SM_LH["SageMaker\nLakehouse"]
        GLUE["AWS Glue\nData Catalog"]
        ATH["Amazon\nAthena"]
        S3_SM["S3 Bucket\n(managed storage)"]
    end

    %% Path 1: Athena (SQL)
    CONN_ATH -->|DSN config| OPDG_ATH
    OPDG_ATH -->|ODBC 443| ATH
    ATH --> GLUE
    GLUE --> S3_SM

    %% Path 2: S3 direct
    FAB_SM -->|Shortcut or Pipeline| CONN_S3B
    CONN_S3B -->|HTTPS IAM key| S3_SM

    %% Path 3: Notebook
    NB_SM -->|Python Athena client| ATH
    NB_SM -->|Python S3 client| S3_SM

    SM_LH --- GLUE
    SM_LH --- S3_SM
```

#### Connectivity Options

| Method | How it works | Gateway? | Pros | Cons | Recommended for |
|---|---|---|---|---|---|
| **Amazon Athena connector** | SQL queries via Athena ODBC driver against Glue Data Catalog tables | **OPDG** (Athena requires ODBC driver installed on gateway) | Full SQL interface; query any table in the SageMaker Lakehouse catalog | Requires OPDG and Athena ODBC driver; Athena per-query costs | Semantic Models (Import/DQ), Dataflow Gen2 |
| **S3 Shortcut (OneLake)** | Direct shortcut to the underlying S3 bucket where SageMaker stores data | **No** | Zero-copy; no gateway; real-time access | Must know the S3 path; no SQL interface; read-only | Analytics on Parquet/Delta/Iceberg files in SageMaker storage |
| **S3 Pipeline (Copy Activity)** | Copy data from the underlying S3 bucket to Lakehouse/Warehouse | **No** (public) / OPDG (VPC endpoint) | Can transform/stage data; supports write to Fabric | Data duplication; not real-time | ETL/staging workflows |
| **Fabric Notebook** | Use `pyathena` (Python Athena client) or `boto3` to query Athena or read S3 directly | **No** | Full flexibility; combine with Spark transforms | Requires code; not suitable for direct Semantic Model binding | Data science / advanced ETL |

#### Amazon Athena ODBC — Setup on OPDG

| Setting | Value |
|---|---|
| Driver | `Amazon Athena ODBC Driver` v2.x (install on every OPDG node) |
| DSN Type | **System DSN** (not User DSN — see Limitations) |
| Region | e.g., `us-east-1` |
| S3 Output Location | `s3://athena-query-results-<account>/` |
| Authentication | IAM Access Key (Key ID + Secret) or SAML/Entra ID federation |
| SSL | Enabled (always) |

> **Important**: The Athena ODBC driver must be registered as a **System DSN**, not a User DSN. The OPDG service runs under a service account that does not load user-specific registry hives, causing `Data source name not found` errors if registered under User DSN.

#### IAM Policy — Minimum Permissions for Athena + SageMaker Lakehouse

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "FabricAthenaReadOnly",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults",
        "athena:StopQueryExecution"
      ],
      "Resource": "arn:aws:athena:*:*:workgroup/fabric-workgroup"
    },
    {
      "Sid": "GlueCatalogReadOnly",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:GetTable",
        "glue:GetTables",
        "glue:GetPartitions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3DataAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::sagemaker-lakehouse-bucket",
        "arn:aws:s3:::sagemaker-lakehouse-bucket/*"
      ]
    },
    {
      "Sid": "AthenaQueryResults",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::athena-query-results-*",
        "arn:aws:s3:::athena-query-results-*/*"
      ]
    }
  ]
}
```

---

## 4. End-to-End Flow by Fabric Feature

### 4.1 OneLake Shortcut → S3

```mermaid
sequenceDiagram
    participant User as Report User
    participant Fabric as Fabric Service
    participant OL as OneLake
    participant S3 as AWS S3 Bucket

    User->>Fabric: Open report / query Lakehouse table
    Fabric->>OL: Read Shortcut metadata
    OL->>S3: HTTPS GET (IAM Access Key in header)
    S3-->>OL: Parquet / Delta files
    OL-->>Fabric: Return data to query engine
    Fabric-->>User: Render results
```

**Key points**:
- No gateway involved — OneLake service calls S3 directly
- IAM Access Key is stored in the Fabric Shortcut connection
- Data is read at query time (no copy); caching controlled by OneLake
- S3 bucket must allow `s3:GetObject` + `s3:ListBucket` from Fabric IPs

### 4.2 Semantic Model (Import) → Redshift (Direct by Default, OPDG Optional)

```mermaid
sequenceDiagram
    participant Fabric as Fabric Service
    participant SB as Azure Service Bus
    participant GW as OPDG Node
    participant RS as AWS Redshift

  Fabric->>RS: Direct HTTPS/TLS connection (port 5439)
  Note over Fabric,RS: Auth: DB user/password or Entra ID SSO
  RS-->>Fabric: Query results
  Fabric->>Fabric: Data loaded into Semantic Model

  opt Optional routed path via OPDG
        Fabric->>SB: Refresh request
        SB->>GW: Route to healthy node
        GW->>RS: ODBC connection (port 5439, TLS via S2S VPN)
        Note over GW,RS: Auth: DB user/password or Entra ID SSO
        RS-->>GW: Query results (columnar)
        GW-->>SB: Encrypted payload
        SB-->>Fabric: Data loaded into Semantic Model
  end
```

**Key points**:
- The Amazon Redshift connector supports **cloud connections** (no gateway) when Redshift has a **public endpoint** — the Power BI / Fabric service connects directly over HTTPS/TLS
- OPDG is **not mandatory** for Redshift; it is an **optional** network/runtime pattern
- When using the optional OPDG path, the Amazon Redshift ODBC driver must be installed on every OPDG node
- **Microsoft Entra ID SSO** is supported both via cloud connection (no gateway) and via OPDG — enable the *Redshift SSO* tenant setting in the Power BI Admin Portal
- Credential stored in Fabric Connection (encrypted at rest)

### 4.3 Pipeline Copy Activity → Databricks SQL Warehouse (Cloud Connection or OPDG)

```mermaid
sequenceDiagram
    participant PIPE as Fabric Pipeline
    participant SB as Azure Service Bus
    participant GW as OPDG Node
    participant DBR as Databricks SQL WH

    alt Databricks has public endpoint (no gateway)
        PIPE->>DBR: Direct HTTPS 443 (OAuth / PAT)
        Note over PIPE,DBR: Auth: PAT, OAuth M2M, or Username/Password
        DBR-->>PIPE: Result set (Arrow/CSV)
        PIPE->>PIPE: Sink to Lakehouse / Warehouse
    else Databricks behind Private Link (gateway required)
        PIPE->>SB: Copy Activity trigger
        SB->>GW: Route request
        GW->>DBR: HTTPS 443 via S2S VPN (OAuth M2M token)
        Note over GW,DBR: Auth: Client ID + Secret → Bearer token
        DBR-->>GW: Result set (Arrow/CSV)
        GW-->>SB: Return data
        SB-->>PIPE: Sink to Lakehouse / Warehouse
    end
```

---

## 5. Security Hardening Checklist

| # | Control | AWS S3 | Redshift | Databricks | SageMaker Unified Studio |
|---|---|---|---|---|---|
| 1 | **Encrypt in transit** | TLS 1.2+ (enforced via bucket policy `aws:SecureTransport`) | SSL Mode = `verify-full` | HTTPS always (443) | TLS 1.2+ (Athena/S3 endpoints) |
| 2 | **Encrypt at rest** | SSE-S3 or SSE-KMS | AES-256 cluster encryption | Managed by Databricks (AWS KMS) | SSE-S3/SSE-KMS (underlying S3 storage) |
| 3 | **Restrict source IPs** | Bucket Policy with `aws:SourceIp` condition | Security Group inbound rules | IP Access List on workspace | Athena workgroup policies + S3 bucket policy |
| 4 | **Least-privilege IAM/DB** | `s3:GetObject`, `s3:ListBucket` only | `GRANT SELECT` on specific schemas | `GRANT SELECT` on Unity Catalog schemas | `athena:GetQueryResults`, `glue:GetTable`, `s3:GetObject` only |
| 5 | **Credential rotation** | IAM key rotation every 90 days | DB password rotation via Secrets Manager | OAuth secret rotation every 180 days | IAM key rotation every 90 days |
| 6 | **Audit logging** | CloudTrail S3 data events | Redshift audit logging (STL_QUERY) | Databricks audit logs (Account Console) | CloudTrail Athena + S3 data events |
| 7 | **Private networking** | S3 VPC Endpoint (Gateway type) | Enable private VPC subnet | Databricks Private Link | Athena VPC endpoint + S3 VPC endpoint |
| 8 | **Block public access** | `s3:PublicAccessBlock` on account level | Disable public accessibility | Disable public network access | `s3:PublicAccessBlock` on underlying buckets |
| 9 | **Monitor anomalies** | GuardDuty S3 protection | Redshift Advisor alerts | Databricks SQL alert on unusual queries | CloudWatch Athena query volume alerts |
| 10 | **Secret storage** | Azure Key Vault → Fabric Connection | Azure Key Vault → Fabric Connection | Azure Key Vault → Fabric Connection | Azure Key Vault → Fabric Connection |

---

## 6. Troubleshooting — Common Issues

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Shortcut shows "Access Denied" | IAM key lacks `s3:ListBucket` or bucket policy denies Fabric IP | Add missing IAM action; add Fabric service IP to bucket policy `Condition` |
| Redshift timeout from OPDG | Port 5439 blocked by NSG or AWS SG | Open outbound 5439 on OPDG NSG; add inbound 5439 on Redshift SG for VPN CIDR |
| Redshift SSL handshake failure | ODBC driver set to `verify-full` but CA bundle not found | Re-install Amazon Redshift ODBC driver (includes CA bundle); verify `SSLCertPath` |
| Databricks "401 Unauthorized" | PAT expired or OAuth secret rotated without updating Fabric connection | Re-generate PAT/secret; update Fabric Connection credential |
| Databricks "403 Forbidden" | SP not granted `CAN USE` on SQL Warehouse or `SELECT` in Unity Catalog | Run `GRANT` statements (see § 3.3 checklist) |
| Pipeline S3 copy returns 0 rows | Bucket or prefix path is wrong; or IAM key has access to different bucket | Verify `bucket/prefix` in connection; check IAM policy `Resource` ARN |
| VPN tunnel down | IKE phase 1/2 mismatch or BGP hold timer expired | Check Azure VPN Diagnostics + AWS VPN tunnel status; restart tunnel |
| Slow Redshift queries through OPDG | VPN bandwidth saturated or Redshift WLM queue contention | Scale VPN GW to VpnGw3+; tune Redshift WLM; use `UNLOAD` to S3 + Shortcut pattern |
| Athena query returns "Access Denied" | IAM user lacks `athena:GetQueryResults` or `glue:GetTable` | Add missing IAM actions; verify Athena workgroup permissions |
| Athena ODBC DSN not found on OPDG | Driver registered under User DSN instead of System DSN | Reconfigure as System DSN (ODBC Data Source Administrator → System DSN tab) |

---

## 7. Architecture Decision Records

### ADR-01: Prefer S3 Shortcut over Pipeline Copy for read-only analytics

- **Context**: S3 data can be consumed via OneLake Shortcut (zero-copy) or Pipeline Copy (data duplication).
- **Decision**: Use Shortcut as default for read-only analytics; reserve Pipeline Copy for transformation / staging.
- **Rationale**: Shortcuts avoid data duplication, reduce storage cost, and keep data fresh. Pipeline Copy is needed only when data must be transformed, joined with other sources, or loaded into a specific Lakehouse schema.

### ADR-02: Use OAuth M2M over PAT for Databricks production

- **Context**: Databricks supports both PAT and OAuth M2M for machine connectivity.
- **Decision**: Use OAuth M2M (service principal + client credentials) for all production workloads.
- **Rationale**: PATs are user-scoped, don't auto-expire, and create audit ambiguity. OAuth M2M ties operations to a service principal, supports automatic token refresh, and aligns with enterprise identity governance.

### ADR-03: Use direct cloud connection for Redshift by default; keep OPDG optional

- **Context**: The Amazon Redshift connector in Fabric/Power BI supports **cloud connections** (no gateway) and should be treated as the default pattern for Redshift connectivity.
- **Decision**: Use the **direct cloud connection** as the standard Redshift path. Keep OPDG available only as an **optional** routing/runtime choice when teams deliberately want a gateway-mediated or privately routed design.
- **Rationale**: Direct cloud connectivity removes unnecessary infrastructure and aligns with current connector capabilities. OPDG remains available for specific network or operational preferences, but it is not a prerequisite for Redshift integration.

### ADR-04: Use cloud connection for public Databricks workspaces; OPDG only for Private Link

- **Context**: The Databricks connector supports cloud connections (no gateway) when the workspace has a public endpoint. Auth types include PAT, OAuth (OIDC), and Username/Password.
- **Decision**: Public Databricks workspaces use **cloud connection** (no OPDG). Workspaces behind **Private Link** use OPDG with S2S VPN.
- **Rationale**: Cloud connections reduce infrastructure overhead and are supported natively by the Databricks connector in Power Query Online. Private Link workspaces have no public endpoint and therefore require OPDG to bridge the network.

### ADR-05: Access SageMaker Unified Studio data via Athena or S3 — no native connector

- **Context**: AWS SageMaker Unified Studio does not have a native Power Query connector. Data in SageMaker Lakehouse is stored in S3 and catalogued via AWS Glue Data Catalog, which is queryable via Amazon Athena.
- **Decision**: Use **Amazon Athena** connector (ODBC via OPDG) for structured/SQL queries against SageMaker Lakehouse tables. Use **S3 Shortcuts** or **S3 Pipeline connector** for direct file access to the underlying S3 storage.
- **Rationale**: Athena provides a SQL interface to the Glue Data Catalog (which SageMaker Lakehouse uses). S3 Shortcuts provide a zero-copy, gateway-free path when data is in open formats (Parquet/Delta/Iceberg). There is no benefit to waiting for a native connector when these paths cover all use cases.

---

## 8. Cost Considerations

| Component | Estimated Monthly Cost | Notes |
|---|---|---|
| Azure VPN Gateway (VpnGw2) | ~$340 | Active-active; covers S2S VPN to AWS |
| AWS VGW + VPN Connection | ~$36 + data transfer | $0.05/hr + $0.09/GB egress |
| AWS Direct Connect (1 Gbps) | ~$220 + port + partner fees | Use for high-volume Redshift extracts |
| S3 requests (GET/LIST) | $0.0004 / 1,000 GET | Shortcut reads incur S3 request charges |
| Redshift RA3 (2 nodes, ra3.xlplus) | ~$1,400 | On-demand; consider reserved for savings |
| Databricks SQL Warehouse (Small) | ~$300-600 | Serverless scales to 0 when idle |
| OPDG VM (D8s_v5 × 3 nodes) | ~$900 | Shared across all AWS + on-prem workloads |

> **Data egress is the hidden cost** — AWS charges $0.09/GB for data leaving the AWS region. Shortcuts that read large S3 datasets frequently will accumulate egress charges. Consider co-locating Fabric capacity in the region closest to the AWS region, and use delta/parquet columnar formats to minimise bytes read.

---

*End of document.*
