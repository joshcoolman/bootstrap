# Setup wizard

The interactive script that wires a real Supabase project: CLI check → login →
link-or-create project → derive keys → write `.env.local` → provision the
single user via the admin API. **You write it; the user runs it.** Never run
it yourself — it opens a browser for login and asks for the user's account
credentials.

The script below is **verbatim and battle-tested** — it survived live runs
that killed two naive versions. The "why each weird bit exists" ledger follows
the code; do not simplify any of those bits away, however tempting.

Adapt points (only these): the banner title, the default project name, and
the done-message URL (`<dev-port>` from discovery).

## `package.json` — the script name

```json
"setup:supabase": "node scripts/setup.mjs"
```

**Never name it `setup`.** pnpm's built-in `setup` command shadows package
scripts — `pnpm setup` installs the pnpm standalone binary and edits the
user's shell rc. Reference the command as `pnpm setup:supabase` everywhere
(docs, error messages, the adapter's NOT_CONFIGURED string).

## ESLint — Node globals for `scripts/`

Shell configs typically define browser globals only, so `scripts/*.mjs` fails
lint with dozens of `no-undef` errors. Add this block to the flat config
(merge into whatever shape discovery found, before the prettier entry):

```js
{
  files: ['scripts/**/*.mjs'],
  languageOptions: {
    globals: {
      process: 'readonly',
      console: 'readonly',
      Buffer: 'readonly',
      fetch: 'readonly',
      setTimeout: 'readonly',
    },
  },
  rules: {
    'no-empty': ['error', { allowEmptyCatch: true }],
  },
},
```

## `.gitignore`

Add `supabase/.temp` — the user's `supabase link` writes machine-local state
(project-ref, pooler URLs) there, which must not land in a commit.

## `scripts/setup.mjs`

```js
#!/usr/bin/env node
// Interactive auth setup: links (or creates) a hosted Supabase project, writes
// .env.local, and provisions the single login. Re-running is safe.
process.env.NODE_NO_WARNINGS = '1'
import { execSync } from 'child_process'
import { readFileSync, writeFileSync } from 'fs'
import { input, confirm } from '@inquirer/prompts'

const run = (cmd) => execSync(cmd, { stdio: 'inherit' })
const capture = (cmd) =>
  execSync(cmd, { stdio: ['pipe', 'pipe', 'pipe'] })
    .toString()
    .trim()
const canRun = (cmd) => {
  try {
    execSync(cmd, { stdio: 'ignore' })
    return true
  } catch {
    return false
  }
}
// The Supabase CLI prints `null` (not `[]`) for empty lists in JSON mode.
const listJson = (cmd) => JSON.parse(capture(cmd)) ?? []

// `--reveal` only exists on some CLI versions (others print full keys without
// it). Returns the key list, or null when no project is linked.
function fetchApiKeys() {
  for (const cmd of [
    'supabase projects api-keys --reveal --output json',
    'supabase projects api-keys --output json',
  ]) {
    try {
      return JSON.parse(capture(cmd)) ?? []
    } catch {
      /* unknown flag or not linked — try next */
    }
  }
  return null
}

console.log('\n┌──────────────────────────────────────┐')
console.log('│      web-starter auth setup wizard    │')
console.log('└──────────────────────────────────────┘\n')
console.log('This links a Supabase project for email/password login,')
console.log('writes .env.local, creates your account (the only login), and')
console.log('locks down public sign-ups so no one else can create one.\n')
console.log('Tip: to skip macOS Keychain prompts entirely, Ctrl+C now, create a')
console.log('token at https://supabase.com/dashboard/account/tokens, then run:')
console.log('  export SUPABASE_ACCESS_TOKEN=sbp_...')
console.log('and re-run this wizard.\n')

await confirm({ message: 'Ready to continue?', default: true })

// ── Check CLI ───────────────────────────────────────────────
console.log('\nChecking CLI...')
if (!canRun('supabase --version')) {
  console.log('\nMissing the Supabase CLI — install it first:\n')
  console.log('  brew install supabase/tap/supabase')
  console.log('\nThen re-run: pnpm setup:supabase')
  process.exit(1)
}
console.log('  ✓ Supabase CLI')

// ── Step 1: Log in ──────────────────────────────────────────
console.log('\n── 1/3  Supabase login ─────────────────────────\n')

if (!canRun('supabase projects list --output json')) {
  console.log('Log in to Supabase (opens browser).')
  console.log('  Note: macOS may ask if your terminal can access the Keychain —')
  console.log('  and will keep asking on nearly every command below, not just once.')
  console.log('  "Deny" is safe but repeats through the rest of this wizard;')
  console.log('  granting access is also safe (it\'s the official CLI) and stops')
  console.log('  the repeats. The SUPABASE_ACCESS_TOKEN tip above avoids this')
  console.log('  dialog entirely.\n')
  run('supabase login')
}
console.log('  ✓ Logged in\n')

// ── Step 2: Link or create a project ────────────────────────
console.log('── 2/3  Project ────────────────────────────────\n')

const isLinked = fetchApiKeys() !== null

if (!isLinked) {
  let projects = listJson('supabase projects list --output json')

  if (projects.length === 0) {
    console.log('  No Supabase projects found.\n')
    const createNew = await confirm({
      message: 'Create a new project now?',
      default: true,
    })

    if (createNew) {
      const orgs = listJson('supabase orgs list --output json')
      let orgId
      if (orgs.length === 0) {
        console.log('  No Supabase organizations found on this account.')
        console.log('  Create one at https://supabase.com/dashboard, then re-run.')
        process.exit(1)
      }
      if (orgs.length === 1) {
        orgId = orgs[0].id
        console.log(`  Using org: ${orgs[0].name}\n`)
      } else {
        orgs.forEach((o, i) => console.log(`  ${i + 1}. ${o.name}`))
        const pick = await input({
          message: 'Choose org number:',
          validate: (v) => (Number(v) >= 1 && Number(v) <= orgs.length) || 'Enter a valid number',
        })
        orgId = orgs[Number(pick) - 1].id
      }

      const projectName = await input({
        message: 'Project name:',
        default: 'web-starter',
      })

      const REGIONS = [
        { code: 'us-east-1', label: 'N. Virginia, USA' },
        { code: 'us-west-1', label: 'N. California, USA' },
        { code: 'us-west-2', label: 'Oregon, USA' },
        { code: 'eu-west-1', label: 'Ireland' },
        { code: 'eu-central-1', label: 'Frankfurt, Germany' },
        { code: 'ap-southeast-1', label: 'Singapore' },
        { code: 'ap-northeast-1', label: 'Tokyo, Japan' },
      ]
      console.log('\n  Regions:')
      REGIONS.forEach((r, i) => console.log(`  ${i + 1}. ${r.code.padEnd(18)} ${r.label}`))
      const regionInput = await input({
        message: 'Type the number next to your region:',
        default: '1',
      })
      const region = REGIONS[Number(regionInput) - 1]?.code ?? regionInput

      // Auth-only setup never needs this password again, but project creation
      // requires one and Supabase won't show it later — only reset it.
      console.log('\n  You need a database password for this project.')
      console.log('  Supabase does not let you retrieve it later — only reset it.')
      console.log('  Save it somewhere safe before continuing.\n')

      const dbPassword = await input({
        message: 'Database password (shown as typed):',
      })

      const border = '─'.repeat(dbPassword.length + 4)
      console.log(`\n  ┌${border}┐`)
      console.log(`  │  ${dbPassword}  │`)
      console.log(`  └${border}┘`)
      console.log('  ^ Save this password now — you will not see it again.\n')

      await confirm({ message: "I've saved my password, continue", default: true })

      console.log('\n  Creating project...')
      run(
        `supabase projects create "${projectName}" --org-id ${orgId} --db-password "${dbPassword}" --region ${region}`,
      )

      process.stdout.write('\n  Waiting for project to be ready')
      while (true) {
        await new Promise((r) => setTimeout(r, 3000))
        projects = listJson('supabase projects list --output json')
        if (projects.length > 0) break
        process.stdout.write('.')
      }
      console.log(' ✓')
    } else {
      console.log('  Create a free project at: https://supabase.com/dashboard/new/_\n')
      while (true) {
        projects = listJson('supabase projects list --output json')
        if (projects.length > 0) break
        await confirm({
          message: 'Press enter once your project is created',
          default: true,
        })
      }
    }
  }

  if (projects.length === 1) {
    run(`supabase link --project-ref ${projects[0].ref}`)
  } else {
    console.log('Select a project to link:\n')
    run('supabase link')
  }
}
console.log('  ✓ Project linked\n')

// ── Step 3: Keys → .env.local ───────────────────────────────
console.log('── 3/3  Keys ───────────────────────────────────\n')

const rawKeys = fetchApiKeys()
if (!rawKeys || rawKeys.length === 0) {
  console.error('  ✗ Could not read API keys from the linked project.')
  console.log('  Find them at: Supabase dashboard → Project settings → API keys')
  process.exit(1)
}
// Legacy projects return JWT keys labeled anon/service_role (under `id` or
// `name` depending on CLI version); newer projects return sb_publishable_* /
// sb_secret_* keys instead.
const findKey = (label, prefix) =>
  rawKeys.find((k) => k.id === label || k.name === label)?.api_key ??
  rawKeys.find((k) => k.api_key?.startsWith(prefix))?.api_key
const anon = findKey('anon', 'sb_publishable_')
const serviceRole = findKey('service_role', 'sb_secret_')

if (!anon || !serviceRole) {
  console.error('  ✗ Could not identify the anon + service_role keys.')
  console.log(`  Keys returned: ${rawKeys.map((k) => k.id ?? k.name).join(', ')}`)
  console.log('  Find them at: Supabase dashboard → Project settings → API keys')
  process.exit(1)
}
if (/\*|REDACTED/i.test(anon + serviceRole)) {
  console.error('  ✗ Your Supabase CLI redacts keys and is too old for --reveal.')
  console.log('  Upgrade it: brew upgrade supabase, then re-run pnpm setup:supabase')
  process.exit(1)
}

function deriveUrl() {
  // Legacy anon keys are JWTs whose payload carries the project ref.
  try {
    const payload = JSON.parse(Buffer.from(anon.split('.')[1], 'base64').toString())
    if (payload.ref) return { url: `https://${payload.ref}.supabase.co`, via: 'anon JWT' }
  } catch {
    /* not a JWT */
  }
  // `supabase link` writes the ref here.
  try {
    const ref = readFileSync('supabase/.temp/project-ref', 'utf8').trim()
    if (ref) return { url: `https://${ref}.supabase.co`, via: 'linked project ref' }
  } catch {
    /* no linked ref file */
  }
  return null
}

let derived = deriveUrl()
if (!derived) {
  const url = await input({
    message: 'Project URL (Supabase dashboard → Project settings → API):',
    validate: (v) => v.startsWith('https://') || 'Enter the full https:// URL',
  })
  derived = { url: url.replace(/\/$/, ''), via: 'manual entry' }
}
const supabaseUrl = derived.url
console.log(`  ✓ Project URL: ${supabaseUrl} (via ${derived.via})`)

writeFileSync(
  '.env.local',
  [
    `VITE_SUPABASE_URL=${supabaseUrl}`,
    `VITE_SUPABASE_ANON_KEY=${anon}`,
    `SUPABASE_SERVICE_ROLE_KEY=${serviceRole}`,
  ].join('\n') + '\n',
)
console.log('  ✓ .env.local written (points at your cloud project)')

// ── Create the single user ──────────────────────────────────
console.log('\n── Almost done! ────────────────────────────────\n')
console.log('Create your account (this is the only login — there is no signup):\n')

const email = await input({
  message: 'Email:',
  validate: (v) => v.includes('@') || 'Enter a valid email',
})
const pass = await input({
  message: 'Password (min 6 chars):',
  validate: (v) => v.length >= 6 || 'At least 6 characters',
})

const res = await fetch(`${supabaseUrl}/auth/v1/admin/users`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${serviceRole}`,
    apikey: serviceRole,
  },
  body: JSON.stringify({ email, password: pass, email_confirm: true }),
})

if (res.ok) {
  console.log(`\n  ✓ Account created: ${email}`)
} else {
  const err = await res.json().catch(() => ({}))
  const message = err.msg ?? err.message ?? ''
  if (/already.*registered/i.test(message)) {
    console.log(`\n  ✓ ${email} already exists — you're set.`)
  } else {
    console.error(`\n  ✗ Could not create account: ${message || 'unknown error'}`)
    console.log('  Create one manually: Supabase dashboard → Authentication → Users\n')
    process.exit(1)
  }
}

// ── Lock the door: disable public signups ───────────────────
console.log('\n── Security ─────────────────────────────────────\n')
console.log('This app treats ANY account in this Supabase project as')
console.log('"logged in" — it never checks which user it is. Left on,')
console.log("anyone who finds your app's public anon key can create their")
console.log('own account and sign in as if they were you.\n')
console.log('Disable it now:')
console.log('  Supabase dashboard → Authentication → Sign In / Providers')
console.log('  → Email → turn OFF "Allow new users to sign up"')
console.log('  (exact wording varies by dashboard version)\n')

let locked = false
while (!locked) {
  locked = await confirm({
    message: "I've disabled public sign-ups for this project",
    default: false,
  })
}

console.log('\n✓ Setup complete!')
console.log('  pnpm dev → http://localhost:5180/login\n')
```

## Why each weird bit exists (do not simplify these away)

Every entry below is a live-fire fix or a real friction point observed in a
genuine wizard run. A future edit that "cleans up" any of them reintroduces a
crash or a lie.

- **`listJson` / `JSON.parse(...) ?? []`** — the CLI prints the JSON literal
  `null` (not `[]`) for empty lists. A zero-project account (exactly the
  greenfield user this wizard targets) crashes `.length` without it.
- **`fetchApiKeys()` tries with and without `--reveal`** — the flag is
  version-gated; the CLI surface drifts between releases. Worse, a
  linked-check built on the `--reveal` form alone silently reports "not
  linked" on CLI versions without the flag, re-running the link flow forever.
- **`k.id === label || k.name === label`** — the key label field drifted
  across CLI versions.
- **`sb_publishable_*` / `sb_secret_*` prefix fallback** — newer Supabase
  projects return these instead of legacy JWT anon/service_role keys.
- **Redaction detection (`/\*|REDACTED/`)** — some CLI versions redact keys
  and need `--reveal`; if output is masked, tell the user to upgrade the CLI
  rather than writing garbage to `.env.local`.
- **`deriveUrl()` chain: anon-JWT decode → `supabase/.temp/project-ref` →
  manual prompt** — legacy anon keys are JWTs carrying the project ref; new
  key formats aren't, so fall back to the ref file `supabase link` writes,
  then to asking. The script logs which path was taken.
- **The DB-password ceremony** — Supabase never shows the password again;
  project creation requires one even though auth-only setups never use it
  after. The show-and-confirm box prevents a locked-out user.
- **The Keychain note (and the `SUPABASE_ACCESS_TOKEN` banner tip)** — macOS
  asks if the terminal may access the Keychain during `supabase login`, and —
  confirmed against a live run — it keeps re-asking on nearly every later CLI
  call (`projects list`, `link`, `api-keys`, …), not just once. The original
  "Click Deny, it still works" advice was technically true but practically
  misleading: a user who takes it gets interrupted repeatedly through the
  rest of the wizard and can end up granting persistent OS-level access just
  to make it stop. `SUPABASE_ACCESS_TOKEN` is Supabase's own documented
  non-interactive auth path (built for CI, but works here too) and sidesteps
  native Keychain access altogether — the banner surfaces it *before* login,
  not after the friction has already happened. No control-flow change: the
  existing `canRun('supabase projects list --output json')` check already
  succeeds transparently once the var is set.
- **Admin-API provisioning with `email_confirm: true`** — "no signup UI"
  means the single user must be created server-side; without `email_confirm`
  the user can never log in (no confirmation email flow exists in the app).
- **"already been registered" treated as success** — makes re-running the
  wizard idempotent (re-link, re-derive keys, rewrite `.env.local`, no-op the
  user).
- **The signup-lockdown `confirm()` loops instead of pausing once** — every
  other `confirm()` in this script is a checkpoint (its answer is never
  read). This one blocks on `false` and re-asks, because the app has no
  concept of "which user" — leaving public sign-ups on means anyone who
  extracts the public anon key from the deployed bundle can register their
  own account and pass the login gate identically to the real owner. Silent
  failure here is a real security hole, not a rough edge, so it's the one
  step that must not be skippable.
- **`SUPABASE_SERVICE_ROLE_KEY` has no `VITE_` prefix** — Vite only exposes
  `VITE_*` to `import.meta.env`, so the admin key can never reach the client
  bundle. It lives in `.env.local` (gitignored) for the script only.

## Gate

`pnpm lint` green and `node --check scripts/setup.mjs` passes. **Do NOT run
the wizard** — that is the user's step (see verify-handoff).
