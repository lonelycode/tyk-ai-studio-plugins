# Changelog

## [1.1.1] - 2026-10-14

### Fixed
- Upstream selection no longer races under concurrent requests
- Upstream URLs must be absolute http(s) URLs; URLs are redacted in logs, health output and audit entries, and the audit file is created with mode 0600
- At start-up, stored entries that fail validation (an upstream without an ID or a valid URL, blank slug patterns, a pool left without upstreams, a failover to a missing pool) are dropped with a logged error instead of loaded unchecked. The configuration page shows the configuration as loaded, so saving it removes the dropped entries

### Changed
- Requires AI Studio 2.3.0

## [1.1.0] - 2026-09-02

### Fixed
- Repair the licensing vet failure and align the module Go version

### Changed
- Minimum AI Studio version is now 2.1
