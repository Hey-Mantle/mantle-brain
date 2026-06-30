# Affiliate Programs on Shopify — a build guide

*Part of [mantle-brain](../README.md): an open-source library to help Shopify
Partners recreate Mantle's tools as Mantle winds down.*

This guide explains **how Mantle's Affiliates system worked and how to rebuild it
in your own app** — a referral/partner program where affiliates share a tracked
link, the installs they drive are attributed to them, the revenue those installs
generate earns commission, and that commission is paid out through your own
payment provider.

If you'd rather hand the job to a coding agent, this folder ships two companions:

- **[`implementation-spec.md`](./implementation-spec.md)** — a dense, imperative
  build spec (data model, exact API shapes, pseudocode, acceptance tests). Paste
  it into Claude/Cursor and say "implement this in my app."
- **[`skill/shopify-affiliate-program/`](./skill/shopify-affiliate-program/)** — a
  Claude Code / Agent skill. Copy it into your app's `.claude/skills/` and the
  agent will walk the build for you. See its
  [README](./skill/shopify-affiliate-program/README.md).

---

## The problem it solves

You want other people — agencies, influencers, content creators, existing
customers — to drive installs of your Shopify app, and you want to pay them a cut
of the revenue those installs produce. That needs four things to hang together:

- a **tracked link** each partner can share, so you know who referred whom;
- **attribution** that survives the gap between a click and an install (often days
  later, on a different page, after the Shopify App Store has stripped your query
  params);
- a **commission engine** that turns each referred install's billing into money
  owed, under rules you control (percentage, recurring vs one-time, eligibility
  gates);
- and a **payout step** that actually moves money to the affiliate.

Mantle built all four. The whole thing is organized around one unit of identity.

---

## The core idea

> Every affiliate, in every program, gets a unique **handle** — the `mref` code.
> That one string is the spine of the entire pipeline.

```
   Affiliate joins a program ──► issued a unique handle (the mref code)
           │
           ▼
   shares a link:  https://apps.shopify.com/your-app?mref=<handle>
           │
   visitor clicks ──► tiny JS snippet stores the mref in a cookie (~30 days)
           │
   visitor installs / signs up ──► your app forwards the stored mref
           │
           ▼
   ATTRIBUTION   one row links  (affiliate + program + this install)
           │
   install gets billed over time (subscription / usage / one-time)
           │
   a daily SWEEP reads those transactions ──► COMMISSION rows per program rules
           │
   commissions accumulate ──► aggregated into a PAYOUT when past a minimum
           │
           ▼
   PAID   disbursed through your payment provider, then marked paid
```

Four stages — **referral → attribution → commission → payout** — and a handful of
support systems around them (a join/approval flow, an affiliate-facing portal, an
event/webhook fan-out).

The single most replication-critical, easy-to-get-wrong piece is **commission
rule resolution** (how a program's rate, a group's rate, and a per-affiliate
override combine). It is a *deliberate non-merge*, and it's where the money bugs
live. More on that below.

---

## Mental model

| Concept | What it means |
|---|---|
| **Affiliate user** | A *global* login identity — one person, unique email, who can be an affiliate across many merchants/programs. Holds payout identity (where to send their money) and the portal login. |
| **Affiliate** | A *per-merchant* participant record that actually earns commission. Linked to an affiliate-user by email. (If your app is single-tenant, this and the affiliate-user collapse into one row.) |
| **Program** | A referral program you (the merchant) own. Holds the commission **rules** and the join policy. |
| **Membership** | Joins an affiliate to a program. **Its `handle` field *is* the `mref` referral code.** Carries the join status and any per-affiliate rule override. |
| **Handle / mref** | The referral code, unique within a program. Appears as `?mref=<handle>` in the shared link. |
| **Attribution** | The output of attribution: one row linking *affiliate + program + install*. At most one active attribution per install. |
| **Commission** | One earned amount, tied to a billing transaction and/or an install. Has a `type` (`signup` / `percent` / `install` / `custom`) and a `cancelled` flag. |
| **Rules (JSON)** | The program economics — a small JSON blob (percent, duration, per-install bounty, signup bonus, eligibility gate, which revenue counts). Can live on the program, a group, a membership, or be snapshotted onto an attribution. |
| **Group** | A named rate *tier* inside a program (its own rules). Optional. |
| **Payout** | A mutable bucket that aggregates an affiliate's uncancelled, unpaid commissions over a period. Carries a per-merchant sequential invoice **number**. |
| **Event** | An immutable lifecycle-event row (joined, referral, payout_pending…). The spine of the webhook + notification fan-out. |

The most important structural fact: **commissions are not created at billing
time.** A daily sweep reads each referred install's transactions and *derives*
commissions from them. This keeps the system idempotent and lets you re-run it
safely — but it means there's a lag between a charge and the commission.

---

## How it lives — end to end

### 1. An affiliate joins a program
You either pre-create them (CSV import, or an invite) or they request to join
(optionally gated by approval and terms acceptance). On join they're issued a
unique `handle` derived from their name/username/email with a collision suffix.
Status becomes `enrolled` (or `pending` if the program requires approval).

### 2. They share a link
`link = baseUrl + (baseUrl.includes("?") ? "&" : "?") + "mref=" + handle`, where
`baseUrl = program.customLink || "https://apps.shopify.com/<your-app>"`.
The `apps.shopify.com` base is the only Shopify-specific bit — swap in your own
landing/signup URL for a non-App-Store funnel.

### 3. A visitor clicks
A ~200-line vanilla-JS snippet on your marketing pages reads `mref` (falling back
to `utm_source`, then `ref`) from the URL, stores it in a cookie for 30 days
(falling back to `sessionStorage`), and rewrites your signup links to carry the
code. It's directly reusable.

### 4. The visitor installs / signs up
Your app reads the stored code and forwards it to your attribution endpoint at
account-creation time. (In Mantle this was the `/v1/identify` call with an `mref`
field.)

### 5. Attribution
Resolve the handle → the enrolled membership → the affiliate, scoped to your app.
Write **one** attribution row linking affiliate + program + install. Then fire the
side effects: create the one-time install bounty, reset the affiliate's
"last scanned" cursor so the sweep re-checks history, and emit a `referral` event.

### 6. Transactions become commissions (the daily sweep)
A daily job scans each referred install's billing transactions since a per-
affiliate cursor and, for each one, applies the program rules to mint commission
rows (percentage of revenue, a one-time signup bonus, etc.). See
[the commission math](#the-commission-math).

### 7. Commissions roll into a payout
A second step buckets an affiliate's outstanding (uncancelled, unpaid)
commissions. Once the running total clears a minimum threshold, it opens a
`pending` payout with a sequential number and a billing period.

### 8. The payout is paid
You disburse the payout to the affiliate through your own payment provider (PayPal
Payouts, Stripe Connect, bank transfer — your call; that integration is out of
scope for this guide) and mark it `paid`. An auto-payout scheduler can run this on
a weekly/biweekly/monthly cadence.

Alongside all of this runs an **affiliate portal** (a separate login where
affiliates see their stats, grab creative assets, and request payouts) and an
**event fan-out** (every lifecycle transition emits an event that drives outbound
webhooks and operator notifications).

---

## The attribution mechanism

Attribution is the part most people underestimate, because of one Shopify-
specific wrinkle: **the App Store strips your query parameters on install**, so
the `mref` doesn't automatically arrive at your backend. There are two honest ways
to bridge that click→install gap:

1. **Carry it yourself (recommended).** Capture `mref` client-side, persist it in
   a cookie, and forward it from your own signup/identify call. This is fully
   under your control and is all you need if you own your funnel.
2. **Reconstruct it from analytics.** Mantle *also* matched installs back to
   referral clicks via a Google-Analytics→BigQuery bridge, because some App Store
   installs never pass through a page you control. **This is Mantle-specific
   scaffolding to work around the App Store — skip it** unless you have the same
   constraint.

However the code arrives, resolution is the same: find the **enrolled** membership
whose `handle` matches, **scoped to your app/program** (the same handle string can
exist in different programs), and write one attribution.

### Pick a de-dup policy on purpose
"One active attribution per install" is an application-level invariant, not a
database constraint. When a second referral lands on an install that's already
attributed, you must choose:

- **Last-write-wins** — the new affiliate takes over; the prior attribution is
  soft-deleted.
- **First-write-wins** — the original affiliate keeps the install; later referrals
  are ignored, guarded by a per-install lock.

**Pick one, write it down, and apply it consistently across every entry path,
enforced with a uniqueness constraint or a per-install lock.** A lock matters
because the natural ordering key (`createdAt`) doesn't reliably reflect commit
order under concurrency.

---

## The commission math

### The rules shape
Program economics are a small JSON blob. The full set of honored fields:

```js
{
  percentCommission: 20,                 // recurring % of each qualifying transaction
  durationMonths: 12,                    // commission window length; 0/absent = lifetime
  amountPerInstall: 5.00,                // one-time flat bounty per referred install
  signupBonus: 10.00,                    // one-time bonus on the install's FIRST eligible transaction
  minPlanValue: 49.00,                   // gate: skip if the active plan's price is below this
  revenueComponents: ["subscription"],   // subset of ["subscription","usage","one-time"]; empty = all
  revenueType: "gross"                   // "gross" or "net" — which side of the transaction to pay on
}
```

Note what's **not** there: **no cap field.** The only things that bound a
commission are `durationMonths` (a time window) and `minPlanValue` (an eligibility
gate). If you promise affiliates a "max payout per referral," you're building that
yourself.

### Rule resolution — the subtle, money-critical part
Rules can be defined at four levels. Precedence, highest first:
**attribution snapshot > membership (custom) override > group tier > program**.

The trap: this is **not a deep merge.** A `custom` membership override or an
attribution snapshot **drops all four** *commission fields* (`percentCommission`,
`durationMonths`, `revenueComponents`, `minPlanValue`) from the parent. A `group`
override is gentler — a **per-field merge** that drops only the commission fields
the group itself sets, leaving the rest to fall through from the program.

> Switching a membership to a `custom` override — *even with an empty `{}`* —
> strips the program's `minPlanValue`, `durationMonths`, `revenueComponents`, and
> `percentCommission`. A custom flat rate does **not** silently inherit the
> program's eligibility gates. (The non-commission fields — `amountPerInstall`,
> `signupBonus`, `revenueType` — *do* still fall through, at every tier.)

This is the logic that's easiest to get wrong when you reimplement it. If you take one thing from
this guide: **port the rule-resolution unit tests verbatim** and treat that
function as load-bearing.

### The engine is a sweep, not inline
Commissions are derived by a daily job (plus an immediate re-run when an
attribution changes), not created when a charge happens. The sweep pages an
affiliate's transactions since a stored cursor (`lastTransactionDate`), so it's
incremental and idempotent. Per transaction it walks a chain of skip-gates:

1. Pick `gross` or `net` amount per the rules.
2. **Skip non-positive amounts** — a transaction whose *gross* amount is `<= 0`
   (a refund or credit) earns nothing and is not clawed back. (This is a policy
   choice — see "design decisions" below.)
3. Skip if the transaction predates the program's or the affiliate's
   "start counting from" cutoff.
4. **revenueComponents filter** — skip if this transaction's kind
   (subscription / usage / one-time) isn't in the allow-list.
5. **minPlanValue gate** — skip if the install's *active* subscription is priced
   below the threshold.
6. **Dedup** — if a commission already exists for this `(affiliate, transaction)`,
   stop; one transaction yields at most one program's commissions.

### The formulas
- **Percent** (recurring, per eligible transaction): `amount = transactionAmount ×
  percentCommission / 100`. The window is `durationMonths` long and **starts at
  the date of the affiliate's *first* commission** on that install — not the
  attribution date. A new percent commission is minted for *every* eligible
  transaction until the window closes. No monthly cap.
- **Signup bonus** (one-time): a flat `signupBonus`, created only when no earlier
  commission exists for that program+install — so it attaches to the first
  commission-eligible transaction.
- **Install bounty** (one-time per install): a flat `amountPerInstall`, created at
  attribution time, independent of any transaction and not subject to the
  duration window. Skipped if ≤ 0.
- **Custom**: only ever created by a manual admin action, never by the engine.

### Totals
- **Total earned** = sum of commissions that are not deleted and not cancelled.
- **Outstanding** (payable) = the same, further restricted to commissions with no
  payout yet *or* on a still-`pending` payout. A cancelled commission counts as
  zero in both, even if it's sitting on a pending payout.

---

## Payouts

Two clearly separated stages: **aggregate** (no money moves), then **pay** (money
moves, through your payment provider).

### Stage 1 — Aggregation
Per affiliate, under a per-affiliate lock:
- Sum the outstanding commissions (`unpaid, uncancelled, not deleted`).
- Reuse the open `pending` payout if one exists, else create a new one — **but
  only create one if the running total clears the minimum threshold.** (Once a
  pending payout exists, further commissions append even below the minimum.)
- A new payout gets the next per-merchant sequential `number`, `status =
  pending`, and a billing period that **chains** off the previous paid payout's
  end. Emit a `payout_pending` event.

### Stage 2 — Payment
Disbursing a payout — the actual money movement — runs through **your own payment
provider** (PayPal Payouts, Stripe Connect, a bank transfer, whatever you already
use). **That integration is out of scope for this guide** — it's the part most
specific to your stack and your provider's API. All this layer needs from it is a
simple contract: hand a payable payout to the provider, and when the disbursement
is confirmed, mark the payout `paid` (set `amount_paid`); if it fails, leave it
`pending` so it re-aggregates. Idempotency, verifying the destination can receive
funds, any fee handling, and reconciling partial failures all live inside that
provider integration.

### Status lifecycle
`pending → requested → processing → paid`, plus `cancelled` (which detaches the
commissions back to unpaid so they re-aggregate; refused once paid/processing).
Both `pending` and `requested` payouts are eligible to be paid.

### Auto-payout
A daily scheduler checks each merchant's weekly/biweekly/monthly cadence (with a
catch-up window and a `lastRunAt` guard), selects eligible payouts — **excluding
affiliates on payout-hold and those without a usable payout method** — and
disburses them through your provider.

---

## The affiliate portal & auth

The portal runs on a **completely separate auth stack** from your merchant/admin
login — a signed session cookie holding the affiliate-user identity. Login is
**passwordless / OTP-first**: submit an email, get a 6-digit code (and a magic
link) by email, verify it. (There's a `password` field in the schema, but it isn't
checked at login — the OTP is the real gate.) "Remember me" is recomputed
server-side as "email verified within 30 days, or a linked Google account,"
so the stored session flag isn't the authority. Google OAuth is an optional
add-on.

**Identity linking** is the bit that makes pre-creation work: when an affiliate-
user signs up, any unlinked affiliate rows with the same (case-insensitive) email
are adopted. That's how a CSV-imported affiliate "becomes" a real login later.

Everything else the portal offers — program **groups** (rate tiers), downloadable
**assets** (creative with `{{affiliate_link}}` / `{{affiliate_code}}`
substitution), a public **marketplace** of joinable programs, and Liquid-templated
**branding** — is a nice-to-have. None of it is required for attribution or
commissions to work. Build the OTP login + identity linking first; add the rest
when you need it.

---

## Events & notifications

Every lifecycle transition flows through one chokepoint that writes an immutable
event row and fans out to (1) outbound HTTP webhooks and (2) operator
notifications. There are a fixed set of event types — join requested/approved/
denied, joined, referral, referral requested, payment requested, payout pending,
payout cancelled.

A reality check worth internalizing before you design your own: in Mantle there is
**no `commission_earned` and no `payout_paid` event.** Commission earning is
folded into the `referral` event; the money lifecycle surfaces only as
`payout_pending` / `payment_requested` / `payout_cancelled`. If your customers
will want "you got paid" notifications, **add those events yourself** — don't
assume the obvious ones exist.

If you expose outbound webhooks, the signing scheme is simple and worth copying so
verification is familiar to integrators:

```
X-Mantle-Hmac-SHA256 = hex( HMAC-SHA256(secret, `${unixSeconds}.${JSON.stringify(payload)}`) )
plus headers: X-Timestamp, X-Mantle-Webhook-Topic, X-Mantle-Org-Id
POST, 5s timeout, retried up to 10× with exponential backoff (1s base).
```

---

## Design decisions & sharp edges

These are the calls you have to make consciously when you build this — the places
where a quiet default has real consequences. Decide each one on purpose.

- **Rule override is intentionally not a deep merge.** Switching a membership to a
  `custom` override — even an empty one — drops the program's `minPlanValue`,
  `durationMonths`, `revenueComponents`, and `percentCommission` (a `group`
  override drops only the commission fields it sets). This is by design so a custom
  rate doesn't silently inherit program gates — but it's the #1 thing to get right.
- **Refunds are a policy choice.** A non-positive (`gross <= 0`) transaction earns
  nothing and isn't auto-reversed; voiding an existing commission is an explicit
  admin action. If you want to advertise "claw back on refund," build that
  deliberately — it isn't automatic.
- **The duration window starts at the first commission's date,** not the
  attribution date. A late first transaction shifts the whole window.
- **The signup bonus attaches to the first *eligible* transaction.** If the first
  transaction is a refund or fails a gate, the bonus lands on a later one.
- **Pick one attribution de-dup policy and enforce it.** When a second referral
  lands on an already-attributed install, first-write-wins and last-write-wins are
  both defensible — choose one, write it down, and back it with a uniqueness
  constraint or a per-install lock (so concurrency can't create two active rows).
- **The minimum threshold blocks only *new* payout creation.** Commissions still
  append to an existing pending payout below the minimum.
- **Scope affiliate-asset downloads.** When you add downloadable creative, look the
  asset up scoped to the requesting membership's program and honor its
  visibility/group/affiliate scoping — don't resolve an asset by id alone — and
  return a clean 404 (not a crash) for a missing/unauthorized id.
- **Two email-uniqueness scopes.** The global affiliate-user email is unique
  everywhere; the per-merchant affiliate email is unique only within a merchant.
  They're distinct identities linked by id.
- **Re-importing an affiliate by email revives a soft-deleted record** (clears its
  deleted flag). Decide whether that's what you want.
- **Currency is single-rail by default.** Amounts are in the transaction's native
  units with no FX, and the importers assume USD. If your merchants aren't all USD,
  normalize currency explicitly rather than assuming one.

---

## What to build vs. what to skip

**Reproduce (the real mechanics):**
1. The identity trio — a global affiliate-user login (drop if single-tenant), a
   per-merchant affiliate, and a membership whose `handle` is the `mref` code
   (unique per program).
2. Program config: the commission **rules** JSON (the 7 fields), a join policy,
   and a custom-link override.
3. The **client capture script** — reuse it almost verbatim (`mref || utm_source
   || ref`, 30-day cookie, link rewriting).
4. The **attribution write** — resolve the enrolled membership by handle (scoped
   to your app), write one row, pick a de-dup policy explicitly, fire the side
   effects (install bounty, re-scan trigger, referral event).
5. The **commission engine** — the exact rule-resolution precedence *including the
   commission-field omission*, plus a sweep over referred installs' transactions
   applying the skip-gate chain and the four commission formulas. **Port the unit
   tests.** Decide your refund policy on purpose.
6. **Payout aggregation** — per-affiliate bucketing, minimum gate, per-merchant
   sequential number, chained periods, idempotent re-run under a lock.
7. A **payout status machine** — `pending → requested → processing → paid`, plus
   `cancelled` — with a thin contract to your payment provider (hand it a payable
   payout; mark `paid` on confirmation, `pending` on failure). The provider
   integration itself (PayPal, Stripe, bank transfer, …) is **out of scope here.**
8. An **event emit-point + webhook signing**, and a notification subscription
   model with per-(event, rule, recipient) idempotency. Add the money events
   Mantle lacks if your customers want them.

**Substitute with your own infra:**
- The job queue and the distributed locks → any durable queue + any lock (or a DB
  unique constraint). **Keep the "sweep, not inline" idea** — or, if you create
  commissions inline in your billing webhook, dedup hard.
- Columnar analytics reads (ClickHouse) and search-index reads (Elasticsearch) →
  plain SQL/ORM queries. The `uncancelled / not-deleted / payout-status` filters
  are the part that matters.

**Skip (Mantle-specific scaffolding):**
- **The Google-Analytics→BigQuery attribution reconstruction.** It exists only
  because the Shopify App Store strips query params; if you control your funnel you
  never need it. This is the single biggest thing you can drop.
- **The competitor-importer pipeline.** If you want migration, copy only the
  *import shape* (match affiliates by case-insensitive email; import only already-
  paid historical payouts; set a per-affiliate "start counting from" cutoff so you
  don't regenerate commissions for pre-migration revenue) — not the prepare/preview
  job machinery.
- The internal flow/automation engine, OTEL metrics, audit-log plumbing, and
  super-user impersonation.

---

## Where to go next

- Building by hand? Work down the "Reproduce" list; reach for
  [`implementation-spec.md`](./implementation-spec.md) when you need exact API
  shapes and pseudocode.
- Want an agent to do it? Paste
  [`implementation-spec.md`](./implementation-spec.md) into your coding agent, or
  install the [skill](./skill/shopify-affiliate-program/) into your app's
  workspace.
