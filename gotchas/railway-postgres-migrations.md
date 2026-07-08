# Railway + Postgres: run migrations automatically

Default to this at scaffold time — don't leave it as a later decision.

## The pattern

Railway services have a **Pre-Deploy Command**
(`deploy.preDeployCommand`) that runs once, to completion, before the new
deploy's start command boots. This is the first-party place to run DB
migrations — not a manual `railway ssh -- <migrate>` step, not a
boot-time check inside the app's own start script.

## Why not just migrate on app boot

Running migrations inside the app's own startup is riskier under
multiple replicas (two containers racing to migrate) and buries
migration failures inside app-crash-loop logs instead of a clean,
separate deploy-log entry you can check before the app even starts.

## Setup

1. Ship a plain migration-runner script with no interactive CLI
   dependency at runtime — e.g. a Node script that reads generated SQL
   migration files and applies them directly via the DB driver. Avoid
   needing the ORM's CLI (e.g. `drizzle-kit`) installed in the
   production image just to run migrations.
2. Set `preDeployCommand` on the service. As of Railway CLI v4.26 there's
   no plain CLI flag for this — use the dashboard, the GraphQL API, or
   the Railway MCP server's `railway-agent` tool.
3. Verify by reading the **deploy** log (not the build log) of the next
   deployment — you should see the migration script's own stdout (e.g.
   "Migrations applied") before the start command's output.

First hit in: `prompt-smith` — see its `gotchas/railway-postgres-migrations.md`
and README `## Status` for the specific deploy (`6bb98169`) that confirmed
this working.
