# SaaS Analytics Reports — a build guide

*Part of [mantle-brain](../README.md): an open-source library to help Shopify
Partners recreate Mantle's tools as Mantle winds down.*

This guide explains **how Mantle's analytics reporting worked and how to rebuild
it in your own app** — the 16 reports that make up a SaaS/app-billing analytics
suite (MRR, ARR, revenue, active subscriptions, active customers, growth, churn,
retention & cohorts, ARPU, LTV, trials, install funnel, installs & uninstalls
with reasons, usage metrics, forecasting, and traffic sources). It is about
**strategy, not source code** — the formulas, the data model, the
reconstruction technique, and the edge cases that decide whether your numbers
are trustworthy.

If you'd rather hand the job to a coding agent, this folder ships two companions:

- **[`implementation-spec.md`](./implementation-spec.md)** — a dense,
  imperative build spec (data model, report-by-report definitions, the as-of
  reconstruction algorithm, the gotchas, acceptance checks). Paste it into
  Claude/Cursor and say "implement this in my app."
- **[`skill/saas-analytics-reports/`](./skill/saas-analytics-reports/)** — a
  Claude Code / Agent skill. Copy it into your app's `.claude/skills/` and the
  agent will walk the build for you. See its
  [README](./skill/saas-analytics-reports/README.md).

---

## The problem these reports solve

A billing app accumulates a stream of facts — subscriptions activate, convert,
upgrade, downgrade, pause, churn; merchants install and uninstall; metered
events pour in; money is billed, refunded, and paid out. Founders want a handful
of **trustworthy numbers** out of that stream: *What is my MRR? Is it growing?
Who's churning and why? What's a customer worth? Where does my traffic come
from?*

The hard part isn't the arithmetic. It's three things:

1. **"As-of" questions.** "What was MRR on March 1st?" requires reconstructing
   the state of the world on a past date from an ever-changing event history.
2. **Definition drift.** "MRR" is not one formula — annual plans, discounts,
   usage charges, trials, and frozen subscriptions each fork the definition. Two
   reasonable people compute different MRRs.
3. **Trust.** A financial metric that silently rewrites last quarter's number,
   or disagrees with itself depending on which screen you look at, is worse than
   no metric at all.

This guide is mostly about #1 and #3 — the machinery — with #2 nailed down
report by report.

---

## The core architecture

> Normalize hard math at **write time**; reconstruct every metric **as-of each
> time bucket** at read time; compute with **aggregations, never row counts**;
> and guard the whole thing with **cross-checks** because nothing is snapshotted.

Mantle splits storage into three roles. You don't need the same vendors, but you
almost certainly need the same three roles:

```
   ┌──────────────────────┐   the transactional source of truth.
   │  OLTP (Postgres)      │   subscriptions, plans, installs, customers,
   │  system of record     │   discounts, lifecycle timestamps.
   └──────────┬───────────┘
              │ one enrichment pass, dual-written
      ┌───────┴────────────────────────────┐
      ▼                                     ▼
┌──────────────────────┐         ┌──────────────────────────┐
│ Search/document index│         │ Columnar OLAP store       │
│ (Elasticsearch)      │         │ (ClickHouse)              │
│                      │         │                           │
│ one denormalized doc │         │ one row per metered event │
│ per subscription /    │         │ high-volume, append-heavy │
│ per lifecycle event, │         │                           │
│ with PRECOMPUTED      │         │ usage metrics, traffic    │
│ monthlyAmount, churn  │         │ events, funnels at scale  │
│ dates, netChange.     │         │                           │
│                      │         │                           │
│ backs all $ / sub /   │         │                           │
│ retention metrics via │         │                           │
│ as-of reconstruction. │         │                           │
└──────────────────────┘         └──────────────────────────┘
```

- **OLTP** owns truth and relations. Reports rarely query it directly except to
  resolve filters (which apps, which installs belong to a customer/segment).
- **The search index** holds two denormalized shapes: a **per-subscription
  document** (lifecycle dates + a precomputed normalized `monthlyAmount`) and a
  **per-event document** (`subscribed`, `unsubscribed`, `upgraded`, … each with
  a precomputed `netChange` = MRR delta). Almost every money/subscription metric
  is an aggregation over one of these.
- **The columnar store** holds the raw metered-event firehose and marketing/GA
  traffic events — anything too high-volume to put in the search index.

Why precompute at write time? Because the alternative — applying cadence
normalization, discounts, and proration inside every read query — is both slow
and a recipe for inconsistency. Push it into the indexer once; keep read-time
math to **sums and date comparisons**.

---

## The one technique that matters: as-of reconstruction

There are **no snapshot tables.** Mantle does not store "MRR was $X on day D."
Instead, to draw a time series it generates the bucket dates and, for each bucket
end date **D**, runs an aggregation whose *filter reconstructs "active at D"*:

```
generate buckets [start … end] at the chosen interval
for each bucket end date D:
    filter the subscription index to docs that were live at D:
        tenantId matches              (hard tenant boundary)
        test = false, customer.test = false
        conversionDate <= D            (had converted/activated by D)
        (churnDate is missing OR churnDate > D)   (not yet churned at D)
        not paused as-of D
        [+ app / plan / installation / billingType filters]
    aggregate: sum(monthlyAmount) grouped by interval (monthly vs annual)
```

Each bucket is an independent "rebuild the world at D and sum it" query. Fan them
out as one multi-search, **batched** (e.g. 15 sub-queries at a time) so a wide
range × a huge filter list doesn't blow up memory.

This design has a beautiful property and a dangerous one:

- **Beautiful:** a single late correction — say a backdated churn — automatically
  flows into *every* historical point. History stays self-consistent with the
  source.
- **Dangerous:** because nothing is frozen, **last month's number can change.**
  That's "retroactive drift," and it's why you need validators (below).

Two reconstruction methods, deliberately kept in parallel:

| Method | How | Good for |
|---|---|---|
| **State-based** | Rebuild active-set at each D from subscription docs, sum `monthlyAmount`. | The default; robust, self-correcting. |
| **Event-based** | Sum `netChange` of all events before the window = a starting balance, then running-total per-bucket `netChange`. | Faster (2 aggregations vs N); an accounting "movement" view. |

They should agree within a small tolerance. A validator runs both daily and
flags divergence — see *Trust*.

### Two universal edge buckets

Every time series fetches **one extra hidden bucket *before* the visible range**
so the first visible point can compute a correct period-over-period change; strip
it before returning. And the **trailing bucket is always partial** (events still
arriving) — drop it before any drift comparison or you'll chase phantom changes.

---

## Mental model

| Concept | What it means |
|---|---|
| **Stock vs flow** | *Stock* metrics (MRR, active subs) are a **level as-of D** — reconstructed, inherently cumulative, summary = last point. *Flow* metrics (new revenue, net installs, churn) are **independent per-bucket sums** — summary = sum of buckets. Get this wrong and your scorecard shows the last day instead of the month. |
| **Normalize at write time** | Cadence (annual→monthly), permanent discounts, and billing-type resolution are baked into `monthlyAmount` / `netChange` when the doc is indexed — not recomputed per query. |
| **Definition is a family** | "MRR" depends on org-level toggles: include usage? include trials? include annual? paid plans only? count by customer or by subscription? rolling vs cohort churn? Treat each metric as a *configurable family*, not one formula. |
| **As-of predicate** | The single filter that says "this subscription was live at date D." Define it **once** and reuse it for the MRR sum, the active-count, *and* the consistency cross-check — or the cross-check tests nothing. |
| **Missing-field semantics** | Absent fields carry meaning: missing `paused` = not paused; missing `churnDate` = not churned; missing `billingType` = your default platform. These defaults are part of the as-of predicate. |
| **Aggregations, not hits** | Counts and sums come from aggregations (`sum`, `value_count`, `cardinality`, `date_histogram`), never from a total-hits number. Distinct counts use approximate cardinality; group-by series cap at top-N. |
| **Retroactive drift** | History is recomputable, so it can move. Snapshot each run to cheap storage and diff overlapping dates to catch unexpected rewrites. |

---

## The 16 reports

Each entry below is the **definition + the trap**. Exact formulas and parameters
live in [`implementation-spec.md`](./implementation-spec.md).

### Recurring revenue

1. **MRR** — Sum of normalized monthly recurring amount across subscriptions
   live as-of D. **Cadence rule:** a 30-day plan passes through as-is; any other
   cadence is annualized (`amount × periods-per-year ÷ interval-count`) then
   divided by 12. **Discount rule:** only *permanent* discounts reduce the
   indexed monthly amount — *temporary* discounts are excluded from the baseline.
   Discounts delivered as *app credits* (not price reductions) don't touch MRR at
   all; they surface in payout. Optional add-ons: `includeUsage` (adds metered
   charges), `includeTrials` (adds trial-plan value), `includeAnnual` (folds
   annualized plans into the total).
2. **ARR** — `MRR × 12`. Nothing more. It inherits every MRR toggle and assumes
   today's MRR repeats unchanged for a year (no churn/expansion modeling).

### Cash & revenue

3. **Revenue / net revenue / payout** — Money actually billed, from the
   transactions/charges store (not the subscription index). **Gross** = sum of
   charged amounts. **Net** = gross minus the platform's revenue share (the share
   rate can change once an account crosses a lifetime-revenue threshold, so two
   transactions in one period may use different rates). **Payout** = net minus
   refunds, taxes, and app credits, bucketed into the platform's **payout
   windows** (e.g. 1st–15th / 16th–EOM), not calendar months. Trap: revenue is a
   *flow* metric; refunds are negative rows; app credits reduce payout but never
   MRR.

### Subscribers & customers

4. **Active subscriptions** — Count of subscriptions live as-of D (the same
   as-of predicate as MRR, aggregated with `value_count`). Trap: trials are
   excluded unless `includeTrials` flips the activation gate from `conversionDate`
   to `activatedAt`.
5. **Active customers / active installs** — Two flavors. *Active customers* =
   distinct customers with a live subscription (`cardinality` on customer id —
   approximate). *Active installs* = installs whose **latest install event is more
   recent than their latest uninstall event** as-of D, computed by a per-doc
   scripted aggregation over denormalized event-timestamp arrays. Install-level,
   not subscription-level: an active install can hold a frozen subscription.

### Movement

6. **Growth / net installs** — `(installed + reinstalled + reactivated) −
   (uninstalled + deactivated)` per bucket, from the event store. A *flow* metric;
   it can go negative. Churn *within* a bucket cancels growth within that bucket
   (correct).
7. **Churn** — Three lenses, each *gross* (losses only) or *net* (losses minus
   recoveries/expansion):
   - **Logo churn** = (uninstalls − reinstalls) ÷ active installs at the start of
     the prior period.
   - **Subscription churn** = (unsubscribes − resubscribes) ÷ active subs at
     start.
   - **Revenue churn** = lost MRR (downgrades + cancels + usage-down) ÷ MRR at
     start; *net* revenue churn subtracts upgrades/expansion and can be negative
     (healthy). Denominator is always **start-of-prior-period**, computed over a
     rolling 30-day window by default.
8. **Retention & cohorts** — Group entities by their first month (cohort), then
   track the % surviving N months later, as a triangular cohort table. Three
   variants mirror churn: **logo**, **subscription**, and **revenue** retention
   (revenue retention captures expansion, so NRR can exceed 100%). Built by
   applying per-cohort event deltas month over month. Traps: resubscribes mask
   true churn; the most-recent cohort has the shortest window (recency bias);
   small cohorts are noisy.

### Unit economics

9. **ARPU** — `MRR ÷ active population`, per bucket. Population is configurable:
   subscriptions, or distinct customers (`activeSubscribersByCustomerOnly`), or
   paying customers only. Guard the zero-population case → 0, not NaN.
10. **LTV** — `ARPU ÷ monthly churn rate` (instantaneous, forward-looking — *not*
    cohort-cumulative). Trap: as churn → 0, LTV → ∞; guard it. This is a
    run-rate estimate, not measured lifetime revenue.

### Acquisition

11. **Trials** — From subscriptions carrying trial dates. *Started* = has a trial
    start; *converted* = survived past trial end (no churn ≤ trial end);
    *canceled* = churned during the trial. **Conversion rate** = converted ÷
    (converted + canceled). Edge: a churn exactly at trial-end counts as canceled.
12. **Install funnel** — An ordered sequence of stages (optionally:
    marketing-site visit → app-store view → add-app click → install → subscribe),
    counted per user with a scripted state-machine over event-timestamp arrays.
    Conversion at stage N = count(N) ÷ count(stage 0). A skip-allowed flag decides
    whether intermediate stages may be bypassed. Marketing stages join a separate
    traffic store and are *capped* so downstream counts can't exceed upstream.
13. **Installs & uninstalls with reasons** — Install/uninstall counts over time
    from the event store, plus an **uninstall-reason breakdown**. Reasons
    originate from the platform's uninstall survey (a free-text/enum reason),
    normalized to a canonical code via a lookup table, with an **LLM classifier
    fallback** for unknown/foreign-language/composite reasons. Optionally exclude
    *store-closure* reasons (scheduled cancellation, deactivation, store
    closing/pausing) to isolate customer-driven churn.

### Usage

14. **Usage metrics** — Configurable aggregations over the metered-event firehose
    in the columnar store: `count`, `sum(property)`, `unique(property)`, `max`,
    plus billing-specialized sums. Time-bucketed histograms, optionally grouped
    by event name, billing status, or a top-N property value. The money side
    ("usage charges") comes from the *charges* index, not the event store — the
    two are reconciled against each other.

### Forward-looking

15. **Forecasting** — A heuristic **growth-rate extrapolation** off run-rate
    history (no ML/regression). Take the last ~6 monthly growth rates, blend them
    (recency-weighted by default), damp for deceleration and volatility, floor at
    a small positive if history was positive, apply optional seasonal factors,
    then compound the latest MRR forward N months. It's directional; it carries no
    confidence interval and biases optimistic for healthy accounts.
16. **Traffic sources (GA4 + BigQuery)** — Pull GA4's BigQuery event export
    (daily-sharded `events_*` tables) via a service account, extract install /
    page-view / add-app events with their `traffic_source` and UTM dimensions, and
    bridge GA's pseudonymous `user_pseudo_id` to your real installs. Aggregate
    reporting is **cohort-by-UTM**: visitors (from the traffic store), app-store
    actions (from listing page views), and revenue (from subscriptions joined back
    through installs) are each grouped by the same UTM key and merged — directional
    attribution, not deterministic per-conversion.

---

## Trust: the part everyone skips

Because metrics are reconstructed, not stored, two failure modes are invisible
until a customer notices: **the index drifted from the source**, and **history
silently changed.** Mantle runs two daily validators; copy the idea even if not
the code.

1. **Source ⇄ index consistency.** Recompute each active subscription's expected
   normalized amount and billing type *with the same helpers the indexer uses*,
   then diff against the index by id. Flag: missing-from-index (lag/loss),
   stale-in-index (live in index, gone from source), and per-field mismatches
   (use an absolute epsilon like $0.01 plus a percentage threshold). Cross-check
   that the latest time-series total equals the sum of contributing docs — where
   "contributing" is defined by the **exact same exclusion rules** the series
   uses.
2. **Method drift + retroactive drift.** Run both MRR methods (state-based and
   event-based) over ~90 days; flag cross-method divergence beyond a tolerance.
   Snapshot each run to cheap storage; on the next run, diff overlapping dates and
   flag any past day that moved more than ~1%. **Always drop the trailing partial
   bucket before comparing.**

---

## Cross-cutting conventions worth copying

- **One response schema for every report.** A flat
  `{ value, format, period, periodStart, periodEnd, timeSeries?, timeSeriesInterval? }`
  contract, where `format` is a display hint (`money` / `percent` / `count` /
  `number`) decoupled from the number, and `timeSeries` is present only when an
  interval was requested. One schema → one client renderer → trivial docs.
- **Two front doors, one query layer.** A rich internal dispatch (a registry of
  per-domain query functions) and a set of thin, documented public endpoints both
  call the *same* query functions. There is never a second implementation of MRR
  math to drift.
- **Universal scoping.** Every report accepts the tenant id (never
  client-supplied), plus app / installation / customer / segment / plan filters.
  Empty app filter means "all apps." Customer and segment resolve down to
  installation ids (which can be tens of thousands → drives the batched
  multi-search).
- **Org-level defaults.** Unset toggles fall back to per-tenant reporting
  preferences (paid-plans-only, app-events-for-MRR, paying-customers, churn
  strategy, count-by-customer). The same query computes differently per tenant.
- **Cache read-through.** Key on `metric + tenant + the full query object`; short
  TTL (≈10 min prod). The query object *is* the cache key, so send canonical
  params.
- **Columnar-store security.** Bind all *values* as typed query parameters;
  *identifiers* you can't parameterize (property names, group-by fields) must be
  whitelisted with a strict regex/enum (reject hyphens — `properties.plan-id`
  parses as subtraction). Always scope by tenant id.
- **Dedup is a cost dial.** On a `ReplacingMergeTree`-style table, force
  query-time dedup (`FINAL`) *only* for billing-exact numbers; skip it for
  proportional/exploratory charts and accept transient merge-window duplicates
  (it can be 60× faster).
- **Tolerate dirty JSON.** Dynamic-typed event properties need an explicit cast
  per use and "or-zero" / "cast-or-null" variants so one bad row can't fail the
  whole query.

---

## Gotchas — the lessons that cost real trust

- **Stock vs flow decides your summary value.** Last-point for stock, sum for
  flow. Defaulting everything to last-point silently under-reports flows.
- **The hidden pre-window bucket** must be fetched, used for the first delta,
  then sliced off. Off-by-one here corrupts every delta indicator.
- **The trailing bucket is partial** — exclude it from drift checks and treat its
  value as provisional.
- **Define the as-of predicate once.** Duplicating it (in the query and again in
  the validator) means the validator validates nothing.
- **Missing fields are meaningful.** Treat absent `paused`/`churnDate`/
  `billingType` as their defaults explicitly; a naive `field == false` breaks
  history.
- **Normalize cadence and discounts at write time**, and apply *only permanent*
  discounts to the baseline. Temporary discounts and app-credit discounts do not
  reduce MRR.
- **Churn/retention denominators are start-of-prior-period**, not average or
  end-of-period. Net variants can be negative / exceed 100% — that's expansion,
  not a bug.
- **Guard divisions:** zero population → ARPU 0; zero churn → don't divide for
  LTV.
- **Recency bias** in the latest cohort and the latest forecast month — both have
  the least data; label them provisional.
- **Reconciliation, not faith.** Usage *events* and usage *charges* live in
  different stores; reconcile them. Source and index drift; validate them.
- **GA identity is fragile.** The pseudonymous-id → install bridge breaks on new
  device, cleared cookies, or unresolved shop id; those installs get no
  attribution. Cohort-by-UTM is directional, not per-conversion truth.

---

## What to build vs. what to skip

**Reproduce (the actual reporting engine):**
1. The three storage roles: OLTP source of truth; a denormalized
   per-subscription + per-event index for stock/movement metrics; a columnar
   store for the metered-event firehose and traffic.
2. Write-time normalization into the index (`monthlyAmount`, `netChange`,
   resolved billing type) so read-time is sums + date math.
3. The **as-of reconstruction** loop (one aggregation per bucket), with the
   hidden leading bucket and dropped trailing bucket.
4. One canonical **as-of predicate**, reused everywhere.
5. The report definitions in [`implementation-spec.md`](./implementation-spec.md),
   each as a configurable family with org-level defaults.
6. One **response schema** and **two front doors** over one query layer.
7. The **validators** (source⇄index, method-drift, retroactive-drift). This is
   what makes the numbers trustworthy.
8. The gotchas above.

**Skip or substitute (Mantle-specific):**
- The exact vendors. Any OLTP + any aggregating store + any columnar store will
  do; the roles matter, not Elasticsearch/ClickHouse specifically.
- The two-parallel-MRR-methods setup is belt-and-suspenders — one method plus a
  source⇄index check is enough to start; add the second method only if you need
  the cross-validation.
- The LLM uninstall-reason classifier is a nicety; a lookup table covers most
  reasons. Add the classifier only when free-text/foreign reasons matter.
- The AI usage-metric generator is config tooling, not reporting math — skip
  until you have many apps to onboard.

---

## Where to go next

- Building by hand? Work down the "Reproduce" list; reach for
  [`implementation-spec.md`](./implementation-spec.md) when you need the exact
  per-report formulas, parameters, and the reconstruction algorithm.
- Want an agent to do it? Paste
  [`implementation-spec.md`](./implementation-spec.md) into your coding agent, or
  install the [skill](./skill/saas-analytics-reports/) into your app's workspace.
