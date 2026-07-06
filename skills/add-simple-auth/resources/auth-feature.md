# Auth feature

The vendor-agnostic core. Everything in this resource is **verbatim code** —
it was validated end-to-end in a real app, and several gotchas live only in
the comments, so copy the comments too. The only adaptation is *placement*:
the files go wherever discovery found the feature convention
(`<features-dir>`, e.g. `src/features/auth/`). All imports between these files
are relative, so the folder is portable as-is.

Why the design is the way it is — do not "simplify" these away:

- **The adapter is lazy and env-tolerant.** CI runs with zero secrets, and a
  fresh clone has no `.env.local`. A module-top-level `createClient(...)` that
  throws on missing env breaks CI *and* blanks the whole app. Unconfigured
  behavior: signed-out everywhere, and `signIn` returns a helpful error.
- **Env access is literal-property only** (`import.meta.env.VITE_SUPABASE_URL`)
  — Vite does not statically replace dynamic keys.
- **Stale-session verification, once per page load.** The persisted
  localStorage session can outlive its user (deleted account, wizard re-run
  against a different Supabase project). The first `getSession()` after load
  verifies with `getUser()` and force-signs-out if invalid; later calls stay
  local so route guards never pay a network round-trip per navigation.
- **Never `await` other auth calls inside an `onAuthStateChange` callback** —
  supabase-js holds a lock while dispatching and deadlocks.
- **Errors cross the contract as plain strings**, not vendor error objects —
  keeps the seam serializable and vendor-neutral.
- **The mock adapter is the proof the seam works** — the contract tests run
  against it with no network and no vendor SDK. A second vendor later is a
  second adapter file; nothing else changes.

## Dependencies and scaffolding (before the feature files)

- `pnpm add @supabase/supabase-js`
- `pnpm add -D @inquirer/prompts` (used by the setup wizard, dev-only)
- Ensure `.gitignore` covers `.env.local` (a `*.local` glob counts) and add a
  plain `.env` line if absent.
- Write `.env.example` at the repo root:

```bash
# Copy of what `pnpm setup:supabase` writes to .env.local — run that wizard
# instead of filling this in by hand.

# Client-exposed (VITE_ prefix = bundled into the app; both are public-safe).
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key

# Scripts only — no VITE_ prefix, so Vite can never bundle it. NEVER expose.
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

- Write `src/env.d.ts` (types the vars as `string | undefined`, which forces
  the env guard at the type level too):

```ts
interface ImportMetaEnv {
  readonly VITE_SUPABASE_URL?: string
  readonly VITE_SUPABASE_ANON_KEY?: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

## `types.ts` — the contract

```ts
// The vendor-agnostic auth contract. Everything outside this feature (and the
// provider/routes inside it) programs against these types only — vendor SDKs
// stay inside adapter files.

export interface AuthUser {
  id: string
  email: string | null
}

export interface AuthSession {
  user: AuthUser
  /** Epoch seconds; null if the vendor doesn't report expiry. */
  expiresAt: number | null
}

export interface SignInResult {
  session: AuthSession | null
  /** Human-readable message; null on success. */
  error: string | null
}

export type Unsubscribe = () => void

export interface AuthClient {
  signIn(email: string, password: string): Promise<SignInResult>
  signOut(): Promise<void>
  getSession(): Promise<AuthSession | null>
  onAuthStateChange(callback: (session: AuthSession | null) => void): Unsubscribe
}
```

## `mock-adapter.ts` — the test double

```ts
// In-memory AuthClient for tests — no network, no vendor SDK. Deliberately not
// exported from index.ts; it exists to test the contract and the provider.

import type { AuthClient, AuthSession, Unsubscribe } from './types'

export interface MockCredentials {
  email: string
  password: string
}

export function createMockAuthClient(
  valid: MockCredentials,
  options: { initialSession?: AuthSession | null } = {},
): AuthClient {
  let session: AuthSession | null = options.initialSession ?? null
  const listeners = new Set<(session: AuthSession | null) => void>()

  const emit = () => {
    for (const listener of listeners) listener(session)
  }

  return {
    async signIn(email, password) {
      if (email !== valid.email || password !== valid.password) {
        return { session: null, error: 'Invalid login credentials' }
      }
      session = {
        user: { id: 'mock-user-id', email },
        expiresAt: null,
      }
      emit()
      return { session, error: null }
    },

    async signOut() {
      session = null
      emit()
    },

    async getSession() {
      return session
    },

    onAuthStateChange(callback): Unsubscribe {
      listeners.add(callback)
      return () => listeners.delete(callback)
    },
  }
}
```

## `supabase-adapter.ts` — the one file that knows about Supabase

Adapt point: the `NOT_CONFIGURED` message names the wizard script — keep it in
sync with the `package.json` script name.

```ts
// The one file that knows about Supabase. Lazy and env-tolerant: with no
// VITE_SUPABASE_* env vars (CI, fresh clone) nothing throws — the app just
// behaves signed-out and signIn explains how to configure.

import { createClient, type Session, type SupabaseClient } from '@supabase/supabase-js'

import type { AuthClient, AuthSession } from './types'

const NOT_CONFIGURED = 'Auth is not configured — run pnpm setup:supabase, then restart pnpm dev.'

let supabase: SupabaseClient | null = null

function getSupabase(): SupabaseClient | null {
  // Literal property access only — Vite doesn't replace dynamic env keys.
  const url = import.meta.env.VITE_SUPABASE_URL
  const key = import.meta.env.VITE_SUPABASE_ANON_KEY
  if (!url || !key) return null
  return (supabase ??= createClient(url, key))
}

function toSession(session: Session | null): AuthSession | null {
  if (!session) return null
  return {
    user: { id: session.user.id, email: session.user.email ?? null },
    expiresAt: session.expires_at ?? null,
  }
}

// The persisted localStorage session can outlive its user (deleted account,
// setup re-run against a different Supabase project). Verify with the server
// once per page load; every later getSession() stays local so route guards
// don't pay a network round-trip per navigation.
let verifiedThisLoad = false

export function createSupabaseAuthClient(): AuthClient {
  return {
    async signIn(email, password) {
      const client = getSupabase()
      if (!client) return { session: null, error: NOT_CONFIGURED }
      const { data, error } = await client.auth.signInWithPassword({ email, password })
      if (error) return { session: null, error: error.message }
      return { session: toSession(data.session), error: null }
    },

    async signOut() {
      await getSupabase()?.auth.signOut()
    },

    async getSession() {
      const client = getSupabase()
      if (!client) return null
      const { data } = await client.auth.getSession()
      if (!data.session) return null
      if (!verifiedThisLoad) {
        verifiedThisLoad = true
        const { error } = await client.auth.getUser()
        if (error) {
          await client.auth.signOut()
          return null
        }
      }
      return toSession(data.session)
    },

    onAuthStateChange(callback) {
      const client = getSupabase()
      if (!client) {
        callback(null)
        return () => {}
      }
      // Never await other auth calls inside this callback — supabase-js
      // holds a lock while dispatching and deadlocks.
      const { data } = client.auth.onAuthStateChange((_event, session) => {
        callback(toSession(session))
      })
      return () => data.subscription.unsubscribe()
    },
  }
}

let authClient: AuthClient | null = null

export function getAuthClient(): AuthClient {
  return (authClient ??= createSupabaseAuthClient())
}
```

## `auth-context.ts` — context + hook (split from the provider for fast-refresh/lint cleanliness)

```ts
import { createContext, useContext } from 'react'

import type { AuthSession, AuthUser, SignInResult } from './types'

export interface AuthContextValue {
  user: AuthUser | null
  session: AuthSession | null
  /** True until the initial session check resolves — gate redirects on this. */
  loading: boolean
  signIn: (email: string, password: string) => Promise<SignInResult>
  signOut: () => Promise<void>
}

export const AuthContext = createContext<AuthContextValue | null>(null)

export function useAuth(): AuthContextValue {
  const value = useContext(AuthContext)
  if (!value) throw new Error('useAuth must be used inside <AuthProvider>')
  return value
}
```

## `auth-provider.tsx`

The `loading` invariant matters: it resolves **only** via the initial
`getSession()` — auth-state events can carry the persisted-but-unverified
session before verification settles, so they update `session` but never
`loading`. Every redirect decision in the routes gates on `!loading`.

```tsx
import { useEffect, useMemo, useState, type ReactNode } from 'react'

import { AuthContext, type AuthContextValue } from './auth-context'
import { getAuthClient } from './supabase-adapter'
import type { AuthClient, AuthSession } from './types'

interface AuthProviderProps {
  /** Injectable for tests; defaults to the Supabase adapter singleton. */
  client?: AuthClient
  children: ReactNode
}

export function AuthProvider({ client, children }: AuthProviderProps) {
  const [auth] = useState<AuthClient>(() => client ?? getAuthClient())
  const [session, setSession] = useState<AuthSession | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    let cancelled = false
    void auth.getSession().then((initial) => {
      if (cancelled) return
      setSession(initial)
      setLoading(false)
    })
    // `loading` resolves only via getSession above — auth-state events can
    // carry the persisted-but-unverified session before verification settles.
    const unsubscribe = auth.onAuthStateChange((next) => {
      if (!cancelled) setSession(next)
    })
    return () => {
      cancelled = true
      unsubscribe()
    }
  }, [auth])

  const value = useMemo<AuthContextValue>(
    () => ({
      user: session?.user ?? null,
      session,
      loading,
      signIn: (email, password) => auth.signIn(email, password),
      signOut: () => auth.signOut(),
    }),
    [auth, session, loading],
  )

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}
```

## `index.ts` — the public surface

```ts
// Public surface of the auth feature. Consumers import from here only —
// never from the adapter or other internals.

export type { AuthClient, AuthSession, AuthUser, SignInResult, Unsubscribe } from './types'
export { getAuthClient } from './supabase-adapter'
export { AuthProvider } from './auth-provider'
export { useAuth, type AuthContextValue } from './auth-context'
```

## `auth.test.tsx` — contract + provider + CI-safety tests

Three suites: the contract via the mock (proves the seam), the provider/hook
with an injected mock (no network), and the Supabase adapter with env
explicitly stubbed empty — that last one proves the CI-safety property itself.
`vi.stubEnv` is used so the test stays correct even after a real `.env.local`
exists locally. Assumes Vitest + Testing Library + jsdom (the standard shell
setup); adapt imports if discovery found a different runner.

```tsx
import { act, render, screen } from '@testing-library/react'
import { describe, expect, it, vi } from 'vitest'

import { useAuth } from './auth-context'
import { AuthProvider } from './auth-provider'
import { createMockAuthClient } from './mock-adapter'
import { createSupabaseAuthClient } from './supabase-adapter'
import type { AuthClient, AuthSession } from './types'

const CREDS = { email: 'me@example.com', password: 'hunter22' }

describe('AuthClient contract (mock adapter)', () => {
  it('signs in with valid credentials', async () => {
    const client = createMockAuthClient(CREDS)
    const result = await client.signIn(CREDS.email, CREDS.password)
    expect(result.error).toBeNull()
    expect(result.session?.user.email).toBe(CREDS.email)
    expect(await client.getSession()).toEqual(result.session)
  })

  it('returns an error string, not a session, on bad credentials', async () => {
    const client = createMockAuthClient(CREDS)
    const result = await client.signIn(CREDS.email, 'wrong')
    expect(result.session).toBeNull()
    expect(result.error).toMatch(/invalid/i)
    expect(await client.getSession()).toBeNull()
  })

  it('signOut clears the session', async () => {
    const client = createMockAuthClient(CREDS)
    await client.signIn(CREDS.email, CREDS.password)
    await client.signOut()
    expect(await client.getSession()).toBeNull()
  })

  it('notifies subscribers on sign-in and sign-out, and stops after unsubscribe', async () => {
    const client = createMockAuthClient(CREDS)
    const seen: Array<AuthSession | null> = []
    const unsubscribe = client.onAuthStateChange((session) => seen.push(session))

    await client.signIn(CREDS.email, CREDS.password)
    await client.signOut()
    expect(seen).toHaveLength(2)
    expect(seen[0]?.user.email).toBe(CREDS.email)
    expect(seen[1]).toBeNull()

    unsubscribe()
    await client.signIn(CREDS.email, CREDS.password)
    expect(seen).toHaveLength(2)
  })

  it('does not notify on failed sign-in', async () => {
    const client = createMockAuthClient(CREDS)
    const listener = vi.fn()
    client.onAuthStateChange(listener)
    await client.signIn(CREDS.email, 'wrong')
    expect(listener).not.toHaveBeenCalled()
  })
})

function Probe() {
  const { user, loading } = useAuth()
  if (loading) return <p>loading</p>
  return <p>{user ? `signed in as ${user.email}` : 'signed out'}</p>
}

function renderWithProvider(client: AuthClient) {
  let auth!: ReturnType<typeof useAuth>
  function Capture() {
    auth = useAuth()
    return null
  }
  const view = render(
    <AuthProvider client={client}>
      <Probe />
      <Capture />
    </AuthProvider>,
  )
  return { view, auth: () => auth }
}

describe('AuthProvider / useAuth', () => {
  it('starts loading, then settles signed out', async () => {
    renderWithProvider(createMockAuthClient(CREDS))
    expect(screen.getByText('loading')).toBeInTheDocument()
    expect(await screen.findByText('signed out')).toBeInTheDocument()
  })

  it('restores an existing session', async () => {
    const client = createMockAuthClient(CREDS, {
      initialSession: { user: { id: 'u1', email: CREDS.email }, expiresAt: null },
    })
    renderWithProvider(client)
    expect(await screen.findByText(`signed in as ${CREDS.email}`)).toBeInTheDocument()
  })

  it('surfaces the user after signIn and clears it after signOut', async () => {
    const { auth } = renderWithProvider(createMockAuthClient(CREDS))
    await screen.findByText('signed out')

    await act(() => auth().signIn(CREDS.email, CREDS.password))
    expect(screen.getByText(`signed in as ${CREDS.email}`)).toBeInTheDocument()

    await act(() => auth().signOut())
    expect(screen.getByText('signed out')).toBeInTheDocument()
  })

  it('useAuth throws outside a provider', () => {
    const spy = vi.spyOn(console, 'error').mockImplementation(() => {})
    expect(() => render(<Probe />)).toThrow(/inside <AuthProvider>/)
    spy.mockRestore()
  })
})

describe('Supabase adapter without env vars', () => {
  // CI has no env and no network — the adapter must degrade, never throw.
  it('reports signed out and a helpful signIn error', async () => {
    vi.stubEnv('VITE_SUPABASE_URL', '')
    vi.stubEnv('VITE_SUPABASE_ANON_KEY', '')
    try {
      const client = createSupabaseAuthClient()
      expect(await client.getSession()).toBeNull()

      const result = await client.signIn('a@b.c', 'password')
      expect(result.session).toBeNull()
      expect(result.error).toMatch(/not configured/i)

      const listener = vi.fn()
      const unsubscribe = client.onAuthStateChange(listener)
      expect(listener).toHaveBeenCalledWith(null)
      expect(() => unsubscribe()).not.toThrow()

      await expect(client.signOut()).resolves.toBeUndefined()
    } finally {
      vi.unstubAllEnvs()
    }
  })
})
```

## The feature boundary doc

If discovery found a per-feature boundary-doc convention (e.g. a `CLAUDE.md`
in each feature folder), write one following the repo's format. Adapt the
"Read more" pointers to the repo's actual docs. Content to convey:

- **Owns**: the `AuthClient` contract; the Supabase adapter (lazy,
  env-tolerant); the React surface (`AuthProvider` + `useAuth`); the mock
  adapter (test double, not public).
- **Does NOT own**: route guarding (routes call `getAuthClient().getSession()`
  in `beforeLoad` themselves); user provisioning (`scripts/setup.mjs`);
  styling.
- **Key decisions**: consumers import only `index.ts`; the vendor appears in
  exactly one file — a second vendor is a second adapter, nothing else
  changes; errors cross as plain strings; the once-per-load stale-session
  verification; the onAuthStateChange deadlock rule.

## Gates

- After contract + mock + tests: `pnpm test` green **with zero env vars** —
  the seam is proven before any vendor code exists.
- After adapter + provider + index: full suite green (including the no-env
  adapter suite) AND `pnpm build` green with **no** `.env.local` present —
  this simulates CI exactly.
