# Changelog

## [1.1.2] - 2026-09-18

### Changed
- Every asset type now registers a portal detail path (`/portal/plugins/asset-catalog#/assets/{id}`), so AI Portal catalog cards and the built-in detail page link straight to the asset's own page in the plugin, where schema fields, gated values and the access-request form live. Previously a Prompt's built-in detail page said access is managed by the plugin and offered no route to the prompt body. Existing installs pick the path up on the next resource-type sync after upgrading. Deep links from catalog cards need a Studio build that includes the portal catalog deep-link change (2.2.0).

## [1.1.1] - 2026-09-15

### Removed
- The `require_license_feature` configuration option and the `feature_asset_catalog` entitlement claim. The plugin now requires a valid enterprise license unconditionally; any enterprise license is accepted and there is no switch to relax the check.

## [1.1.0] - 2026-09-14

### Changed
- Every permission row in the role editor is now honoured end to end. The catalog no longer keys on a single administrator flag: *Assets* read/write/delete/publish, the per-type *Assets: <type>* rows, *Asset types* write and *Access requests* read/write each unlock exactly what their label says, on both the admin and portal routes. Gated values still need ownership, a grant or the write row.
- Reviewers holding *Assets: publish* can release and send back assets they do not own.
- Asset methods, type listing and the new `admin_transition` / `admin_revoke_grant` aliases are gated on the plugin's base read at the platform and decided by the catalog, so roles scoped to one asset type work. Every catalog role starts with the base read.
- Admin pages hide controls the role cannot use and show explanatory empty states; the portal page relies on server-computed flags (`can_manage`, `can_delete`).

### Added
- `can_manage` and `can_delete` on `AssetView`.
- Service scope `rbac.register` (per-type permission rows registered at runtime; optional).

## [1.0.0] - 2026-09-10

### Added
- Initial release
