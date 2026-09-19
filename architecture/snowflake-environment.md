# Snowflake Environment

Multi-tenant architecture and infrastructure design for the Ditteau Data platform.

---

## Multi-Tenancy Design

Each school receives complete isolation:

| Component | Pattern | Per School |
|-----------|---------|------------|
| Databases | `{CODE}_DD_{ENV}` | 3 (DEV, TEST, PROD) |
| Schemas | `deposit`, `deterge`, `distribute`, `governance` | 4 per database |
| Warehouses | Transform + Analytics | 4 total |
| Roles | Environment-scoped access | 13 |
| Service Accounts | `svc_{code}_dbt_{env}` | 3 |

This isolation ensures:
- FERPA compliance through physical data separation
- Per-school billing via warehouse metering
- Independent deployment cycles per environment

---

## Database Naming

```
{SCHOOL_CODE}_DD_{ENVIRONMENT}
```

| School | DEV | TEST | PROD |
|--------|-----|------|------|
| Merrimack | `MERRIMACK_DD_DEV` | `MERRIMACK_DD_TEST` | `MERRIMACK_DD_PROD` |
| Saint Anselm | `ANSELM_DD_DEV` | `ANSELM_DD_TEST` | `ANSELM_DD_PROD` |
| DEMEAU (demo) | `DEMEAU_DD_DEV` | `DEMEAU_DD_TEST` | `DEMEAU_DD_PROD` |
| Springfield | `SPRINGFIELD_DD_DEV` | `SPRINGFIELD_DD_TEST` | `SPRINGFIELD_DD_PROD` |

---

## Schema Layout

Each database contains four schemas aligned to the medallion architecture:

| Schema | Layer | Purpose |
|--------|-------|---------|
| `DEPOSIT` | Bronze | Raw source data landing zone |
| `DETERGE` | Silver | Cleaned, conformed, integrated data |
| `DISTRIBUTE` | Gold | Analytics-ready dimensional models |
| `GOVERNANCE` | N/A | Access control objects, entitlement tables |

---

## Warehouse Strategy

Four warehouses per school provide workload isolation:

| Warehouse | Pattern | Purpose |
|-----------|---------|---------|
| Transform DEV | `{CODE}_TRANSFORM_DEV` | Development builds |
| Transform TEST | `{CODE}_TRANSFORM_TEST` | Testing and validation |
| Transform PROD | `{CODE}_TRANSFORM_PROD` | Production dbt runs |
| Analytics PROD | `{CODE}_ANALYTICS_PROD` | Dashboard queries, ad-hoc analysis |

### Sizing

| Warehouse | Size | Auto-Suspend | Auto-Resume |
|-----------|------|--------------|-------------|
| Transform (all) | X-Small | 60 seconds | Yes |
| Analytics PROD | Small | 300 seconds | Yes |

Transform warehouses are small because dbt runs are infrequent. Analytics warehouses may be larger to support concurrent dashboard users.

---

## Resource Monitors

Each school has a resource monitor with credit limits:

```sql
CREATE RESOURCE MONITOR {CODE}_MONITOR
  WITH CREDIT_QUOTA = 100
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;
```

Monitors are attached to all school warehouses to prevent runaway costs.

---

## Internal Stages

Each school's DEPOSIT schema includes an internal stage for data loading:

```
@{CODE}_DD_{ENV}.DEPOSIT.INGEST_STAGE/
  jenzabar_cx/<table>.csv
  powerfaids/<table>.csv
  workday_student/<table>.csv
  slate/<table>.csv
  manual/<file>.csv
```

File formats defined:
- `csv_standard` — Standard CSV (comma-delimited, header row)
- `csv_pipe` — Pipe-delimited
- `csv_tab` — Tab-delimited

---

## Shared Database

Platform-wide reference data lives in `DITTEAU_SHARED`:

| Schema | Content |
|--------|---------|
| `IPEDS` | Federal IPEDS survey data |
| `SCORECARD` | College Scorecard data |

All schools have read access to shared data. Loads use the deposit_loader with explicit `--database DITTEAU_SHARED` flags.

---

## Platform Roles

Account-level roles (created once, not per-school):

| Role | Purpose |
|------|---------|
| `DITTEAU_ADMIN` | Full access to all schools, all environments |
| `DITTEAU_ENGINEER` | Cross-school access; dev/test full, prod read-only |
| `GOVERNANCE_VIEWER` | Read access to governance objects (PII access logs) |

---

## Governance Tags

Account-level tags for data classification:

| Tag | Values |
|-----|--------|
| `SENSITIVITY_LEVEL` | `PUBLIC`, `INTERNAL`, `CONFIDENTIAL`, `RESTRICTED` |
| `COMPLIANCE_TYPE` | `FERPA`, `PCI`, `HIPAA`, `NONE` |
| `DATA_DOMAIN` | `STUDENT_ACADEMIC`, `FINANCIAL_AID`, `ADMISSIONS`, etc. |
| `CONTAINS_PII` | `TRUE`, `FALSE` |
| `DATA_OWNER` | `REGISTRAR`, `FINANCIAL_AID`, `BURSAR`, `ADMISSIONS`, `IR` |

Tags are applied at the table and column level to enable governance queries against `ACCOUNT_USAGE`.

---

## Generated SQL Files

School setup generates 6-7 SQL files, executed in numbered order:

| File | Role Required | Creates |
|------|---------------|---------|
| `01_databases_warehouses.sql` | SYSADMIN | 3 databases, 4 warehouses |
| `02_rbac.sql` | USERADMIN/SECURITYADMIN/SYSADMIN | 13 roles, grants |
| `03_users.sql` | USERADMIN/SECURITYADMIN | 3 service accounts |
| `04_resource_monitors.sql` | ACCOUNTADMIN | 1 resource monitor |
| `05_governance.sql` | ACCOUNTADMIN/SYSADMIN | Tags, policies, entitlement tables |
| `06_network_policy.sql` | ACCOUNTADMIN | Network policy (if IP allowlist provided) |
| `07_internal_stages.sql` | SYSADMIN | Internal stage, file formats |

---

## Provisioning New Schools

Use the `generate_school.py` script:

```bash
cd ditteau_data_infra/school_setup
python generate_school.py \
  --name "College Name" \
  --code COLLEGECODE \
  --domain college.edu \
  --credit-limit 100 \
  --retention-days 1 \
  --alert-email alerts@college.edu
```

See [Snowflake New School Setup](../runbooks/snowflake-new-school.md) for the full onboarding runbook.

---

## Environment Promotion

| Stage | Purpose | Access |
|-------|---------|--------|
| DEV | Development, experimentation | Engineers (full) |
| TEST | Integration testing, UAT | Engineers (full), analysts (read) |
| PROD | Production analytics | Analysts (read), apps (read), ETL (write) |

Promotion follows: DEV → TEST → PROD. No direct changes to PROD without TEST validation.

---

## Related Documentation

- [Roles & Permissions](../snowflake/roles-permissions.md) — RBAC model
- [Data Flow](data-flow.md) — Medallion architecture
- [New School Setup](../runbooks/snowflake-new-school.md) — Onboarding runbook
