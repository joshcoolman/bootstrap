# Feature module pattern — framework-agnostic core, auth/validation at the boundary-into-core

Draft written from a conversation describing the target architecture for the
shell app. Not yet reconciled against anything else in this repo — treat as
raw input for the agent that already has full context on the goal.

## The architecture being described

A layered design where the core business logic doesn't know or care what
framework or client is calling it:

```
Frameworks & Drivers   (Express route, Next handler, CLI command, MCP tool, test runner...)
        |
        v
Interface Adapters      (thin — parses the transport-specific request into a
        |                plain call, forwards it, translates the result/error
        |                back into the transport's shape)
        v
Core / Use Cases        (the actual business logic — auth check, input
        |                validation, orchestration — lives here, not in the
        |                adapter)
        v
Entities / Models       (domain types + domain errors — the innermost layer,
                         no dependency on anything above it)
```

The key move: **the first touchpoint (the adapter) is not where enforcement
happens.** Any framework or client can call into the core, so the adapter's
job is only to translate — parse the incoming shape, call the core, translate
the result back out. The actual rule ("no session id → not allowed") is
enforced one layer in, inside the core/use-case logic, so it's:

- enforced identically no matter which framework/client is calling it
- testable directly, with no HTTP server, no router, no mocked request object
- impossible to bypass by adding a new entry point that forgets the check

Concretely: if a session id is missing or invalid, the core throws an
**`UnauthenticatedError`**. That error type is defined at the **entities**
layer (the bottom) — not in the adapter, not in the use case that throws it.
Entities own the vocabulary of what can go wrong; use cases and adapters both
depend on that vocabulary, it doesn't depend on them.

This is a shell app — no real features exist yet. The point of documenting
the pattern now is so that when the first feature gets built, it's built
inside this skeleton instead of improvising a shape per-feature.

## Reaction / opinion

Agreed on the core call: pushing auth + validation into the use-case layer
instead of the adapter is the right default. The two things worth being
deliberate about as this gets codified into "how to build a feature":

**1. Give the use-case layer something to depend on an abstraction of, not a
concrete session store.** If "check session id" reaches directly into (say) a
concrete cookie store or a specific DB client, the use case is no longer
actually framework/vendor-agnostic — it's just relocated, not decoupled. Define
a small port/interface at the use-case layer (e.g. `SessionReader.get(id):
Session | null`) and inject an implementation. The adapter or a composition
root wires the real implementation in; tests wire in a fake. This is the same
shape as a repository/gateway interface — nothing exotic, just don't skip it
because "there's only one feature so far."

**2. One error taxonomy, not one error class per feature.** `UnauthenticatedError`
should be one of a small, fixed set of domain error types defined once at the
entities layer and reused by every feature — e.g. `UnauthenticatedError`,
`UnauthorizedError`, `ValidationError`, `NotFoundError`, `ConflictError`. Each
adapter maps that fixed set to its transport's shape once (HTTP status codes,
CLI exit codes, MCP error payloads, whatever) — one mapping table per adapter
type, not per feature. This is what makes adding a second transport (say, an
MCP server next to the HTTP API) cheap: it only has to write one mapping, not
re-derive error handling per feature.

**3. Contract-test the port, not just the feature.** Whatever port the
use-case layer depends on (session reader, whatever comes next) should have
one shared test suite that runs against every implementation of it — the real
one and the fake. That's the guarantee that the fake used in feature tests
actually behaves like the real thing, not just a convenient stand-in. (Same
idea as testing a repository interface against both a real DB and an in-memory
version.)

**4. Decide now whether errors are thrown or returned, and apply it
everywhere.** Throwing `UnauthenticatedError` from deep inside a use case is
fine, but it means every adapter needs a catch-and-translate step, and every
use-case-to-use-case call needs to know it might throw. The alternative is a
`Result<T, DomainError>` return type threaded through explicitly. Throwing is
less ceremony and reads fine for "this one thing is truly exceptional and
should unwind the stack" (auth is a reasonable fit for that). Returning is
better when a caller needs to branch on *which* error came back without
relying on `instanceof`/catch blocks. Pick one as the default for this shell
and only deviate with a stated reason — don't let it vary feature to feature.

## What the markdown file for "how to build a feature" should cover

Once the above is settled, the feature-authoring doc should walk through, in
order:

1. **Folder shape** — one folder per feature, with entities/types, use cases,
   ports (interfaces), adapters (one per transport/vendor), and a public
   surface file. Mirrors the "seam" idea: consumers only import the public
   surface, never reach into another feature's internals.
2. **Where each piece of new logic goes** — a short decision list: "is this a
   domain type or error? → entities. Is this a rule/orchestration step? → use
   case. Is this transport-specific parsing/formatting? → adapter."
3. **The auth/validation checklist for a new use case** — what every use case
   must do at its own top (check the session via the port, validate input
   shape) before any adapter is written, so the check is never adapter-owned
   by accident.
4. **Testing shape** — use cases are tested directly against a fake
   implementation of their ports, no transport involved. Ports get one
   contract-test suite run against every implementation. Adapters get a thin
   test that only proves translation (right status/shape for each domain
   error), not business logic.
5. **A worked example** — one small feature built end-to-end through every
   layer, including its tests, as the template to copy for the next one.

Not written yet — this file is the input for that conversation, not the
finished doc.
