---
name: shopify-customer-events
description: >-
  Build a pipeline that turns the Shopify Partner API app-events feed into clean
  customer lifecycle events (subscribed, upgraded, downgraded, unsubscribed,
  uninstalled, trial_expired) for MRR and churn analytics. Use when the user
  wants to ingest Shopify partner/app events, derive subscription lifecycle
  events, compute MRR deltas (netChange), suppress plan-change "cancel" noise,
  derive trials, normalize uninstall reasons, or migrate off Mantle's
  customer-events pipeline into their own codebase.
---

# Shopify Customer Events

## What this skill does

Helps you build the pipeline that compiles Shopify's raw **Partner API
app-events feed** into clean **customer lifecycle events** — the
`subscribed` / `upgraded` / `downgraded` / `unsubscribed` / `uninstalled` /
`trial_expired` stream every MRR and churn chart is built on.

The mechanism: keep the raw feed verbatim in one table, then derive clean events
from it with a **per-install state machine** replayed in time order. The hard
parts — and the reasons this isn't a simple type-to-type map — are the **cancel
trap** (a lone cancel is usually a plan change, not churn, including the
no-activation-webhook paid→free case), **monthly normalization** of `netChange`
MRR deltas, **trial derivation** (nothing in the feed fires at trial end), and
**uninstall-reason normalization** (preset map → LLM fallback, with store
closures flagged separately).

The full, precise mechanics live in
**[`references/implementation-spec.md`](./references/implementation-spec.md)**.
Read it before writing code — it has the exact state-machine branches, the
suppression windows, the normalization math, and the edge cases. This file is the
*procedure*; the spec is the *reference*.

## When to use it

Trigger on intents like: "ingest Shopify partner/app events," "turn the app-events
feed into lifecycle events," "build MRR/churn from Shopify subscription events,"
"my plan changes are showing up as churn," "compute netChange / MRR deltas,"
"detect trials from Shopify events," "categorize uninstall reasons," or "migrate
off Mantle's customer-events pipeline."

## How to drive the implementation

Work through these phases. **Do not dump code blindly** — discover the user's
stack and existing event/subscription code first, then map the spec onto it.

### Phase 1 — Orient (ask, then read)
1. Determine the stack: language, web framework, ORM/database, job/cron runner,
   and how they already talk to the Shopify **Partner API** (do they have
   Partner API access and an `appEvents` feed query?).
2. Find existing code: any current partner/app-event ingestion, plan model,
   subscription model, a customer-events/analytics table, cron/scheduled tasks,
   and any LLM/translation helpers (for uninstall reasons).
3. Ask which slices they need:
   - raw ingestion + basic lifecycle events only?
   - + the cancel trap (plan-change suppression)?
   - + `netChange`/MRR deltas and an MRR rollup?
   - + trial derivation? + uninstall-reason normalization?
   - + a search/columnar fan-out, or is a SQL query over the clean table enough?

### Phase 2 — Read the reference
Read [`references/implementation-spec.md`](./references/implementation-spec.md)
end to end. The invariants in its §0 are non-negotiable; the §9 MUST/MUST NOT
rules are the bugs you're being paid to avoid.

### Phase 3 — Data model (spec §1)
Add/confirm: a verbatim `raw_app_events` table with the
`(install, type, occurredAt, chargeGid)` dedup key; a clean `app_events` table
with a **unique deterministic `platformEventId`**, `netChange`,
`previousSubscriptionId`, and a soft-delete column; a `uninstall_app_events`
child table; and enough of the Shopify `charge`/`plan` to normalize money. Write a
migration. Settle the clean lifecycle-type enum (spec §1.5).

### Phase 4 — Poll the feed (spec §2)
Implement cursor-paginated polling of the `appEvents` feed with an overlapping
since-buffer, persisting raw events first and deduping on the unique key. Catch
per-event errors. Wire it to the user's scheduler.

### Phase 5 — The state machine (spec §3)
Implement the per-install, time-ordered fold carrying `currentState`,
`currentAmount`, and `mostRecentSubscription`. Implement monthly normalization
(spec §5.1) and the activation branch (subscribe / resubscribe / upgrade /
downgrade). Log-and-skip unknown types. Set `previousSubscriptionId` (spec §3.5).

### Phase 6 — The cancel trap (spec §4) — do this carefully
Implement the three suppression checks (activation-after 60s,
activation-before-same-charge 5s, chained paid→free `upgraded`/`downgraded`
±60s), the soft-delete-on-suppress, and the **state re-baseline** for the
paid→free case. Add the offline reconciler for stray `unsubscribed` events.

### Phase 7 — netChange / MRR (spec §5)
Attach the signed monthly delta to each event, reflect discounts (effective
price, not list, unless app-credit discounts), and prove MRR = sum of `netChange`
over non-ignored events.

### Phase 8 — Trials (spec §6, optional)
Derive trial status from local dates; add the nightly idempotent `trial_expired`
sweep; handle trial-converts-via-plan-change so it isn't counted as churn.

### Phase 9 — Uninstall reasons (spec §7, optional)
Build the enrichment job: translate → preset map → LLM fallback → cache, store
`reasonCodes[]` + primary, and flag the store-closure subset separately.

### Phase 10 — Verify
Walk the spec §10 acceptance scenarios as a checklist. At minimum prove: first
subscribe + annual normalization; a mid-cycle plan change emits one event and
**zero** churn; paid→free suppresses the cancel and re-baselines; real churn
emits `unsubscribed`; full replay is idempotent.

## Guardrails

- **Preserve the five invariants** (spec §0): two layers (raw + derived), a
  per-install time-ordered state machine, monthly-normalized money, the
  guilty-until-proven cancel trap, and idempotency on a deterministic
  `platformEventId`. Breaking any of them produces phantom churn or MRR drift.
- **The cancel trap is the highest-value rule.** A lone cancel is presumed a plan
  change; only suppress-or-emit per spec §4 — never blindly record churn.
- **Handle paid→free explicitly** — it emits no activation webhook; detect the
  chained plan-change event and re-baseline the state machine.
- **Normalize money to monthly before comparing** (annual ÷ 12; flex from plan
  nominal) — or annual plans break every chart.
- **Don't delete non-counting events** — mark them `ignored` at the read boundary.
- **Never throw on an unknown raw type** — log and skip.
- **Treat search/columnar stores as derived projections** — their failures never
  block the authoritative clean-event write.
- Prefer editing the user's existing models/clients over inventing parallel ones.
  Match their conventions.
