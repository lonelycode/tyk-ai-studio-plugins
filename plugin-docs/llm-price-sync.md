# LLM Price Sync

**Keep AI Studio's model price list — and therefore your cost tracking and budgets — automatically up to date with current market prices.**

| | |
|---|---|
| Plugin ID | `com.tyk.llm-price-sync` |
| Tier | Community |
| Runs in | AI Studio only |
| Hooks | `studio_ui` (dashboard); syncs run via the platform scheduler |
| Admin UI | **Price Sync → Price Sync Dashboard** (`/admin/price-sync/dashboard`) |
| Requires | AI Studio ≥ 2.6.0 |

## What it does

AI Studio computes spend, budgets, and analytics from its **Model Prices** table. Left alone, that table drifts out of date as providers change pricing and release new models — silently skewing every cost figure in the platform.

This plugin fetches current per-token prices from [llm-prices.com](https://www.llm-prices.com) on a schedule (every 6 hours by default) and reconciles them into AI Studio's Model Prices:

- **Updates** entries whose prices (input, output, or cache prices) have changed (it also corrects a wrong vendor assignment on an existing entry).
- **Creates** entries for models that don't exist yet (optional, on by default) — so a newly released model starts being cost-tracked the day your users start calling it.
- **Skips** everything unchanged, unmapped vendors, and (optionally) vendors you've filtered out.

Source prices are quoted per million tokens and are converted to AI Studio's per-token format automatically. Prices are stored in USD. Cache-*read* prices are synced when the source provides them; cache-*write* prices are not published by the source and are set to 0. **This zeroing applies on every sync**: if you have hand-set cache-write prices (or cache-read prices for models where the source publishes none), each sync resets them to 0.

## When to use it

- **Accurate chargeback/showback** — if you report AI spend to teams or customers, stale prices mean wrong invoices. This keeps the basis current without anyone remembering to update it.
- **Meaningful budgets** — budget enforcement is only as good as the prices behind it; a provider price cut (or increase) is reflected within hours.
- **New-model coverage** — with auto-create on, models your users adopt appear in cost tracking without an admin manually entering prices.

## Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `prices_url` | `https://www.llm-prices.com/current-v1.json` | Where to fetch prices — a JSON object of the form `{"updated_at": …, "prices": [...]}` |
| `sync_interval_cron` | `0 */6 * * *` (every 6 hours) | Cron expression used when the schedule is first created (see note below) |
| `auto_create_models` | `true` | Create price entries for models not yet in the system |
| `vendor_filter` | *(empty = all)* | Only sync these vendors (see the mapping below); values are matched exactly, so use lowercase |
| `dry_run` | `false` | Log what *would* change without writing anything — useful before first enabling it on a curated price list. (Dry runs still appear in the dashboard's created/updated counts and history, with no special marker.) |

On startup, the plugin registers a schedule named "LLM Price Sync" (in the UTC timezone, with a 5-minute run timeout) if one doesn't already exist. **Changing `sync_interval_cron` later does not update that existing schedule** — edit or delete the "LLM Price Sync" schedule itself to change the cadence.

### Vendor mapping

llm-prices.com vendor names are mapped to AI Studio vendor slugs; models from unmapped vendors are skipped:

| Source vendor | AI Studio slug |
|---------------|----------------|
| anthropic | `anthropic` |
| openai | `openai` |
| google / google ai | `google_ai` |
| amazon / aws bedrock / bedrock | `bedrock` |
| vertex / vertex ai | `vertex` |
| hugging face / huggingface | `huggingface` |
| mistral / mistral ai | `mistral` |
| deepseek | `deepseek` |
| xai | `xai` |
| qwen | `qwen` |
| minimax | `minimax` |
| moonshot-ai | `moonshot-ai` |

The `vendor_filter` accepts either the AI Studio slug or the source vendor name.

## The dashboard

The **Price Sync** section in the admin sidebar shows:

- Current status and configuration, plus the outcome of the last sync (models created / updated / skipped, errors, duration)
- A **manual sync trigger** for immediate reconciliation — e.g. right after a provider announces new pricing
- Sync history (the last 10 runs are retained)

## Things to know

- **It writes to your Model Prices table.** If you maintain deliberately non-market prices (e.g. internally marked-up rates for chargeback), either point `prices_url` at your own JSON feed, restrict `vendor_filter`, or don't run this plugin — a sync will overwrite hand-edited prices for models it recognizes. Run with `dry_run: true` first to see exactly what it would touch.
- Model matching is by model name (the source's model ID, e.g. `claude-sonnet-4-6` — globally unique across vendors).
- A failed fetch aborts the run before anything is written. Per-model errors are recorded and shown on the dashboard, but models synced successfully in the same run remain written.
- The plugin needs outbound HTTPS access from AI Studio to the configured `prices_url`.
