# Part: schema reference

A generated, read-only map of the database — one markdown page per table,
rendered in the `/docs` viewer. Applies to any project with a Postgres.

Proven in `bootsy` (the live `next-railway-app`), which is why this is extracted
rather than speculative.

## The gap this fills

`migrations/*.sql` is a **changelog** (what changed, when), not a **map** (what
is). Knowing the current shape of a table means replaying every migration that
ever touched it, in order. In bootsy that was six migrations for one table — and
the first one created `owner_id` as `text`, which a later one swapped to a `uuid`
foreign key. So reading any single migration told you the wrong thing.

The database itself always knows the answer. What's missing is a **readable copy
of it**, in one place, that a human or an agent can scan.

The payoff is precision in conversation. You look at the map and say "add three
colors to `admin_theme`" instead of "add some colors to the theme thing," and the
agent doesn't guess. That is the point — this is a **communication device**, not
a type-safety mechanism.

## What it is NOT

**Not an ORM, and not a step toward one.** No query is rewritten, no query
builder is introduced, no dependency is added. Migrations stay hand-written SQL;
SQL remains the source of truth. This adds a *reading surface*, nothing else.

**Not a source of truth.** The generated pages are reference only. Nothing
imports them. Editing them changes nothing. The database is the truth; these are
a rendering of it.

## The three ideas that make it work

Take these even if you re-implement everything else. They are the load-bearing
parts, and only the first is obvious.

**1. Markdown, not TypeScript.** The tempting version generates a
`src/db/schema/*.ts` folder of interfaces. Don't. A generated `.ts` file that
nothing imports still *looks* like code — someone will eventually edit it, it
will typecheck, and they will believe they changed the database. Markdown refuses
that misreading by its extension. (If you later want row types derived from the
schema so a renamed column breaks the build, that is a real and separate benefit
— add a `.ts` emitter alongside. Just don't confuse it with this.)

**2. A hand-maintained meaning layer.** Introspection gives you *shape* and never
*meaning*. Postgres has no idea which of your tables is the heart of the app and
which is periphery. Ten tables listed alphabetically is not a map — it's an
inventory. So one small prose file assigns each table a purpose, a group, and its
place in the reading order. It is prose, not contract: a stale line there is a
mild annoyance, never a lying schema, so it carries none of the drift risk of a
hand-written type mirror.

**3. A CI check, or none of this holds.** Codegen alone is still *discipline* — a
generated file is accurate only if somebody remembered to regenerate it. Forget
once and you have a confident, authoritative-looking document that is wrong,
which is worse than no document. The regenerate-and-diff step is what converts
discipline into **mechanism**: land a migration without regenerating and the
build goes red.

If you skip #3, skip the whole part. A stale map is a liability.

## Requires

- **Postgres.** The introspection queries hit `information_schema` and
  `pg_catalog`. MySQL/SQLite need different queries — the *pattern* ports, this
  script does not.
- **The `postgres` client** (`postgres` on npm), which a Postgres project already
  has. No new dependency. Deliberately no `pg_dump` — that would need a
  system-level postgres-client install in CI.
- **A `DATABASE_URL` reachable from CI.** This is the real constraint, and it is
  not negotiable: without it, the tripwire in #3 cannot run and you are back to
  "someone remembered." If CI has no database, don't adopt this.

## Known limits

"Extracted from a live run" guarantees less than it sounds like. It has met
exactly one schema — ten tables, one enum, single-column foreign keys, composite
primary keys, composite uniques, partial indexes. Everything below is either
handled-but-never-exercised or not handled at all. If you hit one, **fix it
here**, and delete the line.

- **Composite foreign keys** — implemented (`conkey`/`confkey` return ordered
  column arrays, rendered as ``→ `t` (a, b) via (x, y)``), never exercised.
- **CHECK and EXCLUSION constraints** — not rendered at all. A `check (price > 0)`
  is invisible on the map, which is a real omission: it is enforcement the map
  does not show.
- **Views** — excluded (`table_type = 'BASE TABLE'`). Deliberate: otherwise the
  first view anyone adds gets treated as a table and refuses to regenerate until
  annotated. Widening this is a one-line change, noted in the script.
- **Only the `public` schema** is read.
- **Generated / identity columns, partitioned tables, domains** — untested.

One warning from how this part got its scars. The obvious `information_schema`
join (`key_column_usage` × `constraint_column_usage`) **cross-products a
composite constraint into per-column rows**, and rendering those independently
makes the map *lie*: `unique (owner_id, image_url)` comes back as "owner_id is
unique" and "image_url is unique" — claiming a uniqueness neither column has.
That shipped, in four tables, before it was caught. The script reads
`pg_constraint` instead for exactly this reason. Don't "simplify" it back.

A map that is merely incomplete is fine — you go look at the migration. A map
that is confidently wrong is worse than no map, because you don't go look.

## `scripts/gen-schema.mjs`

Project-agnostic — contains no facts about any particular app. Copy verbatim.
Read-only: catalog `select`s and nothing else, which matters when `DATABASE_URL`
points at a shared or production database.

```js
// Generates docs/schema/*.md — a readable map of the live database.
//
// Usage: pnpm db:gen
//
// The database is the source of truth. migrations/*.sql is a changelog (what
// changed, when), not a map (what is): knowing the current shape of a table
// means replaying every migration that ever touched it. This script asks
// Postgres what it actually looks like right now, and writes it down.
//
// The output is REFERENCE ONLY. Nothing imports it, nothing depends on it,
// editing it changes nothing. To change the schema, write a migration.
//
// CI regenerates and fails on any diff, so the map cannot go stale — see the
// "Schema reference is current" step in .github/workflows/ci.yml.
//
// This file is deliberately project-agnostic: every fact about this particular
// app lives in schema-annotations.mjs. Point it at a different Postgres, swap
// that one file, and it works unchanged.

import { existsSync, mkdirSync, readdirSync, rmSync, writeFileSync } from 'node:fs'
import { join } from 'node:path'
import postgres from 'postgres'
import { EXCLUDED_TABLES, GROUPS, TABLES } from './schema-annotations.mjs'

const ENV_LOCAL_PATH = new URL('../.env.local', import.meta.url)
if (existsSync(ENV_LOCAL_PATH)) process.loadEnvFile(ENV_LOCAL_PATH)

const databaseUrl = process.env.DATABASE_URL
if (!databaseUrl) {
  console.error('DATABASE_URL is not set.')
  process.exit(1)
}

const OUT_DIR = new URL('../docs/schema/', import.meta.url).pathname

const BANNER = [
  '> _Generated from the live database by `pnpm db:gen`. Do not edit._',
  '> _To change the schema, write a migration in `migrations/` and regenerate._',
].join('\n')

// Read-only: catalog selects only. The local DATABASE_URL points at the same
// Postgres as production — this script must never write.
const sql = postgres(databaseUrl)

// Base tables only. information_schema.columns also reports views, which would
// otherwise be picked up as tables and demand an annotation — a confusing
// failure for anyone who adds their first view and suddenly cannot regenerate.
// If you want views on the map, widen this to include 'VIEW' and give them
// annotations like any other entry.
const baseTables = await sql`
  select table_name
  from information_schema.tables
  where table_schema = 'public' and table_type = 'BASE TABLE'
`
const baseTableNames = new Set(baseTables.map((t) => t.table_name))

const columns = await sql`
  select table_name, column_name, udt_name, is_nullable, column_default, ordinal_position
  from information_schema.columns
  where table_schema = 'public'
  order by table_name, ordinal_position
`

// Primary keys, unique constraints and foreign keys — one row per CONSTRAINT,
// with its columns as an ordered array.
//
// Read from pg_constraint rather than information_schema. The obvious
// information_schema query joins key_column_usage (one row per column) against
// constraint_column_usage, which cross-products a composite constraint into
// per-column rows — and rendering those independently makes the map LIE: a
// `unique (owner_id, image_url)` comes back as "owner_id is unique" and
// "image_url is unique", which claims a uniqueness neither column has.
// conkey/confkey give the real thing: ordered column arrays, composites intact.
const constraints = await sql`
  select
    con.conname as name,
    rel.relname as table_name,
    con.contype as type,
    tgt.relname as foreign_table,
    con.confdeltype as delete_type,
    (
      select array_agg(att.attname order by k.ord)
      from unnest(con.conkey) with ordinality k(attnum, ord)
      join pg_attribute att on att.attrelid = con.conrelid and att.attnum = k.attnum
    ) as columns,
    (
      select array_agg(att.attname order by k.ord)
      from unnest(con.confkey) with ordinality k(attnum, ord)
      join pg_attribute att on att.attrelid = con.confrelid and att.attnum = k.attnum
    ) as foreign_columns
  from pg_constraint con
  join pg_class rel on rel.oid = con.conrelid
  join pg_namespace ns on ns.oid = rel.relnamespace
  left join pg_class tgt on tgt.oid = con.confrelid
  where ns.nspname = 'public' and con.contype in ('p', 'u', 'f')
  order by con.conname
`

// Enum types and their variants, so a status column can show what it accepts.
const enums = await sql`
  select t.typname as name, e.enumlabel as value
  from pg_type t
  join pg_enum e on e.enumtypid = t.oid
  order by t.typname, e.enumsortorder
`

const indexes = await sql`
  select tablename as table_name, indexname as name, indexdef as def
  from pg_indexes
  where schemaname = 'public'
  order by tablename, indexname
`

await sql.end()

const enumValues = new Map()
for (const row of enums) {
  const list = enumValues.get(row.name) ?? []
  list.push(row.value)
  enumValues.set(row.name, list)
}

const tableNames = [...new Set(columns.map((c) => c.table_name))].filter(
  (t) => baseTableNames.has(t) && !EXCLUDED_TABLES.includes(t),
)

// Reading order comes from the annotations file, not the alphabet: a map should
// follow the way the data actually flows. Anything in the database but not yet
// annotated sorts to the end, where the unannotated check below will catch it.
const annotationOrder = Object.keys(TABLES)
tableNames.sort((a, b) => {
  const ia = annotationOrder.indexOf(a)
  const ib = annotationOrder.indexOf(b)
  return (ia === -1 ? Infinity : ia) - (ib === -1 ? Infinity : ib)
})

// A new table must not slip onto the map unlabelled.
const unannotated = tableNames.filter((t) => !TABLES[t])
if (unannotated.length > 0) {
  console.error(
    `No annotation for: ${unannotated.join(', ')}\n` +
      `Add a purpose and group to scripts/schema-annotations.mjs, then re-run.`,
  )
  process.exit(1)
}

// Annotations for tables that no longer exist are equally a bug — a dropped
// table would otherwise linger in the map's source with no output to match.
const orphaned = Object.keys(TABLES).filter((t) => !tableNames.includes(t))
if (orphaned.length > 0) {
  console.error(
    `Annotated but not in the database: ${orphaned.join(', ')}\n` +
      `Remove them from scripts/schema-annotations.mjs, then re-run.`,
  )
  process.exit(1)
}

// Numeric prefixes are not decoration: the docs viewer (src/features/docs/
// build-docs.ts) reads them as sort order, so the sidebar — and a plain GitHub
// folder listing — reads in the order the data flows rather than alphabetically.
const fileFor = (table) =>
  `${String(tableNames.indexOf(table) + 1).padStart(2, '0')}-${table.replace(/_/g, '-')}.md`

// Postgres echoes its internal casts back at you — `'{}'::text[]`,
// `'pending'::sub_job_status`, `ARRAY['done'::sub_job_status]`. They're accurate
// and they're noise in a document whose whole job is being readable. The type is
// already in the Type column.
function stripCasts(value) {
  return value.replace(/::[a-z_]+(\[\])?/gi, '')
}

function formatDefault(value) {
  if (value == null) return ''
  return `\`${stripCasts(value)}\``
}

// Postgres spells the on-delete rule as a single char in confdeltype.
const DELETE_RULES = { c: 'on delete cascade', n: 'on delete set null', d: 'on delete set default', r: 'on delete restrict' }

// A composite constraint must never render as though each of its columns
// carried it alone — "unique" on owner_id and "unique" on image_url claims a
// uniqueness that neither column has. When a constraint spans several columns,
// name them all, so the reader can see it is the combination that is
// constrained.
// Notes read primary key → foreign key → unique, always. Left to pg_constraint's
// own row order the notes would shuffle between runs, and the CI diff check
// would flag a schema that had not changed.
const CONSTRAINT_ORDER = { p: 0, f: 1, u: 2 }

function notesFor(table, column) {
  const notes = []
  const name = column.column_name
  const forColumn = constraints
    .filter((c) => c.table_name === table && (c.columns ?? []).includes(name))
    .sort((a, b) => CONSTRAINT_ORDER[a.type] - CONSTRAINT_ORDER[b.type] || a.name.localeCompare(b.name))

  for (const c of forColumn) {
    const cols = c.columns ?? []
    const composite = cols.length > 1
    const list = cols.map((col) => `\`${col}\``).join(', ')

    if (c.type === 'p') {
      notes.push(composite ? `primary key (${list})` : 'primary key')
    } else if (c.type === 'u') {
      notes.push(composite ? `unique (${list})` : 'unique')
    } else if (c.type === 'f') {
      const target = c.foreign_columns ?? []
      const rule = DELETE_RULES[c.delete_type]
      const suffix = rule ? ` (${rule})` : ''
      const arrow =
        cols.length > 1 || target.length > 1
          ? `→ \`${c.foreign_table}\` (${target.join(', ')}) via (${cols.join(', ')})`
          : `→ \`${c.foreign_table}.${target[0]}\``
      notes.push(`${arrow}${suffix}`)
    }
  }

  const values = enumValues.get(column.udt_name)
  if (values) notes.push(values.map((v) => `\`${v}\``).join(' / '))

  return notes.join(' · ')
}

// `_text` is how Postgres names text[] internally; nobody reading a reference
// doc wants to decode that.
function displayType(udtName) {
  return udtName.startsWith('_') ? `${udtName.slice(1)}[]` : udtName
}

function renderTable(table) {
  const { purpose } = TABLES[table]
  const cols = columns.filter((c) => c.table_name === table)

  const rows = cols.map((c) => {
    const cells = [
      `\`${c.column_name}\``,
      displayType(c.udt_name),
      c.is_nullable === 'YES' ? 'yes' : 'no',
      formatDefault(c.column_default),
      notesFor(table, c),
    ]
    return `| ${cells.join(' | ')} |`
  })

  const lines = [
    `# ${table}`,
    '',
    BANNER,
    '',
    purpose,
    '',
    '| Column | Type | Null | Default | Notes |',
    '| --- | --- | --- | --- | --- |',
    ...rows,
  ]

  const tableIndexes = indexes.filter((i) => i.table_name === table)
  if (tableIndexes.length > 0) {
    lines.push('', '## Indexes', '')
    for (const index of tableIndexes) {
      // Strip the boilerplate prefix; what matters is the columns and any
      // partial WHERE clause.
      const def = stripCasts(index.def.replace(/^CREATE (UNIQUE )?INDEX \S+ ON \S+ USING \w+ /i, ''))
      const unique = /^CREATE UNIQUE/i.test(index.def) ? 'unique · ' : ''
      lines.push(`- \`${index.name}\` — ${unique}${def}`)
    }
  }

  return `${lines.join('\n')}\n`
}

function renderIndex() {
  const lines = ['# Schema', '', BANNER, '']

  lines.push(
    'The shape of the database as it exists right now, one page per table.',
    'Grouped by what each table is for, because Postgres does not know that and you do.',
    '',
  )

  for (const group of GROUPS) {
    const inGroup = tableNames.filter((t) => TABLES[t].group === group)
    if (inGroup.length === 0) continue
    lines.push(`## ${group}`, '')
    for (const table of inGroup) {
      lines.push(`- [\`${table}\`](${fileFor(table)}) — ${TABLES[table].summary}`)
    }
    lines.push('')
  }

  return `${lines.join('\n').trimEnd()}\n`
}

// Rebuild the folder from scratch so a dropped table's page cannot survive as
// a stale file that nothing regenerates.
if (existsSync(OUT_DIR)) {
  for (const file of readdirSync(OUT_DIR)) {
    if (file.endsWith('.md')) rmSync(join(OUT_DIR, file))
  }
} else {
  mkdirSync(OUT_DIR, { recursive: true })
}

writeFileSync(join(OUT_DIR, 'index.md'), renderIndex())
for (const table of tableNames) {
  writeFileSync(join(OUT_DIR, fileFor(table)), renderTable(table))
}

console.log(`docs/schema — index.md + ${tableNames.length} tables`)
```

## `scripts/schema-annotations.mjs`

The meaning layer — the **only** hand-maintained file in the pipeline. This is a
template; replace the tables with your own.

Order within a group follows declaration order here, not the alphabet: the map
should read the way the data flows (a user submits a request, which fans out into
jobs), and that is a fact only a human knows.

```js
// The meaning layer for `pnpm db:gen`.
//
// Introspection gives the generator the *shape* of the database. It can never
// give it *meaning* — Postgres has no idea which table is the heart of the app
// and which is periphery. Ten tables listed alphabetically is not a map. These
// lines are what make it one.
//
// Prose, not contract: a stale description here is a mild annoyance, never a
// lying schema.
//
// Add a table without adding it here and `pnpm db:gen` fails, on purpose.

// Section order in the generated index.
export const GROUPS = ['Core flow', 'Admin & tuning', 'User-scoped']

// `summary` is the one-liner on the index page; `purpose` is the fuller
// description at the top of the table's own page.
export const TABLES = {
  users: {
    group: 'Core flow',
    summary: 'People who can log in — every other table hangs off this one.',
    purpose: 'People who can log in. Every other table hangs off this one.',
  },
  // ... one entry per table, in reading order
}

// Bookkeeping tables that are not domain data — kept off the map.
export const EXCLUDED_TABLES = ['schema_migrations']
```

## `package.json`

```json
"db:gen": "node scripts/gen-schema.mjs && prettier --write --log-level silent docs/schema"
```

Prettier keeps the output byte-stable so regeneration is deterministic — which
the CI diff check depends on.

## Docs viewer — three changes

The generated pages land in `docs/schema/`, which the `/docs` viewer already
globs. Three adjustments to the classifier in [docs](../skills/next-app/resources/docs.md) (`build-docs.ts` in
the Next variant, `docs.tsx` in the Vite one — the logic is identical):

**Give schema its own section**, or eleven pages flood "Start here":

```ts
const SECTION_ORDER = ['Start here', 'Schema', 'Knowledge', 'Working notes']

function sectionFor(path: string): string {
  if (path.startsWith('/docs/schema/')) return 'Schema'
  // ... existing cases
}
```

**Label table pages with their exact table name.** The sidebar is where you go to
find the precise word to say to an agent. `sub_jobs` is the thing; "Sub jobs" is
a paraphrase of it:

```ts
function titleFor(path: string, name: string): string {
  if (path.startsWith('/docs/schema/')) return name.replace(/-/g, '_')
  if (/^(README|CLAUDE)$/i.test(name)) return name
  return titleCase(name.replace(/-/g, ' '))
}
```

**Let a folder's `index.md` lead its section**, titled "Overview" — otherwise the
map sorts under "I" and lands in the middle of its own tables:

```ts
const isIndex = /^index$/i.test(name)
return {
  id: slugify([...segs, name].join('/')),
  section: sectionFor(path),
  title: isIndex ? 'Overview' : titleFor(path, name),
  order: isIndex ? -1 : order,
}
```

Ordering inside the section is already handled: the viewer reads the `NN-`
filename prefix as sort order, and the generator emits `01-users.md`,
`02-generation-requests.md`, … in annotation order. So the sidebar — and a plain
GitHub folder listing — reads in the order the data flows.

These are pure functions with three special cases now. Worth a small test file;
see `src/features/docs/build-docs.test.ts` in bootsy.

## CI — the step that makes it real

In `.github/workflows/ci.yml`, **after** the migration step (so the CI database
is at head) and before tests:

```yaml
- name: Schema reference is current
  run: |
    pnpm db:gen
    # --intent-to-add so a brand-new table's page (untracked, and therefore
    # invisible to `git diff`) still trips the check.
    git add --intent-to-add docs/schema
    git diff --exit-code docs/schema
```

The `--intent-to-add` line is not optional and is easy to leave out. A bare
`git diff --exit-code` **ignores untracked files** — so adding a table and
forgetting to regenerate produces a new, untracked page that `git diff` cannot
see, and the check passes green for the exact case it exists to catch.

## CLAUDE.md

Two lines, so a fresh agent session doesn't reconstruct the schema from
migrations or hand-edit generated files:

```md
- `docs/schema/` is **generated, reference only** — `pnpm db:gen` introspects the
  live database and writes it. Never edit those files; to change the schema,
  write a migration and regenerate. Run `pnpm db:gen` whenever you add a
  migration (CI fails if the map is stale), and add the new table's purpose to
  `scripts/schema-annotations.mjs` (the generator refuses to run without it).
```

And in the "read first" list: `docs/schema/index.md` — the shape of the database.
Read this instead of replaying `migrations/*.sql`.

## What "done" looks like

- `pnpm db:gen` writes `docs/schema/index.md` plus one page per table
- `/docs` shows a **Schema** section; "Overview" leads it, then the tables in
  data-flow order, each labelled with its exact table name (`sub_jobs`)
- Running `pnpm db:gen` twice produces no git diff (output is deterministic)
- Hand-editing a generated page and committing it, then regenerating, produces a
  diff — CI would go red
- Adding a table without annotating it exits 1, naming the table and the file to
  fix
- Spot-check one table's page against every migration that touched it: all
  columns present, foreign keys pointing at the *current* target, all indexes
  listed

## Porting the pattern past Postgres

The script is Postgres-bound. The pattern isn't. Anywhere you have a live source
of truth that can be interrogated — an API surface, a job registry, a config
schema, an event catalog — the same three moves apply: generate a read-only
markdown reference from the real thing, add a hand-written layer supplying the
meaning that introspection can't, and put a regenerate-and-diff check in CI so it
cannot go stale. Swap the introspection queries; keep the shape.
