# Snowflake Account Setup & Administration Guide

> Comprehensive guide for first-time Snowflake **Business Critical** deployments including account creation, security hardening, and Microsoft Entra ID integration.

[![Snowflake](https://img.shields.io/badge/Snowflake-Business%20Critical-29B5E8?style=flat&logo=snowflake)](https://www.snowflake.com)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> ⚠️ **Disclaimer:** This is an **unofficial** guide provided as-is for educational purposes. Always refer to [official Snowflake documentation](https://docs.snowflake.com) for the most current information.

---

## Table of Contents

1. [Snowflake Architecture Overview](#1-snowflake-architecture-overview)
2. [Account Creation & Management](#2-account-creation--management)
3. [Day Zero Best Practices](#3-day-zero-best-practices)
4. [Security & Role-Based Access Control](#4-security--role-based-access-control)
5. [Resource Setup & Warehouse Management](#5-resource-setup--warehouse-management)
6. [Data Protection & Governance](#6-data-protection--governance)
7. [Microsoft Entra ID Integration](#7-microsoft-entra-id-integration)
8. [Microsoft MFA Integration](#8-microsoft-mfa-integration)
9. [Monitoring & Cost Management](#9-monitoring--cost-management)
10. [Business Critical Features](#10-business-critical-features)
11. [Support & Housekeeping](#11-support--housekeeping)
12. [Quick Reference](#12-quick-reference)

---

## 1. Snowflake Architecture Overview

### Multi-Cluster Shared Data Architecture

Snowflake uses a unique architecture that differs from traditional data warehouses:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SNOWFLAKE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         CLOUD SERVICES LAYER                           │ │
│  │  Authentication | Metadata | Query Parsing | Optimization | Security   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   ETL_WH     │  │   DEV_WH     │  │   BI_WH      │  │   PROD_WH    │     │
│  │  (Compute)   │  │  (Compute)   │  │  (Compute)   │  │  (Compute)   │     │
│  │              │  │              │  │              │  │              │     │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │     │
│  │  │SSD Cache│ │  │  │SSD Cache│ │  │  │SSD Cache│ │  │  │SSD Cache│ │     │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
│              INDEPENDENTLY SCALABLE VIRTUAL WAREHOUSES                      │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      CENTRALIZED STORAGE LAYER                         │ │
│  │     Micro-partitions | Columnar | Compressed | Encrypted | Immutable   │ │
│  │                  (Cloud Object Storage: S3/Azure Blob/GCS)             │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Three-Layer Architecture

| Layer | Description | Key Features |
|-------|-------------|--------------|
| **Cloud Services** | Brain of Snowflake - handles authentication, metadata, query optimization | Always-on, shared across customers |
| **Compute (Warehouses)** | Virtual warehouses that execute queries | Independently scalable, auto-suspend/resume |
| **Storage** | Centralized cloud storage for all data | Automatic scaling, compression, encryption |

### Adaptive Caching

Snowflake uses three levels of caching to optimize performance:

| Cache Type | Location | Description | Benefit |
|------------|----------|-------------|---------|
| **Metadata Cache** | Cloud Services | Table statistics, micro-partition info | Fast query planning |
| **Data Cache (SSD)** | Virtual Warehouse | Active working set on local SSD | Reduced storage I/O |
| **Query Result Cache** | Cloud Services | Cached result sets (24-hour TTL) | Zero compute for repeated queries |

```sql
-- Query result cache reuse (no warehouse needed)
-- If same query was run in last 24 hours, results returned instantly
SELECT COUNT(*) FROM ANALYTICS.CORE.CUSTOMERS;

-- Check if result came from cache
SELECT query_id, query_text, bytes_scanned, 
       CASE WHEN bytes_scanned = 0 THEN 'Result Cache Hit' ELSE 'Executed' END as source
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
ORDER BY start_time DESC LIMIT 10;
```

---

## 2. Account Creation & Management

### 2.1 Creating a Business Critical Account

#### Prerequisites
- Must have `ORGADMIN` role
- Both source and target regions must be in the same region group (commercial regions are in `PUBLIC` group)

#### View Available Regions
```sql
USE ROLE ORGADMIN;
SHOW REGIONS;

-- Filter for specific cloud/region
SHOW REGIONS LIKE '%azure%eastus2%';
```

#### Create Business Critical Account in Azure US East 2
```sql
USE ROLE ORGADMIN;

CREATE ACCOUNT acme_prod_analytics
  ADMIN_NAME = 'admin'
  ADMIN_PASSWORD = '<YourSecurePassword123!>'
  MUST_CHANGE_PASSWORD = TRUE
  ADMIN_USER_TYPE = PERSON
  FIRST_NAME = 'System'
  LAST_NAME = 'Admin'
  EMAIL = 'snowflake-admin@yourcompany.com'
  EDITION = BUSINESS_CRITICAL
  REGION = 'azure_eastus2'
  COMMENT = 'Production Business Critical account in Azure US East 2';
```

#### Key Parameters

| Parameter | Values | Description |
|-----------|--------|-------------|
| `REGION` | `azure_eastus2`, `aws_us_east_1`, etc. | Target cloud region |
| `EDITION` | `BUSINESS_CRITICAL` | Highest security and compliance tier |
| `ADMIN_USER_TYPE` | `PERSON`, `SERVICE` | Human (enforces MFA) or service account |
| `MUST_CHANGE_PASSWORD` | `TRUE`/`FALSE` | Force password change on first login |

#### Business Critical Edition Benefits

| Feature | Benefit |
|---------|---------|
| **HIPAA & PCI DSS Support** | Compliance-ready for regulated industries |
| **Tri-Secret Secure** | Customer-managed encryption keys |
| **90-Day Time Travel** | Extended data recovery window |
| **Private Connectivity** | AWS PrivateLink / Azure Private Link |
| **Failover/Failback** | Database replication and disaster recovery |
| **PHI Data Support** | Protected Health Information handling |

#### Region IDs Reference

| Cloud | Region | Region ID |
|-------|--------|-----------|
| **AWS** | US East (N. Virginia) | `aws_us_east_1` |
| **AWS** | US West (Oregon) | `aws_us_west_2` |
| **AWS** | EU (Frankfurt) | `aws_eu_central_1` |
| **Azure** | East US 2 (Virginia) | `azure_eastus2` |
| **Azure** | West Europe (Netherlands) | `azure_westeurope` |
| **Azure** | US Gov Virginia | `azure_usgovvirginia` |
| **GCP** | US Central1 (Iowa) | `gcp_us_central1` |

---

### 2.2 Account Naming & Identifiers

#### Account URL Format
```
<orgname>-<account_name>.snowflakecomputing.com
```

**Example:** `myorg-acme_prod_analytics.snowflakecomputing.com`

#### Get Current Account Identifier
```sql
SELECT CURRENT_ORGANIZATION_NAME() || '-' || CURRENT_ACCOUNT_NAME() AS account_identifier;
```

#### Naming Conventions (Recommended)
```
<company>_<purpose>_<region_hint>

Examples:
  acme_analytics_azeast2
  acme_dataplatform_useast
  acme_warehouse_euwest
```

---

### 2.3 Renaming Accounts

#### Syntax
```sql
USE ROLE ORGADMIN;

ALTER ACCOUNT <old_name> RENAME TO <new_name>;
```

#### Example
```sql
USE ROLE ORGADMIN;

ALTER ACCOUNT old_account_name RENAME TO acme_prod_analytics;

-- Verify
SHOW ACCOUNTS LIKE 'acme_prod%';
```

#### Impact of Renaming

| Item | Impact |
|------|--------|
| **URL** | Changes immediately (old URL stops working) |
| **Connections** | All connection configs must be updated |
| **Data Shares** | Continue to work (internal locator unchanged) |
| **DNS** | May take a few minutes to propagate |

---

### 2.4 ORGADMIN Best Practices

| Principle | Recommendation |
|-----------|----------------|
| **Minimal Assignment** | Grant to 2-3 trusted users only |
| **MFA Required** | Enforce MFA for all ORGADMIN users |
| **No Day-to-Day Use** | Use ACCOUNTADMIN within the account for operations |
| **Audit Trail** | Monitor via `ORGANIZATION_USAGE.LOGIN_HISTORY` |

```sql
USE ROLE ORGADMIN;
SHOW GRANTS OF ROLE ORGADMIN;
```

---

## 3. Day Zero Best Practices

> ⚠️ **Execute these immediately after account creation** to establish a secure, cost-controlled foundation.

### 3.1 Account-Level Settings

```sql
USE ROLE ACCOUNTADMIN;

-- Set default Time Travel retention to 7 days (adjust later per database)
ALTER ACCOUNT SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- Prevent runaway queries (6-hour timeout)
ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 21600;

-- Enforce password complexity
ALTER ACCOUNT SET 
  PASSWORD_MIN_LENGTH = 14,
  PASSWORD_MAX_AGE_DAYS = 90,
  PASSWORD_MAX_RETRIES = 5,
  PASSWORD_LOCKOUT_TIME_MINS = 30;

-- Session security
ALTER ACCOUNT SET 
  SESSION_IDLE_TIMEOUT_MINS = 60,
  CLIENT_SESSION_KEEP_ALIVE = FALSE;

-- Require MFA for all users (recommended for Business Critical)
ALTER ACCOUNT SET REQUIRE_MFA_FOR_LOGIN = TRUE;
```

### 3.2 Initial Resource Monitor (Critical Safety Net)

```sql
USE ROLE ACCOUNTADMIN;

-- Create account-level resource monitor with multiple notification thresholds
CREATE OR REPLACE RESOURCE MONITOR ACCT_MONTHLY_RM
  WITH CREDIT_QUOTA = 1000
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 25 PERCENT DO NOTIFY
    ON 50 PERCENT DO NOTIFY
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND
    ON 110 PERCENT DO SUSPEND_IMMEDIATE;

ALTER ACCOUNT SET RESOURCE_MONITOR = ACCT_MONTHLY_RM;
```

> 💡 **Tip:** Enable email notifications in Snowsight: **Admin** → **Cost Management** → **Resource Monitors** → Configure notification recipients.

### 3.3 Storage Cost Optimization

Use short-lived table types for temporary/intermediate data:

| Table Type | Visibility | Time Travel | Fail-Safe | Lifespan | Use Case |
|------------|------------|-------------|-----------|----------|----------|
| **TEMPORARY** | Session only | Up to 1 day | ❌ No | Session | ETL staging, temp calculations |
| **TRANSIENT** | All users | Up to 1 day | ❌ No | Until dropped | Intermediate pipeline tables |
| **PERMANENT** | All users | Up to 90 days | ✅ 7 days | Until dropped | Production data |

```sql
-- Use TEMPORARY tables for session-scoped staging
CREATE TEMPORARY TABLE staging_data AS
SELECT * FROM external_source WHERE load_date = CURRENT_DATE();

-- Use TRANSIENT tables for intermediate pipeline data (no fail-safe = lower storage cost)
CREATE TRANSIENT TABLE pipeline_intermediate AS
SELECT * FROM raw_data WHERE processed = FALSE;
```

### 3.4 Day Zero Checklist

| Task | Priority | SQL/Action | Status |
|------|----------|------------|--------|
| Set `DATA_RETENTION_TIME_IN_DAYS = 7` | 🔴 Critical | `ALTER ACCOUNT SET...` | ☐ |
| Set `STATEMENT_TIMEOUT_IN_SECONDS = 21600` | 🔴 Critical | `ALTER ACCOUNT SET...` | ☐ |
| Create account-level resource monitor | 🔴 Critical | `CREATE RESOURCE MONITOR...` | ☐ |
| Enable email notifications for monitors | 🔴 Critical | Snowsight UI | ☐ |
| Secure ACCOUNTADMIN role | 🔴 Critical | Create dedicated admin users | ☐ |
| Configure network policy | 🔴 Critical | `CREATE NETWORK POLICY...` | ☐ |
| Subscribe to status page | 🟠 High | https://status.snowflake.com | ☐ |
| Set up support access | 🟠 High | Snowsight UI | ☐ |

---

## 4. Security & Role-Based Access Control

### 4.1 Snowflake Security at a Glance

| Pillar | Components |
|--------|------------|
| **1. Network Controls** | TLS 1.2, Network Policies (IP allowlisting), Private Link (Azure/AWS/GCP) |
| **2. Identity & Access** | SCIM provisioning, Native credentials, MFA, Key-pair auth, SAML 2.0 SSO, OAuth 2.0 |
| **3. Data Governance** | RBAC, Column-level security, Dynamic masking, Row access policies, Tagging, Classification |
| **4. Encryption** | Always encrypted in-flight and at-rest, Hierarchical key model, Auto key rotation, Tri-Secret Secure |
| **5. Data Protection** | Time Travel (up to 90 days), Fail-Safe (7 days), Cross-region replication, Failover groups |
| **6. Auditing** | Comprehensive audit trail via ACCOUNT_USAGE schema, Query history, Login history, Access history |

### 4.2 System-Defined Roles

```
                    ACCOUNTADMIN
                         │
            ┌────────────┼────────────┐
            │            │            │
       SECURITYADMIN  SYSADMIN    USERADMIN
            │            │            │
            │            │            └── PUBLIC (all users)
            │            │
            │     ┌──────┴──────┐
            │     │             │
            │  Custom Roles  Custom Roles
            │
       (manages security)
```

| Role | Description | Best Practice |
|------|-------------|---------------|
| **ACCOUNTADMIN** | Top-level role with full system access. Can manage billing, resource monitors, and all account operations. | **Never use for day-to-day operations.** Assign to 2-3 trusted admins only. |
| **SECURITYADMIN** | Manages grants and can create/modify/drop roles. Can grant privileges to roles. | Use for security administration tasks. |
| **USERADMIN** | Manages users and roles (create, modify, drop). Cannot grant privileges. | Use for user provisioning workflows. |
| **SYSADMIN** | Creates and manages databases, schemas, warehouses, and other objects. | Default role for object administration. |
| **PUBLIC** | Automatically granted to all users. | Grant minimal permissions here. |

### 4.3 RBAC Best Practices

> ⚠️ **Critical Rules:**
> 1. **Never use ACCOUNTADMIN for day-to-day operations**
> 2. **Grant minimum necessary privileges** - Follow principle of least privilege
> 3. **Create dedicated roles per application/function** - Don't share roles across different use cases
> 4. **Use role hierarchy** - Grant child roles to parent roles for inheritance

#### Create Functional & Access Roles

```sql
USE ROLE SECURITYADMIN;

-- Functional roles (what people DO)
CREATE ROLE DATA_ENGINEER;
CREATE ROLE DATA_ANALYST;
CREATE ROLE DATA_SCIENTIST;
CREATE ROLE BI_DEVELOPER;
CREATE ROLE APP_SERVICE_ACCOUNT;

-- Access roles (what they can ACCESS)
CREATE ROLE DB_RAW_READ;
CREATE ROLE DB_RAW_WRITE;
CREATE ROLE DB_ANALYTICS_READ;
CREATE ROLE DB_ANALYTICS_WRITE;
CREATE ROLE DB_SANDBOX_FULL;

-- Build hierarchy: Functional roles inherit from access roles
GRANT ROLE DB_ANALYTICS_READ TO ROLE DATA_ANALYST;
GRANT ROLE DB_ANALYTICS_READ TO ROLE DATA_SCIENTIST;
GRANT ROLE DB_ANALYTICS_WRITE TO ROLE DATA_ENGINEER;
GRANT ROLE DB_RAW_WRITE TO ROLE DATA_ENGINEER;
GRANT ROLE DB_SANDBOX_FULL TO ROLE DATA_ANALYST;
GRANT ROLE DB_SANDBOX_FULL TO ROLE DATA_SCIENTIST;

-- Grant functional roles to SYSADMIN (for management)
GRANT ROLE DATA_ENGINEER TO ROLE SYSADMIN;
GRANT ROLE DATA_ANALYST TO ROLE SYSADMIN;
GRANT ROLE DATA_SCIENTIST TO ROLE SYSADMIN;
GRANT ROLE BI_DEVELOPER TO ROLE SYSADMIN;
```

### 4.4 Network Policy (Required for Business Critical)

```sql
USE ROLE ACCOUNTADMIN;

-- Create network policy
CREATE NETWORK POLICY corp_network_policy
  ALLOWED_IP_LIST = (
    '10.0.0.0/8',           -- Internal network
    '192.168.1.0/24',       -- Office IP range
    '203.0.113.50/32'       -- VPN egress IP
  )
  BLOCKED_IP_LIST = ()
  COMMENT = 'Corporate network access only';

-- Test on single user first (don't lock yourself out!)
ALTER USER test_user SET NETWORK_POLICY = corp_network_policy;

-- Apply account-wide (after validation)
ALTER ACCOUNT SET NETWORK_POLICY = corp_network_policy;
```

### 4.5 Private Connectivity (Business Critical)

```sql
-- For Azure Private Link / AWS PrivateLink / GCP Private Service Connect
-- Contact Snowflake Support to enable private connectivity

-- Verify private connectivity status
SELECT SYSTEM$GET_PRIVATELINK_CONFIG();
```

---

## 5. Resource Setup & Warehouse Management

### 5.1 Warehouse Best Practices

| Principle | Recommendation |
|-----------|----------------|
| **Unique warehouses for unique workloads** | Separate ETL, BI, ad-hoc, and production workloads |
| **Start small, scale up** | Begin with X-SMALL and test scaling up based on query performance |
| **Always set AUTO_SUSPEND** | Prevent idle warehouses from consuming credits |
| **Use multi-cluster for concurrency** | Leverage MCW when you need to handle concurrent users |

### 5.2 Create Warehouses

```sql
USE ROLE SYSADMIN;

-- Development/ad-hoc warehouse (start small!)
CREATE WAREHOUSE DEV_WH WITH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60          -- Suspend after 1 minute idle
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Development and ad-hoc queries';

-- ETL/Pipeline warehouse  
CREATE WAREHOUSE ETL_WH WITH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'ETL and data pipeline workloads';

-- BI/Reporting warehouse (longer suspend for dashboard caching)
CREATE WAREHOUSE BI_WH WITH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 300         -- 5 minutes - keep warm for dashboards
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'BI tools and reporting';

-- Production warehouse (multi-cluster for concurrency)
CREATE WAREHOUSE PROD_WH WITH
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'STANDARD'
  COMMENT = 'Production workloads - multi-cluster';

-- Grant access to roles
GRANT USAGE ON WAREHOUSE DEV_WH TO ROLE DATA_ANALYST;
GRANT USAGE ON WAREHOUSE DEV_WH TO ROLE DATA_SCIENTIST;
GRANT USAGE ON WAREHOUSE ETL_WH TO ROLE DATA_ENGINEER;
GRANT USAGE ON WAREHOUSE BI_WH TO ROLE BI_DEVELOPER;
GRANT USAGE ON WAREHOUSE PROD_WH TO ROLE DATA_ENGINEER;
```

### 5.3 Warehouse Sizing Guide

| Size | Nodes | Use Case | Credits/Hour |
|------|-------|----------|--------------|
| X-Small | 1 | Development, testing, light queries | 1 |
| Small | 2 | Light ETL, small reports | 2 |
| Medium | 4 | Standard workloads, dashboards | 4 |
| Large | 8 | Complex queries, large ETL | 8 |
| X-Large | 16 | Heavy analytics, ML training | 16 |
| 2X-Large+ | 32+ | Enterprise-scale workloads | 32+ |

> 💡 **Tip:** Doubling warehouse size typically halves query time. Monitor with Query Profile to right-size.

### 5.4 Create Database Structure

```sql
USE ROLE SYSADMIN;

-- Raw/landing zone (use TRANSIENT for staging to save storage costs)
CREATE DATABASE RAW;
CREATE SCHEMA RAW.LANDING;
CREATE TRANSIENT SCHEMA RAW.STAGING;  -- No fail-safe = lower cost

-- Transformed/curated data
CREATE DATABASE ANALYTICS;
CREATE SCHEMA ANALYTICS.CORE;
CREATE SCHEMA ANALYTICS.MARTS;
CREATE SCHEMA ANALYTICS.REPORTING;

-- Development sandbox
CREATE DATABASE SANDBOX;
CREATE TRANSIENT SCHEMA SANDBOX.DEV;   -- Transient for dev work
CREATE TRANSIENT SCHEMA SANDBOX.TEST;

-- Grant access - RAW database
GRANT USAGE ON DATABASE RAW TO ROLE DB_RAW_READ;
GRANT USAGE ON ALL SCHEMAS IN DATABASE RAW TO ROLE DB_RAW_READ;
GRANT SELECT ON FUTURE TABLES IN DATABASE RAW TO ROLE DB_RAW_READ;

GRANT ALL ON DATABASE RAW TO ROLE DB_RAW_WRITE;
GRANT ALL ON ALL SCHEMAS IN DATABASE RAW TO ROLE DB_RAW_WRITE;
GRANT ALL ON FUTURE TABLES IN DATABASE RAW TO ROLE DB_RAW_WRITE;

-- Grant access - ANALYTICS database
GRANT USAGE ON DATABASE ANALYTICS TO ROLE DB_ANALYTICS_READ;
GRANT USAGE ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE DB_ANALYTICS_READ;
GRANT SELECT ON FUTURE TABLES IN DATABASE ANALYTICS TO ROLE DB_ANALYTICS_READ;

GRANT ALL ON DATABASE ANALYTICS TO ROLE DB_ANALYTICS_WRITE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE DB_ANALYTICS_WRITE;
GRANT ALL ON FUTURE TABLES IN DATABASE ANALYTICS TO ROLE DB_ANALYTICS_WRITE;

-- Grant access - SANDBOX database
GRANT ALL ON DATABASE SANDBOX TO ROLE DB_SANDBOX_FULL;
GRANT ALL ON ALL SCHEMAS IN DATABASE SANDBOX TO ROLE DB_SANDBOX_FULL;
GRANT ALL ON FUTURE TABLES IN DATABASE SANDBOX TO ROLE DB_SANDBOX_FULL;
```

---

## 6. Data Protection & Governance

### 6.1 Time Travel

Time Travel allows you to access historical data at any point within the retention period.

#### Configure Time Travel (Business Critical: Up to 90 Days)

```sql
USE ROLE ACCOUNTADMIN;

-- Set at account level (initial setting)
ALTER ACCOUNT SET DATA_RETENTION_TIME_IN_DAYS = 7;

USE ROLE SYSADMIN;

-- Override per database based on criticality
ALTER DATABASE ANALYTICS SET DATA_RETENTION_TIME_IN_DAYS = 90;  -- Critical data
ALTER DATABASE RAW SET DATA_RETENTION_TIME_IN_DAYS = 30;
ALTER DATABASE SANDBOX SET DATA_RETENTION_TIME_IN_DAYS = 1;     -- Minimize for sandbox
```

#### Time Travel SQL Syntax

```sql
-- Query data at a specific timestamp
SELECT * FROM ANALYTICS.CORE.CUSTOMERS
AT(TIMESTAMP => '2026-01-15 10:00:00'::TIMESTAMP);

-- Query data from a specific offset (e.g., 1 hour ago)
SELECT * FROM ANALYTICS.CORE.CUSTOMERS
AT(OFFSET => -3600);

-- Query data before a specific statement was executed
SELECT * FROM ANALYTICS.CORE.CUSTOMERS
BEFORE(STATEMENT => '<query_id>');
```

### 6.2 Zero-Copy Cloning

Cloning creates instant copies without duplicating data - metadata-only operation with no additional storage cost (until data diverges).

```sql
-- Clone entire database for testing
CREATE DATABASE ANALYTICS_TEST CLONE ANALYTICS;

-- Clone a table from a specific point in time
CREATE TABLE ANALYTICS.CORE.CUSTOMERS_BACKUP 
  CLONE ANALYTICS.CORE.CUSTOMERS
  AT(OFFSET => -3600);  -- Clone from 1 hour ago

-- Restore accidentally dropped table
UNDROP TABLE ANALYTICS.CORE.CUSTOMERS;

-- Restore accidentally dropped schema
UNDROP SCHEMA ANALYTICS.CORE;

-- Restore accidentally dropped database
UNDROP DATABASE ANALYTICS;
```

> 💡 **Use Case:** Create instant dev/test environments by cloning production. Test code on your entire production dataset without copying data!

### 6.3 Masking Policies (for PII/PHI)

```sql
USE ROLE ACCOUNTADMIN;

-- Email masking
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('DATA_ENGINEER', 'ACCOUNTADMIN') THEN val
    ELSE REGEXP_REPLACE(val, '.+@', '****@')
  END;

-- SSN masking
CREATE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN') THEN val
    ELSE 'XXX-XX-' || RIGHT(val, 4)
  END;

-- PHI masking (for HIPAA compliance)
CREATE MASKING POLICY phi_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN', 'PHI_AUTHORIZED') THEN val
    ELSE '[REDACTED-PHI]'
  END;

-- Apply to columns
ALTER TABLE ANALYTICS.CORE.CUSTOMERS MODIFY COLUMN email SET MASKING POLICY email_mask;
ALTER TABLE ANALYTICS.CORE.CUSTOMERS MODIFY COLUMN ssn SET MASKING POLICY ssn_mask;
```

### 6.4 Row Access Policies

```sql
USE ROLE ACCOUNTADMIN;

-- Region-based row access
CREATE ROW ACCESS POLICY region_access AS (region_col STRING) RETURNS BOOLEAN ->
  CASE
    WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN', 'DATA_ENGINEER') THEN TRUE
    WHEN CURRENT_ROLE() = 'ANALYST_US' AND region_col = 'US' THEN TRUE
    WHEN CURRENT_ROLE() = 'ANALYST_EU' AND region_col = 'EU' THEN TRUE
    ELSE FALSE
  END;

-- Apply to table
ALTER TABLE ANALYTICS.CORE.SALES ADD ROW ACCESS POLICY region_access ON (region);
```

### 6.5 Data Classification Tags

```sql
USE ROLE ACCOUNTADMIN;

CREATE TAG data_classification 
  ALLOWED_VALUES 'public', 'internal', 'confidential', 'restricted', 'phi';

CREATE TAG pii_type
  ALLOWED_VALUES 'email', 'ssn', 'phone', 'address', 'dob', 'name';

-- Apply to databases
ALTER DATABASE RAW SET TAG data_classification = 'confidential';
ALTER DATABASE ANALYTICS SET TAG data_classification = 'internal';

-- Apply to columns
ALTER TABLE ANALYTICS.CORE.CUSTOMERS MODIFY COLUMN email SET TAG pii_type = 'email';
ALTER TABLE ANALYTICS.CORE.CUSTOMERS MODIFY COLUMN ssn SET TAG pii_type = 'ssn';
```

---

## 7. Microsoft Entra ID Integration

### 7.1 Overview

Microsoft Entra ID (formerly Azure AD) provides:
- **Single Sign-On (SSO)** - Users authenticate once via Entra ID
- **Centralized MFA** - MFA policies managed in Azure
- **SCIM Provisioning** - Automatic user/group sync
- **Conditional Access** - Location/device-based access controls
- **Unified Audit** - Login events in Microsoft 365 Defender

### 7.2 Prerequisites

- Snowflake **Business Critical Edition**
- **Entra ID P1/P2** license (for Conditional Access)
- **Global Administrator** or **Application Administrator** role in Entra ID
- **ACCOUNTADMIN** role in Snowflake

### 7.3 Step 1: Create Entra ID Enterprise Application

1. Go to **Azure Portal** → **Microsoft Entra ID** → **Enterprise Applications**
2. Click **+ New application** → Search for **Snowflake**
3. Select the Snowflake gallery app and click **Create**
4. Note the **Application (client) ID** and **Directory (tenant) ID**

### 7.4 Step 2: Configure SAML SSO in Entra ID

1. In the Snowflake enterprise app, go to **Single sign-on** → Select **SAML**
2. Configure **Basic SAML Configuration**:

| Setting | Value |
|---------|-------|
| **Identifier (Entity ID)** | `https://<account_identifier>.snowflakecomputing.com` |
| **Reply URL (ACS URL)** | `https://<account_identifier>.snowflakecomputing.com/fed/login` |
| **Sign on URL** | `https://<account_identifier>.snowflakecomputing.com` |

3. Download the **Federation Metadata XML** or note:
   - **Login URL**
   - **Azure AD Identifier**
   - **Certificate (Base64)**

### 7.5 Step 3: Configure SAML in Snowflake

```sql
USE ROLE ACCOUNTADMIN;

-- Create SAML2 security integration
CREATE SECURITY INTEGRATION entra_id_sso
  TYPE = SAML2
  ENABLED = TRUE
  SAML2_ISSUER = 'https://sts.windows.net/<tenant-id>/'
  SAML2_SSO_URL = 'https://login.microsoftonline.com/<tenant-id>/saml2'
  SAML2_PROVIDER = 'CUSTOM'
  SAML2_X509_CERT = '-----BEGIN CERTIFICATE-----
MIIDxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END CERTIFICATE-----'
  SAML2_SP_INITIATED_LOGIN_PAGE_LABEL = 'Microsoft Entra ID'
  SAML2_ENABLE_SP_INITIATED = TRUE
  SAML2_SNOWFLAKE_ACS_URL = 'https://<account_identifier>.snowflakecomputing.com/fed/login'
  SAML2_SNOWFLAKE_ISSUER_URL = 'https://<account_identifier>.snowflakecomputing.com';

-- Verify integration
DESCRIBE SECURITY INTEGRATION entra_id_sso;
```

### 7.6 Step 4: Configure User Attribute Mapping

In Entra ID Enterprise App → **Attributes & Claims**:

| Claim Name | Source Attribute |
|------------|------------------|
| `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` | `user.userprincipalname` |
| `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` | `user.mail` |
| `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname` | `user.givenname` |
| `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname` | `user.surname` |

### 7.7 Step 5: Create Snowflake Users (Federated)

```sql
USE ROLE USERADMIN;

-- Create user that matches Entra ID UPN
CREATE USER john.doe@yourcompany.com
  TYPE = PERSON
  EMAIL = 'john.doe@yourcompany.com'
  DEFAULT_ROLE = DATA_ANALYST
  DEFAULT_WAREHOUSE = DEV_WH
  COMMENT = 'Federated via Entra ID SSO';

-- No password is set - SSO only

USE ROLE SECURITYADMIN;
GRANT ROLE DATA_ANALYST TO USER "john.doe@yourcompany.com";
```

### 7.8 Step 6: Enable SCIM Provisioning (Recommended)

SCIM automatically syncs users and groups from Entra ID to Snowflake.

#### In Snowflake:
```sql
USE ROLE ACCOUNTADMIN;

-- Create SCIM integration
CREATE SECURITY INTEGRATION entra_id_scim
  TYPE = SCIM
  SCIM_CLIENT = 'AZURE'
  RUN_AS_ROLE = 'AAD_PROVISIONER';

-- Get the SCIM endpoint and token
SELECT SYSTEM$GENERATE_SCIM_ACCESS_TOKEN('ENTRA_ID_SCIM');
```

#### In Entra ID:
1. Go to Enterprise App → **Provisioning** → **Get started**
2. Set **Provisioning Mode** to **Automatic**
3. Enter:
   - **Tenant URL**: `https://<account_identifier>.snowflakecomputing.com/scim/v2`
   - **Secret Token**: Token from Snowflake
4. Click **Test Connection** → **Save**
5. Enable provisioning

### 7.9 Verify SSO Configuration

```sql
-- Check security integrations
SHOW SECURITY INTEGRATIONS;

-- Check federated users (SSO-only = no password)
SELECT name, email, has_password, ext_authn_duo, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL AND has_password = FALSE;

-- Check login history for SSO logins
SELECT user_name, authentication_method, first_authentication_factor, event_timestamp
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE authentication_method = 'SAML2'
  AND event_timestamp >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY event_timestamp DESC;
```

---

## 8. Microsoft MFA Integration

### 8.1 MFA Options Comparison

| Method | Where Configured | Pros | Cons |
|--------|------------------|------|------|
| **Entra ID Conditional Access** | Azure Portal | Centralized, rich policies | Requires P1/P2 license |
| **Snowflake Built-in MFA (Duo)** | Snowflake | No extra license | Separate from corporate MFA |
| **Okta/Other IdP** | External IdP | Flexibility | Additional vendor |

**Recommendation:** Use **Entra ID Conditional Access** for unified MFA experience.

### 8.2 Configure Entra ID Conditional Access for Snowflake

1. Go to **Azure Portal** → **Microsoft Entra ID** → **Security** → **Conditional Access**
2. Click **+ New policy**
3. Configure:

| Setting | Value |
|---------|-------|
| **Name** | `Require MFA for Snowflake` |
| **Users** | All users (or specific groups) |
| **Cloud apps** | Select your Snowflake enterprise app |
| **Conditions** | Optional: Exclude trusted locations |
| **Grant** | **Require multifactor authentication** |
| **Session** | Sign-in frequency: 8 hours (optional) |

4. Set policy to **On** → **Create**

### 8.3 Snowflake-Side MFA Configuration

```sql
USE ROLE ACCOUNTADMIN;

-- If using Entra ID SSO, MFA is handled by Entra ID Conditional Access
-- No additional Snowflake configuration needed

-- For users NOT using SSO, enable Snowflake's built-in MFA (Duo)
ALTER USER local_admin SET EXT_AUTHN_DUO = TRUE;

-- Require MFA enrollment
ALTER USER local_admin SET MINS_TO_BYPASS_MFA = 0;

-- Account-wide MFA requirement (recommended for Business Critical)
ALTER ACCOUNT SET REQUIRE_MFA_FOR_LOGIN = TRUE;
```

### 8.4 Handling Service Accounts (No MFA)

```sql
-- Service accounts should use RSA key authentication, not MFA
CREATE USER svc_etl_pipeline
  TYPE = SERVICE
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...'
  COMMENT = 'ETL pipeline service account - RSA auth';

-- Exclude service accounts from MFA requirements
-- In Entra ID: Create user group "Snowflake Service Accounts"
-- In Conditional Access: Exclude this group from MFA policy
```

---

## 9. Monitoring & Cost Management

### 9.1 Resource Monitors

```sql
USE ROLE ACCOUNTADMIN;

-- Account-level monitor (already created in Day Zero)
-- Add warehouse-specific monitors for cost attribution

CREATE RESOURCE MONITOR dev_wh_monitor WITH
  CREDIT_QUOTA = 200
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 50 PERCENT DO NOTIFY
    ON 80 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE DEV_WH SET RESOURCE_MONITOR = dev_wh_monitor;

CREATE RESOURCE MONITOR prod_wh_monitor WITH
  CREDIT_QUOTA = 1000
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 50 PERCENT DO NOTIFY
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY;
    -- No suspend for production

ALTER WAREHOUSE PROD_WH SET RESOURCE_MONITOR = prod_wh_monitor;
```

### 9.2 Cost Attribution Tags

```sql
USE ROLE ACCOUNTADMIN;

CREATE TAG cost_center ALLOWED_VALUES 'engineering', 'analytics', 'finance', 'marketing', 'operations';
CREATE TAG project;

-- Apply to warehouses
ALTER WAREHOUSE DEV_WH SET TAG cost_center = 'engineering';
ALTER WAREHOUSE ETL_WH SET TAG cost_center = 'engineering';
ALTER WAREHOUSE BI_WH SET TAG cost_center = 'analytics';
ALTER WAREHOUSE PROD_WH SET TAG cost_center = 'operations';
```

### 9.3 Key Monitoring Queries

#### Credit Usage by Warehouse
```sql
SELECT 
  warehouse_name,
  SUM(credits_used) AS total_credits,
  SUM(credits_used) * 4.00 AS estimated_cost_usd  -- Business Critical rate
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY total_credits DESC;
```

#### Failed Logins (Security Alert)
```sql
SELECT user_name, client_ip, error_message, event_timestamp
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE is_success = 'NO'
  AND event_timestamp >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY event_timestamp DESC;
```

#### Long-Running Queries
```sql
SELECT 
  query_id, user_name, warehouse_name,
  total_elapsed_time / 1000 / 60 AS minutes,
  LEFT(query_text, 100) AS query_preview
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE total_elapsed_time > 600000  -- > 10 minutes
  AND start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

#### User and Role Overview
```sql
SELECT 
  u.name AS user_name, u.default_role, u.has_mfa, u.disabled,
  LISTAGG(g.role, ', ') AS granted_roles
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS u
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS g ON u.name = g.grantee_name
WHERE u.deleted_on IS NULL
GROUP BY u.name, u.default_role, u.has_mfa, u.disabled
ORDER BY u.name;
```

---

## 10. Business Critical Features

### 10.1 Extended Time Travel (90 Days)

```sql
ALTER DATABASE ANALYTICS SET DATA_RETENTION_TIME_IN_DAYS = 90;
```

### 10.2 Fail-Safe (7 Days Beyond Time Travel)

- **Total recovery window:** Up to 97 days (90 Time Travel + 7 Fail-safe)
- **Fail-safe:** Contact Snowflake Support for recovery

### 10.3 Tri-Secret Secure (Customer-Managed Keys)

```sql
-- Requires coordination with Snowflake Support and your cloud provider
SELECT SYSTEM$GET_SNOWFLAKE_PLATFORM_INFO();
```

### 10.4 Private Connectivity

```sql
SELECT SYSTEM$GET_PRIVATELINK_CONFIG();
SELECT SYSTEM$GET_PRIVATELINK_AUTHORIZED_ENDPOINTS();
```

### 10.5 Database Failover/Failback

```sql
-- Enable replication for disaster recovery
ALTER DATABASE ANALYTICS ENABLE REPLICATION TO ACCOUNTS myorg.dr_account;

-- Create failover group
CREATE FAILOVER GROUP analytics_failover
  OBJECT_TYPES = DATABASES
  ALLOWED_DATABASES = ANALYTICS
  ALLOWED_ACCOUNTS = myorg.dr_account
  REPLICATION_SCHEDULE = '10 MINUTE';
```

---

## 11. Support & Housekeeping

### 11.1 Setting Up Support Access

> ⚠️ **Do this immediately** - Set up support access before you need it!

#### Step 1: Email Verification
1. In Snowsight: **User Menu** → **Profile**
2. Enter your First name, Last name, and email address
3. Save and verify email via activation link

#### Step 2: Grant Support Privileges

| Privilege | Access Level | Granted By |
|-----------|--------------|------------|
| `MANAGE ORGANIZATION SUPPORT CASES` | View/manage all org cases | ORGADMIN |
| `MANAGE ACCOUNT SUPPORT CASES` | View/manage all account cases | ACCOUNTADMIN |
| `MANAGE USER SUPPORT CASES` | View/manage own cases only | ACCOUNTADMIN |

```sql
USE ROLE ACCOUNTADMIN;

-- Grant support case access to specific role
GRANT MANAGE ACCOUNT SUPPORT CASES TO ROLE SYSADMIN;
```

#### Step 3: Enable Support
1. In Snowsight: **User Menu** → **Support**
2. Click **Enable Support**
3. Accept terms

### 11.2 Subscribe to Status Updates

Subscribe to Snowflake status updates for your deployment region:
- **URL:** https://status.snowflake.com
- Select your cloud provider and region
- Enable email/SMS notifications

### 11.3 Support Severity Levels

| Severity | Response Time | Description |
|----------|---------------|-------------|
| **Sev 1** | 24x7 | Production down, data corruption |
| **Sev 2** | 12x5 (business hours) | Major feature impacted |
| **Sev 3** | 12x5 | Minor feature impacted |
| **Sev 4** | 12x5 | General questions |

### 11.4 Housekeeping Checklist

- [ ] Set up support access for 2-3 team members
- [ ] Subscribe to status updates for your region
- [ ] Document emergency contacts and escalation procedures
- [ ] Create runbooks for common administrative tasks
- [ ] Schedule regular access reviews (quarterly)

---

## 12. Quick Reference

### Account Creation SQL Template
```sql
USE ROLE ORGADMIN;

CREATE ACCOUNT <account_name>
  ADMIN_NAME = '<admin_username>'
  ADMIN_PASSWORD = '<secure_password>'
  MUST_CHANGE_PASSWORD = TRUE
  ADMIN_USER_TYPE = PERSON
  FIRST_NAME = '<first>'
  LAST_NAME = '<last>'
  EMAIL = '<email>'
  EDITION = BUSINESS_CRITICAL
  REGION = 'azure_eastus2'
  COMMENT = '<description>';
```

### Day Zero Commands
```sql
USE ROLE ACCOUNTADMIN;

-- Essential account settings
ALTER ACCOUNT SET DATA_RETENTION_TIME_IN_DAYS = 7;
ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 21600;
ALTER ACCOUNT SET PASSWORD_MIN_LENGTH = 14;
ALTER ACCOUNT SET SESSION_IDLE_TIMEOUT_MINS = 60;

-- Resource monitor
CREATE RESOURCE MONITOR ACCT_MONTHLY_RM WITH CREDIT_QUOTA = 1000
  FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 75 PERCENT DO NOTIFY ON 100 PERCENT DO SUSPEND;
ALTER ACCOUNT SET RESOURCE_MONITOR = ACCT_MONTHLY_RM;
```

### Key SQL Commands

| Task | Command |
|------|---------|
| View accounts | `SHOW ACCOUNTS;` |
| View regions | `SHOW REGIONS;` |
| Check grants | `SHOW GRANTS OF ROLE <role>;` |
| View users | `SHOW USERS;` |
| View warehouses | `SHOW WAREHOUSES;` |
| Query history | `SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())` |
| Account parameters | `SHOW PARAMETERS IN ACCOUNT;` |


### Important URLs

| Resource | URL |
|----------|-----|
| Account Access | `https://<org>-<account>.snowflakecomputing.com` |
| Snowsight | `https://app.snowflake.com` |
| Documentation | `https://docs.snowflake.com` |
| Status Page | `https://status.snowflake.com` |
| Community | `https://community.snowflake.com` |
| User Groups | `https://usergroups.snowflake.com` |

---

## References

- [Snowflake CREATE ACCOUNT](https://docs.snowflake.com/en/sql-reference/sql/create-account)
- [Account Identifiers](https://docs.snowflake.com/en/user-guide/admin-account-identifier)
- [Business Critical Edition](https://docs.snowflake.com/en/user-guide/intro-editions#business-critical-edition)
- [Time Travel](https://docs.snowflake.com/en/user-guide/data-time-travel)
- [Zero-Copy Cloning](https://docs.snowflake.com/en/user-guide/tables-storage-considerations#label-cloning-tables)
- [SAML SSO Configuration](https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-configure-saml)
- [SCIM Provisioning](https://docs.snowflake.com/en/user-guide/scim-intro)
- [Microsoft Entra ID Integration](https://learn.microsoft.com/en-us/azure/active-directory/saas-apps/snowflake-tutorial)
- [Tri-Secret Secure](https://docs.snowflake.com/en/user-guide/security-encryption-manage)
- [Private Connectivity](https://docs.snowflake.com/en/user-guide/admin-security-privatelink)
- [Resource Monitors](https://docs.snowflake.com/en/user-guide/resource-monitors)
- [Access Control](https://docs.snowflake.com/en/user-guide/security-access-control-overview)

---

## Contributing

This is an **unofficial** guide based on Snowflake best practices and documentation. Contributions welcome via pull request.

---

*Last Updated: January 2026*
