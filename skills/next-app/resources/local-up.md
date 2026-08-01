# Part: local:up

One command from a fresh clone to a working login: `pnpm local:up`.

## Why this exists

A scaffolded app boots fine with no configuration and then fails at the **first
request** with `AUTH_SESSION_SECRET is not set`. An unconfigured app is
therefore indistinguishable from a broken one — the failure arrives at request
time, names a variable nothing told you about, and offers no next step.

The gap it closes is concrete. `pnpm auth:add-user` prints an
`email:salt:hash` line, and from there the operator has to already know that it
belongs in `.env.local` under `AUTH_USERS`, and that a second variable has to
exist alongside it. Neither the script's output nor `pnpm dev` says so. That is
two pieces of undocumented tribal knowledge standing between a clone and a
login.

**`.env.local` is the source of truth, and re-running is the reset story.** The
dev password lives in that file; every run re-syncs the allowlist entry from it.
Editing `LOCAL_DEV_PASSWORD` and re-running *is* the password reset — no
separate command, and no way for the file to disagree with the app.

Ported from `bootsy`'s `scripts/local-up.mjs`, minus the Docker, Postgres, and
MinIO phases — this scaffold's entire state is a signed cookie and an
allowlist, so there is nothing to containerise. If a later layer adds Postgres,
that is where the container phases come back.

## `scripts/local-up.mjs`

Zero new dependencies — Node builtins plus the `hash.mjs` the auth feature
already ships.

```js
// One command to a working local app: .env.local, a session secret, and a
// login. No database and no containers — this app's whole state is a signed
// cookie and an allowlist. Idempotent; run it as often as you like.
//
// .env.local is the source of truth: the dev entry in AUTH_USERS is re-synced
// whenever it stops matching LOCAL_DEV_PASSWORD, so editing that value and
// re-running IS the password reset.
//
// Usage: pnpm local:up
import { copyFileSync, existsSync, readFileSync, writeFileSync } from 'node:fs'
import { randomBytes } from 'node:crypto'
import { hashPassword, matchesHash, normalizeEmail } from '../src/features/auth/hash.mjs'

const ROOT = new URL('../', import.meta.url)
const ENV_LOCAL = new URL('.env.local', ROOT)
const ENV_LOCAL_EXAMPLE = new URL('.env.local.example', ROOT)
const ENV_EXAMPLE = new URL('.env.example', ROOT)
// Named to match the `*.local` rule already in .gitignore. `.env.local.bak`,
// the obvious name, is NOT covered by that glob — a backup full of credentials
// would sit there committable.
const ENV_BACKUP = new URL('.env.previous.local', ROOT)

/** Read one value out of a .env file's text. */
function read(key, text) {
  return (text.match(new RegExp(`^${key}=(.*)$`, 'm')) ?? [])[1]
}

/** Replace in place when present, append when not — comments and order survive. */
function upsert(text, key, value) {
  const line = `${key}=${value}`
  const pattern = new RegExp(`^${key}=.*$`, 'm')
  return pattern.test(text) ? text.replace(pattern, line) : `${text.replace(/\s*$/, '')}\n${line}\n`
}

/** AUTH_USERS as trimmed, non-empty `email:salt:hash` entries. */
function entries(authUsers) {
  return (authUsers ?? '')
    .split(',')
    .map((e) => e.trim())
    .filter(Boolean)
}

const emailOf = (entry) => normalizeEmail(entry.split(':')[0] ?? '')

/** This email's stored `salt:hash`, if it has an entry. */
function storedHashFor(authUsers, email) {
  const entry = entries(authUsers).find((e) => emailOf(e) === email)
  return entry ? entry.split(':').slice(1).join(':') : undefined
}

/** Replace this email's allowlist entry, keep everyone else's. */
function upsertUser(authUsers, email, entry) {
  const others = entries(authUsers).filter((e) => emailOf(e) !== email)
  return [...others, entry].join(',')
}

/** Blank, or the all-zeros placeholder some templates ship. */
function isPlaceholder(value) {
  return !value || /^0+$/.test(value)
}

// The port lives in package.json's dev script. Parsed rather than duplicated
// here, so changing the port is still a one-line edit in one file.
function devPort() {
  const pkg = JSON.parse(readFileSync(new URL('package.json', ROOT), 'utf8'))
  return (pkg.scripts?.dev?.match(/-p\s+(\d+)/) ?? [])[1] ?? '3000'
}

// 1. .env.local
const existed = existsSync(ENV_LOCAL)
if (!existed) {
  // Without this the copy throws a raw ENOENT stack trace, which reads as a
  // broken script rather than a missing file someone forgot to commit.
  if (!existsSync(ENV_LOCAL_EXAMPLE)) {
    console.error('.env.local.example is missing — there is nothing to copy from.')
    console.error('It ships with the scaffold; restore it from git, or create')
    console.error('.env.local by hand from .env.example.')
    process.exit(1)
  }
  copyFileSync(ENV_LOCAL_EXAMPLE, ENV_LOCAL)
  console.log('created .env.local from .env.local.example')
}

// 2. Load it, then read the dev credentials. Read *after* the load so the file
// is the source of truth — and a shell export still wins, because loadEnvFile
// does not overwrite a value that is already set.
process.loadEnvFile(ENV_LOCAL)
const DEV_EMAIL = normalizeEmail(process.env.LOCAL_DEV_EMAIL ?? 'dev@<app-name>.local')
const DEV_PASSWORD = process.env.LOCAL_DEV_PASSWORD ?? '<app-name>local'

let text = readFileSync(ENV_LOCAL, 'utf8')
const before = text

// 3. Session secret. Generated only when missing — never regenerated on a
// re-run. Rotating it signs every session out everywhere, which is a
// deliberate act, not something a provisioner should do behind your back.
if (isPlaceholder(read('AUTH_SESSION_SECRET', text))) {
  text = upsert(text, 'AUTH_SESSION_SECRET', randomBytes(32).toString('hex'))
  console.log('generated AUTH_SESSION_SECRET')
}

// 4. The dev login. Provisioning IS the allowlist (there is no signup route),
// so a fresh .env.local is unusable without this.
//
// Re-hashed ONLY when the stored entry does not already verify the current
// password. scrypt salts randomly, so hashing unconditionally yields a
// different entry every run — the file would be rewritten forever and this
// script could never be idempotent. Checking first keeps the reset story
// intact: change LOCAL_DEV_PASSWORD and the check fails, so the entry is
// rewritten. Anyone added by `pnpm auth:add-user` meanwhile is preserved.
const stored = storedHashFor(read('AUTH_USERS', text), DEV_EMAIL)
if (!stored || !(await matchesHash(DEV_PASSWORD, stored))) {
  const entry = `${DEV_EMAIL}:${await hashPassword(DEV_PASSWORD)}`
  const users = upsertUser(read('AUTH_USERS', text), DEV_EMAIL, entry)
  text = upsert(text, 'AUTH_USERS', users)
  const others = users.split(',').length - 1
  console.log(`synced ${DEV_EMAIL}${others ? ` (${others} other user(s) preserved)` : ''}`)
}

// 5. Write, backing up only when something actually changed — an unchanged
// re-run must leave no trace at all, or the backup stops signalling anything.
if (text !== before) {
  if (existed) copyFileSync(ENV_LOCAL, ENV_BACKUP)
  writeFileSync(ENV_LOCAL, text)
  console.log(`wrote .env.local${existed ? ' (previous saved as .env.previous.local)' : ''}`)
} else {
  console.log('.env.local already current — nothing to change')
}

// 6. Anything else the app declares but this file cannot fill. Warn, never
// fail: a missing API key means one feature is dead, not that the app won't
// run, and failing here would block a login that is otherwise ready.
if (existsSync(ENV_EXAMPLE)) {
  const declared = readFileSync(ENV_EXAMPLE, 'utf8')
    .split('\n')
    .map((line) => line.trim())
    .filter((line) => line && !line.startsWith('#'))
    .map((line) => line.split('=')[0].trim())
  const missing = declared.filter((name) => !read(name, text))
  if (missing.length) {
    console.warn(`\nnot set in .env.local: ${missing.join(', ')}`)
    console.warn('the app runs, but anything depending on those will fail.')
  }
}

console.log(`
Ready.
  pnpm dev                 http://localhost:${devPort()}
  sign in as               ${DEV_EMAIL} / ${DEV_PASSWORD}

Change the password in .env.local and re-run this to reset it.
`)
```

Add to `package.json`:

```json
"local:up": "node scripts/local-up.mjs"
```

## Why each weird bit exists

- **The backup is `.env.previous.local`, not `.env.local.bak`.** `.gitignore`
  carries `*.local`, which covers the first name and **not** the second. A
  `.bak` file full of password hashes would sit in `git status` waiting to be
  committed. Renaming the backup is cheaper than adding a `.gitignore` line that
  a future edit could drop.
- **The session secret is generated once and never regenerated.** A rotation
  invalidates every session everywhere — it is this design's only global
  sign-out, and it belongs to the operator. A provisioner that quietly rotated
  it on every run would sign you out each time you re-ran it, which reads as a
  session bug rather than as the command's own doing.
- **The dev user is verified first, then re-hashed only on a mismatch.** Two
  wrong versions sit either side of this. "Create if missing" lets `.env.local`
  and the app disagree the moment someone edits `LOCAL_DEV_PASSWORD`, with no
  error to say so. "Re-hash every run" — which this shipped first — is worse in
  a way that reads as correct: **scrypt salts randomly**, so an unchanged
  password produces a *different* `salt:hash` on every call, the file is
  rewritten forever, `.env.previous.local` is recreated every run, and the
  "already current" path is unreachable. Caught on the second run of a live
  test; reading the script would never show it. `matchesHash` against the
  stored entry is what makes the command idempotent while keeping the file
  authoritative.
- **`upsertUser` filters by email rather than replacing `AUTH_USERS`
  wholesale.** Clobbering a working `.env.local` is a defect this repo's setup
  wizard already had to fix once (`parts/setup-wizard.md`, fix 2). Anyone added
  with `pnpm auth:add-user` between runs survives. This only shows up on a
  second run, which is exactly the run nobody tests.
- **The write is conditional, and a first run makes no backup.** Backing up
  unconditionally means an unchanged re-run still produces a
  `.env.previous.local`, so the backup stops signalling anything — and on a
  first run it would back up a file the script itself just created from the
  template. No change, no trace.
- **Missing vars warn and never fail.** The check reads `.env.example`, so a
  later layer that adds an API key gets the warning for free with no edit here.
  Failing instead would block a login that is otherwise ready to use.
- **The port is parsed from `package.json`.** Hardcoding it here means two
  places to edit and a Ready block that can silently start lying.
- **`process.loadEnvFile` needs Node ≥22**, already required by
  `engines.node`. It is why no `dotenv` dependency appears.
- **Values that get rewritten are read from the file text, not `process.env`.**
  `loadEnvFile` does not overwrite an already-set variable, so a stale shell
  export would otherwise be written back into the file as though it came from
  there.

## Adaptation slots

| Slot | Filled from |
| --- | --- |
| `<app-name>` | the app name — the default dev email and password |

The dev port is not a slot; it is read from `package.json`.

## What "done" looks like

Run it three times. Each pass proves a different thing, and the second and
third are the ones that matter:

1. **Fresh** — `.env.local` created, secret generated, dev user synced, and
   **no** `.env.previous.local`. `pnpm dev`, then sign in with the printed
   credentials and land on `/dashboard`.
2. **Re-run, nothing changed** — prints `.env.local already current`, writes
   nothing, creates no backup. Diff the file against the previous pass: it must
   be **byte-identical**. This is the pass that catches the salt bug in the
   ledger above, and it is invisible on run 1.
3. **Re-run after a mutation** — add a second person with `pnpm auth:add-user`,
   paste their entry in, edit `LOCAL_DEV_PASSWORD`, re-run. The second person
   survives, the dev entry is rewritten, `AUTH_SESSION_SECRET` is unchanged,
   `.env.previous.local` exists, and `AUTH_USERS` holds exactly two entries —
   not three, and not one.
4. **Re-run once more** — back to "already current". A command that is
   idempotent before a mutation but not after it is still broken.

Confirm the passwords actually verify rather than trusting the file's shape:
the new dev password matches, the **old one no longer does**, and the second
person's still does.
