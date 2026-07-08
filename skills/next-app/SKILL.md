---
name: next-app
description: Bootstrap a new Next.js App Router app with the Paper & Ink design system — a server-capable sibling to vite-app for apps that need real server-side compute, secrets, or per-user auth. Use when asked to scaffold a new Next.js app, create a starter project that needs a backend, or bootstrap a Next.js React app.
---

You are about to scaffold a complete, production-ready Next.js App Router
app. Read each resource file in order before writing any code — they contain
the exact file contents, CSS values, and implementation details you need.

## Before you start — check first, only ask if genuinely blocked

Look at the target directory before asking anything (default: the current
directory — confirmed live that asking the three questions below
unconditionally, even on an empty throwaway folder, is pure friction with no
payoff).

**If the directory is empty** (no files, or only OS cruft like `.DS_Store`,
or an empty `.git`): infer everything below and go straight to Step 1 — no
questions.

- **App name** — the directory's own basename, kebab-cased if it isn't
  already. Used in `package.json`, `app-meta.ts`, and the docs sidebar.
- **GitHub repo URL** — a plain placeholder, `https://github.com/your-username/<app-name>`.
  Never call `gh` or otherwise look up who the user actually is on GitHub —
  this field is cosmetic (used only in `app-meta.ts`'s `repo` field), so a
  clearly-fake value costs nothing and doesn't assume an identity that
  wasn't given. This skill only runs `git init` locally; it never creates or
  queries anything on GitHub itself.
- **Target directory** — the current directory. Always.

**If the directory is NOT empty:** stop and ask before writing anything.
Name exactly what you found — "this already looks like a git repo with a
`package.json`" / "there are already N files here, including an `images/`
folder — is that meant to be part of this app, or should I use a different
directory?" Don't guess whether existing content belongs to the scaffold or
is unrelated clutter; that's the user's call, not an assumption to make
silently.

## Resources — read these in order

- [Shell](resources/shell.md) — stack, all config files, entry points, exact
  `next.config.ts`
- [Styles](resources/styles.md) — Paper & Ink design system, exact CSS
  values, @theme bridge, the theme-init/theme-toggle client components
- [Docs](resources/docs.md) — docs/ folder structure and the full `/docs`
  viewer implementation (the one new, unvalidated piece — see the file)
- [Knowledge](resources/knowledge.md) — feature seams and knowledge/ folder
  pattern
- [CI/CD](resources/cicd.md) — GitHub Actions workflow

## Build order

Execute each step fully before moving to the next. Confirm `pnpm install`
succeeds after Step 1 before writing any source files.

### Step 1 — Shell

`git init` if `.git` doesn't already exist, write all config and entry files per
the shell and styles resources: `package.json`, `next.config.ts`,
`postcss.config.mjs`, `eslint.config.mjs`, `tsconfig.json`,
`vitest.config.ts`, `src/test-setup.ts`, `.gitignore`, `src/app-meta.ts`,
`src/app/layout.tsx`, `src/app/page.tsx`, `src/components/theme-init.tsx`,
`src/components/theme-toggle.tsx`.

Run `pnpm install`. Fix any install errors before continuing.

### Step 2 — Styles

Write the four CSS files per the styles resource: `src/styles/tokens.css`,
`src/styles/base.css`, `src/styles/typography.css`, `src/styles/index.css`.

Already wired — `layout.tsx` imports `#/styles/index.css` and renders
`ThemeInit`/`ThemeToggle`. Confirm the app runs with `pnpm dev`, the home
page renders styled, and the theme toggle works.

### Step 3 — Docs

Write the four markdown files in `docs/`: `OVERVIEW.md`, `SPEC.md`,
`PLAN.md`, `STYLE.md`.

Write `src/features/docs/build-docs.ts`, `src/app/docs/page.tsx`, and
`src/components/docs-viewer.tsx` using the full implementation from the docs
resource.

Add `react-markdown` and `remark-gfm` to `package.json` and run
`pnpm install`.

Confirm `/docs` loads and all four files appear in the sidebar.

### Step 4 — Knowledge

Create `src/features/` with subfolders: `core`, `generate`, `verify`,
`knowledge`, `prefs`. Write a `CLAUDE.md` in each describing its
responsibility boundary.

Write `src/features/knowledge/index.ts` with the `fs` + `server-only` loader.

Create `knowledge/` at the repo root with placeholder `guidance.md` and
`rubric.md`.

### Step 5 — CI/CD

Write `.github/workflows/ci.yml` per the cicd resource.

### Step 6 — Verify, then launch

Run `pnpm build`, `pnpm test`, `pnpm lint` — all must pass, with zero env
vars set.

Write `CLAUDE.md` at the repo root (agent orientation: what the app does,
current phase, what to read first).

Write `README.md` (public summary: stack, boundary, local dev commands).

Once all three gates pass, finish with the actual handoff moment, not just a
pass/fail report:

1. Start `pnpm dev` as a tracked background process (not shell `&`) so it's
   stoppable rather than orphaned.
2. Poll `http://localhost:<dev-port>` until it responds — don't open a
   browser to a connection-refused page.
3. Open the browser to the home page (`open http://localhost:<dev-port>` on
   macOS — `xdg-open` on Linux, `start` on Windows).
4. Leave the dev server running. That's the point: the user picks up from a
   live, already-open app, not one they have to start themselves.
5. Mention that `/docs` is live on the same server.

## What the finished repo looks like

```
your-app/
├── CLAUDE.md
├── README.md
├── docs/
│   ├── OVERVIEW.md
│   ├── PLAN.md
│   ├── SPEC.md
│   └── STYLE.md
├── knowledge/
│   ├── guidance.md
│   └── rubric.md
├── src/
│   ├── app-meta.ts
│   ├── test-setup.ts
│   ├── app/
│   │   ├── layout.tsx               ← fonts + theme wiring here
│   │   ├── page.tsx
│   │   └── docs/
│   │       └── page.tsx             ← Server Component, reads fs
│   ├── components/
│   │   ├── theme-init.tsx
│   │   ├── theme-toggle.tsx
│   │   └── docs-viewer.tsx          ← Client Component
│   ├── features/
│   │   ├── core/CLAUDE.md
│   │   ├── generate/CLAUDE.md
│   │   ├── verify/CLAUDE.md
│   │   ├── docs/build-docs.ts
│   │   ├── knowledge/
│   │   │   ├── CLAUDE.md
│   │   │   └── index.ts             ← server-only
│   │   └── prefs/CLAUDE.md
│   └── styles/
│       ├── index.css
│       ├── tokens.css
│       ├── base.css
│       └── typography.css
├── .github/workflows/ci.yml
├── package.json
├── next.config.ts
├── postcss.config.mjs
├── eslint.config.mjs
├── tsconfig.json
├── vitest.config.ts
└── .gitignore
```

Phase 0 complete. The dev server is already running and the browser is
already open on the home page — the docs viewer works, the style system is
live, and the feature seams are in place, on a framework that can also host
real server-side compute when a feature needs it. Pick up from there.
