# Changelog

## [1.3.1] - 2026-10-29

Requires AI Studio 2.3.0.

### Fixed
- **`X-Cache-Status: MISS` (and `BYPASS`) reach streamed and bridged
  responses.** The status of a request that went upstream was only set on a
  buffered response's body, so streams, and `/ai` and `/v1` calls to
  non-Bedrock LLMs (whose inner call streams), never carried it. It is now
  also set with the response headers. `X-Cache-Key` on a miss is still only
  on responses buffered end to end, where the answer is stored.

## [1.3.0] - 2026-10-23

Requires AI Studio 2.3.0. Built on the 2.3 Plugin SDK (64 MiB plugin message
limit).

### Fixed
- **Bedrock.** AWS errors (`X-Amzn-ErrorType`, `{"message": ...}` and
  `__type` bodies) are never cached; before, they were stored and served as
  answers. Tokens saved are counted from Converse's camelCase usage.
  `converse-stream` and `invoke-with-response-stream` requests go upstream
  uncached.
- Tokens saved are read with the Plugin SDK's `ResponseUsage`, so Google
  `usageMetadata` counts too.
- A hit no longer replays the headers other plugins set about the original
  request (`X-Tyk-Compression*`, `X-RateLimit-*`, `X-LB-*`, `Server-Timing`).
- A streamed response whose status is not 2xx is never cached.
- Every completed stream releases its pending entry, including when
  `cache_streaming_responses` is off; before, those entries stayed until the
  five-minute cleanup.

## [1.2.1] - 2026-10-14

### Added
- The marketplace entry now lists the service scopes the plugin uses (`apps.read`, `kv.readwrite`, `control.send`); AI Studio 2.3.0 enforces approved scopes, so approve them when upgrading

### Fixed
- Background goroutines stopped at shutdown, and a repeated shutdown no longer panics on a double close
- The plugin no longer writes a debug log under `/tmp`; logging goes through the AI Studio plugin logger
- Edge instances without a configured ID get a unique generated one instead of a shared default
- Old cache operations are cleaned up as intended

### Changed
- Requires AI Studio 2.3.0

## [1.2.0] - 2026-09-30

### Fixed
- Cache hits through Studio's OpenAI-compatible endpoints (`/ai/{slug}/v1` and
  the unified `/v1` router) failed with a 502 ("unexpected EOF"): a streamed hit
  carried the stored JSON's `Content-Length`, so the gateway wrote an empty body.
  Hits now set `Content-Type` and `Content-Length` for the body actually sent
- An LLM could be served another LLM's cached response (for example an Anthropic
  LLM getting an OpenAI-compatible LLM's JSON for the same model name and
  messages). Entries are now scoped to the LLM, its vendor and the API path
- Gemini requests all shared one entry (the prompt is in `contents`, the model
  in the path). The key now covers the whole request, including `max_tokens`,
  `response_format`, `n`, `top_p`, stop sequences, `tool_choice`, Anthropic tool
  definitions and tool results; only `stream`/`stream_options` are left out
- Streamed tool calls were replayed without their call. Only plain-text,
  single-choice answers are converted between JSON and a stream; anything else
  is replayed in the form it was stored in, or goes upstream
- Gemini (`:streamGenerateContent`) and Ollama (streams by default) streaming
  requests were served JSON. Streaming is now detected per vendor, and vendors
  whose stream the cache cannot produce are never served a cached stream

### Changed
- Existing cache entries are not reused after the upgrade (the key changed)

## [1.1.0] - 2026-09-02

### Fixed
- Correct response compression handling
- Align the plugin module with the parent module's Go version

### Changed
- Minimum AI Studio version is now 2.1

## [1.0.0] - 2025-12-04

### Added
- Initial release of LLM Response Cache plugin
- Deterministic SHA-256 cache keys based on model, messages, tools, and temperature
- Prompt normalization for improved cache hit rates
- Namespace isolation (API key, app ID, org ID)
- LRU eviction when cache is full
- Configurable TTL expiration
- Cache bypass via header or query parameter
- Admin dashboard with real-time metrics
- Response headers for cache status, age, and TTL

## [1.0.1] - 2026-03-08
- Fix compression header handling