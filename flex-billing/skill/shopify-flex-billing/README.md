# `shopify-flex-billing` skill

An Agent skill that implements **Shopify flex billing** — flexible, tiered app
subscriptions with in-place upgrades/downgrades, proration, and automatic
usage-based tiering, without triggering Shopify's subscription re-approval flow —
in *your* codebase.

## Install

Copy this folder into your project's skills directory and let your agent pick it
up:

```bash
# from the root of YOUR app's repo
mkdir -p .claude/skills
cp -R path/to/mantle-brain/flex-billing/skill/shopify-flex-billing .claude/skills/
```

The folder is self-contained — `references/implementation-spec.md` travels with
it, so the skill works after being copied out of this repo.

## Use

Open your app's workspace with Claude Code (or any agent that loads
`.claude/skills/`) and ask for what you want, e.g.:

> "Add flexible tiered Shopify billing so customers can upgrade and downgrade
> plans mid-cycle without re-approving the subscription."

> "Migrate my app off Mantle Flex Billing into our own codebase."

The agent will load the skill, read the bundled spec, inspect your stack, and
work through the implementation phases described in
[`SKILL.md`](./SKILL.md).

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The procedure the agent follows (phases, guardrails). |
| `references/implementation-spec.md` | The precise, self-contained build spec (data model, API shapes, pseudocode, edge cases, acceptance tests). |

## Prerequisites the agent will check

- Your Shopify app has **public distribution** (private/dev apps can't use the
  Billing API).
- You have **Partner API** access (required for downgrade credits via
  `appCreditCreate`).
