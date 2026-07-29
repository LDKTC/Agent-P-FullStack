---
name: dev-testing
description: Run a full-stack feature against the real running app during development — start the service, send one request or drive one browser task, then read the HTTP response, the server log, and the database delta together as a single verdict. Use after implementing a vertical slice (endpoint, form, flow), when reproducing a reported bug, or whenever a layer is about to be declared working without the app having actually run.
---

# Development-oriented testing

Testing as a step *inside* development, not a phase after it. The failure
this exists to catch: a change that looks correct in the diff, returns
`200 OK`, renders without visible complaint — and silently persisted
nothing, or logged an unhandled rejection the browser swallowed.

A single signal is not a verdict. **Response, log, and stored state are read
together**, because the interesting full-stack bugs live precisely in the gap
between them.

Run this after each vertical slice, not once at the end. A slice tested
while it's the only thing that changed localizes its own failure; five
slices tested together do not.

## Backend probe — one request, three signals

Per endpoint touched:

1. **Clear the ports.** Kill whatever is still listening on the ports the
   service needs. A stale process from an earlier run answering the probe is
   the single most misleading result available — you test the old build and
   conclude the new one works.
2. **Snapshot the database** before the request: the row count, or the
   specific rows the endpoint should touch.
3. **Start the service** with output captured to a log file, and wait for
   readiness by polling the log for its actual ready line — not a fixed
   `sleep`, which is either flaky or slow and usually both.
4. **Send exactly one request** — method, URL, headers, body — so the log
   lines and the database delta belong unambiguously to it.
5. **Read all three signals**, then tear the service down.

Verdict rules:

- A 2xx with no corresponding row change, on an endpoint whose job is to
  write, is a **FAIL** — report it as such rather than as a pass.
- A clean response with a stack trace, unhandled rejection, or ORM warning
  in the log is a **FAIL**; the error was caught and swallowed somewhere,
  which is a bug even when this request happened to survive it.
- A 4xx/5xx is only a pass when the probe was deliberately testing the
  rejection path — say which it was.
- Cover the failure paths too: missing required field, unauthenticated,
  wrong-owner. An endpoint probed only on its happy path is half-probed.

## Frontend probe — a user task, not a URL fetch

1. Clear the ports, snapshot the database, start the dev server, wait for
   its ready line.
2. Drive a **browser-level task stated the way a user would describe it** —
   "register a new account, then sign in with it and open the dashboard" —
   navigating from the landing page rather than deep-linking straight to the
   route under test. Deep-linking skips exactly the navigation, guard, and
   auth-state wiring that most often breaks.
3. Stop on either outcome: the task completed, or an uncaught console error
   appeared.
4. Collect: what the page actually showed at the end, every console error
   and failed network request, and the database delta.

Verdict rules:

- An **uncaught console error is a FAIL** even when the page looks correct —
  the render that succeeded here is the render that breaks on the next state
  change.
- A completed UI flow with no matching database write is a FAIL — the form
  submitted into a void.
- A 4xx/5xx in the network panel is a FAIL even when the UI hides it behind
  a spinner or an empty state.
- If no browser automation is available in this environment, say so and
  probe the endpoints the flow calls instead — then report frontend coverage
  as unverified rather than implying the flow was exercised.

## Database delta

Snapshot before, compare after. What to check, in order of how often it
catches something real:

- Row **count** changed by exactly the expected amount — catches both the
  silent no-op and the accidental duplicate insert.
- The written row's **values** are what was sent — catches truncation,
  timezone shifts, and silent type coercion.
- **Related** tables: the join row, the foreign key, the audit entry.
- Nothing was written that shouldn't have been — a partial write left behind
  by a failed request that should have rolled back.

## Environment hygiene

- Always tear the service down, including after a failed probe. A probe that
  leaves a listener behind poisons the next one.
- Probe against a development or test database, never a shared or production
  one — these probes write.
- If startup itself fails, that is the finding. Report the startup log; do
  not retry with a different command until something happens to boot.

## Report format

```
PROBE: <endpoint + method | browser task, in one line>
SETUP: <start command, ports, database probed against>
REQUEST/ACTIONS: <the single request, or the user-level steps driven>
HTTP RESPONSE: <status + body, or "n/a — frontend probe">
CONSOLE/SERVER LOG: <errors and warnings, or "clean">
DATABASE DELTA: <rows added/changed/removed vs. the before-snapshot, or "none — and whether that's expected">
VERDICT: PASS | FAIL — <which signal decided it>
NOT COVERED: <paths this probe didn't exercise, stated plainly — or "none">
```

## What not to do

- Don't infer a pass from the diff. Nothing here is satisfied by reading
  code; the app runs or the probe didn't happen.
- Don't report a 2xx as a pass without having looked at the log and the
  stored state.
- Don't batch many requests into one probe — a mixed log and a mixed delta
  can't be attributed to any single call.
- Don't wait for startup with a fixed sleep when the log announces
  readiness.
- Don't leave a dev server or database container running after the probe.
- Don't quietly downgrade a browser task to an endpoint check and still call
  the flow verified — name the substitution.
