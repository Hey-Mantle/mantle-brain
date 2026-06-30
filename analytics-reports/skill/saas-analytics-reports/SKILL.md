---
name: saas-analytics-reports
description: >-
  Build a SaaS / app-billing analytics reporting suite — the 16 reports MRR
  (with discount & cadence rules), ARR, revenue/payout, active subscriptions,
  active customers/installs, growth, churn (logo/subscription/revenue),
  retention & cohorts, ARPU, LTV, trials, install funnel, installs & uninstalls
  with reasons, usage metrics, forecasting, and traffic sources (GA4 +
  BigQuery). Use when the user wants subscription/revenue analytics, MRR/churn/
  retention reporting, a metrics API, time-series dashboards reconstructed
  "as-of" past dates, or to migrate off Mantle's analytics into their own
  codebase.
---

# SaaS Analytics Reports

## What this skill does

Helps you implement a **trustworthy SaaS analytics suite** — 16 reports over
subscription, event, and metered-usage data — in your own app. The hard parts it
gets right:

- **"As-of" reconstruction:** answering "what was MRR / active subs on a past
  date?" by rebuilding state per time-bucket from lifecycle dates, with no
  snapshot tables.
- **Definition discipline:** MRR/churn/retention as *configurable families*
  (annual cadence, permanent vs temporary discounts, usage, trials, frozen subs,
  count-by-customer, rolling vs cohort).
- **Trust:** validators that catch source⇄index drift, MRR method divergence, and
  silently-rewritten history.

The full, precise mechanics live in
**[`references/implementation-spec.md`](./references/implementation-spec.md)**.
Read it before writing code — it has the data model, the per-report formulas, the
reconstruction algorithm, and the edge cases. This file is the *procedure*; the
spec is the *reference*.

## When to use it

Trigger on intents like: "build MRR / ARR / churn / retention reporting," "a
metrics API for my billing app," "time-series dashboards for subscriptions," "LTV
and ARPU," "trial conversion," "install funnel," "uninstall reasons," "usage
metrics from my event stream," "MRR forecasting," "traffic-source attribution
from GA4," or "migrate off Mantle analytics."

## How to drive the implementation

Work through these phases. **Do not dump code blindly** — discover the user's
data shape and existing analytics first, then map the spec onto it.

### Phase 1 — Orient (ask, then read)
1. Determine the stack: language, OLTP database, whether they already have a
   search/document store (Elasticsearch/OpenSearch) and/or a columnar/OLAP store
   (ClickHouse/BigQuery/Snowflake/DuckDB), and a job/cron runner.
2. Find existing analytics: any current MRR/churn code, an event log, a metrics
   API, dashboards.
3. Establish the source-of-truth for: subscriptions + lifecycle dates
   (activated/converted/churn/paused), plans + cadence + discounts, installs +
   install/uninstall events, metered usage events, and charges/transactions.
4. Scope which of the 16 reports they need now (recurring revenue core →
   subscribers → movement → unit economics → acquisition → usage → forward-looking
   → traffic). Recommend building in that order.

### Phase 2 — Read the reference
Read [`references/implementation-spec.md`](./references/implementation-spec.md)
end to end. The §0 invariants are non-negotiable; the §5 MUST/MUST NOT rules are
the bugs you're being paid to avoid; §6 is what makes the numbers trustworthy.

### Phase 3 — Storage roles & write-time normalization (spec §1)
Confirm the three roles (OLTP source of truth; a denormalized per-subscription +
per-event index for stock/movement metrics; a columnar store for the metered
firehose and traffic). Implement the **write-time precompute**: `monthlyAmount`
(cadence §1.5 + permanent-discount only), `netChange` per event, resolved
`billingType`. Map every field name onto the user's schema. If they have no
columnar store, usage/traffic can start in the OLTP/index store and move later.

### Phase 4 — The as-of engine (spec §2)
Implement the **one shared as-of predicate** (§2.1) with missing-field semantics
(§2.2), the per-bucket reconstruction loop (§2.3) with batched multi-search, and
the two edge buckets (§2.6: hidden leading, dropped trailing). Add Method B
(event-cumulative §2.4) only if they want cross-validation.

### Phase 5 — Cross-cutting pipeline (spec §3)
Build one **response schema** (§3.4), the period/interval parser with a **single**
range→interval ladder (§3.3), read-through caching (§3.5), and universal scoping +
org-level reporting defaults (§3.6). Stand up the query layer behind one (or two)
front doors — never two implementations of the same metric.

### Phase 6 — The reports (spec §4)
Implement the reports they scoped, each as a configurable family. Suggested order:
1. **MRR** (§4.1) — get cadence + discount rules right first; everything builds on
   it. 2. **ARR** (§4.2). 3. **Revenue/payout** (§4.3). 4. **Active subs/customers/
   installs** (§4.4–4.5). 5. **Growth** (§4.6). 6. **Churn** (§4.7). 7. **Retention/
   cohorts** (§4.8). 8. **ARPU/LTV** (§4.9–4.10). 9. **Trials** (§4.11). 10.
   **Funnel** (§4.12). 11. **Installs/uninstalls + reasons** (§4.13). 12. **Usage**
   (§4.14). 13. **Forecasting** (§4.15). 14. **Traffic/GA4** (§4.16).

### Phase 7 — Trust (spec §6)
Implement at least the **source⇄index consistency** validator and the
**retroactive-drift** snapshot/diff. Add **method-drift** if Method B exists. Wire
them to the user's scheduler with severity thresholds (absolute epsilon +
percentage).

### Phase 8 — Verify
Walk the spec §7 acceptance scenarios as a checklist. At minimum prove: as-of MRR
reconstructs correctly and a backdated churn rewrites that historical point;
cadence + discount rules hold; stock metrics summarize to last-point and flow
metrics to sum; churn denominator is start-of-prior-period; ARPU/LTV guard their
divisions; the leading bucket drives the first delta and the trailing bucket is
provisional; the validators flag injected drift.

## Guardrails

- **Preserve the invariants** (spec §0): normalize at write time; reconstruct
  as-of (no snapshot tables); aggregate not count hits; one as-of predicate; tenant
  id is the hard boundary; trust via validators. Breaking any reintroduces slow
  queries, inconsistent numbers, or silent drift.
- **Only permanent discounts in the MRR baseline** — never temporary or app-credit
  discounts (spec §1.5).
- **Stock vs flow** decides the summary value — last-point vs sum (spec §3.4).
- **Define the as-of predicate once** and reuse it in queries *and* validators, or
  the validator validates nothing (spec §6.1).
- **Missing fields are meaningful** — handle absent `paused`/`churnDate`/
  `billingType` explicitly (spec §2.2).
- **Columnar store:** parameterize values, whitelist identifiers (reject hyphens),
  `FINAL`/dedup only for billing-exact numbers (spec §4.14).
- **Guard divisions** — zero population, zero churn, zero prior-MRR (spec §5).
- **GA4 attribution is directional** (cohort-by-UTM), and its identity bridge is
  fragile — don't present it as deterministic per-conversion truth (spec §4.16).
- Prefer editing the user's existing models/stores over inventing parallel ones.
  Match their conventions and field names.
