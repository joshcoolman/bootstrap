# Middleware — session refresh + route gating

Genuinely new territory for this plugin: **no bootstrap skill has written
Next.js middleware before this one.** Ported verbatim (logic-wise) from
`~/repos/effective`'s `src/lib/supabase/middleware.ts` + `src/proxy.ts`.

## The Next.js 16 naming gotcha — read this first

**Next.js 16 renamed the middleware convention from `middleware.ts` to
`src/proxy.ts`, exporting a `proxy()` function instead of `middleware()`.**
`next-app` targets Next 16. It is very tempting to "fix" this back to the
familiar `middleware.ts`/`middleware()` shape out of habit — don't. Write
`src/proxy.ts` exactly as shown below. If a future Next version renames
this again, that's a real adaptation to make consciously, not a guess to
make now.

## `src/lib/supabase/middleware.ts`

```ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'
import { SUPABASE_PUBLISHABLE_KEY, SUPABASE_URL, isSupabaseConfigured } from './env'
import { getAuthClaims } from './session'

// Anyone may reach these without a session; everything else requires auth.
// Adapt point: add more public paths here (e.g. a marketing page) — never
// remove '/login' itself.
const PUBLIC_PATHS = ['/', '/login']

const isPublic = (pathname: string) =>
  PUBLIC_PATHS.includes(pathname) || pathname.startsWith('/login') || pathname.startsWith('/auth')

/**
 * Refreshes the Supabase session on every request (rotating the auth cookies)
 * and gates non-public routes. Pages redirect to /login; API routes get a 401.
 * No-ops entirely until Supabase is configured.
 */
export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({ request })

  if (!isSupabaseConfigured) return response

  const supabase = createServerClient(SUPABASE_URL!, SUPABASE_PUBLISHABLE_KEY!, {
    cookies: {
      getAll() {
        return request.cookies.getAll()
      },
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
        response = NextResponse.next({ request })
        cookiesToSet.forEach(({ name, value, options }) => response.cookies.set(name, value, options))
      },
    },
  })

  // IMPORTANT: nothing between createServerClient and getClaims — getClaims
  // verifies the JWT and, when needed, rotates the session cookies.
  const claims = await getAuthClaims(supabase)

  const { pathname } = request.nextUrl
  if (!claims && !isPublic(pathname)) {
    if (pathname.startsWith('/api')) {
      return NextResponse.json({ code: 'unauthorized' }, { status: 401 })
    }
    const loginUrl = request.nextUrl.clone()
    loginUrl.pathname = '/login'
    loginUrl.searchParams.set('redirect', pathname)
    return NextResponse.redirect(loginUrl)
  }

  return response
}
```

## `src/proxy.ts`

```ts
import { type NextRequest } from 'next/server'
import { updateSession } from '#/lib/supabase/middleware'

export async function proxy(request: NextRequest) {
  return updateSession(request)
}

export const config = {
  // Run on everything except Next internals and static image assets.
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'],
}
```

## Why each piece is the way it is

- **Page-vs-API branching** — an unauthenticated page request gets a
  redirect to `/login?redirect=<original path>` (so the login action can
  send the user back where they were headed); an unauthenticated API
  request gets a plain `401 { code: 'unauthorized' }` JSON body — a browser
  redirect makes no sense for a `fetch()` call.
- **`?redirect=` carries only the pathname**, not a full href — combined
  with the open-redirect guard in `routes.md`'s login action
  (`startsWith('/') && !startsWith('//')`), this prevents the redirect
  param from ever sending a signed-in user somewhere off-site.
- **The matcher excludes static assets** — running the session-refresh
  logic on every image/favicon/`_next/static` request would be pure
  overhead with no auth decision to make.
- **No-ops entirely when `!isSupabaseConfigured`** — this is what makes the
  zero-env-var CI gate possible: with no Supabase project, every route
  (including ones that would otherwise be gated) renders normally, because
  the middleware never even constructs a Supabase client.

## Gate

With zero env vars: every route (including the guarded one, once
`routes.md`'s files exist) renders without a redirect loop or a crash —
`isSupabaseConfigured` is false so `updateSession` returns immediately.
`pnpm build` succeeds (Next 16 type-checks `proxy.ts`'s exports as part of
the build). With a real Supabase project configured (live verification
only, not headless): visiting a non-public page while signed out redirects
to `/login?redirect=...`; visiting a non-public `/api/*` route while signed
out returns a 401 JSON body, not a redirect.
