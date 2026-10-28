# ADR-001: Organization Isolation and Regional Cells

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-28 |
| **Ticket** | P0.01 |
| **Deciders** | Platform architecture (owner), Product, Finance |
| **Stack** | Laravel 12 · Inertia · React · PostgreSQL · `stancl/tenancy` v3 · PgBouncer |
| **Supersedes / superseded by** | — |

> **Source documents.** This ADR is written to be reconciled against BRD v3.1, the Master
> Feature Catalogue v3.1 (Features, Modules, Pricing, Message Templates sheets) and the
> Financial Model v3.1. Those documents are **not yet committed to this repository**, so
> every figure here that should come from them is tagged **[VERIFY: doc]**. Please confirm
> or correct those figures before moving this ADR to *Accepted*.

---

## TL;DR — the one table a new engineer needs

| Market | Home cell | Who buys ASMS (= the **Organization**) | Isolation unit (one PostgreSQL database per…) | Associations per database | Why |
|---|---|---|---|---|---|
| **India** | `in-1` (Mumbai) | The housing society / RWA / apartment owners' association itself | **Association** | Exactly 1 | The society is the buyer and the legal data owner. It signs, pays, audits, churns and asks for its data back on its own. |
| **UAE (Dubai)** | `ae-1` (UAE) | The owners' association management company | **Management company** | 1 … thousands | The management company is the buyer. Its staff work across its whole portfolio every day, and one database per association would mean hundreds of thousands of databases. |
| **Canada (Ontario)** | `ca-1` (Canada Central) | The condominium management firm | **Management company** | 1 … thousands | Same as UAE |
| **US (Florida)** | `us-1` (US East) | The community association management company | **Management company** | 1 … thousands | Same as UAE. CINC alone reports 50,000+ communities, so one database per association does not scale. |
| Any non-India market, self-managed association | the market's cell | The association itself | Management company shape, with one association | 1 | It is still the management-company shape (an "organization of one"). It is not the India shape, so the code path stays the same. |

**The rule:** *an Organization is the paying customer, and it always gets its own database.
What changes by market is who the paying customer is.* In India that is the association. In
the UAE, Canada and the US it is the management company.

Three invariants hold in every market:

1. **One Organization ⇒ exactly one database ⇒ exactly one home cell.**
2. **Every domain row carries `association_id`**, even in India, where it is always the same value. Every market runs the same schema and the same code.
3. **No query, job, report or API call ever joins or reads data across cells.**

---

## 1. Context

### 1.1 What ASMS is

ASMS is a multi-organization SaaS that runs residential associations: billing and
maintenance dues, accounting, collections, amenities, visitors and gate, helpdesk,
documents, meetings and votes, and notices. It sells by subscription to customers in four
markets, in this order: **India first, then Dubai (UAE), Ontario (Canada) and Florida
(US).** [VERIFY: BRD v3.1 — market sequence and launch dates]

### 1.2 Vocabulary (binding for code, UI, docs and conversations)

| Term | Meaning | Not to be confused with |
|---|---|---|
| **Organization** | The SaaS customer: the party that signs the contract, pays the invoice and owns the data. It is the unit of isolation, billing, plan entitlements and data export. | — |
| **Association** | One residential community being administered: a housing society or RWA (India), an owners' association or jointly owned property (UAE), a condominium corporation (Ontario), or an HOA or condo association (Florida). | In India, Organization and Association refer to the same real-world entity. They are still two different records. |
| **Management company** | A firm that administers many associations under contract. In the UAE, Canada and the US it is the Organization. | In India a facility-management firm is **not** an Organization. See §2.6. |
| **Unit** | A flat, villa, apartment, lot or parcel inside an association. | — |
| **Occupant** | A person living in a unit: an owner-occupier, a **renter**, or a family member. | **Never call a renter a "tenant".** In this domain "tenant" means a renter, and in SaaS jargon it means a customer. We avoid the word completely so the two meanings can never be confused. |
| **Cell** | A complete, independent deployment of ASMS in one cloud region (app, workers, PgBouncer, PostgreSQL clusters, Redis, object storage, search). | Not a "shard". A cell is bigger than a database cluster. |
| **Cluster** | One PostgreSQL primary plus its standbys, holding many Organization databases. A cell has 1…N clusters. | — |

`stancl/tenancy` uses the word *tenant* internally. That word may appear **only** in
`config/tenancy.php` and in `app/Platform/Isolation/**`, the adapter layer that maps
stancl's concepts onto ours. See §5, M-10.

### 1.3 Forces

1. **Who the buyer is differs by market.**
   * **India.** The society buys directly, through its managing committee (chairman,
     secretary, treasurer). The committee changes at elections. The society is audited
     on its own (the statutory audit under state co-operative societies acts or the
     apartment ownership acts). It switches vendors on its own, and it is the data
     fiduciary for its residents' personal data under the DPDP Act 2023. Professional
     management companies exist in India, but they do not hold the customer relationship
     for most societies. [VERIFY: BRD v3.1 — India buyer persona]
   * **UAE, Ontario, Florida.** Most associations are run by licensed management
     companies (RERA-registered OA managers in Dubai; CMRAO-licensed managers in Ontario;
     CAM-licensed firms in Florida). The firm chooses the software and runs it across its
     whole portfolio. One portfolio manager typically handles 10–40 associations, and the
     back office (accounts payable, collections, bank reconciliation, compliance calendars)
     works across all of them at once. Large firms run thousands of associations.
     CINC Systems alone reports **50,000+ communities** on its platform.
2. **Scale of the isolation unit.** If the database were always the association:
   * India: ~1 database per customer. This is fine.
   * Western markets: a 10,000-customer cell with an average of 25 associations per
     customer would need **~250,000 databases**, and one large firm would need thousands.
     Portfolio dashboards would have to fan out across hundreds of connections per page
     load.
3. **Data residency and sovereignty.** India (DPDP Act 2023), UAE (PDPL, Federal
   Decree-Law 45/2021), Canada (PIPEDA, Ontario Condominium Act record-keeping) and the US
   (state law, Florida Chapters 718/720 records and website requirements) each expect
   resident data to stay where the customer is. Data never needs to cross a border to
   serve a customer. [VERIFY: legal review per market before each launch]
4. **Customer expectations of isolation.** Indian societies and Western management
   companies both ask "is our data separate?" during sales. A physically separate
   database is the simplest truthful answer. It also makes the following easy:
   per-customer export, deletion on churn, restore, and moving a customer between clusters.
5. **Operating cost.** Thousands of databases need connection pooling, automated
   provisioning, migrations orchestrated per database, and monitoring per database. The
   design must stay cheap at India price points. [VERIFY: Financial Model v3.1 — India
   ARPU and infra cost per org targets]
6. **Team size.** One codebase and one schema. We will not maintain an "India edition"
   and a "Western edition".

---

## 2. Decision

### 2.1 The Organization is the isolation boundary, everywhere

Every Organization gets **its own PostgreSQL database** (`org_<ulid>`), provisioned by
`stancl/tenancy` in multi-database mode. Requests, queue jobs, scheduled tasks and Artisan
commands always run *inside* exactly one Organization's database, or inside the cell
catalog (§2.4). They never run inside two Organizations at once.

### 2.2 Who the Organization is depends on the market

| Market profile | `organizations.shape` | Organization = | Associations in its DB |
|---|---|---|---|
| `IN` | `single_association` | the association | exactly 1. Enforced by a DB constraint and by the domain service. |
| `AE`, `CA`, `US` | `portfolio` | the management company (or a self-managed association acting as its own manager) | 1…n |

`shape` is fixed when the Organization is created, from its market profile. It is not
a free-form setting. It controls:

* whether the UI shows the portfolio switcher and the cross-association reports;
* whether `CreateAssociation` is allowed after the first association exists;
* which plans apply (per-association pricing in India, per-door or per-unit portfolio
  pricing in the West). [VERIFY: Feature Catalogue v3.1 — Pricing sheet]

### 2.3 One schema for both shapes

Every domain table (units, occupants, ledgers, invoices, receipts, visitors, tickets,
documents, message logs, and so on) has a non-null `association_id`. India DBs contain
exactly one `associations` row. Portfolio DBs contain many. The consequences:

* There is one set of migrations, one set of models and one test suite.
* A self-managed Florida HOA and an Indian society run the same code. Only the
  cardinality differs.
* Isolation *between associations inside one portfolio database* is a logical boundary,
  enforced in the application (§5, M-8). Isolation *between Organizations* is always
  physical (a separate database).
* **All primary keys are ULIDs.** Rows can then be copied between databases without
  remapping keys. This matters when an HOA leaves one management company for another,
  which is a routine Western event. See §2.7.

### 2.4 Regional cells

* Each Organization has exactly **one home cell**, chosen when it signs up from its
  market (`IN → in-1`, `AE → ae-1`, `CA → ca-1`, `US → us-1`). Later cells within a
  jurisdiction (`in-2`, …) are added when a cell reaches its ceiling (§6).
* A cell contains:
  * its own **cell catalog** database (`asms_cell`). This is stancl's "central" database:
    organizations, domains, `db_cluster` assignment, schema version per database, and
    provisioning state;
  * 1…N **PostgreSQL clusters**, each holding up to the density limit of Organization
    databases;
  * PgBouncer, app servers, queue workers, Redis, object storage bucket and search, all in
    the same region.
* A thin **global control plane** stores only what it needs to route and bill:
  `organization_id`, slug, custom domains, `home_cell`, status, plan, and billing-provider
  IDs. It stores **no association, unit, occupant or financial data** and no personal
  data beyond the billing contact.
* **Routing.** `{slug}.asms.app` or a custom domain → the global edge looks up
  `home_cell` → the request is forwarded to `{cell}.asms.app`. Login by email first
  resolves the cell from a hashed-email index in the control plane, then authenticates
  inside the cell.
* **No cross-cell queries, ever.** The same rule covers every way data could cross a cell:
  * *Platform analytics.* Cells **push** aggregated, de-identified metrics (counts,
    usage, MRR inputs) to the control plane. Nothing ever pulls rows across cells.
  * *A person who belongs to Organizations in two cells* (for example, an owner with a
    flat in Pune and a villa in Dubai) has two identities, one per cell. The control
    plane can link them for a "switch account" menu. It never merges their data.
  * *Moving an Organization to a different cell* is a planned, offline export and import
    that we run as a support procedure. It is never a live cross-cell operation.

### 2.5 `stancl/tenancy` configuration (summary)

* `tenant_model` → `App\Platform\Isolation\OrganizationTenant`, which extends stancl's
  model and is wrapped by the domain model `App\Models\Organization`.
* Multi-database mode with `PostgreSQLDatabaseManager` (one *database* per Organization,
  not schemas). Database names use `prefix = 'org_'`.
* Each Organization record stores which cluster holds it (the stancl internal attribute
  `tenancy_db_connection` → `pg_cluster_03`, …), so one cell can span many clusters.
* The connection points at **PgBouncer** (§5, M-1). Migrations and provisioning use a
  direct (non-pooled) connection.
* Identification by domain or subdomain for web requests. Queued jobs carry the
  organization ID and restore context through stancl's `QueueTenancyBootstrapper`.
* Cache, filesystem and Redis bootstrappers are enabled, so cache keys, storage paths and
  queues are namespaced by Organization.

### 2.6 India: management companies and federations

When an Indian facility-management firm manages several societies, each society is still
its own Organization with its own database. The firm gets a **Partner console**. It lists
the Organizations that have granted the firm access, and every action switches into one
Organization at a time. The console never reads across Organizations; any combined view
it offers is built from data each society has explicitly shared. The same applies to
federations and apex bodies.

### 2.7 Western markets: an association changes management company

An association moves from Organization A to Organization B in the same cell as a
**transfer job**:

1. Export the association's subtree from A's DB. ULIDs mean no key remapping is needed.
2. Import it into B's DB.
3. Freeze the association in A as read-only and archived. A keeps its own records as
   retention law requires.

This runs entirely inside one cell. It is the only flow that writes the same association's
data into two Organization databases, and it is explicit, audited and needs consent.

---

## 3. Alternatives considered

| # | Option | Verdict | Why |
|---|---|---|---|
| A | **One database per association, everywhere** | Rejected | It fits India perfectly but breaks in the West. A cell would need ~250k databases, one large management company would need thousands, and portfolio queries (aged receivables across 120 HOAs, consolidated accounts payable, staff workload) would fan out across hundreds of connections. CINC's 50,000+ communities in a single vendor shows the scale of the problem. |
| B | **One database per management company, everywhere** (India societies grouped under a synthetic or "house" org) | Rejected | In India the society is the buyer and the data fiduciary. Putting unrelated societies in one database breaks the isolation promise, makes per-society export, restore and deletion a row-level operation, and lets one large society slow down its neighbours. |
| C | **Shared database, `organization_id` column + PostgreSQL RLS, everywhere** | Rejected | It is cheapest at small scale (roughly $1/org/month cheaper in an India cell at 10k orgs; see §6.4) and has a single migration target. But one bug or one missed policy leaks data between paying customers. Restore and export per customer are hard. A noisy neighbour affects everyone. It is a weak answer to "is our data separate?", and every query depends on the tenancy filter being correct. |
| D | **Schema per Organization in shared databases** (`search_path` switching) | Rejected | It has the same catalogue bloat as many databases without the operational benefits. pg_dump and restore per Organization is clumsier, `search_path` combined with PgBouncer transaction pooling is fragile, and stancl's schema mode is less proven for us at thousands of schemas. |
| E | **One global deployment, regional databases only** | Rejected | Queues, caches, search indexes, logs and object storage would sit in one region, which breaks residency. A single blast radius also covers every market. |
| F | **Hybrid chosen here: database per Organization, where Organization = association (IN) or management company (AE/CA/US), in regional cells** | **Accepted** | It matches the buyer in each market, keeps physical isolation between paying customers, caps database counts at about one per customer, and keeps one schema and one codebase. |

---

## 4. Consequences

### 4.1 Positive

* **Clear security boundary.** A query cannot reach another customer's data, because it
  is in a different database. Per-customer encryption keys and IP allowlists remain
  possible later.
* **Lifecycle per customer is simple.** Export means `pg_dump org_x`. Deletion on churn
  means `DROP DATABASE`, which gives clean DPDP and PDPL erasure evidence. Restoring one
  customer does not touch the others.
* **Noisy neighbours are contained.** A heavy Organization can be moved to a dedicated
  cluster without code changes, because it is just a change to the cluster attribute.
* **Residency by construction.** All of a customer's data, caches, queues and files live
  in its home cell.
* **One schema, one codebase.** Indian and Western behaviour differ by `shape` and market
  profile, not by fork.
* **Western portfolio features are cheap.** Cross-association reports are ordinary SQL
  inside one database.

### 4.2 Negative

* **Thousands of databases per cell** (≈1 per Indian society). This needs the
  mitigations in §5 from day one, not as later hardening.
* **Schema migrations become a fleet rollout.** A migration is "done" only when every
  database in every cell has it. Schema N and N-1 must coexist for hours.
* **Cross-customer queries are impossible by design.** Platform analytics, support
  search and benchmarking ("societies like yours…") must be built from pushed aggregates.
* **Two shapes to test.** Every feature must be tested for `single_association` and for
  `portfolio`. This is enforced through the Pest dataset (§5, M-9).
* **Isolation between associations inside a Western Organization is logical, not
  physical.** A bug in association scoping could show HOA X's data to a manager of HOA Y
  *within the same management company*. That is a real defect, but it is not a breach
  between customers.
* **Higher infrastructure cost per org than a shared database** at India scale
  (≈$1/org/month at 10k orgs, §6.4). We accept this as the price of the isolation promise.

---

## 5. Mitigations

| ID | Risk | Mitigation |
|---|---|---|
| **M-1** | Connection exhaustion across thousands of databases | **PgBouncer in transaction-pooling mode** in front of every cluster: two instances per cluster for HA, behind a cell-internal load balancer. Each Organization DB gets a small pool on demand (`default_pool_size=2`, `min_pool_size=0`, `server_idle_timeout=60`), so idle societies hold **zero** server connections. `max_db_connections` caps any one org. A wildcard `[databases] * = host=<cluster>` plus `auth_query` means new databases need no PgBouncer reload. Laravel rules for transaction mode: no session-level `SET` (use `SET LOCAL` inside transactions), no session advisory locks (use Redis locks), and prepared statements handled by PgBouncer ≥ 1.21 `max_prepared_statements` or `PDO::ATTR_EMULATE_PREPARES`. Migrations and `CREATE DATABASE` bypass PgBouncer on a direct connection. |
| **M-2** | Slow, manual or half-finished provisioning | **Automated, idempotent provisioning pipeline** (`ProvisionOrganization` job chain triggered by stancl `TenantCreated`): pick the least-loaded cluster → `CREATE DATABASE org_x TEMPLATE asms_template_vN` (a template DB already at the latest schema, so provisioning takes seconds rather than replaying every migration) → apply any migrations newer than the template → seed market data (chart of accounts, tax config, message templates per market and language) → create the first association and admin → mark it `active` in the cell catalog and the control plane. Every step is retriable. The target is **under 60 s from signup to usable**. Failed provisioning is reaped automatically after 24 h. |
| **M-3** | Problems invisible inside one of thousands of databases | **Monitoring per database.** A collector job per cluster records, for every DB: size, connections, transactions/s, cache hit ratio, deadlocks, long transactions, **transaction-ID age (wraparound is per database)**, autovacuum lag, top queries (`pg_stat_statements` by `dbid`), applied schema version and last successful backup. Metrics flow to Prometheus with the org ID as a label only for the top-N heaviest DBs plus a fleet histogram, which keeps label cardinality bounded. Full per-DB detail goes to the cell catalog table `org_db_health`. Alerts fire on schema drift, wraparound risk, a missed backup, or a DB growing faster than 3σ. |
| **M-4** | Migrations across thousands of DBs fail partway or block deploys | **Migration orchestration per cell** (`asms:fleet:migrate --cell=in-1`). (1) Migrate `asms_template_vN+1`. (2) Run in waves: internal/demo orgs → 1% canary → 10% → 100%. (3) Fan out through a queue with **bounded concurrency per cluster** (default 8), `lock_timeout=5s` and `statement_timeout` for each migration. (4) Record each result in `schema_rollouts`, and halt the wave automatically if more than 0.5% fail. (5) Retry failed DBs, or quarantine them for manual work. Migrations must be **expand/contract**: additive in release N, destructive only in N+2 after the fleet confirms it is complete, so code always runs against schema N or N-1. Cells roll out independently, smallest and least critical first. Estimated fleet time at 10k DBs, 10 clusters × 8 parallel at ~6 s/DB: **≈13 min**. |
| **M-5** | Backup/restore at fleet scale | Cluster-level PITR (pgBackRest or the managed equivalent, with continuous WAL) for disaster recovery, **plus** a nightly logical dump per DB to object storage in the same cell. The nightly dump is used for single-Organization restore, customer export and churn archives. Restore drills are automated weekly on a random sample of orgs. |
| **M-6** | Catalogue bloat and vacuum load on a cluster | **Density limit:** ≤ 1,000 DBs per cluster for `single_association` (small DBs) and ≤ 200 for `portfolio` (large, busy DBs). Tune `autovacuum_max_workers` and `autovacuum_naptime` for many DBs. Monitor `pg_database` count. Above the limit, new orgs go to a new cluster. Heavy orgs are rebalanced by logical replication or dump/restore in a maintenance window. |
| **M-7** | Cross-cell leakage | Every cell has its own network, credentials, KMS keys and buckets. App servers in cell X have **no network route or credentials** to cell Y's databases. The control plane API has no endpoint that returns domain data. Contract tests assert that the control-plane schema contains no domain tables. |
| **M-8** | Association leakage inside a portfolio DB | The `BelongsToAssociation` trait applies a global scope keyed by the `CurrentAssociation` context. A middleware resolves the association from the route and authorises it against the user's portfolio grants. Unscoped queries need an explicit `withoutAssociationScope()` call, and a PHPStan rule bans that call outside `app/Reporting/Portfolio/**`. PostgreSQL RLS on `association_id` as defence in depth is deferred to **ADR-00x (RLS within portfolio databases)**, because RLS must use `SET LOCAL` under PgBouncer transaction mode. |
| **M-9** | Shape-specific bugs | The Pest dataset `organizationShapes()` runs every feature test against `single_association` and against `portfolio` with ≥ 2 associations, and the portfolio run includes a test that one association cannot see another's data. CI fails if a feature test does not use the dataset. |
| **M-10** | Vocabulary drift ("tenant") | A CI check rejects `/\btenants?\b/i` in `app/`, `database/`, `resources/js/`, `routes/` and `lang/`, except in `app/Platform/Isolation/**` and `config/tenancy.php`. Renters are `Occupant` with `occupancy_type = renter`. |
| **M-11** | An Organization outgrows a shared cluster (for example, a management company with 3,000 associations) | Move it to a **dedicated cluster** in the same cell. Its routing changes only through the `tenancy_db_connection` attribute. This becomes a priced Enterprise option. [VERIFY: Feature Catalogue v3.1 — Pricing sheet] |

---

## 6. Cost model per cell

### 6.1 Assumptions

The figures below are infrastructure only, in USD/month, using on-demand list prices
rounded from AWS-class managed services. A 1-year commitment typically removes 30–40%.
They exclude people, support, the global control plane, messaging (SMS, WhatsApp and
email costs scale with message volume, not with isolation) and payment gateway fees.
**[VERIFY: Financial Model v3.1 — replace with its unit prices, growth curve and ARPU]**

| Item | India cell (`single_association`) | Western cell (`portfolio`) |
|---|---|---|
| Associations per org | 1 | 25 (average) |
| DB size per org (yr 1–2) | 1 GB | 15 GB |
| Documents per org (object storage) | 3 GB | 60 GB |
| DBs per cluster (density) | 1,000 (`M` at 500 orgs, `L` beyond) | 200 (`L`) |
| HA | Multi-AZ primary + standby | Multi-AZ + 1 read replica (portfolio reports) |
| PG cluster price, Multi-AZ | M (4 vCPU/32 GB) $700 · L (8 vCPU/64 GB) $1,400 | L $1,400 + replica $700 |
| Storage incl. backups and WAL | $0.30/GB-mo | $0.30/GB-mo |
| App/worker node (4 vCPU/8 GB) | $120 | $120 |
| PgBouncer | 2 × $60 per cluster | 2 × $60 per cluster |
| Observability | $400 base + $0.50/DB | same |
| Edge (LB, WAF, egress) | $300 + $0.25/org | $300 + $2/org |

### 6.2 India cell (`in-1`): database per association

| Organizations (= DBs) | 500 | 3,500 | 10,000 |
|---|---:|---:|---:|
| PG clusters | 1 × M | 4 × L | 10 × L |
| PostgreSQL compute | 700 | 5,600 | 14,000 |
| DB storage + backups | 150 | 1,050 | 3,000 |
| App + queue nodes | 720 (3+3) | 1,920 (9+7) | 5,160 (25+18) |
| PgBouncer | 120 | 480 | 1,200 |
| Redis | 300 | 600 | 1,200 |
| Object storage | 38 | 262 | 750 |
| Observability | 650 | 2,150 | 5,400 |
| Edge | 425 | 1,175 | 2,800 |
| **Total / month** | **≈ $3,100** | **≈ $13,200** | **≈ $33,500** |
| **Per Organization / month** | **≈ $6.20 (≈ ₹520)** | **≈ $3.80 (≈ ₹315)** | **≈ $3.35 (≈ ₹280)** |

### 6.3 Western cell (`ae-1` / `ca-1` / `us-1`): database per management company

| Organizations (= DBs) | 500 | 3,500 | 10,000 † |
|---|---:|---:|---:|
| Associations (rows) | 12,500 | 87,500 | 250,000 |
| PG clusters | 3 × L + replicas | 18 × L + replicas | 50 × L + replicas |
| PostgreSQL compute | 6,300 | 37,800 | 105,000 |
| DB storage + backups | 2,250 | 15,750 | 45,000 |
| App + queue nodes | 1,920 (9+7) | 12,120 (59+42) | 34,080 (167+117) |
| PgBouncer | 360 | 2,160 | 6,000 |
| Redis | 300 | 600 | 1,200 |
| Object storage | 750 | 5,250 | 15,000 |
| Observability | 650 | 2,150 | 5,400 |
| Edge | 1,300 | 7,300 | 20,300 |
| **Total / month** | **≈ $13,800** | **≈ $83,100** | **≈ $232,000** |
| **Per Organization / month** | **≈ $27.70** | **≈ $23.75** | **≈ $23.20** |
| **Per association / month** | ≈ $1.11 | ≈ $0.95 | ≈ $0.93 |

† 10,000 management companies is more than any single Western region is likely to reach.
This column is a stress point that shows the design stays linear. It is not a forecast.

### 6.4 What the model tells us

* **India.** Cost per org falls from ≈$6.20 to ≈$3.35 as fixed costs are spread more
  thinly, then flattens because each cluster holds a fixed number of DBs. A shared-database
  design (Alternative C) would cost roughly **≈$2.30/org at 10k**, so the isolation
  premium is **≈$1/org/month (≈₹85)**. This is affordable only if India ARPU per society
  is comfortably above ≈₹1,500/month. **[VERIFY: Financial Model v3.1 — India ARPU and
  gross-margin target]**
* **West.** Cost per association (≈$0.93–1.11) is well under typical per-door pricing at
  any realistic association size. The design is not constrained by cost there.
* **Counterfactual (Alternative A in the West).** At 10,000 orgs, one DB per association
  would mean **250,000 databases, ≈250 clusters at India density.** Fleet migrations would
  take about 25× longer, and every portfolio report would fan out. This confirms the
  decision.
* **Cell ceiling.** We plan to split a cell (`in-1` → `in-2`) at about **10,000 DBs or 20
  clusters**, whichever comes first. A cell keeps a blast radius and migration window we
  can manage, and growth is handled by adding cells, not by making one cell bigger.

---

## 7. Acceptance criteria

This ADR is *Accepted* when all of the following hold:

1. **A new engineer can answer three questions for any market from the TL;DR table
   alone:** (a) who the Organization is, (b) what one database contains, and (c) why.
   Verify with two engineers new to the project during review.
2. Each [VERIFY] tag has been reconciled with BRD v3.1, Feature Catalogue v3.1 or Financial
   Model v3.1, or deliberately overridden with a note.
3. Follow-up tickets exist for M-1 … M-11 and for ADR-00x (RLS within portfolio databases).
4. The vocabulary in §1.2 has been added to the project glossary and is enforced by the
   CI check in M-10.

### Quick self-test for reviewers

| Question | Answer |
|---|---|
| A 300-flat society in Bengaluru signs up. How many databases? | One, in `in-1`, containing one association. |
| A Dubai OA management firm with 140 communities signs up. How many databases? | One, in `ae-1`, containing 140 association rows. |
| A self-managed 80-lot HOA in Tampa signs up. | One DB in `us-1`, `shape = portfolio`, with one association. |
| An Indian FM company manages 30 societies on ASMS. | 30 Organizations, 30 DBs, plus Partner-console grants. No shared DB. |
| A Florida HOA switches from management company A to B. | A transfer job inside `us-1` copies the association subtree from A's DB to B's DB. ULIDs mean no key remapping. |
| Can the founders' dashboard show "total units managed across all markets"? | Yes, from aggregates each cell pushes to the control plane. Never by querying cells. |
| What do we call a person renting a flat? | An **Occupant** (`occupancy_type = renter`). Never a tenant. |
