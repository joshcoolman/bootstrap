# Routes

Two new route files plus one edit to the root route. **The logic is verbatim —
the skin is not.** Copy the control flow exactly (validateSearch, the
`beforeLoad` guard, the `!loading` gates, `router.history.push`); restyle the
markup and classNames to the styling idiom discovery found. Replace
`#/features/auth` with `<alias>` + the feature's public index path.

Design rationale — why it's built this way:

- **Guard via `beforeLoad` + module import, NOT router context.** The guard
  runs before any protected code renders (no flash). Router context
  (`createRootRouteWithContext`) would be a second injection seam duplicating
  what the feature's `getAuthClient()` already provides — skip it.
- **`beforeLoad` stays in the eager route module** under autoCodeSplitting, so
  importing the feature pulls supabase-js (~30 kB gz) into the eager graph.
  Acceptable for most apps; the escape hatch if bundle size ever matters is a
  dynamic `const { getAuthClient } = await import('...')` inside `beforeLoad`.
- **`?redirect=` carries the original destination** (`location.href`, a raw
  path+search string) through the login round-trip. Navigate back with
  `router.history.push(redirect)` — a typed `to:` can't round-trip an
  arbitrary href.
- **Every redirect decision gates on `!loading`.** The login page's
  already-signed-in effect and the dashboard's backstop both wait for the
  initial session check; skipping this causes redirect bounces on load.
- **The dashboard keeps a backstop effect** because `beforeLoad` only runs on
  navigation — a session that dies while the user is parked on the page
  (signed out elsewhere, token revoked) is caught by the `!loading && !user`
  effect.
- **No redirect loops by construction:** login navigates only after `signIn`
  resolves, at which point the session is already persisted, so the guard
  passes immediately on arrival.

## `src/routes/login.tsx`

```tsx
import { createFileRoute, useNavigate, useRouter } from '@tanstack/react-router'
import { useEffect, useState, type FormEvent } from 'react'

import { useAuth } from '#/features/auth'

export const Route = createFileRoute('/login')({
  validateSearch: (search: Record<string, unknown>) => ({
    redirect: typeof search.redirect === 'string' ? search.redirect : undefined,
  }),
  component: LoginPage,
})

function LoginPage() {
  const { user, loading, signIn } = useAuth()
  const { redirect } = Route.useSearch()
  const navigate = useNavigate()
  const router = useRouter()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState<string | null>(null)
  const [submitting, setSubmitting] = useState(false)

  const destination = redirect ?? '/dashboard'

  useEffect(() => {
    if (!loading && user) void router.history.push(destination)
  }, [loading, user, destination, router])

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault()
    setSubmitting(true)
    setError(null)
    const result = await signIn(email, password)
    setSubmitting(false)
    if (result.error) {
      setError(result.error)
      return
    }
    void router.history.push(destination)
  }

  return (
    <main className="bg-bg text-text flex min-h-screen flex-col items-center justify-center px-6">
      <form
        onSubmit={handleSubmit}
        className="border-border bg-surface flex w-full max-w-sm flex-col gap-4 rounded-md border p-8"
      >
        <h1 className="house-section">Sign in</h1>

        <label className="flex flex-col gap-1">
          <span className="text-text-muted font-mono text-xs">email</span>
          <input
            type="email"
            required
            autoComplete="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="border-border bg-bg text-text focus:border-accent rounded-md border px-3 py-2 text-sm outline-none"
          />
        </label>

        <label className="flex flex-col gap-1">
          <span className="text-text-muted font-mono text-xs">password</span>
          <input
            type="password"
            required
            autoComplete="current-password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
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
          disabled={submitting}
          className="bg-accent text-accent-contrast rounded-md px-3 py-2 text-sm transition-opacity hover:opacity-90 disabled:opacity-50"
        >
          {submitting ? 'Signing in…' : 'Sign in'}
        </button>

        <button
          type="button"
          onClick={() => void navigate({ to: '/' })}
          className="text-text-muted font-mono text-xs underline underline-offset-4"
        >
          ← back home
        </button>
      </form>
    </main>
  )
}
```

Keep: `role="alert"` on the error (accessibility + testability),
`autoComplete` attributes, the disabled-while-submitting button. Restyle: all
classNames (the ones above are the Paper & Ink token idiom from the reference
app). No signup or forgot-password links — deliberately.

## `src/routes/dashboard.tsx` (or the protected route the user chose)

```tsx
import { createFileRoute, redirect, useNavigate } from '@tanstack/react-router'
import { useEffect } from 'react'

import { getAuthClient, useAuth } from '#/features/auth'

export const Route = createFileRoute('/dashboard')({
  beforeLoad: async ({ location }) => {
    const session = await getAuthClient().getSession()
    if (!session) {
      throw redirect({ to: '/login', search: { redirect: location.href } })
    }
  },
  component: DashboardPage,
})

function DashboardPage() {
  const { user, loading, signOut } = useAuth()
  const navigate = useNavigate()

  // beforeLoad only runs on navigation — this catches a session that dies
  // while the user is parked here (signed out elsewhere, token revoked).
  useEffect(() => {
    if (!loading && !user) void navigate({ to: '/login', search: { redirect: undefined } })
  }, [loading, user, navigate])

  return (
    <main className="bg-bg text-text flex min-h-screen flex-col items-center justify-center gap-4 px-6 text-center">
      <h1 className="house-section">Dashboard</h1>
      <p className="text-text-muted text-sm">
        Signed in{user?.email ? ` as ${user.email}` : ''}. Nothing here yet — this is where the
        first real feature lands.
      </p>
      <button
        type="button"
        onClick={() => void signOut()}
        className="text-accent font-mono text-xs underline decoration-[var(--accent-soft)] underline-offset-4 transition-colors hover:decoration-[var(--accent)]"
      >
        sign out →
      </button>
    </main>
  )
}
```

Type gotcha: because login's `validateSearch` declares the `redirect` key,
TanStack treats it as a required search param — any `navigate({ to: '/login' })`
must pass `search: { redirect: undefined }` or `tsc` fails.

## Root route edit — mount the provider

Edit the existing `src/routes/__root.tsx`, preserving whatever it already
renders; only wrap the tree in `<AuthProvider>`. Reference shape:

```tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'
import { ThemeToggle } from '#/components/theme-toggle'
import { AuthProvider } from '#/features/auth'

export const Route = createRootRoute({
  component: () => (
    <AuthProvider>
      <ThemeToggle />
      <Outlet />
    </AuthProvider>
  ),
})
```

## Gate — the routeTree.gen quirk (hard requirement)

`pnpm build` typically runs `tsc && vite build`, and **tsc runs before the
router plugin regenerates `routeTree.gen.ts`**. After creating the route
files:

1. Run `pnpm dev` once (a few seconds is enough) — this regenerates
   `src/routeTree.gen.ts` with the new routes.
2. Then `pnpm build` must pass.
3. The regenerated `routeTree.gen.ts` **must be committed**, or CI's
   type-check fails on routes it has never heard of.
