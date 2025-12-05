# Changelog

## [1.0.0] - 2025-12-04

### Added
- Initial release of GitHub RAG Ingestion plugin
- Multiple repository support with authentication (PAT, SSH)
- Incremental sync with git diff-based updates
- Chunking strategies: simple, code-aware, hybrid
- Scheduled ingestion via cron expressions
- Job tracking with detailed statistics and logs
- Secrets management (KV storage or HashiCorp Vault)
- Rich metadata for each chunk (repo info, file paths, line numbers, GitHub URLs)
- Web UI for repository and job management
- Datasource cloning feature
