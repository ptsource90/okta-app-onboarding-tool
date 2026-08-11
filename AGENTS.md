# AGENTS.md

Guidance for AI coding agents working in the Okta App Onboarding Tool repo.

## What this is

An internal (later multi-tenant SaaS) tool for creating, editing, deleting,
importing, and promoting Okta apps (OIDC/SAML) across environments. Full
product context, confirmed decisions, and conventions live in
[CLAUDE.md](CLAUDE.md) — **read it first; it is the source of truth** and this
file intentionally does not duplicate it.

## Current phase: planning (read before touching `src/`)

This repo is **planning/architecture only**. `src/backend` and `src/frontend`
hold empty `.gitkeep` placeholders (see [src/README.md](src/README.md)).
**Do not write application code under `src/` unless the user explicitly asks
to start implementation.** Default work here is editing docs in
[docs/](docs/), agent-customization files, and the planning READMEs.

## Where knowledge lives (link, don't re-explain)

| Topic                                   | Doc                                                                              |
| --------------------------------------- | -------------------------------------------------------------------------------- |
| Requirements, roles/permissions         | [docs/01-requirements.md](docs/01-requirements.md)                               |
| Clean Architecture layers, use cases    | [docs/02-architecture.md](docs/02-architecture.md)                               |
| Create / edit / promote / import flows  | [docs/03-app-creation-flow.md](docs/03-app-creation-flow.md)                     |
| Data model, entities, `RowVersion`      | [docs/04-data-model.md](docs/04-data-model.md)                                   |
| AuthN/AuthZ, secrets, audit scoping     | [docs/05-security.md](docs/05-security.md)                                       |
| Tech-stack choices + rationale          | [docs/06-tech-stack.md](docs/06-tech-stack.md)                                   |
| Roadmap (MVP → multi-env → SaaS)        | [docs/07-roadmap.md](docs/07-roadmap.md)                                         |
| Genuinely open questions (don't assume) | [docs/08-open-questions.md](docs/08-open-questions.md)                           |
| Drift-sync job (Okta → local)           | [docs/09-drift-sync.md](docs/09-drift-sync.md)                                   |
| Session/token lifecycle                 | [docs/10-session-and-token-lifecycle.md](docs/10-session-and-token-lifecycle.md) |

When answering a design question, cite the relevant doc and state explicitly
if you are deviating from it and why. If a topic is listed in
[docs/08-open-questions.md](docs/08-open-questions.md), treat it as undecided —
do not silently assume a resolution.

## Non-negotiable rules an agent will get wrong otherwise

- **Controllers, never Minimal API.** The API is MVC-style (`ControllerBase`
  - attribute routing). Do not use `app.MapGet`/`MapPost` anywhere in
    `OnboardingTool.Api`.
- **Clean Architecture dependency direction.** Domain depends on nothing;
  Application depends only on Domain; Infrastructure implements Application's
  interfaces; API composes. No EF Core / Okta SDK / ASP.NET Core types in
  Domain. Keep controllers thin (validation + orchestration only).
- **Backend talks to Okta itself**, per-environment, via an OAuth 2.0 service
  app (`client_credentials` + `private_key_jwt`) — never delegated through
  the caller's Okta identity.
- **Secrets are environment variables only** (IIS App Pool level in prod) —
  never in `web.config`, source control, or a cloud secret manager. Locally
  use `dotnet user-secrets`. `appsettings.json` holds only the env-var _name_
  and non-secret values (e.g. `OktaServiceAppClientId`).
- **Every state-changing action is audited** (who/what/when/environment/
  result) before returning success. Viewing credentials is the one audited
  read (`AppCredentialsViewed`).
- **Every CRUD action needs explicit user confirmation** in the UI (review
  screen for create/edit, dialog for the rest) — separate from the Production
  delete approval gate.
- **Approval gate is Production-delete-only.** A `User` deleting their own
  Production app creates an `ApprovalRequest`; all other deletes/edits/
  (de)activations are immediate. Delete requires the app already deactivated.
- **Promotion is Staging → Production only**, one direction.
- **Ownership/role scoping is applied before any caller-supplied Sieve
  filter** — a `User` can never filter past the apps they own.
- **Design for tenancy later, don't build it now** — keep a `TenantId`
  retrofittable without implementing multi-tenancy in the MVP.

## Stack (do not substitute without discussion)

Backend: ASP.NET Core Web API (controllers), EF Core, MSSQL, Okta .NET SDK,
Serilog, FluentValidation, Sieve, Hangfire (`Hangfire.SqlServer`).
Frontend: React + TypeScript + Vite, TanStack Query + Table, Mantine, React
Hook Form + Zod, `@okta/okta-react`, `react-router-dom` 5.x.
Rationale for each is in [docs/06-tech-stack.md](docs/06-tech-stack.md).

## Specialist agents

Six Claude Code specialist subagents are defined in
[.claude/agents/](.claude/agents/): `software-engineer`, `okta-expert`,
`cybersecurity-engineer`, `devops-engineer`, `payment-integration-engineer`,
`test-engineer`. Defer Okta specifics to `okta-expert`, security reviews to
`cybersecurity-engineer`, infra/CI to `devops-engineer`, and test strategy to
`test-engineer`.

## Style

Prefer a well-maintained open-source library over hand-rolled infrastructure;
justify any new dependency in one sentence. Less code, simple over clever.
Only add what was asked for.
