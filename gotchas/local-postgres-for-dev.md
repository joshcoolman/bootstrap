# Local dev needs its own Postgres, separate from Railway's private one

Default to this at scaffold time when a project uses Railway's
private-network Postgres (no public port) — don't leave local dev
DB access as an unresolved later decision.

## The trap

A Railway Postgres provisioned with no public TCP proxy (the usual,
more-secure default) is only reachable from other services inside the
same Railway project's private network. It is **not** reachable from a
laptop running `pnpm dev` — not via the plain `DATABASE_URL`, not via
`railway run`, which only injects env vars, not a network tunnel.
`railway connect <service>` opens a psql/mongosh shell via a tunnel, but
that's a one-off interactive shell, not something the app's own Postgres
client can use.

If a project's `.env.local` has a `DATABASE_URL` pointing at
`localhost:5432` but nothing has ever actually run there, that's the
tell — local dev against a real Postgres was never exercised, only
planned.

## The fix: a disposable local Postgres container

```bash
docker run -d --name <project>-local-pg \
  -e POSTGRES_USER=user -e POSTGRES_PASSWORD=<match .env.local> \
  -e POSTGRES_DB=<db name> -p 5432:5432 postgres:16-alpine
```

Then run migrations against it directly — plain `node scripts/migrate.mjs`
(or equivalent) does **not** load `.env.local` the way Vite's dev server
does, so pass it explicitly:

```bash
node --env-file=.env.local scripts/migrate.mjs
```

For a repeatable, checked-in version, add a `docker-compose.yml` with
`POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` read from shell env
(no hardcoded default for the password — never commit a real credential,
even a local-only one, into a tracked compose file).

This is a genuinely separate database from whatever Railway hosts — fine
for exploratory/manual testing, but don't treat data in it as durable.

First hit in: `prompt-smith` (Phase 3 UI verification) — see its
`gotchas/local-postgres-for-dev.md`.
