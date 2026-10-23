# LLM Load Balancer Plugin (Enterprise)

An enterprise plugin for Tyk AI Studio that provides intelligent load balancing across multiple LLM backend endpoints. It enables high availability, fault tolerance, and optimized routing for LLM requests.

## Features

- **Multiple Load Balancing Strategies**
  - Round Robin - Distribute requests evenly across upstreams
  - Weighted Round Robin - Distribute based on configured weights
  - Latency-Based - Route to the fastest responding upstream
  - Least Connections - Route to the upstream with fewest active requests

- **Health Checks**
  - HTTP health checks with configurable endpoints
  - TCP connectivity checks
  - Inference-based health checks (validate actual LLM responses)
  - Configurable thresholds for healthy/unhealthy status

- **Resilience**
  - Circuit breakers to prevent cascading failures
  - Automatic failover to backup pools
  - Sticky sessions for conversation continuity

- **Advanced Features**
  - Token-aware routing (route based on context size)
  - Cost-aware routing (optimize for cost vs latency)
  - Priority tiers for upstream selection
  - Audit logging for routing decisions

## Installation

The plugin is automatically available when running Tyk AI Studio with an enterprise license that includes the `feature_llm_load_balancer` capability.

## Configuration

### Via the UI

1. Navigate to **LLM Load Balancer** in the sidebar
2. Go to **Settings** to configure global options (strategy, health checks, circuit breakers)
3. Go to **Upstream Pools** to create and manage pools of backend endpoints

### Creating an Upstream Pool

1. Click **Create Pool**
2. Enter a **Pool ID** (unique identifier, e.g., `openai-cluster`)
3. Enter a **Pool Name** (display name)
4. **Select LLMs** - Choose which LLM entries should use this pool
   - The plugin will automatically attach itself to selected LLMs
   - Use the **Advanced** section for wildcard patterns (e.g., `gpt-*`)
5. Choose a **Load Balancing Strategy**
6. Add one or more **Upstreams** with:
   - Upstream ID and Name
   - Backend URL (an absolute `http://` or `https://` URL with a host; other schemes are rejected)
   - Priority (lower = higher priority)
   - Weight (for weighted strategies)
7. Click **Save**

In a stored configuration, an upstream's `enabled` defaults to `true` when it is left out; set `"enabled": false` to take an upstream out of rotation.

### Configuration Structure

```json
{
  "enabled": true,
  "default_strategy": "round_robin",
  "upstream_pools": {
    "my-pool": {
      "id": "my-pool",
      "name": "My Pool",
      "slug_patterns": ["OpenAI", "gpt-*"],
      "llm_ids": [1, 24],
      "strategy": "round_robin",
      "upstreams": [
        {
          "id": "primary",
          "name": "Primary Backend",
          "url": "https://api.openai.com",
          "priority": 1,
          "weight": 100,
          "enabled": true
        },
        {
          "id": "fallback",
          "name": "Fallback Backend",
          "url": "https://backup.openai.com",
          "priority": 2,
          "weight": 50,
          "enabled": true
        }
      ]
    }
  },
  "health_check": {
    "enabled": true,
    "type": "http",
    "interval_seconds": 30,
    "timeout_seconds": 5,
    "http_path": "/health",
    "unhealthy_threshold": 3,
    "healthy_threshold": 2
  },
  "circuit_breaker": {
    "enabled": true,
    "failure_threshold": 5,
    "failure_window_seconds": 60,
    "recovery_time_seconds": 30,
    "half_open_requests": 3
  },
  "sticky_session": {
    "enabled": false,
    "affinity_type": "user",
    "ttl_seconds": 3600
  }
}
```

## How It Works

### Request Flow

1. A request arrives at the AI Studio proxy for an LLM
2. The plugin's `post_auth` hook intercepts the request
3. The pool matcher finds the appropriate pool based on slug patterns
4. The balancer selects an upstream using the configured strategy
5. Health status and circuit breaker state are checked
6. The request URL is rewritten to the selected upstream
7. On response, metrics are updated and latency is recorded

### LLM Auto-Attachment

When you select LLMs in a pool configuration, the plugin automatically adds itself to those LLMs' plugin chains. This means:

- You don't need to manually configure each LLM to use the load balancer
- The plugin is automatically invoked for requests to those LLMs
- Removing an LLM from all pools does not automatically detach the plugin (manual cleanup may be needed)

### Health Check Types

| Type | Description |
|------|-------------|
| `http` | Makes HTTP GET/HEAD requests to a health endpoint |
| `tcp` | Checks if the upstream port is reachable |
| `inference` | Sends a test prompt and validates the response |

### Audit Logging

With `audit.enabled`, routing decisions and other events are logged to stdout or, with `output_type: "file"`, to `file_path`. The file is created with mode `0600` (owner read/write only), and an existing file is tightened to `0600` when the plugin opens it. Logged upstream URLs never include credentials, query strings or fragments. With `redact_urls: true`, only the scheme and host are logged.

### Circuit Breaker States

| State | Description |
|-------|-------------|
| Closed | Normal operation, requests flow through |
| Open | Too many failures, requests are rejected |
| Half-Open | Testing if upstream has recovered |

A request counts as a failure for the circuit breaker and the error metrics when the upstream failed, refused or throttled it:
- a streamed response ends with a status outside 2xx (before 1.2.0, only 5xx counted), or, for Bedrock, carries an exception event such as `throttlingException` after its 200;
- a buffered response body has an `error` field (OpenAI, Anthropic, Google, Azure), or, for Bedrock, is an AWS error: an `X-Amzn-ErrorType` header, or a `{"message": ...}` / `__type` body. A Bedrock 403 from an upstream that rejects the signature counts.

A client error does not count: the status 400, 413 or 422, an error typed `invalid_request_error` (OpenAI, Anthropic; OpenAI's `invalid_api_key` and `model_not_found` still count, as they are about the upstream), `request_too_large`, `INVALID_ARGUMENT`, `content_filter` or `context_length_exceeded`, a Google or Azure error whose numeric `code` or `status` is 400, 413 or 422, and Bedrock's `ValidationException`. Every upstream would refuse such a request the same way, so one app's malformed requests cannot open the circuit for everyone; the breaker counts the response as a success, since the upstream answered.

Streaming is detected per vendor, so Bedrock `converse-stream` and `invoke-with-response-stream` requests and Gemini `streamGenerateContent` requests are tracked as streams. Token-aware routing estimates input from string content and from content blocks' text (OpenAI, Anthropic, Bedrock Converse `messages`/`system`, Gemini `contents`/`systemInstruction`), and output from `max_tokens`, `max_completion_tokens`, `max_output_tokens`, Converse `inferenceConfig.maxTokens` or Gemini `generationConfig.maxOutputTokens`.

## Plugin Order

An LLM's plugins run in one order, the order of the LLM's plugin list. `pre_auth` plugins run before any `post_auth` plugin, and each `post_auth` plugin gets the body the previous one returned. A `post_auth` block (a cache hit, a 429, a firewall block) answers at once: later plugins, request filters and every response hook are skipped for that request. On `/ai/`, `/v1/`, routers and `/anthropic/` the chain runs on the gateway's internal `/llm/call/` hop, once per failover attempt, on the vendor's native body.

Recommended `post_auth` order: LLM Firewall (when it runs in `post_auth`), response caches, Rate Limiter, Prompt Compressor, LLM Load Balancer.

- **Put the load balancer last.** It never blocks and never changes the body; it only sets the upstream. Token-aware routing estimates tokens from the body it sees, so after the Prompt Compressor it estimates what the upstream will receive (before it, it overestimates, which is safe). Last also means a request blocked in `post_auth` never counts as an active connection.
- A request blocked after the load balancer, by a request filter or guardrail, gets no response phase. It holds its active connection until the pending entry expires after five minutes.
- The load balancer retries nothing itself. Gateway failover runs the chain again for each attempt.
- Microgateway edges managed by AI Studio follow the order set on the LLM from AI Studio 2.3. Older edges run an LLM's plugins in the order the plugins were installed.

## Monitoring

### Dashboard

The **Dashboard** page shows:
- Overall system health
- Active pools and their status
- Recent routing decisions
- Error rates

### Health Status

The **Health Status** page shows:
- Per-upstream health status
- Circuit breaker states
- Last check times and results

### Metrics

The **Metrics** page shows:
- Request counts per pool/upstream
- Latency histograms
- Error rates over time

## API

The plugin exposes RPC methods for programmatic access:

| Method | Description |
|--------|-------------|
| `getFullConfig` | Get the complete configuration |
| `updateConfig` | Update the configuration |
| `validateConfig` | Validate a configuration without applying |
| `getHealth` | Get health status of all upstreams |
| `getUpstreamStatus` | Get detailed upstream metrics |
| `resetCircuitBreaker` | Reset a circuit breaker to closed state |
| `listLLMs` | List available LLMs for selection |

## Requirements

- Tyk AI Studio 2.3.0 or later (version 1.2.0 of the plugin; see [CHANGELOG.md](CHANGELOG.md))
- Enterprise license

## Troubleshooting

### Plugin not intercepting requests

1. Verify the plugin is attached to the LLM (check LLM settings)
2. Verify the pool's slug patterns match the LLM name (empty patterns are rejected and never match)
3. Check that the plugin is active in the Plugins list
4. Check the plugin log for `invalid stored config`. The UI validates a configuration when you save it. When the plugin starts, it drops anything in its stored configuration that would fail that validation, and logs each item it drops: upstreams without an ID or an absolute http(s) URL, blank slug patterns, pools left with no upstream, and failovers to a missing pool. The plugin still loads, so valid pools keep routing. The configuration page shows the configuration as loaded, without the dropped entries, so the next save from it removes them from the stored configuration as well. To keep a dropped pool or upstream, re-enter it with valid settings before saving.

### Health checks failing

1. Verify the health check endpoint is correct
2. Check network connectivity to upstreams
3. Review timeout settings (may need to be increased)

### Circuit breaker staying open

1. Check upstream availability
2. Review failure threshold settings
3. Use the "Reset Circuit Breaker" action in the UI

## License

This plugin requires an enterprise license. Contact Tyk for licensing information.
