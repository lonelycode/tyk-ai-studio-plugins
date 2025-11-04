# Custom Authentication with Management UI

A hybrid plugin that provides both custom token-based authentication and a live web-based management interface.

## Description

The Custom Auth UI plugin demonstrates the power of AI Studio's hybrid plugin architecture by combining:
- **Authentication enforcement** via the `auth` hook
- **Management UI** via the `studio_ui` hook

This allows administrators to manage authentication tokens through a user-friendly interface while the same plugin enforces those tokens at runtime.

## Features

### Authentication
- Custom token-based authentication
- Map tokens to specific App IDs and User IDs
- Bearer token support (automatically strips "Bearer " prefix)
- Configurable behavior for unknown tokens
- Thread-safe token operations

### Management UI
- Live token management interface in AI Studio sidebar
- **Add tokens**: Create new authentication tokens with app/user mappings
- **Edit tokens**: Update App ID, User ID, and descriptions
- **Delete tokens**: Remove tokens with confirmation
- **Token masking**: Shows only first 4 and last 4 characters for security
- **Real-time stats**: View total tokens and configuration status
- **Modern UI**: Responsive design with Material-style components

## Configuration

The plugin accepts token configuration in JSON format:

```json
{
  "tokens": [
    {
      "id": "token-1",
      "token": "my-secret-token-123",
      "app_id": 1,
      "user_id": "user123",
      "description": "Production API token"
    }
  ],
  "reject_unknown_tokens": true,
  "default_app_id": 1
}
```

### Configuration Options

- **tokens** (array): List of token configurations
  - **id**: Unique identifier for the token
  - **token**: The actual authentication token
  - **app_id**: App ID to associate with this token
  - **user_id**: User ID to associate with this token
  - **description**: Human-readable description

- **reject_unknown_tokens** (boolean): If true, reject tokens not in the list; if false, accept with default_app_id
- **default_app_id** (integer): App ID to use for unknown tokens when reject_unknown_tokens is false

## Permissions

This plugin requires:

- **apps.read**: Read app configurations for validation
- **kv.readwrite**: Store token configurations in plugin storage
- **sidebar.register**: Add navigation item to sidebar
- **route.register**: Register management UI route

## Installation

### From Marketplace

1. Navigate to **Plugins → Marketplace** in AI Studio
2. Search for "Custom Authentication"
3. Click **Install**
4. Review and approve permissions
5. Configure initial tokens (optional)
6. Complete the installation

### Manual Installation (OCI)

```bash
oci://localhost:5101/plugins/custom-auth-ui@sha256:724cfadd3ef45afbbb525a7eeb268d098803c83d24871a4ac371fb5cdf2496cb
```

## Usage

### Managing Tokens via UI

1. After installation, navigate to **Custom Auth** in the sidebar
2. Click **Token Management**
3. The management interface shows:
   - List of all configured tokens (masked)
   - Add new token button
   - Edit/Delete actions for each token

#### Adding a Token
1. Click **Add Token**
2. Enter:
   - Token value
   - App ID
   - User ID
   - Description (optional)
3. Click **Save**
4. Token is immediately active for authentication

#### Editing a Token
1. Click the **Edit** button next to a token
2. Modify App ID, User ID, or description
3. Click **Save**
4. Changes apply immediately

#### Deleting a Token
1. Click the **Delete** button
2. Confirm the deletion
3. Token is immediately revoked

### Using Tokens for Authentication

Once configured, clients use tokens in the Authorization header:

```bash
curl -X POST https://your-studio.com/llm/rest/my-llm/chat/completions \
  -H "Authorization: Bearer my-secret-token-123" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4", "messages": [{"role": "user", "content": "Hello"}]}'
```

The plugin:
1. Extracts the token from the Authorization header
2. Looks up the token in its configuration
3. Sets the App ID and User ID for the request
4. Allows or denies based on configuration

## How It Works

### Authentication Flow
1. Client sends request with `Authorization: Bearer <token>` header
2. Plugin's `auth` hook intercepts the request
3. Token is validated against stored configuration
4. If valid: Request enriched with App ID and User ID
5. If invalid: Request rejected (if `reject_unknown_tokens` is true)

### UI Management Flow
1. Admin navigates to Custom Auth management UI
2. UI loads via Web Component
3. UI calls plugin RPC methods to:
   - List tokens
   - Add new tokens
   - Update tokens
   - Delete tokens
4. Plugin stores changes in KV storage
5. Changes are immediately reflected in auth hook

## Architecture

This is a **hybrid plugin** demonstrating:
- Multiple hook types in single binary (`auth` + `studio_ui`)
- RPC communication for UI-backend interaction
- KV storage for persistent configuration
- Web Component-based UI integration
- Embedded assets (UI code + icons)

## Requirements

- AI Studio version: 2.6.0 or higher
- OCI plugin support enabled
- At least one app configured

## Source Code

Available in the main Tyk AI Studio repository:
https://github.com/TykTechnologies/midsommar/tree/main/examples/plugins/studio/custom-auth-ui

## License

Apache 2.0

## Support

- Community Forum: https://community.tyk.io
- Issues: https://github.com/TykTechnologies/midsommar/issues
