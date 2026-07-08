# Part: styles

The same "Paper & Ink" design system as `vite-app` — warm paper, soft
near-black ink, one muted clay accent, with light and dark modes. Confirmed
byte-identical in the proven reference migration except for how fonts are
named (self-hosted via `next/font`, not loaded from a CDN `<link>`) and one
dead selector Next has no use for. Same bet as `vite-app`: `src/styles/` is a
self-contained, portable unit — copy it and five (here, four) touch-points and
the whole visual identity moves.

## The four CSS files

```
src/styles/
  index.css       entry — Tailwind import + partials + the @theme bridge
  tokens.css      the design contract: colors, fonts, scales (light + dark)
  base.css        html/body styling
  typography.css  house-* roles + the .prose reading layer + .compact variant
```

`tokens.css` is the single source of truth. Everything else reads from it via
`var(--token-name)`. **To reskin the whole app, edit `tokens.css` only.**

### `tokens.css`

Same color, spacing, radius, type-scale, motion, and border-width contract
as `vite-app`'s, adapted in one way: the four `--font-*` stacks reference
the CSS variables `next/font/google` generates in `layout.tsx`
(`--font-ibm-plex-sans`, `--font-martel`, `--font-space-mono`) instead of raw
Google Fonts family-name strings — the fonts are self-hosted, not loaded from
a CDN, so there's no `'IBM Plex Sans'` string to fall back on if the variable
isn't set yet.

**One correction from a live run, shared with `vite-app`'s copy of this
file:** the spacing/radius/type-scale/motion/border-width values below were
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

Identical to `vite-app`'s except the min-height selector drops `#root` —
App Router has no such element, so `html, body` alone covers it:

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

Byte-identical to `vite-app`'s — copy verbatim. Two things:

1. **House typographic roles** — `.house-title`, `.house-dek`, `.house-section`,
   `.house-eyebrow`. Apply by class name anywhere.
2. **`.prose`** / **`.prose.compact`** — the long-form reading layer used by
   the `/docs` viewer (see `docs.md`).

Full contents are identical to `skills/vite-app/resources/styles.md`'s
`typography.css` block — copy it as-is, no adaptation needed.

### `index.css`

Same order and same `@theme inline` mapping as `vite-app` — Tailwind v4's
import syntax doesn't change based on whether it's wired through the Vite
plugin or PostCSS:

```css
@import 'tailwindcss';
@import './tokens.css';
@import './base.css';
@import './typography.css';

@custom-variant dark ([data-theme='dark'] &);

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

## The four touch-points (App Router setup)

`vite-app` has five touch-points split across `index.html` and `main.tsx`.
Next collapses the document entry point into `layout.tsx`, so this becomes
four:

1. **`src/styles/`** — copy all four files verbatim (with the `tokens.css`
   font-variable adaptation above).
2. **Import in `layout.tsx`** — `import '#/styles/index.css'`. This is what
   makes Tailwind process utilities. See `shell.md`.
3. **Fonts via `next/font/google` in `layout.tsx`** — self-hosted, no CDN
   `<link>` tags, no preconnects to manage. See `shell.md`.
4. **`data-theme="dark"` + `suppressHydrationWarning` on `<html>`, plus
   `<ThemeInit />`** — see below. This replaces `vite-app`'s inline pre-paint
   `<script>`.

## Theme: `ThemeInit` + `ThemeToggle`

`vite-app` gets a zero-flash theme by running an inline `<script>` in
`<head>` *before* React ever mounts — a trick that only works because it's a
plain SPA with one static `index.html`. Next's root layout is
server-rendered, so there's no equivalent blocking-script slot that also
covers client-side navigations between routes. The proven reference app
instead uses a client component that corrects `data-theme` on every mount:

`src/components/theme-init.tsx`:

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
hydration — unlike `vite-app`, where the pre-paint script runs before the
browser paints anything at all. This is the accepted v1 shape, ported as-is
from the proven reference rather than inventing an untested fix. If this
flash is visible enough in practice to bother, the fix is a blocking inline
`<script>` in `layout.tsx` via `dangerouslySetInnerHTML` (the same technique
`vite-app` uses, just moved from `index.html` into the layout's `<body>`) —
flag it as friction and it can be added then.

`src/components/theme-toggle.tsx` — same behavior as `vite-app`'s, plus the
`'use client'` directive App Router requires for anything with interactivity:

```tsx
'use client'

import { Moon, Sun } from 'lucide-react'

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
      className="border-border bg-surface text-text-muted hover:text-text fixed top-5 right-5 z-40 inline-flex h-9 w-9 items-center justify-center rounded-full border transition-colors"
    >
      <Moon size={16} className="dark:hidden" />
      <Sun size={16} className="hidden dark:block" />
    </button>
  )
}
```

Both are rendered once, in `layout.tsx` (see `shell.md`) — the App Router
equivalent of `vite-app` rendering `<ThemeToggle />` in `__root.tsx`.

## The home page

`src/app/page.tsx` — minimal centered placeholder, same shape as `vite-app`'s
`home.tsx`:

```tsx
import Link from 'next/link'
import { appMeta } from '#/app-meta'

export default function HomePage() {
  return (
    <main className="bg-bg text-text flex min-h-screen flex-col items-center justify-center gap-4 px-6 text-center">
      <h1 className="house-section">{appMeta.name}</h1>
      <Link
        href="/docs"
        className="text-accent font-mono text-xs underline decoration-[var(--accent-soft)] underline-offset-4 transition-colors hover:decoration-[var(--accent)]"
      >
        read the plan →
      </Link>
    </main>
  )
}
```

Uses `next/link`, not a plain `<a>` — unlike `vite-app`'s home page. See
`docs.md`: `eslint-config-next`'s `no-html-link-for-pages` rule flags a
plain anchor elsewhere in this skill (the docs sidebar's link back to `/`),
so both internal links here use `next/link` consistently rather than
leaning on that rule's inconsistent detection.

## The one rule

**Feature styles never go in `src/styles/`.** That folder is the frozen
baseline — treat it as read-only during feature work.

- Default: use token-based Tailwind utilities in the component
  (`className="bg-surface text-text-muted rounded-md"`).
- If a feature genuinely needs custom CSS: co-locate it as
  `src/features/<x>/<x>.module.css`.

## What "done" looks like

- `pnpm dev` renders the home page, centered, with correct Paper & Ink colors
- Theme toggle works: clicking swaps light ↔ dark
- Reloading the page after toggling to light shows the known one-frame flash
  described above — confirm it's there (or isn't) but don't treat either as a
  bug to silently fix; it's a documented trade-off, not an oversight
