---
name: shopify-billing-migration
description: >-
  Implement standard Shopify-managed app billing in an app — where Shopify owns
  the billing clock and collects charges, and the app keeps a reconciled local
  mirror — and migrate merchants onto that rail across billing models (flat,
  annual, usage/metered, hybrid, one-time, free) in the correct order of
  operations. Use when the user wants to add Shopify app subscriptions, mirror
  Shopify subscriptions locally, migrate merchants from Stripe/manual/Apple onto
  Shopify billing, handle the confirmation/return-url activation flow, or
  reconcile Shopify charge state. NOT for re-approval-free in-place tier changes
  (that is flex billing — a separate skill).
---

# Shopify Billing Migration

## What this skill does

Helps you implement **standard Shopify-managed app billing** as a **wrapper**:
Shopify is the system of record for *approval and collection*, your database is
the system of record for *the business agreement* (plans, trials, discounts,
metered metrics, analytics). It also covers **migrating** merchants onto this
rail — from no subscription, a different plan, or a different biller — in the
order of operations that avoids dropped cycles, double charges, and split-brain
state.

The mechanism: create a **pending** local subscription, push a matching
`AppSubscription` to Shopify (`appSubscriptionCreate`), send the merchant to the
returned `confirmationUrl`, and on the `returnUrl` callback **verify against
Shopify** before flipping the local row active and **atomically canceling any
other active subscription** for that install. From then on **Shopify collects on
its own cycle** — you do **not** run a charging cron — and scheduled
reconciliation keeps the mirror honest.

The full, precise mechanics live in
**[`references/implementation-spec.md`](./references/implementation-spec.md)**.
Read it before writing code — it has the exact GraphQL shapes, the data model,
the ordered activation procedure, per-model line items, and the edge cases. This
file is the *procedure*; the spec is the *reference*.

> **Out of scope:** *flex billing* — the `$0`-recurring + usage-record technique
> that lets you move merchants between tiers in-place without re-approval. If the
> user wants re-approval-free upgrades/downgrades or auto-tiering, that is a
> different system (mantle-brain ships it as the `flex-billing` topic / the
> `shopify-flex-billing` skill). Don't build it here.

## When to use it

Trigger on intents like: "add Shopify app subscriptions," "bill merchants
through Shopify," "mirror Shopify subscriptions in our DB," "migrate merchants
from Stripe (or manual billing) onto Shopify billing," "handle the Shopify
charge confirmation/return flow," "reconcile our subscriptions against Shopify,"
or "support usage/metered/one-time/annual Shopify billing." If the intent is
in-place tier changes without re-approval, use the **flex billing** skill
instead.

## How to drive the implementation

Work through these phases. **Do not dump code blindly** — discover the user's
stack and existing billing code first, then map the spec onto it.

### Phase 1 — Orient (ask, then read)
1. Determine the stack: language, web framework, ORM/database, job/scheduler,
   and how they already talk to Shopify (Admin API client? Partner API access?).
2. Find existing billing code: any current `appSubscriptionCreate`, plan model,
   subscription model, return-url/confirmation handler, webhook handlers,
   scheduled jobs.
3. Confirm prerequisites: the app has **public distribution** (private/dev apps
   can't use the Billing API); Partner API access if they want charge/transaction
   reconciliation.
4. Establish what they're migrating **from** (greenfield, a different plan
   structure, or another biller like Stripe/manual/Apple) and which **billing
   models** they actually sell (flat, annual, usage, hybrid, one-time, free).

### Phase 2 — Read the reference
Read [`references/implementation-spec.md`](./references/implementation-spec.md)
end to end. The §0 invariants are non-negotiable (especially: **no charging
cron — Shopify collects**); the §8 MUST/MUST NOT rules are the bugs you're being
paid to avoid.

### Phase 3 — Data model
Map spec §1 onto the user's schema. Add/confirm:
- `subscriptions` that mirror one Shopify `AppSubscription`: `billingType` rail,
  `active`/`activatedAt`/`canceledAt`/`frozenAt`, mirrored period dates,
  trial/discount fields, `confirmationUrl`, and a pin to the charge mirror.
- A **charge-mirror** row holding the Shopify charge GID + status.
- `subscription_line_items` persisting Shopify **line-item GIDs** (recurring,
  usage, add-on) and usage cap/balance mirrors.
- `charges` with both raw and normalized amount/currency.
Write a migration.

### Phase 4 — Subscribe (spec §3)
Implement signup: create the pending local row, derive line items per model,
call `appSubscriptionCreate`, persist the subscription GID + usage line-item GID
+ `confirmationUrl`, and redirect. **Do not** mark active or charge here. Handle
the `$0`-no-usage shortcut (activate locally, no Shopify call) and the
private-app error.

### Phase 5 — Activation order of operations (spec §4)
Implement the `returnUrl` handler with the load-bearing sequence: **verify**
against Shopify (`status == active`) → **mirror** dates/line items → **atomic
activate-self + cancel-others** in one transaction → side effects (idempotent
webhook registration, cache bust, reconcile job). On verify failure, return the
merchant and heal asynchronously. Lock per (install, subscription) for
idempotency.

### Phase 6 — Migration by billing model (spec §5)
Implement `buildLineItems` per model the user sells (flat, annual, usage,
hybrid, one-time via `appPurchaseOneTimeCreate`, free). For a **provider
migration** (Stripe/manual → Shopify), enforce the ordering: **new Shopify sub
confirmed active → cancel old rail → flip `billingType` last.** Collect
outstanding usage before canceling.

### Phase 7 — Cancellation & sync (spec §6–§7)
Implement cancellation (`appSubscriptionCancel`, tolerate already-uninstalled,
optional `cancelAtPeriodEnd`) and **scheduled reconciliation**: period-clock
sync (Shopify doesn't webhook rollovers), orphan repair (without clobbering
`canceledAt`), charge/transaction backfill, divergence reporting, and an
abandoned-pending-subscription cleanup sweep. Register `APP_UNINSTALLED`
idempotently.

### Phase 8 — Verify
Walk the spec §10 acceptance scenarios as a checklist. At minimum prove:
subscribe→verify→activate (no self-posted charge), the single-active invariant
under a plan change, a provider migration with no billing gap, usage records
mirroring balance, orphan repair, and idempotent re-runs.

## Guardrails

- **Preserve the six invariants** (spec §0): Shopify owns the clock (no charging
  cron), the local row mirrors one Shopify subscription, verify-before-activate,
  one active subscription per install, new-before-old + flip-rail-last, reconcile
  on a schedule.
- **Never trust the return redirect** — re-query Shopify before activating.
- **Never cancel a collecting subscription before its replacement verifies.**
- **Don't build a charging cron** — if the user needs to drive charges
  themselves or wants re-approval-free tier changes, that's flex billing; stop
  and point them at that skill.
- **Don't create a Shopify object for `$0`-no-usage plans**, and teach
  reconciliation not to flag it.
- **Keep activation, cancel, webhook registration, and sync idempotent.**
- If the app is private/dev or lacks the access it needs, **stop and tell them**.
- Prefer editing the user's existing models/clients over inventing parallel ones.
  Match their conventions.
</content>
