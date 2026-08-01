# Setup wizard

`pnpm setup` — one command from a configured local app to a live, protected URL.

Point an agent at this file: *"read `parts/setup-wizard.md` and add the setup
wizard."* It ships `scripts/setup.mjs` into the app.

## Why this exists

An app you can't stand up in five minutes is an app you won't stand up. The
whole point of these templates is that an exploration starts in minutes and
gets torn down without regret — and the thing that actually delivers that is
one command that handles provisioning, env vars, and deploy.

This is the piece worth keeping from a retired repo: not the app, the pipeline.

## Three ways to stand an app up — one job each

This wizard used to own local configuration too, via a `--local-only` flag.
**It no longer does, and must not grow it back.** `next-app` scaffolds
`pnpm local:up`, which does that job non-interactively and idempotently. Two
commands writing `.env.local` is exactly the conflicting signal `CLAUDE.md`
names as the failure to hunt for — an agent picking between them flips a coin,
and different sessions flip it differently.

| Command | Where it lives | Job |
| --- | --- | --- |
| `pnpm local:up` | in the app (`next-app` ships it) | local `.env.local`, session secret, dev login. No prompts, safe to re-run. |
| `pnpm setup` | in the app (this part) | Railway: link, provision, push vars, deploy. Interactive — it asks for API keys. |
| `deploy-next-railway` | the skill | the same deploy, agent-driven, when Claude is already in the loop |

So: `pnpm local:up` to work on it, `pnpm setup` to put it on the internet
yourself, the skill to have an agent do the latter. The wizard assumes
`local:up` has already run and reads the `.env.local` it produced.

## The design that makes it reusable

Do **not** hardcode the app's environment variables. The wizard reads
`.env.example` and classifies each name:

| Class | How it's filled | Examples |
|---|---|---|
| **Generated** | random, no prompt | `AUTH_SESSION_SECRET` |
| **Interactive** | a purpose-built prompt | `AUTH_USERS` (email + password) |
| **Provisioned** | from Railway, if that service is added | `DATABASE_URL`, `BUCKET_*` |
| **Prompted** | asked for, masked | `ANTHROPIC_API_KEY`, `FAL_KEY`, … |

Anything unrecognised falls through to *prompted*. So the same wizard works
unchanged in an app with three env vars and an app with twelve — adding a new
API key to `.env.example` is all it takes for the wizard to start asking for it.

## The three fixes

This is ported from a working wizard, with three defects corrected. Do not
reintroduce them.

**1. Mask secret prompts — with asterisks, not silence.** The original used
`input()` for API keys, which echoes them to the terminal and therefore into
any screenshot or screen share. This has already leaked one live key.

Use `password()` — but **always pass `mask: '*'`**. Without it, `password`
renders a blank line as you type, so you can't tell whether a keystroke
registered. That silent-prompt experience is bad enough that it's *why* the
original used plain `input()` in the first place. Asterisks give you the
feedback and none of the exposure; a fix that's annoying to use gets reverted.

**2. Never clobber `.env.local`.** The original overwrote it unconditionally,
with no merge and no backup — destroying working credentials on a re-run. Read
the existing file, merge, and only overwrite keys the user confirms.

**3. A missing host CLI must not be a dead end.** The original hard-exited when
the Railway CLI was absent, with no way past it. It no longer needs a
`--local-only` escape hatch — `pnpm local:up` already gets you a working local
app with no account and no network calls — but it must still say plainly what
to install and that local development is unaffected, rather than exiting on an
error that reads like the app is broken.

**A fourth rule, from narrowing this to deploy only: never push the local
session secret.** The deployed app gets its own, generated here. The secret is
the only thing binding a session cookie to an environment, so a shared value
means a production cookie validates against localhost and signs you in as a
user the local allowlist has no entry for — every scoped query returns empty,
which is indistinguishable from a fresh install.

## `scripts/setup.mjs`

```js
#!/usr/bin/env node
// From a configured local app to a live, protected URL.
// Reads .env.example and fills each variable by class, so adding a new key to
// .env.example is all it takes for this wizard to start asking for it.
//
// Local configuration is NOT this script's job — `pnpm local:up` owns that,
// and this reads the .env.local it produced. The one value deliberately not
// carried over is AUTH_SESSION_SECRET: the deployed app gets its own.
//
// Usage: pnpm setup
import { execSync } from 'node:child_process'
import { existsSync, readFileSync, writeFileSync, copyFileSync } from 'node:fs'
import { randomBytes } from 'node:crypto'
import { confirm, input, password, select } from '@inquirer/prompts'
import { hashPassword, normalizeEmail } from '../src/features/auth/hash.mjs'

const run = (cmd) => execSync(cmd, { stdio: 'inherit' })
const capture = (cmd) => execSync(cmd, { stdio: ['pipe', 'pipe', 'pipe'] }).toString().trim()
const canRun = (cmd) => { try { execSync(cmd, { stdio: 'ignore' }); return true } catch { return false } }

// ── Classify ────────────────────────────────────────────────
const GENERATED = { AUTH_SESSION_SECRET: () => randomBytes(32).toString('hex') }
const PROVISIONED = ['DATABASE_URL', 'BUCKET_NAME', 'BUCKET_ACCESS_KEY_ID',
  'BUCKET_SECRET_ACCESS_KEY', 'BUCKET_REGION', 'BUCKET_ENDPOINT']
const SECRET_HINT = /KEY|SECRET|TOKEN|PASSWORD/

function readEnvExample() {
  if (!existsSync('.env.example')) {
    console.error('No .env.example found — nothing to configure.')
    process.exit(1)
  }
  return readFileSync('.env.example', 'utf8')
    .split('\n')
    .map((l) => l.trim())
    .filter((l) => l && !l.startsWith('#'))
    .map((l) => l.split('=')[0].trim())
}

function readExistingEnv() {
  if (!existsSync('.env.local')) return {}
  const existing = {}
  for (const line of readFileSync('.env.local', 'utf8').split('\n')) {
    const trimmed = line.trim()
    if (!trimmed || trimmed.startsWith('#')) continue
    const idx = trimmed.indexOf('=')
    if (idx > 0) existing[trimmed.slice(0, idx)] = trimmed.slice(idx + 1)
  }
  return existing
}

console.log('\n┌────────────────────────────────┐')
console.log('│         setup wizard           │')
console.log('└────────────────────────────────┘\n')

const names = readEnvExample()
const existing = readExistingEnv()
const values = { ...existing }

if (Object.keys(existing).length) {
  console.log(`  Found an existing .env.local with ${Object.keys(existing).length} value(s).`)
  console.log('  Existing values are kept unless you choose to replace them.\n')
}

// ── Auth ────────────────────────────────────────────────────
console.log('── Auth ─────────────────────────────\n')

// Values that go to the host but NOT into .env.local. The session secret lives
// here and nowhere else: sharing it with local means a production cookie
// validates against localhost, signing you in as a user the local allowlist
// has no entry for — an empty app that reads as a fresh install.
const deployOnly = {}

for (const [name, generate] of Object.entries(GENERATED)) {
  if (!names.includes(name)) continue
  deployOnly[name] = generate()
  console.log(`  ✓ ${name} generated for the deployed app (local keeps its own)`)
}

if (names.includes('AUTH_USERS')) {
  const hasUsers = Boolean(values.AUTH_USERS)
  const action = hasUsers
    ? await select({
        message: 'AUTH_USERS already set:',
        choices: [
          { name: 'Keep as is', value: 'keep' },
          { name: 'Add another person', value: 'add' },
          { name: 'Replace everyone', value: 'replace' },
        ],
      })
    : 'replace'

  if (action !== 'keep') {
    const email = await input({
      message: 'Email:', validate: (v) => v.includes('@') || 'Enter a valid email',
    })
    // Masked — never echo a credential to a terminal that might be shared.
    const pass = await password({
      message: 'Password (min 8 chars):', mask: '*',
      validate: (v) => v.length >= 8 || 'At least 8 characters',
    })
    const entry = `${normalizeEmail(email)}:${await hashPassword(pass)}`
    values.AUTH_USERS = action === 'add' ? `${values.AUTH_USERS},${entry}` : entry
    console.log(`  ✓ ${email} can sign in`)
  }
}

// ── Remaining keys ──────────────────────────────────────────
const remaining = names.filter(
  (n) => !(n in GENERATED) && n !== 'AUTH_USERS' && !PROVISIONED.includes(n),
)

if (remaining.length) {
  console.log('\n── API keys ─────────────────────────\n')
  for (const name of remaining) {
    if (values[name]) { console.log(`  ✓ ${name} (already set)`); continue }
    const msg = `${name} (enter to skip):`
    // mask: '*' shows one asterisk per character. Without it, `password`
    // renders a blank line and you can't tell a keystroke registered —
    // which is why the original reached for `input` and echoed keys in clear.
    const value = SECRET_HINT.test(name)
      ? await password({ message: msg, mask: '*' })
      : await input({ message: msg })
    if (value) values[name] = value
  }
}

// ── Write .env.local back (merge, with a backup) ────────────
// Only the keys you were just prompted for — an API key you typed is worth
// keeping locally too. `deployOnly` values never land here.
// Backup name matches the `*.local` gitignore rule; `.env.local.bak` would not.
if (existsSync('.env.local')) copyFileSync('.env.local', '.env.previous.local')

const body = names.filter((n) => values[n]).map((n) => `${n}=${values[n]}`).join('\n')
writeFileSync('.env.local', body + '\n')
console.log('\n  ✓ .env.local updated (previous saved as .env.previous.local)')

// ── Railway ─────────────────────────────────────────────────
console.log('\n── Railway ──────────────────────────\n')

if (!canRun('railway --version')) {
  console.log('  Railway CLI not found. Install it:\n')
  console.log('    brew install railway\n')
  console.log('  Then re-run `pnpm setup`.')
  console.log('  Local development is unaffected — `pnpm local:up && pnpm dev` still works.\n')
  process.exit(1)
}
if (!canRun('railway whoami')) {
  console.log('  Log in to Railway (opens a browser).\n')
  run('railway login')
}
console.log(`  ✓ ${capture('railway whoami')}\n`)

let linked = true
try { capture('railway status') } catch { linked = false }

if (!linked) {
  const create = await confirm({ message: 'No project linked. Create one?', default: true })
  run(create ? 'railway init' : 'railway link')
}
console.log('  ✓ Project linked\n')

if (names.includes('DATABASE_URL') && !values.DATABASE_URL) {
  if (await confirm({ message: 'This app wants Postgres. Add it?', default: true })) {
    run('railway add --database postgres')
    console.log('  ✓ Postgres added — set DATABASE_URL to ${{Postgres.DATABASE_URL}} in the dashboard')
  }
}

// Push everything except provisioned vars, which Railway wires itself.
// deployOnly wins over values — that is how the deployed app gets a session
// secret distinct from the local one.
console.log('\n  Pushing environment variables...')
const pushed = { ...values, ...deployOnly }
for (const name of names.filter((n) => pushed[n] && !PROVISIONED.includes(n))) {
  try {
    execSync(`railway variables --set ${JSON.stringify(`${name}=${pushed[name]}`)}`,
      { stdio: ['pipe', 'ignore', 'pipe'] })
    console.log(`  ✓ ${name}`)
  } catch (err) {
    console.log(`  ✗ ${name}: ${err.stderr?.toString().trim() || 'failed'}`)
  }
}

if (await confirm({ message: '\nDeploy now?', default: true })) {
  run('railway up')
  console.log('\n  Generating a public URL...')
  try { run('railway domain') } catch { console.log('  (a domain may already exist — check `railway domain`)') }
}

console.log('\n  Setup complete. Sign in with the account you created above.\n')
```

## Wiring

```json
"setup": "node scripts/setup.mjs"
```

Add `@inquirer/prompts` to `devDependencies`. No `.gitignore` change is needed:
the backup is `.env.previous.local`, which the existing `*.local` rule already
covers. `.env.local.bak` — the obvious name, and the one an earlier draft of
this used — is **not** covered by that glob, and would leave a file of
credentials sitting committable.

**Note on the script name:** `setup` is safe in `package.json`, but `pnpm setup`
is also a pnpm built-in (it configures the pnpm home directory). `pnpm run
setup` always resolves to the script. If that ambiguity bothers you, name it
`init` instead — the original repo hit this and worked around it by never
calling the script `setup` at all.

## Verification

Starts from an app already configured by `pnpm local:up`. Three passes, each
proving a different thing:

1. **No Railway CLI installed** — prints the install line, says local dev is
   unaffected, exits. It must not read as the app being broken.
2. **Re-run over the existing `.env.local`** — nothing is destroyed. Existing
   values are offered, not clobbered. Choose "add another person" and confirm
   both accounts can sign in locally. Confirm `.env.previous.local` exists.
3. **Full run** — links a project, pushes variables, deploys, prints a URL.
   Open it, sign in, confirm you land on the dashboard and that a signed-out
   browser is redirected to `/login`.

Test 2 is the one that matters most. Clobbering a working `.env.local` is the
defect this port exists to fix, and it only shows up on a second run — which is
exactly the run nobody tests.

After test 3, check the one thing narrowing this introduced: `railway variables`
must show an `AUTH_SESSION_SECRET` **different** from the one in `.env.local`.
Equal values mean the deploy-only path leaked, and the symptom — a production
cookie signing you in locally as a user with no allowlist entry — looks like an
empty app, not like a secret problem.
