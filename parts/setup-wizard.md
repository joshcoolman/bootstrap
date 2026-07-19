# Setup wizard

`pnpm setup` — one command from a fresh clone to a live, protected URL.

Point an agent at this file: *"read `parts/setup-wizard.md` and add the setup
wizard."* It ships `scripts/setup.mjs` into the app.

## Why this exists

An app you can't stand up in five minutes is an app you won't stand up. The
whole point of these templates is that an exploration starts in minutes and
gets torn down without regret — and the thing that actually delivers that is
one command that handles login, provisioning, env vars, and deploy.

This is the piece worth keeping from a retired repo: not the app, the pipeline.

**Relationship to `deploy-next-railway`:** that skill is agent-driven — you ask
Claude and it provisions. This is a script *in the app* that you run yourself.
They overlap deliberately. Use the skill when an agent is already in the loop;
use the wizard when you've cloned a repo on a new machine, or want the thing
live without opening Claude at all.

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

**1. Mask secret prompts.** The original used `input()` for API keys, which
echoes them to the terminal — and therefore into any screenshot or screen
share. This has already leaked one live key. Use `password()` from
`@inquirer/prompts` for anything secret.

**2. Never clobber `.env.local`.** The original overwrote it unconditionally,
with no merge and no backup — destroying working credentials on a re-run. Read
the existing file, merge, and only overwrite keys the user confirms.

**3. Deploy is optional.** The original hard-exited when the host CLI was
missing, with no way past it. `--local-only` should get you a configured local
app with no account, no deploy, and no network calls beyond what you ask for.

## `scripts/setup.mjs`

```js
#!/usr/bin/env node
// One command from a fresh clone to a live, protected URL.
// Reads .env.example and fills each variable by class, so adding a new key to
// .env.example is all it takes for this wizard to start asking for it.
//
// Usage: pnpm setup            full run, provisions and deploys
//        pnpm setup --local-only    configure .env.local only, no account needed
import { execSync } from 'node:child_process'
import { existsSync, readFileSync, writeFileSync, copyFileSync } from 'node:fs'
import { randomBytes } from 'node:crypto'
import { confirm, input, password, select } from '@inquirer/prompts'
import { hashPassword, normalizeEmail } from '../src/features/auth/hash.mjs'

const LOCAL_ONLY = process.argv.includes('--local-only')

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

// ── Auth: session secret + first user ───────────────────────
console.log('── Auth ─────────────────────────────\n')

for (const [name, generate] of Object.entries(GENERATED)) {
  if (!names.includes(name)) continue
  if (values[name] && !(await confirm({
    message: `${name} already set — regenerate? (invalidates all sessions)`, default: false,
  }))) continue
  values[name] = generate()
  console.log(`  ✓ ${name} generated`)
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
    const ask = SECRET_HINT.test(name) ? password : input
    const value = await ask({ message: `${name} (enter to skip):`, mask: '*' })
    if (value) values[name] = value
  }
}

// ── Write .env.local (merge, with a backup) ─────────────────
if (existsSync('.env.local')) copyFileSync('.env.local', '.env.local.bak')

const body = names.filter((n) => values[n]).map((n) => `${n}=${values[n]}`).join('\n')
writeFileSync('.env.local', body + '\n')
console.log('\n  ✓ .env.local written' + (existsSync('.env.local.bak') ? ' (previous saved as .env.local.bak)' : ''))

if (LOCAL_ONLY) {
  console.log('\n  Local setup complete. Run `pnpm dev`.\n')
  process.exit(0)
}

// ── Railway ─────────────────────────────────────────────────
console.log('\n── Railway ──────────────────────────\n')

if (!canRun('railway --version')) {
  console.log('  Railway CLI not found. Install it:\n')
  console.log('    brew install railway\n')
  console.log('  Then re-run `pnpm setup`, or use `pnpm setup --local-only`.\n')
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
console.log('\n  Pushing environment variables...')
for (const name of names.filter((n) => values[n] && !PROVISIONED.includes(n))) {
  try {
    execSync(`railway variables --set ${JSON.stringify(`${name}=${values[name]}`)}`,
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

Add `@inquirer/prompts` to `devDependencies`, and `.env.local.bak` to
`.gitignore`.

**Note on the script name:** `setup` is safe in `package.json`, but `pnpm setup`
is also a pnpm built-in (it configures the pnpm home directory). `pnpm run
setup` always resolves to the script. If that ambiguity bothers you, name it
`init` instead — the original repo hit this and worked around it by never
calling the script `setup` at all.

## Verification

Run it three times. Each pass proves a different thing:

1. **Fresh clone, `--local-only`** — no Railway account touched. `.env.local`
   gets a generated secret and one user; `pnpm dev` starts; you can sign in.
2. **Re-run over the existing `.env.local`** — nothing is destroyed. Existing
   values are offered, not clobbered. Choose "add another person" and confirm
   both accounts can sign in. Confirm `.env.local.bak` exists.
3. **Full run** — links a project, pushes variables, deploys, prints a URL.
   Open it, sign in, confirm you land on the dashboard and that a signed-out
   browser is redirected to `/login`.

Test 2 is the one that matters most. Clobbering a working `.env.local` is the
defect this port exists to fix, and it only shows up on a second run — which is
exactly the run nobody tests.
