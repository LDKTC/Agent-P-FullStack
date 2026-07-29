---
name: nodejs
description: Node.js runtime-specific expertise — what blocks the single-threaded event loop and what to do instead, async correctness (sequential `await` in a loop vs `Promise.all`, unhandled rejections, errors that never propagate out of a callback), streams and backpressure for large payloads, ESM vs CommonJS interop, configuration and secrets through the environment, graceful shutdown on SIGTERM, and Node-specific security pitfalls (path traversal, `child_process`, unvalidated dynamic `require`). Use when `package.json` declares an `engines.node` or Node scripts, when code imports `node:` builtins (`fs`, `http`, `path`, `crypto`, `child_process`, `worker_threads`), when a Dockerfile runs a `node` image, or when the work targets a Node server runtime (Express/Fastify/Nest/Next route handlers), including when asked directly (Thai or English) to เขียนหรือตรวจโค้ด Node.js ให้ถูกต้องและไม่บล็อก event loop. Not a replacement for backend-dev's framework-agnostic route/validation/business-logic work — this is the runtime layer underneath it.
---

# Node.js

`backend-dev`'s runtime-specific expertise layer: that agent is
framework-agnostic and matches whatever HTTP framework the project already
uses — this skill is the Node knowledge underneath it, the part that stays
true whether the framework is Express, Fastify, Nest, Hono, or a Next.js
route handler. Language-level typing rules are `skills/typescript/SKILL.md`;
database specifics belong to the matching engine skill.

## Step 1 — Confirm the runtime is actually Node

Check `package.json` for `engines.node`, `type`, or scripts running `node`;
imports of `node:` builtins; a `node` base image in a Dockerfile; or a
server file under a Node framework. If the runtime is Deno, Bun, or an edge
runtime (Cloudflare Workers, Vercel Edge, Supabase Edge Functions), most of
the event-loop and async rules still hold but the builtin/API surface does
not — check that runtime's supported APIs before using `fs`,
`child_process`, or native modules at all.

## Step 2 — The event loop is single-threaded

One thread runs all JavaScript, so any synchronous work delays *every*
pending request, not just the one being served.

- No `fs.readFileSync`, `crypto.pbkdf2Sync`, `execSync`, or a
  `JSON.parse` of a multi-megabyte body inside a request path — the async
  form exists for exactly this reason. Sync calls are fine at startup,
  before the server accepts traffic.
- CPU-bound work (image processing, hashing a password, parsing a huge CSV,
  a tight loop over 10⁶ rows) doesn't become non-blocking by being wrapped
  in a promise — a promise around synchronous code still blocks. Move it to
  `worker_threads`, a child process, or a job queue.
- `await` inside a `for` loop over N items is N sequential round trips.
  Use `Promise.all` when the calls are independent, and a bounded
  concurrency helper (`p-limit`, or chunked `Promise.all`) when N is large
  enough to flood the database pool or the upstream API.
- Long synchronous loops also starve timers and health checks — the symptom
  is a service that "randomly" fails liveness probes under load.

## Step 3 — Async correctness

- Every promise is awaited, returned, or explicitly `.catch()`ed. A floating
  promise loses its error and finishes after the response is already sent.
- `async` callbacks passed to APIs that ignore return values —
  `array.forEach(async …)`, an EventEmitter listener, most middleware
  signatures that don't await `next` — swallow both the result and the
  rejection. Use `for … of` with `await`, or `Promise.all(map(…))`.
- Errors thrown inside a `setTimeout`/`setInterval`/`emitter.on` callback
  don't reach the surrounding `try`/`catch` — that stack frame is gone.
  Handle inside the callback.
- `process.on('unhandledRejection')` and `'uncaughtException'` are
  last-resort logging plus a controlled exit, not an error-handling
  strategy — a process that continues after an uncaught exception is in an
  undefined state.
- `Promise.all` rejects on the first failure and abandons the rest; use
  `Promise.allSettled` when every result matters, and remember neither one
  cancels the in-flight work.

## Step 4 — Streams and backpressure

- Read/write large files and proxied HTTP bodies as streams — buffering a
  file into a string to send it holds the whole thing in memory per
  concurrent request, which is how a service OOMs under mild load.
- Use `pipeline` (from `node:stream/promises`) rather than `.pipe()` chains:
  `.pipe()` doesn't forward errors or destroy the upstream on failure, which
  leaks file descriptors and sockets.
- Respect backpressure — a producer that ignores a `false` from `write()`
  grows the internal buffer without bound. `pipeline` handles this for you;
  hand-rolled loops usually don't.
- Cap request body size at the framework/proxy layer; an unbounded JSON body
  parser is a denial-of-service surface, not just a memory concern.

## Step 5 — Modules: ESM vs CommonJS

- `"type": "module"` makes `.js` files ESM: no `require`, no `__dirname`, no
  `module.exports`. Use `import`, `import.meta.url` +
  `fileURLToPath`/`import.meta.dirname`, and named exports. `.cjs`/`.mjs`
  override the package-level setting per file.
- ESM import specifiers need the file extension (`./util.js`, not `./util`)
  — a missing extension is the most common "works in ts-node, breaks after
  build" failure.
- A CJS file can't `require()` an ESM-only package; use dynamic `import()`
  (which is async) or stay on a version of the dependency that still ships
  CJS. Don't convert the whole project's module system to fix one import
  without saying so.
- Match whatever the repo already uses (`stack-briefing`) — mixing systems
  in one package is how dual-package hazards and duplicated singletons
  appear.

## Step 6 — Config, secrets, and process lifecycle

- Config comes from the environment, validated once at startup into a typed
  config object — not `process.env.FOO` read ad hoc deep inside a handler
  where a typo silently becomes `undefined`. Fail fast at boot when a
  required variable is missing.
- Never commit `.env`; ship `.env.example` with keys and empty values. Env
  wiring, secret storage, and deploy config are `devops-dev`'s lane.
- Handle `SIGTERM`/`SIGINT`: stop accepting new connections, finish in-flight
  requests with a timeout, close the database pool, then exit. A container
  killed mid-transaction leaves half-written state and connection leaks.
- Don't `process.exit()` from inside request handling — it drops in-flight
  responses and pending writes.
- Keep the process stateless where a scheduler may run several replicas: an
  in-memory cache, session store, or `setInterval` cron in the app process
  behaves differently at one instance versus three.

## Step 7 — Node-specific security

- Never interpolate user input into `exec`/`execSync` — use
  `execFile`/`spawn` with an argument array, and prefer a library over
  shelling out at all.
- Any path built from user input must be resolved and checked to still sit
  inside the intended root (`path.resolve` then a prefix check) — `../` in a
  filename is the whole path-traversal class.
- Never `require`/`import()` a module path derived from user input, and
  don't `eval`/`new Function` on request data.
- Use `crypto.randomUUID()`/`crypto.randomBytes` for tokens — `Math.random`
  is not a CSPRNG. Compare secrets with `crypto.timingSafeEqual`.
- Run the app as a non-root user in the container, pin dependency versions
  via the committed lockfile, and treat a `postinstall` script in a new
  dependency as something to look at before installing.

## Step 8 — Self-check before reporting done

Confirm nothing synchronous or CPU-heavy runs inside a request path, and
that every `await`-in-a-loop is either intentional ordering or converted to
bounded parallelism. Confirm every promise is awaited or explicitly handled,
and no `async` function was passed where its rejection would be dropped.
Confirm large payloads stream through `pipeline` rather than buffering, that
required env vars are validated at startup, and that SIGTERM closes the
server and the pool. Then run the change through
`skills/dev-testing/SKILL.md` — response, log, and database delta together.

## What not to do

- Don't call a `…Sync` API, hash a password, or parse a huge payload inside
  a request handler — that stalls every other connection.
- Don't leave a floating promise, and don't pass an `async` callback to
  something that ignores the returned promise.
- Don't rely on `uncaughtException`/`unhandledRejection` handlers to keep a
  broken process alive.
- Don't buffer a whole file or proxied response into memory, and don't chain
  `.pipe()` where `pipeline` would propagate the error.
- Don't mix ESM and CJS ad hoc, or omit file extensions in ESM imports.
- Don't read `process.env` deep in application code, and don't commit `.env`.
- Don't build a shell command or a filesystem path out of raw user input.
- Don't write the framework's routes, validation, or business logic here —
  that's `backend-dev`; this skill is the runtime knowledge it applies.
