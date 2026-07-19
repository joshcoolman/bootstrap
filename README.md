# bootstrap

Claude Code skills for standing up small, private web apps fast — the kind you
build to explore one idea, put online behind a login, and tear down when it
stops being interesting.

Every skill here was developed by building the real thing in a live repo first,
then codifying what actually worked — so what's packaged is validated
sequencing, gates, and gotcha ledgers, not templates.

## The stack, and why it's only one

One framework, one host, one auth approach. The alternative — a shell here, a
vendor there, a different auth per app — is how four apps end up with four
architectures and none of them portable.

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js App Router | server-side compute and secrets on every app |
| Auth | self-hosted: scrypt + signed cookie | ~200 lines, no vendor, no database |
| Host | Railway | app, Postgres, and bucket are one service and one bill |
| Storage | S3 API (Railway Buckets, or R2) | commodity, swappable by endpoint |
| Database | a Postgres connection string | only when an app actually needs one |

The rule underneath: **depend on the commodity, never the differentiator.** RLS,
hosted auth, proprietary storage SDKs, and platform-level access gates all feel
like features and all make an app unportable.

## Install (as a Claude Code plugin)

```
/plugin marketplace add joshcoolman/bootstrap
/plugin install bootstrap@bootstrap
```

Then, from a brand-new empty folder:

```
mkdir my-app && cd my-app && claude
> /bootstrap:next-app              # the whole app: styled, login, empty dashboard
> /bootstrap:deploy-next-railway   # take it live
```

Skills also auto-trigger from plain requests ("scaffold a new app here", "put
this online"). For local development of this repo itself, load it directly:
`claude --plugin-dir ~/repos/bootstrap` (then `/reload-plugins` after edits).

## Skills

| Skill | What it does |
|-------|--------------|
| [next-app](skills/next-app/SKILL.md) | Scaffold a complete Next.js App Router app — Paper & Ink design system, a self-hosted login, deny-by-default route protection, an empty dashboard, `/docs` viewer, feature seams, CI. One pass, ready to build on. |
| [deploy-next-railway](skills/deploy-next-railway/SKILL.md) | Deploy an existing Next.js app to Railway — provision Postgres and a Storage Bucket if needed, wire env vars, set the Pre-Deploy Command for migrations, verify live via the deploy log and browser. |

`next-app` creates a repo from nothing and produces a *complete* app rather
than a shell you then layer onto — the login and the guard are part of the
scaffold, not a second step. `deploy-next-railway` targets Railway project
config rather than repo files.

## Auth: what you get, and what you don't

**Allowlist auth.** A short, known set of people can sign in. Access is granted
out-of-band by adding an entry to `AUTH_USERS`; there is no signup route, so a
stranger who finds the URL cannot self-serve access.

That covers "just me" and "me plus whoever I show it to," which is what these
apps are for. It costs about 200 lines of standard library — `node:crypto`
scrypt for passwords, Web Crypto HMAC for the session cookie — and no vendor,
no database, and no monthly bill.

Not included, deliberately: signup, password reset, email verification, OAuth,
roles, per-user data isolation. Sessions are stateless, so there is no
server-side revocation — signing out clears the cookie on that browser, and
rotating `AUTH_SESSION_SECRET` is the global sign-out. If an app grows a real
user base, that's the signal to adopt `better-auth`, not to extend this.

## Parts and recipes (the human-readable form)

Instruction-level docs — what to build and why, not line-by-line code — for
reading, or for applying a single slice to an existing repo:

| Part | What it covers |
|------|---------------|
| [setup-wizard](parts/setup-wizard.md) | `pnpm setup` — one command from fresh clone to a live, protected URL. Reads `.env.example` and fills each variable by class, so it works unchanged as an app grows keys |
| [image-store](parts/image-store.md) | Object storage for uploads and generated images: a four-method contract, Railway Buckets by default, R2 as the swap, and the `/img` route that keeps presigned URLs cacheable |
| [schema-reference](parts/schema-reference.md) | Generated, always-current DB map in `/docs`, with a CI staleness gate |
| [agent-surface](parts/agent-surface.md) | Giving a personal tool an agent surface: one core, two surfaces; blast-radius framing for personal MCP servers |

| Recipe | What it produces |
|--------|-----------------|
| [next-railway-app](recipes/next-railway-app.md) | Next.js on Railway: Postgres, bucket storage, allowlist auth, proven via a CRUD demo |
| [next-selfhost-app](recipes/next-selfhost-app.md) | Self-hostable Next.js app: SQLite, local disk storage, BYOK, Docker/volume deploy — zero required vendor accounts |

## How this repo grows

New skills are developed, not written: build the feature for real in a
throwaway "mule" repo, log every friction point, validate end-to-end, then
codify. The full authoring contract lives in [CLAUDE.md](CLAUDE.md).

## License

[MIT](LICENSE). Free — install it, fork it, build your own bootstrap on top of
it. This repo tracks what works for my own projects, so I'm not seeking
contributions and there's no support: if something doesn't work the way you
want, fork it and fix it.

## Status

**Last shipped:** a hard narrowing, and a fresh start. bootstrap went from five
skills to two.

- **Next.js only.** `vite-app`, `add-simple-auth`, and the Vite/TanStack
  recipes are gone. Every app these templates target needs server-side keys, so
  a no-server shell was the wrong default — and `npm create vite` already covers
  the case a skill would have saved little ceremony on.
- **Supabase is out.** `add-user-auth` is deleted rather than renamed. Auth is
  now self-hosted code the app owns, folded into `next-app` as a build step
  rather than layered on afterwards.
- **One skill produces the whole app** — styled home page, login, empty guarded
  dashboard — instead of a shell plus a separate auth pass.

Three fixes went in over the reference implementation it was lifted from: one
shared hash module instead of two copies that must be kept identical by hand
(drift there is a silent auth failure); an opaque hashed user id, because the
session cookie splits on `.` and an email as the id breaks the parse; and a
login action that preserves the typed email, because React 19's
`useActionState` resets uncontrolled fields on every call.

**The first consumer run happened (2026-07-18).** `/bootstrap:next-app` scaffolded
`prompt-smith` end to end: install, styles, auth, docs, seams, CI, then the real
login flow driven in a browser — redirect when signed out, wrong password
preserving the typed email, sign-in, reload, sign-out, re-gate. The auth code
works. Three fixes came back from it, all now in the skill:

- **`typescript@^5` is a required pin.** Unpinned, pnpm resolves TypeScript 7.x,
  which Next 16.2.10 does not recognise as a TypeScript install — `next build`
  tries to reinstall it and dies on `The "id" argument must be of type string`.
  Nothing in that output names the version.
- **`vitest.config.ts` must stub `server-only`.** The skill mandates both
  `import 'server-only'` in `credentials.ts` and a `credentials.test.ts`; without
  an alias the suite cannot load, and the error blames a Client Component.
- **`typography.css` was unrecoverable from the skill.** `styles.md` pointed at
  `skills/vite-app/resources/styles.md`, deleted in `2ba11ff`. The run had to
  recover it from git history. Now inlined.

The pattern in all three: a failure that surfaces as a tool crash or a misleading
error, never as the thing that is actually wrong.

**Up next:** the storage part and the setup wizard still have not executed —
`prompt-smith` used neither. Both remain unrun code. The next consumer run that
needs object storage or a clone-to-live path is their gate.
