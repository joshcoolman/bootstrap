# Part: shell

> **Note.** This file compares against `vite-app`, a Vite + TanStack Router
> shell that has since been deleted. Those comparisons are kept because the
> *reasoning* still explains why each choice was made — but the referent is
> history, not something you can go read. Next.js is the only shell now.

The base scaffold every server-capable app starts from. Next.js App Router
instead of Vite — same conventions, project shape, and Vercel deploy target as
`vite-app`, adapted to a framework that type-checks and can host real
server-side compute. No app logic — just the runnable skeleton with correct
configuration.

## Stack

- **Runtime:** Node.js / pnpm (not npm)
- **Framework:** Next.js 16, App Router
- **Language:** TypeScript (strict)
- **Styles:** Tailwind CSS v4 (via `@tailwindcss/postcss` — PostCSS, not a Vite
  plugin)
- **Testing:** Vitest + React Testing Library
- **Linting:** ESLint (flat config, `eslint-config-next`)
- **Formatting:** Prettier
- **Deploy target:** Vercel (Next native — server-rendered, not static)

## Why this exists (not just a Vite reskin)

`vite-app` produces a client-only SPA — no story for server-side compute,
secrets, or real per-user auth. That gap was confirmed empirically: a live
app (`effective`) started on `vite-app`, hit a hard wall (Vite's dev/build
split can't host a Vercel serverless function reliably — `vercel dev` imports
raw `.ts` while `vercel build` transpiles per-file), and migrated to Next.js
mid-build. Everything framework-agnostic (the Paper & Ink styles, the
`knowledge/` folder, the `src/features/` seam) ported straight across; only
the shell needed replacing. This skill is that shell, drafted directly from
what's now proven in that migration.

## Key files and their purpose

### `package.json`

Dev port is app-specific — pick one that doesn't conflict with sibling repos
(`<dev-port>` below). Use pnpm.

```json
"scripts": {
  "dev": "next dev -p <dev-port>",
  "build": "next build",
  "start": "next start",
  "test": "vitest run",
  "lint": "eslint src",
  "format": "prettier --write ."
}
```

Unlike `vite-app`, **`build` does NOT need a `tsc --noEmit &&` prefix** —
`next build` type-checks the whole project natively as part of the build.
Vite's build only checks that `src/routeTree.gen.ts` exists; Next's doesn't
have that gap.

Install dependencies with `pnpm add` — never hand-write the `dependencies`/
`devDependencies` blocks (the same lesson `vite-app` learned the hard way: a
hand-picked version pin turned out type-incompatible with what everything
else resolved to). Let `pnpm add` resolve everything together:

```
pnpm add next react react-dom lucide-react server-only
pnpm add -D typescript@^5 @types/node @types/react @types/react-dom tailwindcss @tailwindcss/postcss eslint@^9 eslint-config-next eslint-config-prettier prettier vitest @testing-library/react @testing-library/jest-dom jsdom
```

`lucide-react` and `server-only` are **production** dependencies, not dev
deps — `theme-toggle.tsx` renders icons in the shipped bundle, and
`server-only` is imported by server-side modules (see `knowledge.md`) so Next
fails the build loudly if one is ever pulled into a client bundle by mistake,
instead of silently shipping `node:fs` to the browser. `jsdom` must be
installed explicitly; vitest's `environment: 'jsdom'` setting doesn't pull it
in on its own. `react-markdown`/`remark-gfm` are added later, in the Docs
step — same sequencing as `vite-app`.

**`typescript@^5` and `eslint@^9` are real, confirmed pins, not hand-picked
guesses** — the two exceptions to "let `pnpm add` resolve everything together."
Both were caught live in a fresh scaffold, and both fail as *tool crashes*
rather than as anything a reader would recognise as a version problem.

Left unpinned, `pnpm add -D typescript` resolves TypeScript **7.x** (currently
7.0.2, the native rewrite). Next 16.2.10 does not recognise it as a TypeScript
install at all: `next build` compiles, reaches "Running TypeScript ...", then
announces "It looks like you're trying to use TypeScript but do not have the
required package(s) installed," re-runs `pnpm add -D typescript` on its own,
and dies with `The "id" argument must be of type string. Received undefined`.
Nothing in that output names the version, so it reads as a broken scaffold.
Re-check this pin when Next ships a release that detects TypeScript 7.

Left unpinned,
`pnpm add` resolves `eslint@10.x`, which `eslint-config-next@16.2.10`
declares support for (its peer range is `>=9.0.0`, no upper bound) but which
its own bundled `eslint-plugin-react@7.37.5` does not — running `pnpm lint`
crashes outright with `TypeError: contextOrFilename.getFilename is not a
function`, not a lint finding but a tool crash. Confirmed live in a fresh
scaffold. Re-check this pin whenever `eslint-config-next` ships a version
whose bundled `eslint-plugin-react` supports ESLint 10 — it may no longer be
needed.

### `next.config.ts`

Bare. No app-specific bundler tuning (that's for a real app to add later, not
this starter):

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {}

export default nextConfig
```

### `postcss.config.mjs`

Tailwind v4 runs through PostCSS here, not a Vite plugin:

```js
const config = {
  plugins: {
    '@tailwindcss/postcss': {},
  },
}

export default config
```

### `eslint.config.mjs`

Copy verbatim (adapt point: none yet — a future layering skill may add a
`scripts/**/*.mjs` override block into this file later, before the
`prettier` entry):

```js
import { defineConfig, globalIgnores } from 'eslint/config'
import nextVitals from 'eslint-config-next/core-web-vitals'
import nextTs from 'eslint-config-next/typescript'
import prettier from 'eslint-config-prettier'

const eslintConfig = defineConfig([
  ...nextVitals,
  ...nextTs,
  prettier,
  globalIgnores(['.next/**', 'out/**', 'build/**', 'next-env.d.ts']),
])

export default eslintConfig
```

`defineConfig`/`globalIgnores` come from the `eslint` package itself (its
flat-config helper API) — no extra dependency beyond `eslint`.

### `tsconfig.json`

Strict mode on, plus `noUncheckedIndexedAccess: true` for the same reason
`vite-app` sets it — same alias convention too (`#/*` → `src/*`). Next's own
shape requires the `next` TS plugin and both type-output directories in
`include` — Next 16 generates dev-mode route types under `.next/dev/types/`
in addition to the older `.next/types/` build path, and omitting either
causes spurious type errors depending on whether you last ran `dev` or
`build`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "#/*": ["./src/*"] }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts"
  ],
  "exclude": ["node_modules"]
}
```

### `vitest.config.ts`

Same alias, same `passWithNoTests` reasoning as `vite-app` — a fresh scaffold
has no test files yet and `pnpm test` must still exit 0. Two Next-specific
additions: excluding `.next/` from vitest's file scan (Next's build output
would otherwise get walked looking for test files), and stubbing `server-only`
(see below):

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  resolve: {
    alias: {
      '#': '/src',
      // `server-only` throws on import outside a React Server Component, so any
      // test touching a server module (credentials.ts) would fail to load. The
      // real guard is `next build`, which is what actually enforces the
      // boundary; here the package just has to be inert.
      'server-only': new URL('./src/test-server-only-stub.ts', import.meta.url).pathname,
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test-setup.ts'],
    passWithNoTests: true,
    exclude: ['**/node_modules/**', '**/.next/**'],
  },
})
```

**The `server-only` alias is required, not optional** — without it the auth
step's `credentials.test.ts` cannot even load, because `credentials.ts` starts
with `import 'server-only'` and that package's non-`react-server` entry point
throws on sight: `Error: This module cannot be imported from a Client Component
module. It should only be used from a Server Component.` The error names a
Client Component, so it reads as a mistake in the test rather than an
environment artifact. Confirmed live. Nothing is lost by stubbing it: the
boundary it protects is enforced by `next build`, which is a required gate
anyway.

Also create `src/test-setup.ts`:

```ts
import '@testing-library/jest-dom'
```

and `src/test-server-only-stub.ts`:

```ts
// Inert stand-in for the `server-only` package under vitest. See the alias in
// vitest.config.ts for why. Never imported by application code.
export {}
```

### `src/app-meta.ts`

Same contract as `vite-app`'s — one small file naming the app, reused by the
layout's `<title>`, the home page, and the docs viewer's sidebar header:

```ts
export type AppMeta = { name: string; tagline: string; repo: string }

export const appMeta: AppMeta = {
  name: 'your-app-name',
  tagline: 'One-line description of what goes in and what comes out.',
  repo: 'https://github.com/joshcoolman/your-app-name',
}
```

### `src/app/layout.tsx`

Root layout replaces `index.html` + `main.tsx` + `__root.tsx` all at once —
there's no separate document entry point in App Router. Fonts are
self-hosted via `next/font/google` (no `<link>` tags, no Google Fonts CDN
round-trip) and exposed as CSS variables that `tokens.css` reads:

```tsx
import type { Metadata } from 'next'
import { IBM_Plex_Sans, Martel, Space_Mono } from 'next/font/google'
import { appMeta } from '#/app-meta'
import { ThemeInit } from '#/components/theme-init'
import { ThemeToggle } from '#/components/theme-toggle'
import '#/styles/index.css'

const ibmPlexSans = IBM_Plex_Sans({
  subsets: ['latin'],
  weight: ['400', '500', '600', '700'],
  style: ['normal', 'italic'],
  variable: '--font-ibm-plex-sans',
  display: 'swap',
})
const martel = Martel({
  subsets: ['latin'],
  weight: ['400', '600', '700', '800'],
  variable: '--font-martel',
  display: 'swap',
})
const spaceMono = Space_Mono({
  subsets: ['latin'],
  weight: ['400', '700'],
  variable: '--font-space-mono',
  display: 'swap',
})

export const metadata: Metadata = {
  title: appMeta.name,
  description: appMeta.tagline,
}

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <html
      lang="en"
      data-theme="dark"
      suppressHydrationWarning
      className={`${ibmPlexSans.variable} ${martel.variable} ${spaceMono.variable}`}
    >
      <body>
        <ThemeInit />
        <ThemeToggle />
        {children}
      </body>
    </html>
  )
}
```

`data-theme="dark"` is the static default (matches the proven reference
app); `suppressHydrationWarning` on `<html>` is required because `ThemeInit`
corrects that attribute client-side and React would otherwise flag the
mismatch. See `styles.md` for why this isn't a zero-flash pre-paint script
the way `vite-app`'s `index.html` is, and what that trade-off costs.

### `.gitignore`

```
node_modules
.next
out
build
.DS_Store
*.local
.env
*.tsbuildinfo
next-env.d.ts
.vercel
```

## What "done" looks like

- `pnpm install` runs clean
- `pnpm dev` serves the app on the chosen port
- `pnpm build` succeeds — this is also the full type-check, unlike `vite-app`
  where `tsc` is a separate prefix step
- `pnpm test` exits 0 (no test files is fine — `passWithNoTests: true` is set)
- `pnpm lint` passes
