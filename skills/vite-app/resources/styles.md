# Part: styles

The "Paper & Ink" design system — warm paper, soft near-black ink, one muted
clay accent, with light and dark modes. A self-contained, portable unit: copy
`src/styles/` and wire five touch-points and the whole visual identity moves.

## The four CSS files

```
src/styles/
  index.css       entry — Tailwind import + partials + the @theme bridge
  tokens.css      the design contract: colors, fonts, scales (light + dark)
  base.css        html/body/root element styling
  typography.css  house-* roles + the .prose reading layer + .compact variant
```

`tokens.css` is the single source of truth. Everything else reads from it via
`var(--token-name)`. **To reskin the whole app, edit `tokens.css` only.**

### `tokens.css`

Copy verbatim. Defines CSS custom properties on `:root` (light defaults), with
`[data-theme='light']` restated (so a nested light frame beats a dark ancestor)
and a `[data-theme='dark']` block. Key values:

```css
:root {
  --bg: #e9e6df;
  --surface: #f1efe8;
  --surface-raised: #f7f5ef;
  --surface-sunken: rgba(26, 22, 16, 0.045);
  --text: #1a1a18;
  --text-body: #26241f;
  --text-muted: #6a655b;
  --text-faint: #938d81;
  --border: #c7c1b4;
  --border-soft: rgba(26, 22, 16, 0.13);
  --border-faint: rgba(26, 22, 16, 0.07);
  --accent: #8f523a;
  --accent-contrast: #f7f5ef;
  --accent-soft: rgba(143, 82, 58, 0.12);

  --font-display: 'IBM Plex Sans', system-ui, sans-serif;
  --font-serif-stack: 'Martel', Georgia, serif;
  --font-body: 'IBM Plex Sans', system-ui, sans-serif;
  --font-mono: 'Space Mono', 'SF Mono', Menlo, monospace;

  /* spacing: --space-1 (4px) through --space-9 (80px) */
  /* radius: --radius-sm (2px) through --radius-pill (999px) */
  /* type: --text-xs (11px) through --text-title (clamp(36px,6vw,56px)) */
  /* motion: --ease, --dur-fast, --dur-base, --dur-slow */
  /* border-width: --border-width (1px), --border-width-strong (2px) —
     typography.css's .prose table rules read these directly */
}

[data-theme='dark'] {
  --bg: #14130f;
  --surface: #1f1d18;
  --surface-raised: #262320;
  --surface-sunken: rgba(0, 0, 0, 0.3);
  --text: #ece7dd;
  --text-body: #d7d1c5;
  --text-muted: #9c958a;
  --text-faint: #6e685e;
  --border: #38342d;
  --border-soft: rgba(255, 255, 255, 0.12);
  --border-faint: rgba(255, 255, 255, 0.07);
  --accent: #c67e5e;
  --accent-contrast: #14130f;
  --accent-soft: rgba(198, 126, 94, 0.16);
}
```

### `base.css`

Copy verbatim. Sets `background`/`color` on `html`/`body` from tokens. Uses
`#root` (not `#app`) as the min-height target for a Vite SPA. Adds a
`quickFadeIn` animation on `body` (respects `prefers-reduced-motion`). No
`* { margin:0 }` — Tailwind v4 Preflight already handles that; an unlayered
reset here would override every margin/padding utility.

```css
/* ════════════════════════════════════════════════════════════════════════════
   base.css — base element styling (html / body).

   Deliberately minimal. NOTE: there is no `* { margin:0; padding:0 }` reset —
   Tailwind v4's Preflight already does that inside `@layer base`. An UNLAYERED
   copy here would override every margin/padding utility (mx-auto, px-*, pt-*, …)
   because unlayered CSS beats any @layer, so it must not exist.
   ════════════════════════════════════════════════════════════════════════════ */

html {
  background: var(--bg);
}

html,
body,
#root {
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

Copy verbatim. Two things:

1. **House typographic roles** — `.house-title`, `.house-dek`, `.house-section`,
   `.house-eyebrow`. Apply by class name anywhere.

2. **`.prose`** — long-form reading: Martel serif body, IBM Plex Sans display
   heads, em-dash list markers, generous rhythm. Each surface sets its own
   `max-width` on the wrapper — `.prose` itself sets none.

   **`.prose.compact`** — for the `/docs` viewer. Same type system, smaller
   sizes, tighter rhythm, left-border blockquote instead of dramatic pull quote.

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
   body = Martel serif, mono = Space Mono). This is plain CSS, not the Tailwind
   typography plugin — it is the single source of truth for reading type, so it
   can be ported or restyled without depending on a plugin.
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

Order matters (`@import` must come first):

1. `@import 'tailwindcss'`
2. `@import './tokens.css'`
3. `@import './base.css'`
4. `@import './typography.css'`
5. `@custom-variant dark ([data-theme='dark'] &)` — binds Tailwind's `dark:`
   to `data-theme` instead of `prefers-color-scheme`
6. `@theme inline { ... }` — re-exposes tokens as Tailwind utilities

The `@theme inline` block must use this exact mapping (note `--font-sans` and
`--font-serif`, not `--font-display` and `--font-mono` as the utility names):

```css
@theme inline {
  --color-bg: var(--bg);
  --color-surface: var(--surface);
  --color-surface-raised: var(--surface-raised);
  --color-surface-sunken: var(--surface-sunken);
  --color-text: var(--text);
  --color-text-body: var(--text-body);
  --color-text-muted: var(--text-muted);
  --color-text-faint: var(--text-faint);
  --color-border: var(--border);
  --color-border-soft: var(--border-soft);
  --color-border-faint: var(--border-faint);
  --color-accent: var(--accent);
  --color-accent-contrast: var(--accent-contrast);
  --color-accent-soft: var(--accent-soft);

  --font-sans: var(--font-body);
  --font-serif: var(--font-serif-stack);
  --font-display: var(--font-display);
  --font-mono: var(--font-mono);
}
```

## The five touch-points (SPA setup)

This wiring is for a plain Vite SPA (no SSR). All five go in `index.html` and
`src/main.tsx` — NOT in `__root.tsx`.

1. **`src/styles/`** — copy all four files verbatim
2. **Import in `main.tsx`** — `import './styles/index.css'`. This is what makes
   Tailwind process utilities. Do NOT use `?url` import in the router.
3. **Font `<link>`s in `index.html`** — preconnects + Google Fonts:
   `IBM+Plex+Sans:ital,wght@0,400;0,500;0,600;0,700;1,400;1,500`
   `Martel:wght@400;600;700;800`
   `Space+Mono:wght@400;700`
4. **Pre-paint theme script in `index.html`** — inline `<script>` before the
   font links:
   ```js
   try{var t=localStorage.getItem('site-theme');if(t!=='light'&&t!=='dark'){t=matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light'}document.documentElement.setAttribute('data-theme',t)}catch(e){}
   ```
5. **`data-theme="light"` on `<html>` in `index.html`** — the static default;
   the script corrects it before first paint.

## Theme toggle

`src/components/theme-toggle.tsx` — fixed button (top-right) that flips
`data-theme` on `<html>` and saves to `localStorage['site-theme']`. Uses CSS
`dark:hidden` / `hidden dark:block` for icon swap — no React state needed
because the `@custom-variant dark` binding makes it work via the DOM attribute:

```tsx
import { Moon, Sun } from 'lucide-react'

export function ThemeToggle() {
  function toggle() {
    const root = document.documentElement
    const next = root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark'
    root.setAttribute('data-theme', next)
    try { localStorage.setItem('site-theme', next) } catch {}
  }

  return (
    <button
      type="button"
      onClick={toggle}
      aria-label="Toggle light or dark theme"
      className="border-border bg-surface text-text-muted hover:text-text fixed top-5 right-5 z-40 inline-flex h-9 w-9 items-center justify-center rounded-full border transition-colors"
    >
      <Moon size={16} className="dark:hidden" />
      <Sun size={16} className="hidden dark:block" />
    </button>
  )
}
```

Render it once in `__root.tsx`.

## The one rule

**Feature styles never go in `src/styles/`.** That folder is the frozen
baseline — treat it as read-only during feature work.

- Default: use token-based Tailwind utilities in the component
  (`className="bg-surface text-text-muted rounded-md"`).
- If a feature genuinely needs custom CSS: co-locate it as
  `src/features/<x>/<x>.module.css`.
