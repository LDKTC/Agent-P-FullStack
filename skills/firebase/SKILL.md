---
name: firebase
description: Firebase-specific expertise — the client SDK vs Admin SDK trust boundary (the client config is public by design; the service account bypasses all rules and stays server-side), Security Rules as the real authorization layer including the rule that rules are not filters so queries must be constrained to match them, Firestore data modeling for the read shape (denormalization, subcollections vs arrays, the 1 MiB document limit, hot-document write limits, required composite indexes), the per-document-read cost model and how listeners and unbounded queries turn into a bill, Cloud Functions practices (idempotent background handlers, cold starts, region colocation, Secret Manager), server-side ID token verification with custom claims for roles, and the Emulator Suite for local development. Use when `package.json` lists `firebase` or `firebase-admin`, when `firebase.json`, `.firebaserc`, `firestore.rules`, or `firestore.indexes.json` exist, when `GoogleService-Info.plist`/`google-services.json` is present, or when asked directly (Thai or English) to ต่อ/ตั้งค่า Firebase, เขียน Firestore Security Rules, หรือทำ auth ด้วย Firebase. Not a replacement for backend-dev's business logic or api-integration-dev's client wiring — this is the Firebase-specific expertise both draw on.
---

# Firebase

A backend-as-a-service where the client talks to the database directly, so
the same boundary question as Supabase applies: **authorization lives in
Security Rules, not in application code**, because the application code is
in the user's hands. `backend-dev` owns business logic and
`api-integration-dev` owns SDK wiring; this skill is the Firebase knowledge
they apply.

## Step 1 — Confirm Firebase is actually in play

Check `package.json` for `firebase` (client) or `firebase-admin` (server);
`firebase.json`, `.firebaserc`, `firestore.rules`, `storage.rules`, or
`firestore.indexes.json`; a `functions/` directory; or
`google-services.json`/`GoogleService-Info.plist` in a mobile app. Note
which products are in use — Firestore, Realtime Database, Auth, Storage,
Functions, Hosting — since the rules language and the data model differ
between Firestore and the Realtime Database, and advice for one does not
transfer to the other.

## Step 2 — Client SDK vs Admin SDK

- The client `firebaseConfig` (apiKey, projectId, appId) is **public by
  design**. It identifies the project; it does not authorize anything.
  Finding it in a bundle is not a leak, and hiding it in an env var buys
  nothing — Security Rules are what protect the data.
- The **Admin SDK** (a service account JSON, or `applicationDefault()` on a
  Google runtime) **bypasses every Security Rule**. It belongs only in
  server code and Cloud Functions. A service account key committed to the
  repo or bundled into a client is a full project compromise — rotate it,
  don't just delete the file.
- Because the Admin SDK skips rules, server-side code must do its own
  authorization checks. "It works from the server" says nothing about
  whether a user could have done it.
- App Check adds attestation that requests come from your real app; it
  reduces abuse but is not a substitute for rules.

## Step 3 — Security Rules are the authorization layer

- Start from default deny and grant narrowly. A rule of
  `allow read, write: if true`, or one gated only on
  `request.auth != null`, means any signed-in user (including one who just
  signed up) can read every other user's data.
- **Rules are not filters.** A query is rejected outright unless the rules
  can determine that *every* document it could return is allowed — so the
  query itself must carry the constraint (`where('ownerId', '==', uid)`)
  that the rule checks. This is the single most common Firestore rules
  confusion: a rule allowing `resource.data.ownerId == request.auth.uid`
  does not silently trim a collection-wide query, it fails it.
- Validate writes in rules, not only in the client: check field types,
  required and forbidden fields, immutability of `ownerId`/`createdAt`
  (`request.resource.data.ownerId == resource.data.ownerId`), and value
  ranges. The client SDK writes arbitrary documents otherwise.
- Keep rule-driven `get()`/`exists()` lookups minimal — each is a billed
  read and counts against a per-request limit; denormalize the field the
  rule needs onto the document instead.
- Roles belong in custom claims (set server-side via the Admin SDK) or in a
  document the user can't write — never in a user-writable profile field.
- Storage has its own `storage.rules`: gate by path prefix (e.g.
  `users/{uid}/…`), and cap `request.resource.size` and `contentType` in the
  rule rather than trusting the upload.
- Test rules with the emulator's rules unit-testing library, as an
  unauthenticated user, a wrong user, and the right user. Deploy rules
  together with the code change that depends on them.

## Step 4 — Firestore data modeling

- Model for the read shape. Firestore has no joins, so the query the UI
  actually issues determines the document layout — denormalize the handful
  of fields a list view renders instead of fetching each referenced
  document.
- Denormalized copies need an update path (a Cloud Function on write, or a
  batched write) or they drift. Say which one owns the copy.
- Subcollections for unbounded child data; an array field for a small,
  bounded set. An array that grows without limit hits the 1 MiB document
  limit and makes every read of the parent expensive.
- Documents are limited to ~1 MiB, and a single document sustains roughly
  one write per second — a global counter, a "likes" total on a popular
  post, or an append-only log in one document is a hotspot. Use distributed
  counters or a subcollection of events aggregated asynchronously.
- Sequential document IDs (timestamps, incrementing numbers) create hotspots
  in the index; prefer auto-generated IDs unless a deterministic ID is
  needed for idempotency.
- Compound queries need composite indexes. Add them to
  `firestore.indexes.json` and deploy them — don't rely on clicking the link
  in a runtime error message, which leaves the repo and production out of
  sync.
- Transactions are optimistic and retry, so a transaction body must be a
  pure function of its reads with no side effects; batched writes (up to
  500 operations) are atomic but do no reads.
- `array-contains`, `in`, and `!=` have query limits and can't be combined
  freely; there is no native full-text search — that needs a search service.

## Step 5 — The cost model is the performance model

- Billing is per document read, written, and deleted. A screen that reads
  200 documents to display 10 costs 20× more than it looks in the code.
- Never read a collection to count it — use the aggregation `count()` query,
  or maintain a counter.
- Paginate with cursors (`startAfter` on the last document), not by fetching
  everything and slicing. There is no `OFFSET` that skips reads — offset
  still reads and discards.
- `onSnapshot` listeners bill for the initial result set plus each changed
  document; a listener attached in a component that mounts repeatedly, or
  never unsubscribed, is both a leak and a bill. Return the unsubscribe from
  every effect.
- Set a `limit()` on every list query. An unbounded query on a collection
  that grows is a latent outage.
- Enable offline persistence deliberately — it changes read semantics
  (cached results, pending writes) and can mask a connectivity failure.

## Step 6 — Cloud Functions

- Background (trigger) functions are **at-least-once**: they can run twice
  for one event. Make handlers idempotent — key on the event ID or a
  document flag — or duplicate emails and double charges will appear
  eventually.
- A write from a function that triggers the same function is an infinite
  loop with a real bill; guard with a field check or scope the trigger path.
- Colocate the function region with the Firestore location; a cross-region
  trigger adds latency to every invocation.
- Cold starts dominate for infrequent functions: keep dependencies small,
  initialize the Admin SDK and other clients at module scope (reused across
  invocations), and use `minInstances` only where the latency is worth the
  cost.
- Secrets go in Secret Manager (or the runtime's secret bindings), never
  hardcoded and not in the deprecated `functions.config()`.
- HTTPS callable functions receive a verified `auth` context; a plain HTTP
  function does not — verify the ID token yourself there, and verify the
  provider's signature for inbound webhooks.
- Always return the promise from an async trigger; a function that resolves
  before its writes finish has them killed with the instance.

## Step 7 — Auth and local development

- On the server, verify the ID token with `admin.auth().verifyIdToken()` —
  never trust a decoded JWT, a `uid` sent in a request body, or a client's
  claim about who it is.
- Custom claims are the role mechanism; they're set server-side and only
  appear in the client's token after a refresh, so a role change needs a
  forced token refresh to take effect.
- Firebase ID tokens expire in an hour and the SDK refreshes them; don't
  cache a token or use it as a long-lived session credential.
- Google sign-in via Firebase Auth still follows
  `skills/google-auth/SKILL.md` for flow choice, redirect handling, and
  scopes.
- Use the Emulator Suite (Firestore, Auth, Functions, Storage) for local
  development and tests. Pointing a test suite at a real project is slow,
  costs money, and eventually deletes production data.
- Keep `firestore.rules`, `storage.rules`, and `firestore.indexes.json` in
  the repo and deploy them from CI with the code — that's `devops-dev`'s
  lane, but the files are part of this change, not a dashboard setting.

## Step 8 — Self-check before reporting done

Confirm no service account key is in the repo or reachable from a client
build. Confirm every collection touched has rules that deny by default,
constrain by owner or role, validate written fields, and that each new query
carries the constraint its rule requires. Confirm rules were exercised in
the emulator as an unauthenticated user, a different user, and the intended
user. Confirm composite indexes for new compound queries are committed,
every list query has a `limit()`, every listener has an unsubscribe, and no
new write concentrates on a single hot document. Confirm background
functions are idempotent and can't retrigger themselves. Then verify the
feature against the emulator with `skills/dev-testing/SKILL.md` — response,
log, and document delta together, as a non-privileged user.

## What not to do

- Don't commit or bundle a service account key, and don't treat the public
  client config as a secret.
- Don't ship `allow read, write: if request.auth != null` as a real rule.
- Don't expect rules to filter a query — the query must match the rule.
- Don't rely on client-side validation for document shape; validate in
  rules.
- Don't store roles in a user-writable document instead of custom claims.
- Don't read a collection to count it, page with offsets, or leave a list
  query unbounded.
- Don't leave an `onSnapshot` listener unsubscribed.
- Don't write a non-idempotent background function, or one whose write
  retriggers itself.
- Don't trust a `uid` from a request body — verify the ID token.
- Don't create indexes by clicking a console error link and leaving
  `firestore.indexes.json` behind, and don't run tests against a real
  project when the emulator exists.
