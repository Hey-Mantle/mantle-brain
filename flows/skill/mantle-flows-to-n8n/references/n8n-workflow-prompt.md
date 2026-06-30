# n8n Workflow Generation Prompt

A ready-to-use LLM prompt that turns **one Mantle flow** into **importable n8n
workflow JSON**. Give it a filled
[inventory](./inventory-template.md) row (or a prose description of the flow); it
returns JSON you can paste into n8n (**Workflow menu → Import from File**, or
paste into the canvas).

**How to use:** copy everything in the fenced block below, replace
`<<<FLOW DESCRIPTION>>>` with your flow, and send it to a capable LLM. Then import
the result and **bind credentials + fix placeholders before enabling** (the
model cannot know your API keys, endpoints, or exact field names).

Generate **one flow at a time** — it keeps the output focused and importable.

---

```text
You are an expert n8n workflow author. Convert the described automation ("flow")
into a single valid n8n workflow JSON document that imports cleanly into n8n.

## FLOW TO CONVERT
<<<FLOW DESCRIPTION>>>
# e.g.:
# Name: Trial nurture
# Trigger: derived — "trial expiring in 3 days" (a daily scan of trials whose
#   end date is exactly 3 days out). Runs daily at 09:00.
# Conditions: customer's plan == "Pro"
# Steps in order:
#   1. Send email "Your trial ends soon"
#   2. Wait 2 days
#   3. IF customer is still on trial -> Slack message to the account manager
# Write-backs: none

## OUTPUT CONTRACT
Return ONLY raw JSON (no markdown fences, no prose). The JSON MUST have this
shape:

{
  "name": "<flow name>",
  "nodes": [
    {
      "parameters": { ... },
      "id": "<unique uuid-like string>",
      "name": "<unique human-readable label>",
      "type": "n8n-nodes-base.<nodeType>",
      "typeVersion": <number>,
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

## NODE MAPPING (choose nodes from this set)
- Raw event trigger (a webhook arrives): "n8n-nodes-base.webhook"
    typeVersion 2. Set a descriptive "path". This is the entry node.
- Derived / scheduled trigger (a daily/periodic scan): "n8n-nodes-base.scheduleTrigger"
    typeVersion 1.2. Configure the interval (e.g. daily at a given hour).
- Single condition / branch: "n8n-nodes-base.if" (typeVersion 2). TWO outputs:
    index 0 = true, index 1 = false.
- Multi-way branch: "n8n-nodes-base.switch" (typeVersion 3). One output per case.
- Drop non-matching items: "n8n-nodes-base.filter" (typeVersion 2).
- Wait / delay: "n8n-nodes-base.wait" (typeVersion 1.1). For a fixed delay use
    resume "timeInterval" with amount + unit; for "until a moment" use resume
    "specificTime".
- Send email: "n8n-nodes-base.emailSend" (typeVersion 2). (Or a Gmail node.)
- Slack message: "n8n-nodes-base.slack" (typeVersion 2.2).
- HTTP request / custom webhook / call an external or your-own API:
    "n8n-nodes-base.httpRequest" (typeVersion 4.2).
- Run custom code/script: "n8n-nodes-base.code" (typeVersion 2).
- Run an AI agent: "@n8n/n8n-nodes-langchain.agent" (typeVersion 1.7).
- Pass-through / placeholder: "n8n-nodes-base.noOp" (typeVersion 1).
- Loop over many items (bulk run): "n8n-nodes-base.splitInBatches" (typeVersion 3).

## ACTIONS THAT WRITE BACK TO THE APP
Tagging a customer, setting a custom field, extending a trial, assigning an
owner, creating a CRM object, etc. have NO native node. Model each as an
"n8n-nodes-base.httpRequest" POST to a placeholder URL like
"={{ $env.APP_API_BASE }}/customers/{{ $json.customerId }}/tags" and add a short
note in the node "name" (e.g. "Tag customer (HTTP - set URL)"). Do NOT invent a
real endpoint or auth.

## RULES
1. Exactly ONE trigger node (webhook OR scheduleTrigger), and it is the entry of
   the graph.
2. Every node "name" is unique. "connections" reference nodes BY NAME.
3. For a DERIVED trigger, after the scheduleTrigger add an httpRequest node named
   like "Query <thing> (set URL)" that represents the data scan, THEN a
   "Remove Duplicates"/guard step or a note, so the scan fires once per match.
   Use "n8n-nodes-base.removeDuplicates" (typeVersion 2) for the fire-once guard
   when appropriate.
4. Translate conditions into IF/Switch placed BEFORE the actions they guard. Wire
   BOTH the true (0) and false (1) outputs of every IF — send the false branch to
   a noOp named "Exit" if nothing else happens there.
5. Translate every wait into a Wait node with the stated duration/units.
6. Convert merge fields like "{{ customer.name }}" into n8n expressions like
   "={{ $json.customer.name }}". For any value you cannot determine (URLs, IDs,
   field names, channel names), use a clearly-named placeholder string and note
   it in the node name — never fabricate a real value.
7. Do NOT embed credentials. Leave credential bindings out; the human will set
   them in n8n after import.
8. Lay out nodes left-to-right: start near x=240, increment x by ~220 per step;
   offset branch outputs vertically (e.g. y=200 for true, y=420 for false).
9. typeVersion must be a number that exists for that node; the versions above are
   safe defaults.
10. Output RAW JSON only. No commentary. It must parse with JSON.parse and import
    into n8n without edits to its structure.

## CHECKLIST BEFORE YOU ANSWER
- One trigger node, correct type for raw vs derived.
- Conditions are IF/Switch nodes ahead of actions; both IF branches wired.
- Waits present with correct durations.
- Write-back actions are httpRequest with placeholder URLs + notes.
- Derived flows include a query node and a fire-once guard.
- Valid, parseable JSON; raw (no fences).
```

---

## After you import

The generated JSON is a correct **skeleton**, not a finished workflow. n8n will
import the right nodes, branches, and waits — but it cannot know your
environment. Always:

1. **Bind credentials** on every email / Slack / HTTP / AI node.
2. **Replace placeholders** — the `(set URL)` HTTP nodes, channel names, field
   names, and any merge field the model guessed at.
3. **Confirm the derived-trigger query** actually returns the right rows, and the
   **fire-once guard** dedupes (so a daily scan doesn't re-send tomorrow).
4. **Test with one real customer** end to end — trigger → branch → wait → action
   — before you enable the workflow.

> The prompt gets the **structure** right (nodes, branching, delays); it cannot
> get your **environment** right (keys, endpoints, exact fields). Verifying step
> 1–4 is the difference between an import and a working migration.
