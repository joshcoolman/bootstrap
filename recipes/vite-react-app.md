# Recipe: vite-react-app

One-shot guide to bootstrapping a new public app to the standard starting
point. Composes all four parts in the right order. After following this, the
repo is runnable, deployable to Vercel, and ready for Phase 1 feature work.

**Parts this recipe composes:**
- [shell](../parts/shell.md)
- [styles](../parts/styles.md)
- [docs](../parts/docs.md)
- [knowledge](../parts/knowledge.md)
- [cicd](../parts/cicd.md)

---

## Step 1 — Scaffold (shell)

1. Create the repo directory and initialize git.
2. Set up `package.json` with the stack from the **shell** part. Choose a dev
   port that doesn't conflict with sibling repos.
3. Configure `vite.config.ts` with the TanStack Router plugin, the React
   plugin, and the `#` alias pointing to `src/`.
4. Write `tsconfig.json` with strict mode and `noUncheckedIndexedAccess: true`.
5. Write `tsr.config.json` pointing at `src/routes/` and `src/routeTree.gen.ts`.
6. Write `eslint.config.js`, `prettier.config.js`, `.prettierignore`.
7. Write `pnpm-workspace.yaml` (add `onlyBuiltDependencies` only if native
   addons are required).
8. Create `public/` with `favicon.ico`, `manifest.json` (update app name),
   `robots.txt`, `logo192.png`, `logo512.png`.
9. Write `src/app-meta.ts` with the app's name, tagline, and repo URL.
10. Write `src/routes/__root.tsx` — skeleton only for now, styles and theme
    script come in step 2.
11. Write `src/routes/index.tsx` wiring the `/` route to a `Home` component.
12. Write `src/components/home.tsx` — a minimal placeholder that displays
    `appMeta.name` and a link to `/docs`. It will be replaced by the first
    vertical feature slice.
13. Run `pnpm install` and confirm `pnpm dev` serves the app without errors.

## Step 2 — Style system (styles)

1. Create `src/styles/` and write the four CSS files: `tokens.css`, `base.css`,
   `typography.css`, `index.css`. Use the **styles** part as the spec and
   `outpaint-studio/src/styles/` as the canonical source.
2. Update `src/routes/__root.tsx`:
   - Import the stylesheet: `import appCss from '../styles/index.css?url'`
   - Add the Google Fonts `<link>`s (preconnects + IBM Plex Sans / Martel /
     Space Mono stylesheet)
   - Add the pre-paint theme script (inline `<script>` before `<HeadContent />`)
   - Set `data-theme="light"` and `suppressHydrationWarning` on `<html>`
3. Write `src/components/theme-toggle.tsx`. Render it once in the `<body>` of
   `__root.tsx`.
4. Update the home component to use the design tokens (`bg-bg`, `text-text`,
   `house-section`, etc.) so the style system is visibly exercised.
5. Run `pnpm dev` and confirm: light mode renders correctly, the theme toggle
   switches to dark, the preference persists on reload, no flash of wrong theme.
6. Write `docs/STYLE.md` — the in-repo version of the style contract and port
   recipe (see the **styles** part for the content).

## Step 3 — Docs structure and viewer (docs)

1. Create `docs/` and write the four standard markdown files:
   - `docs/OVERVIEW.md` — umbrella vision
   - `docs/SPEC.md` — what/why for this app, the welded boundary
   - `docs/PLAN.md` — the concrete phased build order
   - `docs/STYLE.md` — already written in step 2
2. Write the `/docs` route: copy `outpaint-studio/src/routes/docs.tsx` verbatim,
   updating only the app name in the sidebar back-link.
3. Add `react-markdown` and `remark-gfm` to `package.json` and install.
4. Add a link to `/docs` from the home component (e.g. "read the plan →").
5. Run `pnpm dev` and navigate to `/docs`. Confirm all four docs files appear
   in the sidebar under "Start here", `README.md` appears, `CLAUDE.md` appears
   under "Working notes".

## Step 4 — Feature seams and knowledge (knowledge)

1. Create `src/features/` with one subfolder per domain concern. For a typical
   BYO-key utility: `<core>`, `generate`, `verify`, `knowledge`, `prefs`.
2. Write a `CLAUDE.md` in each feature folder describing its responsibility
   boundary (see the **knowledge** part for guidance).
3. Write `types.ts` in any feature whose contracts are already clear. At
   minimum, define the core record type and the verifier verdict type.
4. Create the `knowledge/` folder at the repo root. Write the initial knowledge
   files — at least one guidance file and one rubric file appropriate for the
   app's domain. These should be real content, not placeholders, because they
   immediately inform the first feature slice.
5. Confirm the knowledge loader pattern is documented in
   `src/features/knowledge/CLAUDE.md` (even if not yet implemented).

## Step 5 — CI/CD (cicd)

1. Create `.github/workflows/ci.yml` with the quality gate workflow from the
   **cicd** part. Runs lint, test, and build on every push and PR.
2. Connect the repo to Vercel via the dashboard (native Git integration — no
   CLI or secrets needed). Confirm a preview deployment fires on the next push.
3. Install the Claude GitHub App on the repository so `@claude` is available
   in PR comments.

## Step 6 — Polish and verify

1. Write `CLAUDE.md` at the repo root (already done in step 4 — verify it's current). Include: what the app does (the welded
   boundary, one paragraph), current build state ("Phase 0 complete — runnable
   shell"), what to read first (`docs/PLAN.md` then `docs/SPEC.md`), and a
   summary of what's in place.
2. Write `README.md` — the public-facing summary. Stack, boundary, local dev
   commands. Should read like the outpaint-studio README.
3. Run `pnpm build` and confirm no errors.
4. Run `pnpm test` and confirm Vitest exits cleanly (even with no tests yet).
5. Run `pnpm lint` and confirm no errors.
6. Push to GitHub. Confirm Vercel picks it up and deploys successfully.

## What the repo looks like when done

```
your-app/
├── CLAUDE.md                    # agent orientation
├── README.md                    # public summary
├── docs/
│   ├── OVERVIEW.md
│   ├── PLAN.md
│   ├── SPEC.md
│   └── STYLE.md
├── knowledge/
│   ├── <guidance>.md
│   └── <rubric>.md
├── src/
│   ├── app-meta.ts
│   ├── router.tsx
│   ├── routeTree.gen.ts
│   ├── components/
│   │   ├── home.tsx             # placeholder, replaced by feature work
│   │   └── theme-toggle.tsx
│   ├── features/
│   │   ├── <core>/
│   │   │   ├── CLAUDE.md
│   │   │   └── types.ts
│   │   ├── generate/CLAUDE.md
│   │   ├── verify/CLAUDE.md
│   │   ├── knowledge/CLAUDE.md
│   │   └── prefs/CLAUDE.md
│   ├── routes/
│   │   ├── __root.tsx
│   │   ├── index.tsx
│   │   └── docs.tsx
│   └── styles/
│       ├── index.css
│       ├── tokens.css
│       ├── base.css
│       └── typography.css
├── public/
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsr.config.json
├── eslint.config.js
├── prettier.config.js
└── pnpm-workspace.yaml
```

At this point the repo is at "Phase 0 complete". The shell is runnable, the
docs viewer works, the style system is live, and the feature seams are in place.
The next step is Phase 1: the first end-to-end vertical slice.
