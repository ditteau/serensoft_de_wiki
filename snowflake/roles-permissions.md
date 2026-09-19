# Roles & Permissions

RBAC (Role-Based Access Control) model for the Ditteau platform.

---

## Role Hierarchy Overview

```
                    ACCOUNTADMIN
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     SECURITYADMIN  SYSADMIN      USERADMIN
          │              │              │
          │         ┌────┴────┐         │
          │         │         │         │
          │   DITTEAU_ADMIN  ...        │
          │         │                   │
          └─────────┼───────────────────┘
                    │
            DITTEAU_ENGINEER
                    │
        ┌───────────┼───────────┐
        │           │           │
    {CODE}_DBT_*  {CODE}_*_*  ...
```

---

## Platform Roles

Account-level roles created once for the entire account:

| Role | Purpose | Access |
|------|---------|--------|
| `DITTEAU_ADMIN` | Platform administration | Full access to all schools, all environments |
| `DITTEAU_ENGINEER` | Engineering work | Cross-school; dev/test full, prod read-only |
| `GOVERNANCE_VIEWER` | Compliance and audit | Read access to PII access logs |

### Platform Role Grants

```sql
-- DITTEAU_ADMIN inherits from platform roles
GRANT ROLE SYSADMIN TO ROLE DITTEAU_ADMIN;
GRANT ROLE SECURITYADMIN TO ROLE DITTEAU_ADMIN;

-- DITTEAU_ENGINEER has cross-school access
GRANT ROLE {CODE}_DBT_DEV TO ROLE DITTEAU_ENGINEER;
GRANT ROLE {CODE}_DBT_TEST TO ROLE DITTEAU_ENGINEER;
GRANT ROLE {CODE}_READ_PROD TO ROLE DITTEAU_ENGINEER;
```

---

## School Roles (13 per School)

Each school gets 13 roles organized by function and environment:

### Transform Roles

Execute dbt builds and write to schemas:

| Role | Environment | Permissions |
|------|-------------|-------------|
| `{CODE}_TRANSFORM_DEV` | DEV | Write to all schemas |
| `{CODE}_TRANSFORM_TEST` | TEST | Write to all schemas |
| `{CODE}_TRANSFORM_PROD` | PROD | Write to all schemas |

### DBT Roles

Service account roles for automated dbt runs:

| Role | Environment | Permissions |
|------|-------------|-------------|
| `{CODE}_DBT_DEV` | DEV | Transform + metadata |
| `{CODE}_DBT_TEST` | TEST | Transform + metadata |
| `{CODE}_DBT_PROD` | PROD | Transform + metadata |

### Read Roles

Read-only access for analytics:

| Role | Environment | Permissions |
|------|-------------|-------------|
| `{CODE}_READ_DEV` | DEV | SELECT on DISTRIBUTE |
| `{CODE}_READ_TEST` | TEST | SELECT on DISTRIBUTE |
| `{CODE}_READ_PROD` | PROD | SELECT on DISTRIBUTE |

### Write Roles

Data loading via deposit_loader:

| Role | Environment | Permissions |
|------|-------------|-------------|
| `{CODE}_WRITE_DEV` | DEV | INSERT/UPDATE on DEPOSIT |
| `{CODE}_WRITE_TEST` | TEST | INSERT/UPDATE on DEPOSIT |
| `{CODE}_WRITE_PROD` | PROD | INSERT/UPDATE on DEPOSIT |

### Reporting Role

End-user analytics access:

| Role | Environment | Permissions |
|------|-------------|-------------|
| `{CODE}_REPORTING_PROD` | PROD | SELECT on DISTRIBUTE marts |

---

## Persona Roles (Governance)

Domain-scoped access roles for row-level security:

| Role | Domain Access | PII Unmask |
|------|---------------|------------|
| `{CODE}_REGISTRAR_ROLE` | student_academic: FULL | NAME, DOB |
| `{CODE}_ADVISOR_ROLE` | student_academic: SCOPED | None |
| `{CODE}_FA_ROLE` | financial_aid: FULL, admissions: FULL | FINANCIAL_AMOUNT |
| `{CODE}_IR_ANALYST_ROLE` | All domains: FULL | None (de-identified) |
| `{CODE}_FINANCE_ROLE` | student_academic: FULL, financial_aid: FULL | None |
| `{CODE}_ADMISSIONS_ROLE` | admissions: FULL, financial_aid: AGGREGATED | None |

Persona roles are granted row and column access via:
- `governance.role_domain_access` — Row access tier per domain
- `governance.role_pii_unmask` — PII classes unmasked

---

## Streamlit Owner Roles

Special roles for Streamlit in Snowflake apps:

| Role | Purpose |
|------|---------|
| `{CODE}_STREAMLIT_OWNER_{ENV}` | App ownership (no domain access) |
| `{CODE}_STREAMLIT_OWNER_{PERSONA}_{ENV}_ROLE` | Per-persona app owners |

**Important:** Streamlit owner roles must NOT appear in `role_domain_access`. Access is granted via named exceptions in `seed_streamlit_owner_exceptions.csv`.

---

## Service Accounts

Each school has three service accounts for automated processes:

| Account | Role | Purpose |
|---------|------|---------|
| `svc_{code}_dbt_dev` | `{CODE}_DBT_DEV` | DEV dbt runs |
| `svc_{code}_dbt_test` | `{CODE}_DBT_TEST` | TEST dbt runs |
| `svc_{code}_dbt_prod` | `{CODE}_DBT_PROD` | PROD dbt runs |

Service accounts authenticate via RSA key pairs, not passwords.

---

## Schema Permissions

### DEPOSIT Schema

| Role | SELECT | INSERT | UPDATE | DELETE |
|------|--------|--------|--------|--------|
| `{CODE}_READ_*` | ✓ | | | |
| `{CODE}_WRITE_*` | ✓ | ✓ | ✓ | ✓ |
| `{CODE}_TRANSFORM_*` | ✓ | | | |

### DETERGE Schema

| Role | SELECT | INSERT | UPDATE | DELETE | CREATE |
|------|--------|--------|--------|--------|--------|
| `{CODE}_READ_*` | ✓ | | | | |
| `{CODE}_TRANSFORM_*` | ✓ | ✓ | ✓ | ✓ | ✓ |

### DISTRIBUTE Schema

| Role | SELECT | INSERT | UPDATE | DELETE | CREATE |
|------|--------|--------|--------|--------|--------|
| `{CODE}_READ_*` | ✓ | | | | |
| `{CODE}_REPORTING_PROD` | ✓ | | | | |
| `{CODE}_TRANSFORM_*` | ✓ | ✓ | ✓ | ✓ | ✓ |
| Persona roles | Governed by RAP | | | | |

### GOVERNANCE Schema

| Role | SELECT | INSERT | UPDATE |
|------|--------|--------|--------|
| `DITTEAU_ADMIN` | ✓ | ✓ | ✓ |
| `{CODE}_TRANSFORM_*` | ✓ | | |
| Persona roles | Entitlement tables only | | |

---

## Warehouse Permissions

| Warehouse | Roles with USAGE |
|-----------|------------------|
| `{CODE}_TRANSFORM_DEV` | `{CODE}_TRANSFORM_DEV`, `{CODE}_DBT_DEV` |
| `{CODE}_TRANSFORM_TEST` | `{CODE}_TRANSFORM_TEST`, `{CODE}_DBT_TEST` |
| `{CODE}_TRANSFORM_PROD` | `{CODE}_TRANSFORM_PROD`, `{CODE}_DBT_PROD` |
| `{CODE}_ANALYTICS_PROD` | `{CODE}_READ_PROD`, `{CODE}_REPORTING_PROD`, persona roles |

---

## Default Role Configuration

All person users have:
- `DEFAULT_ROLE = PUBLIC`
- `DEFAULT_SECONDARY_ROLES = ()`

This ensures:
1. RCR (Restricted Caller's Rights) apps fail closed
2. No ambient privilege escalation
3. Explicit role assumption required

The `PUBLIC` role holds no domain access by design. This is asserted by `assert_public_role_holds_no_tier.sql`.

---

## Role Escalation

To perform privileged operations:

```sql
USE ROLE ACCOUNTADMIN;   -- Stepping stone only
USE ROLE USERADMIN;      -- CREATE ROLE / ALTER USER
USE ROLE SECURITYADMIN;  -- GRANT ROLE
USE ROLE SYSADMIN;       -- CREATE ROW ACCESS POLICY / MASKING POLICY
USE ROLE DITTEAU_ADMIN;  -- Writes to GOVERNANCE.*
```

**Never leave ACCOUNTADMIN active for routine work.**

---

## Common Operations

### Granting a User Access

```sql
USE ROLE SECURITYADMIN;
GRANT ROLE MERRIMACK_READ_PROD TO USER jsmith;
```

### Creating a New Persona Role

```sql
USE ROLE USERADMIN;
CREATE ROLE MERRIMACK_NEWPERSONA_ROLE;

USE ROLE SECURITYADMIN;
GRANT ROLE MERRIMACK_NEWPERSONA_ROLE TO ROLE MERRIMACK_READ_PROD;

-- Add to governance tables
USE ROLE DITTEAU_ADMIN;
INSERT INTO MERRIMACK_DD_PROD.GOVERNANCE.ROLE_DOMAIN_ACCESS ...
```

---

## Related Documentation

- [Snowflake Environment](../architecture/snowflake-environment.md) — Infrastructure
- [Row Access Policies](../runbooks/row-access-policies.md) — RAP implementation
- [New School Setup](../runbooks/snowflake-new-school.md) — Role provisioning
