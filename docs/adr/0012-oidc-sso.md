# 12. SSO: OIDC only, one identity provider per org, JIT provisioning

## Status
Accepted

## Context
ADR 0010 explicitly deferred SSO "until a real customer names an identity
provider to federate to." That's now the top blocking item on the roadmap's
"first design-partner pilot needs" list - password accounts alone aren't
enough once a real customer's IT/security team is in the loop.

## Decision
- **OIDC only, not SAML.** Authorization Code + PKCE covers the identity
  providers a pilot customer is actually likely to name - Okta, Google
  Workspace, Azure Entra, Auth0 - with far less code and attack surface than
  SAML. SAML/WorkOS stays a documented, deferred option if a specific
  customer's IdP demands it; nothing here forecloses adding it later behind
  the same `OrgSsoConfigStore` shape.
- **No new heavy dependency.** `pyjwt[crypto]` (already a dependency, extended
  with the `cryptography` extra for RS256/ES256 verification) plus `httpx`
  (already used by every search provider) implement the whole flow -
  discovery, PKCE, token exchange, `id_token` verification via
  `PyJWKClient`. No `authlib`.
- **One identity provider per org**, not multi-IdP or a WorkOS-style
  aggregator. `org_sso_config` is a single row per `org_id` (migration 0012).
  `allowed_email_domain` is globally unique across orgs - it's the only thing
  `/auth/sso/login` has to resolve an org from, given nothing but the email a
  user types in (no separate per-tenant login URL to remember or bookmark).
- **Just-in-time provisioning, no SCIM.** The first successful SSO login for
  an email in the allowed domain auto-creates a `MEMBER` account with
  `email_verified_at` set immediately - the IdP already vouched for the
  email, so there's no reason to re-run this codebase's own verify-link flow
  on top of it. SCIM (provisioning/deprovisioning driven by the IdP, not by
  login) stays on the roadmap as later work; JIT alone means an employee
  removed from the IdP can still use an existing session until it expires
  (`AUTH_TOKEN_TTL_HOURS`), same revocation trade-off ADR 0010 already
  accepted for password accounts.
- **Session issuance is unchanged.** `/auth/sso/callback` calls the exact
  same `issue_token()` the password login path uses, producing the same
  `AuthClaims` shape. Nothing downstream of "there is a valid session JWT" -
  every other endpoint, the whole web app - needed to change at all.
- **No server-side session store**, matching ADR 0010's stateless-JWT
  approach: the PKCE `code_verifier` and OIDC `nonce` have to survive the
  redirect round trip to the IdP and back, so they're carried inside a
  short-lived (10 min), signed JWT handed to the IdP as the opaque `state`
  parameter (`SsoState` / `issue_sso_state_token` in `auth.py`) rather than
  kept in a session table.
- **`users.password_hash` is now nullable**; a new `users.sso_subject`
  column holds the IdP's `sub` claim, unique per `(org_id, sso_subject)`. A
  `CHECK` constraint (migration 0012) requires at least one of
  `password_hash`/`sso_subject` to be set - enforced at the database level,
  not just in the Pydantic model, so a bug anywhere upstream can't insert a
  user with no way to authenticate at all.
- **Existing password accounts link automatically on first SSO login**: if
  an email already has a password-based account in the target org,
  `/auth/sso/callback` attaches the IdP's `sub` to that same row rather than
  creating a duplicate. An email that belongs to a *different* org is
  rejected (409) - `get_by_email`'s global uniqueness already guarantees
  there's at most one such row to find.
- **`/auth/sso/callback` hands the browser a bare token, not a session.** It's
  a redirect (`/sso-callback#token=...`, a URL fragment so the token never
  lands in a server access log), not a fetch response, so it can't return the
  full `AuthResponse` body the password login/register endpoints do. A new
  `GET /auth/me` (any valid bearer token, returns the same session fields
  minus the token itself) lets the frontend turn "I have a token" into a full
  `Session` without minting a second one.
- **`login_hint` is passed to the authorization URL** (a standard OIDC
  param, not our invention) - the email the user already typed into the login
  form pre-fills the IdP's own login screen. Real IdPs still make the user
  authenticate; it's a convenience, not a trust boundary - `/auth/sso/callback`
  never trusts anything about *who* logged in except the verified `id_token`.

## Consequences
- **`client_secret` is stored as plaintext** in `org_sso_config` today - a
  real gap for a security review to flag, not an oversight. Every other
  secret in this codebase (`ANTHROPIC_API_KEY`, `SMTP_PASSWORD`, ...) lives
  in an env var scoped to one deployment; this is the first secret that's
  both per-tenant and DB-resident, which is a different threat model
  (database access exposes every configured org's IdP credential at once).
  Application-level encryption at rest is the natural fast-follow, not done
  here to keep this phase scoped to the OIDC flow itself.
- The `id_token`'s signature is verified against asymmetric algorithms only
  (`RS256`/`ES256`) - `HS256`/`none` are explicitly excluded in
  `oidc.verify_id_token`, since accepting a symmetric algorithm for a token
  signed by a third party would open an algorithm-confusion forgery path.
- SSO login is public/unauthenticated by construction - it's how a user
  authenticates in the first place. `/auth/sso/login` confirms only that an
  org has SSO configured for a domain (a fairly standard disclosure - Okta,
  Slack, and others do the same "enter your email to find your workspace"
  pattern); `/auth/sso/callback` independently re-checks the returned
  `id_token`'s email domain against the configured `allowed_email_domain`
  rather than trusting the earlier resolution, since a misconfigured or
  compromised IdP could otherwise hand back an out-of-domain identity.
- **`PUBLIC_APP_BASE_URL` has to be listed explicitly in `docker-compose.yml`'s
  `api.environment` block**, not just set in `.env.local` - `--env-file` only
  drives variable *substitution* inside `docker-compose.yml` (the `${VAR}`
  syntax), it does not automatically forward every `.env.local` entry into the
  container's process environment. Missing this the first time round produced
  a silent, hard-to-diagnose failure: `_public_app_base_url()` correctly
  raised `HTTPException(500, ...)`, but Starlette doesn't log a traceback for
  deliberate `HTTPException`s (only genuinely unhandled exceptions get one),
  so the API logs showed nothing but a bare `500` with no explanation. Worth
  remembering for the next env var that only some services need.
- **e2e testing runs against a real local OIDC provider**
  (`web/e2e/fake-idp.ts`), not a mocked transport - real discovery, real PKCE
  verification, a real RS256-signed `id_token` checked against a real JWKS,
  same "no mocked providers" bar as the rest of this suite. Identity comes
  from `login_hint` rather than a test-only side channel. It also surfaced a
  second local-dev-only networking wrinkle worth knowing: the API runs in a
  Docker container while the browser and the fake IdP run on the host, and
  Docker Desktop's `host.docker.internal` (container → host) and `127.0.0.1`
  (host → host) are *not* interchangeable - the fake IdP's discovery document
  hands out a `host.docker.internal` issuer/token/jwks trio (server-to-server,
  API-only calls) alongside a `127.0.0.1` `authorization_endpoint` (the only
  endpoint the browser itself ever navigates to). Nothing about this is
  specific to the fake IdP; a real IdP wouldn't have this problem since it's
  reachable at one public hostname from anywhere.
- Still open, not addressed by this ADR: SCIM/deprovisioning, SAML, multiple
  IdPs per org, and `client_secret` encryption at rest.
