# Plan — self-host the commodity layers, drop Supabase

## Why

Across five platforms and four image apps, the same thing kept happening: each
app was built on a *platform differentiator* — Supabase RLS, Vercel Deployment
Protection, Blob's proprietary API, Railway's persistent process — and each time
the app became unportable and eventually got abandoned.

The layers the apps actually need are commodity everywhere:

| Need | Commodity form | Differentiator that traps you |
| --- | --- | --- |
| Auth | password hash + signed cookie | RLS, Clerk, Deployment Protection |
| Database | a Postgres connection string | supabase-js in the browser, D1 |
| Storage | S3 API + a stable key | Blob's SDK, presign-only buckets |
| Generation | an API key over HTTPS | — already commodity |

**Rule: depend on the commodity, never the differentiator.**

The goal is a template that produces, in one command, a deployed app with a
login and a blank dashboard — so an exploration can start in minutes and be torn
down without regret. One exploration, one app. Teardown is routine.

Requirement that settles the auth question: *"let someone else log in if they
want to check it out."* Deployment Protection can't do that — it gates to the
Vercel account. Owned auth can: adding a viewer is adding a row.

## Target shape

`pnpm setup` on a fresh clone produces a live URL with:

- `/login` — email + password, no signup route
- `/dashboard` — blank, guarded, one nav slot
- deny-by-default middleware; everything except `/login` is gated
- `pnpm add-user <email> <password>`
- optional Postgres + migration runner
- Tailwind + Paper & Ink tokens
- `.env.local` written, env pushed to host, deployed, URL printed

Base framework: **Next.js only.** Both recent apps are Next, and the auth guard
is Next middleware — the one piece that doesn't lift verbatim to TanStack.
Maintaining three shells is how the template becomes the monolith it exists to
prevent. Deploy target stays a choice; it is only the wizard's provider block.

## Phase 1 — save what's at risk

`~/repos/gimme-image/agent-surface-big-ideas.md` is 247 uncommitted lines that
exist on exactly one machine. Not gimme-image-specific: a brief on giving a
personal tool an agent surface ("one core, two surfaces"), blast-radius framing
for personal MCP servers, and an appendix of concepts (`rich input ↔ shareable
output ↔ no persistence: pick two`; constructor-vs-designer).

Move it to `parts/agent-surface.md`. Do this before anything else — it is the
only irreplaceable artifact in this plan.

## Phase 2 — owned auth

New skill `add-selfhost-auth`. Nothing is deleted — the existing Supabase skills
get renamed instead.

**Rename, don't delete.** `add-simple-auth` and `add-user-auth` are validated
work and Supabase auth is still a legitimate fast path if you don't mind the
vendor. The problem is the naming: neither says Supabase, which is how a repo
ends up accidentally coupled to it.

They also split on the wrong axis. Simple-vs-multi-user is a *credential store*
difference — a flag, not a skill boundary. The boundary that matters is vendor:

| Now | Becomes |
| --- | --- |
| `add-simple-auth` | `add-supabase-auth --mode=shared` |
| `add-user-auth` | `add-supabase-auth --mode=users` |
| — | `add-selfhost-auth` (new, and the recommended default) |

Two auth skills, split by vendor, each with a mode flag. `add-selfhost-auth` is
what the README points at; `add-supabase-auth` stays available and honestly
labelled.

Source: `~/repos/bootsy/src/features/auth/` — running in production, ~200 lines.

Shared session layer, identical for both backends:

- `session.ts` — cookie value `userId.issuedAtMs.hmacSha256`, signed with
  `AUTH_SESSION_SECRET`. **Web Crypto only**, so it runs in Edge middleware.
  Hand-written `timingSafeEqualHex` (Web Crypto has no timing-safe compare),
  plus an age check. Stateless — no sessions table, so no server-side
  revocation; logout clears the cookie.
- `credentials.ts` — `node:crypto` scrypt, stored as `salt:hexkey`, verified
  with `timingSafeEqual`. Unknown-email logins hash against a `DUMMY_HASH` so
  the timing side-channel doesn't leak which emails exist.
- `proxy.ts` (Next 16's renamed middleware) — deny-by-default matcher; every
  path except `/login` and static assets redirects when unsigned.
- `current-user.ts` — re-reads the cookie server-side and **throws** if absent,
  so data access never trusts the middleware alone.

Two credential backends:

- **`--backend=env`** — `AUTH_USERS` holds `email:salt:hash` entries. No
  database. Adding a viewer is an env entry plus a redeploy. This is the default
  and covers most explorations.
- **`--backend=db`** — a `users` table, `pnpm add-user` inserts or resets. Use
  when the app has Postgres anyway.

Carry over from bootsy, deliberately:

- The Edge/Node import split. Middleware must import `./session` directly, never
  the barrel — the barrel re-exports `credentials.ts`, which pulls `node:crypto`
  and breaks the Edge bundle. bootsy documents this in a comment; the skill
  should make it structural (no barrel that mixes the two).

Fix on the way in:

- bootsy duplicates its scrypt shape between `credentials.ts` and
  `scripts/hash-lib.mjs`, with a comment admitting they must stay identical.
  Drift there is a silent auth failure. One module, imported by both.

Explicitly not included: signup, password reset, email verification, OAuth,
roles. This is **allowlist auth**. If an app ever needs more, that is the signal
to adopt better-auth — not a reason to pre-build it here.

## Phase 3 — storage part

New `parts/image-store.md`: a contract plus one reference implementation.

```ts
interface ImageStore {
  put(key: string, bytes: Uint8Array, contentType: string): Promise<{ key: string }>
  displayUrl(key: string): Promise<string>   // async ALWAYS
  delete(keys: string[]): Promise<void>
  list(prefix: string): Promise<{ key: string }[]>
}
```

Two deliberate choices, both learned the hard way:

1. **`displayUrl` is async and its result is always treated as ephemeral**, even
   when the implementation returns a permanent URL. A presigned URL dies in an
   hour; a public one doesn't. Code written against the permanent case is
   silently broken on the other.
2. **Keyed by a stable key, never by a URL.** The trap presigning sets is using
   the display URL as an identity key — saves, dedupe, and references all break
   an hour later. Carry identity and display separately.

Reference implementation: **Cloudflare R2 via plain `@aws-sdk/client-s3`.**

- Fully S3-compatible from any host — no Workers, no Cloudflare runtime
- Public URLs without presigning
- Egress genuinely $0; storage $0.015/GB-mo; free tier 10 GB + 1M writes + 10M
  reads per month
- Lock-in is near zero: it is the AWS SDK pointed at a different endpoint

Two caveats to write into the part:

- `r2.dev` is explicitly **not for production** and rate-limits to 429s. Use a
  custom domain on Cloudflare DNS from day one. If there's no domain there, the
  advantage shrinks and Supabase Storage or B2 are fair comparisons.
- Use **content-addressed keys** (hash in the filename, immutable). This makes
  the CDN cache/delete skew problem — stale objects and cached 404s — disappear
  entirely.

Note the one thing the contract cannot hide: **background work after the
response.** Railway's persistent process runs an un-awaited copy to completion;
Vercel freezes the lambda and needs `waitUntil`, which has its own timeout. Same
signature, different guarantees. The part should state the constraint (must be
idempotent, must be safe to lose) and mandate bootsy's retry-on-read backstop —
null column means not-yet-done, every read opportunistically re-fires, an
in-memory Set de-dupes, and writes are conditional (`where key is null`) so
racing repairs can't clobber.

And: **log the failure.** bootsy's background copy swallows errors silently, so
a permanently-broken copy path would retry forever with zero signal.

## Phase 4 — the wizard

Port `~/repos/gimme-image/scripts/setup.mjs` (~308 LOC, `@inquirer/prompts`,
proven end-to-end). Roughly two-thirds is provider-agnostic: CLI preflight, host
link, env push, `.env.local` write, deploy, tolerate-already-exists.

Steps: preflight CLIs → provision backend → link host → prompt keys → push env
prod+preview → write `.env.local` → deploy → create first user.

Three fixes, all real:

1. **Mask the key prompts.** It uses `input()` for API keys, so they render in
   the terminal — and in any screenshot. Use `password()`. This has already
   caused one live key exposure.
2. **Don't clobber `.env.local`.** It overwrites unconditionally, no merge, no
   backup. Detect and confirm.
3. **Make deploy optional.** It hard-exits if the host CLI is missing, with no
   skip flag. A `--local-only` path should get you running without deploying.

Provider block is pluggable: Railway (Postgres + bucket) or Vercel (Neon or no
database). Supabase is not an option.

## Order

1. Save `agent-surface.md` — nothing else can lose it
2. Owned auth skill — the largest piece, and it unblocks retiring Supabase
3. Storage part — small, self-contained
4. Wizard — depends on knowing what auth needs provisioned

Stop after each for review.

## Consequences

- `add-simple-auth` + `add-user-auth` merge into `add-supabase-auth` with a
  `--mode` flag; `add-selfhost-auth` is added as the recommended default. Nothing
  validated is thrown away. Skill directory renames break any saved slash
  command referencing the old names.
- README's install example currently shows `/bootstrap:add-simple-auth` and
  `/bootstrap:add-user-auth` — both need updating, and the skills table too.
- Recipes (`next-railway-app.md`, `next-selfhost-app.md`,
  `nest-tanstack-railway-app.md`, `vite-react-app.md`) reference Supabase auth
  and need a pass — each should say which auth skill it assumes now that there
  are two.
- Issue #1 (`add-user-auth`: real per-user auth) is largely satisfied by
  Phase 2 and should be reconciled or closed.
- `gimme-image` can be archived once Phase 4 lifts the wizard.

## Not in scope

- TanStack or Vite variants of the auth guard. Next only, for now.
- Any adapter for auth, database, or hosting. Those have never actually been
  swapped; the abstraction would cost more than a rewrite. Storage gets a seam
  because it has now been written twice.
- Migrating any existing app onto this. The template is for new explorations;
  eve and bootsy stay as they are.
