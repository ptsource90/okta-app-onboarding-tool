# Security

## Authentication

- The API is a relying party for whichever third-party identity store the
  deployment is configured with (Firebase, Okta, or a generic OIDC/SSO
  provider) — it never issues its own primary credentials. **This stays
  provider-agnostic even though the frontend's own auth library is now
  Okta-specific** (`@okta/okta-react`, `06-tech-stack.md`,
  `10-session-and-token-lifecycle.md`) — JWT/JWKS validation here doesn't
  care which SDK issued the token, so the backend's swappability isn't
  affected by that frontend commitment.
- Every request to a protected endpoint must carry a bearer token that the
  backend validates: signature, issuer, audience, and expiry, using the
  identity provider's public keys (JWKS) rather than a shared secret where
  the provider supports it.
- The frontend never makes authorization decisions that the backend
  trusts blindly — role-based UI hiding is a UX convenience only.

## Authorization

- Two roles: **Admin** (unrestricted) and **User** (owned resources only).
- Role is enforced in the Application layer alongside each use case
  (e.g. the "list apps" query filters by `OwnerUserId` for a `User`
  caller; the "delete app" command checks ownership before proceeding for
  a `User` caller), not only in a single global middleware — a coarse
  "is the user authenticated" check in middleware plus a fine-grained
  ownership check per use case.
- Explicitly test for IDOR-style gaps: a `User` supplying another user's
  app ID directly (e.g. via the URL or request body) must be rejected, not
  just hidden from the list view.
- Role changes (Admin promoting/demoting a user) are themselves an
  audited, Admin-only action.
- **Edit authorization:** an Admin can edit any app; a `User` can edit only
  apps they own (`OwnerUserId = currentUser.Id`), enforced the same way as
  the ownership check on delete/list. Editing is never approval-gated in
  any environment, including Production — unlike delete, there is no
  pending-request branch to also verify.
- **Assignment authorization:** same ownership rule as Edit (Admin: any
  app; User: only apps they own), never approval-gated in any
  environment. The search/typeahead endpoint (`SearchOktaPrincipalsQuery`)
  is subject to the same ownership check as assigning itself — a User
  shouldn't be able to search an app's Okta org just because they can see
  the app in a list, without also passing the ownership check that
  assigning would require. Minimum-2-character enforcement is server-side,
  not just a frontend nicety, so a caller can't force a wide,
  Okta-rate-limit-costly search with a 1-character or empty query.
- **Delete-approval workflow authorization (confirmed scope):** the gate
  applies only when a User deletes an app whose environment is
  **Production**. A User deleting a Staging app, and any Admin deletion in
  any environment, call the Okta delete API directly. For a gated
  (Production) delete, the User's action must never call the Okta delete
  API directly — it may only create an `ApprovalRequest`. Any Admin
  (confirmed: no separate approver role) can transition an
  `ApprovalRequest` to `Approved` or `Rejected`; approval is the sole
  trigger that performs the actual Okta deletion. Verify both the
  environment check and the approval gate at the Application layer (not
  just hidden in the UI), so a User cannot reach the immediate-delete code
  path for a Production app ID, and cannot approve their own request by
  calling the approval endpoint directly.
- **Delete precondition (confirmed):** every delete code path —
  immediate delete, requesting a gated delete, and approving a pending
  delete request — validates `Application.IsActive = false` server-side
  before proceeding, in every environment. This is a hard precondition,
  not just a hidden/disabled button in the UI; approval-time
  re-validation matters specifically because the app could have been
  reactivated in the time between the request being created and an Admin
  reviewing it.
- **Deactivate/Reactivate authorization (confirmed: neither is ever
  approval-gated):** an Admin can deactivate or reactivate any app; a
  `User` can deactivate or reactivate only apps they own — same
  ownership check as edit. Neither ever creates an `ApprovalRequest`, in
  any environment, for either role. (An earlier draft gated a User's
  Production deactivation like delete; that was reconsidered and
  removed.)
- **Owner-reassignment authorization (confirmed: Admin-only):** a `User`
  cannot reassign ownership of any app, including one they currently own
  — there is no ownership-based path into this action the way there is
  for edit/delete/assign. The Application layer must reject a User-role
  caller outright, not merely hide the control in the UI. A reassignment
  request without a non-empty `Reason` is rejected server-side (the
  reason is a required field, not a client-side nicety). Writes an
  `OwnerReassigned` audit entry before returning success, same as every
  other mutating action. No Okta call is made — see `04-data-model.md`.
- **Credentials-tab authorization:** identical to viewing the app itself
  (Admin: any app; User: only apps they own) — there is no separate
  "can view credentials" permission to configure independently, and no
  path exists to fetch credentials for an app the caller couldn't
  otherwise see. See "Secrets management" above for the handling rules
  once a caller is authorized.
- **Edit-conflict handling is not an authorization concern, but is a
  defense-in-depth one:** the `RowVersion` mismatch check
  (`04-data-model.md`) happens server-side regardless of what the
  frontend last fetched, so a stale client can't force an Okta write
  based on an outdated view of the app just by resubmitting an old form.
  An explicit "keep mine" override still requires the caller to first
  receive the *current* `RowVersion` from the conflict response — it
  can't be replayed blind.
- **Duplicate-name check is a UX affordance, not a security boundary:**
  the `onBlur` availability check reads via the same ownership/role rules
  as the rest of app listing (nothing here should leak a Production app's
  name/type/owner to a User who couldn't otherwise see that
  environment's apps — but since environment access itself isn't scoped
  per user in this model, this doesn't introduce a new exposure; see
  `01-requirements.md`'s "Environment access" section). The actual
  enforcement is Okta's own rejection at submit time, which this check
  can never bypass or weaken.
- **Import authorization:** importing existing Okta apps is Admin-only
  (it exposes org-wide app inventory and assigns ownership on another
  user's behalf) — this is still a default to confirm, not yet explicitly
  asked about; tracked in `08-open-questions.md`.
- **Scheduled sync job authorization:** the job is not triggered by, or
  acting on behalf of, any user — it runs under the same per-environment
  OAuth 2.0 service app credential (`OktaServiceAppClientId` + private
  key, private_key_jwt, below) the rest of the backend already uses for
  that environment, so there's no separate user-authorization check to
  make. Its writes are still audited (`UserId = null`, a sync-specific
  `Action`) so they're distinguishable from human actions. See
  `09-drift-sync.md`.

## Secrets management

- **Confirmed: production secrets are supplied via environment
  variables**, not a cloud secret manager (Key Vault/Secrets Manager) —
  this was a deliberate hosting-model decision (`06-tech-stack.md`,
  `08-open-questions.md`), not an oversight, so don't reintroduce a
  cloud-secret-manager design without re-confirming it supersedes this.
- **Confirmed: backend-to-Okta authentication is OAuth 2.0 for Okta
  APIs (`client_credentials` grant, `private_key_jwt` client
  authentication)** — not a static Okta API token (SSWS token). This
  isn't a style preference: Okta only supports `private_key_jwt` for a
  service app minting access tokens with Okta scopes. See
  `04-data-model.md`'s "Backend-to-Okta authentication" section for the
  full setup/runtime flow, and `02-architecture.md` for where the
  resulting short-lived access token is cached and refreshed.
- The per-environment **private key** (a JWK) and any OIDC/SAML client
  secrets or signing certificates handled during app creation/copy are
  secrets:
  - Never committed to source control.
  - Never stored in plaintext in the application database.
  - Never logged, under any circumstance — this includes the
    `client_assertion` JWT signed with it and the resulting Okta access
    token, neither of which should ever appear in application logs even
    at debug level.
  - **Never included in API responses — with one narrow, deliberate
    exception:** the Credentials tab (`01-requirements.md` requirement
    14) *does* return an OIDC Client Secret or a SAML signing
    certificate, but only (a) to a caller who already passes the same
    ownership/role check as viewing the app itself, (b) fetched live from
    Okta on each request — never read from a cache or from
    `ConfigurationJson` — and (c) logged as its own audited read
    (`AppCredentialsViewed`) rather than silently returned. Every other
    endpoint keeps the blanket "never in a response" rule — **this
    explicitly includes the backend's own service-app private key and
    the access tokens it obtains**, which are never surfaced via any
    endpoint, not even to an Admin.
  - Referenced by the `Environment` entity via an **environment-variable
    name** (e.g. `OKTA_STAGING_PRIVATE_KEY`), never embedded in it — see
    `04-data-model.md`. The service app's **Client ID is not treated as
    a secret** (an identifier, not a credential) and is stored as a
    plain `Environment` column instead.
- **IIS-hosted deployment note:** an ASP.NET Core app under IIS does not
  automatically inherit ordinary user/session-level Windows environment
  variables — set secrets at the **Application Pool level** (IIS
  Configuration Editor / `appcmd`, backed by `applicationHost.config` on
  the server itself), not inside the deployed app's own `web.config`.
  `web.config` ships as part of the publish/deploy artifact, so anything
  written into its `<aspNetCore><environmentVariables>` block sits
  alongside source-controlled build output — the app-pool-level setting
  keeps the secret on the server, outside anything that gets built,
  packaged, or checked in. Coordinate with `devops-engineer` on the exact
  provisioning mechanism per environment. **Note the private key's value
  is a JSON-serialized JWK, not a plain string token** — confirm the
  App Pool environment-variable mechanism handles a JSON value (escaping,
  length) cleanly before relying on it, rather than assuming it behaves
  identically to the simpler string values this project previously
  stored this way.
- Local development uses a mechanism like `dotnet user-secrets`, never a
  checked-in `appsettings.json` value.
- Secrets are rotatable independently of application deploys (an
  environment-variable value can be changed and the app pool recycled
  without a code deploy).
- The `appsettings.json`-based environment seed config (`04-data-model.md`,
  used to populate `Environment` rows on first migration) and the
  drift-sync cron setting (`09-drift-sync.md`) are both non-secret and
  fine to check in — the seed entries contain `OktaServiceAppClientId`
  directly (not a secret) but only the environment-variable *name* for
  each environment's private key, never the key value itself, per the
  rule above.

## Audit logging as a security control

- Every create/edit/delete/copy action, every role change, and every
  owner reassignment writes an `AuditLogEntry` (see `04-data-model.md`)
  before the operation is reported as successful to the caller — the
  audit write is part of the operation, not a best-effort side effect.
  Viewing the Credentials tab (`AppCredentialsViewed`) is the one
  read-only exception logged this way, precisely because it's a
  secret-disclosure event rather than an ordinary read.
- Audit entries are append-only from the application's perspective (no
  update/delete endpoint exists for them); if correction is ever needed
  it happens via a new corrective entry, not by editing history.
- **Confirmed: Admin sees the full/global audit log; a `User` sees the
  audit trail scoped to apps they own** — `WHERE ApplicationId IN
  (SELECT Id FROM Application WHERE OwnerUserId = currentUser.Id)`,
  enforced in the Application layer, not just hidden in the UI (same
  pattern as every other ownership check in this doc). This includes
  every entry tied to one of their apps regardless of who or what
  performed it — their own actions, an Admin's edit/delete of their app,
  their own delete-approval-request transitions, and system entries from
  the drift-sync job (`AppSyncedFromOkta`, `AppDeletedInOkta`,
  `ApprovalRequestAutoCancelled`) — because the point is a complete
  history of *that app*, not just of the User's own clicks. A `User`
  never sees entries with `ApplicationId = null` (e.g. `RoleChanged`) or
  entries for apps they don't own.

## General application security

- Validate and sanitize all input server-side regardless of client-side
  validation (OIDC redirect URI formats, SAML certificate formats, etc.).
- Rate-limit sensitive endpoints (app creation/edit/copy) per user to
  reduce abuse and to stay well inside Okta's own API rate limits. **This
  is distinct from handling Okta's own rate-limit (429) responses to
  this app's outbound calls** (retry/backoff, especially for the
  drift-sync job's `ListAllAsync` and the import scan) — **now
  substantially covered by `Okta.Sdk`'s own built-in 429 retry**
  (configurable `MaxRetries`/`RequestTimeout`), confirming the defaults
  are appropriate is the remaining task, not a from-scratch build; see
  `08-open-questions.md`.
- Paginated, searchable/filterable endpoints (app list, audit log —
  `06-tech-stack.md`) enforce a server-side maximum `pageSize` regardless
  of what the client requests, so a caller can't force an unbounded,
  expensive query by asking for an enormous page.
- **Sieve filter/sort properties are explicitly allow-listed**
  (`[Sieve(CanFilter = true, CanSort = true)]` per query DTO property) —
  never all EF Core properties by default — so a caller can't filter or
  sort on a field that wasn't deliberately exposed.
- **Mandatory ownership/role scoping is applied before the caller's own
  Sieve filters, not merged alongside them as just another condition.**
  A User's `WHERE OwnerUserId = currentUser.Id` (or `ApplicationId IN
  (owned apps)` for the audit log) narrows the query first; Sieve's
  `filters=` then only ever searches within that already-scoped result
  set. This matters because Sieve's filter syntax is caller-controlled —
  it must never be trusted to enforce scope on its own.
- Keep dependencies (NuGet and npm) current and scanned for known
  vulnerabilities as part of CI (coordinate with `devops-engineer`).
- Apply the OWASP Top 10 as a review lens for every new endpoint —
  the `cybersecurity-engineer` agent should review before a feature is
  considered done, not just at the end of the project.

## Threat notes (non-exhaustive, expand during design review)

- A compromised low-privilege `User` account could attempt to enumerate
  or guess other users' app IDs — mitigated by ownership checks above.
- A leaked environment's private key would allow minting valid Okta
  access tokens and app creation/deletion in that environment directly
  against Okta — mitigated by scoped (least-privilege), rotatable
  (dual-key overlap, `04-data-model.md`), per-environment keys and
  monitoring/alerting on unusual Okta API activity (coordinate with
  `devops-engineer`). Worse than a leaked static token in one respect
  (a private key can keep minting new access tokens indefinitely until
  rotated, not just until one token's own expiry) — all the more reason
  rotation is a designed-in capability, not an afterthought.
- The promotion flow (Staging → Production only) is a route to accidentally
  push a staging-only configuration value (e.g. a test redirect URI) into
  production — mitigated by the mandatory human confirmation step in
  `03-app-creation-flow.md`.
