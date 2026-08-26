# LLM Load Balancer (Enterprise)

**High availability for LLM traffic: spread requests across multiple backend endpoints with health checks, circuit breakers, and automatic failover.**

| | |
|---|---|
| Plugin ID | `com.tyk.enterprise.llm-load-balancer` |
| Tier | Enterprise (requires a valid enterprise license) |
| Runs in | AI Studio (embedded gateway) and Microgateway (edge) |
| Hooks | `post_auth` (routing decision), `on_response` (metrics/latency recording), `studio_ui` |
| Admin UI | **LLM Load Balancer** sidebar section: Dashboard, Settings, Upstream Pools, Health Status, Metrics (under `/admin/enterprise/llm-lb/…`) |
| Requires | AI Studio ≥ 2.6.0, Microgateway ≥ 1.0.0 |

## What it does

Instead of every request for an LLM going to the single URL configured on that LLM, this plugin routes each request across a **pool of upstream endpoints**. On each request it:

1. Matches the request's LLM to a pool by slug pattern (selecting LLMs in the pool editor stores their slugs as exact patterns; wildcards like `gpt-*` are available under Advanced).
2. Picks an upstream using the pool's strategy, skipping upstreams that are unhealthy or whose circuit breaker is open. If the pool has no selectable upstream at all, traffic can fail over to a configured **failover pool**.
3. Rewrites the request URL to the chosen upstream.
4. Records latency and outcome on the response, feeding the circuit breakers, metrics, and latency-based routing.

### Routing strategies

| Strategy | Behavior |
|----------|----------|
| `round_robin` (default) | Even distribution across upstreams |
| `weighted_round_robin` | Distribution proportional to configured weights |
| `latency_based` | Prefer the fastest-responding upstream |
| `least_connections` | Prefer the upstream with fewest in-flight requests |

Upstreams also carry a **priority** (lower number = preferred): higher-priority upstreams serve traffic while healthy, with lower-priority ones acting as fallbacks.

## When to use it

- **Provider redundancy** — front an LLM with a primary endpoint plus a fallback (a second region, a second account, or a compatible alternative host); an outage flips traffic automatically instead of paging someone.
- **Self-hosted model fleets** — spread load across several GPU inference servers (vLLM, TGI, Ollama clusters) with least-connections or latency-based routing.
- **Capacity pooling across accounts** — distribute traffic over multiple provider accounts/keys to stay under per-account rate limits.
- **Blue/green model rollouts** — weight a new backend at a small percentage, watch metrics, then shift weight over.
- **Conversation continuity for stateful backends** — sticky sessions keep a user's requests on the same upstream when backends hold session-local state (e.g. local KV caches).

## Key capabilities

### Health checks

Configurable probing marks upstreams healthy/unhealthy so traffic avoids dead backends:

| Type | What it does |
|------|--------------|
| `http` | Requests a health endpoint (configurable path) |
| `tcp` | Checks the port is reachable |
| `inference` | Sends a test prompt and validates a real model response |

Interval, timeout, and healthy/unhealthy thresholds (consecutive successes/failures required to change state) are configurable — defaults: every 30s, 5s timeout, unhealthy after 3 failures, healthy again after 2 successes.

### Circuit breakers

Each upstream gets a circuit breaker so a failing backend can't drag down request latency:

| State | Meaning |
|-------|---------|
| **Closed** | Normal operation |
| **Open** | Too many recent failures — requests skip this upstream |
| **Half-Open** | After a recovery period, a limited number of trial requests test whether it has recovered |

Failure threshold, failure window, recovery time, and half-open trial count are configurable (defaults: 5 failures / 60s window / 30s recovery / 3 half-open trials), and a breaker can be manually reset from the UI.

### Sticky sessions

Optional session affinity pins a caller to an upstream for a TTL (default 3600s). Affinity can be keyed by `user`, `app`, `session`, or a `custom_header`.

### Token- and cost-aware routing

Optional advanced rules can factor request context size (token-aware) and cost-vs-latency optimization (cost-aware) into upstream selection.

### Audit logging

An independent audit option logs routing decisions, health check results, circuit-breaker events, and failovers to stdout, file, or syslog (with URL redaction available).

## Setting it up

1. Open **LLM Load Balancer → Settings** for global options: enable/disable, default strategy, health checks, circuit breakers, sticky sessions.
2. Open **Upstream Pools → Create Pool**:
   - Give the pool an ID (e.g. `openai-cluster`) and display name.
   - **Select the LLMs** the pool should serve — on save from AI Studio, the plugin automatically attaches itself to those LLMs' plugin chains, so no per-LLM wiring is needed. Wildcard slug patterns (e.g. `gpt-*`) are available under Advanced.
   - Choose the strategy and add upstreams (ID, name, backend URL, priority, weight, enabled flag). Optionally set a **failover pool** to receive traffic when this pool has no available upstream.
3. Save. Requests to the selected LLMs are now load-balanced.

> Removing an LLM from all pools does **not** automatically detach the plugin from that LLM — clean up the LLM's plugin chain manually if you decommission a pool.

## Monitoring

- **Dashboard** — request totals, error rate, latency percentiles (p50/p99/avg/max), failover count, and per-pool upstream cards with health indicators.
- **Health Status** — per-upstream health, circuit breaker states, last check results, and a per-breaker Reset button.
- **Metrics** — request counts and error rates per pool/upstream, plus latency distribution and percentiles.

RPC methods (including `getFullConfig`, `updateConfig`, `validateConfig`, `getHealth`, `getUpstreamStatus`, `getMetrics`, `getConfig`, `getLicenseStatus`, `resetCircuitBreaker`, `listLLMs`) expose the same data for automation.

Non-streaming responses also carry `X-LB-Upstream`, `X-LB-Pool`, and `X-LB-Latency-Ms` headers identifying the routing decision.

## License behavior

The plugin validates the enterprise license when its session starts; without a valid enterprise license it logs a fatal error and shuts itself down — no load balancing occurs and its admin pages become unavailable, while traffic to LLMs continues via their normally configured endpoints. There is no degraded/community fallback mode.

## Troubleshooting

- **Requests aren't being intercepted** — confirm the plugin appears in the LLM's plugin chain, the pool's LLM selection/slug patterns actually match, and the plugin is active in the Plugins list. The `X-LB-*` response headers show whether (and where) a request was routed.
- **Health checks failing** — verify the health path, network reachability from the gateway, and consider a longer timeout.
- **Circuit breaker stuck open** — check the upstream is actually recovered, review the failure threshold, or use **Reset Circuit Breaker** in the UI.
