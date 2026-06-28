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
what it deliberately does not do), the key design decisions, the two-depth build
strategy (shallow first, richer later only if earned). Readable by a non-coder.

### `docs/PLAN.md`

The concrete build order. The how — the sequence of phases a coding agent
executes against. Each phase has a clear output ("after this phase, X is
runnable"). Updated as phases complete. This is the primary orientation doc for
agents starting work.

### `docs/STYLE.md`

The style system contract and port recipe. Describes the four CSS files, the
five touch-points for porting, and the one rule (feature styles never in
`src/styles/`). See the **styles** part for the full detail — `STYLE.md` in
each repo is the in-repo version of that same content.

## The in-app `/docs` viewer

A `/docs` route that renders all markdown files from `docs/`, `knowledge/`,
`README.md`, and `CLAUDE.md` in a sidebar-nav reader. Zero wiring — drop a
file anywhere in those locations and it appears in the nav automatically.

### Dependencies

```
react-markdown
remark-gfm
```

### How it works

Uses Vite's `import.meta.glob` to bundle all matching markdown files at build
time:

```ts
const RAW = import.meta.glob(
  ['/README.md', '/CLAUDE.md', '/docs/**/*.md', '/knowledge/*.md'],
  { query: '?raw', import: 'default', eager: true },
)
```

Section assignment comes from file path, not per-file configuration:
- `/knowledge/` → "Knowledge"
- `/CLAUDE.md` → "Working notes"
- `/README.md` and `/docs/` → "Start here"

File ordering uses an optional `NN-` filename prefix (e.g. `01-overview.md`
shows as "Overview" and sorts before `02-spec.md`). Files without a prefix
sort alphabetically after numbered ones. Reorder by renaming files — no code
changes.

### Layout

Two-column desktop layout: sticky left sidebar with grouped nav, scrollable
right reading pane. On mobile, the sidebar collapses behind a hamburger menu.
The reading pane uses `.prose.compact` from the styles system. Hash-based URL
for the active doc (`/docs#overview`).

### `src/routes/docs.tsx`

The canonical implementation lives in `outpaint-studio/src/routes/docs.tsx`.
Copy it verbatim — it has no app-specific logic. The only part that needs
updating is the app name in the sidebar back-link:

```tsx
<Link to="/" className="...">
  your-app-name   {/* ← update this */}
</Link>
```

Everything else (the glob, the classifier, the section ordering, the layout)
is identical across apps.

## What "done" looks like

- `pnpm dev` → navigate to `/docs` → all markdown files in `docs/` and
  `knowledge/` appear in the sidebar
- Clicking a nav item shows that doc in the reading pane
- Mobile: hamburger opens the nav drawer
- Hash in URL reflects the active doc
- `README.md` and `CLAUDE.md` appear under "Working notes" / "Start here"
