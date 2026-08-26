# OAuth2 Client Credentials Auth (Enterprise)

**Let machine clients authenticate to the AI gateway with JWT access tokens from your own identity provider — Microsoft Entra ID, Auth0, Okta, or any OIDC-compliant IdP — instead of platform API keys.**

| | |
|---|---|
| Plugin ID | `com.tyk.enterprise.oauth2-client-credentials` |
| Tier | Enterprise (requires a valid enterprise license) |
| Runs in | Microgateway (token validation) with AI Studio management UI |
| Hooks | `auth` (custom authentication), `studio_ui` |
| Admin UI | **OAuth2 Auth** sidebar section: Identity Providers (`/admin/enterprise/oauth2-auth/providers`), App OAuth2 Settings |
| Requires | AI Studio ≥ 2.6.0, Microgateway ≥ 1.0.0 |

## What it does

Out of the box, applications call the AI gateway with AI Studio-issued credentials. This plugin replaces that with the **OAuth2 client-credentials flow**: services obtain a JWT access token from your corporate identity provider and present it as a Bearer token; the gateway validates the token cryptographically and maps it to an AI Studio **App**, so all existing per-App governance (LLM access, budgets, rate limits, analytics) applies unchanged.

A client obtains a token from your IdP with its client ID and secret (e.g. `POST {issuer}/oauth/token` with `grant_type=client_credentials`), then calls the gateway with `Authorization: Bearer <token>`. On each request the plugin:

1. Reads the Bearer token and determines its issuer.
2. Matches the issuer against your configured **identity providers** (several providers may share an issuer, differentiated by audience).
3. Fetches the signing key from the provider's JWKS endpoint (cached) and validates the token — signature, issuer, audience, and expiry, with configurable clock-skew leeway. Only asymmetric signature algorithms (RS256/384/512, ES256/384/512) are accepted; HMAC and unsigned tokens are rejected.
4. Extracts the **tenant** and **client** identity from configurable claims.
5. Resolves that identity to an AI Studio App — either one you bound manually, or one created on the fly by **auto-provisioning** (see below).

A validated request is attributed to the resolved App; the authentication context records the provider, tenant ID, client ID, and auth method. Since this is the client-credentials (machine-to-machine) flow, no user identity is involved. If no provider matches, the token fails validation, or no App can be resolved, the request is rejected as unauthenticated.

## When to use it

- **Centralized machine identity** — your organization manages service credentials in Entra ID/Auth0/Okta; teams shouldn't be handed a second, parallel set of gateway keys to store and rotate.
- **Zero-touch onboarding of internal services** — with auto-provisioning, a service granted the right IdP permission can call the gateway immediately; a governed App is created for it on first request, cloned from a template you control.
- **Multi-tenant SaaS** — tokens carry a tenant claim; each tenant/client combination maps to its own App, giving per-tenant budgets and analytics for free.
- **Compliance** — short-lived, IdP-issued, cryptographically verified tokens instead of long-lived static keys.

## Configuring identity providers

Manage providers on the **Identity Providers** page. Each provider has:

| Field | Default | Description |
|-------|---------|-------------|
| `id`, `name` | *(required)* | Identifier and display name. The `id` must be unique — it is baked into App bindings (`provider_id:tenant:client`), so recreating a provider under a new `id` orphans existing bindings |
| `issuer` | *(required)* | Token issuer URL — must be HTTPS; trailing slashes are ignored when matching |
| `jwks_uri` | *(auto-discovered)* | JWKS endpoint for signature keys — must be HTTPS. Leave empty to auto-discover it from `{issuer}/.well-known/openid-configuration` |
| `audience` | — | Expected `aud` value (the API identifier your IdP stamps into tokens, e.g. `api://my-app`); also lets multiple providers share one issuer. **Leaving it empty skips audience validation** |
| `tenant_claim` | `tid` | Claim holding the tenant identifier (Entra ID uses `tid`) |
| `client_id_claim` | `azp` | Claim holding the client identifier |
| `additional_claims` | — | Extra claim name → expected value pairs that must match |
| `enabled` | `true` | Toggle the provider |

Global validation settings:

| Option | Default | Description |
|--------|---------|-------------|
| `token_leeway_seconds` | `30` (the settings form's default; omitted in raw config it falls back to 0) | Clock-skew tolerance for expiry checks |
| `index_rebuild_interval_seconds` | `60` | How often the gateway rebuilds its App lookup index from synced Apps |

(The settings form also shows `jwks_cache_ttl_seconds` and `require_not_before`; in the current version neither affects validation — JWKS refresh is driven by key rotation, see *Things to know*, and `nbf` is checked by default whenever present in a token.)

## Mapping tokens to Apps

Two ways to connect an IdP client to an AI Studio App:

### 1. Manual binding

On the **App OAuth2 Settings** page, bind an existing App to a provider/tenant/client combination. The binding is stored in the App's metadata (`oauth2_provider_id`, `oauth2_tenant_id`, `oauth2_client_id`), and the gateway indexes all Apps by these fields (refreshed every `index_rebuild_interval_seconds`).

### 2. Auto-provisioning

With `auto_provision_enabled`, an unknown-but-valid client is provisioned automatically on its first request:

1. The gateway reads the token's **permissions claim** (`permissions_claim`, default `permissions` — Auth0's default; for Entra ID, whose app roles arrive in the `roles` claim, set this to `roles`).
2. Permissions are matched against your **template mappings** (`permission_tag` → `template_app_id`).
3. If a mapping matches, the gateway sends a provisioning request up to AI Studio, which creates a new App copying the template's **LLM access, tools, datasources, and monthly budget**, owned by the configured **system user** and stamped with the client's OAuth2 metadata. When several permission tags match several templates, the new App gets the union of their LLMs/tools/datasources and the largest budget. The App is named after the first template plus the client ID.
4. The gateway waits for the new App to arrive (up to `provision_timeout_seconds`, default 10) and then completes the request. Concurrent first requests from the same client are de-duplicated (per gateway, with a Studio-side idempotency check), and an existing App with matching OAuth2 metadata is reused rather than duplicated.

> **Prerequisite:** the plugin must also be installed and enabled in AI Studio itself — the Studio side receives provisioning requests and creates the Apps. With a gateway-only deployment, auto-provisioning requests simply time out.

| Option | Default | Description |
|--------|---------|-------------|
| `auto_provision_enabled` | `false` | Enable first-seen provisioning |
| `system_user_id` | *(required when enabled)* | Studio user that owns auto-provisioned Apps — create a dedicated system user first |
| `permissions_claim` | `permissions` | JWT claim holding permission tags |
| `provision_timeout_seconds` | `10` | How long the gateway waits for Studio to create the App |
| `template_mappings` | *(required when enabled)* | List of `{permission_tag, template_app_id, description}` |

**Template Apps** are ordinary Apps you configure once with the desired LLM access, tools, datasources, and budget; every auto-provisioned client gets a copy of those. Different permission tags can map to different templates (e.g. `ai.basic` → limited template, `ai.premium` → full-access template).

## Failure behavior

- Tokens with an unknown issuer, a failed signature/audience/expiry check, or missing tenant/client claims are rejected. The caller sees a short error (e.g. "unknown token issuer", or a generic "authentication failed" for validation failures); full details are logged server-side.
- If auto-provisioning is disabled and no App matches, the request is rejected.
- If provisioning is enabled but no template mapping matches the token's permissions, or provisioning times out, the request is rejected.

## Things to know

- The gateway needs outbound HTTPS access to each provider's JWKS (and, when auto-discovering, OIDC discovery) endpoint; issuer and JWKS URLs are required to be HTTPS.
- Key rotation at the IdP is picked up automatically: when a token arrives signed with an unknown key ID, the gateway re-fetches the JWKS (throttled to at most once per 5 minutes per endpoint).
- Auto-provisioning is a gateway → Studio round trip on the *first* request of a new client; subsequent requests resolve locally. Size `provision_timeout_seconds` accordingly if your hub link is slow.
- The **App OAuth2 Settings** page includes a token inspector: paste a JWT to see its claims and check which provider it matches — the fastest way to debug a token that isn't authenticating.
- The plugin validates the platform's enterprise license at startup; without a valid enterprise license the plugin shuts down and requests relying on it fail.
