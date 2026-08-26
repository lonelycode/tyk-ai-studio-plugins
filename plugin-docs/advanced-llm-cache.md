# Advanced LLM Cache (Enterprise)

**Production-grade LLM response caching: Redis-backed and shared across gateways, with policy-driven TTLs, compliance-aware bypass rules, and audit logging.**

| | |
|---|---|
| Plugin ID | `com.tyk.enterprise.advanced-llm-cache` |
| Tier | Enterprise (requires a valid enterprise license) |
| Runs in | AI Studio (embedded gateway) and Microgateway (edge) |
| Hooks | `post_auth` (request phase), `on_response` (response phase), `studio_ui` |
| Admin UI | **Advanced LLM Cache** sidebar section: Configuration, Cache Dashboard, Cache Entries, Backend Health, App Cache Settings (under `/admin/enterprise/llm-cache/…`) |
| Requires | AI Studio ≥ 2.6.0, Microgateway ≥ 1.0.0 |

## What it does

This is the enterprise edition of the [LLM Response Cache](llm-cache.md). It provides the same core behavior — deterministic cache keys with prompt normalization, namespace isolation, streaming response caching and replay, LRU eviction, per-request bypass — and adds the capabilities needed to run caching as shared production infrastructure:

- **Distributed backends** — Redis (single node or cluster mode, with connection pooling and key prefixing) instead of per-instance memory, so all gateway instances share one cache and entries survive restarts.
- **Hierarchical TTL policies** — cache lifetimes decided per request by user tier, endpoint path, and/or token cost, with a configurable priority order.
- **Fine-grained bypass rules** — skip caching by token thresholds, model family, user tier, or regulatory classification (e.g. PII/HIPAA-tagged apps).
- **Audit logging** — cache decisions written to stdout, file, or syslog, with content redaction on by default for compliance.
- **Advanced observability** — latency percentiles for cache operations, per-namespace statistics, backend health monitoring, and a cache entry browser.

## When to use it

Use the community [LLM Response Cache](llm-cache.md) for a single gateway and default policies. Move to this plugin when you need:

- **A shared cache across a gateway fleet** — one Redis-backed cache means a response cached by any instance is a HIT on every instance, and cached entries survive restarts and deploys.
- **Different cache lifetimes for different content** — e.g. embeddings cached for a day, chat completions for an hour, free-tier traffic for less; expensive high-token responses kept longer than cheap ones.
- **Compliance boundaries** — guarantee that apps tagged with a regulatory class (PII, HIPAA) are never served from, or written to, the cache, and keep an audit trail of cache decisions.

## Configuration

Configuration is managed from the **Configuration** page in the plugin's admin UI. The base options are the same as the community plugin (`enabled`, `ttl_seconds` 3600, `max_entry_size_kb` 2048, `max_cache_size_mb` 256, `namespaces` `["api_key"]`, `normalize_prompts`, `expose_cache_key_header`, `cache_streaming_responses`, `report_interval_seconds`). The enterprise sections:

### Backend

| Option | Default | Description |
|--------|---------|-------------|
| `backend.type` | `memory` | `memory` or `redis` |
| `backend.redis.address` | `localhost:6379` | Redis address (single-node mode) |
| `backend.redis.cluster_mode` | `false` | Enable Redis Cluster mode |
| `backend.redis.cluster_addrs` | — | Cluster node addresses |
| `backend.redis.password` | — | Redis auth |
| `backend.redis.db` | `0` | Database selection (single-node mode) |
| `backend.redis.pool_size` | `10` | Connection pool size |
| `backend.redis.connect_timeout_seconds` | `5` | Connection timeout |
| `backend.redis.key_prefix` | `llm-cache:` | Namespace within Redis |

Connect to Redis over a trusted network: TLS options appear in the settings schema but are not applied by the current version (see *Current limitations*).

### TTL policy

`ttl_policy.enabled` turns on rule-based TTLs; `ttl_policy.priority` (default `["user_tier", "endpoint", "token_cost"]`) decides which rule set wins when several match:

- `token_cost_rules` — `{min_tokens, max_tokens, ttl_seconds}`: cache expensive responses longer (`max_tokens: 0` = unbounded).
- `endpoint_rules` — `{path_pattern, ttl_seconds}`: e.g. long TTL for `/v1/embeddings`.
- `user_tier_rules` — `{tier, ttl_seconds}`: e.g. `enterprise` 7200s, `free` 1800s.

### Bypass rules

`bypass_rules`: `min_tokens_to_cache` / `max_tokens_to_cache` (0 = no bound), `bypass_model_families`, `bypass_user_tiers`, and `bypass_regulatory_classes`. A request bypassed by a token threshold is tagged with an `X-Cache-Bypass-Reason` header for debugging.

### Audit logging

`audit.enabled` (default `false`) with per-event toggles (`log_cache_hits`, `log_cache_misses`, `log_bypass`, `log_expired`), an output (`stdout` default, `file` with `file_path`, or `syslog` with address and facility), and redaction controls — `redact_content` is **on** by default so prompt/response text never enters audit logs; `redact_keys` optionally masks cache keys too.

## User tiers and regulatory classes

TTL and bypass rules can key off two per-app attributes: **user tier** (e.g. `free`, `pro`, `enterprise`) and **regulatory class** (e.g. `pii`, `hipaa`). The **App Cache Settings** page in the plugin UI is where you assign these — the values are stored in each App's metadata and picked up by the policy engine on every request.

## The admin UI

- **Cache Dashboard** — real-time hit/miss/bypass statistics, latency percentiles (p50/p90/p99), per-namespace statistics, and backend health.
- **Cache Entries** — browse, filter, and delete individual cache entries.
- **Backend Health** — connection status for the configured backend, with a connectivity test.
- **Configuration** — the full settings described above.
- **App Cache Settings** — per-app tier and regulatory class assignment.

## License behavior

The plugin validates the platform's enterprise license shortly after startup. Without a valid enterprise license it degrades gracefully to community-level behavior: in-memory backend only, with the policy engine (TTL/bypass rules) and audit logging disabled. The dashboard shows a "Community" badge when running in this degraded mode.

## What API consumers see

Consumer-facing behavior matches the community plugin — `X-Cache-Status` (HIT/MISS/BYPASS), `X-Cache-Age`/`X-Cache-TTL` on hits, and the `X-Cache: bypass` header or `cache=bypass` query parameter to skip the cache for one request — plus `X-Cache-Backend` identifying the serving backend and `X-Cache-Bypass-Reason` on token-threshold bypasses.

## Current limitations

The settings schema declares a few options ahead of their implementation; in the current version the following have **no runtime effect** and should not be relied on:

- **Failover / stale serving** (`failover.*`) — expired entries are retained by the memory backend, but no code path serves them on upstream failure yet, and the `X-Cache-Stale` header is never emitted.
- **Cost rules** (`cost_rules.*`) — not read by the plugin.
- **Sharding** (`sharding.*`) — not wired into the backend factory.
- **Redis TLS** (`tls_enabled`, `tls_skip_verify`) — not applied to Redis connections.
- **`bypass_rules.bypass_on_high_load`** — not enforced by the policy engine.
