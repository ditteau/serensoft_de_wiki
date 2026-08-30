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

**2. ~~No dedicated per-school pools yet.~~ Dedicated per-school pools, adopted the same
day.** *(Amended 2026-08-29, LVP. This clause originally deferred dedicated pools on the
grounds that each pool's `MIN_NODES ≥ 1` floor would roughly triple the baseline. **That
reasoning was wrong.** A pool bills for running services, not for existing:
`SYSTEM_COMPUTE_POOL_GPU` has existed since 2026-02-24, is suspended, and appears **zero
times** in six months of metering. A per-school pool therefore costs exactly what that
school's apps consume, and a school with no apps costs nothing. The only real penalty is
packing — two schools active **simultaneously** pay two nodes where a shared pool might
have used one, which surfaces only under concurrent multi-school load, i.e. exactly when
attribution matters most.)*

`DEMEAU_POOL_APPS_PROD` created and `GOVERNANCE_SIS` moved onto it
(`add_school_app_compute_pools_2026-08-29.sql`). Naming is
`{SCHOOL}_POOL_APPS_{ENV}`, env-suffixed so a DEV experiment cannot affect the latency
of a PROD app during a client demo. `MIN_NODES 1 / MAX_NODES 2 / AUTO_SUSPEND_SECS 300`,
created `INITIALLY_SUSPENDED` so creation does not start the billing clock.

Pools are created **on demand, not in advance** — Anselm and Merrimack have no Streamlit
apps, and empty pools would be free but meaningless.

> ⚠️ **Attribution is not retroactive.** Everything metered before this migration is
> pooled under `SYSTEM_COMPUTE_POOL_CPU` and cannot be split by school. The 25.45
> credits already spent stay unattributable.

**3. The monitoring gap is recorded as an open item, not solved.** No mechanism is being
built in this ADR. See Open Decisions below.

#### Open Decisions

| # | Question | Owner | Blocking |
|---|---|---|---|
| **CP-1** | What mechanism watches compute pool spend, given resource monitors structurally cannot? Candidates: a scheduled query over `ACCOUNT_USAGE.SNOWPARK_CONTAINER_SERVICES_HISTORY` with alerting, or a periodic manual review. | WDT | No — but the exposure is uncapped |
| ~~**CP-2**~~ | ~~Do dedicated per-school pools become a go-live requirement?~~ **RESOLVED 2026-08-29 (LVP): yes, per-school pools for billing attribution.** Adopted immediately rather than deferred, once the cost objection was found to be unfounded. `DEMEAU_POOL_APPS_PROD` is live. | LVP | Closed |
| ~~**CP-5**~~ | ~~`MAX_NODES = 2` is sized for one demo school with one app. Does the ceiling need raising, and does per-persona multiply the node floor?~~ **RESOLVED 2026-08-30 (LVP): raised to 4; and no, per-persona does *not* multiply the node floor.** See below — the premise was measurably wrong. | LVP | Closed |
| **CP-3** | Who is responsible for suspending abandoned preview services, and is there a routine sweep? Previewing leaves a billing service behind with no prompt. ⚠️ **Harder than it looks — see below.** | Unassigned | No |
| **CP-4** | Should apps be suspended between demos as standing practice, accepting container cold-start latency in front of a client? | LVP | Before first client demo |

#### CP-5 resolved — services pack onto a node

*Added 2026-08-30 (LVP), applied by `resize_demeau_app_pool_cp5_2026-08-30.sql`.*

CP-5 asked two questions. The second one had an answer sitting in this ADR's own figures.

**Does per-persona multiply the node floor? No.** `CPU_X64_S` bills at **0.11
credits/node-hour**. This ADR records `SYSTEM_COMPUTE_POOL_CPU` burning a flat **2.637
credits/day**, which is 0.1099 credits/hour — **exactly one node** — and it did so while
running **two to three** Streamlit services throughout. Services pack onto a node. Node
count follows resource demand, not service count, and five applications do not imply five
nodes.

> ⚠️ **The daily figures above were the evidence all along; nobody had divided them by the
> node rate.** The original CP-5 wording ("does per-persona multiply the node floor?")
> assumed a per-service floor that the account's own metering already disproved. Worth
> keeping as a reminder that a measurement can be present and still unread.

> ⚠️ **This does not make applications free, and the correct reading is nearly the
> opposite.** ~2.64 credits/day is one node pinned by a running service — roughly **80
> credits/month**, still an order of magnitude above the `DEMEAU_ANALYTICS_PROD` warehouse
> it supports. What the finding says is that spend is driven by **whether any app is
> running**, not by how many. Five apps left open cost about what one app left open costs.
> The lever is idle apps (CP-3, CP-4), not app count.

**Does the ceiling need raising? Yes, to 4 — because a ceiling is not a cost.** A pool
bills for *running* nodes: `SYSTEM_COMPUTE_POOL_CPU` has carried `MAX_NODES = 150` for six
months at one node's spend, and `SYSTEM_COMPUTE_POOL_GPU` has existed since February and
appears **zero times** in six months of metering. Raising the ceiling costs nothing while
idle, and buys headroom for the one moment it matters — several personas opened at once
during a client demo, where the failure mode is an app that will not start in front of the
client.

**4 rather than 8, deliberately.** Resource monitors structurally cannot meter compute
pools — the central finding of this ADR, and **CP-1 is still open** — so the ceiling is
currently the *only* bound on a runaway. A ceiling with no monitor behind it should stay
close to measured need. 4 is double the observed requirement for five apps and bounds an
unnoticed runaway at ~10.5 credits/day rather than ~21.

`MIN_NODES` stays 1; `AUTO_SUSPEND_SECS` stays 300 and remains ineffective while any
service runs, which is untouched by this change.

**Watch implied node-hours, not credits:**

```sql
SELECT compute_pool_name,
       DATE_TRUNC('day', start_time)      AS day,
       ROUND(SUM(credits_used), 3)        AS credits,
       ROUND(SUM(credits_used) / 0.11, 1) AS implied_node_hours
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWPARK_CONTAINER_SERVICES_HISTORY
WHERE start_time > DATEADD('day', -14, CURRENT_TIMESTAMP())
GROUP BY 1, 2 ORDER BY 2 DESC, 1;
```

A move from ~24 to ~48 node-hours/day means a second node became durably active. At five
apps that would mean the packing behaviour stopped holding — which is a reason to reopen
CP-5, not to raise the ceiling again reflexively.

**Scope of this decision:** DEMEAU PROD only, five applications (Registrar, FA, IR,
Admissions, Finance). Anselm and Merrimack still have no apps and no pools, per the
on-demand rule above. ⚠️ Cabinet and Student Success were considered and **excluded**:
both are `AGGREGATED` with no `DISTRIBUTE` grant, and granting one trips the I.4
suppression interlock (access policy O-7b) immediately.

---


#### There is no SQL way to stop a running Streamlit service

Established 2026-08-29 while clearing the abandoned services. Every obvious lever fails:

| Attempt | Result |
|---|---|
| `ALTER SERVICE … SUSPEND` | `SQL access control error` — even as `ACCOUNTADMIN`. The services are `SYSTEM$MANAGED`; the Streamlit object owns them |
| `ALTER STREAMLIT … SUSPEND` | `SQL compilation error` — no such command |
| `ALTER STREAMLIT … SET IDLE_AUTO_SHUTDOWN_TIME_SECONDS` | Works, but the floor is 24 hours |
| `DROP STREAMLIT` | Works. Destroys the app |

So the complete set of remedies is: **the UI `Active` toggle, a ≥24-hour idle timer, or
deletion.** Nothing scriptable stops a service while keeping the app.

That has a direct consequence for CP-3: a routine sweep **cannot be automated as a task
or scheduled query**. Any cleanup is either a person clicking a toggle, or dropping
apps — which is only safe for previews. It also means an app someone opens daily can be
stopped only by hand.

Two abandoned preview apps (`Preview of … Test_Gov`, `Preview of …
ditteau_data_transform/streamlit`) were dropped rather than suspended for exactly this
reason. Both billing services belonged to previews; the one real personal app was
already suspended and costing nothing.

⚠️ **Previews are the main leak.** Both came from clicking Run in a Workspace. A preview
creates a durable, billing service with no prompt, no expiry and no scriptable off
switch, and it survives long after the tab is closed.

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
- **Attribution starts now and is not retroactive.** The 25.45 credits already spent on
  the shared pool cannot be split by school.
- **Packing inefficiency under concurrent load.** Each school's pool has its own
  `MIN_NODES = 1` floor, so N schools active at once cost N nodes where a shared pool
  might have packed them onto fewer. Accepted as the price of attribution.
- **A pool per school is a pool per school to operate** — each needs its own grants,
  sizing review and eventual monitoring. That multiplies with tenant count.
- The two abandoned services in `USER$LVANPELT.PUBLIC` are in a personal database and
  were left untouched. They continue to bill against the shared pool — unattributable to
  any school — until suspended by hand.
- `MAX_NODES = 2` is sized for today's single app and will need revisiting when D-1
  lands. See CP-5.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Dedicated pool per school + 24h idle shutdown (chosen)** | Cost attributable per tenant, which is a billing requirement; removes noisy-neighbour risk; costs nothing for schools with no apps, since suspended pools do not bill | Not retroactive; packing penalty under concurrent multi-school load; one more object per school to operate |
| Shared pool, defer dedicated pools | No new infrastructure | No attribution — and the cost argument for deferring turned out to be false, since pools bill for running services rather than for existing |
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
