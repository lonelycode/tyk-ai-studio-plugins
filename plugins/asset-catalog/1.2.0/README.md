# Asset Catalog (Enterprise)

Register, own, version, relate and govern the AI assets your organisation cares about but the gateway never proxies: **agents, prompts, skills, agent configurations, hooks, guardrails, business applications, integrations, external AI services**. The catalog is first an internal governance record: what each agent or application is made of (catalog assets and AI Studio LLMs, tools, data sources, MCP servers, routers and Apps), how risky it is, who answers for it, and the history behind it. Assets reach the AI Portal only when a publisher publishes them, to everyone or to chosen teams. Each asset type is a first-class AI Portal resource with ownership, lineage, lifecycle, community submissions and an access-request workflow that publishes events for downstream automation.

## What you get

| Capability | Where |
|---|---|
| **Asset types** defined at runtime with a JSON Schema for their metadata, gated fields, an approval flag, a lifecycle policy and allowed relationship kinds. Two bundled types: **Agent** and **Prompt** (plus two sample assets). | Admin → Asset Catalog → Asset Types |
| **Assets** with three ownership roles (created by / responsible / accountable), tags, validated metadata plus free-form extra metadata, a version history with change notes and lineage, typed relationships (`depends_on`, `uses`, `recommended_with`, `derived_from`, `supersedes`) with cycle detection, and lifecycle stages `draft → experimental → in_review → approved → production → deprecated`. | Portal → Asset Catalog; Admin → Asset Catalog → Assets |
| **Community submissions**: portal users propose new assets through the platform's Submission form (the type's schema drives the form); admins review them in the Submission Queue; approval creates the asset owned by the submitter. | Portal → Contribute |
| **Access requests**: types (or individual assets) can require approval. Users fill in a form; admins get an in-Studio notification; approval records a grant and unlocks the asset's gated fields for that user. | Portal asset page; Admin → Asset Catalog → Access Requests |
| **First-class resource types**: every asset type is registered with AI Studio, so assets appear in the Teams plugin-resource sections and in the portal's unified catalog. Assets are not offered in App forms and are not shipped to gateways in the config snapshot; access to gated fields comes only from the catalog's own access requests. | Teams, Portal catalog |
| **Governance Metadata**: assets of governed types carry the platform's admin-defined governance fields (owners, lifecycle state, risk tier, data classification, regulatory applicability, …) with the same schemas, vocabularies, enforcement level and compliance report as LLMs, Tools and Data Sources. Studio validates and stores them; an enforcing schema blocks the save. | Admin and portal asset forms; Governance → Metadata schemas / compliance |
| **Publication**: assets are internal until published to everyone or to chosen teams (core team grants: the Teams page and the catalog agree). Publishing needs an approved or production asset, an accountable owner and an accepted risk tier within `max_publish_tier` (a publisher can override with a note). | Admin asset page → Overview |
| **Dependency graph**: typed edges to catalog assets (optionally pinned to a version) and to AI Studio LLMs, tools, data sources, MCP servers, model and semantic routers and Apps; Apps and routers expand into what they bind or route to. Drift markers flag pinned targets with newer versions, inactive or missing components and changes since the last approval. | Admin asset page → Components |
| **Risk rating**: high-water mark over `depends_on` / `uses` edges, raised one step each for autonomous operation, irreversible tool actions and external exposure; unrated components mark the rating incomplete. Computed and accepted (overridden) tiers, with the paths that set them. | Admin asset page → Risk |
| **Activity timeline**: every catalog change (including tags, ownership, grants, publication and risk), merged with the governed-metadata history and the AI Studio audit trail of each component. | Admin asset page → Activity |
| **Approval snapshots**: each approval stores the resolved graph and risk; compare any snapshot with now. | Admin asset page → Approvals |
| **Composer**: one editor to build a composite asset from components with a live graph and risk preview. Portal users compose drafts from published assets they can see and submit them for review. Changing an approved asset's components without the publish permission sends it back to review and withdraws it from the portal. | Admin → Assets → New composite; portal → My assets |
| **Export**: CycloneDX 1.6 JSON (AI/ML bill of materials) or catalog JSON, current or as approved. Gated values and secrets are never exported. | Admin asset page → Export |
| **App governance chain**: link an Agent or Business Application to the AI Studio App it runs as. The App's bindings become managed components, resynced when the App changes; deprecating the asset flags the App, retiring it deactivates the App, a lapsed review suspends it, and a departed owner flags it until someone takes the asset over. | Admin asset page → Linked App |
| **Periodic review**: due dates by accepted tier, notices before and after, App suspension after the grace period. | Admin asset page → Periodic review |
| **Events** on the internal bus for every change, so a webhook, email or ticketing plugin can automate follow-on work. | see below |

## Requirements

- Tyk AI Studio ≥ 2.2.0 with a valid enterprise license. The plugin exits at start-up without one. The 1.2 governance features (team publication, core components, audit timeline, App lifecycle control) need Studio 2.2.1 or later; against an older Studio they degrade gracefully (published assets are visible to everyone, core components cannot be added).
- Service scopes:
  - `kv.readwrite`, `license.read`, `notifications.write`, `resource-types.manage`;
  - `metadata.read`, `metadata.write` (Governance Metadata; without them the plugin still works, governance is skipped);
  - `rbac.register` (one *Assets: <type>* permission row per asset type; without it only the static rows exist);
  - `resource-access.manage` (publish to teams through core grants);
  - `llms.read`, `tools.read`, `datasources.read`, `apps.read`, `mcp-servers.read`, `routers.read` (core components in the graph);
  - `audit.read` (component audit trail in the timeline);
  - `apps.lifecycle` (flag, suspend and reactivate linked Apps; it cannot change what an App may access);
  - `scheduler.manage` (the daily `review-check` schedule).

  Upgrading from 1.1 asks an administrator to approve the new scopes.
- The event bus is only available when Studio runs with `GATEWAY_MODE=control`; in standalone mode the plugin still works but events are logged as skipped.

## Configuration

| Key | Default | Description |
|---|---|---|
| `seed_examples` | `true` | Create the two sample assets on first start (bundled types are always created). |
| `default_requires_approval` | `false` | Approval requirement for types that do not declare one. |
| `notify_admins` | `true` | Raise an in-Studio notification for every new access request. |
| `event_topic_prefix` | `asset_catalog` | Events are published as `<prefix>.<kind>`. |
| `risk_model.tier_field` | `risk_tier` | Governed metadata field holding a component's tier; `tier_values` maps stored values to tiers. |
| `risk_model.classification_field`, `classification_order` | — | Governed metadata data classification; the highest across the graph is reported. |
| `risk_model.privacy_bands` | 25 low / 50 medium / 75 high / 100 critical | Tier of a component rated only by its privacy score. |
| `risk_model.max_publish_tier` | `high` | Publishing an asset with a higher accepted tier needs a publisher's override. |
| `review_model.intervals` | critical 90, high 180, medium/low/unrated 365 days | Next review after approval, by accepted tier. |
| `review_model.notice_days`, `grace_days` | 14, 14 | Notice before the due date; App suspension this long after it. |
| `review_model.due_field` | — | Governed metadata date field that overrides the computed due date. |
| `review_model.disabled` | `false` | Turn periodic review off. |
| `fallback_owner` | — | `{user_id, email, name}` taking over the assets of departed accountable owners. Without it they are only flagged. |

## Data model

- **Asset type**: `slug`, `name`, `description`, `icon`, `schema` (JSON Schema object; mark a property with `"x-gated": true` to hide it from users without access; a top-level `"x-property-order": [names]` array, written by the schema builder, fixes the order forms and views render properties in, listed names first and the rest alphabetically), `requires_approval`, `access_request_schema` (JSON Schema for the request form, same `x-property-order` hint; default is a single required `justification`), `lifecycle_policy { initial, admin_only_stages }`, `relationship_kinds`, `has_privacy_score`, `schema_version`.
- **Asset**: `id`, `type_slug`, `name`, `description`, `tags`, `owners { created_by, responsible, accountable }`, `lifecycle`, `lifecycle_history`, `metadata` (validated), `extra` (free-form), `current_version`, `lineage`, `relationships[] { kind, target_kind, target_asset_id | target_ref, target_version, note, managed }`, `requires_approval` (optional per-asset override), `grants[]`, `community_submitted`, `submission_id`, `privacy_score`, `publication { state: internal|published, audience: everyone|teams, by, at, note }` (the teams themselves are core grants), `risk_profile { declared_tier, autonomy, tool_action_class, exposure }`, `risk_override { tier, justification, by, at }`, `linked_app { app_id, app_name, linked_by, linked_at, synced_at, missing }`, `review { due_at, interval_days, last_reviewed_at, last_reviewed_by, lapsed, suspended_app }`, `owner_pending { since, departed_owner, held_by }`.
- **Activity log** and **approval snapshots**: append-only per asset in plugin KV (`asset:<id>:log:<n>`, `asset:<id>:snap:<n>`).
- **Version**: full snapshot of the asset at that version, `change_notes`, `changed_by`, `parent_version`. Content edits (name, description, metadata, lineage, relationships) create a version; tag, ownership and lifecycle changes are recorded in history only. An earlier version can be restored: this appends a new version with the old content (change notes `Restored from version N`) rather than rewinding history, and the restored content must still satisfy the type's current schema.
- **Access request**: `asset_id`, `requester`, `requester_groups`, `form`, `status` (`pending | approved | denied | cancelled`), `reviewer`, `decision_note`.

### Rules

- Portal users see an asset when it is active, in `approved`, `production` or `deprecated`, **published**, and their team holds a grant for it (published to everyone = the Default team). Owners always see their own assets, internal ones included; holders of *Assets: read* (general or per type) see everything. `retired` assets are visible in the admin pages only.
- Lifecycle: `draft → experimental → in_review → approved → production → deprecated → retired`. `retired` is terminal and read-only, and deactivates a linked App. Leaving the usable stages (except to `deprecated`) withdraws an asset from the portal.
- A change to an approved or production asset's components by someone without the publish permission (including a change in a linked App's bindings) creates a version, moves it to `in_review` and unpublishes it. Portal users may add only catalog assets they can see as components; existing core components are kept.
- Gated fields are visible to owners, asset managers (*Assets: write*) and users with a grant. If the asset (or its type) does not require approval, everyone sees everything. Reading the catalog alone never unlocks gated values.
- Portal users create assets only through the Submission form. Asset managers create directly. Owners and asset managers edit and version assets. Content edits require `change_notes`.
- `approved`, `production` and `deprecated` are reserved stages by default (per-type policy): moving an asset into one needs the *Assets: publish* (or per-type `assets-<type>:publish`) permission. Publish holders may also move any asset between the other stages (send it back for rework).
- `depends_on` relationships cannot form cycles. Deprecating or deleting an asset that others depend on requires `force` from an asset manager and emits `asset.dependency_warning`.
- Type schemas can be extended at any time. Removing a property that existing assets still use requires `force`. Existing assets are re-validated on their next edit only.

## RPC contract (UI ↔ plugin)

All UI calls go through one router. Portal pages call `portalPluginAPI.call(method, payload)`; admin pages call `pluginAPI.call(method, payload)`. Both carry the authenticated user.

**Permissions.** `admin_*` methods are gated by Studio's role-based access control. The manifest declares the plugin's permission resources (`rbac.resources`) and the permission each admin method needs (`rbac.rpc_methods`); Studio enforces them on `POST /plugins/:id/rpc/:method` before the call reaches the plugin, the router checks them again, and the catalog applies the finer rows on every call. In the role editor the plugin appears under **Plugins → Asset Catalog** with these rows:

| Row | Actions | Unlocks |
|---|---|---|
| *Asset Catalog* (base) | read, write, execute | **read** (every catalog role starts here): open the plugin's pages, list types, `admin_stats`, and call the asset methods, whose outcome is then decided by the rows below; without an *Assets* read row the Assets page lists only the publicly visible stages, like the portal. **write**: catalog administrator, i.e. every row below on every type, plus `admin_reseed_examples` and editing the plugin configuration. |
| *Asset types* | read, write, delete | write: define, edit, deactivate and reactivate types (`delete` is declared for future use). |
| *Assets* | read, write, delete, publish | **read**: see every asset of every type, including drafts and inactive ones (gated values still need access). **write**: create assets; edit, version, relate, restore, transfer ownership, revoke grants, set the privacy score and approval flag on any asset; `force` deprecations; see grants and gated values. **delete**: deactivate or hard-delete. **publish**: move any asset into an `approved`, `production` or `deprecated` stage, and between the other stages as a reviewer. |
| *Assets: <type>* (one row per active type, registered at runtime) | read, write, delete, publish | the same four, narrowed to one asset class; a role holding only `Assets: Agent: write` manages Agents and cannot see draft Prompts. |
| *Access requests* | read, write | read: list and inspect every request; write: approve, deny or cancel any pending request. |

Rules of thumb: every catalog role needs the base **read**; a narrower row never implies the base read, and Studio only lets a role open the plugin's pages and asset methods with it. The per-type rows need the `rbac.register` service scope (declared in the manifest, approved by an administrator); without it the static rows still apply. Full administrators and roles holding *Installed plugins: execute* hold every row. Owners keep their existing rights on their own assets. Portal (`portal-rpc`) calls are not governed by roles, but a full administrator or base-write holder browsing the portal keeps their rights there.

Example roles:

| Role | Rows | Can |
|---|---|---|
| Catalog viewer | base read | open the pages; nothing else (the Viewer system role also receives every read row automatically) |
| Reviewer | base read + *Assets: publish* (or *Assets: Prompt: publish*) | release any asset (of that type) into a reserved stage or send it back; cannot edit, create or delete |
| Access reviewer | base read + *Access requests: write* | see and decide every access request |
| Type designer | base read + *Asset types: write* | define and retire types |
| Agent editor | base read + *Assets: Agent: write* | create and edit any Agent asset, see its gated values and grants; nothing on other types |
| Catalog administrator | base write | everything, as before |

Every response has the shape:

```json
{ "ok": true, "data": { ... } }
{ "ok": false, "error": "human readable", "code": "not_found | forbidden | invalid | conflict | not_required | not_ready | internal" }
```

### User methods

| Method | Payload | Returns |
|---|---|---|
| `list_types` | `{}` | `{ types: Type[], lifecycle_stages: string[], relationship_kinds: string[] }` (active types; each has `gated_fields`, `properties`) |
| `get_type` | `{ slug }` | `Type` |
| `list_assets` | `{ type_slug?, q?, lifecycle?, tags?: string[], page?, page_size? }` | `{ items: AssetView[], total, page, page_size }` |
| `get_asset` | `{ id }` | `AssetView` (see below), including `governed`, `governance_display` (portal-visible governance fields) and, for editors, `governance` (every value) |
| `get_governance_schema` | `{ type_slug }` | `{ available, has_fields, schema: { fields, vocabularies, enforcement, json_schema, schema_slugs } }` — the resolved Governance Metadata schema, for the portal edit form |
| `list_versions` | `{ id }` | `{ versions: [{ version, parent_version, change_notes, changed_by, created_at }] }` newest first |
| `get_version` | `{ id, version }` | `{ version, parent_version, change_notes, changed_by, created_at, snapshot: Asset }` |
| `restore_version` | `{ id, version, change_notes? }` | `AssetView` (owner or asset manager). Creates a new version whose content equals `version`; history is never rewritten |
| `my_assets` | same as `list_assets` | assets the caller owns (any role), including drafts |
| `update_asset` | `{ id, name?, description?, tags?, metadata?, extra?, lineage?, governance?, change_notes }` | `AssetView` (content changes need `change_notes`; `governance` replaces the Governance Metadata values without creating a version; asset managers may also send `privacy_score`, `requires_approval`, `clear_requires_approval`) |
| `set_relationships` | `{ id, relationships: [{ kind, target_asset_id, note? }], change_notes? }` | `AssetView` (replaces all outbound edges) |
| `transfer_ownership` | `{ id, role: "responsible" \| "accountable", to: { user_id, email, name } }` | `AssetView` (accountable owner or asset manager) |
| `transition` | `{ id, stage, note?, force? }` | `AssetView` |
| `get_access_request_form` | `{ asset_id }` | `{ schema, requires_approval, has_access, pending_request_id }` |
| `request_access` | `{ asset_id, form }` | `AccessRequest` (`code: not_required` when the caller already has access) |
| `my_requests` | `{ status? }` | `{ requests: AccessRequest[] }` |
| `cancel_request` | `{ id }` | `AccessRequest` |
| `my_grants` | `{}` | `{ grants: [{ asset_id, asset_name, type_slug, granted_at, granted_by, request_id }] }` |
| `revoke_grant` | `{ asset_id, user_id }` | `AssetView` (accountable owner or asset manager) |
| `whoami` | `{}` | `{ user_id, email, name, is_admin, groups }` |
| `get_graph` | `{ id, direction?: down\|up\|both, depth? }` | `Graph { root, nodes[], edges[], hidden, truncated }`; components the caller may not see collapse into `hidden` |
| `get_risk` | `{ id }` | `RiskAssessment` (tiers only for non-owners) |
| `list_activity` | `{ id, before?, limit? }` | activity entries, newest first (owners and catalog readers) |
| `get_timeline` | `{ id, include_components?, sources?, limit? }` | merged catalog, governance and audit history |
| `list_snapshots` / `get_snapshot` | `{ id }` / `{ id, number }` | approval snapshots |
| `list_component_candidates` | `{ type_slug, kinds?, q?, limit? }` | palette items (portal: visible catalog assets only) |
| `compose_asset` | `{ id?, type_slug, name, description?, tags?, metadata?, risk_profile?, governance?, components: [{ kind, ref, relationship_kind?, pin_version?, note? }], change_notes?, submit? }` | `AssetView`; new assets start in the type's initial stage, `submit` moves them to review |
| `preview_risk` | same as `compose_asset` | `{ risk, graph }` without saving |
| `resync_app` | `{ id }` | `AssetView` |
| `take_ownership` | `{ id }` | `AssetView` (the fallback owner holding it or an asset manager) |

### Admin methods

| Method | Permission | Payload | Returns |
|---|---|---|---|
| `admin_list_types` | `read` (base) | `{}` | like `list_types` but includes inactive types |
| `admin_upsert_type` | `asset-types:write` | `{ slug, name, description?, icon?, schema?, requires_approval?, access_request_schema?, lifecycle_policy?, relationship_kinds?, has_privacy_score?, force? }` | `Type` |
| `admin_deactivate_type` / `admin_reactivate_type` | `asset-types:write` | `{ slug }` | `Type` |
| `admin_list_assets` | `read` (base); the catalog lists what `assets:read` / `assets-<type>:read` allow | like `list_assets` | all assets the caller may see, including inactive |
| `admin_create_asset` | `read` (base); the catalog needs `assets:write` or `assets-<type>:write` (+ `assets:publish` or `assets-<type>:publish` for a reserved `lifecycle`) | `{ type_slug, name, description?, tags?, metadata?, extra?, relationships?, lineage?, responsible?, accountable?, requires_approval?, privacy_score?, lifecycle?, governance?, change_notes? }` | `AssetView` (`governance` is validated and stored by Studio before the asset is persisted) |
| `admin_delete_asset` | `read` (base); the catalog needs `assets:delete` or `assets-<type>:delete` | `{ id, hard?, force? }` | `{ deleted, id, hard }` |
| `admin_<method>` aliases | `read` (base); the same catalog rules as the user method | as the user method | aliases of `transition`, `revoke_grant`, `get_asset`, `list_versions`, `get_version`, `update_asset`, `set_relationships`, `restore_version`, `transfer_ownership`, `get_graph`, `get_risk`, `list_activity`, `get_timeline`, `list_snapshots`, `get_snapshot`, `list_component_candidates`, `compose_asset`, `preview_risk`, `resync_app`, `take_ownership`, so roles narrower than base write pass Studio's gate on the admin pages |
| `admin_list_requests` | `access-requests:read` | `{ status? }` | `{ requests: AccessRequest[] }` |
| `admin_get_request` | `access-requests:read` | `{ id }` | `AccessRequest` |
| `admin_decide_request` | `access-requests:write` | `{ id, decision: "approved" \| "denied", note? }` | `AccessRequest` |
| `admin_get_publication` / `admin_set_publication` | `read` (base); setting needs `assets:publish` or the per-type row | `{ id }` / `{ id, state, audience, team_ids?, note?, force? }` | `{ publication, teams, blockers, risk, can_publish }` |
| `admin_set_risk_override` | `read` (base); raising needs write, lowering needs publish and a justification | `{ id, tier, justification? }` (`tier: ""` clears) | `RiskAssessment` |
| `admin_export_graph` | `read` (base); the catalog needs owner or *Assets: read* | `{ id, format: cyclonedx\|json, snapshot?, include_admin_governance? }` | `{ filename, content_type, document }` |
| `admin_link_app` / `admin_unlink_app` | `read` (base); the catalog needs `assets:write` | `{ id, app_id }` / `{ id }` | `AssetView` |
| `admin_record_review` | `read` (base); the catalog needs publish or write | `{ id, note, interval_days? }` | `AssetView` |
| `admin_run_review_check` | `write` (base) | `{}` | `{ checked, notified, overdue, lapsed, suspended, errors }` |
| `admin_reseed_examples` | `write` (base) | `{}` | `{ seeded: true }` |
| `admin_stats` | `read` (base) | `{}` | `{ types, assets, by_type, by_lifecycle, pending_requests, total_requests, grants }` |

Why base read: Studio gates each method on one fixed permission, while the per-type rows are created at runtime, so the asset methods are opened to every catalog role and the catalog decides per asset. Unlisted methods would need the base write, which is why the admin page calls the `admin_*` aliases. The user method `transition` into a reserved stage needs `assets:publish` or `assets-<type>:publish`; owners move assets through the other stages, and publish holders may move any asset of the type between stages.

### `AssetView`

The asset plus: `type_name`, `gated_fields` (names of gated properties), `has_access`, `is_owner`, `can_edit`, `can_manage` (holds the assets or per-type write row), `can_delete` (the delete row), `effective_requires_approval`, `pending_request_id`, `inbound[] { kind, from_asset_id, from_name, from_type }`, `related[] { kind, asset_id, name, type_slug, lifecycle, note }`, `allowed_transitions`. When `has_access` is false the gated keys are absent from `metadata` and `grants` is empty. Single-asset reads (`get_asset`) also carry `governed`, `governance_display[] { key, label, type, value }` (the portal-visible Governance Metadata, for everyone who can see the asset) and, for editors only, `governance` (every stored value).

## Events

Published on the internal bus with direction `local`. Topics are exact strings (the bus has no wildcards): subscribe to each topic or use `SubscribeAll` and filter on the payload's `event` field.

| Topic (`asset_catalog.` prefix) | When |
|---|---|
| `asset.created`, `asset.updated`, `asset.version_created`, `asset.lifecycle_changed`, `asset.relationships_changed`, `asset.ownership_changed`, `asset.deleted`, `asset.dependency_warning`, `asset.publication_changed` | asset mutations |
| `asset.app_linked`, `asset.review_required` (a linked App's reach changed or the App is gone), `asset.deprecated`, `asset.owner_departed` | App governance chain |
| `asset.review_due`, `asset.review_overdue`, `asset.review_lapsed`, `asset.reviewed` | periodic review |
| `access_request.created`, `access_request.approved`, `access_request.denied`, `access_request.cancelled` | access workflow |
| `type.created`, `type.updated`, `type.deactivated` | type management |

Payload:

```json
{
  "event": "asset_catalog.access_request.created",
  "occurred_at": "2026-09-10T08:00:00Z",
  "plugin_id": 42,
  "actor": { "user_id": 7, "email": "alice@acme.com", "name": "Alice", "is_admin": false },
  "asset": { "id": "ast_…", "type_slug": "agent", "name": "…", "description": "…", "lifecycle": "production",
             "version": 3, "owners": { … }, "tags": [], "requires_approval": true, "privacy_score": 20,
             "is_active": true, "relationships": [], "metadata": { "purpose": "…" }, "gated_fields": ["endpoint_url"] },
  "access_request": { "id": "req_…", "status": "pending", "requester": { … }, "requester_groups": ["engineering"],
                      "form": { "justification": "…", "intended_use": "internal_app" }, "created_at": "…" },
  "links": { "portal": "/portal/plugins/asset-catalog#/assets/ast_…", "admin": "/admin/enterprise/asset-catalog/requests#req_…" }
}
```

Gated metadata values are never included in events. `version`, `type` and `extra` fields are present when relevant (e.g. `extra.from` / `extra.to` on lifecycle changes, `extra.dependents` on dependency warnings).

### Consuming events from another plugin

```go
import acevents "github.com/TykTechnologies/midsommar/enterprise/plugins/asset-catalog/events"

func (p *WebhookPlugin) OnSessionReady(ctx plugin_sdk.Context) {
    topic := acevents.Topic("", acevents.KindRequestCreated) // "asset_catalog.access_request.created"
    ctx.Services.Events().Subscribe(topic, func(e plugin_sdk.Event) error {
        payload, err := acevents.Decode(e.Payload)
        if err != nil { return err }
        return p.post(payload.Links.Admin, payload.AccessRequest)
    })
}
```

## Notifications

- New access request → every admin with notifications enabled (title `Access request: <asset> (<type>)`, markdown body with the form answers and an admin link).
- Decision → the requester (`Access approved: …` / `Access denied: …`).
- A linked App's change sends an asset back to review, an asset is deprecated, a review falls due, becomes overdue or lapses → the owners (and admins for overdue and lapsed).
- An accountable owner leaves → admins and the fallback owner.
- New community submissions use the platform's own submission notifications.

## Governance Metadata (Enterprise)

Assets take part in the platform's [Governed Metadata](../../../docs/site/docs/governed-metadata.md): the admin-defined governance layer (owners, lifecycle state, risk tier, data classification, regulatory applicability, approved consumers, support contact, expiration date, plus custom schemas) shared with LLMs, Tools and Data Sources. Governance fields are **not** part of a type's own `schema`; Studio validates and stores them, keyed by asset ID, and lists the assets in **Governance → Metadata compliance**.

- **Opt-in per type.** Every asset type is governed by default (`governed: true`, editable in the Asset Types editor). Governed types are registered with `supports_metadata`, so they appear under *Applies to* in the schema form as `plugin_resource:<plugin id>:<slug>`. Opting a type out stops new writes; existing records are left to Studio.
- **Create and edit.** `admin_create_asset` and `update_asset` accept a `governance` object. The plugin validates its own metadata schema first, then calls `SetObjectMetadata`, then persists. When an enforcing schema rejects the values the RPC fails with code `governance_invalid`, `details` carries `{ valid, enforced, errors[{ field, code, message }], warnings }` and nothing is saved; advisory schemas only report. If persisting the asset fails after a successful write, the previous record is restored (or removed for a new asset). Governance is not versioned content: changing it never creates a version, and restoring a version leaves it untouched.
- **Community submissions** carry no governance form. The asset is created without a record and shows up as missing in the compliance report until an owner or administrator fills it in.
- **Delete and deactivate.** Deleting an asset removes its record first (a failure keeps the asset so the delete can be retried). Deactivating a type removes the records of its assets, since Studio does not cascade.
- **Rendering.** The admin form embeds the host `<governed-metadata-fields>` element (object type `plugin_resource:<plugin id>:<slug>`, `plugin_id` is returned by `list_types` / `admin_list_types` under `governance`). Portal users only ever receive the portal-visible display list (`governance_display`), shown with `<governed-metadata-badges>`; owners edit through the same form element fed with `get_governance_schema`, and user-picker fields are hidden there (their stored values are preserved on save).
- **Unavailable governance** (Community Edition, an older Studio, scopes not yet approved, or a type not yet synced) is skipped: saves proceed, nothing is rendered, and one warning is logged. Any other Studio error fails the save.
- Optional follow-ups not implemented: shipping a default governance schema from `manifest.json` (`metadata.schemas`, the seeded *Governance Core* schema already applies) and reacting to `system.governed_metadata.updated` / `.deleted` events.

## Development

```bash
# Go
cd enterprise/plugins/asset-catalog
go test -tags enterprise ./...
go build -tags enterprise -o asset-catalog .

# UI (web components, esbuild -> ui/dist, committed and embedded)
cd ui && npm install && npm run build
npm test   # node:test unit tests for the pure helpers (ui/test)

# Dev environment (plugin watcher builds enterprise plugins automatically)
make dev-full-ent-plugins

# Browser walkthrough of the permission rows (from the repository root, dev stack running)
cd tests/ui && npx playwright test tests/asset-catalog-rbac.spec.ts --project=chromium
```

Register the binary in Admin → Plugins with `file:///app/bin/plugins/asset-catalog`, grant the service scopes listed above, and activate it. Release with `make plugin-publish NAME=asset-catalog` from the repository root.
