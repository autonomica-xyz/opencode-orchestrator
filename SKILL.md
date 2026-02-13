---
name: opencode-orchestration
description: Orchestrate multiple OpenCode instances - dispatch tasks, monitor progress, handle permissions. Use when asked to coordinate work across projects or manage OpenCode agents.
---

# OpenCode Orchestration

You orchestrate multiple OpenCode instances via the `oc` CLI at `oc-orchestrator/oc`.

## Quick Reference

```bash
# Discovery & state
~/.claude/skills/oc-orchestrator/oc instances                          # Health check all
~/.claude/skills/oc-orchestrator/oc sync [<inst>]                      # Discover sessions/projects from server
~/.claude/skills/oc-orchestrator/oc <inst> status                      # Active sessions + permissions + questions
~/.claude/skills/oc-orchestrator/oc <inst> models                      # List connected providers & models
~/.claude/skills/oc-orchestrator/oc <inst> sessions [--model]          # List sessions (--model shows active model)

# Session lifecycle
~/.claude/skills/oc-orchestrator/oc <inst> create --title "Fix auth" --directory myproject --yolo
~/.claude/skills/oc-orchestrator/oc <inst> prompt <sid> "do the thing" # Send work (async, verifies delivery)
~/.claude/skills/oc-orchestrator/oc <inst> messages <sid> --limit 5    # Check progress
~/.claude/skills/oc-orchestrator/oc <inst> abort <sid>                 # Cancel if needed
~/.claude/skills/oc-orchestrator/oc <inst> diagnose <sid>              # Check session health

# Model override (provider/model format, e.g. anthropic/claude-sonnet-4-5)
~/.claude/skills/oc-orchestrator/oc <inst> create --model anthropic/claude-sonnet-4-5 --title "Task" --directory proj --yolo
~/.claude/skills/oc-orchestrator/oc <inst> prompt --model openai/o3 <sid> "do the thing"

# Permissions & questions
~/.claude/skills/oc-orchestrator/oc <inst> permissions                 # List pending
~/.claude/skills/oc-orchestrator/oc <inst> approve <rid>               # Approve once
~/.claude/skills/oc-orchestrator/oc <inst> approve <rid> --always      # Approve always
~/.claude/skills/oc-orchestrator/oc <inst> reject <rid>                # Reject
~/.claude/skills/oc-orchestrator/oc <inst> questions                   # List pending questions
~/.claude/skills/oc-orchestrator/oc <inst> answer <rid> "response"     # Answer a question
```

## Configuration

Instance config lives at `oc-orchestrator/instances.json`:

```json
{
  "allowed_models": [
    "openai/gpt-5.3-codex",
    "zai-coding-plan/glm-4.7"
  ],
  "local": {
    "url": "http://localhost:4096",
    "projects_path": "/home/user/projects"
  },
  "dev-server": {
    "url": "http://192.168.1.100:4096",
    "projects_path": "/home/user/dev/projects"
  }
}
```

- **allowed_models**: Whitelist of `provider/model` strings. The CLI rejects any `--model` value not in this list.
- **url**: OpenCode instance URL (started with `opencode serve --port <port>`)
- **projects_path**: Base directory for projects on that machine. `--directory myproject` resolves to `projects_path/myproject`. Absolute paths also work.

Multiple instances on the same machine share session storage — sessions belong to the directory/project, not the instance. Each instance's CWD determines its default project scope.

## Models

Check what models are available on an instance before dispatching work:

```bash
oc <inst> models             # Connected providers only
oc <inst> models --all       # All available providers
oc <inst> sessions --model   # See which model each session is using
```

Override model per session or per prompt using `provider/model` format:

```bash
# Set model at session creation (sticky — used for all prompts in that session)
oc local create --model anthropic/claude-sonnet-4-5 --title "Complex refactor" --directory proj --yolo

# Override model on a specific prompt
oc local prompt --model openai/o3 <sid> "analyze this"
```

Model set via `create --model` is stored in local state and auto-applied on subsequent `prompt` calls to that session. Explicit `prompt --model` overrides it. If no model is specified, the instance's default is used.

The first model used in a session becomes sticky — OpenCode reuses it for subsequent messages via its "last model" fallback.

## State Management

The `oc` CLI maintains `state.json` mapping sessions to directories. This is a **cache**, not the source of truth.

- **Listing commands** (`sessions`, `status`, `permissions`, `questions`) always discover fresh from the server
- **Targeted commands** (`prompt`, `messages`, `abort`) use cached state to resolve the directory header
- `sync` does a full crawl and rebuilds state — run it after manual changes on the server

## Orchestration Patterns

### Dispatch and monitor

1. Create a session: `oc <inst> create --title "descriptive title" --directory myproject --yolo`
2. Send the task: `oc <inst> prompt <session_id> "your instructions here"`
3. Check status periodically: `oc <inst> status`
4. When busy->idle, read results: `oc <inst> messages <session_id> --limit 5`
5. Handle any permissions that come up: `oc <inst> permissions` then `oc <inst> approve <id>`

Use `--yolo` to auto-approve all permissions (sets wildcard allow rules). Only use for trusted tasks.

### New project on remote machine

```bash
oc dev-server create --title "Deploy app" --directory new-project --yolo
```

If `new-project` doesn't exist under `projects_path`, the CLI auto-bootstraps it: creates a temporary session, runs `mkdir -p && git init`, waits for completion, then creates the real session.

### Parallel work across instances

When coordinating across multiple instances (e.g. backend + frontend):

1. Create sessions on each instance
2. Dispatch independent tasks in parallel (multiple `oc prompt` calls)
3. Poll `status` on each instance to track progress
4. Handle permissions as they appear on any instance
5. When all are idle, read messages from each to synthesize results

### Session reuse

Sessions persist. You can send multiple prompts to the same session — the OpenCode agent retains context from previous messages. Use this for iterative work:

1. First prompt: "Analyze the authentication module"
2. Read the response
3. Follow-up prompt: "Now refactor the token validation to use jose library"

### Permission handling

OpenCode agents request permission for potentially destructive operations (file writes, shell commands, etc). When you see pending permissions:

- **approve once**: Safe for one-off operations you've reviewed
- **approve always**: For repeated safe operations (e.g. running tests)
- **reject**: When the agent is going down the wrong path

Always review what the permission is for before approving.

### Status interpretation

`oc <inst> status` shows each session as one of:
- **idle**: Agent completed work and has produced output. Read messages to see results.
- **idle/empty**: Agent is idle but has never produced output — likely a delivery failure or directory mismatch. Use `diagnose` to investigate.
- **busy**: Agent is working. Check back later or check permissions.

### Diagnosing broken sessions

When a session goes idle but produces no output, use `diagnose` to check everything:

```bash
oc <inst> diagnose <session_id>
```

This checks: local state mapping, instance reachability, session existence on server, directory header resolution, session status, message presence, and path normalization.

### Error recovery

If an agent gets stuck or goes off track:
1. `oc <inst> abort <session_id>` to stop it
2. Read recent messages to understand where it went wrong
3. Send a corrective prompt or create a new session

## Key Learnings

- OpenCode API has **no trailing slashes** — `/session` works, `/session/` returns HTML from the proxy fallback.
- All session/permission/question endpoints are **project-scoped** via `x-opencode-directory` header. Without it, the server uses its CWD.
- Multiple instances on the same machine share session storage. Sessions belong to directories, not instances.
- Model override is **per-message**, not per-session. But the first model used becomes sticky via OpenCode's "last model" fallback.
- New directories must exist on the remote machine before creating sessions in them. The `create` command handles this automatically via bootstrapping.

## Tips

- Keep prompts specific and scoped. "Fix the login bug where email validation fails on unicode characters" beats "fix login".
- For large tasks, break them into steps and send them as sequential prompts to the same session.
- Check permissions promptly — agents block waiting for approval.
- When coordinating cross-project changes (e.g. API contract changes), do the provider side first, verify, then the consumer side.
- Use `oc <inst> models` to check available models before choosing one — different instances may have different providers configured.
- Run `oc sync` after adding new instances or when state seems stale.
