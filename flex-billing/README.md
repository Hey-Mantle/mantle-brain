# Flex Billing on Shopify — a build guide

*Part of [mantle-brain](../README.md): an open-source library to help Shopify
Partners recreate Mantle's tools as Mantle winds down.*

This guide explains **how Mantle's Flex Billing worked and how to rebuild it in
your own app** — flexible, tiered Shopify subscriptions where you can move a
merchant up or down a pricing tier *in place*, charge or credit the difference,
and never send them back through Shopify's subscription approval screen.

If you'd rather hand the job to a coding agent, this folder ships two companions:

- **[`implementation-spec.md`](./implementation-spec.md)** — a dense,
  imperative build spec (data model, exact API shapes, pseudocode, acceptance
  tests). Paste it into Claude/Cursor and say "implement this in my app."
- **[`skill/shopify-flex-billing/`](./skill/shopify-flex-billing/)** — a Claude
  Code / Agent skill. Copy it into your app's `.claude/skills/` and the agent
  will walk the migration for you. See its
  [README](./skill/shopify-flex-billing/README.md).

---

## The problem flex billing solves

Shopify's app subscriptions are built around a fixed **recurring price**. The
catch: **changing that recurring price requires the merchant to re-approve the
subscription** on a Shopify-hosted confirmation screen. That's fine for a
one-time signup, but it's death for a product that wants to:

- move a customer between plan tiers smoothly,
- bump them to a higher tier automatically when their usage grows,
- or pro-rate the difference mid-cycle,

…because every one of those would throw a "please re-approve your subscription"
wall in front of the customer. Many never click it, and your upgrade silently
fails.

**Flex billing is the workaround.** It sidesteps re-approval entirely.

---

## The core trick

> Set the Shopify **recurring price to $0.00**, and collect the real money
> through **usage charges**.

Here's the shape of it:

1. You create the Shopify subscription with **two line items**:
   - a **recurring** line priced at **$0** (this is the part that would require
     re-approval if it ever changed — so it never changes), and
   - a **usage** line carrying a high **`cappedAmount`** (the maximum Shopify
     will let you bill against it without re-approval).
2. The merchant approves this **once**.
3. From then on, you bill the monthly fee by posting a **usage record**
   (`appUsageRecordCreate`) against that usage line — at the start of each
   billing period.
4. To upgrade someone mid-cycle, you post a **pro-rated usage record** for the
   difference. To downgrade, you issue a **pro-rated credit**. The recurring
   price is still $0, so **Shopify never asks for re-approval.**

Everything else — when the period starts and ends, when to charge, trials,
proration math, the tier ladder — you manage in **your own database**. Shopify's
recurring cycle is irrelevant; you never read it.

```
   Merchant approves ONCE
           │
           ▼
   ┌───────────────────────────────────────────────┐
   │  Shopify App Subscription (per merchant)       │
   │                                                │
   │   Recurring line ── $0.00  (never changes)     │
   │   Usage line     ── cappedAmount: $5000        │◄── every charge posts here
   └───────────────────────────────────────────────┘
           ▲                         ▲
           │ monthly fee             │ upgrade proration
           │ (usage record)          │ (usage record)
           │                         │
       your daily cron          your tier-change flow ──► downgrade = app credit
```

---

## Mental model

A few ideas you need to hold in your head:

| Concept | What it means |
|---|---|
| **Flex plan** | A normal plan row with a `flexBilling = true` flag. Its Shopify recurring price is forced to $0; revenue comes through usage charges. **Each pricing tier is its own flex plan.** |
| **Usage line item** | The Shopify line that carries the spend cap. You store its GID and post every usage record against it. This GID is the single most important thing to persist. |
| **Capped amount** | The Shopify spend ceiling on the usage line. **Must comfortably exceed the largest fee + proration you'll ever post**, because the monthly fee rides through this line too. |
| **Billing period** | `currentPeriodStart` / `currentPeriodEnd` / `nextBillingDate`, tracked entirely in *your* DB. Advanced on every charge — even $0 charges advance it. |
| **Tier ladder** | Lower tiers point at the next-higher tier (`autoUpgradeToPlanId`) and declare a usage band (`limitMin`/`limitMax`). Cross the band's top and you auto-upgrade. |
| **Audit event** | An append-only row for each tier change and each recurring charge. Pure observability — it must never gate real money. |

The most important structural fact: **flex bills in advance (prepayment), not in
arrears.** At signup (no trial) you collapse the first period to "today" so your
very next cron run charges the first period up front.

---

## How a flex subscription lives — end to end

### 1. Plan setup
Create each tier as its own plan with `flexBilling = true` and a generous
`usageChargeCappedAmount`. For automatic tiering, give each tier a usage band
(`limitMin`/`limitMax` against some metric) and point lower tiers at the next one
up (`onUsageLimitReached = "upgrade"` + `autoUpgradeToPlanId`).

### 2. Subscribe
Call `appSubscriptionCreate` with the two line items (a $0 recurring line and the
capped usage line). Send `trialDays: 0` to Shopify even if your plan has a trial
— **trials are managed locally, never by Shopify.** Persist the subscription GID
and, critically, the **usage line item's GID**. Redirect the merchant to the
returned `confirmationUrl`; they approve once.

Then collapse the first period: `currentPeriodStart = currentPeriodEnd =
nextBillingDate = today`. Don't charge during signup — your next cron run will
bill the first period as prepayment.

### 3. Usage accrues
Your app reports metered events as usual. If you offer metered overage, those
usage records post against the **same** usage line item. When usage crosses a
tier's `limitMax`, you queue an auto-upgrade (step 6).

### 4. The daily cron charges due periods
Once a day, select every active flex subscription whose `nextBillingDate` has
passed, and for each one:

- take a per-subscription lock (so a simultaneous manual change can't
  double-charge),
- compute the (possibly discounted) amount,
- post a usage record for it against the usage line item,
- write a charge row and an audit event,
- **advance the period** (`nextBillingDate`, `currentPeriodStart/End`).

If the amount is $0 (a 100%-off discount, say), skip Shopify but **still advance
the period** — Shopify rejects $0 usage records.

### 5. In-place tier change (the magic part)
When a customer moves between tiers mid-cycle, you do **not** touch the Shopify
subscription. Instead:

1. Collect any outstanding charge first (so a same-day change doesn't lose a
   period's fee).
2. Compute **day-based proration** over the current cycle (formula below).
3. If the change costs more (**upgrade**): post a pro-rated usage record for the
   difference — *if* it fits under the cap. If it doesn't fit, fall back to a
   fresh confirmable subscription.
4. If it costs less (**downgrade**): issue a pro-rated **credit** via the Partner
   API, capped at what you've actually collected this period.
5. Cancel the old subscription *locally only* (you reuse the one Shopify
   subscription), copy the billing cycle and usage line forward, and start
   charging the new amount next cycle.

### 6. Automatic upgrade
When metered usage crosses a tier's `limitMax`, create a subscription to the next
tier — **through the exact same in-place path as a manual change**, so it
pro-rates identically. Copy the billing cycle forward so the date doesn't reset,
tag it with the triggering usage-charge id (so you can tell auto from manual
later), dedupe against an existing target-tier subscription, and cap how many
times it can cascade.

### 7. Record the event
Every tier change and every recurring charge writes one audit row. This is for
your dashboards and analytics — it gates no money.

---

## The proration math

Day-based, over the current cycle:

```
daysInCycle    = whole days between currentPeriodStart and currentPeriodEnd
daysRemaining  = clamp(whole days from today to currentPeriodEnd, 0, daysInCycle)

dailyRateOld   = oldPlanAmount(after discount) / daysInCycle
dailyRateNew   = newPlanAmount(after discount) / daysInCycle

netProRated    = (dailyRateNew − dailyRateOld) × daysRemaining
```

- `netProRated > 0` → **charge** that amount as a usage record (upgrade).
- `netProRated < 0` → **credit** `−netProRated` (downgrade), but capped at what
  you've collected for the current period.
- `netProRated == 0` → no money moves, but the change still commits in place
  (don't fall through to a re-approval).
- `daysInCycle == 0` → a degenerate cycle (e.g. a tier change the same day as
  signup, before the first charge). Treat proration as $0 rather than dividing by
  zero. Collecting the outstanding charge first normally advances the period out
  of this state.

One subtlety: the **label** ("upgraded" vs "downgraded") is decided by comparing
list prices (`newPlan.amount > oldPlan.amount`), *independently* of the proration
sign. An equal-priced swap gets labeled "downgraded" with $0 proration — that's
expected, not a bug.

---

## The Shopify API surface

You only need a handful of mutations:

| Mutation | Used for | Notes |
|---|---|---|
| `appSubscriptionCreate` | Initial signup | Two line items: $0 recurring + capped usage. Returns `confirmationUrl`. |
| `appUsageRecordCreate` | Monthly fee **and** upgrade proration | The workhorse. **Currency is effectively USD** (see caveat). $0 records are rejected. |
| `appSubscriptionLineItemUpdate` | Resize the cap | **Raising the cap triggers re-approval** — size generously up front to avoid this. |
| `appCreditCreate` (**Partner API**) | Downgrade credits | Not the Admin API `appCredit`. Credits are **irreversible**. |
| `appSubscriptionCancel` | Real cancellation | During tier changes you cancel *locally only* and reuse the one Shopify sub. |

**Webhooks:** register only `APP_UNINSTALLED`. Shopify does **not** give you a
subscription-update or usage webhook — your daily cron plus the one-time
charge-approval return callback drive everything.

---

## Gotchas — the lessons that cost real money

These are the sharp edges. Each one is a bug someone already hit.

- **Bill the first period in advance.** Collapse the first period to today so the
  next cron bills it. There's no free first cycle.
- **A full cap silently drops the monthly fee.** If the usage cap is full when
  the period charge fires, Shopify rejects the usage record — and in Mantle's
  implementation the period clock advances *anyway*, so that month's fee is
  **lost**. The fix is twofold: (a) **size the cap so it never fills** under
  normal use, and (b) if you want to be safe, check whether the charge actually
  succeeded before advancing the period. Don't blindly inherit the
  advance-no-matter-what behavior.
- **$0 charges still advance the period** — just never POST a $0 usage record.
- **Collect outstanding charges before a tier change**, using a day-level date
  comparison, or a same-day change loses that period's fee.
- **Cap downgrade credits** at what you actually collected this period. A day-1
  downgrade (before any charge) issues **$0**.
- **Make downgrade credits opt-out-able** per app — they're irreversible.
- **Fall back to a confirmable subscription** when an upgrade proration won't fit
  under the cap, instead of dropping the upgrade.
- **Keep trials local.** Never send `trialDays` to Shopify; set
  `nextBillingDate` to the trial end. Clear trial state on conversion so it can't
  be re-triggered.
- **Lock per subscription** so the cron and a manual change can't double-charge.
- **Skip paused subscriptions** in the main cron and sweep them separately.
- **Private/dev apps can't use the Billing API** — Shopify rejects them; swallow
  that per-subscription instead of failing the whole batch.

### A note on currency
Mantle's flex charge rail is **hard-coded to USD** — `appUsageRecordCreate`,
proration charges, and credits all post in USD, even though the usage cap is
created in the plan's currency. So a "€50" plan actually posts a **$50 USD**
charge, with no FX. If you serve non-USD merchants, either restrict flex plans to
USD or align the record currency, credit currency, and cap currency with a
consistent FX conversion. **Don't copy the hard-coded USD blindly.**

---

## What to build vs. what to skip

If you're porting off Mantle, here's the line between the real mechanics and
Mantle-internal infrastructure.

**Reproduce (the actual billing engine):**
1. A plans table with an immutable `flexBilling` flag and a usage cap; one plan
   per tier; usage bands + auto-upgrade links for tiering.
2. Signup = $0 recurring line + capped usage line, with the first period
   collapsed to today.
3. Your own billing period in your DB; skip paused subs.
4. A daily cron that charges due periods, advances the clock, and handles the $0
   case — backed by a partial index matching the cron's filter, and a
   per-subscription lock.
5. In-place tier changes: collect-first, day-based proration, cap-gated upgrade
   charge with a confirmable-subscription fallback, Partner-API downgrade credit
   capped at collected, with an opt-out.
6. Automatic upgrade through the same in-place path, with loop guards.
7. An append-only audit table (non-load-bearing).
8. The gotchas above.

**Skip or substitute (Mantle-specific):**
- A columnar analytics pipeline (ClickHouse/Kafka in Mantle's case) — a plain
  query over your audit table is fine until you're at real scale.
- Reconstructing "subscribed" events from a search index — just **store** a
  subscribed event when a flex subscription activates.
- Any internal MRR-reconstruction-from-usage-stream scheme — not needed to bill
  correctly.
- Backfill/remediation one-offs — operational cleanup, not core mechanics.

---

## Where to go next

- Building by hand? Work straight down the "Reproduce" list above; reach for
  [`implementation-spec.md`](./implementation-spec.md) when you need exact API
  shapes and pseudocode.
- Want an agent to do it? Paste
  [`implementation-spec.md`](./implementation-spec.md) into your coding agent, or
  install the [skill](./skill/shopify-flex-billing/) into your app's workspace.
