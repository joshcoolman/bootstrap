# Part: CI/CD

> **Note.** This file compares against `vite-app`, a Vite + TanStack Router
> shell that has since been deleted. Those comparisons are kept because the
> *reasoning* still explains why each choice was made — but the referent is
> history, not something you can go read. Next.js is the only shell now.

Same three tools as `vite-app`, same division of labor, no overlap:

- **Vercel** — native Git integration for deploys and previews
- **GitHub Actions** — lint + test + build quality gate
- **Claude GitHub App** — `@claude` in PRs for review, questions, and fixes

## Vercel deployment

Use Vercel's native Git integration — connect the repo in the Vercel
dashboard and it handles the rest. Preview deployments on every PR,
production on push to `main`. No `vercel.json` needed. This matters more
here than it did for `vite-app`: the whole reason this skill exists is to
host real server-side compute (API routes, server actions), and Vercel's
native Next.js integration is what makes that work reliably — the exact gap
that made a plain Vite SPA insufficient in the first place.

Do NOT use the Vercel CLI + GitHub Actions approach unless you have a
specific reason (custom pre-deploy steps, GitHub Enterprise). The native
integration is strictly simpler.

## GitHub Actions — quality gate

A single workflow that runs on every push and PR. Node 22, matching what
Next 16 requires (`engines.node` in `package.json`):

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

      # No `version:` — package.json's packageManager field is the single source
      # of truth. Specifying both makes pnpm/action-setup@v4 error out.
      - uses: pnpm/action-setup@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
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

`pnpm build` runs `next build`, which type-checks the whole project natively
— lint + test + build together are the complete quality signal, same as
`vite-app`, and it must stay green with **zero secrets and zero env vars**
for a bare scaffold with no backend wired up yet.

**`pnpm/action-setup` deliberately has no `version:`.** `package.json`'s
`packageManager` field pins pnpm (see `shell.md`), and `pnpm/action-setup@v4`
errors out when a version is given in both places — a failure that breaks every
PR at once, independent of whatever change is under test. Pin in exactly one
place, and let it be `packageManager`, since the deploy builder reads that too
and the workflow file is not.

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
human developer would read. No additional configuration needed.

## What "done" looks like

- Pushing a branch opens a Vercel preview URL automatically
- Opening a PR triggers the CI workflow — lint, test, and build must pass
- `@claude` is available in PR comments for review or questions
- Merging to `main` triggers a Vercel production deployment
