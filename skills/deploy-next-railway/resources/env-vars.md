# Environment variables

Covers Step 5: wiring the web service's env vars on Railway, after Steps 3–4
have confirmed/provisioned Postgres and (if in scope) a bucket.

Set every var below on the app's **web service** (not the Postgres or bucket
service itself) via `railway-agent`/MCP or the dashboard (service →
Variables).

## `DATABASE_URL` — the internal reference, never the public proxy

```
DATABASE_URL = ${{Postgres.DATABASE_URL}}
```

(Exact reference syntax depends on the Postgres service's name in this
project — `railway-agent` or the dashboard's variable-reference picker will
surface the correct token; don't hand-type a guessed service name.)

**Why this matters, explicitly:** Railway's Postgres has both a public proxy
URL (reachable from a laptop, for local dev) and an internal URL (reachable
only from other services in the same private network, but faster and not
exposed to the internet). A production web service should always use the
**internal** reference — pointing it at the public proxy URL works but is
the wrong choice, and pointing local dev at the *same* instance as
production (rather than a separate local Postgres, per
`gotchas/local-postgres-for-dev.md`) is a real footgun a live app built from
this exact recipe is currently living with. Don't repeat it: local dev and
this deployed instance must be different databases.

Verify after setting: read the var back and confirm it's the reference
token, not a literal `postgresql://...proxy.rlwy.net...` string — a literal
proxy URL pasted in here is the bug this step exists to prevent.

## `AUTH_SESSION_SECRET` — freshly generated, never reused

```
AUTH_SESSION_SECRET = <output of: openssl rand -hex 32>
```

Generate a **new** value for this deployment — do not copy the value from
`.env.local` or any other environment. If local dev's session secret leaks
into production (or vice versa), a session cookie signed in one environment
would verify as valid in the other.

## `BUCKET_*` — only if the app uses storage

From Step 4's `railway bucket credentials --bucket <name> --json` output,
mapped to whatever names `.env.example` actually uses (confirm the exact
names via discovery — the recipe's convention is these four, but a
hand-built app might differ slightly):

```
BUCKET_ENDPOINT          = <endpoint from credentials output>
BUCKET_ACCESS_KEY_ID     = <access key id from credentials output>
BUCKET_SECRET_ACCESS_KEY = <secret access key from credentials output>
BUCKET_NAME              = <bucket name>
```

Skip this section entirely if discovery found the app doesn't use bucket
storage — don't invent vars the app never reads.

## Verification gate

After setting every var, read the web service's variables back (via
`railway-agent`/MCP or the dashboard) and confirm:

- `DATABASE_URL` is a variable reference, not a literal connection string
  (and specifically not the public proxy URL)
- Every var name `.env.example` lists is present
- No placeholder values (`<...>`, `TODO`, empty strings) remain
