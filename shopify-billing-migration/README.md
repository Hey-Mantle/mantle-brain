# Shopify Billing Migration — a build guide

*Part of [mantle-brain](../README.md): an open-source library to help Shopify
Partners recreate Mantle's tools as Mantle winds down.*

This guide explains **how Mantle's Shopify billing wrapper worked and how to
rebuild it in your own app** — the architecture where **subscriptions live on
Shopify** and your app keeps a reconciled local mirror, plus **how to migrate
merchants onto Shopify-managed billing** correctly across each billing model,
in the right **order of operations**.

> **Scope.** This covers *standard* Shopify-managed billing — where Shopify owns
> the billing clock and collects the recurring charge for you. **Flex billing**
> (the `$0`-recurring + usage-record trick that lets you move merchants between
> tiers *in-place* without re-approval) is a different beast and has its own
> topic: **[`../flex-billing/`](../flex-billing/README.md)**. Read this one
> first for the wrapper mental model; reach for flex billing only when you need
> re-approval-free tier changes.

If you'd rather hand the job to a coding agent, this folder ships two companions:

- **[`implementation-spec.md`](./implementation-spec.md)** — a dense, imperative
  build spec (data model, exact API shapes, ordered procedures, acceptance
  tests). Paste it into Claude/Cursor and say "implement this in my app."
- **[`skill/shopify-billing-migration/`](./skill/shopify-billing-migration/)** —
  a Claude Code / Agent skill. Copy it into your app's `.claude/skills/` and the
  agent will walk the migration for you. See its
  [README](./skill/shopify-billing-migration/README.md).

---

## The problem this solves

You're building (or porting) an app that bills merchants, and you want
**Shopify to be the payment rail**: Shopify shows the approval screen, collects
the money, handles refunds and regional compliance, and emits the charge events.
That's the "managed" part — you don't run a payment processor.

But Shopify's Billing API is thin. It gives you a handful of GraphQL mutations
and **almost no webhooks**. It does *not* model:

- your plan catalog (features, add-ons, metered metrics, trials, discounts),
- the business meaning of a plan change ("upgrade" vs "downgrade"),
- a clean migration path from another biller (Stripe, manual, Apple) onto
  Shopify,
- analytics, MRR, churn, reconciliation.

So you build a **wrapper**: a local model that *mirrors* the Shopify
subscription and carries all the business logic Shopify doesn't. The migration
problem is then: **how do you move a merchant from "no subscription" or "some
other subscription" onto a correctly-mirrored Shopify subscription — without
dropping a charge, double-charging, or leaving the two systems disagreeing?**

---

## The core idea — the mirror pattern

> **Shopify is the system of record for *"is this approved and collecting?"*.
> Your database is the system of record for *"what is the business agreement?"*.**

Concretely:

1. You define plans, trials, discounts, and metered metrics **in your DB**.
2. To subscribe a merchant, you create a **local subscription row in a pending
   state**, then push a matching `AppSubscription` to Shopify with
   `appSubscriptionCreate`.
3. Shopify returns a **`confirmationUrl`**. You send the merchant there; they
   approve **once**.
4. Shopify redirects back to your **`returnUrl`**. You **verify against Shopify**
   (don't trust the redirect), then flip your local row to active and **mirror
   Shopify's billing dates and line-item GIDs** back into your DB.
5. From then on, **Shopify collects the recurring charge on its own cycle** and
   emits charge events. You **reconcile** those into your mirror via the
   `APP_UNINSTALLED` webhook + the Partner API + scheduled sync jobs.

```
   Your DB (pending)                 Shopify
   ───────────────                  ─────────
   subscription.active=false  ──►  appSubscriptionCreate
                                        │
                                   confirmationUrl
                                        │
   merchant approves ONCE  ◄────────────┘
        │
   returnUrl callback  ──►  GET app install (VERIFY status == active)
        │                        │
        ▼                        ▼
   mirror back:  active=true, activatedAt, currentPeriodEnd,
                 usage line-item GIDs, capped amounts
        │
        ▼
   cancel any OTHER active subscription for this install
        │
        ▼
   Shopify collects on its own cycle ──► charge events
        │
   nightly sync + webhooks ──► reconcile mirror
```

The single most important structural fact, and the thing that most distinguishes
this from flex billing:

> **In standard Shopify billing you do NOT run a charging cron. Shopify owns the
> billing clock and collects automatically.** Your job is to *mirror* its cycle,
> not to drive it. (Flex billing inverts this — see the sibling topic.)

---

## Mental model

| Concept | What it means |
|---|---|
| **Local subscription** | Your row that mirrors one Shopify `AppSubscription`. Carries plan, trial, discount, period dates, and a pointer to the Shopify charge. The business agreement. |
| **`billingType`** | Which rail this subscription rides: `shopify` / `stripe` / `apple` / `manual` / `test`. Routes every create/cancel/sync to the right provider contract. Migration = changing the rail, which is a *data* migration, not a flag flip. |
| **Charge mirror** | A row mirroring Shopify's `AppSubscription` charge object: its GID, status (`pending`/`active`/`declined`/`frozen`/`expired`), and billing-on date. Synced from Shopify, never invented locally. |
| **Line item** | One priced component (recurring base, usage/metered, add-on). You persist Shopify's **line-item GID** so you can post usage records and resize caps later. |
| **`confirmationUrl`** | The one Shopify-hosted approval page. Immutable per charge. The merchant clicks it exactly once. |
| **`returnUrl`** | Where Shopify redirects after approval. Your activation entry point — but **verify with Shopify before trusting it**. |
| **Source of truth** | Shopify: approval, collection, current status, usage balance. You: plan catalog, trials, discounts, "upgrade vs downgrade", analytics. |
| **Eventual consistency** | Shopify gives you few webhooks; mirrors drift. Scheduled reconciliation is not optional — it's how the mirror heals. |

---

## How a Shopify subscription lives — end to end

### 1. Plan setup
Define each plan in your DB: `amount`, `currencyCode`, `interval`
(`EVERY_30_DAYS` or `ANNUAL`), `trialDays`, any metered `usageCharges`
(per-unit price + a monthly `cappedAmount`), and feature/add-on relationships.
These are *your* concepts; Shopify only ever sees the line items you derive from
them at subscribe time.

### 2. Subscribe (create + confirm)
Create a **pending** local subscription, then call `appSubscriptionCreate` with
line items derived from the plan (a recurring line, and/or a usage line with a
cap). Send Shopify the trial via `trialDays` (standard billing lets Shopify run
the trial — unlike flex). Persist the returned subscription GID, the **usage
line-item GID**, and the `confirmationUrl`. Redirect the merchant to the
`confirmationUrl`.

### 3. Activate (the return round-trip)
Shopify redirects to your `returnUrl`. **Re-query Shopify** for the install's
active subscriptions and confirm the charge status is `active`. Only then:
mirror the billing dates/line items, flip your row to `active`, **cancel any
other active subscription for this install**, mark your charge mirror `active`,
bust caches, and kick off a reconciliation sync. If the verify call fails
(Shopify lag), still return the merchant to your app and let the async sync heal
the state.

### 4. Steady state (Shopify collects)
Shopify charges the recurring fee on its own cycle and emits charge events. You
do nothing to collect. Periodically sync the **current period dates**
(`currentPeriodStart/End`, next billing date) from Shopify into your mirror,
because Shopify does **not** webhook each cycle rollover.

### 5. Plan change / migration
A plan change is "create a new subscription, then cancel the old one" — but the
**ordering is load-bearing** (see below). A *provider* migration (Stripe →
Shopify) is the same shape: stand up the Shopify subscription, confirm it's
collecting, *then* tear down the old rail, *then* flip `billingType`.

### 6. Cancellation
Call `appSubscriptionCancel` (tolerate "already uninstalled" errors), then mark
your row canceled — optionally `cancelAtPeriodEnd` so the merchant keeps service
until the period they paid for ends.

### 7. Reconcile, forever
The `APP_UNINSTALLED` webhook + scheduled jobs over the Partner API keep the
mirror honest: repair orphaned states, pull in charges/transactions, advance
period dates, and flag (or fix) divergence.

---

## Migration by billing model

The wrapper is the same for every plan; what changes per model is **the line
items you send** and **the edge cases around proration, caps, and confirmation**.
Build the models you actually sell.

> Flex billing (`$0` recurring + usage records for re-approval-free tier moves)
> is intentionally **not** covered here — see
> [`../flex-billing/`](../flex-billing/README.md).

### Flat-rate recurring (the baseline)
One recurring line item: `appRecurringPricingDetails` with `price` +
`interval`. Trials go to Shopify via `trialDays`. Discounts attach to the
recurring line (`discount.value` as amount or percentage, plus
`durationLimitInIntervals`). Shopify handles the rest. **Gotcha:** the Shopify
`interval` must exactly match the plan's (`EVERY_30_DAYS` vs `ANNUAL`); there's
no normalization on Shopify's side.

### Annual vs 30-day
Same as flat-rate, different `interval`. Keep your plan's interval and Shopify's
interval in lockstep — a reconciliation check should compare them, because an
annual plan accidentally created as 30-day silently under-bills by 12×.

### Usage-based / metered
A separate `appUsagePricingDetails` line item carrying `terms` and a monthly
`cappedAmount`. The subscription create only establishes the *cap*; you bill
actual usage afterward with `appUsageRecordCreate` against the **usage
line-item GID**. **Sharp edges:**
- Shopify **rejects a usage record that would exceed the cap** — size the cap
  with headroom, and decide what happens at the ceiling (reject / re-approve a
  higher cap / auto-upgrade).
- **Raising a cap re-triggers merchant approval** (`appSubscriptionLineItemUpdate`
  returns a `confirmationUrl`). Size generously up front.
- Mirror Shopify's returned `balanceUsed`/`cappedAmount` back after each record;
  it's the only place that state lives.

### Hybrid (recurring + usage)
Two line items in one `appSubscriptionCreate` — a recurring base and a usage
line. Note **discounts apply to the recurring line only**, not usage. Decide how
pending usage behaves across a plan change (roll over or reset).

### One-time purchase
Not a subscription at all: `appPurchaseOneTimeCreate`. It returns its own
`confirmationUrl` and a one-time charge object. **No trial, no discount, no
recurring cycle.** Confirmation lands via webhook rather than a subscription
activation. Keep these out of the subscription mirror; track them as standalone
charges.

### Free / `$0` plans
If the plan is `$0` and has no usage line, **don't call Shopify at all** —
activate the local subscription immediately, no confirmation round-trip. (If it
has a usage line, you still create a Shopify subscription with just the usage
line.) **Gotcha:** a `$0` plan with an active *local* subscription but **no
Shopify subscription** will look like a mismatch to a naive reconciler — teach
the reconciler that `$0`-no-usage means "no Shopify object expected."

---

## The order of operations (why sequencing is the whole game)

Most migration bugs are ordering bugs. The rules:

1. **Create the new subscription before touching the old one.** Never cancel a
   paying subscription (Shopify *or* Stripe) until its replacement is confirmed
   **active** on Shopify. Cancel-first opens a service gap and can lose a cycle.
2. **Verify with Shopify before activating locally.** The return redirect only
   means "the merchant came back," not "the charge is active." Re-query the
   install; trust Shopify's `status == active`, not the URL.
3. **Cancel *other* active subscriptions only after the new one verifies.** Do it
   in the **same transaction** as the local activation, so you can never end up
   with two active subscriptions (double charge) or zero (gap).
4. **Flip `billingType` last.** In a provider migration, only switch the
   merchant's default rail to `shopify` *after* every Shopify subscription is
   live and the old rail is torn down. Flip-first strands new operations on a
   rail that isn't ready.
5. **Collect anything outstanding before you cancel** (mostly a metered/flex
   concern): if the old subscription had un-billed usage, charge it before
   canceling or you lose that revenue.
6. **Register the `APP_UNINSTALLED` webhook and reconcile** after activation —
   idempotently, so re-runs are safe.

Failure modes this ordering prevents:

- **Double charge:** new activated while old still active → bill twice until
  reconciliation notices. Prevented by rule 3 (atomic cancel-others-on-activate).
- **Service gap / lost cycle:** old canceled before new confirmed → merchant
  unbilled and unserved until they re-approve. Prevented by rules 1–2.
- **Stranded migration:** `billingType` flipped early → new charges try the
  wrong rail. Prevented by rule 4.
- **Abandoned confirmation:** merchant never clicks `confirmationUrl` → a pending
  local row lingers forever. There's no Shopify timeout — add your own cleanup
  job for stale pending subscriptions.

---

## The Shopify API surface

A small set of mutations and one query carry the whole wrapper.

| Operation | Used for | Notes |
|---|---|---|
| `appSubscriptionCreate` | New subscription (any recurring model) | `lineItems` (recurring and/or usage), `trialDays`, `returnUrl`, `test`. Returns `confirmationUrl` + the subscription and line-item GIDs. **Unique per shop** — calling twice for the same intent errors; handle it. |
| `appPurchaseOneTimeCreate` | One-time charge | Returns its own `confirmationUrl`. No trial/discount/recurring. |
| `appUsageRecordCreate` | Metered charge against a usage line | Needs the **usage line-item GID**. Rejected if it would exceed the cap. |
| `appSubscriptionLineItemUpdate` | Resize a usage cap | **Raising the cap returns a `confirmationUrl`** (re-approval). Lowering usually doesn't. |
| `appSubscriptionCancel` | Real cancellation | Tolerate 401/404 ("already uninstalled"). |
| `appSubscriptionTrialExtend` | Extend a trial | Optional. |
| *Query:* app install / active subscriptions | **Verify** on the return round-trip, and reconcile | The ground truth for status, billing dates, line items, usage balance. |

**Webhooks:** register **`APP_UNINSTALLED`** (idempotently — Shopify rejects
duplicates with a benign "address already taken"). Do **not** count on
`app_subscriptions/update` or any per-cycle webhook to drive billing — they're
unreliable for that. Scheduled reconciliation over the Partner API is what keeps
the mirror correct.

---

## Gotchas — the lessons that cost real money

- **Verify, don't trust the redirect.** The `returnUrl` callback must re-query
  Shopify and confirm `status == active` before activating. A merchant can hit
  the return URL without a live charge.
- **Cancel-others must be atomic with activate-self.** One transaction, after
  verification. Otherwise you get two active subs (double charge) or zero (gap).
- **Never cancel the old rail before the new one is collecting.** True within
  Shopify (plan change) and across rails (Stripe → Shopify).
- **No charging cron.** Shopify collects; you mirror. If you find yourself
  posting recurring charges on a schedule, you've drifted into flex-billing
  territory — go read that topic instead.
- **Sync the period clock yourself.** Shopify doesn't webhook each cycle
  rollover; a scheduled job must advance `currentPeriodStart/End` and the next
  billing date in your mirror.
- **Raising a usage cap re-approves.** Size caps generously so you never hit
  `appSubscriptionLineItemUpdate` on the hot path.
- **Cap-full usage records are rejected.** Decide the ceiling policy
  deliberately (reject / re-approve / auto-upgrade); don't silently drop the
  charge.
- **`$0`-no-usage plans create no Shopify object.** Activate locally; teach
  reconciliation not to flag the "missing" Shopify subscription.
- **One active subscription per install — enforce it.** When a new charge
  activates, cancel the rest.
- **Repair orphaned states.** If Shopify says frozen/canceled but your mirror
  still says active (a dropped webhook), reconciliation must fix it without
  clobbering an already-set `canceledAt`.
- **Abandoned confirmations never expire.** Add a cleanup job for stale pending
  subscriptions; Shopify won't do it for you.
- **Idempotency everywhere.** Activation, cancellation, webhook registration, and
  sync all re-run; make each safe to repeat.
- **Mind currency.** Keep both the raw charged amount/currency and a normalized
  amount for reporting. Some rails (e.g. usage records) may be effectively
  single-currency — don't assume the cap currency and the record currency agree.

---

## What to build vs. what to skip

**Reproduce (the actual wrapper + migration engine):**
1. A local subscription model that mirrors one Shopify `AppSubscription`, with a
   `billingType` rail selector, period dates, trial/discount fields, and a
   charge-mirror row holding the Shopify charge GID + status.
2. Line-item rows that persist Shopify GIDs (recurring, usage, add-on).
3. Subscribe = pending row → `appSubscriptionCreate` → store
   `confirmationUrl` + GIDs → redirect.
4. A `returnUrl` activation handler that **verifies against Shopify**, mirrors
   dates/line items, and **atomically activates-self + cancels-others**.
5. Per-billing-model line-item derivation (flat, annual, usage, hybrid,
   one-time, free) — only the models you sell.
6. Migration ordering: new-before-old, verify-before-activate, collect-before-
   cancel, flip-`billingType`-last.
7. `APP_UNINSTALLED` webhook (idempotent) + scheduled reconciliation: period-date
   sync, orphan repair, charge/transaction backfill, divergence reporting.
8. A cleanup job for abandoned pending subscriptions.
9. The gotchas above.

**Skip or substitute (Mantle-specific infrastructure):**
- A columnar analytics pipeline (ClickHouse/Kafka in Mantle's case) — a plain
  query over your own charge/subscription tables is fine until real scale.
- Multi-provider abstractions (Apple/Stripe/manual) unless you actually bill on
  them — the wrapper shape is the same, but each rail is its own contract.
- Bespoke backfill/remediation one-offs — those are operational cleanup, not
  core mechanics.
- Flex billing — separate topic.

---

## Where to go next

- Building by hand? Work down the "Reproduce" list above; reach for
  [`implementation-spec.md`](./implementation-spec.md) when you need exact API
  shapes, the data model, and the ordered procedures.
- Want an agent to do it? Paste
  [`implementation-spec.md`](./implementation-spec.md) into your coding agent, or
  install the [skill](./skill/shopify-billing-migration/) into your app's
  workspace.
- Need re-approval-free tier changes / in-place upgrades? That's
  [`../flex-billing/`](../flex-billing/README.md).
</content>
</invoke>
