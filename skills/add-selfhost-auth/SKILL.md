---
name: add-selfhost-auth
description: Add self-hosted email/password auth to an existing Next.js App Router app — scrypt password hashing and an HMAC-signed session cookie you own, deny-by-default middleware, a /login route, and a guarded /dashboard. No vendor, no signup route, no database required. Use when asked to add a login, gate an app behind sign-in, protect an app so only you (or a short allowlist) can reach it, or add auth without Supabase/Clerk/an auth service.
---

You are about to add auth to the current repo — the one you are running in.
This skill assumes the stack (Next.js App Router + pnpm) but not the repo's
shape: Step 0 discovers it and fills adaptation slots (`<alias>`,
`<features-dir>`, `<dev-port>`) that later resources reference. Most steps
**edit existing files** — add blocks, change lines, preserve what's there.

Unlike `add-supabase-auth`, there is no vendor and no setup wizard to hand off.
Everything here is code you own, so you write and verify all of it. The only
thing the user runs is a script that prints a credential line for them to paste
into their env.

## What this is

**Allowlist auth.** A short, known set of people can sign in. Access is granted
out-of-band by an operator, never by the visitor.

- ~200 lines, all standard library — `node:crypto` scrypt for passwords, Web
  Crypto HMAC for the session cookie
- No auth service, no `users` signup flow, no OAuth, no password reset
- No database, by default

The design is lifted from a production app (`bootsy`), with two deliberate
changes noted in the auth-feature resource.

**Explicitly not in scope:** signup, password reset, email verification, OAuth,
roles, per-user data isolation. If an app grows a real user base, that is the
signal to adopt `better-auth` — not a reason to extend this. Say so plainly
rather than bolting features on.

## Before you start — ask the user

1. **Confirm the target** — this repo? Must be Next.js App Router + pnpm.
   Step 0 verifies and you stop if it isn't.
2. **Confirm the boundary** — allowlist auth, as above. Anyone in the allowlist
   gets identical access; the app does not distinguish between them beyond an
   opaque user id. There is no signup route, so a stranger cannot self-serve
   access even if they find the URL.
3. **Credential backend** — default `env`, which needs no database:

   | Backend | Credential lives in | Add a person by |
   | --- | --- | --- |
   | `env` *(default)* | `AUTH_USERS` env var | adding an entry, redeploying |
   | `db` | a `users` table in Postgres | running `pnpm auth:create-user` |

   Choose `env` unless the app already has Postgres. Migrating `env` → `db`
   later is additive and the session layer is unchanged.
4. **Protected route** — default is a new empty `/dashboard`; or should an
   existing route be gated instead?

## Resources — read these in order

- [Discovery](resources/discovery.md) — the Step-0 read-the-repo pass: preflight, probes, adaptation slots
- [Auth feature](resources/auth-feature.md) — the session layer, both credential backends, the user-add script, tests (verbatim code)
- [Middleware](resources/middleware.md) — deny-by-default gating and the Edge/Node import split
- [Routes](resources/routes.md) — login page, server actions, guarded route, sign-out
- [Verify + handoff](resources/verify-handoff.md) — driving the real flow in a browser, the handoff message, close-out

## Build order

Execute each step fully and pass its gate before moving to the next.

### Step 0 — Discover the repo
Run the discovery pass. **Gate:** preflight confirms Next.js App Router + pnpm;
the discovery summary (alias, features dir, dev port, styling idiom, existing
routes, CI shape, whether `DATABASE_URL` already exists) is reported.

### Step 1 — Env scaffolding
Add `AUTH_SESSION_SECRET` and (for `env` backend) `AUTH_USERS` to
`.env.example`; ensure `.gitignore` covers `.env` and `.env.local`. **Gate:**
lint, test, build still green with zero env vars set.

### Step 2 — Session layer + tests
Write `session.ts` and its test suite. This is the Edge-safe half: Web Crypto
only, no `node:*` imports. **Gate:** `pnpm test` green — sign/verify round-trip,
tampered signature rejected, tampered userId rejected, expired timestamp
rejected, malformed input returns null rather than throwing.

### Step 3 — Credentials + user-add script
Write `credentials.ts` for the chosen backend, the shared `hash.ts`, and
`scripts/add-user.mjs`. **Gate:** `pnpm test` green — correct password accepted,
wrong password rejected, unknown email rejected; `node --check` passes on the
script.

### Step 4 — Middleware
Write `proxy.ts` per the middleware resource. **Gate:** `pnpm build` green.
Verify the import split — middleware imports `./session` directly and the
build does not pull `node:crypto` into the Edge bundle.

### Step 5 — Routes
Write the login page, its server action, the guarded route, and sign-out.
Logic verbatim; markup restyled to the repo's idiom as found in Step 0.
**Gate:** `pnpm build` green.

### Step 6 — Verify live
Generate a credential, set it locally, and drive the real flow in a browser:
protected route redirects to `/login`; wrong password shows an error and
preserves the typed email; correct password lands on the protected route; hard
reload stays signed in; sign-out returns to `/login` and the protected route is
gated again. **Gate:** every step observed in a browser, not inferred from
tests.

### Step 7 — Hand off + close out
Deliver the credential-generation message from the verify-handoff resource.
Update the repo's docs and `CLAUDE.md`. Final lint/test/build; commit,
excluding `.env.local`.

## What the finished repo looks like

A delta over the existing app — `← new` is created, `← edited` is modified in
place. The auth feature lands wherever discovery found the feature convention.

```
your-app/
├── CLAUDE.md                     ← edited (auth boundary, commands)
├── .env.example                  ← edited (AUTH_SESSION_SECRET, AUTH_USERS)
├── .gitignore                    ← edited (.env, .env.local)
├── package.json                  ← edited (auth:add-user script)
├── scripts/
│   └── add-user.mjs              ← new (prints a credential line)
└── src/
    ├── proxy.ts                  ← new (deny-by-default middleware)
    ├── features/
    │   └── auth/
    │       ├── CLAUDE.md         ← new (boundary doc, repo's format)
    │       ├── session.ts        ← new (Edge-safe: Web Crypto only)
    │       ├── session.test.ts   ← new
    │       ├── hash.ts           ← new (scrypt, shared with the script)
    │       ├── credentials.ts    ← new (env or db backend)
    │       ├── credentials.test.ts ← new
    │       ├── current-user.ts   ← new (throws when unauthenticated)
    │       ├── actions.ts        ← new (login / logout server actions)
    │       └── index.ts          ← new (public surface — never Edge-imported)
    └── app/
        ├── login/page.tsx        ← new
        └── dashboard/page.tsx    ← new (or the chosen protected route)
```

## The two things most likely to go wrong

Both are called out again at their point of use, but know them going in.

1. **The Edge/Node split.** `proxy.ts` runs on the Edge runtime and must import
   `./session` directly — never `index.ts`, which re-exports `credentials.ts`
   and pulls in `node:crypto`. That breaks the Edge bundle. This skill keeps
   them in separate modules so the split is structural rather than a comment
   someone has to remember.

2. **Dots in the session value.** The cookie is
   `userId.issuedAtMs.signature`, split on `.`. Any userId containing a dot —
   an email, most obviously — breaks the parse. The `env` backend therefore
   derives an opaque id by hashing the email rather than using it directly.
