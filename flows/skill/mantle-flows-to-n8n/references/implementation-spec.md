# Flows → n8n — Migration Spec (for agents)

> **Audience:** a coding agent migrating a partner off **Mantle Flows**
> (lifecycle automation) onto **n8n** (or Make / Zapier). This is a precise,
> self-contained migration spec — no external files required. It is the
> reference bundled with the `mantle-flows-to-n8n` skill; the procedure for using
> it is in the skill's `SKILL.md`. The standalone n8n-generation prompt and
> inventory template travel alongside it in this `references/` folder.
>
> **Provenance:** distilled from the behavior of Mantle's Flows engine. This is
> a *migration* spec, not a clone of Mantle's internals: the goal is to
> reproduce the **observable behavior** of a partner's flows in n8n, not to
> rebuild Mantle's flow engine. Keep guidance at the level of "what the flow
> does," not "how Mantle stored it."

---

## 0. Goal & the two hard parts

Recreate the partner's Mantle flows as n8n workflows such that the **same events
cause the same actions for the same customers.** Two things make this more than
a boxes-and-arrows port:

1. **The derived events stream.** Mantle synthesized events from the partner's
   data on a schedule ("trial expiring in 3 days," "usage crossed the cap,"
   enriched churn). n8n only receives **raw webhooks**. The agent **MUST rebuild
   each derived event** as a scheduled scan, or the flows that depend on it never
   fire.
2. **In-flight state.** A customer parked in a "wait N days" step exists only as
   a scheduled wake-up inside Mantle. That state **cannot be migrated** into n8n.
   The agent **MUST produce a drain/re-enroll plan** for every flow containing a
   wait, or customers are silently abandoned mid-sequence.

Everything else is mechanical mapping (§4).

---

## 1. The anatomy of a flow (the model to map FROM)

A flow is a **directed graph of steps**. Map each flow to these four primitives;
the mapping table in §4 turns each primitive into an n8n node.

| Primitive | Definition | Key sub-kinds |
|---|---|---|
| **Trigger** | What starts the flow (entry trigger) or advances it to a step (step trigger). | `raw` event / `derived` event / `scheduled` (bulk) / `time.passed` (wait) / `manual` / `api`. |
| **Condition** | A guard evaluated **before** a step's actions. Failing it skips the step (or exits the customer). Never performs work. | property comparison, segment/filter, script, previous-action-output check. |
| **Wait** | A delay between steps. Stored as a scheduled wake-up; the customer is parked until it fires. | fixed duration (`wait N units`), until-a-moment (e.g. trial end). |
| **Action** | The effect a step produces, executed in order within the step. | see the action catalog in §5. |

Execution model (what to preserve, not how Mantle implements it):

- A customer **enters** on the entry trigger, subject to an **enrollment guard**
  (run-once vs allow-repeat, plus an optional "don't re-enter within X" window).
- The customer **walks the graph**: at each step, conditions are evaluated; if
  they pass, actions run in order; then the engine follows the step's
  connections to the next step(s). Fan-out = branch.
- A **wait** step parks the customer until its wake-up time, then continues.
- Per-customer **position** and per-step **status** are tracked so a customer
  isn't double-processed. (In n8n this becomes one execution per enrollment plus
  dedupe; you do not need Mantle's status tables.)

**Do NOT** try to reproduce Mantle's internal tables (step-status, action-status,
wake-up queue, version snapshots). They're implementation detail. Reproduce the
*behavior*: right trigger, right guards, right order, right delays, right
actions, no double-sends.

---

## 2. Triggers: raw vs derived (the classification that drives the work)

For **every** flow, classify its entry trigger (and any step trigger) as **raw**
or **derived**. This classification is the migration's critical path.

### 2.1 RAW — arrives as a webhook; becomes an n8n Webhook Trigger

Re-register the platform webhook and point it at an n8n Webhook node. No
computation.

- **Shopify:** subscribed/activated, upgraded, downgraded, cancelled,
  installed, uninstalled, reactivated/deactivated, one-time-charge activated,
  refunded, frozen/unfrozen, review created/edited, platform-plan changed.
- **Stripe:** invoice finalized, **invoice payment failed**.
- **In-app actions you already control:** custom-field changed, deal/task/contact
  lifecycle, list add/remove, checklist completed, affiliate events, manual/API
  triggers. (These fire from *your* app, so emit them yourself — a webhook or a
  direct workflow call.)

### 2.2 DERIVED — nothing pushes it; becomes a Schedule Trigger + query + fire-once guard

Each derived event is its **own small n8n workflow** that scans data on a
schedule and continues the dependent flow when a condition becomes true.

| Derived event | Detection rule | Rebuild |
|---|---|---|
| **trial-expiring-in-N-days** | trials whose end date == today + N | Schedule (daily) → query trials ending in N days → per match, continue flow. One scan per distinct N, or parameterize. |
| **trial-expired** | trial end ≤ now | Daily scan with end ≤ today, OR model as a Wait-until-trial-end inside the flow. |
| **usage-approaching-cap / usage-exceeds-limit** | running metered total crosses a % of cap / the cap | On usage write or on schedule, aggregate usage vs cap; fire on crossing; guard so it fires once per period. |
| **enriched uninstall / churn-reason** | raw uninstall + classification (reason bucket, tenure) | On the raw uninstall webhook, run a classification step (rules or an AI node) **before** the rest of the flow. |
| **win-back eligible** | cancelled ≥ N days ago AND not resubscribed | Schedule (daily) → query → continue win-back flow. |
| **SLA breach / approaching** | ticket open past threshold | Schedule (frequent) → query open tickets past SLA → continue. |

**Rule:** a trigger that is *a moment relative to data* ("N days before/after X,"
"when the total crosses Z") is **derived** → Schedule + query + guard. A trigger
that is *a thing that happened* (webhook landed) is **raw** → Webhook node.

### 2.3 Fire-once guard (MUST)

A scheduled scan re-finds the same rows on the next run. **MUST** dedupe on a
stable key (e.g. `customerId:eventType:targetDate`) using a small store (DB flag,
n8n static data, or a "Remove Duplicates" step), so a customer isn't
re-triggered on every scan. Without this, drips re-send daily.

---

## 3. In-flight cutover procedure (MUST run before disabling any flow)

A customer parked in a wait step is **invisible** (no error, no red dashboard)
and **non-transferable** (n8n can't inherit Mantle's pending wake-ups). For
**each flow that contains a wait step**:

1. **Count the parked population** — customers with a pending future wake-up in
   that flow. Record it (inventory "in-flight count"). **Zero → migrate freely.**
2. **Choose a strategy:**
   - **Drain (default, safest):** stop *new* enrollments in Mantle, route new
     entrants to n8n, and let Mantle finish the in-flight tail. Run both in
     parallel for at least one full sequence length, then decommission.
   - **Re-enroll:** export parked customers and bulk-trigger the n8n workflow for
     them. **Only if re-entry from the top is idempotent** (dedupe guards,
     repeat-safe messages) — else duplicate sends.
   - **Abandon:** accept the loss. Acceptable only for low-stakes nudges; **never**
     for anything tied to revenue or a promised follow-up.
3. **Notify if high-stakes** — a parked customer who silently never hears back is
   a broken promise; drain or re-enroll those.

**MUST NOT** hard-cut (disable in Mantle, enable in n8n same instant) a flow that
has a non-zero parked population. **MUST** prefer drain for any wait-bearing flow.

---

## 4. The mapping table (Mantle building block → n8n node)

| Mantle building block | n8n node (`type`) | Make | Zapier |
|---|---|---|---|
| Trigger — raw event | Webhook (`n8n-nodes-base.webhook`) | Custom webhook | Webhooks — Catch Hook |
| Trigger — derived/scheduled | Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) → query → dedupe | Scheduled scenario + search | Schedule + Find |
| Trigger — bulk run over a segment | Schedule Trigger → query → Split In Batches (`splitInBatches`) | Scheduled + iterator | Schedule + loop |
| Trigger — manual / API | Webhook or Manual Trigger (`manualTrigger`) | On-demand | Webhooks — Catch Hook |
| Condition (single) | IF (`n8n-nodes-base.if`) | Filter | Filter by Zapier |
| Condition (multi-way branch) | Switch (`n8n-nodes-base.switch`) | Router | Paths |
| Condition (drop non-matches) | Filter (`n8n-nodes-base.filter`) | Filter | Filter by Zapier |
| Wait — fixed duration | Wait (`n8n-nodes-base.wait`, mode *after time interval*) | Sleep / Delay | Delay For |
| Wait — until a timestamp | Wait (`n8n-nodes-base.wait`, mode *until specified time*) | Sleep until | Delay Until |
| Action — send email | Send Email (`n8n-nodes-base.emailSend`) / Gmail | Email | Email / Gmail |
| Action — Slack | Slack (`n8n-nodes-base.slack`) | Slack | Slack |
| Action — HTTP / custom webhook / custom API | HTTP Request (`n8n-nodes-base.httpRequest`) | HTTP | Webhooks by Zapier |
| Action — tag/untag, set field, extend trial, assign owner | HTTP Request → your app API or Shopify Admin API | HTTP / Shopify | Shopify / Webhooks |
| Action — execute script | Code (`n8n-nodes-base.code`) | Tools — function | Code by Zapier |
| Action — run AI agent | AI Agent (`@n8n/n8n-nodes-langchain.agent`) | OpenAI / Anthropic | AI by Zapier |
| Action — create deal/task/contact, log activity | HTTP Request → CRM (or native CRM node) | CRM / HTTP | CRM / Webhooks |
| Enrollment dedupe (run-once / repeat window) | Remove Duplicates (`removeDuplicates`) / DB flag / static data | Data store | Storage by Zapier |
| Liquid template `{{ customer.x }}` | n8n expression `{{ $json.customer.x }}` | Mapping | Field mapping |
| Previous-action output | `{{ $node["NodeName"].json.field }}` | Reference module | Reference step |

Structural rules:

- **One flow → one workflow.** One Mantle *step* usually expands to several n8n
  nodes: an IF (conditions) followed by one node per action, chained in `order`.
- **Step connections → node connections.** Fan-out → IF (2-way) / Switch (n-way).
- **Templates → expressions.** Translate every `{{ liquid }}` merge field into an
  n8n `{{ }}` expression. Unknown fields → leave a clearly-named placeholder.
- **Actions that write back to the partner's app** (tag, custom field, trial,
  owner, CRM objects) have **no generic node** — they're an **HTTP Request** to
  the partner's own API or the platform Admin API. The agent **MUST** ask for the
  endpoint/auth rather than invent it.

---

## 5. Action catalog (what a step can do → how to rebuild it)

Group the Mantle actions; each maps to an n8n node per §4.

- **Messaging:** send email, Slack message, send notification → Email / Slack /
  (HTTP to your notification system).
- **Integration:** HTTP request, custom API action, execute script, run AI agent
  → HTTP Request / Code / AI Agent.
- **Customer writes:** tag/untag, set custom field, add/remove from list, assign
  account owner, extend trial, apply trial feature, set spend limit → **HTTP
  Request to the partner's app API** (no native node).
- **CRM:** create/update deal, create task, create contact/customer, log
  activity, add timeline comment → CRM node or HTTP.
- **Support (ticket) actions:** tag/assign/set-priority/set-status/reply/note on
  tickets → HTTP to the helpdesk API. Rebuild only if the partner automates
  tickets.

**Async actions** (HTTP, AI agent, script) may not return instantly — in n8n the
node simply completes when the call returns; you don't need Mantle's async
delivery/retry tables. Use the node's built-in retry/error settings.

---

## 6. The n8n workflow JSON contract

n8n imports a workflow as JSON. The agent (or the bundled prompt) emits JSON of
this shape. Minimum viable contract:

```json
{
  "name": "<flow name>",
  "nodes": [
    {
      "parameters": { /* node-specific */ },
      "id": "<uuid>",
      "name": "<unique human label>",
      "type": "n8n-nodes-base.<nodeType>",
      "typeVersion": 1,
      "position": [<x>, <y>]
    }
  ],
  "connections": {
    "<source node name>": {
      "main": [ [ { "node": "<target node name>", "type": "main", "index": 0 } ] ]
    }
  },
  "settings": {},
  "pinData": {}
}
```

Rules the JSON MUST satisfy:

- **Exactly one trigger node** (Webhook or Schedule Trigger) as the entry.
- **Unique `name` per node**; connections reference nodes **by name**.
- **`position`** laid out left→right (e.g. x += 220 per step) so the import is
  readable.
- **IF** has two outputs (true=index 0, false=index 1); **Switch** has one per
  case. Wire both branches or the false branch silently drops items.
- **Wait** node configured for duration or until-timestamp per the source step.
- **Credentials are NOT embedded** — leave credential references empty/named for
  the human to bind in n8n. The same for unknown URLs/field names: explicit
  placeholders, never invented values.
- Emit **raw JSON only** (no markdown fences) when the target is direct import.

The full generation prompt is in [`n8n-workflow-prompt.md`](./n8n-workflow-prompt.md).

---

## 7. MUST / MUST NOT (the migration's load-bearing rules)

- **MUST classify every trigger raw vs derived** before building anything. The
  derived set is the real scope.
- **MUST rebuild each derived event** as Schedule + query + **fire-once guard**.
  A scheduled scan without a dedupe guard re-sends on every run.
- **MUST count the parked (in-flight) population per wait-bearing flow** before
  disabling it, and **MUST** pick drain/re-enroll/abandon deliberately. Default
  to **drain**; never hard-cut a flow with parked customers.
- **MUST treat write-back actions (tag/field/trial/owner/CRM) as HTTP calls to
  the partner's own API** — ask for the endpoint and auth; never invent them.
- **MUST translate conditions into branch nodes (IF/Switch) ahead of the
  actions**, not into the actions. Wire **both** branches.
- **MUST NOT** attempt to migrate Mantle's parked wake-ups into n8n — they're not
  transferable; re-enroll instead.
- **MUST NOT** reproduce Mantle-internal machinery (status tables, version
  snapshots, the wake-up queue, bulk-run staging). Reproduce behavior only.
- **MUST treat generated/imported JSON as a skeleton** — credentials, endpoints,
  and field names are environment-specific. Verify with a real test customer
  before enabling.
- **SHOULD de-duplicate derived scans across flows** — many flows share one
  "trial-expiring" or "usage" scan; build it once.
- **SHOULD preserve enrollment semantics** (run-once vs repeat, repeat-window)
  with a dedupe/guard, or customers re-run flows they shouldn't.

---

## 8. Migration procedure (end to end)

1. **Inventory.** List every flow; per flow record trigger + trigger-kind,
   conditions, ordered steps (waits + actions), parked count, derived-event
   dependencies. (Template: [`inventory-template.md`](./inventory-template.md).)
2. **Derived-event backlog.** Collect the distinct derived events across all
   flows. Build one Schedule-Trigger workflow per derived event, each with a
   fire-once guard. This is the bulk of the work.
3. **Raw intake.** Register Shopify/Stripe webhooks → n8n Webhook nodes. Emit
   your own in-app events for app-action triggers.
4. **Rebuild flows.** One n8n workflow per Mantle flow via §4/§5 — or generate
   each with [`n8n-workflow-prompt.md`](./n8n-workflow-prompt.md) and import.
5. **Wire credentials & field names**, replacing every placeholder.
6. **Verify** each flow with one real test customer end to end (trigger → branch
   → wait → action), including that the fire-once guard prevents re-sends.
7. **Cut over** per the §3 in-flight plan: stop new Mantle enrollments, run both
   in parallel, drain the tail, decommission.

---

## 9. Acceptance scenarios

Verify each:

1. **Raw trigger.** A Shopify/Stripe webhook lands → the matching n8n workflow
   runs → the action fires. (e.g. payment-failed → dunning email.)
2. **Derived trigger.** The daily scan finds a trial ending in 3 days → continues
   the flow once; the **next day's** scan does **not** re-fire for the same
   customer (fire-once guard works).
3. **Condition branch.** A customer who fails the condition takes the false
   branch (or exits); one who passes takes the true branch. Both branches wired.
4. **Wait.** A "wait N days" step parks the execution and resumes after N days
   with the next action.
5. **Write-back action.** A tag/custom-field action calls the partner's app API
   and the change is visible in their app.
6. **Enrollment guard.** A run-once flow does not re-enroll a customer who already
   completed it; a repeat-window flow respects the window.
7. **In-flight drain.** For a wait-bearing flow, parked customers in Mantle finish
   their tail while new entrants run in n8n — nobody is dropped mid-sequence.
8. **Generated JSON imports cleanly** into n8n with one trigger, wired
   connections, both IF branches present, and credential placeholders to bind.
