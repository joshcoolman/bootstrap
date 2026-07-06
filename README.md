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

Skills also auto-trigger from plain requests ("scaffold a new app here",
"add auth to this app"). For local development of this repo itself, load it
directly: `claude --plugin-dir ~/repos/bootstrap` (then `/reload-plugins`
after edits).

## Skills

| Skill | What it does |
|-------|--------------|
| [vite-app](skills/vite-app/SKILL.md) | Scaffold a new Vite + React + TanStack Router shell on the Paper & Ink design system — runnable app, docs viewer, feature seams, CI. |
| [add-simple-auth](skills/add-simple-auth/SKILL.md) | Layer a single shared-credential Supabase email/password gate onto an existing app — not multi-user auth, one login guards the whole app: vendor-agnostic seam, `/login` + guarded `/dashboard`, and an interactive setup wizard that also locks down public sign-ups. |

`vite-app` creates a repo from nothing. `add-*` skills layer onto an existing
app: they assume the stack (Vite + React + TanStack Router + pnpm) but
*discover* the repo's shape — feature conventions, import alias, styling
idiom — and adapt to it, so they aren't welded to the vite-app output.

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

**Last shipped:** renamed `add-auth` → `add-simple-auth` for transparency —
the skill was always a single shared-credential gate (not multi-user auth),
but the name didn't say so.

**Up next:** [#1 — add-user-auth](https://github.com/joshcoolman/bootstrap/issues/1),
real per-user auth plus the enforcement discipline from
`FEATURE-MODULE-PATTERN.md` (auth/authorization checks in the use-case/data
layer, not the UI). Not started — needs its own mule-repo development pass
per the admission bar above, probably starting from a clone of
`~/repos/view-down`.
