# Part: knowledge

Two things: the `knowledge/` folder (plain markdown files that encode domain
taste) and the `src/features/` seam pattern (the structural skeleton that
separates mechanism from knowledge and keeps features from tangling).

These belong together because the feature seam pattern is specifically designed
to keep knowledge in one place and mechanism in another.

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

A small loader (the "palette-forge pattern", ~15 lines) reads all files at
build time via Vite's glob:

```ts
const files = import.meta.glob('/knowledge/*.md', {
  query: '?raw',
  import: 'default',
  eager: true,
})

export function getKnowledge(name: string): string {
  const key = `/knowledge/${name}.md`
  return (files[key] as string) ?? ''
}
```

This lives in `src/features/knowledge/`. Call `getKnowledge('rubric')` to get
the contents of `knowledge/rubric.md` as a string — pass it directly into a
prompt or verifier.

### The key property

Knowledge files are plain markdown. A non-coder (or a non-programmer agent
run) can open `knowledge/rubric.md`, read it, and update it to change the
app's behavior without touching code. The mechanism stays in code; the taste
stays here. This is the knowledge-as-extension-surface bet — it's what makes
the app tunable.

## The `src/features/` seam pattern

Each feature gets its own folder under `src/features/`. Nothing is placed
directly in `src/` except router, styles, app-meta, and shared utilities.

### Folder structure

```
src/features/
  <feature-name>/
    CLAUDE.md      what this feature owns and how to build it
    types.ts       the domain types / contracts this feature defines (if any)
    index.ts       public API of the feature (if it has one)
```

Start with just `CLAUDE.md`. Add `types.ts` when the contracts are clear. Add
implementation files as phases complete. Keep the folder thin until there's
real work to do.

### What features to create

One folder per domain concern. Common set for a BYO-key, single-user utility:

- **The core feature** — the domain logic (e.g. `outpaint`, `improve`). Owns
  the primary record type and the orchestration seam.
- **`generate`** — the AI generation step. Owns the provider call and the
  prompt assembly (using knowledge files).
- **`verify`** — the AI verification step. Owns the vision or rubric check
  and emits a schema-validated verdict.
- **`knowledge`** — the loader that reads `knowledge/*.md` and exposes it to
  the rest of the app.
- **`prefs`** — local user preferences: BYO API key, provider selection,
  free-tier state. Uses IndexedDB, no backend.

### `CLAUDE.md` per feature

Every feature folder gets a `CLAUDE.md`. It describes:
- What this feature owns (its responsibility boundary)
- What it does NOT own (where adjacent features take over)
- Key design decisions (e.g. "records not React state", "thin orchestrator")
- Where to read more (usually `docs/PLAN.md`)

This is the primary orientation doc for an agent starting work on that feature.
Keep it short — a page or less. It should answer "what is this and what do I
touch here?" not explain every implementation detail.

### `types.ts` per feature

Define the core domain types here before writing any implementation. Types are
the contract between features. They should be stable — implementation changes
constantly, types change rarely.

Schema-validate verifier outputs (use Zod or a plain type guard). A verifier
that returns untyped JSON is not a verifier.

### The core rule

**Features are isolated.** A feature imports from its own folder, from shared
utilities, or from another feature's public `index.ts`. It does NOT reach into
another feature's internals. The `knowledge` feature is the one exception —
it's the universal provider and everything can import from it.

The orchestrator (the core feature) sequences the others. It calls `generate`,
then `verify`, then decides what to do based on the verdict. It holds no
provider logic or vision logic itself — those live in their respective
features.
