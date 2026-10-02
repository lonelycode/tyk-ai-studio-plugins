# OAuth2 Client Credentials Auth (Enterprise)

Authenticates gateway requests with JWT access tokens from an external identity provider (Entra ID, Auth0, Okta, …) and maps each token to an AI Studio App by its client binding: the `(provider, tenant, client)` stored in the App's metadata.

## Where it authenticates

Attach the plugin to an endpoint as an auth plugin. An endpoint with auth plugins is authenticated by them alone, and app keys are refused there:

- An LLM: the LLM's plugins.
- A datasource, tool, model router, semantic router or custom-endpoint plugin: the **Authentication plugins** section on its detail page.

The App the token maps to must still hold the endpoint. Auth plugin lists on endpoints other than LLMs need AI Studio 2.2.1 or later.

## Bindings and revocation

Studio owns the binding table (`bindings/`):

- **Computing it.** The Studio half of the plugin builds the table from the active Apps' metadata. Apps awaiting approval (`oauth2_approval_status: pending`) are left out, and so is a client that two Apps claim: it fails closed, is logged, and shows as **Conflict** in the App OAuth2 settings.
- **When it rebuilds.** After any App change (`system.app.*` events), right after a binding is saved or removed in the plugin UI (metadata patches send no App event), and every 30 s as a backstop. Rebuilds are debounced by 2 s.
- **Publishing.** When the table changes it goes to every connected gateway (`oauth2-auth.bindings`, DirDown). Every 60 s Studio announces the table's hash (`oauth2-auth.bindings-hash`).

On the gateway:

- **Lookups use the table alone** once one has arrived. They used to read a KV cache first whose entries were never removed, so an unbound, rebound or deactivated App kept working.
- **Hash mismatch.** A gateway whose table hash differs from the heartbeat (it was disconnected, or just started) asks for the table (`oauth2-auth.bindings-resync`, DirUp, at most every 30 s).
- **Restarts.** The last table is persisted in plugin KV (`oauth2:bindings`) and restored on start, so a revoked binding stays revoked from the first request after a restart, before Studio has answered. (An edge cannot start without reaching its hub, so there is no offline start to cover.)
- **Hub unreachable at runtime.** The table in force stays in force; the gateway's own App rows, still stale until the next configuration push, never override it.
- **Before the first table** (the plugin's first start, before Studio answers), lookups use the gateway's own App rows, refreshed every `index_rebuild_interval_seconds`.

**Timing.**
- **Connected gateways:** a rebind, unbind or deactivation takes effect within about 2 s.
- **A gateway that missed the event:** catches up within 60 s.

The gateway still needs the App row itself, which arrives with a configuration push. A newly bound App authenticates once it is on the edge.

## Auto-provisioning and approval

With auto-provisioning on, the IdP drives App creation. You map IdP permission tags to template Apps once. From then on, a token from a client with no binding whose permissions match a mapping gets an App provisioned for it, with the matching templates' LLMs, tools, datasources and largest budget, owned by `system_user_id`. Studio matches the templates from the token's permissions itself; it does not take the template list a gateway sends.

Whether that App needs a human is up to you:

- **`require_approval` off (the default).** The App is live as soon as it is created: the IdP's permission assignment is the approval. The gateway asks Studio for the App and waits for it (up to `provision_timeout_seconds`, default 10), so the client's first call is served. This is the mode for IAM-driven and agentic setups, with no human in the loop.
- **`require_approval` on.** The App is created inactive, flagged through the governance API (`oauth2_pending_approval`) and marked `oauth2_approval_status: pending`, so it authenticates nothing.
  - The client is refused ("client pending approval" in the gateway log; the caller gets 401). The gateway asks Studio again for that client at most every 10 minutes.
  - Admins are notified once per client.
  - **To approve:** review and edit the App in Apps (owner, grants, budget), then approve it under OAuth2 Auth → App OAuth2 Settings → Pending approvals. Activating the App in Apps also approves it. Approval activates the App and sends it to the gateways, so it authenticates within seconds, before the next configuration push.
  - **To reject:** rejecting deletes the App.
- **Per mapping.** Each template mapping can override the plugin-wide setting: required, not required, or default. An App waits for approval if any mapping it was provisioned through requires approval. For example, auto-approve a cheap-model template and gate an expensive one.
- **Turning approval off later.** An App still waiting is approved on its client's next request.

The approval mode needs the `apps.lifecycle` and `notifications.write` service scopes (approve them when upgrading) and AI Studio 2.2.1.

## Validation

- **Providers.** A provider that fails validation is refused by the provider RPCs: missing ID, name or issuer, a non-HTTPS issuer or JWKS URI, or an empty additional-claim name. The plugin also drops it at start-up, so config saved through the generic plugin config API can't activate it. The provider list shows it as **Invalid: not active**.
- **Auto-provisioning.** Invalid auto-provisioning settings turn auto-provisioning off, with an error in the log.
- **Bindings.** A binding to a client another App already holds (active or not) is refused at save.

## Audit identity

The auth response carries:

- **The user id:** the token's subject claim, `subject_claim` per provider (default `sub`).
- **`auth_actor`:** the acting agent, which is `act.sub` for a delegated (RFC 8693) token, otherwise `azp`, otherwise `client_id`.

The gateway records them with the request as **For** and **Agent** on Studio's proxy logs (AI Studio 2.2.1 or later). They are for audit only; the client binding decides the App.
