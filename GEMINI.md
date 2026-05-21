# Open Design (OD) - Project Context

Open Design is an open-source, agent-native, local-first design workspace. It serves as a transparent alternative to proprietary design tools, delegating tasks to existing coding agent CLIs on the user's system to generate design artifacts.

## 🏗️ Architecture & Technology Stack

- **Monorepo:** Managed with `pnpm` workspaces.
- **Frontend (`apps/web`):** Next.js 16 (App Router), React 18, TypeScript.
- **Daemon (`apps/daemon`):** Node.js 24 (ESM), Express, SQLite (`better-sqlite3`). Handles agent spawning, file management, and project state.
- **Desktop (`apps/desktop`):** Electron shell wrapping the web app.
- **Agent Bridge:** Spawns local CLIs (Claude Code, Cursor, Gemini, etc.) and streams output back to the UI.
- **Storage:** Local `.od/` directory contains `app.sqlite` and per-project artifact folders.

## 🚀 Core Workflows & Commands

All development lifecycle tasks must go through the `tools-dev` control plane.

| Task | Command |
| --- | --- |
| **Run (Foreground)** | `pnpm tools-dev run web` (starts daemon + web) |
| **Start (Background)** | `pnpm tools-dev start web` |
| **Stop All** | `pnpm tools-dev stop` |
| **Check Status** | `pnpm tools-dev status` |
| **View Logs** | `pnpm tools-dev logs` |
| **Type Check** | `pnpm typecheck` |
| **Style/Policy Guard** | `pnpm guard` |
| **Desktop Status** | `pnpm tools-dev inspect desktop status` |

### Environment Requirements
- **Node.js:** `~24`
- **pnpm:** `~10.33.x` (Use `corepack enable` to pin the version).

## 📁 Directory Structure

- `apps/daemon/`: The backend server and `od` CLI source.
- `apps/web/`: The Next.js frontend application.
- `apps/desktop/`: Electron shell source.
- `packages/contracts/`: Shared TypeScript types and API contracts (Pure TS, no Node/Browser APIs).
- `skills/`: Catalog of `SKILL.md` bundles defining design capabilities.
- `design-systems/`: Catalog of `DESIGN.md` files defining brand visual languages.
- `tools/dev/`: The development lifecycle control plane.
- `e2e/`: Playwright end-to-end tests.
- `.od/`: (Gitignored) Local runtime data, projects, and database.

## 🛠️ Development Conventions

### Coding Standards
- **Quotes:** Use **single quotes** for strings in JS/TS.
- **Comments:** Must be in **English**. Avoid redundant narration; focus on non-obvious intent.
- **Types:**
  - `apps/web/`: Strict TypeScript.
  - `apps/daemon/`: Plain ESM JavaScript with JSDoc for types.
- **Boundaries:** `apps/web` must **never** import from `apps/daemon/src`. Use `packages/contracts` for shared logic.

### Testing & Validation
- **Red Spec First:** For bug fixes, always start with a failing test case.
- **Cheapest Layer:** Use the lightest test layer possible (Unit -> Integration -> E2E).
- **Validation:** Run `pnpm guard` and `pnpm typecheck` before pushing.
- **No Root Aliases:** Do not use `pnpm dev` or `pnpm test` at the root; use package-scoped commands or `tools-dev`.

## 📖 Domain Language

- **Project:** A workspace containing conversations and design files.
- **Normal Artifact:** A generated design output (e.g., HTML/CSS) with a manifest.
- **Artifact Entry File:** The primary file (e.g., `index.html`) of an artifact.
- **Discovery Form:** The interactive questionnaire presented at the start of a design task.
- **Active Project:** The project currently open in the UI, used as a default for MCP tools.
- **Chip Rail:** The UI row for selecting composer surfaces (Image, Video, Prototype, etc.).

## 🤖 Agent Adapter Integration

Open Design detects agents on the system `PATH`. New adapters are added to `apps/daemon/src/agents.ts`.
- **Stream Formats:** Supports `claude-stream-json`, `acp-json-rpc`, `json-event-stream`, and `plain`.
- **Stdin Handling:** Long prompts are fed via `stdin` to avoid OS command length limits.

---
*This file serves as the primary instructional context for Gemini CLI when operating within this repository.*
