# Changelog

## [1.0.0] - 2025-12-04

### Added
- Initial release of LLM Response Cache plugin
- Deterministic SHA-256 cache keys based on model, messages, tools, and temperature
- Prompt normalization for improved cache hit rates
- Namespace isolation (API key, app ID, org ID)
- LRU eviction when cache is full
- Configurable TTL expiration
- Cache bypass via header or query parameter
- Admin dashboard with real-time metrics
- Response headers for cache status, age, and TTL
