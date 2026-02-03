---
name: debug
description: Debug agent issues. Use when things aren't working, agent subprocess fails, authentication problems, or to understand how the system works. Covers logs, environment variables, and common issues.
---

# NanoClaw Agent Debugging

This guide covers debugging the agent execution system.

## Architecture Overview

```
Host (Node.js)                        Agent Subprocess
─────────────────────────────────────────────────────────────
src/container-runner.ts               container/agent-runner/
    │                                      │
    │ spawns node subprocess               │ runs Claude Agent SDK
    │ with env vars for paths              │ with MCP servers
    │                                      │
    ├── WORKSPACE_GROUP ──────────> groups/{folder}/
    ├── WORKSPACE_GLOBAL ─────────> groups/global/
    ├── WORKSPACE_IPC ────────────> data/ipc/{folder}/
    └── CLAUDE_CONFIG_DIR ────────> data/sessions/{folder}/.claude/
```

## Log Locations

| Log | Location | Content |
|-----|----------|---------|
| **Main app logs** | `logs/nanoclaw.log` | Host-side WhatsApp, routing, agent spawning |
| **Main app errors** | `logs/nanoclaw.error.log` | Host-side errors |
| **Agent run logs** | `groups/{folder}/logs/agent-*.log` | Per-run: input, stderr, stdout |
| **Claude sessions** | `data/sessions/{folder}/.claude/` | Per-group Claude session data |

## Enabling Debug Logging

Set `LOG_LEVEL=debug` for verbose output:

```bash
# For development
LOG_LEVEL=debug npm run dev

# For launchd service, add to plist EnvironmentVariables:
<key>LOG_LEVEL</key>
<string>debug</string>
```

Debug level shows:
- Environment variable configuration
- Real-time agent stderr
- Full agent output

## Common Issues

### 1. "Claude Code process exited with code 1"

**Check the agent log file** in `groups/{folder}/logs/agent-*.log`

Common causes:

#### Missing Authentication
```
Invalid API key · Please run /login
```
**Fix:** Ensure `.env` file exists with either OAuth token or API key:
```bash
cat .env  # Should show one of:
# CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-...  (subscription)
# ANTHROPIC_API_KEY=sk-ant-api03-...        (pay-per-use)
```

### 2. Agent Runner Not Found

```
Agent spawn error: spawn node ENOENT
```
or agent log shows missing file.

**Fix:** Build the agent runner:
```bash
cd container/agent-runner
npm install
npx tsc
ls dist/index.js  # Should exist
```

### 3. Session Not Resuming

If sessions aren't being resumed (new session ID every time):

**Check session directory exists:**
```bash
ls -la data/sessions/{groupFolder}/.claude/
```

**Verify sessions in logs:**
```bash
grep "Session initialized" logs/nanoclaw.log | tail -5
# Should show the SAME session ID for consecutive messages in the same group
```

### 4. MCP Server Failures

If an MCP server fails to start, the agent may exit. Check the agent logs for MCP initialization errors.

### 5. Environment Variables Not Reaching Agent

The host passes paths via environment variables. To debug:

```bash
LOG_LEVEL=debug npm run dev
# Look for "Spawning agent subprocess" entries with env details
```

Key env vars set by the host:
- `WORKSPACE_GROUP` - Group's working directory
- `WORKSPACE_GLOBAL` - Global memory directory
- `WORKSPACE_IPC` - IPC directory for this group
- `CLAUDE_CONFIG_DIR` - Claude session storage

## Manual Agent Testing

### Test the full agent flow:
```bash
echo '{"prompt":"What is 2+2?","groupFolder":"test","chatJid":"test@g.us","isMain":false}' | \
  WORKSPACE_GROUP=$(pwd)/groups/test \
  WORKSPACE_GLOBAL=$(pwd)/groups/global \
  WORKSPACE_IPC=$(pwd)/data/ipc/test \
  CLAUDE_CONFIG_DIR=$(pwd)/data/sessions/test/.claude \
  node container/agent-runner/dist/index.js
```

## SDK Options Reference

The agent-runner uses these Claude Agent SDK options:

```typescript
query({
  prompt: input.prompt,
  options: {
    cwd: process.env.WORKSPACE_GROUP,
    allowedTools: ['Bash', 'Read', 'Write', ...],
    permissionMode: 'bypassPermissions',
    allowDangerouslySkipPermissions: true,
    settingSources: ['project'],
    mcpServers: { ... }
  }
})
```

**Important:** `allowDangerouslySkipPermissions: true` is required when using `permissionMode: 'bypassPermissions'`. Without it, Claude Code exits with code 1.

## Rebuilding After Changes

```bash
# Rebuild main app
npm run build

# Rebuild agent runner
cd container/agent-runner && npx tsc
```

## Session Persistence

Claude sessions are stored per-group in `data/sessions/{group}/.claude/` for isolation. Each group has its own session directory, preventing cross-group access to conversation history.

To clear sessions:

```bash
# Clear all sessions for all groups
rm -rf data/sessions/

# Clear sessions for a specific group
rm -rf data/sessions/{groupFolder}/.claude/

# Also clear the session ID from NanoClaw's tracking
echo '{}' > data/sessions.json
```

## IPC Debugging

The agent communicates back to the host via files in the IPC directory:

```bash
# Check pending messages
ls -la data/ipc/{groupFolder}/messages/

# Check pending task operations
ls -la data/ipc/{groupFolder}/tasks/

# Check current tasks snapshot
cat data/ipc/{groupFolder}/current_tasks.json

# Check available groups (main channel only)
cat data/ipc/main/available_groups.json
```

**IPC file types:**
- `messages/*.json` - Agent writes: outgoing WhatsApp messages
- `tasks/*.json` - Agent writes: task operations (schedule, pause, resume, cancel, refresh_groups)
- `current_tasks.json` - Host writes: read-only snapshot of scheduled tasks
- `available_groups.json` - Host writes: read-only list of WhatsApp groups (main only)

## Quick Diagnostic Script

Run this to check common issues:

```bash
echo "=== Checking NanoClaw Setup ==="

echo -e "\n1. Authentication configured?"
[ -f .env ] && (grep -q "CLAUDE_CODE_OAUTH_TOKEN=sk-" .env || grep -q "ANTHROPIC_API_KEY=sk-" .env) && echo "OK" || echo "MISSING - add CLAUDE_CODE_OAUTH_TOKEN or ANTHROPIC_API_KEY to .env"

echo -e "\n2. Agent runner built?"
[ -f container/agent-runner/dist/index.js ] && echo "OK" || echo "MISSING - run: cd container/agent-runner && npm install && npx tsc"

echo -e "\n3. Groups directory?"
ls -la groups/ 2>/dev/null || echo "MISSING - run setup"

echo -e "\n4. Recent agent logs?"
ls -t groups/*/logs/agent-*.log 2>/dev/null | head -3 || echo "No agent logs yet"

echo -e "\n5. Session continuity working?"
SESSIONS=$(grep "Session initialized" logs/nanoclaw.log 2>/dev/null | tail -5 | awk '{print $NF}' | sort -u | wc -l)
[ "$SESSIONS" -le 2 ] && echo "OK (recent sessions reusing IDs)" || echo "CHECK - multiple different session IDs, may indicate resumption issues"

echo -e "\n6. Service running?"
launchctl list 2>/dev/null | grep nanoclaw || echo "Service not loaded"
```
