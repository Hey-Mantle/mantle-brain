# Flow Inventory Template

Catalogue your Mantle flows **before** you migrate. Filling this in is what
surfaces (a) the **derived events** you have to rebuild and (b) the **in-flight
populations** you have to drain. Migrate from the catalogue, not from memory.

One row per flow. Copy the table below and fill it in.

---

## Inventory

| # | Flow name | Trigger | Trigger kind | Conditions | Steps (waits + actions, in order) | In-flight count | Derived events to rebuild | Cutover strategy | n8n status |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Trial nurture | trial-expiring-in-3-days | derived | plan = Pro | email "trial ends soon" → wait 2d → IF still-on-trial → Slack AM | 142 | trial-expiring scan | drain | ☐ todo |
| 2 | Churn win-back | unsubscribed | raw | — | wait 14d → IF not-resubscribed → email "come back" | 38 | win-back scan | drain | ☐ todo |
| 3 | Upgrade thank-you | subscription.upgraded | raw | — | tag "upgraded" → Slack #wins | 0 | — | hard-cut (no waits) | ☐ todo |
| 4 | Dunning | invoice.payment_failed | raw | — | email "payment failed" → wait 3d → IF still-unpaid → email #2 | 7 | — | drain | ☐ todo |
| 5 | Usage upsell | usage-exceeds-limit | derived | — | email "you're hitting your cap" | 0 | usage-threshold scan | hard-cut | ☐ todo |
| … | | | | | | | | | |

---

## Column notes

- **Trigger** — the event that *starts* the flow. Use Mantle's trigger name
  (e.g. `subscription.upgraded`, `trial-expiring-in-3-days`).
- **Trigger kind** — one of:
  - `raw` — a webhook from Shopify/Stripe, or an action inside your own app. You
    get it for free; re-point the webhook. → n8n **Webhook Trigger**.
  - `derived` — Mantle *computed* it on a schedule (a moment relative to your
    data: "N days before/after X," "total crossed Z"). **Nothing pushes it.** You
    must rebuild it as a scan. → n8n **Schedule Trigger + query + fire-once
    guard**.
  - `scheduled` — a bulk run over a segment. → **Schedule Trigger + query +
    Split-In-Batches**.
  - `manual` / `api` — triggered by hand or by an API call. → **Webhook / Manual
    Trigger**.
- **Conditions** — the guards before the flow (or a step) proceeds. These become
  **IF / Switch / Filter** nodes in n8n.
- **Steps (in order)** — the sequence of **waits** and **actions**. Write them
  left-to-right exactly as they run, e.g. `email → wait 2d → IF still-trial →
  Slack`. This is the spine of the n8n workflow.
- **In-flight count** — how many customers are **parked in a wait step right
  now**. This drives your cutover strategy.
  - **0** → no one is mid-sequence; safe to hard-cut.
  - **> 0** → real people are waiting; do **not** hard-cut. Drain or re-enroll.
  - If you can't query the exact number, estimate from enrollment volume ×
    sequence length, and lean conservative.
- **Derived events to rebuild** — name the scan(s) this flow needs. **De-dupe
  across rows**: many flows share one "trial-expiring" or "usage-threshold" scan
  — build it once, list it on every row that needs it.
- **Cutover strategy** — one of:
  - `hard-cut` — disable in Mantle, enable in n8n same instant. **Only when
    in-flight count = 0.**
  - `drain` — keep Mantle running the in-flight tail; new entrants go to n8n; run
    both in parallel for one full sequence length, then decommission. **Default
    for any flow with waits.**
  - `re-enroll` — bulk-trigger the n8n workflow for parked customers. **Only if
    re-entry from the top is idempotent** (dedupe + repeat-safe messages).
  - `abandon` — accept the loss. **Only for low-stakes nudges**, never for
    revenue-tied or promised follow-ups.
- **n8n status** — your build tracker: `☐ todo` → `◐ built` → `◑ tested` →
  `☑ live`.

---

## Derived-events backlog (de-duplicated)

After filling the inventory, collect the distinct `derived` triggers here. Each
is one small Schedule-Trigger workflow to build **once** and share across flows.

| Derived event | Detection rule | Schedule | Fire-once key | Flows that depend on it | Status |
|---|---|---|---|---|---|
| trial-expiring-in-3-days | trial end == today + 3 | daily 09:00 | `customerId:trial3:endDate` | #1 | ☐ todo |
| usage-threshold | running usage ≥ cap | hourly | `customerId:usageCap:period` | #5 | ☐ todo |
| win-back | cancelled ≥ 14d ago, no resub | daily 10:00 | `customerId:winback` | #2 | ☐ todo |
| … | | | | | |

---

## Cutover checklist (per flow with waits)

- [ ] Parked (in-flight) population counted.
- [ ] Strategy chosen (drain / re-enroll / abandon) and justified.
- [ ] New enrollments in Mantle stopped; new entrants routed to n8n.
- [ ] If drain: both engines running in parallel for ≥ one full sequence length.
- [ ] If re-enroll: idempotency/dedupe verified so parked customers don't get
      duplicate messages.
- [ ] High-stakes sequences: parked customers accounted for (not silently
      dropped).
- [ ] Old flow decommissioned only after the tail has fully drained.
