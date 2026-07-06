# Discovery

This skill runs inside an existing app. Before writing anything, read the repo
and fill in the adaptation slots every later resource references. Assume the
stack (Vite + React + TanStack Router + pnpm); never assume the shape.

## Preflight — confirm the stack, or stop

Check `package.json` and the repo layout for all of:

- `vite` + `@vitejs/plugin-react` (or equivalent React plugin)
- `@tanstack/react-router` + file-based routes under `src/routes/`
- `pnpm-lock.yaml` (pnpm is the package manager)

If any is missing, stop and tell the user this skill targets Vite + React +
TanStack Router + pnpm apps — do not improvise a port to another stack.

## Concrete probes (mechanical — read these files)

- **Root `CLAUDE.md` and `docs/`** — how the repo orients agents; which docs
  exist (SPEC? PLAN? OVERVIEW? a using-this-repo note?). You will edit some of
  these in the docs re-weld step.
- **`package.json`** — scripts (note the dev port from the `dev` script — this
  is `<dev-port>`), any existing `setup*` script names (avoid collisions),
  test runner, lint/format commands.
- **`tsconfig.json` + `vite.config.ts`** — the import alias (`paths` /
  `resolve.alias`). This is `<alias>` (e.g. `#/*` → `./src/*`). Use it in every
  import you write; never use relative `../../` paths if an alias exists.
- **`eslint.config.js`** — flat config or legacy? Which globals blocks exist?
  You will add a Node-globals block for `scripts/` (see the setup-wizard
  resource).
- **`src/routes/`** — confirm `__root.tsx` exists; check that `/login` and the
  protected route (default `/dashboard`) don't already exist; note whether
  `src/routeTree.gen.ts` is present (it must be regenerated and committed
  after you add routes — see the routes resource).
- **`.gitignore` + `.env*` files** — is `.env.local` ignored (a `*.local` glob
  counts)? Does `.env.example` exist? Is `supabase/` or `supabase/.temp`
  ignored?
- **CI workflow** (`.github/workflows/`) — confirm what it runs. If it runs
  install → lint → test → build with no secrets, everything you write must
  stay green with zero env vars. (The auth-feature resource is designed for
  exactly this.)

## Judgment extractions (read the code, form a view)

- **`<features-dir>` — the feature convention.** Where does isolated feature
  code live and what does a feature look like? (Example shape: a
  `src/features/<name>/` folder with a `CLAUDE.md` boundary doc, a `types.ts`
  contract, and a public `index.ts` that consumers import exclusively.) Your
  auth feature must look like a native citizen of whatever convention you
  find. If there is no feature convention, use `src/features/auth/` and say so
  in your discovery summary.
- **Styling idiom.** Design-token utility classes? Plain Tailwind? CSS
  modules? Read an existing component (a home page, a nav) and note the exact
  idiom — the login/dashboard markup in the routes resource must be restyled
  to match it.
- **Docs that assert "no auth".** Grep the docs for claims like "no backend",
  "no auth", "no database". List every file and line — the docs re-weld step
  edits all of them.
- **Docs viewer pickup.** If the repo has an in-app docs viewer, note what it
  globs — files you add to `docs/` may appear in the running app.

## Output — the discovery summary

Before touching anything, report to the user (and keep for your own
reference):

```
Preflight: ✓ Vite + React + TanStack Router + pnpm
<alias>        = #/           (tsconfig paths)
<features-dir> = src/features/<name>/ with CLAUDE.md + types.ts + index.ts
<dev-port>     = 5180         (from the dev script)
Styling        = design-token Tailwind utilities (bg-surface, text-text-muted, …)
Docs to re-weld: docs/SPEC.md:19, docs/00-using-this-repo.md:9, docs/OVERVIEW.md:17
Existing routes: /, /docs — /login and /dashboard are free
CI: install → lint → test → build, no secrets
```

Every later resource references these slots. If a probe surprised you
(no alias, no docs, an existing auth remnant), surface it now — do not
silently adapt.
