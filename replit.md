# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

## Artifacts

### `artifacts/point-buy-calculator` — RPG Calculator Platform
Full-featured RPG Calculator Factory Platform — D&D 5e / Pathfinder 2e toolkit with SEO build engine.

**Routes:**
- `/` → Directory (platform landing, tool cards, search/filter, build promo)
- `/tools/point-buy-calculator` → Full point buy calculator
- `/tools/:slug` → Individual tool pages (8 tools total)
- `/builds` → Build Library (221+ preset builds, search, tag filters)
- `/builds/:slug` → Individual SEO build pages (221+ pages — auto-generated from dataset)

**Calculator Tools (`/tools/`):**
- Point Buy Calculator — D&D 5e / Pathfinder 2e / Custom, radar+bar charts, optimizer, saves, URL sharing
- Ability Modifier Calculator — Interactive score input + full 1-30 reference table
- Dice Probability Calculator — d4–d20, Normal/Advantage/Disadvantage, distribution chart
- Spell Slot Planner — All 5e spellcasting classes, levels 1-20, neon slot indicators
- 4d6 Stat Roller — Roll 4d6 drop lowest, animated results, roll history
- Encounter Difficulty, Carrying Capacity, Proficiency Bonus (coming soon stubs)

**Build SEO Engine (`/builds/`):**
- 22 preset builds: Barbarian, Wizard, Rogue, Paladin, Cleric, Fighter, Ranger, Monk, Sorcerer, Warlock, Bard, Druid, Artificer, subclasses
- Each build page: radar chart, stats table, playstyle, strengths/weaknesses, FAQ accordions, related builds, "Try This Build" CTA
- "Try This Build in Calculator" links use URL params: `?build=STR15-DEX14-...&system=dnd5e&name=...`
- getBuildUrl() / getRelatedBuilds() utility functions in builds.ts

**Platform Features:**
- Sticky NavBar: Logo | Directory | Point Buy | Builds | All Tools button
- Active route highlighting in NavBar
- Breadcrumb navigation on all inner pages
- Global search + tag filter on both tools directory and builds gallery
- framer-motion animations: whileInView, whileHover, whileTap throughout
- SEO content blocks on every page (FAQs, how-it-works, strategy tips)
- 100% client-side, no backend, no API

**Key data files:**
- `src/data/calculators.ts` — TOOLS registry (8 tools)
- `src/data/builds.ts` — BUILDS + BUILD_MAP (221+ builds total), getBuildUrl, getRelatedBuilds; merges 22 hand-crafted builds with builds.json dataset
- `src/data/builds.json` — 200-entry auto-generated build dataset (scalable to 500/1000/5000+); covers all 12 classes × multiple roles; every stat array validated to point-buy rules (all costs ≤27, scores 8–15, zero duplicates)
- `src/lib/point-buy.ts` — SYSTEMS, getCostForSystem, PRESETS, generateRandomValidBuild

**CRITICAL:** Never use `@/components/ui/select` — causes React duplicate context crash. Use native `<select>` with inline styles instead.

**Stack:** React + Vite + wouter, Framer Motion, Three.js, Canvas API, Tailwind CSS
**Theme:** Dark space — deep navy/black backgrounds, neon cyan (#00ffff) + purple (#aa00ff) accents, glassmorphism `.glass-panel` cards, `.text-gradient` / `.text-glow` utilities

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   └── api-server/         # Express API server
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts (single workspace package)
│   └── src/                # Individual .ts scripts, run via `pnpm --filter @workspace/scripts run <script>`
├── pnpm-workspace.yaml     # pnpm workspace (artifacts/*, lib/*, lib/integrations/*, scripts)
├── tsconfig.base.json      # Shared TS options (composite, bundler resolution, es2022)
├── tsconfig.json           # Root TS project references
└── package.json            # Root package with hoisted devDeps
```

## TypeScript & Composite Projects

Every package extends `tsconfig.base.json` which sets `composite: true`. The root `tsconfig.json` lists all packages as project references. This means:

- **Always typecheck from the root** — run `pnpm run typecheck` (which runs `tsc --build --emitDeclarationOnly`). This builds the full dependency graph so that cross-package imports resolve correctly. Running `tsc` inside a single package will fail if its dependencies haven't been built yet.
- **`emitDeclarationOnly`** — we only emit `.d.ts` files during typecheck; actual JS bundling is handled by esbuild/tsx/vite...etc, not `tsc`.
- **Project references** — when package A depends on package B, A's `tsconfig.json` must list B in its `references` array. `tsc --build` uses this to determine build order and skip up-to-date packages.

## Root Scripts

- `pnpm run build` — runs `typecheck` first, then recursively runs `build` in all packages that define it
- `pnpm run typecheck` — runs `tsc --build --emitDeclarationOnly` using project references

## Packages

### `artifacts/api-server` (`@workspace/api-server`)

Express 5 API server. Routes live in `src/routes/` and use `@workspace/api-zod` for request and response validation and `@workspace/db` for persistence.

- Entry: `src/index.ts` — reads `PORT`, starts Express
- App setup: `src/app.ts` — mounts CORS, JSON/urlencoded parsing, routes at `/api`
- Routes: `src/routes/index.ts` mounts sub-routers; `src/routes/health.ts` exposes `GET /health` (full path: `/api/health`)
- Depends on: `@workspace/db`, `@workspace/api-zod`
- `pnpm --filter @workspace/api-server run dev` — run the dev server
- `pnpm --filter @workspace/api-server run build` — production esbuild bundle (`dist/index.cjs`)
- Build bundles an allowlist of deps (express, cors, pg, drizzle-orm, zod, etc.) and externalizes the rest

### `lib/db` (`@workspace/db`)

Database layer using Drizzle ORM with PostgreSQL. Exports a Drizzle client instance and schema models.

- `src/index.ts` — creates a `Pool` + Drizzle instance, exports schema
- `src/schema/index.ts` — barrel re-export of all models
- `src/schema/<modelname>.ts` — table definitions with `drizzle-zod` insert schemas (no models definitions exist right now)
- `drizzle.config.ts` — Drizzle Kit config (requires `DATABASE_URL`, automatically provided by Replit)
- Exports: `.` (pool, db, schema), `./schema` (schema only)

Production migrations are handled by Replit when publishing. In development, we just use `pnpm --filter @workspace/db run push`, and we fallback to `pnpm --filter @workspace/db run push-force`.

### `lib/api-spec` (`@workspace/api-spec`)

Owns the OpenAPI 3.1 spec (`openapi.yaml`) and the Orval config (`orval.config.ts`). Running codegen produces output into two sibling packages:

1. `lib/api-client-react/src/generated/` — React Query hooks + fetch client
2. `lib/api-zod/src/generated/` — Zod schemas

Run codegen: `pnpm --filter @workspace/api-spec run codegen`

### `lib/api-zod` (`@workspace/api-zod`)

Generated Zod schemas from the OpenAPI spec (e.g. `HealthCheckResponse`). Used by `api-server` for response validation.

### `lib/api-client-react` (`@workspace/api-client-react`)

Generated React Query hooks and fetch client from the OpenAPI spec (e.g. `useHealthCheck`, `healthCheck`).

### `scripts` (`@workspace/scripts`)

Utility scripts package. Each script is a `.ts` file in `src/` with a corresponding npm script in `package.json`. Run scripts via `pnpm --filter @workspace/scripts run <script>`. Scripts can import any workspace package (e.g., `@workspace/db`) by adding it as a dependency in `scripts/package.json`.
