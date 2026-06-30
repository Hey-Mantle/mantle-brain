# Flows — a migration guide (Mantle → n8n)

*Part of [mantle-brain](../README.md): an open-source library to help Shopify
Partners recreate Mantle's tools as Mantle winds down.*

This guide explains **how Mantle's Flows worked and how to rebuild them in
[n8n](https://n8n.io)** (or Make / Zapier) — the lifecycle automations you used
to send a trial-ending email, Slack your team on a churn, tag a customer after
an upgrade, or run a multi-step onboarding sequence.

The hard part of the migration is **not** drawing the boxes-and-arrows in a new
tool. It's two things Mantle did quietly on your behalf:

1. **Mantle computed a stream of *derived* events** ("trial expiring in 3 days,"
   "usage crossed the cap," "churned") that your flows triggered on. n8n only
   sees the *raw* webhooks from Shopify/Stripe. **You have to reproduce the
   derived stream yourself** — that's most of the real work.
2. **Mantle held the in-flight state** of every customer sitting in a "wait 3
   days" step. When Mantle stops, those waiting customers are **stranded
   mid-sequence**. If you don't plan for it, you silently drop people halfway
   through their onboarding.

This guide is about getting both of those right. The boxes-and-arrows part is a
mechanical mapping table near the end.

If you'd rather hand the job to a coding agent, this folder ships companions:

- **[`implementation-spec.md`](./implementation-spec.md)** — a dense, imperative
  migration spec (the anatomy, the derived-event recipes, the n8n node mapping,
  the in-flight cutover procedure). Paste it into Claude/Cursor.
- **[`n8n-workflow-prompt.md`](./n8n-workflow-prompt.md)** — a ready-to-use LLM
  prompt that turns a description of one Mantle flow into **importable n8n
  workflow JSON**.
- **[`inventory-template.md`](./inventory-template.md)** — a copy-paste template
  for cataloguing your existing flows *before* you migrate, so nothing is lost.
- **[`skill/mantle-flows-to-n8n/`](./skill/mantle-flows-to-n8n/)** — a Claude
  Code / Agent skill that walks the whole migration. See its
  [README](./skill/mantle-flows-to-n8n/README.md).

---

## The anatomy of a flow

Every Mantle flow — and every n8n workflow, Make scenario, or Zap — is built
from exactly four kinds of building block. If you can name these four for each
of your flows, you can rebuild them anywhere.

| Block | What it is | The one question it answers |
|---|---|---|
| **Trigger** | The event that *starts* the flow, or that *advances* it to a step. | "When does this run?" |
| **Condition** | A filter or branch that decides whether to continue (or which way to go). | "Should this customer continue?" |
| **Wait** | A delay — a fixed duration ("wait 3 days") or until a moment ("at trial end"). | "When does the next step happen?" |
| **Action** | The thing the flow *does* — send an email, tag a customer, call a webhook. | "What happens?" |

A flow is a **graph of steps**, not a straight list. Each step carries a trigger
(what wakes it), optional conditions (whether it proceeds), and an ordered list
of actions (what it does). Steps are wired together with connections, so a flow
can branch and fan out. A customer **enters** a flow on the entry trigger and
**walks the graph** one step at a time; the engine records where each customer
is and, for wait steps, *when to wake them up*.

```
   Trigger: subscription.trial_expiring_in_days (value: 3)
        │
        ▼
   ┌─────────────┐   conditions pass?
   │   Step 1    │── no ─────────────► (customer skips / exits)
   │ Condition:  │
   │ plan = Pro  │
   └─────────────┘
        │ yes
        ▼
   ┌─────────────┐
   │   Step 2    │  Action: send "your trial ends soon" email
   └─────────────┘
        │
        ▼  Wait: 2 days   ◄── in-flight state lives HERE (a scheduled wake-up)
   ┌─────────────┐
   │   Step 3    │  Condition: still on trial?  Action: Slack the AM
   └─────────────┘
```

The three things to internalize:

- **Triggers come in two flavours** — *push* (an event arrives: subscribed,
  uninstalled, payment failed) and *poll* (Mantle looked at your data on a
  schedule and decided something was true: "this trial expires in 3 days"). The
  poll flavour is the derived stream you have to rebuild — see the next section.
- **Waits are where customers live between steps.** A "wait 3 days" step doesn't
  busy-loop; the engine stores a wake-up time and moves on. That stored wake-up
  is the *only* record that a customer is mid-sequence. Hold that thought for the
  [in-flight warning](#the-in-flight-sequence-warning).
- **Conditions are guards, not actions.** They're evaluated *before* a step runs.
  A failed condition skips the step (or the customer), it doesn't "do" anything.

---

## Where triggers come from — the post-Mantle events stream

This is the crux of the migration.

Mantle sat between Shopify/Stripe and your flows and **manufactured events**.
Some it passed straight through; many it *computed*. When you leave Mantle, the
pass-through events are easy (Shopify and Stripe will still send them). **The
computed ones disappear** — and a lot of your most valuable flows trigger on the
computed ones.

### Raw events — you get these for free

These arrive as **webhooks** straight from the platform. In n8n they're a
**Webhook Trigger** node. Register the same webhook with Shopify/Stripe and
you're done.

| Event | Source |
|---|---|
| Subscribed / activated, upgraded, downgraded, cancelled | Shopify app-subscription / charge webhooks |
| App installed / uninstalled / reactivated | Shopify `app/uninstalled` etc. |
| One-time charge activated, refunded | Shopify |
| Frozen / unfrozen | Shopify |
| Invoice finalized, **payment failed** | Stripe (`invoice.payment_failed`) |
| Review created / edited | Shopify |

> ⚠️ Caveat: some "raw" events are *enriched* before they're useful. Mantle's
> uninstall event, for example, doesn't just say "uninstalled" — it classifies a
> **churn reason** (found-an-alternative, too-expensive, poor-support…) and
> computes time-on-app. The Shopify webhook gives you the bare fact; the
> classification was Mantle's. Decide whether you need the enrichment (see
> [derived events](#derived-events--you-must-rebuild-these)).

### Derived events — you must rebuild these

Nothing pushes these. Mantle **ran a schedule, queried your data, and emitted an
event when a condition became true.** That's a pattern you reproduce directly in
n8n with a **Schedule Trigger → query → branch** — one small workflow per
derived event, feeding the flows that depend on it.

| Derived event | How Mantle computed it | How to rebuild it (n8n) |
|---|---|---|
| **Trial expiring in N days** | Daily scan of subscriptions for trials whose end date falls N days out. | **Schedule Trigger** (daily) → query your DB/Shopify for trials ending in N days → for each, continue the flow. |
| **Trial expired** | Compared trial-end to "now." | Same daily scan, end-date = today (or just trigger off the trial-end wait). |
| **Usage approaching cap / exceeding limit** | Aggregated metered usage against the plan cap; emitted when a threshold was crossed. | On each usage write (or on a schedule), **aggregate usage vs. cap**, fire when it crosses. Track "already fired" so it fires once. |
| **Churn reason / enriched uninstall** | Classified the raw uninstall (often with an LLM) and computed tenure. | On the raw uninstall webhook, run a **classification step** (rules or an AI node) before the rest of the flow. |
| **Win-back eligible** | *(Not a built-in event)* inferred from "cancelled + time elapsed." | **Schedule Trigger** → find subs cancelled ≥ N days ago with no resubscribe → enter the win-back flow. |
| **"Wait N days/at date" elapsed** | An internal queue of scheduled wake-ups. | **Wait** node inside the workflow (see [waits](#the-anatomy-of-a-flow)). |

**The rule of thumb:** if a trigger is a *moment in time relative to your data*
("3 days before X," "N days after Y," "when the running total crosses Z"), it
was derived, and it becomes a **Schedule Trigger + a query + a fire-once guard**
in n8n. If a trigger is *a thing that just happened* (a webhook landed), it's
raw, and it becomes a **Webhook Trigger**.

```
   RAW (push)                          DERIVED (poll)
   ──────────                          ──────────────
   Shopify / Stripe                    n8n Schedule Trigger (e.g. daily 9am)
        │ webhook                            │
        ▼                                    ▼
   n8n Webhook Trigger                  Query your DB / Shopify Admin API
        │                                    │  "trials ending in exactly 3 days"
        ▼                                    ▼
   your flow                            Fire-once guard (dedupe) ──► your flow
```

The "fire-once guard" matters: a daily scan will re-find the same trial tomorrow.
Mantle remembered what it had already emitted. In n8n, dedupe with a key
(customer + event + date) in a small store or a DB flag, so a customer doesn't
get the "trial ending" email three days running.

### The full Mantle trigger catalog (for reference)

So you can find every flow's trigger in your inventory and classify it. Group →
trigger names:

- **App lifecycle:** installed, uninstalled, reinstalled, deactivated,
  reactivated, reviewed, review edited, access-token invalid, first identify.
- **Subscription:** subscribed, unsubscribed, upgraded, downgraded, resubscribed,
  frozen, unfrozen.
- **Trial *(derived)*:** trial-expiring-in-N-days, trial expired, trial extended.
- **Billing / usage:** charge abandoned, one-time-charge activated, refunded,
  transaction created, usage-charge created, invoice finalized, invoice
  payment-failed; **usage approaching cap *(derived)*, usage exceeds limit
  *(derived)***; bundle discount applied / removed; capped-amount updated;
  platform plan changed.
- **Onboarding:** checklist completed, checklist step completed.
- **Support (tickets):** thread created, message received, updated, closed,
  reopened, priority/status changed, assigned, team/inbox assigned, SLA breach
  *(derived)* / approaching *(derived)*.
- **Sales:** deal created, stage updated, closed-won, closed-lost, flow changed;
  task created/status-changed/completed.
- **Contacts / lists:** contact created/updated/deleted, tag added, added-to /
  removed-from customer, list added/removed.
- **Affiliates:** join requested/approved/denied/joined, referral, payment
  requested, payout pending, commission earned.
- **System:** **time-passed *(derived wait)***, action completed, notification
  CTA, scheduled (bulk run), webhook received, API trigger, manual.

Items marked *(derived)* don't arrive on their own — rebuild them per the table
above. Everything else is a raw webhook or an in-app action you already control.

---

## The Mantle → n8n mapping table

This is the mechanical part. Each Mantle building block has a direct n8n node,
with the Make and Zapier equivalents alongside.

| Mantle building block | n8n node | Make module | Zapier |
|---|---|---|---|
| **Trigger — raw event** (subscribed, uninstalled, payment failed…) | **Webhook** (`n8n-nodes-base.webhook`) | Custom webhook | Webhooks by Zapier — Catch Hook |
| **Trigger — derived/scheduled** (trial-expiring, usage-threshold, win-back) | **Schedule Trigger** (`scheduleTrigger`) → query node → fire-once guard | Scheduled scenario + search module | Schedule by Zapier + a Find/Search step |
| **Trigger — bulk / scheduled run** (run flow over a segment) | **Schedule Trigger** → query → **Split In Batches** | Scheduled scenario + iterator | Schedule + (loop) |
| **Trigger — manual / API** | **Webhook** or **Manual Trigger** | On-demand / webhook | Webhooks Catch Hook |
| **Condition (single)** | **IF** (`if`) | Filter on a route | Filter by Zapier |
| **Condition / branch (multi-way)** | **Switch** (`switch`) | Router | Paths by Zapier |
| **Condition that drops non-matches** | **Filter** (`filter`) | Filter | Filter by Zapier |
| **Wait — fixed delay** ("wait 3 days") | **Wait** (`wait`, *after time interval*) | Sleep / Delay | Delay For by Zapier |
| **Wait — until a moment** ("at trial end") | **Wait** (`wait`, *until specified time*) | Sleep until | Delay Until by Zapier |
| **Action — send email** | **Send Email** (`emailSend`) or **Gmail** | Email | Email / Gmail |
| **Action — Slack message** | **Slack** (`slack`) | Slack | Slack |
| **Action — HTTP request / custom webhook / custom API** | **HTTP Request** (`httpRequest`) | HTTP | Webhooks by Zapier |
| **Action — tag/untag, set custom field, extend trial, assign owner** (write back to your app) | **HTTP Request** to your app's API or the Shopify Admin API | HTTP / Shopify | Shopify / Webhooks |
| **Action — execute script** | **Code** (`code`) | Tools — custom function | Code by Zapier |
| **Action — run AI agent** | **AI Agent** (`@n8n/n8n-nodes-langchain.agent`) | OpenAI / Anthropic | AI by Zapier / OpenAI |
| **Action — create deal/task/contact, log activity** | **HTTP Request** to your CRM (or the CRM's native node) | CRM module / HTTP | CRM app / Webhooks |
| **Enrollment dedupe** (`allowRepeatRuns`, block-repeats window) | **Remove Duplicates** / a DB flag / workflow static data | Data store | Storage by Zapier / dedupe |
| **Liquid templates / merge fields** (`{{ customer.name }}`) | n8n **expressions** (`{{ $json.customer.name }}`) | Mapping panel | Field mapping |
| **Previous-step outputs** (`previousActions[id].outputs`) | Reference upstream nodes (`{{ $node["Name"].json… }}`) | Reference earlier module | Reference earlier step |

Two structural notes when you rebuild:

- **One Mantle flow ≈ one n8n workflow.** A Mantle *step* (trigger + conditions +
  ordered actions) usually expands into *several* n8n nodes: an IF for the
  conditions, then one node per action, chained. Mantle's step connections become
  n8n's node connections.
- **Branches.** Mantle's fan-out (a step with multiple outgoing connections)
  becomes an **IF** (two-way) or **Switch** (n-way) in n8n. Put the
  condition on the branch node, not buried in each downstream action.

---

## The in-flight sequence warning

**Read this before you flip the switch.** It is the single most likely way to
hurt customers during the migration.

### What "in-flight" means

When a customer reaches a **wait** step ("wait 3 days, then send email 2"), the
flow doesn't hold a thread open for three days. The engine records *one fact*:
**"wake customer X up at step Y at timestamp T."** That scheduled wake-up is the
**entire** in-flight state. Everything else — which emails they've gotten, where
they are in the sequence — is implied by that one pending wake-up.

So at any moment you have a population of customers who are **not done and not
running** — they're parked, waiting for a future timestamp. Onboarding
sequences, trial-nurture drips, win-back series: all of them have people parked
mid-way at all times.

### Why migration strands them

```
   Mantle (winding down)                 n8n (new home)
   ─────────────────────                 ──────────────
   Customer parked at:                   Has NO idea this customer exists.
   "wake at step 3, T = +2 days"  ──X──► You cannot import Mantle's pending
                                         wake-ups. n8n holds its own waiting
   When Mantle stops, the wake-up        executions in its own store; there's
   never fires. The customer is          no bridge from Mantle's parked
   silently abandoned mid-sequence.      customers into it.
```

Two independent facts combine to bite you:

1. **Mantle's pending wake-ups won't fire** once the engine is off (or once you
   deactivate the flow). The parked customers just… stop.
2. **n8n can't inherit them.** A waiting n8n execution lives in n8n's own
   database, created when *that* workflow ran. There's no way to hand Mantle's
   parked customers to n8n as "already at step 3." If you want them continued,
   you have to **re-enter** them.

### What to do about it

Before cutover, for **each flow that contains a wait step**:

1. **Count who's parked.** Get the population of customers with a pending
   future wake-up in that flow. (This is your "in-flight count" column in the
   [inventory](#the-inventory-template).) If it's zero, you're free — migrate and
   move on.
2. **Pick a strategy per flow:**
   - **Drain (safest).** Keep Mantle running the *old* flow for in-flight
     customers until they finish, while **new** entrants go to n8n. Stop new
     enrollments in Mantle; let the tail drain; decommission when the longest
     wait has elapsed. Run both for one full sequence-length.
   - **Re-enroll.** Export the parked customers and **bulk-trigger** the n8n
     workflow for them. Only safe if re-entering from the top is harmless
     (idempotent emails, dedupe guards) — otherwise people get duplicate
     messages.
   - **Abandon (only if you accept the loss).** Let the tail drop. Fine for a
     low-stakes nudge; **not** fine for anything tied to revenue or a promised
     follow-up.
3. **Tell customers if it matters.** For a high-stakes sequence (a promised
   discount on day 7, a renewal reminder), a parked customer who silently never
   hears from you is a broken promise. Drain or re-enroll those.

> The trap is that in-flight customers are **invisible** — nothing errors, no
> dashboard goes red. They just never reach the next step. Audit the parked
> population *before* you turn anything off.

---

## The inventory template

Don't migrate flow-by-flow from memory. **Catalogue first**, then rebuild from
the catalogue. The act of filling this in is what surfaces the derived events you
need to rebuild and the in-flight populations you need to drain.

Fill one row per flow. The full copy-paste version (with field notes) is in
[`inventory-template.md`](./inventory-template.md); the shape:

| Flow | Trigger | Trigger kind | Conditions | Steps (waits + actions, in order) | In-flight count | Derived events to rebuild | n8n status |
|---|---|---|---|---|---|---|---|
| Trial nurture | trial-expiring-in-3-days | **derived** | plan = Pro | email → wait 2d → IF still-trial → Slack AM | 142 | trial-expiring scan | ☐ todo |
| Churn win-back | unsubscribed | raw | — | wait 14d → IF no-resub → email | 38 | win-back scan | ☐ todo |
| Upgrade thank-you | subscription.upgraded | raw | — | tag customer → Slack | 0 | — | ☐ todo |

Columns that earn their keep:

- **Trigger kind** — `raw` / `derived` / `scheduled` / `manual`. Every `derived`
  is a small Schedule-Trigger workflow you owe yourself.
- **In-flight count** — how many customers are parked in a wait *right now*. Drives
  your [drain-vs-abandon](#the-in-flight-sequence-warning) decision. Zero means
  free migration.
- **Derived events to rebuild** — the dependency list. De-duplicate across flows:
  ten flows may all need the one "trial-expiring" scan.

---

## The n8n workflow JSON prompt

n8n workflows are JSON, and you can **import JSON directly** (Workflow menu →
Import from File / paste). That means you can hand a description of one Mantle
flow to an LLM and get back an importable workflow.

[`n8n-workflow-prompt.md`](./n8n-workflow-prompt.md) is a ready-to-use prompt
that does exactly this. Give it one filled inventory row (or a prose description
of the flow) and it emits valid n8n workflow JSON — Webhook/Schedule trigger,
IF/Switch for conditions, Wait nodes for delays, HTTP/Email/Slack/Code/AI-Agent
for actions, with the connections wired up and credential fields left as
placeholders for you to fill in n8n.

Workflow:

1. Fill in the [inventory](#the-inventory-template).
2. For each row, paste the prompt + that row into your LLM.
3. Import the returned JSON into n8n.
4. **Wire credentials and verify** — the LLM can't know your API keys, your
   app's endpoints, or your exact field names. Treat the output as a correct
   *skeleton*, not a finished workflow. Test with one real customer before
   enabling.

Don't skip step 4. The JSON gets the **structure** right (the right nodes, the
right branching, the right waits); it cannot get your **environment** right.

---

## Gotchas — the lessons worth pre-empting

- **The derived stream is the project.** Re-pointing webhooks is an afternoon.
  Rebuilding "trial expiring in 3 days," "usage crossed the cap," and the churn
  enrichment is the actual migration. Budget for it.
- **Dedupe your scheduled scans.** A daily "trials ending in 3 days" scan
  re-finds the same trial tomorrow. Without a fire-once guard you'll re-send. Key
  the dedupe on customer + event + date.
- **In-flight customers are invisible.** Nothing errors when a parked customer is
  abandoned. Count the parked population per flow *before* cutover.
- **Run old and new in parallel during cutover** for any flow with a wait, so the
  tail drains while new entrants go to n8n. Don't hard-cut a flow that has people
  parked in it.
- **Enrichment ≠ raw.** If a flow keys off Mantle's churn-reason or
  tenure-bucketing, the Shopify webhook alone won't reproduce it — you need the
  classification step.
- **Conditions are guards, evaluated before the step.** Don't model a condition
  as an action that "decides" — model it as the branch node (IF/Switch) ahead of
  the actions.
- **n8n holds its own in-flight state.** A long Wait pauses the execution in
  n8n's database. That's the same idea as Mantle's wake-up queue — just don't
  expect to migrate one engine's parked state into the other's.
- **Imported JSON is a skeleton.** Credentials, your app's endpoints, and exact
  field names are always wrong out of the LLM. Verify with a real test before
  enabling.

---

## What to build vs. what to skip

**Reproduce (the parts that actually run your business):**
1. **Webhook intake** for the raw events your flows trigger on (Shopify + Stripe).
2. **The derived-events stream** — one small Schedule-Trigger workflow per
   computed trigger (trial-expiring, usage-threshold, win-back, enriched churn),
   each with a fire-once guard.
3. **One n8n workflow per Mantle flow**, built from the [mapping
   table](#the-mantle--n8n-mapping-table): trigger → conditions (IF/Switch) →
   waits (Wait) → actions (HTTP/Email/Slack/Code/AI).
4. **The in-flight drain plan** for every flow that contains a wait.
5. **A dedupe/enrollment guard** wherever a customer shouldn't run a flow twice.

**Skip or substitute (Mantle-internal machinery you don't need):**
- Mantle's flow *versioning/snapshot* system — your n8n workflows are
  version-controlled by n8n (or git-exported) instead.
- The resource-agnostic engine (the same flow machinery applied to tickets,
  deals, affiliates) — rebuild only the resource types you actually automate.
- Mantle's bulk-run staging/targeting pipeline — a Schedule Trigger + a query +
  Split-In-Batches covers the same ground at partner scale.
- The flow-health/analytics dashboards — n8n's own execution log covers
  "did it run / did it fail" until you need more.

---

## Where to go next

- Migrating by hand? Work down the "Reproduce" list above; keep
  [`inventory-template.md`](./inventory-template.md) open as you go and reach for
  [`implementation-spec.md`](./implementation-spec.md) when you need the precise
  derived-event recipes and node mapping.
- Want an agent to do it? Install the
  [skill](./skill/mantle-flows-to-n8n/) into your app's workspace, or paste
  [`implementation-spec.md`](./implementation-spec.md) into your coding agent and
  feed it your filled inventory.
- Generating workflows? Use [`n8n-workflow-prompt.md`](./n8n-workflow-prompt.md)
  one flow at a time, then import and verify.
