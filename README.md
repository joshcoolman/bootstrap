# bootstrap

Modular setup guides for getting new repos to a known baseline. Each part is
self-contained and executable by a coding agent. Recipes compose parts into a
complete starting point.

## Parts

| Part | What it covers |
|------|---------------|
| [shell](parts/shell.md) | Scaffold: pnpm, Vite, TanStack Router, React 19, TypeScript strict, Tailwind v4, Vitest, ESLint, Prettier, Vercel |
| [styles](parts/styles.md) | Paper & Ink design system: tokens, dark mode, fonts, theme toggle |
| [docs](parts/docs.md) | Docs folder structure + in-app `/docs` viewer (react-markdown) |
| [knowledge](parts/knowledge.md) | Knowledge folder + feature seam pattern (`src/features/`) |
| [cicd](parts/cicd.md) | GitHub Actions quality gate + Vercel native deploy + Claude GitHub App |

## Recipes

| Recipe | What it produces |
|--------|-----------------|
| [vite-react-app](recipes/vite-react-app.md) | Runnable shell with all four parts composed — the standard starting point for new public apps |

## How to use

For a new app, follow the recipe in order. For targeted additions to an
existing repo, apply individual parts. Parts are written at instruction level —
they describe what to build and why, not line-by-line code. A coding agent
(or a human) should be able to execute any part or recipe directly.
