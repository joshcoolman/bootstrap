# pnpm build-approvals preflight

Covers Step 2 — run this **before** the first `railway up`, as an actual
check with a gate, not background reading. Full reasoning lives in
[gotchas/pnpm-install-approvals.md](../../../gotchas/pnpm-install-approvals.md) —
read it, don't re-derive it. Summary of the mechanics and why this is a
preflight step, not a note:

## The trap, briefly

pnpm v11 replaced `onlyBuiltDependencies` with `allowBuilds` in
`pnpm-workspace.yaml`. A local `pnpm install` can look completely clean even
when this key is missing or stale — the global pnpm store on your machine
caches prior build approvals from other projects, silently papering over the
gap. A fresh Railway build environment has no such cache: the first deploy
build fails outright with `ERR_PNPM_IGNORED_BUILDS`, and this is a real,
previously-hit failure mode (three consecutive Railway build failures in one
project's history before this was understood).

## The check

1. Open `pnpm-workspace.yaml` and confirm an `allowBuilds` key exists and is
   current — i.e. it lists every package with a native build step that the
   app's actual dependencies pull in right now (e.g. `sharp` if image
   processing is used, `unrs-resolver` and similar transitive natives).
   Don't assume it's fine just because it exists; a dependency added after
   the key was last written won't be covered.
2. If missing or stale, run:
   ```
   pnpm approve-builds --all
   ```
   and confirm the resulting `pnpm-workspace.yaml` diff looks sane (only
   `allowBuilds` entries added/changed, nothing else).
3. Commit the updated `pnpm-workspace.yaml` if it changed — this must be in
   the repo Railway builds from, not just present locally.

## Gate

Confirmed by **reading the file**, not by a locally-green `pnpm install` —
that green result is exactly the false signal this check exists to catch.
