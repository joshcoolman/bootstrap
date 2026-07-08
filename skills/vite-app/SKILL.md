---
name: vite-app
description: Bootstrap a new Vite + React + TanStack Router app with the Paper & Ink design system. Use when asked to scaffold a new app, create a starter project, or bootstrap a Vite React app.
---

You are about to scaffold a complete, production-ready Vite + React + TanStack Router app. Read each resource file in order before writing any code — they contain the exact file contents, CSS values, and implementation details you need.

## Before you start — check first, only ask if genuinely blocked

Look at the target directory before asking anything (default: the current
directory — confirmed live, via `next-app`'s identical pattern, that asking
the three questions below unconditionally, even on an empty throwaway
folder, is pure friction with no payoff).

**If the directory is empty** (no files, or only OS cruft like `.DS_Store`,
or an empty `.git`): infer everything below and go straight to Step 1 — no
questions.

- **App name** — the directory's own basename, kebab-cased if it isn't
  already. Used in `package.json`, `app-meta.ts`, `manifest.json`, and the
  docs sidebar.
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

- [Shell](resources/shell.md) — stack, all config files, entry points, exact vite.config.ts
- [Styles](resources/styles.md) — Paper & Ink design system, exact CSS values, @theme bridge
- [Docs](resources/docs.md) — docs/ folder structure and the full /docs viewer implementation
- [Knowledge](resources/knowledge.md) — feature seams and knowledge/ folder pattern
- [CI/CD](resources/cicd.md) — GitHub Actions workflow

## Build order

Execute each step fully before moving to the next. Confirm `pnpm install` succeeds after Step 1 before writing any source files.

### Step 1 — Shell
`git init` if `.git` doesn't already exist, write all config files per the shell resource:
`package.json`, `vite.config.ts`, `vitest.config.ts`, `tsconfig.json`, `tsr.config.json`,
`eslint.config.js`, `prettier.config.js`, `.prettierignore`, `pnpm-workspace.yaml`,
`index.html`, `src/main.tsx`, `src/app-meta.ts`, `src/test-setup.ts`,
`src/routes/__root.tsx`, `src/routes/index.tsx`, `src/components/home.tsx`,
`src/components/theme-toggle.tsx`, `public/manifest.json`, `public/robots.txt`.

Run `pnpm install`. Fix any install errors before continuing.

### Step 2 — Styles
Write the four CSS files per the styles resource:
`src/styles/tokens.css`, `src/styles/base.css`, `src/styles/typography.css`, `src/styles/index.css`.

The styles are already wired — `main.tsx` imports `./styles/index.css` and `index.html` has the fonts and pre-paint script. Confirm the app runs with `pnpm dev` and the theme toggle works.

**Important:** starting `pnpm dev` here also generates `src/routeTree.gen.ts` via the
TanStack Router Vite plugin. That file must exist before `pnpm build` can type-check
cleanly in Step 6.

### Step 3 — Docs
Write the four markdown files in `docs/`: `OVERVIEW.md`, `SPEC.md`, `PLAN.md`, `STYLE.md`.

Write `src/routes/docs.tsx` using the full implementation from the docs resource.

Add `react-markdown` and `remark-gfm` to `package.json` and run `pnpm install`.

Confirm `/docs` loads and all four files appear in the sidebar.

### Step 4 — Knowledge
Create `src/features/` with subfolders: `core`, `generate`, `verify`, `knowledge`, `prefs`.
Write a `CLAUDE.md` in each describing its responsibility boundary.
Write `src/features/knowledge/index.ts` with the glob loader.
Create `knowledge/` at the repo root with placeholder `guidance.md` and `rubric.md`.

### Step 5 — CI/CD
Write `.github/workflows/ci.yml` per the cicd resource.

### Step 6 — Verify, then launch
Run `pnpm build`, `pnpm test`, `pnpm lint` — all must pass.

Write `CLAUDE.md` at the repo root (agent orientation: what the app does, current phase, what to read first).

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
├── index.html                      ← pre-paint script + fonts here
├── docs/
│   ├── OVERVIEW.md
│   ├── PLAN.md
│   ├── SPEC.md
│   └── STYLE.md
├── knowledge/
│   ├── guidance.md
│   └── rubric.md
├── src/
│   ├── main.tsx                    ← CSS imported here
│   ├── app-meta.ts
│   ├── routeTree.gen.ts
│   ├── components/
│   │   ├── home.tsx
│   │   └── theme-toggle.tsx
│   ├── features/
│   │   ├── core/CLAUDE.md
│   │   ├── generate/CLAUDE.md
│   │   ├── verify/CLAUDE.md
│   │   ├── knowledge/
│   │   │   ├── CLAUDE.md
│   │   │   └── index.ts
│   │   └── prefs/CLAUDE.md
│   ├── routes/
│   │   ├── __root.tsx
│   │   ├── index.tsx
│   │   └── docs.tsx
│   └── styles/
│       ├── index.css
│       ├── tokens.css
│       ├── base.css
│       └── typography.css
├── public/
├── .github/workflows/ci.yml
├── package.json
├── vite.config.ts
├── vitest.config.ts
├── tsconfig.json
├── tsr.config.json
├── pnpm-workspace.yaml
├── eslint.config.js
└── prettier.config.js
```

Phase 0 complete. The dev server is already running and the browser is already open on the home page — the docs viewer works, the style system is live, and the feature seams are in place. Pick up from there.
