# `shopify-customer-events` skill

An Agent skill that builds a **Shopify customer-events pipeline** — turning the
raw Partner API app-events feed into clean lifecycle events (`subscribed`,
`upgraded`, `downgraded`, `unsubscribed`, `uninstalled`, `trial_expired`) with
correct MRR deltas, plan-change suppression, trial derivation, and uninstall-reason
normalization — in *your* codebase.

## Install

Copy this folder into your project's skills directory and let your agent pick it
up:

```bash
# from the root of YOUR app's repo
mkdir -p .claude/skills
cp -R path/to/mantle-brain/customer-events/skill/shopify-customer-events .claude/skills/
```

The folder is self-contained — `references/implementation-spec.md` travels with
it, so the skill works after being copied out of this repo.

## Use

Open your app's workspace with Claude Code (or any agent that loads
`.claude/skills/`) and ask for what you want, e.g.:

> "Ingest our Shopify Partner app-events feed and turn it into clean subscription
> lifecycle events for MRR and churn."

> "My plan changes are being recorded as churn — build the cancel-suppression
> logic properly."

> "Migrate our customer-events pipeline off Mantle into our own codebase."

The agent will load the skill, read the bundled spec, inspect your stack, and
work through the implementation phases described in [`SKILL.md`](./SKILL.md).

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The procedure the agent follows (phases, guardrails). |
| `references/implementation-spec.md` | The precise, self-contained build spec (data model, state machine, cancel trap, normalization math, edge cases, acceptance tests). |

## Prerequisites the agent will check

- You are a **Shopify Partner** with **Partner API** access and can query the
  `appEvents` feed for your app(s).
- You have somewhere to run a **scheduled poll** (and a nightly trial sweep).
- For uninstall-reason normalization: access to a **translation** step and an
  **LLM** for the free-text fallback (optional — you can ship the preset-map tier
  first).
