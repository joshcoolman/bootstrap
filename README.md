# bootstrap

Battle-tested Claude Code skills for scaffolding and layering web apps. Every
skill here was developed by building the real thing in a live repo first, then
codifying what actually worked — so what's packaged is validated sequencing,
gates, and gotcha ledgers, not templates.

## Install (as a Claude Code plugin)

```
/plugin marketplace add joshcoolman/bootstrap
/plugin install bootstrap@bootstrap
```

Then, from any directory — including a brand-new empty folder:

```
mkdir my-app && cd my-app && claude
> /bootstrap:vite-app      # scaffold the shell (git init + files + install)
> /bootstrap:add-simple-auth # later: layer on a single shared-credential Supabase login (not multi-user auth)
```

Or, for an app that needs real server-side compute:

```
mkdir my-app && cd my-app && claude
> /bootstrap:next-app      # scaffold a Next.js App Router shell instead
> /bootstrap:add-user-auth # later: layer on genuine multi-user Supabase auth
```

Skills also auto-trigger from plain requests ("scaffold a new app here",
"add auth to this app"). For local development of this repo itself, load it
directly: `claude --plugin-dir ~/repos/bootstrap` (then `/reload-plugins`
after edits).

## Skills

| Skill | What it does |
|-------|--------------|
| [vite-app](skills/vite-app/SKILL.md) | Scaffold a new Vite + React + TanStack Router shell on the Paper & Ink design system — runnable app, docs viewer, feature seams, CI. |
| [next-app](skills/next-app/SKILL.md) | Scaffold a new Next.js App Router shell on the same Paper & Ink design system, for apps that need real server-side compute, secrets, or (later) per-user auth. v1 — validated by porting from a live Next.js migration, not yet run as its own consumer. |
| [add-simple-auth](skills/add-simple-auth/SKILL.md) | Layer a single shared-credential Supabase email/password gate onto an existing Vite app — not multi-user auth, one login guards the whole app: vendor-agnostic seam, `/login` + guarded `/dashboard`, and an interactive setup wizard that also locks down public sign-ups. |
| [add-user-auth](skills/add-user-auth/SKILL.md) | Layer genuine multi-user Supabase auth onto an existing `next-app`-scaffolded Next.js app — distinct accounts, per-user data isolation via Postgres RLS + a service-layer identity check, an optional example feature proving the isolation end-to-end, and an interactive setup wizard. v1 — drafted directly from a live reference app, not yet run as its own consumer. |
| [deploy-next-railway](skills/deploy-next-railway/SKILL.md) | Deploy an existing Next.js App Router app to Railway — provision Postgres and a Storage Bucket if needed, wire env vars (internal `DATABASE_URL` reference, session secret, bucket credentials), set the Pre-Deploy Command for migrations, and verify live via the deploy log and browser. v1 — extracted directly from the `next-railway-app` recipe's Step 6, itself proven only in individually-tested pieces, not yet run as one integrated flow. |

`vite-app` and `next-app` both create a repo from nothing — pick the shell
based on whether the app needs a server. `add-*` skills layer onto an
existing app: they assume the stack but *discover* the repo's shape —
feature conventions, import alias, styling idiom — and adapt to it, so
they aren't welded to either shell's output. `add-simple-auth` targets
`vite-app`; `add-user-auth` targets `next-app` — pick the auth skill that
matches the shell underneath. `deploy-next-railway` is a different kind of
layering skill: it targets Railway project config rather than repo files,
and pairs with the `next-railway-app` recipe's earlier steps rather than a
shell skill directly.

## Parts and recipes (the human-readable form)

The same knowledge also lives as instruction-level docs — what to build and
why, not line-by-line code — for reading, or for applying a single slice to an
existing repo:

| Part | What it covers |
|------|---------------|
| [shell](parts/shell.md) | Scaffold: pnpm, Vite, TanStack Router, React 19, TypeScript strict, Tailwind v4, Vitest, ESLint, Prettier, Vercel |
| [styles](parts/styles.md) | Paper & Ink design system: tokens, dark mode, fonts, theme toggle |
| [docs](parts/docs.md) | Docs folder structure + in-app `/docs` viewer (react-markdown) |
| [knowledge](parts/knowledge.md) | Knowledge folder + feature seam pattern (`src/features/`) |
| [cicd](parts/cicd.md) | GitHub Actions quality gate + Vercel native deploy + Claude GitHub App |

| Recipe | What it produces |
|--------|-----------------|
| [vite-react-app](recipes/vite-react-app.md) | Runnable shell with all five parts composed — the standard starting point for new public apps |
| [next-selfhost-app](recipes/next-selfhost-app.md) | Self-hostable Next.js app: SQLite, local disk storage, optional single-user auth, BYOK, Docker/volume deploy to Railway or Fly — zero required vendor accounts |
| [next-railway-app](recipes/next-railway-app.md) | Next.js on Railway: Postgres, bucket storage, whitelist-only multi-user auth (no Supabase), proven via a todos CRUD demo |

## How this repo grows

New skills are developed, not written: build the feature for real in a
throwaway "mule" repo, log every friction point, validate end-to-end, then
codify. The full authoring contract lives in [CLAUDE.md](CLAUDE.md).

## License

[MIT](LICENSE). Free — install it, fork it, build your own bootstrap on top
of it. This repo tracks what works for my own projects, so I'm not seeking
contributions and there's no support: if something doesn't work the way you
want, fork it and fix it.

## Status

**Last shipped:** two skills drafted together on one branch, both by the
same deviation — drafted directly from a live, proven reference app
(`~/repos/effective`) rather than a fresh disposable mule, each noted as a
one-off exception to this repo's usual mule-first loop:

- [#2 — next-app](https://github.com/joshcoolman/bootstrap/issues/2) v1.
  Already run once end-to-end as a fresh consumer in a scratch folder,
  which surfaced and fixed real friction: an `eslint@10` +
  `eslint-config-next` peer mismatch that crashed `pnpm lint` outright (now
  pinned to `eslint@^9`), a hash-sync `useEffect` that tripped
  `react-hooks/set-state-in-effect` (now a lazy `useState` initializer —
  also fixed a real flash-of-wrong-doc bug), two plain anchors swapped for
  `next/link`, and a `tokens.css` gap shared with `vite-app`
  (spacing/radius/type-scale/motion/border-width were only ever described
  in a comment, never declared — now filled in). All five gates pass clean
  with zero env vars. One non-blocking Turbopack build warning remains,
  documented as accepted in `resources/docs.md`.
- [#1 — add-user-auth](https://github.com/joshcoolman/bootstrap/issues/1)
  v1, layered on top of `next-app`. The issue assumed a
  `FEATURE-MODULE-PATTERN.md` already existed to draw enforcement
  discipline from — it didn't, anywhere; the discipline (an identity port
  every feature depends on, RLS + an explicit `.eq('user_id', ...)` filter
  as double enforcement, a fixed discriminated-union error taxonomy, checks
  living in the service/use-case layer not the UI) only existed as code and
  comments in the reference app, so this skill is what writes it down for
  the first time, as a `docs/FEATURE-MODULE-PATTERN.md` it installs into
  consuming repos. Reimplements the reference app's Effect-based identity
  port as plain async/await TypeScript (`next-app` has zero extra runtime
  deps today; no reason to add `effect` just to port this). Ships an
  optional example feature (`snippets`, deliberately not `todos` — a prior
  naming collision from repeated demo builds) proving per-user isolation
  end-to-end when accepted. **Not yet run end-to-end as a fresh
  consumer** — that's the next real test for both skills.

**Up next:** integration-test both as a consumer, in sequence: scaffold a
throwaway repo with `/bootstrap:next-app`, then run
`/bootstrap:add-user-auth` against it and drive it through a real
(disposable) Supabase project — both without and with the optional
`snippets` feature — logging whatever friction surfaces, the same way
`next-app`'s own first live run did.
