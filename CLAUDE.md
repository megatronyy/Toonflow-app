# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Toonflow is an AI-powered short drama production tool that converts novels into screenplays, then generates images and videos. It's an Electron desktop app with a Node.js/Express backend. The frontend is a separate repo (Toonflow-web); this repo contains the compiled frontend in `data/web/`.

## Development Commands

```bash
yarn install          # Install dependencies
yarn dev              # Start backend API only (port 10588), no frontend UI
yarn dev:gui          # Start Electron desktop client (full experience)
yarn dev:gui-vite     # Electron + Vite dev mode
yarn start            # Run production build (requires `yarn build` first)
yarn build            # Compile TypeScript via esbuild → data/serve/app.js + build/main.js
yarn lint             # TypeScript type checking (tsc --noEmit)
yarn dist:win         # Build Windows installer
yarn dist:mac         # Build macOS installer
yarn dist:linux       # Build Linux AppImage
```

Node.js >= 23.11.1 required. Yarn is the package manager.

## Architecture

### Entry Points

- **`src/app.ts`** — Express server entry. When not running in Electron, auto-starts on port 10588. When in Electron, the main process calls `startServe()` with optional random port.
- **`scripts/main.ts`** — Electron main process entry.
- **`scripts/build.ts`** — esbuild bundler (not tsc). Produces `data/serve/app.js` (backend) and `build/main.js` (Electron main).

### Auto-Generated Router

`src/router.ts` is **auto-generated** by `src/core.ts`. It scans `src/routes/**/*.ts`, generates imports and `app.use("/api/...")` registrations. A hash in the file header tracks changes — it only rewrites when routes change. **Do not edit `src/router.ts` manually.**

Route file paths map to API paths: `src/routes/login/login.ts` → `/api/login/login`.

### Three-Layer Agent System

Two agents, each with three layers (decision/execution/supervision):

- **ScriptAgent** (`src/agents/scriptAgent/`) — Converts novels into structured scripts (skeleton → adaptation → script)
- **ProductionAgent** (`src/agents/productionAgent/`) — Manages storyboards, assets, and video generation on the infinite canvas

Each layer's prompt is externalized as a Markdown Skill file in `data/skills/`, allowing runtime editing without code changes.

### Agent Memory

`src/utils/agent/memory.ts` implements persistent cross-session memory using:
- Short-term: recent messages
- Long-term: summaries
- RAG: local ONNX vector embeddings via `@huggingface/transformers`

### AI Provider System (Vendors)

Uses Vercel AI SDK (`ai` package) with multiple providers: OpenAI, Anthropic, Google, DeepSeek, 智谱, MiniMax, 通义千问, xAI. Vendors are configurable at runtime through the settings UI — TypeScript vendor logic is stored in the database and executed via `vm2`.

### Real-time Communication

Agent interactions use **Socket.IO** namespaces:
- `/api/socket/scriptAgent`
- `/api/socket/productionAgent`

Defined in `src/socket/index.ts`. Each namespace handler is in `src/socket/routes/`.

### Database

SQLite via Knex.js (`better-sqlite3`). Schema initialization in `src/lib/initDB.ts`. Tables use `o_` prefix (e.g., `o_project`, `o_novel`, `o_script`). The Knex instance is accessed globally via `u.db()`.

### Key Utilities

All centralized in `src/utils/`:
- `db.ts` — Knex database instance
- `getPath.ts` — Resolves runtime data paths (oss, skills, assets, web, models)
- `ai.ts` — AI model invocation helpers
- `oss.ts` — Object storage helpers
- `getConfig.ts` — Runtime configuration from DB

The `u` utility object is imported from `@/utils` and used pervasively.

### Path Aliases

TypeScript path alias `@/*` → `src/*` (configured in `tsconfig.json` and esbuild alias in `scripts/build.ts`).

### Environment

`src/env.ts` sets `NODE_ENV`: `"dev"` for bare Node.js, `"prod"` for Electron. Dev mode triggers router regeneration in `src/app.ts`.

## Key Conventions

- PRs go to `develop` branch, not `master`
- Database tables are prefixed with `o_`
- New API routes: create a `.ts` file in `src/routes/`, restart dev server (auto-regenerates `src/router.ts`)
- Agent prompts: edit Markdown files in `data/skills/` (no code rebuild needed)
- Build uses esbuild (not `tsc`) — `yarn lint` (`tsc --noEmit`) is for type checking only
- Default login: `admin` / `admin123`
