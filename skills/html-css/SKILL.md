---
name: html-css
description: Write framework-agnostic semantic HTML and modern CSS — correct landmark/heading structure, accessible forms and images, keyboard/focus operability, color contrast, responsive/fluid layout, and CSS conventions (custom properties, logical layout, container queries) — for static pages, email templates, landing pages, or any markup with no framework in front of it. Use when authoring or reviewing raw .html/.css files, when a page has no component framework, or when checking accessibility/semantics of existing markup. Not for framework component work (React/Vue/Svelte state, props, styling-in-JS) — that's frontend-dev, which assumes this craft is already in place and builds framework conventions on top of it.
---

# HTML & CSS

The framework-agnostic layer underneath `frontend-dev`. `frontend-dev`
implements components inside an established framework and matches its
existing conventions — it does not own raw markup/CSS craftsmanship as a
topic. This skill does: semantic structure and accessibility fundamentals
first, responsive layout and CSS conventions second. Use this skill directly
for a static page, an email template, a landing page with no framework, or
to review/fix markup quality — dispatch `frontend-dev` instead once a
framework and component conventions are the actual unit of work.

## Step 1 — Confirm the layer

If the project has a framework (React/Vue/Svelte/etc.) and this is
component work inside it, stop and hand off to `frontend-dev` — it matches
that project's existing conventions. Use this skill for raw `.html`/`.css`,
templates, or auditing markup semantics/accessibility regardless of what
sits on top of it.

## Step 2 — Semantic structure (primary)

- One `<h1>` per page; heading levels descend without skipping
  (`h2`→`h3`, never `h2`→`h4`).
- Landmarks over generic `<div>`s: `<header>`, `<nav>`, `<main>`,
  `<article>`, `<aside>`, `<footer>`. One `<main>` per page.
- Lists are `<ul>`/`<ol>`/`<dl>`, not styled `<div>` stacks. Tables are
  `<table>` with `<th scope>` for data grids, never for layout.
- Buttons vs. links: `<button>` triggers an action, `<a href>` navigates.
  Never a `<div onclick>` for either — it breaks keyboard and screen-reader
  access outright.
- Use the element that already means what you want (`<button>`, `<dialog>`,
  `<details>`/`<summary>`) before reaching for ARIA to fake it.

## Step 3 — Accessibility fundamentals (primary)

- **Forms**: every input has a `<label for>` (or wraps it) — placeholder
  text is not a label. Group related inputs in `<fieldset>`/`<legend>`.
  Associate error/help text with `aria-describedby`.
- **Images**: meaningful images get descriptive `alt`; purely decorative
  images get `alt=""` (never omit the attribute). Icon-only buttons need an
  accessible name (`aria-label` or visually-hidden text).
- **Focus visibility**: never `outline: none` without a replacement focus
  style — every interactive element needs a visible focus state.
- **Keyboard operability**: everything a mouse can do, Tab/Enter/Space can
  do too. Check tab order follows visual/reading order; don't fight it with
  arbitrary `tabindex` values (0 or -1 only).
- **Color contrast**: body text ≥ 4.5:1, large text (18px+ bold or 24px+)
  ≥ 3:1 against its background — check this, don't eyeball it.
- **ARIA is a last resort**: reach for it only when no native element
  expresses the relationship (e.g. `aria-expanded` on a custom disclosure,
  `role="alert"` for a live region) — never to relabel or override an
  element's native semantics when the right element would've done it for
  free.

## Step 4 — Responsive & fluid layout (secondary)

- Layout with Flexbox/Grid, not floats or absolute positioning for page
  structure.
- Fluid before breakpoints: `clamp()`, `min()`/`max()`, percentage/`fr`
  widths get you most of the way; add `@media`/`@container` breakpoints
  only where content actually breaks.
- Mobile-first: base styles unconstrained, `min-width` media queries add
  constraints going up — cheaper to reason about than the reverse.
- Container queries (`@container`) when a component's layout depends on
  its container's size, not the viewport.

## Step 5 — Modern CSS conventions (secondary)

- Custom properties (`--token`) for repeated values (color, spacing,
  radius) instead of hard-coded literals scattered through the file.
- Logical properties (`margin-inline`, `padding-block`) over physical
  (`margin-left`) when RTL/writing-mode could ever matter — cheap
  correctness, skip only if the project is deliberately physical-only.
- One layout method per problem: Grid for two-dimensional layout, Flexbox
  for one-dimensional — don't nest Flexboxes to fake a grid.
- Keep specificity flat: classes over IDs/deep nesting for styling hooks,
  so overrides don't turn into a specificity fight.

## Step 6 — Self-check before reporting done

Tab through the page/component with keyboard only. Confirm heading
structure with the elements alone (no CSS). Check contrast on the actual
rendered colors, not the design mockup.

## What not to do

- Don't add ARIA roles/attributes where a native semantic element already
  provides them — that's the accessibility-tree equivalent of dead code.
- Don't ship a focus-style removal without a replacement.
- Don't reach for JS to do what CSS already does (show/hide via
  `<details>`, form validation via `required`/`pattern`, native `<dialog>`
  for modals).
- Don't dispatch this skill for framework component work — that's
  `frontend-dev`; this skill is what its output should already be built on.
