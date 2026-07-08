# Example feature — `snippets` (optional)

**Offered as a choice, never forced.** Ask during Step 1 of the build order
(alongside the other scope questions in `docs-reweld.md`): "Would you like
me to also scaffold a small example feature to prove the multi-tenant
isolation works end-to-end, or just the auth layer?" If declined, skip this
resource entirely — `auth-feature.md`, `middleware.md`, and `routes.md`
already deliver a complete, working identity layer with nothing further
required.

## Why `snippets`, not `todos`

The reference app's own example feature is called `todos`. Don't reuse
that name here: the person building on this skill has already built
similar demo features multiple times across reused Supabase projects, and
"todos" is exactly the kind of common name that collides with leftover
tables from a prior run. `snippets` (table `user_snippets`) is deliberately
distinctive — a tiny per-user list of saved text snippets. If discovery
finds that even this collides with something already in the target
Supabase project, ask for an alternative name rather than guessing (see
`discovery.md`).

## What it demonstrates

The same double enforcement as the reference app's `todos`: Postgres RLS
is the real enforcement (a row is only ever visible/writable by its owner,
regardless of what application code does or forgets to check), and an
explicit `.eq('user_id', ...)` filter in every query makes the scoping
visible in the code too — belt-and-suspenders, not redundant redundancy.

## `supabase/migrations/<timestamp>_create_user_snippets.sql`

```sql
create table if not exists public.user_snippets (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users (id) on delete cascade,
  body text not null,
  created_at timestamptz not null default now()
);

alter table public.user_snippets enable row level security;

-- The real enforcement: a row is only ever visible/writable by its owner,
-- regardless of what application code does or forgets to check.
create policy "user_snippets are private to their owner"
  on public.user_snippets
  for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

## `<features-dir>/snippets/types.ts`

```ts
export interface Snippet {
  readonly id: string
  readonly body: string
  readonly createdAt: string
}

export type SnippetsError = { readonly _tag: 'Unauthenticated' } | { readonly _tag: 'DbError'; message: string }
```

## `<features-dir>/snippets/index.ts`

Plain-TypeScript equivalent of the reference app's `TodosRepo` — each
function is scoped to the current user both explicitly (the `.eq('user_id',
...)` filter) and structurally (the table's RLS policy).

```ts
import 'server-only'
import { createClient } from '#/lib/supabase/server'
import { getCurrentUser } from '#/features/auth'
import type { Snippet, SnippetsError } from './types'

type Result<T> = { ok: true; value: T } | { ok: false; error: SnippetsError }

async function requireUserId(): Promise<{ ok: true; userId: string } | { ok: false; error: SnippetsError }> {
  const result = await getCurrentUser()
  if (!result.ok) return { ok: false, error: { _tag: 'Unauthenticated' } }
  return { ok: true, userId: result.session.userId }
}

export async function listSnippets(): Promise<Result<Snippet[]>> {
  const session = await requireUserId()
  if (!session.ok) return session
  const supabase = await createClient()
  const { data, error } = await supabase
    .from('user_snippets')
    .select('id, body, created_at')
    .eq('user_id', session.userId)
    .order('created_at', { ascending: true })
  if (error) return { ok: false, error: { _tag: 'DbError', message: error.message } }
  return {
    ok: true,
    value: (data ?? []).map((row): Snippet => ({ id: row.id, body: row.body, createdAt: row.created_at })),
  }
}

export async function createSnippet(body: string): Promise<Result<null>> {
  const session = await requireUserId()
  if (!session.ok) return session
  const supabase = await createClient()
  const { error } = await supabase.from('user_snippets').insert({ body, user_id: session.userId })
  if (error) return { ok: false, error: { _tag: 'DbError', message: error.message } }
  return { ok: true, value: null }
}

export async function removeSnippet(id: string): Promise<Result<null>> {
  const session = await requireUserId()
  if (!session.ok) return session
  const supabase = await createClient()
  const { error } = await supabase.from('user_snippets').delete().eq('id', id).eq('user_id', session.userId)
  if (error) return { ok: false, error: { _tag: 'DbError', message: error.message } }
  return { ok: true, value: null }
}
```

## `src/app/dashboard/page.tsx` — replaces the placeholder from `routes.md`

Checks `isSupabaseConfigured` itself before ever calling into the feature —
same dormant-mode discipline as the auth layer: with no Supabase project,
there's no `user_snippets` table to query, so show a "set up Supabase"
state rather than attempting (and failing) a live call with the dev
session.

```tsx
import { isSupabaseConfigured } from '#/lib/supabase/env'
import { listSnippets } from '#/features/snippets'
import { SnippetsList } from './snippets-list'

export default async function DashboardPage() {
  if (!isSupabaseConfigured) {
    return (
      <main className="bg-bg text-text flex flex-1 flex-col items-center justify-center gap-4 px-6 text-center">
        <h1 className="house-section">Dashboard</h1>
        <p className="text-text-muted text-sm">
          Signed in. Set up Supabase (pnpm setup:supabase) to try the per-user snippets demo.
        </p>
      </main>
    )
  }

  const result = await listSnippets()
  const snippets = result.ok ? result.value : []

  return (
    <main className="bg-bg text-text flex flex-1 flex-col items-center gap-6 px-6 py-12">
      <h1 className="house-section">Your snippets</h1>
      <SnippetsList initialSnippets={snippets} />
    </main>
  )
}
```

`SnippetsList` (a small client component with a text input calling a
`createSnippet` server action and a delete button per row) is left as an
implementation detail to write in the repo's own idiom at build time — the
architecturally load-bearing part is `index.ts`'s double enforcement, not
this UI.

## The live multi-tenant proof (see `verify-handoff.md` Stage 3)

This is the one thing this skill's own gate cannot verify headlessly — it
needs a real Supabase project. When the user runs the setup wizard and
creates two accounts (see `setup-wizard.md`), walk this with them: sign in
as account A, create a snippet, sign out, sign in as account B, confirm
account B's list is empty and cannot see account A's snippet.

## Gate

Headless (zero env vars): `pnpm build && pnpm test && pnpm lint` green —
the dashboard page's `isSupabaseConfigured` check means the feature is
never actually queried in this state, so nothing here can fail for lack of
a database. Live (requires a real Supabase project, see
`verify-handoff.md`): two distinct accounts each see only their own rows.
