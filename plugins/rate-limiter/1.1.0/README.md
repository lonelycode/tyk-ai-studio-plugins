# Rate Limiter Plugin

Sliding-window rate limiting with composable key dimensions for LLM requests.

## Features

- **Request Rate Limiting** - Limit requests per time window
- **Token Rate Limiting** - Limit token consumption per time window
- **Concurrent Request Limits** - Cap simultaneous in-flight requests
- **Composable Key Dimensions** - Rate limit by user, app, model, or any combination
- **Sliding Window** - Smooth rate limiting without burst spikes at window boundaries
- **Admin UI** - Manage rate limit rules from the Studio dashboard

## Installation

Install from the Tyk AI Studio marketplace or via OCI:

```
docker.tyk.io/studio-plugins/rate-limiter:1.0.0
```

## Configuration

Configure rate limit rules through the admin UI at **Rate Limiting > Rate Limit Rules**.

## Hooks

| Hook | Purpose |
|------|---------|
| `post_auth` | Evaluate rate limits before proxying to LLM |
| `on_response` | Track token usage from LLM responses |
| `studio_ui` | Admin dashboard for rule management |

## License

AGPL v3
