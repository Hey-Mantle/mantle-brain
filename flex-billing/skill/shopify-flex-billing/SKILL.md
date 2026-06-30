---
name: shopify-flex-billing
description: >-
  Implement Shopify "flex billing" in an app — flexible tiered app subscriptions
  with in-place upgrades/downgrades, proration, and automatic usage-based
  tiering, all without triggering Shopify's subscription re-approval flow. Use
  when the user wants to add flexible/tiered Shopify billing, charge usage
  records against a capped usage line, pro-rate plan changes, auto-upgrade on
  usage limits, or migrate off Mantle Flex Billing into their own codebase.
---

# Shopify Flex Billing

## What this skill does

Helps you implement **flex billing** in a Shopify app: tiered subscriptions where
you move a merchant between pricing tiers in-place — charging or crediting the
difference — **without forcing them through Shopify's subscription re-approval
screen**.

The mechanism: the Shopify recurring price is fixed at **$0**, and real revenue is
collected through **usage records** (`appUsageRecordCreate`) posted against a
**capped usage line item**. Because the recurring price never changes, tier
changes never require re-confirmation. You own the billing clock in your own DB
and bill in advance via a daily cron.

The full, precise mechanics live in **[`references/implementation-spec.md`](./references/implementation-spec.md)**.
Read it before writing code — it has the exact GraphQL shapes, the data model,
proration math, and the edge cases. This file is the *procedure*; the spec is the
*reference*.

## When to use it

Trigger on intents like: "add tiered/flexible Shopify billing," "let customers
upgrade/downgrade plans without re-approval," "pro-rate plan changes,"
"auto-upgrade when usage crosses a limit," "charge usage records for a monthly
fee," or "migrate off Mantle Flex Billing."

## How to drive the implementation

Work through these phases. **Do not dump code blindly** — discover the user's
stack and existing billing code first, then map the spec onto it.

### Phase 1 — Orient (ask, then read)
1. Determine the stack: language, web framework, ORM/database, job/cron runner,
   and how they already talk to Shopify (Admin API client? Partner API access?).
2. Find existing billing code: any current `appSubscriptionCreate` usage, plan
   model, subscription model, webhook handlers, cron/scheduled tasks.
3. Confirm two prerequisites that flex billing **requires**:
   - The app has **public distribution** (private/dev apps can't use the Billing
     API).
   - They have **Partner API** access (needed for downgrade credits via
     `appCreditCreate`).
4. Ask whether they need the full feature set or a subset:
   - recurring fee only (no tier changes)?
   - + manual upgrades/downgrades with proration?
   - + automatic usage-based upgrades?
   - + trials? + discounts? + non-USD? (non-USD needs the §8 currency work.)

### Phase 2 — Read the reference
Read [`references/implementation-spec.md`](./references/implementation-spec.md)
end to end. The invariants in its §0 are non-negotiable; the §7 MUST/MUST NOT
rules are the bugs you're being paid to avoid.

### Phase 3 — Data model
Map spec §1 onto the user's schema. Add/confirm:
- `plans`: `flexBilling` (immutable), `usageChargeCappedAmount`, `flexBillingTerms`,
  recurring interval, and (for tiering) the usage band + `autoUpgradeToPlanId`.
- `subscriptions`: the period fields (`currentPeriodStart/End`, `nextBillingDate`,
  `billingCycleAnchor`), `pausedUntil`, `shopifySubscriptionId`,
  `triggeredByUsageChargeId`, trial fields.
- A usage `subscription_line_items` row holding the Shopify usage line **GID**.
- `charges` with `flexBilling` + `isCredit` + `occurredAt`.
- An append-only `flex_billing_events` audit table.
Write a migration. Add a **partial index** matching the cron selector (spec §4.1).

### Phase 4 — Subscribe (spec §3)
Implement signup: $0 recurring line + capped usage line, `trialDays: 0` to
Shopify, persist the usage line GID, redirect to `confirmationUrl`, and **collapse
the first period to today** (or to trial-end if trialing). Do not charge during
signup.

### Phase 5 — Daily charge cron (spec §4)
Implement the selector, the per-(org, subscription) lock, the period roll, the
discount-aware amount, the usage-record post, and the **unconditional-vs-checked
period advance** decision (read the cap-blocked-fee warning in §7 and pick a
policy deliberately). Wire it to the user's scheduler. Handle the $0 rollover and
the private-app skip.

### Phase 6 — In-place tier changes (spec §5)
Implement collect-outstanding-first, day-based proration, the cap-gated upgrade
charge with the **confirmable-subscription fallback**, and the Partner-API
downgrade credit **capped at collected-this-period** with the
`disableDowngradeCredits` opt-out. Reuse the one Shopify subscription; copy the
cycle + usage line forward on commit.

### Phase 7 — Automatic upgrade (spec §6, optional)
If they want usage-based tiering: enqueue an upgrade check when a `unit_limits`
metric crosses `limitMax`, route it through the **same** tier-change path (so it
pro-rates), tag `triggeredByUsageChargeId`, and add the loop guards (existing-sub
dedupe + cascade cap).

### Phase 8 — Verify
Walk the spec §9 acceptance scenarios as a checklist. At minimum prove: first
charge fires on the next cron (not at signup), steady-state charges once per
period, mid-cycle upgrade pro-rates and the next cycle bills the new amount,
downgrade credit is capped at collected, and no double-charge under concurrent
cron + manual change.

## Guardrails

- **Preserve the five invariants** (spec §0): $0 recurring, you own the clock,
  bill in advance, no billing webhook (cron-driven), one Shopify subscription per
  merchant. Breaking any of them reintroduces re-approval friction or
  double-billing.
- **Don't post $0 usage records** — advance the clock instead.
- **Don't blindly copy the hard-coded USD** if the user serves non-USD merchants
  (spec §8).
- **Don't let the audit table gate money** — it's observability.
- If the user's app is private/dev or lacks Partner API access, **stop and tell
  them** — flex billing can't work without those.
- Prefer editing the user's existing models/clients over inventing parallel ones.
  Match their conventions.
