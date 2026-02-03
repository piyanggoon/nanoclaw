# AGENTS.md

Guidelines for AI coding agents working in this repository.

## Project Overview

NanoClaw is a personal WhatsApp assistant. A single Node.js (ESM) process connects to WhatsApp via Baileys, routes messages to Claude Agent SDK running as a subprocess. Each WhatsApp group gets an isolated filesystem and memory (`groups/{name}/CLAUDE.md`).

Two separate TypeScript projects share this repo:
- **Host process** (`src/`, root `tsconfig.json`) -- WhatsApp connection, message routing, IPC, scheduling
- **Agent runner** (`container/agent-runner/`) -- runs as a subprocess, executes Claude Agent SDK, has its own `package.json` and `tsconfig.json`

## Build / Run Commands

```bash
npm run build          # tsc -- compile host process
npm run dev            # tsx src/index.ts -- run with hot reload
npm run start          # node dist/index.js -- run compiled output
npm run typecheck      # tsc --noEmit -- type-check without emitting
npm run auth           # tsx src/whatsapp-auth.ts -- WhatsApp QR auth
```

### Agent Runner (separate project)

```bash
cd container/agent-runner
npm install
npx tsc                # Compile agent runner
```

### Service Management (macOS)

```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

## Tests

There is no test framework configured. No test files exist. To verify changes, use:

```bash
npm run typecheck      # Catches type errors across the host process
```

## Linting / Formatting

No linter or formatter is configured (no eslint, prettier, biome). Follow the existing code style described below.

## Code Style

### Module System

- ESM only (`"type": "module"` in package.json)
- `"module": "NodeNext"` / `"moduleResolution": "NodeNext"` in tsconfig
- **Always use `.js` extension** in relative imports (required by NodeNext):
  ```ts
  import { STORE_DIR } from './config.js';
  import { RegisteredGroup } from './types.js';
  ```

### Imports

- Node built-ins as default imports: `import fs from 'fs'`, `import path from 'path'`
- External packages as default or named: `import pino from 'pino'`, `import Database from 'better-sqlite3'`
- Project modules as named imports: `import { POLL_INTERVAL, STORE_DIR } from './config.js'`
- No import aliases or path mapping

### File Organization

- One module per file, no classes -- purely functional architecture
- All shared interfaces live in `src/types.ts`
- Module-specific interfaces can be defined inline (e.g., `ContainerInput` in `container-runner.ts`)
- Constants and configuration live in `src/config.ts`
- Utility functions in `src/utils.ts`

### Naming Conventions

| What | Convention | Examples |
|------|-----------|----------|
| Files | `kebab-case.ts` | `container-runner.ts`, `task-scheduler.ts` |
| Variables, functions | `camelCase` | `runAgent`, `chatJid`, `getHomeDir` |
| Interfaces, types | `PascalCase` | `RegisteredGroup`, `ContainerOutput`, `NewMessage` |
| Constants | `SCREAMING_SNAKE_CASE` | `POLL_INTERVAL`, `STORE_DIR`, `OUTPUT_START_MARKER` |
| Database columns | `snake_case` | `chat_jid`, `sender_name`, `next_run` |

### Type Patterns

- `strict: true` in all tsconfig files
- Explicit return types on exported functions:
  ```ts
  export function initDatabase(): void { ... }
  export function getNewMessages(chatJid: string, since: string): NewMessage[] { ... }
  ```
- Use `Record<string, T>` for dictionaries
- Use `as` assertions for SQLite query results: `.all() as NewMessage[]`
- Use union string literals for enums: `'success' | 'error'`, `'cron' | 'interval' | 'once'`
- Use `Omit<>`, `Partial<Pick<>>` for type derivation
- Zod for runtime validation in the agent runner MCP server

### Error Handling

- `try/catch` with structured pino logging:
  ```ts
  logger.error({ err, group: group.name }, 'Agent error');
  ```
- Return `null` on failure instead of throwing (e.g., `runAgent` returns `string | null`)
- Safe error extraction: `err instanceof Error ? err.message : String(err)`
- Empty catch blocks with comments explaining why: `catch { /* column already exists */ }`
- Container runner resolves with error status objects, never rejects promises

### Logging

- Use `pino` with `pino-pretty` transport
- Create a module-level logger instance:
  ```ts
  const logger = pino({
    level: process.env.LOG_LEVEL || 'info',
    transport: { target: 'pino-pretty', options: { colorize: true } }
  });
  ```
- Use structured logging with context objects: `logger.info({ group, chatJid }, 'Processing message')`

### State Management

- Module-level `let` for mutable singletons (`let db`, `let sock`)
- Polling loops use recursive `setTimeout`, not `setInterval`
- Filesystem-based IPC via atomic JSON writes (temp file + rename)

### General Patterns

- Prefer `fs.mkdirSync(dir, { recursive: true })` for directory creation
- Use `path.join()` / `path.resolve()` for all path construction
- Use `process.cwd()` for project root, `process.env.HOME` for home directory
- `Promise` constructor wrapping around `child_process.spawn` for async execution
- Environment variables with fallback defaults: `process.env.X || 'default'`
- Parse numeric env vars explicitly: `parseInt(process.env.TIMEOUT || '300000', 10)`

## Architecture Constraints

- **Skills over features**: New capabilities are contributed as `.claude/skills/` markdown instructions, not source code. CI enforces that PRs cannot mix skill additions with source changes.
- **Source PRs**: Only bug fixes, security fixes, and simplifications are accepted as source code changes.
- **Per-group isolation**: Each group has its own filesystem, CLAUDE.md memory, and IPC namespace.

## Key Files Reference

| File | Purpose |
|------|---------|
| `src/index.ts` | Entry point: WhatsApp connection, message routing, IPC handling |
| `src/config.ts` | All constants, paths, env var defaults, trigger pattern |
| `src/container-runner.ts` | Spawns agent subprocess, sets up env, parses output |
| `src/task-scheduler.ts` | Polls for due scheduled tasks, runs them |
| `src/db.ts` | SQLite schema, queries (messages, tasks, chats) |

| `src/types.ts` | All shared TypeScript interfaces |
| `src/utils.ts` | `loadJson` / `saveJson` helpers |
| `container/agent-runner/src/index.ts` | Subprocess entry: reads stdin, runs Claude Agent SDK |
| `container/agent-runner/src/ipc-mcp.ts` | MCP server for IPC tools (send_message, schedule_task) |
| `groups/{name}/CLAUDE.md` | Per-group persistent memory |
| `groups/global/CLAUDE.md` | Shared memory across all groups |
