---
name: mantle-flows-to-n8n
description: >-
  Migrate Mantle Flows (lifecycle automations — trial nurture, dunning, churn
  win-back, onboarding, usage upsell) into n8n (or Make / Zapier). Use when the
  user wants to recreate their Mantle flows elsewhere, rebuild the "derived"
  trigger events Mantle computed (trial-expiring, usage-threshold, enriched
  churn), avoid stranding customers mid-sequence during cutover, or generate
  importable n8n workflow JSON from their existing flows.
---

# Mantle Flows → n8n

## What this skill does

Helps a partner **migrate off Mantle Flows onto n8n** (or Make / Zapier). A flow
is a graph of **triggers → conditions → waits → actions**; recreating the
boxes-and-arrows is easy. The two parts that actually take work — and that this
skill is built around — are:

1. **The derived events stream.** Mantle *computed* many triggers on a schedule
   ("trial expiring in 3 days," "usage crossed the cap," enriched churn reasons).
   n8n only receives **raw** webhooks. You must **rebuild each derived event** as
   a Schedule-Trigger scan with a fire-once guard, or the dependent flows never
   fire.
2. **In-flight sequences.** A customer parked in a "wait N days" step exists only
   as a scheduled wake-up inside Mantle and **cannot be transferred** to n8n. If
   you cut over without a plan, those customers are silently abandoned
   mid-sequence.

The full mechanics live in **[`references/implementation-spec.md`](./references/implementation-spec.md)**.
Read it before building. This file is the *procedure*; the spec is the
*reference*. Two ready-to-use artifacts ship alongside it:
[`references/inventory-template.md`](./references/inventory-template.md) (catalogue
your flows first) and [`references/n8n-workflow-prompt.md`](./references/n8n-workflow-prompt.md)
(generate importable n8n JSON per flow).

## When to use it

Trigger on intents like: "migrate my Mantle flows to n8n," "recreate my trial /
dunning / win-back automations," "Mantle is shutting down, how do I move my
automations," "rebuild the events that trigger my flows," or "turn my flows into
n8n workflows."

## How to drive the migration

Work through these phases. **Inventory before you build** — the catalogue is what
reveals the real scope (the derived events and the in-flight populations).

### Phase 1 — Orient (ask, then read)
1. Confirm the target: **n8n** (default), Make, or Zapier. The spec's mapping
   table covers all three; the generation prompt targets n8n JSON.
2. Find where the partner's flows live today (a Mantle export, screenshots, or a
   list). Determine: which events trigger them, what conditions/waits/actions
   each has.
3. Establish what the partner already has in n8n: an instance, webhook intake,
   and an API/DB the flows can query and write back to.

### Phase 2 — Read the reference
Read [`references/implementation-spec.md`](./references/implementation-spec.md)
end to end. §0 (the two hard parts), §2 (raw vs derived), and §3 (in-flight
cutover) are the non-negotiable core; §7 is the MUST/MUST-NOT checklist.

### Phase 3 — Inventory
Catalogue every flow using
[`references/inventory-template.md`](./references/inventory-template.md). Per flow
record: trigger + **trigger kind (raw / derived / scheduled / manual)**,
conditions, ordered steps (waits + actions), the **in-flight count** (customers
parked in a wait right now), and the derived-event dependencies. Then collapse
the derived events into a **de-duplicated backlog** (many flows share one scan).

### Phase 4 — Rebuild the events stream (the real work)
For each distinct **derived** event, build one small n8n workflow: **Schedule
Trigger → query the partner's data → fire-once guard → continue the flow**
(spec §2.2–2.3). Build the **raw** intake too: register Shopify/Stripe webhooks
to n8n Webhook nodes, and emit in-app events for app-action triggers.

### Phase 5 — Rebuild the flows
One n8n workflow per Mantle flow, via the mapping table (spec §4) and action
catalog (spec §5): trigger → conditions (IF/Switch ahead of actions, both
branches wired) → waits (Wait nodes) → actions. **Write-back actions** (tag, set
field, extend trial, assign owner, CRM) are **HTTP Request** nodes to the
partner's own API — ask for the endpoint and auth; never invent them. To
accelerate, generate each flow with
[`references/n8n-workflow-prompt.md`](./references/n8n-workflow-prompt.md) and
import the JSON.

### Phase 6 — Wire and verify
Bind credentials, replace every placeholder (URLs, field names, channels), and
**test each flow with one real customer** end to end (trigger → branch → wait →
action). Confirm the fire-once guard prevents re-sends on the next scheduled
scan.

### Phase 7 — Cut over (the in-flight plan)
For **each flow with a wait step**, execute the spec §3 plan: count the parked
population, pick **drain** (default) / re-enroll / abandon, stop new Mantle
enrollments, run both engines in parallel for one full sequence length, then
decommission. **Never hard-cut a flow with parked customers.**

## Guardrails

- **Classify every trigger raw vs derived first** — the derived set is the
  migration's real scope. Re-pointing webhooks is trivial; rebuilding the scans
  is the project.
- **Every derived scan needs a fire-once guard** — without it, a daily scan
  re-sends to the same customer every day.
- **Never strand in-flight customers.** Count the parked population per
  wait-bearing flow before disabling it. Default to draining; never hard-cut a
  flow that has people parked in a wait.
- **Write-back actions are HTTP calls to the partner's own API** — ask for the
  endpoint and credentials; do not fabricate them.
- **Conditions are branch nodes (IF/Switch) ahead of the actions** — and wire
  both branches.
- **Don't rebuild Mantle's internals** (status tables, version snapshots, the
  wake-up queue, bulk-run staging). Reproduce *behavior*, not machinery.
- **Generated/imported JSON is a skeleton** — credentials, endpoints, and field
  names are environment-specific. Verify with a real test before enabling.
- **De-duplicate derived scans across flows** — build the shared "trial-expiring"
  or "usage" scan once.
