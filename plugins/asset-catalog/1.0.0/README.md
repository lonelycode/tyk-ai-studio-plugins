# Asset Catalog (Enterprise)

Register, own, version, relate and govern the AI assets your organisation cares about but the gateway never proxies: **agents, prompts, skills, agent configurations, hooks, guardrails, external AI services**. The plugin turns each asset type into a first-class AI Portal resource with ownership, lineage, lifecycle tags, relationships, community submissions and an access-request workflow that publishes events for downstream automation.

## What you get

| Capability | Where |
|---|---|
| **Asset types** defined at runtime with a JSON Schema for their metadata, gated fields, an approval flag, a lifecycle policy and allowed relationship kinds. Two bundled types: **Agent** and **Prompt** (plus two sample assets). | Admin → Asset Catalog → Asset Types |
| **Assets** with three ownership roles (created by / responsible / accountable), tags, validated metadata plus free-form extra metadata, a version history with change notes and lineage, typed relationships (`depends_on`, `uses`, `recommended_with`, `derived_from`, `supersedes`) with cycle detection, and lifecycle stages `draft → experimental → in_review → approved → production → deprecated`. | Portal → Asset Catalog; Admin → Asset Catalog → Assets |
| **Community submissions**: portal users propose new assets through the platform's Submission form (the type's schema drives the form); admins review them in the Submission Queue; approval creates the asset owned by the submitter. | Portal → Contribute |
| **Access requests**: types (or individual assets) can require approval. Users fill in a form; admins get an in-Studio notification; approval records a grant and unlocks the asset's gated fields for that user. | Portal asset page; Admin → Asset Catalog → Access Requests |
| **First-class resource types**: every asset type is registered with AI Studio, so assets can be attached to Apps, assigned to groups, and appear in the portal sidebar and the gateway config snapshot. | App form, Teams |
| **Governance Metadata**: assets of governed types carry the platform's admin-defined governance fields (owners, lifecycle state, risk tier, data classification, regulatory applicability, …) with the same schemas, vocabularies, enforcement level and compliance report as LLMs, Tools and Data Sources. Studio validates and stores them; an enforcing schema blocks the save. | Admin and portal asset forms; Governance → Metadata schemas / compliance |
| **Events** on the internal bus for every change, so a webhook, email or ticketing plugin can automate follow-on work. | see below |

## Requirements

- Tyk AI Studio ≥ 2.2.0 with a valid enterprise license (optionally gated on the `feature_asset_catalog` entitlement via configuration).
- Service scopes: `kv.readwrite`, `license.read`, `notifications.write`, `resource-types.manage`, `metadata.read`, `metadata.write` (the last two for Governance Metadata; without them the plugin still works, governance is simply skipped).
- The event bus is only available when Studio runs with `GATEWAY_MODE=control`; in standalone mode the plugin still works but events are logged as skipped.

## Configuration

| Key | Default | Description |
|---|---|---|
| `seed_examples` | `true` | Create the two sample assets on first start (bundled types are always created). |
| `default_requires_approval` | `false` | Approval requirement for types that do not declare one. |
| `notify_admins` | `true` | Raise an in-Studio notification for every new access request. |
| `event_topic_prefix` | `asset_catalog` | Events are published as `<prefix>.<kind>`. |
| `require_license_feature` | `false` | Exit unless the license lists `feature_asset_catalog`. |

## Data model

- **Asset type**: `slug`, `name`, `description`, `icon`, `schema` (JSON Schema object; mark a property with `"x-gated": true` to hide it from users without access), `requires_approval`, `access_request_schema` (JSON Schema for the request form; default is a single required `justification`), `lifecycle_policy { initial, admin_only_stages }`, `relationship_kinds`, `has_privacy_score`, `schema_version`.
- **Asset**: `id`, `type_slug`, `name`, `description`, `tags`, `owners { created_by, responsible, accountable }`, `lifecycle`, `lifecycle_history`, `metadata` (validated), `extra` (free-form), `current_version`, `lineage`, `relationships[] { kind, target_asset_id, note }`, `requires_approval` (optional per-asset override), `grants[]`, `community_submitted`, `submission_id`, `privacy_score`.
- **Version**: full snapshot of the asset at that version, `change_notes`, `changed_by`, `parent_version`. Content edits (name, description, metadata, lineage, relationships) create a version; tag, ownership and lifecycle changes are recorded in history only. An earlier version can be restored: this appends a new version with the old content (change notes `Restored from version N`) rather than rewinding history, and the restored content must still satisfy the type's current schema.
- **Access request**: `asset_id`, `requester`, `requester_groups`, `form`, `status` (`pending | approved | denied | cancelled`), `reviewer`, `decision_note`.

### Rules

- Listings are visible to every portal user; non-owners see only `approved`, `production` and `deprecated` assets. Owners and admins see everything.
- Gated fields are visible to owners, admins and users with a grant. If the asset (or its type) does not require approval, everyone sees everything.
- Portal users create assets only through the Submission form. Admins create directly. Owners edit and version their own assets. Content edits require `change_notes`.
- `approved`, `production` and `deprecated` are admin-only stages by default (per-type policy).
- `depends_on` relationships cannot form cycles. Deprecating or deleting an asset that others depend on requires an admin `force` and emits `asset.dependency_warning`.
- Type schemas can be extended at any time. Removing a property that existing assets still use requires `force`. Existing assets are re-validated on their next edit only.

## RPC contract (UI ↔ plugin)

All UI calls go through one router. Portal pages call `portalPluginAPI.call(method, payload)`; admin pages call `pluginAPI.call(method, payload)`. Both carry the authenticated user; `admin_*` methods require an administrator.

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
| `restore_version` | `{ id, version, change_notes? }` | `AssetView` (owner or admin). Creates a new version whose content equals `version`; history is never rewritten |
| `my_assets` | same as `list_assets` | assets the caller owns (any role), including drafts |
| `update_asset` | `{ id, name?, description?, tags?, metadata?, extra?, lineage?, governance?, change_notes }` | `AssetView` (content changes need `change_notes`; `governance` replaces the Governance Metadata values without creating a version; admins may also send `privacy_score`, `requires_approval`, `clear_requires_approval`) |
| `set_relationships` | `{ id, relationships: [{ kind, target_asset_id, note? }], change_notes? }` | `AssetView` (replaces all outbound edges) |
| `transfer_ownership` | `{ id, role: "responsible" \| "accountable", to: { user_id, email, name } }` | `AssetView` (accountable owner or admin) |
| `transition` | `{ id, stage, note?, force? }` | `AssetView` |
| `get_access_request_form` | `{ asset_id }` | `{ schema, requires_approval, has_access, pending_request_id }` |
| `request_access` | `{ asset_id, form }` | `AccessRequest` (`code: not_required` when the caller already has access) |
| `my_requests` | `{ status? }` | `{ requests: AccessRequest[] }` |
| `cancel_request` | `{ id }` | `AccessRequest` |
| `my_grants` | `{}` | `{ grants: [{ asset_id, asset_name, type_slug, granted_at, granted_by, request_id }] }` |
| `revoke_grant` | `{ asset_id, user_id }` | `AssetView` (accountable owner or admin) |
| `whoami` | `{}` | `{ user_id, email, name, is_admin, groups }` |

### Admin methods

| Method | Payload | Returns |
|---|---|---|
| `admin_list_types` | `{}` | like `list_types` but includes inactive types |
| `admin_upsert_type` | `{ slug, name, description?, icon?, schema?, requires_approval?, access_request_schema?, lifecycle_policy?, relationship_kinds?, has_privacy_score?, force? }` | `Type` |
| `admin_deactivate_type` / `admin_reactivate_type` | `{ slug }` | `Type` |
| `admin_list_assets` | like `list_assets` | all assets, including inactive |
| `admin_create_asset` | `{ type_slug, name, description?, tags?, metadata?, extra?, relationships?, lineage?, responsible?, accountable?, requires_approval?, privacy_score?, lifecycle?, governance?, change_notes? }` | `AssetView` (`governance` is validated and stored by Studio before the asset is persisted) |
| `admin_delete_asset` | `{ id, hard?, force? }` | `{ deleted, id, hard }` |
| `admin_list_requests` | `{ status? }` | `{ requests: AccessRequest[] }` |
| `admin_get_request` | `{ id }` | `AccessRequest` |
| `admin_decide_request` | `{ id, decision: "approved" \| "denied", note? }` | `AccessRequest` |
| `admin_reseed_examples` | `{}` | `{ seeded: true }` |
| `admin_stats` | `{}` | `{ types, assets, by_type, by_lifecycle, pending_requests, total_requests, grants }` |

### `AssetView`

The asset plus: `type_name`, `gated_fields` (names of gated properties), `has_access`, `is_owner`, `can_edit`, `effective_requires_approval`, `pending_request_id`, `inbound[] { kind, from_asset_id, from_name, from_type }`, `related[] { kind, asset_id, name, type_slug, lifecycle, note }`, `allowed_transitions`. When `has_access` is false the gated keys are absent from `metadata` and `grants` is empty. Single-asset reads (`get_asset`) also carry `governed`, `governance_display[] { key, label, type, value }` (the portal-visible Governance Metadata, for everyone who can see the asset) and, for editors only, `governance` (every stored value).

## Events

Published on the internal bus with direction `local`. Topics are exact strings (the bus has no wildcards): subscribe to each topic or use `SubscribeAll` and filter on the payload's `event` field.

| Topic (`asset_catalog.` prefix) | When |
|---|---|
| `asset.created`, `asset.updated`, `asset.version_created`, `asset.lifecycle_changed`, `asset.relationships_changed`, `asset.ownership_changed`, `asset.deleted`, `asset.dependency_warning` | asset mutations |
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

# Dev environment (plugin watcher builds enterprise plugins automatically)
make dev-full-ent-plugins
```

Register the binary in Admin → Plugins with `file:///app/bin/plugins/asset-catalog`, grant the service scopes listed above, and activate it. Release with `make plugin-publish NAME=asset-catalog` from the repository root.
