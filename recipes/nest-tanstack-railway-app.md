# Recipe: nest-tanstack-railway-app

Guide to building a **single-user, login-only** web app on a split
stack: a **NestJS** API and a **TanStack Start** (SSR) web app in one pnpm
monorepo, **Better Auth** email/password over **Drizzle + Postgres**, deployed
as **two Railway services on custom-domain subdomains** (`app.` + `api.`) with
a session cookie shared across them. The app itself is deliberately blank: `/`
renders "hello world", `/login` signs the single user in (no signup anywhere),
`/dashboard` is a blank page gated behind auth.

**Status — v0 draft, not yet run end-to-end.** Unlike this repo's other
recipes, this one does **not** compose an already-proven skill for its shell —
there is no NestJS, TanStack Start, or Better Auth skill/part in the repo yet,
so Steps 1–6 are new instruction-level content reasoned from each tool's
current docs, not code ported from a live production app. What *is* proven and
reused: the Paper & Ink token system and the three Railway/pnpm gotchas. Treat
the whole thing as a first draft to harden by building a real app from it (the
"mule"), logging friction, and folding it back — the same path every other
recipe here took to reach v1. The **version-sensitivity notice** below lists
the fast-moving API surfaces the executor must re-check against live docs
before writing the auth code.

**What this recipe composes:**
- [parts/styles.md](../parts/styles.md) — the Paper & Ink token system, copied
  verbatim in Step 2. Proven; the only adaptation is relocating its five
  touch-points from a Vite `index.html` into TanStack Start's `__root.tsx`
  document (Start has no static `index.html`).
- [FEATURE-MODULE-PATTERN.md](../FEATURE-MODULE-PATTERN.md) — the
  entities→use-cases→adapters layering, applied to the NestJS side in Step 3
  (one fixed `DomainError` union, an identity port, thin translate-only
  controllers).
- [parts/docs.md](../parts/docs.md) + [parts/cicd.md](../parts/cicd.md) — the
  `docs/` set and the install→lint→test→build CI gate (Steps 1, 7, 10).
- The three gotchas —
  [local-postgres-for-dev](../gotchas/local-postgres-for-dev.md),
  [pnpm-install-approvals](../gotchas/pnpm-install-approvals.md),
  [railway-postgres-migrations](../gotchas/railway-postgres-migrations.md) —
  baked into Steps 1, 3, and 8.
- **New inline, no standalone part/skill yet:** the Nest + Better Auth mount,
  the TanStack Start SSR guard, the Drizzle runtime-migrator, and the Railway
  multi-service Dockerfile-per-app strategy. Per this repo's authoring
  contract, extraction into parts/skills waits until each has been run for
  real once — until then it lives here directly, not split out prematurely.

This is a genuinely different auth shape from the repo's existing auth work.
`add-user-auth` is Supabase multi-user with RLS; `next-railway-app`'s Step 3 is
a hand-rolled signed-cookie whitelist. This recipe is **Better Auth** (a real
auth library) mounted inside NestJS, with the twist that the session cookie has
to cross from the API subdomain to the web subdomain. That cross-subdomain
cookie is the load-bearing design decision the whole topology is built around.

---

## Version-sensitivity notice — verify against live docs before Step 3

These surfaces move fast. The specifics below were verified against live docs
during authoring (URLs cited inline in the steps); still re-confirm before
writing auth code, because they will keep drifting:

- **Better Auth** — mount the Express bridge with `toNodeHandler(auth)` from
  `better-auth/node` (do NOT hand-roll a web-Request bridge; the library handles
  method/url/headers/body and multiple `Set-Cookie`). Config keys
  `trustedOrigins`, `advanced.crossSubDomainCookies`, `advanced.useSecureCookies`,
  `emailAndPassword.disableSignUp` are current. **Critical:** `disableSignUp`
  blocks sign-up on **both** the HTTP route and server-side `auth.api.signUpEmail`
  — there is no bypass, so the seed CLI MUST use the **admin plugin**
  (`plugins: [admin()]`) and `auth.api.createUser`. Generate the Drizzle schema
  with `npx auth@latest generate` (the CLI now lives on the npm `auth` package),
  not by hand — the admin plugin adds columns.
- **TanStack Start** — config is `vite.config.ts` with the `tanstackStart()`
  plugin from `@tanstack/react-start/plugin/vite` (**`app.config.ts` is gone**);
  Node/`.output` output needs an explicit Nitro plugin. The root document uses
  `createRootRoute({ head: () => ({...}) })` + `<HeadContent />` / `<Scripts />`.
  The SSR request accessor is `getRequest()` from `@tanstack/react-start/server`
  (renamed from `getWebRequest`), and it **must** be called inside a
  `createServerFn(...)` handler — `beforeLoad` runs on the client during SPA
  navigation, so a bare call crashes. `createFileRoute` / `beforeLoad` /
  `throw redirect()` are current.
- **NestJS CLI** — scaffold with `@nestjs/cli` (`nest new`, `nest generate`),
  not hand-written files. `nest new` has **no `--directory` flag** (run it from
  inside `apps/`); `--package-manager pnpm`, `--skip-git`, `--skip-install` are
  valid; Express is still the default platform.
- **Tailwind v4** — `@import "tailwindcss"`, `@theme inline`, `@custom-variant`,
  and the `@tailwindcss/vite` plugin (not PostCSS), composed alongside
  `tanstackStart()` in `vite.config.ts`.
- **drizzle-kit** — `drizzle.config.ts` via `defineConfig({ dialect:
  'postgresql', dbCredentials: { url } })`; `drizzle-kit generate` for SQL; a
  **plain runtime migrator** (`migrate` from `drizzle-orm/postgres-js/migrator`)
  at deploy time — the production image must not depend on `drizzle-kit`.
- **Turborepo** — the top-level key is `tasks` (v2 renamed it from `pipeline`).
- **pnpm** — `allowBuilds:` is the live key in `pnpm-workspace.yaml` (v11 removed
  `onlyBuiltDependencies`, now silently ignored).
- **Node** — pin **22 LTS** (Node 20 is EOL; Vite 7 needs ≥20.19/22.12).

**House rules for every step:** no emojis in code or UI. Features import each
other only through their public `index.ts`. Everything vendor/DB-touching is
**dormant when unconfigured** so CI stays green with zero secrets. Placeholders
`yourdomain.com`, `app.yourdomain.com`, `api.yourdomain.com` throughout — the
executor substitutes the real domain in Step 9.

---

## Step 1 — Monorepo scaffold (Turborepo + pnpm workspaces)

1. Scaffold the workspace root:
   ```
   package.json          # private; "packageManager": "pnpm@11.x"; scripts delegate to turbo
   pnpm-workspace.yaml    # packages: apps/*, packages/*  +  allowBuilds:
   turbo.json             # tasks: build (^build), lint, test, dev (persistent, no cache)
   tsconfig.base.json     # strict; path aliases to packages/*
   .nvmrc                 # 22 (Node 20 is EOL)
   .gitignore             # node_modules, dist, .env, .env.local, .turbo, .output, .nitro
   .env.example           # every env var the system reads, documented, no secrets
   docs/{OVERVIEW,SPEC,PLAN,STYLE}.md
   CLAUDE.md              # root orientation
   packages/shared/       # shared domain types (below)
   packages/config-ts/    # shared tsconfig presets + eslint flat config
   apps/                  # filled in Steps 2–3
   ```
2. `pnpm-workspace.yaml` — use `allowBuilds:` (NOT `onlyBuiltDependencies`,
   silently ignored in pnpm v11; see
   [gotchas/pnpm-install-approvals.md](../gotchas/pnpm-install-approvals.md)):
   ```yaml
   packages:
     - 'apps/*'
     - 'packages/*'
   allowBuilds:
     esbuild: true
     '@tailwindcss/oxide': true
     # add any package named by ERR_PNPM_IGNORED_BUILDS after the first install
   ```
3. `packages/shared` exports the **one fixed domain error union** and the
   identity contract, imported by both apps (this is the taxonomy the whole
   system maps to, defined once):
   ```ts
   // packages/shared/src/errors.ts
   export type DomainError =
     | { kind: 'Unauthenticated' }
     | { kind: 'Unauthorized' }
     | { kind: 'Validation'; issues: string[] }
     | { kind: 'NotFound' }
     | { kind: 'Conflict'; message?: string }

   // packages/shared/src/identity.ts
   export interface SessionUser { id: string; email: string }
   ```
4. `turbo.json` — top-level key is **`tasks`** (Turborepo v2 renamed it from
   `pipeline`): `{ "tasks": { "build": { "dependsOn": ["^build"] }, "dev": {
   "persistent": true, "cache": false }, "lint": {}, "test": {} } }`. Root
   scripts: `dev`, `build`, `lint`, `test` (plus `auth:create-user`,
   `db:generate`, `db:migrate` added in later steps).
5. Write `docs/` (OVERVIEW/SPEC/PLAN/STYLE) and root `CLAUDE.md` per
   [parts/docs.md](../parts/docs.md). `docs/PLAN.md` records the locked
   decisions (Express adapter, `toNodeHandler` Better Auth mount, admin-plugin
   seed path, Dockerfile-per-app, Pre-Deploy migrations) and the pre-DNS cookie
   limitation from Step 6 — so the executor doesn't later chase it as a bug.

**Gate:** `pnpm install` completes. **Grep the install output for
`ERR_PNPM_IGNORED_BUILDS`** and add any named package to `allowBuilds` — a
local green install is not proof (the pnpm store caches approvals; the real
test is the clean CI/Railway store in Steps 7–8). `pnpm -w exec tsc --noEmit -p
tsconfig.base.json` is clean.

## Step 2 — Web shell: TanStack Start + Tailwind v4 + Paper & Ink

Build a running UI first, so there is something to guard in Step 5.

1. Scaffold `apps/web` (TanStack Start is a **Vite plugin** now — config lives
   in `vite.config.ts`; `app.config.ts` no longer exists):
   ```
   apps/web/package.json
   apps/web/vite.config.ts                      # tanstackStart() + @tailwindcss/vite + Nitro
   apps/web/tsconfig.json
   apps/web/src/router.tsx
   apps/web/src/routes/__root.tsx               # root route: head() + <HeadContent/> + <Scripts/>
   apps/web/src/routes/index.tsx                # "/" -> blank "hello world"
   apps/web/src/routes/login.tsx                # stub now, real in Step 5
   apps/web/src/routes/dashboard.tsx            # stub now, guarded in Step 5
   apps/web/src/styles/{index,tokens,base,typography}.css
   apps/web/src/components/theme-toggle.tsx
   apps/web/src/env.ts                          # centralised env with dormant fallback
   apps/web/src/features/                        # feature modules land here
   apps/web/CLAUDE.md
   ```
   `vite.config.ts` — the plugin order matters, and Node/`.output` output needs
   an explicit Nitro plugin (`vite build` alone no longer emits a Nitro server):
   ```ts
   import { defineConfig } from 'vite'
   import { tanstackStart } from '@tanstack/react-start/plugin/vite'
   import viteReact from '@vitejs/plugin-react'
   import tailwindcss from '@tailwindcss/vite'
   import { nitro } from 'nitro/vite'   // Nitro v3; older escape hatch: @tanstack/nitro-v2-vite-plugin
   export default defineConfig({
     plugins: [tailwindcss(), tanstackStart(), nitro(), viteReact()],
   })
   ```
   Start command for the built app (web Dockerfile, Step 8):
   `node .output/server/index.mjs`.
2. **Paper & Ink — copy the four `src/styles/` files verbatim** from
   [parts/styles.md](../parts/styles.md) (`tokens.css` is the single source of
   truth: light default, `[data-theme='dark']` block, accent clay `#8f523a` /
   dark `#c67e5e`, IBM Plex Sans / Martel / Space Mono). The reference targets a
   Vite SPA with a static `index.html`; Start has none, so **relocate its five
   touch-points into `__root.tsx`'s rendered document**:
   1. Import `./styles/index.css` in the client entry — this is what makes
      Tailwind process utilities.
   2. `index.css` order is load-bearing: `@import 'tailwindcss'` →
      `@import './tokens.css'` → base → typography →
      `@custom-variant dark ([data-theme='dark'] &)` →
      `@theme inline { --color-bg: var(--bg); ... --font-sans: var(--font-body); ... }`.
   3. `__root.tsx` is a `createRootRoute({ head: () => ({ links: [...], scripts: [...] }) })`
      rendering an `<html data-theme="light">` whose `<head>` contains
      `<HeadContent />` and whose `<body>` ends with `<Scripts />`. Put the
      Google Fonts preconnect + links (IBM Plex Sans / Martel / Space Mono
      weights per the reference) in `head()`. The **pre-paint no-flash script
      must still be a raw `<script>` rendered as the FIRST element inside
      `<head>`** (before `<HeadContent />`) so `data-theme` is set before first
      paint — do not route it through `head().scripts`, which defers it:
      ```tsx
      <script dangerouslySetInnerHTML={{ __html:
        "try{var t=localStorage.getItem('site-theme');if(t!=='light'&&t!=='dark'){t=matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light'}document.documentElement.setAttribute('data-theme',t)}catch(e){}"
      }} />
      ```
      SSR caveat: the server always renders `data-theme="light"`; the inline
      synchronous script corrects it pre-paint. Keep it inline and synchronous
      to avoid both a flash and a hydration mismatch on the attribute.
   4. `theme-toggle.tsx` verbatim (flips `data-theme`, writes
      `localStorage['site-theme']`, swaps a lucide `Moon`/`Sun` via
      `dark:hidden` / `hidden dark:block`, no React state). Render once in
      `__root.tsx`.
3. **Env seam (build-time vs runtime — the tricky part, previewed here):** the
   browser bundle needs the API origin at **build time** because Vite inlines
   `VITE_*`. Put it in `apps/web/src/env.ts` with a dormant fallback so the app
   still renders "hello world" when unset:
   ```ts
   export const API_URL = import.meta.env.VITE_API_URL ?? ''          // browser -> api origin
   export const API_INTERNAL_URL =
     (typeof process !== 'undefined' && process.env.API_INTERNAL_URL) || API_URL // SSR fetch target
   ```
   Keep the public/private split explicit: browser code reads `API_URL`; SSR
   code reads `API_INTERNAL_URL`.

**Gate:** `pnpm --filter web dev` serves `/` as "hello world"; view-source
shows `data-theme` set before paint with no FOUC; toggling flips colors and
persists across reload; `pnpm --filter web build` (with the Nitro plugin
present) produces a `.output/server/index.mjs` that runs via
`node .output/server/index.mjs`, with zero type errors and **zero env vars set**.

## Step 3 — NestJS API + clean-architecture layering + Drizzle/Postgres (local)

**Two decisions, stated with their rationale so the executor doesn't
re-litigate them:**

- **Use the Express platform (`@nestjs/platform-express`), not Fastify.** Better
  Auth ships an official Express bridge (`toNodeHandler`); the integration is
  best-documented and lowest-risk on Express. Fastify's content-type parsers and
  reply-hijacking add failure surface for zero benefit here.
- **Mount Better Auth via its official `toNodeHandler(auth)` bridge behind your
  identity port — do not hand-roll a web-Request bridge.** `toNodeHandler` (from
  `better-auth/node`) handles method/url/headers/body and pipes the response
  **including multiple `Set-Cookie`** — the previously "most bug-prone spot" is
  solved library-side. Keep it behind your own `IdentityService` port so the
  rest of the app never imports Better Auth directly and CI can run it dormant.
  Better Auth's docs now also officially document `@thallesp/nestjs-better-auth`
  (which does exactly this body-parser dance for you); using it is a legitimate
  option, but the default here is the thin official bridge you control.

1. **Scaffold with the NestJS CLI, don't hand-write it.** `nest new` and `nest
   generate` produce the correct module/provider/controller wiring and their
   `.spec.ts` stubs for free — far less error-prone than authoring Nest
   boilerplate by hand:
   ```bash
   pnpm add -Dw @nestjs/cli
   # nest new has NO --directory flag — run it from inside apps/, name the app "api"
   cd apps && pnpm dlx @nestjs/cli new api --package-manager pnpm --skip-git --skip-install
   cd api
   pnpm dlx @nestjs/cli generate module   features/identity --flat=false
   pnpm dlx @nestjs/cli generate service  features/identity --flat
   pnpm dlx @nestjs/cli generate controller features/identity/auth --flat
   pnpm dlx @nestjs/cli generate filter   common/domain-error
   ```
   Then `pnpm install` at the **workspace root** (the `--skip-install` above
   avoids a standalone lockfile inside `apps/api`).
   **Monorepo reconciliation (a real friction point — do it right after `nest
   new`):** `nest new` creates a standalone project, so (a) make its
   `tsconfig.json` `extends` the workspace `tsconfig.base.json`; (b) let it keep
   its **own dependencies in `apps/api/package.json`** (pnpm workspaces hoist
   them) and remove any duplicate root-level installs; (c) `nest new` sets up
   **jest**, while the web app uses vitest — that split is fine, keep each app on
   its framework's default rather than forcing uniformity (turbo runs both under
   `pnpm test`); (d) keep the generated `nest-cli.json`, `.eslintrc`/eslint
   config aligned with `packages/config-ts`. The result is the layering below.
2. Arrange the generated files into the feature-module + clean-architecture
   layering:
   ```
   apps/api/package.json
   apps/api/tsconfig.json
   apps/api/nest-cli.json
   apps/api/Dockerfile
   apps/api/drizzle.config.ts                    # drizzle-kit generate — dev/CI only
   apps/api/src/main.ts                          # bodyParser:false; mountAuth() before json; CORS; PORT
   apps/api/src/app.module.ts
   apps/api/src/config/env.ts                    # typed reader + isAuthConfigured / isDbConfigured
   apps/api/src/common/domain-error.filter.ts    # the ONE error-union -> HTTP mapping
   apps/api/src/db/{client,schema,migrate}.ts     # globalThis-cached pool; BA tables (generated); runtime migrator
   apps/api/src/features/identity/
     types.ts            # SessionUser, AuthResult<T> = { ok:true; value } | { ok:false; error: DomainError }
     auth.ts             # betterAuth(...) instance — dormant when unconfigured; admin() plugin
     identity.service.ts # the port: getCurrentUser(headers) -> AuthResult<SessionUser>
     mount-auth.ts       # exports mountAuth(expressApp): express.all('/api/auth/{*splat}', toNodeHandler(auth))
     identity.module.ts
     index.ts            # public surface (incl. mountAuth)
     CLAUDE.md
   docker-compose.yml                            # disposable local postgres:16-alpine
   ```
3. **Layering (enforce it):** entities = domain types + the `DomainError` union
   from `packages/shared` (no upward deps). Services = auth + business
   validation enforced **here** — `IdentityService.getCurrentUser` is *the one
   place any feature asks who is asking* — depending on injected ports (DB
   client, auth instance) via Nest DI. Controllers = thin, translate
   transport↔domain and let a single Nest exception filter map the union to HTTP
   **once**: `Unauthenticated→401, Unauthorized→403, Validation→422,
   NotFound→404, Conflict→409`.
4. **DTOs at the boundary, config, and tests** (Nest-idiomatic, aligns with the
   Encore best-practices guide):
   - **DTOs vs entities:** request shapes live in `<feature>/dto/`, validated
     with `class-validator` + a global `ValidationPipe` (`whitelist: true`).
     Drizzle table types live in `db/schema.ts`. Boundary/shape validation is
     the DTO+pipe's job; **domain/business validation stays in the service** (per
     the layering above) — the pipe rejects malformed input, the service
     enforces rules. Keep them distinct, don't collapse one into the other.
   - **Config — deliberate divergence from "fail fast".** The Encore guide says
     crash on missing config. This recipe keeps **required infra** config
     fail-fast (e.g. `PORT`) but **vendor** config (`DATABASE_URL`,
     `BETTER_AUTH_SECRET`) **dormant, not fatal** — that is what lets CI build
     and boot with zero secrets. `config/env.ts` exposes typed getters plus
     `isDbConfigured` / `isAuthConfigured`; the auth/db features no-op
     observably when false. This is a conscious tradeoff, noted in `docs/PLAN.md`.
   - **Tests:** co-locate unit `.spec.ts` with the code (Nest's default); put any
     e2e specs in `apps/api/test/`. The DI container makes provider mocking a
     one-liner — use it for the identity-port contract test (Step 7).
5. **Better Auth config (`identity/auth.ts`) — env-driven, dormant when
   unconfigured** (verify keys per the version notice):
   ```ts
   import { admin } from 'better-auth/plugins'

   export const isAuthConfigured = Boolean(env.DATABASE_URL && env.BETTER_AUTH_SECRET)

   export const auth = isAuthConfigured ? betterAuth({
     database: drizzleAdapter(db, { provider: 'pg', schema }),
     baseURL: env.BETTER_AUTH_URL,                    // api origin
     secret: env.BETTER_AUTH_SECRET,
     basePath: '/api/auth',
     trustedOrigins: env.TRUSTED_ORIGINS.split(','),   // web origin(s)
     emailAndPassword: { enabled: true, disableSignUp: true },  // LOGIN ONLY
     plugins: [admin()],   // REQUIRED: the only way to create the single user (see below)
     advanced: {
       crossSubDomainCookies: env.COOKIE_DOMAIN
         ? { enabled: true, domain: env.COOKIE_DOMAIN }          // ".yourdomain.com"
         : { enabled: false },                                   // *.up.railway.app fallback
       useSecureCookies: env.NODE_ENV === 'production',          // Secure; SameSite=Lax is BA's default
     },
   }) : null   // dormant: mount/service return an observable "not configured", never throw
   ```
   **`disableSignUp: true` blocks sign-up on BOTH the public route and the
   server-side `auth.api.signUpEmail` call — there is no bypass.** So the single
   user cannot be created through sign-up at all; the seed CLI (Step 4) must use
   the **admin plugin**'s `auth.api.createUser`, which is why `admin()` is in
   `plugins` above. The admin plugin adds `role`/`banned`/… columns to `user` and
   `impersonatedBy` to `session` — one more reason to CLI-generate the schema
   (below) rather than hand-write it.
6. **Mount the handler with `toNodeHandler` — the exact Express seam:** the
   order relative to the JSON body parser is the whole game. Better Auth's docs
   warn: **do not run `express.json()` before the auth handler** or the client
   API hangs on "pending". So disable Nest's global body parser and re-apply JSON
   for everything *except* `/api/auth`:
   ```ts
   // main.ts
   import { toNodeHandler } from 'better-auth/node'
   import { json as expressJson } from 'express'
   import { auth } from './features/identity'   // dormant-safe; mountAuth no-ops when auth is null

   const app = await NestFactory.create(AppModule, { bodyParser: false })
   const server = app.getHttpAdapter().getInstance()   // the Express instance
   if (auth) server.all('/api/auth/{*splat}', toNodeHandler(auth))   // BEFORE json; Express v5 wildcard
   server.use((req, res, next) =>
     req.path.startsWith('/api/auth') ? next() : expressJson()(req, res, next))
   ```
   `toNodeHandler` handles the Node↔web Request/Response bridge and **all
   `Set-Cookie` headers** for you — no hand-rolled `toWebRequest`. Wrap the mount
   in a `mountAuth(server)` helper exported from the identity feature so the app
   never imports Better Auth outside that seam. `IdentityService.getCurrentUser`
   reads the session server-side via
   `auth.api.getSession({ headers: fromNodeHeaders(req.headers) })` (also from
   `better-auth/node`). Still write a focused test (Step 7) asserting a login
   response carries the `Set-Cookie` through.
7. **Drizzle/Postgres, gotchas baked in:** `db/client.ts` = a
   `globalThis`-cached postgres-js client (avoid connection storms / HMR
   duplication). **`db/schema.ts` is CLI-generated, not hand-written:** after
   `auth.ts` exists, run `npx auth@latest generate` (the Better Auth CLI now
   lives on the npm `auth` package) — it reads your auth config, so it emits the
   Drizzle schema **including the admin-plugin columns**, avoiding silent
   column-drift errors. Then `drizzle-kit generate` emits SQL into
   `apps/api/drizzle/`. `db/migrate.ts` = a **plain runtime migrator** (`migrate`
   from `drizzle-orm/postgres-js/migrator`) reading `DATABASE_URL` — the
   production image must NOT need `drizzle-kit`. `docker-compose.yml` runs a
   disposable `postgres:16-alpine`, creds from shell env, **local dev only —
   never point at prod DB** ([local-postgres-for-dev](../gotchas/local-postgres-for-dev.md)).
   A plain Node migrator does not auto-load `.env`; run it as
   `node --env-file=apps/api/.env.local apps/api/dist/db/migrate.js`.

**Gate:**
1. `docker compose up -d` starts local Postgres.
2. `npx auth@latest generate` writes `db/schema.ts`; `pnpm db:generate` emits
   migration SQL; `pnpm db:migrate` (with `--env-file`) applies it; psql `\dt`
   shows `user/session/account/verification` (plus admin-plugin columns).
3. `pnpm --filter api start:dev` boots; `curl localhost:3000/health` OK.
4. `curl -i -X POST localhost:3000/api/auth/sign-in/email -H 'content-type:
   application/json' -d '{...}'` returns a `Set-Cookie` (after a user exists) or
   a clean auth error (route live + body parsing works — proves the
   `toNodeHandler`-before-`json` order is right); a POST to `sign-up/email`
   returns **HTTP 400** with body code `EMAIL_PASSWORD_SIGN_UP_DISABLED` (proves
   login-only — note it's 400, not 403).
5. **CI-parity:** with all secrets unset, `pnpm --filter api build && node
   dist/main.js` boots dormant — the auth controller returns an observable "not
   configured", nothing throws.

## Step 4 — Seed CLI: `pnpm auth:create-user`

The single user has no signup path, so provisioning is a CLI. Because
`disableSignUp` blocks `signUpEmail` **even server-side** (Step 3), the only
creation path is the admin plugin's `createUser`.

1. `apps/api/src/scripts/create-user.ts` reads `AUTH_SEED_EMAIL` /
   `AUTH_SEED_PASSWORD`, imports the same `auth` instance (which has `admin()`
   in `plugins`), and creates the user server-side:
   ```ts
   await auth.api.createUser({ body: { email, password, name: email } })
   ```
   No session headers are needed for a server-side `auth.api` call.
2. Idempotent: catch the "user already exists" error, log, and exit 0 (safe to
   re-run in provisioning / as a Pre-Deploy step). No interactive prompts.
3. Root script `auth:create-user` runs the compiled script with
   `node --env-file=...`.

**Gate:** against local Postgres, `AUTH_SEED_EMAIL=… AUTH_SEED_PASSWORD=… pnpm
auth:create-user` creates exactly one `user` row via `auth.api.createUser`;
re-run reports "already exists" and exits 0; the Step 3 sign-in `curl` then
returns a valid session cookie.

## Step 5 — Auth wiring across web ↔ api

Two distinct paths: the **SSR route guard** (server→server, forwards the
cookie) and **browser login** (client→api, `credentials:'include'`).

1. Scaffold the web auth feature:
   ```
   apps/web/src/features/auth/
     client.ts          # better-auth/react createAuthClient (browser)
     session.server.ts  # SSR: fetch api /api/auth/get-session forwarding Cookie
     types.ts
     index.ts           # public surface: useSession/signIn (client) + getServerSession (SSR)
     CLAUDE.md
   apps/web/src/routes/login.tsx       # real form, NO signup link
   apps/web/src/routes/dashboard.tsx   # guarded, blank authed page
   ```
2. **Browser client (`client.ts`):**
   ```ts
   import { createAuthClient } from 'better-auth/react'
   export const authClient = createAuthClient({
     baseURL: import.meta.env.VITE_API_URL,        // api origin
     fetchOptions: { credentials: 'include' },     // send/receive the cross-origin cookie
   })
   ```
   `login.tsx` posts via `authClient.signIn.email({ email, password })`; on
   success the api sets the session cookie (scoped `.yourdomain.com` in prod),
   then `router.navigate({ to: '/dashboard' })`. No signup link anywhere.
3. **SSR session check (`session.server.ts`) — MUST be a server function.**
   `beforeLoad` runs on the **client** during SPA navigation (e.g. the
   post-login `router.navigate`), so a bare function calling the request
   accessor crashes there. Wrap it in `createServerFn` so it always executes
   server-side; the browser's cookies ride the RPC automatically. Also note the
   accessor is `getRequest()` (renamed from `getWebRequest`):
   ```ts
   import { createServerFn } from '@tanstack/react-start'
   import { getRequest } from '@tanstack/react-start/server'

   export const getServerSession = createServerFn({ method: 'GET' }).handler(async () => {
     const cookie = getRequest().headers.get('cookie') ?? ''      // FORWARD the browser cookie
     const res = await fetch(`${API_INTERNAL_URL}/api/auth/get-session`, { headers: { cookie } })
     if (!res.ok) return null
     const data = await res.json()
     return data?.user ?? null                                    // get-session returns 200 + null body when anon
   })
   ```
   Note the **two URLs**: browser uses `VITE_API_URL` (public, must resolve
   publicly); the server function uses `API_INTERNAL_URL` (same public origin, or
   a Railway private host). Default `API_INTERNAL_URL` to `VITE_API_URL` when unset.
4. **`/dashboard` guard (`beforeLoad`):**
   ```ts
   export const Route = createFileRoute('/dashboard')({
     beforeLoad: async () => {
       const user = await getServerSession()   // server fn — works on both SSR and client nav
       if (!user) throw redirect({ to: '/login', search: { redirect: '/dashboard' } })
       return { user }
     },
     component: Dashboard,   // blank authed page
   })
   ```
   On the initial SSR request an unauthenticated visitor never renders
   `/dashboard`; on client navigation the server function still runs server-side.
   `/login` reads `search.redirect` and navigates there after a successful sign-in.
5. **Dormant behavior:** when `VITE_API_URL` / the api is unset,
   `getServerSession` returns `null` (guard redirects to `/login`; the login
   form shows "auth not configured") — nothing throws, CI stays green.

**Gate (local; you cannot exercise cross-*subdomain* cookies on localhost, so
verify the mechanics same-origin first):** unauthenticated `/dashboard` 302s to
`/login?redirect=/dashboard` (observe the server response, not a client flash);
logging in lands a session cookie (visible in DevTools) and redirects to
`/dashboard`, which renders; reloading `/dashboard` stays (SSR guard passes via
the forwarded cookie); sign-out clears the cookie and `/dashboard` redirects
again; no signup UI exists and the sign-up route returns 400
(`EMAIL_PASSWORD_SIGN_UP_DISABLED`).

## Step 6 — CORS + cookies, fully env-driven

Consolidate the cross-service config so one build runs on both
`*.up.railway.app` (pre-DNS) and `*.yourdomain.com` (post-DNS) by env alone.

1. **NestJS CORS (`main.ts`):**
   ```ts
   app.enableCors({
     origin: env.CORS_ORIGIN.split(','),   // web origin(s) — NEVER '*' with credentials
     credentials: true,                     // required for cookies
     methods: ['GET', 'POST', 'OPTIONS'],
     allowedHeaders: ['content-type'],
   })
   ```
2. **Env matrix** (document all of these in `.env.example`):

   | Var | Local | Railway pre-DNS | Custom domain |
   |---|---|---|---|
   | api `BETTER_AUTH_URL` / `BASE_URL` | http://localhost:3000 | https://api-xxx.up.railway.app | https://api.yourdomain.com |
   | `TRUSTED_ORIGINS` | http://localhost:3001 | https://web-xxx.up.railway.app | https://app.yourdomain.com |
   | `CORS_ORIGIN` | http://localhost:3001 | https://web-xxx.up.railway.app | https://app.yourdomain.com |
   | `COOKIE_DOMAIN` | *(empty)* | *(empty)* | .yourdomain.com |
   | web `VITE_API_URL` | http://localhost:3000 | https://api-xxx.up.railway.app | https://api.yourdomain.com |
   | web `API_INTERNAL_URL` | =VITE_API_URL | api host (public or private) | =VITE_API_URL |

3. **Honest limitation — write this into `docs/PLAN.md` so it isn't chased as a
   bug:** `up.railway.app` is on the **Public Suffix List**, so `web-xxx.` and
   `api-xxx.up.railway.app` are **different sites**, not just different origins.
   Two consequences pre-DNS: (a) a browser refuses a `Domain=.up.railway.app`
   cookie, so `COOKIE_DOMAIN` must be **empty** and Better Auth sets a host-only
   cookie on the api origin; (b) that cookie is `SameSite=Lax`, and Lax cookies
   are **not sent on cross-site `fetch`** — only on top-level navigations. So a
   `fetch` from the web SPA to the api does **not** carry the cookie, and the SSR
   server function forwarding the web-host cookie has nothing to forward.
   **Net: no browser-based auth works pre-DNS — neither the client path nor the
   SSR guard.** Do NOT reach for `SameSite=None` to force it: it only half-works
   under 2026 third-party-cookie rules and weakens the prod posture if left on.
   Pre-DNS, verify auth by **curl against the api origin only** (Step 8); full
   browser auth is a **custom-domain guarantee** that lights up in Step 9 once
   both apps share `.yourdomain.com` and the cookie becomes same-site. Do not
   block the deploy on browser auth pre-DNS.

**Gate:** locally flip the matrix values and confirm the app reads them (log the
resolved config at boot behind a debug flag). `enableCors` rejects an untrusted
origin (a curl with a bogus `Origin` gets no `Access-Control-Allow-Origin`) and
allows the configured one with `Access-Control-Allow-Credentials: true`.

## Step 7 — CI (GitHub Actions, zero secrets)

1. `.github/workflows/ci.yml` — one workflow on push/PR to `main`: checkout →
   `pnpm/action-setup@v4` → `setup-node@v4` (node 22, `cache: pnpm`) → `pnpm
   install --frozen-lockfile` → `pnpm lint` → `pnpm test` → `pnpm build` (turbo
   builds both apps). See [parts/cicd.md](../parts/cicd.md).
2. **Must pass with ZERO secrets** — the forcing function for
   dormant-when-unconfigured. `--frozen-lockfile` in a clean CI store is what
   actually surfaces the `allowBuilds` gotcha the local store hides.
3. `pnpm test` includes: the identity-port contract test (fake vs real behave
   the same), the auth-mount test (a login response carries `Set-Cookie` through
   the `bodyParser:false` + `toNodeHandler`-before-`json` split), and the
   `DomainError`→HTTP-status mapping test.

**Gate:** open a PR; CI goes green end-to-end with no repo secrets configured.
If `ERR_PNPM_IGNORED_BUILDS` fires here, fix `allowBuilds` and re-push — this is
exactly the failure a clean store surfaces that local installs mask.

## Step 8 — Railway provisioning + first deploy (pre-DNS, `*.up.railway.app`)

**Strategy — Dockerfile-per-app, not Nixpacks root-dir.** Nixpacks auto-detect
on a Turborepo root is ambiguous (which app? which start command?). A
`Dockerfile` per app makes each service's build explicit and reproducible and
keeps `drizzle-kit` out of the production api image. Each does a workspace-aware
`pnpm install --frozen-lockfile` then `pnpm --filter <app> build`, and runs only
that app. Config-as-code via a `railway.json` per service — and the api's
carries the **Pre-Deploy Command** (migrations) so it's reviewable, not a manual
dashboard step:
```jsonc
// apps/api/railway.json
{ "build": { "builder": "DOCKERFILE", "dockerfilePath": "apps/api/Dockerfile" },
  "deploy": { "preDeployCommand": "node dist/db/migrate.js && node dist/scripts/create-user.js" } }
```
Pre-Deploy runs between build and deploy in a separate container with the
service's env vars, so the idempotent migrate + seed chain works there
(migrations never on app boot — that races replicas and buries failures in
crash-loop logs;
[railway-postgres-migrations](../gotchas/railway-postgres-migrations.md)). Use
the Railway MCP for provisioning + variables.

**Ordering matters — connecting a repo triggers an immediate build, so set every
build-time var BEFORE the first build:**
1. `create_project`.
2. Provision **Postgres** (private-network only).
3. `create_service` **api** and **web** from the GitHub repo (Dockerfile per
   app). Note: this may kick off an immediate first build — the web one is
   expected to be broken until `VITE_API_URL` is set (step 6), which is fine.
4. `generate_domain` on both api and web (Railway `*.up.railway.app`) — domains
   don't require a successful deploy, so do this now to learn the URLs.
5. `add_reference_variable` to inject `DATABASE_URL` into api from the Postgres
   service (cross-service reference — internal URL, not the public proxy).
6. `set_variables` with the now-known URLs:
   - api: `BETTER_AUTH_SECRET` (generate), `BETTER_AUTH_URL`/`BASE_URL` (api
     domain), `TRUSTED_ORIGINS` + `CORS_ORIGIN` (web domain), `COOKIE_DOMAIN`
     **empty**, `NODE_ENV=production`, `AUTH_SEED_EMAIL`, `AUTH_SEED_PASSWORD`.
   - web: `VITE_API_URL` (api domain — **build-time**; the web Dockerfile must
     `ARG VITE_API_URL` in the stage that runs `pnpm build`, else it's not
     inlined), `API_INTERNAL_URL`.
7. **Deploy — force a fresh build of web** so the new `VITE_API_URL` is inlined
   (a var change stages a redeploy; confirm it rebuilds the image, not just
   restarts). Deploy api too; its Pre-Deploy runs migrate + seed.

**Gate:** `list_deployments` shows both services SUCCESS; `get_logs` on the
**deploy** log (not build log) of api shows the migrate + create-user stdout
before the start command; `curl https://api-xxx.up.railway.app/health` OK and
`.../api/auth/get-session` returns 200 with a null body (anon). **Verify auth by
curl against the api origin only** (pre-DNS, browser auth cannot work per Step 6):
`POST .../api/auth/sign-in/email` with the seeded creds returns `Set-Cookie`;
replay that cookie on `GET .../api/auth/get-session` and it returns the user.
`https://web-xxx.up.railway.app/` = "hello world"; `/dashboard` redirects to
`/login`. Do not expect browser login to work until Step 9.

## Step 9 — Custom domain + cookie cutover (Cloudflare)

1. Attach custom domains: `generate_domain` `api.yourdomain.com` on api and
   `app.yourdomain.com` on web. Railway returns the CNAME target records.
2. Create the CNAMEs in Cloudflare (via API token or dashboard) as **DNS-only
   (grey cloud), not proxied** — Railway must terminate TLS and see the real
   host; Cloudflare's proxy would break Railway's ACME issuance and add another
   cookie/host layer. (Proxying is a separate later hardening task.)
3. `domain_status` until both custom domains show a certificate issued (Railway
   auto-issues TLS).
4. **Flip the env matrix to custom-domain values and redeploy:**
   - api: `BASE_URL` / `BETTER_AUTH_URL` = `https://api.yourdomain.com`,
     `TRUSTED_ORIGINS` = `https://app.yourdomain.com`, `CORS_ORIGIN` =
     `https://app.yourdomain.com`, **`COOKIE_DOMAIN=.yourdomain.com`**.
   - web: `VITE_API_URL` = `https://api.yourdomain.com` (**rebuild** — it's
     build-time), `API_INTERNAL_URL` = `https://api.yourdomain.com`.
5. Redeploy web (rebuild for the new inlined `VITE_API_URL`) and api. Better
   Auth now sets `Domain=.yourdomain.com; SameSite=Lax; Secure; HttpOnly`, so
   the cookie is visible to both `app.` and `api.` — the SSR guard's
   forwarded-cookie fetch fully works and cross-subdomain session sharing lights
   up.

**Gate:** `https://app.yourdomain.com/` = "hello world", TLS valid on both
subdomains; login shows the session cookie with
`Domain=.yourdomain.com; Secure; HttpOnly; SameSite=Lax`; a hard reload of
`/dashboard` (fresh SSR request) stays authenticated; `/dashboard` in a private
window 302s to `/login`; the api answers the `app.yourdomain.com` origin with
`Access-Control-Allow-Credentials: true` and a foreign origin with none.

## Step 10 — End-to-end verification + docs finalization

1. Full happy path on the custom domain: unauth `/dashboard`→redirect; login as
   the seeded user; land `/dashboard`; reload persists; sign-out; redirect
   again.
2. Negative paths: wrong password → a human-readable form error (not a stack
   trace); the sign-up route → 400 `EMAIL_PASSWORD_SIGN_UP_DISABLED` (login-only
   holds in prod).
3. Confirm migrations ran via Pre-Deploy on the last deploy (deploy log).
4. Confirm CI is still green with zero secrets.
5. Finalize `docs/OVERVIEW.md` (topology: app/api subdomains, cookie domain,
   CORS, Postgres private net), `docs/SPEC.md`, `docs/PLAN.md` (this build order
   + the Railway/pnpm gotchas + the pre-DNS cookie note), `docs/STYLE.md` (Paper
   & Ink), and root + per-feature `CLAUDE.md`.

**Final Gate:** an end-to-end run on `https://app.yourdomain.com` passes
login→dashboard→reload→logout; unauthenticated `get-session` returns 200 with a
null body; sign-up returns 400 `EMAIL_PASSWORD_SIGN_UP_DISABLED`; CI green; both
Railway services healthy in `list_deployments`.

## Tricky-seam quick reference

1. **Better Auth in Nest** — Express; `NestFactory.create(AppModule, {
   bodyParser: false })`, mount `toNodeHandler(auth)` on
   `server.all('/api/auth/{*splat}', …)` **before** re-applying `express.json()`
   to non-auth paths (json-before-handler hangs the client); `toNodeHandler`
   preserves multiple Set-Cookie; kept behind the identity port; session read via
   `auth.api.getSession({ headers: fromNodeHeaders(...) })`. **Seed = admin
   plugin `auth.api.createUser`** (`disableSignUp` blocks signUp even
   server-side).
2. **Cross-service auth in Start** — browser
   `createAuthClient({ baseURL: VITE_API_URL, fetchOptions: { credentials:'include' } })`;
   SSR guard = `beforeLoad` awaiting a `createServerFn` handler that calls
   `getRequest()` (not `getWebRequest`) and forwards the inbound `Cookie` to
   `${API_INTERNAL_URL}/api/auth/get-session`; unauth →
   `throw redirect({ to: '/login' })`. The server-fn wrapper is mandatory —
   `beforeLoad` runs client-side on SPA nav.
3. **CORS + cookies** — Nest `enableCors({ origin: CORS_ORIGIN.split(','),
   credentials: true })`; Better Auth `trustedOrigins` +
   `advanced.crossSubDomainCookies` + `defaultCookieAttributes`; all env-driven.
   **PSL honesty:** `*.up.railway.app` is on the Public Suffix List, so the two
   Railway subdomains are cross-*site* — no shared cookie AND `SameSite=Lax` isn't
   sent on cross-site fetch, so **no browser auth pre-DNS**; verify by curl, full
   browser auth only on the custom domain.
4. **Railway monorepo** — Dockerfile-per-app, explicit `pnpm --filter`
   build/run, `railway.json` per service (api's holds `deploy.preDeployCommand`);
   `VITE_API_URL` is **build-time** (`ARG` in the build stage, set before the
   build), `API_INTERNAL_URL` is runtime.
5. **Migrations** — `drizzle-kit generate` at dev/CI time; a **plain runtime
   migrator** (no drizzle-kit in the prod image) run as the api **Pre-Deploy
   Command** via `railway.json`; verify in the deploy log.
6. **Disable signup** — `emailAndPassword.disableSignUp: true` blocks signUp on
   **both** the HTTP route (400 `EMAIL_PASSWORD_SIGN_UP_DISABLED`) and
   server-side; the seed CLI creates the single user via the **admin plugin's
   `auth.api.createUser`**, idempotently, from `AUTH_SEED_EMAIL`/`AUTH_SEED_PASSWORD`.

## What the repo looks like when done

```
your-app/
├── package.json                       # root; packageManager pnpm@11; turbo scripts
├── pnpm-workspace.yaml                 # apps/*, packages/*  +  allowBuilds:
├── turbo.json
├── tsconfig.base.json
├── .env.example                        # every env var, documented (see Step 6 matrix)
├── docker-compose.yml                  # disposable local postgres:16-alpine
├── .github/workflows/ci.yml            # install → lint → test → build, zero secrets
├── docs/{OVERVIEW,SPEC,PLAN,STYLE}.md
├── CLAUDE.md
├── packages/
│   ├── shared/                         # DomainError union + SessionUser contract
│   └── config-ts/                      # shared tsconfig + eslint presets
└── apps/
    ├── api/                            # NestJS (Express)
    │   ├── Dockerfile
    │   ├── railway.json
    │   ├── drizzle.config.ts           # drizzle-kit generate — dev/CI only
    │   ├── nest-cli.json
    │   ├── CLAUDE.md
    │   └── src/
    │       ├── main.ts                 # bodyParser:false; mountAuth() before json; CORS
    │       ├── app.module.ts
    │       ├── config/env.ts           # typed reader + isAuthConfigured/isDbConfigured
    │       ├── common/domain-error.filter.ts   # error union → HTTP, once
    │       ├── db/{client,schema,migrate}.ts   # cached pool; generated BA schema; runtime migrator
    │       ├── scripts/create-user.ts  # pnpm auth:create-user — admin plugin createUser
    │       └── features/identity/      # the auth port + Better Auth adapter
    │           ├── types.ts
    │           ├── auth.ts             # betterAuth(...) + admin() — dormant when unconfigured
    │           ├── identity.service.ts # getCurrentUser — "who is asking"
    │           ├── mount-auth.ts       # mountAuth(server): toNodeHandler(auth), Set-Cookie handled
    │           ├── identity.module.ts
    │           ├── index.ts
    │           └── CLAUDE.md
    └── web/                            # TanStack Start (SSR)
        ├── Dockerfile                  # ARG VITE_API_URL (build-time); start: node .output/server/index.mjs
        ├── railway.json
        ├── vite.config.ts              # tanstackStart() + @tailwindcss/vite + Nitro
        ├── CLAUDE.md
        └── src/
            ├── router.tsx
            ├── env.ts                  # VITE_API_URL / API_INTERNAL_URL split
            ├── routes/
            │   ├── __root.tsx          # root route head(); pre-paint theme script first in <head>
            │   ├── index.tsx           # "/" → hello world
            │   ├── login.tsx           # no signup link
            │   └── dashboard.tsx       # beforeLoad guard
            ├── components/theme-toggle.tsx
            ├── styles/{index,tokens,base,typography}.css   # Paper & Ink, verbatim
            └── features/auth/
                ├── client.ts           # createAuthClient (browser)
                ├── session.server.ts   # createServerFn cookie-forwarding session check
                ├── types.ts
                ├── index.ts
                └── CLAUDE.md
```

---

This is **v0** — every seam was verified against current official docs during
authoring (Better Auth's `toNodeHandler` + `disableSignUp`/admin-plugin
behavior, TanStack Start's `vite.config.ts`/`createServerFn`/`getRequest`,
Turborepo `tasks`, the `up.railway.app` PSL entry, Railway's build-time-ARG and
`preDeployCommand` mechanics), but nothing here has been ported from a live
production app and the recipe has not been run start-to-finish. Those two deps
(Better Auth, TanStack Start) still move fast enough that the version-sensitivity
notice is not optional — re-check before writing the auth code. Treat the first
real build from this recipe as the actual admission test, the same way this
repo's other recipes were hardened: scaffold a throwaway app from it, log every
friction point as it happens, then fold what's learned back into these steps and
their gates. Only after that round-trip should this lose the v0 label.
