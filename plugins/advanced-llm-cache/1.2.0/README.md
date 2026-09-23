# Advanced LLM Cache Plugin (Enterprise)

Enterprise-grade LLM response caching with distributed backends, advanced observability, failover mode, hierarchical TTL policies, audit logging, and cost-based caching rules.

## Overview

The Advanced LLM Cache Plugin extends the community LLM Cache Plugin with enterprise features designed for production deployments at scale. It requires a valid enterprise license with the `feature_advanced_llm_cache` entitlement.

## Features

### Core Features (Inherited from Community)
- Deterministic cache key generation from request components
- Prompt normalization for improved hit rates
- Namespace isolation (api_key, user_id, app_id, org_id)
- LRU eviction when cache is full
- Streaming response caching and replay
- Edge-to-control statistics aggregation

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

#### Cache Sharding
- Consistent hashing for key distribution
- Weighted shard assignment
- Failover shard configuration

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
      "database": 0,
      "tls_enabled": false,
      "pool_size": 10,
      "key_prefix": "llm-cache:"
    }
  }
}
```

### Redis Cluster Mode

```json
{
  "backend": {
    "type": "redis",
    "redis": {
      "cluster_mode": true,
      "cluster_addresses": [
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
      "timeout_ms": 1500
    }
  }
}
```

- `threshold` is cosine similarity (0.5 to 1.0; higher is stricter). Start at 0.95 and lower it only after checking hits in the audit log.
- For Azure OpenAI set `api_version`; the key is then sent as `api-key`. For Ollama use `"base_url": "http://ollama:11434/v1"` with, for example, `nomic-embed-text` (768 dimensions).
- The API key is redacted in `getConfig`/`getFullConfig`. A redacted key is kept on save only while `base_url` is unchanged.
- With the Redis backend, vectors live in the same Redis in one index per embedding model (`<key_prefix>vecidx:<id>`). Plain Redis 7 without the query engine and Redis cluster mode are not supported: semantic caching reports the error in `getBackendHealth` and exact caching keeps working.
- Apps can opt out with metadata `"semantic_cache": "off"` (Cache Settings page) and requests with `X-Cache: exact-only`.

Response headers on a hit: `X-Cache-Status: HIT`, `X-Cache-Match: exact|semantic`, and for semantic hits `X-Cache-Similarity`.

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

### Sharding

```json
{
  "sharding": {
    "enabled": true,
    "shard_count": 4,
    "hash_algorithm": "consistent",
    "shard_weights": {
      "0": 2,
      "1": 1,
      "2": 1,
      "3": 1
    },
    "failover_shard": "0"
  }
}
```

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
2. Consider enabling sharding
3. Review max entry size settings
4. Monitor p99 latency in dashboard

## Support

For enterprise support, contact your Tyk account representative or visit the support portal.
