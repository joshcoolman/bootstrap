# Part: docs

The docs folder structure plus the in-app `/docs` viewer. Same idea as
`vite-app`: the markdown files are the human/agent-readable record of intent,
and the viewer makes them reachable without leaving the app.

**This file is the one genuinely new, unvalidated piece of `next-app`.**
`vite-app`'s docs viewer leans on Vite's `import.meta.glob`, a bundler-level
API with no Next.js equivalent — the proven reference app hit this exact gap
during its migration and simply dropped the `/docs` route rather than solve
it. What follows is a first design, not a ported-and-confirmed one. If it
breaks or feels wrong in actual use, that's expected — report it as friction
rather than assuming it's a mistake in reading these instructions.

## The `docs/` folder

Same four standard files as `vite-app`, same purpose, written before
significant code lands:

### `docs/OVERVIEW.md`

The umbrella vision — the why behind the whole project. Where it fits in the
broader set of apps, what problem it solves, what it explicitly is not.
Written once, rarely updated.

### `docs/SPEC.md`

The what and why of this specific app. The welded boundary (what it does and
what it deliberately does not do), the key design decisions. Readable by a
non-coder.

### `docs/PLAN.md`

The concrete build order. Each phase has a clear output ("after this phase, X
is runnable"). Updated as phases complete. The primary orientation doc for
agents starting work.

### `docs/STYLE.md`

The style system contract — same content as `vite-app`'s, adapted to name
this skill's four touch-points (`shell.md`) instead of the Vite SPA's five.

## The in-app `/docs` viewer

A `/docs` route rendering all markdown from `docs/`, `knowledge/`,
`README.md`, and `CLAUDE.md` in a sidebar-nav reader — same feature set as
`vite-app`'s, different mechanism underneath.

### Dependencies

```
react-markdown
remark-gfm
```

### Why the split: a Server Component reading files, a Client Component rendering them

`import.meta.glob` did two jobs at once in `vite-app`: it read the files
*and* bundled them in, eagerly, at build time. Next has no single API that
does both. The replacement splits those jobs across the Server/Client
boundary App Router already has:

- A **Server Component** (`src/app/docs/page.tsx`) reads the files with
  Node's `fs` — this only runs on the server (build time for a static route
  like this one, since nothing here depends on the request), never ships to
  the browser.
- A **Client Component** (`src/components/docs-viewer.tsx`) receives the
  already-read docs as a plain serializable prop (an array of
  `{ id, title, section, order, content }` — all strings and numbers, safe to
  cross the Server→Client boundary) and owns the interactive part: sidebar
  nav, active-doc state, mobile drawer.

The classification logic (turning a file path into a title/section/order) is
identical in spirit to `vite-app`'s — it's pulled out into its own pure,
framework-agnostic function so it doesn't care whether the raw content came
from a Vite glob or a Node `fs` read.

### `src/features/docs/build-docs.ts`

```ts
export type Doc = { id: string; title: string; section: string; order: number; content: string }

const SECTION_ORDER = ['Start here', 'Knowledge', 'Working notes']

function titleCase(s: string): string {
  return s.charAt(0).toUpperCase() + s.slice(1)
}

function slugify(path: string): string {
  return path.replace(/^\//, '').replace(/\.md$/i, '').replace(/\//g, '-').toLowerCase()
}

function parseOrder(base: string): { order: number; name: string } {
  const m = base.match(/^(\d+)[-_.]\s*(.+)$/)
  return m ? { order: parseInt(m[1]!, 10), name: m[2]! } : { order: Infinity, name: base }
}

function sectionFor(path: string): string {
  if (path.startsWith('/knowledge/')) return 'Knowledge'
  if (path === '/CLAUDE.md') return 'Working notes'
  if (path === '/README.md' || path.startsWith('/docs/')) return 'Start here'
  return 'Working notes'
}

function classify(path: string): Omit<Doc, 'content'> {
  const segs = path.split('/')
  const base = segs.pop()!.replace(/\.md$/i, '')
  const { order, name } = parseOrder(base)
  const title = /^(README|CLAUDE)$/i.test(name) ? name : titleCase(name.replace(/-/g, ' '))
  return { id: slugify([...segs, name].join('/')), section: sectionFor(path), title, order }
}

export function buildDocs(raw: Record<string, string>): Doc[] {
  return Object.entries(raw)
    .map(([path, content]) => ({ content, ...classify(path) }))
    .sort((a, b) => a.order - b.order || a.title.localeCompare(b.title))
}

export function groupBySection(docs: Doc[]): { section: string; docs: Doc[] }[] {
  const bySection = new Map<string, Doc[]>()
  for (const d of docs) {
    const list = bySection.get(d.section) ?? []
    list.push(d)
    bySection.set(d.section, list)
  }
  return SECTION_ORDER.filter((s) => bySection.has(s)).map((section) => ({
    section,
    docs: bySection.get(section) ?? [],
  }))
}
```

Byte-for-byte the same classification rules as `vite-app`'s docs route
(`/knowledge/` → "Knowledge", `/CLAUDE.md` → "Working notes", `/README.md`
and `/docs/` → "Start here", optional `NN-` filename prefix for manual
ordering) — only the input shape changed, from a glob result to a plain
`Record<string, string>`.

### `src/app/docs/page.tsx`

Reads the four sources from disk. `readdirSync(dir, { recursive: true })`
needs Node 18.17+ — already required by Next 16, so no new constraint:

```tsx
import { existsSync, readdirSync, readFileSync } from 'node:fs'
import { join } from 'node:path'
import { buildDocs } from '#/features/docs/build-docs'
import { DocsViewer } from '#/components/docs-viewer'
import 'server-only'

function readIfExists(path: string): string | null {
  return existsSync(path) ? readFileSync(path, 'utf-8') : null
}

function readMarkdownDir(dirName: string): Record<string, string> {
  const root = join(process.cwd(), dirName)
  if (!existsSync(root)) return {}
  const entries: Record<string, string> = {}
  for (const file of readdirSync(root, { recursive: true }) as string[]) {
    if (!file.endsWith('.md')) continue
    entries[`/${dirName}/${file}`] = readFileSync(join(root, file), 'utf-8')
  }
  return entries
}

export default function DocsPage() {
  const raw: Record<string, string> = {}

  const readme = readIfExists(join(process.cwd(), 'README.md'))
  if (readme) raw['/README.md'] = readme

  const claude = readIfExists(join(process.cwd(), 'CLAUDE.md'))
  if (claude) raw['/CLAUDE.md'] = claude

  Object.assign(raw, readMarkdownDir('docs'))
  Object.assign(raw, readMarkdownDir('knowledge'))

  return <DocsViewer docs={buildDocs(raw)} />
}
```

No `'use client'` here — this is a Server Component by default, which is
exactly what lets it call `node:fs` directly. Nothing here is
request-dependent, so Next renders it once at build time like any other
static route — the docs shown are whatever was in the repo at build time,
same as `vite-app`'s build-time glob.

### `src/components/docs-viewer.tsx`

Owns all interactivity — same layout, same behavior as `vite-app`'s
`DocsPage`, just fed `docs` as a prop instead of computing them from a glob:

```tsx
'use client'

import { useState } from 'react'
import Link from 'next/link'
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import { Menu, X } from 'lucide-react'
import { appMeta } from '#/app-meta'
import { groupBySection, type Doc } from '#/features/docs/build-docs'

export function DocsViewer({ docs }: { docs: Doc[] }) {
  const sections = groupBySection(docs)
  const defaultId = sections[0]?.docs[0]?.id ?? docs[0]?.id ?? ''
  const [activeId, setActiveId] = useState(() => {
    if (typeof window === 'undefined') return defaultId
    const fromHash = window.location.hash.replace(/^#/, '')
    return docs.some((d) => d.id === fromHash) ? fromHash : defaultId
  })
  const [navOpen, setNavOpen] = useState(false)

  function select(id: string) {
    setActiveId(id)
    setNavOpen(false)
    window.history.replaceState(null, '', `#${id}`)
    window.scrollTo({ top: 0 })
  }

  const active = docs.find((d) => d.id === activeId) ?? docs[0]
  if (!active) {
    return (
      <main className="bg-bg text-text-muted flex min-h-screen items-center justify-center">
        No documents found.
      </main>
    )
  }

  const nav = (
    <nav className="flex flex-col gap-7">
      {sections.map(({ section, docs: items }) => (
        <div key={section} className="flex flex-col gap-0.5">
          <p className="text-text-faint mb-2 px-3 font-mono text-[11px] font-semibold tracking-[0.12em] uppercase">
            {section}
          </p>
          {items.map((d) => (
            <button
              key={d.id}
              type="button"
              onClick={() => select(d.id)}
              className={`block w-full rounded-md px-3 py-1.5 text-left text-[13px] transition ${
                d.id === active.id
                  ? 'bg-surface-raised text-text font-medium'
                  : 'text-text-muted hover:text-text hover:bg-surface-sunken'
              }`}
            >
              {d.title}
            </button>
          ))}
        </div>
      ))}
    </nav>
  )

  return (
    <main className="bg-bg text-text flex min-h-screen">
      <aside className="border-border bg-surface sticky top-0 hidden h-screen w-[272px] shrink-0 overflow-y-auto border-r px-5 pt-9 pb-12 sm:block">
        <Link href="/" className="text-text-muted hover:text-accent mb-9 block px-3 font-mono text-xs tracking-tight transition-colors">
          {appMeta.name}
        </Link>
        {nav}
      </aside>
      <div className="min-w-0 flex-1">
        <article className="mx-auto max-w-[760px] px-6 pt-12 pb-28 sm:px-10 sm:pt-20">
          <div className="mb-10 flex items-center gap-3 sm:hidden">
            <button
              type="button"
              aria-label="Open docs menu"
              onClick={() => setNavOpen(true)}
              className="border-border text-text-body rounded-md border p-2"
            >
              <Menu size={16} />
            </button>
            <span className="text-text text-sm font-medium">{active.title}</span>
          </div>
          <div className="prose compact">
            <ReactMarkdown remarkPlugins={[remarkGfm]}>{active.content}</ReactMarkdown>
          </div>
        </article>
      </div>
      {navOpen && (
        <div className="fixed inset-0 z-50 bg-black/60 sm:hidden" onClick={() => setNavOpen(false)}>
          <div
            className="border-border bg-surface h-full w-72 max-w-[80%] overflow-y-auto border-r p-5"
            onClick={(e) => e.stopPropagation()}
          >
            <div className="mb-5 flex items-center justify-between">
              <span className="text-text text-sm font-semibold">Docs</span>
              <button
                type="button"
                aria-label="Close docs menu"
                onClick={() => setNavOpen(false)}
                className="border-border text-text-body rounded-md border p-2"
              >
                <X size={16} />
              </button>
            </div>
            {nav}
          </div>
        </div>
      )}
    </main>
  )
}
```

**Uses `next/link`, unlike `vite-app`'s plain `<a>` convention** —
confirmed live: `eslint-config-next`'s `@next/next/no-html-link-for-pages`
rule flags a plain anchor to `/` here (though, curiously, not the home
page's plain anchor to `/docs` in `styles.md` — the rule's detection is
asymmetric depending on which file the anchor lives in). Rather than lean on
that asymmetry, both internal links in this skill use `next/link` — it's
also just the correct idiom for internal navigation in Next, enabling
client-side transitions instead of a full page reload.

**The hash-sync logic uses a lazy `useState` initializer, not an
effect+`setState`** — confirmed live: the original effect-based version
(`useEffect` reading `window.location.hash` then calling `setActiveId`)
trips `react-hooks/set-state-in-effect` ("calling setState synchronously
within an effect can trigger cascading renders"). Reading the hash inside
the state initializer instead (guarded with `typeof window === 'undefined'`
for the server-rendered pass) fixes the lint error *and* a real bug the
effect-based version had: linking to `/docs#some-doc` used to flash the
default doc first, then swap to the linked one once the effect ran after
paint — the initializer reads the hash before the first render instead, so
there's no flash.

## Known, non-blocking build warning

`pnpm build` succeeds (exit 0) but prints a Turbopack warning naming
`next.config.ts` in `docs/page.tsx`'s trace ("A file was traced that
indicates that the whole project was traced unintentionally"), caused by the
`fs` calls above using `process.cwd()`-based paths that Turbopack can't
fully statically analyze. Confirmed live that neither an
`outputFileTracingIncludes` entry nor `/* turbopackIgnore: true */` comments
on the `join()` calls suppress it — this looks like a Turbopack-specific
quirk in this Next version rather than something fixable from userland right
now. It doesn't fail the build or ship anything wrong; leave it rather than
chase it further, and revisit if a future Next release changes the tracer's
behavior.

## What "done" looks like

- Navigate to `/docs` → all markdown files appear in the sidebar
- Clicking a nav item shows the doc in the reading pane
- Mobile: hamburger opens the nav drawer
- Hash in URL reflects the active doc
- `README.md` appears under "Start here", `CLAUDE.md` under "Working notes"
- If any of the above is wrong or awkward in practice, that's the expected
  first-friction report for this skill — this file is where to look first
