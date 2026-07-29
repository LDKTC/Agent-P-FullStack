---
name: typescript
description: TypeScript-specific expertise — strict-mode compiler settings and why each one catches real bugs, the boundary rule that types are erased at runtime (so parsed JSON, env vars, and request bodies must be validated with a schema and the type derived from it), disciplined use of the three escape hatches (`any`, `as`, `!`) with `unknown` + narrowing as the default, modeling with discriminated unions and literal types instead of boolean/optional soup, generics only where a real input↔output relationship exists, and `catch (e: unknown)` error handling. Use when a `tsconfig.json` exists, when authoring or reviewing `.ts`/`.tsx` files, when a type error needs diagnosing, or when asked directly (Thai or English) to เขียนหรือตรวจ TypeScript ให้ type ปลอดภัยและไม่ใช้ any. Not a replacement for react's component rules or nodejs's runtime rules — this is the language/typing layer that applies across every layer of the stack.
---

# TypeScript

The language layer that cuts across the whole roster: `frontend-dev`,
`backend-dev`, and `api-integration-dev` all write TypeScript in most
modern projects, and the rules here apply the same in a React component, an
Express handler, and a migration script. Framework-specific rules stay in
`skills/react/SKILL.md`; runtime specifics stay in
`skills/nodejs/SKILL.md`.

## Step 1 — Confirm TypeScript is actually in play

Check for `tsconfig.json`, `.ts`/`.tsx` sources, or `typescript` in
`devDependencies`. Read the existing `tsconfig.json` before writing code —
`strict`, `moduleResolution`, `paths`, and `verbatimModuleSyntax` all change
what valid code looks like. Match the project's settings rather than
assuming defaults, and never loosen a compiler flag to make an error go
away.

## Step 2 — Strict mode is the baseline

- `strict: true` is the starting point, not an aspiration. If the project
  has it off, don't silently write code that only compiles because it's off
  — say so and keep the new code strict-clean.
- `noUncheckedIndexedAccess` makes `arr[i]` and `record[key]` yield
  `T | undefined`, which is the truth — turning it on surfaces a real class
  of runtime crashes. Recommend it where the project can absorb the churn.
- `tsc --noEmit` (or the project's typecheck script) is the actual gate. A
  bundler/transpiler such as esbuild, SWC, or Babel strips types without
  checking them, so "it builds" says nothing about type correctness.
- `@ts-ignore` disables checking for the whole next line, silently, forever.
  Prefer `@ts-expect-error` with a comment: it errors when the underlying
  problem is fixed, so it can't rot.

## Step 3 — Types are erased; boundaries need runtime validation

This is the single most common source of "TypeScript lied to me": an
annotation on data that came from outside the program is an assertion, not a
check.

```ts
// Wrong: a claim, verified by nothing
const body = await req.json() as CreateUserInput;

// Right: validated at the boundary, type derived from the schema
const body = CreateUserSchema.parse(await req.json());
type CreateUserInput = z.infer<typeof CreateUserSchema>;
```

- Validate at every trust boundary: HTTP request bodies and query params,
  `process.env`, `JSON.parse` results, `localStorage`, third-party API
  responses, and untyped database driver rows.
- Derive the TypeScript type *from* the schema (`z.infer` and equivalents)
  so the two can't drift; don't hand-maintain a parallel `interface`.
- Server-side validation stays required regardless of what the client
  validates — see `skills/dry-and-cor/SKILL.md`; the client is not a trust
  boundary and a shared type is not a check.

## Step 4 — The three escape hatches

- **`any`** turns off checking for everything that value touches, and it
  spreads through assignments. Use `unknown` instead and narrow with a
  `typeof`/`in`/`instanceof` check or a type predicate — `unknown` forces
  the narrowing that `any` skips.
- **`as`** should be reachable only when you genuinely know more than the
  compiler (a `readonly` widening, a well-understood DOM cast) and carries a
  comment saying why it's sound. `as any as T` and `as unknown as T` are the
  same thing with more steps — treat both as a defect being introduced, not
  a fix.
- **`!`** (non-null assertion) claims a value can't be null. Prefer an early
  `if (!x) throw` — same brevity, real runtime guarantee, better error.
- A type predicate (`function isUser(v: unknown): v is User`) is only as
  correct as its body; the compiler does not verify the claim, so keep the
  body an exhaustive check, not a `typeof v === 'object'` shrug.

## Step 5 — Model states, don't accumulate flags

- Discriminated unions beat optional-property soup. `{ status: 'loading' } |
  { status: 'ok'; data: T } | { status: 'error'; error: E }` makes the
  impossible states (`data` *and* `error` both set) unrepresentable, and
  `switch` narrowing gives exhaustiveness for free.
- Get exhaustiveness enforced: a `default` branch assigning to
  `const _never: never = value` turns a newly added variant into a compile
  error instead of a silent fall-through.
- Prefer literal unions (`type Role = 'admin' | 'user'`) to `enum` — unions
  are erased and structural, while `enum` emits runtime code and `const
  enum` behaves differently across build setups.
- Use `satisfies` to check a config object against a type while keeping its
  narrow literal inference; a plain `: Type` annotation widens it and loses
  the key/value literals.
- `readonly` and `as const` on data that shouldn't be mutated communicate
  more than a comment does.
- Type-only imports/exports (`import type { … }`) matter under
  `verbatimModuleSyntax` and `isolatedModules` — a value import of a type
  can break the emitted module graph.

## Step 6 — Generics and inference judgment

- Add a type parameter only to express a relationship the caller cares
  about — input type determines output type, or two arguments must match.
  A generic that appears in exactly one position is usually a disguised
  `any` and should be a concrete type or `unknown`.
- Constrain type parameters (`<T extends { id: string }>`) instead of
  asserting inside the function body.
- Let inference do the work on locals and return values; annotate exported
  function signatures and module boundaries, where an accidental inferred
  change is a breaking change.
- Deeply conditional/mapped type gymnastics that nobody on the team can read
  is a cost, not a win — a plain overload or an explicit union is often the
  better engineering call.

## Step 7 — Errors, async, and null handling

- `catch (e)` gives `unknown` under strict settings, and that's correct —
  anything can be thrown. Narrow with `instanceof Error` before touching
  `.message`, and keep a fallback branch for non-Error throws.
- Don't type a function `Promise<T>` and then return a rejected path that
  callers can't see — model expected failures in the return type
  (a result union) or throw a typed error class consistently, but pick one
  per codebase.
- `??` and `?.` respect `0`, `''`, and `false`; `||` and truthiness checks
  don't. Using `||` for a default on a numeric or boolean field is a real
  bug, not a style preference.
- Watch `void` vs `undefined` in callbacks: a `() => void` signature accepts
  a function returning anything, which is how a floating promise slips into
  an event handler (see `skills/nodejs/SKILL.md`).

## Step 8 — Self-check before reporting done

Confirm `tsc --noEmit` passes — not just that the bundler produced output.
Confirm no `any`, no unexplained `as`, and no `!` was added; every external
input is parsed by a schema with its type derived from that schema. Confirm
new state shapes use a discriminated union with exhaustiveness enforced
rather than a set of optional flags, and that any new generic parameter
appears in more than one position. Confirm `catch` blocks narrow `unknown`
before using the error.

## What not to do

- Don't use `any`, and don't launder it through `as unknown as T`.
- Don't `as`-cast parsed JSON, `req.body`, or `process.env` into a type —
  validate, then derive the type from the validator.
- Don't add `@ts-ignore`; use `@ts-expect-error` with a reason, or fix it.
- Don't loosen `tsconfig.json` strictness to make an error disappear.
- Don't use `!` where an early `if (!x) throw` gives a real guarantee.
- Don't model state as several optional fields when a discriminated union
  makes the invalid combinations unrepresentable.
- Don't introduce a generic that's used once, and don't write a conditional
  type where an overload would be readable.
- Don't use `||` for defaults on numbers or booleans — `??` is what you mean.
- Don't treat a successful bundle as a passing typecheck.
