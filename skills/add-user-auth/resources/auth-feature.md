# Auth feature — the identity layer

Ported from `~/repos/effective`'s `src/lib/supabase/*` and
`src/features/auth/*`, de-Effected to plain async/await (see "Why plain
TypeScript, not Effect" below). Everything here is **verbatim-adapted
code** — it was validated end-to-end in a live, deployed app; only the
`Effect.Service` machinery was rewritten, the actual logic (dormant-mode
degrade, cookie handling, claims verification) is unchanged. Placement:
`src/lib/supabase/` for the vendor client setup, `<features-dir>/auth/`
(e.g. `src/features/auth/`) for the identity port.

## Why the design is the way it is — do not "simplify" these away

- **Dormant by default, never throw.** `next-app`'s CI runs install → lint →
  test → build with **zero secrets**. A module-top-level `createClient(...)`
  that throws on missing env breaks CI and blanks the whole app. Every
  Supabase-touching file checks `isSupabaseConfigured` first and degrades to
  an observable "not configured" state instead.
- **No client-side `AuthProvider`/`useAuth`/React Context — deliberately,
  unlike `add-simple-auth`.** That skill targets a client-rendered Vite SPA,
  where the browser has no other way to know "am I signed in" except a
  persisted client session + an `onAuthStateChange` listener. Next.js App
  Router doesn't have that problem: session state lives in cookies, and
  Server Components/Actions can just re-check it on every request. Building
  a client Context here would be solving a problem App Router doesn't have
  — the entire `loading`/`onAuthStateChange`-deadlock/stale-session-verify
  class of complexity in `add-simple-auth`'s provider does not carry over,
  and should not be re-invented.
- **Env access is `process.env.NEXT_PUBLIC_*`**, not `import.meta.env.VITE_*`
  — Next's own build-time env replacement, needs no `env.d.ts` (Next already
  types `process.env` as `string | undefined` per key by default).
- **`getClaims()`, not `getUser()`** — verifies the JWT locally without a
  network round-trip per call site, and (per the Supabase SSR docs) rotates
  session cookies when needed. Every call site goes through one helper
  (`getAuthClaims`) so nobody hand-rolls the claims shape.
- **Nothing runs between `createServerClient` and `getClaims()` in
  middleware** — `getClaims()` is what triggers cookie rotation; inserting
  another Supabase call in between can miss it. This is a real gotcha from
  the reference app's own code comments — preserve the ordering.
- **Server Components can't set cookies.** `server.ts`'s `setAll` wraps the
  cookie write in a try/catch — middleware is what actually refreshes the
  session, so a Server Component's write attempt throwing is expected and
  safe to swallow.
- **The identity port (`getCurrentUser`) is the one place any feature asks
  "who is asking."** Features (e.g. the optional `snippets` example) depend
  on this, not on the UI/route layer passing a user id down. This is the
  crux of the enforcement discipline `docs/FEATURE-MODULE-PATTERN.md` (see
  `docs-reweld.md`) documents.
- **Errors cross the identity port as a fixed discriminated union**
  (`AuthError`), not exceptions — callers pattern-match on `_tag` rather
  than catching. UI-facing errors (e.g. a failed login) stay plain
  human-readable strings, same as `add-simple-auth` — that's a different
  seam (form feedback) from the internal identity-port taxonomy.

## Why plain TypeScript, not Effect

The reference app builds `CurrentUser`/`TodosRepo` on `Effect.Service`
(from the `effect` package). `next-app` has zero extra runtime dependencies
today, and this skill should not be the first thing to require one just to
port a pattern that has a straightforward plain-async equivalent. Everything
architecturally meaningful — an identity port features depend on rather
than reaching for `user.id` directly, RLS + explicit-filter double
enforcement, a fixed per-feature error taxonomy — survives the rewrite to
plain `async`/`await` functions returning a `Result`-shaped discriminated
union. If a consuming app already uses Effect elsewhere, porting this layer
onto `Effect.Service` is a mechanical follow-up, not a blocker to using this
skill.

## Dependencies (before the feature files)

```
pnpm add @supabase/ssr @supabase/supabase-js
pnpm add -D @inquirer/prompts
```

(`@inquirer/prompts` is used by the setup wizard, dev-only — see
`setup-wizard.md`.)

Ensure `.gitignore` covers `.env`, `.env.local` (`next-app`'s `*.local` glob
already covers this) and add `supabase/.temp` (the CLI's local link-state
directory — see `setup-wizard.md`).

Write `.env.local.example` at the repo root:

```bash
# Copy to .env.local and fill in (pnpm setup:supabase does this for you).
# Restart `pnpm dev` after changing.

# Supabase Auth (Project Settings → API). Publishable key, NOT the secret key.
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=
```

Only these two vars are needed at runtime — the secret/service-role key is
used only by the setup wizard script, never by the app itself, and never
gets a `NEXT_PUBLIC_` prefix (Next only exposes `NEXT_PUBLIC_*` vars to the
client bundle; the wizard's key stays server/script-only).

## `src/lib/supabase/env.ts`

```ts
// Auth is vendor-agnostic in spirit but Supabase is the first (only) adapter.
// These two public env vars are all the client needs; when either is missing
// the auth layer stays dormant (middleware no-ops, protected pages degrade to
// a "not configured" state) so the app runs before Supabase is provisioned.

export const SUPABASE_URL = process.env.NEXT_PUBLIC_SUPABASE_URL
export const SUPABASE_PUBLISHABLE_KEY = process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY

export const isSupabaseConfigured = Boolean(SUPABASE_URL && SUPABASE_PUBLISHABLE_KEY)
```

## `src/lib/supabase/client.ts` — browser client

Not called anywhere in this skill's own auth flow (sign-in/out both go
through server actions) — included because it's the standard third file of
Supabase's Next.js SSR setup, and a future feature that needs a browser-side
Supabase call (e.g. realtime subscriptions) will need it.

```ts
'use client'

import { createBrowserClient } from '@supabase/ssr'
import { SUPABASE_PUBLISHABLE_KEY, SUPABASE_URL } from './env'

/** Browser-side Supabase client for Client Components. */
export function createClient() {
  return createBrowserClient(SUPABASE_URL!, SUPABASE_PUBLISHABLE_KEY!)
}
```

## `src/lib/supabase/server.ts` — server client

```ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import { SUPABASE_PUBLISHABLE_KEY, SUPABASE_URL } from './env'

/**
 * Server-side Supabase client for Server Components, Route Handlers, and
 * Server Actions. Reads/writes the session through Next's cookie store.
 */
export async function createClient() {
  const cookieStore = await cookies()

  return createServerClient(SUPABASE_URL!, SUPABASE_PUBLISHABLE_KEY!, {
    cookies: {
      getAll() {
        return cookieStore.getAll()
      },
      setAll(cookiesToSet) {
        // Server Components can't set cookies; the middleware refreshes the
        // session so this throwing here is safe to ignore.
        try {
          cookiesToSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options))
        } catch {
          // called from a Server Component — ignore
        }
      },
    },
  })
}
```

## `src/lib/supabase/session.ts` — the one place that knows the claims shape

```ts
import type { SupabaseClient } from '@supabase/supabase-js'

/** The subset of Supabase's JWT claims the app actually uses. Extend here —
 *  not per call site — if a feature ever needs e.g. role or app_metadata. */
export interface AuthClaims {
  readonly userId: string
  readonly email: string | undefined
}

/**
 * The one place in the repo that knows the shape of Supabase's getClaims()
 * response. Every call site (middleware, the guarded route, getCurrentUser)
 * goes through this instead of hand-rolling it.
 */
export async function getAuthClaims(supabase: SupabaseClient): Promise<AuthClaims | null> {
  const { data } = await supabase.auth.getClaims()
  const claims = data?.claims
  return claims ? { userId: claims.sub, email: claims.email } : null
}
```

## `<features-dir>/auth/types.ts`

```ts
export interface Session {
  readonly userId: string
  readonly email: string | undefined
}

export type AuthError = { readonly _tag: 'Unauthenticated' }
```

## `<features-dir>/auth/index.ts` — the identity port, plain TypeScript

`import 'server-only'` is the explicit adaptation beyond de-Effecting —
`next-app`'s own `knowledge` feature establishes this convention (a
server-only module fails the build loudly if a Client Component ever
imports it, instead of silently trying to ship `next/headers` to the
browser). The reference app doesn't do this explicitly (its `server.ts`
filename signals intent instead), but it's a cheap, real safety net worth
adding.

```ts
// Server-only: the identity port — the seam between Supabase identity and
// any feature that needs to know who's asking. Dormant-mode (no Supabase
// configured) resolves to a fixed local session, matching middleware's
// no-op — but nothing should actually query real data in dormant mode; a
// feature should check isSupabaseConfigured itself and show a "set up
// Supabase" state before ever calling getCurrentUser (see example-feature.md).

import 'server-only'
import { createClient } from '#/lib/supabase/server'
import { getAuthClaims } from '#/lib/supabase/session'
import { isSupabaseConfigured } from '#/lib/supabase/env'
import type { AuthError, Session } from './types'

export type { AuthError, Session } from './types'

const DEV_SESSION: Session = { userId: 'local-dev', email: undefined }

export type GetCurrentUserResult = { ok: true; session: Session } | { ok: false; error: AuthError }

export async function getCurrentUser(): Promise<GetCurrentUserResult> {
  if (!isSupabaseConfigured) return { ok: true, session: DEV_SESSION }

  try {
    const supabase = await createClient()
    const claims = await getAuthClaims(supabase)
    if (!claims) return { ok: false, error: { _tag: 'Unauthenticated' } }
    return { ok: true, session: { userId: claims.userId, email: claims.email } }
  } catch {
    return { ok: false, error: { _tag: 'Unauthenticated' } }
  }
}
```

## The feature boundary doc — `<features-dir>/auth/CLAUDE.md`

Write one following the repo's `src/features/*/CLAUDE.md` format
(`next-app`'s `knowledge.md` shows the shape). Content to convey:

- **Owns**: `getCurrentUser()` (the identity port every feature depends on),
  the Supabase client setup in `src/lib/supabase/`, the `AuthError`
  taxonomy.
- **Does NOT own**: route guarding (that's `src/proxy.ts` + each guarded
  route's own defense-in-depth check — see `middleware.md`/`routes.md`);
  user provisioning (`scripts/setup-supabase.mjs`); UI/styling.
- **Key decisions**: server-rendered identity, no client Context; dormant
  mode resolves to a fixed dev session but nothing should query real data
  while dormant; a second vendor is a new `src/lib/<vendor>/` + a new
  `getCurrentUser()` implementation behind the same port, not a rewrite of
  every call site.

## Gate

`pnpm build && pnpm test && pnpm lint` all green with **zero env vars** — no
`.env.local` present. `isSupabaseConfigured` is false, `getCurrentUser()`
resolves to `DEV_SESSION` without throwing, and nothing in this step alone
should be reachable from a route yet (that's `middleware.md`/`routes.md`).
