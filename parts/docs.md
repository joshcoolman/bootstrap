# Part: docs

The docs folder structure plus the in-app `/docs` viewer. Every public app
gets both: the markdown files as the human/agent-readable record of intent, and
the viewer so those docs are accessible without leaving the app.

## The `docs/` folder

Four standard files. All are written before significant code lands.

### `docs/OVERVIEW.md`

The umbrella vision — the why behind the whole project. Where it fits in the
broader set of apps, what problem it solves, what it explicitly is not. Written
once, rarely updated.

### `docs/SPEC.md`

The what and why of this specific app. The welded boundary (what it does and
what it deliberately does not do), the key design decisions. Readable by a
non-coder.

### `docs/PLAN.md`

The concrete build order. Each phase has a clear output ("after this phase, X
is runnable"). Updated as phases complete. The primary orientation doc for
agents starting work.

### `docs/STYLE.md`

The style system contract. Describes the four CSS files, the five touch-points
for porting, and the one rule (feature styles never in `src/styles/`).

## The in-app `/docs` viewer

A `/docs` route that renders all markdown files from `docs/`, `knowledge/`,
`README.md`, and `CLAUDE.md` in a sidebar-nav reader.

### Dependencies

```
react-markdown
remark-gfm
```

### `src/routes/docs.tsx`

Write this from scratch using the spec below. The full implementation:

```tsx
import { createFileRoute, Link } from '@tanstack/react-router'
import { useEffect, useMemo, useState } from 'react'
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import { Menu, X } from 'lucide-react'
import { appMeta } from '#/app-meta'

export const Route = createFileRoute('/docs')({ component: DocsPage })

const RAW: Record<string, string> = import.meta.glob(
  ['/README.md', '/CLAUDE.md', '/docs/**/*.md', '/knowledge/*.md'],
  { query: '?raw', import: 'default', eager: true },
)

type Doc = { id: string; title: string; section: string; order: number; content: string }
const SECTION_ORDER = ['Start here', 'Knowledge', 'Working notes']

function titleCase(s: string) { return s.charAt(0).toUpperCase() + s.slice(1) }
function slugify(path: string) {
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
function buildDocs(): Doc[] {
  return Object.entries(RAW)
    .map(([path, content]) => ({ content, ...classify(path) }))
    .sort((a, b) => a.order - b.order || a.title.localeCompare(b.title))
}

function DocsPage() {
  const docs = useMemo(buildDocs, [])
  const sections = useMemo(() => {
    const bySection = new Map<string, Doc[]>()
    for (const d of docs) { const list = bySection.get(d.section) ?? []; list.push(d); bySection.set(d.section, list) }
    return SECTION_ORDER.filter((s) => bySection.has(s)).map((section) => ({ section, docs: bySection.get(section) ?? [] }))
  }, [docs])

  const defaultId = sections[0]?.docs[0]?.id ?? docs[0]?.id ?? ''
  const [activeId, setActiveId] = useState(defaultId)
  const [navOpen, setNavOpen] = useState(false)

  useEffect(() => {
    const fromHash = window.location.hash.replace(/^#/, '')
    if (fromHash && docs.some((d) => d.id === fromHash)) setActiveId(fromHash)
  }, [docs])

  function select(id: string) {
    setActiveId(id); setNavOpen(false)
    window.history.replaceState(null, '', `#${id}`)
    window.scrollTo({ top: 0 })
  }

  const active = docs.find((d) => d.id === activeId) ?? docs[0]
  if (!active) return <main className="bg-bg text-text-muted flex min-h-screen items-center justify-center">No documents found.</main>

  const nav = (
    <nav className="flex flex-col gap-7">
      {sections.map(({ section, docs: items }) => (
        <div key={section} className="flex flex-col gap-0.5">
          <p className="text-text-faint mb-2 px-3 font-mono text-[11px] font-semibold tracking-[0.12em] uppercase">{section}</p>
          {items.map((d) => (
            <button key={d.id} type="button" onClick={() => select(d.id)}
              className={`block w-full rounded-md px-3 py-1.5 text-left text-[13px] transition ${d.id === active.id ? 'bg-surface-raised text-text font-medium' : 'text-text-muted hover:text-text hover:bg-surface-sunken'}`}>
              {d.title}
            </button>
          ))}
        </div>
      ))}
    </nav>
  )

  return (
    <main className="bg-bg text-text flex min-h-screen">
      <aside className="border-border bg-surface sticky top-0 hidden h-screen w-[272px] shrink-0 overflow-y-auto px-5 pt-9 pb-12 border-r sm:block">
        <Link to="/" className="text-text-muted hover:text-accent mb-9 block px-3 font-mono text-xs tracking-tight transition-colors">{appMeta.name}</Link>
        {nav}
      </aside>
      <div className="min-w-0 flex-1">
        <article className="mx-auto max-w-[760px] px-6 pt-12 pb-28 sm:px-10 sm:pt-20">
          <div className="mb-10 flex items-center gap-3 sm:hidden">
            <button type="button" aria-label="Open docs menu" onClick={() => setNavOpen(true)} className="border-border text-text-body rounded-md border p-2"><Menu size={16} /></button>
            <span className="text-text text-sm font-medium">{active.title}</span>
          </div>
          <div className="prose compact">
            <ReactMarkdown remarkPlugins={[remarkGfm]}>{active.content}</ReactMarkdown>
          </div>
        </article>
      </div>
      {navOpen && (
        <div className="fixed inset-0 z-50 bg-black/60 sm:hidden" onClick={() => setNavOpen(false)}>
          <div className="border-border bg-surface h-full w-72 max-w-[80%] overflow-y-auto border-r p-5" onClick={(e) => e.stopPropagation()}>
            <div className="mb-5 flex items-center justify-between">
              <span className="text-text text-sm font-semibold">Docs</span>
              <button type="button" aria-label="Close docs menu" onClick={() => setNavOpen(false)} className="border-border text-text-body rounded-md border p-2"><X size={16} /></button>
            </div>
            {nav}
          </div>
        </div>
      )}
    </main>
  )
}
```

### How it works

Uses Vite's `import.meta.glob` to bundle all matching markdown files at build
time. Section assignment comes from file path:
- `/knowledge/` → "Knowledge"
- `/CLAUDE.md` → "Working notes"
- `/README.md` and `/docs/` → "Start here"

File ordering uses an optional `NN-` filename prefix. Files without a prefix
sort alphabetically. Hash-based URL for the active doc (`/docs#overview`).

### Layout

Two-column desktop layout: sticky left sidebar (272px), scrollable right reading
pane. On mobile, sidebar is hidden behind a hamburger menu overlay. The reading
pane uses `.prose.compact` from the styles system.

## What "done" looks like

- Navigate to `/docs` → all markdown files appear in the sidebar
- Clicking a nav item shows the doc in the reading pane
- Mobile: hamburger opens the nav drawer
- Hash in URL reflects the active doc
- `README.md` appears under "Start here", `CLAUDE.md` under "Working notes"
