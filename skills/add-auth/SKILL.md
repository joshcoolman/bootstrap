---
name: add-auth
description: Add a single shared-credential Supabase email/password gate (not multi-user auth) to an existing Vite + React + TanStack Router app — a vendor-agnostic AuthClient seam, a /login route, a guarded /dashboard, and an interactive setup wizard the user runs themselves. Use when asked to add auth, add a login page, gate routes behind sign-in, or wire up Supabase auth in an existing Vite app.
---

You are about to add auth to the current repo — the one you are running in.
This skill assumes the stack (Vite + React + TanStack Router + pnpm) but not
the repo's shape: Step 0 discovers the shape and fills adaptation slots
(`<alias>`, `<features-dir>`, `<dev-port>`) that every later resource
references. Unlike a scaffolding skill, most steps here **edit existing
files** — add blocks and change lines, preserving what's there. You write all
the code, including the setup wizard; **the user runs the wizard** — you never
drive the interactive Supabase steps.

## Before you start — ask the user

1. **Confirm the target** — this repo? It must be a Vite + React + TanStack
   Router + pnpm app; Step 0 verifies and you stop if it isn't.
2. **Confirm Supabase** as the auth vendor (the only vendor this skill
   supports; the code seam keeps a second vendor cheap later), and that they
   have — or can create — a free Supabase account.
3. **Confirm the boundary** — this is a single shared-credential gate, not
   user management: any account that exists in the linked Supabase project
   gets identical access, because the app never checks *which* user is
   signed in. No signup UI, no password reset, no OAuth, no roles. Because of
   that, the setup wizard requires disabling public sign-ups in the Supabase
   dashboard as a mandatory security step — otherwise anyone who finds the
   app's public anon key could create their own account and pass the gate.
   (Password reset needs `detectSessionInUrl` + `PASSWORD_RECOVERY`
   handling — explicitly out of scope.)
4. **Protected route** — default is a new empty `/dashboard`; or should an
   existing route be gated instead?

## Resources — read these in order

- [Discovery](resources/discovery.md) — the Step-0 read-the-repo pass: preflight, probes, adaptation slots
- [Docs re-weld](resources/docs-reweld.md) — updating the docs that currently say "no auth"; the scope-question ritual
- [Auth feature](resources/auth-feature.md) — the vendor-agnostic contract, adapters, provider, tests, env scaffolding (verbatim code)
- [Routes](resources/routes.md) — login + guarded route + root mount; the routeTree.gen quirk
- [Setup wizard](resources/setup-wizard.md) — scripts/setup.mjs verbatim, ESLint globals, and the CLI-drift ledger
- [Verify + handoff](resources/verify-handoff.md) — the headless gate, the handoff message, live verification, close-out

## Build order

Execute each step fully and pass its gate before moving to the next.

### Step 0 — Discover the repo
Run the discovery pass. **Gate:** preflight confirms the stack; the discovery
summary (alias, features dir, dev port, styling idiom, docs to re-weld,
existing routes, CI shape) is reported to the user.

### Step 1 — Re-weld the docs boundary
Edit every doc that asserts "no auth" per the docs re-weld resource; settle
the scope questions. **Gate:** no doc still claims "no auth"; lint green.

### Step 2 — Deps + env scaffolding
`pnpm add @supabase/supabase-js`, `pnpm add -D @inquirer/prompts`; write
`.env.example` and `src/env.d.ts`; ensure `.gitignore` covers `.env`,
`.env.local`, and `supabase/.temp`. **Gate:** install succeeds; lint, test,
build still green with zero env vars.

### Step 3 — Contract + mock + tests
Write the feature boundary doc, `types.ts`, `mock-adapter.ts`, and the
contract test suite in `<features-dir>`. **Gate:** `pnpm test` green with zero
env vars — the seam is proven before any vendor code exists.

### Step 4 — Supabase adapter + provider + public index
Write `supabase-adapter.ts`, `auth-context.ts`, `auth-provider.tsx`,
`index.ts`, and the provider + no-env adapter test suites. **Gate:** full test
suite green AND `pnpm build` green with **no** `.env.local` present — this
simulates CI exactly.

### Step 5 — Routes + root mount
Write `login.tsx` and the guarded route (logic verbatim, markup restyled to
the repo's idiom); wrap `__root.tsx` in `<AuthProvider>`. Run `pnpm dev` once
to regenerate `src/routeTree.gen.ts`. **Gate:** `pnpm build` green.

### Step 6 — Setup wizard
Write `scripts/setup.mjs` verbatim (adapt banner, default project name,
done-message port); add the `setup:supabase` script (never `setup` — pnpm
built-in); add the ESLint Node-globals block for `scripts/`. **Gate:**
`pnpm lint` green and `node --check scripts/setup.mjs` passes. Do NOT run the
wizard.

### Step 7 — Headless unconfigured-state verification
With no env vars, drive the running app: protected route → redirect to
`/login?redirect=…` → form renders → submit → "not configured" error; existing
routes unaffected. **Gate:** flow observed; lint, test, build all green with
zero env.

### Step 8 — Hand off
Deliver the wizard-preview message from the verify-handoff resource. The user
runs `pnpm setup:supabase`. Wait.

### Step 9 — Verify live + close out
Walk the signed-in flow with the user (redirect round-trip, hard-reload
persistence, wrong-password error, sign-out). Update the PLAN doc and root
`CLAUDE.md`; final lint/test/build; commit **including the regenerated
`routeTree.gen.ts`**, excluding `.env.local` and `supabase/.temp`.

## What the finished repo looks like

A delta over the existing app — `← new` is created, `← edited` is modified in
place. The auth feature lands wherever discovery found the feature convention.

```
your-app/
├── CLAUDE.md                        ← edited (current phase, commands)
├── .env.example                     ← new
├── .gitignore                       ← edited (.env, supabase/.temp)
├── docs/                            ← edited (boundary re-weld: SPEC, PLAN, …)
├── eslint.config.js                 ← edited (scripts/ Node globals block)
├── package.json                     ← edited (deps + setup:supabase script)
├── scripts/
│   └── setup.mjs                    ← new (the user runs this)
└── src/
    ├── env.d.ts                     ← new
    ├── routeTree.gen.ts             ← regenerated (must be committed)
    ├── features/                    ← or wherever features live in this repo
    │   └── auth/
    │       ├── CLAUDE.md            ← new (boundary doc, repo's format)
    │       ├── types.ts             ← new (the AuthClient contract)
    │       ├── supabase-adapter.ts  ← new (the only file that knows Supabase)
    │       ├── mock-adapter.ts      ← new (test double)
    │       ├── auth-context.ts      ← new
    │       ├── auth-provider.tsx    ← new
    │       ├── auth.test.tsx        ← new
    │       └── index.ts             ← new (public surface)
    └── routes/
        ├── __root.tsx               ← edited (AuthProvider wrap)
        ├── login.tsx                ← new
        └── dashboard.tsx            ← new (or the chosen protected route)
```

The code layer is verified; the account wiring belongs to the user's wizard
run.
