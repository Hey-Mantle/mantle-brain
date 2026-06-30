# Affiliate Program — Implementation Spec (for agents)

> **Audience:** a coding agent implementing a referral/affiliate program — tracked
> links, attribution, commission accrual, and affiliate payouts — in an arbitrary
> Shopify (or non-Shopify) app. This is a precise, self-contained build spec; no
> external files required. The procedure that drives this spec lives in
> [`SKILL.md`](../SKILL.md); a human-readable walkthrough lives in the
> `affiliates/README.md` of the mantle-brain library.
>
> **Provenance:** distilled from the production behavior of Mantle's Affiliates
> subsystem. Stack-agnostic; map the generic table/field names below onto your own
> ORM. Where a rule encodes a hard-won lesson or a real bug, it is marked
> **MUST** / **MUST NOT**.

---

## 0. Goal & invariants

Build a program where affiliates share a tracked link, the installs they drive are
attributed to them, the revenue those installs generate accrues commission under
your rules, and commission is paid out via PayPal or Stripe.

The invariants that keep it correct:

1. **The handle is the spine.** Every (affiliate, program) membership has a unique
   `handle` — the `mref` code — and it appears in the link as `?mref=<handle>`.
   Resolution is always *scoped to the program/app*: the same handle string may
   exist in another program.
2. **One active attribution per install.** This is enforced in *application code +
   a lock*, not by the schema. You MUST pick a de-dup policy (first- or last-
   write-wins) and apply it consistently.
3. **Commissions are derived, not posted inline.** A sweep reads each referred
   install's transactions since a per-affiliate cursor and mints commission rows.
   The sweep is incremental and idempotent (dedup by `(affiliate, transaction)`).
4. **Rule resolution is a deliberate non-merge.** When a lower tier (group /
   membership / attribution) overrides, the parent's *commission fields* are
   dropped, not inherited. This is the highest-bug-risk logic in the system.
5. **Payout = aggregate then pay.** Stage 1 buckets outstanding commissions into a
   payout (gated by a minimum). Stage 2 moves money on one of two rails. They are
   separate; money only moves in stage 2.
6. **Money movements are idempotent and gated.** Every external charge/transfer
   carries an idempotency key; Stripe transfers are gated on the connected
   account's *live* capability; aggregation and payment run under per-affiliate
   locks.

If you preserve these six, the rest is bookkeeping.

---

## 1. Data model

Field names are illustrative — map them to your schema. If your app is
single-tenant, drop the `organization_id` scoping everywhere and collapse
`affiliate_user` + `affiliate` into one table.

### 1.1 `affiliate_users` — the global login identity
One person, across all merchants/programs. Holds payout identity + portal auth.
- `id`, `email` (**globally unique**), `name?`, `username?` (unique),
  `email_verified_at?`, `image_url?`
- payout identity: `paypal_email?`, `stripe_account_id?`, `stripe_account JSON?`,
  `country_code?`, `type` (`person`|`company`), address fields, `tax_id?`
- `google_id_token JSON?` (optional OAuth), `password?` (**legacy — not checked at
  login**, see §9)

### 1.2 `affiliates` — the per-merchant earner
- `id`, `organization_id`, `email` (**unique only within a merchant**, NOT
  global), `name?`, `paypal_email?`, `user_id?` → `affiliate_users`
- `agreed_to_terms_at?` (merchant-level terms), `payout_hold` (bool — blocks
  payouts), `tags String[]`
- sweep cursors: `last_transaction_date?`, `last_app_installation_date?` (so the
  commission scanner only re-processes new transactions/installs)
- `start_counting_commission_from?` (per-affiliate commission cutoff, set on import)
- `deleted_at?` (soft delete), `redacted_at?` (PII scrub)

### 1.3 `affiliate_programs` — a program (per app)
- `id`, `organization_id`, `name`, `app_id?` (programs are per-app)
- `rules JSON @default("{}")` (the economics — see §2)
- join policy: `require_approval_to_join` (bool), `require_terms_acceptance`
  (bool), `terms_url?`, `required_affiliate_fields JSON`
- `custom_affiliate_link?` (overrides the default base URL),
  `remove_on_uninstall_days Int @default(-1)` (-1 = never auto-remove attribution
  on uninstall; ≥0 = grace window), `start_counting_commission_from?`
- portal flags: `branding JSON?`, `show_in_marketplace` (bool),
  `allow_affiliate_dashboard_access` (bool)

### 1.4 `affiliate_memberships` — joins an affiliate to a program
- `id`, `organization_id`, `affiliate_id`, `affiliate_program_id`
- **`handle?` — THE `mref` referral code.** Unique per
  `(organization_id, affiliate_program_id, handle)`.
- `rules JSON?` (per-affiliate override), `rules_override String?` — a
  **discriminator**, one of `null` / `"group"` / `"custom"` (selects which rule
  tier wins — see §6.2). `rules` is ignored unless `rules_override === "custom"`.
- `status String @default("pending")` — observed values `pending`, `enrolled`,
  `rejected`
- `group_id?` → `affiliate_groups`, `from_marketplace` (bool)
- invite lifecycle: `invited_at?`, `invited_by_id?`, `invite_accepted_at?`,
  `invite_rejected_at?`
- approval lifecycle: `approval_requested_at?`, `approved_at?`, `rejected_at?`,
  `approval_decision_by_id?`
- `agreed_to_terms_at?` (program-level terms), `deleted_at?`

### 1.5 `affiliate_groups` — a named rate tier inside a program (optional)
- `id`, `organization_id`, `affiliate_program_id`, `name`, `rules JSON?`,
  `show_in_marketplace`

### 1.6 `affiliate_attributions` — the output of attribution
- `id`, `organization_id`, `affiliate_id`, `affiliate_program_id`,
  `app_installation_id` → the referred install
- `date?` (referral date), `rules JSON?` (**snapshot at attribution time —
  highest precedence**), `deleted_at?`
- **No DB uniqueness on `app_installation_id`** — the "one active attribution"
  invariant is code + lock (§5).

### 1.7 `affiliate_commissions` — one earned amount
- `id`, `organization_id`, `affiliate_id`, `affiliate_program_id`
- `affiliate_attribution_id?`, `transaction_id?` (the billing event that generated
  it), `app_installation_id?`
- `type String?` — `signup` | `percent` | `install` | `custom`
- `amount Float`, `date?`, `name?`, `notes?`
- `cancelled Boolean @default(false)`, `cancel_reason?` (manual void — see §6.5)
- `payout_id?` (set when rolled into a payout; **nulled on cancel**),
  `deleted_at?`
- No DB uniqueness on `(transaction, type)` — dedup is in code (§6.4).

### 1.8 `affiliate_payouts` — the aggregation bucket
- `id`, `organization_id`, `affiliate_id`, `affiliate_program_id?`,
  `funding_source_id?`
- `number Int?` — **per-merchant sequential** invoice-like number
  (`unique(organization_id, number)`)
- `status String @default("pending")` — `pending` → `requested` → `processing` →
  `paid`, or `cancelled`
- `amount Float` (gross commission total), `amount_paid Float?` (net actually paid)
- `period_start?`, `period_end?` (chains off the previous paid payout's end),
  `paid_at?`, `paid_by_id?`, `payment_method?`, `payment_method_data JSON?`,
  `payment_requested_at?`

### 1.9 Payout-rail bookkeeping (substitute with your own PSP plumbing)
- `affiliate_payout_funding_sources` — per-app/division Stripe funding:
  `stripe_customer_id?`, `stripe_payment_method_id?`, `minimum_payout_amount?`,
  `payout_method?`
- `affiliate_paypal_integrations` — per-merchant PayPal OAuth: `access_token`,
  `refresh_token?`, `email?`, `expires_at?`
- `affiliate_batch_paypal_payments` + items — mirror a PayPal Payouts batch
- `affiliate_batch_payments` + items — Stripe-rail equivalent, with
  `org_fees` / `affiliate_fees` fee splitting

### 1.10 `affiliate_events` — the lifecycle-event spine
- `id`, `organization_id`, `affiliate_id`, `affiliate_program_id?`,
  `membership_id?`, `attribution_id?`, `payout_id?`
- `type String` (§10), `date?`

### 1.11 Merchant-level payout config (on your `organization`/settings)
- `affiliate_minimum_payout_amount` (default 0), `affiliate_payout_start_number`
  (default 1000)
- `affiliate_payout_fee_split` (default 0.5), `affiliate_payout_fee_percent`
  (default 0.1)
- `affiliate_auto_payout_enabled` (default false), `affiliate_auto_payout_schedule?`
  (`weekly`|`biweekly`|`monthly`), `affiliate_auto_payout_day_of_week?` /
  `_day_of_month?`, `affiliate_auto_payout_last_run_at?`

---

## 2. The rules JSON shape

The economics of a program (or group, or membership override, or attribution
snapshot). Sanitize on write: coerce numerics with `parseFloat`/`parseInt`, filter
`revenueComponents` against the allow-list, default an invalid `revenueType` to
`"gross"`, and **drop any absent field**.

```js
{
  percentCommission: 20,                 // recurring % of each qualifying transaction
  durationMonths: 12,                    // commission window length; 0/absent = lifetime
  amountPerInstall: 5.00,                // one-time flat bounty per referred install
  signupBonus: 10.00,                    // one-time bonus on the install's FIRST eligible transaction
  minPlanValue: 49.00,                   // gate: skip if the install's ACTIVE plan price < this
  revenueComponents: ["subscription"],   // subset of ["subscription","usage","one-time"]; empty/absent = all
  revenueType: "gross"                   // "gross" | "net" — which transaction amount to pay on
}
```

- **There is no cap field.** The only bounds are `durationMonths` (time) and
  `minPlanValue` (eligibility). A per-referral payout cap is your job to add.
- **`durationMonths` is coerced behind a truthy check**, so a literal
  `durationMonths: 0` is silently dropped (treated as absent = lifetime). You
  cannot persist an explicit `0`; treat absent as lifetime.

The four **commission fields** — `["percentCommission","durationMonths",
"revenueComponents","minPlanValue"]` — are the ones subject to the non-merge rule
in §6.2. The rest (`amountPerInstall`, `signupBonus`, `revenueType`) always fall
through.

---

## 3. External surface

### 3.1 The tracked link
```
baseUrl = program.custom_affiliate_link || "https://apps.shopify.com/<your-app-slug>"
link    = baseUrl + (baseUrl.includes("?") ? "&" : "?") + "mref=" + membership.handle
```
The `apps.shopify.com` base is the only Shopify-specific part. **For a non-App-
Store funnel, replace `baseUrl` with your own signup/landing URL.**

### 3.2 The client capture script (reuse almost verbatim)
A ~200-line vanilla-JS snippet placed on your marketing pages:
1. Read the referral code from the URL: `mref || utm_source || ref` (that
   precedence).
2. Persist it under a stable key (e.g. `affiliate_referral`) in a **cookie with a
   30-day expiry**, falling back to `sessionStorage` if cookies are unavailable.
3. Expose it (e.g. `window.affiliateTrack.referralCode`) so your signup page can
   read it.
4. (Optional) If given a `link_url` param, rewrite every `<a>` whose href points at
   your signup host to carry `?mref=<code>`.

Your signup/account-creation call MUST forward the stored code to the attribution
endpoint (§5).

### 3.3 PayPal Payouts (Rail A)
- Auth: per-merchant OAuth (`POST /v1/oauth2/token`, `Basic base64(clientId:secret)`,
  `grant_type=refresh_token`), then Bearer. Host `https://api-m.paypal.com` (prod)
  / `https://api-m.sandbox.paypal.com` (sandbox).
- Create batch: `POST /v1/payments/payouts`
  ```js
  {
    sender_batch_header: { sender_batch_id: batch.id },
    items: [{
      recipient_type: "EMAIL",
      amount: { value, currency: "USD" },
      note: `Payout for <name>. Reference #<number>.`,
      sender_item_id: payout.id,
      receiver: affiliate.paypal_email || affiliate.email,   // MUST fall back to regular email
    }]
  }
  ```
- Reconcile via PayPal webhook (`payouts_item`): `transaction_status: SUCCESS` →
  mark paid; `RETURNED` / `FAILED` → revert payout to `pending`.

### 3.4 Stripe Connect (Rail B)
- Onboard the affiliate: `stripe.accounts.create({ type: "express", capabilities:
  { transfers: { requested: true } }, tos_acceptance: { service_agreement:
  countryCode === "US" ? "full" : "recipient" } })` + `stripe.accountLinks.create`.
- **Capability gate** (MUST): before charging, re-fetch the account and require
  `account.capabilities.transfers === "active"`. Fail closed.
- Charge yourself: `stripe.paymentIntents.create({ amount: amountCharged*100,
  customer, payment_method, off_session: true, confirm: true, transfer_group:
  "affiliate-payouts-<batchId>" }, { idempotencyKey: batchId })`.
- Pay each affiliate (on `payment_intent.succeeded`): `stripe.transfers.create({
  amount: (item.amount - item.affiliateFee)*100, destination:
  affiliate.stripe_account_id, source_transaction: paymentIntent.latest_charge })`.

---

## 4. Procedure: enroll / join

```
enrollAffiliate(org, program, affiliate, { requestedAt?, requireApproval, handleSeed }):
    # idempotent: if a non-deleted membership already exists, return it unchanged
    existing = membership(org, program, affiliate, deletedAt=null)
    if existing: return existing

    # assign a unique handle (the mref code)
    handle = handleSeed || username || slug(name) || localPart(email)
    while handleExists(org, program, handle):
        handle = `${handle}-${rand()}`

    # status
    if requestedAt and program.require_approval_to_join: status = "pending"
    else:                                                status = "enrolled"

    m = createMembership(org, program, affiliate, handle, status)
    emit AffiliateEvent(requestedAt ? "join_requested" : "joined")
    return m

approveMembership(m):  m.status = "enrolled"; emit "join_approved"; emit "joined"   # TWO events
rejectMembership(m):   m.status = "rejected"; emit "join_denied"
```

- **MUST keep enrollment idempotent** — a re-join returns the existing membership.
- Approval emits **both** `join_approved` and `joined`.

---

## 5. Procedure: attribution

```
attributeReferralCode(code, appInstallation, { affiliateProgram or appId }):
    membership = findMembership(handle=code, deletedAt=null, program scoped to appId,
                                status="enrolled")        # MUST scope to the app/program
    if not membership: return null
    return attributeAffiliateToAppInstallation(membership.affiliate, membership.program,
                                               appInstallation, date)

attributeAffiliateToAppInstallation(affiliate, program, appInstallation, date):
    active = appInstallation.attributions where deletedAt=null
    existing = active.find(a => a.affiliate==affiliate AND a.program==program)

    # de-dup policy (LAST-write-wins shown; choose ONE and document it):
    if active.length > 0:
        softDelete(active where id != existing?.id)       # deletedAt = now

    if not existing:
        attribution = createAttribution(affiliate, program, appInstallation, date,
                                        rules=optionalSnapshot)
        runAttributionCreatedSideEffects(...)             # see below
    else:
        if existing.date != date: updateDate(existing, date)
        attribution = existing
    return attribution

runAttributionCreatedSideEffects(affiliate, program, appInstallation, attribution):
    affiliate.last_transaction_date = null                # force the sweep to re-scan history
    enqueue commission sweep for affiliate
    createInstallCommission(affiliate, program, appInstallation, attribution)   # §6.5 INSTALL
    emit AffiliateEvent("referral")
```

- **MUST scope handle resolution to the app/program** — the same handle can exist
  in another program.
- **MUST pick a single de-dup policy.** First-write-wins is recommended (return the
  existing attribution untouched). The shown last-write-wins soft-deletes prior
  active attributions and recreates. **Guard concurrency with a per-install lock**
  (e.g. `attributeHandle:<appInstallationId>`) — the natural `created_at` ordering
  does not reliably reflect commit order.
- **MUST only attribute `enrolled` memberships.**
- Side effects are part of attribution: reset the re-scan cursor, mint the install
  bounty, emit `referral`.

> **Shopify note:** the App Store strips query params on install, so the `mref`
> won't reach your backend automatically. Carry it yourself (§3.2 → your signup
> call). Mantle *also* reconstructed attribution from a Google-Analytics→BigQuery
> bridge to work around this — **skip that entirely** if you control your funnel.

---

## 6. Procedure: the commission sweep

### 6.1 Trigger
Two ways into the same function:
1. Enqueued immediately when an attribution is created/changed (debounced per
   affiliate).
2. A **daily cron** (Mantle: `0 0 9 * * *`).

### 6.2 Rule resolution (the non-merge — port the unit tests verbatim)
Precedence highest-first: **attribution.rules > membership(custom) > group >
program**. Let `COMMISSION_FIELDS = ["percentCommission","durationMonths",
"revenueComponents","minPlanValue"]`.

```
resolveRules(program, membership, attribution):
    fallback = { ...program.rules,
                 ...(membership.rules_override == "group"  ? membership.group.rules : {}),
                 ...(membership.rules_override == "custom" ? membership.rules       : {}) }

    if attribution.rules:                       # attribution snapshot wins outright
        return { ...omit(fallback, COMMISSION_FIELDS), ...attribution.rules }

    if membership.rules_override == "custom":   # program commission fields do NOT leak through
        return { ...omit(fallback, COMMISSION_FIELDS), ...(membership.rules || {}) }

    if membership.rules_override == "group" and (group.rules.percentCommission or
                                                 group.rules.amountPerInstall):
        toOmit = COMMISSION_FIELDS.filter(f => group.rules[f] !== undefined)
        return { ...omit(fallback, toOmit), ...group.rules }

    return fallback
```

- **MUST drop the parent's commission fields when a lower tier overrides** — even a
  `custom` override with empty `{}` strips program `minPlanValue` / `durationMonths`
  / `revenueComponents` / `percentCommission`. An explicit `minPlanValue: 0` in the
  override DOES win (it is a set value, not absent).
- `amountPerInstall`, `signupBonus`, `revenueType` are NOT commission fields and
  always fall through.

### 6.3 The scan
```
checkCommissionsForAffiliate(affiliate):
    cursor = max(affiliate.last_transaction_date, affiliate.start_counting_commission_from)
    for each batch of 200 transactions of affiliate's attributed installs,
        ordered by date asc, where date > cursor:
        for tx in batch:
            for each (program, attribution) attributing tx's install:
                processTransaction(affiliate, program, attribution, tx)
        affiliate.last_transaction_date = batch.last.date      # advance cursor (idempotent)
```

### 6.4 The per-transaction gate chain (each is a "skip/continue")
```
processTransaction(affiliate, program, attribution, tx):
    rules = resolveRules(program, membership, attribution)
    amount = rules.revenueType == "net" ? tx.netAmount : tx.grossAmount

    if tx.grossAmount <= 0:                          continue   # refund/credit SILENTLY SKIPPED (no clawback)
    if tx.date < program.start_counting_commission_from:  continue
    if tx.date < affiliate.start_counting_commission_from: continue
    component = ({subscription_sale:"subscription", usage_sale:"usage",
                 one_time_sale:"one-time"})[tx.type]
    if rules.revenueComponents?.length and component not in rules.revenueComponents: continue
    if rules.minPlanValue and install.activeSubscription
       and install.activeSubscription.plan.amount < rules.minPlanValue:            continue
    if commissionExists(affiliate, tx):              break      # dedup: one tx → at most one program's commissions

    # ---- mint commissions ----
    earliest = earliestCommission(affiliate, program, install)

    # SIGNUP (one-time): only if no earlier commission exists for this program+install
    if not earliest and rules.signupBonus:
        createCommission(type="signup", amount=parseFloat(rules.signupBonus), tx)

    # PERCENT (recurring, per eligible transaction):
    if rules.percentCommission:
        windowStart = earliest?.date                 # window starts at FIRST commission's date, else null
        durationMonths = rules.durationMonths || 0
        if windowStart and durationMonths and tx.date > addMonths(windowStart, durationMonths):
            # outside the window → skip percent
        else:
            createCommission(type="percent",
                             amount = amount * (parseFloat(rules.percentCommission) / 100), tx)
```

### 6.5 The other two commission types
- **INSTALL** (one-time per install): `createInstallCommission` at attribution time
  (§5). `amount = parseFloat(rules.amountPerInstall || 0)`; skip if ≤ 0; dated the
  install's `installed_at`; not subject to `durationMonths`.
- **CUSTOM**: only created by a manual admin action, never by the engine.

### 6.6 Cancellation (the only clawback path)
There is **no automatic refund clawback**. Voiding is a manual admin action that
sets `cancelled = true`, nulls `payout_id`, and re-runs payout generation. A
cancelled commission counts as zero in all totals.

### 6.7 Totals
- `totalCommission` = `sum(amount)` where `deleted_at = null AND cancelled = false`.
- `outstandingCommission` = same, further restricted to `payout_id = null OR
  payout.status = pending`.
- Both return **null (not 0)** when nothing matches — handle the empty case.

---

## 7. Procedure: payout aggregation (no money moves)

```
generatePayoutsForAffiliate(affiliate):
    lock(`AffiliatePayout:${affiliate.id}`, 5min):              # per-affiliate
        outstanding = commissions(affiliate, payout_id=null, deleted_at=null, cancelled=false)
        total = sum(outstanding.amount)
        minimum = fundingSource.minimum_payout_amount ?? org.affiliate_minimum_payout_amount ?? 0

        payout = openPendingPayout(affiliate)
        if not payout:
            if total < minimum: return                          # MUST gate NEW payout on minimum
            payout = createPayout(
                number      = max(lastOrgPayout.number + 1, org.affiliate_payout_start_number || 1),
                status      = "pending",
                period_start= previousPaidPayout?.period_end,    # chain periods
                period_end  = now)
            emit AffiliateEvent("payout_pending")

        stamp outstanding with payout.id
        payout.amount = sum(payout.commissions.amount)
```

- **MUST gate only NEW payout creation on the minimum.** Once a pending payout
  exists, more commissions append even below the minimum.
- **MUST run under a per-affiliate lock** and re-read inside it (idempotent re-run).
- Payout `number` is a per-merchant sequential invoice number, distinct from `id`.
- In Mantle, generation is enqueued **off the commission sweep** (so it rides the
  daily 9am cron), per merchant.

---

## 8. Procedure: payment (two rails)

Both terminate in `markPayoutAsPaid(payout, method)` → `status="paid"`, `paid_at`,
`amount_paid`, `payment_method`.

### 8.a Rail A — PayPal Payouts (pass-through, no fee, simplest)
Merchant connects their own PayPal (§3.3). Push one batch to affiliate emails. No
up-front charge, no fee split. Reconcile async off the PayPal webhook.
**MUST use `affiliate.paypal_email || affiliate.email` as the recipient.**

### 8.b Rail B — Stripe Connect (you broker; optional fee)
Charge yourself for the batch total, then transfer to each affiliate's Express
account. Fee math (round each to 2 dp):
```
feePercent  = org.affiliate_payout_fee_percent        # default 0.10
feeSplit    = org.affiliate_payout_fee_split           # default 0.5
per payout:  fee          = round2(amount × feePercent)
             orgFee       = round2(fee × feeSplit)        # you absorb
             affiliateFee = round2(fee − orgFee)          # affiliate absorbs
batch:       amountCharged = Σ(amount + orgFee)           # what YOU pay in
             amountPaid    = Σ(amount − affiliateFee)      # what affiliates RECEIVE
```
Defaults → org pays 1.05×, affiliate gets 0.95×, you keep the full 10%. Zero
`feePercent` to not monetize.

**MUST** on this rail: idempotency key on the PaymentIntent (the batch id);
re-query the batch's payouts inside the charge to avoid double-pay under a race;
the §3.4 live-capability gate before charging; **one funding source per batch**
(reject mixed Stripe customers); and a **partial-failure reconcile** — only an
all-fail batch auto-refunds, so a separate sweep must refund the net shortfall
when some transfers succeed and some fail, treating Stripe's own refund list
(incl. manual dashboard refunds) as source of truth.

### 8.c Status lifecycle
`pending → requested → processing → paid`; `cancelled` detaches the commissions
back to `payout_id = null` (so they re-aggregate) and is refused once
paid/processing. Both `pending` and `requested` are payable.

### 8.d Auto-payout (optional)
A daily cron (Mantle: `0 0 10 * * *`) checks each merchant's
weekly/biweekly/monthly schedule (with a catch-up window + `last_run_at` guard),
selects eligible payouts — **excluding `payout_hold` affiliates and those without
a connected payout account, after the live-capability check** — groups by funding
source, and runs the batch off-session.

---

## 9. Affiliate portal & auth

A **separate auth stack** from your merchant/admin login — a signed session cookie
holding `{ id, email, otpVerified }`.

- **Passwordless / OTP-first.** POST email → find the affiliate-user (401 if none).
  **MUST NOT check the `password` field at login** (it is legacy). Write a 6-digit
  OTP to a short-TTL store (10 min, reused within TTL), email it as a code **and**
  a magic link. Verifying the OTP sets `otpVerified = true` and stamps
  `email_verified_at = now`.
- **"Remember me" is recomputed server-side** on each authenticated fetch as
  `email_verified_at within 30 days OR google_id_token present` — the stored
  session flag is not authoritative for the window.
- **Google OAuth** optional (code exchange → decode id_token → find/create user →
  `otpVerified = true`).
- **Identity linking (MUST):** on affiliate-user creation, connect any unlinked
  `affiliates` rows with the same (case-insensitive) email. This is what lets a
  pre-created/imported affiliate be adopted when the person signs up.
- **Two-tier terms:** merchant-level (`affiliate.agreed_to_terms_at`) and
  program-level (`membership.agreed_to_terms_at`).

Groups (rate tiers), assets (creative), a public marketplace, and Liquid branding
are **nice-to-haves** — none are required for attribution or commissions.

> **MUST scope asset downloads.** A real bug to avoid: authorize the *membership*
> (it belongs to the user) AND look up the asset **scoped to that membership's
> program**, enforcing the asset's own `visibility` / `group_ids` / `affiliate_ids`
> — do not fetch an asset by id alone, or any logged-in affiliate can read any
> asset across merchants. Guard the not-found/null case as a clean 404.

---

## 10. Events & webhooks

Route **every** lifecycle transition through one chokepoint that writes an
immutable event row and fans out to (1) outbound HTTP webhooks and (2) operator
notifications.

**The event types** (exactly these in Mantle): `join_requested`, `join_approved`,
`join_denied`, `joined`, `referral`, `referral_requested`, `payment_requested`,
`payout_pending`, `payout_cancelled`.

- **MUST NOT assume the obvious events exist.** There is **no `commission_earned`
  and no `payout_paid`** — commission earning is folded into `referral`; the money
  lifecycle is only `payout_pending` / `payment_requested` / `payout_cancelled`.
  **Add `commission_earned` / `payout_paid` yourself** if your customers want them.
- **Outbound webhook signing** (copy verbatim for integrator compatibility):
  ```
  secret  = webhook.signSecret || webhook.client.secret
  X-Mantle-Hmac-SHA256 = hex( HMAC-SHA256(secret, `${unixSeconds}.${JSON.stringify(payload)}`) )
  headers: X-Timestamp (unix seconds), X-Mantle-Webhook-Topic, X-Mantle-Hmac-SHA256, X-Mantle-Org-Id
  POST, 5s timeout; non-2xx throws → retried (attempts: 10, exponential backoff, 1s base)
  ```
- **Notifications** are gated by a subscription model (which event types, which
  channel — slack/discord/email, recipients), deduped per `(event, rule, recipient)`.

---

## 11. MUST / MUST NOT rules (the gotchas)

Each encodes a real bug or hard lesson. Treat as acceptance criteria.

- **MUST scope handle resolution to the app/program.** Same handle can exist in
  another program.
- **MUST pick and consistently apply ONE de-dup policy** for "one active
  attribution per install," guarded by a per-install lock. Mantle is non-uniform
  across entry paths — that inconsistency is itself the bug.
- **MUST treat rule resolution as a non-merge** — overriding drops the parent's
  commission fields (`percentCommission`, `durationMonths`, `revenueComponents`,
  `minPlanValue`), even with an empty override. **Port the rule-resolution unit
  tests verbatim.**
- **MUST decide your refund policy on purpose.** A non-positive transaction
  (`grossAmount <= 0`) is *silently skipped, not clawed back*. Cancellation is
  manual-only. If you promise "claw back on refund," build it.
- **MUST start the percent window at the first commission's date,** not the
  attribution date.
- **MUST attach the signup bonus to the first *eligible* transaction** (it fires
  only when no earlier commission exists).
- **MUST dedup per `(affiliate, transaction)`** — one transaction yields at most
  one program's commissions; the dedup `break`s the program loop.
- **MUST gate only NEW payout creation on the minimum** — commissions still append
  to an existing pending payout below it.
- **MUST run aggregation and payment under per-affiliate locks**, and key every
  external charge/transfer with an idempotency key.
- **MUST gate Stripe transfers on the connected account's LIVE `transfers`
  capability** before charging — fail closed (real money-leak fix).
- **MUST handle Stripe partial-batch failure** — you charged the full batch but
  only some transfers landed; reconcile the shortfall (only all-fail auto-refunds).
- **MUST use `paypal_email || email`** as the PayPal recipient.
- **MUST scope affiliate-asset downloads** to the membership's program and the
  asset's visibility (see §9) and guard the null case.
- **MUST NOT check the portal `password` field at login** — OTP is the gate.
- **MUST link affiliate rows by case-insensitive email** on user creation
  (pre-creation adoption).
- **Two email-uniqueness scopes** — global `affiliate_user.email`, per-merchant
  `affiliate.email`. Keep them distinct, linked by id.
- **Re-importing an affiliate by email revives a soft-deleted record** (clears
  `deleted_at`). Decide if that's desired.
- **MUST treat the event/audit table as non-load-bearing** — never let a webhook
  or notification failure block or corrupt a commission/payout write.

---

## 12. Caveats (read before going to production)

- **Currency.** Mantle does **no FX anywhere**; amounts are in the transaction's
  native units and the importers hardcode `USD`; PayPal items post `currency:
  "USD"`. If you serve non-USD merchants, normalize currency explicitly across
  transaction amount, commission amount, and payout — don't assume single-currency.
- **Float precision.** Amounts are stored via `parseFloat` with no rounding at
  creation (rounding happens only in the Stripe fee math and reconcile). If you
  need exact money, store integer minor units.
- **Skip the Shopify-only scaffolding.** The Google-Analytics→BigQuery attribution
  reconstruction exists solely because the App Store strips query params; drop it
  if you own your funnel. The competitor-importer pipeline is Mantle-specific —
  reuse only the import *shape* (match by case-insensitive email; import only
  already-paid historical payouts; set a per-affiliate
  `start_counting_commission_from` so you don't regenerate pre-migration
  commissions).

---

## 13. Acceptance scenarios

Implement and verify each:

1. **Enroll.** Joining issues a unique handle; re-joining returns the same
   membership unchanged. Approval emits both `join_approved` and `joined`.
2. **Link + capture.** The shared link carries `?mref=<handle>`; the client script
   stores the code 30 days and your signup forwards it.
3. **Attribution.** Forwarding the code writes ONE attribution (affiliate +
   program + install), fires the install bounty, resets the re-scan cursor, and
   emits `referral`. A second referral on the same install obeys your stated
   de-dup policy. Concurrent attribution doesn't create two active rows (lock).
4. **Handle scoping.** The same handle string in two programs resolves to the
   correct program when scoped by app.
5. **Percent commission.** An eligible subscription transaction mints a `percent`
   commission = `amount × pct/100`; each subsequent eligible transaction mints
   another until the `durationMonths` window (from the first commission's date)
   closes.
6. **Signup bonus.** Fires once, on the first eligible transaction; not again.
7. **Rule override non-merge.** Setting a membership to `custom` with `{}` drops
   the program's `minPlanValue`/`durationMonths`/`revenueComponents`/
   `percentCommission`; `amountPerInstall`/`signupBonus`/`revenueType` still apply.
8. **Gates.** A refund (`grossAmount <= 0`) mints nothing and is not clawed back;
   a sub-`minPlanValue` install mints nothing; a `revenueComponents`-excluded
   transaction mints nothing.
9. **Dedup.** One transaction never produces commissions for two programs.
10. **Payout aggregation.** Outstanding commissions roll into a `pending` payout
    only once past the minimum; below-minimum commissions still append to an
    existing pending payout; the payout gets the next per-merchant number and a
    chained period.
11. **PayPal payment.** Batch pays to `paypal_email || email`; `SUCCESS` webhook
    marks paid; `RETURNED`/`FAILED` reverts to pending.
12. **Stripe payment.** Charge gated on live `transfers` capability + idempotency
    key; fee split matches the formula; a partial-failure batch reconciles the
    shortfall.
13. **Cancellation.** Cancelling a commission nulls its `payout_id`, zeroes it in
    totals, and re-runs aggregation.
14. **Auto-payout.** Runs on the merchant's cadence; skips `payout_hold` and
    accounts without a connected payout method.
15. **Portal.** OTP login works without a password; "remember me" honors the
    30-day verified window; a pre-created affiliate is adopted on signup by
    matching email; an asset download cannot read another program's asset id.
