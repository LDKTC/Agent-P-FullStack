---
name: google-auth
description: Google Sign-In / OAuth 2.0 / OIDC-specific expertise — choosing the right flow (server-side authorization code + PKCE, or Google Identity Services for sign-in only; never implicit, no client secret in a public client), the ID token vs access token distinction and full server-side verification (signature via Google's JWKS, `iss`, `aud`, `exp`, `email_verified`, and `sub` as the stable user key), `state`/nonce for CSRF and replay, exact redirect-URI matching and safe post-login redirects, minimal and incremental scopes, refresh tokens (`access_type=offline`, encrypted at rest, revocation handling), session cookies minted by your own app rather than storing Google tokens in the browser, and the account-linking hazards around unverified email. Use when `package.json` lists `google-auth-library`, `@react-oauth/google`, `passport-google-oauth20`, or an auth framework's Google provider (NextAuth/Auth.js, Lucia, Supabase, Firebase), when `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` appear in env files, or when asked directly (Thai or English) to ทำ Login with Google / เข้าสู่ระบบด้วย Google หรือแก้ปัญหา OAuth. Not a replacement for api-integration-dev's SDK wiring or security-auditor's audit — this is the protocol-level expertise both draw on.
---

# Google Auth (OAuth 2.0 / OpenID Connect)

The protocol layer under "Sign in with Google". `api-integration-dev` wires
the SDK and `backend-dev` owns the session and the user record; this skill
is what makes the result actually authenticate someone. Most Google-sign-in
bugs are not SDK bugs — they are a token used for the wrong purpose, a check
that was skipped, or a redirect that wasn't pinned.

## Step 1 — Confirm the task and read what's already wired

Check `package.json` for `google-auth-library`, `googleapis`,
`@react-oauth/google`, `passport-google-oauth20`, or an auth framework's
Google provider; `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` in
`.env.example`; or an existing `/auth/google/callback` route. If a framework
already owns the flow (Auth.js, Lucia, Supabase Auth, Firebase Auth),
configure *it* correctly rather than hand-rolling a parallel flow — two
session mechanisms in one app is its own vulnerability. This skill then
applies to the parts the framework leaves to you: scopes, verification of
anything you decode yourself, session cookie settings, and redirect
handling.

## Step 2 — Pick the right flow

- **Authorization Code + PKCE, exchanged on your server** — the default for
  a web app that needs a session, and mandatory if you need any Google API
  access beyond identity. The `code` is exchanged server-side using the
  client secret; the browser never sees a Google token.
- **Google Identity Services (One Tap / Sign in with Google button)** —
  correct when all you need is identity. It returns an ID token (a JWT) that
  your server verifies. No client secret involved, no access token, no
  Google API access.
- **Implicit flow (`response_type=token`)** is deprecated and must not be
  used — tokens land in the URL fragment, in history and logs.
- **Public clients** (SPA, mobile, desktop, Electron) have **no client
  secret**. A secret shipped in a bundle is public; if one is there, treat
  it as compromised and rotate it. Use PKCE, and on desktop use a loopback
  redirect or the system browser rather than an embedded webview (Google
  blocks embedded webviews for sign-in, and they defeat the user's ability
  to see the real origin).
- Register **exact** redirect URIs per environment (dev, staging, prod).
  Google matches them exactly — no wildcards, no path prefixes, scheme and
  port included.

## Step 3 — ID token vs access token

These are not interchangeable, and confusing them is the most common design
error:

- **ID token** (`id_token`, a JWT) answers *who the user is*. It is for your
  application, and it is what you verify to establish identity.
- **Access token** answers *what Google APIs the bearer may call*. It is
  opaque to you, is meant for Google's servers, and must never be used as
  proof of identity or as a session credential in your app.
- Neither one is your session. After verifying identity once, mint **your
  own** session (see Step 5); don't ask the browser to hold Google tokens
  and re-present them.

## Step 4 — Verify the ID token properly, server-side

Use Google's library (`google-auth-library`'s `verifyIdToken`, or an
equivalent OIDC library with Google's JWKS) — never `jwt.decode` without
verification, and never trust a token the client says is valid. The checks
that must all pass:

- **Signature** against Google's published JWKS, with a supported algorithm
  (`RS256`) — reject `alg: none` and reject an algorithm the code didn't
  expect.
- **`aud`** equals *your* client ID. Skipping this accepts a valid Google
  token minted for a different application — a complete authentication
  bypass.
- **`iss`** is `https://accounts.google.com` or `accounts.google.com`.
- **`exp`** (and `iat`) still valid, with only small clock skew allowed.
- **`nonce`** matches the one you generated for this login, when the flow
  supplies one — that's the replay defense.
- **`email_verified`** is true before trusting `email` for anything,
  especially account linking.
- Use **`sub`** as the stable user identifier in your database. Email
  addresses change and can be reassigned; `sub` does not.
- If it's a Workspace-only app, check the `hd` (hosted domain) claim
  server-side — filtering with the `hd` request parameter alone is a hint to
  the login UI, not enforcement.

## Step 5 — Sessions after login

- Mint your own session on success: an opaque session ID (or your own signed
  JWT) in a cookie that is `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict`
  where the flow allows), `Path=/`, with a sane expiry.
- Regenerate the session identifier at login to prevent session fixation,
  and invalidate server-side on logout — deleting a cookie isn't a logout if
  the session is still valid.
- Don't store ID tokens, access tokens, or refresh tokens in
  `localStorage`/`sessionStorage`, where any XSS reads them.
- Provide real logout: clear your session, and revoke the Google refresh
  token if you hold one.

## Step 6 — `state`, redirects, and callback safety

- Send a cryptographically random, single-use `state` and verify it on the
  callback — that is the CSRF defense for the authorization code flow. Bind
  it to the browser (a short-lived cookie), don't just keep a global set.
- Store the PKCE `code_verifier` per-attempt server-side (or in a secure
  cookie) and use it exactly once.
- A post-login "return to" destination must be validated against an
  allowlist of internal paths, or you have built an open redirect that
  phishing will use. Never redirect to an absolute URL supplied in a query
  parameter.
- Handle the user-denies and error responses (`error=access_denied`)
  explicitly instead of leaving a blank screen or an unhandled exception.
- The callback route is unauthenticated and internet-reachable: rate-limit
  it and don't leak token contents in logs or error pages.

## Step 7 — Scopes, refresh tokens, and API access

- Request the minimum: `openid email profile` covers sign-in. Every extra
  scope is more consent friction, a bigger blast radius, and — for sensitive
  or restricted scopes (Gmail, Drive, Calendar contents) — a Google
  verification review with a security assessment.
- Ask for additional scopes incrementally, at the moment the feature needs
  them, not all at first login.
- A refresh token is only issued with `access_type=offline`, and typically
  only on the first consent — so `prompt=consent` is needed to get a new one
  after the first. Code that assumes a refresh token on every login silently
  breaks on the second sign-in.
- Refresh tokens are long-lived credentials: encrypt them at rest, never log
  them, and scope database access to them tightly. Handle
  `invalid_grant` (user revoked access, password change, token expiry after
  inactivity, or a project still in testing status) by clearing the stored
  token and re-prompting — not by retrying.
- Access tokens expire in about an hour; refresh on demand and cache
  server-side, don't request one per API call.
- Client secret, client ID, and redirect URIs are environment config —
  `devops-dev` wires them; never commit values.

## Step 8 — Account linking and identity edge cases

- Linking a Google login to an existing local account **by email alone is an
  account takeover path** unless `email_verified` is true and the local
  account's email is verified too. Require an explicit link step from a
  logged-in session when either side is unverified.
- One person can hold several Google accounts, and a Workspace address can
  be deleted and reissued to a different human — key on `sub`, and treat an
  email change as a profile update, not a new identity.
- Users can revoke your app's access at any time from their Google account;
  the next API call fails, and the app must degrade to a re-consent prompt.
- Deleting a user in your app should revoke the stored Google token too.

## Step 9 — Self-check before reporting done

Confirm the flow is authorization code + PKCE (or GIS ID token) with no
client secret in anything shipped to a browser or device. Confirm the ID
token is verified server-side with signature, `aud`, `iss`, `exp`, and
nonce, that `email_verified` gates any use of email, and that `sub` is the
stored key. Confirm `state` is generated, single-use, and checked; that
redirect URIs are exact per environment and any post-login destination is
allowlisted. Confirm the app mints its own `HttpOnly`/`Secure` session
cookie, regenerated at login, and that no Google token is in browser
storage. Confirm scopes are minimal and any refresh token is encrypted with
`invalid_grant` handled. Then run the real flow end to end
(`skills/dev-testing/SKILL.md`) — a fresh user, an existing user, a denied
consent, and a second login — and check the session and user row that
result, not just the redirect. Route anything touching the trust boundary
past `security-auditor`.

## What not to do

- Don't use the implicit flow, and don't put a client secret in an SPA,
  mobile app, or Electron bundle.
- Don't decode an ID token without verifying its signature, and never skip
  the `aud` check.
- Don't trust `email` before `email_verified`, and don't key users by email.
- Don't use a Google access token as your app's session or as proof of
  identity.
- Don't store Google tokens in `localStorage`, and don't log them.
- Don't skip `state`/nonce, and don't redirect after login to a URL taken
  from a query parameter.
- Don't assume a refresh token arrives on every login, and don't retry
  blindly on `invalid_grant`.
- Don't request scopes the feature doesn't use yet.
- Don't auto-link a Google identity to an existing account on an unverified
  email match.
- Don't hand-roll a second session mechanism next to the auth framework the
  project already uses.
