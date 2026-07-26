# Part: styles

The "Paper & Ink" design system — warm paper, soft near-black ink, one muted
clay accent, with light and dark modes. Fonts are self-hosted via `next/font`
rather than loaded from a CDN `<link>`. `src/styles/` is a self-contained,
portable unit — copy it and its four touch-points and the whole visual
identity moves.

## The four CSS files

```
src/styles/
  index.css       entry — imports the three partials, nothing else
  tokens.css      the design contract: colors, fonts, scales (light + dark)
  base.css        L1 — the reset + bare-tag baseline
  typography.css  house-* roles + the .prose reading layer + .compact variant
```

`tokens.css` is the single source of truth. Everything else reads from it via
`var(--token-name)`. **To reskin the whole app, edit `tokens.css` only.**

### `tokens.css`

The color, spacing, radius, type-scale, motion, and border-width contract.

**Every color is `hsl(H S L)` / `hsl(H S L / A)`** — one notation, opaque or
translucent, per `project-standard`. This file is the reskin-by-eye surface, and
the edits that actually come up map to channels: drop the saturation, reduce the
contrast (lightness), take the opacity down 10%. Hex is fine to paste, not to
nudge — and it can't express the alpha variants at all, which is how a palette
ends up in two notations.

One relationship the conversion made visible: `--accent-soft` is literally
`--accent` at 12% (`hsl(17 42% 39%)` both), which the old
`#8f523a` / `rgba(143, 82, 58, 0.12)` pair hid. Retune the accent and its soft
variant now has to move with it — in hex they silently drift apart.
The four `--font-*` stacks reference the CSS variables `next/font/google`
generates in `layout.tsx` (`--font-ibm-plex-sans`, `--font-martel`,
`--font-space-mono`) rather than raw Google Fonts family-name strings — the
fonts are self-hosted, not loaded from a CDN, so there's no `'IBM Plex Sans'`
string to fall back on if the variable isn't set yet.

**One correction from a live run:** the
spacing/radius/type-scale/motion/border-width values below were
previously only described in a comment (`/* spacing: --space-1 (4px)
through --space-9 (80px) */`), never actually declared — `typography.css`
and `base.css` reference `var(--space-4)`, `var(--dur-base)`, etc., but those
custom properties didn't exist, so they silently resolved to nothing (a
missing CSS custom property is not an error — it's just absent, so `.prose`
spacing, the `body` fade-in, and the `.compact` heading sizes would have
quietly been wrong or inert). The declarations below are freshly authored to
match what the comment described, not recovered from a prior working
version — they haven't been visually checked against a designed intent
beyond "matches the numbers the comment already promised," so treat them as
a first pass, not a verified port, and look closely if anything in `.prose`
or the fade-in animation looks off:

```css
:root {
  --bg: hsl(42 19% 89%);
  --surface: hsl(47 24% 93%);
  --surface-raised: hsl(45 33% 95%);
  --surface-sunken: hsl(36 24% 8% / 4.5%);
  --text: hsl(60 4% 10%);
  --text-body: hsl(43 10% 14%);
  --text-muted: hsl(40 8% 39%);
  --text-faint: hsl(40 8% 54%);
  --border: hsl(41 15% 74%);
  --border-soft: hsl(36 24% 8% / 13%);
  --border-faint: hsl(36 24% 8% / 7%);
  --accent: hsl(17 42% 39%);
  --accent-contrast: hsl(45 33% 95%);
  --accent-soft: hsl(17 42% 39% / 12%);

  /* Overlay scrim. A framework supplies this implicitly (`bg-black/60`); with
     none, it has to exist as a real token. */
  --scrim: hsl(0 0% 0% / 60%);

  --font-display: var(--font-ibm-plex-sans), system-ui, sans-serif;
  --font-serif-stack: var(--font-martel), Georgia, serif;
  --font-body: var(--font-ibm-plex-sans), system-ui, sans-serif;
  --font-mono: var(--font-space-mono), 'SF Mono', Menlo, monospace;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 24px;
  --space-6: 32px;
  --space-7: 48px;
  --space-8: 64px;
  --space-9: 80px;

  --radius-sm: 2px;
  --radius-md: 6px;
  --radius-lg: 12px;
  --radius-pill: 999px;

  --text-xs: 11px;
  --text-sm: 13px;
  --text-lg: 22px;
  --text-xl: 28px;
  --text-subtitle: clamp(20px, 3vw, 26px);
  --text-title: clamp(36px, 6vw, 56px);

  --ease: cubic-bezier(0.4, 0, 0.2, 1);
  --dur-fast: 120ms;
  --dur-base: 200ms;
  --dur-slow: 320ms;

  --border-width: 1px;
  --border-width-strong: 2px;
}

[data-theme='dark'] {
  --bg: hsl(48 14% 7%);
  --surface: hsl(43 13% 11%);
  --surface-raised: hsl(30 9% 14%);
  --surface-sunken: hsl(0 0% 0% / 30%);
  --text: hsl(40 28% 90%);
  --text-body: hsl(40 18% 81%);
  --text-muted: hsl(37 8% 58%);
  --text-faint: hsl(37 8% 40%);
  --border: hsl(38 11% 20%);
  --border-soft: hsl(0 0% 100% / 12%);
  --border-faint: hsl(0 0% 100% / 7%);
  --accent: hsl(18 48% 57%);
  --accent-contrast: hsl(48 14% 7%);
  --accent-soft: hsl(18 48% 57% / 16%);
}
```

### `base.css`

The min-height selector covers `html, body` only — App Router has no `#root`
element to include:

```css
/* ════════════════════════════════════════════════════════════════════════════
   base.css — L1: the reset + a bare-tag baseline.

   L1 OWNS THE RESET. With no framework there is no Preflight, so if this block
   is removed the app inherits every UA default (body margin, list bullets,
   inline-image baseline gap, unstyled form controls). That is the single
   load-bearing consequence of dropping the framework — it fails as "everything
   is slightly wrong," not as an error.
   ════════════════════════════════════════════════════════════════════════════ */

*,
*::before,
*::after {
  box-sizing: border-box;
}

* {
  margin: 0;
}

img,
picture,
video,
canvas,
svg {
  display: block;
  max-width: 100%;
}

input,
button,
textarea,
select {
  font: inherit;
  color: inherit;
  background: none;
  border: none;
}

button {
  cursor: pointer;
}

ul[role='list'],
ol[role='list'] {
  list-style: none;
  padding: 0;
}

a {
  color: inherit;
  text-decoration: none;
}

html {
  background: var(--bg);
}

html,
body {
  min-height: 100%;
}

body {
  background: var(--bg);
  color: var(--text);
  font-family: var(--font-body);
  line-height: 1.6;
  animation: quickFadeIn var(--dur-base) var(--ease);
}

@keyframes quickFadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@media (prefers-reduced-motion: reduce) {
  body {
    animation: none;
  }
}
```

### `typography.css`

Copy verbatim, no adaptation needed. Two things:

1. **House typographic roles** — `.house-title`, `.house-dek`, `.house-section`,
   `.house-eyebrow`. Apply by class name anywhere.
2. **`.prose`** / **`.prose.compact`** — the long-form reading layer used by
   the `/docs` viewer (see `docs.md`).

The full block is inlined below so this skill is self-contained:

```css
/* ════════════════════════════════════════════════════════════════════════════
   typography.css — the type layer.

   Two parts:
     1. house-* roles  — page-chrome type (title, dek, section head, eyebrow).
                         Plain global classes any surface opts into by name.
     2. .prose          — the long-form reading style (headings, lists, quotes,
                         code, tables), with a `.compact` variant for dense
                         reference docs (the /docs viewer).

   All sizing/weight/family comes from the tokens (display = IBM Plex Sans,
   body = Martel serif, mono = Space Mono). Plain CSS and the single source of
   truth for reading type, so it ports or restyles with no plugin involved.
   ════════════════════════════════════════════════════════════════════════════ */

/* ── House typographic roles (page-chrome) ───────────────────────────────── */

.house-title {
  font-family: var(--font-display);
  font-size: var(--text-title);
  font-weight: 700;
  line-height: 1.08;
  letter-spacing: -0.02em;
  color: var(--text-body);
}

.house-dek {
  font-family: var(--font-serif-stack);
  font-size: var(--text-subtitle);
  font-style: italic;
  line-height: 1.4;
  color: var(--text-body);
}

.house-section {
  font-family: var(--font-display);
  font-size: clamp(28px, 4vw, 40px);
  font-weight: 600;
  line-height: 1.15;
  letter-spacing: -0.01em;
  color: var(--text);
}

.house-eyebrow {
  font-family: var(--font-mono);
  font-size: var(--text-sm);
  font-weight: 600;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--text-muted);
}

/* ── Long-form reading prose ───────────────────────────────────────────────
   Serif body (Martel), display heads (IBM Plex Sans), generous leading and
   rhythm. Each surface supplies its own reading measure (max-width) on the
   wrapper. */

.prose {
  font-family: var(--font-serif-stack);
  font-size: 17px;
  font-weight: 400;
  line-height: 1.72;
  letter-spacing: 0.008em;
  color: var(--text-body);
}

.prose *::selection {
  background: var(--accent);
  color: var(--accent-contrast);
}

.prose p {
  margin: 0 0 24px;
}

.prose h1 {
  font-family: var(--font-display);
  font-size: clamp(32px, 5vw, 44px);
  font-weight: 700;
  line-height: 1.1;
  letter-spacing: -0.01em;
  color: var(--text-body);
  margin: 0 0 24px;
}

.prose h2 {
  font-family: var(--font-display);
  font-size: clamp(28px, 4vw, 40px);
  font-weight: 600;
  line-height: 1.15;
  letter-spacing: -0.01em;
  color: var(--text-body);
  margin: 64px 0 32px;
  scroll-margin-top: 24px;
}

.prose h3 {
  font-family: var(--font-display);
  font-size: 22px;
  font-weight: 600;
  line-height: 1.3;
  letter-spacing: -0.01em;
  color: var(--text-body);
  margin: 40px 0 20px;
  scroll-margin-top: 24px;
}

.prose h4 {
  font-family: var(--font-display);
  font-size: 17px;
  font-weight: 600;
  line-height: 1.4;
  color: var(--text-body);
  margin: 28px 0 8px;
}

.prose ul,
.prose ol {
  margin: 28px 0;
  padding-left: 24px;
}

.prose ul {
  list-style: none;
}

.prose ul li {
  position: relative;
  padding-left: 20px;
}

.prose ul li::before {
  content: '\2014';
  position: absolute;
  left: 0;
  color: var(--text-faint);
}
/* Em-dash markers — the house list style. Ordered lists keep real numbers. */

.prose ol {
  list-style: decimal;
}

.prose ol li {
  padding-left: 4px;
}

.prose li {
  margin-bottom: 10px;
}

.prose strong {
  font-weight: 600;
  color: var(--text);
}

.prose em {
  font-style: italic;
}

.prose a {
  color: var(--accent);
  text-decoration: none;
  border-bottom: 1px solid var(--accent-soft);
  transition: border-color 0.2s;
}

.prose a:hover {
  border-color: var(--accent);
}

.prose blockquote {
  font-family: var(--font-serif-stack);
  font-size: clamp(22px, 3vw, 32px);
  font-weight: 400;
  font-style: italic;
  line-height: 1.3;
  letter-spacing: -0.01em;
  color: var(--text-body);
  margin: 48px 0;
  padding: 40px 0;
  border-top: 2px solid var(--border);
  border-bottom: 1px solid var(--border);
  text-align: center;
}

.prose blockquote p {
  margin: 0;
}

.prose hr {
  border: none;
  border-top: 1px solid var(--border);
  margin: 64px 0;
}

.prose code {
  font-family: var(--font-mono);
  font-size: 14px;
  background: var(--surface-sunken);
  padding: 2px 6px;
  border-radius: var(--radius-sm);
  color: var(--text);
}

.prose pre {
  margin: 28px 0;
  border-radius: 8px;
  overflow-x: auto;
}

.prose pre code {
  background: none;
  padding: 0;
  border-radius: 0;
}

.prose table {
  width: 100%;
  border-collapse: collapse;
  margin: 28px 0;
  font-size: 15px;
}

.prose th {
  text-align: left;
  padding: 8px 12px;
  border-bottom: var(--border-width-strong) solid var(--border);
  color: var(--text);
  font-weight: 600;
  font-family: var(--font-serif-stack);
}

.prose td {
  padding: 8px 12px;
  border-bottom: var(--border-width) solid var(--border);
  color: var(--text-body);
}

.prose tr:hover td {
  background: var(--surface);
}

/* First child sits flush to the top of the reading column (no stray leading
   margin above the opening heading/paragraph). */
.prose > :first-child {
  margin-top: 0;
}

/* ── Compact reading variant ────────────────────────────────────────────────
   For reference docs (the /docs viewer): keeps the serif body + display heads
   but dials heading sizes and rhythm down to suit dense, scannable material —
   and trades the dramatic centered blockquote for a quiet left-border one. */

.prose.compact {
  font-size: 16px;
  line-height: 1.65;
  letter-spacing: 0;
}

.prose.compact h1 {
  font-size: var(--text-xl); /* 28px */
  margin: 0 0 16px;
}

.prose.compact h2 {
  font-size: var(--text-lg); /* 22px */
  font-weight: 500;
  margin: 44px 0 14px;
}

.prose.compact h3 {
  font-size: 18px;
  margin: 28px 0 8px;
}

.prose.compact h4 {
  font-size: 15px;
  margin: 20px 0 6px;
}

.prose.compact p,
.prose.compact li {
  font-size: 16px;
}

.prose.compact ul,
.prose.compact ol {
  margin: 16px 0;
}

.prose.compact blockquote {
  font-size: clamp(18px, 2.4vw, 24px);
  margin: 28px 0;
  padding: 4px 0 4px 20px;
  border: none;
  border-left: 3px solid var(--border);
  text-align: left;
}

.prose.compact hr {
  margin: 36px 0;
}
```

### `index.css`

Just the import order — the tokens are consumed directly as `var(--…)`, so
there is no bridge layer and no framework import:

```css
@import './tokens.css';
@import './base.css';
@import './typography.css';
```

That is the entire file. If it ever grows a `@theme`, a `@custom-variant`, or a
framework import, something has been reintroduced that this stack removed on
purpose.

**Dark mode needs no variant machinery.** `tokens.css` remaps under
`:root[data-theme='dark']`, so every `var(--…)` follows automatically. A
component that needs a *structural* dark-mode difference (not just a color)
writes it in its own module:

```css
:global([data-theme='dark']) .icon { display: none; }
```

## The four touch-points (App Router setup)

Next collapses the document entry point into `layout.tsx`, so the setup lands
in four touch-points:

1. **`src/styles/`** — copy all four files verbatim (with the `tokens.css`
   font-variable adaptation above).
2. **Import in `layout.tsx`** — `import '#/styles/index.css'`. The single
   global stylesheet; every other style is a co-located module. See `shell.md`.
3. **Fonts via `next/font/google` in `layout.tsx`** — self-hosted, no CDN
   `<link>` tags, no preconnects to manage. See `shell.md`.
4. **`data-theme="dark"` + `suppressHydrationWarning` on `<html>`, plus
   `<ThemeInit />`** — see below.

## Theme: `ThemeInit` + `ThemeToggle`

A zero-flash theme is classically done by running an inline `<script>` in
`<head>` *before* React ever mounts — but that trick only works in a plain
SPA with one static `index.html`. Next's root layout is server-rendered, so
there's no equivalent blocking-script slot that also covers client-side
navigations between routes. The proven reference app instead uses a client
component that corrects `data-theme` on every mount:

`src/components/theme-init/theme-init.tsx` — one folder per component, every
component, no exceptions (`project-standard`). `theme-init` has no styling of its
own, so it is the one case with no paired `.module.css`:

```tsx
'use client'

import { useLayoutEffect } from 'react'

export function ThemeInit() {
  useLayoutEffect(() => {
    try {
      const stored = localStorage.getItem('site-theme')
      const theme =
        stored === 'light' || stored === 'dark'
          ? stored
          : matchMedia('(prefers-color-scheme: dark)').matches
            ? 'dark'
            : 'light'
      document.documentElement.setAttribute('data-theme', theme)
    } catch {}
  }, [])

  return null
}
```

**Known trade-off, not yet fixed:** because the server always renders
`data-theme="dark"` first, a visitor whose stored preference is `light` can
see one frame of dark before `ThemeInit`'s effect corrects it after
hydration — the effect runs after hydration, not before the browser's first
paint. This is the accepted v1 shape, ported as-is from the proven reference
rather than inventing an untested fix. If this flash is visible enough in
practice to bother, the fix is a blocking inline `<script>` in `layout.tsx`
via `dangerouslySetInnerHTML` (running the theme-init logic before first
paint, from the layout's `<body>`) — flag it as friction and it can be added
then.

`src/components/theme-toggle/theme-toggle.tsx` — plus the `'use client'`
directive App Router requires for anything with interactivity:

```tsx
'use client'

import { Moon, Sun } from 'lucide-react'
import styles from './theme-toggle.module.css'

export function ThemeToggle() {
  function toggle() {
    const root = document.documentElement
    const next = root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark'
    root.setAttribute('data-theme', next)
    try {
      localStorage.setItem('site-theme', next)
    } catch {}
  }

  return (
    <button
      type="button"
      onClick={toggle}
      aria-label="Toggle light or dark theme"
      className={styles.toggle}
    >
      <Moon size={16} className={styles.moon} />
      <Sun size={16} className={styles.sun} />
    </button>
  )
}
```

`src/components/theme-toggle/theme-toggle.module.css` — the template for
styling a primitive: every value a token, and the dark-mode difference expressed
with `:global([data-theme='dark'])` rather than a framework variant.

```css
.toggle {
  position: fixed;
  top: var(--space-5);
  right: var(--space-5);
  z-index: 40;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2.25rem;
  height: 2.25rem;
  border: 1px solid var(--border);
  border-radius: 999px;
  background: var(--surface);
  color: var(--text-muted);
  transition: color var(--dur-base) var(--ease);
}

.toggle:hover {
  color: var(--text);
}

/* One icon shows per theme. Light shows the moon (what you'd switch to). */
.sun {
  display: none;
}

:global([data-theme='dark']) .moon {
  display: none;
}

:global([data-theme='dark']) .sun {
  display: block;
}
```

Both are rendered once, in `layout.tsx` (see `shell.md`) — the App Router's
single shared shell for every route.

### `src/components/index.ts` — the one barrel

`src/components/` is a library, so it gets a single root barrel and the folders
below carry no `index.ts` of their own:

```ts
export * from './theme-init/theme-init'
export * from './theme-toggle/theme-toggle'
```

Consumers import `#/components`. App-internal folders (`app/_components/`, a
route's own `_components/`) get **no** barrel at all — import the named file.

## The home page

`src/app/page.tsx` — minimal centered placeholder:

```tsx
import Link from 'next/link'
import { appMeta } from '#/app-meta'
import styles from './page.module.css'

export default function HomePage() {
  return (
    <main className={styles.main}>
      <h1 className="house-section">{appMeta.name}</h1>
      <Link href="/docs" className={styles.link}>
        read the plan →
      </Link>
    </main>
  )
}
```

`house-section` stays a bare global class — it is an L1 typography *role* from
`typography.css`, not a utility. Roles are the one thing components reference by
name rather than restyling.

`src/app/page.module.css`:

```css
.main {
  display: flex;
  min-height: 100vh;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: var(--space-4);
  padding-inline: var(--space-6);
  text-align: center;
  background: var(--bg);
  color: var(--text);
}

.link {
  color: var(--accent);
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  text-decoration: underline;
  text-decoration-color: var(--accent-soft);
  text-underline-offset: 4px;
  transition: text-decoration-color var(--dur-base) var(--ease);
}

.link:hover {
  text-decoration-color: var(--accent);
}
```

Uses `next/link`, not a plain `<a>`. See `docs.md`: `eslint-config-next`'s
`no-html-link-for-pages` rule flags a plain anchor elsewhere in this skill
(the docs sidebar's link back to `/`), so both internal links here use
`next/link` consistently rather than leaning on that rule's inconsistent
detection.

## The rules

**One obvious way to express a style**, so an agent never flips a coin between
idioms and two sessions never diverge. The layers, values flowing up from L0:

```
 L3  scope override    .compact { --h2-size: 1.5rem }   redefine a token per subtree
 L2  component module  .card { … }                      the workhorse
 L1  base elements     h1,h2,p { … var(--…) }           reset + bare-tag floor
 L0  tokens            :root { --surface --ink … }      the values, one source
```

- **`src/styles/` is the frozen baseline** — read-only during feature work. Every
  component styles itself with a co-located `*.module.css` beside its `.tsx`.
- **Every value is a token.** Never a raw color above L0; raw numbers only for a
  one-off structural value with no sensible token (a `z-index`, a 20px icon box).
- **No Tailwind, no PostCSS, no CSS-in-JS.** Adding a utility class is the
  regression — with the `@theme` bridge gone it fails *silently*, rendering
  unstyled rather than erroring.
- Two idioms recur and are the house style: **base + modifier**
  (`` className={`${styles.thumb} ${error ? styles.thumbError : ''}`} ``, the
  modifier setting only the delta) and **active/inactive conditional class**
  (base always on, exactly one of `.xActive` / `.xInactive` added).
- Style children through the **direct-child** combinator (`.card > h2`), not a
  descendant selector, which would reach into nested child components. Keep
  descendant rules one level deep.

## What "done" looks like

- `pnpm dev` renders the home page, centered, with correct Paper & Ink colors
- Theme toggle works: clicking swaps light ↔ dark
- Reloading the page after toggling to light shows the known one-frame flash
  described above — confirm it's there (or isn't) but don't treat either as a
  bug to silently fix; it's a documented trade-off, not an oversight
