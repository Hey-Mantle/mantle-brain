# Shopify Billing Migration — Implementation Spec (for agents)

> **Audience:** a coding agent implementing **standard Shopify-managed app
> billing** in an arbitrary app — the architecture where Shopify owns the
> billing clock and collects charges, and the app keeps a reconciled local
> mirror — plus the **migration** of merchants onto that rail, **per billing
> model**, in the correct **order of operations**. A human-readable walkthrough
> of the same system lives in [`README.md`](./README.md).
>
> **Out of scope:** *flex billing* (the `$0`-recurring + usage-record technique
> for re-approval-free in-place tier changes). That is a separate system with
> its own spec — see [`../flex-billing/implementation-spec.md`](../flex-billing/implementation-spec.md).
> Where the two interact, this spec points at it but does not duplicate it.
>
> **Provenance:** distilled from the production behavior of Mantle's Shopify
> billing wrapper. Stack-agnostic; map the generic table/field names below onto
> your own ORM. Where a rule encodes a hard-won lesson it is marked **MUST** /
> **MUST NOT**.

---

## 0. Goal & invariants

Build app subscriptions where **Shopify is the payment rail and the system of
record for approval and collection**, while **your database is the system of
record for the business agreement** (plan catalog, trials, discounts, metered
metrics, analytics). Then be able to **migrate** a merchant onto this rail —
from no subscription, from a different plan, or from a different biller — without
double-charging, dropping a cycle, or leaving the two systems disagreeing.

The invariants that make it work:

1. **Shopify owns the billing clock.** You do **NOT** run a charging cron for
   standard billing. Shopify collects the recurring charge on its own cycle and
   emits charge events. Your local period dates are a **mirror**, refreshed from
   Shopify — never authoritative. *(This is the single biggest difference from
   flex billing, which inverts it.)*
2. **The local subscription mirrors exactly one Shopify `AppSubscription`.** It
   carries a pointer to the Shopify charge GID and Shopify's status. You never
   invent charge state locally; you mirror it.
3. **Approval is verified, not assumed.** A subscription becomes active locally
   only after you **re-query Shopify** and confirm the charge `status == active`.
   The `returnUrl` redirect is a trigger, not proof.
4. **Exactly one active subscription per (app install).** When a new charge
   verifies active, all other active subscriptions for that install are canceled
   **in the same transaction**.
5. **New before old; verify before activate; flip the rail last.** Every
   migration creates and confirms the replacement before tearing down what it
   replaces, and only switches the merchant's default billing rail once
   everything is live.
6. **Shopify gives you almost no webhooks.** Register only `APP_UNINSTALLED`.
   All cycle-rollover tracking, orphan repair, and charge backfill are driven by
   **your scheduled reconciliation** over the Partner API, which must be
   idempotent.

If you preserve these six invariants, everything else is bookkeeping.

---

## 1. Data model

Field names are illustrative — map them to your schema. There is no single
"migration" table; migration is a *procedure* over the tables below.

### 1.1 `plans`
| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `name` | string | |
| `amount` | decimal | Recurring price. `0` ⇒ free plan (see §5.6). |
| `currencyCode` | string | |
| `interval` | enum | `EVERY_30_DAYS` / `ANNUAL`. **MUST be sent to Shopify verbatim** — no normalization on Shopify's side. |
| `trialDays` | int? | For standard billing this is sent to Shopify (Shopify runs the trial). |
| `billingType` | enum (default `shopify`) | The rail: `shopify` / `stripe` / `apple` / `manual` / `test`. |
| usage charges | relation | Zero or more metered components (per-unit price + monthly `cappedAmount` + `terms`). Presence ⇒ a usage line item. |
| add-ons / features | relation | Your catalog concepts; Shopify never sees them directly. |

### 1.2 `subscriptions`
| Field | Notes |
|---|---|
| `planId` | |
| `billingType` | Mirrors the plan's rail. Routes create/cancel/sync to the right provider contract. |
| `active` | `true` only when verified active on Shopify. |
| `activatedAt` | Set at verified activation. Revenue-recognition moment. |
| `canceledAt`, `cancelOn`, `cancelAtPeriodEnd` | Cancellation state. **MUST NOT** be clobbered by reconciliation if already set. |
| `frozenAt` | Mirror of Shopify "frozen" (e.g. payment failure). |
| `currentPeriodStart`, `currentPeriodEnd`, `nextBillingDate` | **Mirror of Shopify's cycle**, refreshed by sync. Not authoritative. |
| `billingCycleAnchor` | Mirror; copied forward across plan changes so the date doesn't reset. |
| `trialStartsAt`, `trialExpiresAt` | Local trial window (computed from plan/trial). For standard billing, expected to roughly track Shopify's `trialDays`. |
| `shopifyAppChargeId` | **Unique.** FK to the charge mirror (§1.4) — the pin tying this row to Shopify. |
| `appInstallationId` | The (app, shop) install. Single-active invariant is scoped to this. |
| `confirmationUrl` | The Shopify approval URL, stored at create. Immutable per charge. |
| `test` | `true` for Shopify test charges. |

### 1.3 `subscription_line_items`
One row per priced component.
| Field | Notes |
|---|---|
| `subscriptionId` | |
| `type` | `subscription` (recurring) / `usage` (metered) / `addon`. |
| `key` | Stable key for matching the same line across updates (`base`, `overage`, …). |
| `platformId` | **The Shopify line-item GID.** Required for `appUsageRecordCreate` and cap resizes. **MUST persist for usage lines.** |
| `amount`, `currencyCode` | Computed price at create time (may differ from plan if discounted). |
| `cappedAmount` | Usage lines: the Shopify cap. Mirror. |
| `balanceUsed` | Usage lines: Shopify's consumed balance. Mirror, refreshed from each usage-record response and from sync. |

Unique key: `(subscriptionId, type, key)`.

### 1.4 `shopify_app_charge` (the charge mirror)
Mirrors one Shopify `AppSubscription` (or one-time purchase) charge object.
| Field | Notes |
|---|---|
| `chargeId` | **The Shopify charge GID.** |
| `name`, `amount`, `currencyCode` | As submitted to Shopify. |
| `billingOn` | Shopify's next-billing date. |
| `status` | `pending` / `active` / `declined` / `frozen` / `expired` / `cancelled`. **Synced from Shopify; never invented.** |
| `subscriptionId` | One-to-one back to the local subscription. |

### 1.5 `charges`
One row per money movement Shopify reports (recurring charge, usage charge,
one-time, credit). Keep both the **raw** charged amount/currency and a
**normalized** amount for reporting.
| Field | Notes |
|---|---|
| `subscriptionId` | Null for one-time purchases. |
| `type` | `subscription` / `usage` / `one_time` / `credit`. |
| `amount` / `currencyCode` | Normalized. |
| `chargedAmount` / `chargedCurrencyCode` | Raw, as Shopify charged. |
| `platformId` | Shopify charge/transaction GID. |
| `status`, `occurredAt` | |

> There is **no flex-billing audit table here** — that belongs to the flex
> topic. Standard billing's "audit" is the charge/transaction stream Shopify
> reports, mirrored into `charges`.

---

## 2. Shopify Billing API surface

| Operation | Used for | Key inputs / outputs |
|---|---|---|
| `appSubscriptionCreate` | New subscription (any recurring model) | `name`, `lineItems[]`, `trialDays`, `returnUrl`, `test`. Returns `appSubscription { id, lineItems { id, plan { pricingDetails { __typename } } } }` + `confirmationUrl` + `userErrors`. |
| `appPurchaseOneTimeCreate` | One-time charge | `name`, `price`, `returnUrl`, `test`. Returns its own `confirmationUrl`. No trial/discount/recurring. |
| `appUsageRecordCreate` | Metered charge | `subscriptionLineItemId` (the usage GID), `price`, `description`. Rejected when it would exceed the cap. |
| `appSubscriptionLineItemUpdate` | Resize a usage cap | `{ id, cappedAmount }`. **Raising the cap returns a `confirmationUrl` (re-approval).** |
| `appSubscriptionCancel` | Real cancellation | `{ id }`. Tolerate 401/404 (shop uninstalled). |
| `appSubscriptionTrialExtend` | Extend a trial | `{ id, days }`. Optional. |
| **App-install query** | **Verify** on return + reconcile | Returns the install's `activeSubscriptions { status, currentPeriodEnd, lineItems { id, plan { pricingDetails { … balanceUsed cappedAmount } } } }`. Ground truth. |

### 2.1 Line items
- **Recurring:**
  ```
  { plan: { appRecurringPricingDetails: {
      price: { amount, currencyCode }, interval,    # EVERY_30_DAYS | ANNUAL
      discount?: { value: { amount | percentage }, durationLimitInIntervals? }
  } } }
  ```
- **Usage:**
  ```
  { plan: { appUsagePricingDetails: { terms, cappedAmount: { amount, currencyCode } } } }
  ```
- After create, read back each `lineItems[i].id` and classify by
  `plan.pricingDetails.__typename` (`AppRecurringPricing` vs `AppUsagePricing`).
  **MUST** persist the usage line's GID to `subscription_line_items.platformId`.

### 2.2 Webhooks
Register **only `APP_UNINSTALLED`**, **idempotently** (Shopify rejects a
duplicate with "Address for this topic has already been taken" — catch and
ignore). **MUST NOT** rely on `app_subscriptions/update` or any per-cycle
webhook to drive billing.

---

## 3. Procedure: subscribe (create + confirm)

```
function subscribe(merchant, plan, opts):
    # 1. Free, no usage → no Shopify object at all
    if plan.amount == 0 and plan has no usage charges:
        sub = createLocalSubscription(plan, active=true, activatedAt=now)   # §5.6
        return { activatedImmediately: true, subscription: sub }

    # 2. Create the LOCAL row first, pending
    sub = createLocalSubscription(plan, active=false)        # handle to track the flow

    # 3. Derive line items from the plan (§5 per model)
    lineItems = buildLineItems(plan, discount = activeDiscountFor(merchant, plan))

    # 4. Push to Shopify
    res = appSubscriptionCreate(name, test, lineItems, returnUrl, trialDays = plan.trialDays ?? 0)
    if res.userErrors: handle (see §3.1)

    # 5. Persist the pins
    sub.shopifySubscriptionId          = res.appSubscription.id
    chargeMirror = createChargeMirror({ chargeId: res.appSubscription.id, status: "pending", ... })
    sub.shopifyAppChargeId             = chargeMirror.id
    sub.confirmationUrl                = res.confirmationUrl
    for li in res.appSubscription.lineItems:
        persist subscription_line_items { platformId: li.id, type: classify(li), ... }

    # 6. Send the merchant to approve — ONCE
    return { confirmationUrl: res.confirmationUrl, subscription: sub }
```
- **MUST NOT** mark `active` here. Activation only happens after verification
  (§4).
- **MUST NOT** charge here. Shopify collects after approval, on its own cycle.
- For one-time purchases use `appPurchaseOneTimeCreate` instead (§5.5).

### 3.1 `appSubscriptionCreate` userErrors
- A Shopify subscription is **unique per shop intent**; a duplicate create for
  the same pending intent errors — detect and reuse the existing pending row /
  `confirmationUrl` instead of creating a second.
- "Apps without a public distribution cannot use the Billing API" — private/dev
  apps can't bill. Surface this clearly; don't silently produce a broken row.

---

## 4. Procedure: order of operations — the activation round-trip

This is the load-bearing sequence. Order is not stylistic; each step guards a
money bug.

```
# Shopify redirects the merchant to your returnUrl after they approve.
function onChargeReturn(subscriptionId):                 # the returnUrl handler
    sub = load(subscriptionId)
    lock = mutexLock("activate:{install}:{sub}", short timeout)   # idempotency / anti-race
    try:
        # STEP 1 — VERIFY against Shopify (do NOT trust the redirect)
        install = queryShopifyAppInstall(sub.appInstallationId)
        live = install.activeSubscriptions.find(s => s.id == sub.shopifySubscriptionId)
        if not live or live.status != "active":
            enqueue ReconcileJob(sub)            # heal asynchronously
            return redirectToApp(returnUrl)      # still return the merchant; don't hard-error

        # STEP 2 — MIRROR Shopify's truth into the local row
        sub.currentPeriodStart = live.currentPeriodStart
        sub.currentPeriodEnd   = live.currentPeriodEnd
        sub.billingCycleAnchor = live.currentPeriodEnd    # or Shopify's anchor
        for li in live.lineItems:
            updateLineItem(sub, li.id, cappedAmount = li.cappedAmount, balanceUsed = li.balanceUsed)
        chargeMirror.status = "active"; chargeMirror.billingOn = live.currentPeriodEnd

        # STEP 3 — ATOMIC: activate self AND cancel every OTHER active sub for this install
        transaction:
            for other in activeSubscriptions(sub.appInstallationId) where other.id != sub.id:
                other.active = false; other.canceledAt = now    # local cancel; new one supersedes
            sub.active = true; sub.activatedAt = now; sub.canceledAt = null; sub.frozenAt = null

        # STEP 4 — side effects (best-effort, after the money-true transaction)
        registerWebhooksIfNeeded(sub.appInstallationId)   # APP_UNINSTALLED, idempotent
        bustCaches(sub.appInstallationId)
        enqueue ReconcileJob(sub, delaySeconds = 20)      # let webhooks/transactions land first
        return redirectToApp(returnUrl)
    finally:
        unlock()
```

**Why each step is ordered this way:**
- **Verify before activate** — the return redirect fires even if the merchant
  declined or the charge is still pending. Activating on the redirect alone
  recognizes revenue that doesn't exist.
- **Mirror before activate** — you want Shopify's real period dates and line
  GIDs on the row the instant it goes active, so usage records and cycle sync
  work immediately.
- **Cancel-others atomic with activate-self** — split them and a crash between
  leaves either two active subs (double charge) or zero (gap). One transaction
  closes that window.
- **Side effects after the transaction** — webhook registration and cache busts
  must never be able to roll back, or block, the money-true state change.
- **Don't hard-error on verify failure** — Shopify lag is common; return the
  merchant to the app and let idempotent reconciliation finish the job.

---

## 5. Procedure: migration by billing model

What changes per model is **`buildLineItems`** and the edge cases. Build only the
models you sell. Flex billing is **out of scope** (see the sibling spec).

### 5.1 Flat-rate recurring
```
buildLineItems = [ recurringLine(price = plan.amount, interval = plan.interval, discount?) ]
```
- `trialDays = plan.trialDays` → Shopify runs the trial.
- Discount attaches to the recurring line (`value` amount|percentage +
  `durationLimitInIntervals`).
- **MUST** send `interval` matching the plan exactly.

### 5.2 Annual vs 30-day
Identical to §5.1 with `interval = ANNUAL`. **MUST** have a reconciliation check
comparing local interval vs Shopify's — a 30-day created where annual was meant
under-bills 12×.

### 5.3 Usage-based / metered
```
buildLineItems = [
   (plan.amount > 0 ? recurringLine(...) : nothing),
   usageLine(terms = join(plan.usageCharges.terms), cappedAmount = plan.usageCap or sum(usageCharges.cappedAmount)),
]
```
- Create establishes only the **cap**. Bill actual usage afterward via
  `appUsageRecordCreate` against the persisted usage GID.
- **MUST** mirror Shopify's returned `balanceUsed`/`cappedAmount` after each
  record.
- **MUST** decide a ceiling policy: Shopify rejects records that exceed the cap.
  Options: reject / re-approve a higher cap (`appSubscriptionLineItemUpdate` →
  `confirmationUrl`) / auto-upgrade to a larger plan.
- **SHOULD** size the cap with generous headroom so the hot path never re-approves.

### 5.4 Hybrid (recurring + usage)
Both line items in one `appSubscriptionCreate`. **Discount applies to the
recurring line only.** Decide pending-usage behavior across plan changes (roll
over vs reset) and document it.

### 5.5 One-time purchase
Use `appPurchaseOneTimeCreate` (not `appSubscriptionCreate`). Returns its own
`confirmationUrl`; confirmation lands via webhook. **No trial, no discount, no
recurring cycle.** Track as a standalone `charges` row of `type = one_time`;
**MUST NOT** put it in the subscription mirror.

### 5.6 Free / `$0` plans
- No recurring price **and** no usage line ⇒ **MUST NOT** call Shopify; activate
  the local subscription immediately, no confirmation round-trip.
- `$0` base **with** a usage line ⇒ still create a Shopify subscription carrying
  just the usage line.
- **MUST** teach reconciliation that a `$0`-no-usage active local subscription
  has **no** Shopify object and is not a mismatch.

### 5.7 Provider migration (Stripe / manual / Apple → Shopify)
Same shape as a plan change, across rails:
```
1. Create the Shopify subscription (§3) and confirm it active (§4).   # NEW first
2. Only then cancel the old-rail subscription (Stripe sub / manual record).   # OLD after
3. Only then flip the merchant's default billingType to "shopify".    # rail LAST
```
- **MUST NOT** cancel the paying old-rail subscription before the Shopify one is
  verified active — that loses the cycle and the service.
- **MUST** flip `billingType` last, after every subscription is live, or new
  operations route to a rail that isn't ready.
- If the old rail had un-billed usage, **collect it before canceling** (§6).

---

## 6. Procedure: cancellation & replacement (no-gap ordering)

### 6.1 Plan change (Shopify → Shopify)
```
function changePlan(prevSub, newPlan):
    # flex→flex in-place changes are handled by the FLEX spec, not here.
    newSub = subscribe(merchant, newPlan)        # §3 — creates pending + confirmationUrl
    # merchant approves → onChargeReturn (§4) verifies the new sub AND cancels prevSub atomically
    return newSub.confirmationUrl
```
The single-active invariant in §4 step 3 does the old-sub cancellation **only
after** the new one verifies. There is intentionally **no** "cancel prev first"
branch for standard billing.

### 6.2 Real cancellation
```
function cancel(sub, { immediate }):
    collectOutstanding(sub)                       # §6.3, mostly metered
    if sub.shopifyAppChargeId:
        try appSubscriptionCancel(sub.shopifySubscriptionId)
        catch 401/404: pass                       # shop already uninstalled
    if not immediate and app.cancelAtPeriodEnd:
        sub.cancelAtPeriodEnd = true; sub.cancelOn = sub.currentPeriodEnd   # keep service till paid-through
    else:
        sub.active = false; sub.canceledAt = now
    enqueue ReconcileJob(sub)
```

### 6.3 Collect-before-cancel
If the subscription has un-billed metered usage, post it (`appUsageRecordCreate`)
**before** marking canceled. Otherwise the final period's usage revenue is lost.
(For flex billing this is a hard requirement with extra rules — see that spec.)

---

## 7. Procedure: sync & reconciliation (the mirror heals itself)

Standard billing has **no per-cycle webhook**, so a scheduled job is the only
thing keeping the mirror correct. **MUST** be idempotent.

```
function reconcile(install):
    live = queryShopifyAppInstall(install)        # ground truth

    # 7.1 Period-clock sync — Shopify advanced the cycle; mirror it
    for sub in activeLocalSubs(install):
        s = live.activeSubscriptions.find(matching sub)
        if s: sub.currentPeriodStart/End, nextBillingDate, billingOn = s.<dates>

    # 7.2 Orphan repair — Shopify says gone, mirror says active (dropped webhook)
    for sub in activeLocalSubs(install):
        s = live.activeSubscriptions.find(matching sub)
        if not s or s.status in {cancelled, frozen, expired, declined}:
            sub.active = false
            sub.frozenAt = (s.status == frozen ? now : sub.frozenAt)
            if not sub.canceledAt and s.status in {cancelled, expired, declined}: sub.canceledAt = now
            # MUST NOT overwrite an already-set canceledAt

    # 7.3 Charge / transaction backfill — pull Shopify's charges into `charges`
    for t in shopifyTransactionsSince(install, lastSync):
        upsert charges { platformId: t.id, ... }   # idempotent on platformId

    # 7.4 Divergence report — local vs Shopify mismatch (amount/interval/trial)
    for sub in activeLocalSubs(install):
        compare(sub.amount, sub.interval, sub.trialExpiresAt) vs Shopify
        if mismatch beyond tolerance: report (do NOT silently auto-"fix" amounts)
```

- **Webhook registration** (`APP_UNINSTALLED`) belongs here too — register once
  per install, idempotently.
- **Abandoned-confirmation cleanup:** a separate periodic sweep cancels local
  subscriptions stuck `active=false` with a `confirmationUrl` older than N days
  (Shopify never times these out).
- **Single source of truth:** when in doubt, Shopify wins for *status / dates /
  balance*; you win for *plan / discount / trial intent*. Reconciliation reports
  catalog divergence rather than overwriting your intent.

---

## 8. MUST / MUST NOT rules (the gotchas)

Each encodes a real bug. Treat as acceptance criteria.

- **MUST NOT run a recurring charging cron** for standard billing. Shopify
  collects. If you need to drive charges yourself, you're doing flex billing —
  use that spec.
- **MUST verify against Shopify before activating.** The `returnUrl` redirect is
  a trigger, not proof; re-query and require `status == active`.
- **MUST activate-self and cancel-others in one transaction**, after
  verification — never zero or two active subscriptions per install.
- **MUST create the new subscription before canceling the old**, within Shopify
  and across rails. Never cancel a collecting subscription before its
  replacement verifies.
- **MUST flip `billingType` last** in a provider migration.
- **MUST mirror, never invent, charge state and period dates.** Pull them from
  Shopify; don't compute "next billing date" yourself for standard billing.
- **MUST sync the period clock on a schedule** — Shopify doesn't webhook cycle
  rollovers.
- **MUST size usage caps with headroom**; **raising a cap re-approves**
  (`appSubscriptionLineItemUpdate` → `confirmationUrl`).
- **MUST decide a cap-ceiling policy** (reject / re-approve / auto-upgrade);
  Shopify rejects over-cap usage records — don't silently drop them.
- **MUST NOT create a Shopify object for `$0`-no-usage plans**; activate locally
  and teach reconciliation it's expected.
- **MUST repair orphaned states** without clobbering an already-set
  `canceledAt`.
- **MUST clean up abandoned pending subscriptions** with your own sweep —
  Shopify never expires a `confirmationUrl`.
- **MUST make activation, cancellation, webhook registration, and sync
  idempotent** — they all re-run.
- **MUST tolerate "already uninstalled" (401/404) on cancel** per-subscription
  rather than failing the batch.
- **MUST swallow "private/dev app can't use the Billing API"** clearly at create
  time.
- **MUST keep raw + normalized amounts** for multi-currency reporting; don't
  assume the usage-record currency equals the cap currency (see §9).
- **MUST treat the charge/transaction mirror as eventually consistent** — it
  heals via reconciliation, not via trusting a single webhook.

---

## 9. Currency caveat

Keep both the **raw** charged amount/currency (as Shopify charged) and a
**normalized** amount for reporting. Be aware that some rails are effectively
single-currency: in particular the **usage-record rail can be hard-coded to a
single currency** even when the plan/cap is denominated otherwise (see the flex
spec's currency section for the canonical version of this trap). For correct
multi-currency billing, align the usage-record currency, the cap currency, and
the recurring currency, applying FX consistently — or constrain plans to one
currency. **Do not assume they agree.**

---

## 10. Acceptance scenarios

Implement and verify each:

1. **Flat-rate subscribe + activate.** Create → `confirmationUrl` → approve →
   `returnUrl` verifies `status == active` → local row goes active with mirrored
   period dates. No charge posted by you; Shopify collects.
2. **Verify-failure path.** Hit `returnUrl` while Shopify still shows the charge
   pending → local row stays inactive, merchant is still returned to the app, a
   reconcile job is enqueued, and a later sync activates it.
3. **Single-active invariant.** Subscribe to plan B while plan A is active →
   on B's activation, A is canceled in the same transaction; never two active.
4. **Annual plan.** Created with `interval = ANNUAL`; reconciliation confirms
   Shopify's interval matches.
5. **Usage / metered.** Subscribe with a usage line; post `appUsageRecordCreate`
   against the persisted GID; `balanceUsed` mirrors back; an over-cap record is
   rejected and handled per the chosen ceiling policy.
6. **Hybrid.** Recurring + usage in one create; discount applies only to the
   recurring line.
7. **One-time.** `appPurchaseOneTimeCreate` → standalone `charges` row, no
   subscription mirror, no trial/discount.
8. **Free `$0`-no-usage.** No Shopify call; local row active immediately;
   reconciliation does not flag the missing Shopify object.
9. **Plan change (no gap, no double-charge).** New created and confirmed before
   old canceled; exactly one active throughout.
10. **Provider migration (Stripe → Shopify).** Shopify sub confirmed active →
    old Stripe sub canceled → `billingType` flipped to `shopify`, in that order;
    no billing gap.
11. **Cancellation at period end.** `cancelAtPeriodEnd` keeps service until
    `currentPeriodEnd`; outstanding usage collected before cancel.
12. **Orphan repair.** Simulate a dropped cancel webhook (Shopify gone, local
    active) → reconciliation flips local inactive without clobbering an existing
    `canceledAt`.
13. **Abandoned confirmation.** A pending subscription whose `confirmationUrl`
    is never clicked is cleaned up by the stale-pending sweep.
14. **Idempotent re-runs.** Re-invoking activation, cancel, webhook
    registration, and reconcile produces no duplicates and no double-charge.
</content>
