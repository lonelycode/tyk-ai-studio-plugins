# Changelog

## [1.3.0] - 2026-10-23

Requires AI Studio 2.3.0. Built on the 2.3 Plugin SDK (64 MiB plugin message
limit).

### Changed
- **Cache keys are scoped to the LLM.** The exact key now covers the LLM (ID
  and vendor), the vendor path after the route slug and the model, as well as
  the prompt. The same prompt sent to two LLMs or two models is cached twice.
  `/llm/rest/`, `/llm/stream/` and `/llm/call/` share entries for the same LLM
  and path. Upgrading re-keys every entry, so the cache starts empty.
- The model, streaming flag and token usage are read with the Plugin SDK
  helpers, so models named in the request path (Bedrock `/model/{modelId}/`,
  including escaped ARNs and inference profiles; Gemini `models/{model}:`) and
  camelCase usage are handled.

### Fixed
- **Bedrock (Converse).** Different Bedrock models no longer share entries.
  `inferenceConfig`, `toolConfig`, `additionalModelRequestFields`,
  `additionalModelResponseFieldPaths`, `guardrailConfig` and `promptVariables`
  are part of the key and of the semantic partition. Bedrock errors
  (`{"message": ...}` bodies, `X-Amzn-ErrorType`, anything that is not a
  Converse answer) are never cached. `converse-stream` requests bypass the
  cache, so a cached `/converse` answer is never served to a stream. Token
  savings are counted for Bedrock hits.
- Tool calls, tool results and non-text content blocks (any vendor) are part
  of the exact key; prompts that differed only there used to share an entry.
- The semantic layer skips Converse tool turns and media/document blocks, and
  reads the temperature from `inferenceConfig`.
- A streamed response with a non-2xx status is never cached.
- A hit no longer replays headers that described the request that was
  stored: the Prompt Compressor's `X-Tyk-Compression*`, the rate limiter's
  `X-RateLimit-*`, the load balancer's `X-LB-*`, `Server-Timing` and request
  IDs. Those plugins do not run for a hit, so the headers reported another
  request's compression, usage and upstream. A vendor's own `X-RateLimit-*`
  headers (OpenAI's remaining-requests counters, say) describe the original
  request too and are dropped with them; the vendor's other headers are still
  replayed.

## [1.2.2] - 2026-10-14

### Added
- Declares the `apps.write` service scope, needed to save per-app cache settings on AI Studio 2.3.0. Approve it when upgrading

### Fixed
- Redis `tls_enabled` now takes effect (it was ignored and the connection was plaintext). Check that your Redis endpoint accepts TLS before upgrading, and rotate the Redis password if it crossed the network in plaintext
- The Redis database and cluster addresses set in the configuration form now apply. A stored `tls_skip_verify` stays ignored, with a warning; use `tls_insecure` to turn off certificate verification
- The Redis password is masked in the configuration form
- Configuration updates no longer race in-flight requests, and audit events are never written to a closed audit log
- Backend swaps no longer race requests, and shutdown is idempotent
- The audit file is created with mode 0600

### Removed
- The sharding, Redis read/write timeout and cost-rule settings, which the plugin never used

### Changed
- Requires AI Studio 2.3.0

## [1.2.1] - 2026-09-25

### Fixed
- Semantic caching missed its first question after a restart or config push: the first embedding also opens the provider connection (about 1.4 s to OpenAI), timed out at the 1500 ms default, and the prompt was never indexed, so its rewording could not hit. The embedder is now warmed up in the background when semantic caching is configured, and a prompt whose lookup embedding times out is embedded again in the background (longer timeout, at most 4 at a time) so its response is still indexed.

### Changed
- Default embedder timeout (`semantic.embedder.timeout_ms`) is now 3000 ms. Configs that set a value explicitly keep it.

## [1.2.0] - 2026-09-23

### Added
- Semantic caching (off by default): after an exact-match miss, the last user message is embedded and the cached answer to a similar prompt is served. Matches stay within the same app, endpoint, model, system prompt, conversation history, tools, output settings and temperature.
- Any OpenAI-compatible embeddings endpoint (OpenAI, Azure OpenAI, Ollama, vLLM, TEI, LiteLLM); vectors in memory or in Redis with the query engine (Redis 8+ / Redis Stack)
- Semantic tab with a Test Embedder button, dashboard tiles, per-app opt-out, `X-Cache: exact-only` request header
- `X-Cache-Match` and `X-Cache-Similarity` response headers, semantic metrics and audit events

### Fixed
- Exact cache keys ignored prompts outside `messages`: Gemini requests, OpenAI Responses `input`/`instructions`, Anthropic system blocks and Anthropic tools could be served another prompt's cached answer. Plain chat keys are unchanged.
- Streamed cache hits returned an empty body on the gateway (stale `Content-Length`)

### Notes
- Embeddings do not capture word order or direction ("miles to km" vs "km to miles" scores above a real paraphrase). Enable semantic caching for FAQ-style traffic; see the README.

## [1.1.0] - 2026-09-02

### Security
- Redact the Redis password in RPC config responses

### Fixed
- Stop the hit-count update resurrecting deleted cache entries
- Repair the licensing vet failure and align the module Go version

### Changed
- Minimum AI Studio version is now 2.1

## [1.0.3] - 2026-03-08

### BugFix
- Fix compression header handling

## [1.0.2] - 2025-12-04

### Added
- Multiple bugfixes for tier-based cache policies.

## [1.0.2] - 2025-12-04

### Added
- Confiuguratiuon UI
- Quick start UI


## [1.0.1] - 2025-12-04

### Added
- Added advanced tuning fields to RedisConfig: ConnectTimeoutSeconds, AsyncUpdateTimeoutSeconds, ScanBatchSize, MaxEntrySizeBytes, LogAsyncErrors
- Added logAsyncErrors and scanBatchSize fields
- Implemented configurable connect timeout
- Configurable replicas

## [1.0.0] - 2025-12-04

### Added
- Initial release of Advanced LLM Cache (Enterprise)
- Redis backend support (single node and cluster mode)
- Failover mode with stale response serving
- Hierarchical TTL policies (token cost, endpoint, user tier)
- Fine-grained bypass rules (model family, user tier, regulatory class, load-based)
- Cost-based caching rules
- Audit logging (stdout, file, syslog) with content redaction
- Advanced observability (latency histograms p50/p90/p99, per-namespace stats)
- Cache sharding with consistent hashing
- Enterprise dashboard with real-time metrics
- License validation with graceful degradation to community features
