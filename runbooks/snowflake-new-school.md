# Runbook: Snowflake New School Setup

Complete guide for provisioning a new school in the Ditteau Data platform.

---

## Overview

Each school receives complete Snowflake isolation:
- **3 Databases:** `{CODE}_DD_DEV`, `{CODE}_DD_TEST`, `{CODE}_DD_PROD`
- **4 Schemas per database:** `deposit`, `deterge`, `distribute`, `governance`
- **4 Warehouses:** Transform (DEV/TEST/PROD) + Analytics (PROD)
- **13 Roles:** Environment-scoped access control
- **3 Service Accounts:** `svc_{code}_dbt_{dev|test|prod}`

The `generate_school.py` script automates all SQL generation.

---

## Prerequisites

### Business Prerequisites

- [ ] School contract signed
- [ ] School code assigned (uppercase, max 20 chars, e.g., `SPRINGFIELD`)
- [ ] School domain identified (e.g., `springfield.edu`)
- [ ] Primary SIS identified (Jenzabar CX, Jenzabar One, Workday, Banner)
- [ ] Credit limit approved (monthly Snowflake credits)
- [ ] Alert email configured for resource monitor notifications

### Technical Prerequisites

- [ ] Platform roles exist (`DITTEAU_ADMIN`, `DITTEAU_ENGINEER`)
  - If not, run `school_setup/platform/00_platform_roles.sql` once
- [ ] Account-level governance tags exist
  - If not, run `school_setup/platform/governance_platform.sql` once
- [ ] Access to DITTEAU_DATA Snowflake account with appropriate roles

### Required Snowflake Roles

You will need to switch between these roles during execution:

| Role | Used For |
|------|----------|
| `SYSADMIN` | Databases, warehouses, schemas, stages |
| `USERADMIN` | Creating roles and users |
| `SECURITYADMIN` | Granting roles, security policies |
| `ACCOUNTADMIN` | Resource monitors, network policies |

---

## Step 1: Generate SQL Scripts

```bash
cd ~/ditteau_data_infra/school_setup

python generate_school.py \
  --name "Springfield College" \
  --code SPRINGFIELD \
  --domain springfield.edu \
  --credit-limit 100 \
  --retention-days 1 \
  --alert-email data-alerts@springfield.edu
```

### Command Options

| Option | Required | Description |
|--------|----------|-------------|
| `--name` | Yes | Full school name (quoted) |
| `--code` | Yes | Uppercase code, max 20 chars |
| `--domain` | Yes | School's email domain |
| `--credit-limit` | No | Monthly credit limit (default: 100) |
| `--retention-days` | No | Time Travel retention (default: 1) |
| `--alert-email` | No | Resource monitor alert recipient |
| `--ip-allowlist` | No | Comma-separated CIDR blocks for network policy |

### Output

Scripts are generated to `school_setup/output/{CODE}/`:

```
output/SPRINGFIELD/
├── 01_databases_warehouses.sql
├── 02_rbac.sql
├── 03_users.sql
├── 04_resource_monitors.sql
├── 05_governance.sql
├── 06_network_policy.sql      # Only if --ip-allowlist provided
└── 07_internal_stages.sql
```

---

## Step 2: Review Generated Scripts

Before executing, review each script:

```bash
ls -la output/SPRINGFIELD/
cat output/SPRINGFIELD/01_databases_warehouses.sql
```

**Check for:**
- Correct school code in all object names
- Appropriate warehouse sizes
- Expected role names
- Correct credit limits

---

## Step 3: Execute SQL Scripts

Execute scripts **in numbered order**. Each script specifies the required role in its header comments.

### 3.1: Databases and Warehouses

```sql
-- Run as SYSADMIN
USE ROLE SYSADMIN;
```

Execute `01_databases_warehouses.sql`. This creates:
- 3 databases (`{CODE}_DD_DEV`, `{CODE}_DD_TEST`, `{CODE}_DD_PROD`)
- 4 schemas per database (`deposit`, `deterge`, `distribute`, `governance`)
- 4 warehouses (Transform DEV/TEST/PROD, Analytics PROD)

### 3.2: RBAC (Roles and Grants)

```sql
-- Requires multiple roles
USE ROLE USERADMIN;    -- For CREATE ROLE
USE ROLE SECURITYADMIN; -- For GRANT ROLE
USE ROLE SYSADMIN;      -- For object grants
```

Execute `02_rbac.sql`. This creates 13 roles:
- `{CODE}_TRANSFORM_{ENV}` — dbt build execution
- `{CODE}_DBT_{ENV}` — Service account roles
- `{CODE}_READ_{ENV}` — Read access
- `{CODE}_WRITE_{ENV}` — Data loading
- `{CODE}_REPORTING_PROD` — End-user analytics

### 3.3: Service Users

```sql
-- Run as USERADMIN then SECURITYADMIN
USE ROLE USERADMIN;
```

Execute `03_users.sql`. This creates:
- `svc_{code}_dbt_dev`
- `svc_{code}_dbt_test`
- `svc_{code}_dbt_prod`

**Important:** Passwords in this file are temporary placeholders. Rotate to RSA key-pair authentication before production use.

### 3.4: Resource Monitors

```sql
-- Run as ACCOUNTADMIN
USE ROLE ACCOUNTADMIN;
```

Execute `04_resource_monitors.sql`. This creates:
- Resource monitor with credit quota
- Alert triggers at 75%, 90%, 100%
- Suspend action at 100%

### 3.5: Governance

```sql
-- Run as ACCOUNTADMIN then SYSADMIN
USE ROLE ACCOUNTADMIN;  -- For tag references
USE ROLE SYSADMIN;      -- For policy creation
```

Execute `05_governance.sql`. This creates:
- Object-level governance tags
- Row access policy infrastructure
- Masking policy infrastructure
- Entitlement tables (`role_domain_access`, `user_domain_access`)

### 3.6: Network Policy (Optional)

Only generated if `--ip-allowlist` was provided.

```sql
-- Run as ACCOUNTADMIN
USE ROLE ACCOUNTADMIN;
```

Execute `06_network_policy.sql`. This creates:
- Network policy with IP allowlist
- Policy attachment to service users

### 3.7: Internal Stages

```sql
-- Run as SYSADMIN
USE ROLE SYSADMIN;
```

Execute `07_internal_stages.sql`. This creates:
- Internal stage `INGEST_STAGE` in each DEPOSIT schema
- File formats (`csv_standard`, `csv_pipe`, `csv_tab`)

---

## Step 4: Post-Execution Configuration

### 4.1: Rotate Service Account Credentials

Replace temporary passwords with RSA key-pair authentication:

```bash
# Generate key pair
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out svc_springfield_dbt_prod.p8 -nocrypt
openssl rsa -in svc_springfield_dbt_prod.p8 -pubout -out svc_springfield_dbt_prod.pub

# Set public key in Snowflake
USE ROLE SECURITYADMIN;
ALTER USER svc_springfield_dbt_prod SET RSA_PUBLIC_KEY='<contents of .pub file>';
```

Store private keys in `~/.ssh/` with appropriate permissions (600).

### 4.2: Configure dbt Profile

Add targets to `~/.dbt/profiles.yml`:

```yaml
ditteau_data_transform:
  target: springfield_dev
  outputs:
    springfield_dev:
      type: snowflake
      account: xy12345.us-east-1
      user: svc_springfield_dbt_dev
      private_key_path: ~/.ssh/svc_springfield_dbt_dev.p8
      role: SPRINGFIELD_DBT_DEV
      warehouse: SPRINGFIELD_TRANSFORM_DEV
      database: SPRINGFIELD_DD_DEV
      schema: deterge
      threads: 4

    springfield_test:
      # ... similar for test

    springfield_prod:
      # ... similar for prod
```

### 4.3: Create Run Script

Create `scripts/run_springfield_dev.sh`:

```bash
#!/bin/bash
set -e

VARS='{
  "school_code": "SPRINGFIELD",
  "has_banner": true,
  "has_jenzabar_cx": false,
  "has_jenzabar_one": false,
  "has_workday": false,
  "has_slate": false,
  "has_powerfaids": false,
  "primary_sis": "BANNER"
}'

dbt "$@" --target springfield_dev --vars "$VARS"
```

### 4.4: Create Deposit Tables

For each source system the school uses:

```bash
cd ~/ditteau_data_infra/deposit_loader

# For Banner (Springfield's SIS)
python deposit_loader.py --school SPRINGFIELD create-tables banner

# Verify
python deposit_loader.py --school SPRINGFIELD list-tables banner
```

### 4.5: Grant Platform Role Access

```sql
USE ROLE SECURITYADMIN;

-- Grant school roles to platform roles
GRANT ROLE SPRINGFIELD_DBT_DEV TO ROLE DITTEAU_ENGINEER;
GRANT ROLE SPRINGFIELD_DBT_TEST TO ROLE DITTEAU_ENGINEER;
GRANT ROLE SPRINGFIELD_READ_PROD TO ROLE DITTEAU_ENGINEER;

GRANT ROLE SPRINGFIELD_DBT_DEV TO ROLE DITTEAU_ADMIN;
GRANT ROLE SPRINGFIELD_DBT_TEST TO ROLE DITTEAU_ADMIN;
GRANT ROLE SPRINGFIELD_DBT_PROD TO ROLE DITTEAU_ADMIN;
```

---

## Step 5: Verification

### 5.1: Verify Objects Created

```sql
-- Databases
SHOW DATABASES LIKE 'SPRINGFIELD%';

-- Warehouses
SHOW WAREHOUSES LIKE 'SPRINGFIELD%';

-- Roles
SHOW ROLES LIKE 'SPRINGFIELD%';

-- Users
SHOW USERS LIKE 'SVC_SPRINGFIELD%';

-- Resource monitors
SHOW RESOURCE MONITORS LIKE 'SPRINGFIELD%';
```

### 5.2: Test dbt Connection

```bash
cd ~/ditteau_data_transform

PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_springfield_dev.sh debug
```

### 5.3: Test Data Load

```bash
cd ~/ditteau_data_infra/deposit_loader

# Load a test file
python deposit_loader.py --school SPRINGFIELD \
  validate banner spriden ~/inbox/banner/spriden.csv

python deposit_loader.py --school SPRINGFIELD \
  load banner spriden ~/inbox/banner/spriden.csv

# Verify
python deposit_loader.py --school SPRINGFIELD status
```

### 5.4: Test dbt Build

```bash
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_springfield_dev.sh build --select stg_banner__spriden
```

---

## Step 6: Commit and Document

### 6.1: Archive Generated Scripts

```bash
cd ~/ditteau_data_infra/school_setup
mv output/SPRINGFIELD springfield/
git add springfield/
git commit -m "infra: add Springfield College provisioning scripts"
git push
```

### 6.2: Update School Inventory

Add the school to any tracking systems or documentation.

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| `Object already exists` | Re-running scripts | Scripts are mostly idempotent (`IF NOT EXISTS`); safe to ignore |
| `Insufficient privileges` | Wrong role active | Check script header for required role; `USE ROLE <role>` |
| `Resource monitor not found` | Execution order | Run scripts in numbered order |
| dbt `does not exist or not authorized` | Missing grants | Re-run `02_rbac.sql`; check role grants |
| `Authentication failed` | Key pair issue | Verify private key path and permissions (600) |
| `Warehouse suspended` | Credit limit reached | Check resource monitor; increase limit or wait for reset |

---

## Appendix: What Each Script Creates

### 01_databases_warehouses.sql

| Object | Count | Pattern |
|--------|-------|---------|
| Databases | 3 | `{CODE}_DD_{DEV\|TEST\|PROD}` |
| Schemas | 12 | 4 per database |
| Warehouses | 4 | Transform ×3, Analytics ×1 |

### 02_rbac.sql

| Role | Purpose |
|------|---------|
| `{CODE}_TRANSFORM_{ENV}` | Execute dbt builds |
| `{CODE}_DBT_{ENV}` | Service account role |
| `{CODE}_READ_{ENV}` | Read access to DISTRIBUTE |
| `{CODE}_WRITE_{ENV}` | Write access to DEPOSIT |
| `{CODE}_REPORTING_PROD` | End-user analytics |

### 05_governance.sql

| Object | Purpose |
|--------|---------|
| Governance tags | Object classification |
| `role_domain_access` | Row access entitlements |
| `user_domain_access` | User-specific entitlements |
| Row access policies | Domain-scoped filtering |
| Masking policies | PII protection |

---

## Related Documentation

- [Snowflake Environment](../architecture/snowflake-environment.md)
- [Roles & Permissions](../snowflake/roles-permissions.md)
- [Source Refresh](source-refresh.md)
- [Row Access Policies](row-access-policies.md)
