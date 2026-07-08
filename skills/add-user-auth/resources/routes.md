# Routes — /login, guarded route, sign-out

Ported from `~/repos/effective`'s `src/app/login/*` and
`src/app/dashboard/{actions.ts,layout.tsx}`. **The control flow is
verbatim; the markup is not** — restyle classNames to whatever discovery
found, though in practice `next-app` and the reference app share the same
Paper & Ink token classes (`bg-bg`, `text-text`, `border-border`,
`bg-surface`, `text-accent`, …), so little should need to change.

## No root layout edit needed — unlike `add-simple-auth`

`add-simple-auth` edits `__root.tsx` to wrap the tree in a client
`<AuthProvider>`. **This skill does not touch `src/app/layout.tsx` at
all.** There's no client session context to mount — auth state is derived
server-side, per request, wherever it's needed. Leave `<ThemeInit />` /
`<ThemeToggle />` and everything else in the existing root layout exactly
as `next-app` scaffolded it.

## `src/app/login/page.tsx`

```tsx
import { LoginForm } from './login-form'

export default async function LoginPage({
  searchParams,
}: {
  searchParams: Promise<{ redirect?: string }>
}) {
  const { redirect } = await searchParams

  return (
    <main className="bg-bg text-text flex min-h-screen flex-col items-center justify-center px-6">
      <LoginForm redirectTo={redirect ?? '/dashboard'} />
    </main>
  )
}
```

(The reference app also renders a `<ForceDark />` component here to pin the
login page to dark mode regardless of the app's theme state — that
component doesn't exist in `next-app`. Skip it; `next-app`'s
`data-theme="dark"` default plus its own theme toggle already cover this,
and inventing a new component to force a theme is out of scope for this
skill.)

## `src/app/login/login-form.tsx`

```tsx
'use client'

import { useActionState } from 'react'
import { login } from './actions'

export function LoginForm({ redirectTo }: { redirectTo: string }) {
  const [error, formAction, pending] = useActionState(login, null)

  return (
    <form
      action={formAction}
      className="border-border bg-surface flex w-full max-w-sm flex-col gap-4 rounded-md border p-8"
    >
      <h1 className="house-section">Sign in</h1>

      <input type="hidden" name="redirect" value={redirectTo} />

      <label className="flex flex-col gap-1">
        <span className="text-text-muted font-mono text-xs">email</span>
        <input
          type="email"
          name="email"
          required
          autoComplete="email"
          className="border-border bg-bg text-text focus:border-accent rounded-md border px-3 py-2 text-sm outline-none"
        />
      </label>

      <label className="flex flex-col gap-1">
        <span className="text-text-muted font-mono text-xs">password</span>
        <input
          type="password"
          name="password"
          required
          autoComplete="current-password"
          className="border-border bg-bg text-text focus:border-accent rounded-md border px-3 py-2 text-sm outline-none"
        />
      </label>

      {error && (
        <p role="alert" className="text-accent font-mono text-xs">
          {error}
        </p>
      )}

      <button
        type="submit"
        disabled={pending}
        className="bg-accent text-accent-contrast rounded-md px-3 py-2 text-sm transition-opacity hover:opacity-90 disabled:opacity-50"
      >
        {pending ? 'Signing in…' : 'Sign in'}
      </button>
    </form>
  )
}
```

Keep: `role="alert"` on the error, `autoComplete` attributes, the
disabled-while-submitting button, the hidden `redirect` field. No signup or
forgot-password links — deliberately (see `docs-reweld.md`'s scope
questions).

## `src/app/login/actions.ts`

Adapt point: the `NOT_CONFIGURED` message should name the wizard script —
keep in sync with `setup-wizard.md`'s `package.json` script name.

```ts
'use server'

import { redirect } from 'next/navigation'
import { createClient } from '#/lib/supabase/server'
import { isSupabaseConfigured } from '#/lib/supabase/env'

/** Sign in with email + password. There is no signup — users are provisioned
 *  out of band (Supabase dashboard / admin API) and public signup is disabled. */
export async function login(_prev: string | null, formData: FormData): Promise<string | null> {
  if (!isSupabaseConfigured) return 'Auth is not configured — run pnpm setup:supabase, then restart pnpm dev.'

  const email = String(formData.get('email') ?? '')
  const password = String(formData.get('password') ?? '')
  // startsWith('/') alone would admit protocol-relative URLs like "//evil.com".
  const requested = String(formData.get('redirect') ?? '')
  const destination = requested.startsWith('/') && !requested.startsWith('//') ? requested : '/dashboard'

  const supabase = await createClient()
  const { error } = await supabase.auth.signInWithPassword({ email, password })
  if (error) return error.message

  redirect(destination)
}
```

## `src/app/dashboard/actions.ts` (or wherever the guarded route lives)

```ts
'use server'

import { redirect } from 'next/navigation'
import { createClient } from '#/lib/supabase/server'
import { isSupabaseConfigured } from '#/lib/supabase/env'

export async function signOut() {
  if (isSupabaseConfigured) {
    const supabase = await createClient()
    await supabase.auth.signOut()
  }
  redirect('/login')
}
```

## `src/app/dashboard/layout.tsx` — defense in depth

Middleware already gates this route (see `middleware.md`); this re-check
is deliberate belt-and-suspenders, not redundant — a Server Component
render can't assume the request that reached it was actually gated (e.g. a
misconfigured matcher), so it re-verifies.

```tsx
import { redirect } from 'next/navigation'
import { createClient } from '#/lib/supabase/server'
import { getAuthClaims } from '#/lib/supabase/session'
import { isSupabaseConfigured } from '#/lib/supabase/env'
import { signOut } from './actions'

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  // Middleware gates this route; re-check server-side as defense in depth.
  if (isSupabaseConfigured) {
    const supabase = await createClient()
    const claims = await getAuthClaims(supabase)
    if (!claims) redirect('/login')
  }

  return (
    <div className="flex min-h-dvh flex-col">
      <nav className="border-border flex items-center justify-end gap-4 border-b px-6 py-3">
        <form action={signOut}>
          <button
            type="submit"
            className="text-accent font-mono text-xs underline underline-offset-4"
          >
            sign out →
          </button>
        </form>
      </nav>
      {children}
    </div>
  )
}
```

`next-app` has no existing nav component to integrate with (unlike the
reference app's `GlobalNav`) — this inline sign-out form is intentionally
minimal. If the target repo's guarded route is something other than
`/dashboard`, adapt the path but keep the same layout/defense-in-depth
shape.

## `src/app/dashboard/page.tsx`

A minimal placeholder — the first real feature (or the optional
`snippets` example, see `example-feature.md`) replaces this:

```tsx
export default function DashboardPage() {
  return (
    <main className="bg-bg text-text flex flex-1 flex-col items-center justify-center gap-4 px-6 text-center">
      <h1 className="house-section">Dashboard</h1>
      <p className="text-text-muted text-sm">Signed in. Nothing here yet — this is where the first real feature lands.</p>
    </main>
  )
}
```

## Gate

`pnpm build` green. With zero env vars: visiting `/dashboard` renders (the
defense-in-depth check is skipped entirely while `!isSupabaseConfigured`,
matching middleware's own dormant no-op — see `verify-handoff.md` for why
this specific behavior, not a redirect, is the correct dormant state).
Visiting `/login` renders the form; submitting shows the "not configured"
error inline.
