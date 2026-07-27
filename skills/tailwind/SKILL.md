---
name: tailwind
description: Write and review utility-first Tailwind CSS — composing utilities directly in markup instead of custom CSS, using the project's design-token scale (spacing/color/font-size/radius from tailwind.config) instead of arbitrary values, mobile-first responsive prefixes (sm:/md:/lg:), state variants (hover:/focus:/dark:/aria-*) instead of hand-written pseudo-class CSS, and keeping class strings maintainable as a project grows (@apply vs. component extraction, config-driven tokens and dark mode). Use when a project's package.json lists tailwindcss as a dependency (or a tailwind.config/@theme block exists), or when asked directly to write, style, or review something with Tailwind — any framework, or plain HTML, not just React — including when asked (Thai or English) to เขียนหรือจัดสไตล์ด้วย Tailwind CSS ให้ถูกแบบ utility-first. Not for the underlying markup semantics/accessibility itself — that's html-css, which this skill styles on top of.
---

# Tailwind CSS

The utility-first styling layer on top of `html-css`: that skill owns
semantic/accessible markup, this skill owns how it gets styled once Tailwind
is the tool in use. Semantics and accessibility don't stop mattering because
classes replaced a stylesheet — apply `html-css` underneath every component
this skill touches. Framework-agnostic: Tailwind classes work the same in
React JSX, Vue templates, Svelte, or plain HTML, so nothing here assumes
React. When the framework does happen to be React, `frontend-dev` (and
`skills/react/SKILL.md` for React specifically) is what matches component
conventions — this skill is what governs the class strings inside whatever
gets built.

## Step 1 — Confirm Tailwind is actually in play

Check `package.json` for a `tailwindcss` dependency and find the config:
`tailwind.config.{js,ts,mjs}` (v3-style) or an `@theme` block in the main CSS
file (v4-style config-in-CSS). If neither exists and nothing asked directly
for Tailwind, stop — this isn't the right skill; use `html-css` for plain
CSS instead.

## Step 2 — Read the token scale before writing a single class

Open the config and note any extended/overridden `spacing`, `colors`,
`fontSize`, `borderRadius`, `screens`. These are the project's real design
tokens — prefer them over Tailwind's defaults whenever the project has
customized them (`bg-brand-500` over a default palette color that happens to
look close):

```js
// tailwind.config.js
theme: {
  extend: {
    colors: { brand: { DEFAULT: '#3b82f6', dark: '#1d4ed8' } },
    spacing: { 18: '4.5rem' },
  },
},
```

## Step 3 — Compose utilities in markup; extract a component before reaching for @apply

Reach for existing utilities before writing custom CSS — `flex items-center
gap-4 rounded-lg border p-4` beats a hand-rolled `.card` rule doing the same
thing. When a combination repeats, extract a component (React/Vue/etc.), not
an `@apply` class — the component *is* the reuse mechanism. Reserve `@apply`
for the narrow case with no component boundary: styling markup you don't
control (CMS/markdown output, third-party widgets) or a global reset layered
over Tailwind's preflight. A growing `@apply` block is hand-written CSS
again with extra indirection — a signal to extract a component, not to keep
the block.

## Step 4 — Scale classes over arbitrary values

`p-4`, `text-lg`, `rounded-md`, `bg-brand-500`, `gap-6` before `p-[17px]`,
`text-[15px]`, `rounded-[7px]`, `bg-[#3b82f6]`. Arbitrary `[...]` values are
an escape hatch for a genuine one-off (a pixel-perfect overlay position, a
third-party widget's fixed dimension) — not the default when a scale step is
one step off from the mockup. The same arbitrary value showing up three-plus
times isn't a one-off anymore — move it into the config (Step 2) so a design
change is one edit, not a repo-wide find-and-replace.

## Step 5 — Responsive prefixes, mobile-first

Unprefixed utilities are the mobile/base style; `sm:`/`md:`/`lg:`/`xl:`/
`2xl:` add overrides going up (`min-width`), matching `html-css`'s
mobile-first convention:

```html
<div class="flex flex-col gap-4 md:flex-row md:gap-8">
```

Never simulate desktop-first by overriding down with `max-*` variants as the
default pattern — that inverts the cascade and gets harder to reason about
with every added breakpoint.

## Step 6 — State variants and dark mode, driven by config

`hover:`, `focus:`, `focus-visible:`, `active:`, `disabled:` for interaction
states; `aria-*`/`data-*` variants (`aria-expanded:`, `data-[state=open]:`)
for component state instead of custom class toggling in JS; `group-hover:`/
`peer-*` for parent/sibling-driven state. Stack with responsive prefixes in
order (`md:hover:bg-brand-600`) rather than writing a separate `:hover` rule
for something Tailwind already expresses inline.

Set `darkMode` once in config (`class` for user-toggleable, `media` for
OS-only), then use `dark:` consistently. Back theme colors with CSS
variables wired into `theme.extend` so a rebrand or new theme is edits in
one place, not scattered `dark:bg-gray-800 dark:text-white`-style pairs on
every element:

```js
colors: { surface: 'rgb(var(--color-surface) / <alpha-value>)' }
```

## Step 7 — Keep class strings readable

Use a consistent ordering (layout → box model → typography → visual →
state/responsive) so diffs are readable — or defer to
`prettier-plugin-tailwindcss` if it's already a devDependency instead of
hand-ordering. For conditional classes, use the project's existing merge
helper (`clsx`, or `cn` built on `tailwind-merge`) instead of manual string
concatenation or nested ternaries; check for one before adding a new
dependency. A class string too long to read at a glance is a signal to
extract a component (Step 3), not to keep golfing the string shorter.

## Step 8 — Self-check before reporting done

Confirm focus states are still visible (`focus-visible:` present on every
interactive element — `html-css` Step 3 still applies). Confirm the base
(no-prefix) view renders correctly as the true mobile view, then check each
breakpoint up. Grep the diff for `[` arbitrary-value brackets and justify
each against Step 4, and for `@apply` blocks against Step 3. Confirm `dark:`
variants exist everywhere a light-mode color utility was added, if the
project supports dark mode.

## What not to do

- Don't write a custom CSS class or `@apply` block for something a utility
  combination already expresses, or as a substitute for extracting a
  component.
- Don't reach for arbitrary `[...]` values as the default instead of the
  token scale — they're for genuine one-offs; three-plus repeats belong in
  config.
- Don't hand-write `:hover`/`:focus`/`:disabled` CSS rules when the matching
  variant does the same job inline.
- Don't write desktop-first with `max-*` overrides as the default responsive
  pattern.
- Don't scatter ad hoc `dark:` color pairs across every element when the
  need is one semantic token defined once in config.
- Don't skip `html-css`'s semantic/accessibility fundamentals because the
  styling is utility classes — they're layered on top, not a replacement.
- Don't assume React — this skill applies identically in any framework or
  plain HTML using Tailwind.
