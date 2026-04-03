# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Context

This is an **archival/reference repository** containing the leaked `src/` directory of Anthropic's Claude Code CLI (leaked 2026-03-31 via a `.map` file in the npm registry). There is no `package.json`, build system, or test suite — this is source code only. The original project builds with **Bun** and uses **TypeScript strict mode**.

## Tech Stack

- **Runtime**: Bun (not Node.js — uses `bun:bundle` feature flags for dead code elimination)
- **Language**: TypeScript strict
- **Terminal UI**: React 18 + Ink (React for CLI)
- **CLI Parsing**: Commander.js with extra-typings
- **Schema Validation**: Zod v4
- **State Management**: Zustand
- **LLM API**: Anthropic SDK (streaming, tool use)
- **Search**: ripgrep (wrapped by GrepTool)
- **Protocols**: MCP SDK, LSP

## Architecture Overview

### Core Query Loop

`main.tsx` → `entrypoints/init.ts` → `components/App.tsx` → **`QueryEngine.ts`** → tool execution loop → response

`QueryEngine.ts` (~46K lines) is the heart of the system: it manages the Anthropic API streaming calls, tool-call loops, thinking mode, retry logic, token budgets, and context compaction.

### Tool System (`src/tools/`)

Every action Claude can perform is a self-contained Tool module with:
- Zod input schema
- Permission model (user approval gating)
- Execution logic + progress state

Key tools: `BashTool` (shell exec with security parsing), `FileReadTool/WriteEditTool`, `GlobTool/GrepTool`, `AgentTool` (spawns sub-agents), `SkillTool`, `WebFetchTool/WebSearchTool`, `MCPTool`, `EnterWorktreeTool/ExitWorktreeTool`, `CronCreateTool`, `RemoteTriggerTool`.

### Command System (`src/commands/`)

~50 slash commands (e.g., `/commit`, `/review`, `/doctor`, `/compact`, `/mcp`, `/memory`, `/resume`). `commands.ts` (~25K lines) is the registry.

### Permission System (`src/hooks/toolPermission/`)

Every tool invocation is gated here. Modes: `default` (ask user), `plan` (analyze first), `auto`, `bypassPermissions`. Per-tool rules: `alwaysAllow`, `alwaysDeny`, `alwaysAsk`.

### Bridge System (`src/bridge/`)

Bidirectional IDE integration (VS Code, JetBrains) via JSON-RPC-like protocol. `bridgeMain.ts` drives the loop; `jwtUtils.ts` handles auth; `sessionRunner.ts` manages session lifecycle.

### Service Layer (`src/services/`)

| Service | Purpose |
|---|---|
| `api/` | Anthropic SDK client, file API, bootstrap data |
| `mcp/` | Model Context Protocol server management |
| `oauth/` | OAuth 2.0 auth flow |
| `lsp/` | Language Server Protocol manager |
| `analytics/` | GrowthBook feature flags + telemetry |
| `compact/` | Context compression (auto, snip, reactive) |
| `extractMemories/` | Auto-extract memories from conversations |
| `policyLimits/` | Org policy enforcement |

### Multi-Agent / Coordinator (`src/coordinator/`)

`AgentTool` spawns sub-agents with isolated state. `coordinator/coordinatorMode.ts` orchestrates teams. `TeamCreateTool`/`TeamDeleteTool` manage agent teams. Feature-flagged via `COORDINATOR_MODE`.

### Skills & Plugins

- **Skills** (`src/skills/`): Reusable markdown-defined workflows loaded from `.claude/skills/`, home config, or bundled. Executed via `SkillTool`.
- **Plugins** (`src/plugins/`): Third-party extensibility loaded by `src/services/plugins/`.

### Feature Flags

Dead code is eliminated at Bun build time via `feature()` from `bun:bundle`. Notable flags: `PROACTIVE`, `KAIROS` (assistant mode), `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `COORDINATOR_MODE`, `HISTORY_SNIP`, `REACTIVE_COMPACT`.

### Startup Optimization

`main.tsx` fires `startMdmRawRead()` and `startKeychainPrefetch()` as side-effects before heavy imports. Heavy modules (OpenTelemetry ~400KB, gRPC ~700KB) are lazy-loaded via dynamic `import()`.

## Key Files

| File | Role |
|---|---|
| `src/main.tsx` | CLI entrypoint (Commander.js + React/Ink init, parallel prefetch) |
| `src/QueryEngine.ts` | Core LLM engine — streaming, tool loops, retries, token budget |
| `src/Tool.ts` | Base types/interfaces for all tools |
| `src/commands.ts` | Slash command registry |
| `src/tools.ts` | Tool registry |
| `src/context.ts` | System/user context collection (git status, CLAUDE.md loading) |
| `src/components/App.tsx` | Main React + Ink UI orchestrator |
| `src/state/AppState.tsx` | Central Zustand state store |
| `src/query/` | Query pipeline modules (config, deps, transitions, token budget) |
| `src/utils/config.ts` | Settings management |
| `src/utils/claudemd.ts` | CLAUDE.md discovery and loading logic |
