# Architecture

## Guiding principles

Clean Architecture + SOLID, applied pragmatically: enough separation to keep
the domain testable and the Okta/EF Core dependencies swappable, without
over-engineering a small MVP. Favor open-source libraries over custom
infrastructure. Dependencies point inward.

## Backend — Clean Architecture layers

```
src/backend/
├── OnboardingTool.Domain/         # Entities, value objects, domain rules. No dependencies on anything below.
├── OnboardingTool.Application/    # Use cases (e.g. CreateAppCommand, PromoteAppCommand), interfaces for
│                                  #   infrastructure (IOktaAppService, IAppRepository, ICacheService,
│                                  #   IAuditLogger), DTOs, validation.
├── OnboardingTool.Infrastructure/ # EF Core DbContext + migrations, Okta SDK client implementing
│                                  #   IOktaAppService, cache implementation, audit log writer,
│                                  #   Hangfire.SqlServer storage config.
└── OnboardingTool.Api/            # ASP.NET Core Web API: MVC-style Controllers (not Minimal API), auth
                                   #   middleware, DI composition root, request/response models,
                                   #   Hangfire server + recurring-job registration (one per Environment).
```

- **Domain** — `Application` (Okta app record), `Environment`, `Role`,
  `ApprovalRequest`, ownership rules (e.g. "a User-role owner can only act
  on their own apps") and the gating rule ("a delete requires an
  `ApprovalRequest` only when the caller is a User and the app's
  environment is Production — every other combination deletes
  immediately"). No EF Core, no Okta SDK, no ASP.NET Core references here.
- **Application** — orchestrates use cases. In addition to
  `CreateAppCommand` and `PromoteAppCommand` (Staging → Production only —
  the command validates this direction before doing anything else):
  - `ListAppsQuery` — Admin: every `Application` row (not soft-deleted,
    or including the locked "deleted in Okta" ones per UI needs). User:
    same query, scoped to `WHERE OwnerUserId = currentUser.Id`. That
    ownership scope is applied first; Sieve then layers the caller's
    search/filter/sort/page on top (Name search, Environment/Type/Status
    filters, Owner filter for Admin) and returns a Sieve-paginated
    result — see `06-tech-stack.md`.
  - `EditAppCommand` — Admin: any app; User: only apps they own. Never
    approval-gated, in any environment. Takes the `RowVersion` the
    frontend loaded the edit form with; if it no longer matches the
    current row, the handler does **not** call
    `IOktaAppService.UpdateAsync` at all — it short-circuits with a
    conflict result carrying the current `ConfigurationJson`/`RowVersion`
    (translating what would otherwise be a raw EF Core
    `DbUpdateConcurrencyException` into a structured response the API
    layer maps to 409). Only on a matching `RowVersion` does it proceed:
    call `IOktaAppService.UpdateAsync` (type-specific) for the app's own
    environment, then update the local `ConfigurationJson`/`RowVersion`
    and write an `AppEdited` audit entry with a diff of changed fields.
    See `04-data-model.md` and `03-app-creation-flow.md` for the
    "load latest vs. keep mine" frontend choice this enables.
    **Implementation flag for SAML edits specifically:** a known Okta
    .NET SDK issue means an `attributeStatements` update via
    `ReplaceApplicationAsync` can report success while silently applying
    no change unless `destinationOverride` is explicitly included (even
    as `null`) — verify this before trusting a 200 response from
    `UpdateAsync` as proof the write landed. See
    `03-app-creation-flow.md` and `08-open-questions.md`.
  - `CheckAppNameAvailabilityQuery` — takes a `Name` and `EnvironmentId`;
    checks the local `Application` table for an existing
    non-soft-deleted row with the same name in that environment. Backs
    the wizard's `onBlur` duplicate-name pre-check (`01-requirements.md`
    requirement 16); returns the conflicting app's Name/Type/Owner when
    found. Deliberately **not** the enforcement point — Okta's own
    rejection at submit time (`Application label must not be the same as
    an existing application label`) remains authoritative, since this
    query only reflects data as fresh as the last drift-sync cycle.
  - `GetApplicationCredentialsQuery` — Admin: any app; User: only apps
    they own — same ownership rule as `EditAppCommand`. Calls
    `IOktaAppService.GetCredentialsAsync` (a new interface member: OIDC
    returns Client ID + Client Secret, SAML returns SSO URL/Issuer/
    Audience + signing certificate) live against Okta, every call —
    never reads or writes `ConfigurationJson`, never goes through
    `ICacheService`. Writes an `AppCredentialsViewed` audit entry before
    returning the result (`01-requirements.md` requirement 14,
    `05-security.md`).
  - `ReassignOwnerCommand` — **Admin-only**, including for an app a
    `User` currently owns. Takes a `Reason` and rejects the request
    server-side if it's empty. Updates `Application.OwnerUserId` only —
    no Okta call, since ownership is a concept internal to this tool.
    Writes an `OwnerReassigned` audit entry (previous owner, new owner,
    reason) and invalidates the cached app list/detail entry for *both*
    the previous and new owner, since a `User`'s list view is filtered
    by `OwnerUserId` (`01-requirements.md` requirement 15).
  - `SearchOktaPrincipalsQuery` — Admin: any app; User: only apps they
    own. Takes a `PrincipalType` (User/Group) and a search term (≥2
    characters, enforced server-side too, not just in the frontend); calls
    the corresponding Okta user-search or group-search API against the
    app's own environment's org (using that environment's service
    credential); returns the top 5 matches. Not persisted, not cached —
    this is a live typeahead against Okta, not something that goes stale
    slowly the way app config does.
  - `AssignPrincipalCommand` / `UnassignPrincipalCommand` — same
    ownership rule as `EditAppCommand`; never approval-gated, in any
    environment. Calls the Okta SDK's assign/unassign API for the given
    app, then inserts/deletes the corresponding `ApplicationAssignment`
    row and writes an `AssignmentAdded`/`AssignmentRemoved` audit entry.
    Unassign is a hard delete of the row (see `04-data-model.md`).
  - `ListUnmanagedAppsQuery` — calls `IOktaAppService.ListAllAsync` for an
    environment's Okta org and filters out apps that already have a local
    `Application` record, to power the import screen. Not paginated for
    now — bounded by how many apps a single Okta org realistically has
    and how Okta's own list API already pages results; revisit only if
    that assumption breaks down.
  - `ImportAppCommand` — Admin-only; reads full config for a chosen Okta
    app, creates a local `Application` record with `Origin =
    ImportedFromOkta` and the assigned owner.
  - `DeleteAppCommand` — used for the two immediate-delete paths (Admin,
    or a User deleting their own Staging app): first validates
    `Application.IsActive = false` (rejects otherwise — deactivation is a
    hard precondition, checked here regardless of the UI hiding the
    button), then calls `IOktaAppService.DeleteAsync` directly.
  - `RequestDeleteCommand` — used when a User deletes their own Production
    app: same `IsActive = false` precondition check as `DeleteAppCommand`,
    then creates an `ApprovalRequest` (`RequestedAction = Delete`) instead
    of calling Okta.
  - `DeactivateAppCommand` / `ReactivateAppCommand` — Admin: any app;
    User: only apps they own. **Neither is ever approval-gated, in any
    environment, for either role** — both call
    `IOktaAppService.DeactivateAsync`/`ActivateAsync` directly and set
    `Application.IsActive` accordingly, the same immediate shape as
    `EditAppCommand`. (An earlier draft gated `DeactivateAppCommand` the
    same way as delete for a User's Production app; that was reconsidered
    and removed.)
  - `ApproveDeleteCommand` / `RejectDeleteCommand` — Admin-only; the
    approve path **re-verifies `IsActive = false`** (the app may have
    been reactivated since the request was created) before calling
    `IOktaAppService.DeleteAsync` and soft-deleting the local record.
    Rejecting leaves the app untouched (still deactivated, not deleted)
    and records a reason.
  - `CancelDeleteRequestCommand` — the original requester withdrawing
    their own pending request.
  - `ListAuditLogQuery` — Admin: every `AuditLogEntry`, unfiltered. User:
    same query, but the Application layer enforces `WHERE ApplicationId
    IN (Application rows the caller owns)` before returning results — a
    User calling this never sees the global log or another user's app
    history (see `05-security.md`). That mandatory scope is applied
    before Sieve layers on the caller's own search/filter/sort/page
    (app-name search, Action/Environment/Result/date-range filters,
    Performed-by filter for Admin) — this is the view most likely to
    grow large, being append-only.
  - `SyncEnvironmentConfigurationCommand` — invoked by a Hangfire
    recurring job (`RecurringJob.AddOrUpdate`, cron read from
    `appsettings.json`, defaulting to `*/15 * * * *`), one registration
    per `Environment`, registered in `OnboardingTool.Api`'s composition
    root. Not triggered by a user. Calls
    `IOktaAppService.ListAllAsync` and compares against locally tracked
    `Application` rows for that environment, updating `ConfigurationJson`
    (config drift) or soft-deleting with `DeletionSource =
    DetectedInOkta` (deletion drift), auto-cancelling any pending
    Production `ApprovalRequest` for an app found deleted in Okta. Writes
    `AuditLogEntry` rows with `UserId = null`. See `09-drift-sync.md`.

  All of the above depend only on Domain and their own interfaces
  (`IOktaAppService`, `IAppRepository`, `IApprovalRequestRepository`,
  `ICacheService`, `IAuditLogger`) — the actual Okta SDK and EF Core are
  injected via those interfaces, not referenced directly.
- **Infrastructure** — implements the interfaces: EF Core repository/
  DbContext for MSSQL, Okta .NET SDK (`Okta.Sdk`) wrapper (one
  implementation per app type or a shared client with type-specific
  request builders — needs a `ListAllAsync` capability per
  environment/org for the import flow, a `GetCredentialsAsync`
  capability for the Credentials tab (`GetApplicationCredentialsQuery`
  above), an `UploadLogoAsync` capability for the shared General
  Settings App logo field (`01-requirements.md` requirement 18 — a
  separate Okta API call from app creation itself, not an inline field
  on the create request), and, flagged rather than committed to, a
  possible `ListInlineHooksAsync` read-only capability if Assertion
  Inline Hook selection is built, and the same possible-not-committed
  treatment for a `ListAuthenticatorsAsync` read-only capability if
  CIBA's Preferred-authenticator selection is built (`01-requirements.md`
  requirement 19, `08-open-questions.md`), in addition to
  `CreateAsync`/`DeleteAsync`). **Authentication to each environment's
  Okta org is SDK-native, not a custom component** — a per-environment
  `Configuration` (`AuthorizationMode.PrivateKey`, `ClientId`,
  `PrivateKey`, `Scopes`) constructed once and reused, per "Backend-to-
  Okta authentication" below; no separate token-provider interface is
  needed. Cache provider (in-memory now, swappable for Redis).
- **Api** — **Controller-based** (not Minimal API) thin controllers that map
  HTTP requests to Application commands/queries and map results back to
  HTTP responses; owns authentication (validating the third-party IdP
  token) and coarse authorization (role checks); fine-grained ownership
  checks live in the Application layer alongside the use case they
  protect. One controller per resource (e.g. `AppsController`,
  `EnvironmentsController`, `ApprovalRequestsController`), consistent
  attribute routing, `[Authorize]` at the controller or action level.

A CQRS-lite pattern (simple command/query handler classes, not necessarily
a full mediator library) is a reasonable fit for the Application layer given
the mix of reads (app lists, filtered by role) and writes (create/delete/
copy) — introduce a library like MediatR only if the number of use cases
makes the wiring genuinely annoying, not by default.

## Frontend

```
src/frontend/
├── src/
│   ├── api/            # Typed API client (generated or hand-written thin wrapper), react-query hooks
│   ├── features/
│   │   └── app-wizard/ # Multi-step create-app wizard: environment selection, type selection, OIDC form, SAML form, review
│   ├── components/     # Shared/reusable UI built on the chosen component library
│   ├── routes/          # Route-level pages (App list, App detail with its own
│   │                    #   audit history, Admin global audit log, etc.)
│   └── auth/            # Third-party IdP session/token handling
```

- Vite + React + TypeScript.
- Server state via a data-fetching library (e.g. TanStack Query) rather
  than hand-rolled fetch/loading-state management.
- Forms (especially the OIDC/SAML wizard steps, which have real
  validation needs — redirect URI formats, certificate formats, etc.) via a
  form library with schema validation (e.g. React Hook Form + Zod) rather
  than custom validation logic.
- UI components from an open-source component library (candidates and
  rationale in `06-tech-stack.md`) rather than building buttons/dialogs/
  data grids from scratch.
- Role-aware rendering: the frontend still hides Admin-only views from
  Users, but this is a UX convenience — the backend is the actual
  authorization boundary and must not trust the frontend's role gating.

## Cross-cutting concerns

- **Logging** — structured logging (e.g. Serilog) with two destinations:
  (1) an application-level audit log table for user-facing actions (create/
  delete/copy), queried by the Admin's global audit view and, scoped by
  ownership, by a User's own per-app audit history (see `05-security.md`);
  (2) a standard ops log
  sink for diagnostics, request tracing, and errors. These are different
  concerns with different retention/access rules — don't merge them into
  one mechanism.
- **Caching** — `ICacheService` abstraction in Application; `IMemoryCache`
  implementation for a single-instance MVP; Redis-backed implementation
  when the app is horizontally scaled (see `devops-engineer` agent).
  Cache keys are scoped by environment and, where results are role-
  filtered, by the requesting user/role to avoid leaking data across the
  ownership boundary.
- **Validation** — input validation both client-side (fast feedback in the
  wizard) and server-side (source of truth) using the same schema
  definitions where practical (e.g. shared constraints documented once,
  implemented with FluentValidation server-side and Zod client-side).
  **Conditional-field validation is server-side, not just
  frontend-enforced:** the SAML form's many "shown only if X" fields
  (`01-requirements.md` requirement 18 — e.g. Encryption Algorithm only
  valid when Assertion Encryption is Encrypted, Update-username-on only
  valid when Application username format is Custom) are re-checked as a
  group in FluentValidation, not trusted to have been correctly hidden
  by the frontend's conditional rendering. **Same principle for OIDC's
  platform-dependent grant types** (requirement 19) — a grant type valid
  for Web but not SPA must be rejected server-side if submitted for an
  SPA app, not just excluded from that platform's dropdown; likewise
  DPoP-with-Implicit-grant-type is a server-side rejection, not only a
  disabled checkbox.

## Data flow — create app (illustrative)

```
React wizard --submit--> API controller --> Application.CreateAppCommandHandler
   --> IOktaAppService.CreateAsync (Infrastructure: Okta SDK) --Okta app created-->
   --> IAppRepository.AddAsync (Infrastructure: EF Core/MSSQL)
   --> IAuditLogger.LogAsync (Infrastructure)
   --> ICacheService.Invalidate(appsListCacheKey)
   --> 201 Created response (includes the new app's ID)
   --> React invalidates the list (react-query) AND immediately navigates
       to the new app's detail page with the Assignments tab active —
       **confirmed: not left on the wizard or the list**, since assigning
       people/groups right after creation is the expected next action;
       see `01-requirements.md` requirement 2 and
       `03-app-creation-flow.md`'s "App assignments" section for the
       entry-point detail
```

The import flow and the delete flow's Production-approval branch follow the
same shape (Application-layer command → `IOktaAppService`/`IAppRepository`/
`IAuditLogger`/`ICacheService`) — see `03-app-creation-flow.md` for the full
diagrams rather than duplicating them here.

## Note on Okta org topology

**Confirmed:** each `Environment` maps to a fully separate Okta org (not
different auth servers within one shared org). Practically this means:
`IOktaAppService` implementations are always constructed per-environment
(their own base URL + OAuth 2.0 service app credential, below), a copy or
import always crosses org boundaries, and Okta API rate limits are
naturally isolated per environment rather than shared across Staging and
Production.

## Backend-to-Okta authentication (confirmed: SDK-native, not hand-rolled)

**Confirmed:** the backend authenticates to each environment's Okta org
via OAuth 2.0 for Okta APIs (`client_credentials` grant, `private_key_jwt`
client auth) — see `04-data-model.md`'s "Backend-to-Okta authentication"
section for the full setup flow and `05-security.md` for the
secret-handling rules. This replaces a static-API-token model.

**Resolves the earlier "hand-roll vs. SDK-native" flag: the official
`Okta.Sdk` NuGet package (current stable major version `10.x`) supports
this natively**, confirmed from its own README
(https://github.com/okta/okta-sdk-dotnet/blob/master/README.md#oauth-20)
— no custom JWT-assertion signing or token-endpoint call needs to be
hand-written:

```csharp
var config = new Configuration
{
    OktaDomain = environment.OktaOrgUrl,
    AuthorizationMode = AuthorizationMode.PrivateKey,
    ClientId = environment.OktaServiceAppClientId,
    Scopes = new List<string> { "okta.apps.manage", "okta.apps.read",
                                 "okta.groups.read", "okta.users.read" },
    PrivateKey = new JsonWebKeyConfiguration(privateKeyJsonFromEnvVar)
};
var applicationApi = new ApplicationApi(config);
```

`JsonWebKeyConfiguration` accepts the raw JWK JSON string directly (the
same JSON this tool's environment variable holds, per
`04-data-model.md`) — no manual parsing into individual JWK fields
needed on this project's side. **"When using this approach you won't
need an API Token because the SDK will request an access token for
you"** (Okta's own words) — the SDK constructs and signs the
`client_assertion` JWT and calls the token endpoint internally.

- **One `Configuration`/API-client instance per `Environment`**,
  constructed once (e.g. at DI-registration/startup time, keyed by
  environment) and reused across requests — not reconstructed per call.
  `IOktaAppService` implementations for a given environment hold onto
  their own long-lived client instance rather than rebuilding one per
  operation.
- **Flagged, not resolved by the README excerpt above:** whether the SDK
  caches and reuses the access token internally across multiple calls
  made through the *same* long-lived client instance (efficient), or
  re-authenticates on every call (wasteful, and would matter for Okta
  rate limits) — the README confirms token acquisition is automatic but
  doesn't spell out the reuse/refresh behavior across calls. Confirm with
  `okta-expert` before assuming either way; this affects whether any
  additional caching wrapper is needed on top of the SDK's own client, or
  whether the SDK already handles it end to end.
- **Built-in 429 retry, relevant to the previously-backlogged rate-limit
  item (`08-open-questions.md`):** the SDK ships a configurable built-in
  retry strategy for 429 responses (`RequestTimeout`, `MaxRetries` on
  `Configuration`), plus a Polly-based custom-retry escape hatch if more
  control is ever needed. This substantially covers what was previously
  flagged as unbuilt — see `08-open-questions.md` for the updated status
  (confirm the default `MaxRetries`/`RequestTimeout` values are
  appropriate rather than treating this as a from-scratch build).
- **DPoP note, unrelated to but easy to conflate with the DPoP field
  already documented for onboarded OIDC apps** (`01-requirements.md`
  requirement 19): starting from the SDK's `8.x` series, if *this tool's
  own* service app has DPoP enabled, the SDK transparently obtains a
  DPoP-bound token and generates a proof JWT per request with no extra
  configuration. Two entirely separate DPoP contexts — one about apps
  this tool onboards for others, one about this tool's own credential —
  don't merge them in code or docs.
- **Not the same cache as `ICacheService`** (which caches Okta *app data*
  for the frontend, per the caching design elsewhere in this doc) — to
  the extent any token caching is needed beyond what the SDK's client
  instance already does internally (see the flagged item above), it
  would be a distinct, backend-internal concern, not routed through
  `ICacheService`.
- **Failure handling:** an authentication failure against a given
  environment (expired/rotated-out key, revoked scope, token-endpoint
  outage) should fail loudly for that environment's calls, not silently
  retry into a degraded state — coordinate with `devops-engineer` on
  alerting for SDK-level auth exceptions specifically, since this is a
  new failure mode this project didn't have under the static-token model.
