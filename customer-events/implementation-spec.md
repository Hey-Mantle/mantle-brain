# Customer Events from Shopify — Implementation Spec (for agents)

> **Audience:** a coding agent building a pipeline that turns the Shopify
> **Partner API app-events feed** into clean, deduplicated **customer lifecycle
> events** (installed, subscribed, upgraded, downgraded, unsubscribed,
> uninstalled, …) suitable for MRR/churn analytics. This is a precise,
> self-contained build spec — no external files required. A human-readable
> walkthrough of the same system lives in [`README.md`](./README.md).
>
> **Provenance:** distilled from the production behavior of Mantle's
> Shopify-events pipeline. Stack-agnostic; map the generic table/field names
> below onto your own ORM and event store. Where a rule encodes a hard-won
> lesson, it is marked **MUST** / **MUST NOT**.

---

## 0. Goal & invariants

Shopify gives a Partner a single, append-only **app-events feed** per app: a
firehose of low-level facts like `RELATIONSHIP_INSTALLED`,
`SUBSCRIPTION_CHARGE_ACTIVATED`, `SUBSCRIPTION_CHARGE_CANCELED`. It is **not** a
clean lifecycle. The same merchant action ("change my plan") shows up as a
*cancel* immediately followed by an *activate*; a "downgrade to the free plan"
emits **no activation event at all**; an annual plan's charge amount is 12× the
monthly MRR. Your job is to compile that raw stream into the events a billing
dashboard actually wants.

The mechanism, and the invariants that make it correct:

1. **Two layers, never one.** Keep the **raw** feed verbatim in its own table,
   and derive **clean** lifecycle events from it in a separate pass. Never
   classify on the wire. You will re-run the derivation many times; the raw
   layer is the immutable source of truth.
2. **The derivation is a per-install state machine, replayed in time order.**
   You cannot classify an `ACTIVATED` event in isolation — whether it is
   `subscribed`, `resubscribed`, `upgraded`, or `downgraded` depends on the
   install's *current state and current MRR*, which you accumulate by folding
   the events in order.
3. **Normalize money to a monthly figure before you compare it.** An annual
   charge of 1200 and a monthly charge of 100 are the *same* MRR. Upgrade vs.
   downgrade is decided on the normalized monthly amount, not the raw charge.
4. **A lone cancel is guilty until proven innocent.** A
   `SUBSCRIPTION_CHARGE_CANCELED` is *usually* one half of a plan change, not a
   churn. Suppress it unless you can show no activation (or chained plan-change
   event) accompanies it. This is the single highest-value rule in the spec.
5. **Idempotent on a stable key.** Every clean event carries a deterministic
   `platformEventId` derived from its raw source. Re-deriving the whole history
   must converge to the same set of clean events — no duplicates, no drift.

If you preserve these five invariants, everything else is bookkeeping.

---

## 1. Data model

Two event tables (raw + clean), plus a child table for uninstall detail. Field
names are illustrative — map them to your schema.

### 1.1 `raw_app_events` (the verbatim feed)
One row per fact pulled from the Partner API. **Immutable.**
| Field | Notes |
|---|---|
| `id` | Your PK. Also used as the clean event's `platformEventId` (§8.1). |
| `type` | The **raw** Shopify type string, e.g. `SUBSCRIPTION_CHARGE_ACTIVATED`. Stored verbatim. |
| `occurredAt` | Shopify's event timestamp. The sort key for the state machine. |
| `appInstallationId` | The (app, shop) install this event belongs to. The state-machine partition key. |
| `chargeId` / `chargeGid` | FK to the charge this event references (subscription/usage/one-time/credit), when present. |
| `reason`, `description` | Free-text, present on uninstall events (§7). |
| ingest metadata | cursor, fetchedAt, etc. |

**Dedup key (MUST be unique):** `(appInstallationId, type, occurredAt, chargeGid)`,
with a sentinel like `"NO_CHARGE"` when `chargeGid` is null. The feed overlaps
itself on every poll (§2.2); this constraint is what makes re-polling free.

### 1.2 `charges` (referenced by events)
You need enough of the Shopify charge to normalize money. Minimum:
| Field | Notes |
|---|---|
| `gid` | Shopify charge GID. |
| `amount`, `currencyCode` | The raw charge amount. |
| `subscriptionId` | The subscription this charge belongs to (for plan-change correlation, §4). |
| `plan.interval` | `EVERY_30_DAYS` / `ANNUAL` / … — drives normalization (§5.1). |
| `plan.amount` | The plan's nominal price (used for flex/usage plans where the charge is $0). |
| `plan.flexBilling` | If true, normalize from `plan.amount`, not the charge (§5.1). |

### 1.3 `app_events` (the clean lifecycle events — what analytics reads)
One row per **derived** lifecycle event.
| Field | Notes |
|---|---|
| `id` | |
| `type` | The **clean** lifecycle type (§1.5). |
| `platformEventId` | **UNIQUE.** Deterministic, derived from the raw event (§8.1). The idempotency key. |
| `occurredAt` | Carried from the raw event (with the expiry back-date adjustment, §3.4). |
| `netChange` | Signed monthly MRR delta this event represents (§5). |
| `planAmount`, `planCurrencyCode`, `planInterval` | The plan in effect *after* this event. |
| `planChanges` | JSON: structured before/after plan(s) for upgrades/downgrades (optional, richer than `netChange`). |
| `appInstallationId`, `organizationId` | Tenant + install. |
| `subscriptionId` | The subscription this event concerns. |
| `previousSubscriptionId` | The prior subscription, when the install moved between subscriptions (§3.5). Drives plan-change correlation and trial-conversion logic. |
| `shopifyAppEventId` | FK back to the raw row it was derived from. |
| `deletedAt` | Soft-delete marker (the cancel trap deletes suppressed events; §4). |

**Indexes:** `(organizationId, occurredAt)`, `(organizationId, type)`,
`(platformEventId)` unique, `(appInstallationId)`.

### 1.4 `uninstall_app_events` (uninstall detail — child of an `uninstalled` event)
| Field | Notes |
|---|---|
| `appEventId` | **UNIQUE.** The `uninstalled` clean event this enriches. |
| `reason`, `description` | Raw, any language, from the uninstall feedback. |
| `reasonEnglish`, `descriptionEnglish` | Translated to English before classification (§7). |
| `reasonCode` | The **primary** normalized code (first of `reasonCodes`). |
| `reasonCodes` | Array of canonical codes (multi-category uninstalls allowed). |
| `secondsSinceFirstInstall`, `secondsSinceLastInstall`, `secondsToUninstall` | Timing metadata for "time-to-churn" analytics. |

### 1.5 Clean lifecycle event types (the target enum)
Two families. Map your own labels; keep them stable once analytics depends on them.

**Account lifecycle:**
`installed`, `reinstalled`, `uninstalled`, `deactivated`, `reactivated`,
`subscription_approaching_capped_amount`, `subscription_capped_amount_updated`.

**Subscription lifecycle:**
`subscribed`, `resubscribed`, `upgraded`, `downgraded`, `unsubscribed`,
`subscription_frozen`, `subscription_unfrozen`, `trial_expired`,
`trial_converted`, `charge_abandoned`, `one_time_charge_activated`,
`credit_applied`.

> **`subscribed` vs `resubscribed`:** a first paid activation is `subscribed`; a
> paid activation after the install has *ever* been subscribed before is
> `resubscribed` (a win-back). The state machine decides this from history, not
> from the raw event.

---

## 2. The Partner API event feed

### 2.1 Raw event types Shopify emits
The feed you poll (`appEvents` on the Partner API) yields these (non-exhaustive):

| Raw type | Meaning |
|---|---|
| `RELATIONSHIP_INSTALLED` | App installed on a shop. |
| `RELATIONSHIP_UNINSTALLED` | App removed. |
| `RELATIONSHIP_REACTIVATED` / `RELATIONSHIP_DEACTIVATED` | Shop relationship toggled (e.g. shop frozen/unfrozen by Shopify). |
| `SUBSCRIPTION_CHARGE_ACTIVATED` | A recurring charge became active (signup **or** plan change). |
| `SUBSCRIPTION_CHARGE_CANCELED` | A recurring charge was canceled (churn **or** plan change — see §4). |
| `SUBSCRIPTION_CHARGE_FROZEN` / `SUBSCRIPTION_CHARGE_UNFROZEN` | Billing paused/resumed (e.g. shop payment issue). |
| `SUBSCRIPTION_CHARGE_DECLINED` / `SUBSCRIPTION_CHARGE_EXPIRED` | Charge never approved. |
| `SUBSCRIPTION_APPROACHING_CAPPED_AMOUNT` / `SUBSCRIPTION_CAPPED_AMOUNT_UPDATED` | Usage-cap signals. |
| `ONE_TIME_CHARGE_ACTIVATED` / `ONE_TIME_CHARGE_DECLINED` / `ONE_TIME_CHARGE_EXPIRED` | One-time purchases. |
| `CREDIT_APPLIED` / `CREDIT_FAILED` / `CREDIT_PENDING` | App credits (refund-like). |
| `USAGE_CHARGE_APPLIED` | A usage record posted. |

You do not have to map all of them; map the ones your analytics needs and log
the rest as "unknown" rather than crashing (§9).

### 2.2 Polling strategy
- Query the feed **cursor-paginated** (~50/page). Page until the cursor is
  exhausted.
- Track a per-app high-water mark and re-query with an **overlapping since
  buffer** (e.g. `occurredAtMin = lastSync − 60 minutes`). The feed can deliver
  events slightly out of order or late; the overlap plus the §1.1 dedup key
  makes re-ingestion idempotent.
- **MUST** persist every raw event before deriving anything. Ingestion and
  derivation are separate passes.
- Per-event errors **MUST** be caught and reported without aborting the page.

There is **no clean "lifecycle" webhook** and no usage/subscription-update
webhook you can trust to drive this. The poll is the engine. (A real-time
`app/uninstalled` webhook can *enrich* uninstall reason — §7 — but the
authoritative uninstall fact still comes through the feed.)

---

## 3. The state machine (raw → clean)

Run **per install**, over that install's raw events **sorted by `occurredAt`
ascending**. Fold the stream, carrying:

- `currentState` — the last clean state emitted (`none` initially).
- `currentAmount` — the install's current **normalized monthly** MRR.
- `mostRecentSubscription` — the subscription the install is currently on (for
  `previousSubscriptionId`).

For each raw event, compute `normalizedChargeAmount` (§5.1), classify, emit a
clean event, and update the carried state.

### 3.1 Account events (state-independent)
```
RELATIONSHIP_INSTALLED   → already emitted an `installed`? → reinstalled : installed
RELATIONSHIP_UNINSTALLED → uninstalled        (set currentState = uninstalled)
RELATIONSHIP_REACTIVATED → reactivated        (account-level; does not move MRR)
RELATIONSHIP_DEACTIVATED → deactivated        (account-level; does not move MRR)
SUBSCRIPTION_APPROACHING_CAPPED_AMOUNT → subscription_approaching_capped_amount (+usage metadata)
SUBSCRIPTION_CAPPED_AMOUNT_UPDATED     → subscription_capped_amount_updated
ONE_TIME_CHARGE_ACTIVATED              → one_time_charge_activated
```

### 3.2 Activation — the branch that needs state
On `SUBSCRIPTION_CHARGE_ACTIVATED`:
```
if currentState ∈ { installed, reinstalled, unsubscribed, none }:
    # no active paid subscription → this is an entry
    if the install has EVER emitted `subscribed` before:
        emit resubscribed;  netChange = +normalizedChargeAmount
    else:
        emit subscribed;    netChange = +normalizedChargeAmount
    currentAmount = normalizedChargeAmount

elif currentState ∈ { subscribed, resubscribed, upgraded, downgraded, subscription_unfrozen }:
    # already paying → this activation is a plan change
    if normalizedChargeAmount >= currentAmount:
        emit upgraded
    else:
        emit downgraded
    netChange     = normalizedChargeAmount − currentAmount     # signed
    currentAmount = normalizedChargeAmount
```
Note the boundary: **equal-or-greater is `upgraded`** (an equal-price swap is
labeled `upgraded` with `netChange = 0`). Pick a side and be consistent.

### 3.3 Cancel — defer to the cancel trap
On `SUBSCRIPTION_CHARGE_CANCELED`, run §4. If it is suppressed, emit nothing
(and clean up any previously-emitted cancel). Otherwise:
```
emit unsubscribed;  netChange = −normalizedChargeAmount;  currentAmount = 0
```

### 3.4 Freeze / decline / expire / credit
```
SUBSCRIPTION_CHARGE_FROZEN    → subscription_frozen;   netChange = −normalizedChargeAmount; currentAmount = 0
SUBSCRIPTION_CHARGE_UNFROZEN  → subscription_unfrozen; netChange = +normalizedChargeAmount; currentAmount = normalized
SUBSCRIPTION_CHARGE_DECLINED  → charge_abandoned
ONE_TIME_CHARGE_DECLINED      → charge_abandoned
SUBSCRIPTION_CHARGE_EXPIRED   → charge_abandoned, AND back-date occurredAt (see below)
ONE_TIME_CHARGE_EXPIRED       → charge_abandoned, AND back-date occurredAt (see below)
CREDIT_APPLIED                → credit_applied;        netChange = −charge.amount
```
**Expiry back-dating:** an `*_EXPIRED` event fires when Shopify gives up on a
charge the merchant never approved — but it arrives ~days after the *real*
abandonment. Back-date `occurredAt` (Mantle subtracts 60 hours) so the abandoned
charge lands near the install, not days later — but clamp it to never precede the
install's first interaction. Without this, funnel/abandonment analytics smear.

### 3.5 `previousSubscriptionId`
On any **subscription** event (`subscribed`/`resubscribed`/`upgraded`/
`downgraded`/`unsubscribed`/`frozen`/`unfrozen`), set
`previousSubscriptionId = mostRecentSubscription.id` **iff** it differs from this
event's subscription. Then update `mostRecentSubscription`. This single field
powers plan-change correlation (§4) and trial-conversion-via-plan-change (§6).

### 3.6 Unknown events
Any raw type you don't map: **log and skip** (do not throw, do not emit). A
single unrecognized type must never break the install's derivation.

---

## 4. The cancel trap (plan-change suppression) — the money rule

A `SUBSCRIPTION_CHARGE_CANCELED` is **usually not churn.** When a merchant
changes plans, Shopify cancels the old recurring charge and activates a new one,
milliseconds apart. If you record the cancel as `unsubscribed`, you double-count:
once as churn (−old MRR) and once as the plan change. Worse, your churn numbers
are now fiction.

**Suppress the cancel if ANY of these hold:**

### 4.a Activation just after (the common plan change)
An `SUBSCRIPTION_CHARGE_ACTIVATED` for this install exists with
`occurredAt ∈ [cancel.occurredAt, cancel.occurredAt + 60s)`. The new plan's
activation immediately follows the old plan's cancel → it's a plan change.

### 4.b Activation just before, same charge (re-emit / ordering)
An `SUBSCRIPTION_CHARGE_ACTIVATED` exists with
`occurredAt ∈ (cancel.occurredAt − 5s, cancel.occurredAt]` **and the same
`chargeGid`**. Guards against feed ordering quirks where the activate lands just
ahead of the cancel for the same charge.

### 4.c Chained plan-change event (paid → free / flex) — the subtle one
**A move to a free or flex ($0-recurring) plan produces no
`SUBSCRIPTION_CHARGE_ACTIVATED` at all** — the new subscription bypasses
Shopify's billing entirely. So §4.a/§4.b can't see it. Instead, the code that
performs the plan change writes the `upgraded`/`downgraded` clean event
**directly** into your store. Detect *that*:

```
only when NO activation-before and NO activation-after were found:
  look for an existing upgraded|downgraded clean event where
    previousSubscriptionId == cancel.charge.subscriptionId
    AND |occurredAt − cancel.occurredAt| <= 60s
```
If found, the cancel is the tail of a paid→free (or paid→flex) plan change.
Suppress it.

### Suppression action
```
if (activationAfter OR chainedPlanChange OR (activationBefore && sameCharge)):
    delete any clean event already derived from this raw cancel (soft-delete)
    if chainedPlanChange:
        # No activation will follow to advance the machine. Adopt the chained
        # event's post-change state so the NEXT real activation computes its
        # delta against the new (free/flex) baseline, not the canceled paid amount.
        currentState  = chainedPlanChange.type
        currentAmount = chainedPlanChange.planAmount || 0
    continue          # emit nothing
else:
    emit unsubscribed (§3.3)
```

**Why the state fix in §4.c matters:** without it, after a paid→free downgrade
the machine still thinks `currentAmount = oldPaidAmount`. The next time the
merchant upgrades to a paid plan, `netChange` would be computed against the wrong
baseline and your MRR drifts. Adopt the chained event's amount.

**Backfill cleanup.** Because suppression depends on *both* halves being present
and ordering can race, also run an **offline reconciler** that finds
`unsubscribed` events with a matching `upgraded`/`downgraded`
(`previousSubscriptionId == unsubscribed.subscriptionId`, within ±60s) and
soft-deletes the stray `unsubscribed` from every store (DB + search index). The
live guard catches most; the reconciler catches the races.

---

## 5. netChange — monthly MRR deltas

`netChange` is the signed monthly MRR impact of an event. **MRR over any window
is the sum of `netChange` for the install's non-ignored events** (§8.3). Get this
field right and MRR is free; get it wrong and every dashboard lies.

### 5.1 Normalize to monthly first
```
function normalizedMonthly(charge):
    if charge.plan.flexBilling:           # $0 recurring; real price is the plan nominal
        return charge.plan.amount
    if charge.plan.interval == EVERY_30_DAYS: return charge.amount
    if charge.plan.interval == ANNUAL:        return charge.amount / 12
    # extend for QUARTERLY etc.; default 0 when no charge/amount
```
Annual plans are the classic trap: a 1200/yr plan must contribute **100** to MRR,
not 1200.

### 5.2 The delta per event type
| Event | `netChange` |
|---|---|
| `subscribed` / `resubscribed` | `+normalizedMonthly` |
| `upgraded` / `downgraded` | `normalizedMonthly(new) − currentAmount` (signed; the state machine already holds `currentAmount`) |
| `unsubscribed` / `subscription_frozen` | `−normalizedMonthly` |
| `subscription_unfrozen` | `+normalizedMonthly` |
| `credit_applied` | `−charge.amount` |
| account events (install/uninstall/…) | none / null |

### 5.3 Discounts
If a plan can carry a discount, MRR should reflect the **discounted** price the
merchant actually pays, not list price. When computing the amount for `netChange`,
prefer `priceAfterDiscount` when an active discount covers the period — *unless*
the discount is delivered as app credits rather than a price reduction (credits
move money on a different rail and would double-count). Keep both list and
effective amounts if you can; report on the effective one.

### 5.4 Direction label vs. sign
Where you can, label `upgraded`/`downgraded` by **list price comparison**, kept
consistent with the sign of `netChange`. An equal-price swap is fine to label
`upgraded` with `netChange = 0` — just be deterministic.

---

## 6. Trial derivation

Shopify can host a trial, but you should **own trial state locally** (the same
discipline as flex billing) and *derive* trial lifecycle events rather than
trusting a webhook.

### 6.1 State, from dates
Carry `trialExpiresAt` (and optionally `trialStartsAt`) on the subscription (or
install, for app-level trials). Derive status:
```
function trialStatus(sub):
    if not sub.trialExpiresAt: return none
    if sub.canceledAt or sub.frozenAt:
        cancelDate = sub.canceledAt ?? sub.frozenAt
        return cancelDate < sub.trialExpiresAt ? canceled_during : canceled_after
    return sub.trialExpiresAt < now ? trial_converted : trial_active
```
- `trial_active` — still inside the window.
- `trial_converted` — window passed and the subscription is still alive (it
  rolled into paid).
- `canceled_during` / `canceled_after` — churned before/after the window.

### 6.2 The `trial_expired` event (a derived, time-driven event)
Nothing in the feed fires when a trial simply ends. Run a **nightly sweep** that
selects subscriptions/installs where:
```
trialExpiresAt IS NOT NULL
AND trialExpiresAt < now
AND trialExpiredAt IS NULL          # not already processed
AND still active (not canceled/frozen before the window closed)
```
For each, emit a `trial_expired` clean event at `occurredAt = trialExpiresAt`,
then stamp `trialExpiredAt` and recompute `trialStatus`. Use a deterministic
`platformEventId` (e.g. `trial_expired/{installId}/{date}`) so the sweep is
idempotent.

### 6.3 Trial types
Distinguish where the trial is enforced: **platform** (Shopify-hosted),
**local** (you enforce it; Shopify sees no trial), or **hybrid** (local first,
then platform). It changes which date field is authoritative in §6.1. **MUST**
keep local/hybrid trials off Shopify's `trialDays` and drive the first charge
from your own `trialExpiresAt`.

### 6.4 Conversion via plan change
A trialing subscription that **upgrades/downgrades** to a *different*
subscription has converted, not churned — even though the trial sub gets
canceled. The `previousSubscriptionId` chain (§3.5) is how you tell: if a churn's
subscription is the `previousSubscriptionId` of a nearby `upgraded`/`downgraded`,
treat it as a conversion, not a trial-abandonment. (This is the same correlation
idea as §4, applied to trial accounting.)

---

## 7. Uninstall reason normalization

When the merchant uninstalls, Shopify may carry a free-text reason and
description (often localized). Raw reasons are unusable for analytics — normalize
them to a small **canonical taxonomy** with a two-stage pipeline.

### 7.1 The taxonomy
Define ~15–25 stable codes covering the real reasons, e.g.: `not_using`,
`hard_to_setup`, `poor_support`, `limited_features`, `high_cost`,
`security_or_privacy_issues`, `not_compatible_or_not_working`,
`doesnt_satisfy_needs`, `found_alternative`, `app_performance_issues`,
`not_needed_anymore`, `unexpected_charges`, `prefer_native_features`,
`testing_multiple_apps`, `store_closing_or_pausing`, `deactivated`,
`scheduled_cancellation`, `unknown_other`.

**MUST** flag the **store-closure** subset (`store_closing_or_pausing`,
`deactivated`, `scheduled_cancellation`) separately: these are *not* product
churn (the merchant's store closed; it's not a verdict on your app). Analytics
must be able to exclude them from "why are we losing customers" without losing
them from raw counts.

### 7.2 The two-stage pipeline (build this as a job off the `uninstalled` event)
```
on uninstalled event:
    reason, description = raw feedback
    # 0. Special-cases first
    if event was a `deactivated`:             reasonCodes = [deactivated]
    elif reason == "scheduled_cancellation":  reasonCodes = [scheduled_cancellation]
    else:
        # 1. Translate to English (reasons are localized)
        reasonEnglish = translate(reason); descriptionEnglish = translate(description)
        # 2. Deterministic preset map: exact Shopify reason label → code
        codes = mapPresetLabels(reasonEnglish)        # array, or null if ANY token unknown
        if codes is null and (reasonEnglish or descriptionEnglish):
            # 3. LLM fallback for free text the preset map can't cover
            codes = classifyWithLLM(reasonEnglish, descriptionEnglish)   # may return several
        reasonCodes = codes ?? [unknown_other]
    persist reasonCodes (+ primary reasonCode = reasonCodes[0]), translations, timing metadata
```
- **Preset map first, LLM second.** Shopify ships a fixed menu of reason labels;
  map those deterministically (handle the comma-separated multi-select, and any
  re-wordings Shopify rolls out). Only fall through to the model for genuine free
  text. **Fail the preset map to null if *any* token is unrecognized** rather than
  guessing — let the LLM handle the whole string.
- **Cache** LLM results by a hash of `(reasonEnglish, descriptionEnglish)` so
  identical feedback isn't re-classified.
- Allow **multiple codes** per uninstall (`reasonCodes[]`); keep the first as the
  primary for single-value rollups.
- Compute timing (`secondsSinceFirstInstall`, etc.) for time-to-churn analytics.

---

## 8. The pipeline (storage, indexing, idempotency)

### 8.1 Deterministic clean-event identity
Every clean event's `platformEventId` is a pure function of its source — e.g.
the raw event's `id`, or `app_api/{type}/{sourceId}` for derived ones (trial
sweep, chained plan change). **MUST** be unique. Re-deriving an install's history
upserts on this key, so the operation is idempotent and convergent.

### 8.2 Re-derivation is normal, not exceptional
Treat "replay the whole install from the raw feed" as a routine operation (new
event arrives, a backfill runs, the classifier changes). Because the state
machine is a deterministic fold and clean events upsert on `platformEventId`,
replays are safe. When a replay would create a subscription that a prior run
already created, **reuse the existing one** (track a `linkedSubscriptionId`)
rather than duplicating.

### 8.3 The `ignored` flag (analytics correctness without deleting data)
Don't delete events that shouldn't count toward MRR — **mark them `ignored`** at
index time and let queries filter on it. The canonical case: cross-subscription
churn noise. When an install has overlapping subscriptions, a churn on a
*different* subscription within a short window (e.g. 5 minutes for Shopify) is
plumbing, not real churn — ignore it. But **do not ignore** a trial sub's churn
that actually converted via plan change (§6.4). Keep this logic at the
read/index boundary so the raw and clean layers stay honest.

### 8.4 Fan-out to analytics stores (optional, scale-dependent)
The clean `app_events` table is the source of truth. For dashboards at scale,
fan out (e.g. via a queue) into:
- a **search index** (Elasticsearch-style) for filtered timelines and
  aggregations — enrich each doc here with `ignored`, trial status, conversion/
  churn dates, discount detail, and the uninstall reason codes;
- a **columnar store** (ClickHouse-style) for high-volume rollups — dedupe with a
  `ReplacingMergeTree(version)`-style engine keyed on the event identity, so
  replays converge.

**MUST** treat these as derived projections: an indexing failure may never
corrupt or block the authoritative clean-event write. Until you're at real scale,
a plain SQL query over `app_events` is enough — skip the columnar tier.

---

## 9. MUST / MUST NOT rules (the gotchas)

Each encodes a real bug or hard lesson. Treat as acceptance criteria.

- **MUST keep raw and clean events in separate layers.** Derive, never classify
  on ingest. You will re-derive.
- **MUST sort by `occurredAt` and run the state machine per install.** Classifying
  an activation without the install's current state/amount is wrong by
  construction.
- **MUST normalize money to monthly before comparing** (annual ÷ 12; flex/usage
  from plan nominal). Upgrade/downgrade and MRR both depend on it.
- **MUST suppress lone cancels that are plan changes** — activation-after (60s),
  activation-before-same-charge (5s), or a chained paid→free/flex
  `upgraded`/`downgraded` (±60s). A naive pipeline double-counts churn.
- **MUST handle paid→free with no activation webhook.** It emits no
  `SUBSCRIPTION_CHARGE_ACTIVATED`; detect the chained plan-change event and adopt
  its post-change amount as the new baseline.
- **MUST run an offline reconciler** for stray `unsubscribed` events that race
  past the live suppression guard.
- **MUST make derivation idempotent** on a deterministic `platformEventId`, and
  dedupe the raw feed on `(install, type, occurredAt, chargeGid)`.
- **MUST back-date `*_EXPIRED` abandonment events** (and clamp to first
  interaction) or funnel analytics smear days late.
- **MUST own trial state locally and emit `trial_expired` from a nightly sweep**
  — nothing in the feed fires at trial end. Keep local trials off Shopify's
  `trialDays`.
- **MUST treat trial→plan-change as conversion, not churn**, via the
  `previousSubscriptionId` chain.
- **MUST normalize uninstall reasons** (preset map → LLM fallback, translate
  first, cache, allow multi-code) and **flag store-closure reasons separately**
  from product churn.
- **MUST mark non-counting events `ignored` rather than deleting them**, and keep
  the ignore logic at the read/index boundary.
- **MUST log-and-skip unknown raw types** — never throw on an unrecognized event.
- **MUST treat search/columnar stores as derived** — their failures never block
  the authoritative write.
- **MUST poll with an overlapping since-buffer** and rely on the dedup key; the
  feed is late and out of order.

---

## 10. Acceptance scenarios

Implement and verify each:

1. **First subscribe.** Install → activation → exactly one `subscribed`,
   `netChange = +monthly`, MRR rises by the monthly amount.
2. **Annual normalization.** Activation on a 1200/yr plan emits `subscribed` with
   `netChange = 100`, not 1200.
3. **Mid-cycle upgrade.** Cancel+activate to a pricier plan → one `upgraded`
   (`netChange = new − old`), **zero** `unsubscribed`. The cancel is suppressed.
4. **Mid-cycle downgrade (paid→paid).** Same as #3 but `netChange < 0`, labeled
   `downgraded`.
5. **Paid → free.** No activation webhook; one `downgraded` (chained), the cancel
   suppressed, and the next paid upgrade computes its delta against the free
   baseline (not the old paid amount).
6. **Real churn.** A cancel with no nearby activation or chained event → one
   `unsubscribed`, `netChange = −monthly`, MRR drops.
7. **Win-back.** Activation after a prior `subscribed`+`unsubscribed` → emits
   `resubscribed`, not `subscribed`.
8. **Freeze/unfreeze.** Frozen drops MRR to 0; unfrozen restores it; no spurious
   churn recorded.
9. **Charge expired.** `*_EXPIRED` → `charge_abandoned`, back-dated, never before
   first interaction.
10. **Trial end (converted).** Nightly sweep emits one idempotent `trial_expired`
    at `trialExpiresAt`; re-running the sweep emits nothing new.
11. **Trial converted via plan change.** Trial sub upgrades to a paid plan →
    counted as conversion; the trial sub's churn is not counted.
12. **Uninstall (preset).** A Shopify menu reason maps deterministically to a
    code, no LLM call.
13. **Uninstall (free text).** A localized free-text reason is translated, then
    LLM-classified into one or more codes; identical feedback hits the cache.
14. **Store closure.** A store-closing uninstall is coded into the store-closure
    subset and excluded from product-churn reporting but present in raw counts.
15. **Replay.** Re-deriving an install's entire history produces the identical
    clean-event set (idempotent on `platformEventId`); no duplicate subscriptions.
16. **Out-of-order feed.** Re-polling with the since-buffer overlap creates no
    duplicate raw or clean events.
