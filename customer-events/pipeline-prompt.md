# Pipeline prompt

A single, self-contained prompt for a coding agent (Claude, Cursor, etc.). Paste
it into your app's workspace and let the agent build the whole Shopify
customer-events pipeline end to end. It encodes the same mechanics as
[`implementation-spec.md`](./implementation-spec.md) in a compact, imperative
form — use the spec as the reference when the agent needs exact detail.

---

> **Task: build a pipeline that turns the Shopify Partner API app-events feed
> into clean customer lifecycle events for MRR/churn analytics.**
>
> First inspect my stack: language, web framework, ORM/database, job/cron runner,
> how I already talk to the Shopify Partner API, and any existing
> event/subscription/plan models. Map everything below onto my conventions —
> prefer editing my existing models over inventing parallel ones. Ask me before
> making schema or architectural choices you can't infer.
>
> Build it in two layers — never classify on the wire:
>
> **1. Raw layer.** Poll the Partner API `appEvents` feed, cursor-paginated
> (~50/page), with an overlapping since-buffer (re-query from ~60 min before the
> last sync — the feed is late and out of order). Persist every event verbatim
> into a `raw_app_events` table, deduped with a unique key on
> `(appInstallationId, type, occurredAt, chargeGid)` (use a `"NO_CHARGE"`
> sentinel when there's no charge). Store the raw Shopify type string as-is. Catch
> per-event errors without aborting the page. There is no trustworthy real-time
> lifecycle webhook — the poll drives everything.
>
> **2. Derivation layer.** For each install, sort its raw events by `occurredAt`
> ascending and fold them through a **state machine**, carrying `currentState`,
> `currentAmount` (normalized monthly MRR), and `mostRecentSubscription`. Emit
> clean events into an `app_events` table, each with a deterministic, unique
> `platformEventId` derived from its source so re-derivation is idempotent.
>
> Classification rules:
> - `RELATIONSHIP_INSTALLED` → `installed`, or `reinstalled` if an `installed`
>   was already emitted for this install.
> - `RELATIONSHIP_UNINSTALLED` → `uninstalled`. `RELATIONSHIP_REACTIVATED` /
>   `RELATIONSHIP_DEACTIVATED` → `reactivated` / `deactivated` (account-level, no
>   MRR move).
> - `SUBSCRIPTION_CHARGE_ACTIVATED` from a non-paying state
>   (`installed`/`reinstalled`/`unsubscribed`/`none`) → `subscribed`, or
>   `resubscribed` if the install ever emitted `subscribed` before;
>   `netChange = +normalizedMonthly`.
> - `SUBSCRIPTION_CHARGE_ACTIVATED` while already paying → `upgraded` if
>   `normalizedMonthly >= currentAmount`, else `downgraded`;
>   `netChange = normalizedMonthly − currentAmount`.
> - `SUBSCRIPTION_CHARGE_FROZEN` → `subscription_frozen`, MRR to 0;
>   `SUBSCRIPTION_CHARGE_UNFROZEN` → `subscription_unfrozen`, MRR restored.
> - `*_DECLINED` / `*_EXPIRED` → `charge_abandoned`; for `*_EXPIRED`, back-date
>   `occurredAt` (e.g. −60h) but never before the install's first interaction.
> - `ONE_TIME_CHARGE_ACTIVATED` → `one_time_charge_activated`; `CREDIT_APPLIED` →
>   `credit_applied` with `netChange = −charge.amount`; usage-cap signals →
>   their own events.
> - Unknown raw types → log and skip, never throw.
> - On any subscription event, set `previousSubscriptionId` to the prior
>   subscription when it differs from this event's, then update
>   `mostRecentSubscription`.
>
> **Normalize money to monthly before any comparison:** flex/$0-recurring plans
> use the plan's nominal amount; `EVERY_30_DAYS` uses the charge amount; `ANNUAL`
> uses charge ÷ 12. Reflect active discounts (the price actually paid, unless the
> discount is delivered as app credits). MRR over any window = sum of `netChange`
> over the install's non-ignored events.
>
> **The cancel trap (most important rule).** A `SUBSCRIPTION_CHARGE_CANCELED` is
> usually one half of a plan change, not a churn. Suppress it if ANY of:
> (a) an activation exists within `[cancel, cancel+60s)`;
> (b) an activation exists within `(cancel−5s, cancel]` with the same `chargeGid`;
> (c) only when neither (a) nor (b): a chained `upgraded`/`downgraded` clean event
> exists with `previousSubscriptionId == cancel.subscriptionId` and
> `|occurredAt − cancel| <= 60s` — this is the **paid→free / flex** case, which
> emits no activation webhook at all. On suppression, soft-delete any clean event
> already derived from this cancel; and for case (c), advance the state machine's
> `currentState`/`currentAmount` to the chained event's values so the next
> activation's delta is computed against the new (free) baseline. Only a cancel
> with none of these becomes `unsubscribed` (`netChange = −normalizedMonthly`).
> Also add an offline reconciler that soft-deletes stray `unsubscribed` events
> that have a matching plan-change event within ±60s (catches ordering races).
>
> **Trials.** Own trial state locally; keep local trials off Shopify's
> `trialDays`. Derive `trial_active` / `trial_converted` / `canceled_during` /
> `canceled_after` from `trialExpiresAt` vs now and the sub's cancel/freeze dates.
> Add a nightly sweep that emits an idempotent `trial_expired` event (deterministic
> `platformEventId`) for trials whose window just closed and that are still active,
> then stamp `trialExpiredAt`. A trial sub that upgrades/downgrades to a different
> subscription **converted** — don't count it as churn (use the
> `previousSubscriptionId` chain).
>
> **Uninstall reasons.** When an `uninstalled` event lands, enrich it in a job:
> translate the (often localized) reason to English, map it through a
> deterministic preset table of Shopify's known reason labels (handle
> comma-separated multi-select; fail to null if any token is unknown), and only
> then fall back to an LLM classifier for free text — storing one or more canonical
> codes (`reasonCodes[]` + primary `reasonCode`). Cache LLM results by a hash of
> the translated reason+description. Define a stable taxonomy and **flag the
> store-closure subset** (`store_closing_or_pausing`, `deactivated`,
> `scheduled_cancellation`) separately so it can be excluded from product-churn
> analytics without losing it from raw counts.
>
> **Idempotency & projections.** Re-deriving an install's whole history must
> converge to the same clean events (upsert on `platformEventId`); reuse an
> existing subscription rather than duplicating on replay. Don't delete events
> that shouldn't count toward MRR — mark them `ignored` at the read/index boundary
> (e.g. cross-subscription churn noise within ~5 min), but never ignore a trial
> churn that converted via plan change. The `app_events` table is the source of
> truth; if you fan out to a search index or columnar store for dashboards, treat
> those as disposable projections whose failures never block the authoritative
> write. A plain SQL query over `app_events` is enough until real scale.
>
> **Verify** against these scenarios before calling it done: first subscribe;
> annual normalizes to monthly; mid-cycle upgrade and downgrade emit one
> plan-change event and zero churn; paid→free suppresses the cancel and
> re-baselines; real churn emits one `unsubscribed`; win-back emits
> `resubscribed`; freeze/unfreeze move MRR without churn; `*_EXPIRED` back-dates;
> nightly trial sweep is idempotent; trial-converts-via-plan-change isn't churn;
> uninstall preset vs free-text classification; store closures excluded from
> product churn; full replay produces an identical clean-event set; re-polling
> creates no duplicates.
