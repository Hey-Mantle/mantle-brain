# mantle-brain

Open-source build guides for recreating Mantle's billing, analytics, and automation tooling in your own app.

## What this is

`mantle-brain` is a library of self-contained guides that explain **how Mantle's core features worked and how to rebuild them yourself**. Each top-level folder covers one capability — the data model, the formulas, the edge cases, and the order of operations — so you can stand the feature back up in your own stack instead of losing it when Mantle goes away.

This is about *strategy, not source code*. The guides hand you the hard-won decisions (how MRR is reconstructed as-of a past date, how to bill flexibly without re-triggering Shopify's approval screen, how to reproduce the "derived" events your automations relied on) rather than a drop-in Mantle codebase.

## Who it's for

Shopify Partners and app developers who relied on Mantle and need to keep running after it's gone.

The guides are **stack-agnostic**: they describe behavior, data models, and algorithms, not Mantle's internals. Wherever a step is specific to Mantle's own infrastructure, the guide says so and tells you to skip or substitute it — so you can map the rest onto whatever you already run (any language, any database, your own queue or warehouse).

## What's inside

| Topic | What it covers | Installable skill |
| --- | --- | --- |
| [affiliates](./affiliates/) | A referral / partner program — affiliates share a tracked link, the installs they drive are attributed to them, the revenue those installs generate earns commission, and you pay it out through your own payment provider. | `shopify-affiliate-program` |
| [analytics-reports](./analytics-reports/) | The 16-report SaaS analytics suite — MRR, ARR, churn (logo / subscription / revenue), retention & cohorts, ARPU, LTV, trials, the install funnel, usage metrics, and forecasting — including reconstructing any metric *as-of* a past date. | `saas-analytics-reports` |
| [customer-events](./customer-events/) | Turning Shopify's raw Partner API app-events feed into a clean customer-lifecycle stream (`subscribed` / `upgraded` / `downgraded` / `unsubscribed` / `uninstalled`) that every MRR and churn chart is built on. | `shopify-customer-events` |
| [flex-billing](./flex-billing/) | Flexible, tiered Shopify subscriptions with in-place upgrades / downgrades, proration, and usage-based auto-tiering — without sending merchants back through Shopify's subscription re-approval screen. | `shopify-flex-billing` |
| [flows](./flows/) | Migrating Mantle's lifecycle automations (trial nurture, dunning, churn win-back, onboarding, usage upsell) into [n8n](https://n8n.io) / Make / Zapier — including the *derived* trigger events and the in-flight state Mantle managed for you. | `mantle-flows-to-n8n` |
| [shopify-billing-migration](./shopify-billing-migration/) | Standard Shopify-managed app billing — Shopify owns the billing clock and collects the charge, your app keeps a reconciled local mirror — and migrating merchants onto that rail across every billing model. | `shopify-billing-migration` |

## Three ways to use each guide

Every topic offers three on-ramps to the same material — pick the one that fits how you want to work:

1. **Read it yourself.** Each topic's `README.md` is a human-readable walkthrough of how the feature worked and what it takes to rebuild.
2. **Hand it to a coding agent.** Paste the topic's `implementation-spec.md` into Claude Code, Cursor, or similar and say *"implement this in my app."* It's a dense, imperative build spec — data model, exact shapes, pseudocode, and acceptance tests.
3. **Install the skill.** Each topic ships an installable Claude Code / Agent skill under `skill/<name>/` that drives the build for you (see below).

Some topics also ship extra companions — a one-shot pipeline prompt, an n8n workflow generator, an inventory template. Check the topic's own README for what's included.

## Installing a skill

Each skill lives at `<topic>/skill/<skill-name>/`. To use one, copy that folder into your project's `.claude/skills/` directory:

```bash
cp -R flex-billing/skill/shopify-flex-billing /path/to/your-app/.claude/skills/
```

Open the project in Claude Code and the skill is available — the agent follows its `SKILL.md`, pulling in the bundled spec and references as needed. Each skill folder is self-contained, so you can copy it out on its own. The `implementation-spec.md` files are editor-agnostic if you'd rather paste them into another agent directly.

## License

[MIT](./LICENSE) © Hey-Mantle
