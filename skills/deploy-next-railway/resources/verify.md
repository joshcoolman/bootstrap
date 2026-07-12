# Verify

Covers Steps 8–9: after the deploy (Step 7) reaches a terminal state, confirm
it actually worked — not just that Railway reports success.

## Step 8 — Verify via the deploy log, not the build log

Railway keeps separate build and deploy logs. The build log shows the
Nixpacks/Docker build succeeding; it says nothing about whether the
Pre-Deploy Command ran or whether the migration actually applied. Read the
**deploy** log via `mcp__railway__get-logs` (or the dashboard's Deploy Logs
tab, not Build Logs):

- Confirm the migration script's own stdout appears (e.g. "Migrations
  applied", or whatever your `scripts/migrate.mjs` actually prints) —
  **before** the app's start-command output (e.g. Next.js's "Ready in
  Xms").
- If migration output is absent, or the log shows an error from the
  migration step, **stop here** — do not proceed to live verification on a
  deployment whose schema state is unknown. Diagnose (common causes: the
  Pre-Deploy Command wasn't actually set — re-check Step 6's gate; the
  `DATABASE_URL` reference resolved to the wrong service; the migration
  script itself has a bug not exercised locally).

## Step 9 — Verify live, then close out

Visit the deployment's generated URL (`*.up.railway.app`, or the custom
domain if one was wired in Step 1). Exercise the thing that actually proves
the seams are wired, not just that a page renders:

- **Auth in scope:** log in with a user provisioned via the app's own
  provisioning script (e.g. `pnpm auth:create-user`) — confirms
  `AUTH_SESSION_SECRET` and the `users` table are correctly wired.
- **A real record round-trip:** create something that writes to Postgres
  (the todos demo from `recipes/next-railway-app.md` Step 5, or whatever the
  app's actual data model is), reload the page, confirm it persisted —
  confirms `DATABASE_URL` is pointed at a real, migrated database, not just
  that the connection string parses.
- **Storage in scope:** upload a file and view it back — confirms the
  `BUCKET_*` vars and presigned-URL flow actually work against the live
  bucket, not just that credentials were accepted at provisioning time.

**Gate:** the live app round-trips real data through Postgres (and the
bucket, if in scope), observed directly in the browser — "the deployment
succeeded" is not the same claim as "the app works," and this step is what
closes that gap.

## Close out

If the target repo tracks deploy/phase status (a `## Status` section in
`README.md`, a "current phase" line in `CLAUDE.md`), update it with the live
URL and what was verified. No commit is needed unless Step 2's pnpm
preflight actually changed `pnpm-workspace.yaml` — if it did, that change
should already be committed before Step 7's deploy, not after.
