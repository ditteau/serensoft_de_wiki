# ADR-010: Compute Pool Economics for Streamlit in Snowflake

**Status:** Accepted

**Date:** 2026-08-29

**Author:** LVP

---

### Context

Streamlit in Snowflake under Workspaces runs on the **container runtime only**. A
container runtime requires a compute pool, and compute pools bill on a fundamentally
different model from warehouses: **per node-hour while the pool is running**, not per
query. A pool with `MIN_NODES = 1` that stays active bills continuously whether or not
anyone opens an application.

Nothing in the platform was watching this. `SYSTEM_COMPUTE_POOL_CPU` has consumed
**25.3 credits since 2026-07-20**, running at a sustained **~2.64 credits/day** —
roughly **58 to 79 credits/month**. For comparison, `DEMEAU_ANALYTICS_PROD`, the
warehouse behind every dashboard, consumed **5.78 credits in all of August**. The pool
costs an order of magnitude more than the warehouse it supports, and it did so
unnoticed for six weeks.

The reason it went unnoticed is structural and is the most important finding in this
ADR:

> **Snowflake resource monitors meter warehouse credits only. They do not cover compute
> pools.** Every monitor in the account, `DITTEAU_ACCOUNT_MONTHLY` included, is blind to
> this spend. There is no quota, no notify trigger, and no suspend action available.

That gap is easy to miss precisely because the account *looks* well governed. On
2026-08-29 considerable care went into resizing `DEMEAU_MONITOR_ANALYTICS_PROD` from 25
to 50 credits and converting its 100% trigger from suspend to notify, so a client demo
could not be cut off mid-session. That work governs a warehouse consuming ~6
credits/month while an unmonitored pool beside it consumed ~70.

The immediate cause of the burn is abandoned services. A pool cannot auto-suspend while
any service runs on it, and `SYSTEM_COMPUTE_POOL_CPU` carries four:

| Service | State | Location |
|---|---|---|
| `STPLATSTREAMLIT392910632` | RUNNING | `DEMEAU_DD_PROD.DISTRIBUTE` — `GOVERNANCE_SIS` |
| `STPLATSTREAMLIT392910628` | RUNNING | `USER$LVANPELT.PUBLIC` — a preview created 2026-08-29 |
| `STPLATSTREAMLIT392910624` | RUNNING | `USER$LVANPELT.PUBLIC` — leftover |
| `STPLATSTREAMLIT392910620` | SUSPENDED | `USER$LVANPELT.PUBLIC` |

Three running, two of them abandoned personal-workspace apps that nobody suspended.
The pool's own `AUTO_SUSPEND_SECS = 600` never fires. **Merely previewing an app in a
Workspace leaves a service behind that bills until someone notices.**

---

### Decision

**Stay on the shared `SYSTEM_COMPUTE_POOL_CPU` for now. Set idle shutdown where it
exists. Treat compute pool spend as a manually monitored cost until a mechanism exists.**

Three parts:

**1. Idle shutdown is set on deployed apps, with a known-weak ceiling.**
`ALTER STREAMLIT ... SET IDLE_AUTO_SHUTDOWN_TIME_SECONDS` is applied to
`DEMEAU_DD_PROD.DISTRIBUTE.GOVERNANCE_SIS` at **86,400 seconds (24 hours)**. Every other
Streamlit object in the account is currently unset.

> ⚠️ **86,400 is the floor, not a choice.** The property rejects anything outside
> **86,400 to 604,800** seconds:
> `invalid value '3,600' for property 'IDLE_AUTO_SHUTDOWN_TIME_SECONDS': must be between
> 86,400 and 604,800 (in seconds)`.
> This is *not* a between-sessions control. It shuts down an app nobody has opened for at
> least a full day. **An app opened daily never idles out and bills continuously.** The
> only per-session lever is the manual **Active** toggle in the app UI.

**2. No dedicated per-school pools yet.** Each pool carries a standing `MIN_NODES ≥ 1`
floor, so three provisioned schools with dedicated pools would roughly triple today's
baseline in order to serve the one school that actually uses SiS.

**3. The monitoring gap is recorded as an open item, not solved.** No mechanism is being
built in this ADR. See Open Decisions below.

#### Open Decisions

| # | Question | Owner | Blocking |
|---|---|---|---|
| **CP-1** | What mechanism watches compute pool spend, given resource monitors structurally cannot? Candidates: a scheduled query over `ACCOUNT_USAGE.SNOWPARK_CONTAINER_SERVICES_HISTORY` with alerting, or a periodic manual review. | WDT | No — but the exposure is uncapped |
| **CP-2** | Do dedicated per-school pools become a **go-live requirement** for the first paying tenant? Cost attribution per tenant is a billing requirement, not a preference. | WDT | First real tenant |
| **CP-3** | Who is responsible for suspending abandoned preview services, and is there a routine sweep? Previewing leaves a billing service behind with no prompt. | Unassigned | No |
| **CP-4** | Should apps be suspended between demos as standing practice, accepting container cold-start latency in front of a client? | LVP | Before first client demo |

---

### Consequences

#### Pros

- Removes an indefinite billing tail on `GOVERNANCE_SIS`: previously the service would
  have run forever, now it stops 24 hours after last use.
- Avoids tripling the baseline for isolation nobody needs yet — one school uses SiS.
- Names the resource-monitor blind spot explicitly, so the next person tuning warehouse
  quotas knows the larger number sits outside that system entirely.
- Defers the dedicated-pool conversation with WDT to a point where measured per-app
  consumption exists, which is a far stronger basis than a forecast.
- Records the 86,400-second floor, which is genuinely surprising and would otherwise be
  rediscovered by someone assuming warehouse-style `AUTO_SUSPEND` semantics.

#### Cons

- **The exposure remains uncapped.** Nothing prevents compute pool spend from growing;
  this ADR measures and documents it rather than bounding it. CP-1 is the real fix and
  is not done.
- **Idle shutdown does not solve the daily-use case.** An app opened every day never
  idles out, so the 24-hour floor buys nothing during an active demo period — exactly
  when consumption is highest.
- **Manual suspension is the only per-session control**, which means it depends on
  someone remembering. That is the failure mode that produced this ADR.
- Shared pool means **consumption is not attributable per school**, so there is no way
  to bill or even report a tenant's true cost today.
- Shared pool retains **noisy-neighbour risk**: another service on
  `SYSTEM_COMPUTE_POOL_CPU` can affect demo latency, and the pool is `is_exclusive:
  false`.
- The two abandoned services in `USER$LVANPELT.PUBLIC` are in a personal database and
  were left untouched by this decision. They continue to bill until suspended by hand.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Shared pool + 24h idle shutdown (chosen)** | No new infrastructure; stops the indefinite tail; keeps cost at one baseline | Uncapped; no attribution; ineffective for daily-use apps |
| Dedicated pool per school | Cost attributable per tenant; removes noisy neighbour; the right end state | `MIN_NODES ≥ 1` floor per pool ≈ 3× baseline today, to serve one active school; routes through WDT as an infrastructure scope change |
| Status quo — no idle shutdown, no ADR | Zero effort | ~58 to 79 credits/month accruing invisibly and indefinitely; the situation that prompted this |
| Suspend the pool itself between demos | Strongest cost control available | A suspended pool cannot serve any app; breaks anyone opening a dashboard unannounced; does not survive multi-tenant use |
| Move dashboards back to warehouse-runtime SiS | Warehouse runtime is covered by resource monitors, so the spend becomes governable with existing tooling | Abandons Workspaces and the git-backed deployment path; warehouse runtime cannot use `pyproject.toml` dependency management. Worth revisiting if CP-1 finds no workable mechanism |

---

### References

- [Streamlit in Snowflake — Workspaces Runbook](../runbooks/streamlit-in-snowflake-workspaces.md) — deployment procedure, §J for the `CURRENT_USER()` finding
- [ADR-003: Row Access Policy Architecture](adr-003-row-access-policy-architecture.md) — the entitlement model these apps run under
- `ditteau_data_transform/streamlit/snowflake.yml` — deployed app definition, `run_mode: SpcsOnly`
- `ditteau_data_transform/streamlit/governance_identity_probe/README.md` §F — why an External Access Integration is not an acceptable substitute for the Snowflake-hosted PyPI mirror
- `ditteau_data_infra/school_setup/migrations/alter_resource_monitors_demo_readiness_2026-08-29.sql` — the warehouse-side monitor work this ADR contrasts with
- Measurement source: `SNOWFLAKE.ACCOUNT_USAGE.SNOWPARK_CONTAINER_SERVICES_HISTORY`
