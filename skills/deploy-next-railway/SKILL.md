---
name: deploy-next-railway
description: Deploy an existing Next.js App Router app (already scaffolded — e.g. via the next-app skill or the next-railway-app recipe) to Railway — provision Postgres and a Storage Bucket if not already present, wire env vars (internal DATABASE_URL reference, AUTH_SESSION_SECRET, BUCKET_* vars), set the Pre-Deploy Command for migrations, run a pnpm build-approvals preflight, trigger the deploy, and verify it live. Use when asked to deploy a Next.js app to Railway, ship it to production, or set up Railway hosting for an app that already has Postgres/auth/storage code in place.
---

You are about to deploy the current repo to Railway. This is a **deploy-only**
skill — it writes no application code. It assumes the target repo already has
a migration runner (`scripts/migrate.mjs` or equivalent, no interactive-CLI
dependency at runtime) and, if the app uses uploads, the bucket-consuming
storage feature — i.e. the shape `recipes/next-railway-app.md` Steps 2–4
produce, or a hand-built equivalent. If that precondition is missing, say so
and stop — do not improvise a migration script or a storage feature to fill
the gap; that's out of this skill's contract.

Prefer Railway MCP tools (`mcp__railway__*`, especially `railway-agent`, the
most capable entry point for multi-step operations) over raw `railway` CLI
commands for provisioning services and setting the Pre-Deploy Command — as of
Railway CLI v4.26 there is no plain CLI flag for Pre-Deploy Command at all, so
MCP or the dashboard are the only paths. Fall back to CLI + explicit
dashboard click-path instructions to the user only when MCP tools aren't
available in the session.

**Extracted directly from `recipes/next-railway-app.md` Step 6** (itself
sourced from a live production app, `bootsy`) rather than built fresh in a
disposable mule — a deliberate deviation from this repo's
usual mule-first loop. This is a v1 draft **one step further from proven**
than that precedent: `next-railway-app.md` Step 6 is validated only in its
individually-proven pieces (Postgres provisioning, bucket credentials,
Pre-Deploy Command wiring, deploy-log verification), not yet run as one
integrated end-to-end flow. Treat friction on a first real run — MCP tool
availability, Railway variable-reference syntax, `railway-agent` behavior —
as expected signal, not a sign these instructions are wrong.

## Before you start — check first, only ask if genuinely blocked

Run the discovery pass (`resources/discovery.md`) first — infer what's
inferable from the repo and the Railway account, only ask when something is
genuinely ambiguous or irreversible-ish:

1. **Which Railway project/environment** — if none is linked yet, ask whether
   to create a new project or attach to an existing one. Never silently
   create a new project.
2. **An existing Postgres or bucket service under an ambiguous name** — if
   discovery finds more than one candidate, or a name that doesn't obviously
   match the app, surface it and ask which to use rather than guessing or
   provisioning a duplicate.
3. **An existing Pre-Deploy Command set to something else** — surface the
   value verbatim and ask before overwriting it.
4. **A custom domain** — ask only if the user wants one wired now; default is
   Railway's generated `*.up.railway.app` URL, which is enough to verify live.

Everything else — exact var names, the internal-vs-public-URL choice for
`DATABASE_URL`, migration wiring mechanics — is fixed by convention below,
not asked.

## Resources — read these in order

- [Discovery](resources/discovery.md) — the Step-0 read-the-repo-and-the-Railway-project
  pass: preflight, probes, the judgment-call list, summary format
- [Provisioning](resources/provisioning.md) — creating or confirming Postgres
  and a Storage Bucket via `railway-agent`/MCP, obtaining bucket credentials
- [Environment variables](resources/env-vars.md) — the exact var list and
  wiring mechanics, especially the internal-vs-public `DATABASE_URL` distinction
- [pnpm preflight](resources/pnpm-preflight.md) — the build-approvals check
  that must pass before a first `railway up`, not just background reading
- [Verify](resources/verify.md) — the deploy-log-before-build-log gate, then
  live browser verification

Also read directly (not duplicated here):
[gotchas/railway-postgres-migrations.md](../../gotchas/railway-postgres-migrations.md) —
why Pre-Deploy Command, not boot-time migration, and how to set it.

## Build order

Execute each step fully and pass its gate before moving to the next.

### Step 0 — Discover the repo and the Railway project
Run the discovery pass. **Gate:** the migration-script precondition is
confirmed, or you've stopped and told the user why; the discovery summary
(Railway project/environment, existing services, existing vars, existing
Pre-Deploy Command, `allowBuilds` state) is reported.

### Step 1 — Settle project/service scope
Resolve the "ask explicitly" items from Before You Start: target Railway
project/environment, any ambiguous existing service, any existing Pre-Deploy
Command. **Gate:** target project/environment id confirmed; every ambiguity
resolved with the user, none guessed.

### Step 2 — pnpm build-approvals preflight
Per `resources/pnpm-preflight.md`. **Gate:** `pnpm-workspace.yaml` contains a
current `allowBuilds` key (not the removed `onlyBuiltDependencies`) —
confirmed by reading the file, not by trusting a locally-green `pnpm
install` (the local pnpm store caches prior approvals and hides this from
you on your own machine).

### Step 3 — Provision Postgres (or confirm existing)
Only if the app uses Postgres (discovery confirms this). Per
`resources/provisioning.md`. **Gate:** exactly one Postgres service exists
on the target project/environment, its identity known.

### Step 4 — Provision Storage Bucket (or confirm existing)
Only if the app uses bucket storage (discovery confirms this). Per
`resources/provisioning.md`; then retrieve credentials via `railway bucket
credentials --bucket <name> --json`. **Gate:** exactly one bucket service
exists; its four credential values retrieved (not yet wired).

### Step 5 — Wire environment variables
Per `resources/env-vars.md`. **Gate:** reading the web service's variables
back confirms `DATABASE_URL` is a Railway variable reference (not a literal
public-proxy connection string) and every var the repo's `.env.example`
lists is present with no placeholder values left.

### Step 6 — Set the Pre-Deploy Command
Per `gotchas/railway-postgres-migrations.md` and `resources/provisioning.md`
— via `railway-agent`/MCP or the dashboard, never a CLI flag (none exists as
of Railway CLI v4.26). **Gate:** reading the service config back confirms
`preDeployCommand` is the exact migration command, not blank or a stale
prior value.

### Step 7 — Trigger the deploy
`railway up`, a push to the connected GitHub repo, or `mcp__railway__create-deployment`/`redeploy`;
poll `mcp__railway__get-status`/`list-deployments` until terminal. **Gate:**
the deployment reaches SUCCESS, or a FAILED state with logs pulled for
diagnosis — never proceed to verification on an in-progress deploy.

### Step 8 — Verify via deploy log
Per `resources/verify.md` — read the **deploy** log via
`mcp__railway__get-logs`, not the build log. **Gate:** the migration
script's own stdout (e.g. "Migrations applied") is present and precedes the
start command's output; stop and diagnose if it's absent or errors.

### Step 9 — Verify live and close out
Visit the generated URL (or the custom domain if wired in Step 1); exercise
the seam that proves Postgres (and the bucket, if in scope) are actually
live — log in with a provisioned user, create/read a real record, upload and
view a file if storage is in scope. Update `CLAUDE.md`/`README.md` deploy
status if the repo tracks that. **Gate:** the live app round-trips real data
through Postgres (and the bucket, if in scope), observed in the browser —
not just "the deploy succeeded."

## What changes

This skill mostly touches **Railway project config**, not repo files — the
delta over the existing app is intentionally small:

```
your-app/
├── pnpm-workspace.yaml   ← edited (only if Step 2 found a stale/missing allowBuilds map)
├── CLAUDE.md             ← edited (deploy status, if this repo tracks phase state)
└── README.md             ← edited (deploy status / live URL, if this repo tracks that)
```

**What changes on Railway, not in the repo:**
- Postgres service (newly provisioned, or confirmed existing)
- Storage Bucket service (newly provisioned, or confirmed existing — only if
  the app uses storage)
- Env vars on the app's web service: `DATABASE_URL` (internal variable
  reference), `AUTH_SESSION_SECRET`, the four `BUCKET_*` vars (only if
  storage is in scope)
- `deploy.preDeployCommand` on the web service
- The live deployment itself

This skill assumes `scripts/migrate.mjs` (or equivalent), the storage
feature, and a correct `.env.example` already exist from the scaffolding
skill/recipe — it does not create them.
