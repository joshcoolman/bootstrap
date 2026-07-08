# Docs re-weld + the pattern doc

Docs-first repos state their boundary in writing, and `next-app`'s almost
certainly says some form of "no auth" (or is a Phase-0 placeholder that
hasn't scoped it either way yet). This step comes FIRST — before any code —
because writing the new boundary is where the scope questions get settled,
and because `docs/FEATURE-MODULE-PATTERN.md` (written here) is what
`auth-feature.md` and `example-feature.md` both point back to.

## The scope questions (settle them here, or earlier if discovery raised them)

- **Signup?** No app UI for it — accounts are provisioned by the setup
  wizard via the admin API, or manually in the Supabase dashboard. The
  wizard also requires disabling Supabase's own public-signup project
  setting.
- **Password reset?** No — same as `add-simple-auth`, deliberately out of
  scope for v1 (needs `detectSessionInUrl` + `PASSWORD_RECOVERY` handling).
- **The optional example feature — building it or not?** Settle this now if
  it wasn't already asked. If yes, confirm the table name (`user_snippets`
  by default — see `example-feature.md` for why not `todos`).
- **What stays out?** OAuth, roles/permissions beyond "this row belongs to
  this user," and any UI for account management (password change, email
  change) — all explicitly out of scope for v1.

## No `FEATURE-MODULE-PATTERN.md` exists yet — you are writing the first one

GitHub issue #1 (the origin of this skill) assumed this file already
existed somewhere to draw enforcement discipline from. It doesn't — not in
this plugin, not in the reference app `~/repos/effective` it was drawn
from. The discipline only exists as scattered code and comments in that
repo's `src/features/auth/server.ts` and `src/features/todos/server.ts`.
This step is where it gets written down for the first time, as
`docs/FEATURE-MODULE-PATTERN.md` in the target repo.

### What it must cover

Write this synthesized from the actual code in `auth-feature.md` and
`example-feature.md` (both already ported from the reference app) — don't
invent new discipline, describe what the code you just wrote (or are about
to write) actually does:

1. **An identity port, not a passed-down user id.** Any feature that needs
   to know "who is asking" calls `getCurrentUser()` (from
   `<features-dir>/auth`) itself — the UI/route layer never passes a user
   id down as a prop or argument. This is what makes it structurally hard
   for a feature to forget the check: the port is the only way to get a
   user id at all.
2. **RLS is the real enforcement; an explicit filter is defense-in-depth
   and documentation of intent.** Every table a feature owns gets a Postgres
   RLS policy (`using (auth.uid() = user_id) with check (auth.uid() =
   user_id)`, or equivalent) — that's what actually can't be bypassed by
   application code, a bug, or a future edit. The feature's own queries
   *also* filter explicitly (`.eq('user_id', ...)`) so the scoping is
   visible in the code, not just implied by a database policy someone has
   to go look up.
3. **A fixed discriminated-union error taxonomy per feature.** Each feature
   defines its own error type (e.g. `SnippetsError = { _tag:
   'Unauthenticated' } | { _tag: 'DbError'; message }`) in its `types.ts`.
   Callers pattern-match on `_tag`; errors never cross the service/use-case
   boundary as thrown exceptions or untyped strings.
4. **Enforcement lives in the service/use-case layer, never only in the
   UI.** A page or component may also check auth state for UX (e.g. to show
   a sign-in prompt instead of a blank list), but that check is never the
   *only* thing standing between a request and someone else's row. The
   service function (`listSnippets`, `createSnippet`, …) re-derives the
   current user itself and re-applies the filter, regardless of what the
   caller already believes.
5. **A second table/feature follows the same shape**: a migration with an
   RLS policy scoped to `auth.uid()`, a `types.ts` with a fixed error union,
   and service functions that call `getCurrentUser()` and filter
   explicitly. Point at `<features-dir>/snippets/` (if included) as the
   worked example, or at `<features-dir>/auth/` alone if it wasn't.

### Where it goes

`docs/FEATURE-MODULE-PATTERN.md` at the target repo root — alongside
`OVERVIEW.md`/`SPEC.md`/`PLAN.md`/`STYLE.md` if `next-app`'s doc convention
is in place (it becomes viewable at `/docs` automatically, no viewer code
changes needed — `next-app`'s docs route globs everything under `docs/`).
If the target repo uses a different docs convention, ask rather than
assume — this file's home should match wherever `SPEC.md`/`PLAN.md` already
live.

## The hunt list — docs asserting "no auth"

From discovery you have the list of docs making this claim. Typical
locations and the edit each needs (adapt voice to the repo, keep the
substance):

- **Phase-0 placeholder docs** — if `SPEC.md`/`OVERVIEW.md` is still an
  unscoped placeholder with no concrete "no auth" claim to find, skip it;
  the `PLAN.md` phase entry and root `CLAUDE.md` update are the only
  re-weld required. Common when this skill runs soon after `next-app`,
  before the app has been scoped.
- **SPEC-style doc** — add a "does" bullet for the guarded route and
  per-user data isolation; narrow any "does not talk to any backend/auth"
  bullet to name what's still actually out (signup UI, password reset,
  OAuth).
- **OVERVIEW-style doc** — a "not opinionated about backend or auth" line
  becomes "auth ships as per-user Supabase email/password sign-in, with
  Postgres RLS enforcing data isolation — see
  `docs/FEATURE-MODULE-PATTERN.md`."
- **PLAN-style doc** — add the auth phase entry:

  > ## Phase N — Auth
  >
  > Per-user Supabase email/password sign-in — distinct accounts, each
  > user's data isolated via Postgres RLS. No signup UI; accounts are
  > provisioned by `pnpm setup:supabase` (admin API) or the dashboard.
  >
  > 1. Identity layer (`src/lib/supabase/*`, `<features-dir>/auth/`) —
  >    dormant by default, degrades gracefully with zero env vars.
  > 2. `src/proxy.ts` + `src/lib/supabase/middleware.ts` — session refresh
  >    and route gating.
  > 3. `/login` + a guarded route, with server-side defense-in-depth.
  > 4. *(if included)* an example per-user feature (`<features-dir>/snippets/`
  >    + its RLS-backed migration) proving the isolation end-to-end.
  > 5. `docs/FEATURE-MODULE-PATTERN.md` — the enforcement discipline every
  >    future per-user feature in this repo should follow.
  > 6. `pnpm setup:supabase` — the user runs it: link or create the hosted
  >    project, apply migrations, provision account(s), disable public
  >    sign-ups.
  >
  > After this phase: visiting the guarded route signed-out redirects to
  > `/login`; signing in lands back where headed; two distinct accounts see
  > only their own data.

- **Root `CLAUDE.md`** — current-phase note; the auth feature (and
  `snippets`, if included) in the structure list; `pnpm setup:supabase` in
  commands (flag the pnpm collision: "Not `pnpm setup` — that's a pnpm
  built-in"); a pointer to `docs/FEATURE-MODULE-PATTERN.md` for any future
  per-user feature work.

## Gate

No doc in the repo still claims "no auth"; `docs/FEATURE-MODULE-PATTERN.md`
exists and is reachable from root `CLAUDE.md`; the scope questions are
settled and recorded; lint (and the `/docs` viewer, if driven) is happy
with the edited/new files.
