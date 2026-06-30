# `mantle-flows-to-n8n` skill

An Agent skill that migrates **Mantle Flows** — lifecycle automations (trial
nurture, dunning, churn win-back, onboarding, usage upsell) — into **n8n** (or
Make / Zapier), in *your* environment.

It focuses on the two parts of the migration that actually take work: rebuilding
the **derived events** Mantle computed for you (trial-expiring, usage-threshold,
enriched churn) as scheduled scans, and **not stranding customers** who are
parked mid-sequence in a "wait" step at cutover.

## Install

Copy this folder into your project's skills directory and let your agent pick it
up:

```bash
# from the root of YOUR repo
mkdir -p .claude/skills
cp -R path/to/mantle-brain/flows/skill/mantle-flows-to-n8n .claude/skills/
```

The folder is self-contained — `references/` carries the spec, the inventory
template, and the n8n-generation prompt, so the skill works after being copied
out of this repo.

## Use

Open your workspace with Claude Code (or any agent that loads `.claude/skills/`)
and ask for what you want, e.g.:

> "Migrate my Mantle flows to n8n."

> "Rebuild the events that trigger my automations now that Mantle is shutting
> down — especially the trial-expiring and usage ones."

> "Turn my trial-nurture flow into an importable n8n workflow."

The agent will load the skill, read the bundled spec, inventory your flows,
rebuild the derived-events stream, recreate each flow, and walk the in-flight
cutover — per the phases in [`SKILL.md`](./SKILL.md).

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The procedure the agent follows (phases, guardrails). |
| `references/implementation-spec.md` | The precise migration spec (anatomy, raw-vs-derived events, node mapping, in-flight cutover, acceptance scenarios). |
| `references/inventory-template.md` | Copy-paste template for cataloguing your flows before migrating. |
| `references/n8n-workflow-prompt.md` | LLM prompt that turns one flow into importable n8n workflow JSON. |

## What the agent will check

- Which flow triggers are **raw** (re-point a webhook) vs **derived** (rebuild a
  scheduled scan) — the derived set is the real scope.
- The **in-flight population** of every flow with a wait step, before anything is
  disabled — so no customer is dropped mid-sequence.
- That you have an n8n instance, webhook intake, and an API/DB the flows can
  query and write back to.
