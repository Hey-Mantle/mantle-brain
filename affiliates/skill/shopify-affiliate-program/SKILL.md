---
name: shopify-affiliate-program
description: >-
  Build a referral/affiliate program in an app — tracked referral links,
  click-to-install attribution, commission accrual on referred revenue
  (percentage / recurring / one-time bounties), and affiliate payouts via PayPal
  or Stripe Connect, plus an affiliate-facing portal. Use when the user wants to
  add an affiliate or partner program, track referrals with mref/ref links,
  attribute installs to referrers, calculate or pay out commissions, build an
  affiliate dashboard, or migrate off Mantle Affiliates into their own codebase.
---

# Shopify Affiliate Program

## What this skill does

Helps you implement a **referral/affiliate program**: affiliates share a tracked
link, the installs they drive are attributed to them, the revenue those installs
generate accrues commission under rules you control, and that commission is paid
out via PayPal or Stripe.

The pipeline is **referral → attribution → commission → payout**, organized around
one unit of identity: a unique **handle** (the `mref` code) per (affiliate,
program) membership, embedded in the shared link as `?mref=<handle>`.

The full, precise mechanics live in
**[`references/implementation-spec.md`](./references/implementation-spec.md)**.
Read it before writing code — it has the data model, the exact PayPal/Stripe API
shapes, the commission math, the rule-resolution logic, and the edge cases. This
file is the *procedure*; the spec is the *reference*.

## When to use it

Trigger on intents like: "add an affiliate/referral/partner program," "track
referral links," "attribute installs to a referrer," "pay affiliates a commission
on revenue," "recurring vs one-time affiliate commissions," "build an affiliate
dashboard/portal," "pay out affiliates via PayPal/Stripe," or "migrate off Mantle
Affiliates."

## How to drive the implementation

Work through these phases. **Do not dump code blindly** — discover the user's
stack, billing model, and existing data first, then map the spec onto it.

### Phase 1 — Orient (ask, then read)
1. Determine the stack: language, web framework, ORM/database, job/cron runner,
   and how they bill (Shopify subscription/usage transactions? Stripe? something
   else — the commission engine needs a stream of billing *transactions* to read).
2. Find existing data: any current notion of a referrer, partner, coupon, or UTM
   capture; the billing/transactions table; the install/customer table.
3. Confirm scope — which parts do they need?
   - tracked links + attribution only (know who referred whom)?
   - + commission accrual (percentage / recurring / one-time bounty)?
   - + payouts (PayPal pass-through, and/or Stripe Connect with a fee)?
   - + an affiliate-facing portal (OTP login, stats, assets, marketplace)?
4. Confirm payout prerequisites for whichever rail they want:
   - **PayPal Payouts** — a business PayPal account + Payouts API access.
   - **Stripe Connect** — a Stripe account with Connect enabled (Express
     accounts) for affiliate onboarding.
5. Multi-tenant or single-tenant? If single-tenant, collapse the
   `affiliate_user` + `affiliate` identities and drop `organization_id` scoping.

### Phase 2 — Read the reference
Read [`references/implementation-spec.md`](./references/implementation-spec.md)
end to end. The invariants in §0 are non-negotiable; the §11 MUST/MUST NOT rules
are the bugs you're being paid to avoid. Pay special attention to §6.2 (rule
resolution — the non-merge) and §8 (the payout rails).

### Phase 3 — Data model
Map spec §1 onto the user's schema. Add/confirm: the identity trio
(`affiliate_users` global login, `affiliates` per-merchant earner, `memberships`
whose **`handle` is the `mref` code**); `affiliate_programs` with the `rules`
JSON; `affiliate_attributions`; `affiliate_commissions` (with `cancelled` +
`payout_id`); `affiliate_payouts` (with the sequential `number` + period); and the
payout-rail bookkeeping for whichever rail(s) they chose. Write a migration. Add
the indexes the sweep and aggregation selectors need.

### Phase 4 — Links + capture (spec §3.1–3.2)
Build the link generator (`baseUrl + ?mref=<handle>`, with `baseUrl` =
the user's own signup/landing URL, not hardcoded `apps.shopify.com`). Drop in the
client capture script (`mref || utm_source || ref`, 30-day cookie, link
rewriting) and wire the stored code into their signup/account-creation call.

### Phase 5 — Attribution (spec §5)
Resolve the handle → enrolled membership (scoped to the app/program) → write ONE
attribution. **Pick a de-dup policy explicitly** (first-write-wins recommended)
and guard it with a per-install lock. Fire the side effects: install bounty,
re-scan cursor reset, `referral` event. If they're on the Shopify App Store, note
the query-param-stripping problem and that carrying the code themselves (not the
GA/BigQuery bridge) is the answer.

### Phase 6 — Commission engine (spec §6)
Implement rule resolution **with the commission-field omission exactly as
specified** (port the unit tests), then the sweep: scan referred installs'
transactions since a per-affiliate cursor, apply the gate chain (non-positive
skip, cutoffs, revenueComponents, minPlanValue, dedup-by-(affiliate,transaction)),
and mint `signup` / `percent` / `install` commissions per the formulas. **Make the
refund policy an explicit decision** (the engine has no auto-clawback).

### Phase 7 — Payouts (spec §7–8)
Implement aggregation (bucket outstanding commissions, minimum gate, sequential
number, chained period, per-affiliate lock) and at least one payment rail. PayPal
is simplest (pass-through, `paypal_email || email`, webhook reconcile). Stripe
Connect if they want a fee (charge-then-transfer, live-capability gate,
idempotency keys, partial-failure reconcile). Add the status machine and, if
wanted, the auto-payout scheduler.

### Phase 8 — Portal, events, verify (spec §9–10, §13)
If they want a portal: OTP-first login over a signed session, identity-linking by
email, and **properly scoped asset downloads**. Wire the event chokepoint +
webhook signing, and add the `commission_earned` / `payout_paid` events Mantle
lacks if their customers want them. Then walk the spec §13 acceptance scenarios as
a checklist.

## Guardrails

- **Preserve the six invariants** (spec §0): the handle is the spine; one active
  attribution per install (code + lock); commissions are a derived sweep, not
  inline; rule resolution is a non-merge; aggregate-then-pay; money movements are
  idempotent + gated.
- **Rule resolution is the #1 bug risk.** An override (even empty) drops the
  parent's commission fields. Port the unit tests verbatim; don't "fix" it into a
  deep merge unless the user wants different semantics.
- **No silent money bugs.** Decide the refund-clawback policy on purpose (the
  engine skips refunds, never claws back). Gate Stripe transfers on the live
  capability. Handle Stripe partial-batch failure. Use `paypal_email || email`.
- **Scope asset downloads** to the membership's program + the asset's visibility —
  don't fetch an asset by id alone.
- **Don't blindly copy USD** — Mantle does no FX. If the user serves non-USD
  merchants, normalize currency explicitly.
- **Skip the Mantle-specific scaffolding**: the Google-Analytics→BigQuery
  attribution reconstruction (only needed because the App Store strips params) and
  the competitor-importer pipeline (reuse only the import *shape*).
- If the user wants a payout rail they don't have access to (no PayPal Payouts / no
  Stripe Connect), **say so and build attribution + commissions first** — those
  work without a payout rail.
- Prefer editing the user's existing models/clients over inventing parallel ones.
  Match their conventions.
