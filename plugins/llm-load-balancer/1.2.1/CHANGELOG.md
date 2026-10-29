# Changelog

## [1.2.1] - 2026-10-29

Requires AI Studio 2.3.0.

### Fixed
- **Pools saved from the plugin's pages load on edges and after a reload.**
  The Upstream Pools and Settings pages, and the configuration form, store
  the configuration as top-level keys (`upstream_pools`, `circuit_breaker`,
  ...), but the plugin only read a JSON string under `config` when it
  started. The pools only lived in the Studio process that received the
  save: edges and a reloaded plugin started with no pools, and every request
  went to the LLM's own endpoint. The plugin now reads the top-level keys; a
  configuration stored under `config` still loads, and top-level keys win
  over it.
- **`X-LB-Upstream` and `X-LB-Pool` reach streamed and bridged responses.**
  They were only set on a buffered response's body, so streams, and `/ai` and
  `/v1` calls to non-Bedrock LLMs (whose inner call streams), never carried
  them. They are now also set with the response headers. `X-LB-Latency-Ms`
  still measures a whole buffered response, so it is still only on buffered
  responses that are buffered end to end (`/llm/` calls, Bedrock bridges).

## [1.2.0] - 2026-10-23

Requires AI Studio 2.3.0. Built on the 2.3 Plugin SDK (64 MiB plugin message
limit).

### Changed
- A streamed response with any status outside 2xx, except a client error,
  counts as a failure for the circuit breaker and the error metrics. Before,
  only 5xx counted on streams.
- Client errors (400, 413, 422, and the vendors' invalid-request error types,
  such as `invalid_request_error` or Bedrock's `ValidationException`) no
  longer count as failures, on buffered responses or streams. Before, any
  buffered error body counted, so one app's malformed requests could open the
  circuit for every app.

### Fixed
- **An upstream saved without `enabled` is used.** The configuration schema
  defaults `enabled` to true, but a stored upstream without the field was read
  as disabled and never selected: every request went to the LLM's own
  endpoint, and the log blamed health ("no healthy upstreams"). A missing
  `enabled` now means true; an explicit `false` still disables the upstream.
  A pool with no enabled upstream logs "no enabled upstream in pool ...".
- **Bedrock.** AWS errors (`X-Amzn-ErrorType`, `{"message": ...}` and
  `__type` bodies) count as failures, including the 403 an upstream returns
  when it rejects the request signature; before, they counted as successes and
  never opened the circuit. An exception event after a 200 on a Bedrock stream
  (for example `throttlingException`) counts as a failure.
- Streaming is detected per vendor with the Plugin SDK helper, so Bedrock
  `converse-stream` / `invoke-with-response-stream` and Gemini
  `streamGenerateContent` requests are tracked as streams.
- Token-aware routing reads content blocks (Anthropic, Bedrock Converse
  `messages` and `system`, Gemini `contents` and `systemInstruction`) and the
  vendors' output limits (`max_completion_tokens`, `max_output_tokens`,
  `inferenceConfig.maxTokens`, `generationConfig.maxOutputTokens`). Before, a
  request with content blocks was estimated from its raw size.
- A request whose response hooks never run (answered from post_auth by
  another plugin such as a cache, or a stream that broke off) now gives its
  active connection back when its pending entry expires after five minutes.
  Before, it held the upstream's count up for good, skewing least-connections
  routing and concurrency limits.

## [1.1.1] - 2026-10-14

### Fixed
- Upstream selection no longer races under concurrent requests
- Upstream URLs must be absolute http(s) URLs; URLs are redacted in logs, health output and audit entries, and the audit file is created with mode 0600
- At start-up, stored entries that fail validation (an upstream without an ID or a valid URL, blank slug patterns, a pool left without upstreams, a failover to a missing pool) are dropped with a logged error instead of loaded unchecked. The configuration page shows the configuration as loaded, so saving it removes the dropped entries

### Changed
- Requires AI Studio 2.3.0

## [1.1.0] - 2026-09-02

### Fixed
- Repair the licensing vet failure and align the module Go version

### Changed
- Minimum AI Studio version is now 2.1
