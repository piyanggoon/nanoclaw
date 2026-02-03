# NanoClaw Security Model

## Trust Model

| Entity | Trust Level | Rationale |
|--------|-------------|-----------|
| Main group | Trusted | Private self-chat, admin control |
| Non-main groups | Untrusted | Other users may be malicious |
| Agent subprocesses | Semi-trusted | Run on host with limited scope |
| WhatsApp messages | User input | Potential prompt injection |

## Security Boundaries

### 1. Per-Group Isolation

Each group operates in its own context:
- **Separate working directory** - Each group has its own folder under `groups/`
- **Separate IPC namespace** - Each group has its own IPC directory under `data/ipc/{folder}/`
- **Separate session storage** - Each group has isolated Claude sessions at `data/sessions/{group}/.claude/`
- Groups cannot see other groups' conversation history or IPC messages

### 2. IPC Authorization

Messages and task operations are verified against group identity:

| Operation | Main Group | Non-Main Group |
|-----------|------------|----------------|
| Send message to own chat | Yes | Yes |
| Send message to other chats | Yes | No |
| Schedule task for self | Yes | Yes |
| Schedule task for others | Yes | No |
| View all tasks | Yes | Own only |
| Manage other groups | Yes | No |

### 3. Credential Handling

**Available to agents:**
- Claude auth tokens (inherited from host environment)

**NOT available to agents:**
- WhatsApp session (`store/auth/`) - host process only

> **Note:** Anthropic credentials are available in the subprocess environment so that Claude Code can authenticate. The agent itself can discover these credentials via environment inspection. **PRs welcome** if you have ideas for credential isolation.

## Privilege Comparison

| Capability | Main Group | Non-Main Group |
|------------|------------|----------------|
| Project root access | Yes (via WORKSPACE_PROJECT env) | No |
| Group folder | Read-write | Read-write (own only) |
| Global memory | Read-write | Read-only |
| Network access | Unrestricted | Unrestricted |
| MCP tools | All | All |

## Security Architecture Diagram

```
+------------------------------------------------------------------+
|                        UNTRUSTED ZONE                             |
|  WhatsApp Messages (potentially malicious)                        |
+--------------------------------+---------------------------------+
                                 |
                                 v  Trigger check, input escaping
+------------------------------------------------------------------+
|                     HOST PROCESS (TRUSTED)                        |
|  - Message routing                                                |
|  - IPC authorization                                              |
|  - Agent lifecycle                                                |
|  - Credential management                                          |
+--------------------------------+---------------------------------+
                                 |
                                 v  Env vars: paths, credentials
+------------------------------------------------------------------+
|                   AGENT SUBPROCESS (SEMI-TRUSTED)                 |
|  - Agent execution (Claude Agent SDK)                             |
|  - Bash commands (runs on host)                                   |
|  - File operations (scoped to group directory)                    |
|  - Network access (unrestricted)                                  |
+------------------------------------------------------------------+
```
