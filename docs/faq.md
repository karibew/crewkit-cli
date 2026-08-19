# Frequently Asked Questions

---

## General

### What is crewkit?

crewkit is the institutional memory and governance layer for AI-assisted
engineering. It gives Claude Code your team's context, enforces your
standards with role-based configurations, and measures which agent versions
actually work.

### Is crewkit free?

crewkit offers both free and paid tiers. Check https://crewkit.io/pricing
for details.

### Is the source code open source?

The source code is currently private, but we track issues and documentation
publicly on GitHub.

### What languages/frameworks does crewkit support?

crewkit works with any project that Claude Code supports. Agent
configurations can be customized for:
- Rails, Python, Node.js, Go, Rust, Java, etc.
- Frontend frameworks (React, Vue, Angular, etc.)
- Any language or framework

### How is this different from committing a CLAUDE.md to the repo?

A committed CLAUDE.md is a snapshot with no history, no inheritance, and no
feedback loop. crewkit configurations are versioned (every save is an
immutable version you can roll back), inherited (platform → organization →
project), and measured — every session records the exact version of every
agent that ran, so you can see which configuration changes actually improved
outcomes, and A/B test them.

### Does crewkit send my code to your servers?

Session telemetry (events, timing, token usage, cost, outcomes) goes to your
organization's crewkit account over HTTPS — nowhere else. Hook events are
sanitized before upload. Request/response bodies are captured only when your
organization enables `full` capture mode, and are secret-redacted first.
Sessions started with `--sensitive` are excluded from content capture and
search indexing entirely. See the
[Privacy & Data Handling](../README.md#privacy--data-handling) section for
the full picture.

### Can crewkit help with compliance (e.g., Quebec's Law 25)?

That's a core use case. crewkit gives you an audit trail of AI-assisted
work: which agent version ran in which session, under which conventions, at
what cost, with org-controlled content capture and redaction. If you need to
show an auditor how AI participates in your engineering process, that record
is the product.

---

## Installation & Setup

### What are the system requirements?

- **OS**: macOS (Apple Silicon), Linux (x64), or Windows (x64) via WSL —
  native Windows builds are paused
- **Claude Code**: installed and on your PATH
- **Node.js 18+**: only if you install via npm — the CLI itself is a native
  binary with no runtime dependency

### Can I use crewkit without Claude Code?

No, crewkit requires Claude Code to function. It manages agents that run
within Claude Code.

### How do I update crewkit?

Curl-installed binaries auto-update in the background. To update manually:

```bash
crewkit update                # curl installs
brew upgrade crewkit          # Homebrew
npm update -g @crewkit/cli    # npm
choco upgrade crewkit         # Chocolatey (frozen at 0.4.1: native Windows
                              #   builds are paused, so this reports "up to
                              #   date" forever — use WSL instead)
```

### Can I use crewkit in CI/CD?

Yes. Headless mode runs a full observed session from a script:

```bash
crewkit code -p "fix the failing tests" --output-format json
```

Set `CREWKIT_TOKEN` in your CI secret store for authentication. Headless
runs get the same observability as interactive sessions. See the
[Headless Mode](../README.md#headless-mode) section of the README.

---

## Authentication

### How does authentication work?

crewkit uses OAuth 2.1 device flow. You authenticate via browser, and tokens
are stored encrypted at rest in a local vault under `~/.config/crewkit/` —
no OS keychain required, and never in plain text.

### How long do tokens last?

- Access tokens: 4 hours
- Refresh tokens: automatically renew access tokens and are rotated on
  every use

### Can I use multiple accounts?

Currently, crewkit supports one authenticated account at a time. To switch
accounts, log out and log in again.

---

## Project Setup

### What is `.agent/config.yml`?

This file pins your project to crewkit when git-remote detection is
ambiguous. It contains only slugs — no secrets:
```yaml
org: your-org
project: your-project
```

### Should I commit `.agent/config.yml` to git?

Yes. It contains only your organization and project slugs, and committing it
means all team members resolve to the same project.

### What is `.claude/agents/`?

This directory contains the agent configurations crewkit syncs from your
organization. crewkit manages these files and backs up anything it
overwrites to `.claude/.backups/`.

### Should I commit `.claude/` to git?

That's your team's choice — many teams commit their own `.claude/` content
(rules, commands, CLAUDE.md). What you generally *shouldn't* commit are the
crewkit-managed files, since they're synced from crewkit.io on every
session. If your team commits `.claude/`, scope the ignores:

```
.claude/agents/
.claude/.backups/
.crewkit/
```

`.crewkit/` is crewkit's per-project state (debug logs, caches) and should
always be ignored — workspace init adds it to `.gitignore` for you.

---

## Agent Management

### What are base agents?

Base agents are predefined agent configurations like:
- `rails-expert`
- `frontend-expert`
- `python-expert`

They serve as starting points that can be customized at organization and
project levels.

### What are organization overrides?

Organization-level configs that apply to all projects in your organization
(e.g., company coding standards, tools, style guides).

### What are project overrides?

Project-specific configs that only apply to one project (e.g., project
architecture, specific libraries used).

### What are role modifiers?

Role-based behavior changes, applied automatically from your team role:

- **Junior**: coaching mode — agents guide and explain, don't write code
- **Intermediate**: collaborative mode — agents suggest and implement with
  explanation
- **Senior**: autonomous mode — agents execute with minimal hand-holding

### How do I modify an agent?

1. Log in to https://crewkit.io
2. Navigate to your project
3. Edit agent configurations in the web UI
4. Run `crewkit code` to sync changes locally

---

## Experiments

### What are experiments?

A/B testing for agent configurations. You can:
- Create variants of agent configs
- Compare performance (quality scores, speed, cost)
- Deploy the winning variant

### How do I create an experiment?

```bash
crewkit experiments create rails-expert
```

This creates an experiment with a 3-word name like `swift-amber-falcon`.

### How do I view experiment results?

```bash
crewkit experiments metrics swift-amber-falcon
```

### How do I deploy a winning experiment?

```bash
crewkit experiments deploy swift-amber-falcon
```

This promotes the experiment variant to production.

---

## File Syncing

### How does syncing work?

When you run `crewkit code`:
1. CLI detects your project from git remote
2. Fetches effective configs from crewkit.io (base + org + project + role)
3. Writes all agents to `.claude/agents/`
4. Creates checksums and backups
5. Launches Claude Code

### What if I manually edit `.claude/agents/` files?

crewkit detects changes and logs them to your session. However, manual edits
are not synced back to crewkit.io. Use the web UI to make permanent changes.

### Where are backups stored?

`.claude/.backups/{agent}/{timestamp}.md`

### How do I restore a backup?

```bash
cp .claude/.backups/rails-expert/2026-08-17T14-30-22.md .claude/agents/rails-expert.md
```

---

## Troubleshooting

### Why aren't agents syncing?

Check:
1. Authentication: `crewkit auth status`
2. Project config: `cat .agent/config.yml`
3. Network connection
4. Project exists in https://crewkit.io

See [Troubleshooting Guide](troubleshooting.md) for more.

### Why is Claude Code not launching?

Ensure:
1. Claude Code is installed
2. It's in your PATH (`which claude`)
3. You have the latest version

---

## Best Practices

### Should I use coaching mode for junior developers?

Yes. Coaching mode prevents agents from writing code directly, instead
guiding juniors through the process. This helps them learn.

### How often should I create experiments?

When you notice patterns:
- Same modifications across sessions
- Team members tweaking agents similarly
- Wanting to test new approaches

### Should I create agents for every tech stack?

Start with broad agents (rails-expert, frontend-expert). Add specialized
agents as needed.

### How do I share configs with my team?

1. Configure agents in the web UI at org or project level
2. Team members run `crewkit code` to sync
3. Agents are automatically synced for everyone

---

## Security

### Is it safe to store tokens on my machine?

Tokens are stored encrypted at rest in a local vault under
`~/.config/crewkit/` — never in plain text. See [SECURITY.md](../SECURITY.md)
for the full data-handling picture.

### What if I lose my laptop?

Access tokens expire within 4 hours on their own. Review recent account
activity on your profile at https://crewkit.io, and contact
security@crewkit.io to revoke credentials for the lost device.

### Does crewkit collect telemetry?

Session telemetry is the product — it goes to your organization's crewkit
account and nowhere else (see "Does crewkit send my code to your servers?"
above). Separately, the CLI reports crashes to Sentry by default; disable
with `CREWKIT_NO_TELEMETRY=1` or `DO_NOT_TRACK=1`.

---

## Support

### How do I report a bug?

[Open an issue](https://github.com/karibew/crewkit-cli/issues/new?template=bug_report.yml) with:
- CLI version (`crewkit --version`) and install method
- OS
- Steps to reproduce
- Error messages, and `.crewkit/debug-latest.log` if relevant

### How do I request a feature?

[Open a feature request](https://github.com/karibew/crewkit-cli/issues/new?template=feature_request.yml).

### Where can I ask questions?

- [GitHub Discussions](https://github.com/karibew/crewkit-cli/discussions)
- [GitHub Issues](https://github.com/karibew/crewkit-cli/issues/new?template=question.yml)

### How do I contact support?

- **Email**: support@crewkit.io
- **Security**: security@crewkit.io

---

## Still have questions?

- [Installation Guide](installation.md)
- [Authentication Guide](authentication.md)
- [Troubleshooting Guide](troubleshooting.md)
- [Command Reference](../README.md#commands)
- [Ask in Discussions](https://github.com/karibew/crewkit-cli/discussions)
