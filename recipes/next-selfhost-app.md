# Recipe: next-selfhost-app

Guide to building a self-hostable Next.js app with zero required vendor
accounts: SQLite as the database (a file, not a hosted service), local disk
for uploads/generated files, an optional single-user login, and
bring-your-own-key for AI features. After following this, the repo runs fully
offline with `pnpm dev` and no `.env.local` at all, and deploys unchanged to
Railway or Fly.io by mounting one persistent volume.

**What this recipe composes:**
- [next-app skill](../skills/next-app/SKILL.md) — Step 1 only. Run the skill
  itself rather than re-deriving its build order here: shell, Paper & Ink
  styles, docs viewer, feature seams, CI.
- Steps 2–7 below are new instruction-level content specific to this recipe.
  No standalone `parts/*.md` exists yet for the SQLite/Drizzle data layer,
  the storage seam, the single-user auth gate, BYOK, or the Docker/volume
  convention — that extraction should wait until this recipe has been run
  for real once, per this repo's authoring contract. Until then it lives
  here directly, not split out prematurely.

The auth gate in Step 4 is deliberately **not** Supabase-based — unlike this
repo's `add-simple-auth`/`add-user-auth` skills, it has zero vendor
dependency and zero DB dependency. Don't assume it reuses those skills' seam.

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

## Step 2 — Data layer (SQLite + Drizzle)

1. Add `drizzle-orm` and `drizzle-kit` to `package.json`. Use **`node:sqlite`**
   (Node's built-in module) as the driver, via `drizzle-orm/node-sqlite` —
   zero npm install for the driver itself, zero native build step, so there's
   no pnpm `allowBuilds` wrangling for the DB layer at all. Pin
   `"engines": { "node": ">=22.5.0" }`. Note the tradeoff plainly in the
   repo's own README: `node:sqlite` is pre-1.0 ("Release Candidate" as of
   Node v25.7) — Node 22.x may still need the `--experimental-sqlite` CLI
   flag; Node 24 LTS or newer runs it without one. If a specific app needs
   `better-sqlite3`'s more mature ecosystem tooling, or `@libsql/client`'s
   optional exit ramp to hosted Turso, that's a one-file change at the seam
   in the next item — not a reason to avoid starting here.
2. Create `src/lib/db.ts` as the single swap seam: lazily initializes one
   `db` instance, reads the database file path from `process.env.DATABASE_PATH`
   (default `./data/app.db` for local dev — create the parent directory if
   missing). This is the **only** file in the repo that imports a
   driver-specific `drizzle-orm/*` entry point; every other file imports `db`
   from here.
3. Create `src/lib/schema.ts` with an initial table suited to the app idea
   (or a placeholder single-column table if the app has no data model yet —
   never leave `schema.ts` empty; `drizzle-kit generate` needs at least one
   table to prove the pipeline).
4. Write `drizzle.config.ts` pointing at `schema.ts` and the same
   `DATABASE_PATH`. Add `pnpm db:generate` (`drizzle-kit generate`) and
   `pnpm db:migrate` — a plain Node script that applies the generated SQL
   files directly via the driver, so the production image never needs
   `drizzle-kit` installed at runtime.
5. Gitignore `data/` but keep a tracked `.gitkeep`, so `pnpm dev` never needs
   a manual `mkdir` on first run.
6. Run `pnpm db:generate && pnpm db:migrate` and confirm `data/app.db` is
   created; insert one throwaway row via a scratch script and confirm it
   reads back — proves the seam end-to-end before any feature code depends
   on it.

## Step 3 — Local file storage seam

1. Create a `storage` feature folder under `src/features/` with a
   vendor-agnostic contract in `types.ts` — a `FileStore` interface (`put`,
   `get`, `delete`, `urlFor`) shaped deliberately like the `AuthClient`
   contract in `skills/add-simple-auth`: a stable seam, not a rewrite point.
2. Write `local-adapter.ts`: writes under `process.env.STORAGE_ROOT`
   (default `./data/uploads`); reject or normalize any key containing `..`
   or an absolute path — no path-traversal escape hatch. `urlFor(key)`
   returns an in-app route path (e.g. `/api/files/<key>`), never a raw
   filesystem path.
3. Write `src/app/api/files/[...key]/route.ts` — a GET handler that streams
   bytes back through `FileStore.get()`, never `fs.readFile` directly in the
   route, setting `Content-Type` from stored metadata.
4. Write `index.ts` exporting `getFileStore()` — a factory returning the
   local adapter today. Note in the feature's `CLAUDE.md`: a future S3/R2
   adapter is one new file plus a branch in this factory, not a rewrite —
   same discipline as the auth seam.
5. Ensure `STORAGE_ROOT`'s directory is created if missing at boot, same
   handling as the DB directory in Step 2.
6. Run `pnpm dev`; write a small file through the store and fetch it back
   via `/api/files/...` (scratch script or a throwaway test) and confirm
   bytes and `Content-Type` round-trip correctly.

## Step 4 — Single-user auth gate (optional — evaluate against the app idea first)

Decide from the target app's description whether it needs a login at all.
Skip this entire step for a public toy, game, or anonymous tool — don't
create any of these files. Apply it only when the app idea calls for a
single operator gate (e.g. an admin tool, a personal utility meant to be
reachable from anywhere but not open to the public).

1. Write `src/features/auth/session.ts` — Web Crypto only, zero `node:*`
   imports (must run in the Edge proxy/middleware runtime). Cookie value
   `${issuedAtMs}.${signatureHex}`: HMAC-SHA256 over the raw timestamp
   string, signed with a secret from `AUTH_SESSION_SECRET`. Verify by
   recomputing the signature and comparing with a manual constant-time hex
   compare (Web Crypto has no raw timing-safe compare outside
   `subtle.verify`), then checking `Date.now() - issuedAtMs < maxAgeMs` —
   expiry is re-checked at verify time against a fixed max-age (e.g. 30
   days), not embedded as a separate signed claim. Wrap the whole thing in
   try/catch so malformed or absent cookie values never throw, only return
   false.
2. Write `src/features/auth/credentials.ts` — marked `import 'server-only'`
   and deliberately **not** re-exported from any barrel `index.ts`, so
   `node:crypto` never reaches the Edge bundle. `hashPassword()`: 16-byte
   random salt (hex), `node:crypto` scrypt with default cost params, 64-byte
   key length, returns `${salt}:${keyHex}`. `verifyCredentials(email,
   password)` reads `AUTH_EMAIL`/`AUTH_PASSWORD_HASH` from env and does a
   timing-safe compare on **both** fields, not just the password.
3. Write the login server action and `/login` page. On failure, return one
   generic error message ("Invalid email or password") regardless of
   whether the email or the password was wrong — never field-specific.
4. Write the route gate (`src/proxy.ts` on Next 16+, `src/middleware.ts` on
   earlier versions — check which convention the scaffolded Next version
   uses). Matcher excludes only Next internals (`_next/static`,
   `_next/image`, `favicon.ico`) — **never** `/login` itself. This is a
   named, previously-hit gotcha: the already-authenticated →
   redirect-away-from-`/login` branch lives inside the function body and
   needs `/login` requests to actually reach it; a matcher that excludes
   `/login` at the routing layer breaks that redirect silently, with no
   error — it just never fires.
5. Write `scripts/hash-password.mjs`, exposed as `pnpm auth:hash
   '<password>'`, printing the `salt:hex` string to paste into `.env.local`
   as `AUTH_PASSWORD_HASH`. Document generating `AUTH_SESSION_SECRET` via
   `openssl rand -hex 32`.
6. Write the test suite: round-trip (create → verify true), tamper (flip one
   signature character → false), expiry (a correctly-signed but stale
   timestamp → false, proving expiry is checked independent of signature
   validity), and malformed/absent input never throws.
7. Run `pnpm auth:hash '<throwaway-password>'`, paste it plus a generated
   `AUTH_SESSION_SECRET` into `.env.local`, run `pnpm dev`, and confirm: an
   unauthenticated request to `/` redirects to `/login`; correct login
   redirects back and sets a persistent cookie; an authenticated request to
   `/login` redirects to `/`; `pnpm test` is green.

## Step 5 — Bring-your-own-key for AI features

1. If the app idea includes an AI feature, use the provider's own
   conventional env var name (e.g. `GEMINI_API_KEY`, `OPENAI_API_KEY`,
   `ANTHROPIC_API_KEY`) — the same name the vendor's own SDK auto-detects,
   so no extra plumbing is needed.
2. Document it in `.env.example` with a one-line comment on where to obtain
   a key. Never persist the key to the database; never proxy it through a
   billing layer — every request uses the operator's own key directly.
3. Read the key only in server-only code (route handlers / server actions) —
   same `server-only` discipline as `credentials.ts` in Step 4.
4. Run `pnpm dev` with the var unset and confirm the feature declines
   gracefully with a clear "add your API key to `.env.local`" message —
   never a raw SDK stack trace.

## Step 6 — Docker + volume-mount convention (local / Railway / Fly)

1. Set `output: "standalone"` in `next.config.ts`.
2. Write a 3-stage `Dockerfile` (deps → builder → runner). The runner stage
   copies only `.next/standalone`, `.next/static`, and `public/`; runs as
   the non-root `node` user; `CMD ["node", "server.js"]`.
3. Confirm every DB and storage path flows through `DATABASE_PATH`/
   `STORAGE_ROOT` (already true from Steps 2–3) — never a hardcoded
   `./data/...` outside those two reads, since the identical mount path
   (e.g. `/data`) must work unchanged across local Docker, Railway, and Fly.
4. Add `.dockerignore` (`node_modules`, `.next`, `data/`, `.env*`).
5. Verify locally: `docker build -t <app> .`, then `docker run -p 3000:3000
   -e DATABASE_PATH=/data/app.db -e STORAGE_ROOT=/data/uploads -v
   $(pwd)/data:/data <app>` — confirm the app serves and data persists
   across a container restart.
6. Document the Railway path: one volume mounted at `/data` (Railway allows
   exactly one volume per service, and it is not mounted during the build —
   only at deploy/start time). Run migrations via Railway's
   `preDeployCommand`, not inside the app's own boot sequence and never at
   build time. Set `RAILWAY_RUN_UID=0` if the non-root container needs
   volume write access.
7. Document the Fly path: `fly.toml` with a `[[mounts]]` block (`source =
   "data"`, `destination = "/data"`). Named gotcha: a Fly volume is a
   physical slice on one host in one region — not network-attached — so a
   new machine or region needs its own empty volume. This is fine for the
   single-instance case this recipe targets, but it is not silently
   highly-available; Fly's own docs recommend two-plus volumes for HA.
   Litestream (continuous SQLite backup to object storage) and LiteFS
   (distributed SQLite) are optional hardening for later, not required
   here.
8. Confirm identical behavior across all three: local `docker run`,
   `railway up`, `fly deploy` — same image, same env-var-driven paths, zero
   platform-specific code branches in the app itself.

## Step 7 — Polish and verify

1. Write/update `CLAUDE.md` — current phase ("self-hostable shell
   complete"), what's in place (shell, data, storage, auth if applied, BYOK
   if applied, deploy config), what to read first.
2. Write `README.md` with two literal handoff lines: local dev is "clone it,
   `pnpm dev`, done"; deploy is "`railway up` when you want it public" (or
   the Fly equivalent) — plus an env-var table (`DATABASE_PATH`,
   `STORAGE_ROOT`, and, only if applicable, `AUTH_EMAIL`/
   `AUTH_PASSWORD_HASH`/`AUTH_SESSION_SECRET` and the BYOK var).
3. Run `pnpm build`, `pnpm test`, `pnpm lint` — all green with zero env vars
   set, proving the zero-account local path genuinely works standalone.
4. Fresh-clone check: `pnpm install`, `pnpm dev`, exercise the app with no
   `.env.local` at all — confirm nothing crashes; only auth/BYOK features
   (if present) decline gracefully.
5. Push to GitHub; run `railway up` (or `fly deploy`) once and confirm the
   Docker image builds and serves through the mounted volume in the real
   environment.

## What the repo looks like when done

```
your-app/
├── CLAUDE.md                        # phase state, what's in place
├── README.md                        # clone-it / pnpm-dev / railway-up handoff
├── .env.example                     # DATABASE_PATH, STORAGE_ROOT, auth + BYOK vars
├── .dockerignore
├── Dockerfile                       # 3-stage: deps → builder → runner
├── fly.toml                         # [[mounts]] destination = "/data"
├── drizzle.config.ts
├── next.config.ts                   # output: "standalone"
├── docs/
│   ├── OVERVIEW.md
│   ├── PLAN.md
│   ├── SPEC.md
│   └── STYLE.md
├── knowledge/
│   ├── <guidance>.md
│   └── <rubric>.md
├── data/                            # gitignored; holds app.db + uploads/ locally
│   └── .gitkeep
├── scripts/
│   ├── hash-password.mjs            # only if Step 4 applied — pnpm auth:hash
│   └── migrate.mjs                  # applies drizzle-generated SQL, no CLI at runtime
├── src/
│   ├── app-meta.ts
│   ├── proxy.ts                     # only if Step 4 applied — route gate
│   ├── lib/
│   │   ├── db.ts                    # the ONLY file importing drizzle-orm/node-sqlite
│   │   └── schema.ts
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── login/page.tsx           # only if Step 4 applied
│   │   ├── docs/page.tsx
│   │   └── api/
│   │       └── files/[...key]/route.ts
│   ├── components/
│   │   ├── theme-init.tsx
│   │   ├── theme-toggle.tsx
│   │   └── docs-viewer.tsx
│   ├── features/
│   │   ├── core/CLAUDE.md
│   │   ├── generate/CLAUDE.md       # AI feature, reads BYOK env var
│   │   ├── verify/CLAUDE.md
│   │   ├── knowledge/CLAUDE.md
│   │   ├── prefs/CLAUDE.md
│   │   ├── storage/
│   │   │   ├── CLAUDE.md
│   │   │   ├── types.ts             # FileStore contract
│   │   │   ├── local-adapter.ts
│   │   │   └── index.ts             # getFileStore() factory
│   │   └── auth/                    # only if Step 4 applied
│   │       ├── CLAUDE.md
│   │       ├── session.ts           # Edge-safe, Web Crypto only
│   │       ├── credentials.ts       # server-only, node:crypto scrypt
│   │       ├── session.test.ts
│   │       └── credentials.test.ts
│   └── styles/
│       ├── index.css
│       ├── tokens.css
│       ├── base.css
│       └── typography.css
├── .github/workflows/ci.yml
├── package.json
├── postcss.config.mjs
├── eslint.config.mjs
├── tsconfig.json
├── vitest.config.ts
└── .gitignore
```

This is v1 — the SQLite driver choice, storage seam, and auth pattern are
each individually proven (the auth gate is reused byte-for-byte in approach
from a live production app; `node:sqlite`, Railway volumes, and Fly volumes
are each confirmed against current official docs), but the recipe as a whole
has not yet been run start-to-finish as its own fresh consumer. Treat that
as the next real test, the same way this repo's other v1 skills were
hardened: build a throwaway app from it, log the friction, then fold what's
learned back in.
