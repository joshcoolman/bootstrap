# Recipe: next-railway-app

Guide to building a Next.js App Router app deployed to Railway: Postgres as
the database, a Railway Storage Bucket for uploads, and whitelist-only
multi-user email/password auth — no Supabase, no external auth vendor at
all. Proven end-to-end via a minimal to-do CRUD demo for logged-in users.
This is the shape a live production app (`bootsy`) actually runs in.

**What this recipe composes:**
- [next-app skill](../skills/next-app/SKILL.md) — Step 1 only. Run the skill
  itself rather than re-deriving its build order here: shell, Paper & Ink
  styles, docs viewer, feature seams, CI.
- Steps 2–6 below are new instruction-level content specific to this recipe.
  No standalone `parts/*.md` exists yet for the Postgres data layer, the
  whitelist auth gate, the bucket storage seam, or the Railway deploy
  convention — that extraction should wait until this recipe has been run
  for real once, per this repo's authoring contract. Until then it lives
  here directly, not split out prematurely.

This repo already has an `add-user-auth` skill — that one is **Supabase**-based
multi-user auth with Postgres RLS. The auth gate in Step 3 here is a
genuinely different, Railway-native alternative: zero vendor dependency
beyond Postgres itself, a hand-rolled signed-cookie session, and a
whitelist-only `users` table with no signup route. Don't assume this recipe
reuses `add-user-auth`'s seam, and don't assume this is worse — it's simply
a different tradeoff (no vendor account, no RLS, an operator-provisioned
whitelist instead of open signup).

---

## Step 1 — Shell (next-app skill)

1. Run `/bootstrap:next-app` in the target directory (or invoke the skill
   directly if not running inside Claude Code). Do not hand-scaffold
   Next.js — this recipe assumes the skill's output byte-for-byte.
2. Let the skill run its own six build steps to completion, including its
   own verification gates.
3. Confirm the skill's handoff: `pnpm build`, `pnpm test`, `pnpm lint` all
   green with zero env vars, `pnpm dev` serves the home page and `/docs`
   without errors. This is the checkpoint every later step builds on.

## Step 2 — Postgres data layer

1. Add `postgres` (the `postgres.js` package) to `package.json`.
2. Create a per-feature database client — each feature that needs one gets
   its own file (e.g. `src/features/core/db.ts`), not a single shared
   client, so features stay isolated except through their public `index.ts`.
   Anchor the client to `globalThis` so dev-mode Fast Refresh module
   re-evaluation doesn't spawn a fresh pool on every edit and exhaust
   `max_connections`:
   ```ts
   import postgres from 'postgres'

   const globalForSql = globalThis as typeof globalThis & {
     __appSql?: ReturnType<typeof postgres>
   }

   function getDatabaseUrl(): string {
     const url = process.env.DATABASE_URL
     if (!url) throw new Error('DATABASE_URL is not set')
     return url
   }

   export const sql = (globalForSql.__appSql ??= postgres(getDatabaseUrl(), {
     transform: postgres.camel,
   }))
   ```
   `transform: postgres.camel` auto-converts `snake_case` columns to
   camelCase JS properties and back — write schema columns snake_case,
   query results come back camelCase for free.
3. Create a flat `migrations/*.sql` folder at the repo root, numbered
   filename prefixes (`0001_init.sql`, `0002_users.sql`, ...). Write
   `scripts/migrate.mjs` (exposed as `pnpm db:migrate`) that:
   - loads `.env.local` if present (`process.loadEnvFile`),
   - creates a `schema_migrations(id text primary key, applied_at
     timestamptz default now())` tracking table if it doesn't exist,
   - applies any `migrations/*.sql` file not yet recorded, in filename-sort
     order, each inside its own transaction (`sql.begin`).
   This script must have **no interactive CLI dependency at runtime** — it's
   run by Railway's `preDeployCommand` (Step 6), which needs a plain Node
   script, not an ORM CLI installed in the production image.
4. Document `DATABASE_URL` in `.env.example`.
5. **Local dev needs its own Postgres, separate from Railway's.** See
   [gotchas/local-postgres-for-dev.md](../gotchas/local-postgres-for-dev.md)
   — spin up a disposable local Postgres container rather than pointing
   `DATABASE_URL` at Railway's instance (public proxy or otherwise). This
   is not optional polish: a live app built from this exact stack currently
   has local dev and production pointed at the **same** Postgres instance,
   an unintentional footgun this recipe should not repeat. Treat that gotcha
   doc as the fix, applied at scaffold time, not a later decision.
6. Gate: run `pnpm db:migrate` against the local container, confirm
   `schema_migrations` gets created and the first migration is recorded;
   insert one throwaway row via a scratch script and read it back.

## Step 3 — Multi-user whitelist auth (not Supabase)

1. Write `src/features/auth/session.ts` — **Web Crypto only, zero `node:*`
   imports** (this file must run on the Edge runtime that `proxy.ts`
   executes on). Cookie value shape: `${userId}.${issuedAtMs}.${signatureHex}`,
   HMAC-SHA256 over `${userId}.${issuedAtMs}` signed with a secret from
   `AUTH_SESSION_SECRET`. Verify by recomputing the signature and comparing
   with a manual constant-time hex compare (Web Crypto has no raw
   timing-safe compare outside `subtle.verify`), then checking
   `Date.now() - issuedAtMs < maxAgeMs` (e.g. 30 days) — expiry is
   re-checked at verify time, not embedded as a separate signed claim. Wrap
   the whole thing so malformed or absent cookie values return `null`,
   never throw.
2. Write `src/features/auth/credentials.ts` — marked `import 'server-only'`
   and **never re-exported from the feature's `index.ts` barrel**, so
   `node:crypto` never reaches the Edge bundle `proxy.ts` pulls from.
   `hashPassword()`: 16-byte random salt (hex), `node:crypto` scrypt with
   default cost params, 64-byte key length, stored as `${salt}:${keyHex}`.
   `verifyCredentials(email, password)` looks up the `users` table by
   email and, **when no row matches, still runs the scrypt compare against
   a fixed dummy hash** before returning failure — so a login attempt takes
   the same time whether or not the email is registered (a timing
   side-channel that would otherwise leak which emails are provisioned).
3. Write `src/features/auth/current-user.ts` — `getCurrentUserId()` reads
   the session cookie via `next/headers` and returns the authenticated
   user's id, throwing if there's no valid session. Every owner-scoped
   query in the app calls this to resolve "who is asking."
4. Write `src/features/auth/actions.ts` — `login` (server action via
   `useActionState`: verify credentials, set the cookie, `redirect('/')`)
   and `logout` (clear the cookie, `redirect('/login')`). On failure,
   return one generic error message ("Invalid email or password") — never
   field-specific, for the same enumeration-resistance reason as Step 2.
5. Write `src/features/auth/db.ts` — this feature's own database client
   (Step 2's pattern), separate from any other feature's.
6. Add the `users` migration: `id uuid primary key default
   gen_random_uuid()`, `email text unique not null`, `password_hash text
   not null`, `created_at timestamptz default now()`.
7. Write the route gate — `src/proxy.ts` on Next 16+ (the file `next dev`
   actually loads; check which convention the scaffolded Next version
   uses — Next 16 renamed `middleware.ts` to `proxy.ts` and the exported
   function to `proxy`, not `middleware`). Matcher excludes only Next
   internals (`_next/static`, `_next/image`, `favicon.ico`) — **never**
   `/login` itself. Named gotcha: the already-authenticated →
   redirect-away-from-`/login` branch lives inside the function body and
   needs `/login` requests to actually reach it; excluding `/login` at the
   matcher breaks that redirect silently, with no error.
8. Write `scripts/hash-lib.mjs` (the shared scrypt implementation) and
   `scripts/create-user.mjs`, exposed as `pnpm auth:create-user '<email>'
   '<password>'` — this is the **only** provisioning path (no signup
   route). Running it against an email that already exists should prompt
   to overwrite the password instead — this doubles as the password-reset
   mechanism. Keep the scrypt call shape in `hash-lib.mjs` identical to
   `credentials.ts`'s, or verification will silently fail.
9. Write the `/login` page and form (client component, `useActionState`).
10. Write the test suite: HMAC round-trip (create → verify true), tamper
    (flip one signature character → false), expiry (a correctly-signed but
    stale timestamp → false, proving expiry is checked independent of
    signature validity), malformed/absent cookie never throws, and the
    dummy-hash timing-safe path for unregistered emails.
11. Document `AUTH_SESSION_SECRET` in `.env.example` (generate via `openssl
    rand -hex 32`).
12. Gate: run `pnpm auth:create-user 'you@example.com' '<throwaway
    password>'`, run `pnpm dev`, confirm an unauthenticated request to `/`
    redirects to `/login`; correct login redirects back and sets a
    persistent cookie; an authenticated request to `/login` redirects to
    `/`; `pnpm test` is green.

## Step 4 — Object storage (Railway bucket)

1. Add `@aws-sdk/client-s3` and `@aws-sdk/s3-request-presigner` — Railway's
   Storage Bucket is S3-compatible.
2. Write `src/features/storage/client.ts` — a `globalThis`-cached
   `S3Client` (same HMR-safety pattern as Step 2's db client):
   ```ts
   new S3Client({
     region: 'auto',
     endpoint: process.env.BUCKET_ENDPOINT,
     credentials: {
       accessKeyId: process.env.BUCKET_ACCESS_KEY_ID!,
       secretAccessKey: process.env.BUCKET_SECRET_ACCESS_KEY!,
     },
   })
   ```
3. Write an upload path: a server action or route handler that takes an
   uploaded `File`/`Blob` and writes it via `PutObjectCommand` to
   `process.env.BUCKET_NAME` under a generated key (e.g. a `uuid` plus
   original extension). Keep this in `src/features/storage/upload.ts`, not
   inlined in a feature that consumes it.
4. Write `src/features/storage/display.ts` — `resolveDisplayUrl(storageKey)`
   returning a presigned GET URL (`getSignedUrl` + `GetObjectCommand`, a
   short TTL such as 1 hour). Presigning is a local, cheap operation — no
   network round trip to Railway needed to generate one.
5. Write `src/features/storage/index.ts` exporting the public surface
   (`uploadFile`, `resolveDisplayUrl`, `deleteAsset` via
   `DeleteObjectCommand`) — the only way other features touch storage.
6. Document `BUCKET_ENDPOINT`, `BUCKET_ACCESS_KEY_ID`,
   `BUCKET_SECRET_ACCESS_KEY`, `BUCKET_NAME` in `.env.example` — obtained
   locally via `railway bucket credentials --bucket <name> --json` once
   the bucket is provisioned (Step 6).
7. Gate: upload a file through the action/route (scratch script or a
   throwaway test), confirm the returned presigned URL fetches back the
   same bytes.

## Step 5 — Demo feature: todos CRUD

This is the "proof the auth + DB + (optionally) storage seams work
end-to-end" feature — the first thing a real project should delete and
replace with actual domain logic. Keep it deliberately minimal.

1. Add a `todos` migration: `id uuid primary key default
   gen_random_uuid()`, `owner_id uuid not null references users(id)`,
   `title text not null`, `done boolean not null default false`,
   `created_at timestamptz default now()`.
2. Create `src/features/todos/` — a `db.ts` (own client), and server
   actions for create/toggle/delete, **every query scoped by
   `getCurrentUserId()`** (`where owner_id = ${ownerId}`), never trusting a
   client-supplied owner id.
3. Write a simple `/todos` page: a list, an add form, toggle/delete
   controls. Minimal Paper & Ink styling — this page is a proof, not a
   product.
4. Gate: log in as two different users (created via `pnpm
   auth:create-user`), confirm each only ever sees their own todos — the
   isolation is the actual thing being proven here, not the UI.

## Step 6 — Railway deploy

1. No `Dockerfile` or `railway.json` is needed — Railway's Nixpacks
   builder auto-detects a Next.js app from `package.json`'s `build`
   (`next build`) and `start` (`next start`) scripts.
2. Provision a Postgres service and a Storage Bucket on the Railway
   project (dashboard, `railway add postgres`, or the Railway MCP
   `railway-agent` tool).
3. Wire env vars on the deployed service: `DATABASE_URL` as a **Railway
   variable reference** to the Postgres service's internal URL (not the
   public proxy URL — that's for local dev only, and only against the
   *separate* local Postgres from Step 2, never pointed at the same
   instance backing production), `AUTH_SESSION_SECRET`, and the four
   `BUCKET_*` vars from `railway bucket credentials`.
4. **Run migrations via the service's Pre-Deploy Command, not on app
   boot.** See
   [gotchas/railway-postgres-migrations.md](../gotchas/railway-postgres-migrations.md)
   — set `deploy.preDeployCommand` to `node scripts/migrate.mjs` (or `pnpm
   db:migrate`). Running migrations inside the app's own startup risks two
   replicas racing to migrate, and buries failures inside app-crash-loop
   logs instead of a clean deploy-log entry. As of Railway CLI v4.26
   there's no plain CLI flag for this — use the dashboard, the GraphQL
   API, or the `railway-agent` MCP tool.
5. Gate: `railway up` (or push to the connected GitHub repo), read the
   **deploy** log (not the build log) of the resulting deployment and
   confirm the migration script's own stdout appears before the start
   command's output. Visit the live URL, log in with a user provisioned
   via `pnpm auth:create-user`, create a todo, confirm it persists across a
   page reload.

## Step 7 — Polish and verify

1. Write/update `CLAUDE.md` — current phase ("Railway shell complete: auth,
   Postgres, storage, todos demo"), what's in place, what to read first.
2. Write `README.md` with an env-var table (`DATABASE_URL`,
   `AUTH_SESSION_SECRET`, `BUCKET_ENDPOINT`, `BUCKET_ACCESS_KEY_ID`,
   `BUCKET_SECRET_ACCESS_KEY`, `BUCKET_NAME`) and two literal handoff
   lines: local dev ("clone it, start a local Postgres container per the
   gotcha doc, `pnpm db:migrate`, `pnpm auth:create-user`, `pnpm dev`") and
   deploy ("`railway up` once Postgres + bucket are provisioned and env
   vars are set").
3. Run `pnpm build`, `pnpm test`, `pnpm lint` — all green.
4. Fresh-clone check: `pnpm install` into a clean checkout, confirm
   `pnpm dev` fails clearly (not a raw stack trace) with no `.env.local` —
   this app is not the zero-account case `next-selfhost-app.md` targets,
   so a clear "set DATABASE_URL" error is the correct outcome, not silent
   success.

## What the repo looks like when done

```
your-app/
├── CLAUDE.md                        # phase state, what's in place
├── README.md                        # env var table, local-dev / deploy handoff
├── .env.example                     # DATABASE_URL, AUTH_*, BUCKET_*
├── migrations/
│   ├── 0001_init.sql
│   ├── 0002_users.sql
│   └── 0003_todos.sql
├── scripts/
│   ├── migrate.mjs                  # pnpm db:migrate — no CLI dep at runtime
│   ├── hash-lib.mjs                 # shared scrypt implementation
│   └── create-user.mjs              # pnpm auth:create-user — only provisioning path
├── docs/
│   ├── OVERVIEW.md
│   ├── PLAN.md
│   ├── SPEC.md
│   └── STYLE.md
├── knowledge/
│   ├── guidance.md
│   └── rubric.md
├── src/
│   ├── app-meta.ts
│   ├── proxy.ts                     # route gate — matcher never excludes /login
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── login/page.tsx
│   │   ├── todos/page.tsx
│   │   └── docs/page.tsx
│   ├── components/
│   │   ├── theme-init.tsx
│   │   ├── theme-toggle.tsx
│   │   └── docs-viewer.tsx
│   ├── features/
│   │   ├── core/CLAUDE.md
│   │   ├── knowledge/CLAUDE.md
│   │   ├── prefs/CLAUDE.md
│   │   ├── auth/
│   │   │   ├── CLAUDE.md
│   │   │   ├── session.ts           # Edge-safe, Web Crypto only
│   │   │   ├── credentials.ts       # server-only, node:crypto scrypt
│   │   │   ├── current-user.ts
│   │   │   ├── actions.ts
│   │   │   ├── db.ts
│   │   │   ├── session.test.ts
│   │   │   └── credentials.test.ts
│   │   ├── storage/
│   │   │   ├── CLAUDE.md
│   │   │   ├── client.ts            # globalThis-cached S3Client
│   │   │   ├── upload.ts
│   │   │   ├── display.ts           # presigned GET URLs
│   │   │   └── index.ts
│   │   └── todos/
│   │       ├── CLAUDE.md
│   │       └── db.ts
│   └── styles/
│       ├── index.css
│       ├── tokens.css
│       ├── base.css
│       └── typography.css
├── .github/workflows/ci.yml
├── package.json
├── next.config.ts
├── postcss.config.mjs
├── eslint.config.mjs
├── tsconfig.json
├── vitest.config.ts
└── .gitignore
```

This is v1 — every individual piece is proven (the auth gate, Postgres
client pattern, and storage client are ported directly from a live
production app; the Railway migration and local-Postgres conventions are
each confirmed named gotchas from real deploys), but the recipe as a whole
has not yet been run start-to-finish as its own fresh consumer. Treat that
as the next real test, the same way this repo's other v1 recipes and
skills were hardened: build a throwaway app from it, log the friction, then
fold what's learned back in.
