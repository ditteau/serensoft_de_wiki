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
| Transformation | dbt Core | integrated in Snowflake |
| Orchestration | TBD | Scheduled jobs |
| Ingestion | Snowflake Share / Custom Python | Source-dependent |
| Version Control | GitHub | ditteau organization |

---

## Data Engineering Team

| Handle | Name | Primary Area | Email | Slack |
|---|---|---|---|---|
| LVP | Laurie | Architecture & Strategic Planning | [laurie@serensoft.com](mailto:laurie@serensoft.com) | [DM](slack://user?team=T0YH9MKJR&id=U113UC6AZ) |
| WDT | Will | Systems & Security | [will@serensoft.com](mailto:will@serensoft.com) | [DM](slack://user?team=T0YH9MKJR&id=U0YGZ6BQU) |
| KKM | Kelly | Data Governance & Quality | [kelly@serensoft.com](mailto:kelly@serensoft.com) | [DM](slack://user?team=T0YH9MKJR&id=U02Q1KU9E3B) |
| RDT | Richard | Strategic Placement & Marketing | [rdt@serensoft.com](mailto:rdt@serensoft.com) | [DM](slack://user?team=T0YH9MKJR&id=U0YH9E2SK) |

---

## About this wiki

- **Source:** [github.com/ditteau/serensoft_de_wiki](https://github.com/ditteau/serensoft_de_wiki)
- Maintained by the DE team — found an error or gap? Open a PR.
- For questions about a specific area, ping the handle listed in the relevant doc.
