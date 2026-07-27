---
name: html-css
description: Write framework-agnostic semantic HTML and modern CSS — correct landmark/heading structure, accessible forms/images/keyboard operability, color contrast, responsive/fluid layout, and CSS conventions (custom properties, logical properties, Grid/Flexbox, container queries) — for static pages, email templates, landing pages, or any markup with no framework in front of it. Use when authoring or reviewing raw .html/.css files, when a page has no component framework yet, or when checking accessibility/semantics of existing markup, including when asked (Thai or English) to เขียนหรือตรวจ HTML/CSS ให้ถูก semantic และเข้าถึงได้ (accessible). Not for framework component work (React/Vue/Svelte state, props, styling-in-JS) — that's frontend-dev, which assumes this craft is already in place and builds framework conventions on top of it.
---

# HTML & CSS

The framework-agnostic layer underneath `frontend-dev`: that agent assumes a
framework and existing component/styling conventions are already in place
and builds within them — it does not own raw markup/CSS craftsmanship as a
topic. This skill does. Use it directly for a static page, an email
template, a landing page with no framework, or to review/fix markup
quality; hand off to `frontend-dev` once a framework and component
conventions are the actual unit of work.

## Step 1 — Confirm the layer

If the project has a framework (React/Vue/Svelte/etc.) and this is
component work inside it, stop and use `frontend-dev` instead — it matches
that project's existing conventions. Use this skill for raw `.html`/`.css`,
templates, or auditing markup semantics/accessibility, regardless of what
framework (if any) sits on top of it.

## Step 2 — Semantic structure

- One `<h1>` per page; heading levels descend without skipping
  (`h2`→`h3`, never `h2`→`h4`).
- Landmarks over generic `<div>`s: `<header>`, `<nav>`, `<main>`,
  `<article>`, `<aside>`, `<footer>`. One `<main>` per page.
- Lists are `<ul>`/`<ol>`/`<dl>`; tabular data is a real `<table>` with
  `<th scope>` (never used for layout); captioned images are
  `<figure>`/`<figcaption>`.
- Buttons vs. links: `<button>` triggers an action, `<a href>` navigates.
  Never a `<div onclick>` for either — it breaks keyboard and
  screen-reader access outright.
- Use the element that already means what you want (`<button>`, `<dialog>`,
  `<details>`/`<summary>`) before reaching for ARIA to fake it.

## Step 3 — Accessibility fundamentals

- **Forms**: every input has a `<label for>` (or wraps it) — placeholder
  text is not a label. Group related inputs in `<fieldset>`/`<legend>`.
  Associate error/help text with `aria-describedby`.
- **Images**: meaningful images get descriptive `alt`; purely decorative
  images get `alt=""` (never omit the attribute). Icon-only buttons need an
  accessible name (`aria-label` or visually-hidden text).
- **Focus visibility**: never remove `:focus`/`:focus-visible` styling
  without a replacement — every interactive element needs a visible focus
  state.
- **Keyboard operability**: everything a mouse can do, Tab/Enter/Space can
  do too. Tab order follows visual/reading order; don't fight it with
  arbitrary `tabindex` values (0 or -1 only).
- **Color contrast**: body text ≥ 4.5:1, large text (18px+ bold or 24px+
  regular) ≥ 3:1 against its background — check this on actual rendered
  colors, don't eyeball it.
- **ARIA is a last resort**: reach for it only when no native element
  expresses the relationship (`aria-expanded` on a custom disclosure,
  `role="alert"` for a live region) — never to relabel or override an
  element's native semantics when the right element would've done it for
  free.

## Step 4 — Responsive & fluid layout

- Flexbox for one-dimensional, content-driven layout (a nav bar, a button
  row); Grid for two-dimensional or layout-driven sizing (a page shell, a
  card grid). Don't nest Flexboxes to fake a grid.
- Fluid before breakpoints: `clamp()`, `minmax()`/`auto-fit` grid tracks,
  and percentage/`fr` widths cover most sizing with no media query at all;
  add `@media` breakpoints only for real structural shifts (nav collapses,
  layout reflows) — not one per component.
- Mobile-first: base styles unconstrained, `min-width` media queries add
  constraints going up — cheaper to reason about than the reverse.
- Container queries (`@container`) when a component's layout should depend
  on the space it's given, not the viewport — what lets the same component
  work correctly in a sidebar and a full-width section without a
  breakpoint per context.

## Step 5 — Modern CSS conventions

- Custom properties as design tokens (color, spacing, radius, type scale)
  instead of literals repeated through the file — what makes a theme, dark
  mode, or rebrand a handful of edits instead of a find-and-replace:

  ```css
  :root {
    --space-3: 0.75rem;
    --radius-md: 0.5rem;
    --color-accent: oklch(0.6 0.15 250);
  }
  ```

- Logical properties (`margin-inline`, `padding-block`) over physical
  (`margin-left`, `padding-top`) — costs nothing left-to-right and is what
  keeps the same CSS correct if the page ever needs `dir="rtl"`.
- Keep specificity flat: classes over IDs or deep nesting for styling
  hooks. No `!important` — if a rule isn't winning, fix the specificity or
  source order instead of overriding it with a bigger hammer.

## Step 6 — Self-check before reporting done

Tab through the page/component with keyboard only. Confirm heading
structure and reading order hold with CSS off. Check contrast on the
actual rendered colors, not the design mockup. Resize from mobile to wide
desktop and confirm nothing breaks between breakpoints.

## What not to do

- Don't add ARIA roles/attributes where a native semantic element already
  provides them.
- Don't ship a focus-style removal without a replacement.
- Don't reach for `!important` or an ID selector to win a specificity
  fight — fix the actual specificity.
- Don't use this skill for framework component work — that's
  `frontend-dev`; this skill is what its output should already be built on.
