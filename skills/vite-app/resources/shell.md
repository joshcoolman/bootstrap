# Part: shell

The base scaffold every public app starts from. Sets up the full toolchain,
project conventions, and Vercel deployment target. No app logic — just the
runnable skeleton with correct configuration.

## Stack

- **Runtime:** Node.js / pnpm (not npm)
- **Bundler:** Vite with `@tanstack/router-plugin/vite`
- **Framework:** React 19
- **Language:** TypeScript (strict)
- **Styles:** Tailwind CSS v4 (via `@tailwindcss/vite`)
- **Router:** TanStack Router (file-based, auto-generated route tree)
- **Testing:** Vitest + React Testing Library
- **Linting:** ESLint (flat config)
- **Formatting:** Prettier
- **Deploy target:** Vercel (SPA / static)

## Key files and their purpose

### `index.html`

The Vite entry point. Put three things here directly (not in `__root.tsx`):
1. The pre-paint theme script — runs before React so there's no flash
2. Google Fonts `<link>`s — preconnects + the IBM Plex Sans/Martel/Space Mono stylesheet
3. `data-theme="light"` on `<html>` — the default before the script corrects it

```html
<!doctype html>
<html lang="en" data-theme="light">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>your-app</title>
    <script>try{var t=localStorage.getItem('site-theme');if(t!=='light'&&t!=='dark'){t=matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light'}document.documentElement.setAttribute('data-theme',t)}catch(e){}</script>
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:ital,wght@0,400;0,500;0,600;0,700;1,400;1,500&family=Martel:wght@400;600;700;800&family=Space+Mono:wght@400;700&display=swap" rel="stylesheet" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### `src/main.tsx`

Import the CSS here (not via `?url` in the router). This is what makes Tailwind
process and inject all utilities.

```ts
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { createRouter, RouterProvider } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'
import './styles/index.css'

const router = createRouter({ routeTree, scrollRestoration: true, defaultPreload: 'intent' })

declare module '@tanstack/react-router' {
  interface Register { router: typeof router }
}

createRoot(document.getElementById('root')!).render(
  <StrictMode><RouterProvider router={router} /></StrictMode>
)
```

### `package.json`

Scripts: `dev`, `build`, `preview`, `test`, `lint`, `format`. Dev port is
app-specific — pick one that doesn't conflict with sibling repos. Use pnpm.

Core deps: `react`, `react-dom`, `@tanstack/react-router`.
Dev deps: `vite`, `@vitejs/plugin-react`, `@tanstack/router-plugin`,
`tailwindcss`, `@tailwindcss/vite`, `typescript`, `vitest`,
`@testing-library/react`, `@testing-library/jest-dom`, `eslint`,
`eslint-config-prettier`, `prettier`, `lucide-react`.

Note: the Vite plugin package is `@tanstack/router-plugin`, not
`@tanstack/react-router-plugin`.

### `vite.config.ts`

All three plugins are required. `tailwindcss()` must be present or Tailwind
utility classes will not be generated — the CSS custom properties in `tokens.css`
will still load, but `bg-surface`, `text-text-muted`, `flex`, `hidden`, etc.
will all be missing.

Do NOT put `test` config here — use a separate `vitest.config.ts` (see below).
vitest v2 bundles its own Vite 5 types; mixing `defineConfig` from `vitest/config`
into this file causes a type conflict with the Vite 6 plugins.

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [TanStackRouterVite({ autoCodeSplitting: true }), tailwindcss(), react()],
  resolve: { alias: { '#': '/src' } },
})
```

The `#` alias maps to `src/` — use `#/components/foo` not `../../components/foo`.

### `vitest.config.ts`

Keep test config separate from `vite.config.ts` to avoid a Vite 5/6 type conflict.
Repeat the `resolve.alias` so the `#` import path works in tests.
Set `passWithNoTests: true` so `pnpm test` exits 0 on a fresh scaffold with no test
files yet.

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  resolve: { alias: { '#': '/src' } },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test-setup.ts'],
    passWithNoTests: true,
  },
})
```

Also create `src/test-setup.ts`:

```ts
import '@testing-library/jest-dom'
```

### `tsconfig.json`

Strict mode on. Include `noUncheckedIndexedAccess: true`. Target `ES2022`.
Path alias: `"#/*": ["./src/*"]`.

Add `"types": ["vite/client"]` so `import.meta.glob` is typed throughout `src/`.
Do NOT include `vite.config.ts` in `include` — it lives outside the app's type
context and including it causes the same Vite 5/6 conflict as above.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "types": ["vite/client"],
    "paths": { "#/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

### `tsr.config.json`

```json
{ "routesDirectory": "./src/routes", "generatedRouteTree": "./src/routeTree.gen.ts" }
```

### `pnpm-workspace.yaml`

Required to allow esbuild's postinstall script (pnpm v9+ blocks native
build scripts by default):

```yaml
allowBuilds:
  esbuild: true
onlyBuiltDependencies:
  - esbuild
```

### `eslint.config.js` / `prettier.config.js`

Standard flat ESLint config with TypeScript and React rules. Prettier with
single quotes, no semicolons, trailing commas. Keep `.prettierignore` to
exclude `src/routeTree.gen.ts` and `dist/`.

Add `'no-empty': ['error', { allowEmptyCatch: true }]` — the `catch {}` pattern
in `theme-toggle.tsx` is intentional and triggers this rule by default.

### `src/app-meta.ts`

```ts
export type AppMeta = { name: string; tagline: string; repo: string }

export const appMeta: AppMeta = {
  name: 'your-app-name',
  tagline: 'One-line description of what goes in and what comes out.',
  repo: 'https://github.com/joshcoolman/your-app-name',
}
```

### `src/routes/__root.tsx`

Simple outlet wrapper — no full document render, no stylesheet link (CSS is
imported in `main.tsx`). Just the ThemeToggle and the Outlet:

```tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'
import { ThemeToggle } from '#/components/theme-toggle'

export const Route = createRootRoute({
  component: () => (
    <>
      <ThemeToggle />
      <Outlet />
    </>
  ),
})
```

### `src/routes/index.tsx`

Thin — wires the `Home` component to `/`:

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { Home } from '#/components/home'

export const Route = createFileRoute('/')({ component: Home })
```

### `src/components/home.tsx`

Minimal centered placeholder that renders `appMeta.name` and a link to `/docs`.
Uses `house-section` (not `house-title`) for the app name:

```tsx
import { appMeta } from '#/app-meta'

export function Home() {
  return (
    <main className="bg-bg text-text flex min-h-screen flex-col items-center justify-center gap-4 px-6 text-center">
      <h1 className="house-section">{appMeta.name}</h1>
      <a
        href="/docs"
        className="text-accent font-mono text-xs underline decoration-[var(--accent-soft)] underline-offset-4 transition-colors hover:decoration-[var(--accent)]"
      >
        read the plan →
      </a>
    </main>
  )
}
```

### `src/routeTree.gen.ts`

Auto-generated by TanStack Router. Never edit by hand.

### `public/`

Standard public folder: `favicon.ico`, `manifest.json` (update app name),
`robots.txt`, `logo192.png`, `logo512.png`.

## What "done" looks like

- `pnpm install` runs clean
- `pnpm dev` serves the app on the chosen port
- `pnpm build` produces a `dist/` folder — **must run `pnpm dev` at least once first**
  so the TanStack Router plugin can generate `src/routeTree.gen.ts`; without it, `tsc`
  will fail because `src/main.tsx` imports from that file
- `pnpm test` exits 0 (no test files is fine — `passWithNoTests: true` is set)
- `pnpm lint` passes
- Home route renders, centered, with correct Paper & Ink colors
- Theme toggle works: clicking swaps light ↔ dark with no flash on reload
