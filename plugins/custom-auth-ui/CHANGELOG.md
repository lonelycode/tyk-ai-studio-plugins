# Changelog

All notable changes to the Custom Authentication with Management UI plugin will be documented in this file.

## [1.0.0] - 2025-11-04

### Added
- Initial release of Custom Auth UI plugin
- Token-based authentication with Bearer token support
- Live token management UI accessible from Admin sidebar
- CRUD operations for tokens (Create, Read, Update, Delete)
- Token masking for security in UI
- App ID and User ID mapping
- Configurable behavior for unknown tokens
- Thread-safe token storage
- RPC-based UI-backend communication
- Web Component-based UI integration
- KV storage for persistent token configuration

### Features
- Hybrid plugin with auth + studio_ui hooks
- Real-time token statistics
- Token descriptions for better management
- Automatic Bearer prefix stripping
- Responsive Material-style UI
- Embedded assets using Go embed
