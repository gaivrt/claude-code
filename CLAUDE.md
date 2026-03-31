# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Claude Code CLI — Anthropic's official interactive CLI for Claude. A TypeScript/React terminal application using Ink for UI rendering. Built with **Bun** as runtime and bundler.

## Build & Tooling

- **Runtime/Bundler**: Bun — imports use `bun:bundle` for feature flags
- **Linter**: Biome (with legacy ESLint custom rules: `no-process-env-top-level`, `no-top-level-side-effects`, `safe-env-boolean-check`)
- **Validation**: Zod v4 (`zod/v4`)
- **CLI Framework**: `@commander-js/extra-typings`
- **Version**: `MACRO.VERSION` is inlined at build time

## Architecture

### Entry Points (`src/entrypoints/`)

- `cli.tsx` — Bootstrap entrypoint. Fast-paths for `--version`, `--dump-system-prompt`, daemon workers, MCP servers. All imports are dynamic to minimize module evaluation.
- `init.ts` — Initialization logic
- `mcp.ts` — MCP server mode
- `agentSdkTypes.ts` — Agent SDK type definitions

`src/main.tsx` is the core orchestration file (~800KB) — coordinates command parsing, context loading, API init, tool registration, and the React/Ink render loop.

### Commands (`src/commands/`, registry: `src/commands.ts`)

~88 commands. Each command is either:
- A directory `src/commands/{name}/index.ts` exporting metadata + lazy-loaded main file
- A single file `src/commands/{name}.js`

Feature-gated commands use `require()` with `feature()` checks for dead code elimination.

### Tools (`src/tools/`, registry: `src/tools.ts`)

44+ tools. Each in `src/tools/{ToolName}/{ToolName}.tsx` with supporting files (UI, permissions, security). Key tools: `BashTool`, `FileEditTool`, `FileReadTool`, `FileWriteTool`, `GlobTool`, `GrepTool`, `AgentTool`, `WebFetchTool`, `WebSearchTool`, `LSPTool`, `SkillTool`, `MCPTool`.

Base types in `src/Tool.ts`. Tools use Zod schemas for input validation.

### Services (`src/services/`)

22+ service modules: `api/`, `analytics/`, `mcp/`, `oauth/`, `plugins/`, `policyLimits/`, `remoteManagedSettings/`, `SessionMemory/`.

### State & Context

- `src/state/AppState.tsx` — Central app state
- `src/context.ts` — System/user context management
- `src/history.ts` — Conversation history
- `src/query.ts` / `src/QueryEngine.ts` — Query processing

### UI (`src/components/`, `src/ink/`)

React components rendered via a custom Ink-based terminal framework (`src/ink/`). 33+ component directories. Hooks in `src/hooks/` (80+).

## Key Conventions

### Feature Flags

```typescript
import { feature } from 'bun:bundle'
if (feature('PROACTIVE')) { ... }  // Dead code eliminated in builds without this flag
```

Feature-gated code uses `require()` (not `import`) so Bun can DCE it. Common flags: `PROACTIVE`, `KAIROS`, `AGENT_TRIGGERS`, `BRIDGE_MODE`, `VOICE_MODE`, `DAEMON`, `CCR_REMOTE_SETUP`.

### Internal vs External

```typescript
process.env.USER_TYPE === 'ant'  // Anthropic-internal features
```

Ant-only tools/commands use conditional `require()` gated on `USER_TYPE`.

### Environment Variables

Access env vars through helpers (e.g., `isEnvTruthy()`), not raw `process.env` at module top level — the `no-process-env-top-level` lint rule enforces this. Key vars: `CLAUDE_CODE_REMOTE`, `CLAUDE_CODE_BRIEF`, `CLAUDE_CODE_SIMPLE`, `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`.

### Import Style

- Dynamic `import()` for lazy loading (especially in entrypoints)
- `require()` only for feature-gated dead code elimination paths
- `// biome-ignore-all assist/source/organizeImports` on files with `ANT-ONLY` import markers that must not be reordered

### Adding a New Tool

1. Create `src/tools/{ToolName}/{ToolName}.tsx`
2. Define input schema with Zod
3. Register in `src/tools.ts`
4. Follow existing patterns (e.g., `BashTool`, `FileEditTool`)

### Adding a New Command

1. Create `src/commands/{name}/index.ts` with metadata
2. Implement in a companion file
3. Register in `src/commands.ts`
