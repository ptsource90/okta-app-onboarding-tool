# Roadmap

## Phase 1 — MVP (single tenant/customer)

- Auth against one configured third-party identity provider.
- Two roles: Admin, User (ownership-scoped).
- Create/edit/delete app (OIDC or SAML) via the wizard/edit form. Edit is
  never approval-gated, in any environment.
- Import (adopt) apps that already exist in an environment's Okta org but
  weren't created through this tool (Admin-only).
- Create an app directly in any configured environment (no per-user
  environment restriction — including Production; creation is not
  approval-gated in any environment).
- Promote an app from Staging to Production only — one-directional, with
  the human-confirmation step for environment-specific fields. Each
  environment is a separate Okta org.
- Delete-approval gate: a User deleting their own app in Production
  creates a pending request; any Admin can approve or reject it. Every
  other delete (Admin, or a User's own Staging app) is immediate.
- Deactivate/reactivate an app: both always immediate, ownership-scoped
  (Admin any app, User only apps they own) — never approval-gated, in any
  environment, for either role. Deactivating is a required precondition
  for deleting (see above), but since it's immediate this doesn't add a
  second approval cycle.
- Audit log of create/edit/import/delete(-requested/approved/rejected/
  cancelled)/deactivate/reactivate/promote/role-change/sync actions.
- Scheduled per-environment sync job that detects config, active/inactive
  status, or deletion drift between Okta and the local DB (an app
  edited, deactivated, reactivated, or deleted directly in the Okta
  console rather than through this tool) and reconciles it automatically
  — see `09-drift-sync.md`.
- Audit trail visibility: Admin sees the full/global log; a User sees
  the audit trail scoped to apps they own (their own actions, Admin
  actions on their app, and drift-sync system entries) — not a full
  global view. See `05-security.md`.
- In-memory caching of app list/org metadata reads.
- Deployed as a single instance, on a **Windows Server + IIS** (confirmed
  hosting model), in either AWS or Azure (cloud provider itself still
  open — see `08-open-questions.md`).
- Credential delivery (Credentials tab), Admin-initiated owner
  reassignment (with a required reason), and a conflict-aware Edit flow
  (vs. drift-sync/concurrent edits) — see `01-requirements.md` (14-16),
  `03-app-creation-flow.md`, and `04-data-model.md`.

## Phase 2 — Hardening & scale-out

- Move caching to a distributed cache (Redis) once running more than one
  instance.
- Support more than two named environments per deployment (the data model
  already allows it — this is UI/UX and Okta-org-management polish).
- Revisit whether approval needs a dedicated **Approver** role distinct
  from Admin (Phase 1 confirmed default: any Admin can approve) if the
  Admin group grows large enough that "anyone with Admin" is too broad a
  set of reviewers.
- Consider extending the `ApprovalRequest` gate to other high-risk actions
  (e.g. gating app creation directly in Production, not just deletion) —
  the entity's `RequestedAction` field was deliberately left generic to
  support this without a schema change, even though Phase 1 exercises it
  for `Delete` only today. (`Deactivate` was considered for the same
  gating and deliberately removed — see requirement 8 in
  `01-requirements.md` — so it is *not* a second example of this
  extensibility in the current build, despite earlier drafts of this
  roadmap saying otherwise.)
- CI/CD maturity: automated dependency scanning, environment promotion
  gates (see `devops-engineer`).

## Phase 3 — SaaS

- Introduce `TenantId`/`AccountId` on `User`, `Environment`, and
  `Application` (see `04-data-model.md` — additive, not a rewrite, if
  Phase 1 modeling left room for it).
- Introduce billing/subscription integration (see
  `payment-integration-engineer` agent): a well-established hosted
  payment/subscription provider, plan/tier limits (apps, environments,
  seats) enforced independently of the Admin/User role distinction.
- Introduce usage metering feeding both the audit log (for user-facing
  history) and billing (for plan enforcement/overage) without duplicating
  the underlying event capture.
- Revisit multi-tenant data isolation guarantees (row-level security or
  equivalent) as part of this phase, reviewed by `cybersecurity-engineer`.

## Explicitly deferred, revisit only when a phase above is reached

- Multi-tenancy.
- Billing/payment integration.
- Support for identity providers beyond whatever is configured for Phase 1
  (the auth layer should be provider-agnostic in design, but only one
  provider needs to be wired up initially).
