# Giving a Personal Tool an Agent Surface — Big Ideas & Exploration Brief

> **Provenance.** Written during the gimme-image era and rescued from that repo
> before it was archived. The running example is an image-generation app, but
> the frame is general — it applies to any personal tool you want an agent to
> drive. Kept close to as-written.

**Context:** a personal, login-gated image-generation app (FAL + Vercel AI
Gateway). The goal is to let a coding agent do, on your behalf, everything you
can do in the UI — generate images at specified parameters, get them back, drop
them into the repo, keep coding — without you ever logging in.

**What this document is.** It was written *without access to the repo*, so it's
deliberately high-level: the conceptual frame plus a set of concrete questions
for a repo-aware agent to answer against the actual code. Two audiences: you, and
the coding agent you'll hand this to inside your dev environment. The final
section is addressed to that agent.

---

## 1. The one idea underneath all of it

You already wrote the hard part — the generation pipeline. The web UI is just
*one caller* of it. Adding an agent surface doesn't mean building a second app;
it means giving that same pipeline a **second caller** that speaks the agent's
language.

> **One core, two surfaces.** Extract the generation logic into a single shared
> function. The browser UI calls it. An MCP tool calls it. Neither is the source
> of truth; both are thin surfaces over the same core.

This is the same pattern that shows up everywhere once you look for it: a website
that serves an HTML page *and* a JSON mirror from one render model; a tool that
serves humans *and* agents from one function. Build the core once, wrap it twice.

---

## 2. Split the task — only half of it needs your app

The workflow is *"generate images and put them in the public folder."* Those are
two different actions with two different owners:

- **Generate** — needs your app's pipeline (your FAL + Gateway keys, your
  parameter choices). This is what you expose as a tool.
- **Write files to `/public`** — the coding agent already does this. In a dev
  environment it has shell and filesystem access to the repo. You don't expose
  "write to my repo" as a capability; the agent already has it.

So the tool surface you're building is small: essentially **one verb** —
`generate_image` — that returns image URLs or bytes. The agent handles placing
them. Keeping the surface this narrow is a feature, not a limitation.

---

## 3. Auth is simpler than the big platforms — because you're one user

The heavyweight auth you see on Canva/Figma/Adobe MCP servers (dynamic client
registration, per-user OAuth consent screens) exists to let *strangers* delegate
*their own* accounts. You are not that. You need exactly one secret.

And the deployment shape changes how much auth you need at all:

| Shape | Auth surface | When |
|---|---|---|
| **Local stdio MCP server** | Nearly none — nothing listens on a public port; keys live in the local process env; the agent never sees them. | **Start here.** Best for "I'm coding and want images inline." |
| **Remote MCP server** | A single bearer token you store in the client config and can rotate. | When you want it from multiple machines or clients. |
| **Authenticated API route (no MCP)** | Same single bearer token. | If you'd rather not build a server at all yet — see §5. |

The key insight: a **local stdio server spawned by your own agent** has almost no
attack surface. There's no internet-reachable endpoint to protect, so the whole
"don't let people burn my tokens" problem — which the login screen solves for
humans — mostly evaporates for the agent path. Your real keys stay server-side;
the agent only ever holds a handle to *your tool*, never your FAL key.

---

## 4. Blast-radius thinking (the safety frame)

Don't ask "is this safe?" — ask **"what authority does the agent actually get,
and how bad is the worst case?"**

- The agent's entire reach here is *"can spend my FAL credits generating images."*
  Bounded, cheap, non-destructive. Worst case is wasted credits, not damage.
- Keys stay in the server's environment and are never returned to or seen by the
  agent.
- Everything is revocable: rotate the token, or simply don't spawn the local
  server.

Contrast this with, say, a Cloudflare MCP ("mutate my production infra") — same
protocol, vastly larger blast radius. The danger was never the mechanism; it's
the *scope of what you delegate*. Here, keep the scope tiny on purpose.

---

## 5. The spectrum of how much to build

You don't have to jump straight to a full MCP server. Pick by appetite:

1. **Authenticated API route + bearer token.** You may already be ~90% here — the
   app has a generate route; add a token-checked entry point. The agent calls it
   with `fetch`/`curl`, gets URLs, downloads to `/public`. ~20 lines, no new
   server. This is the lightest possible thing that works.
2. **Local stdio MCP server.** Wrap the same core as a proper tool with a schema,
   discoverable and callable by name across agent clients. More ceremony, better
   ergonomics, reusable. **This is the recommended target** if the goal is to gain
   real MCP experience — it's the smallest build that's genuinely an MCP.
3. **Remote MCP server.** Deploy alongside the app for multi-machine access. Only
   here does the bearer token become load-bearing.

Be honest with yourself about *why* you're picking: option 1 is the pragmatic
minimum; option 2 is the learning goal. Both reach the same end result.

---

## 6. Reference architecture

```
                    ┌──────────────────────────┐
                    │   generate()  (the core) │  ← FAL + Vercel AI Gateway,
                    │   prompt, aspect, count, │    keys in env, one place
                    │   model, seed → images   │
                    └────────────┬─────────────┘
                     ┌───────────┴───────────┐
             ┌───────▼────────┐      ┌────────▼─────────┐
             │  Web UI route  │      │  MCP tool        │
             │  (you, in a    │      │  generate_image  │
             │   browser)     │      │  (the agent)     │
             └────────────────┘      └────────┬─────────┘
                                              │ returns URLs / bytes + metadata
                                     ┌────────▼─────────┐
                                     │  Coding agent    │
                                     │  writes /public  │
                                     └──────────────────┘
```

**Tool shape (starting point):**
`generate_image(prompt, aspect_ratio, count=1, model?, seed?)` →
`{ images: [{ url | b64, width, height }], meta: { model, seed, params } }`

Mirror the UI's real parameter set in the schema — whatever aspect ratios,
models, and counts the UI already offers.

---

## 7. Design principles worth carrying in

- **Return structured, legible results.** Echo back what actually ran (resolved
  model, seed, dimensions), so the agent can reason about the output and log
  provenance — not just a bare image blob. A little metadata makes generations
  reproducible and lets the agent write a `manifest.json` alongside the files.
- **Respect that generation costs money.** Consider a param that caps count, or a
  cheap "estimate/echo params without generating" mode, so an agent loop can't
  quietly burn credits. Cost-awareness is the analog of rate-limiting here.
- **Good tool descriptions are the interface.** The agent picks tools from their
  names and descriptions. Write them like docs, with the when-to-use and an
  example — the same care you'd put into a `SKILL.md`.
- **Keep it stateless where you can.** The generation call itself needn't persist
  anything on the server; the agent owns where results land (the repo). Persistence
  only enters if you later want shareable history — and that's when auth/ownership
  questions return.

---

## 8. Bigger possibilities (grounded, not sci-fi)

Once the basic loop works, the interesting moves are about the agent bringing
*context your UI can't*:

- **Batch & variation in one call** — N aspect ratios or N seeds per request, so
  the agent can generate a full set (hero, social, thumbnail) from one intent.
- **Repo-aware art direction** — the agent reads your actual content (a blog post,
  a product page) and generates matching imagery, then places and references it.
  This is the thing a plain UI structurally can't do: it doesn't know the moment
  or the surrounding material. The agent does.
- **Provenance logging** — every generation writes its prompt/model/seed to a
  manifest, so any image in `/public` is reproducible and re-generatable later.
- **Preview-before-commit** — the tool hands back a URL to view a result before
  the agent commits it to `/public`, keeping you in the loop as the taste
  authority without slowing the flow.
- **Grow into a personal creative MCP** — the same server later exposes more
  verbs (resize, palette, variations) as discoverable tools. One tool today,
  a personal creative toolkit over time.

A recurring line worth keeping in mind while choosing among these: **features that
let the agent *author* more (richer params, batch, context-aware prompts) keep you
as the one judging the result. Features that let the agent *judge* for itself
(auto-selecting the "best" output) quietly hand it your taste.** Both are valid;
just choose deliberately which you're building.

---

## 9. For the repo-aware agent: what to investigate and advise on

*You have the code; this document doesn't. Please read the repo and answer these,
then propose a concrete plan.*

1. **Where does generation live today?** Is it a clean function, or is the FAL /
   Gateway logic tangled inside a route handler or a React component? Identify the
   refactor needed to extract a single shared `generate()` core.
2. **How are the FAL and Vercel AI Gateway keys loaded?** Env vars? Can they live
   in a local Node process's environment so an MCP server can reuse them without
   exposure?
3. **Runtime gotcha:** is the generate path bound to the Vercel *edge* runtime? If
   so, it may not be importable into a plain Node stdio MCP process as-is. Check
   this early — it decides whether option 5.2 is a clean import or needs an HTTP
   hop to the deployed route.
4. **What params does the UI actually expose?** Aspect ratios, models, count,
   seed, anything else — the tool's input schema should mirror them. Reuse any
   existing zod/TypeScript schema rather than re-declaring.
5. **What does the generate call return** — a URL, base64, a stream? This shapes
   the tool's return type and how the agent downloads to `/public`.
6. **How is the login gating implemented?** So the agent path's token gate is
   consistent with it (or can reuse the same check).
7. **Recommend a shape** (§5: API route vs local stdio MCP vs remote) *for this
   specific codebase*, with reasoning grounded in what you found.
8. **Propose concretely:** the extract-shared-core refactor, the tool schema, the
   file-drop convention for `/public` (and whether to write a provenance
   manifest), and a minimal first milestone.

**Suggested first milestone:** a local stdio MCP server exposing one
`generate_image` tool that calls the extracted `generate()` core, with the agent
writing results into `/public`. Prove the full loop end-to-end before adding
anything.

---

## Appendix — the exploration this came out of

Concepts established along the way, preserved so they're not lost:

- **Agent-ready keyless site** — a URL grammar an agent can drive with no key,
  where malformed input degrades gracefully and every response says what it did
  (`ignored` / `notes`). Guessing is safe because nothing fails hard.
- **The persistence line** — *rich input ↔ shareable output ↔ no persistence: pick
  two.* Push everything you can onto the stateless side; **auth, persistence, and
  ownership travel together**, and moderation/identity problems only appear once
  you cross into persistence.
- **Rich-input, still stateless** — a structured POST body in, a canonical
  shareable URL back: authoring power without either auth or a database.
- **Semantic over mechanical levers** — intent params (`audience`, `context`,
  `mood`) expand what an agent can *mean* while keeping craft encoded in the tool;
  mechanical params (`letter-spacing`) do the opposite.
- **Constructor vs designer** — the load-bearing distinction. Levers that add
  authoring range keep the human as taste authority; levers that add
  self-evaluation (the agent judging its own output) make the agent the designer.
  Different products; choose on purpose.
- **MCP auth is delegated human authority, not agent identity** — nothing on the
  wire proves "this is really the trusted agent"; instead a verified account
  holder delegates their *own* scoped, revocable authority. Safety = the scope of
  what you granted, not the protocol.
- **One core, two surfaces** — the pattern tying it all together: build the
  capability once, wrap it for humans and for agents separately.
