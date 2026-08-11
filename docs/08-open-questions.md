# Open questions

Things intentionally left undecided rather than assumed. Each should be
resolved (or explicitly deferred to a named phase) before the relevant part
of the plan is implemented.

## Infrastructure / hosting

- **Cloud provider — partially resolved.** Hosting *model* is confirmed:
  **Windows Server + IIS**, regardless of which cloud it runs on
  (`06-tech-stack.md`). **Still open: AWS vs. Azure specifically.** This
  used to also decide the secret-store service (Key Vault vs. Secrets
  Manager) — that part is now moot, since secrets are confirmed to be
  environment variables, not a cloud secret manager (`05-security.md`).
  What's still genuinely blocked on this choice: the concrete
  Redis-compatible service `devops-engineer` would pick for Phase 2's
  distributed cache, and any cloud-specific VM/networking specifics.
- **CI/CD platform — resolved (assumed).** Source control is confirmed
  as **GitHub**; CI/CD is assumed to be **GitHub Actions** as the natural
  default given that, rather than introducing a second platform without a
  stated reason. This is an inference, not something explicitly
  reconfirmed as the CI *runner* specifically — flag to `devops-engineer`
  if a different CI/CD tool is actually intended.

## Scope decisions

- **Import authorization** — currently defaulted to Admin-only (`05-security.md`),
  since importing exposes the full org app inventory and assigns ownership
  on someone else's behalf. Alternative: let a User "self-claim" an
  existing app they know is theirs (no org-wide browsing, just paste an
  Okta app ID) without needing Admin involvement. Worth confirming which
  matches how imports will actually happen in practice.
- **Bulk actions** — the whole plan currently assumes one app at a time for
  create, delete, copy, and import. If onboarding will involve importing
  or copying many apps at once, bulk versions of these flows are a
  meaningfully different (and higher-risk, for copy/delete) design than
  looping the single-app flow client-side.
- **Approver role** — Phase 1 confirmed "any Admin" can approve a pending
  Production delete. `07-roadmap.md` already flags revisiting this with a
  dedicated Approver role if the Admin group grows large; worth a gut-check
  now if that's likely soon rather than later.

## Not yet designed

- **Notifications** — nothing currently pushes information to users; the
  audit log and in-app "pending approval" badge are pull-only. Worth
  deciding if email/Slack notification matters, especially for: a
  Production delete request needing review, and any action that fails.
- **Drift detection: fully resolved.** Design lives in `09-drift-sync.md`
  — a Hangfire recurring job, one per environment, every 15 minutes
  (interval read from `appsettings.json`), config drift auto-applied,
  deletion drift soft-deletes with a locked "deleted in Okta" view and
  auto-cancels any pending Production deletion approval for that app.
  Nothing left open here beyond the two items below.
- **Environment domain re-sync after initial seed** — an `Environment`
  row's `OktaOrgUrl` (and other seeded fields) is only populated from
  `appsettings.json` on first database creation/migration
  (`04-data-model.md`). If a domain changes afterward (e.g. an Okta org
  migration), whether updating it is a manual DB/admin action or should
  have a deliberate re-sync path is not decided.
- **Whether the drift-sync cron expression is configurable per deployment
  or hard-coded** — confirmed it's read from `appsettings.json`
  (`09-drift-sync.md`), not yet decided whether there's validation/a
  sensible fallback if the setting is missing or malformed.
- **Testing strategy: resolved.** Both, not either — a fake
  `IOktaAppService` for unit/integration tests (fast, deterministic, no
  external dependency) plus a smaller end-to-end suite against a real
  Okta developer/sandbox org (catches anything the fake doesn't
  faithfully model, e.g. actual Okta error codes like the duplicate-label
  rejection). See `06-tech-stack.md`'s new "Testing strategy" section.
  A dedicated `test-engineer` subagent (`.claude/agents/test-engineer.md`)
  owns keeping this split enforced as the codebase grows.
- **Partial-failure reconciliation** — create, edit, delete,
  deactivate/reactivate, and assign/unassign all call Okta and then write
  to the local database as two separate steps. If the Okta call succeeds
  but the local write fails (or vice versa), the local record and Okta
  silently disagree until something notices. `01-requirements.md`'s
  non-functional requirements say this "should reconcile visibly," but no
  concrete mechanism is chosen yet (e.g. a compensating action/saga, an
  outbox table, a periodic reconciliation job, or accepting it as a rare
  manual-fix case for MVP).
- **Whether deleting an app should auto-deactivate it first if it's still
  active: resolved.** Neither — deactivation is now a hard precondition
  for delete (`Application.IsActive = false` required, checked at both
  request and approval time). Deactivating itself is always immediate
  (never approval-gated), so this precondition never adds a second
  approval cycle. See `01-requirements.md`'s "Deletion approval workflow"
  and `03-app-creation-flow.md`.
- **People-vs-group assignment search UI: assumed, not confirmed** —
  `03-app-creation-flow.md`'s assignments flow defaults to two separate
  search fields/tabs (People and Groups), since Okta's own user-search
  and group-search are separate APIs. A single combined search box was
  considered but not confirmed either way.
- **Assignments on Promote: assumed, not confirmed** — the Promote flow
  (`03-app-creation-flow.md`, "Promote an app: Staging → Production
  only") lists which app-config fields carry over as-is vs. get
  remapped, but `ApplicationAssignment` rows (people/groups) aren't on
  either list. Step 5 creates the Production app via "the same
  type-specific creation path as a fresh creation," and a freshly
  created app always starts unassigned (see "Assignments during
  creation" earlier in the same doc) — so the working assumption is
  that a promoted app **also starts unassigned in Production**, with
  the owner expected to assign people/groups afterward via the
  Assignments tab, same as any new app. This isn't stated as an
  explicit rule anywhere (unlike, e.g., Attribute Statements' "Confirmed:
  carries over as-is on Promote" — `01-requirements.md` requirement 17),
  it's inferred from the field list's silence plus the shared creation
  path. Worth a explicit confirm-or-correct pass, partly because the
  reasoning has a real limit: `OktaPrincipalId` values are org-scoped,
  so even if someone *wanted* Staging assignments to carry over, the
  IDs wouldn't reliably resolve to the same people/groups in the
  Production org without a remapping step of their own — closer to the
  environment-specific fields (redirect URIs, certificates, etc.) than
  to the unconditional carry-over fields, if this is ever revisited.
- **Credential delivery to the app owner: resolved.** A Credentials tab,
  styled after Okta's own Admin Console, shows the OIDC Client ID/Secret
  or SAML metadata/certificate a person needs to actually configure their
  application — fetched live from Okta on each view, never cached or
  persisted, and logged as its own audited read (`AppCredentialsViewed`).
  See `01-requirements.md` requirement 14, `03-app-creation-flow.md`, and
  `05-security.md`.
- **Ownership continuity (an app's owner leaving/being reassigned):
  resolved.** An Admin can reassign `Application.OwnerUserId` to a
  different user, gated by a required reason and its own audit entry
  (`OwnerReassigned`). Admin-only — a `User` cannot reassign ownership of
  an app they own. See `01-requirements.md` requirement 15 and
  `03-app-creation-flow.md`.
- **Edit vs. drift-sync (or concurrent edit) race condition: resolved.**
  `Application.RowVersion` (an EF Core concurrency token) is captured
  when the Edit form opens and re-checked at submit time. A mismatch
  doesn't silently apply either side — the user is shown what changed
  and chooses "load latest" (discard their edit) or "keep mine" (an
  explicit override). See `04-data-model.md` and `03-app-creation-flow.md`.
- **SAML AIW-parity API surface: flagged for verification, not a design
  decision.** Several things `okta-expert` should confirm directly
  against current Okta behavior before the full-parity SAML form
  (`01-requirements.md` requirement 18) is built, rather than this repo
  asserting any of them:
  1. **Confirmed source:** Okta's own AIW SAML field reference
     (https://help.okta.com/oie/en-us/content/topics/apps/aiw-saml-reference.htm)
     states that with the Early Access **"Entitlement SAML Assertions and
     OIDC Claims"** feature enabled on an org, Attribute Statements and
     Group Attribute Statements only appear when *editing* an app, not
     during creation. Confirm whether per-org EA status is detectable via
     the API, and if not, decide whether every SAML creation should
     assume a follow-up Edit call may be needed for these two sections
     rather than trusting they were set at creation time.
  2. A known Okta .NET SDK issue: updating `attributeStatements` via
     `ReplaceApplicationAsync` can report success while silently
     applying no change unless `destinationOverride` is explicitly
     included (even as `null`). Relevant to `EditAppCommand`'s SAML
     branch (`02-architecture.md`) if the .NET SDK ends up being the
     `IOktaAppService` implementation.
  3. Exact dropdown enumerations for Name ID format, Application
     username format, Signature/Digest Algorithm, Encryption/Key
     Transport Algorithm, and Authentication context class — drafted in
     `03-app-creation-flow.md` from stable SAML 2.0/Okta convention, but
     not verified against the current Okta SDK's actual enum strings.
  4. Whether **Assertion Inline Hook** is worth building at all for MVP —
     it depends on Inline Hooks already existing in the org (hook
     creation/management is out of scope for this tool), so it would need
     at minimum a new read-only "list existing hooks" capability for one
     narrow, likely-rarely-used field.
  5. The "Logout" section's Early-Access user-/app-initiated SLO
     configuration is deliberately **not** being built — this tool
     implements the classic, GA Enable Single Logout/Single Logout
     URL/SP Issuer fields instead. Revisit if/when the EA feature
     reaches general availability.
  See `01-requirements.md` requirements 17-18 and `03-app-creation-flow.md`
  for the (otherwise resolved) design this sits alongside.
- **OIDC AIW-parity API surface: flagged for verification, not a design
  decision.** Several things `okta-expert` should confirm directly
  against current Okta behavior before the full-parity OIDC form
  (`01-requirements.md` requirement 19) is built. (The platform list —
  Web/SPA/Native only, "service" explicitly excluded — was flagged here
  previously and is now a confirmed decision, not an open item; see
  requirement 19.)
  1. **Network IP** (network-zone token restriction) is marked Early
     Access in some parts of Okta's own reference and not explicitly
     marked in others — an inconsistency in Okta's own docs, not
     resolved here. Confirm actual current status before deciding
     whether to build it or exclude it like the Logout section below.
  2. CIBA's Preferred-authenticator field depends on authenticators
     already configured in the org (authenticator management is out of
     scope for this tool) — needs at minimum a read-only "list existing
     authenticators" capability for one narrow, likely CIBA-only field;
     not yet decided as worth building.
  3. Client secret rotation (a second, independently-activatable secret)
     and Public key/Private key as an alternative to a client secret
     entirely are both real Okta capabilities not designed here — the
     Credentials tab (requirement 14) currently assumes exactly one
     active secret. Flagged as likely Phase 2 given the added complexity.
  4. The dedicated OIDC "Logout" (Single Logout) section is Early Access,
     same treatment as SAML's excluded EA Logout section — this tool
     implements the classic, GA Sign-out redirect URIs field instead.
  See `01-requirements.md` requirement 19 and `03-app-creation-flow.md`
  for the (otherwise resolved) design this sits alongside.
- **Backend-to-Okta authentication: fully resolved, including the SDK
  support question.** OAuth 2.0 for Okta APIs (`private_key_jwt`/
  `client_credentials`, replacing a static API token — see
  `04-data-model.md`) is confirmed **SDK-native**: `Okta.Sdk` (current
  stable `10.x`) supports `private_key_jwt` directly via
  `Configuration { AuthorizationMode = AuthorizationMode.PrivateKey,
  ClientId, PrivateKey, Scopes }`, confirmed from the SDK's own README
  (https://github.com/okta/okta-sdk-dotnet/blob/master/README.md#oauth-20)
  — no hand-rolled JWT-signing or token-endpoint call needed. One
  narrower thing still flagged: whether the SDK reuses/refreshes the
  resulting access token efficiently across multiple calls on the same
  long-lived client instance, since that's not spelled out in the README
  excerpt — confirm with `okta-expert` before assuming either way. See
  `02-architecture.md`'s "Backend-to-Okta authentication" section.
- **Okta's own rate-limit (429) responses to this app's outbound Okta
  API calls: substantially covered by the SDK's built-in retry, not a
  from-scratch build.** `Okta.Sdk` ships a configurable built-in retry
  strategy for 429s (`Configuration.RequestTimeout`,
  `Configuration.MaxRetries`), plus a Polly-based custom-retry option if
  more control is ever needed — confirmed from the SDK's own README.
  Distinct from this app rate-limiting its *own* endpoints
  (`05-security.md`, already designed). What's left, not fully resolved:
  confirming the default `MaxRetries`/`RequestTimeout` values are
  actually appropriate for the drift-sync job's `ListAllAsync` and the
  import scan (the highest-volume Okta callers), rather than accepting
  SDK defaults untested — a much smaller task than the "explicitly
  backlogged, not yet designed" status this item had before.
- **Duplicate app names: resolved (and a prior assumption corrected).**
  Okta's own API does **not** allow a duplicate app label within an org
  (confirmed against Okta's API error reference:
  `Application label must not be the same as an existing application
  label`) — scoped per org, i.e. per `Environment` here. Rather than only
  discovering that at submit time, the wizard's Name field checks for a
  same-environment duplicate on blur and surfaces the conflicting app's
  Name/Type/Owner; Okta's own rejection at submit time stays the
  authoritative check regardless. See `01-requirements.md` requirement 16
  and `03-app-creation-flow.md`.
