# Part: knowledge

Two things: the `knowledge/` folder (plain markdown files that encode domain
taste) and the `src/features/` seam pattern (the structural skeleton that
separates mechanism from knowledge and keeps features from tangling). The loader
reads files with Node's `fs`, since there's no bundler-level glob in Next.

## The `knowledge/` folder

A folder of plain markdown files at the root of the repo. Each file encodes
one aspect of domain taste or quality criteria — the kind of thing a
non-coder could read and rewrite to tune the app's behavior without touching
code.

### What goes here

Files are app-specific, but the pattern is consistent:

- **Guidance files** — what the app should produce and how (composition
  principles, style preferences, domain-specific craft rules). These feed the
  generation or improvement prompt.
- **Rubric files** — what "good" looks like, expressed as failure modes and
  pass criteria. These feed the verifier or checker. A rubric file names each
  failure mode, describes it concretely, and says when a result passes.

### How knowledge is loaded

A bundler-level API like `import.meta.glob` is build-time and has no Next
equivalent, so this reads the file directly with Node's `fs` instead. The public
shape is a one-liner — `getKnowledge(name)` returns a string:

```ts
import { readFileSync } from 'node:fs'
import { join } from 'node:path'
import 'server-only'

export function getKnowledge(name: string): string {
  try {
    return readFileSync(join(process.cwd(), 'knowledge', `${name}.md`), 'utf-8')
  } catch {
    return ''
  }
}
```

This lives in `src/features/knowledge/index.ts`. Call `getKnowledge('rubric')`
to get the contents of `knowledge/rubric.md` as a string — pass it directly
into a prompt or verifier.

**`import 'server-only'`** is the one real adaptation this loader needs
beyond swapping the read mechanism. `node:fs` cannot run in the browser, so
if any Client Component ever imports this module — even transitively through
another module — the build must fail loudly with a clear "this is server-only"
error, not silently try to ship `node:fs` into a client bundle (which fails
anyway, just with a much more confusing error pointing at a random Node
built-in instead of at the actual mistake). Every feature that reads
`knowledge/*.md` must do so from a Server Component, a Route Handler, or a
Server Action — never a Client Component.

### The key property

Knowledge files are plain markdown. A non-coder (or a non-programmer agent
run) can open `knowledge/rubric.md`, read it, and update it to change the
app's behavior without touching code. The mechanism stays in code; the taste
stays here. This is the knowledge-as-extension-surface bet — it's what makes
the app tunable.

## The `src/features/` seam pattern

Each feature gets its own folder under `src/features/`. Nothing is placed
directly in `src/` except the App Router tree (`src/app/`), styles, app-meta,
and shared utilities.

### Folder structure

```
src/features/
  <feature-name>/
    CLAUDE.md      what this feature owns and how to build it
    types.ts       the domain types / contracts this feature defines (if any)
    index.ts       public API of the feature (if it has one)
```

Start thin — a `CLAUDE.md` where the folder is a real boundary (see below),
nothing if it isn't yet. Add `types.ts` when the contracts are clear, and
implementation files as work lands. Keep the folder thin until there's real work
to do.

### What features to create

One folder per domain concern. Common set for a BYO-key, single-user utility:

- **The core feature** — the domain logic (e.g. `outpaint`, `improve`). Owns
  the primary record type and the orchestration seam.
- **`generate`** — the AI generation step. Owns the provider call and the
  prompt assembly (using knowledge files).
- **`verify`** — the AI verification step. Owns the vision or rubric check
  and emits a schema-validated verdict.
- **`knowledge`** — the loader that reads `knowledge/*.md` and exposes it to
  the rest of the app. Server-only (see above) — anything downstream that
  needs it must also live server-side, or receive the already-read string as
  a prop/argument from something that does.
- **`prefs`** — local user preferences: BYO API key, provider selection,
  free-tier state. Client-side (IndexedDB or localStorage), no backend — this
  one *is* fine to import from Client Components.

### `CLAUDE.md` per feature

A feature folder gets a `CLAUDE.md` **only where it's a boundary you could
violate without reading it** — an invariant, an ownership rule, a dependency
direction, a "don't do X here." This is the standard's test, and a missing one
is correct when there's nothing non-obvious to say: an empty seam earns no
`CLAUDE.md`. `knowledge` earns one (its server-only boundary fails silently if
crossed); an empty `core` does not, until it owns something.

Where one is warranted, it describes:
- What this feature owns (its responsibility boundary)
- What it does NOT own (where adjacent features take over) — name who owns
  each not-owned thing instead
- Invariants that fail silently if broken, and decisions worth not
  relitigating, with the *why* (e.g. "records not React state", "thin
  orchestrator", "server-only, never imported by a Client Component")

Not a file inventory, not a restatement of the global rules. Keep it short — it's
re-read every time an agent touches the folder, so length tracks the boundary's
complexity. For orientation on what to build next, an agent reads the README
`## Status` block and open issues, not a plan file.

### `types.ts` per feature

Define the core domain types here before writing any implementation. Types
are the contract between features. They should be stable — implementation
changes constantly, types change rarely.

Schema-validate verifier outputs (use Zod or a plain type guard). A verifier
that returns untyped JSON is not a verifier.

### The core rule

**Features are isolated.** A feature imports from its own folder, from shared
utilities, or from another feature's public `index.ts`. It does NOT reach
into another feature's internals. The `knowledge` feature is the one
exception — it's the universal provider and everything can import from it
(subject to the server-only boundary above).

The orchestrator (the core feature) sequences the others. It calls
`generate`, then `verify`, then decides what to do based on the verdict. It
holds no provider logic or vision logic itself — those live in their
respective features.
