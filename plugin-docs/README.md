# Tyk AI Studio Plugin Documentation

End-user documentation for the plugins in this repository: what each plugin does, when to use it, how to configure it, and what its behavior looks like from the outside. Written for platform administrators and API consumers — not plugin developers. (For building your own plugins, see the Plugin SDK documentation on the Tyk AI Studio docs site.)

## Community plugins

| Plugin | What it does |
|--------|--------------|
| [LLM Response Cache](llm-cache.md) | Serves repeated LLM requests from an in-memory cache — cuts cost and latency, with streaming replay, per-App isolation, and a live dashboard. |
| [Rate Limiter](rate-limiter.md) | Sliding-window request, token, and concurrency limits scoped by app, user, model, or API key, with shadow mode and standard `X-RateLimit-*` headers. |
| [LLM Firewall](llm-firewall.md) | Blocks prompts containing disallowed phrases or regex patterns (prompt-injection strings, banned terms) before they reach the provider. |
| [LLM Price Sync](llm-price-sync.md) | Keeps AI Studio's model price table current by syncing from llm-prices.com on a schedule, so cost tracking and budgets stay accurate. |
| [Git Connector](github-rag-ingest.md) | Ingests Git repositories into RAG datasources — cloned, filtered, chunked, embedded, and kept in sync incrementally on a schedule. |

## Enterprise plugins

These require a valid Tyk AI Studio enterprise license.

| Plugin | What it does |
|--------|--------------|
| [Advanced LLM Cache](advanced-llm-cache.md) | The enterprise edition of the response cache: Redis-backed and shared across gateways, with stale-serving failover, policy-driven TTLs, compliance bypass rules, audit logging, and sharding. |
| [LLM Load Balancer](llm-load-balancer.md) | Routes LLM traffic across pools of backend endpoints with health checks, circuit breakers, weighted/latency/least-connections strategies, and sticky sessions. |
| [OAuth2 Client Credentials Auth](oauth2-client-credentials.md) | Authenticates machine clients with JWTs from your own IdP (Entra ID, Auth0, Okta, …), mapping tokens to AI Studio Apps — with optional template-based auto-provisioning. |

## Where plugins run

- **AI Studio (hub)** — plugins with management UIs, schedulers, or Studio-side services (Price Sync and Git Connector run only here).
- **Microgateway (edge)** — request-path plugins (cache, rate limiter, firewall, load balancer, OAuth2 auth) run wherever traffic flows: at edge gateways and/or AI Studio's embedded gateway. Edge instances report statistics back to the hub during heartbeats.

Each document's header table states where that plugin runs, the hooks it uses, its admin UI location, and minimum platform versions.
