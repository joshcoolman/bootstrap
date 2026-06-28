# Part: CI/CD

Three tools, each doing one thing, no overlap:

- **Vercel** — native Git integration for deploys and previews
- **GitHub Actions** — lint + test + build quality gate
- **Claude GitHub App** — `@claude` in PRs for review, questions, and fixes

## Vercel deployment

Use Vercel's native Git integration — connect the repo in the Vercel dashboard
and it handles the rest. Preview deployments on every PR, production on push to
`main`. No `vercel.json` needed for a Vite SPA; no GitHub Actions secrets to
manage for deployment.

Do NOT use the Vercel CLI + GitHub Actions approach unless you have a specific
reason (custom pre-deploy steps, GitHub Enterprise). For these repos, the native
integration is strictly simpler.

## GitHub Actions — quality gate

A single workflow that runs on every push and PR. Acts as the gate before
merges land.

**`.github/workflows/ci.yml`:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: latest

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - name: Install dependencies
        run: pnpm install

      - name: Lint
        run: pnpm lint

      - name: Test
        run: pnpm test

      - name: Build
        run: pnpm build
```

The `build` step catches TypeScript errors and bundler failures in addition to
producing a deployable artifact. Lint + test + build together are the complete
quality signal.

## Claude GitHub App — AI review

Install the Claude GitHub App on your repositories. Once installed, `@claude`
is available in any PR comment or issue thread.

```
@claude run a code review
@claude why is this test failing?
@claude fix the type errors in src/features/improve/types.ts
```

Claude reads `CLAUDE.md` at the repo root (and any feature-level `CLAUDE.md`
files under `src/features/`) as its orientation context — the same files a
human developer would read. No additional configuration needed; the CLAUDE.md
files that are already part of the baseline are sufficient.

**Why Claude over Vercel Agent for AI review:** if you're already using
Anthropic tools (Claude Code, the Anthropic SDK), Claude is the coherent
choice — same model, same context files, no middleman. Vercel Agent is a
separate AI layer that duplicates capability you already have.

## What "done" looks like

- Pushing a branch opens a Vercel preview URL automatically
- Opening a PR triggers the CI workflow — lint, test, and build must pass
- `@claude` is available in PR comments for review or questions
- Merging to `main` triggers a Vercel production deployment
