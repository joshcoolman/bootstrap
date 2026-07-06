# Verify and hand off

The finish is two-stage: an agent-verifiable gate proving *the code is right*,
then a user-run stage proving *the account wiring is right*. Don't blur them —
the gate needs no Supabase project at all, which is the point.

## Stage 1 — the headless unconfigured-state gate (you)

Because the adapter degrades instead of throwing, the entire auth flow is
observable with **no `.env.local` and no Supabase project**. Verify it in the
running app before involving the user:

1. Confirm no `.env.local` exists (move it aside if one does).
2. Start `pnpm dev` and drive the app — with a headless browser if one is
   available (system Chrome via `playwright-core` works without downloads),
   otherwise ask the user to click through this checklist:
   - Visit `/dashboard` (or the chosen protected route) → lands on
     `/login?redirect=%2Fdashboard`, the sign-in form renders.
   - Submit any email/password → an inline alert reads
     "Auth is not configured — run pnpm setup:supabase…".
   - Visit `/` and any other pre-existing routes → unaffected by the provider
     mount.
   - Both themes render the form correctly (if the app has a theme toggle).
3. Stop the `pnpm dev` process once this checklist is done. A leftover
   process from this step, plus one from `routes.md`'s routeTree-regen step,
   is exactly how stray Vite servers accumulate and silently push the real
   one to a drifted port.
4. `pnpm lint && pnpm test && pnpm build` — all green with zero env vars.
   This is exactly what CI will run.

Only when this gate passes do you hand off.

## Stage 2 — the handoff (the user)

Deliver a message with this substance (adapt tone; keep every warning). End
it by repeating the bare command as the final line — the explanation scrolls,
and the last visible line should be the exact next thing to do:

> The setup wizard will walk you through:
>
> 1. **Supabase CLI check** — installs via `brew install supabase/tap/supabase`
>    if missing.
> 2. **Login** — opens a browser. macOS may ask about Keychain access; click
>    **Deny**, the login still works.
> 3. **Project** — pick an existing Supabase project or create one. Creating
>    one asks for a database password shown once and never retrievable —
>    save it before continuing (auth never needs it again, but the project
>    does).
> 4. **Your account** — the email and password that will be the app's only
>    login. There is no signup; re-running the wizard later is safe.
> 5. **Security lockdown** — the wizard shows you exactly how to disable
>    public sign-ups in the Supabase dashboard, then blocks until you confirm
>    it's done. This is required, not optional: the app treats any account in
>    the project as "logged in," so leaving sign-ups on means anyone who
>    finds your app's public anon key could create their own account and
>    sign in as if they were you.
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

- Visiting the protected route signed-out redirects to `/login` and returns
  to the original destination after sign-in (the `?redirect=` round-trip).
- A hard reload stays signed in (localStorage persistence).
- A wrong password shows the provider's error inline.
- Sign out lands back on `/login`.
- Public sign-ups are actually off: call `supabase.auth.signUp()` (or POST
  `/auth/v1/signup`) with a throwaway email/password against the real
  project's anon key, and confirm it's rejected ("Signups not allowed for
  this instance"). A rejection has no side effect, so this is safe to run
  against the live project.

## Close-out checklist

- Update the PLAN-style doc: auth phase → complete; next phase named.
- Update root `CLAUDE.md`: current phase, the auth feature in the structure
  list, `pnpm setup:supabase` in commands.
- Final `pnpm lint && pnpm test && pnpm build` green.
- Commit — **including the regenerated `src/routeTree.gen.ts`** (CI's tsc
  runs before the route tree regenerates; a stale committed tree fails the
  build). Confirm `.env.local` and `supabase/.temp` are NOT in the commit.

The code layer is verified; the account wiring belongs to the user's wizard
run.
