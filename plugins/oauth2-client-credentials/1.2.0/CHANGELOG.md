# Changelog

## [1.2.0] - 2026-10-02

### Fixed
- Revocation: unbinding, rebinding or deactivating an App now stops its client within seconds. Studio computes the client-to-App binding table and pushes it to the gateways on every change (with a 60-second heartbeat for gateways that missed one); gateways authenticate from it alone, instead of a KV cache whose entries were never removed.
- A provider that fails validation (a plain-HTTP issuer or JWKS URI, a missing field) is refused when saved and never activated, instead of being logged as a warning and run anyway.
- Binding a client that another App already holds is refused at save; a conflict saved some other way authenticates nobody and shows as Conflict in the App list.
- Code and manifest versions agree.

### Added
- The auth response carries who the token is for (the subject, `subject_claim` per provider, default `sub`) and the acting agent (`act.sub`, else `azp`), which AI Studio 2.2.1 records with each request for audit.
- Works on datasource, tool (REST and MCP), router and custom-endpoint routes as well as LLMs, through AI Studio 2.2.1's per-endpoint auth plugin lists.

### Changed
- Minimum AI Studio version is now 2.2.1.
- The raw plugin configuration is no longer logged at start-up.

## [1.1.0] - 2026-09-02

### Fixed
- Repair the licensing vet failure and align the module Go version

### Changed
- Minimum AI Studio version is now 2.1
