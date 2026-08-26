# LLM Response Cache

**Cut LLM costs and latency by serving repeated requests from a cache instead of calling the provider again.**

| | |
|---|---|
| Plugin ID | `com.tyk.llm-cache` |
| Tier | Community |
| Runs in | AI Studio (embedded gateway) and Microgateway (edge) |
| Hooks | `post_auth` (request phase), `on_response` (response phase), `studio_ui` (dashboard) |
| Admin UI | **LLM Cache → Cache Dashboard** (`/admin/llm-cache/dashboard`) |
| Requires | AI Studio ≥ 2.6.0, Microgateway ≥ 1.0.0 |

## What it does

When two requests ask the same model the same thing, the second one doesn't need to hit the LLM provider at all. This plugin intercepts each LLM request after authentication, computes a deterministic fingerprint of the request, and checks an in-memory cache:

- **Cache HIT** — the stored response is returned immediately. The upstream LLM call is skipped entirely, so you pay nothing and the response returns in milliseconds.
- **Cache MISS** — the request proceeds to the provider as normal, and the response is stored for next time.

Both regular JSON and streaming (SSE) responses are supported. Streaming responses are reconstructed into a complete response for storage; when a streaming request gets a cache HIT, the cached response is converted back into an SSE stream, so streaming clients work unchanged. Stream reconstruction supports Anthropic, Google (Gemini/Vertex), and OpenAI-compatible response formats (unrecognized vendors are treated as OpenAI-compatible, which also covers Ollama).

Error responses from the provider (bodies containing an `error` field) are never cached, so a transient upstream failure won't be replayed to later callers.

## When to use it

- **Repeated identical prompts** — FAQ-style bots, canned assistant greetings, classification or extraction pipelines that see the same inputs repeatedly.
- **Development and CI** — test suites and demo environments replay the same prompts constantly; caching makes them fast and nearly free.
- **Evaluation runs** — when re-running an eval set with unchanged prompts, only changed cases hit the provider.
- **Traffic spikes on shared content** — many users triggering the same generation (e.g. a "summarize this article" button) get one upstream call instead of thousands.

One consideration: a cache HIT returns the *same* response every time until the entry expires. For endpoints where response variety matters (creative writing at high temperature), use a short TTL or have clients bypass the cache (see below).

## How requests are matched

The cache key is a SHA-256 hash over the parts of the request that determine the answer:

- **Namespace** — a tenant isolation prefix (see *Namespace isolation* below)
- **Model** — the model identifier
- **Messages** — all chat messages (role + content; base64-embedded images are fingerprinted by a hash of their content, while URL-referenced images are keyed by their URL)
- **System prompt** — top-level `system` field, when present
- **Tools** — tool definitions, sorted by name so ordering doesn't matter
- **Temperature** — requests with different temperatures never share a cache entry (treated as `1.0` when not set)

With **prompt normalization** enabled (the default), formatting differences like extra whitespace and JSON property ordering are ignored, so the same prompt written slightly differently still matches.

### Namespace isolation

By default, cache entries are isolated per **App**: responses cached for one App are never served to another. Both the `api_key` (default) and `app_id` namespace settings currently isolate at App granularity — the plugin cannot see the raw API key, so it uses the App identity as a proxy. This means different API keys belonging to the *same* App share cache entries. The available settings:

- `api_key` — isolates per App (App identity stands in for the key; default)
- `app_id` — isolates per App
- `org_id` — shared across an organization (taken from request metadata)

Broader namespaces increase hit rates but share responses across more callers; only widen them when the cached content isn't caller-specific. Be careful with `org_id` as the *only* namespace: if the organization can't be determined from a request, entries fall into a single shared default namespace.

## Configuration

Configure the plugin from its settings form in AI Studio:

| Option | Default | Description |
|--------|---------|-------------|
| `enabled` | `true` | Turn caching on or off |
| `ttl_seconds` | `3600` | How long entries live (60–86400 seconds) |
| `max_entry_size_kb` | `2048` | Responses larger than this are not cached (1–10240 KB) |
| `max_cache_size_mb` | `256` | Total cache budget; least-recently-used entries are evicted beyond this (1–4096 MB) |
| `namespaces` | `["api_key"]` | Isolation fields: `api_key`, `app_id`, `org_id` |
| `normalize_prompts` | `true` | Collapse whitespace / canonicalize JSON before hashing |
| `expose_cache_key_header` | `false` | Add `X-Cache-Key` to responses (debugging) |
| `cache_streaming_responses` | `true` | Cache SSE responses and re-stream them on HIT |
| `report_interval_seconds` | `60` | How often edge gateways report cache stats to the control plane (10–300) |

Expired entries are also swept by a background cleanup that runs every minute.

## What API consumers see

Non-streaming responses passing through the plugin carry a status header (streaming responses only carry cache headers on a HIT, because a streaming HIT is served as a synthesized stream — a streaming MISS or BYPASS passes through unmarked):

| Header | Meaning |
|--------|---------|
| `X-Cache-Status` | `HIT`, `MISS`, or `BYPASS` |
| `X-Cache-Age` | Seconds since the entry was cached (on HIT) |
| `X-Cache-TTL` | Seconds until the entry expires (on HIT) |
| `X-Cache-Key` | The cache key (only when `expose_cache_key_header` is on) |

### Bypassing the cache per request

Clients can force a fresh provider call for a single request:

```bash
# Via header (case-insensitive)
curl -H "X-Cache: bypass" https://gateway.example.com/v1/chat/completions ...

# Via query parameter
curl "https://gateway.example.com/v1/chat/completions?cache=bypass" ...
```

(The query check is a substring match — any request whose URL contains `cache=bypass` bypasses the cache.) A bypassed request is answered by the provider and marked `X-Cache-Status: BYPASS`.

> **Debugging tip:** a response larger than `max_entry_size_kb` is never stored, but is still reported as `MISS` — if a request never produces a HIT, check the response size against that limit.

## The Cache Dashboard

The plugin adds an **LLM Cache** section to the AI Studio admin sidebar with a real-time dashboard showing:

- Hit rate, hit/miss/bypass counts
- Estimated tokens saved by cache hits
- Active entry count, memory usage vs. capacity, and eviction count
- A **clear cache** action — note that clearing the cache also resets the dashboard counters to zero

## Running at the edge (Microgateway)

The plugin also runs on edge gateways. Each edge keeps its own local cache and periodically reports statistics to AI Studio (at `report_interval_seconds`), where the dashboard aggregates metrics across edges. Clearing the cache from the dashboard triggers a **distributed clear**: connected edges are instructed to flush their local caches and acknowledge completion, and the dashboard tracks the operation's status.

## Limitations

- **In-memory only** — the cache does not survive a restart. (Persisted, distributed backends such as Redis are available in the enterprise [Advanced LLM Cache](advanced-llm-cache.md) plugin.)
- **Per-instance** — gateway instances do not share cache entries with each other; only statistics are aggregated centrally.
- **Local metrics reset** — an instance's own counters live in memory and reset on restart (and when the cache is cleared). Only the per-edge statistics aggregated on the control plane are persisted across restarts.
