# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

This is the **Serensoft Data Engineering Wiki** — a Docsify-powered documentation site for the Ditteau platform data engineering team. It documents dbt conventions, Snowflake architecture, ETL runbooks, and architectural decisions for the multi-tenant higher education data platform.

**Framework:** Docsify (client-side rendering, no build process)
**Hosting:** GitHub Pages at `https://github.com/ditteau/serensoft_de_wiki`
**Repository:** `ditteau/serensoft_de_wiki`

---

## Working with this Wiki

### Local Development

To preview changes locally:

```bash
# Serve with Python (recommended)
python -m http.server 8000

# Or with Node
npx http-server

# Visit: http://localhost:8000
```

**No build step required** — Docsify renders markdown in the browser. Edit `.md` files and refresh to see changes.

### Deployment

Push to `main` branch → GitHub Pages auto-deploys. The `.nojekyll` file ensures GitHub Pages serves content without Jekyll processing.

---

## File Structure

```
/
├── index.html          # Docsify configuration (plugins, theme, search)
├── _sidebar.md         # Left sidebar navigation menu
├── _navbar.md          # Top navigation bar with external links
├── README.md           # Home page (team roster, quick links, tech stack)
│
├── architecture/       # Data platform architecture docs
├── dbt/                # dbt development standards
├── snowflake/          # Snowflake-specific documentation
├── source-systems/     # Source system integration docs
├── runbooks/           # Operational procedures
├── governance/         # Data quality & compliance
├── decisions/          # Architectural Decision Records (ADRs)
└── assets/             # Static assets (favicon, etc.)
```

### Navigation Files

- **`_sidebar.md`**: Defines the left sidebar menu. Update when adding new pages.
- **`_navbar.md`**: Top navigation links (Google Drive, Snowflake, external wiki).
- **`README.md`**: Home page content with quick links table and team roster.

When adding new documentation pages, always update `_sidebar.md` to make them discoverable.

---

## Content Standards

### Document Structure

All documentation should follow this pattern:

```markdown
# Page Title

Brief introduction explaining purpose and scope.

---

## Section Headers

Content organized with h2 (##) and h3 (###) headers.

- Use horizontal rules (`---`) to separate major sections
- Use tables for structured data
- Use code blocks with language tags (```sql, ```python, ```bash)
```

### Docsify Syntax Highlighting

Supported languages (configured in `index.html`):
- `sql`
- `python`
- `bash`
- `yaml`

Always use language tags in code fences:

````markdown
```sql
SELECT * FROM dim_student
```
````

### Internal Links

Use relative paths for internal wiki links:

```markdown
[dbt Conventions](dbt/conventions.md)
[ADR Template](decisions/adr-template.md)
```

External links open in new tabs automatically for Google Drive links.

---

## Architectural Decision Records (ADRs)

When documenting decisions, use the ADR template at `decisions/adr-template.md`:

1. Copy `adr-template.md` to `adr-NNN-title-in-kebab-case.md`
2. Fill in status, context, decision, consequences
3. Add link to `_sidebar.md` under "Decisions (ADRs)"

ADR numbering: Use sequential 3-digit numbers (001, 002, 003...)

---

## dbt Documentation Standards

When documenting dbt patterns or conventions, include:

1. **Context**: Why this pattern exists (multi-tenancy, source optionality, etc.)
2. **Code examples**: Real SQL snippets showing the pattern in use
3. **Required elements**: Metadata columns, tags, var guards
4. **Cross-references**: Link to related docs (macros, testing, etc.)

Example dbt standards documented in `dbt/conventions.md`:
- 5 required metadata columns via `add_source_metadata()` macro
- Var guard pattern: `{{ config(enabled=var('has_<source>', false)) }}`
- Staging tags: `['staging', '<source_system>']`
- Naming: `stg_<source>__<entity>`
- Materializations: views for staging, tables for marts

---

## Runbook Standards

Operational runbooks (in `runbooks/`) should be:

1. **Step-by-step**: Numbered procedures with clear actions
2. **Context-aware**: Explain why each step matters
3. **Complete**: Include prerequisites, validation checks, rollback steps
4. **Cross-referenced**: Link to related architecture or governance docs

Example: `runbooks/snowflake-new-school.md` documents the 6-step process for onboarding new schools.

---

## Team & Ownership

The DE team uses handles in documentation:

| Handle | Name | Primary Area |
|--------|------|--------------|
| LVP | Laurie | Architecture & Strategic Planning |
| WDT | Will | Systems & Security |
| KKM | Kelly | Data Governance & Quality |
| RDT | Richard | Strategic Placement & Marketing |

When documenting ownership or contacts, use these handles.

---

## Platform Context

Key facts about the Ditteau platform (for context when writing docs):

### Multi-Tenant Architecture
- One Snowflake database per school: `{SCHOOLTAG}_DB`
- Shared reference data in `DITTEAU_SHARED` database
- Schools have different source systems → dbt var guards required

### Source Systems (6 active)
- **SIS**: Jenzabar CX, Jenzabar One, Workday Student, Ellucian Banner
- **CRM**: Slate
- **Financial Aid**: PowerFAIDS

### Tech Stack
- **Data Warehouse**: Snowflake
- **Transformation**: dbt Core (only tool for transformations; no stored procedures)
- **Ingestion**: Fivetran (most sources) or custom Python
- **Orchestration**: TBD (scheduled jobs)
- **Version Control**: GitHub (ditteau organization)

### dbt Standards
- Staging: `view` materialization, 1:1 with source tables
- Intermediate: `ephemeral` or `view`
- Marts: `table` materialization
- All models: 5 required metadata columns
- Optional sources: Use `var('has_<source>', false)` guards

---

## Placeholder Pages

These pages are referenced in `_sidebar.md` but not yet written:

**Architecture:**
- `architecture/snowflake-environment.md`
- `architecture/data-flow.md`

**dbt:**
- `dbt/project-structure.md`
- `dbt/testing.md`
- `dbt/macros.md`

**Snowflake:**
- `snowflake/sql-style.md`
- `snowflake/roles-permissions.md`
- `snowflake/performance.md`

**Source Systems:**
- `source-systems/jenzabar-cx.md`
- `source-systems/jenzabar-one.md`
- `source-systems/workday-student.md`
- `source-systems/banner.md`
- `source-systems/slate.md`
- `source-systems/powerfaids.md`

**Governance:**
- `governance/data-contracts.md`

**Runbooks:**
- `runbooks/source-refresh.md`

When creating these pages, follow the content standards above and update `_sidebar.md` if needed.

---

## Docsify Configuration

Key settings in `index.html`:

```javascript
window.$docsify = {
  name: 'Serensoft DE Wiki',
  repo: 'https://github.com/ditteau/serensoft_de_wiki',
  loadSidebar: true,
  loadNavbar: true,
  subMaxLevel: 3,          // Auto-expand h1/h2/h3 in sidebar
  search: { depth: 4 },    // Search across all headers
  pagination: true,        // Previous/Next navigation
  copyCode: true           // Copy button on code blocks
}
```

Plugins loaded:
- Search (client-side, depth 4)
- Copy code (auto-copy button on code blocks)
- Pagination (prev/next links at bottom)
- Prism syntax highlighting (SQL, Python, Bash, YAML)

Custom styling:
- Theme color: `#4285f4` (Ditteau blue)
- Headings: `#740049` (Ditteau purple)
- Sidebar border: 3px purple

---

## Git Workflow

- **Main branch only** (no feature branches configured yet)
- **PRs encouraged** for team review ("found an error or gap? Open a PR")
- **Auto-deploy**: Push to main → GitHub Pages publishes

Standard commit message format:
```
<action> <what>

Examples:
- add snowflake environment doc
- update dbt conventions with testing strategy
- fix broken link in sidebar
```

---

## Writing Technical Documentation

When adding or updating documentation:

1. **Read existing docs first** to understand style and structure
2. **Use tables** for structured data (tech stack, team roster, conventions)
3. **Use horizontal rules** (`---`) to separate sections visually
4. **Cross-reference** related pages using relative markdown links
5. **Be concise** — prioritize clarity and scannability over exhaustive detail
6. **Include examples** — code snippets, SQL patterns, yaml config
7. **Document decisions** — use ADRs for architectural choices
8. **Update navigation** — add new pages to `_sidebar.md`

---

## SQL Style (for code examples)

When including SQL examples in documentation:

- **Keywords**: UPPERCASE (`SELECT`, `FROM`, `WHERE`)
- **Identifiers**: snake_case (`dim_student`, `_source_system`)
- **Indentation**: 2 spaces
- **CTEs**: Always use named CTEs for readability
- **Comments**: Use `--` for inline, avoid `/* */` blocks

Example:

```sql
WITH source AS (
  SELECT * FROM {{ source('jenzabar_cx', 'student') }}
),

renamed AS (
  SELECT
    student_id,
    first_name,
    last_name,
    {{ add_source_metadata() }}
  FROM source
)

SELECT * FROM renamed
```

This matches the Snowflake SQL conventions documented in Laurie's global preferences.
