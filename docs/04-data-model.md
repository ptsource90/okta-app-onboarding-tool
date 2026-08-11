# Conceptual data model

This is a conceptual model to guide the eventual EF Core entities and
migrations — not a finished schema. Exact column types/lengths/indexes are
an implementation-phase decision for `software-engineer`.

## Core entities

### User
- `Id` (internal)
- `ExternalSubjectId` — the `sub` claim (or provider-equivalent) from the
  third-party identity store
- `IdentityProvider` — which third-party store this user authenticated via
  (Firebase / Okta / generic SSO), for audit/debugging clarity
- `DisplayName`, `Email` (as provided by the IdP)
- `Role` — `Admin` or `User`
- `CreatedAt`

### Environment
- `Id`
- `Name` — e.g. `Staging`, `Production` (not hard-coded to exactly two;
  the model should support adding more without a schema change)
- `OktaOrgUrl` — the Okta org base URL for this environment. **Confirmed:
  each environment is a fully separate Okta org** (not different auth
  servers within one org) — so there is no shared Okta-level entity
  between two environments; a copy or import always crosses org
  boundaries, and rate limits are naturally isolated per environment.
- `PromotesToEnvironmentId` — nullable, self-referencing FK. **Confirmed
  rule: promotion only runs Staging → Production.** Modeling it as a
  pointer on `Environment` (e.g. Staging's row points at Production's row)
  rather than hard-coding a `"Staging"`/`"Production"` name check means the
  rule survives adding more environments later (`Name` strings could
  change or multiply; this FK is the actual source of truth for "what can
  this environment promote into, if anything"). Most environments
  (including Production) will have this `null` — nothing promotes out of
  Production.
- `OktaServiceAppClientId` — the Client ID of the OAuth 2.0 **service
  app** (Okta's "API Services" sign-in method) this environment's org has
  registered for this tool's own backend-to-Okta-API access. **Not a
  secret** (an OIDC/OAuth2 client ID is an identifier, not a credential)
  — stored as a plain column, unlike the private key below.
- Reference to the **environment-variable name** holding this
  environment's **private key** (a JWK, JSON-serialized), used to sign
  the `client_assertion` JWT for `private_key_jwt` client authentication
  — never the key value itself, stored in this table. **Confirmed: this
  replaces a static Okta API token (SSWS token) model** — see
  "Backend-to-Okta authentication" below and `05-security.md` for why,
  and `06-tech-stack.md`/`02-architecture.md` for how the resulting
  short-lived access token is obtained and cached. **Confirmed:
  production secrets are supplied via environment variables**, not a
  cloud secret manager (see `05-security.md`), so this column holds a
  lookup key, the same role a secret-store reference key would have
  played, just against a different backing mechanism — same pattern as
  before, just holding a JWK instead of a static token string now.

### Backend-to-Okta authentication (confirmed: OAuth 2.0 for Okta APIs,
### not a static API token)

**Confirmed:** the backend authenticates to each environment's Okta org
using **OAuth 2.0 for Okta APIs** — a `client_credentials` grant with
`private_key_jwt` client authentication — not a long-lived static SSWS
API token. Per Okta's own documentation, `private_key_jwt` is **the only
supported authentication method** for an OAuth 2.0 service app minting
access tokens with Okta scopes; `client_credentials`/a plain secret isn't
an option here, so this isn't a preference among several equally-valid
choices.

**Unrelated to, and not to be confused with, the "service" app type
explicitly excluded from the creation wizard** (`01-requirements.md`
requirement 19) — that exclusion was about apps *this tool onboards on
behalf of other teams*. This is a completely different concern: the
credential *this tool itself* uses to call each Okta org's own
Management API, provisioned once per environment during setup (an
Okta-admin/`devops-engineer` task, not something an end user of this
tool ever sees or triggers), never surfaced through the wizard.

**Per-environment setup (one-time, outside this tool):**
1. An Okta admin creates an OAuth 2.0 service app (Admin Console: Sign-in
   method = "API Services") in that environment's org.
2. A JWKS key pair is generated — **confirmed recommendation: generate
   the key pair outside Okta and register only the public key/JWKS URI**,
   rather than letting Okta generate and briefly hold a copy of the
   private key, so the private key never touches Okta's own systems at
   any point. (Okta's Admin Console does support "Save keys in Okta" for
   simple/testing setups, but that means Okta holds a copy — avoid that
   for Production at minimum.)
3. The service app is granted the specific Okta scopes this tool
   actually needs (least privilege, not a blanket admin scope) — e.g.
   `okta.apps.manage`, `okta.apps.read`, `okta.groups.read`,
   `okta.users.read`, and, only if those flagged features are ever built,
   `okta.inlineHooks.read`/`okta.authenticators.read`
   (`08-open-questions.md`).
4. The resulting Client ID and private key are what populate
   `OktaServiceAppClientId` and the private-key environment variable
   above, per environment.

**Runtime flow (per outbound call to that environment's Okta org) —
confirmed: SDK-native, not hand-implemented.** The protocol mechanics
below are what happens on the wire; this project does not implement
these steps by hand — `Okta.Sdk` (current stable `10.x`) does this
internally when its `Configuration` is set to
`AuthorizationMode.PrivateKey`, confirmed from the SDK's own README
(https://github.com/okta/okta-sdk-dotnet/blob/master/README.md#oauth-20):
1. Construct a `client_assertion` JWT: `iss`/`sub` = the service app's
   Client ID, `aud` = that org's token endpoint
   (`{orgUrl}/oauth2/v1/token`), a short `exp` (recommended: a few
   minutes, not the access token's own hour-long lifetime — this is a
   different, shorter-lived JWT than the access token it's requesting),
   and a unique `jti` (replay prevention). Signed with the environment's
   private key — the SDK does this given `PrivateKey =
   new JsonWebKeyConfiguration(jwkJson)` (the SDK accepts the JWK as a
   raw JSON string directly, matching how it's stored, per the
   environment-variable field above).
2. `POST {orgUrl}/oauth2/v1/token` with `grant_type=client_credentials`,
   `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`,
   `client_assertion=<the signed JWT>`, and the requested `scope`(s) —
   the SDK issues this call itself; **"you won't need an API Token
   because the SDK will request an access token for you"** (Okta's own
   words).
3. Receives a standard OAuth2 access-token response — **confirmed: this
   access token is short-lived (Okta's stated figure: ~1 hour)**, and is
   what's sent as `Authorization: Bearer <token>` on the actual
   Management API call (create/edit/delete/list an app, etc.), not the
   JWT from step 1.
4. **Flagged, not resolved by the README:** whether the SDK caches and
   reuses this access token across multiple calls made through the same
   long-lived client instance, or re-authenticates every call. A static
   SSWS token never expired, so this refresh concern didn't exist under
   the old model — confirm the SDK's actual behavior (`okta-expert`)
   before assuming either way; see `02-architecture.md`'s
   "Backend-to-Okta authentication" section and `08-open-questions.md`.

**Key rotation (confirmed pattern, sourced from Okta's own
org-to-org OAuth guidance):** register both the old and new key under the
same service app's JWKS array (each with its own `kid`) during a
rotation window, verify the new key works, then remove the old one — not
a hard cutover that risks an outage if the new key has a problem.

**Confirmed: seeded from appsettings on first create, not config-driven at
runtime.** `Environment` stays a real DB table (this doc's original
model) — it is not read live from appsettings on every request. But on
first database creation/migration, the initial rows are populated from a
config section in `appsettings.json` (e.g. `OktaEnvironmentSeed`: one
entry per environment with `Name`, `OktaOrgUrl`, `OktaServiceAppClientId`,
the environment-variable name holding the private key, and which
environment it promotes to, if any) rather than being hand-inserted or
hard-coded in a migration's `HasData`. This keeps the actual
per-deployment domain values (`okta-production-domain`, etc.) out of
source-controlled migration code, while `Environment` itself remains the
runtime source of truth the rest of the app reads from. The private key
itself is never part of this seed file — only the environment-variable
*name* is, same as the rest of the model; the Client ID isn't a secret,
so it's fine to seed directly.
**Open question (not yet decided, see `08-open-questions.md`):** if a
domain changes after initial seeding (e.g. an Okta org migration), is
updating the `Environment` row a manual DB/admin action, or should there
be a re-sync path that re-reads the config? Flagging rather than
assuming — "first create/migration" only was confirmed, not ongoing sync.

**Confirmed: no per-user environment-access entity.** There is
deliberately no `UserEnvironmentAccess` (or similar) join table between
`User` and `Environment`. Every authenticated user can target any
`Environment` — the only access scoping in this model is `Application.
OwnerUserId` (ownership) and `User.Role` (Admin vs. User). This is
possible because the backend authenticates to each Okta org with its own
per-environment OAuth 2.0 service app credential
(`OktaServiceAppClientId` + private key, above), never by delegating
through the calling user's own Okta identity — so there's no
Okta-org-level access to gate per user in the first place.

### Application (an onboarded Okta app record)
- `Id`
- `Name`
- `Type` — `OIDC` or `SAML`
- `Origin` — `CreatedByTool` or `ImportedFromOkta`; set once at creation/
  import time, kept for audit/UI context (e.g. "imported on <date>")
- `OktaAppId` — the ID of the corresponding object in Okta
- `EnvironmentId` — FK to `Environment`
- `OwnerUserId` — FK to `User`; the User-role visibility rule is
  `WHERE OwnerUserId = currentUser.Id` unless the caller is Admin
- `IsActive` — bool, default `true`. Independent axis from `DeletedAt`/
  `DeletionSource` below: an app can be active-and-not-deleted,
  deactivated-and-not-deleted, or deleted (regardless of whether it was
  active or deactivated right before deletion — deletion takes precedence
  for status-filter purposes, see `01-requirements.md`'s Status filter).
  Toggled by the deactivate/reactivate flow (`03-app-creation-flow.md`)
  and also kept in sync by the drift-sync job if Okta's own
  ACTIVE/INACTIVE status is changed directly in the Okta console
  (`09-drift-sync.md`).
- `SourceApplicationId` — nullable self-referencing FK, set when this app
  was created via the promotion flow, pointing at the Staging app it was
  promoted from (for traceability). Always null for an app created fresh
  or imported.
- `CreatedAt`, `UpdatedAt` (nullable — set on each edit), `DeletedAt`
  (nullable — prefer soft delete so audit history stays meaningful)
- `DeletionSource` — nullable enum, set only when `DeletedAt` is set:
  `ToolInitiated` (the normal delete flow — behaves exactly as before,
  simply disappears from the active list) or `DetectedInOkta` (the
  scheduled sync job found it missing from Okta — stays visible in the
  UI as a locked, view-only, clearly-labeled "deleted in Okta" record;
  see `09-drift-sync.md`).
- `LastSyncedAt` — nullable timestamp; updated by the scheduled sync job
  every time it checks this app against Okta, whether or not drift was
  found (lets the UI show "last verified against Okta at ...").
- Type-specific configuration: store as a structured JSON column
  (`ConfigurationJson`) rather than a wide table with many nullable
  OIDC-or-SAML-only columns — the two types' field sets differ enough that
  a shared rigid schema would be mostly null either way. Validate the JSON
  shape in the Application layer per `Type`. **Illustrative shape (not
  exhaustive — see `03-app-creation-flow.md` for the full field list per
  type, now full parity with Okta's own AIW reference for both types,
  `01-requirements.md` requirements 18 (SAML) and 19 (OIDC)):**
  ```json
  // Type = SAML — grouped to mirror the form's own General/Advanced split,
  // not a flat property bag, since several fields are only meaningful
  // together (e.g. assertionEncryption gates the three encryption fields)
  {
    "general": {
      "ssoUrl": "https://sp.example.com/sso/saml",
      "useSsoUrlForRecipientAndDestination": true,
      "recipientUrl": null,
      "destinationUrl": null,
      "audienceUri": "https://sp.example.com",
      "defaultRelayState": null,
      "nameIdFormat": "EmailAddress",
      "applicationUsernameFormat": "Email",
      "updateApplicationUsernameOn": null
    },
    "advanced": {
      "responseSigned": true,
      "assertionSigned": true,
      "signatureAlgorithm": "RSA_SHA256",
      "digestAlgorithm": "SHA256",
      "assertionEncryption": "Unencrypted",
      "encryptionAlgorithm": null,
      "keyTransportAlgorithm": null,
      "encryptionCertificate": null,
      "signatureCertificate": null,
      "singleLogoutEnabled": false,
      "singleLogoutUrl": null,
      "spIssuer": null,
      "signedRequests": false,
      "otherRequestableSsoUrls": [],
      "assertionInlineHookId": null,
      "authenticationContextClass": "PasswordProtectedTransport",
      "honorForceAuthn": true,
      "samlIssuerId": null,
      "maxSessionLifetimeMinutes": null,
      "sendSessionLifetimeInResponse": false,
      "attributeStatements": [
        { "name": "email", "nameFormat": "Basic", "value": "user.email" },
        { "name": "role", "nameFormat": "Basic", "filterType": "STARTS_WITH", "filterValue": "Team-" }
      ]
    }
  }
  ```
  ```json
  // Type = OIDC, Platform = Web — grouped the same way, since several
  // fields are only meaningful together (e.g. requireConsent gates the
  // three consent-screen fields; clientAuthMethod's valid values and the
  // grantTypes actually offered both depend on platform, per
  // `03-app-creation-flow.md`)
  {
    "platform": "Web",
    "general": {
      "notesForEndUsers": null,
      "notesForAdmins": null,
      "requireDPoP": false,
      "grantTypes": ["authorization_code", "refresh_token"],
      "refreshTokenRotation": "ROTATE_EVERY_USE",
      "refreshTokenPersistenceMinutes": null,
      "signInRedirectUris": ["https://app.example.com/callback"],
      "allowWildcardRedirect": false,
      "signOutRedirectUris": ["https://app.example.com/logout"],
      "clientAuthMethod": "ClientSecret",
      "pkceRequired": false
    },
    "login": {
      "loginInitiatedBy": "EITHER_OKTA_OR_APP",
      "applicationVisibility": true,
      "loginFlow": "OKTA_SIMPLIFIED",
      "initiateLoginUri": null
    },
    "consent": {
      "requireConsent": false,
      "termsOfServiceUri": null,
      "policyUri": null,
      "logoUri": null
    },
    "advanced": {
      "emailVerificationCallbackUri": null,
      "groupsClaimFilter": { "name": "groups", "filterType": "STARTS_WITH", "filterValue": "Team-" },
      "appRateLimitPercent": 50,
      "issuer": "OrgUrl",
      "trustedOriginBaseUris": [],
      "ciba": { "preferredAuthenticatorId": null }
    }
  }
  ```
  **Note: the App logo and App visibility fields (requirement 2's shared
  General Settings step) are deliberately absent from this JSON.** App
  visibility is a small enough app-level flag it could live as its own
  `Application` column rather than inside `ConfigurationJson` (open,
  minor, implementation-time call) — but the **App logo specifically is
  never stored here or anywhere in this database**, per the confirmed
  pass-through-only handling in requirement 18: it's forwarded to Okta's
  own logo-upload endpoint and not kept as a local copy. **Note the OIDC
  example's `login.applicationVisibility` and `consent.logoUri` are
  different fields from that shared App logo/visibility** — an
  OIDC-specific tile-visibility flag and a consent-screen branding URL,
  respectively, not the same setting under a different name (see
  requirement 19's explicit caution about this).
  The `attributeStatements` array remains the one field in this JSON blob
  documented in the most detail elsewhere (`01-requirements.md`
  requirement 17, `03-app-creation-flow.md`) because — unlike most other
  fields here — its internal shape (rows discriminated by `value` vs.
  `filterType`/`filterValue`) matters for validation, Promote's carry-over
  behavior, and the `IOktaAppService` mapping. `groupsClaimFilter` above
  is its OIDC counterpart and gets the same treatment for the same
  reason.
- `RowVersion` — an EF Core concurrency token (`[Timestamp]`/`rowversion`
  column), bumped on every write to this row, whether from a user's edit,
  the drift-sync job, or a deactivate/reactivate/assignment change.
  **Confirmed use: conflict-aware editing, not automatic
  last-write-wins.** The Edit form captures this value when opened and
  sends it back on submit; a mismatch means something else changed the
  app in the meantime (most likely the 15-minute drift-sync job, or a
  second concurrent edit) — the backend does not silently apply either
  side. Instead it returns a conflict response carrying the current
  `ConfigurationJson`/`RowVersion`, and the frontend asks the user to
  either load the latest version (their in-progress edit is discarded) or
  push their edit through anyway (an explicit override using the
  freshly-fetched `RowVersion`). See `03-app-creation-flow.md`. This is
  distinct from the still-open partial-failure question in
  `08-open-questions.md` (that one is about a *single* request's own
  Okta-write-then-local-write disagreeing with itself, not two
  independent writers disagreeing with each other).

### ApplicationAssignment (which people/groups are assigned to an app)

**Confirmed: persisted locally**, not fetched live from Okta on every
page view. Reasoning: regular Users have no direct Okta access at all —
every assignment change is a pass-through (User → this tool's backend →
Okta), so this tool's own DB is the practical place to read from for
display, the same way `ConfigurationJson` mirrors the rest of an app's
config rather than being re-fetched from Okta on every list render. Kept
in sync with Okta by the scheduled sync job (see below and
`09-drift-sync.md`), the same "Okta always wins on drift" pattern as
everything else that job reconciles.

- `Id`
- `ApplicationId` — FK to the assigned app
- `PrincipalType` — enum: `User` or `Group` (Okta has separate
  people-assignment and group-assignment concepts/APIs; this column
  distinguishes which one a given row represents)
- `OktaPrincipalId` — the Okta-side ID of the assigned person or group
  (an Okta user ID or Okta group ID, depending on `PrincipalType`) —
  **not** a FK to this tool's own `User` table; the assignable universe
  is Okta's entire people/group directory for that app's environment,
  which has no necessary relationship to who has an account in this tool
- `DisplayName` — cached label (person's name/email, or group name) so
  the assignments list doesn't need a live Okta call just to render;
  refreshed whenever the sync job touches this row
- `AssignedAt`
- `AssignedByUserId` — nullable FK to `User`; **null represents a
  system-initiated assignment change** (the sync job finding a
  directly-in-Okta assignment change), same null-means-system convention
  as `AuditLogEntry.UserId`
- `LastSyncedAt` — nullable timestamp, same purpose as
  `Application.LastSyncedAt`

**Confirmed rules:**
- Unique on (`ApplicationId`, `PrincipalType`, `OktaPrincipalId`) — an
  assignment either exists or it doesn't, no duplicate rows for the same
  person/group on the same app.
- Authorization mirrors Edit: Admin can manage assignments on any app; a
  `User` can manage assignments only on apps they own. Not
  approval-gated in any environment — same immediate shape as edit/
  deactivate/reactivate, only delete is gated (see `01-requirements.md`).
- Un-assignment is a hard delete of the row (not a soft delete) — there's
  no equivalent need for an assignment-history audit trail beyond what
  `AuditLogEntry` already separately records for the add/remove action
  itself.
- **Sync job also reconciles assignments per app** (per
  `09-drift-sync.md`): fetches the app's current people/group assignments
  from Okta (the `_links.users.href` / `_links.groups.href` hypermedia
  links Okta returns on the app resource) and diffs against local
  `ApplicationAssignment` rows for that `ApplicationId`. Any difference —
  an assignment added or removed directly in Okta rather than through
  this tool — is corrected locally (Okta wins) and logged as its own
  audit entry describing which principal was added/removed and that it
  was system-detected.

### AuditLogEntry
- `Id`
- `UserId` — nullable FK; who performed the action. **Null represents a
  system-initiated action** (currently only the scheduled sync job) —
  there is no synthetic "system user" row; the combination of a null
  `UserId` and a sync-specific `Action` value is what identifies it as
  system-initiated.
- `Action` — e.g. `AppCreated`, `AppEdited`, `AppDeleted`, `AppPromoted`,
  `RoleChanged`, `AppDeactivated`, `AppReactivated`,
  `AppDeletionRequested`, `AppDeletionApproved`, `AppDeletionRejected`,
  `AppDeletionCancelled`, `AssignmentAdded`, `AssignmentRemoved`,
  `AssignmentSyncedFromOkta` (system: an assignment was added or removed
  directly in Okta, detected and reconciled by the sync job),
  `AppSyncedFromOkta` (system: config or active/inactive-status drift
  auto-applied), `AppDeletedInOkta` (system: sync detected an external
  deletion), `ApprovalRequestAutoCancelled` (system: a pending Production
  deletion request was cancelled because the sync job found the app
  already gone), `OwnerReassigned` (Admin-only; `Details` holds the
  previous owner, the new owner, and the required reason —
  `01-requirements.md` requirement 15), `AppCredentialsViewed` (the one
  read that's still audited — someone viewed a Client Secret or SAML
  certificate on the Credentials tab, requirement 14)
- `ApplicationId` — nullable FK, when the action is app-scoped
- `SourceEnvironmentId`, `TargetEnvironmentId` — nullable, populated for
  promotion actions (always Staging → Production when both are set)
- `Result` — `Success` or `Failure` (+ an error summary if failed)
- `Details` — structured (JSON) supplementary info, e.g. which fields were
  remapped during a promotion
- `Timestamp`

### ApprovalRequest

Supports the delete-approval workflow (User-initiated deletes only —
Admin deletes bypass this entirely). Modeled as its own entity, rather than
a status flag on `Application`, so other actions could require approval
later without a schema change to `Application` — **currently only
`Delete` uses this** (an earlier draft also gated `Deactivate` through
this same table; that gating was reconsidered and removed — deactivation
is always immediate now, see requirement 8 in `01-requirements.md`). The
enum is kept multi-valued anyway in case a future action needs the same
mechanism.

- `Id`
- `ApplicationId` — FK to the affected app
- `RequestedAction` — enum; only `Delete` is used today, kept extensible
- `RequestedByUserId` — FK to `User`
- `RequestedAt`
- `Status` — `Pending`, `Approved`, `Rejected`, or `Cancelled`
- `ReviewedByUserId` — nullable FK to `User` (the approving/rejecting Admin)
- `ReviewedAt` — nullable
- `ReviewNote` — nullable, e.g. a rejection reason

An app's "pending deletion" state is derived by checking for an open
(`Status = Pending`) `ApprovalRequest` against it, rather than duplicating
that state as a column on `Application` — one source of truth.

**Confirmed rules:**
- An `ApprovalRequest` is only ever created when a **User** (non-Admin)
  deletes an app whose `Environment.Name = Production`. A User deleting a
  Staging app, and any Admin deletion regardless of environment, call the
  Okta delete directly and never touch this table. **Deactivating,
  reactivating, and editing never create an `ApprovalRequest`, in any
  environment, for either role** — only delete does. This check is a
  runtime Application-layer rule (read `Application.EnvironmentId` →
  `Environment.Name`), not a separate flag stored on `Application`.
- `ReviewedByUserId` may be set to **any** user with the Admin role — there
  is no separate "approver" role or reviewer allowlist in the MVP.
- **Confirmed precondition: delete requires `Application.IsActive =
  false` already.** Enforced independently of the Production-only
  approval gate above, and in every environment — an active app cannot be
  deleted (immediately or via a request) until it's deactivated first
  (deactivating itself is always immediate, so this doesn't add a second
  approval cycle — see requirement 8). Re-checked again when an Admin
  approves a pending delete request, not just when the request was
  created, in case the app was reactivated in between.
- **A `Status = Cancelled` request is not only ever self-withdrawn by the
  original requester.** The scheduled sync job (`09-drift-sync.md`) can
  also cancel a pending request, when it independently discovers the
  underlying app was already deleted directly in Okta. In that case
  `ReviewedByUserId` stays null (no human reviewed it) and `ReviewNote`
  holds a system-generated reason — the corresponding
  `ApprovalRequestAutoCancelled` audit entry (`UserId = null`) is what
  distinguishes this from a human's own withdrawal in the audit trail.

## Notes for the SaaS phase (do not implement now)

- A `TenantId` (or `AccountId`) would eventually be added to `User`,
  `Environment`, and `Application` to scope everything to a customer
  account. Keep this in mind when naming/shaping tables now so it's an
  additive column later, not a redesign — but do not add it prematurely.
- A `Subscription`/`Plan` entity would live alongside `Tenant` once billing
  is introduced — see the `payment-integration-engineer` agent and
  `07-roadmap.md`.
