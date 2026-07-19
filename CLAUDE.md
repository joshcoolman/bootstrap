# bootstrap — agent orientation

Read the [README](README.md) for what this repo is and how it's consumed.
This file is the **authoring contract**: the rules for adding or changing
skills without eroding the guarantees that make them one-shot reliable.

The value of this repo is not code templates — models can write a Next.js app.
The value is validated judgment: sequencing that's known to work, verification
gates, and gotcha ledgers earned from live runs. Every rule below protects
that.

## The admission bar

**No skill lands without having been run for real.** Skills are developed by
building the actual feature in a live "mule" repo first, logging every
friction point as it happens, validating end-to-end (including the
interactive human steps — that's where the live-fire bugs are), and only then
codifying. A skill written from imagination is a prompt, not a skill.

## The development loop for a new skill

1. **Scaffold a mule.** Fresh folder, `/bootstrap:next-app` (dogfoods the
   app skill every time). For a skill that layers on something other than the
   app, start from whatever it layers on.
2. **Build the feature for real** in the mule. Create `docs/SKILL-NOTES.md`
   at the start and log findings as they happen — the notes file is the
   durable artifact; the mule repo is disposable.
3. **Validate end-to-end**, including the parts only a human can drive
   (browser logins, vendor dashboards, interactive wizards). Fix what breaks;
   log why it broke.
4. **Codify** into `skills/<name>/` per the packaging conventions below,
   harvesting SKILL-NOTES. Embedded code must match the validated mule files
   byte-for-byte apart from explicitly marked adapt points — diff them
   mechanically, don't eyeball.
5. **Integration-test the skill as a consumer**: fresh folder,
   `claude --plugin-dir <this repo>`, run the real flow
   (`/bootstrap:next-app` → `/bootstrap:deploy-next-railway`). Use
   `/reload-plugins` to iterate on skill edits mid-session.
6. Commit. Delete the mule whenever.

## The `add-*` contract

Skills that layer onto an existing app must:

- **Assume the stack, discover the shell.** Next.js App Router + pnpm may be
  assumed; the repo's shape may not. Step 0 is a discovery pass that fills
  adaptation slots (`<alias>`, `<features-dir>`, `<dev-port>`, the styling
  idiom, the docs to update) which every later step references. See
  `skills/deploy-next-railway/resources/discovery.md` for the template.
- **Edit, don't clobber.** Instructions phrased as "add this block / change
  this line, preserving what's there" — layering skills modify files they
  don't own.
- **Re-weld the docs.** If the target repo documents its boundary, the skill
  updates that boundary — code that contradicts the repo's own docs is a bug.
- **Show a delta tree**, not a full tree: `← new` / `← edited` annotations
  over the existing app.

## The finish contract

- **An agent-verifiable gate comes before any human handoff.** Design the
  feature so its code layer is provable without external accounts — `next-app`
  builds and tests green with zero env vars set, and its auth is self-hosted,
  so the whole login flow verifies locally with no vendor account at all.
- **Interactive vendor steps always belong to the user.** The skill authors
  wizards into the target repo (re-runnable, idempotent) and hands off with a
  preview of exactly what the wizard will ask. The agent never drives browser
  logins or types the user's credentials.
- **Everything must stay green in CI with zero secrets** — scaffolded CI runs
  install → lint → test → build with no env vars.

## Packaging conventions

- Layout: `skills/<kebab-name>/SKILL.md` + `resources/*.md`. Mirror the
  existing skills section-for-section: frontmatter with a single `description`
  key ("what + use when" trigger phrasing); orientation sentence; "Before you
  start — ask the user"; "Resources — read these in order"; numbered "Build
  order" where **every step ends with a verification gate**; "What the
  finished repo looks like".
- **Verbatim embeds carry their ledgers.** Where fidelity matters, embed the
  validated code in full — comments included, since gotchas often live only
  in comments — and follow it with a "why each weird bit exists" ledger so a
  future edit doesn't simplify away a live-fire fix. Mark the few adapt
  points explicitly; everything else is do-not-improvise.
- **Prefer deleting the vendor to abstracting it.** A seam is worth building
  only where a swap has actually happened twice (storage). Where the answer is
  ~200 lines of standard library, own the code instead — an adapter for a
  dependency you can delete is pure cost.
- The plugin manifest lives in `.claude-plugin/` (`plugin.json` +
  `marketplace.json`). New skills are auto-discovered from `skills/` — no
  manifest changes needed to add one.

## Structure

- `skills/` — the executable form; what the plugin serves.
- `parts/` + `recipes/` — cross-cutting knowledge that no single skill owns,
  as human-readable instruction docs. When a skill and a part cover the same
  ground, the skill is the source of truth for exact code; keep the part at
  instruction level.
- `dev-skills.md` — pointers to external skills worth installing; not part of
  this plugin.
