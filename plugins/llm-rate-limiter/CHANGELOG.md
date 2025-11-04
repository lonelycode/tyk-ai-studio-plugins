# Changelog

All notable changes to the LLM Rate Limiter plugin will be documented in this file.

## [1.2.0] - 2025-11-04

### Added
- Full hybrid multi-phase plugin implementation
- Policy-based rate limiting with TPM, RPM, and concurrent request controls
- Management UI with Web Components
- Policy Manager interface for creating and editing policies
- App Assigner interface for assigning policies to apps
- Real-time usage tracking and analytics
- Sliding window rate limit counters
- KV storage for policies and state

### Features
- Multiple hook types in single plugin (post_auth, on_response, studio_ui)
- Web Component-based UI integration
- RPC endpoints for UI-backend communication
- Per-app rate limit enforcement
- Token-based and request-based limiting
- Concurrent request throttling

## [1.0.0] - 2025-10-31

### Added
- Initial release of LLM Rate Limiter plugin
- Basic rate limiting functionality
- UI integration via sidebar
