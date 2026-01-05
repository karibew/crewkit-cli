# crewkit

[![npm version](https://img.shields.io/npm/v/@crewkit/cli.svg)](https://www.npmjs.com/package/@crewkit/cli)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Build Status](https://img.shields.io/github/actions/workflow/status/karibew/crewkit-cli/ci.yml?branch=main)](https://github.com/karibew/crewkit-cli/actions)

**Your standards, every session.**

Stop reviewing AI-generated inconsistency. crewkit syncs your team's Claude Code agent configurations, tracks what works, and lets you experiment with improvements.

<!-- TODO: Add demo GIF -->

## Install

```bash
npm install -g @crewkit/cli
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

Test changes to your agent configurations with real usage data:

```bash
crewkit experiments create rails-expert       # Creates experiment with auto-generated slug
crewkit experiments metrics swift-amber-falcon # View results
crewkit experiments deploy swift-amber-falcon  # Deploy the winner
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
| `crewkit experiments create <agent>` | Create A/B test experiment |
| `crewkit experiments list` | List all experiments |
| `crewkit experiments metrics <slug>` | View experiment results |
| `crewkit experiments deploy <slug>` | Deploy experiment winner |
| `crewkit org info` | Show organization details |
| `crewkit project list` | List projects |
| `crewkit project info` | Show current project details |
| `crewkit members list` | List team members |
| `crewkit whoami` | Show current user |

Run `crewkit --help` for full command reference.

## Headless Mode

Run crewkit in scripts and CI pipelines:

```bash
# Run with a prompt
crewkit code -p "fix the failing tests"

# JSON output for parsing
crewkit code -p "add input validation" --output-format json

# Quiet mode with timeout
crewkit code -p "refactor auth module" -q --timeout 600
```

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

crewkit auto-detects your project from git remotes. For manual configuration:

```bash
crewkit init
```

This creates `.agent/config.yml` with your organization and project settings.

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
| `NO_COLOR` | Disable colored output |

## Links

- [Website](https://crewkit.io)
- [Documentation](https://docs.crewkit.io)
- [npm Package](https://www.npmjs.com/package/@crewkit/cli)
- [GitHub](https://github.com/karibew/crewkit-cli)

## License

MIT
