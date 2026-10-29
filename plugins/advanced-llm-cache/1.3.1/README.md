# Advanced LLM Cache Plugin (Enterprise)

Enterprise-grade LLM response caching with distributed backends, advanced observability, failover mode, hierarchical TTL policies, audit logging, and cost-based caching rules.

## Overview

The Advanced LLM Cache Plugin extends the community LLM Cache Plugin with enterprise features designed for production deployments at scale. It requires a valid enterprise license with the `feature_advanced_llm_cache` entitlement.

Version 1.3.0 requires AI Studio 2.3.0 or later. See [CHANGELOG.md](CHANGELOG.md).

## Features

### Core Features (Inherited from Community)
- Deterministic cache key generation from request components
- Prompt normalization for improved hit rates
- Namespace isolation (api_key, user_id, app_id, org_id)
- LRU eviction when cache is full
- Streaming response caching and replay
- Edge-to-control statistics aggregation

### Keys and vendors

The exact key covers the namespace, the LLM (ID and vendor), the vendor path after the route slug, the model, and the prompt: messages including tool calls, tool results and non-text blocks, the system prompt, tools, temperature, and other answer-shaping fields (Gemini `generationConfig`, OpenAI Responses `input`/`instructions`, Bedrock Converse `inferenceConfig`, `toolConfig`, `additionalModelRequestFields`, `guardrailConfig`, ...). The same prompt sent to two LLMs or two models is cached twice. `/llm/rest/`, `/llm/stream/` and `/llm/call/` share entries for the same LLM and path. Version 1.3.0 added the LLM and path to the key, so upgrading starts with an empty cache.

The model, streaming flag and token usage are read with the Plugin SDK helpers (`RequestModel`, `IsStreamingRequest`, `ResponseUsage`), so Bedrock's model in the `/model/{modelId}/` path (including escaped ARNs) and its camelCase usage are handled.

Bedrock specifics:
- Only Converse answers are cached. A body without `output`, or a response with `X-Amzn-ErrorType`, is an error and is never cached.
- `converse-stream` requests bypass the cache (no lookup, no store): their AWS eventstream answer cannot be replayed yet. A cached `/converse` answer is never served to a `/converse-stream` request.

A streamed response whose status is not 2xx is never cached, for any vendor.

### Enterprise Features

#### Distributed Cache Backends
- **Redis** (single node and cluster mode)
- **In-Memory** (with enterprise metrics)
- Pluggable interface for additional backends

#### Failover Mode
- Serve stale cached responses when LLM upstream is unavailable
- Configurable stale TTL (how long past expiry to keep entries)
- `X-Cache-Stale` header indicates failover response

#### Hierarchical TTL Policy
- Token cost-based TTL rules (expensive responses cached longer)
- Endpoint-based TTL rules (different TTL per route)
- User tier-based TTL rules (premium users get longer cache)
- Configurable rule priority

#### Fine-Grained Bypass Rules
- Minimum/maximum token thresholds
- Bypass by model family (e.g., skip caching for opus models)
- Bypass by user tier (e.g., no cache for free tier)
- Bypass by regulatory classification (PII, HIPAA)
- Bypass on high cache load (>90% full)

#### Cost-Based Caching
- Only cache responses above token threshold
- Serve from cache when user is over budget
- Endpoint prioritization for caching

#### Audit Logging
- Log cache hits, misses, bypasses, failovers
- Output to stdout, file, or syslog
- Content redaction for compliance
- Syslog facility configuration

#### Semantic Caching
- After an exact-match miss, the last user message is embedded and compared with cached prompts
- Hits only within the same partition: app namespace, vendor, LLM endpoint, model, system prompt, earlier turns, tools, output settings (`response_format`, `max_tokens`, ...), temperature and embedding model
- Skips requests where a similar-prompt answer is unsafe: tool calls and results, images/files, conversations deeper than `max_user_turns`, and (optionally) declared tools or high temperatures
- Any OpenAI-compatible embeddings endpoint (OpenAI, Azure OpenAI, Ollama, vLLM, TEI, LiteLLM)
- Vector index in memory, or in Redis with the query engine (Redis 8+, Redis Stack, or a managed Redis with search) when the Redis backend is used
- Fails open: embedding or search errors fall back to a normal miss
- Per-request (`X-Cache: exact-only`) and per-app opt-out

#### Advanced Observability
- Latency histograms (p50, p90, p99) for GET/SET operations
- Per-namespace statistics
- Backend health monitoring
- Real-time dashboard

## Configuration

### Basic Configuration

```json
{
  "enabled": true,
  "ttl_seconds": 3600,
  "max_entry_size_kb": 2048,
  "max_cache_size_mb": 256,
  "namespaces": ["api_key"],
  "normalize_prompts": true,
  "cache_streaming_responses": true
}
```

### Redis Backend

```json
{
  "backend": {
    "type": "redis",
    "redis": {
      "address": "localhost:6379",
      "password": "your-password",
      "db": 0,
      "tls_enabled": false,
      "pool_size": 10,
      "key_prefix": "llm-cache:"
    }
  }
}
```

With `tls_enabled: true`, the plugin connects to Redis over TLS 1.2+ and verifies the server certificate against the system trust store. This covers both the cache and the semantic vector index. Before 1.2.2 the setting was ignored, and the cache connected in plaintext. `tls_insecure: true` skips certificate verification. It logs a security warning every time a connection is set up, and should only be used for testing.

**Enter the Redis password itself, not a `$SECRET/` reference.** AI Studio resolves `$SECRET/` references only in top-level configuration values, and `backend.redis.password` is nested, so the plugin receives the reference text as the password, in AI Studio and on edges. Redis then refuses it (`WRONGPASS`), and the plugin logs `Failed to upgrade to redis backend, keeping memory` and keeps serving from a per-process in-memory cache: requests still work, but the cache is not shared. The password field is masked in the configuration form.

Before 1.2.2 the configuration form and this README used `database`, `cluster_addresses` and `tls_skip_verify`, which the plugin ignored (it parses `db`, `cluster_addrs` and `tls_insecure`): the database was always 0, cluster mode had no node addresses, and verification stayed on. `database` and `cluster_addresses` are still accepted, and the current names win when a config has both. **`tls_skip_verify` stays ignored**, so an upgrade never turns verification off by itself: a config that sets it to `true` logs a warning, and only `tls_insecure` disables verification. The form's sharding settings, read/write timeouts and cost rules were never used by the plugin and have been removed.

### Redis Cluster Mode

```json
{
  "backend": {
    "type": "redis",
    "redis": {
      "cluster_mode": true,
      "cluster_addrs": [
        "redis-1:6379",
        "redis-2:6379",
        "redis-3:6379"
      ],
      "password": "your-password"
    }
  }
}
```

### Failover Mode

```json
{
  "failover": {
    "enabled": true,
    "max_stale_seconds": 3600,
    "mark_stale_header": true,
    "stale_header_name": "X-Cache-Stale"
  }
}
```

### Semantic Caching

```json
{
  "semantic": {
    "enabled": true,
    "threshold": 0.95,
    "max_user_turns": 1,
    "skip_when_tools": true,
    "max_temperature": 1.0,
    "max_text_chars": 8000,
    "max_vectors": 100000,
    "embedder": {
      "provider": "openai",
      "base_url": "https://api.openai.com/v1",
      "api_key": "sk-...",
      "model": "text-embedding-3-small",
      "dimensions": 1536,
      "timeout_ms": 3000
    }
  }
}
```

- `threshold` is cosine similarity (0.5 to 1.0; higher is stricter). Start at 0.95 and lower it only after checking hits in the audit log.
- For Azure OpenAI set `api_version`; the key is then sent as `api-key`. For Ollama use `"base_url": "http://ollama:11434/v1"` with, for example, `nomic-embed-text` (768 dimensions).
- The API key is redacted in `getConfig`/`getFullConfig`. A redacted key is kept on save only while `base_url` is unchanged.
- With the Redis backend, vectors live in the same Redis in one index per embedding model (`<key_prefix>vecidx:<id>`). Plain Redis 7 without the query engine and Redis cluster mode are not supported: semantic caching reports the error in `getBackendHealth` and exact caching keeps working.
- Apps can opt out with metadata `"semantic_cache": "off"` (Cache Settings page) and requests with `X-Cache: exact-only`.

- `timeout_ms` (default 3000) bounds the embedding on the request path; on timeout the request continues as a normal miss. The first embedding of a new plugin process also opens the provider connection (about 1.4 s to OpenAI), so the embedder is warmed up in the background whenever semantic caching is (re)configured. A prompt whose lookup embedding still times out is embedded again in the background, with a longer timeout and at most 4 at a time, so its response is indexed anyway.

Response headers on a hit: `X-Cache-Status: HIT`, `X-Cache-Match: exact|semantic`, and for semantic hits `X-Cache-Similarity`. Older gateways drop these on the OpenAI-compatible endpoints (`/ai/{slug}/v1/...` and the unified `/v1` ingress) and only the vendor-native `/llm/call/...` endpoints carry them; current gateways relay them on all three.

**Know the failure mode before enabling it.** Embeddings capture topic, not
word order or direction. Measured with `nomic-embed-text`:

| Prompt pair | Cosine similarity |
|---|---|
| "How do I reset my password?" / "how can I reset my password" | 0.982 |
| "What is the capital of France?" / "What's France's capital city?" | 0.964 |
| "Convert 10 miles to km" / "Convert 10 km to miles" | **0.984** |
| "How do I reset my password?" / "How do I reset my username?" | 0.815 |
| "What is the capital of France?" / "What is the capital of Germany?" | 0.768 |

The unit-conversion pair scores higher than a genuine paraphrase, so no
threshold separates them: it would be served the other direction's answer.
Enable semantic caching for FAQ-style traffic (support bots, documentation Q&A),
opt out apps whose prompts differ by numbers, order or negation, and review
`X-Cache-Similarity` / audit `match=semantic` events after rollout. Scores
differ between embedding models, so re-check the threshold when you change model.

### Hierarchical TTL Policy

```json
{
  "ttl_policy": {
    "enabled": true,
    "priority": ["user_tier", "endpoint", "token_cost"],
    "token_cost_rules": [
      {"min_tokens": 1000, "max_tokens": 0, "ttl_seconds": 7200},
      {"min_tokens": 500, "max_tokens": 999, "ttl_seconds": 3600},
      {"min_tokens": 0, "max_tokens": 499, "ttl_seconds": 1800}
    ],
    "endpoint_rules": [
      {"path_pattern": "/v1/chat/completions", "ttl_seconds": 3600},
      {"path_pattern": "/v1/embeddings", "ttl_seconds": 86400}
    ],
    "user_tier_rules": [
      {"tier": "enterprise", "ttl_seconds": 7200},
      {"tier": "pro", "ttl_seconds": 3600},
      {"tier": "free", "ttl_seconds": 1800}
    ]
  }
}
```

### Bypass Rules

```json
{
  "bypass_rules": {
    "min_tokens_to_cache": 100,
    "max_tokens_to_cache": 50000,
    "bypass_model_families": ["gpt-4-turbo", "claude-3-opus"],
    "bypass_user_tiers": ["free"],
    "bypass_regulatory_classes": ["pii", "hipaa"],
    "bypass_on_high_load": true
  }
}
```

### Audit Logging

```json
{
  "audit": {
    "enabled": true,
    "log_cache_hits": true,
    "log_cache_misses": true,
    "log_bypass": true,
    "log_failover": true,
    "output_type": "syslog",
    "syslog_address": "localhost:514",
    "syslog_facility": "local0",
    "redact_content": true,
    "redact_keys": false
  }
}
```

With `output_type: "file"`, the audit file is created with mode `0600` (owner read/write only). A file created 0644 by an earlier version is tightened to 0600 when the plugin opens it.

## Plugin Order

An LLM's plugins run in one order, the order of the LLM's plugin list. `pre_auth` plugins run before any `post_auth` plugin, and each `post_auth` plugin gets the body the previous one returned. A `post_auth` block (a cache hit, a 429, a firewall block) answers at once: later plugins, request filters and every response hook are skipped for that request. On `/ai/`, `/v1/`, routers and `/anthropic/` the chain runs on the gateway's internal `/llm/call/` hop, once per failover attempt, on the vendor's native body.

Recommended `post_auth` order: LLM Firewall (when it runs in `post_auth`), this cache, Rate Limiter, Prompt Compressor, LLM Load Balancer.

- **Put the cache before the Prompt Compressor.** Exact keys then come from the client's body, and the semantic layer embeds what the client wrote, so changing a compression policy does not empty the cache. Hits never reach the compressor and are not in its savings. With the compressor first, keys come from the compressed body: identical requests under one policy still hit, but each policy change starts the cache again.
- **An entry stored while compression was enforcing holds the model's answer to the compressed prompt.** It is served for the same client body until it expires, including to a client that sent `X-Tyk-Compression: off`, which bypasses compression, not the cache. To get a fresh, uncompressed answer, send `X-Cache: bypass` as well.
- **Cache hits are not rate limited** when the cache comes before the Rate Limiter.
- An entry does not keep the response headers other plugins set about the original request (`X-Tyk-Compression*`, `X-RateLimit-*`, `X-LB-*`, `Server-Timing`, request IDs), because those plugins do not run for a hit.
- Microgateway edges managed by AI Studio follow the order set on the LLM from AI Studio 2.3. Older edges run an LLM's plugins in the order the plugins were installed.

## License Validation

The plugin validates enterprise license at initialization:
- Checks `license_valid` metadata from context
- Verifies `feature_advanced_llm_cache` entitlement
- Falls back to community features if license is invalid

### Graceful Degradation

When enterprise license is not available:
- Backend falls back to in-memory only
- Enterprise metrics are disabled
- Failover mode is disabled
- Audit logging is disabled
- UI shows "Community" license badge

## Dashboard

The enterprise dashboard includes:
- Real-time hit/miss/bypass statistics
- Failover and stale serve counts
- Semantic hits and semantic match rate
- Latency percentiles (p50/p90/p99)
- Per-namespace statistics table
- Backend health status indicator
- Cache entry browser
- Configuration overview

## API Endpoints (RPC)

| Method | Description |
|--------|-------------|
| `getMetrics` | Get cache statistics including enterprise metrics |
| `getConfig` | Get current plugin configuration |
| `getHealth` | Get backend health status |
| `getLicenseStatus` | Check enterprise license status |
| `clearCache` | Clear all cache entries |
| `listEntries` | Browse cache entries (with filters) |
| `deleteEntry` | Delete a specific cache entry |
| `testBackend` | Test backend connectivity |
| `testEmbedder` | Embed a probe string with unsaved embedder settings |

## Service scopes

AI Studio 2.3 and later only let a plugin make the management calls its approved service scopes allow. The plugin declares:

| Scope | Used for |
|-------|----------|
| `apps.read` | Reading Apps for per-app cache settings |
| `apps.write` | Saving per-app cache settings (tier, regulatory class, semantic opt-out) into the App's metadata |
| `kv.readwrite` | Plugin key-value storage |
| `scheduler.manage` | Scheduled backend health check |
| `control.send` | Sending statistics from edge gateways to AI Studio |
| `license.read` | License check |

When upgrading from 1.2.1 or earlier, approve the added `apps.write` scope; without it, saving per-app settings fails.

## Building

```bash
# Build with enterprise tag
go build -tags enterprise -o advanced-llm-cache ./enterprise/plugins/advanced-llm-cache/
```

## Future Enhancements

The following features are planned for future releases:

### Embedding Similarity-Based TTL
- Calculate semantic similarity between cached and new prompts
- Extend TTL for semantically similar requests
- Configurable similarity threshold

### Additional Backends
- **Memcached** - Interface ready, implementation pending
- **DynamoDB** - Interface ready, implementation pending
- **PostgreSQL** - For environments requiring SQL storage

### Per-Namespace Failover
- Enable failover mode per namespace instead of global toggle
- Per-app failover configuration

### Response Transformation
- Transform cached responses before serving
- Add/remove headers based on cache status
- Content modification hooks

### Multi-Region Replication
- Cross-region cache synchronization
- Conflict resolution strategies
- Latency-based routing

### Advanced Analytics
- Cost savings dashboard
- Trend analysis
- Anomaly detection

## Troubleshooting

### Redis Connection Issues
1. Check Redis server is running and accessible
2. Verify password and TLS settings
3. Check firewall rules for port access
4. Use `testBackend` RPC to diagnose

### Low Hit Rate
1. Enable `normalize_prompts` for better key matching
2. Review namespace configuration
3. Check bypass rules aren't too aggressive
4. Increase TTL for stable content

### High Latency
1. Check backend health and latency
2. Review max entry size settings
3. Monitor p99 latency in dashboard

## Support

For enterprise support, contact your Tyk account representative or visit the support portal.
