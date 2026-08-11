# Requirements

## Purpose

Give users a self-service way to create, edit, delete, and copy Okta
applications (OIDC or SAML) across environments (e.g. staging →
production), with role-scoped visibility, an audit trail of actions, and
performant access to Okta data via caching. Users are not scoped to
specific environments — any environment is available to any authenticated
user; only app *ownership* and *role* (Admin vs. User) are scoped.

## Actors

- **Admin** — full visibility and control across all apps, all environments,
  and all users' actions. Can view the audit log for any user. Approves or
  rejects deletion requests submitted by Users. Can import (adopt) apps
  that already exist in Okta but weren't created through this tool.
- **User** — can create, edit, delete, deactivate, and reactivate apps
  *they* created, in any configured environment (no per-user environment
  restriction — see "Environment access" below). Deleting **or
  deactivating** an app they own in **Production** requires Admin
  approval before it takes effect (see "Approval workflow (delete &
  deactivate)" below); doing either in Staging, editing in any
  environment, and **reactivating in any environment**, all happen
  immediately, same as Admin. Cannot see other users' apps or the global
  audit log, and cannot import existing Okta apps.
  **Can see the audit trail for apps they own** — every entry tied to
  one of their apps (their own actions, an Admin's actions on that app,
  and system entries from the drift-sync job), not just a global log.

Identity for both roles comes from a third-party user store (Firebase,
Okta, or a generic SSO/OIDC provider) — this application is a relying
party/consumer of that identity, not an identity provider itself. Role is
assigned within this application (e.g. via a claim/group mapping or an
internal Users table), not assumed from the identity provider.

## Environment access

**Confirmed:** there is no per-user environment-access model. Every
authenticated user (Admin or User) can target any configured
`Environment` when creating, editing, or viewing apps — access is scoped
by **ownership** (a User only acts on apps they created) and **role**
(Admin vs. User), never by which environments a given user is permitted to
reach. All actual Okta operations are performed by the backend using its
own per-environment OAuth 2.0 service app credential (a private key,
`private_key_jwt` — see `04-data-model.md`), never
delegated through the caller's own Okta identity — so there is no
Okta-org-level access to restrict per user in the first place.

## Functional requirements

1. **Authentication** — users sign in via the configured third-party
   identity provider; the backend validates the resulting token on every
   request.
2. **App creation wizard**
   - Step 1: choose the target environment (any configured environment —
     not restricted per user). Creating in any environment, including
     Production, is not approval-gated — only deletion is (see
     requirement 7).
   - Step 2: choose app type — OIDC or SAML.
   - Step 3: **General Settings** (App logo, App visibility — shared
     across both app types, since Okta's own Application Integration
     Wizard treats these as app-level, not SAML- or OIDC-specific; see
     `03-app-creation-flow.md`), then the type-specific form (fields
     differ significantly between OIDC and SAML — see requirement 18 for
     SAML's full field set and `03-app-creation-flow.md`).
   - Step 4: review and submit; app is created in Okta (in the chosen
     environment's org) and recorded locally. **Confirmed: on success,
     immediately navigates to the new app's detail page with the
     Assignments tab active** — not left on the wizard, and not dropped
     on the app list either. Assigning people/groups right after
     creation is treated as the expected next action, even though it
     remains a fully separate, optional step the user can skip or return
     to later (the app already exists and is otherwise fully usable
     unassigned). See `03-app-creation-flow.md`.
3. **App listing** — Admin sees all apps across all environments and
   owners; User sees only apps they created (including any imported apps
   assigned to them). Paginated (`page`/`pageSize` query params, a
   server-enforced max `pageSize`) via Sieve rather than returning
   every matching row — see `06-tech-stack.md`. Search/filter:
   - Free-text search on app **Name** (substring, case-insensitive).
   - Filter dropdowns: **Environment**, **Type** (OIDC/SAML), and
     **Status** — three values, **Active** (default), **Deactivated**,
     and **Deleted in Okta** (`09-drift-sync.md`'s locked, view-only
     records). **Confirmed: no "Pending Deletion" or "Pending
     Deactivation" value** — both only apply to Production actions, so
     folding either into this same filter would mix a
     Production-specific concept into a dropdown that's otherwise
     environment-agnostic; each stays a badge/indicator shown on the app
     within the Active view instead (see "Approval workflow (delete &
     deactivate)" below). "Deactivated" itself is a normal filter value,
     though — unlike the pending states, deactivation isn't
     Production-specific (it can happen, immediately, in any
     environment). Admin only: an additional **Owner** filter
     (irrelevant for a User, who is already scoped to their own apps).
4. **App editing** — Admin can edit any app; a User can edit only apps
   they own. Available in every environment (including Production) and
   never approval-gated, unlike deletion. The edit form is a full
   re-edit using the same type-specific (OIDC/SAML) fields as the
   creation form — nothing is locked post-creation. On submit: the app
   is updated in Okta (its own environment's org), the local
   `ConfigurationJson` is updated, an audit log entry is recorded, and
   the relevant cache entry is invalidated. See `03-app-creation-flow.md`.
   **Confirmed: conflict-aware.** The form captures the app's `RowVersion`
   (`04-data-model.md`) when opened; if that no longer matches at submit
   time — most likely the drift-sync job or another edit changing the app
   in the meantime — the user isn't silently overwritten or silently
   blocked. They're shown what changed and asked to either load the
   latest version (discarding their in-progress edit) or push their edit
   through anyway (an explicit override). See `03-app-creation-flow.md`.
5. **App assignments (people & groups)** — once an app is created, an
   "Assignments" tab lets an Admin (any app) or a User (apps they own)
   assign or unassign people and groups from the Okta org the app lives
   in. Search is autocomplete: minimum 2 characters before searching,
   showing the top 5 matches, against that specific environment's Okta
   directory (not this tool's own `User` table — the assignable universe
   is Okta's whole people/group directory). Not approval-gated in any
   environment, same immediate shape as editing. Persisted locally
   (`ApplicationAssignment`, `04-data-model.md`) rather than fetched live
   from Okta on every view, since regular Users have no direct Okta
   access — assignment changes are a pass-through (User → backend →
   Okta). The scheduled sync job also reconciles assignments made
   directly in Okta (`09-drift-sync.md`). See `03-app-creation-flow.md`.
6. **Import (adopt) existing apps** — Admin first chooses which
   environment to import from (any configured environment — not limited
   to Staging or Production specifically, same as everywhere else in the
   tool), then can scan that environment's Okta org for apps that already
   exist there but aren't yet tracked by this tool, and adopt selected
   ones, assigning each an owner. See `03-app-creation-flow.md`.
7. **App deletion, with approval gated to Production** — **confirmed
   precondition: an app must already be deactivated (`IsActive = false`)
   before it can be deleted.** The delete action is unavailable — hidden/
   disabled in the UI and rejected server-side if attempted directly —
   for any app that's still active; see requirement 8 for deactivation
   (always immediate — the app just needs to be inactive, no approval
   cycle to wait on first). Once deactivated: Admin can delete an app
   directly in any environment. A User deleting an app they own in
   Staging also happens immediately; deleting one in Production instead
   creates a pending deletion request, and the app is only actually
   removed from Okta once an Admin approves it. An Admin can reject a
   request (app remains deactivated but otherwise untouched) or the
   requesting User can cancel their own pending request. See "Deletion
   approval workflow" below and `03-app-creation-flow.md`.
8. **App deactivation and reactivation** — **confirmed: neither is ever
   approval-gated, in any environment, for either role.** Admin can
   deactivate or reactivate any app; a User can deactivate or reactivate
   only apps they own. Both execute immediately, the same way editing
   does — they never touch `ApprovalRequest`. (Earlier drafts of this doc
   gated a User's Production deactivation the same way as delete; that
   was reconsidered and removed — only delete is gated now.)
   Deactivating is still the required first step before an app can be
   deleted at all (requirement 7), but since deactivating itself is
   always immediate, a User only ever goes through **one** approval cycle
   total to fully remove a Production app they own (deactivate
   immediately, then request delete). See `03-app-creation-flow.md`.
9. **Promote an app from Staging to Production** — given a Staging app,
   create an equivalent app in Production, remapping environment-specific
   fields (see `03-app-creation-flow.md`). This is one-directional only:
   there is no Production → Staging path, and no path between any other
   environment pair even if more environments exist. A user can, however,
   *create* a fresh app directly in any environment (see requirement 2) —
   the direction restriction applies specifically to promotion, not to
   creation. Environments are separate Okta orgs, so a promotion always
   crosses org boundaries.
10. **Scheduled Okta configuration sync** — a per-environment scheduled job
    (cron) periodically compares each tracked app's config (including its
    active/inactive status and assignments) in Okta against the local
    record, to catch changes an Admin made directly in the Okta console
    rather than through this tool. One-directional only (Okta → local;
    the reverse already happens synchronously via the edit/delete/
    deactivate/reactivate/assignment flows, never on a schedule):
    - **Config or active/inactive status differs:** Okta wins — the local
      record is auto-updated to match, no human confirmation required for
      this direction. Logged as a system-actor audit entry (see
      `04-data-model.md`).
    - **Assignments differ:** same "Okta wins" treatment, applied
      per-app against `ApplicationAssignment` rows (requirement 5, see
      `04-data-model.md` and `09-drift-sync.md`).
    - **App was deleted directly in Okta:** soft-deleted locally too, but
      unlike a normal delete this stays visible in the UI, clearly marked
      as "deleted in Okta," view-only (editing/deleting/deactivating it
      further is disabled — there's nothing left in Okta to act on).
      Contrast with a delete initiated *through this tool*, which behaves
      as today: it simply disappears from the active list (soft-deleted
      under the hood, audit trail intact, no special ghost view).
    - Does **not** auto-discover brand-new, never-imported apps — that
      stays a deliberate, Admin-initiated action via the existing Import
      flow (requirement 6).
    - See `09-drift-sync.md` for the full design.
11. **Audit logging** — every create, import, edit, assign/unassign,
    delete-requested, delete-approved, delete-rejected, delete-cancelled
    (by the requester or auto-cancelled by the sync job), deleted,
    deactivated, reactivated, copy, owner-reassigned, and scheduled-sync
    action is recorded: who (or, for system actions, that it was the
    system) performed/reviewed it, what was affected, which
    environment(s), when, and whether it succeeded or failed. **One
    deliberate exception to the "mutations only" pattern:** viewing an
    app's Credentials tab (requirement 14) is also logged
    (`AppCredentialsViewed`), since disclosing a secret is audit-worthy
    even though nothing changed. **Visibility is role- and
    ownership-scoped, not one global view for everyone:** Admin sees
    every entry; a User sees only entries tied to apps they own (see
    `05-security.md`), never the full/global log. Paginated the same way
    as app listing (`06-tech-stack.md`) — this view especially needs it,
    since it's append-only and only grows over time. Search/filter:
    - Free-text search on the associated app's **Name**.
    - Filter dropdowns: **Action**, **Environment**, **Result**
      (Success/Failure), and a **date range**. Admin only: a
      **Performed by** filter (irrelevant for a User, already scoped to
      their own apps' history).
12. **Caching** — read-heavy, slow-changing Okta data (app lists, org
    metadata) is cached to reduce latency and stay within Okta API rate
    limits; caches are invalidated on writes (including the sync job's).
13. **Role management** — an Admin can assign/change a user's role.
14. **Credential delivery** — once an app exists, a **Credentials** tab on
    its detail page shows whatever a person actually needs to wire their
    own application up against Okta, styled after Okta's own Admin
    Console rather than inventing a new layout:
    - **OIDC:** Client ID and Client Secret, each with its own Copy
      button; Client Secret is masked by default with a "Reveal" toggle,
      the same interaction as Okta's own console — not a one-time
      reveal, since Okta itself lets an admin come back and view it again
      later.
    - **SAML:** the Identity Provider (Okta) metadata a Service Provider
      needs — SSO URL, Issuer, Audience — and the signing certificate
      (Download button).
    - **Confirmed: never persisted by this tool.** Both the Client Secret
      and the SAML certificate are fetched live from Okta on each view of
      this tab — never cached, never stored in `ConfigurationJson`, never
      written to any log. Viewing this tab is itself an audited action
      (`AppCredentialsViewed`) even though it's a read, not a mutation —
      see `04-data-model.md` and `05-security.md`.
    - Visible to whoever can already see the app (Admin: any app; User:
      only apps they own) — no separate permission beyond that. See
      `03-app-creation-flow.md`.
15. **Owner reassignment** — an Admin can change an app's `OwnerUserId` to
    a different user, from the app's detail page. **Admin-only** — a
    `User`, even the current owner, cannot reassign ownership themselves.
    Submitting a reassignment requires a **reason** (free-text, required,
    not optional) in addition to the general confirmation-before-mutation
    step every action already gets. Recorded as its own audit log entry
    (`OwnerReassigned`) capturing the previous owner, the new owner, the
    Admin who made the change, and the reason. Purely a local-record
    change — Okta has no concept of "owner" the way this tool does, so no
    Okta call is made. See `03-app-creation-flow.md`.
16. **Duplicate app-name check** — Okta itself rejects creating an app
    whose label collides with an existing one in the same org (confirmed:
    Okta's API returns `Application label must not be the same as an
    existing application label`), scoped per org — i.e. per `Environment`
    in this tool's model. Rather than only surfacing that at submit time,
    the wizard's Name field checks for a same-environment duplicate as
    soon as it loses focus (`onBlur`), once an environment is chosen in
    Step 1. A match surfaces the conflicting app's **Name, Type, and
    Owner** in a popup so the user can rename or recognize it as an app
    they already know about. This is an early warning, not the
    enforcement — Okta's own rejection at submit time remains
    authoritative (the local check is only as fresh as the last sync
    cycle). See `03-app-creation-flow.md`.
17. **SAML attribute statement configuration** — the SAML branch of the
    creation/edit form supports two distinct, independently optional
    sections, matching Okta's own SAML app model rather than treating
    "attribute statements" as a single opaque field:
    - **Attribute Statements** — zero or more rows of Name, Name Format
      (Unspecified / URI Reference / Basic — default Basic), and Value (a
      literal or an Okta Expression Language expression, e.g.
      `user.email`; this tool does not validate or evaluate the
      expression itself, only passes it through to Okta).
    - **Group Attribute Statements** — zero or more rows of Name, Name
      Format (same three options), Filter Type (Starts with / Equals /
      Contains / Matches regex), and Filter Value.
    - **Validation (confirmed, matches Okta's own stated rule):** a
      statement's Name is required, capped at 512 characters, and must be
      unique across *both* sections combined — a name can't be reused
      between an Attribute Statement row and a Group Attribute Statement
      row, checked client-side as rows are added and re-validated
      server-side on submit.
    - **Confirmed: carries over as-is on Promote** (this project's only
      environment-to-environment copy mechanism — see requirement 9;
      "copy" elsewhere in this doc refers to the same action, not a
      separate one) — regardless of source/target environment, and
      including every field of every row (Value/Filter Value included,
      not just that a row exists). This differs from environment-scoped
      fields like SSO URL/audience/certificate, which *do* get remapped
      on Promote. See `03-app-creation-flow.md`.
    - See `04-data-model.md` for the `ConfigurationJson` shape and
      `08-open-questions.md` for a flagged (not yet resolved)
      verification item around how this maps onto the current Okta API.
18. **SAML field parity with Okta's Application Integration Wizard
    (confirmed: full parity, not a curated subset)** — the SAML branch of
    the creation/edit form covers every field in Okta's own AIW SAML
    reference (https://help.okta.com/oie/en-us/content/topics/apps/aiw-saml-reference.htm),
    mirroring its own General/Advanced-Settings split rather than
    flattening everything into one form:
    - **SAML Settings — General** (always visible): Single sign-on URL,
      a "use this URL for Recipient/Destination too" toggle (default on;
      unchecked reveals separate Recipient URL and Destination URL
      fields), Audience URI (SP Entity ID), Default RelayState, Name ID
      format, Application username format, and Update application
      username on (only meaningful, and only shown, when Application
      username format is Custom).
    - **SAML Settings — Advanced Settings** (collapsed behind a "Show
      Advanced Settings" toggle, matching Okta's own default-collapsed
      UX): signing (Response, Assertion Signature, Signature Algorithm,
      Digest Algorithm), encryption (Assertion Encryption, and — only
      when set to Encrypted — Encryption Algorithm, Key Transport
      Algorithm, Encryption Certificate), Signature Certificate, Single
      Logout (Enable Single Logout, and — only once a Signature
      Certificate is uploaded and the checkbox is on — Single Logout URL
      and SP Issuer), Signed Requests, Other Requestable SSO URLs,
      Assertion Inline Hook, Authentication context class, Honor Force
      Authentication, SAML Issuer ID (override), Maximum app session
      lifetime, and Attribute Statements / Group Attribute Statements
      (requirement 17).
    - **Confirmed destructive-toggle warning:** enabling Signed Requests
      causes Okta to delete any statically-defined "Other Requestable SSO
      URLs" and read SSO URLs from the signed request instead — the two
      are mutually exclusive on Okta's side, not just in this tool's UI.
      Toggling Signed Requests on when Other Requestable SSO URLs already
      has rows must show an explicit warning before submit, consistent
      with this tool's general confirmation-before-mutation principle —
      not a silent data loss.
    - **Deliberately not built for MVP, despite "full parity":** the
      "Logout" section's Early-Access user-initiated/app-initiated SLO
      configuration. Okta's own reference labels it an Early Access
      release; this tool supports Single Logout through the classic,
      generally-available Enable Single Logout/Single Logout URL/SP
      Issuer fields listed above instead. Revisit once (if) that EA
      feature reaches general availability. See `08-open-questions.md`.
    - **Flagged, not resolved:** Assertion Inline Hook depends on Inline
      Hooks that already exist in the org — this tool does not create or
      manage hooks itself (out of scope), so this would need a read-only
      "list existing hooks" capability at minimum, and it's not yet
      decided whether that's worth building for a field this narrow. See
      `08-open-questions.md`.
    - **App logo** (PNG/JPG/GIF, <1MB) is accepted in the shared General
      Settings step (requirement 2) and forwarded to Okta as part of app
      creation — **confirmed: not persisted by this tool**, same
      "pass-through, don't hoard a copy" principle already applied to
      credentials (requirement 14).
    - Exact dropdown enumerations (Name ID format values, Application
      username format values, Signature/Digest/Encryption/Key-Transport
      algorithm options, Authentication context class values) are drawn
      from stable SAML 2.0/Okta conventions but should be confirmed
      against the current Okta SDK's actual enum values by `okta-expert`
      before implementation, rather than hand-typed from this doc without
      checking — see `03-app-creation-flow.md` for the illustrative list
      and `08-open-questions.md` for the flag.
    - **Assignments during creation — same treatment as OIDC (requirement
      19), not protocol-specific.** Okta's own AIW wizard shell has an
      assignment step (grant to everyone / limit to selected groups /
      skip for now) regardless of app type. Not built as a step inside
      the SAML form — no assignment fields in Step 3/4 — but Step 4's
      success path immediately navigates to the new app's Assignments
      tab (requirement 2), same as for a freshly created OIDC app. The
      app starts unassigned either way and remains fully usable if the
      user navigates away without assigning anything.
19. **OIDC field parity with Okta's Application Integration Wizard
    (confirmed: full parity, not a curated subset)** — the OIDC branch of
    the creation/edit form covers every field in Okta's own AIW OIDC
    guide (https://help.okta.com/oie/en-us/content/topics/apps/apps_app_integration_wizard_oidc.htm),
    which varies more by platform (Web/SPA/Native) than SAML's form does
    by anything comparable — see `03-app-creation-flow.md` for the full
    per-platform breakdown. Summary of what's newly in scope beyond what
    was previously documented:
    - **Platform list, confirmed: Web, SPA, Native only — "service" is
      explicitly excluded as a creatable type.** Okta's own wizard
      doesn't expose "service" as a creatable application type — earlier
      phrasing in this doc and in `okta-expert.md` listed
      "web/native/SPA/service" as if all four were wizard-equivalent
      options; that was wrong and has been corrected. **Decision: this
      tool does not support creating a "service" (machine-to-machine)
      app at all**, via the wizard or otherwise — not deferred, not a
      direct-API-call escape hatch. If a machine-to-machine integration
      is ever needed, it would be created directly in the Okta Admin
      Console (or a future, separately-scoped tool capability), not
      through this tool.
    - **App integration name validation:** Okta's own reference states
      the name may only contain UTF-8 three-byte characters — this
      applies to the same Name field the duplicate-name pre-check
      (requirement 16) already validates, not a separate OIDC-only rule.
    - **Client authentication** is platform-dependent: Web apps choose
      Client secret or Public key/Private key; SPA is forced to None
      (PKCE required); Native defaults to None (recommended) but can also
      use Client secret or Public key/Private key. **Confirmed: Client ID
      and Client Secret continue to surface only through the Credentials
      tab (requirement 14)**, not duplicated into this form. **Flagged,
      likely Phase 2 given complexity:** Okta supports a second,
      independently-rotatable client secret and, separately, a
      Public key/Private key pair as an alternative to a client secret
      entirely — neither is designed here; the Credentials tab as
      currently specified assumes exactly one active secret.
    - **PKCE** (Proof Key for Code Exchange): forced-on default for SPA
      and Native's None auth method; optional for Client secret/Public-
      Private key auth on any platform.
    - **DPoP** (Demonstrating Proof of Possession): a checkbox, mutually
      exclusive with the Implicit (hybrid) grant type — Okta's API
      rejects the combination, so this form must validate the same
      exclusion, not just Okta's API.
    - **Grant type options differ by platform** (Web: Client credentials,
      Authorization code, Refresh Token, Device Authorization, Implicit
      (hybrid), CIBA; SPA: Authorization Code, Refresh Token, Device
      Authorization, Implicit (hybrid); Native: Authorization Code,
      Interaction Code, Refresh Token, Resource Owner Password, SAML 2.0
      Assertion, Device Authorization, Token Exchange, Implicit
      (hybrid)) — **confirmed: this form only shows the grant types valid
      for the platform already chosen**, not the full union with
      invalid options grayed out. Refresh Token has its own sub-config
      (rotate-every-use vs. persistent, with a persistence period if
      rotating). Implicit (hybrid) has its own sub-checkboxes (Allow ID
      token / Allow Access token). **Flagged, dependency-based, same
      shape as Assertion Inline Hook for SAML:** selecting the CIBA grant
      type requires a "Preferred authenticator for CIBA" — this tool
      doesn't manage authenticator creation, so this would need at
      minimum a read-only "list existing authenticators" capability, not
      yet decided as worth building. See `08-open-questions.md`.
    - **Sign-in redirect URIs** (required, multiple, orderable, absolute
      URIs) with a **Confirmed security-warning toggle**: "Allow wildcard
      in sign-in redirect URI" is supported but Okta's own docs
      explicitly caution against it (wildcards can let a malicious actor
      have tokens/codes sent to an attacker-controlled page) — enabling
      it shows a warning, not a silent toggle, consistent with this
      tool's confirmation-before-mutation principle applied to a security
      risk rather than a data-loss risk this time.
    - **Sign-out redirect URIs** (optional, multiple, absolute URIs, no
      wildcards).
    - **Login initiated by** (App Only / Either Okta or App) — when
      "Either Okta or App": **Application visibility** (a second,
      OIDC-specific visibility toggle, distinct from the shared App
      visibility field in requirement 2 — that one hides the icon
      app-wide, this one is specific to the Okta-tile-initiated login
      flow) and **Login flow** (Send ID Token directly to app [Okta
      Simplified], with an OIDC-scopes picker / Implicit, which surfaces
      a read-only "App Embed Link" plus an editable **Initiate login
      URI**).
    - **User Consent section** (visible only when Okta API Access
      Management is enabled for the org — an org-feature-gated field,
      same pattern as Attribute Statements' EA gating in requirement 17):
      Require consent, Terms of Service URI, Policy URI, and **Logo URI**
      — a URL to an image shown on the OAuth consent screen (PNG,
      transparent background, landscape, ≥420×120px). **Do not conflate
      this with the shared App logo file upload in requirement 2** — two
      different images serving two different purposes (dashboard tile
      icon vs. consent-screen branding), one an uploaded file, one a URL.
    - **Email Verification Experience:** a single optional Callback URI
      for customizing where an email magic-link redirects after
      verification.
    - **Groups claim filter** (Sign On tab, "OpenID Connect ID Token"
      section) — the OIDC analog of SAML's Group Attribute Statements
      (requirement 17), same Filter Type options (Starts with / Equals /
      Contains / Matches regex) plus an alternative Expression mode for a
      custom Okta Expression Language filter. **Confirmed UI guidance,
      not a hard block:** Okta's own docs note ID token creation fails
      if more than 100 groups match the filter *and* the grant type isn't
      Authorization Code (with or without PKCE) — surface this as a
      warning when both conditions are plausible, since it's a runtime
      token-issuance failure mode, not something this form can fully
      prevent client-side.
    - **App rate limit** — a percentage slider/number input, default 50%
      of the org's API rate limits, per app.
    - **Issuer** — dropdown: Org URL / Custom URL (only meaningful if the
      target org has a custom URL domain configured) / Dynamic (either,
      based on request domain). Niche, included for parity.
    - **Trusted Origins (Base URIs for CORS)** — Web and Native apps only.
      **Confirmed cross-cutting side-effect flag:** these URIs are added
      to the org's own Trusted Origins list, a security-relevant
      org-wide setting, not scoped to just this app — treat submitting
      this field with the same weight as any org-level security change,
      not as an ordinary per-app field.
    - **Assignments during creation** — Okta's own wizard has an
      assignment step (grant to everyone / limit to selected groups /
      skip for now) as part of app creation. **Confirmed: not built as a
      step inside the wizard itself** — no assignment fields in Step 3/4
      — **but Step 4's success path immediately navigates to the new
      app's Assignments tab** (requirement 2), so assigning right after
      creation is still the expected next action, just via the same
      Assignments tab used for any app at any time rather than
      duplicated wizard-only fields. The app itself starts unassigned
      either way and remains fully usable if the user navigates away
      without assigning anything.
    - **Deliberately not built for MVP, despite "full parity":** the
      dedicated OIDC "Logout" (Single Logout) section — Logout redirect
      URIs, cross-app/Okta-initiated logout propagation, Logout request
      URL, Request binding, User session details — is explicitly labeled
      Early Access by Okta. This tool supports the classic, GA Sign-out
      redirect URIs field instead, same reasoning as SAML's excluded EA
      Logout section (requirement 18).
    - **Flagged, inconsistently labeled in Okta's own docs:** Network IP
      (restricting which network zones a token can be used from) is
      marked Early Access in some parts of Okta's own reference and not
      explicitly marked in others — treated as not-built-for-MVP pending
      `okta-expert` confirming its actual current status, rather than
      this doc guessing.
    - See `04-data-model.md` for the `ConfigurationJson` shape and
      `08-open-questions.md` for the consolidated flagged items above.

## Deletion approval workflow

- **Confirmed scope:** applies only when a User (non-admin) deletes an app
  they own in **Production**. Admin deletions execute immediately in any
  environment, and a User deleting a Staging app also executes immediately
  — the gate exists specifically because a Production deletion is the
  higher-stakes case. **Deactivating, reactivating, and editing are never
  gated, in any environment, for either role** — only delete is.
  (`ApprovalRequest.RequestedAction` still supports a `Deactivate` value
  in the data model, kept extensible per `04-data-model.md`, but nothing
  in this app currently creates one — deactivation gating was considered
  and removed.)
- **Confirmed precondition: delete requires the app to already be
  deactivated (`IsActive = false`).** This is checked independently of
  the approval gate above, and applies in every environment, not just
  Production. Since deactivating is always immediate, a User only ever
  goes through **one** approval cycle total to fully remove a Production
  app they own: deactivate (immediate), then request delete (gated).
  Also re-verified at **approval** time, not just at request time — if
  the app was somehow reactivated in the window between a delete request
  being created and an Admin reviewing it, approval must fail/re-check
  rather than blindly deleting an active app.
- **Confirmed approver:** any Admin can review a pending request — there is
  no separate "approver" role. Revisit only if a dedicated approver role
  becomes necessary at scale (see `07-roadmap.md`).
- Requesting: a User's delete action creates an `ApprovalRequest` (see
  `04-data-model.md`) instead of calling Okta. The app is already
  deactivated (that's the precondition above) but stays visible in the
  meantime, marked as having a pending deletion request.
- Reviewing: any Admin can approve or reject a pending request. Approving
  performs the actual Okta deletion (after re-verifying `IsActive =
  false`) and soft-deletes the local record; rejecting leaves the app
  untouched (still deactivated, not deleted) and records a reason.
- Withdrawing: the original requester can cancel their own pending request
  before it's reviewed.
- Every transition (requested/approved/rejected/cancelled) is its own audit
  log entry — see `05-security.md`.
- If the scheduled sync job (`09-drift-sync.md`, requirement 10) discovers
  that an app with a pending delete request was already deleted directly
  in Okta, it auto-cancels that request (with a logged system reason)
  rather than leaving it pending against an app that's already gone.

## Non-functional requirements

- **Security** — least-privilege access by default; all secrets
  (Okta API tokens, client secrets) stored outside source control and the
  database in plaintext; see `05-security.md`.
- **Reliability** — Okta API failures should surface a clear error and not
  leave the local database and Okta out of sync (or should reconcile
  visibly if they do — the scheduled sync job in requirement 10 is one
  mitigation for drift generally, though the specific in-request
  partial-failure case is still an open question — see
  `08-open-questions.md`).
- **Performance** — app list views should be fast (cache-backed) even
  though the source of truth is a third-party API.
- **Confirmation before mutation** — every CRUD action (create, edit,
  delete, deactivate, reactivate, assign, unassign) requires an explicit
  confirmation step before it executes; nothing fires on a single click.
  This is independent of the Production delete approval-gate — one
  controls whether an Admin must sign off before Okta is called at all,
  the other just guards against an accidental click.
- **Maintainability** — Clean Architecture, SOLID, minimal custom
  infrastructure code; favor open-source libraries.
- **Extensibility toward SaaS** — the data model and role system should not
  need a rewrite to add multi-tenancy and billing later (see
  `07-roadmap.md`); this does not mean building those features now.

## Out of scope for MVP

- Multi-tenancy (multiple customer organizations in one deployment).
- Billing/subscription enforcement (see `payment-integration-engineer`
  agent and `07-roadmap.md` — explicitly a later phase).
- Support for more than two environments (staging/production) — the model
  should not hard-code "two," but building N-environment UI polish is not
  MVP. Each environment is a separate Okta org (confirmed).
- Bulk import/delete/copy (acting on multiple apps in one action) —
  MVP is single-app actions; see `08-open-questions.md`.
- Proactive notifications (email/Slack) on actions are not yet decided —
  tracked in `08-open-questions.md` rather than silently assumed either
  way. (Drift detection and the testing strategy, previously listed here
  alongside notifications as undecided, are both now resolved — see
  `09-drift-sync.md` and `08-open-questions.md` respectively — and aren't
  open items anymore despite earlier phrasing in this doc.)
