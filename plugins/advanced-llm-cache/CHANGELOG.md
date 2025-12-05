# Changelog

## [1.0.1] - 2025-12-04

### Added
- Added advanced tuning fields to RedisConfig: ConnectTimeoutSeconds, AsyncUpdateTimeoutSeconds, ScanBatchSize, MaxEntrySizeBytes, LogAsyncErrors
- Added logAsyncErrors and scanBatchSize fields
- Implemented configurable connect timeout
- Configurable replicas

## [1.0.0] - 2025-12-04

### Added
- Initial release of Advanced LLM Cache (Enterprise)
- Redis backend support (single node and cluster mode)
- Failover mode with stale response serving
- Hierarchical TTL policies (token cost, endpoint, user tier)
- Fine-grained bypass rules (model family, user tier, regulatory class, load-based)
- Cost-based caching rules
- Audit logging (stdout, file, syslog) with content redaction
- Advanced observability (latency histograms p50/p90/p99, per-namespace stats)
- Cache sharding with consistent hashing
- Enterprise dashboard with real-time metrics
- License validation with graceful degradation to community features
