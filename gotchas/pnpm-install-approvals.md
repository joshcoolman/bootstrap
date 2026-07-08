# pnpm build-script approvals (`ERR_PNPM_IGNORED_BUILDS`)

Check this before trusting a green `pnpm install` on your own machine —
local success here is not proof it'll build anywhere else.

## The trap

pnpm blocks a dependency's native/postinstall build scripts unless
explicitly trusted. In pnpm v11 the trust list is `allowBuilds` in
`pnpm-workspace.yaml` (bare package name = all versions approved). The
**older key name, `onlyBuiltDependencies`, was removed in v11 and is
silently ignored** — a config file that reads correctly can be dead.

A fresh CI/deploy environment (Railway, GitHub Actions, a clean clone) has
no prior approvals and will hard-fail `pnpm install --frozen-lockfile`
with:

```
[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: esbuild@x, esbuild@y, ...
```

**Why your machine won't show this:** pnpm's global store caches build
approvals across *all* projects on that machine. If you've ever approved
`esbuild`/`lightningcss`/etc. for some other project, every new project
inherits that trust silently — `pnpm install` looks clean locally even
when the committed config is broken. Real verification requires a clean
environment (the actual CI/host run, or a wiped pnpm store) — not a local
green install.

## What to do at scaffold time

1. After the first `pnpm install`, check for `ERR_PNPM_IGNORED_BUILDS`
   even if the command exits 0 locally — it may only warn, not fail, on a
   machine with prior approvals.
2. Run `pnpm approve-builds` (interactive) or `pnpm approve-builds --all`
   (non-interactive) and commit the resulting `pnpm-workspace.yaml`.
3. Use the **current** key — `allowBuilds:` (a map of `package: true/false`)
   — not `onlyBuiltDependencies` / `ignoredBuiltDependencies` / etc. Those
   were removed in pnpm v11.
4. Don't treat a local `pnpm install` as proof the deploy will build. The
   first real test is the CI/host's fresh install, with an empty pnpm
   store.

First hit in: `prompt-smith`, PR #5 (`fix/railway-pnpm-allowbuilds`) —
3 consecutive Railway build failures traced to this, confirmed directly
from the Railway build log for deployment `e90054eb`.
