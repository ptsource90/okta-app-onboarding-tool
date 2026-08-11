# Project instructions for Claude

## What this project is

An Okta app onboarding tool. Authenticated users (identity federated from a
third-party store — Firebase, Okta, generic SSO/OIDC) can create, edit, and
delete Okta applications (OIDC or SAML) in any configured environment, and
promote an app from Staging to Production (one-directional only — no other
environment pair). **Users are not scoped to specific environments** — any
User or Admin can act against any environment; the backend holds its own
OAuth 2.0 service app credential per environment (private key,
`private_key_jwt` — see `04-data-model.md` / `IOktaAppService`)
and performs every Okta operation itself, never delegated through the
caller's own Okta identity. Two roles exist:

- **Admin** — full visibility and control over all apps, all environments, all users' actions.
- **User** — can only see/manage the apps *they* created.

The tool logs user actions (audit trail) and caches Okta API responses.
This project is expected to grow into a multi-tenant SaaS product, so avoid
decisions that would make multi-tenancy or billing integration hard later.

**Current phase: planning.** Do not write application code in `src/` unless
explicitly asked to move into implementation — this repo is for architecture,
requirements, and agent setup right now. See `docs/` for the plan.

## Tech stack (do not substitute without discussion)

- Backend: ASP.NET Core Web API (.NET, latest LTS) — **Controller-based (MVC-style), not Minimal API**
- ORM: Entity Framework Core
- Database: MSSQL
- Identity/apps provider: Okta .NET SDK / Okta Management API
- Frontend: React + TypeScript + Vite
- UI components: an open-source React component library (see `docs/06-tech-stack.md` for the recommendation — do not hand-roll a component library)
- Logging: structured logging (e.g. Serilog) writing to both a table (for the in-app audit trail) and standard log sinks (for ops)
- Caching: in-memory cache for single instance; Redis-compatible distributed cache once the app runs on more than one instance (see `docs/02-architecture.md`)

## Non-negotiable engineering principles

1. **Clean Architecture / SOLID.** Domain logic has no dependency on EF Core, Okta SDK, or ASP.NET Core. Dependencies point inward. See `docs/02-architecture.md`.
2. **Less code, simple over clever.** Prefer a well-maintained open-source library over hand-rolled infrastructure (form handling, data grids, date pickers, HTTP retries, etc.). Only write custom code for the app's actual differentiator: the Okta app onboarding/promotion logic and the role-scoped access model.
3. **Security by default.** Every endpoint is authenticated; every mutating action is authorized against the caller's role *and* resource ownership (for the `User` role); every Okta API credential is a secret, never a literal in code or committed config. See `docs/05-security.md`.
4. **Every state-changing action is audited.** Create, delete, and promote operations must write an audit log entry (who, what, when, which environment, result) before returning success to the caller.
5. **Every CRUD action requires explicit user confirmation before it executes.** Create, edit, delete, deactivate, reactivate, assign, and unassign all show a confirmation step (a review screen for create/edit, a confirmation dialog for the rest) — none of them fire immediately on a single click. This is independent of, and in addition to, the Production approval-gate on delete: the gate controls *whether Okta gets called at all without an Admin*, confirmation controls *whether this browser tab's click was intentional*.
6. **Design for tenancy later, don't build it now.** Keep environment/org configuration and ownership modeled so a `TenantId` can be introduced later without a rewrite, but don't implement multi-tenancy in the MVP.

## Confirmed product decisions

- **Each environment is a separate Okta org** — not different auth servers
  within a shared org. Promoting an app or importing one always crosses
  org boundaries.
- **`Environment` is a real DB table, not config-driven at runtime — but
  it's seeded from `appsettings.json` on first database
  creation/migration.** Per-deployment values like the Okta org domain
  (e.g. `okta-production-domain`) live in an `appsettings.json` seed
  section rather than hard-coded migration data; after that initial seed,
  `Environment` rows are the runtime source of truth (see
  `04-data-model.md`). The `OktaServiceAppClientId` is seeded directly
  (not a secret); only the **environment-variable name** for each
  environment's private key goes in this seed config — never the
  key itself, and never a cloud secret-manager reference (see next
  bullet).
- **Production secrets are supplied via environment variables, not a
  cloud secret manager** (Key Vault/Secrets Manager) — a deliberate
  hosting-model decision, not a placeholder. Hosting itself is
  **Windows Server + IIS** (cloud provider — AWS or Azure — still open,
  see `docs/08-open-questions.md`); on IIS, set secrets at the
  **Application Pool level** (IIS Configuration Editor/`appcmd`), not
  inside `web.config`, since `web.config` ships as part of the deployed
  build artifact. See `docs/05-security.md` and `docs/06-tech-stack.md`.
- **Backend-to-Okta authentication (confirmed): OAuth 2.0 for Okta APIs
  (`client_credentials` grant, `private_key_jwt` client auth), not a
  static API token.** Each `Environment` has its own OAuth 2.0 service
  app (Client ID, non-secret, plain column) and private key (JWK, secret,
  environment-variable-referenced same as other secrets). **Confirmed
  SDK-native, not hand-rolled**: `Okta.Sdk` (current stable `10.x`)
  supports this directly via `Configuration { AuthorizationMode =
  AuthorizationMode.PrivateKey, ClientId, PrivateKey =
  new JsonWebKeyConfiguration(jwkJson), Scopes }` — the SDK signs the
  `client_assertion` JWT and requests the access token itself, no custom
  JWT-signing component needed (confirmed from `Okta.Sdk`'s own README).
  One `Configuration`/client instance per environment, built once and
  reused. Still flagged: whether the SDK reuses/refreshes the resulting
  token efficiently across calls on that same client instance, or
  re-authenticates every call — not spelled out in the README, confirm
  with `okta-expert`. **Unrelated to, and not to be confused
  with, the "service" app type explicitly excluded from the creation
  wizard** (`docs/01-requirements.md` requirement 19) — that's about apps
  this tool onboards for others; this is the credential this tool itself
  uses to talk to each org's Management API, provisioned once per
  environment outside the wizard entirely. See `docs/04-data-model.md`'s
  "Backend-to-Okta authentication" section, `docs/02-architecture.md`,
  and `docs/05-security.md`.
- **Existing Okta apps can be imported/adopted**, not just apps this tool
  creates. Admin first picks which environment to import from (any
  configured environment — not limited to Staging or Production
  specifically). Import is Admin-only for now (still a default to
  confirm — see `docs/08-open-questions.md`).
- **Apps have an Assignments tab (people & groups)**, persisted locally
  (`ApplicationAssignment`) since regular Users have no direct Okta
  access — assignment changes pass through this tool's backend to Okta,
  never straight from the browser. Autocomplete search: 2+ characters
  minimum (enforced server-side), top 5 matches, against that specific
  app's own environment's Okta directory (People and Groups are separate
  search fields — Okta's own search APIs are separate; not explicitly
  confirmed as the right UI, see `docs/08-open-questions.md`).
  Un-assignment is a hard delete of the row. Same ownership rule as Edit,
  never approval-gated. The sync job also reconciles assignments changed
  directly in Okta (`docs/09-drift-sync.md`).
- **Non-admin deletes are approval-gated in Production only.** An Admin
  deleting anything, and a User deleting their own Staging app, delete
  immediately. A User deleting their own Production app instead creates a
  pending `ApprovalRequest`; any Admin (no separate approver role) can
  approve or reject it. **Deactivating, reactivating, and editing are
  never gated, in any environment, for either role** — only delete is.
  (An earlier draft also gated a User's Production deactivation the same
  way; that was reconsidered and removed.) Full flow in
  `docs/03-app-creation-flow.md`.
- **Deletion requires the app to already be deactivated
  (`Application.IsActive = false`) — in every environment, independent of
  the Production approval gate above.** The delete action isn't
  reachable (hidden/disabled in the UI, and rejected server-side
  regardless) for an active app. Since deactivating is always immediate,
  this doesn't add a second approval cycle — a User only ever requests
  approval once (for the delete) to fully remove a Production app they
  own: deactivate immediately, then request delete. Re-checked again at
  approval time, not just at request time, in case the app was
  reactivated in between. Never auto-deactivates on the caller's
  behalf — see `docs/03-app-creation-flow.md`.
- **A user can create an app directly in any environment** (including
  Production) — creation itself is never approval-gated, only deletion
  and deactivation are.
- **Users are not scoped to specific environments.** There is no
  per-user environment-access model — every authenticated user (Admin or
  User) can target any configured `Environment`. What *is* scoped is
  ownership of `Application` records (a `User` only sees/acts on apps
  they created) and role (Admin vs. User). The backend's per-environment
  OAuth 2.0 service app credential (private key, `private_key_jwt`) is
  what actually talks to Okta; it is a service credential, not delegated
  user access, so there is nothing to restrict per user at the
  Okta-org level.
- **Editing is supported in every environment, for both roles, and is
  never approval-gated** — an Admin can edit any app; a User can edit only
  apps they own. Editing reuses the same type-specific (OIDC/SAML) fields
  as the creation form — full re-edit, not a restricted subset. See the
  "Edit an app" flow in `docs/03-app-creation-flow.md`.
- **Promotion only ever runs Staging → Production**, one direction only.
  There is no Production → Staging path and no path between any other
  environment pair, even once more environments exist (see
  `Environment.PromotesToEnvironmentId` in `docs/04-data-model.md`).
- **A scheduled per-environment job syncs config/deletion drift from Okta
  into the local DB** — one direction only (Okta → local; the reverse
  already happens synchronously via create/edit/delete, never queued for
  a schedule). Runs as a **Hangfire** recurring job, one per
  `Environment`, on a cron expression **read from `appsettings.json`**
  (default `*/15 * * * *`, every 15 minutes), using
  `Hangfire.SqlServer` for job storage in the same MSSQL database as EF
  Core. Config drift is applied automatically (Okta always wins, no
  confirmation needed — this is a background reconciliation, not a
  user action). Deletion drift soft-deletes the local record but keeps it
  visible as a locked, clearly-labeled "deleted in Okta" record —
  different from a delete made *through this tool*, which still just
  disappears from the active list as before. A pending Production
  deletion approval for an app found deleted in Okta is auto-cancelled
  (system reason logged). The job never auto-imports brand-new,
  never-tracked apps — that stays the manual Import flow. See
  `docs/09-drift-sync.md`.
- **A User can see the audit trail for apps they own** — not the global
  audit log. Scoped by `ApplicationId IN (apps they own)`, and includes
  every entry tied to that app regardless of actor: their own actions, an
  Admin's actions on their app, and drift-sync system entries. A User
  never sees entries with no `ApplicationId` (e.g. `RoleChanged`) or
  entries for apps they don't own. See `05-security.md`.
- **A Credentials tab shows the OIDC Client ID/Secret or SAML
  metadata/certificate an app owner needs**, styled after Okta's own
  Admin Console. Fetched live from Okta on every view — never cached,
  never stored in `ConfigurationJson`, never logged verbatim — and the
  view itself is audited (`AppCredentialsViewed`), the one read-only
  exception to "only mutations are audited." Same ownership/role
  authorization as viewing the app itself, no separate permission. See
  `docs/01-requirements.md` requirement 14, `docs/03-app-creation-flow.md`,
  `docs/05-security.md`.
- **An Admin can reassign an app's owner** (`Application.OwnerUserId`) —
  **Admin-only**, even for an app a User currently owns. Requires a
  non-empty reason and writes its own audit entry (`OwnerReassigned`,
  capturing previous owner/new owner/reason). No Okta call — ownership is
  a concept internal to this tool. See `docs/01-requirements.md`
  requirement 15, `docs/03-app-creation-flow.md`.
- **Editing is conflict-aware via `Application.RowVersion`** (an EF Core
  concurrency token). The Edit form captures it on open and resubmits it;
  a mismatch at submit time (drift-sync job or a concurrent edit changed
  the app meanwhile) is never silently resolved either way — the user is
  shown what changed and picks "load latest" (discard their edit) or
  "keep mine" (explicit override with the fresh `RowVersion`). See
  `docs/04-data-model.md`, `docs/03-app-creation-flow.md`.
- **Okta rejects duplicate app labels within an org** (confirmed against
  Okta's own API error reference — the earlier assumption that Okta
  allowed duplicates was wrong). Scoped per org, i.e. per `Environment`
  here. The creation wizard's Name field checks for a same-environment
  duplicate on blur and surfaces the conflicting app's Name/Type/Owner as
  an early warning — Okta's own rejection at submit time stays
  authoritative. See `docs/01-requirements.md` requirement 16,
  `docs/03-app-creation-flow.md`.
- **The SAML branch of the creation/edit form has full field parity with
  Okta's own AIW SAML reference**
  (https://help.okta.com/oie/en-us/content/topics/apps/aiw-saml-reference.htm)
  — not a curated subset. Mirrors Okta's own General/collapsed-Advanced-
  Settings split. A few explicit exceptions, all flagged rather than
  silently decided: the Early-Access "Logout" SLO configuration isn't
  built (the classic, GA Single Logout fields are used instead);
  Assertion Inline Hook selection is undecided pending a "list existing
  hooks" capability; and several dropdown enumerations need confirming
  against the actual Okta SDK rather than trusting this doc's drafted
  values. Enabling Signed Requests is a confirmed destructive toggle (it
  deletes any static Other Requestable SSO URLs on Okta's side) and must
  warn before submit, not silently apply. See `docs/01-requirements.md`
  requirements 17-18, `docs/03-app-creation-flow.md`, and
  `docs/08-open-questions.md`.
- **The OIDC branch has the same treatment: full field parity with
  Okta's own AIW OIDC guide**
  (https://help.okta.com/oie/en-us/content/topics/apps/apps_app_integration_wizard_oidc.htm),
  varying by platform (Web/SPA/Native) more than by a General/Advanced
  split. **Confirmed: the creatable platform list is Web/SPA/Native
  only — "service" (machine-to-machine) apps are explicitly excluded**,
  not just absent from Okta's own wizard; this tool doesn't support
  creating them via any path. Same flag-rather-than-silently-decide
  treatment for what isn't built: the Early-Access OIDC Logout/SLO
  section, Network IP (inconsistently labeled EA in Okta's own docs),
  CIBA's authenticator dependency, and client-secret rotation /
  Public-key-Private-key auth (flagged Phase 2).
  Two OIDC-specific naming traps to not conflate: the shared App logo
  (file upload, requirement 2) vs. the OIDC-only consent-screen Logo URI
  (a URL); and the shared App visibility (requirement 2) vs. the
  OIDC-only Application visibility tied to the Okta-tile login flow. See
  `docs/01-requirements.md` requirement 19, `docs/03-app-creation-flow.md`,
  and `docs/08-open-questions.md`.
- **The API is Controller-based (MVC-style `ControllerBase` + attribute
  routing).** Do not use Minimal API (`app.MapGet`/`MapPost`) anywhere in
  `OnboardingTool.Api` — this is a fixed convention, not a per-endpoint
  choice.
- **App listing and the audit log support search, filtering, and
  pagination**, via **Sieve** on the backend (`filters=`/`sorts=`/
  `page=`/`pageSize=` query params, allow-listed properties only, a
  server-enforced max `pageSize`) and TanStack Table on the frontend for
  table/pagination state, rendered with Mantine's own components. App
  list: text search on Name; filters for Environment, Type, Status
  (three values — Active [default], Deactivated, and "deleted in Okta";
  no "Pending Deletion"/"Pending Deactivation" values since those states
  are Production-specific and stay badges on the Active view instead of
  filters), and Owner (Admin-only). Audit log: text search on the app's
  Name; filters for Action, Environment, Result, date range, and
  Performed-by (Admin-only). Mandatory ownership/role scoping is always
  applied *before* the caller's own Sieve filters — a User's filters can
  never
  reach outside apps they own. The "unmanaged apps" import-scan list is
  not paginated/searchable for now. See `06-tech-stack.md`.

## App creation flow (source of truth)

1. User clicks **Create app**.
2. Wizard step 1: user picks the **target environment** (any configured environment — users are not scoped to a subset) — creating is not approval-gated in any environment, only deleting a Production app is (see "Confirmed product decisions" above).
3. Wizard step 2: user picks app type — **OIDC** or **SAML**.
4. Wizard routes to the form template for that type (OIDC fields vs SAML fields differ significantly — do not force them into one shared form).
5. On submit: validate → create in Okta via the Okta SDK (in the chosen environment's org) → persist app record (owner = current user, environment = chosen in step 2) → write audit log entry → invalidate the relevant app-list cache entry.

Full detail, including the Staging → Production promotion flow, is in `docs/03-app-creation-flow.md`.

## Edit an app (source of truth)

1. User opens an existing app they can see (Admin: any app; User: only
   apps they own).
2. Form is pre-populated from the app's current `ConfigurationJson`, using
   the same type-specific (OIDC/SAML) fields as the creation form — full
   re-edit, nothing locked post-creation.
3. On submit: validate → update the app in Okta via the Okta SDK (the
   app's own environment's org) → update the local `ConfigurationJson` →
   write an audit log entry (`AppEdited`, with a diff of changed fields) →
   invalidate the relevant app cache entry.
4. Never approval-gated, in any environment, for either role — unlike
   Production deletes.

Full detail is in `docs/03-app-creation-flow.md`.

## Scheduled Okta config sync (source of truth)

1. Runs as a Hangfire recurring job, once per `Environment` independently
   (each is a separate Okta org with its own rate limits), on a cron
   read from `appsettings.json` (defaulting to every 15 minutes).
2. Lists all apps in that environment's Okta org, compares against local
   `Application` rows for that environment (excluding already-deleted
   ones).
3. Missing from Okta → soft-delete locally with
   `DeletionSource = DetectedInOkta`; stays visible in the UI, locked
   (no edit/delete), clearly labeled as deleted in Okta. If a pending
   Production `ApprovalRequest` exists for that app, auto-cancel it
   (`Status = Cancelled`, system-generated `ReviewNote`,
   `ReviewedByUserId = null`) and log that cancellation separately.
4. Config or active/inactive status differs → overwrite local
   `ConfigurationJson`/`IsActive` to match Okta (no confirmation step —
   Okta always wins for this direction). Assignments are reconciled the
   same way, independently, per app.
5. Either change writes an `AuditLogEntry` with `UserId = null` (system
   actor) and a sync-specific `Action`; a no-op comparison just updates
   `LastSyncedAt`.
6. Never discovers/adopts brand-new apps that were never imported — that
   stays the manual, Admin-only Import flow.

Full detail is in `docs/09-drift-sync.md`.

## Session & token lifecycle (source of truth)

Scope: the tool's own login (Firebase/Okta/generic-OIDC per
`05-security.md`), **not** the per-environment Okta orgs the backend
manages apps in — those use a separate, backend-only service credential.

1. For an Okta-as-login-IdP deployment (Org Authorization Server,
   confirmed via Okta's own docs): access token = 60 minutes, refresh
   token = 90-day fixed max lifetime, no idle lifetime. This is
   Okta-specific — Firebase/generic-OIDC deployments would need their
   own numbers, not yet worked out.
2. The app's own tracked session length is **1 hour**, tied directly to
   the access token's natural lifetime — one clock, not a separate idle
   timer.
3. Silent renewal: any backend request that would use an expired/
   about-to-expire access token triggers a refresh-token exchange first,
   transparently.
4. A client-side countdown timer (independent of network activity) fires
   a modal **1 minute before** the current token's expiry — only
   noticeable during genuine idle time, since active use keeps
   refreshing and rescheduling this timer. Choices: **stay signed in**
   (explicit refresh, same mechanism as #3) or **log out** (revoke the
   refresh token, redirect to login) — the latter also happens
   automatically if the countdown reaches zero.
5. **Confirmed library: `@okta/okta-react`** (+ required peer
   `@okta/okta-auth-js`) — supersedes the earlier "suggested"
   `oidc-client-ts` pick. Okta-specific, not a generic OIDC client; the
   backend's own JWT/JWKS validation stays provider-agnostic regardless.
   **Must explicitly set `tokenManager: { storage: 'memory' }`** — the
   SDK defaults to `localStorage`, which conflicts with this project's
   in-memory-only decision above. **Confirmed: `react-router-dom` 5.x**,
   using the SDK's packaged `SecureRoute` directly — a deliberate choice
   to stay on 5.x for SDK compatibility (it only supports
   `react-router-dom` 5.x) rather than adopting 6.x+ and writing a custom
   protected-route component. Expect v5 routing APIs throughout
   (`<Switch>`, `component`/`render` props, `useHistory()`), not v6-style
   patterns.
6. Backend independently validates every request's access token via the
   login IdP's JWKS — never trusts the frontend's claim of identity.

Full detail, including flagged assumptions and open questions (token
storage mechanism, multi-tab behavior, proactive-vs-reactive renewal), is
in `docs/10-session-and-token-lifecycle.md`.

Defined in `.claude/agents/`: `software-engineer`, `okta-expert`,
`cybersecurity-engineer`, `devops-engineer`, `payment-integration-engineer`,
`test-engineer`. Prefer delegating Okta-specific questions to
`okta-expert`, security reviews to `cybersecurity-engineer`, pipeline/infra
questions to `devops-engineer`, future billing work to
`payment-integration-engineer`, and test-strategy/coverage questions to
`test-engineer`. Use `software-engineer` for general implementation,
architecture, and code review across both backend and frontend.

## Conventions once implementation starts

- Backend solution lives under `src/backend`, one project per Clean
  Architecture layer (`*.Domain`, `*.Application`, `*.Infrastructure`,
  `*.Api`), plus a test project per layer.
- Frontend app lives under `src/frontend`, scaffolded with Vite's
  `react-ts` template.
- No secrets in source control, ever — use `dotnet user-secrets` locally and
  **environment variables** in deployed environments (set at the IIS
  Application Pool level in production, not in `web.config` — see
  `docs/05-security.md`), not a cloud secret manager.
- Every PR-sized change should keep the Domain layer dependency-free and
  keep controllers thin (validation + orchestration only, no business logic).
- API endpoints are implemented as MVC-style Controllers (`ControllerBase`
  + attribute routing) — do not use Minimal API (`app.MapGet`/`MapPost`
  style) anywhere in `OnboardingTool.Api`.
