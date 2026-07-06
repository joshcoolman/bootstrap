# Docs re-weld

Docs-first repos state their boundary in writing, and that boundary almost
certainly says some form of "no auth". Adding auth without editing those docs
leaves the repo lying about itself. This step comes FIRST — before any code —
because writing the new boundary is where the scope questions get settled.

## The scope questions (settle them here)

Writing the boundary forces these; answer them with the user if the
before-you-start questions didn't already:

- Signup? **No app UI for it** — the one account is provisioned by the setup
  wizard via the admin API. The wizard also requires disabling Supabase's own
  public-signup project setting, since that endpoint stays reachable with the
  public anon key otherwise.
- Password reset? **No** — it needs `detectSessionInUrl` + `PASSWORD_RECOVERY`
  event handling; deliberately out of scope for v1.
- OAuth, multiple users, roles? **No.** This is a single shared-credential
  gate: any account in the linked Supabase project gets identical access,
  because the app never checks which user is signed in.
- What stays out? An app database and any domain backend. Auth is the only
  external service, and it stays behind the feature's public contract.

## The hunt list

From discovery you have the list of docs asserting "no auth". Typical
locations and the edit each needs (use the worked examples below as
adapt-to-the-repo's-voice templates, not paste targets):

- **Phase-0 placeholder docs (nothing to rewrite yet)** — if a probed doc
  (SPEC/OVERVIEW/one-line-bet) is still an unscoped Phase-0 placeholder with
  no concrete "no auth" claim to find, skip it. The `PLAN.md` phase entry +
  root `CLAUDE.md` update are the only re-weld required in that case — this
  is the common path when `add-simple-auth` runs immediately after `vite-app`, before
  the app has been scoped.
- **SPEC-style doc, "does / does not" lists** — add a "does" bullet for the
  gated dashboard; rewrite the "does not talk to any backend/auth" bullet so
  the boundary is precise about what is still out.
- **A using-this-repo / one-line-bet doc** — if it says "no backend, no auth,
  no database server", carve auth out as the one external service. If it has
  a "when not to use this" list naming accounts as a disqualifier, narrow it:
  a single shared-credential gate is in; accounts-as-a-product is out.
- **OVERVIEW-style doc** — a "not opinionated about backend or auth" line
  becomes "auth ships as a single shared-credential login gate behind a
  vendor-agnostic seam".
- **PLAN-style doc** — add the auth phase entry (template below).
- **Root `CLAUDE.md`** — current-phase note, the auth feature in the structure
  list, and the commands section. The command note must warn about the pnpm
  collision: `pnpm setup:supabase — interactive wizard … (Not pnpm setup —
  that's a pnpm built-in.)`

## Worked example: the SPEC edit (adapt the voice, keep the substance)

Does list, added bullet:

> - Gate a dashboard (`/dashboard`) behind a single shared-credential
>   email/password sign-in (`/login`), via a vendor-agnostic auth seam with
>   Supabase as the first adapter. No self-serve signup — the one account is
>   provisioned by `pnpm setup:supabase`, which also disables public sign-ups
>   at the project level.

Does-not list, replacing the old "no backend/auth" bullet:

> - Have an app database or any domain backend. Auth is the only external
>   service, and it stays behind `src/features/auth`'s public contract.
> - Offer signup, password reset, OAuth, or multi-user roles.

Key-decisions addition:

> - **Auth is a seam, not a vendor.** Routes and components consume the
>   `AuthClient` contract from the auth feature; Supabase lives in one adapter
>   file behind it. Swapping vendors means writing a second adapter, not
>   touching the app.

## Worked example: the PLAN phase entry

> ## Phase N — Auth
>
> A single shared-credential email/password sign-in gating an empty
> dashboard — any account in the linked Supabase project gets identical
> access, not per-user roles. Supabase is the first adapter behind a
> vendor-agnostic seam:
>
> 1. The feature's boundary doc and `types.ts` (the `AuthClient` contract)
>    first.
> 2. Implement behind the feature's public `index.ts`: a lazy env-tolerant
>    Supabase adapter, an in-memory mock adapter, a React provider + `useAuth`.
> 3. Test the contract via the mock adapter — no network, green without env
>    vars (CI has none).
> 4. Wire `/login` and the guarded route (`beforeLoad` + `?redirect=` return),
>    mount the provider at the root.
> 5. `scripts/setup.mjs` (`pnpm setup:supabase`) — the user runs it: link or
>    create the hosted Supabase project, write `.env.local`, provision the
>    account via the admin API, and disable public sign-ups. No signup UI
>    exists.
>
> After this phase: visiting the guarded route signed-out redirects to
> `/login`; signing in lands back where you were headed; a hard reload stays
> signed in.

## Gate

No doc in the repo still claims "no auth"; the scope questions are settled and
recorded; lint (and the docs viewer, if any) is happy with the edited files.
