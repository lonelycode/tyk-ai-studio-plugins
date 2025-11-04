# LLM Rate Limiter Plugin

A comprehensive hybrid plugin that provides policy-based rate limiting for LLM requests with a built-in management UI.

## Description

The LLM Rate Limiter is a full-stack plugin that combines:
- **Gateway hooks** for enforcing rate limits on LLM requests
- **Management UI** for creating and managing rate limit policies
- **App assignment** interface for applying policies to specific apps

This plugin demonstrates the power of AI Studio's hybrid plugin architecture, where a single plugin can provide both runtime enforcement and management capabilities.

## Features

### Rate Limiting Capabilities
- **TPM (Tokens Per Minute)**: Limit token consumption per minute
- **RPM (Requests Per Minute)**: Limit number of requests per minute
- **Concurrent Requests**: Control maximum concurrent requests
- **Policy-based**: Create reusable policies and assign to apps

### Management UI
- **Policy Manager**: Create, edit, and delete rate limit policies
- **App Assigner**: Assign policies to specific apps
- **Real-time Status**: View current usage and limits
- **Usage Analytics**: Track rate limit hits and policy effectiveness

### Multi-Phase Hook Integration
- **post_auth**: Checks rate limits before forwarding to LLM
- **on_response**: Updates token counters based on actual LLM usage
- **studio_ui**: Provides management interface in Admin UI

## Configuration

This plugin uses the UI for configuration management. No JSON configuration is required - all settings are managed through the web interface.

## Permissions

This plugin requires the following permissions:

- **apps.read**: Read app configurations to enforce per-app limits
- **apps.write**: Update app metadata with rate limit assignments
- **kv.readwrite**: Store rate limit state (counters, policies)
- **sidebar.register**: Add navigation items to Admin UI
- **route.register**: Register UI routes for policy management

## Installation

### From Marketplace

1. Navigate to **Marketplace** in AI Studio
2. Search for "LLM Rate Limiter"
3. Click **Install**
4. Review and approve permissions
5. Complete the installation

### Manual Installation (OCI)

```bash
oci://localhost:5101/plugins/llm-rate-limiter@sha256:9514697fe4463f20601ebc896c261eaae7af26f5a7845ec4b56138486db3d960
```

## Usage

### Creating a Rate Limit Policy

1. After installation, navigate to **Rate Limiting** in the sidebar
2. Click **Rate Limit Policies**
3. Click **Create Policy**
4. Configure:
   - Policy name (e.g., "Free Tier")
   - TPM limit (e.g., 10,000)
   - RPM limit (e.g., 20)
   - Concurrent requests (e.g., 5)
5. Save the policy

### Assigning Policies to Apps

1. Navigate to **Rate Limiting** → **App Assignments**
2. Select an app from the dropdown
3. Choose a rate limit policy
4. Click **Assign**
5. The policy is now enforced for that app

### Monitoring

The plugin tracks:
- Current token usage
- Request counts
- Rate limit violations
- Policy effectiveness

View these metrics in the Rate Limiting dashboard.

## How It Works

### Request Flow

1. **User makes LLM request** via app token
2. **post_auth hook**: Plugin checks if app has exceeded rate limits
   - If exceeded: Request is rejected with 429 status
   - If within limits: Request proceeds
3. **LLM processes request** and returns response
4. **on_response hook**: Plugin updates counters based on actual token usage
5. **Response returned** to user

### Storage

The plugin uses AI Studio's KV storage to maintain:
- **Policies**: Rate limit policy definitions
- **Assignments**: App-to-policy mappings
- **Counters**: Per-app token and request counts (sliding window)
- **State**: Last reset times, violation counts

## Requirements

- AI Studio version: 2.6.0 or higher
- OCI plugin support enabled
- At least one app configured

## Architecture

This is a **hybrid multi-phase plugin**:
- Implements multiple hook types in a single binary
- Provides both runtime enforcement (gateway hooks) and management UI
- Demonstrates best practices for complex plugin development

## Source Code

The source code is available in the main Tyk AI Studio repository:
https://github.com/TykTechnologies/midsommar/tree/main/examples/plugins/studio/llm-rate-limiter-multiphase

## License

Apache 2.0

## Support

- Community Forum: https://community.tyk.io
- Issues: https://github.com/TykTechnologies/midsommar/issues
