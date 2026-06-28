# Part: styles

The "Paper & Ink" design system — warm paper, soft near-black ink, one muted
clay accent, with light and dark modes. A self-contained, portable unit: copy
`src/styles/` plus five touch-points and the whole visual identity moves.

This is the canonical reference. `outpaint-studio` is the canonical
implementation — read its `src/styles/` and `docs/STYLE.md` for the exact
current CSS. This doc explains the structure and the rules so an agent can
apply or port it correctly.

## The four CSS files

```
src/styles/
  index.css       entry — Tailwind import + partials + the @theme bridge
  tokens.css      the design contract: colors, fonts, scales (light + dark)
  base.css        html/body element styling
  typography.css  house-* roles + the .prose reading layer + .compact variant
```

`tokens.css` is the single source of truth. Everything else reads from it via
`var(--token-name)`. **To reskin the whole app, edit `tokens.css` only.**

### `tokens.css`

Defines CSS custom properties on `:root` (light defaults) with a
`[data-theme='dark']` block that overrides them. Categories:

- **Colors:** `--bg`, `--surface`, `--surface-raised`, `--surface-sunken`,
  `--text`, `--text-body`, `--text-muted`, `--text-faint`, `--border`,
  `--border-soft`, `--border-faint`, `--accent`, `--accent-contrast`,
  `--accent-soft`
- **Fonts:** `--font-display` (IBM Plex Sans), `--font-serif-stack` (Martel),
  `--font-body` (IBM Plex Sans), `--font-mono` (Space Mono)
- **Scales:** `--text-sm`, `--text-base`, `--text-lg`, `--text-xl`,
  `--text-title`, `--text-subtitle`, and spacing/radius/duration/easing tokens

### `base.css`

Sets `background` and `color` on `html` and `body` from tokens, sets
`font-family` and `line-height`, adds a subtle `quickFadeIn` animation on
`body` (respects `prefers-reduced-motion`). No reset here — Tailwind v4's
Preflight handles that. Do NOT add `* { margin: 0; padding: 0 }` — unlayered
CSS beats `@layer` and breaks every spacing utility.

### `typography.css`

Two things:

1. **House typographic roles** — global classes for page chrome:
   `.house-title`, `.house-dek`, `.house-section`, `.house-eyebrow`.
   Apply by class name anywhere without extra props.

2. **`.prose`** — the long-form reading style. Serif body (Martel), display
   heads (IBM Plex Sans), em-dash list markers, generous rhythm. Each surface
   controls its own reading measure (`max-width`) on the wrapper — `.prose`
   itself sets no `max-width`.

   **`.prose.compact`** — a variant for dense reference material (the `/docs`
   viewer). Same type system, smaller sizes and tighter rhythm, left-border
   blockquote instead of the dramatic centered pull quote.

### `index.css`

The entry point imported by Vite. Order matters (CSS spec requires `@import`
first):

1. `@import 'tailwindcss'`
2. `@import './tokens.css'`
3. `@import './base.css'`
4. `@import './typography.css'`
5. `@custom-variant dark ([data-theme='dark'] &)` — binds Tailwind's `dark:`
   to our `data-theme` attribute instead of `prefers-color-scheme`
6. `@theme inline { ... }` — re-exposes tokens as Tailwind utilities:
   `bg-surface`, `text-muted`, `text-text-faint`, `font-serif`, etc. The
   `inline` keyword keeps them as `var()` references so they follow the runtime
   `data-theme` switch instead of freezing a value at build time.

## The five touch-points for porting

To lift the style system into a new project:

1. **`src/styles/`** — copy all four files
2. **Font `<link>`s** — in `__root.tsx`, add preconnects and the Google Fonts
   stylesheet for IBM Plex Sans, Martel, and Space Mono
3. **Pre-paint theme script** — inline `<script>` in `__root.tsx` before
   `<HeadContent />`:
   ```js
   try{var t=localStorage.getItem('site-theme');if(t!=='light'&&t!=='dark'){t=matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light'}document.documentElement.setAttribute('data-theme',t)}catch(e){}
   ```
   This runs before first paint and prevents a flash of the wrong theme.
4. **`data-theme="light"` on `<html>`** — the SSR/static default. The script
   above immediately corrects it to the user's saved preference.
5. **Tailwind v4** — `tailwindcss` + `@tailwindcss/vite`. The `@theme inline`
   bridge requires v4.

## Theme toggle (optional but standard)

`src/components/theme-toggle.tsx` — a fixed `<button>` (top-right corner) that
flips `data-theme` on `<html>` and saves to `localStorage['site-theme']`.
Sun/moon icon swap is driven by the CSS `dark:` variant (no React state, no
hydration flash). Render it once in the `<body>` of `__root.tsx`.

Requires `lucide-react` for the Sun and Moon icons.

## The one rule

**Feature styles never go in `src/styles/`.** That folder is the frozen
baseline — treat it as read-only during feature work.

- Default: use token-based Tailwind utilities in the component
  (`className="bg-surface text-text-muted rounded-md"`).
- If a feature genuinely needs custom CSS: co-locate it as
  `src/features/<x>/<x>.module.css`.
- Add a new global only when it's truly cross-cutting — and then it's a
  deliberate token/role change, not a side effect of feature work.

Following this, the baseline stays a clean, grabbable unit no matter how much
feature work accumulates above it.
