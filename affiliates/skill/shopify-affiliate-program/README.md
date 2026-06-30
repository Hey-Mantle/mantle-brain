# `shopify-affiliate-program` skill

An Agent skill that implements a **referral/affiliate program** — tracked referral
links, click-to-install attribution, commission accrual on referred revenue, and
affiliate payout aggregation (plus an affiliate portal) — in *your* codebase. The
final disbursement step is wired to your own payment provider (out of scope).

## Install

Copy this folder into your project's skills directory and let your agent pick it
up:

```bash
# from the root of YOUR app's repo
mkdir -p .claude/skills
cp -R path/to/mantle-brain/affiliates/skill/shopify-affiliate-program .claude/skills/
```

The folder is self-contained — `references/implementation-spec.md` travels with
it, so the skill works after being copied out of this repo.

## Use

Open your app's workspace with Claude Code (or any agent that loads
`.claude/skills/`) and ask for what you want, e.g.:

> "Add an affiliate program so partners can share a referral link and earn a
> recurring commission on the revenue from installs they drive."

> "Migrate my app off Mantle Affiliates into our own codebase."

The agent will load the skill, read the bundled spec, inspect your stack, and work
through the implementation phases described in [`SKILL.md`](./SKILL.md).

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The procedure the agent follows (phases, guardrails). |
| `references/implementation-spec.md` | The precise, self-contained build spec (data model, pseudocode, commission math, payout aggregation, edge cases, acceptance tests). |

## Prerequisites the agent will check

- A stream of billing **transactions** to read (the commission engine derives
  commissions from your billing events — Shopify subscription/usage transactions,
  Stripe, etc.).
- For payouts: *some* way to pay affiliates (PayPal, Stripe, bank transfer,
  manual). The skill builds everything up to "mark the payout paid"; the actual
  payment-provider integration is yours to wire in and is out of scope.
