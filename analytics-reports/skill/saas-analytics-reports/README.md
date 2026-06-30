# `saas-analytics-reports` skill

An Agent skill that implements a **SaaS / app-billing analytics suite** — 16
reports (MRR with discount & cadence rules, ARR, revenue/payout, active
subscriptions, active customers/installs, growth, churn, retention & cohorts,
ARPU, LTV, trials, install funnel, installs & uninstalls with reasons, usage
metrics, forecasting, and traffic sources via GA4 + BigQuery) — in *your*
codebase, with trustworthy "as-of" time-series reconstruction.

## Install

Copy this folder into your project's skills directory and let your agent pick it
up:

```bash
# from the root of YOUR app's repo
mkdir -p .claude/skills
cp -R path/to/mantle-brain/analytics-reports/skill/saas-analytics-reports .claude/skills/
```

The folder is self-contained — `references/implementation-spec.md` travels with
it, so the skill works after being copied out of this repo.

## Use

Open your app's workspace with Claude Code (or any agent that loads
`.claude/skills/`) and ask for what you want, e.g.:

> "Build MRR, churn, and net-revenue-retention reporting for my subscription app,
> with time-series I can query as-of any past date."

> "Add a metrics API: MRR, ARR, ARPU, LTV, trials, and an install funnel."

> "Migrate my analytics off Mantle into our own codebase."

The agent will load the skill, read the bundled spec, inspect your stack, and work
through the implementation phases described in [`SKILL.md`](./SKILL.md).

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The procedure the agent follows (phases, guardrails). |
| `references/implementation-spec.md` | The precise, self-contained build spec (data model, per-report formulas, the as-of reconstruction algorithm, edge cases, validators, acceptance checks). |

## What the agent will establish first

- Your OLTP source of truth for subscriptions, plans, discounts, installs, and
  charges.
- Whether you have a search/document store and/or a columnar/OLAP store (it will
  adapt — the three storage *roles* matter, not the specific vendors).
- Which of the 16 reports you need now, and the order to build them (recurring
  revenue core first, traffic last).
