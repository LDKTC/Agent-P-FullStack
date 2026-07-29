---
name: dry-and-cor
description: Two structural rules for full-stack code — DRY (one piece of knowledge lives in one place, and the false-DRY traps specific to a client/server split) and Chain of Responsibility (middleware, guards, interceptors, and error-handler pipelines — one decision per handler, order as a contract, explicit termination — applied both to request pipelines and to routing work across a roster of specialists). Use when writing or reviewing code that duplicates a rule across layers, when adding to or reordering a middleware/guard chain, when deciding whether to extract a shared abstraction, or when dispatching/routing a task to the persona that owns it.
---

# DRY and Chain of Responsibility

Two rules that pull in opposite directions and are worth stating together:
DRY collapses repetition, CoR deliberately spreads one request's handling
across several small pieces. Applied without judgment, the first produces
tangled abstractions that couple unrelated callers, and the second produces
control flow nobody can follow. Both sections below are mostly about where
the line sits.

## DRY — one piece of *knowledge*, one place

DRY is about knowledge, not characters. Two blocks that look identical today
but change for different reasons are not duplication, and merging them
creates a coupling that will have to be torn apart later — usually by adding
a flag parameter, which is the tell that it should have stayed two things.

The test before extracting anything:

> If this rule changes, would **every** copy have to change, always and in
> the same way?

Yes → real duplication, one source. No → coincidental similarity, leave it
alone.

### The duplication that matters in a full-stack project

- **A business rule expressed in two layers.** "An order over $10k needs
  approval" living in a frontend guard *and* a backend service *and* a DB
  constraint means three places drift apart. Pick where it's authoritative
  (the server, essentially always), and have the others derive from it or
  defer to it.
- **The API contract, typed twice.** The response shape `backend-dev` states
  and the model `frontend-dev` builds against are one piece of knowledge in
  two files. Where the stack supports it (shared types package, generated
  OpenAPI client, tRPC), generate one from the other instead of maintaining
  hand-written twins that silently diverge on the next field rename.
- **Magic values** — a status string, a role name, a retry limit — repeated
  across layers rather than defined once and imported.
- **Query logic** duplicated across handlers instead of living in the
  repository/service layer the project already has.

### The false-DRY traps

- **Client-side and server-side validation are supposed to both exist.**
  Deleting the server's copy because the form already checks it is a
  vulnerability, not a DRY win — the client is not a trust boundary. The
  correct move is to derive both from one schema where the stack allows
  (a shared Zod/JSON-Schema definition), and to accept honest duplication
  where it doesn't.
- **Tests are allowed to repeat themselves.** Explicit, slightly repetitive
  setup usually reads better than a shared fixture helper that hides what's
  actually under test. Deduplicate test *utilities*, not test *intent*.
- **Two similar-looking endpoints/components serving different domains**
  will diverge. Merging them behind a mode flag couples two roadmaps.
- **Rule of three** — the second occurrence is worth noting, the third is
  worth extracting. Abstracting at the first repetition is guessing at a
  pattern from a single data point.

### When you do extract

Extract to where it belongs in the project's existing layering (per
`fullstack-senior-dev`'s briefing), not to a `utils/` dump. A shared helper
whose callers span two lanes needs an owner — say which one in the handoff.

## CoR — chains of handlers

Chain of Responsibility is already in every web stack you'll touch, under
other names: Express/Fastify **middleware**, NestJS **guards, interceptors,
pipes, and filters**, Django's `MIDDLEWARE` list, frontend **route guards**,
error-handler chains. Each link inspects the request and either handles it
(terminating the chain) or passes it along.

**Use the framework's mechanism rather than hand-rolling one.** A bespoke
chain runner in a project that already has middleware registration is a
parallel architecture — the exact thing every persona in this roster is
written to avoid.

### The rules that keep a chain debuggable

1. **One decision per handler.** A middleware that authenticates *and* rate
   limits *and* logs can't be reordered, reused, or disabled independently.
   Split at the seam where the reasons to change differ.
2. **Order is a contract, so write it down.** Body parsing before
   validation. Authentication before authorization before business rules.
   Error handler last. The sequence carries as much meaning as the handlers
   do, and it is invisible at every individual call site.
3. **Terminate or pass on — never both.** Sending a response and then
   calling `next()` is the classic chain bug; it surfaces as
   "headers already sent" or a double write, often only under load.
4. **Make the fall-off-the-end case explicit.** A request that reaches the
   end of the chain unhandled must fail loudly (404/500), never return a
   silent success. An unhandled request that 200s is the failure mode
   `skills/dev-testing/SKILL.md` exists to catch.
5. **Log which link terminated the chain.** A chain trades local, readable
   control flow for flexibility; the cost is that "why did this request 403?"
   is no longer answerable by reading the handler. That log line is what
   makes the server-log signal in a probe actionable.
6. **Keep handlers order-independent where you can**, so the coupling is
   confined to the few places where order genuinely matters — and comment
   those.

### Ordering mistakes worth catching in review

- An auth guard registered **after** the route it's meant to protect, or
  after a handler that already returned data. This is a security finding,
  not a style note — escalate it.
- Rate limiting placed after expensive work, so the work still runs.
- An error handler registered before the routes it's supposed to catch.
- A CORS or body-parser layer added mid-chain, so earlier handlers see a
  different request shape than later ones.
- A guard that authenticates (who you are) but never authorizes (whether
  this record is yours) — two decisions, and only one of them was made.

### When *not* to reach for a chain

A fixed two-branch decision is an `if`, not a pipeline. Building a chain
abstraction for three conditionals adds indirection with no payoff — the
pattern earns its cost when handlers are genuinely independent, reorderable,
and expected to grow in number.

### The same rules apply to routing work, not just requests

A roster of specialists is a chain too, and the failure modes rhyme. When
dispatching work — whether to subagents or to personas you adopt in sequence
— the rules above translate directly:

- **Most specific handler wins.** Passing work to the first plausible owner
  rather than the narrowest one is the routing equivalent of a middleware
  that catches too much.
- **One owner per unit of work.** Work that forces its handler to decide
  which lane it belongs to was routed compound; split it first.
- **Terminate or pass on, never both.** A handler that does part of the work
  *and* delegates the rest leaves the middle unowned — the same defect as
  responding and calling the next link.
- **Nothing falls off the end.** Work no handler covers gets reported as
  unroutable, loudly. Quietly absorbing it is the silent 200 of
  orchestration.
- **Record which handler took it**, for the same reason a chain logs which
  link terminated it: the flow is no longer readable from any single place.

Escalation — a handler declining and passing along what it isn't scoped to
decide — is the chain working, not a failed route.

## Review checklist

```
DRY
  Knowledge duplicated across layers: <business rule / API type / magic value / query — file:line each, or "none">
  Authoritative source named for each: <where the rule actually lives>
  False-DRY check: <client+server validation kept intentionally, tests left explicit — or "n/a">
  Premature abstraction: <helper with a mode flag, or extraction at first repetition — or "none">

CoR
  Chain(s) touched: <middleware/guard/interceptor/error pipeline>
  Order documented: <the sequence and why it's that sequence>
  One decision per handler: <yes | which handler does more than one>
  Termination: <every path either responds or passes on, never both>
  Unhandled case: <what happens at the end of the chain>
```

## What not to do

- Don't deduplicate code that merely looks alike — check whether it changes
  for the same reason first.
- Don't remove server-side validation because the client already validates.
- Don't add a boolean/mode parameter to make one function serve two callers;
  that's the signal to keep them separate.
- Don't hand-roll a chain runner when the framework already registers
  middleware.
- Don't add a handler to a chain without stating where in the order it goes
  and why.
- Don't leave a chain that can silently fall through to success.
