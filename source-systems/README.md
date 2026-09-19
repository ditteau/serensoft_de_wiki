# Source Systems Overview

This section documents the source systems integrated with the Ditteau platform.

---

## Active Source Systems

| System | Type | Schools Using | Ingestion Method |
|---|---|---|---|
| [Jenzabar CX](jenzabar-cx.md) | SIS | Anselm (archive), Merrimack (archive), DEMEAU | Snowflake Data Share |
| [Jenzabar One](jenzabar-one.md) | SIS | Merrimack, DEMEAU | CSV deposit_loader |
| [Workday Student](workday-student.md) | SIS | Saint Anselm, Colby, Endicott | CSV deposit_loader |
| [Ellucian Banner](banner.md) | SIS | Springfield | CSV deposit_loader |
| [Slate CRM](slate.md) | Admissions | DEMEAU | CSV deposit_loader |
| [PowerFAIDS](powerfaids.md) | Financial Aid | DEMEAU | CSV deposit_loader |

### External Reference Data

| System | Type | Target Database | Ingestion Method |
|---|---|---|---|
| IPEDS | Federal surveys | `DITTEAU_SHARED` | CSV deposit_loader |
| College Scorecard | Federal data | `DITTEAU_SHARED` | CSV deposit_loader |

---

## Primary SIS by School

| School | Primary SIS | `primary_sis` var |
|--------|-------------|-------------------|
| Saint Anselm | Workday Student | `WORKDAY` |
| Merrimack | Jenzabar One | `JENZABAR_ONE` |
| Springfield | Banner | `BANNER` |
| DEMEAU (demo) | Jenzabar One | `JENZABAR_ONE` |

Most schools maintain JCX archive data alongside their primary SIS for historical reporting.

---

## Documentation Standard

Each source system doc should include:

1. **Overview** — What the system does, vendor info
2. **Connection Details** — How we connect (Data Share, CSV deposit, etc.)
3. **Key Tables** — Most important tables for our use cases
4. **Schema Notes** — Quirks, gotchas, data quality issues
5. **Staging Models** — Link to dbt staging models

---

## Adding a New Source

1. Document the source system in this section
2. Add table definitions to `ditteau_schema_registry`
3. Register tables in `deposit_loader/source_registry.yml`
4. Run `create-tables` for the source
5. Create staging models in `models/deterge/staging/<source>/`
6. Add source definition to `_<source>_sources.yml`
7. Wire up to relevant intermediate/mart models
8. Set `has_<source>: true` in school run scripts

---

## Var Guards

Each source system is feature-flagged:

```yaml
vars:
  has_jenzabar_cx: true
  has_jenzabar_one: true
  has_workday: false
  has_banner: false
  has_slate: false
  has_powerfaids: false
```

**Important:** A flag is a claim, not a measurement. Always verify deposit tables exist before enabling.
