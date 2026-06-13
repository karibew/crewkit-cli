# crewkit

[![npm version](https://img.shields.io/npm/v/@crewkit/cli.svg)](https://www.npmjs.com/package/@crewkit/cli)
[![GitHub release](https://img.shields.io/github/v/release/karibew/crewkit-cli)](https://github.com/karibew/crewkit-cli/releases/latest)
[![Homebrew](https://img.shields.io/badge/homebrew-karibew%2Ftap-orange)](https://github.com/karibew/homebrew-tap)

**Your standards, every session.**

Stop reviewing AI-generated inconsistency. crewkit syncs your team's Claude Code agent configurations, tracks what works, and lets you experiment with improvements.

<!-- TODO: Add demo GIF -->

## Install

```bash
brew install karibew/tap/crewkit
```

Or via npm, Chocolatey (Windows), or curl:

```bash
npm install -g @crewkit/cli
choco install crewkit
curl -fsSL https://crewkit.io/install.sh | sh
```

## Why crewkit?

- **Team consistency** - Everyone uses the same agent configs. No more "works on my machine" for AI prompts.
- **Role-based modes** - Juniors get coaching mode (agents guide, don't code). Seniors get full autonomy. Same agents, different behaviors.
- **A/B test your prompts** - Create experiments, measure outcomes, deploy winners. Data-driven agent improvement.
- **Track what works** - See which configs lead to better code, fewer iterations, and faster completion.

## Quick Start

```bash
# 1. Connect your account
crewkit auth login

# 2. Initialize your project
crewkit init

# 3. Start coding with your team's agents
crewkit code
```

That's it. Your team's agent configurations are now synced to `.claude/agents/` and ready to use.

## Features

### Shared Agent Configurations

Define agent configurations at the organization level, then customize per-project. Everyone on your team gets the same base agents with project-specific context.

```bash
crewkit resources list              # See available agents
crewkit resources show rails-expert # View agent details
```

### Role-Based Agent Modes

Different team members need different agent behaviors:

| Role | Mode | Behavior |
|------|------|----------|
| Junior | Coaching | Agents guide and explain, don't write code directly |
| Intermediate | Collaborative | Agents suggest and implement with explanation |
| Senior | Autonomous | Agents execute with minimal hand-holding |

Modes are applied automatically based on your team role.

### A/B Testing for Prompts

Test changes to your agent configurations with real usage data. Experiments are
created and managed in the dashboard; the CLI can inspect them:

```bash
crewkit experiments show <slug>   # View an experiment's details
```

### Session Tracking

Every coding session is tracked for analysis. See which agents perform best, identify patterns, and continuously improve.

## Commands

| Command | Description |
|---------|-------------|
| `crewkit` | Show status overview (auth, org, project, agents) |
| `crewkit code` | Start coding session with synced agents |
| `crewkit auth login` | Connect your account |
| `crewkit auth logout` | Sign out |
| `crewkit auth status` | Check authentication status |
| `crewkit init` | Set up crewkit for current project |
| `crewkit resources list` | List available agents |
| `crewkit resources show <name>` | View agent configuration |
| `crewkit experiments show <slug>` | View experiment details |
| `crewkit org info` | Show organization details |
| `crewkit project list` | List projects |
| `crewkit project info` | Show current project details |
| `crewkit playbooks list` | View playbooks and conventions for your projects |
| `crewkit sessions list` | List recent sessions (current project by default) |
| `crewkit sessions show <id>` | Session detail: timing, cost, analysis, tasks, shares |
| `crewkit sessions share <id>` | Create a share link (`--list` / `--revoke` to manage) |
| `crewkit sessions report` | Team/project digest: sessions, cost, top agents, trends |
| `crewkit sessions import` | Import historical sessions from JSONL |
| `crewkit blueprint <subcommand>` | AI-powered project planning (create, list, show, tasks, ...) |
| `crewkit update` | Update crewkit to the latest version |
| `crewkit feedback <msg>` | Send feedback |
| `crewkit lsp <subcommand>` | Code intelligence (LSP) for Claude Code |
| `crewkit lsp doctor` | Diagnose the LSP setup (`--json` for scripting) |
| `crewkit mcp serve` | Stdio MCP server exposing crewkit's institutional memory |
| `crewkit whoami` | Show current user |

Run `crewkit --help` for full command reference.

## Code Intelligence (LSP)

`crewkit lsp install` registers a Claude Code plugin that runs the built-in
crewkit language server (`crewkit lsp start --stdio`) for go-to-definition
and symbol search across Rust, TypeScript/JavaScript, and Ruby. Symbols are
indexed into `.claude/.crewkit-lsp/index.redb` per project.

When code intelligence isn't working, run the diagnostic:

```bash
crewkit lsp doctor          # human-readable report
crewkit lsp doctor --json   # machine-readable, for scripts/CI
```

It checks, with a pass/warn/fail marker and a fix-it hint per line:

- Claude Code CLI availability
- Plugin installation and enablement (user, project, and local scopes)
- The plugin's pinned binary path (exists, executable, version matches the
  running crewkit — stale after updates is the most common silent failure)
- Conflicting LSP plugins (official rust-analyzer/typescript/ruby LSPs)
- redb index health (exists, size, openable, file/symbol counts, leftover
  per-process index files from dead sessions)
- tree-sitter parser coverage for the languages detected in the repo
- A stdio handshake smoke test (spawns the server, sends `initialize`,
  expects a response, then shuts down cleanly)

Exits non-zero if any check fails. The TUI sidebar's Session section also
shows a one-line LSP status (`active · N files`, `plugin off`, or `failed`
with a pointer to `crewkit lsp doctor`).

## Institutional Memory (MCP)

`crewkit mcp serve` runs a stdio [MCP](https://modelcontextprotocol.io)
server that gives any MCP-capable agent pull access to your project's
institutional memory:

| Tool | What it answers |
|------|-----------------|
| `search_project_context` | Hybrid search over uploaded docs, meeting transcripts, agent emails/Slack, and vectorized past sessions |
| `find_prior_work` | Past conversations where the team already solved something similar |
| `get_active_work` | Who's working on what in this project right now |
| `get_conventions` | Playbook conventions for the project's tech stack |
| `get_blueprint_state` | Active Blueprint progress and your assigned tasks |

**Automatic:** sessions launched via `crewkit code` register the server
automatically (a generated `--mcp-config` pointing back at the crewkit
binary, with org/project pre-resolved).

**Manual** (for `claude` sessions not launched through crewkit, or other
MCP clients):

```bash
claude mcp add crewkit -- crewkit mcp serve
```

Or add to your project's `.mcp.json`:

```json
{
  "mcpServers": {
    "crewkit": {
      "type": "stdio",
      "command": "crewkit",
      "args": ["mcp", "serve"]
    }
  }
}
```

In the manual path the server resolves your org/project from the git remote
(same resolution `crewkit code` uses). Override with `--org` / `--project`
flags or `CREWKIT_ORG_ID` / `CREWKIT_PROJECT_ID` env vars. Authentication
reuses your `crewkit auth login` credentials; unauthenticated calls return
an actionable error instead of failing the session.

## Headless Mode

Run crewkit in scripts and CI pipelines:

```bash
# Run with a prompt
crewkit code -p "fix the failing tests"

# JSON output for parsing (text | json | stream-json)
crewkit code -p "add input validation" --output-format json

# Quiet mode with timeout
crewkit code -p "refactor auth module" -q --timeout 600
```

Headless runs get the same observability as interactive sessions: a
conversation is created with trigger `headless`, hook events stream to the
platform, and the session is closed with a success/failure outcome when the
run finishes (or `timeout` if the time limit is hit — exit code 4). Progress
messages go to stderr so stdout stays parseable with `--output-format json`.

Headless mode requires authentication and a detectable project — it exits
with an error instead of degrading to an untracked run. If your organization
has enabled LLM gateway telemetry, headless runs route Claude's Anthropic API
traffic through the local gateway automatically (see below).

## LLM Gateway

`crewkit code --llm-gateway` starts a local proxy on 127.0.0.1 and points the
spawned Claude Code process at it (`ANTHROPIC_BASE_URL` is set on the child
process only). The gateway captures per-request telemetry — model, token
usage (including cache tokens), duration, stop reason, status — and uploads
it to the platform in batches.

Enablement rules (all must hold):

- The organization has LLM gateway telemetry enabled (org setting)
- The session is not marked `--sensitive`
- The gateway was requested: `--llm-gateway`, or implicitly by headless mode

Privacy:

- Request/response bodies are uploaded only when the organization's capture
  mode is `full`, and are always secret-redacted first. Otherwise only
  metadata is sent (null bodies).
- Local logs in `~/.crewkit/logs/gateway-*.log` follow the same body rules,
  are created owner-only (0600), and are pruned after 7 days.
- Upload failures are logged and dropped (bounded in-memory queue, no
  retries) so a flaky network never blocks your session.

`--llm-gateway-port <PORT>` overrides the default ephemeral port. The gateway
is skipped under Bedrock/Vertex routing and in no-auth/offline mode.

## Session Resume

Continue where you left off:

```bash
crewkit code -c                    # Continue most recent session
crewkit code -r                    # Pick a session to resume
crewkit code -r abc123             # Resume specific session
crewkit code -r abc123 --fork-session  # Fork into new session
```

## Platforms

| Platform | Architecture | Status |
|----------|--------------|--------|
| macOS | Apple Silicon (arm64) | Supported |
| macOS | Intel (x64) | Supported |
| Linux | x64 | Supported |
| Windows | x64 | Supported |

## Configuration

crewkit auto-detects your project from git remotes and registers it on first `crewkit code`. For manual configuration:

```bash
crewkit init               # Pin the org/project mapping locally
crewkit init --workspace   # Create and register a multi-repo workspace project
```

`crewkit init` writes a local `.agent/config.yml` with your organization and project settings — useful when git detection is ambiguous. It does not register the project with the API; registration happens via `crewkit code` (or `crewkit init --workspace` for workspaces).

## Global Flags

| Flag | Description |
|------|-------------|
| `--no-color` | Disable colored output |
| `--plain` | Use ASCII-only output (no Unicode) |
| `-h, --help` | Show help for any command |
| `-V, --version` | Show version |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `CREWKIT_API_URL` | API endpoint (for self-hosted deployments) |
| `CREWKIT_TOKEN` | Access token for CI/scripts (takes precedence over stored login) |
| `CREWKIT_WEB_URL` | Web/dashboard URL override (for self-hosted deployments) |
| `CREWKIT_REPOSITORY_ID` | Explicit repository ID for `crewkit code` (skips detection/linking) |
| `CREWKIT_ENV` | Environment tag for error reporting (default: production) |
| `NO_COLOR` | Disable colored output |

## Links

- [Website](https://crewkit.io)
- [Documentation](https://docs.crewkit.io)
- [Install Script](https://crewkit.io/install.sh)
- [GitHub](https://github.com/karibew/crewkit-cli)

## Development

### Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| [mise](https://mise.jdx.dev) | latest | `brew install mise` |
| [just](https://github.com/casey/just) | latest | `brew install just` |
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | latest | `npm install -g @anthropic-ai/claude-code` |

```bash
# macOS — install mise, just, and Docker
brew install mise just

# PostgreSQL via Docker (recommended) — uses docker-compose.yml in repo root
docker compose up -d       # Postgres 17 on :5432 (password: root)
```

### Getting Started

```bash
# Clone and install tool versions (Ruby, Node, Rust via .mise.toml)
git clone git@github.com:karibew/crewkit.git && cd crewkit
mise install

# API
cd api && bundle install && bin/rails db:setup && cd ..

# Dashboard
cd dashboard && npm install && cd ..

# CLI
cd cli && just setup && just build && just install-dev
```

### Running

```bash
cd api && bin/dev              # Rails API on :3050
cd dashboard && npm run dev    # Next.js dashboard on :3051
crewkit-dev code               # CLI against local API (auto-compiles)
```

`crewkit-dev` is an auto-compile dev runner that points to `localhost:3050` and builds from source on each invocation. Install it with `just install-dev` from the `cli/` directory. Use it from any project directory:

```bash
crewkit-dev code        # Start session against local API
crewkit-dev auth status # Check auth against local API
```

## License

See [Terms of Service](https://crewkit.io/terms) for license details.
