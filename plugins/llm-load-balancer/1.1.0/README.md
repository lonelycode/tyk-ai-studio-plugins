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
   - Backend URL
   - Priority (lower = higher priority)
   - Weight (for weighted strategies)
7. Click **Save**

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

### Circuit Breaker States

| State | Description |
|-------|-------------|
| Closed | Normal operation, requests flow through |
| Open | Too many failures, requests are rejected |
| Half-Open | Testing if upstream has recovered |

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

- Tyk AI Studio 2.0.0 or later
- Enterprise license

## Troubleshooting

### Plugin not intercepting requests

1. Verify the plugin is attached to the LLM (check LLM settings)
2. Verify the pool's slug patterns match the LLM name
3. Check that the plugin is active in the Plugins list

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
