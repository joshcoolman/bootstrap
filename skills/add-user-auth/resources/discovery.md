# Discovery

This skill runs inside an existing app. Before writing anything, read the repo
and fill in the adaptation slots every later resource references. Assume the
stack (Next.js App Router + pnpm, `next-app`-shaped); never assume the shape.

Unlike `add-simple-auth`'s fixed "ask 4 questions" opener, default to
`next-app`'s tone: **infer everything you can from the repo, only ask when
something is genuinely ambiguous.** Run the probes below silently, report the
summary, and only stop for a question if a probe surfaces something that
needs a judgment call (an existing `/login` or auth remnant, a non-empty slot
that should be empty, an already-adopted table name that would collide).

## Preflight — confirm the stack, or stop

Check `package.json` and the repo layout for all of:

- `next` (App Router — a `src/app/` tree, not `pages/`)
- `pnpm-lock.yaml` (pnpm is the package manager)
- TypeScript strict mode (`tsconfig.json`)

If any is missing, stop and tell the user this skill targets Next.js App
Router + pnpm apps (the `next-app` shape) — do not improvise a port to
another stack. This skill does **not** target Vite/TanStack Router apps —
that's `add-simple-auth`'s territory, and it's a single shared-credential
gate, not what this skill builds.

## Concrete probes (mechanical — read these files)

- **Root `CLAUDE.md` and `docs/`** — how the repo orients agents; which docs
  exist (`OVERVIEW.md`? `SPEC.md`? `PLAN.md`? `STYLE.md`?). You will edit some
  of these in the docs re-weld step, and add a new `docs/FEATURE-MODULE-PATTERN.md`.
- **`package.json`** — scripts (note the dev port from the `dev` script —
  this is `<dev-port>`), any existing `setup*` script names (avoid
  collisions), test runner, lint/format commands. Confirm no `@supabase/ssr`
  or `@supabase/supabase-js` are already installed (if they are, another
  auth attempt may already exist — surface this, don't silently layer over
  it).
- **`tsconfig.json`** — the import alias. `next-app` uses `#/*` → `./src/*`;
  confirm it's the same here. This is `<alias>`. Use it in every import you
  write; never use relative `../../` paths if an alias exists.
- **`src/app/`** — confirm `layout.tsx` exists and note what it already
  renders (theme init/toggle, fonts). Check that `/login` and the guarded
  route (default `/dashboard`) don't already exist. Check there is no
  existing `src/proxy.ts` or `middleware.ts` (if there is, this skill would
  be editing existing gating logic, not adding fresh — surface this rather
  than silently overwriting).
- **`src/features/`** — confirm the `<name>/{CLAUDE.md,types.ts,index.ts}`
  convention `next-app` establishes. This is `<features-dir>`. If a
  `src/features/auth/` or similarly-named folder already exists, surface it.
- **`.gitignore` + `.env*` files** — is `.env.local` ignored (a `*.local`
  glob counts, `next-app`'s does)? Does anything already reference Supabase
  env vars? Is `supabase/` or `supabase/.temp` ignored?
- **CI workflow** (`.github/workflows/ci.yml`) — confirm it runs
  install → lint → test → build with no secrets (`next-app`'s does).
  Everything you write must stay green with zero env vars.
- **Styling idiom** — read `src/app/page.tsx` or the docs viewer for the
  exact Tailwind utility classes in use (`bg-bg`, `text-text`, `border-border`,
  `bg-surface`, `text-accent`, …). The login form and guarded-route markup
  in `routes.md` should match this idiom (it's the same Paper & Ink token
  system `next-app` and the reference app `~/repos/effective` both use, so
  it should need little to no restyling — but confirm the tokens actually
  exist in this repo's `tokens.css` rather than assuming).

## Judgment extractions

- **Docs that assert "no auth" or "no backend".** Grep `docs/` and root
  `CLAUDE.md`/`README.md` for claims like this. List every file and line —
  the docs re-weld step edits all of them.
- **Docs viewer pickup.** `next-app`'s `/docs` route globs `docs/` and
  `knowledge/` — a new `docs/FEATURE-MODULE-PATTERN.md` will appear there
  automatically; no viewer code changes needed.
- **The optional example feature's table name.** If offered and accepted
  (see SKILL.md Step 1), confirm `user_snippets` doesn't collide with an
  existing table name the user already has in mind for this project. If it
  might, ask for an alternative rather than guessing.

## Output — the discovery summary

Before touching anything, report to the user (and keep for your own
reference):

```
Preflight: ✓ Next.js App Router + pnpm + TypeScript strict
<alias>        = #/            (tsconfig paths)
<features-dir> = src/features/<name>/ with CLAUDE.md + types.ts + index.ts
<dev-port>     = 3000          (from the dev script)
Styling        = design-token Tailwind utilities (bg-surface, text-text-muted, …)
Docs to re-weld: docs/SPEC.md:19, docs/OVERVIEW.md:12
Existing routes: /, /docs — /login and /dashboard are free
No existing src/proxy.ts or middleware.ts
CI: install → lint → test → build, no secrets
```

Every later resource references these slots. If a probe surprised you (no
alias, an existing `/login` remnant, a table name collision), surface it
now — do not silently adapt.
