# Changelog

## [1.2.0] - 2026-10-02

The catalog becomes an internal governance record first and a portal catalogue
second: what each agent, application and integration is made of, how risky it
is, who answers for it, and the history behind it.

### Added
- **Publication.** Assets are internal until a publisher publishes them to
  everyone or to chosen teams (core team grants, so the Teams page and the
  catalog agree). Publishing needs an approved or production asset, an
  accountable owner and a risk tier within the publishing ceiling (overridable
  with a note). Leaving the usable stages withdraws an asset automatically.
- **Typed dependencies.** Components can be catalog assets (with optional
  version pins) or AI Studio LLMs, tools, data sources, MCP servers, model
  routers, semantic routers and Apps. Apps and routers expand into what they
  bind or route to.
- **Admin asset workspace** (Assets > open an asset): overview, dependency
  graph and table with drift markers, risk, a merged activity timeline
  (catalog changes, governed-metadata history and the AI Studio audit trail of
  each component), approval snapshots compared with now, and export.
- **Risk rating.** Each asset's tier is the highest tier among the components
  it depends on or uses, raised one step each for autonomous operation,
  irreversible tool actions and external exposure. Unrated components mark the
  rating incomplete instead of counting as low. Publishers can accept a lower
  tier with a justification; the computed and accepted tiers are both kept.
- **Composer.** One editor to build a composite asset (agent, business
  application, integration or any type) from components, with a live graph and
  risk preview. Portal users can compose drafts from published assets they can
  see, and submit them for review.
- **Re-approval rule.** A change to an approved or production asset's
  components by someone without the publish permission creates a version,
  sends the asset back to review and withdraws it from the portal. A
  publisher's change keeps the stage and is logged.
- **Approval snapshots.** Each approval stores the resolved dependency graph
  and risk, for drift detection and "export as approved".
- **Export** as CycloneDX 1.6 JSON (AI/ML bill of materials) or catalog JSON.
  Gated field values and secrets are never exported.
- **App governance chain.** Agents and business applications can be linked to
  the AI Studio App they run as. The App's bindings become managed components,
  resynced whenever the App changes (a change in reach sends the asset back to
  review). Deprecating the asset flags the App; the new terminal **retired**
  stage deactivates it.
- **Periodic review.** Approval schedules the next review by accepted tier
  (`review_model` in the configuration). A daily check notifies owners before
  and after the due date and suspends the linked App once the grace period
  ends; recording a review reactivates it.
- **Owner departure.** When an accountable owner is disabled or deleted, their
  assets pass to the configured `fallback_owner` (or are flagged), the linked
  App is flagged `ownerless`, and admins are notified until someone takes
  ownership.
- New seeded types: Business Application and Integration.

### Changed
- Existing installs must approve new scopes: `resource-access.manage`,
  `llms.read`, `tools.read`, `datasources.read`, `apps.read`,
  `mcp-servers.read`, `routers.read`, `audit.read`, `apps.lifecycle` and
  `scheduler.manage`. Until then the features that need them are hidden or
  report that they are unavailable.
- Upgrading keeps everything visible that was visible before: assets in a
  visible stage are marked published to everyone, then the asset types switch
  to explicit team access.
- Submissions are still approved on creation, but start internal.
- Needs AI Studio 2.2.1 or later (the governance RPCs and the
  `apps.lifecycle` scope); against an older Studio the new features degrade
  gracefully.

## [1.1.3] - 2026-09-22

### Fixed
- The schema builder, the asset metadata form, the read-only metadata view and the access-request form keep properties in the order the administrator arranged them instead of alphabetically. The builder writes a top-level `x-property-order` array on the type schema and the access-request schema; every consumer honours it (listed names first, then any others alphabetically) and the plugin validates the hint as an array of property names. Existing schemas without the hint keep rendering alphabetically until they are saved again from the builder.
- Access-request notifications now carry a link, so clicking the bell entry opens the request in the admin review page (or the asset page in the portal for the requester). Needs a Studio build that accepts notification links from plugins (2.2.0); on older builds the notification is delivered without a link, as before.

### Changed
- README: assets are shown in the Teams plugin-resource sections and the portal catalog, but are not offered in App forms and are not shipped to gateways; access to gated fields comes only from the catalog's own access requests. The previous wording claimed App attachment and gateway snapshot delivery.

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
