# `shopify-billing-migration` skill

An Agent skill that implements **standard Shopify-managed app billing** — the
wrapper architecture where subscriptions live on Shopify and your app keeps a
reconciled local mirror — and **migrates merchants onto that rail** across
billing models (flat, annual, usage/metered, hybrid, one-time, free) in the
correct order of operations, in *your* codebase.

> This skill is **not** for re-approval-free in-place tier changes — that's flex
> billing, a separate system (mantle-brain ships it as the `shopify-flex-billing`
> skill).

## Install

Copy this folder into your project's skills directory and let your agent pick it
up:

```bash
# from the root of YOUR app's repo
mkdir -p .claude/skills
cp -R path/to/mantle-brain/shopify-billing-migration/skill/shopify-billing-migration .claude/skills/
```

The folder is self-contained — `references/implementation-spec.md` travels with
it, so the skill works after being copied out of this repo.

## Use

Open your app's workspace with Claude Code (or any agent that loads
`.claude/skills/`) and ask for what you want, e.g.:

> "Add Shopify app subscriptions, mirror them in our DB, and handle the
> confirmation/return-url activation flow."

> "Migrate our merchants from Stripe billing onto Shopify-managed billing."

> "Reconcile our local subscriptions against Shopify and fix orphaned states."

The agent will load the skill, read the bundled spec, inspect your stack, and
work through the implementation phases described in [`SKILL.md`](./SKILL.md).

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The procedure the agent follows (phases, guardrails). |
| `references/implementation-spec.md` | The precise, self-contained build spec (data model, API shapes, ordered procedures, per-model migration, edge cases, acceptance tests). |

## Prerequisites the agent will check

- Your Shopify app has **public distribution** (private/dev apps can't use the
  Billing API).
- **Partner API** access if you want charge/transaction reconciliation against
  Shopify.
</content>
