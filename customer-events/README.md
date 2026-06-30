# Customer Events from Shopify — a build guide

*Part of [mantle-brain](../README.md): an open-source library to help Shopify
Partners recreate Mantle's tools as Mantle winds down.*

This guide explains **how Mantle turned Shopify's raw Partner API app-events feed
into clean customer lifecycle events** — the `subscribed` / `upgraded` /
`downgraded` / `unsubscribed` / `uninstalled` stream that every MRR and churn
chart is built on — and how to rebuild that pipeline in your own app.

If you'd rather hand the job to a coding agent, this folder ships three
companions:

- **[`implementation-spec.md`](./implementation-spec.md)** — a dense, imperative
  build spec (data model, the state machine, the cancel trap, normalization math,
  acceptance tests). Paste it into Claude/Cursor and say "implement this."
- **[`pipeline-prompt.md`](./pipeline-prompt.md)** — a single self-contained
  prompt that asks an agent to build the whole pipeline end to end. The fastest
  way to get a working first cut.
- **[`skill/shopify-customer-events/`](./skill/shopify-customer-events/)** — a
  Claude Code / Agent skill. Copy it into your app's `.claude/skills/` and the
  agent walks the build for you. See its
  [README](./skill/shopify-customer-events/README.md).

---

## The problem this solves

As a Shopify Partner you get one **app-events feed** per app from the Partner
API. It is a firehose of low-level facts:

```
RELATIONSHIP_INSTALLED
SUBSCRIPTION_CHARGE_ACTIVATED
SUBSCRIPTION_CHARGE_CANCELED
SUBSCRIPTION_CHARGE_FROZEN
RELATIONSHIP_UNINSTALLED
...
```

That feed is **not** a clean lifecycle. The events you actually want for
analytics — "this customer subscribed," "this one upgraded," "this one churned" —
have to be *compiled* out of the raw stream. And the raw stream is full of traps:

- **A plan change looks like a churn.** When a merchant switches plans, Shopify
  *cancels* the old charge and *activates* a new one, milliseconds apart. Record
  the cancel naively and you've invented a churn that never happened.
- **A downgrade to free emits no activation at all.** Move a merchant to a free
  or $0-recurring plan and Shopify sends a cancel with *no* matching activate —
  because the new subscription never touches Shopify billing.
- **Money isn't comparable.** An annual charge of 1200 and a monthly charge of
  100 are the same MRR. Compare the raw numbers and every upgrade/downgrade
  decision is wrong.
- **Trials end silently.** Nothing in the feed fires when a trial expires.
- **Uninstall reasons are free text, localized, and messy.**

**The pipeline is the fix.** It turns the firehose into a clean, deduplicated,
analytics-ready event stream.

---

## The core idea

> Keep the raw feed verbatim. Derive clean lifecycle events from it with a
> **per-install state machine** you can replay at will.

Two layers, never one:

```
   Shopify Partner API
   app-events feed (poll)
           │
           ▼
   ┌─────────────────────────┐
   │  raw_app_events         │   verbatim, immutable, deduped
   │  (RELATIONSHIP_*,       │   on (install, type, occurredAt, charge)
   │   SUBSCRIPTION_CHARGE_*) │
   └───────────┬─────────────┘
               │  derive (replayable, idempotent)
               ▼
   ┌─────────────────────────┐
   │  app_events             │   subscribed / upgraded / downgraded /
   │  (clean lifecycle)      │   unsubscribed / uninstalled / trial_expired …
   │  + netChange (MRR delta)│   each keyed by a deterministic platformEventId
   └───────────┬─────────────┘
               │  fan out (optional, at scale)
               ▼
     search index  +  columnar store   → dashboards
```

The derivation is **never** done on the wire. You will re-run it many times — when
a late event arrives, when you fix a classifier, when you backfill. The raw layer
is the immutable truth; the clean layer is a deterministic function of it.

---

## Mental model

| Concept | What it means |
|---|---|
| **Raw event** | A verbatim Partner-API fact (`SUBSCRIPTION_CHARGE_ACTIVATED`). Stored, never interpreted in place. |
| **Clean event** | A derived lifecycle event (`upgraded`) with a signed MRR delta. What analytics reads. |
| **State machine** | A per-install fold over raw events in time order. It carries *current state* and *current MRR* so it can tell a first `subscribed` from a `resubscribed` from an `upgraded`. |
| **Normalized monthly amount** | Every charge reduced to its monthly MRR (annual ÷ 12, flex from plan nominal) *before* any comparison. |
| **netChange** | The signed monthly MRR delta a clean event represents. **MRR = sum of netChange** over an install's non-ignored events. |
| **The cancel trap** | The rule that a lone cancel is *probably a plan change*, and must be suppressed unless proven to be real churn. |
| **`platformEventId`** | A deterministic id for each clean event, derived from its raw source. Makes re-derivation idempotent. |
| **`ignored` flag** | Marks events that shouldn't count toward MRR (cross-subscription noise) without deleting them. |

The single most important structural fact: **whether a
`SUBSCRIPTION_CHARGE_ACTIVATED` is a `subscribed`, `resubscribed`, `upgraded`, or
`downgraded` cannot be known from the event itself** — only from the install's
accumulated state. That's why it's a state machine, not a lookup table.

---

## How the pipeline works — end to end

### 1. Poll the feed
Query the Partner API `appEvents` feed, cursor-paginated (~50/page). Re-poll with
an **overlapping since-buffer** (e.g. "since 60 minutes before my last sync")
because the feed runs late and out of order. Persist every raw event first,
deduped on `(install, type, occurredAt, chargeGid)`. There is no trustworthy
real-time "lifecycle" webhook — the poll is the engine.

### 2. Run the state machine per install
Sort an install's raw events by time and fold them, carrying `currentState` and
`currentAmount` (normalized monthly MRR):

- `RELATIONSHIP_INSTALLED` → `installed` (or `reinstalled` if it ever installed
  before).
- `SUBSCRIPTION_CHARGE_ACTIVATED` from a non-paying state → `subscribed` (or
  `resubscribed` if it ever subscribed before).
- `SUBSCRIPTION_CHARGE_ACTIVATED` while already paying → compare the new
  normalized amount to `currentAmount`: `≥` is `upgraded`, `<` is `downgraded`.
- `SUBSCRIPTION_CHARGE_FROZEN`/`UNFROZEN` → drop MRR to 0 / restore it.
- `RELATIONSHIP_UNINSTALLED` → `uninstalled`.
- Unknown types → log and skip, never crash.

### 3. Spring the cancel trap
When you hit a `SUBSCRIPTION_CHARGE_CANCELED`, **don't assume churn.** Suppress it
if:
- an activation follows within ~60s (the new plan), **or**
- an activation just preceded it for the same charge (ordering races), **or**
- a chained `upgraded`/`downgraded` event already exists pointing back at this
  subscription (the paid→free case, where no activation webhook ever comes).

Only a cancel with *none* of these becomes a real `unsubscribed`. For the
paid→free case, also advance the machine's `currentAmount` to the new (free)
baseline — otherwise the next upgrade computes its MRR delta against the stale
paid amount.

Because the two halves can race past the live guard, also run an **offline
reconciler** that deletes stray `unsubscribed` events that have a matching
plan-change event nearby.

### 4. Compute netChange
For each clean event, attach the signed monthly MRR delta:
`+monthly` on subscribe, `new − old` on a plan change, `−monthly` on churn/freeze.
Normalize annual to monthly first, and reflect discounts (use the price the
merchant actually pays, not list). Sum `netChange` over non-ignored events and
you have MRR, for free.

### 5. Derive trials
Own trial state locally (don't lean on Shopify's hosted trial). Derive
`trial_active` / `trial_converted` / `canceled_during` / `canceled_after` from
`trialExpiresAt` vs. now and the subscription's cancel/freeze dates. Nothing in
the feed fires at trial end, so run a **nightly sweep** that emits an idempotent
`trial_expired` event for trials whose window just closed. And remember: a trial
that *upgrades* to a paid plan converted — it didn't churn.

### 6. Normalize uninstall reasons
When the `uninstalled` event lands, enrich it: translate the (often localized)
reason to English, map it through a **deterministic preset table** of Shopify's
known reason labels, and fall back to an **LLM classifier** only for genuine free
text. Store one or more canonical codes. Crucially, flag **store-closure** reasons
(`store_closing_or_pausing`, `deactivated`, `scheduled_cancellation`) separately —
a closed store isn't a verdict on your app, and shouldn't pollute product-churn
numbers.

### 7. Fan out (only when you need it)
The clean `app_events` table is the source of truth. At scale, project it into a
search index (filtered timelines, aggregations) and a columnar store
(high-volume rollups), deduped on the event identity so replays converge. Treat
both as disposable projections — their failures must never corrupt the clean
write. Until you're at real scale, a plain query over `app_events` is plenty.

---

## The cancel trap, in detail

This is the rule that separates a correct pipeline from a wrong one, so it's
worth its own diagram.

```
        SUBSCRIPTION_CHARGE_CANCELED arrives
                      │
        ┌─────────────┴──────────────────────────────────┐
        │ Is there an activation within 60s AFTER?        │── yes ─┐
        │ Or an activation just BEFORE, same charge?      │── yes ─┤
        │ Or a chained upgraded/downgraded event          │        │
        │   pointing back at this subscription (±60s)?    │── yes ─┤
        └─────────────┬──────────────────────────────────┘        │
                      │ no                                          ▼
                      ▼                                   SUPPRESS the cancel
              emit `unsubscribed`                  (delete any event already made;
              netChange = −monthly                  for paid→free, advance the
              (real churn)                           machine to the new baseline)
```

The third branch — the **chained plan-change event** — is the subtle one. A
paid→free or paid→flex change produces *no* `SUBSCRIPTION_CHARGE_ACTIVATED`,
because the new subscription bypasses Shopify billing. The only signal that the
cancel is a plan change is the `upgraded`/`downgraded` event your own plan-change
code already wrote, linked by `previousSubscriptionId`. Miss this and every move
to a free plan looks like a churn.

---

## Gotchas — the lessons that cost real accuracy

Each of these is a bug someone already hit.

- **A lone cancel is usually a plan change, not a churn.** Suppress it unless you
  can prove otherwise. This is the highest-value rule in the whole pipeline.
- **Paid → free emits no activation.** Detect the chained `downgraded` event by
  `previousSubscriptionId`, and re-baseline the state machine's MRR to the new
  (free) amount — or the next upgrade's delta is computed wrong.
- **Normalize money to monthly before comparing.** A 1200/yr plan contributes
  100 to MRR. Compare raw charge amounts and annual plans break every chart.
- **`*_EXPIRED` events arrive days late.** Back-date abandonment events (Mantle
  uses −60 hours) toward the install, clamped to never precede first interaction,
  or funnel analytics smear.
- **Trials end silently.** Run a nightly sweep to emit `trial_expired`; the feed
  won't do it for you. Keep local trials off Shopify's `trialDays`.
- **A trial that plan-changes converted, it didn't churn.** Use the
  `previousSubscriptionId` chain to tell the difference.
- **Uninstall reasons are localized free text.** Translate first, map known
  labels deterministically, LLM-classify the rest, cache by content hash.
- **Store closures aren't product churn.** Flag the store-closure reason subset so
  it can be excluded from "why are we losing customers."
- **Don't delete non-counting events — mark them `ignored`.** Cross-subscription
  churn noise within a short window is plumbing; flag it at index time, keep the
  data.
- **Make derivation idempotent.** A deterministic `platformEventId` plus a deduped
  raw feed means you can replay an install's whole history and converge to the
  same clean events — no duplicates, no drift.
- **Never throw on an unknown raw type.** Log it and move on; one unrecognized
  event must not break an install's derivation.

---

## What to build vs. what to skip

If you're porting off Mantle, here's the line between the real mechanics and
Mantle-internal infrastructure.

**Reproduce (the actual pipeline):**
1. A polled raw-event table, deduped, with an overlapping since-buffer.
2. A per-install, time-ordered state machine that emits clean lifecycle events.
3. Monthly normalization + `netChange` so MRR is just a sum.
4. The cancel trap (activation-after, activation-before-same-charge, chained
   paid→free) plus the offline reconciler.
5. Local trial state + a nightly `trial_expired` sweep, and trial-converts-via-
   plan-change accounting.
6. Two-stage uninstall-reason normalization (preset map → LLM) with a flagged
   store-closure subset.
7. Idempotent re-derivation keyed on `platformEventId`, and an `ignored` flag for
   non-counting events.

**Skip or substitute (Mantle-specific):**
- A Kafka + ClickHouse + Elasticsearch fan-out — a plain SQL query over your clean
  `app_events` table is fine until you're at real scale; add projections later.
- Mantle's exact reason taxonomy and LLM prompt — keep the *shape* (preset →
  fallback, store-closure flag), tune the codes to your product.
- The webhook-based real-time uninstall enrichment — nice latency win, but the
  feed is the authoritative source; you can start poll-only.

---

## Where to go next

- Building by hand? Work straight down the "Reproduce" list; reach for
  [`implementation-spec.md`](./implementation-spec.md) when you need the exact
  state-machine branches, the suppression windows, and the acceptance tests.
- Want an agent to do it? Paste [`pipeline-prompt.md`](./pipeline-prompt.md) or
  [`implementation-spec.md`](./implementation-spec.md) into your coding agent, or
  install the [skill](./skill/shopify-customer-events/) into your app's workspace.
