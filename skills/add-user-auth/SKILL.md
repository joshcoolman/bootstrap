---
name: add-user-auth
description: Add genuine multi-user Supabase auth to an existing Next.js App Router app (a next-app-scaffolded repo) — distinct accounts, per-user data isolation enforced by Postgres RLS plus a service-layer identity check, an interactive setup wizard the user runs themselves, and an optional example feature proving the isolation end-to-end. Use when asked to add real user accounts, per-user data isolation, or multi-tenant auth to a Next.js app — not for a single shared-credential gate (that's add-simple-auth).
---

You are about to add real, per-user auth to the current repo — the one you
are running in. This skill assumes the stack (Next.js App Router + pnpm,
the `next-app` shape) but not the repo's exact shape: Step 0 discovers the
shape and fills adaptation slots (`<alias>`, `<features-dir>`, `<dev-port>`)
that every later resource references. Most steps here **edit or add to an
existing app**, preserving what's there. You write all the code, including
the setup wizard; **the user runs the wizard** — you never drive the
interactive Supabase steps or handle real credentials.

**Drafted directly from `~/repos/effective`** (a live, deployed Next.js 16
app with genuine per-user Supabase auth already proven in use) rather than
built fresh in a disposable mule — the same deviation `next-app` itself
took from this repo's usual mule-first loop. This is a v1 draft: it hasn't
yet been run end-to-end as a fresh consumer against a throwaway `next-app`
scaffold. Treat friction encountered on a first real run as expected
signal, not a sign you misread these instructions.

**Distinct from `add-simple-auth`.** That skill is a single
shared-credential gate — one login, and the app never checks *which* user
is signed in. This skill is for genuine multi-user auth: distinct accounts,
each user's data kept apart from every other user's, enforced at the
database layer (RLS), not just the UI.

## Before you start — check first, only ask if genuinely blocked

Run the discovery pass (`resources/discovery.md`) first — infer what's
inferable from the repo, only ask when something is genuinely ambiguous
(an existing `/login` remnant, a table-name collision, a non-Next-app
shape). Two things are always worth confirming explicitly rather than
inferring, since they're irreversible-ish scope decisions:

1. **Supabase** as the auth vendor (the only one this skill supports; the
   identity-port seam keeps a second vendor cheap later).
2. **The optional example feature** (`resources/example-feature.md`) — ask
   whether to include it. It's the only way this skill's own gate can prove
   multi-tenant isolation end-to-end rather than just "login works," but
   it's genuinely optional.

## Resources — read these in order

- [Discovery](resources/discovery.md) — the Step-0 read-the-repo pass:
  preflight, probes, adaptation slots
- [Docs re-weld](resources/docs-reweld.md) — updating docs that say "no
  auth"; the scope-question ritual; writing the new
  `docs/FEATURE-MODULE-PATTERN.md`
- [Auth feature](resources/auth-feature.md) — the identity layer:
  Supabase client setup, the `getCurrentUser()` port, plain TypeScript (not
  Effect — see the resource for why)
- [Middleware](resources/middleware.md) — `src/proxy.ts` (Next 16's renamed
  middleware convention) + session refresh and route gating
- [Routes](resources/routes.md) — `/login` + a guarded route; no root
  layout edit needed (unlike `add-simple-auth` — see the resource for why)
- [Example feature](resources/example-feature.md) — the optional `snippets`
  demo proving per-user isolation end-to-end
- [Setup wizard](resources/setup-wizard.md) — `scripts/setup-supabase.mjs`
  verbatim, the CLI-drift ledger, provisioning multiple accounts
- [Verify + handoff](resources/verify-handoff.md) — the headless dormant-state
  gate, the handoff message, live verification, close-out

## Build order

Execute each step fully and pass its gate before moving to the next.

### Step 0 — Discover the repo
Run the discovery pass. **Gate:** preflight confirms the stack; the
discovery summary (alias, features dir, dev port, styling idiom, docs to
re-weld, existing routes) is reported to the user.

### Step 1 — Re-weld the docs boundary + settle scope
Edit every doc that asserts "no auth"; settle the scope questions
(signup — none; password reset — none; the optional example feature —
yes/no); write `docs/FEATURE-MODULE-PATTERN.md`. **Gate:** no doc still
claims "no auth"; the pattern doc exists; lint green.

### Step 2 — Deps + env scaffolding
`pnpm add @supabase/ssr @supabase/supabase-js`, `pnpm add -D
@inquirer/prompts`; write `.env.local.example`; confirm `.gitignore`
covers `.env`/`.env.local`/`supabase/.temp`. **Gate:** install succeeds;
lint, test, build still green with zero env vars.

### Step 3 — Identity layer
Write `src/lib/supabase/{env,client,server,session}.ts` and
`<features-dir>/auth/{types.ts,index.ts,CLAUDE.md}` per the auth-feature
resource. **Gate:** `pnpm build && pnpm test && pnpm lint` green with zero
env vars — `getCurrentUser()` resolves to the dormant dev session without
throwing.

### Step 4 — Middleware
Write `src/lib/supabase/middleware.ts` and `src/proxy.ts` per the
middleware resource — note the Next 16 `proxy.ts` naming, not
`middleware.ts`. **Gate:** `pnpm build` green with zero env vars; every
route still renders (middleware no-ops while unconfigured).

### Step 5 — Routes
Write `src/app/login/{page.tsx,login-form.tsx,actions.ts}` and the guarded
route (`src/app/dashboard/{layout.tsx,page.tsx,actions.ts}` or wherever
discovery/the user placed it). Do **not** edit `src/app/layout.tsx` — no
client provider is needed. **Gate:** `pnpm build` green; the dormant-state
flow (Step 9) is drivable even though you verify it properly later.

### Step 6 — Optional example feature
Only if accepted in Step 1: write the migration and
`<features-dir>/snippets/{types.ts,index.ts}` per the example-feature
resource, and replace the guarded route's placeholder page with the
snippets list. **Gate:** `pnpm build && pnpm test` green with zero env
vars — the feature's own dormant check means it's never actually queried
in this state.

### Step 7 — Setup wizard
Write `scripts/setup-supabase.mjs` verbatim (adapt banner, default project
name, done-message port); add the `setup:supabase` script (never `setup`);
add the ESLint Node-globals block for `scripts/`; add `supabase/.temp` to
`.gitignore`. **Gate:** `pnpm lint` green and `node --check
scripts/setup-supabase.mjs` passes. Do NOT run the wizard.

### Step 8 — Headless dormant-state verification
With no env vars, drive the running app per verify-handoff Stage 1: guarded
route renders directly (no redirect); `/login` renders; submitting shows
the "not configured" error; other routes unaffected. **Gate:** flow
observed; lint, test, build all green with zero env vars.

### Step 9 — Hand off
Deliver the wizard-preview message from verify-handoff Stage 2. The user
runs `pnpm setup:supabase`. Wait.

### Step 10 — Verify live + close out
Walk the signed-in flow with the user per verify-handoff Stage 3 (redirect
round-trip, wrong-password error, sign-out, public-signup-is-actually-off,
and — if the example feature was included — the two-account isolation
proof). Update `docs/PLAN.md` and root `CLAUDE.md`; final lint/test/build;
commit, excluding `.env.local` and `supabase/.temp`.

## What the finished repo looks like

A delta over the existing app — `← new` is created, `← edited` is modified
in place. Shown with the optional `snippets` feature included; omit those
lines if it was declined.

```
your-app/
├── CLAUDE.md                             ← edited (current phase, commands, pattern-doc pointer)
├── .env.local.example                    ← new
├── .gitignore                             ← edited (supabase/.temp)
├── docs/
│   ├── FEATURE-MODULE-PATTERN.md          ← new
│   ├── SPEC.md                            ← edited (boundary re-weld)
│   └── PLAN.md                            ← edited (auth phase entry)
├── eslint.config.mjs                      ← edited (scripts/ Node globals block)
├── package.json                           ← edited (deps + setup:supabase script)
├── scripts/
│   └── setup-supabase.mjs                 ← new (the user runs this)
├── supabase/
│   └── migrations/
│       └── <timestamp>_create_user_snippets.sql   ← new (only if the example feature was included)
└── src/
    ├── proxy.ts                           ← new (Next 16's middleware entry point)
    ├── lib/
    │   └── supabase/
    │       ├── env.ts                     ← new
    │       ├── client.ts                  ← new
    │       ├── server.ts                  ← new
    │       ├── session.ts                 ← new
    │       └── middleware.ts              ← new
    ├── features/                          ← or wherever discovery found features live
    │   ├── auth/
    │   │   ├── CLAUDE.md                  ← new
    │   │   ├── types.ts                   ← new
    │   │   └── index.ts                   ← new (getCurrentUser(), the identity port)
    │   └── snippets/                      ← new, only if the example feature was included
    │       ├── types.ts
    │       └── index.ts
    └── app/
        ├── layout.tsx                     ← NOT edited — no client provider needed
        ├── login/
        │   ├── page.tsx                   ← new
        │   ├── login-form.tsx             ← new
        │   └── actions.ts                 ← new
        └── dashboard/                     ← or the chosen guarded route
            ├── layout.tsx                 ← new (defense-in-depth check + sign-out)
            ├── page.tsx                   ← new
            └── actions.ts                 ← new (signOut)
```

The code layer is verified headlessly; the account wiring, and the actual
proof of per-user isolation (if the example feature was included), belong
to the user's live verification.
