# ADR-001: Store External Reference Data in a Shared Snowflake Database

**Status:** Accepted

**Date:** 2026-05-20

**Author:** LVP

---

### Context

The Ditteau Data platform serves multiple higher education institutions from a single
dbt project, with each school's data isolated in its own Snowflake database
(`{SCHOOL}_DD_{ENV}`). During the initial build of the IPEDS peer benchmarking
stack (May 2026), we needed to load three IPEDS survey tables and a BLS CPI index
into the platform.

Both datasets are public federal data — identical for every school, containing no
PII, with no school-specific variation. The question was where to store them.

The existing pattern would place them in each school's `DEPOSIT` schema, requiring
a separate load per school and per environment. With five active schools and three
environments each, that means up to 15 redundant copies of the same data that must
stay in sync.

---

### Decision

External reference data that is public, school-agnostic, and shared across all
Ditteau institutions is stored once in a dedicated `DITTEAU_SHARED` Snowflake
database rather than in each school's deposit schema.

Schemas within `DITTEAU_SHARED` are organized by source system:

| Schema | Contents |
|--------|----------|
| `DITTEAU_SHARED.REFERENCE` | dbt-managed seed tables (CPI index, etc.) |
| `DITTEAU_SHARED.IPEDS` | NCES IPEDS survey tables |

A new dbt variable `external_sources_database` (defaulting to `DITTEAU_SHARED`) is
used in `_ipeds_sources.yml` and seed `+database` configs, so all schools resolve
to the shared database at runtime without any per-school configuration.

The Deposit Loader (`ditteau_data_infra/deposit_loader/deposit_loader.py`) was
extended with `--database`, `--schema`, `--role`, and `--warehouse` override flags
to support loading into `DITTEAU_SHARED` without requiring a `--school` argument.

---

### Consequences

#### Pros

- **Single load, all schools benefit.** Annual IPEDS and CPI refreshes happen once
  instead of once per school. Five schools → 5× less operational work per cycle.
- **No sync drift.** Impossible for two schools to be running against different
  vintages of the same federal dataset.
- **Clean separation of concerns.** Public reference data is clearly separated from
  school-specific PII-bearing data at the database level, which reinforces our
  security boundary documentation.
- **Scales naturally.** Adding a sixth school requires no IPEDS/CPI work at all —
  the new school's dbt run picks up `DITTEAU_SHARED` automatically.
- **Extensible.** `DITTEAU_SHARED` becomes the natural home for future federal
  datasets: NSC Clearinghouse CDR data, IPEDS Finance, Common Data Set inputs, etc.

#### Cons

- **Cross-database permission surface.** School dbt roles must be granted `SELECT`
  on `DITTEAU_SHARED` schemas in addition to their own databases. This is a new
  permission dependency that must be included in new school setup runbooks.
- **Single point of failure.** A permissions outage or accidental drop of
  `DITTEAU_SHARED` would break all schools simultaneously rather than just one.
- **Shared write ownership.** The database has no school-scoped write isolation —
  a `TRUNCATE` by any operator affects all schools. Documented in the External Data
  Sources runbook; mitigated by restricting writes to `SYSADMIN` / a dedicated
  `DITTEAU_SHARED_WRITE` role (to be created).
- **`DITTEAU_SHARED` has no DEV/TEST/PROD variants.** All environments read the
  same data. This is acceptable for immutable annual federal snapshots but would be
  a problem for any external source that needs environment-specific data.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Per-school deposit (status quo)** | No new infrastructure; consistent with existing pattern | N loads per refresh cycle; drift risk; redundant storage |
| **Shared DB (chosen)** | One load; no drift; clean security boundary | Cross-database permissions; single point of failure |
| **Point Anselm at Merrimack's deposit** | No new database needed | Wrong ownership model; fragile if Merrimack is offboarded; not scalable |
| **dbt source override var per school** | Flexible | Same data still loaded N times; just routed differently |

---

### References

- [External Data Sources Runbook](../runbooks/external-data-sources.md)
- [Deposit Loader README](../../ditteau_data_infra/deposit_loader/Deposit_Loader_README.md)
- `ditteau_data_transform/models/deterge/staging/ipeds/_ipeds_sources.yml`
- `ditteau_data_transform/dbt_project.yml` — `external_sources_database` var and `seed_cpi_index` config
- `ditteau_data_infra/deposit_loader/deposit_loader.py` — `--database`, `--schema`, `--role`, `--warehouse` flags (added May 2026)
