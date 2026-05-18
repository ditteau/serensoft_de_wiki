# Source Systems Overview

This section documents the source systems integrated with the Ditteau platform.

---

## Active Source Systems

| System | Type | Schools Using | Ingestion Method |
|---|---|---|---|
| [Jenzabar CX](jenzabar-cx.md) | SIS | Merrimack, Endicott | Fivetran (SQL Server) |
| [Jenzabar One](jenzabar-one.md) | SIS | — | Fivetran (Oracle) |
| [Workday Student](workday-student.md) | SIS | Saint Anselm | Custom API |
| [Ellucian Banner](banner.md) | SIS | Springfield | Fivetran (Oracle) |
| [Slate CRM](slate.md) | Admissions | Multiple | Fivetran / API |
| [PowerFAIDS](powerfaids.md) | Financial Aid | Multiple | Fivetran (SQL Server) |

---

## Documentation Standard

Each source system doc should include:

1. **Overview** — What the system does, vendor info
2. **Connection Details** — How we connect (Fivetran connector, API, etc.)
3. **Key Tables** — Most important tables for our use cases
4. **Schema Notes** — Quirks, gotchas, data quality issues
5. **Staging Models** — Link to dbt staging models

---

## Adding a New Source

1. Document the source system in this section
2. Set up ingestion (Fivetran or custom)
3. Create staging models in `models/staging/<source>/`
4. Add source definition to `_sources.yml`
5. Wire up to relevant intermediate/mart models
