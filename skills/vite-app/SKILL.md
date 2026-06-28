---
description: Bootstrap a new Vite + React + TanStack Router app with the Paper & Ink design system. Use when asked to scaffold a new app, create a starter project, or bootstrap a Vite React app.
---

You are about to scaffold a complete, production-ready Vite + React + TanStack Router app. Read each resource file in order before writing any code — they contain the exact file contents, CSS values, and implementation details you need.

## Before you start — ask the user

1. **App name** — used in `package.json`, `app-meta.ts`, `manifest.json`, and the docs sidebar
2. **Dev port** — pick one that doesn't conflict with sibling repos (e.g. 5173, 5174, 3001)
3. **GitHub repo URL** — used in `app-meta.ts` (can be a placeholder if the repo doesn't exist yet)
4. **Target directory** — where to create the repo (default: current directory)

## Resources — read these in order

- [Shell](resources/shell.md) — stack, all config files, entry points, exact vite.config.ts
- [Styles](resources/styles.md) — Paper & Ink design system, exact CSS values, @theme bridge
- [Docs](resources/docs.md) — docs/ folder structure and the full /docs viewer implementation
- [Knowledge](resources/knowledge.md) — feature seams and knowledge/ folder pattern
- [CI/CD](resources/cicd.md) — GitHub Actions workflow

## Build order

Execute each step fully before moving to the next. Confirm `pnpm install` succeeds after Step 1 before writing any source files.

### Step 1 — Shell
Create the repo directory, `git init`, write all config files per the shell resource:
`package.json`, `vite.config.ts`, `tsconfig.json`, `tsr.config.json`,
`eslint.config.js`, `prettier.config.js`, `.prettierignore`, `pnpm-workspace.yaml`,
`index.html`, `src/main.tsx`, `src/app-meta.ts`, `src/routes/__root.tsx`,
`src/routes/index.tsx`, `src/components/home.tsx`, `src/components/theme-toggle.tsx`,
`public/manifest.json`, `public/robots.txt`.

Run `pnpm install`. Fix any install errors before continuing.

### Step 2 — Styles
Write the four CSS files per the styles resource:
`src/styles/tokens.css`, `src/styles/base.css`, `src/styles/typography.css`, `src/styles/index.css`.

The styles are already wired — `main.tsx` imports `./styles/index.css` and `index.html` has the fonts and pre-paint script. Confirm the app runs with `pnpm dev` and the theme toggle works.

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

### Step 6 — Verify
Run `pnpm build`, `pnpm test`, `pnpm lint` — all must pass.

Write `CLAUDE.md` at the repo root (agent orientation: what the app does, current phase, what to read first).

Write `README.md` (public summary: stack, boundary, local dev commands).

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
├── tsconfig.json
├── tsr.config.json
├── pnpm-workspace.yaml
├── eslint.config.js
└── prettier.config.js
```

Phase 0 complete. The shell is runnable, the docs viewer works, the style system is live, and the feature seams are in place.
