---
name: next-app
description: Scaffold a complete Next.js App Router app — Paper & Ink design system, a self-hosted login (scrypt + signed cookie, no vendor, no database), deny-by-default route protection, an empty dashboard, docs viewer, feature seams, and CI. The whole starting point in one pass, ready to build an idea on top of. Use when asked to scaffold a new app, start a new project, create a starter with a login, or bootstrap a Next.js app.
---

You are about to scaffold a complete, production-ready Next.js App Router app —
including auth. Read each resource file in order before writing any code; they
contain the exact file contents, CSS values, and implementation details.

## What this produces

A running, deployable app that is **already protected**, with nothing built on
top of it yet:

- Styled home page and the Paper & Ink design system
- `/login` — email + password, self-hosted, no signup route
- `/dashboard` — empty, guarded, ready to build in
- Deny-by-default middleware: everything except `/login` requires a session
- `/docs` viewer reading the repo's own markdown
- Feature seams, CI, and a `pnpm auth:add-user` script

The intended use is one exploration per app: scaffold, build the single idea,
deploy, and tear it down when it stops being interesting. Nothing here assumes
the app will grow into a product.

**Auth is allowlist-shaped** — no signup, no password reset, no OAuth, no
roles. Access is granted out-of-band by adding an entry to `AUTH_USERS`. This
is deliberate and covers "just me" and "me plus whoever I show it to." If an
app ever needs real user accounts, that is the signal to adopt `better-auth`,
not to extend this.

## Before you start — check first, only ask if genuinely blocked

Look at the target directory before asking anything (default: the current
directory — asking questions unconditionally on an empty throwaway folder is
pure friction with no payoff).

**If the directory is empty** (no files, or only OS cruft like `.DS_Store`, or
an empty `.git`): infer everything below and go straight to Step 1 — no
questions.

- **App name** — the directory's own basename, kebab-cased if it isn't already.
  Used in `package.json`, `app-meta.ts`, the docs sidebar, and the session
  cookie name.
- **GitHub repo URL** — a plain placeholder,
  `https://github.com/your-username/<app-name>`. Never call `gh` or otherwise
  look up who the user is — this field is cosmetic, so a clearly-fake value
  costs nothing and doesn't assume an identity that wasn't given. This skill
  only runs `git init` locally; it never creates or queries anything on GitHub.
- **Target directory** — the current directory. Always.

**If the directory is NOT empty:** stop and ask before writing anything. Name
exactly what you found — "this already looks like a git repo with a
`package.json`" / "there are already N files here, including an `images/`
folder — is that meant to be part of this app, or should I use a different
directory?" Don't guess whether existing content belongs to the scaffold or is
unrelated clutter; that's the user's call.

## Resources — read these in order

- [Shell](resources/shell.md) — stack, all config files, entry points, exact `next.config.ts`
- [Styles](resources/styles.md) — Paper & Ink design system, exact CSS values, @theme bridge, theme-init/theme-toggle
- [Auth](resources/auth.md) — session layer, credentials, middleware, routes, tests, the user-add script (verbatim code)
- [Docs](resources/docs.md) — docs/ folder structure and the full `/docs` viewer implementation
- [Knowledge](resources/knowledge.md) — feature seams and knowledge/ folder pattern
- [CI/CD](resources/cicd.md) — GitHub Actions workflow

## Build order

Execute each step fully before moving to the next. Confirm `pnpm install`
succeeds after Step 1 before writing any source files.

### Step 1 — Shell

`git init` if `.git` doesn't already exist. Write all config and entry files per
the shell and styles resources: `package.json`, `next.config.ts`,
`postcss.config.mjs`, `eslint.config.mjs`, `tsconfig.json`, `vitest.config.ts`,
`pnpm-workspace.yaml`, `src/test-setup.ts`, `src/test-server-only-stub.ts`,
`.gitignore`, `src/app-meta.ts`, `src/app/layout.tsx`, `src/app/page.tsx`,
`src/components/theme-init.tsx`, `src/components/theme-toggle.tsx`.

Run `pnpm install`. Fix any install errors before continuing.

Do not skip `pnpm-workspace.yaml` on the grounds that this is a single-package
repo — it carries the `allowBuilds` approvals, without which `pnpm install`
exits 1 on any clean machine. It will still appear to work on the machine you
are scaffolding on, so treat the file as required rather than verifying by
running the install once. See `shell.md`.

### Step 2 — Styles

Write the four CSS files per the styles resource: `src/styles/tokens.css`,
`base.css`, `typography.css`, `index.css`.

Already wired — `layout.tsx` imports `#/styles/index.css` and renders
`ThemeInit`/`ThemeToggle`. Confirm the app runs with `pnpm dev`, the home page
renders styled, and the theme toggle works.

### Step 3 — Auth

Per the auth resource, write:

- `src/features/auth/` — `hash.mjs`, `hash.d.mts`, `session.ts`,
  `credentials.ts`, `current-user.ts`, `actions.ts`, `index.ts`
- `src/features/auth/session.test.ts` and `credentials.test.ts`
- `src/proxy.ts` — deny-by-default middleware
- `src/app/login/page.tsx` and `src/app/dashboard/page.tsx`
- `scripts/add-user.mjs`, plus the `auth:add-user` entry in `package.json`
- `AUTH_SESSION_SECRET` and `AUTH_USERS` in `.env.example`

Login and dashboard markup follows the Paper & Ink idiom established in Step 2 —
the auth resource specifies behaviour and state, not styling. The dashboard is
deliberately empty: a heading, the sign-out control, and nothing else.

Two things the resource explains and you must not shortcut: middleware imports
`./session` directly and never the barrel (Edge bundle), and the `env` backend
hashes the email into an opaque id (the session value splits on `.`).

**Gate:** `pnpm test` green — the session and credentials suites both pass.
`pnpm build` green with zero env vars set.

### Step 4 — Docs

Write the markdown files in `docs/`: `OVERVIEW.md`, `SPEC.md`,
`CODE-STANDARDS.md`, `STYLE.md`. `SPEC.md` must state the auth boundary —
allowlist, no signup, no server-side revocation. `CODE-STANDARDS.md` is the
repo's own copy of the project standard (`docs.md` gives the template) — it is
what makes a freshly-scaffolded app carry the conventions it's meant to follow.

Do **not** write `docs/PLAN.md`. The standard is explicit: plans, tasks, and
bugs live as GitHub issues, never as a markdown file in `docs/`. Build order and
"what's next" belong in the README `## Status` block (Step 7) and in issues.

Write `src/features/docs/build-docs.ts`, `src/app/docs/page.tsx`, and
`src/components/docs-viewer.tsx` using the full implementation from the docs
resource.

Add `react-markdown` and `remark-gfm`, run `pnpm install`, confirm `/docs`
loads and all four files appear in the sidebar. Note `/docs` is gated like
everything else.

### Step 5 — Knowledge

Create `src/features/` seams beyond auth: `core` and `knowledge`. Give
`knowledge` a `CLAUDE.md` — its server-only boundary is a real invariant that
fails silently if violated, exactly what a boundary `CLAUDE.md` is for. `core`
gets one only once it owns something non-obvious; an empty seam with nothing to
say correctly has no `CLAUDE.md` (per the standard — a missing one is right when
there's nothing to warn about). Don't invent app-specific seams — the app
doesn't exist yet, and empty folders named for features that were never built
are noise.

Write `src/features/knowledge/index.ts` with the `fs` + `server-only` loader,
and create `knowledge/` at the repo root with placeholder `guidance.md` and
`rubric.md`.

### Step 6 — CI/CD

Write `.github/workflows/ci.yml` per the cicd resource. It must run lint, test,
and build with **no env vars set** — the app has to build unconfigured.

### Step 7 — Verify, then launch

Run `pnpm build`, `pnpm test`, `pnpm lint` — all must pass with zero env vars.

Then verify auth for real, in a browser. This is the step that catches what
tests can't:

1. Generate a credential: `pnpm auth:add-user 'you@example.com' '<password>'`
2. Write `AUTH_SESSION_SECRET` (`openssl rand -hex 32`) and the printed
   `AUTH_USERS` entry into `.env.local`
3. Start `pnpm dev` as a tracked background process (not shell `&`) so it's
   stoppable rather than orphaned, and poll until it responds
4. Drive the flow: `/dashboard` redirects to `/login` → wrong password shows an
   error **and preserves the typed email** → correct password lands on
   `/dashboard` → hard reload stays signed in → sign out returns to `/login`
   and `/dashboard` is gated again

**Gate:** every one of those observed in a browser, not inferred from tests.
The React 19 uncontrolled-field reset in particular only shows up when a real
form is driven.

Write `CLAUDE.md` at the repo root (agent orientation: what the app does,
current phase, what to read first) and `README.md` (public summary: stack,
auth boundary, local dev commands, how to add a user).

The README **must** end with a `## Status` block — the standard makes it the
primary cross-session orientation surface, paired with open issues. Seed it so
`/where-are-we` works from day one:

```markdown
## Status

**Last shipped:** Phase 0 scaffold — styled home, self-hosted login, guarded
empty dashboard, `/docs` viewer, CI green with zero env vars.

**Up next:** build the one idea in `/dashboard`. Plans and tasks live as GitHub
issues, not here.
```

Finish at the handoff moment, not a pass/fail report: leave the dev server
running, open the browser to the home page, and tell the user they're signed in
and where `/docs` lives. The point is that they pick up from a live app.

## What the finished repo looks like

```
your-app/
├── CLAUDE.md
├── README.md
├── .env.example                      ← AUTH_SESSION_SECRET, AUTH_USERS
├── docs/
│   ├── OVERVIEW.md  SPEC.md  CODE-STANDARDS.md  STYLE.md
├── knowledge/
│   ├── guidance.md  rubric.md
├── scripts/
│   └── add-user.mjs                  ← prints a credential to paste
├── src/
│   ├── app-meta.ts
│   ├── test-setup.ts  test-server-only-stub.ts
│   ├── proxy.ts                      ← deny-by-default middleware
│   ├── app/
│   │   ├── layout.tsx                ← fonts + theme wiring
│   │   ├── page.tsx                  ← home (gated)
│   │   ├── login/page.tsx            ← the only public route
│   │   ├── dashboard/page.tsx        ← empty, gated — build here
│   │   └── docs/page.tsx             ← Server Component, reads fs
│   ├── components/
│   │   ├── theme-init.tsx  theme-toggle.tsx  docs-viewer.tsx
│   ├── features/
│   │   ├── auth/
│   │   │   ├── CLAUDE.md
│   │   │   ├── hash.mjs  hash.d.mts  ← shared with scripts/add-user.mjs
│   │   │   ├── session.ts            ← Edge-safe, Web Crypto only
│   │   │   ├── credentials.ts        ← scrypt, AUTH_USERS allowlist
│   │   │   ├── current-user.ts       ← throws when unauthenticated
│   │   │   ├── actions.ts            ← login / logout
│   │   │   ├── index.ts              ← barrel — never Edge-imported
│   │   │   └── session.test.ts  credentials.test.ts
│   │   ├── core/                       ← seam; no CLAUDE.md until it owns something
│   │   ├── docs/build-docs.ts
│   │   └── knowledge/
│   │       ├── CLAUDE.md
│   │       └── index.ts              ← server-only
│   └── styles/
│       ├── index.css  tokens.css  base.css  typography.css
├── .github/workflows/ci.yml
├── package.json  next.config.ts  postcss.config.mjs
├── eslint.config.mjs  tsconfig.json  vitest.config.ts
├── pnpm-workspace.yaml               ← allowBuilds; required, not optional
└── .gitignore
```

Phase 0 complete. The dev server is running, the browser is open, and you're
signed in. The style system is live, `/docs` works, the app is protected, and
`/dashboard` is empty and waiting. Build the one idea there.

To take it live: `/bootstrap:deploy-next-railway`.
