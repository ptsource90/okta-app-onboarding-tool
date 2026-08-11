# Okta App Onboarding Tool

An internal tool that lets authenticated users (identity from a third-party
store — Firebase, Okta, generic SSO/OIDC) create, edit, delete, import, and
promote Okta applications (OIDC or SAML). Users are not scoped to specific
environments — any user can act against any configured environment; the
backend holds its own OAuth 2.0 service app credential (private key,
`private_key_jwt`) per environment and performs all Okta
operations itself, never on behalf of the user's own Okta identity.
Promotion, however, only ever runs Staging → Production (one direction, no
other environment pair). Each environment is its own Okta org. Visibility/
control is role-based: **Admin** sees and manages every app in every
environment; **User** sees and manages only the apps they created,
regardless of environment. A User deleting their own app in Production
requires Admin approval first; editing and every other delete are
immediate, and editing is conflict-aware (a stale edit is caught and the
user is asked to reload or override, never silently resolved). Each
app's detail page also has a Credentials tab (Okta-console-style,
fetched live, never cached) and, for Admins, an owner-reassignment
action. The tool records an audit trail of user
actions and uses caching to keep Okta API calls fast and within rate
limits. The project is expected to evolve into a multi-tenant SaaS product.

> **Status: planning stage.** This repository currently contains project
> structure, architecture decisions, requirements, and Claude Code agent
> definitions only — no application code has been written yet. `src/` is
> scaffolded but intentionally empty (see `src/README.md`). Hosting is
> confirmed as Windows Server + IIS (cloud provider still open), source
> control is GitHub, and production secrets are environment variables —
> see `docs/06-tech-stack.md` and `docs/08-open-questions.md`.

## Repo map

```
.
├── CLAUDE.md               # Instructions Claude reads automatically in this repo
├── .claude/agents/         # Specialist subagents for Claude Code (see below)
├── docs/                   # Planning docs: requirements, architecture, flows, security, roadmap
└── src/                    # Placeholder for backend + frontend code (empty for now)
```

## Planning docs (`docs/`)

| File | Contents |
|---|---|
| `01-requirements.md` | Functional & non-functional requirements, roles/permissions |
| `02-architecture.md` | Clean Architecture (backend) + frontend architecture, cross-cutting concerns |
| `03-app-creation-flow.md` | Creation wizard (any environment), edit flow, Staging→Production promotion, and import flows |
| `04-data-model.md` | Conceptual entities (App, Environment, User, Role, AuditLog) |
| `05-security.md` | AuthN/AuthZ model, secrets handling, Okta scopes, threat notes |
| `06-tech-stack.md` | Stack choices and rationale |
| `07-roadmap.md` | MVP → multi-env → SaaS/billing phases |
| `08-open-questions.md` | Genuinely undecided items (cloud provider choice, notifications, bulk actions, partial-failure reconciliation, Okta rate-limit handling) — not assumed either way. Hosting model, CI/CD, testing strategy, credential delivery, owner reassignment, edit/drift-sync conflicts, and duplicate app names are now resolved here too |
| `09-drift-sync.md` | Scheduled per-environment job that reconciles config/deletion drift between Okta and the local DB |
| `10-session-and-token-lifecycle.md` | Frontend login session/token handling: silent renewal, idle countdown modal, confirmed Okta token lifetimes |

## Claude Code agents (`.claude/agents/`)

Six specialist subagents are defined for this project. Claude Code will
route to them automatically based on the task, or you can call them
explicitly (e.g. "use the okta-expert subagent to review this scope
configuration").

| Agent | Focus |
|---|---|
| `software-engineer` | ASP.NET Core / EF Core / React implementation, SOLID, Clean Architecture, code review |
| `okta-expert` | Okta SDK/API, OIDC vs SAML app config, environment/org management |
| `cybersecurity-engineer` | AuthN/AuthZ review, secrets, OWASP, threat modeling, audit logging |
| `devops-engineer` | CI/CD, IIS/Windows Server hosting, environment promotion, caching infra, observability |
| `payment-integration-engineer` | Future SaaS billing/subscription integration (Stripe-class provider), plan/tier gating |
| `test-engineer` | Testing strategy across Clean Architecture layers: fake `IOktaAppService` vs. real Okta sandbox org, coverage review |

## Suggested next step

Once the planning docs are reviewed/approved, start implementation with the
`software-engineer` agent scaffolding `src/backend` (ASP.NET Core Clean
Architecture solution) and `src/frontend` (Vite + React + TypeScript app),
following `docs/02-architecture.md` and `docs/06-tech-stack.md`.
