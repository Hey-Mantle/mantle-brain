# Flex Billing — Implementation Spec (for agents)

> **Audience:** a coding agent implementing flexible, tiered Shopify app
> subscriptions ("flex billing") in an arbitrary app. This is a precise,
> self-contained build spec — no external files required. It is the reference
> bundled with the `shopify-flex-billing` skill; the procedure for using it is
> in the skill's `SKILL.md`.
>
> **Provenance:** distilled from the production behavior of Mantle's Flex
> Billing engine. Stack-agnostic; map the generic table/field names below onto
> your own ORM. Where a rule encodes a hard-won lesson, it is marked **MUST** /
> **MUST NOT**.

---

## 0. Goal & invariants

Build Shopify app subscriptions where you can **move a merchant between pricing
tiers in-place — and bill the difference — without sending them through
Shopify's mandatory subscription re-approval flow.**

The mechanism, and the invariants that make it work:

1. **The Shopify recurring price is fixed at $0.00.** Revenue is collected
   through Shopify **usage records** (`appUsageRecordCreate`) posted against a
   **usage line item** that carries a high `cappedAmount`. Because the recurring
   price never changes, no tier change ever requires re-confirmation.
2. **You own the billing clock.** The billing period
   (`currentPeriodStart`/`currentPeriodEnd`/`nextBillingDate`) lives in *your*
   database, not Shopify's subscription cycle. Shopify's recurring cycle is
   irrelevant and never read.
3. **You bill in advance (prepayment).** Each period is charged at its start.
   The first period is collapsed to "today" so the first charge fires on the
   next cron run, not deferred.
4. **Shopify gives you no billing webhook.** The only webhook is
   `APP_UNINSTALLED`. All period advancement and charge collection are driven by
   *your own* daily cron plus the one-time charge-approval return callback.
5. **One Shopify subscription per merchant, forever.** Tier changes reuse it.
   You only ever create a *new* Shopify subscription as a fallback when an
   in-place charge can't fit under the usage cap.

If you preserve these five invariants, everything else is bookkeeping.

---

## 1. Data model

There is no single "flex billing" table. Flex behavior is a set of flags on your
existing plan/subscription/charge tables, plus one append-only audit table.
Field names are illustrative — map them to your schema.

### 1.1 `plans`
| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `name` | string | |
| `amount` | decimal | The tier's monthly fee. **Billed as a usage record, not as recurring price.** |
| `currencyCode` | string | See §6 currency caveat — the usage rail is effectively USD. |
| `interval` | enum | `EVERY_30_DAYS` / `ANNUAL` / `QUARTERLY`. |
| `recurringInterval` + `recurringIntervalCount` | enum + int | The period length you advance by (e.g. `month` × 1). Normalize away from Shopify's day/30 default. |
| `flexBilling` | bool | Master toggle. **MUST be immutable after plan creation.** |
| `usageChargeCappedAmount` | decimal | The Shopify usage line-item spend ceiling. **MUST comfortably exceed the largest monthly fee + largest proration you will ever post**, because the recurring fee rides through this line. |
| `flexBillingTerms` | string | Customer-facing description shown on the usage line (`terms`). |
| `trialDays` | int | Managed locally; never sent to Shopify (see §5.3). |
| `chargeUsageDuringTrial` | bool | |
| `onUsageLimitReached` | enum | `none` / `upgrade`. `upgrade` enables auto-tiering. |
| `autoUpgradeToPlanId` | id? | The next-higher tier. **MUST itself be a flex plan.** |

**Per-tier usage band** (for automatic tiering). Either columns on `plans` or a
child `plan_usage_charges` row of `type = "unit_limits"`:
| Field | Notes |
|---|---|
| `limitMetric` | The metered metric that defines tier membership. |
| `limitMin` / `limitMax` | The usage band `[min, max]` for this tier. Crossing `limitMax` triggers auto-upgrade. |
| `usageLimitsPeriod` | Window the limit is measured over: `month_to_date` / `current_billing_period`. |

### 1.2 `subscriptions`
| Field | Notes |
|---|---|
| `planId` | |
| `activatedAt`, `canceledAt`, `frozenAt` | Lifecycle. The cron acts only on activated, non-frozen, non-canceled rows. |
| `pausedUntil` | **Hard gate.** When set, the main charge path MUST skip this subscription (a separate paused-sweep handles it). |
| `currentPeriodStart`, `currentPeriodEnd` | The period currently paid for. |
| `nextBillingDate` | When the next charge is due. The cron selects `nextBillingDate < now`. |
| `billingCycleAnchor` | Copied forward across tier changes so the billing date doesn't reset. |
| `shopifySubscriptionId` | The single Shopify `AppSubscription` GID. |
| `usageLineItemId` | Convenience pointer to the usage line item (see §1.3). |
| `triggeredByUsageChargeId` | Set when an auto-upgrade created this sub. Drives the `auto` vs `manual` analytics distinction. |
| `test` | `true` for Shopify test subscriptions. |
| trial fields | `trialStartedAt` / `trialEndsAt` etc., local only. |

### 1.3 `subscription_line_items` (the live usage cap)
| Field | Notes |
|---|---|
| `subscriptionId` | |
| `type` | `"usage"` (also `"subscription"` for the $0 recurring line). |
| `platformId` | **The Shopify usage line-item GID.** This is the `subscriptionLineItemId` in every `appUsageRecordCreate`. The single most important id to persist. |
| `cappedAmount` | Mirror of the Shopify cap. |
| `balanceUsed` | Mirror of Shopify's `AppUsagePricing.balanceUsed`, refreshed from each usage-record response. |
| `currentPeriodBilledSpend` | Running spend this period (default 0). |
| `spendPeriodStart` | |

Unique key: `(subscriptionId, type, key)`.

### 1.4 `charges`
One row per money movement (usage record or credit).
| Field | Notes |
|---|---|
| `amount` | Normalized amount (convert non-USD → USD if you mirror Mantle's behavior; keep raw too). |
| `chargedAmount` / `chargedCurrencyCode` | The raw amount/currency actually posted. |
| `platformId` | The Shopify usage-record or app-credit GID. |
| `isCredit` | `true` for downgrade credits. |
| `flexBilling` | `true` to separate flex charges from ordinary recurring charges in reporting **and** so the downgrade-credit cap can sum "what was collected this period". **MUST be set.** |
| `occurredAt` | Used by the downgrade-credit cap (charges since `currentPeriodStart`). |
| `status` | |

### 1.5 `flex_billing_events` (append-only audit — the only flex-specific table)
This is observability, not load-bearing billing. It gates no money movement.
Make it the most reliable write you have, but money correctness must not depend
on it.
| Field | Type | Semantics |
|---|---|---|
| `id` | uuid | |
| `subscriptionId` | id | **On upgrade/downgrade rows this is the _previous_ subscription's id** (the same id appears as both subscription and previousSubscription). |
| `previousSubscriptionId` | id? | `ON DELETE SET NULL`. |
| `organizationId` / tenant id | id | |
| `date` | datetime | |
| `type` | string | `subscription_charged` \| `upgraded` \| `downgraded`. (`subscribed` is *virtual* — reconstruct it, or store it; see §7.) |
| `amount` | decimal? | For `upgraded`/`downgraded`: the **new plan's full list price**. For `subscription_charged`: the **actual amount charged** (discounted price if a discount applied, not list price). |
| `currencyCode` | string? | |
| `interval` | string? | |
| `test` | bool? | |
| `proration` | bool | `netProRatedAmount != 0`. |
| `prorationAmount` | decimal? | Signed net delta. Positive on upgrade. On downgrade it is **rewritten to the positive actual credit** issued (or `0` if no credit). |
| `prorationAmountCurrency` | string? | |
| `prorationPlatformId` | string? | Shopify usage-record (upgrade) or app-credit (downgrade) GID. Null at create, filled when the call settles. |
| `prorationCompletedAt` | datetime? | Null → set when the proration call settles (drives a Pending→Completed UI). |
| `completedAt` | datetime? | |
| `minutesOnPlanBeforeChange` | int? | `now − previousSubscription.activatedAt` in minutes. Powers "time-to-upgrade" analytics. Null when `previousSubscription.activatedAt` is null; only set on upgrade/downgrade. |

---

## 2. Shopify Billing API surface

Five mutations. Four are Admin API; the credit is **Partner API**.

### 2.1 `appSubscriptionCreate` — one subscription, two line items
```graphql
mutation($name:String!, $test:Boolean, $lineItems:[AppSubscriptionLineItemInput!]!, $returnUrl:URL!, $trialDays:Int) {
  appSubscriptionCreate(name:$name, test:$test, lineItems:$lineItems, returnUrl:$returnUrl, trialDays:$trialDays) {
    appSubscription { id test lineItems { id plan { pricingDetails { __typename } } } trialDays currentPeriodEnd }
    confirmationUrl
    userErrors { field message }
  }
}
```
Line items:
- **Recurring (flex), priced at $0:**
  ```
  { plan: { appRecurringPricingDetails: { price: { amount: 0, currencyCode }, interval: "EVERY_30_DAYS" } } }
  ```
- **Usage, with the cap:**
  ```
  { plan: { appUsagePricingDetails: { terms: <flexBillingTerms>, cappedAmount: { amount: <usageChargeCappedAmount>, currencyCode } } } }
  ```
- **`trialDays` sent to Shopify MUST be 0** for flex — trials are local (§5.3).

Capture `appSubscription.id` **and the usage line item's GID** (the `lineItems[]`
entry whose `pricingDetails.__typename == "AppUsagePricing"`). Persist the usage
GID as `subscription_line_items.platformId`. Redirect the merchant to
`confirmationUrl`; they approve **once**, ever.

### 2.2 `appUsageRecordCreate` — the workhorse (recurring fee AND upgrade proration)
```graphql
mutation($description:String!, $price:Decimal!, $subscriptionLineItemId:ID!) {
  appUsageRecordCreate(
    description:$description,
    price:{ amount:$price, currencyCode:USD },   # currency hard-coded USD — see §6
    subscriptionLineItemId:$subscriptionLineItemId
  ) {
    appUsageRecord { id subscriptionLineItem { plan { pricingDetails {
      ... on AppUsagePricing { balanceUsed { amount currencyCode } cappedAmount { amount currencyCode } } } } } }
    userErrors { field message }
  }
}
```
- `subscriptionLineItemId` = `subscription_line_items.platformId`.
- `price` = `amount.toFixed(2)` (decimal string).
- **Treat these two `userErrors` as soft no-ops (return "no charge", do NOT throw):**
  - `"Total price exceeds balance remaining"` → cap is full.
  - `"Subscription is not active"`.
  - Any other userError → throw.
- Mirror the returned `balanceUsed`/`cappedAmount` back into your line-item row.

### 2.3 `appSubscriptionLineItemUpdate` — resize the cap
```
{ id: <usage line item GID>, cappedAmount: { amount, currencyCode } }
```
**Raising the cap returns a `confirmationUrl` requiring merchant re-approval.**
Lowering generally does not. Size the cap generously at create time to avoid
ever needing this on the hot path.

### 2.4 `appCreditCreate` — downgrade credits (PARTNER API)
This is the **Partner API** `appCreditCreate`, *not* the shop Admin API
`appCredit`. Variables: `amount { amount, currencyCode }`, `appId: ID!`,
`shopId: ID!` (use `gid://partners/Shop/<shopPlatformId>`), `description`,
`test`. Shopify credits are **irreversible once issued** — gate them behind an
app-level opt-out and the collected-this-period cap (§4.3).

### 2.5 `appSubscriptionCancel`
Used to cancel the prior subscription *locally only* during in-place tier
changes (you reuse the one Shopify subscription, so pass your equivalent of
`skipShopify`). Used for real only on actual cancellation.

### 2.6 Webhooks
Register **only `APP_UNINSTALLED`.** Do **not** rely on
`app_subscriptions/update` or any usage webhook — they won't drive your billing.
Period advancement = your cron. Charge approval = your `charge_return` callback.

---

## 3. Procedure: subscribe

```
function subscribe(merchant, plan):
    amount = plan.flexBilling ? 0 : plan.amount          # recurring price zeroed for flex
    lineItems = [
        recurringLine(price = {amount, currencyCode}, interval = EVERY_30_DAYS),
        usageLine(terms = plan.flexBillingTerms, cappedAmount = plan.usageChargeCappedAmount),
    ]
    res = appSubscriptionCreate(name, test, lineItems, returnUrl, trialDays = 0)   # trialDays MUST be 0
    persist:
        subscription.shopifySubscriptionId = res.appSubscription.id
        usageLineItem.platformId          = res.appSubscription.lineItems[usage].id
        usageLineItem.cappedAmount        = plan.usageChargeCappedAmount

    if plan.trialDays > 0:
        currentPeriodStart = today
        currentPeriodEnd   = nextBillingDate = today + trialDays      # first charge after trial
    else:
        # COLLAPSE the first period to today so the next cron bills in advance
        currentPeriodStart = currentPeriodEnd = nextBillingDate = startOfDay(today)

    return res.confirmationUrl       # merchant approves ONCE
```
- **MUST NOT** write a `flex_billing_event` for the initial subscribe (the
  `subscribed` event is virtual; see §7).
- **MUST NOT** charge during `appSubscriptionCreate`. The first charge is the
  next cron run. The activation is a $0 recurring line; the real first charge is
  prepayment for period 1, posted by §4.

---

## 4. Procedure: the daily charge cron

Run once per day (e.g. `0 0 12 * * *`).

### 4.1 Selector (build a partial index matching this exactly)
```sql
activatedAt IS NOT NULL AND frozenAt IS NULL AND canceledAt IS NULL
AND nextBillingDate < now()
AND plan.flexBilling = true
AND billingType = 'shopify'
AND appInstall.uninstalledAt IS NULL
AND app.scheduledForDeletionAt IS NULL AND app.removed = false
AND platform.enabled = true
```
Cursor-paginate (~200/page). Per-subscription errors **MUST** be caught and
reported without aborting the batch.

### 4.2 Per-subscription: `checkFlexBillingSubscription`
```
function checkFlexBillingSubscription(sub, ignoreBillingTime = false):
    if sub.pausedUntil is set:                       # BEFORE locking
        return { charged:false, reason:"paused" }    # paused sweep handles these

    unlock = mutexLock("flex:{org}:{sub}", timeout=120s, failAfter=180s)   # per-(org,sub)
    try:
        re-load sub; re-check active + flex          # state may have changed

        # billing-date gate — exact ts for the batch, day-level for plan-change path
        inFuture = ignoreBillingTime
            ? startOfDay(nextBillingDate) > startOfDay(now)
            : nextBillingDate > now
        if inFuture: return { charged:false, reason:"billing_date_in_future" }

        # compute next period (starts where current ended)
        nextPeriodStart = sub.currentPeriodEnd
        nextPeriodEnd   = sub.currentPeriodEnd + (plan.recurringIntervalCount × plan.recurringInterval)

        # amount, with conditional discount
        chargeAmount = plan.amount; currency = plan.currencyCode
        discount = activeDiscountFor(sub, nextPeriodStart)            # discountEndsAt >= nextPeriodStart or null
        if discount and discount.bundle?.discountMethod != "app_credits":
            chargeAmount = discount.priceAfterDiscount

        if chargeAmount > 0:
            createFlexBillingCharge(sub, amount=chargeAmount, currency,
                description="Subscription charge for period {start} to {end}",
                billingPeriodStart=nextPeriodStart)   # "prepayment for the next period"
        # else: $0 → skip Shopify entirely (zero_amount_rollover)

        # ADVANCE THE PERIOD UNCONDITIONALLY (even on $0, even if the charge was capped-out)
        update sub: nextBillingDate=nextPeriodEnd, currentPeriodStart=nextPeriodStart, currentPeriodEnd=nextPeriodEnd
        return { charged: chargeAmount > 0, reason: chargeAmount>0 ? "charge_created" : "zero_amount_rollover" }
    finally:
        unlock()
```

### 4.3 `createFlexBillingCharge`
```
function createFlexBillingCharge(sub, amount, currency, description, ...):
    usageLine = sub.lineItems where type=="usage"        # MUST have platformId, else throw
    res = appUsageRecordCreate(description, price=amount.toFixed(2), subscriptionLineItemId=usageLine.platformId)
    if res is a soft no-op (cap full / not active):
        return undefined                                  # NOTE: see the LOST-FEE warning in §5.1
    write charge row { amount, chargedAmount, platformId=res.id, isCredit:false, flexBilling:true, occurredAt:now }
    write flex_billing_event { type:"subscription_charged", amount, currencyCode:currency, proration:false, completedAt:now }
    refresh usageLine.balanceUsed/cappedAmount from res
    return charge
```

---

## 5. Procedure: in-place tier change (manual flex→flex)

Runs only when the *previous* active Shopify-charged subscription is **also**
flex. Does **not** recreate or re-confirm the Shopify subscription.

```
function changeTier(prevSub, newPlan):
    # 1. collect any outstanding charge first, so a charge isn't lost on/near billing day
    collectOutstandingFlexBillingCharges(prevSub)          # day-level gate; see §5.2
    reload prevSub                                          # pick up advanced period dates

    # 2. day-based proration over the CURRENT cycle
    daysInCycle   = days(prevSub.currentPeriodEnd) − days(prevSub.currentPeriodStart)
    if daysInCycle <= 0:
        # Degenerate/collapsed cycle — e.g. a tier change on the SAME day as signup, before the first
        # charge has run (the period is collapsed to one day at signup, §3). The collect-outstanding step
        # above normally advances the period to a full interval first; this guard makes the division safe
        # either way. A zero-length cycle has nothing to pro-rate.
        netProRated = 0
    else:
        daysRemaining = clamp(days(prevSub.currentPeriodEnd) − days(today), 0, daysInCycle)
        prevAmount    = prevDiscount ? prevDiscount.priceAfterDiscount : prevSub.plan.amount
        newAmount     = newDiscount  ? newDiscount.priceAfterDiscount  : newPlan.amount
        netProRated   = (newAmount/daysInCycle − prevAmount/daysInCycle) × daysRemaining   # >0 charge, <0 credit

    # 3. write the event NOW (direction by LIST PRICE, independent of proration sign)
    type = newPlan.amount > prevSub.plan.amount ? "upgraded" : "downgraded"
    event = createFlexBillingEvent {
        type, subscriptionId: prevSub.id, previousSubscriptionId: prevSub.id,
        amount: newPlan.amount, proration: netProRated != 0, prorationAmount: netProRated,
        prorationAmountCurrency: "USD", minutesOnPlanBeforeChange: minutes(now − prevSub.activatedAt),
        prorationPlatformId: null, prorationCompletedAt: null,    # filled below when it settles
    }

    # Default: stay in place. ONLY a cap-blocked upgrade flips this to false (→ confirmable fallback).
    # Trial, equal-price (netProRated == 0) and downgrade all stay in place with no money fallback.
    continueWithFlexBilling = true
    if not inTrial(prevSub):                  # no proration money moves while in trial; event row still written
        if netProRated > 0:
            continueWithFlexBilling = upgradeCharge(prevSub, newPlan, netProRated, event)   # §5.a; false → fallback
        else if netProRated < 0:
            downgradeCredit(prevSub, newPlan, -netProRated, event)                          # §5.b
        # netProRated == 0 → equal-priced swap: no charge, no credit, commit in place

    if continueWithFlexBilling:
        commitInPlace(prevSub, newPlan, event)        # §5.c
        return newSubscription
    else:
        return createConfirmableSubscription(newPlan) # fallback: real appSubscriptionCreate, merchant re-approves
```

### 5.a Upgrade charge — gated by remaining cap balance
```
function upgradeCharge(prevSub, newPlan, net, event) -> continueWithFlexBilling:
    refetch live balanceUsed/cappedAmount from Shopify
    balanceRemaining = cappedAmount − balanceUsed
    hasEnoughBalance = cappedAmount >= newPlan.amount AND balanceRemaining >= net
    if not hasEnoughBalance: return false                     # → confirmable-subscription fallback
    res = appUsageRecordCreate("Pro-rated charge for subscription upgrade", net.toFixed(2), usageLine.platformId)
    if res is soft no-op (cap raced full): return false        # → fallback
    event.prorationPlatformId = res.id; event.prorationCompletedAt = now
    write charge row { flexBilling:true, isCredit:false, platformId:res.id }
    return true
```

### 5.b Downgrade credit — Partner API, capped at what was collected
```
function downgradeCredit(prevSub, newPlan, theoretical, event):
    if app.disableDowngradeCredits:
        event.prorationAmount = 0; event.prorationCompletedAt = now; return    # opt-out (credits irreversible)
    collectedThisPeriod = sum(charges where flexBilling AND not isCredit AND occurredAt >= prevSub.currentPeriodStart)
    actualCredit = min(theoretical, collectedThisPeriod)        # cannot credit more than was collected
    if actualCredit > 0:
        res = appCreditCreate(amount=actualCredit.toFixed(2), currencyCode="USD",
                              appId=<platformId>, shopId="gid://partners/Shop/<platformId>", description, test)
        write charge row { isCredit:true, flexBilling:true, platformId:res.id }
        event.prorationPlatformId = res.id; event.prorationAmount = actualCredit   # POSITIVE actual credit
        event.prorationCompletedAt = now
    else:
        event.prorationAmount = 0; event.prorationCompletedAt = now               # nothing collected yet (e.g. day-1)
```

### 5.c Commit in place
```
function commitInPlace(prevSub, newPlan, event):
    cancel prevSub locally (skipShopify = true)                 # reuse the one Shopify subscription
    create newSub on newPlan, copying forward from prevSub:
        shopifySubscriptionId, usageLineItem(platformId + key), billingCycleAnchor,
        nextBillingDate, currentPeriodStart, currentPeriodEnd, trial fields
    transfer eligible discounts
    emit Upgraded/Downgraded app event (netChange = newPlan.amount − prevSub.plan.amount)
    event.completedAt = now
    # next cron charges newPlan.amount
```

---

## 6. Procedure: automatic, usage-driven upgrade

When a metered event makes a `unit_limits` charge's limit-metric value cross
`limitMax`, upgrade to the next tier.

```
on usage ingest:
    # The band is INCLUSIVE of limitMax: usage <= limitMax still belongs to this tier; you upgrade only
    # once it is STRICTLY exceeded. Use the SAME boundary (>) here and in the job below, so hitting exactly
    # limitMax never enqueues a job that then declines to upgrade.
    if (cachedLimitMetricValue + thisEvent) > charge.limitMax:
        enqueue CheckAutoUpgradeJob(jobId = "CheckAutoUpgrade:{appInstall}", delay = 60s)   # fixed jobId dedupes

function checkBillingAutoUpgrade(appInstall, cascadeDepth = 0):
    load active sub, its plan.autoUpgradeToPlan (MUST be flex), plan usage charges
    bail unless: current plan flex AND target exists AND target flex AND has usage charges
    for each unit_limits charge:
        value = limitMetric over (usageLimitsPeriod == "current_billing_period" ? period : month_to_date)
        if value > charge.limitMax:
            if a non-canceled subscription already on the target plan exists: bail   # pending-upgrade dedupe (loop guard)
            createSubscription(planIds=[autoUpgradeToPlanId], triggeredByUsageChargeId=charge.id, carryOverDiscount, ...)
            # ^ routes through the SAME flex branch as §5 — so an auto-upgrade IS pro-rated identically
            if upgraded and cascadeDepth < 10:
                enqueue CheckAutoUpgradeJob(jobId includes timestamp, cascadeDepth+1)   # cascade NOT deduped
```
**Key facts:**
- An auto-upgrade is **pro-rated exactly like a manual change** — it calls the
  same `changeTier`/in-place path. There is no "skip proration for auto" branch.
- The auto path additionally **copies the billing cycle forward** (the billing
  date does not reset) and tags the new subscription with
  `triggeredByUsageChargeId` (this is the only thing that distinguishes `auto`
  from `manual` in analytics).
- **Loop guards are mandatory:** dedupe against an existing target-plan
  subscription, and cap cascade depth (Mantle uses 10). Swallow "Subscription is
  not active" during cascades.

---

## 7. MUST / MUST NOT rules (the gotchas)

These each encode a real bug or a hard lesson. Treat them as acceptance
criteria.

- **MUST bill the first period in advance.** No trial → collapse the first
  period to today and set `nextBillingDate = today`, so the next cron charges
  period 1 as prepayment. There is no free first cycle.
- **MUST skip `pausedUntil` subscriptions in the main charge path**, and run a
  *separate* paused-subscription sweep. You need both.
- **MUST advance the period on $0 charges** without posting a usage record
  (Shopify rejects $0 records). Just move the clock (`zero_amount_rollover`).
- **⚠️ Decide your cap-blocked-fee policy deliberately.** In Mantle, when the
  usage cap is full at charge time, `appUsageRecordCreate` returns "Total price
  exceeds balance remaining", the charge helper swallows it and returns nothing
  — **but the period is advanced unconditionally** (the success flag is a
  *price* test `amount > 0`, not a *success* test, and the return value is never
  inspected). **Net effect: that month's fee is silently lost and the clock
  still moves forward.** If you don't want to inherit this bug, inspect the
  charge result and do NOT advance the period (or retry / raise the cap) when
  the post failed. Note `rollOverPendingUsage` does **not** govern this path — it
  only affects the separate metered-overage rollup.
- **MUST size the cap to never fill** under normal operation (fee + proration
  headroom), precisely because of the above.
- **MUST collect outstanding charges before an in-place tier change**, using a
  **day-level** comparison (more aggressive than the cron's exact-timestamp
  gate). Otherwise a same-day plan change before the cron loses that period's
  charge. May charge a few hours early; never loses the cycle.
- **MUST cap downgrade credits** at the sum of flex (non-credit) charges
  collected since `currentPeriodStart`. A day-1 downgrade before any charge
  issues **$0** credit.
- **SHOULD offer a per-app downgrade-credit opt-out** (`disableDowngradeCredits`)
  — Shopify credits are irreversible.
- **MUST fall back to a confirmable subscription** when an upgrade proration
  can't fit under the cap, rather than dropping the change.
- **MUST keep trials local.** Send `trialDays: 0` to Shopify; track the trial in
  your DB; set `nextBillingDate = trialEnd` so the first charge fires
  post-trial. Clear trial fields on conversion to prevent re-trial abuse. No
  proration money moves while in trial (but still write the event row).
- **MUST lock per (org, subscription)** (short timeout, e.g. 2 min) so the cron
  and a simultaneous manual plan change can't double-charge.
- **MUST label direction by list price**, not proration sign:
  `newPlan.amount > prevPlan.amount ? upgraded : downgraded`. Equal-priced swaps
  can be labeled `downgraded` with a `$0` proration — that's expected.
- **MUST swallow "private/dev app can't use Billing API"** ("Apps without a
  public distribution cannot use the Billing API") per-subscription rather than
  failing the batch.
- **Currency is effectively USD on the usage rail** — see §8. Either constrain
  flex plans to USD or fix the FX before posting.
- **`subscribed` events are virtual.** Mantle never stores them; it reconstructs
  them at read time from a search index of `flexBilling = true AND activatedAt`
  subscriptions. **Simpler: just store a `subscribed` event** when a flex
  subscription activates.
- **MUST treat the audit table as non-load-bearing.** Best-effort mirror to any
  analytics store; never let an analytics write failure block or corrupt the
  real charge.

---

## 8. Currency caveat (read before going multi-currency)

In Mantle the entire flex charge rail is **hard-coded to USD**:
- `appUsageRecordCreate` posts `currencyCode: USD`.
- Upgrade proration charge and downgrade `appCreditCreate` use `USD`.
- The audit event's `prorationAmountCurrency` is `"USD"`.

Meanwhile the usage line-item **cap** is created in the plan's currency. So a
non-USD "€50" plan posts a **$50 USD** usage charge against a €-denominated cap,
with no FX — a latent bug for non-USD merchants. For a correct multi-currency
implementation: either (a) restrict flex plans to USD, or (b) align the usage
record currency, the proration/credit currency, and the cap currency, applying
FX consistently. Do not copy the hard-coded `USD` blindly.

---

## 9. Acceptance scenarios

Implement and verify each:

1. **First charge.** Subscribe (no trial) → next cron run posts one usage record
   for `plan.amount` and advances the period by one interval. No charge during
   `appSubscriptionCreate`.
2. **Steady state.** Each cron run on/after `nextBillingDate` posts exactly one
   usage record and advances the period once. Idempotent under the per-sub lock.
3. **$0 plan / 100%-off discount.** Period advances; no usage record posted; no
   charge/audit row.
4. **Mid-cycle upgrade.** Day-based proration charge posted (gated by cap),
   `upgraded` event with `prorationPlatformId` set, billing date unchanged, next
   cron charges the new amount.
5. **Cap-full upgrade.** Proration can't fit → fall back to a confirmable
   subscription (merchant gets a `confirmationUrl`).
6. **Mid-cycle downgrade.** Partner-API credit = min(theoretical, collected this
   period); `downgraded` event with positive `prorationAmount` = actual credit.
7. **Day-1 downgrade (before any charge).** $0 credit; event `prorationAmount` =
   0.
8. **Downgrade with `disableDowngradeCredits`.** No credit; `downgraded` event
   with `prorationAmount` = 0.
9. **Auto-upgrade.** Crossing `limitMax` enqueues the job, upgrades to the next
   tier with proration, tags `triggeredByUsageChargeId`, copies the cycle
   forward, and does not loop (dedupe + cascade cap).
10. **Same-day change before cron.** Outstanding charge is collected first (day-
    level), so the period's fee isn't lost.
11. **Paused subscription.** Skipped by the main cron; handled by the paused
    sweep.
12. **Trial.** No Shopify trial; first charge fires at trial end; no proration
    money while in trial.
```
