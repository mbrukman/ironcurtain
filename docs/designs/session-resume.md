# Design: Session Resume for Docker PTY Sessions

**Status:** Proposed
**Date:** 2026-03-07

## 1. Problem Statement

When a Docker PTY session ends — whether from an OAuth token expiry, a crash, user disconnect, or intentional exit — there is no way to resume it. The user must start a fresh session, losing all conversation context. This is particularly painful for long-running sessions where significant work has been done.

Two independent problems prevent session resume:

1. **Claude Code conversation state is lost.** Claude Code stores conversation history in `~/.claude/projects/` inside the container. This directory is not mounted from the host, so when the container exits, conversation data is destroyed. The `--continue` flag passed to Claude Code is effectively a no-op.

2. **Mux has no resume capability.** The terminal multiplexer can only spawn new sessions. There is no way to resume a previously ended session, even though the session directory (sandbox, audit log, interaction logs) persists on disk.

## 2. Current Architecture

### What persists after a session ends

| Artifact | Location | Survives exit? |
|---|---|---|
| Workspace files | `~/.ironcurtain/sessions/{id}/sandbox/` | Yes |
| Audit log | `~/.ironcurtain/sessions/{id}/audit.jsonl` | Yes |
| LLM interactions | `~/.ironcurtain/sessions/{id}/llm-interactions.jsonl` | Yes |
| Session log | `~/.ironcurtain/sessions/{id}/session.log` | Yes |
| Claude Code conversations | `/root/.claude/projects/` (in-container) | **No** |
| PTY registration | `~/.ironcurtain/pty-registry/session-{id}.json` | **No** (deleted on exit) |
| Proxy sockets | `~/.ironcurtain/sessions/{id}/sockets/` | **No** (cleaned up) |

### Existing resume support

`ironcurtain start --resume <sessionId>` exists but only reuses the session directory. It spawns a fresh container — Claude Code inside starts with no conversation memory.

### Container volume mounts (current)

```
Host                                    Container
{sessionDir}/sandbox        →    /workspace           (rw)
{sessionDir}/sockets        →    /run/ironcurtain     (rw, Linux)
{sessionDir}/orientation    →    /etc/ironcurtain     (ro)
```

## 3. Proposed Design

### 3.1 Persist Claude Code conversation state

Add a host-side directory that maps into the container's `~/.claude/`, containing only non-sensitive files.

**New mount:**
```
{sessionDir}/claude-state/  →    /root/.claude/       (rw)
```

**Pre-populated on first session start:**
```
claude-state/
  projects/          # conversation JSONL (populated by Claude Code)
  settings.json      # copy of host ~/.claude/settings.json (if exists)
  .claude.json       # {"hasCompletedOnboarding": true}
```

**Explicitly excluded** (never copied or mounted):
- `.credentials.json` — OAuth tokens (MITM proxy handles auth)
- `session-env/` — may contain env snapshots with secrets
- `statsig/`, `stats-cache.json` — analytics, unnecessary

**Workspace path stability:** Claude Code keys conversations by the project working directory path. Since the container always mounts the workspace at `/workspace`, the conversation key is stable across container restarts for the same session. `--continue` will find the previous conversation.

### 3.2 Session state snapshot on exit

Write a `session-state.json` to the session directory when a PTY session ends, capturing enough metadata to support resume decisions.

```typescript
interface SessionSnapshot {
  sessionId: string;
  status: 'completed' | 'crashed' | 'auth-failure' | 'user-exit';
  exitCode: number | null;
  lastActivity: string;         // ISO timestamp
  turnCount: number;
  workspacePath: string;        // host-side workspace
  agent: string;                // 'claude-code', 'goose', etc.
  label: string;                // tab label from mux
  resumable: boolean;           // true if sandbox exists and has content
}
```

Written to: `~/.ironcurtain/sessions/{id}/session-state.json`

The exit status can be inferred from:
- OAuth refresh failure → `auth-failure`
- Container exit code 0 → `completed`
- Container exit code != 0 → `crashed`
- User-initiated close → `user-exit`

### 3.3 Mux resume support

#### New mux commands

| Command | Behavior |
|---|---|
| `/resume [sessionId]` | Open a picker of resumable sessions, or resume a specific one |
| `/sessions --all` | List both active and resumable sessions |

#### Resume flow

```
/resume
    ↓
Scan ~/.ironcurtain/sessions/*/session-state.json
    ↓
Filter: resumable == true, sort by lastActivity desc
    ↓
Show picker: "session-id | agent | label | last activity | status"
    ↓
User selects session
    ↓
spawnSession(sessionId, workspacePath)  ← reuses existing sessionId
    ↓
PtyBridge spawns: ironcurtain start --pty --resume <sessionId> --agent <agent>
    ↓
New container starts with same sandbox + claude-state mounts
    ↓
Claude Code runs with --continue, finds previous conversation
    ↓
User continues where they left off
```

#### Auto-resume on mux startup (optional)

When `ironcurtain mux` starts, it could scan for sessions that ended with `auth-failure` or `crashed` status and offer to resume them. This could be a config option (`mux.autoResumeOnStart: boolean`).

### 3.4 Changes to `ironcurtain start --pty`

When `--resume <sessionId>` is passed:

1. Validate the session directory exists
2. Validate `session-state.json` exists and `resumable == true`
3. Reuse `{sessionDir}/sandbox/` as workspace
4. Reuse `{sessionDir}/claude-state/` (contains previous conversations)
5. Create fresh `sockets/` and `orientation/`
6. Start new container with all mounts
7. Append to existing `session.log` and `audit.jsonl` (not overwrite)

## 4. Security Considerations

- **No credentials in claude-state/**: The mounted `claude-state/` directory never contains `.credentials.json`. Auth is handled entirely by the MITM proxy's fake-key swap.
- **Session isolation**: Each session has its own `claude-state/` directory. Conversations from one session are not visible to another.
- **Read-write mount**: Claude Code needs write access to `claude-state/projects/` to save conversations. This is the same trust level as the workspace mount.

## 5. Implementation Plan

### Phase 1: Conversation persistence (enables `--continue`)
1. Create `claude-state/` directory in session setup (`docker-infrastructure.ts`)
2. Pre-populate with `settings.json` and `.claude.json`
3. Add bind mount to container launch (`pty-session.ts`)
4. Verify `--continue` picks up previous conversations

### Phase 2: Session snapshots
1. Define `SessionSnapshot` type
2. Write snapshot on PTY session exit (classify exit reason)
3. Add `--resume` validation logic to session startup

### Phase 3: Mux resume
1. Add `/resume` command to `MuxInputHandler`
2. Add session scanner (read `session-state.json` files)
3. Add resumable session picker UI
4. Wire `spawnSession()` to pass `--resume` flag

### Phase 4: Polish (optional)
1. Auto-resume prompt on mux startup
2. Session age-out / cleanup for old `claude-state/` directories
3. `/sessions --all` command showing resumable sessions

## 6. Open Questions

1. **Session expiry**: How long should sessions remain resumable? Should we auto-clean `claude-state/` after N days?
2. **Multiple resumes**: Should a session be resumable more than once? (Probably yes — the snapshot just gets overwritten each time.)
3. **Cross-agent resume**: If a session was started with `claude-code`, can it be resumed with `goose`? (Probably no — conversation format is agent-specific.)
4. **Mux tab restoration**: Should mux save/restore its full tab layout on restart, not just individual sessions?
