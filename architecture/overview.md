# Architecture Overview

This document provides a high-level view of the Ditteau data platform architecture.

---

## Platform Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SOURCE SYSTEMS                                │
│   Jenzabar CX │ Workday Student │ Banner │ Slate │ PowerFAIDS       │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        INGESTION LAYER                               │
│              Fivetran │ Custom Python │ APIs                         │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        SNOWFLAKE                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │   RAW       │  │   STAGING   │  │   MARTS     │                  │
│  │  (Sources)  │→ │   (dbt)     │→ │   (dbt)     │                  │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        CONSUMPTION                                   │
│              Ditteau App │ Reports │ APIs                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Principles

1. **One database per school** — Multi-tenant isolation at the database level
2. **Source-agnostic marts** — Downstream consumers don't need to know the SIS
3. **dbt for all transformations** — No stored procedures, no manual SQL
4. **Git-first** — All code in version control; PRs required for production changes

---

## Related Docs

- [Snowflake Environment](snowflake-environment.md)
- [Data Flow](data-flow.md)
