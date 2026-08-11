# Tech stack

As specified by the project brief, plus specific choices/rationale for the
open items (UI component library, caching, logging, forms).

## Backend

| Concern | Choice | Rationale |
|---|---|---|
| Web framework | ASP.NET Core Web API — **Controller-based (MVC-style), not Minimal API** | Required by brief; mature, first-class DI; controllers give a consistent, testable shape (attribute routing, model binding, filters) across all the resource endpoints this app needs |
| ORM | Entity Framework Core | Required by brief; migrations give a clean audit trail of schema change |
| Database | MSSQL | Required by brief |
| Okta integration | Okta .NET SDK (Management API) | Official SDK over hand-rolled HTTP client — less code, stays current with Okta API changes |
| Logging | Serilog | Mature, structured logging, wide sink support (file, console, Seq/App Insights/etc.) for the ops-log side of things |
| Caching | `IMemoryCache` (MVP) → Redis-compatible distributed cache (post-MVP/multi-instance) | Start simple; the `ICacheService` abstraction in Application means swapping implementation later doesn't touch use-case code |
| Validation | FluentValidation | Keeps validation rules declarative and out of controllers/handlers |
| CQRS-lite | Plain command/query handler classes; consider MediatR only if wiring becomes unwieldy | Avoid a dependency until it earns its keep |
| Scheduled jobs | **Hangfire** (`Hangfire.SqlServer` storage, same MSSQL database as EF Core) | Confirmed for the drift-sync job (`09-drift-sync.md`): a **recurring job**, `RecurringJob.AddOrUpdate`, on a **15-minute** interval by default, one registration per configured `Environment` so each environment's sync runs and fails independently. Built-in dashboard/retry/history over hand-rolling a `BackgroundService` + `PeriodicTimer` — worth the extra dependency here because retry-on-failure and an inspectable job history matter for something that silently reconciles data. Requires its own Hangfire schema in the MSSQL database (separate tables from the application's own EF Core migrations, created by Hangfire's own storage installer). |
| Configuration | ASP.NET Core's standard `appsettings.json` + `IOptions<T>` binding | Confirmed: all non-secret, environment-shaped or job-tunable settings (Okta org domains used to seed `Environment` on first migration, the drift-sync cron expression, etc.) are read from `appsettings.json` rather than hard-coded — see `04-data-model.md` and `09-drift-sync.md`. **Secrets never go here** — each environment's private key (JWK) stays in environment variables per `05-security.md`; `appsettings.json` only ever holds the environment-variable *name* (and the non-secret `OktaServiceAppClientId` directly), never the key value. |
| Search, filtering & pagination (backend) | **Sieve** (`Sieve.EntityFrameworkCore`) — supersedes the earlier X.PagedList-only pick | Now that search/filtering is in scope alongside pagination, Sieve (widely-used, MIT-licensed, built specifically for ASP.NET Core + EF Core) does all three from one query-string convention (`filters=`, `sorts=`, `page=`/`pageSize=`) against properties explicitly allow-listed with `[Sieve(CanFilter, CanSort)]` on each query DTO, rather than stacking a separate pagination library on top of hand-rolled filter predicates. Applies to the app list (`ListAppsQuery`) and the audit log (`ListAuditLogQuery`). **Security note (see `05-security.md`):** only explicitly allow-listed properties are filterable/sortable — Sieve does not expose arbitrary EF Core properties by default — and the mandatory ownership/role scope (`OwnerUserId`/`ApplicationId IN (...)`) is applied to the query *before* the caller-supplied Sieve filters, so a User can't use a filter to see past their own scope. |
| Concurrency control | EF Core concurrency token (`[Timestamp]`/`rowversion`) on `Application.RowVersion` | Backs the conflict-aware Edit flow (`01-requirements.md` requirement 4, `04-data-model.md`) — a stale write is caught server-side (`DbUpdateConcurrencyException` translated into a structured conflict response), not silently applied, and not a raw 500 either. |

## Hosting & deployment

| Concern | Choice | Rationale |
|---|---|---|
| Hosting model | **Windows Server + IIS** (confirmed) | ASP.NET Core Module (ANCM), **in-process** hosting model (the modern default — avoids the extra reverse-proxy hop out-of-process hosting requires). Requires the ASP.NET Core Hosting Bundle installed on the server. |
| Cloud provider | AWS or Azure — **not yet decided which** | See `08-open-questions.md`. Doesn't block backend design since the hosting model (Windows Server + IIS) and secrets approach (environment variables) are both already confirmed independent of which cloud runs the VM. Mainly affects `devops-engineer`'s choice of Redis-compatible service if/when Phase 2's distributed cache is needed. |
| Source control | GitHub | Confirmed. |
| CI/CD | **Assumed: GitHub Actions**, since GitHub already hosts the repo | Not separately, explicitly confirmed as the CI *runner* (only source control was) — treated as the natural default rather than introducing a second platform (Azure DevOps/GitLab CI/etc.) without a reason to. Flag to `devops-engineer` if a different CI platform is actually intended. |
| Secrets in production | **Environment variables** (confirmed), set at the **IIS Application Pool level**, not inside `web.config` | See `05-security.md` for the reasoning (keeps secrets off anything that gets built/packaged/checked in) and the IIS-specific mechanism. |

## Frontend

| Concern | Choice | Rationale |
|---|---|---|
| Build tool | Vite | Required by brief; fast dev server, simple config |
| Framework | React + TypeScript | Required by brief |
| Server state | TanStack Query (react-query) | Handles caching/loading/error state for API calls instead of hand-rolled hooks |
| Search, filtering & pagination (frontend) | **TanStack Table** (headless) for table/pagination/sort state, rendered with Mantine's own `Table`/`Pagination`/`TextInput`/`Select` components | TanStack Table is the widely-used, open-source (MIT) companion to TanStack Query already in this stack — headless, so it doesn't fight whichever UI kit renders the actual cells/pager (Mantine, per the recommendation below). A text search box plus a small set of dropdown filters (see `01-requirements.md`) map directly onto Sieve's `filters=` query param; page/sort state maps onto `page`/`pageSize`/`sorts=`. Used for both the app list and the audit log table. |
| Forms | React Hook Form + Zod | Schema-driven validation shareable in spirit with backend FluentValidation rules; well-suited to a multi-step wizard |
| Routing | **react-router-dom 5.x** (confirmed) | See "Identity" section below — 5.x is a deliberate choice to use the auth SDK's packaged `SecureRoute` directly rather than adopting 6.x+ and working around the incompatibility. |
| UI component library | See recommendation below | Required to be open-source per brief |

### UI component library recommendation

Two solid open-source options, either is a reasonable pick — flagging as an
explicit decision point rather than assuming:

- **Mantine** — batteries-included component set (forms, data tables,
  notifications, modals) with good TypeScript support; fastest path to a
  full admin-style UI (app lists, wizard, audit log table) with the least
  custom styling work.
- **shadcn/ui (Radix primitives + Tailwind)** — copy-in components you own
  and customize directly, built on accessible Radix primitives; more
  design control, slightly more assembly required for things like data
  tables (would pair with TanStack Table).

Given this is an internal admin-style tool where consistent, functional UI
matters more than bespoke visual design, **Mantine** is the lighter-weight
default recommendation — but this is easy to swap before implementation
starts and worth confirming with whoever owns the frontend look-and-feel.

## Identity (third-party, consumed not built)

- The app authenticates against whichever third-party store the deployment
  configures: Firebase Authentication, Okta (as an OIDC/SSO source, note:
  distinct from the Okta *tenant apps* being managed, which can even be a
  different Okta org), or a generic OIDC/SSO provider. The backend needs a
  standard OIDC/JWT bearer validation setup, not a provider-specific SDK,
  to keep this swappable. **This backend-level swappability is unaffected
  by the frontend library choice below** — JWT/JWKS validation doesn't
  care which SDK issued the token.
- **Confirmed frontend library: `@okta/okta-react`** (built on
  `@okta/okta-auth-js`, current stable major version `6.x`) — supersedes
  the earlier "suggested, not yet confirmed" `oidc-client-ts` pick.
  **Note: this is Okta-specific, not a generic OIDC client** — a
  Firebase or generic-OIDC deployment of this tool's login would need a
  separate frontend integration, not a config swap, unlike the backend's
  validation logic above. See `10-session-and-token-lifecycle.md` for
  the full session/token design (confirmed Okta-specific figures: 60-min
  access token, 90-day refresh token, both fixed on the Org Authorization
  Server), the token-storage override this SDK requires (defaults to
  `localStorage`, must be explicitly set to `memory`), and what the SDK
  provides vs. what's still custom-built (the countdown-modal UX).
- **Confirmed: `react-router-dom` 5.x**, using `@okta/okta-react`'s
  packaged `<SecureRoute>` component directly — a deliberate choice to
  stay on 5.x for SDK compatibility rather than adopting the more modern
  6.x+ and writing a custom protected-route component. Means v5 APIs
  throughout this codebase's routing code (`<Switch>`, `component`/
  `render` props, `useHistory()`), not v6-style patterns. See
  `10-session-and-token-lifecycle.md`'s "Routing" section for detail.
- **A third, distinct auth relationship, easy to conflate with the two
  above — confirmed: OAuth 2.0 for Okta APIs (`private_key_jwt`
  `client_credentials`), not a static API token.** The two bullets above
  are about *this tool's own login* (frontend: `@okta/okta-react`;
  backend: validating the resulting bearer token). This is about how
  *the backend itself authenticates outbound* to each per-environment
  Okta org's Management API — an unrelated credential, provisioned once
  per environment (not something an end user of this tool ever sees).
  See `04-data-model.md`'s "Backend-to-Okta authentication" section for
  the full setup/runtime flow, `02-architecture.md` for the new
  access-token caching component this requires, and `05-security.md` for
  the private-key handling rules.

## Testing strategy

**Confirmed (previously open — see `08-open-questions.md`): a hybrid of a
stub and a real sandbox, not either alone.**

| Layer | Approach | Rationale |
|---|---|---|
| Domain / Application unit tests | A hand-written **fake `IOktaAppService`** (in-memory, deterministic) | Fast, no network dependency, runs on every commit; exercises ownership/role rules, the delete-approval gate, and the new Edit `RowVersion`-conflict branch without needing a real Okta org. |
| Integration tests | Same fake `IOktaAppService`, real EF Core against a local/test MSSQL instance | Verifies repository queries (Sieve filters, ownership scoping) and EF Core concurrency-token behavior for real, without an external dependency. |
| End-to-end tests | A small, slower suite against a **real Okta developer/sandbox org** (one for OIDC-shaped apps, one for SAML-shaped apps) | Catches anything the fake doesn't faithfully model — actual Okta error codes (e.g. the duplicate-label rejection, `01-requirements.md` requirement 16), real credential shapes on the Credentials tab, and real rate-limit behavior. Kept small and deliberately not the default for everyday test runs, given Okta API rate limits and sandbox-org upkeep. **Each sandbox org needs its own OAuth 2.0 service app + private key** (`04-data-model.md`'s "Backend-to-Okta authentication" section) provisioned the same way as a real environment, not a shortcut static token — `test-engineer` and `devops-engineer` should coordinate on keeping test-sandbox credentials out of source control the same as production ones. |

A dedicated `test-engineer` subagent (`.claude/agents/test-engineer.md`)
owns keeping this split enforced as the codebase grows, and reviewing
that new features land tests at the right layer rather than only at the
slow end-to-end layer.
