# Tyk AI Studio Plugin Marketplace

This repository contains the plugin marketplace catalog for Tyk AI Studio.

## Repository Structure

```
tyk-ai-studio-plugins/
├── index.yaml                    # Auto-generated plugin index
├── plugins/
│   ├── echo-agent/
│   │   ├── manifest.yaml        # Plugin metadata
│   │   ├── README.md            # Plugin documentation
│   │   ├── CHANGELOG.md         # Version history
│   │   └── icon.png             # Plugin icon
│   └── <other-plugins>/
└── .github/
    └── workflows/
        └── generate-index.yml   # Auto-generates index.yaml
```

## How It Works

1. **Plugin Submission**: Contributors add a new directory under `plugins/` with a `manifest.yaml` file
2. **CI/CD**: GitHub Actions automatically generates `index.yaml` from all plugin manifests
3. **Distribution**: Actual plugin binaries are hosted on OCI registries (GitHub Container Registry, Nexus, etc.)
4. **Discovery**: AI Studio periodically fetches `index.yaml` to update the marketplace catalog

## Plugin Manifest Format

Each plugin must have a `manifest.yaml` file with the following structure:

```yaml
# Identity & Versioning
id: "com.tyk.plugin-name"
name: "Plugin Name"
version: "1.0.0"
description: "Brief description of what the plugin does"

# OCI Distribution
oci:
  registry: "ghcr.io"
  repository: "tyk-technologies/plugins/plugin-name"
  tag: "1.0.0"
  digest: "sha256:..."
  platform: ["linux/amd64", "linux/arm64", "darwin/amd64"]

# Discovery & Classification
category: "agents"  # agents, connectors, tools, ui-extensions
keywords: ["keyword1", "keyword2"]
maturity: "stable"  # alpha, beta, stable

# Documentation & Support
links:
  documentation: "https://docs.tyk.io/plugins/plugin-name"
  repository: "https://github.com/org/plugin-repo"
  support: "https://community.tyk.io"
  issues: "https://github.com/org/plugin-repo/issues"
  homepage: "https://tyk.io/plugins/plugin-name"

# Display Assets
icon: "https://raw.githubusercontent.com/tyk/plugins-repo/main/plugins/plugin-name/icon.png"
screenshots:
  - "https://raw.githubusercontent.com/tyk/plugins-repo/main/plugins/plugin-name/screen1.png"

# Maintainers
maintainers:
  - name: "Your Name"
    email: "you@example.com"
    organization: "Your Org"

publisher: "tyk-official"  # tyk-official, tyk-verified, community
license: "Apache-2.0"

# Capabilities
capabilities:
  hooks: ["agent"]
  primary_hook: "agent"

# Requirements
requirements:
  min_studio_version: "0.1.0"
  api_versions: ["agent-v1"]
  dependencies: []

# Permissions
permissions:
  services: ["llms.proxy"]
  kv: []
  rpc: []
  ui: []

# Optional: Config Schema URL
config_schema_url: "https://raw.githubusercontent.com/tyk/plugins-repo/main/plugins/plugin-name/config.schema.json"

# Metadata
created_at: "2025-01-15T10:00:00Z"
updated_at: "2025-10-29T10:00:00Z"
deprecated: false
```

## Contributing a Plugin

1. Fork this repository
2. Create a new directory under `plugins/` with your plugin name
3. Add your `manifest.yaml` file
4. Add a `README.md` with plugin documentation
5. Add an `icon.png` (recommended size: 256x256)
6. Create a pull request

## Plugin Categories

- **agents**: AI agent plugins that handle conversations
- **connectors**: Integration plugins for external services
- **tools**: Tool plugins that extend AI capabilities
- **ui-extensions**: UI plugins that add custom interfaces

## Publisher Types

- **tyk-official**: Plugins maintained by Tyk Technologies
- **tyk-verified**: Community plugins verified by Tyk
- **community**: Community-contributed plugins

## Support

For questions or issues, visit the [Tyk Community Forums](https://community.tyk.io).
