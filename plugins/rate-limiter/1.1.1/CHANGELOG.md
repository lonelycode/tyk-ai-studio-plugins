# Changelog

## [1.1.1] - 2026-10-14

### Added
- `redis_password`, masked in the configuration form. It overrides a password in `redis_url` and accepts `$SECRET/NAME` (resolved by AI Studio) or `$ENV/NAME` references. Keep credentials out of `redis_url`, which the form shows in clear

### Fixed
- A malformed `redis_url` was logged with its password
- The Redis connection is closed at shutdown
- A corrupt stored counter is treated as a storage failure (`fail_open` decides), never as zero
- Plugin storage read errors are reported instead of being treated as an empty rule set
- Concurrent rule saves are serialised, so one cannot silently overwrite another

### Changed
- `getRuleStats` no longer reports a usage figure (usage is counted per dimension key, not per rule)
- Requires AI Studio 2.3.0

## [1.1.0] - 2026-09-02

### Fixed
- Align the plugin module with the parent module's Go version

### Changed
- Minimum AI Studio version is now 2.1

## 1.0.0 (2026-04-02)

### Added
- Initial release
- Sliding-window rate limiting for request rate, token rate, and concurrent requests
- Composable key dimensions (user, app, model)
- Admin UI for rule management
- Gateway support for distributed rate limiting
