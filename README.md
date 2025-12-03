# Tyk AI Studio Plugin Marketplace (Community Edition)

This repository contains the community plugin marketplace catalog for Tyk AI Studio.

## Repository Structure

```
tyk-ai-studio-plugins-ce/
├── index.yaml                    # Auto-generated plugin index (DO NOT EDIT)
├── plugins/
│   ├── llm-firewall/
│   │   ├── manifest.yaml        # Plugin metadata (required)
│   │   ├── README.md            # Plugin documentation
│   │   ├── CHANGELOG.md         # Version history
│   │   └── icon.svg             # Plugin icon (SVG recommended)
│   └── <other-plugins>/
└── .github/
    └── workflows/
        └── generate-index.yml   # Auto-generates index.yaml on push
```

## How It Works

1. **Plugin Submission**: Contributors add a new directory under `plugins/` with a `manifest.yaml` file
2. **CI/CD**: GitHub Actions automatically generates `index.yaml` from all plugin manifests when changes are pushed to `main`
3. **Distribution**: Plugin binaries are hosted on OCI registries (GitHub Container Registry, Nexus, etc.)
4. **Discovery**: AI Studio periodically fetches `index.yaml` to update the marketplace catalog

## Plugin Manifest Format

Each plugin **must** have a `manifest.yaml` file. Below is the complete schema with all supported fields:

```yaml
# ============================================
# IDENTITY & VERSIONING (Required)
# ============================================
id: "com.tyk.plugin-name"           # Unique identifier (reverse domain notation)
name: "Plugin Name"                  # Human-readable name
version: "1.0.0"                     # Semantic version
description: "Brief description"     # What the plugin does

# ============================================
# OCI DISTRIBUTION (Required)
# ============================================
oci:
  registry: "ghcr.io"                          # OCI registry hostname
  repository: "tyk-technologies/plugins/name"  # Repository path
  tag: "1.0.0"                                 # Image tag
  digest: "sha256:abc123..."                   # Image digest for verification
  platform:                                    # Supported platforms
    - "linux/amd64"
    - "linux/arm64"
    - "darwin/arm64"

# ============================================
# DISCOVERY & CLASSIFICATION (Required)
# ============================================
category: "security"                 # Plugin category (see below)
keywords:                            # Search keywords
  - "security"
  - "firewall"
maturity: "stable"                   # alpha, beta, stable

# ============================================
# DOCUMENTATION & SUPPORT (Required)
# ============================================
links:
  documentation: "https://github.com/org/repo/tree/main/plugins/name"
  repository: "https://github.com/org/repo"
  support: "https://community.tyk.io"
  issues: "https://github.com/org/repo/issues"
  homepage: "https://tyk.io/ai-studio"

# ============================================
# DISPLAY ASSETS (Required)
# ============================================
icon: "https://raw.githubusercontent.com/org/repo/main/plugins/name/icon.svg"
screenshots: []                      # Optional: Array of screenshot URLs

# ============================================
# MAINTAINERS (Required)
# ============================================
maintainers:
  - name: "Your Name"
    email: "you@example.com"
    organization: "Your Organization"

publisher: "tyk-community"           # tyk-official, tyk-verified, tyk-community
license: "Apache-2.0"                # SPDX license identifier

# ============================================
# CAPABILITIES (Required)
# ============================================
capabilities:
  hooks:                             # All hooks this plugin implements
    - "pre_auth"
    - "post_auth"
  primary_hook: "pre_auth"           # Main hook type for categorization

# ============================================
# REQUIREMENTS (Required)
# ============================================
requirements:
  min_studio_version: "2.0"          # Minimum AI Studio version
  api_versions:                      # Supported API versions
    - "gateway-v1"
  dependencies: []                   # Other plugins this depends on

# ============================================
# PERMISSIONS (Required)
# ============================================
permissions:
  services: []                       # Service access: llms.proxy, llms.read, tools.call, etc.
  kv: []                             # Key-value store access
  rpc: []                            # RPC service access
  ui: []                             # UI extension permissions

# ============================================
# ENTERPRISE (Required)
# ============================================
enterprise_only: false               # true if requires enterprise license

# ============================================
# OPTIONAL FIELDS
# ============================================
config_schema_url: ""                # URL to JSON schema for plugin config

# Attestation/Signing (optional)
attestation:
  enabled: false
  sigstore_bundle_url: ""

# ============================================
# METADATA (Required)
# ============================================
created_at: "2025-01-15T10:00:00Z"   # ISO 8601 timestamp
updated_at: "2025-01-15T10:00:00Z"   # ISO 8601 timestamp
deprecated: false                    # Set to true to mark as deprecated
deprecated_message: ""               # Message explaining deprecation
replacement: ""                      # Plugin ID of replacement (if deprecated)
```

## Required vs Optional Fields

### Required Fields
All fields marked with `(Required)` sections above must be present for the plugin to be indexed correctly.

### Key Required Fields Summary
| Field | Description |
|-------|-------------|
| `id` | Unique plugin identifier (e.g., `com.tyk.llm-firewall`) |
| `name` | Human-readable name |
| `version` | Semantic version string |
| `description` | Brief description |
| `oci.*` | OCI registry distribution details |
| `category` | Plugin category |
| `maturity` | Release maturity level |
| `publisher` | Publisher type |
| `license` | SPDX license identifier |
| `capabilities.hooks` | List of implemented hooks |
| `capabilities.primary_hook` | Main hook type |
| `requirements.min_studio_version` | Minimum compatible version |
| `permissions.*` | Required permissions (can be empty arrays) |
| `enterprise_only` | Enterprise-only flag |
| `created_at` / `updated_at` | Timestamps |

## Contributing a Plugin

1. Fork this repository
2. Create a new directory under `plugins/` with your plugin name (e.g., `plugins/my-plugin/`)
3. Add your `manifest.yaml` file with all required fields
4. Add a `README.md` with plugin documentation
5. Add an `icon.svg` (SVG recommended, or PNG at 256x256)
6. Create a pull request

### Checklist Before Submitting

- [ ] `manifest.yaml` includes all required fields
- [ ] `id` follows reverse domain notation (e.g., `com.yourorg.plugin-name`)
- [ ] OCI image is published and accessible
- [ ] `icon` URL points to a valid image
- [ ] All `links` URLs are valid and accessible
- [ ] `enterprise_only` is set correctly
- [ ] `created_at` and `updated_at` are valid ISO 8601 timestamps

## Plugin Categories

| Category | Description |
|----------|-------------|
| `security` | Security plugins (firewalls, content filtering, policy enforcement) |
| `agents` | AI agent plugins that handle conversations |
| `connectors` | Integration plugins for external services |
| `tools` | Tool plugins that extend AI capabilities |
| `ui-extensions` | UI plugins that add custom interfaces |
| `analytics` | Analytics and monitoring plugins |
| `data` | Data processing and transformation plugins |

## Hook Types

Plugins can implement one or more hooks:

| Hook | Description |
|------|-------------|
| `pre_auth` | Runs before authentication, can modify/reject requests |
| `post_auth` | Runs after authentication with user context |
| `pre_request` | Runs before sending request to LLM |
| `post_request` | Runs after receiving response from LLM |
| `agent` | Full agent implementation |

## Publisher Types

| Publisher | Description |
|-----------|-------------|
| `tyk-official` | Plugins maintained by Tyk Technologies |
| `tyk-verified` | Community plugins verified by Tyk |
| `tyk-community` | Community-contributed plugins |

## Permission Scopes

Available service permissions:

| Scope | Description |
|-------|-------------|
| `llms.proxy` | Access to call LLM services via the proxy |
| `llms.read` | Read LLM configurations |
| `llms.write` | Modify LLM configurations |
| `tools.call` | Execute tool operations |
| `tools.read` | Read tool configurations |
| `datasources.query` | Query datasources |
| `datasources.read` | Read datasource configurations |
| `kv.readwrite` | Read and write plugin key-value storage |

## Index Generation

The `index.yaml` file is automatically generated by GitHub Actions when changes are pushed to `main`. **Do not edit `index.yaml` directly** - your changes will be overwritten.

The index includes all fields from the manifest plus:
- Auto-generated `manifest_url` pointing to the raw manifest file

## Support

For questions or issues:
- [Tyk Community Forums](https://community.tyk.io)
- [GitHub Issues](https://github.com/TykTechnologies/tyk-ai-studio-plugins-ce/issues)
