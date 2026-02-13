# opencode-orchestrator

CLI for managing multiple [OpenCode](https://github.com/sst/opencode) instances from one terminal.

## What this does

If you run OpenCode on several machines or across multiple projects, you end up SSHing around and juggling browser tabs. This tool talks to each instance's HTTP API so you can dispatch work, check status, and approve permissions without leaving your shell.

It ships with a Claude Code skill (`SKILL.md`) so Claude can drive the instances for you.

## Prerequisites

- Python 3.8+
- One or more OpenCode instances running in server mode

Start each instance with:

```bash
opencode serve --port 4096
```

## Setup

1. Clone or copy this repo
2. Edit `instances.json` to point at your OpenCode instances:

```json
{
  "allowed_models": [
    "anthropic/claude-sonnet-4-5",
    "openai/o3"
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

Top-level fields:
- `allowed_models` -- whitelist of `provider/model` strings. The CLI rejects any `--model` value not in this list. If omitted, all model overrides are rejected.

Per-instance fields:
- `url` -- where the OpenCode server is listening
- `projects_path` -- base directory for projects on that machine. Relative `--directory` args resolve against this path.

Optional per-instance fields:
- `password` -- for HTTP basic auth (if your instance is behind auth)
- `username` -- defaults to `opencode` if omitted

3. Make the CLI executable:

```bash
chmod +x oc
```

## Usage

```bash
# Check which instances are reachable
./oc instances

# Discover sessions across all projects
./oc sync

# Create a session on an instance
./oc local create --title "Fix auth bug" --directory myproject --yolo

# Send a prompt
./oc local prompt <session_id> "investigate the login timeout issue"

# Check what's happening
./oc local status

# Read the agent's output
./oc local messages <session_id> --limit 5

# Handle permissions the agent is waiting on
./oc local permissions
./oc local approve <request_id>

# Answer questions the agent asked
./oc local questions
./oc local answer <request_id> "use the v2 API"
```

The `--yolo` flag on `create` auto-approves all permissions. Use it for tasks you trust; skip it when you want to review each action.

## Model override

Pick a model per session or per prompt:

```bash
# Set model at session creation (sticks for the whole session)
./oc local create --model anthropic/claude-sonnet-4-5 --title "Refactor" --directory proj --yolo

# Override on a single prompt
./oc local prompt --model openai/o3 <session_id> "analyze this code"
```

The model whitelist comes from `allowed_models` in `instances.json`. Any `--model` value not in that list gets rejected.

## Using as a Claude Code skill

Copy this repo (or symlink it) into your Claude Code skills directory:

```bash
# symlink into your global skills
ln -s /path/to/opencode-orchestrator ~/.claude/skills/oc-orchestrator
```

Once installed, trigger it from Claude Code with the `/oc-orchestrator` command followed by what you want done:

```
/oc-orchestrator check which instances are up
```

```
/oc-orchestrator create a session on dev-server in the webapp project
and add rate limiting to /api/login -- 5 requests per minute per IP, return 429
```

```
/oc-orchestrator run backend tests on dev-server and frontend tests on local,
both in the myapp project, in parallel
```

```
/oc-orchestrator check status on dev-server and approve any pending permissions
```

```
/oc-orchestrator send a follow-up to session ses_abc123 on local:
refactor the token validation to use the jose library
```

Claude Code reads the skill, figures out which `oc` commands to run, polls until the agents finish, and reports back.

## How it works

OpenCode's server has an HTTP API. Projects are scoped by a directory header (`x-opencode-directory`), so one instance can handle sessions across different repos.

The `oc` CLI keeps a `state.json` cache mapping session IDs to directories. Listing commands (`sessions`, `status`) always query the server fresh. Targeted commands (`prompt`, `messages`) use the cache to set the directory header.

If the cache goes stale, `./oc sync` rebuilds it.

## File overview

| File | What it does |
|------|-------------|
| `oc` | The CLI. Single Python file, no dependencies beyond stdlib. |
| `instances.json` | Your instance configuration. Edit this. |
| `SKILL.md` | Claude Code skill for AI-driven orchestration. |
| `state.json` | Auto-generated session cache. Safe to delete. |

