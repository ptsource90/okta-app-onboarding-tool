# Session & token lifecycle (frontend login, not the per-environment Okta orgs)

**Scope check, since this is easy to conflate with something else in this
project:** this doc is about the authorization server the tool's own
users log into (`05-security.md`'s "Authentication" section — Firebase,
Okta, or a generic OIDC/SSO provider). It is **not** about the
per-environment Okta orgs (Staging/Production/etc.) that this tool
manages apps in — those are accessed via the backend's own per-environment
OAuth 2.0 service app credential (`private_key_jwt`/`client_credentials`,
not a static token — see `04-data-model.md`'s "Backend-to-Okta
authentication" section, `06-tech-stack.md`), never the logged-in user's
own token. Two separate authorization-server relationships; don't merge
them.

**Confirmed scope narrowing:** the frontend auth library is now
committed to Okta specifically (`@okta/okta-react`/`@okta/okta-auth-js`,
below) — these are Okta-only SDKs, not generic OIDC clients. The
backend's own validation stays provider-agnostic exactly as
`05-security.md` describes (any JWT, any issuer, validated via JWKS), so
a Firebase or generic-OIDC deployment is still architecturally possible
on the backend. But on the frontend, swapping the login IdP away from
Okta is no longer a config change — it would mean swapping out
`@okta/okta-react` for a different integration entirely, a materially
bigger lift than the "swappable" framing elsewhere in this doc previously
implied. Worth being explicit about that shift rather than letting the
older framing stand unchallenged.

## Confirmed figures (verified against Okta's docs before committing)

For a deployment where the chosen login IdP is **Okta's Org Authorization
Server** specifically (as opposed to a Custom Authorization Server, which
allows tuning these):
- **Access token lifetime: 60 minutes, fixed** — not configurable on the
  Org AS.
- **Refresh token max lifetime: 90 days, fixed** — also not configurable
  on the Org AS.
- **No idle lifetime on the Org AS refresh token** — it does not expire
  early just from a period of inactivity, only at the 90-day mark.

**This is Okta-specific.** `01-requirements.md` and `06-tech-stack.md`
also allow Firebase or a generic OIDC/SSO provider as the login IdP for a
given deployment — those have their own, different token lifetime
characteristics. The confirmed design below (1-hour session, tied to a
60-minute access token) is written for the Okta case; a Firebase or
generic-OIDC deployment would need its own equivalent numbers worked out
before this design applies as-is. Flagging rather than assuming it
generalizes.

## Confirmed design

- **The app's own tracked "session" length is 1 hour, tied directly to
  the access token's own natural lifetime** — not a separately-tracked,
  arbitrary idle timer. There's one clock, not two.
- **Silent renewal:** any network request from the frontend to this
  tool's backend that would otherwise use an expired (or about-to-expire)
  access token triggers a token refresh using the refresh token first,
  transparently — the user never sees this happen as long as they're
  making requests. Since the Org AS refresh token has no idle lifetime,
  this is safe for as long as the user is active within the 90-day
  window.
- **Countdown modal for the idle case:** a client-side timer (independent
  of network activity — it has to be, since an idle user by definition
  isn't making requests that would otherwise trigger silent renewal) is
  scheduled for **1 minute before** the current access token's natural
  expiry. If the user has been idle long enough for this timer to fire,
  a modal appears with a countdown and two choices:
  - **Stay signed in** → explicitly triggers the same refresh-token
    exchange as silent renewal, gets a new access token, and reschedules
    the timer against the new expiry.
  - **Log out now**, or the countdown reaching zero with no action →
    revoke the refresh token (call the IdP's revoke endpoint) and
    redirect to the login page.
- Every successful silent renewal (whether triggered by a network request
  or by "stay signed in") **reschedules** the countdown timer against the
  new token's expiry — the 1-hour clock effectively never runs out for
  someone actively using the tool; it only becomes visible during a
  genuine idle stretch.
- **Token storage (confirmed: explicitly override the library's
  default, don't inherit it):** `@okta/okta-auth-js`'s `TokenManager`
  defaults to **`localStorage`** (confirmed from Okta's own SDK
  reference), falling back to `sessionStorage` or `cookie` if
  unavailable — none of which match this project's own in-memory-only
  preference (module-level variable or React context, not any browser
  storage API), given the XSS exposure risk of anything written to
  browser storage. The SDK does support this via
  `tokenManager: { storage: 'memory' }` in the `OktaAuth` constructor
  config — **this must be set explicitly**, since accepting the
  library's default silently reintroduces the exact risk this doc
  already decided against. (This isn't a fringe concern — Okta's own
  SDK repository has an open community feature request to change the
  default specifically because it conflicts with OWASP guidance on token
  storage.) A page refresh would lose the in-memory token and require a
  silent re-auth via the IdP (e.g. `oktaAuth.token.getWithoutPrompt()`
  or an equivalent silent redirect) rather than reading a stored token
  back.
- **Backend validation (confirmed, restated from `05-security.md`):** the
  backend never trusts the frontend's claim of who's logged in — every
  request's access token is independently validated (signature, issuer,
  audience, expiry) against the login IdP's JWKS. This is unrelated to,
  and doesn't replace, the per-environment OAuth 2.0 service app
  credentials (`private_key_jwt`) the backend separately holds for
  talking *to* the managed Okta orgs — see `04-data-model.md`.

## Confirmed library: `@okta/okta-react` (+ `@okta/okta-auth-js`)

Supersedes the earlier "suggested, not yet confirmed" `oidc-client-ts`
pick. `@okta/okta-react` is Okta's own SDK, built on top of
`@okta/okta-auth-js` (a required peer dependency, `>=5.3.1`) — current
stable major version series is `6.x`.

**What the library provides out of the box:** the `<Security>` provider
component, a `useOktaAuth()` hook exposing `{ oktaAuth, authState }`,
`<LoginCallback>` for handling the redirect back from Okta's hosted
login, PKCE-based Authorization Code Flow, and `oktaAuth`'s underlying
`TokenManager`/`AuthStateManager` primitives (auto-renewal,
expiry-approaching events, synchronous token access after the first
auth-state determination).

**What it does *not* provide, and this project still has to build:** the
specific 1-hour-session/1-minute-countdown-modal UX confirmed above.
`@okta/okta-auth-js` gives the primitives to build it on
(`tokenManager`'s `expireEarlySeconds` config and `expired`/`renewed`
events, `authStateManager.subscribe()`), not that exact interaction out
of the box. Concretely:
- **Silent renewal** (network-triggered): configure
  `tokenManager.autoRenew: true` (the default) so an expired/expiring
  token is renewed transparently when accessed via `tokenManager.get()`
  — this covers the "any backend request triggers a refresh" behavior
  above without custom renewal logic.
- **Countdown modal** (idle case): still custom — a component
  subscribing to `oktaAuth.authStateManager.subscribe()` (or the
  `tokenManager`'s `expired` event) to know the current access token's
  expiry, scheduling its own 1-minute-before timer, and calling
  `oktaAuth.token.renew()` (stay signed in) or `oktaAuth.signOut()` (log
  out) — the SDK doesn't ship a countdown-modal component.

## Routing: `react-router-dom`

**Confirmed: `react-router-dom` 5.x**, using `@okta/okta-react`'s
packaged `<SecureRoute>` component directly — no custom protected-route
component needed, since `<SecureRoute>` only supports 5.x (confirmed
from the SDK's own documentation) and this project is deliberately
staying on 5.x rather than adopting 6.x+ and working around the
incompatibility. This also means the SDK's own documented usage patterns
apply directly rather than needing adaptation: `<Switch>` wrapping
routes, `component`/`render` props on `<Route>` (not `element`), and
`useHistory()` (not `useNavigate()`) for the `restoreOriginalUri`
callback `<Security>` requires — all v5 APIs, matching what
`@okta/okta-react`'s own README examples use.

**Worth flagging for whoever implements, not a blocker:** `react-router-dom`
5.x is an older major version (6.x released in 2021) — a deliberate,
confirmed trade-off favoring direct SDK compatibility over the latest
router API, not an oversight. Engineers used to 6.x/7.x conventions
(`<Routes>`, `element` props, `useNavigate()`, loaders) should expect v5
idioms throughout this codebase's routing code instead.

## Open questions (flag, don't assume)

(`react-router-dom` major version + `SecureRoute` approach was flagged
here previously and is now a confirmed decision — 5.x with the SDK's
packaged `SecureRoute` — see the "Routing" section above, not an open
item.)

- **Multi-tab behavior** — if a user has this tool open in two browser
  tabs, does logging out (or the countdown timing out) in one tab need to
  synchronize to the other (e.g. via `BroadcastChannel` or a
  `storage` event)? Not designed here.
- **Proactive vs. reactive silent renewal** — should the frontend renew
  slightly *before* the access token actually expires (e.g. at the
  55-minute mark, avoiding a request that fails and then retries), or
  renew reactively only after a request comes back unauthorized? The
  design above assumes proactive (the countdown timer already implies a
  known expiry to schedule against) but this hasn't been explicitly
  confirmed as the intended mechanics for the *network-triggered* silent
  renewal path specifically, as opposed to the idle-countdown path.
- **Firebase / generic-OIDC equivalents** — see the scope note above;
  this doc only firms up the Okta-as-login-IdP case, and with
  `@okta/okta-react` now the confirmed frontend library, supporting a
  non-Okta login IdP would mean a separate frontend integration, not a
  config swap. Not designed here, and not assumed to be a priority.
