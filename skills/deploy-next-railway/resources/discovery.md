# Discovery

This skill deploys an existing app — it does not scaffold one. Before
touching anything, read the repo *and* probe the Railway account/project, and
fill in the judgment calls every later resource depends on. Assume the stack
(Next.js App Router + pnpm, Postgres via `postgres.js`, optionally a
Railway-bucket storage feature); never assume the Railway-side state.

Default to `add-user-auth`'s tone, not `add-simple-auth`'s: **infer
everything you can, only ask when something is genuinely ambiguous or
irreversible-ish.** Run the probes below silently, report the summary, and
only stop for a question if a probe surfaces a real judgment call (an
ambiguous existing service, an already-set Pre-Deploy Command, no linked
project at all).

## Preflight — confirm the precondition, or stop

This skill's one hard precondition: **a migration runner already exists.**
Check for:

- `migrations/*.sql` (or equivalent) at the repo root
- A script that applies them with no interactive-CLI dependency at runtime
  (e.g. `scripts/migrate.mjs`, exposed as `pnpm db:migrate` in
  `package.json`)

If neither exists, stop and tell the user this skill assumes the migration
layer `recipes/next-railway-app.md` Step 2 produces — it does not write one.
Do not improvise a migration script to fill the gap.

Also confirm the basic shape (`next` + `src/app/`, `pnpm-lock.yaml`) — if
this isn't a Next.js App Router + pnpm repo, stop; this skill doesn't target
anything else.

## Concrete probes — the repo

- **`package.json`** — the `db:migrate`-equivalent script name (you'll
  reference its exact command in Step 6's Pre-Deploy Command); the `build`
  and `start` scripts (Railway's Nixpacks builder needs these, not a custom
  Dockerfile — confirm they're the plain `next build` / `next start` Railway
  auto-detects).
- **`.env.example`** — the authoritative list of vars the app expects. Note
  every var name present: at minimum `DATABASE_URL`; if auth is in scope,
  `AUTH_SESSION_SECRET`; if storage is in scope, `BUCKET_ENDPOINT` /
  `BUCKET_ACCESS_KEY_ID` / `BUCKET_SECRET_ACCESS_KEY` / `BUCKET_NAME`. This
  is the checklist Step 5 wires against.
- **Does the app actually use Postgres?** — `postgres` in `package.json`
  dependencies, or `DATABASE_URL` referenced in source. Almost always yes for
  this recipe's shape, but don't provision a service the app doesn't read.
- **Does the app actually use bucket storage?** — `@aws-sdk/client-s3` in
  dependencies, or `BUCKET_*` vars referenced in source. This is genuinely
  optional — some deployments are Postgres-only with no uploads feature. If
  absent, skip Step 4 and the bucket half of Step 5 entirely; don't provision
  a bucket nobody reads from.
- **`pnpm-workspace.yaml`** — does an `allowBuilds` key exist and look
  current (matches the native deps actually in `package.json`, e.g. `sharp`
  if image processing is used)? Note this for Step 2 — don't assume a
  locally-clean `pnpm install` means this is fine; the local pnpm store
  caches prior approvals in a way a fresh Railway build environment won't
  benefit from.

## Concrete probes — the Railway side

Prefer MCP tools (`mcp__railway__list-projects`, `list-services`, `whoami`,
or a natural-language ask to `railway-agent`) over shelling out; fall back to
`railway status` / `railway service` CLI commands or asking the user to check
the dashboard if MCP tools aren't available in this session.

- **Is a Railway project already linked to this repo?** (`.railway/`
  project link locally, or ask `railway-agent`.) If not, this is a genuine
  ask — see Before You Start item 1.
- **What services exist on the target project/environment already?**
  (`list-services`.) Look specifically for: a Postgres plugin/service, a
  Storage Bucket service, and the app's own web service (may not exist yet
  if this is a first deploy).
- **If a Postgres-looking or bucket-looking service already exists**, is its
  name/shape unambiguous, or are there multiple candidates / an unclear
  match? Surface anything ambiguous — never guess which one the app should
  use.
- **Does the web service (if it exists) already have a
  `preDeployCommand` set?** Read it via `railway-agent`/MCP or the dashboard.
  If it's set to something other than what Step 6 would set, this is a real
  ask — never silently overwrite.
- **Does the web service already have any of the target env vars set**, and
  do their values look sane (e.g. is `DATABASE_URL` already a variable
  reference, or is it a raw public-proxy connection string that would be a
  bug to leave in place)? Note this so Step 5 fixes rather than duplicates.

## Judgment extractions

- **No Railway project linked, and none obviously intended.** Ask whether to
  create a new project or attach to an existing one (Before You Start item 1)
  — never silently create one.
- **Ambiguous existing Postgres/bucket service.** Surface exact service
  names found and ask which to use, or whether to provision a fresh one
  instead (Before You Start item 2).
- **Existing `preDeployCommand` with different content.** Surface the exact
  current value and ask before overwriting (Before You Start item 3).

## Output — the discovery summary

Before touching anything, report to the user (and keep for your own
reference through the rest of the build order):

```
Preflight: ✓ Next.js App Router + pnpm; migration runner found at scripts/migrate.mjs (pnpm db:migrate)
App uses: Postgres ✓   Bucket storage ✓ (or: — not used, skipping Steps 4/5b)
.env.example vars: DATABASE_URL, AUTH_SESSION_SECRET, BUCKET_ENDPOINT, BUCKET_ACCESS_KEY_ID, BUCKET_SECRET_ACCESS_KEY, BUCKET_NAME
Railway project: <name/id> (linked) — environment: production
Existing services: Postgres (unambiguous) — Bucket: none yet — web service: none yet (first deploy)
Existing preDeployCommand: none
pnpm-workspace.yaml allowBuilds: present and current / missing — will run `pnpm approve-builds --all` in Step 2
```

Every later step references this summary. If a probe surprised you (no
migration runner, an ambiguous service, an already-set Pre-Deploy Command,
no linked project), surface it now — do not silently proceed.
