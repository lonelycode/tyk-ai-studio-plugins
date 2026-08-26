# Rate Limiter

**Protect your LLM budget and providers with sliding-window request, token, and concurrency limits — scoped to apps, users, models, or API keys.**

| | |
|---|---|
| Plugin ID | `com.tyk.rate-limiter` |
| Tier | Community |
| Runs in | AI Studio (embedded gateway) and Microgateway (edge) |
| Hooks | `post_auth` (limit evaluation), `on_response` (token counting), `studio_ui` (rule manager) |
| Admin UI | **Rate Limiting → Rate Limit Rules** (`/admin/rate-limiter/rules`) |
| Requires | AI Studio ≥ 2.6.0, Microgateway ≥ 1.0.0 |

## What it does

Every LLM request is checked against a set of **rules** you manage in the admin UI. Each rule can limit one of three things:

- **`requests`** — how many requests are allowed per sliding time window.
- **`tokens`** — how many tokens (prompt + completion, from the provider's actual usage report) may be consumed per window.
- **`concurrent`** — how many requests may be in flight at the same time.

When a rule is breached, the request is rejected with **HTTP 429** and a JSON error body naming the rule, the limit, current usage, and the reset time — plus a `Retry-After` header so well-behaved clients know when to come back.

## When to use it

- **Stop runaway spend** — cap tokens per app or per user so a misbehaving script can't burn a month's budget in an hour.
- **Fair sharing** — give every user of a shared assistant the same request allowance instead of letting one power user starve the rest.
- **Protect fragile upstreams** — a `concurrent` limit per model smooths thundering herds against providers with strict parallelism quotas.
- **Tiered plans** — combine a match condition with a limit, e.g. "app 42 gets 10,000 tokens/minute, everyone else gets 1,000".
- **Trial safely** — run a new rule in `log` (shadow) mode first to see who *would* be blocked before enforcing it.

## Rules

A rule has:

- **Dimensions** ("group by") — which counter bucket a request falls into. Valid dimensions: `app_id`, `user_id`, `model`, `llm_id`, `api_key`, `global`. A rule with dimension `user_id` maintains one counter per user; `global` maintains a single shared counter. Dimensions compose: `["app_id", "model"]` limits each app-and-model pair independently.
- **Match conditions** ("where", optional) — restrict which requests a rule applies to, e.g. only `model = gpt-4o`. All conditions must match. (`global` is not valid as a match condition.) Note that the `model` dimension resolves to the LLM's **slug** as configured in AI Studio, which may differ from the vendor model name in the request body.
- **Limit** — the type (`requests`, `tokens`, `concurrent`) and value (must be > 0).
- **Action** — `enforce` (block on breach, the default behavior) or `log` (shadow mode: the breach is logged and flagged in a response header, but the request goes through).
- **Priority** — rules are evaluated top-to-bottom in the order shown in the rule manager; the first enforcing rule that is breached blocks the request.
- **Enabled** — toggle a rule without deleting it.

If a request lacks a dimension a rule needs (e.g. an anonymous request for a `user_id` rule), that rule is skipped for the request.

For the `api_key` dimension, the key's identity comes from auth claims when available, falling back to a SHA-256 hash of the `Authorization` header — the raw credential is never stored.

## How the sliding window works

Counters use a two-bucket sliding-window approximation: the current and previous fixed windows are blended in proportion to how far into the current window "now" is. This gives smooth limiting without storing per-request timestamps. The window length is a single plugin-wide setting shared by all rules (default **60 seconds**, configurable 10–3600).

Token limits are enforced with a check-then-count pattern: the counter is checked *before* the provider call and the response's **actual** reported usage (OpenAI `prompt_tokens`/`completion_tokens` or Anthropic `input_tokens`/`output_tokens` formats) is added *after*. A single request can therefore finish over the limit — subsequent requests are then blocked until the window slides.

Concurrent limits are slot-based: a slot is atomically claimed before the provider call (only if below the limit) and released when a non-streaming response is written. A slot that is never released — including the slot of every *streaming* response (see Limitations) — clears only after its counter has seen no activity for 5 minutes.

## Configuration

Global plugin settings (rules themselves are managed in the UI):

| Option | Default | Description |
|--------|---------|-------------|
| `fail_open` | `true` | If counter storage is unavailable: `true` lets requests through (tagged `X-RateLimit-Error: storage_unavailable`); `false` rejects with **503** |
| `storage_backend` | `kv` | `kv` = built-in KV store (single instance); `redis` = shared counters for multi-instance deployments |
| `redis_url` | *(empty)* | Redis connection URL, e.g. `redis://localhost:6379/0` (only with `storage_backend: redis`) |
| `window_size_seconds` | `60` | Sliding window length (10–3600 seconds) |

If Redis is configured but unreachable at startup, the plugin logs the failure and falls back to the KV backend.

> **Multi-gateway deployments:** with the default `kv` backend, each gateway instance counts independently — a "100 requests/minute" rule across 3 gateways allows up to ~300/minute in total. Use the `redis` backend when you need one shared limit across instances.

## What API consumers see

On allowed requests that matched at least one rule, the response carries standard rate-limit headers reflecting the **most restrictive** matching rule (the one with the least remaining):

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | The limit value |
| `X-RateLimit-Remaining` | How much allowance is left |
| `X-RateLimit-Reset` | Unix timestamp one window from now (with a sliding window, usage decays gradually rather than dropping to zero at that instant) |
| `X-RateLimit-Tokens-Used` | Actual tokens consumed by this request (non-streaming responses only, when a `tokens` or `concurrent` rule matched and the provider reported usage) |
| `X-RateLimit-Shadow-Breach` | Name of a `log`-mode rule this request would have breached |
| `X-RateLimit-Error` | `storage_unavailable` when counters couldn't be read and fail-open let the request through |

On a blocked request (**429**):

```json
{
  "error": "Rate limit exceeded",
  "rule": "per-user-requests",
  "limit_type": "requests",
  "limit_value": 100,
  "current_usage": 100,
  "reset_at": "2026-08-27T12:34:56Z"
}
```

with `X-RateLimit-Limit`, `X-RateLimit-Remaining: 0`, `X-RateLimit-Reset`, and `Retry-After` (seconds) headers.

## The rule manager UI

The plugin adds a **Rate Limiting** section to the AI Studio admin sidebar where you create, edit, reorder, enable/disable, and delete rules. Rule changes are validated (dimension names, limit types, positive values) and versioned. No restart is needed: on AI Studio changes apply immediately; edge Microgateways pick them up with the next configuration push from Studio.

## Limitations

- **Streaming responses don't count tokens.** Token usage is read from complete response bodies; streamed responses are never counted against `tokens` rules (`requests` rules still apply to them normally). Streaming responses also hold their `concurrent` slot until the 5-minute inactivity expiry rather than releasing it at end of stream.
- With the `kv` backend, limits are per gateway instance (see the multi-gateway note above).
- The window size is a single plugin-wide setting shared by all rules, not per-rule.
