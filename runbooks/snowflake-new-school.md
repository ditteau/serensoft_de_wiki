# Runbook: Snowflake New School Setup

Step-by-step guide for setting up a new school in Snowflake.

---

## Prerequisites

- [ ] School contract signed
- [ ] `schooltag` assigned (e.g., `merrimack`, `anselm`)
- [ ] Source system identified
- [ ] Fivetran connector configured (if applicable)

---

## Step 1: Create Database

```sql
-- Run as ACCOUNTADMIN or SYSADMIN
CREATE DATABASE IF NOT EXISTS <SCHOOLTAG>_DB
    COMMENT = 'Data warehouse for <School Name>';
```

---

## Step 2: Create Schemas

```sql
USE DATABASE <SCHOOLTAG>_DB;

-- Raw data from sources
CREATE SCHEMA IF NOT EXISTS RAW;

-- dbt staging models
CREATE SCHEMA IF NOT EXISTS STAGING;

-- dbt mart models
CREATE SCHEMA IF NOT EXISTS MARTS;

-- Analytics/reporting layer
CREATE SCHEMA IF NOT EXISTS ANALYTICS;
```

---

## Step 3: Create Warehouse (if needed)

```sql
CREATE WAREHOUSE IF NOT EXISTS <SCHOOLTAG>_WH
    WAREHOUSE_SIZE = 'XSMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    COMMENT = 'Compute for <School Name>';
```

---

## Step 4: Set Up Roles & Permissions

```sql
-- Create school-specific roles
CREATE ROLE IF NOT EXISTS <SCHOOLTAG>_READER;
CREATE ROLE IF NOT EXISTS <SCHOOLTAG>_WRITER;
CREATE ROLE IF NOT EXISTS <SCHOOLTAG>_ADMIN;

-- Grant database access
GRANT USAGE ON DATABASE <SCHOOLTAG>_DB TO ROLE <SCHOOLTAG>_READER;
GRANT USAGE ON DATABASE <SCHOOLTAG>_DB TO ROLE <SCHOOLTAG>_WRITER;
GRANT ALL ON DATABASE <SCHOOLTAG>_DB TO ROLE <SCHOOLTAG>_ADMIN;

-- Grant schema access (repeat for each schema)
GRANT USAGE ON ALL SCHEMAS IN DATABASE <SCHOOLTAG>_DB TO ROLE <SCHOOLTAG>_READER;
GRANT SELECT ON ALL TABLES IN DATABASE <SCHOOLTAG>_DB TO ROLE <SCHOOLTAG>_READER;
```

---

## Step 5: Configure dbt

1. Add school to `dbt_project.yml` vars
2. Create environment in dbt Cloud
3. Set up CI job for PR builds
4. Set up production job

---

## Step 6: Verify Setup

- [ ] Database and schemas created
- [ ] Roles and permissions configured
- [ ] Fivetran syncing to RAW schema
- [ ] dbt project builds successfully
- [ ] Sample data flowing through to MARTS

---

## Troubleshooting

| Issue | Solution |
|---|---|
| Permission denied | Check role grants; ensure user has correct role active |
| Fivetran sync failing | Check Fivetran dashboard; verify network/credentials |
| dbt build errors | Check `dbt debug`; verify profiles.yml |
