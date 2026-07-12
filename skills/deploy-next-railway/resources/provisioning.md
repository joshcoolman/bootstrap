# Provisioning: Postgres and Storage Bucket

Covers Steps 1, 3, 4, and 6 of the build order: settling which Railway
project/services to use, creating Postgres and a bucket when they don't
already exist, and setting the Pre-Deploy Command.

Prefer `mcp__railway__railway-agent` for anything multi-step or requiring
judgment (it's the most capable entry point per its own tool description);
use the narrower `mcp__railway__*` tools (`list-projects`, `list-services`,
`create-project`, `get-status`) for simple, single-purpose reads/writes. Fall
back to the `railway` CLI or explicit dashboard instructions only when MCP
tools aren't available in this session — confirm with the user before any
destructive action (e.g. `accept-deploy`) regardless of which path is used.

## Project/environment scope (Step 1)

If discovery found no linked Railway project: ask whether to create a new
one or attach to an existing one under the account (`list-projects` first,
so the user is choosing from a real list, not guessing names). Never create
a project silently — this is a billing/identity decision, not a mechanical
one.

If a project is already linked, confirm the target environment (usually
`production`, but ask if discovery found more than one environment and it's
unclear which is intended).

## Postgres (Step 3)

Only if discovery confirmed the app uses Postgres.

- If discovery found an existing, unambiguous Postgres service on the
  target project/environment: use it. Nothing to provision.
- If discovery found an ambiguous set of candidates (more than one
  Postgres-looking service, or one whose name doesn't obviously belong to
  this app): stop and ask which one, or whether to provision a fresh one
  instead. Do not guess.
- If none exists: provision one via `railway-agent` (a plain instruction
  like "add a Postgres database to this project/environment" is enough) or
  `railway add postgres` / the dashboard's "New" → "Database" → "Add
  PostgreSQL" flow as fallback.

**Gate this step against:** exactly one Postgres service is now known to
exist on the target project/environment, and you can name it.

## Storage Bucket (Step 4)

Only if discovery confirmed the app uses bucket storage — skip this step
entirely otherwise, don't provision a bucket nobody reads from.

- Same existing/ambiguous/none logic as Postgres above, but for a Railway
  Storage Bucket service.
- Once a bucket exists (confirmed or freshly provisioned), retrieve its
  credentials:
  ```
  railway bucket credentials --bucket <name> --json
  ```
  This returns the endpoint, access key id, secret access key, and bucket
  name — the four values `resources/env-vars.md` wires into the web
  service's env vars in Step 5. Do not print the secret access key back to
  the user in plain chat if avoidable; treat it like any other credential.

**Gate this step against:** exactly one bucket service is known to exist,
and its four credential values have been retrieved (not yet wired — that's
Step 5).

## Pre-Deploy Command (Step 6)

Full reasoning lives in
[gotchas/railway-postgres-migrations.md](../../../gotchas/railway-postgres-migrations.md) —
read it, don't re-derive it. Summary of the mechanics:

- The web service needs `deploy.preDeployCommand` set to the exact command
  discovery found (Step 0's summary — e.g. `node scripts/migrate.mjs` or
  `pnpm db:migrate`). This runs once, to completion, before each new
  deploy's start command boots — never wire migrations into the app's own
  boot sequence instead.
- **There is no plain `railway` CLI flag for this as of CLI v4.26.** Set it
  via `railway-agent` (a plain instruction like "set the Pre-Deploy Command
  on the web service to `pnpm db:migrate`"), the Railway GraphQL API
  directly, or the dashboard (service → Settings → Deploy → Pre-Deploy
  Command) as fallback.
- If discovery found an existing Pre-Deploy Command with different content:
  surface the exact current value and ask before overwriting — never
  silently clobber it. It may be intentionally chained with another step.

**Gate this step against:** reading the service config back (via
`railway-agent`/MCP or the dashboard) confirms `preDeployCommand` is set to
the exact migration command — not blank, not the pre-existing different
value if one was found and not yet resolved with the user.
