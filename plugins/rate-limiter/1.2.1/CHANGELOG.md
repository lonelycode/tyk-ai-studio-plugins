# Changelog

## [1.2.1] - 2026-10-29

Requires AI Studio 2.3.0.

### Fixed
- **Saving a rule keeps the plugin's settings.** A rule change wrote the
  rules into the stored configuration together with the settings the Studio
  process had started with, so settings saved since in the configuration
  form (Redis backend, URL and password, `fail_open`, window size) were
  reverted without a warning, and edges went back to per-instance counters
  after the next push. A rule change now replaces only `rules` in the stored
  configuration; if the stored configuration cannot be read, the rules are
  not synced (the log says so) rather than written over it.

## [1.2.0] - 2026-10-23

Requires AI Studio 2.3.0. Built on the 2.3 Plugin SDK (64 MiB plugin message
limit).

### Fixed
- **Token rules count Bedrock.** Usage is read with the Plugin SDK's
  `ResponseUsage`, so Bedrock Converse's camelCase usage (`inputTokens`,
  `outputTokens`) and Google's `usageMetadata` are counted. Before, token
  rules never counted Bedrock or Google responses.
- **Streams are counted and released.** When a stream ends, its tokens are
  added to the request's token rules and its concurrent slots are released,
  also when it ends with an upstream error. Before, streams added no tokens,
  and a stream held its concurrent slot until the counter expired five minutes
  later, so concurrent rules could refuse requests that were within the limit.
- **Concurrent slots no longer leak.** A request answered before the
  upstream (a later plugin's block such as a cache hit, a request filter or
  guardrail block) gets no response phase, so its slot was never given back,
  and under steady traffic the counter never expired: the limit crept down
  until every request got a 429. Such slots are now given back once the
  request is ten minutes old, and the rate limiter's own 429 or 503 gives back
  the slots the request took under earlier rules at once.
- A `redis_password` that an edge gateway receives as
  `refused-credential-reference` (a `$SECRET/` reference AI Studio could not
  resolve) is never sent to Redis as the password. The README now says that
  AI Studio 2.3 resolves `$SECRET/` references on edges too.

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
