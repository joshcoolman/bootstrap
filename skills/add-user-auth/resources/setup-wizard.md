# Setup wizard

The interactive script that wires a real Supabase project: CLI check →
login → link-or-create project → apply migrations (if the optional example
feature was included) → derive keys → write `.env.local` → provision one or
more accounts via the admin API. **You write it; the user runs it.** Never
run it yourself — it opens a browser for login and asks for the user's
account credentials.

This merges `add-simple-auth`'s battle-tested wizard shape (CLI-drift
handling, the org/region project-creation flow, the Keychain friction note)
with two things the reference app's own wizard adds that a single
shared-credential gate never needed: applying a migration
(`supabase db push --linked`) and provisioning **more than one** account,
since proving multi-tenant isolation needs at least two.

Adapt points (only these): the banner title, the default project name, the
done-message URL (`<dev-port>` from discovery), and whether the migration
step runs at all (only if the optional `snippets` feature — or any other
migration — exists).

## `package.json` — the script name

```json
"setup:supabase": "node scripts/setup-supabase.mjs"
```

**Never name it `setup`.** pnpm's built-in `setup` command shadows package
scripts — `pnpm setup` installs the pnpm standalone binary and edits the
user's shell rc. Reference the command as `pnpm setup:supabase` everywhere
(docs, error messages, `login/actions.ts`'s `NOT_CONFIGURED` string).

## ESLint — Node globals for `scripts/`

`next-app`'s `eslint.config.mjs` defines browser/React globals only, so
`scripts/*.mjs` fails lint with dozens of `no-undef` errors. Add this block
(merge into the array before the `prettier` entry — see `next-app`'s
`shell.md`, which already flags this as its own future adapt point):

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

Add `supabase/.temp` — `supabase link` writes machine-local state
(project-ref, pooler URLs) there, which must not land in a commit.

## `scripts/setup-supabase.mjs`

```js
#!/usr/bin/env node
// Interactive auth setup: links (or creates) a hosted Supabase project,
// applies migrations, writes .env.local, and provisions one or more
// accounts (there's no signup UI — see docs/FEATURE-MODULE-PATTERN.md).
// Re-running is safe.
process.env.NODE_NO_WARNINGS = '1'
import { execSync } from 'child_process'
import { existsSync, readdirSync, readFileSync, writeFileSync } from 'fs'
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
console.log('│      your-app auth setup wizard       │')
console.log('└──────────────────────────────────────┘\n')
console.log('This links a Supabase project for per-user email/password login,')
console.log('writes .env.local, creates account(s), and locks down public')
console.log('sign-ups so no one else can self-register.\n')
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
console.log('\n── 1/4  Supabase login ─────────────────────────\n')

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
console.log('── 2/4  Project ────────────────────────────────\n')

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
        default: 'your-app',
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

// ── Step 3: Migrations (only if any exist — the example feature is optional) ──
console.log('── 3/4  Schema ─────────────────────────────────\n')

const hasMigrations = existsSync('supabase/migrations') && readdirSync('supabase/migrations').length > 0
if (hasMigrations) {
  run('supabase db push --linked')
  console.log('  ✓ Migrations applied\n')
} else {
  console.log('  (no migrations to apply — auth-only setup)\n')
}

// ── Step 4: Keys → .env.local, then account(s) ──────────────
console.log('── 4/4  Keys + accounts ────────────────────────\n')

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
const publishableKey = findKey('anon', 'sb_publishable_')
const secretKey = findKey('service_role', 'sb_secret_')

if (!publishableKey || !secretKey) {
  console.error('  ✗ Could not identify the publishable + secret keys.')
  console.log(`  Keys returned: ${rawKeys.map((k) => k.id ?? k.name).join(', ')}`)
  console.log('  Find them at: Supabase dashboard → Project settings → API keys')
  process.exit(1)
}
if (/\*|REDACTED/i.test(publishableKey + secretKey)) {
  console.error('  ✗ Your Supabase CLI redacts keys and is too old for --reveal.')
  console.log('  Upgrade it: brew upgrade supabase, then re-run pnpm setup:supabase')
  process.exit(1)
}

function deriveUrl() {
  // Legacy publishable keys are JWTs whose payload carries the project ref.
  try {
    const payload = JSON.parse(Buffer.from(publishableKey.split('.')[1], 'base64').toString())
    if (payload.ref) return { url: `https://${payload.ref}.supabase.co`, via: 'key JWT' }
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
  [`NEXT_PUBLIC_SUPABASE_URL=${supabaseUrl}`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=${publishableKey}`].join('\n') +
    '\n',
)
console.log('  ✓ .env.local written (points at your cloud project)\n')

console.log("There's no signup UI in this app on purpose — accounts are provisioned")
console.log('here (or in the dashboard), not self-registered. Create at least two')
console.log('accounts now if you want to see per-user data isolation in action.\n')

let again = true
while (again) {
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
      Authorization: `Bearer ${secretKey}`,
      apikey: secretKey,
    },
    body: JSON.stringify({ email, password: pass, email_confirm: true }),
  })

  if (res.ok) {
    console.log(`  ✓ Account created: ${email}\n`)
  } else {
    const err = await res.json().catch(() => ({}))
    const message = err.msg ?? err.message ?? ''
    if (/already.*registered/i.test(message)) {
      console.log(`  ✓ ${email} already exists — you're set.\n`)
    } else {
      console.error(`  ✗ Could not create account: ${message || 'unknown error'}\n`)
    }
  }

  again = await confirm({ message: 'Create another account?', default: false })
}

// ── Lock the door: disable public signups ───────────────────
console.log('── One more thing ───────────────────────────────\n')
console.log("Disable public sign-ups so random visitors can't self-register")
console.log('(this app has no signup UI, but the endpoint stays reachable with')
console.log('the public key otherwise):')
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
console.log('  pnpm dev → http://localhost:<dev-port>/login\n')
```

## Why each weird bit exists (do not simplify these away)

Every entry below is a live-fire fix carried over from `add-simple-auth`'s
own wizard (same CLI, same failure modes) plus one new addition for this
skill.

- **`listJson` / `JSON.parse(...) ?? []`** — the CLI prints the JSON literal
  `null` (not `[]`) for empty lists. A zero-project account crashes
  `.length` without it.
- **`fetchApiKeys()` tries with and without `--reveal`** — version-gated
  flag; a linked-check built on the `--reveal` form alone silently reports
  "not linked" on CLI versions without it, re-running the link flow
  forever.
- **`k.id === label || k.name === label`** and the
  **`sb_publishable_*`/`sb_secret_*` prefix fallback** — the key label
  field and key format both drift across CLI versions and project ages.
- **Redaction detection (`/\*|REDACTED/`)** — some CLI versions redact keys
  and need `--reveal`; if output is masked, tell the user to upgrade rather
  than write garbage to `.env.local`.
- **`deriveUrl()` chain: key-JWT decode → `supabase/.temp/project-ref` →
  manual prompt** — legacy keys are JWTs carrying the project ref; newer
  key formats aren't, so fall back to the ref file `supabase link` writes,
  then to asking.
- **The DB-password ceremony** — Supabase never shows the password again;
  project creation requires one even though this wizard never needs it
  again after. The show-and-confirm box prevents a locked-out user.
- **The Keychain note + `SUPABASE_ACCESS_TOKEN` tip** — macOS re-prompts
  Keychain access on nearly every CLI call during a fresh login, not just
  once; the token env var sidesteps it entirely and is surfaced before
  login happens, not after the friction already did.
- **Migrations are conditional on `supabase/migrations/` existing** — this
  skill's example feature is optional, so a plain auth-only setup has no
  migrations directory at all; running `supabase db push --linked` against
  an empty/missing directory would either no-op unhelpfully or error
  depending on CLI version, so check first and skip the step cleanly.
- **The account-creation loop asks "create another?" by default** (new vs.
  `add-simple-auth`, which provisions exactly one account) — proving
  multi-tenant isolation needs at least two distinct accounts, so the
  wizard nudges toward creating more than one rather than stopping after
  the first.
- **Admin-API provisioning with `email_confirm: true`** — no signup UI
  means every account must be created server-side; without
  `email_confirm` the account can never log in.
- **"already registered" treated as success** — makes re-running the
  wizard idempotent to add more accounts later.
- **The signup-lockdown `confirm()` loops instead of pausing once** — every
  other `confirm()` here is a one-time checkpoint. This one blocks until
  `true` because leaving public sign-ups on is a real security hole (anyone
  who extracts the public key from the deployed bundle could register
  their own account), not a rough edge — it's the one step that must not
  be skippable.
- **The secret key never gets a `NEXT_PUBLIC_` prefix** — Next only exposes
  `NEXT_PUBLIC_*` vars to the client bundle, so the admin key can never
  reach the browser. It's used by this script only, never written anywhere
  the app itself reads.

## Gate

`pnpm lint` green and `node --check scripts/setup-supabase.mjs` passes. **Do
NOT run the wizard** — that is the user's step (see `verify-handoff.md`).
