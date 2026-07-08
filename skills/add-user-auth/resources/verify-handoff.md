# Verify and hand off

The finish is two-stage, same discipline as `add-simple-auth`: an
agent-verifiable gate proving *the code is right*, then a user-run stage
proving *the account wiring is right*. Don't blur them — the headless gate
needs no Supabase project at all, which is the point.

## Stage 1 — the headless dormant-state gate (you)

Because every Supabase-touching file checks `isSupabaseConfigured` first,
the entire flow is observable with **no `.env.local` and no Supabase
project**:

1. Confirm no `.env.local` exists (move it aside if one does).
2. Start `pnpm dev` and drive the app — with a headless browser if one is
   available, otherwise ask the user to click through this checklist:
   - Visit `/dashboard` (or the chosen guarded route) → renders directly,
     no redirect (dormant middleware no-ops; the layout's defense-in-depth
     check is skipped while unconfigured — this is the correct dormant
     behavior, not a bug).
   - Visit `/login` → the sign-in form renders.
   - Submit any email/password → an inline alert reads "Auth is not
     configured — run pnpm setup:supabase…".
   - Visit `/` and any other pre-existing routes → unaffected.
   - If the optional `snippets` feature was included: `/dashboard` shows
     the "set up Supabase" message, not a crash or an empty list that looks
     like real data.
3. Stop the `pnpm dev` process once this checklist is done.
4. `pnpm lint && pnpm test && pnpm build` — all green with zero env vars.
   This is exactly what CI will run.

Only when this gate passes do you hand off.

## Stage 2 — the handoff (the user)

Deliver a message with this substance (adapt tone; keep every warning). End
it by repeating the bare command as the final line:

> The setup wizard will walk you through:
>
> 1. **Supabase CLI check** — installs via `brew install supabase/tap/supabase`
>    if missing.
> 2. **Login** — opens a browser. macOS may ask about Keychain access; click
>    **Deny**, the login still works (or set `SUPABASE_ACCESS_TOKEN` first
>    to skip this entirely — the wizard explains how).
> 3. **Project** — pick an existing Supabase project or create one. Creating
>    one asks for a database password shown once and never retrievable —
>    save it before continuing.
> 4. **Schema** *(only if you added the optional example feature)* — applies
>    the migration that creates the per-user table + its RLS policy.
> 5. **Accounts** — create at least two, so you can see per-user data
>    isolation in action. There is no signup UI; re-running the wizard
>    later to add more accounts is safe.
> 6. **Security lockdown** — the wizard shows you exactly how to disable
>    public sign-ups in the Supabase dashboard, then blocks until you
>    confirm it's done. This is required, not optional.
>
> When it finishes: `pnpm dev` and sign in at
> `http://localhost:<dev-port>/login`.
>
> Run this command in your terminal:
>
> ```
> pnpm setup:supabase
> ```

While they run it, ask them to report anything that feels rough — wizard
friction is signal, not noise.

## Stage 3 — live verification (together)

With the user signed in, walk the flow:

- Visiting the guarded route signed-out redirects to `/login` and returns
  to the original destination after sign-in (the `?redirect=` round-trip,
  via middleware).
- A wrong password shows the login action's error inline.
- Sign out lands back on `/login`.
- Public sign-ups are actually off: call `supabase.auth.signUp()` (or POST
  `/auth/v1/signup`) with a throwaway email/password against the real
  project's publishable key, and confirm it's rejected ("Signups not
  allowed for this instance"). A rejection has no side effect, so this is
  safe to run against the live project.
- **If the optional `snippets` feature was included — the actual point of
  this skill:** sign in as account A, create a snippet, sign out. Sign in
  as account B, confirm the list is empty. Confirm account B cannot see or
  delete account A's snippet (e.g. via the browser devtools network tab,
  attempting to hit the delete action with account A's snippet id while
  signed in as B — RLS should reject it regardless of what the UI
  prevents). This is the one thing the headless gate structurally cannot
  prove — it needs two real accounts against a real database.

## Close-out checklist

- Update the PLAN-style doc (`docs/PLAN.md` if `next-app`'s convention):
  auth phase → complete; next phase named.
- Confirm `docs/FEATURE-MODULE-PATTERN.md` exists and root `CLAUDE.md`
  references it (see `docs-reweld.md`).
- Update root `CLAUDE.md`: current phase, the auth feature (and `snippets`,
  if included) in the structure list, `pnpm setup:supabase` in commands.
- Final `pnpm lint && pnpm test && pnpm build` green.
- Commit. Confirm `.env.local` and `supabase/.temp` are NOT in the commit —
  `next-app`'s `.gitignore` already covers `.env.local` via its `*.local`
  glob; confirm `supabase/.temp` was added in `setup-wizard.md`'s step.

The code layer is verified; the account wiring — and the actual proof of
per-user isolation, if the example feature was included — belongs to the
user's live verification.
