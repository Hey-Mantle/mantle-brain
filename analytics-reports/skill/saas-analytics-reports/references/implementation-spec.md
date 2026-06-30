# SaaS Analytics Reports — Implementation Spec (for agents)

> **Audience:** a coding agent implementing a SaaS/app-billing analytics suite —
> 16 reports — in an arbitrary app. This is a precise, self-contained build spec;
> no external files required. A human-readable walkthrough of the same system
> lives in [`README.md`](./README.md).
>
> **Provenance:** distilled from the production behavior of Mantle's analytics
> reporting. Stack-agnostic; map the generic store/field names below onto your own
> infrastructure. Where a rule encodes a hard-won lesson it is marked **MUST** /
> **MUST NOT**.
>
> **Altitude:** this is *strategy*, not Mantle source. It gives formulas, the data
> model, the reconstruction algorithm, and the edge cases — not copy-paste code.

---

## 0. Goal & invariants

Build a reporting layer that answers stock questions ("MRR / active subs **as of**
any past date"), flow questions ("revenue / installs / churn **over** a period"),
and cohort questions ("retention of the March cohort"), trustworthily, from an
ever-changing event history.

The invariants that make it work:

1. **Normalize at write time.** Cadence normalization (annual→monthly),
   permanent-discount application, and billing-type resolution are computed once
   when a subscription/event is indexed and stored as precomputed fields
   (`monthlyAmount`, `netChange`). Read-time math is **sums + date comparisons
   only**.
2. **Reconstruct as-of, don't snapshot.** There are no "MRR on day D" tables.
   Every time series is built by running one aggregation per bucket whose filter
   reconstructs "live at D." A late correction flows into all of history
   automatically.
3. **Aggregations, never hit counts.** Counts/sums come from `sum`, `value_count`,
   `cardinality`, `date_histogram`. Distinct counts are approximate; group-by
   series are top-N.
4. **One as-of predicate, defined once, reused everywhere** — by the MRR sum, the
   active-count, and the consistency cross-check.
5. **Tenant id is the hard boundary.** Every query filters by tenant id (never
   client-supplied) and excludes test data. Inject it in a query builder; don't
   rely on per-call discipline.
6. **Trust is a feature.** Because history is recomputable, it can drift. Daily
   validators (source⇄index, method-drift, retroactive-drift) are part of the
   system, not an afterthought.

Preserve these six and everything else is bookkeeping.

---

## 1. Data model

Three storage roles. Names are illustrative; map to your stack.

### 1.1 OLTP (system of record)
Your transactional DB. Holds `subscriptions`, `plans`, `discounts`,
`appInstallations`, `customers`, `charges`/`transactions`, `customerSegments`.
Reports query it mainly to **resolve filters** (which apps; which installs belong
to a customer or segment) and to recompute expected values in the validators.

### 1.2 Subscription index (denormalized document store) — the workhorse
One document per subscription, refreshed on every change. **MUST** carry:

| Field | Notes |
|---|---|
| `id`, `tenantId` | |
| `appId`, `appInstallationId`, `customerId` | Scoping. |
| `plan.interval` | Cadence bucket key — e.g. `EVERY_30_DAYS` vs `ANNUAL`. |
| `amount` | Raw plan amount. |
| `monthlyAmount` | **Precomputed normalized monthly recurring amount** (cadence + permanent discount baked in). This is what MRR sums. See §1.5. |
| `averageMonthlyUsage` | Precomputed monthly-equivalent metered revenue (for `includeUsage`). |
| `activatedAt` | First activation. |
| `conversionDate` | When it became paid (trial → paid). Gate for MRR. |
| `churnDate` | When it ended. Absent ⇒ not churned. |
| `frozenAt` / `paused` / `pausedUntil` | Pause state. Absent `paused` ⇒ not paused. |
| `trialStartsAt` / `trialExpiresAt` / `trialStatus` | Local trial tracking. |
| `billingType` | `shopify` / `stripe` / `manual`. Absent ⇒ your default platform. |
| `test`, `customer.test` | Both must be false to count. |

### 1.3 Event index (denormalized event log)
One document per lifecycle event. **MUST** carry `type`
(`subscribed`/`unsubscribed`/`upgraded`/`downgraded`/`usage_upgrade`/`usage_downgrade`/
`resubscribed`/`frozen`/`unfrozen`/`installed`/`uninstalled`/`reinstalled`/
`reactivated`/`deactivated`/`trial_started`/…), `occurredAt`, `tenantId`, scoping
ids, `test`, an `ignored` flag, and a precomputed **`netChange`** = the MRR delta
this event caused (signed; positive on upgrade/subscribe, negative on
churn/downgrade). Uninstall events also carry `uninstallEvents.reasonCode` (§4.13).

### 1.4 Columnar event store (OLAP) — the metered firehose
One row per metered usage event and per marketing/GA traffic event. Too
high-volume for the document index. Use a `ReplacingMergeTree`-style engine keyed
for dedup. **MUST**:
- Store the event payload in a JSON/Map column (`properties`) and **promote**
  hot/billing fields to dedicated columns (`billedAmount`, `billingStatus`, …).
- Carry precomputed sort helpers (`primaryEntityId = appInstallationId ?? customerId ?? contactId`, `resolvedAppId`) to keep the sort key cheap.
- Partition by month; order by `(tenantId, resolvedAppId, eventName, primaryEntityId, eventId)`.

### 1.5 Write-time normalization (the precompute helpers)
**Cadence → monthly** (compute `monthlyAmount` at index time):
```
function monthlyAmountFromRecurringRules(amount, recurringInterval, recurringIntervalCount):
    if recurringInterval == "day" and recurringIntervalCount == 30:   # the 30-day-cycle special case
        return amount                                                 #   passes through untouched
    amountPerYear = switch recurringInterval:
        "day"   -> amount * 365 / recurringIntervalCount
        "week"  -> amount * 52  / recurringIntervalCount
        "month" -> amount * 12  / recurringIntervalCount
        "year"  -> amount       / recurringIntervalCount
    return amountPerYear / 12
```
**Discount rule (MUST):** apply **only permanent discounts** (no end date) to the
indexed `monthlyAmount`. **MUST NOT** apply temporary/expiring discounts to the
baseline. Discounts delivered as **app credits** (a credit method, not a price
reduction) **MUST NOT** reduce `monthlyAmount` at all — they reduce payout only.

Multi-line subscriptions: sum line-item amounts (each after its own permanent
discount) before normalizing.

---

## 2. The as-of reconstruction engine (§ the core)

### 2.1 The as-of predicate (define ONCE, reuse everywhere)
"Subscription S is live as-of date D" iff:
```
S.tenantId == tenant
AND S.test == false AND S.customer.test == false
AND S.conversionDate <= D                          # converted/activated by D
   (use activatedAt instead of conversionDate when includeTrials)
AND (S.churnDate is missing OR S.churnDate > D)    # not yet churned at D
AND notPaused(S, D)                                # see §2.2
AND [optional app/plan/installation/billingType filters]
```

### 2.2 Missing-field semantics (MUST)
- `notPaused(S, D)` = `paused` field missing OR `paused == false`, AND
  (`pausedUntil` missing OR `pausedUntil < D`).
- Missing `churnDate` ⇒ not churned.
- Missing `billingType` ⇒ treat as your default platform (e.g. shopify).
- A naive `field == false` test **breaks history** because absent ≠ false.

### 2.3 Method A — state reconstruction (default)
```
buckets = generateDateRange(start, end, interval, timezone)
# MUST prepend ONE hidden bucket before `start` (start -= 1 interval) for delta baseline
for each bucket end date D in buckets:           # fan out as ONE multi-search
    query subscription index:
        filter = asOfPredicate(D)
        agg = terms(plan.interval) -> scripted sum(monthlyAmount [+ averageMonthlyUsage if includeUsage])
results[D] = { monthlyAmount, annualAmount(÷-already-normalized), total }
# MUST batch the multi-search (e.g. 15 sub-queries/request) to bound memory
# MUST drop the hidden leading bucket before returning; use it only for the first delta
```
Active-subscription count uses the **same predicate**, aggregated with
`value_count(id)` — or `cardinality(customerId)` when counting by customer.

### 2.4 Method B — cumulative event sum (`appEventsForMrr`)
```
startingBalance = sum(netChange of all relevant events with occurredAt < windowStart)   # split monthly/annual
for each bucket: change[t] = sum(netChange of events in bucket t)
value[t] = value[t-1] + change[t]   # seed with startingBalance
```
Faster (2 aggregations vs N), an accounting "movement" view. **MUST** rely on
correct `netChange` and `ignored` on every event. Methods A and B are expected to
diverge slightly (trial/usage timing) but not structurally — see §7.

### 2.5 Composition
```
MRR(D) = monthlySubs(D)
       + annualSubs(D)        if includeAnnual    (already normalized to monthly)
       + usage(D)             if includeUsage     (billed-usage sum or usage app-events)
       + trials(D)            if includeTrials    (value of subs in active trial)
```
Emit a "Total" series plus breakdown series (monthly/annual/usage/trials, or
group-by top-N). Group-by (plans/apps/customers) wraps the aggregation in a
`terms` bucket, size = top-N (6–10).

### 2.6 Edge buckets (MUST)
- **Leading hidden bucket:** fetch one extra bucket before the visible window;
  use it to compute the first visible point's period-over-period change; **strip
  it** from the returned series.
- **Trailing partial bucket:** the latest bucket is incomplete (events still
  arriving). Return it as provisional; **MUST drop it before any drift
  comparison** (§7).

---

## 3. Cross-cutting pipeline

### 3.1 Two front doors, one query layer
- **Internal dispatch:** a registry `switch(metric)` over all report keys; each
  case unpacks params, calls the report's query function, returns a rich chart
  shape (arrays of `{name, key, data:[{date,value,change}], total, …}`).
- **Typed public endpoints:** a factory `createMetricsEndpoint({ metricName, queryFunction, buildQueryParams, formatResponse, validation })` that wires one
  REST endpoint per metric, reusing the **same** query function and flattening to
  the standard schema (§3.4).

**MUST NOT** implement a metric's math twice. The two doors share query functions.

### 3.2 Request → response pipeline (public)
```
1. auth + scope gate (e.g. read:metrics)
2. validate interval (allowlist) and period (allowlist) -> 400 before any work
3. resolve scope: appIds (empty ⇒ all non-deleted/non-dev apps);
   appInstallationIds (direct, or from customerId, or from customerSegmentId)
4. buildQueryParams(query, {tenant, user}, appIds, installIds) -> {queryParams, parsedParams}
5. read-through cache (§3.5)
6. result = queryFunction(queryParams)
7. response = formatResponse(result, parsedParams)
8. catch-all 500 (log, never leak)
```

### 3.3 Period & interval
- **Period presets** (allowlist): `last_7_days`, `last_30_days`, `last_90_days`,
  `last_12_months`, `year_to_date`, `all_time` (extend as needed). Default when
  unset: `last_30_days`. Compute preset boundaries **in the user's timezone**, then
  to UTC — not UTC by default.
- **Interval auto-select** when not supplied: `≤1d → hour`, `≤30d → day`,
  `≤90d → week`, else `month`. **MUST** use a single range→interval ladder (don't
  duplicate it per front door — that's a real Mantle wart).
- **Custom ranges:** distinguish ISO timestamps (use as-is, UTC) from `YYYY-MM-DD`
  day strings (start-of-day / end-of-day in user tz → UTC).
- **Future dates clamped** to now unless `allowFutureDates` (only forecasting opts
  in).

### 3.4 Response schema (one shape for all reports)
```
{ value,                         # summary: LAST point (stock) or SUM (flow)
  format,                        # "money" | "percent" | "count" | "number"  (display hint)
  period, periodStart, periodEnd,
  timeSeries?: [ { value, periodStart, periodEnd } ],   # ONLY if interval requested
  timeSeriesInterval? }
```
- `periodStart` of a point = its bucket start; `periodEnd` = next bucket's start
  (or now for the last). Clamp first/last to the actual requested range.
- Round values to 2 decimals unless a custom formatter is given.
- **MUST** return the full envelope with `value: 0`, `timeSeries: []` on empty —
  never an empty body.
- **MUST** override the summary to a **sum** for flow metrics (the default
  last-point is correct only for stock metrics).

### 3.5 Caching (MUST)
Read-through. Key = `Metrics:<metric>:tenant:<id>:<JSON.stringify(query)>`. Short
TTL (≈10 min prod). The query object **is** the key ⇒ send canonical params; no
noisy/non-deterministic fields. Provide a `nocache` escape hatch and a skip-list
for must-be-fresh reports (e.g. funnels).

### 3.6 Universal scoping & org defaults
Every report accepts: `tenantId` (from auth), `appIds`, `appInstallationIds`,
`customerId`, `customerSegmentId`, `planIds`/`excludePlanIds`, `billingType`.
Customer/segment **resolve down to installation ids** (can be tens of thousands ⇒
drives batched multi-search). For any unset toggle, fall back to **per-tenant
reporting defaults**: `paidPlansOnly`, `paidUsageOnly`, `appEventsForMrr`,
`payingCustomers`, `churnReportingStrategy` (default `rolling`),
`activeSubscribersByCustomerOnly`. **Treat each metric as a configurable family.**

**MUST** gate permissions at two layers: report-level access AND app-id
intersection (throw if the caller requests an app they can't see) — not only
inside the query.

---

## 4. Report definitions

Each report: data source · definition/formula · key params · gotchas.

### 4.1 MRR
- **Source:** subscription index (Method A) or event index (Method B).
- **Formula:** §2.5. Sum normalized `monthlyAmount` over the as-of-live set per
  bucket; add annual/usage/trials per toggles.
- **Params:** `includeAnnual` (default true), `includeUsage` (default true),
  `includeTrials` (default false), `paidPlansOnly`, `paidUsageOnly`,
  `appEventsForMrr`, `groupBy`.
- **Cadence rule:** §1.5. **Discount rule:** only permanent discounts in the
  baseline; app-credit discounts excluded.
- **Gotchas:** annual is *already* ÷12 in `monthlyAmount` — don't divide again;
  usage rides on top of the discounted base; frozen subs contribute $0.

### 4.2 ARR
- **Formula:** `ARR = latest MRR × 12`. Inherits all MRR toggles.
- **Gotcha:** pure run-rate; no churn/expansion/seasonality modeling.

### 4.3 Revenue / net revenue / payout
- **Source:** transactions/charges store (NOT the subscription index).
- **Gross revenue** = sum of charged amounts in the period (flow).
- **Net revenue** = gross − platform revenue share. **The share rate can switch**
  when an account crosses a lifetime-revenue threshold; transactions before/after
  the threshold date use different rates (no interpolation).
- **Payout** = net − refunds − taxes − app credits, bucketed into **payout
  windows** (e.g. 1st–15th, 16th–EOM), not calendar months. Charges carry a status
  (`upcoming`/`billed`/`due`/`paid`); accrual vs cash basis depends on which
  statuses you include.
- **Gotchas:** refunds are negative rows; app credits cut payout but never MRR;
  taxes are separate negative line items; revenue is a **flow** (summary = sum).

### 4.4 Active subscriptions
- **Source:** subscription index, as-of predicate, `value_count(id)`.
- **Param:** `includeTrials` flips the activation gate to `activatedAt` (earlier).
- **Gotcha:** point-in-time snapshot; an install→uninstall→reinstall same day may
  net invisible at D.

### 4.5 Active customers / active installs
- **Active customers:** `cardinality(customerId)` over the as-of-live subscription
  set (approximate; precision threshold ~10k).
- **Active installs:** from the install index. An install is active as-of D iff its
  **latest install/reinstall/reactivate timestamp > latest uninstall/deactivate
  timestamp**, both ≤ D. Computed by a per-doc scripted aggregation over
  denormalized timestamp arrays (parse → sort → compare). Install-level, not
  subscription-level.
- **Gotchas:** store-closure uninstalls are NOT excluded here; deactivation ==
  uninstall unless reactivated; reinstall flips state back to active.

### 4.6 Growth / net installs
- **Source:** event index. **Formula (flow):** per bucket,
  `(installed + reinstalled + reactivated) − (uninstalled + deactivated)`.
- **Gotchas:** can go negative; churn within a bucket cancels growth within it
  (correct); store-closure uninstalls *are* counted here (filter them in churn,
  not growth).

### 4.7 Churn (logo / subscription / revenue; gross & net)
- **Source:** event index for movement; subscription index/MRR for denominators.
- **Denominator (MUST):** **start-of-prior-period** base (active installs / active
  subs / MRR as-of the period start), computed over a **rolling 30-day window** by
  default.
- **Logo churn** = (uninstalls − reinstalls) ÷ active installs at start.
- **Subscription churn** = (unsubscribes [− resubscribes for net]) ÷ active subs
  at start. Optional `includeFrozen`: frozen counts as churn, unfrozen reverses.
- **Revenue churn** = lost MRR ÷ MRR at start. **Gross** = downgrades + cancels +
  usage-down. **Net** also subtracts upgrades/expansion/resubscribes ⇒ can be
  **negative** (expansion exceeds contraction). Sum the signed `netChange` per
  event type.
- **First-period recategorization (default on):** upgrades/downgrades within ~30
  days of first conversion are remapped to `subscribed`, so early plan-optimizing
  doesn't inflate churn.
- **Store-closure exclusion (optional):** drop uninstall reasons like
  `scheduled_cancellation` / `deactivated` / `store_closing_or_pausing` to isolate
  customer-driven churn.
- **Gotchas:** rolling windows overlap (a sub appears in several); if start base is
  0, churn rate = 0.

### 4.8 Retention & cohorts (logo / subscription / revenue)
- **Cohort key:** first month — by `conversionDate` (cohort strategy) or
  `firstTransactionAt` (rolling strategy).
- **Build:** for each cohort, for each subsequent month, apply event deltas to the
  initial count/MRR. Output a triangular table: cell `[cohort M, age N]` =
  `{ relative %, absolute }`.
- **Logo retention:** active-install state machine per cohort.
- **Subscription retention:** unsubscribe/resubscribe (±frozen/unfrozen) deltas.
- **Revenue retention / NRR:** apply revenue deltas (down/up/usage/resubscribe);
  **NRR can exceed 100%** with expansion. Denominator = initial-month value.
- **Perf:** compute the state machine with a per-doc scripted aggregation
  (parse-once, binary-search per period → O(installs + periods), not O(n²));
  **batch** concurrent scripted aggregations (~4 at a time) to avoid
  circuit-breaker trips.
- **Gotchas:** resubscribes mask true churn; most-recent cohort has the shortest
  window (recency bias — label provisional); small cohorts are noisy; trial
  inclusion inflates early churn.

### 4.9 ARPU
- **Formula:** `MRR(bucket) ÷ activePopulation(bucket)`.
- **Population (configurable):** subscriptions, distinct customers
  (`activeSubscribersByCustomerOnly`), or paying customers only.
- **Gotcha (MUST):** zero population ⇒ 0, not NaN. Including trials in the
  denominator deflates ARPU.

### 4.10 LTV (predicted)
- **Formula:** `ARPU ÷ (monthlyChurnRate)`. Instantaneous / forward-looking, **not**
  cohort-cumulative.
- **Gotchas (MUST):** churn → 0 ⇒ LTV → ∞; guard the division. Uses the
  amount-at-churn snapshot, not an average. Annual plans are normalized to a
  monthly burn rate, which can mis-state LTV for annual-upfront businesses.

### 4.11 Trials
- **Source:** subscription index with trial dates.
- **Started** = has `trialStartsAt` (and activated). **Converted** = `churnDate`
  missing OR `churnDate > trialExpiresAt`. **Canceled** = `churnDate <=
  trialExpiresAt`. **Conversion rate** = converted ÷ (converted + canceled).
- **Params:** date field toggle (`upcoming` → filter by `trialExpiresAt` vs
  `conversionDate`); `paidPlansOnly` (monthlyAmount ≥ 0.01 OR averageMonthlyUsage ≥
  0.01).
- **Gotchas:** churn exactly at `trialExpiresAt` = canceled; multiple trial cycles
  counted by first activation; dedupe per customer with a `collapse`.

### 4.12 Install funnel
- **Sources:** event index (install/subscribe stages), a listing-visitor index
  (pre-install stages), and the columnar traffic store (marketing stages).
- **Stages (ordered):** [marketing-site visit → marketing page view → engaged →
  CTA click →] app-listing view → add-app click → installed → subscribed
  [→ upgraded/downgraded].
- **Counting:** per-user scripted state machine over denormalized
  hyphen-delimited event-timestamp arrays — parse, sort, walk stages. A
  `allowSkippingEvents` flag decides whether stage N→N+2 (skipping N+1) is allowed.
- **Conversion:** count(stage N) ÷ count(stage 0); drop-off = count(N−1) −
  count(N).
- **Marketing cap (MUST):** when the funnel begins with marketing stages, cap
  downstream counts so installs/subscribes can't exceed the marketing-attributed
  upstream — otherwise you get more subscribers than visitors.
- **Gotchas:** referrer matching is wildcard/case-insensitive; first-period
  upgrades recategorized to `subscribed`; anonymous→authenticated crossover merges
  visitor and install identities.

### 4.13 Installs & uninstalls with reasons
- **Source:** event index. Install/uninstall counts over time (see §4.6 for net).
- **Reason origin:** the platform's uninstall survey supplies a reason
  (free-text/enum). Normalize to a **canonical reason code** via a lookup table
  (`platformReasonLabel → code`). For unknown / foreign-language / composite
  reasons, **fall back to an LLM classifier** that picks the best-fit code from the
  code set's training descriptions; cache the result per (tenant, app,
  reason-text). Store the final `reasonCode` on the uninstall event.
- **Breakdown:** `terms(reasonCode)` top-N (e.g. 6).
- **Store-closure exclusion (optional):** exclude
  `scheduled_cancellation`/`deactivated`/`store_closing_or_pausing` for
  customer-driven churn.
- **Gotchas:** translate before classify; historical events predating the
  classifier may have null codes; reinstall ≠ reactivation.

### 4.14 Usage metrics
- **Source:** columnar event store (event volume) + charges index (usage *money*).
- **A usage metric** = saved definition `{ eventName, calculation, overProperty? }`.
  Calculations: `count` → `COUNT(*)`; `sum(prop)` →
  `SUM(toFloat64OrZero(toString(properties.prop)))`; `unique(prop)` →
  `uniq(CAST(properties.prop AS String))`; `max`; billing sums
  (`sum_billed_amount`, `sum_applied_usage_credits`, …) over promoted columns.
- **Shape:** time-bucketed histogram (`toStartOf<Interval>` timezone-aware),
  zero-filled across the range; group by event name (default), billing status, or a
  **top-N property value** (CTE selects top keys, outer query restricts). Cap
  top-N (1–1000, default 25). Optional previous-period comparison query.
- **Security (MUST):** bind all values as typed params; **whitelist** property /
  group-by / dimension identifiers against `^[a-zA-Z0-9_.]+$` (reject hyphens — they
  parse as subtraction). Always scope by tenant id.
- **Dedup cost dial (MUST):** use query-time dedup (`FINAL`) **only** for
  billing-exact aggregations (event-name, billing-status). Omit it for
  proportional/exploratory charts (pie/donut, property-group) — it can be 60×
  faster, and merge-window duplicates are visually negligible.
- **Gotchas:** always force an `endDate` (default end-of-month) to avoid scanning
  corrupt future partitions; `useBilledDate` widens the partition scan (add a
  ±30-day timestamp bound); tolerate dirty JSON via or-zero / cast-or-null.
- **Reconciliation:** run a periodic report that diffs billed-event totals (event
  store) vs charge totals (charges index) per interval to surface drift.

### 4.15 Forecasting (MRR projection)
- **Method:** growth-rate extrapolation off run-rate history — **no ML/regression,
  no external API.** Inputs: historical monthly MRR + month-over-month growth
  rates for `[start − 6 months … end]`.
- **`average`:** projected rate = mean of last 6 growth rates; compound forward:
  `value = prev × (1 + rate/100)` for `forecastMonths` (default 12).
- **`weighted` (default):** recency-weighted blend of last 6 rates (weights
  ≈ `[.03,.07,.15,.2,.25,.3]`); then: **deceleration damping** (if positive but
  slowing, damp by ≥0.5×), **volatility damping** (`× max(0.9, 1 − vol×0.2)`),
  **positivity floor** (if history positive, floor at `min(weighted×0.5, 5)` %),
  **seasonality** (if ≥12 months all-positive, multiply each projected month by a
  per-calendar-month factor). Pin the latest actual MRR to "today" and compound.
- **Output:** "Historical MRR" + "Projected MRR" series. Set `allowFutureDates`.
- **Gotchas (MUST):** guard divide-by-zero on prior MRR and empty history;
  seasonality needs ≥12 all-positive months; the floors bias optimistic for healthy
  accounts; no confidence interval — label directional.

### 4.16 Traffic sources (GA4 + BigQuery)
- **Sources:** GA4 BigQuery export (daily-sharded `<project>.<dataset>.events_*`),
  via a **GCP service account**; a columnar traffic store (`tracking_events`,
  plain `MergeTree`); listing-page-view + install + subscription tables for the
  conversion/revenue side.
- **Extraction:** per event type, `UNNEST(event_params)` and pivot to one row per
  `(event_name, user_pseudo_id, event_timestamp)` via `MAX(IF(key=..., value))`.
  Capture installs (`shop_id`, `shop_url`, `api_key`, `ga_session_id`,
  `user_pseudo_id`), page views (`page_location`, `page_referrer`,
  `traffic_source.{name,medium,source}`, UTM from URL), add-app clicks. Sync
  incrementally with a per-(tenant,app,event) cursor on `event_timestamp`, ~2h
  lookback overlap, batched (~10k); swallow "table not found"/quota/not-enabled to
  exit gracefully. Derive stable join keys: `anonymousId = uuidv5(user_pseudo_id)`,
  `sessionId = uuidv5(ga_session_id)`.
- **Identity bridge:** GA `user_pseudo_id`. (1) Install → AppInstallation by
  `platformId = shop_id` (fallback `myshopifyDomain = shop_url`), gated on
  `api_key == app.apiKey`; persist `gaUserId`. (2) Page view → install by the same
  pseudo-id, linking pre-install browsing to the eventual install. (3) Affiliate
  first-touch: scan the user's last-30-day page views in order for a referral
  param; first match wins (no multi-attribution).
- **Aggregate report = cohort-by-UTM (not per-user join):** union three
  independently-grouped queries on the same UTM key — marketing visitors (traffic
  store, `uniqExact(anonymousId)`), app-store actions (listing page views), revenue
  (subscriptions → installs → page views, sum plan amount). Merge in app code;
  derive stage conversion rates. A separate **funnel** view counts the 5 ordered
  stages with step-over-step conversion.
- **Gotchas (MUST understand):** GA4 SQL uses **string interpolation** of
  config-derived identifiers (project/dataset/event names) — safe *only* because
  they're not user input; never do this with untrusted input. Cohort-by-UTM is
  **directional, not deterministic** — visitors and subscribers of a campaign come
  from different sources counted independently. Identity bridge breaks on new
  device / cleared cookies / unresolved shop id (those installs get no
  attribution). Plain `MergeTree` has no version dedup — rely on cursor/overlap +
  idempotent ids. Untagged traffic collapses to `(not set)` and is partly excluded.

---

## 5. MUST / MUST NOT rules (the gotchas)

Each encodes a real lesson; treat as acceptance criteria.

- **MUST normalize cadence + discounts at write time** into `monthlyAmount`;
  read-time is sums + date math. Apply **only permanent** discounts; **MUST NOT**
  apply temporary or app-credit discounts to the MRR baseline.
- **MUST reconstruct as-of each bucket** with one shared as-of predicate; **MUST
  NOT** rely on snapshot tables.
- **MUST treat missing fields as meaningful defaults** (`paused`/`churnDate`/
  `billingType`).
- **MUST distinguish stock vs flow** for the summary value (last-point vs sum).
- **MUST fetch a hidden leading bucket** for the first delta and **strip it**;
  **MUST drop the trailing partial bucket** before drift checks.
- **MUST aggregate, never count hits**; use `cardinality` for distinct (it's
  approximate — fine for dashboards, not billing), cap group-by at top-N.
- **MUST scope every query by tenant id** and exclude test data, ideally via a
  query builder.
- **MUST use start-of-prior-period denominators** for churn/retention; net
  variants may be negative / exceed 100% (expansion, not a bug).
- **MUST guard divisions:** zero population ⇒ ARPU 0; zero churn ⇒ no LTV divide;
  zero prior-MRR ⇒ forecast/growth rate 0.
- **MUST parameterize values and whitelist identifiers** in the columnar store;
  reject hyphens in property names.
- **MUST use `FINAL`/dedup only for billing-exact numbers**; skip it for
  proportional charts.
- **MUST reconcile** usage events vs usage charges, and **validate** source⇄index
  and method drift (§6).
- **MUST cache read-through** keyed on the canonical query object.
- **MUST gate permissions at report level AND app-id intersection**, not only in
  the query.
- **SHOULD** keep two MRR methods cross-validated; **SHOULD** offer org-level
  reporting defaults so each metric is a configurable family.

---

## 6. Validators (trust)

Run daily; copy the idea even if not the code.

### 6.1 Source ⇄ index consistency
- Pull the daily total time series (~90 days) via the same query the UI uses.
- Load active, non-test subscriptions from OLTP; recompute each one's expected
  `monthlyAmount` + `billingType` **with the same helpers the indexer uses**;
  scroll the index; diff by id. Flag: **missing-from-index** (HIGH; lag/loss),
  **stale-in-index** (HIGH; live in index, gone from source), per-field
  **mismatch** (`monthlyAmount` epsilon $0.01, >$1 ⇒ HIGH; `billingType`/`planId`/
  `paused`).
- **Cross-check:** latest time-series total == sum of contributing index docs
  within ~2%, where "contributing" uses the **exact same exclusion rules** as the
  series (reuse the predicate, don't reimplement it).

### 6.2 Method-drift + retroactive-drift
- Run both MRR methods over ~90 daily buckets. Flag cross-method divergence >2%
  (>10% HIGH), using the state-based method as denominator.
- Snapshot each run to cheap storage; next run, diff overlapping dates (look back
  up to 7 days to survive skipped runs). Any past day that moved >1% = retroactive
  drift (>5% HIGH).
- **MUST drop the trailing partial bucket** before any comparison (the latest day
  is always in flux).
- Use an **absolute epsilon** plus a **percentage threshold** to avoid float-noise
  false positives.

---

## 7. Acceptance scenarios

Implement and verify each:

1. **As-of MRR.** MRR on a past date D equals the sum of normalized monthly
   amounts of subscriptions whose `conversionDate ≤ D` and (`churnDate` absent or
   `> D`) and not paused/test. Backdating a churn to before D lowers that
   historical point.
2. **Cadence.** A $1200/yr plan contributes $100 to MRR; a 30-day plan contributes
   its amount unchanged.
3. **Discounts.** A permanent 20%-off on a $100 plan ⇒ $80 MRR; a temporary
   discount or an app-credit discount ⇒ still $100 MRR.
4. **Toggles.** `includeUsage`/`includeTrials`/`includeAnnual` each add their
   component; off by default per spec.
5. **ARR.** Equals latest MRR × 12.
6. **Revenue vs payout.** Gross = sum of charges; net applies the (possibly
   threshold-switched) share; payout buckets into payout windows and subtracts
   refunds/taxes/credits. Revenue summary = sum, not last point.
7. **Active subs/customers/installs.** Subs = `value_count` over as-of set;
   customers = `cardinality(customerId)`; installs = latest-install >
   latest-uninstall as-of D.
8. **Growth.** Per bucket = adds − removes; can be negative; sums to the net change
   over the range.
9. **Churn (3 lenses, gross/net).** Denominator is start-of-prior-period; net
   revenue churn can be negative; first-period recategorization suppresses early
   plan changes; store-closure exclusion toggles work.
10. **Retention table.** Triangular; cohort by first month; NRR can exceed 100%;
    latest cohort flagged provisional.
11. **ARPU / LTV.** Zero population ⇒ ARPU 0; zero churn ⇒ LTV guarded (not ∞).
12. **Trials.** Converted vs canceled split on `trialExpiresAt`; conversion rate
    correct; churn at exactly trial-end counts as canceled.
13. **Funnel.** Ordered stage counts; conversion = stage/stage0; skip flag
    honored; marketing cap prevents downstream > upstream.
14. **Uninstall reasons.** Known reasons map via lookup; unknown ones route to the
    classifier; store-closure exclusion isolates customer churn.
15. **Usage metric.** count/sum/unique over the columnar store, time-bucketed,
    zero-filled; values parameterized, identifiers whitelisted; `FINAL` only on
    billing-exact paths.
16. **Forecast.** Weighted projection compounds latest MRR with damping/floors/
    seasonality; guards empty history and zero prior MRR; future dates allowed.
17. **Traffic.** GA4 export extracted and bridged by `user_pseudo_id`; cohort-by-
    UTM merges visitors/app-store/revenue; unresolved installs get no attribution.
18. **Edge buckets.** Leading hidden bucket drives the first delta then is
    stripped; trailing bucket is provisional and excluded from drift.
19. **Validators.** Source⇄index mismatch, method divergence, and retroactive
    drift each raise a flag at the right severity; trailing bucket excluded.
20. **Caching & scope.** Identical canonical queries hit cache; tenant id always
    filtered; empty app filter ⇒ all apps; customer/segment resolve to install ids.
