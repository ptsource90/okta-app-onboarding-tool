# App creation, promotion & import flows

## App creation wizard

```
[Create app] click
     │
     ▼
Step 1 — Choose environment
     │  User picks the target environment (e.g. Staging, Production) — any
     │  configured environment is available; there is no per-user
     │  environment restriction. No approval is required to create in any
     │  environment, including Production — only the delete flow is
     │  approval-gated (see "Delete an app" below).
     ▼
Step 2 — Choose app type
     │  User picks: OIDC  or  SAML
     ▼
Step 3 — General Settings, then the type-specific form
     │  General Settings (shared, both types): App logo (optional,
     │  PNG/JPG/GIF, <1MB — forwarded to Okta, not persisted by this
     │  tool), App visibility (hide icon from end users). Plus the Name
     │  field (see the duplicate-name pre-check below).
     ├── OIDC branch: full parity with Okta's own Application
     │     Integration Wizard field set — see "OIDC app creation form
     │     (Okta AIW field parity)" below, not just the six fields this
     │     shorthand line used to imply. **Confirmed: platform choice is
     │     Web/SPA/Native only** — "service" was listed here previously
     │     but is explicitly excluded; see the new section for detail.
     └── SAML branch: full parity with Okta's own Application
           Integration Wizard field set — see "SAML app creation form
           (Okta AIW field parity)" below, not just the four fields this
           shorthand line used to imply
     ▼
Step 4 — Review & submit
     │  Show a summary of everything entered, including the chosen
     │  environment, before the user confirms.
     ▼
Backend:
  1. Validate input (server-side, same rules as client-side schema)
  2. Authorize: caller must be Admin, or User creating their own app
  3. Create the app in Okta via the Okta SDK for the chosen environment's
     org (type-specific call)
  4. Persist the local app record (owner = current user, environment =
     chosen in Step 1, Okta app ID, type)
  5. Write an audit log entry (who, what, environment, result)
  6. Invalidate the cached app list for the affected environment/owner
  7. Return the created app to the frontend
     ▼
Frontend invalidates the app list (cache invalidation) AND immediately
navigates to the new app's detail page with the **Assignments tab
active** — confirmed: not left on the wizard, not dropped on the app
list. Assigning people/groups is treated as the expected next action,
though it's still fully optional/skippable — the app already exists and
works unassigned if the user navigates away. See "App assignments" below.
```

**Duplicate-name pre-check (Step 3):** as soon as the Name field loses
focus (`onBlur`), and only once an environment is chosen in Step 1, the
frontend calls `CheckAppNameAvailabilityQuery` (`02-architecture.md`) to
ask whether that name already exists on an `Application` record for the
chosen `Environment`. **Confirmed: Okta itself rejects a colliding app
label at creation time**, scoped per org — i.e. per `Environment` in this
tool's model (`Application label must not be the same as an existing
application label`) — so this is a pre-emptive UX check, not the
enforcement point. A match surfaces a popup with the conflicting app's
**Name, Type, and Owner**, so the person can pick a different name or
recognize it as an app they already know about, before spending time on
the rest of the form. Okta's own rejection at submit time stays
authoritative regardless — the local check only reflects data as fresh as
the last drift-sync cycle, so a same-named app created directly in the
Okta console moments ago could still slip past it.

Environment is asked first, before app type, because it changes which
Okta org (and which org-specific values, e.g. existing redirect domains)
the rest of the wizard should be aware of — asking it last would mean
re-validating type-specific fields against a different org after the fact.

Rationale for splitting the form by type at Step 3 rather than one shared
form: OIDC and SAML apps in Okta are different resources with materially
different required fields and validation rules (e.g. redirect URI syntax vs.
X.509 certificate format). Forcing them into a single generic form either
hides fields that don't apply or produces a confusing, over-parameterized
UI. Two focused forms behind a single wizard shell is simpler to build,
test, and validate than one form trying to do both.

## OIDC app creation form (Okta AIW field parity)

**Confirmed: full parity with Okta's own Application Integration Wizard
OIDC field set**
(https://help.okta.com/oie/en-us/content/topics/apps/apps_app_integration_wizard_oidc.htm),
not a curated subset — `01-requirements.md` requirement 19. Unlike SAML,
the field set varies by **platform** (Web / SPA / Native) more than by a
General/Advanced split, so the platform choice made in Step 2/3 narrows
which of the tables below actually apply, rather than everything showing
unconditionally.

**Platform choice, confirmed: Web, SPA, or Native only.** Okta's own
wizard doesn't expose "service" as a fourth option, despite earlier
drafts of this doc listing it. **This tool does not support creating a
"service" (machine-to-machine) app at all** — not via the wizard, and
not via a direct API call as an escape hatch either. A machine-to-machine
integration, if ever needed, is out of scope for this tool entirely,
same as any other capability deliberately excluded here.

### Fields common to all three platforms

| Field | Notes |
|---|---|
| App integration name | Required. **Validation: UTF-8 three-byte characters only** (confirmed from Okta's reference) — the same Name field the duplicate-name pre-check (requirement 16) already validates. |
| Application notes for end users | Optional. Shown on the Okta End-User Dashboard. |
| Application notes for admins | Optional. Shown to admins on the app's own page. |
| Require DPoP header in token requests | Checkbox. **Mutually exclusive with the Implicit (hybrid) grant type** — Okta's API rejects the combination; validate the same exclusion here, not just rely on Okta's own rejection. |
| Grant type | **Only the grant types valid for the already-chosen platform are shown** (see platform tables below), not the full union grayed out. A "Show more grant types" toggle mirrors Okta's own "Click Advanced" affordance. |
| Sign-in redirect URIs | Required, multiple, orderable, absolute URIs. **Confirmed security-warning toggle:** "Allow wildcard in sign-in redirect URI" is supported but shows an explicit warning when enabled (Okta's own docs caution that wildcards can let a malicious actor have tokens/codes sent to an attacker-controlled page) — a security-risk warning, not a data-loss one like Signed Requests in the SAML form, but the same "don't let it be a silent toggle" principle. |
| Sign-out redirect URIs | Optional, multiple, absolute URIs. No wildcards. |
| Client ID / Client Secret | **Confirmed: not duplicated into this form** — continue to surface only via the Credentials tab (requirement 14), consistent with how it's already handled. |
| Client authentication | Platform-dependent — see below. |
| PKCE | Forced-on default for SPA and for Native's None auth method; optional elsewhere. |

### Platform-specific: Client authentication & grant types

| Platform | Client authentication options | Grant types offered |
|---|---|---|
| **Web** | Client secret / Public key-Private key | Client credentials, Authorization code, Refresh Token, Device Authorization, Implicit (hybrid), CIBA |
| **SPA** | None only (forced; PKCE required by default) | Authorization Code, Refresh Token, Device Authorization, Implicit (hybrid) |
| **Native** | None (recommended, default) / Client secret / Public key-Private key | Authorization Code, Interaction Code, Refresh Token, Resource Owner Password, SAML 2.0 Assertion, Device Authorization, Token Exchange, Implicit (hybrid) |

**Sub-configuration when the relevant grant type is selected:**
- **Refresh Token** (any platform): rotate-every-use vs. persistent token, with a persistence-period field shown only when rotating.
- **Implicit (hybrid)** (any platform): two sub-checkboxes, Allow ID token / Allow Access token.
- **CIBA** (Web only): requires **Preferred authenticator for CIBA**. **Flagged, not resolved** — same shape as Assertion Inline Hook in the SAML form: this depends on authenticators already configured in the org (authenticator creation/management is out of scope for this tool), so it needs at minimum a read-only "list existing authenticators" capability, not yet decided as worth building. See `08-open-questions.md`.

**Flagged, likely Phase 2 given complexity:** Okta supports a *second*,
independently-rotatable client secret (rotate by generating a second
secret, then switching which one is "active"), and, separately, a
Public key/Private key pair as an alternative to a client secret
entirely. Neither is designed here — the Credentials tab as currently
specified (requirement 14) assumes exactly one active secret. Revisit
if/when rotation support is prioritized.

### Login flow configuration (all platforms)

| Field | Behavior |
|---|---|
| Login initiated by | App Only, or Either Okta or App. |
| Application visibility | **Shown only when "Either Okta or App."** A second, OIDC-specific visibility toggle — distinct from the shared App visibility field in requirement 2, which hides the icon app-wide; this one is specific to the Okta-tile-initiated login flow. |
| Login flow | **Shown only when "Either Okta or App."** "Send ID Token directly to app (Okta Simplified)" (with an OIDC-scopes picker) or "Implicit" (shows a read-only App Embed Link plus an editable **Initiate login URI**). |

### User Consent (visible only when Okta API Access Management is enabled for the org)

Same org-feature-gated-visibility pattern as Attribute Statements' EA
gating (requirement 17) — this section doesn't render at all for orgs
without that feature on.

| Field | Notes |
|---|---|
| Require consent | Checkbox — prompts an OAuth consent screen. |
| Terms of Service URI | Optional. |
| Policy URI | Optional. |
| Logo URI | Optional. A **URL** to an image (PNG, transparent background, landscape, ≥420×120px) shown on the consent screen. **Do not conflate with the shared App logo file upload in requirement 2** — different image, different purpose (dashboard tile icon vs. consent-screen branding), and a URL rather than an upload. |

### Other fields

| Field | Notes |
|---|---|
| Email Verification Experience — Callback URI | Optional. Customizes where an email magic-link redirect lands after verification. |
| Groups claim filter | Sign On tab, "OpenID Connect ID Token" section — the **OIDC analog of SAML's Group Attribute Statements** (requirement 17): same Filter Type options (Starts with / Equals / Contains / Matches regex), plus an alternative Expression mode for a custom Okta Expression Language filter. **Confirmed: surfaced as a warning, not a hard client-side block** — Okta's own docs note ID token creation fails at runtime if more than 100 groups match the filter *and* the grant type isn't Authorization Code (with or without PKCE); this form can't fully prevent that combination client-side, so it warns instead. |
| App rate limit | Percentage slider/number input, default 50% of the org's API rate limits, per app. |
| Issuer | Dropdown: Org URL / Custom URL (only meaningful if the target org has a custom URL domain configured) / Dynamic. Niche, included for parity. |
| Trusted Origins (Base URIs for CORS) | Web and Native only. **Confirmed cross-cutting side-effect flag:** submitting this adds entries to the org's own Trusted Origins list — a security-relevant, org-wide setting, not scoped to just this app. Treat with the same weight as any org-level security change, not as an ordinary per-app field. |

### Assignments during creation

Okta's own wizard has an assignment step as part of app creation itself
(grant to everyone / limit to selected groups / skip for now).
**Confirmed: not built as a step inside the wizard** — no separate
assignment fields are added to Step 3/4, and the app is created (Step 4)
before any assignment exists. Instead, **Step 4's success path
immediately navigates to the app's own Assignments tab** (see "App
assignments" below and `01-requirements.md` requirement 2) — so the net
effect is close to Okta's own "assign now" wizard step, without
duplicating assignment fields into the wizard's own form. The app itself
starts unassigned either way (it's a fully separate write, not part of
the create call), and the user can navigate away without assigning
anything — the app remains fully usable, just unassigned until someone
does.

### Deliberately not built for MVP, despite "full parity"

- The dedicated OIDC **"Logout" (Single Logout)** section — Logout
  redirect URIs, cross-app/Okta-initiated logout propagation, Logout
  request URL, Request binding, User session details — is explicitly
  labeled Early Access by Okta. This tool supports the classic, GA
  Sign-out redirect URIs field instead, the same reasoning already
  applied to SAML's excluded EA Logout section.

### Flagged items (not resolved here)

- **Network IP** (restrict which network zones a token can be used from)
  is marked Early Access in some parts of Okta's own reference and not
  explicitly marked in others — inconsistent within Okta's own docs.
  Treated as not-built-for-MVP pending `okta-expert` confirming current
  status, not asserted either way here.
- CIBA's Preferred-authenticator dependency and the client-secret-
  rotation/public-private-key gap noted above.

(The platform list — Web/SPA/Native only, "service" excluded — was
flagged here previously and is now a confirmed decision, not an open
item; see above.)

See `08-open-questions.md` for the consolidated tracking of these.

## SAML app creation form (Okta AIW field parity)

**Confirmed: full parity with Okta's own Application Integration Wizard
SAML field set**
(https://help.okta.com/oie/en-us/content/topics/apps/aiw-saml-reference.htm),
not a curated subset — `01-requirements.md` requirement 18. The form
mirrors Okta's own structure rather than flattening everything: a shared
General Settings step, an always-visible SAML General section, and a
SAML Advanced Settings section collapsed behind a toggle by default, same
as Okta's own wizard.

### General Settings (shared — applies to OIDC apps too, not SAML-only)

| Field | Notes |
|---|---|
| App logo | Optional. PNG/JPG/GIF, <1MB. **Confirmed: forwarded to Okta as part of app creation, not persisted by this tool** — same "pass-through, don't hoard a copy" principle as the Credentials tab (requirement 14). Needs a new `IOktaAppService` capability (a logo upload is a separate Okta API call from app creation itself, not an inline field on the create request — confirm exact sequencing with `okta-expert`). |
| App visibility | Checkbox: hide the app icon from end users' Okta dashboard. |

This is technically broader than "the SAML form" — Okta's own reference
includes these as app-level, not SAML-specific — so they're placed in
Step 3 before the type branch and apply to OIDC creation too, rather than
only appearing on the SAML path. Flag if SAML-only was actually intended.

### SAML Settings — General (always visible)

| Field | Default / behavior |
|---|---|
| Single sign-on URL | Required. POST-based ACS URL. **Validation: cannot contain underscores** (confirmed from Okta's reference). |
| *(checkbox)* Use this URL for Recipient and Destination too | Default: **on**. Unchecked reveals two more fields: **Recipient URL** and **Destination URL**. |
| Audience URI (SP Entity ID) | Required. |
| Default RelayState | Optional. |
| Name ID format | Dropdown, default **Unspecified**. Standard SAML 2.0 NameID format values (Unspecified, EmailAddress, Persistent, Transient, X509SubjectName, WindowsDomainQualifiedName, Kerberos, Entity) — confirm exact string encoding (short label vs. full URN) against the Okta SDK's actual enum before implementation. |
| Application username format | Dropdown (e.g. Okta username, Email, Custom expression) — exact current option set to confirm with `okta-expert`, not asserted here from memory. |
| Update application username on | Dropdown: **Create and update** (default) / **Create only**. **Only shown when Application username format is Custom** — otherwise irrelevant per Okta's own reference. |

### SAML Settings — Advanced Settings (collapsed by default, "Show Advanced Settings" toggle)

**Signing & encryption:**

| Field | Default / behavior |
|---|---|
| Response | Signed / Unsigned. |
| Assertion Signature | Signed / Unsigned. |
| Signature Algorithm | e.g. RSA-SHA256 / RSA-SHA1 — confirm exact enum with `okta-expert`. |
| Digest Algorithm | e.g. SHA256 / SHA1 — confirm exact enum with `okta-expert`. |
| Assertion Encryption | Encrypted / Unencrypted, default Unencrypted. |
| Encryption Algorithm | **Shown only when Assertion Encryption = Encrypted.** |
| Key Transport Algorithm | **Shown only when Assertion Encryption = Encrypted.** |
| Encryption Certificate | **Shown only when Assertion Encryption = Encrypted.** File upload (PEM). |
| Signature Certificate | File upload (PEM). **Important distinction:** this is uploaded *by the admin* to validate incoming SP-signed requests/SLO requests — the opposite direction from, and unrelated to, the signing certificate Okta itself generates to sign its own outbound assertions (shown read-only on the Credentials tab, requirement 14). Don't conflate the two in the UI or the data model. |

**Single Logout & requests:**

| Field | Default / behavior |
|---|---|
| Enable Single Logout | Checkbox. **Appears only after a Signature Certificate is uploaded.** |
| Single Logout URL | **Shown only when Enable Single Logout is on.** |
| SP Issuer | **Shown only when Enable Single Logout is on.** |
| Signed Requests | Checkbox. **Appears only after a Signature Certificate is uploaded.** **Confirmed destructive-toggle warning:** enabling this causes Okta to delete any statically-defined Other Requestable SSO URLs and read SSO URLs from the signed request instead — the two are mutually exclusive on Okta's own side, not just a UI restriction here. If Other Requestable SSO URLs already has rows when this is toggled on, show an explicit warning before submit — consistent with this tool's general confirmation-before-mutation principle, not a silent data loss. |
| Other Requestable SSO URLs | Repeatable list of (URL, Index) pairs, for SP-initiated flows with multiple ACS endpoints. Mutually exclusive with Signed Requests, per above. |

**Other:**

| Field | Default / behavior |
|---|---|
| Assertion Inline Hook | Dropdown of existing Inline Hooks configured in the org, default None. **Flagged, not resolved:** this tool doesn't create/manage hooks itself (out of scope) — this field would need at minimum a read-only "list existing hooks" `IOktaAppService`/`IOktaHookService` capability, and it's not yet decided whether that's worth building for a field this narrow. See `08-open-questions.md`. |
| Authentication context class | e.g. PasswordProtectedTransport, Password — standard SAML 2.0 `AuthnContextClassRef` values; confirm exact list needed with `okta-expert`. |
| Honor Force Authentication | Yes / No. Default not stated in Okta's own reference — confirm with `okta-expert` rather than assume. |
| SAML Issuer ID | Optional override of the default `http://www.okta.com/$(org.externalKey)`. Niche — only needed when more than one sign-in flow exists for the same app. Included for full parity, but expect this to be rarely used. |
| Maximum app session lifetime | Number + time-unit dropdown, plus a "Send value in response" checkbox to include it in the SAML assertion. |
| Attribute Statements / Group Attribute Statements | See "SAML attribute statement configuration" below — kept as its own section given how much detail (validation, data shape, Promote behavior) it needs beyond a single table row. |

**Deliberately not built for MVP, despite "full parity" otherwise:** the
"Logout" section's Early-Access user-initiated/app-initiated SLO
configuration (with its own Response URL/SP Issuer/logout request
URL/request binding sub-fields). Okta's own reference explicitly labels
this an Early Access release, separate from the classic Enable Single
Logout/Single Logout URL/SP Issuer fields above, which are what this
form implements instead. Revisit if/when that EA feature reaches general
availability — see `08-open-questions.md`. This is the same reasoning
already applied to the attribute-statement "legacy configuration vs.
newer expressions" question below: build against Okta's stable,
generally-available surface, flag the shifting one rather than build
against it blind.

### Data model / API mapping

The full field set above maps into the SAML branch of `ConfigurationJson`
(`04-data-model.md`) and, from there, into `IOktaAppService.CreateAsync`/
`UpdateAsync`'s call to Okta's `SamlApplicationSettingsSignOn`-shaped
request. Conditional fields (encryption sub-fields, SLO sub-fields,
Update-username-on) are validated as a group in the Application layer —
e.g. rejecting an `EncryptionAlgorithm` value submitted while
`AssertionEncryption` is `Unencrypted` — rather than trusting the
frontend's conditional-visibility logic as the only enforcement.

### Assignments during creation

**Same treatment as OIDC (above), confirmed for SAML too, not
protocol-specific.** Okta's own AIW wizard shell has an assignment step
(grant to everyone / limit to selected groups / skip for now) regardless
of app type — it's part of the generic wizard flow around SAML/OIDC
configuration, not something specific to either protocol. This tool
doesn't build that as a step inside the SAML form either — no assignment
fields in Step 3/4 — but Step 4's success path (`01-requirements.md`
requirement 2) immediately navigates to the new app's Assignments tab
the same way it does for a freshly created OIDC app. A SAML app starts
unassigned either way and remains fully usable if the user navigates
away without assigning anything.

## SAML attribute statement configuration

Part of Step 3's SAML branch (and, unchanged in shape, the Edit form —
see below). Two separate, independently optional sections, matching how
Okta's own SAML app model actually separates them rather than treating
"attribute statements" as one field:

```
Attribute Statements (optional)
  [+ Add attribute]
  ┌─────────────┬───────────────────────┬─────────────────────────────┐
  │ Name        │ Name Format           │ Value                       │
  ├─────────────┼───────────────────────┼─────────────────────────────┤
  │ email       │ Basic (default)       │ user.email                  │
  │ firstName   │ Basic                 │ user.firstName              │
  └─────────────┴───────────────────────┴─────────────────────────────┘
  Name Format options: Unspecified / URI Reference / Basic
  Value: free text — a literal, or an Okta Expression Language
  expression referencing the Okta user profile. This tool does not
  parse, validate, or evaluate the expression; Okta does, at assertion
  time.

Group Attribute Statements (optional)
  [+ Add group attribute]
  ┌─────────────┬───────────────────────┬──────────────┬──────────────┐
  │ Name        │ Name Format           │ Filter Type  │ Filter Value │
  ├─────────────┼───────────────────────┼──────────────┼──────────────┤
  │ role        │ Basic                 │ Starts with  │ Team-        │
  └─────────────┴───────────────────────┴──────────────┴──────────────┘
  Filter Type options: Starts with / Equals / Contains / Matches regex
```

**Validation (confirmed, matches Okta's own stated rule):** a row's Name
is required, capped at 512 characters, and must be unique across *both*
sections combined — checked as rows are added/edited client-side, and
re-validated server-side on submit (`01-requirements.md` requirement 17).

**Data shape** — stored inside the SAML app's `ConfigurationJson` as a
single array, mirroring Okta's own API shape (`settings.signOn.
attributeStatements`) so translating to/from `IOktaAppService` calls is
mechanical rather than a redesign. A row's presence of `value` vs.
`filterType`/`filterValue` is what distinguishes a regular Attribute
Statement from a Group Attribute Statement — same discriminated-by-shape
convention Okta's own API already uses, not something invented here:

```json
"attributeStatements": [
  { "name": "email", "nameFormat": "Basic", "value": "user.email" },
  { "name": "firstName", "nameFormat": "Basic", "value": "user.firstName" },
  { "name": "role", "nameFormat": "Basic", "filterType": "STARTS_WITH", "filterValue": "Team-" }
]
```

**Confirmed: carries over as-is on Promote** (this project's only
environment-to-environment copy mechanism — "copy" elsewhere in this doc
refers to this same action, not a separate flow) — regardless of
source/target environment — the entire array, every field in every
row, not just that the section exists. See "Promote an app" below.

**Flagged for `okta-expert` to verify before implementation (not
resolved here):**
- **Confirmed source, more precise than earlier phrasing here:** Okta's
  own AIW SAML field reference
  (https://help.okta.com/oie/en-us/content/topics/apps/aiw-saml-reference.htm)
  states that if the org has the **Early Access "Entitlement SAML
  Assertions and OIDC Claims"** feature enabled, Attribute Statements and
  Group Attribute Statements **only appear when editing an app
  integration, not during creation**. If any target org has this EA flag
  on, this tool's Step 3 creation wizard cannot set these two sections at
  creation time in that org — they'd need to be set via a follow-up Edit
  call instead, and the creation wizard/API call should not silently
  assume they were applied. Confirm per-org EA status is even detectable
  via the API before deciding whether to special-case this or require a
  manual Edit-follow-up step universally as the safer default.
- A known issue in the Okta .NET SDK: updating an existing SAML app's
  `attributeStatements` via `ReplaceApplicationAsync` can return success
  while silently applying no change, unless `destinationOverride` is
  explicitly included in the request (even as `null`). If the .NET SDK
  is used for `IOktaAppService.UpdateAsync`, this needs an explicit
  workaround, not an assumption that a 200 response means the update
  actually took effect. See `08-open-questions.md`.

## Edit an app

**Confirmed rule:** editing is available in every environment (including
Production), for both roles, and is never approval-gated — unlike delete,
there is no "immediate vs. pending" branch here.

```
User opens an app they can see (Admin: any app; User: only apps they own)
     │
     ▼
[Edit] click
     │
     ▼
Form pre-populates from the app's current ConfigurationJson, using the
same type-specific (OIDC/SAML) fields as the creation wizard's Step 3 —
full re-edit, nothing locked post-creation. The environment and app type
are fixed (not editable — changing either is a delete + create/promote,
not an edit).
     ▼
Review & submit
     ▼
Backend:
  1. Validate input (server-side, same rules as the creation form)
  2. Authorize: caller must be Admin, or the User who owns this app
  3. Update the app in Okta via the Okta SDK, in the app's own
     environment's org (type-specific update call)
  4. Update the local ConfigurationJson to match
  5. Write an audit log entry (AppEdited), including a diff of which
     fields changed
  6. Invalidate the cached app list/detail entry for this app
  7. Return the updated app to the frontend
     ▼
Frontend shows the updated app
```

**Conflict handling (confirmed — previously open, see
`08-open-questions.md`):** the form loads the app's current `RowVersion`
(`04-data-model.md`) alongside its `ConfigurationJson` and sends it back
unchanged with the submit request. Immediately before writing, the
backend re-checks whether that `RowVersion` still matches:

```
Submit
  │
  ▼
RowVersion matches current row?
  │
  ├── Yes → proceed exactly as steps 1-7 above
  │
  └── No → something changed the app since the form opened (most likely
       the 15-minute drift-sync job picking up a direct-in-Okta change,
       or a second person editing concurrently). The Okta update is NOT
       attempted. Backend returns a conflict response carrying the
       latest ConfigurationJson/RowVersion. Frontend shows what changed
       and offers two explicit choices — no silent default:
         ├── "Load latest" — discard the in-progress edit, repopulate
         │    the form from the newer data
         └── "Keep mine" — resubmit with the freshly-fetched RowVersion,
              pushing the original edit through regardless of what
              changed underneath it (an explicit override, not an
              accidental overwrite)
```

This is a different problem from the one below — this is two independent
writers (a user and the sync job, or two users) disagreeing with each
other; the one below is a single request's own two steps disagreeing
with itself.

**Open question (flag if this needs a stricter answer):** if the Okta
update in step 3 succeeds but the local persist in step 4 fails, Okta and
the local `ConfigurationJson` are now out of sync — same class of
reconcile-visibly gap noted for create/delete in `01-requirements.md`'s
non-functional requirements. Not solved here; tracked as a general
partial-failure concern rather than solved differently per-flow.

## App assignments (people & groups)

Shown as an "Assignments" tab on an app's detail page, once the app
exists. **Confirmed primary entry point: immediately after creation**
(Step 4 above navigates here directly, tab pre-selected) — the other
entry point is navigating to an existing app's detail page later (from
the app list, at any time, not just right after creation). Same tab,
same flow, either way; nothing about the flow below differs based on how
the user arrived. **Confirmed: never approval-gated, in any environment,
for either role** — same immediate shape as editing. Authorization
mirrors Edit: Admin can manage assignments on any app; a User only on
apps they own.

```
User opens the "Assignments" tab on an app they can see
     │
     ▼
Frontend loads current assignments for this app from local DB
(ApplicationAssignment rows — not a live Okta call), split into People
and Group lists
     │
     ▼
User types into the search box (People search and Group search are
separate fields/tabs — Okta's own people-search and group-search are
separate APIs)
     │
     ├── Fewer than 2 characters typed → no search triggered
     │
     └── 2+ characters typed → debounced search
              │
              ▼
         Backend calls Okta's user-search or group-search API (per
         PrincipalType) against the app's own environment's org, using
         that environment's service credential — never the searching
         user's own Okta identity, since regular Users have no direct
         Okta access at all
              │
              ▼
         Return top 5 matches to the frontend, shown as a dropdown
              │
              ▼
User selects a match → [Assign] click
     │
     ▼
Backend: authorize (Admin any app; User only if they own this app) →
validate the selected principal still resolves in Okta → call the Okta
SDK's assign-user/assign-group API for this app → insert an
ApplicationAssignment row (AssignedByUserId = caller) → write audit log
entry (AssignmentAdded) → invalidate this app's assignment cache entry →
return the updated list
```

```
User clicks [Unassign] next to an existing assignment
     │
     ▼
Backend: authorize (same rule as assigning) → call the Okta SDK's
unassign-user/unassign-group API for this app → hard-delete the
ApplicationAssignment row (no soft delete — see `04-data-model.md`) →
write audit log entry (AssignmentRemoved) → invalidate cache → return
the updated list
```

**Sync job also reconciles assignments**, per app, alongside config and
active/inactive status — see `09-drift-sync.md`.

## App credentials

Shown as a **Credentials** tab on an app's detail page, styled after
Okta's own Admin Console rather than inventing a new layout — the values
and the audience (whoever integrates against this app) are the same
ones Okta itself already shows.

```
User opens the "Credentials" tab on an app they can see
(Admin: any app; User: only apps they own — same rule as Edit)
     │
     ▼
Backend calls IOktaAppService.GetCredentialsAsync (app's OktaAppId +
environment) — always live, never read from ConfigurationJson or any
cache
     │
     ├── OIDC app → Client ID (shown in the clear) + Client Secret
     │    (masked by default, "Reveal" toggle, Copy button on each —
     │    same interaction as Okta's own General tab; not a one-time
     │    reveal, matching Okta's own console behavior)
     │
     └── SAML app → SSO URL, Issuer, Audience, and the signing
          certificate (Download button, same as Okta's own console)
     │
     ▼
Write an audit log entry (AppCredentialsViewed) — a read, not a
mutation, but still a secret-disclosure event worth a trail
     │
     ▼
Frontend renders the tab; nothing returned here is persisted, logged
verbatim, or cached by this tool
```

Authorization is identical to viewing the app itself — no separate
credentials-specific permission exists. See `05-security.md` for the
handling rules (a deliberate, narrow exception to this app's general
"never return secrets in API responses" rule) and `02-architecture.md`
for `GetApplicationCredentialsQuery`.

## Reassign an app's owner

**Confirmed: Admin-only**, from an app's detail page — a `User`, even the
current owner, cannot reassign ownership themselves.

```
Admin opens an app's detail page → [Reassign owner] click
     │
     ▼
Frontend: Admin picks a new owner from this tool's own Users list (not
an Okta directory search — ownership is a concept internal to this
tool, unlike the Assignments tab above, which searches Okta's own
people/group directory)
     │
     ▼
Admin is prompted for a reason (free-text, required — the confirmation
dialog will not submit without it; in addition to, not instead of, the
general "confirmation before mutation" step every action already gets)
     │
     ▼
Backend:
  1. Authorize: caller must be Admin (rejected even if the caller is the
     app's current owner but not an Admin)
  2. Validate the selected new owner exists as a User in this tool
  3. Update Application.OwnerUserId — no Okta call; Okta has no concept
     of "owner" the way this tool does, so this is purely a local-record
     change
  4. Write an audit log entry (OwnerReassigned): previous owner, new
     owner, the Admin who made the change, and the required reason
  5. Invalidate the cached app list/detail entry for both the previous
     and new owner (a User's list view is scoped by OwnerUserId, so both
     are now stale)
     ▼
Frontend shows the app under its new owner
```

No Production/Staging distinction and never approval-gated — there's no
second Admin review step, since performing the action already requires
an Admin in the first place. See `01-requirements.md` requirement 15 and
`04-data-model.md` for the `OwnerReassigned` audit action.

## Promote an app: Staging → Production only

**Confirmed rule:** promotion only runs in one direction — from **Staging**
to **Production**. There is no Production → Staging path, and no path
between any other environment pair, even if more environments are added
later (see the `PromotesToEnvironmentId` note in `04-data-model.md`). A
User can create an app directly in any environment (see the creation
wizard above), but the *promotion* action specifically only ever runs
Staging → Production.

```
User selects an existing Staging app (must be visible to them per role/ownership)
     │
     ▼
Backend rejects up front if the app's environment isn't Staging, or if no
Production environment is configured to promote it into — there is only
one valid target, so there's no "pick a target environment" step
     │
     ▼
Backend:
  1. Authorize: caller must be Admin, or User who owns the source (Staging) app
  2. Fetch the source app's full configuration from Okta (Staging org)
  3. Determine which fields carry over as-is vs. must be remapped:
       - Carries over: app type, grant types/scopes, name ID format,
         application username format, honor-force-authn, authentication
         context class, response/assertion signing choices, signature/
         digest algorithm, assertion encryption choice, max session
         lifetime, App logo, App visibility, general labels/branding,
         **plus, for OIDC apps specifically:** client authentication
         *method* (which option is chosen — not the Client ID/Secret
         *values*, see the dedicated note below), PKCE setting, DPoP
         requirement, grant-type sub-config (Refresh Token rotation
         choice/period, Implicit's Allow-ID/Allow-Access sub-checkboxes),
         Login initiated by, OIDC-specific Application visibility, Login
         flow choice, Require consent, Terms of Service URI, Policy URI,
         groups claim filter (same reasoning as Group Attribute
         Statements below — filter type/value are portable name
         patterns, not org-scoped IDs), and App rate limit percentage —
         and — **confirmed, resolving what was previously an open
         question here** — Attribute Statements and Group Attribute
         Statements *in full*, every field of every row (Name, Name
         Format, Value, Filter Type, Filter Value), copied
         unconditionally regardless of source/target environment. No
         remapping step or user confirmation is needed for this section
         specifically, unlike the environment-specific fields below.
         Rationale: statement values are typically Okta Expression
         Language expressions referencing the Okta user profile (e.g.
         `user.email`), not environment-specific literals, so there's
         normally nothing in them *to* remap — see
         `01-requirements.md` requirement 17. (If someone has hand-
         entered an environment-specific literal into a Value or Filter
         Value field rather than an expression, it will still carry over
         unchanged — that's a possible footgun worth a UI hint, not a
         reason to add a remapping step that would otherwise be wrong
         for the common case.) The rest of the carries-over list follows
         the same logic: these are behavioral/protocol choices, not
         endpoints or org-scoped identifiers, so there's nothing
         environment-specific in them to remap either.
       - Must be remapped (environment-specific): redirect URIs (both
         SAML's Single sign-on URL/Recipient URL/Destination URL and
         OIDC's Sign-in/Sign-out redirect URIs), Other Requestable SSO
         URLs, audience URI/entity ID, Single Logout URL, SP Issuer,
         SAML Issuer ID, Signature Certificate, Encryption Certificate,
         and, for OIDC, **Initiate login URI, the Email Verification
         Callback URI, Trusted Origins Base URIs (also has the org-level
         side-effect noted in requirement 19 — confirm the target org's
         CORS impact is reviewed, not just remapped), and Issuer (its
         Custom URL option is specific to whichever org's custom domain
         is configured — re-select per target org, don't assume Custom
         URL even means the same thing)** — these reference either the
         SP's own per-environment domains/endpoints or values scoped to
         whichever Okta org is being promoted into, and cannot be copied
         verbatim. **Assertion Inline Hook and CIBA's Preferred
         authenticator are also must-be-remapped, not carries-over**,
         even though neither is a URL: both reference IDs only valid
         within the org they were created in, so a Staging-org ID is
         meaningless in Production — both must be re-selected (or
         explicitly left unset) against Production's own configuration,
         never copied as-is.
       - **Client ID and Client Secret are neither "carries over" nor
         "must be remapped" in the usual sense — they're freshly
         generated by Okta for the new Production app**, the same way
         any fresh creation gets its own credentials. Promote's use of
         the standard creation path (step 5 below) already produces
         this; there's no value to copy from the Staging app at all here,
         unlike every other field in this list.
       - **Default RelayState (SAML) and Logo URI (OIDC's consent-screen
         branding image URL)** are ambiguous enough to call out on their
         own: treated as carries-over by default (typically a constant or
         generic branding asset, not an endpoint), but since either *can*
         legitimately hold an environment-shaped value, the review step
         (step 4 below) should surface both for confirmation alongside
         the fields that are always remapped, rather than silently
         carrying them over unreviewed the way the unambiguous
         carries-over fields are.
  4. Present the remapped fields to the user for confirmation before
     creating anything (do not silently guess environment-specific values)
  5. On confirmation: create the app in Okta (Production org) via the
     Okta SDK, using the same type-specific creation path as a fresh
     creation
  6. Persist the local app record for the new Production app, linked to
     the Staging source app for traceability (`SourceApplicationId`)
  7. Write an audit log entry (`AppPromoted`) capturing the Staging source,
     the Production result, and the fields that were remapped
  8. Invalidate the cached app list for Production
     ▼
Frontend shows the newly created Production app
```

The confirmation step (4) exists specifically because environment-specific
fields cannot be safely automated — a wrong redirect URI or SSO URL copied
into production is a functional break, not just a cosmetic one. See the
`okta-expert` agent for the authoritative list of which OIDC/SAML fields
are environment-specific per Okta's data model, and `cybersecurity-engineer`

**Assignments are not part of this flow.** `ApplicationAssignment` rows
aren't in either carries-over or must-be-remapped list above, and step 5
reuses the fresh-creation path, so the working (not yet explicitly
confirmed) assumption is that the new Production app starts unassigned,
same as any newly created app. See `08-open-questions.md`.
for review of the confirmation UX from a "don't let this become a bypass of
human review" angle.

## Import (adopt) an existing Okta app

Each environment is its own Okta org, so "import" means asking one org for
apps this tool doesn't know about yet. **Not limited to Staging or
Production specifically** — same as everywhere else in this tool, any
configured environment can be imported from; there's nothing in the
design that assumes only two environments or singles out a particular
one.

```
Admin clicks [Import apps]
     │
     ▼
Step 1 — Choose environment
     │  Admin picks which environment's Okta org to scan (any configured
     │  environment — Staging, Production, or any others that exist).
     │  This mirrors the creation wizard's own Step 1.
     ▼
Backend: list all apps in the chosen environment's Okta org (Okta SDK),
         then filter out any whose OktaAppId already has a local
         Application record — the remainder are "unmanaged" apps
     │
     ▼
Frontend shows the unmanaged apps (name, inferred type, Okta app ID)
     │
     ▼
Admin selects which app(s) to import and assigns an owner (User or
themselves) to each
     │
     ▼
Backend, per selected app:
  1. Authorize: Admin only (see assumption note below)
  2. Read the app's full configuration from Okta to infer Type (OIDC/SAML)
     and populate ConfigurationJson (same shape as a tool-created app)
  3. Persist a local Application record: Origin = ImportedFromOkta,
     OwnerUserId = assigned owner, OktaAppId, EnvironmentId = the
     environment chosen in Step 1
  4. Write an audit log entry (AppImported): who imported it, which app,
     which environment, who it was assigned to
  5. Invalidate the cached app list for that environment
     ▼
Frontend shows the newly tracked app in the normal app list
```

**Assumption (flag if wrong):** import is Admin-only, since it exposes apps
across the whole org that may not belong to the requesting user and
requires assigning ownership on someone else's behalf — this wasn't
explicitly asked about, so treat it as a default to confirm rather than a
locked decision.

## Delete an app (immediate vs. Production approval-gate)

**Confirmed rule:** the approval gate applies only to a **User deleting a
Production app**. Every other combination — Admin deleting anything, or a
User deleting their own Staging app — deletes immediately. Any Admin
(no separate approver role) can review a pending request.

**Confirmed precondition (new): the app must already be deactivated.**
The `[Delete app]` action is not shown (or is disabled with an explanatory
tooltip) for any app where `IsActive = true` — the person has to
deactivate it first (see "Deactivate / reactivate an app" below). This
check is enforced server-side as well, not just by hiding the button, and
is re-checked again at approval time for a gated (Production) request, in
case the app was reactivated in the window between the request being
created and an Admin reviewing it.

```
[Delete app] click (only reachable if IsActive = false)
     │
     ▼
Backend: re-verify IsActive = false (defense in depth, not just a hidden
         button) — reject with a clear error if somehow still active
     │
     ▼
Backend: authorize — caller is Admin, or caller is the owning User
     │
     ├── Caller is Admin (any environment) ─────────────────────┐
     │                                                            ▼
     │                                           Delete immediately: call
     │                                           Okta SDK delete, soft-
     │                                           delete local record, write
     │                                           audit log entry (AppDeleted),
     │                                           invalidate cache
     │
     └── Caller is the owning User
              │
              ├── App's environment = Staging ──────────────────┐
              │                                                   ▼
              │                                  Delete immediately, same
              │                                  path as the Admin branch
              │                                  above (AppDeleted, no gate)
              │
              └── App's environment = Production ───────────────┐
                                                                   ▼
                                      Create an ApprovalRequest
                                      (RequestedAction = Delete, Status =
                                      Pending); write audit log entry
                                      (AppDeletionRequested); app stays
                                      deactivated (already was, per the
                                      precondition above) and visible,
                                      shown with a "pending deletion
                                      approval" badge
                                                                   │
                                                                   ▼
                                      Any Admin reviews the pending request:
                                        ├── Approve → re-verify IsActive =
                                        │   false, then call Okta SDK
                                        │   delete, soft-delete local
                                        │   record, audit log
                                        │   (AppDeletionApproved +
                                        │   AppDeleted), invalidate cache
                                        └── Reject → app remains untouched
                                            (still deactivated, not
                                            deleted), audit log
                                            (AppDeletionRejected) with
                                            an optional reason
                                                                   │
                                      (Requester may instead Cancel their
                                      own pending request → audit log
                                      AppDeletionCancelled, app remains
                                      deactivated but not deleted)
```

## Deactivate / reactivate an app

**Confirmed: neither is ever approval-gated, in any environment, for
either role.** Both are full mirror images of the Edit flow in that
sense, not of Delete. (An earlier draft gated a User's Production
deactivation the same way as delete, sharing the `ApprovalRequest`
entity; that was reconsidered and removed — deactivate is always
immediate now.)

```
[Deactivate app] click
     │
     ▼
Backend: authorize — caller is Admin, or caller is the owning User
     │
     ▼
Immediate in every environment, no ApprovalRequest involved at all: call
Okta SDK deactivate, set IsActive = false locally, write audit log entry
(AppDeactivated), invalidate cache
```

```
[Reactivate app] click (only shown for a currently-deactivated app the
caller can act on: Admin any app, User only apps they own)
     │
     ▼
Backend: authorize — caller is Admin, or caller is the owning User
     │
     ▼
Immediate in every environment, no ApprovalRequest involved at all: call
Okta SDK activate, set IsActive = true locally, write audit log entry
(AppReactivated), invalidate cache
```

**Resolved:** deletion never transparently auto-deactivates on the
caller's behalf. Instead, deactivation is a hard precondition — delete
simply isn't reachable until the app is already deactivated (see "Delete
an app" above). Since deactivating is always immediate, this precondition
never turns into a second approval cycle — a User only ever requests
approval once (for the delete itself) to fully remove a Production app.

The environment check happens in the Application layer, before deciding
whether to call Okta directly or create an `ApprovalRequest` — a User
cannot bypass the gate by hitting a delete endpoint with a Production app's
ID and expecting the Staging (immediate) code path.

See `04-data-model.md` for the `ApprovalRequest` entity and
`05-security.md` for the authorization rules around who can approve.
