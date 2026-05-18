# Serensoft Data Engineering Wiki

Technical knowledge base for the Serensoft Data Engineering team — dbt patterns,
Snowflake architecture, ETL runbooks, and engineering decisions for the Ditteau platform.

---

## Quick Links

| I want to… | Start here |
|---|---|
| Understand the data platform architecture | [Architecture Overview](architecture/overview.md) |
| Write or review a dbt model | [dbt Conventions](dbt/conventions.md) |
| Set up a new school in Snowflake | [Snowflake New School Setup](runbooks/snowflake-new-school.md) |
| Look up a source system schema | [Source Systems](source-systems/README.md) |
| Log an architectural decision | [ADR Template](decisions/adr-template.md) |
| Review data quality standards | [DQ Framework](governance/dq-framework.md) |

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Data Warehouse | Snowflake | Multi-tenant; one database per school |
| Transformation | dbt Core | Hosted on dbt Cloud |
| Orchestration | dbt Cloud / Airflow | Scheduled jobs |
| Ingestion | Fivetran / Custom Python | Source-dependent |
| Version Control | GitHub | ditteau organization |
| CI/CD | GitHub Actions + dbt Cloud | PR builds, slim CI |

---

## Data Engineering Team

| Handle | Name | Primary Area | Email |
|---|---|---|---|
| LVP | Laurie | Architecture & DE Lead | laurie@serensoft.com |
| WDT | Will | Infrastructure & Security | will@serensoft.com |
| KKM | Kelly | Data Governance & Quality | kelly@serensoft.com |

---

## About this wiki

- **Source:** [github.com/ditteau/serensoft_de_wiki](https://github.com/ditteau/serensoft_de_wiki)
- Maintained by the DE team — found an error or gap? Open a PR.
- For questions about a specific area, ping the handle listed in the relevant doc.
