# Part: docs

The docs folder structure plus the in-app `/docs` viewer. The markdown files
are the human/agent-readable record of intent, and the viewer makes them
reachable without leaving the app.

**This file is the one genuinely new, unvalidated piece of `next-app`.** The
natural approach — a bundler-level glob like `import.meta.glob` that reads and
bundles the files in one call at build time — has no Next.js equivalent; the
proven reference app hit this exact gap during its migration and simply dropped
the `/docs` route rather than solve it. What follows is a first design, not a
ported-and-confirmed one. If it breaks or feels wrong in actual use, that's
expected — report it as friction rather than assuming it's a mistake in reading
these instructions.

## The `docs/` folder

The standard files, written before significant code lands. One shift to note:
**`PLAN.md` is gone**, replaced by `CODE-STANDARDS.md`. The project standard is
explicit that plans, tasks, and bugs are GitHub issues, never a markdown file in
`docs/` — so the scaffold no longer writes one. Build order and "what's next"
live in the README `## Status` block and in issues.

### `docs/OVERVIEW.md`

The umbrella vision — the why behind the whole project. Where it fits in the
broader set of apps, what problem it solves, what it explicitly is not.
Written once, rarely updated.

### `docs/SPEC.md`

The what and why of this specific app. The welded boundary (what it does and
what it deliberately does not do), the key design decisions. Readable by a
non-coder.

### `docs/CODE-STANDARDS.md`

The repo's own copy of the project standard — the file the standard tells every
repo to state ("A repo states its own copy in `docs/CODE-STANDARDS.md`,
following this"). It is what wires a freshly-scaffolded app to the conventions
it's supposed to follow, so an agent picking up the repo reads *this* rather
than a URL to an external standard. Keep it short — it restates the load-bearing
rules for *this* app, not the whole standard verbatim.

Write it with this content, adapted to the app's alias and name:

```markdown
# Code standards

This repo follows the project standard. The load-bearing rules:

## The `app/` folder

- `name/` is a route (`page.tsx`); `_name/` is supporting code Next ignores
  as a URL (`_actions/ _queries/ _components/ _providers/`); `(name)/` groups
  without adding to the path.
- Every route is the same shape: `page.tsx` · `_queries/` (reads) ·
  `_actions/` (writes) · `_components/` (renders).
- Concerns are folders, never a bare file: `_actions/x.ts`, not `_actions.ts`.
- No `index.ts` barrels inside `app/` — import the named file.
- Name by subject; don't repeat the parent (`activity/_components/counter.tsx`,
  not `activity-counter.tsx`).
- kebab-case; import alias `#/` → `src/`. `src/features/` holds domain code
  shared beyond one route.

## Components

Order of search when a view needs a component:

1. Shop the shelf — build from `src/components/` if you can.
2. A gap the shelf can't fill — build the primitive properly shareable.
3. Hand-build only for a true outlier.

Where a component lives — litmus: does it import `#/features`?

- **Primitive** → `src/components/` (no `#/features`, no `next/*`; stack-portable).
- **App-shared** → `app/_components/` (imports `#/features`, used by 2+ routes).
- **Route-local** → that route's `_components/`.

Build on need — no speculative primitives. **The bar to promote is "properly
shareable," not "used twice"**: a generic, type-safe contract with no coupling to
a feature, route, or the DB. Usage count doesn't decide it; genericness does.

## Styling

CSS only — the Paper & Ink token layers plus co-located CSS Modules
(`docs/STYLE.md`, `src/styles/`). No Tailwind, no PostCSS, no CSS-in-JS.

- `src/styles/` is the frozen baseline; a component styles itself with a
  `name.module.css` beside its `name.tsx`.
- **Values reference tokens, never a raw color or number.** Dark mode is a token
  remap under `:root[data-theme='dark']`, so `var(--…)` follows automatically; a
  structural dark-mode difference uses `:global([data-theme='dark']) .x { }`.
- Base + modifier is the house idiom — the modifier sets only the delta.
- Style children with the direct-child combinator (`.card > h2`), one level deep.

A utility class here fails **silently** (unstyled, no error), so it is the
regression to watch for.

## docs & issues

- Root-level `docs/` holds what the project is and why: `OVERVIEW.md`,
  `SPEC.md`, and `reference/` for notes. Plans, tasks, and bugs are **GitHub
  issues**, never files in `docs/`.
- Orientation = the README `## Status` block + open issues. No plan files.

## Writing the docs

Primary reader is a model booting a session. Brief, bulleted, no prose.

- **The code is the documentation.** A doc restating the file tree goes stale.
- **State a rule once, where it binds.** The same rule in two files drifts into
  two different rules and the reader has to pick a winner.
- **Prose earns its place only where the code cannot show it** — a past failure,
  a consequence, a reason. Stop cutting when removing one more sentence would
  change a decision.
- Reasoning and taste go in `docs/reference/`, not in a file re-read on every edit.

## The README `## Status` block

A snapshot, not a log — read at every session boot.

- **Last shipped — 6 bullets max, one line each, most-recent first.**
- **Up next — a short pointer to the ordered issues.**
- Narrative lives in `git log` and PR descriptions. Overwrite freely.

## CLAUDE.md

Write a subfolder `CLAUDE.md` only where the folder is a boundary you could
violate without reading it (an invariant, an ownership rule, a dependency
direction). A missing one is correct when there's nothing non-obvious to say.
Not a file inventory, not a restatement of global rules.
```

### `docs/STYLE.md`

The style system contract, naming this skill's four style touch-points
(`shell.md`).

## The in-app `/docs` viewer

A `/docs` route rendering all markdown from `docs/`, `knowledge/`,
`README.md`, and `CLAUDE.md` in a sidebar-nav reader.

### Dependencies

```
react-markdown
remark-gfm
```

### Why the split: a Server Component reading files, a Client Component rendering them

A bundler-level glob like `import.meta.glob` does two jobs at once: it reads
the files *and* bundles them in, eagerly, at build time. Next has no single API
that does both. The split divides those jobs across the Server/Client boundary
App Router already has:

- A **Server Component** (`src/app/docs/page.tsx`) reads the files with
  Node's `fs` — this only runs on the server (build time for a static route
  like this one, since nothing here depends on the request), never ships to
  the browser.
- A **Client Component** (`src/components/docs-viewer/docs-viewer.tsx`) receives the
  already-read docs as a plain serializable prop (an array of
  `{ id, title, section, order, content }` — all strings and numbers, safe to
  cross the Server→Client boundary) and owns the interactive part: sidebar
  nav, active-doc state, mobile drawer.

The classification logic (turning a file path into a title/section/order) is
pulled out into its own pure, framework-agnostic function so it doesn't care
whether the raw content came from a build-time glob or a Node `fs` read.

### `src/lib/doc-index.ts`

`src/features/` means shared by more than one route, and `/docs` is one route —
so this is a **mechanism**, not a feature. It lives in `src/lib/`, named for the
index it builds rather than for the `docs/` folder it reads about.


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

The classification rules (`/knowledge/` → "Knowledge", `/CLAUDE.md` → "Working
notes", `/README.md` and `/docs/` → "Start here", optional `NN-` filename prefix
for manual ordering) operate on a plain `Record<string, string>` — a shape any
file source can produce.

### `src/app/docs/page.tsx`

Reads the four sources from disk. `readdirSync(dir, { recursive: true })`
needs Node 18.17+ — already required by Next 16, so no new constraint:

```tsx
import { existsSync, readdirSync, readFileSync } from 'node:fs'
import { join } from 'node:path'
import { buildDocs } from '#/lib/doc-index'
import { DocsViewer } from '#/components'
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
static route — the docs shown are whatever was in the repo at build time.

### `src/components/docs-viewer/docs-viewer.tsx`

Owns all interactivity, fed `docs` as a prop instead of computing them from a
glob:

```tsx
'use client'

import { useState, useSyncExternalStore } from 'react'
import Link from 'next/link'
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import { Menu, X } from 'lucide-react'
import { appMeta } from '#/app-meta'
import { groupBySection, type Doc } from '#/lib/doc-index'

// The URL hash is the single source of truth for which doc is open, which means
// reading it has to survive server rendering: the server has no window, so
// reading it during render would emit the default doc on the server and the
// hashed doc on the client — different text, hydration failure. Deep-linking
// /docs#some-doc and refreshing hit exactly that.
//
// useSyncExternalStore exists for this: React renders getServerSnapshot on the
// server and again during hydration (so both agree), then immediately re-renders
// with the real hash. No state to keep in sync, and no setState in an effect
// (which `react-hooks/set-state-in-effect` rejects — the naive useEffect version
// of this fix fails lint).
function subscribeToHash(onChange: () => void): () => void {
  window.addEventListener('hashchange', onChange)
  return () => window.removeEventListener('hashchange', onChange)
}

const getHash = () => window.location.hash.replace(/^#/, '')
const getServerHash = () => ''

export function DocsViewer({ docs }: { docs: Doc[] }) {
  const sections = groupBySection(docs)
  const defaultId = sections[0]?.docs[0]?.id ?? docs[0]?.id ?? ''
  const hashId = useSyncExternalStore(subscribeToHash, getHash, getServerHash)
  const [navOpen, setNavOpen] = useState(false)

  const activeId = docs.some((d) => d.id === hashId) ? hashId : defaultId

  function select(id: string) {
    setNavOpen(false)
    // replaceState keeps a long docs-browsing session from filling the back
    // button, but it does not fire hashchange — so tell the store ourselves.
    window.history.replaceState(null, '', `#${id}`)
    window.dispatchEvent(new HashChangeEvent('hashchange'))
    window.scrollTo({ top: 0 })
  }

  const active = docs.find((d) => d.id === activeId) ?? docs[0]
  if (!active) {
    return <main className={styles.empty}>No documents found.</main>
  }

  const nav = (
    <nav className={styles.nav}>
      {sections.map(({ section, docs: items }) => (
        <div key={section} className={styles.navSection}>
          <p className={styles.navHeading}>{section}</p>
          {items.map((d) => (
            <button
              key={d.id}
              type="button"
              onClick={() => select(d.id)}
              className={`${styles.navItem} ${d.id === active.id ? styles.navItemActive : ''}`}
            >
              {d.title}
            </button>
          ))}
        </div>
      ))}
    </nav>
  )

  return (
    <main className={styles.shell}>
      <aside className={styles.sidebar}>
        <Link href="/" className={styles.brand}>
          {appMeta.name}
        </Link>
        {nav}
      </aside>
      <div className={styles.body}>
        <article className={styles.article}>
          <div className={styles.mobileBar}>
            <button
              type="button"
              aria-label="Open docs menu"
              onClick={() => setNavOpen(true)}
              className={styles.iconButton}
            >
              <Menu size={16} />
            </button>
            <span className={styles.mobileTitle}>{active.title}</span>
          </div>
          <div className="prose compact">
            <ReactMarkdown remarkPlugins={[remarkGfm]}>{active.content}</ReactMarkdown>
          </div>
        </article>
      </div>
      {navOpen && (
        <div className={styles.scrim} onClick={() => setNavOpen(false)}>
          <div className={styles.drawer} onClick={(e) => e.stopPropagation()}>
            <div className={styles.drawerHeader}>
              <span className={styles.drawerTitle}>Docs</span>
              <button
                type="button"
                aria-label="Close docs menu"
                onClick={() => setNavOpen(false)}
                className={styles.iconButton}
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

`prose compact` stays a bare global pair — those are L1 typography roles from
`typography.css` styling markdown output the component never authors, which is
exactly what a role class is for.

### `src/components/docs-viewer/docs-viewer.module.css`

The sidebar is the one genuinely layout-shaped component in the scaffold, so it
carries a flat set of named element classes — one per semantic region. That is
correct for layout/overlay components, not a smell.

```css
.shell {
  display: flex;
  min-height: 100vh;
  background: var(--bg);
  color: var(--text);
}

.empty {
  display: flex;
  min-height: 100vh;
  align-items: center;
  justify-content: center;
  background: var(--bg);
  color: var(--text-muted);
}

.sidebar {
  position: sticky;
  top: 0;
  display: none;
  height: 100vh;
  width: 272px;
  flex-shrink: 0;
  overflow-y: auto;
  border-right: 1px solid var(--border);
  background: var(--surface);
  padding: var(--space-7) var(--space-5) var(--space-7);
}

.brand {
  display: block;
  margin-bottom: var(--space-6);
  padding-inline: var(--space-3);
  color: var(--text-muted);
  font-family: var(--font-mono);
  font-size: var(--text-xs);
  letter-spacing: -0.01em;
  transition: color var(--dur-base) var(--ease);
}

.brand:hover {
  color: var(--accent);
}

.nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-6);
}

.navSection {
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.navHeading {
  margin-bottom: var(--space-2);
  padding-inline: var(--space-3);
  color: var(--text-faint);
  font-family: var(--font-mono);
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.12em;
  text-transform: uppercase;
}

.navItem {
  display: block;
  width: 100%;
  border-radius: var(--radius-md);
  padding: 0.375rem var(--space-3);
  text-align: left;
  font-size: 13px;
  color: var(--text-muted);
  transition: all var(--dur-base) var(--ease);
}

.navItem:hover {
  background: var(--surface-sunken);
  color: var(--text);
}

/* Modifier sets only the delta from .navItem. */
.navItemActive {
  background: var(--surface-raised);
  color: var(--text);
  font-weight: 500;
}

.body {
  min-width: 0;
  flex: 1;
}

.article {
  margin-inline: auto;
  max-width: 760px;
  padding: var(--space-7) var(--space-5) 7rem;
}

.mobileBar {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  margin-bottom: var(--space-6);
}

.mobileTitle {
  color: var(--text);
  font-size: var(--text-sm);
  font-weight: 500;
}

.iconButton {
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  padding: var(--space-2);
  color: var(--text-body);
}

.scrim {
  position: fixed;
  inset: 0;
  z-index: 50;
  background: var(--scrim);
}

.drawer {
  height: 100%;
  width: 18rem;
  max-width: 80%;
  overflow-y: auto;
  border-right: 1px solid var(--border);
  background: var(--surface);
  padding: var(--space-5);
}

.drawerHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--space-5);
}

.drawerTitle {
  color: var(--text);
  font-size: var(--text-sm);
  font-weight: 600;
}

/* The sidebar/drawer swap. Mobile-first: drawer below, sidebar above. */
@media (min-width: 640px) {
  .sidebar {
    display: block;
  }

  .mobileBar,
  .scrim {
    display: none;
  }

  .article {
    padding: var(--space-9) var(--space-7) 7rem;
  }
}
```

**The one thing to check after scaffolding:** `tokens.css` must define
`--scrim` and `--radius-md`. Tailwind supplied `bg-black/60` and `rounded-md`
implicitly; with the framework gone they have to exist as real tokens, which is
the migration trap `project-standard` names — *tokens the framework gave
implicitly surface as gaps to add explicitly*.

**Uses `next/link` for internal navigation** — confirmed live:
`eslint-config-next`'s `@next/next/no-html-link-for-pages` rule flags a plain
anchor to `/` here (though, curiously, not the home page's plain anchor to
`/docs` in `styles.md` — the rule's detection is asymmetric depending on which
file the anchor lives in). Rather than lean on that asymmetry, both internal
links in this skill use `next/link` — it's also just the correct idiom for
internal navigation in Next, enabling client-side transitions instead of a full
page reload.

**The hash-sync logic uses `useSyncExternalStore`.** Both of the obvious
approaches are wrong, and this skill shipped each of them in turn before
landing here — so don't "simplify" it back:

- **`useEffect` + `setActiveId`** trips `react-hooks/set-state-in-effect`
  ("calling setState synchronously within an effect can trigger cascading
  renders"). Fails lint.
- **A lazy `useState` initializer** reading `window.location.hash` (guarded
  with `typeof window === 'undefined'`) passes lint, and is what this skill
  recommended for a while. It is a **hydration bug**: the server has no
  `window`, so it renders the *default* doc while the client renders the
  *hashed* one. Different text. Deep-linking `/docs#some-doc` and refreshing
  throws "Hydration failed because the server rendered text didn't match the
  client" and makes React throw away the server HTML and re-render the whole
  tree. Caught live in `bootsy`.

`useSyncExternalStore` is the tool built for exactly this: `getServerSnapshot`
renders on the server *and* during hydration (so both sides agree), then React
re-renders with the real client hash. Lint-clean, no state to keep in sync, and
the URL hash becomes the single source of truth for which doc is open.

The cost is a brief flash of the default doc when deep-linking, which the lazy
initializer avoided. That flash is **unavoidable with a hash** under SSR —
fragments are never sent to the server, so no server render can know which doc
you asked for. Correctness beats the flash: a hydration mismatch is not a
cosmetic problem. If the flash ever matters, the fix is not to reintroduce the
bug — it's to move the doc selector into a *search param* (`/docs?doc=some-doc`),
which the server can read, and use `useSearchParams`.

One non-obvious detail: `select()` uses `history.replaceState` so a long
browsing session doesn't fill the back button — but `replaceState` does **not**
fire `hashchange`, so the component dispatches the event itself. Remove that
line and clicking a sidebar item silently stops working.

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
