# Frequently Asked Questions

---

## General

### What is crewkit?

crewkit is a CLI-first platform for managing AI agents (specifically Claude Code) with role-based configurations, A/B testing, and accuracy monitoring.

### Is crewkit free?

crewkit offers both free and paid tiers. Check https://crewkit.io/pricing for details.

### Is the source code open source?

The source code is currently private, but we track issues and documentation publicly on GitHub.

### What languages/frameworks does crewkit support?

crewkit works with any project that Claude Code supports. Agent configurations can be customized for:
- Rails, Python, Node.js, Go, Rust, Java, etc.
- Frontend frameworks (React, Vue, Angular, etc.)
- Any language or framework

---

## Installation & Setup

### What are the system requirements?

- **Node.js**: 18.0.0 or higher
- **OS**: macOS, Linux, or Windows
- **Claude Code**: Latest version installed

### Can I use crewkit without Claude Code?

No, crewkit requires Claude Code to function. It manages agents that run within Claude Code.

### How do I update crewkit?

```bash
npm update -g @crewkit/cli
```

### Can I use crewkit in CI/CD?

Not currently. crewkit is designed for local development with interactive Claude Code sessions.

---

## Authentication

### How does authentication work?

crewkit uses OAuth 2.1 device flow. You authenticate via browser, and tokens are stored securely in your OS keychain (macOS Keychain, Linux Secret Service, Windows Credential Manager).

### Where are my tokens stored?

Tokens are stored in your OS keychain, never in plain text files.

### How long do tokens last?

- Access tokens: 1 hour
- Refresh tokens: Automatically renew access tokens

### Can I use multiple accounts?

Currently, crewkit supports one authenticated account at a time. To switch accounts, log out and log in again.

---

## Project Setup

### What is `.agent/config.yml`?

This file identifies your project to crewkit. It contains:
```yaml
org: your-org
project: your-project
```

### Should I commit `.agent/config.yml` to git?

Yes! This file should be committed so all team members use the same configuration.

### What is `.claude/agents/`?

This directory contains synced agent configurations. It's managed by crewkit and should be `.gitignore`d.

### Should I commit `.claude/` to git?

No. Add this to `.gitignore`:
```
.claude/
```

Agent configurations are fetched from crewkit.io, not committed to git.

---

## Agent Management

### What are base agents?

Base agents are predefined agent configurations like:
- `rails-expert`
- `frontend-expert`
- `python-expert`

They serve as starting points that can be customized at organization and project levels.

### What are organization overrides?

Organization-level configs that apply to all projects in your organization (e.g., company coding standards, tools, style guides).

### What are project overrides?

Project-specific configs that only apply to one project (e.g., project architecture, specific libraries used).

### What are role modifiers?

Role-based behavior changes:
- **Junior/Entry**: Coaching mode (agents guide, don't code)
- **Intermediate/Senior**: Collaborative mode
- **Manager/Admin**: Autonomous mode

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
- Compare performance (accuracy, speed, cost)
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

crewkit detects changes and logs them to your session. However, manual edits are not synced back to crewkit.io. Use the web UI to make permanent changes.

### Where are backups stored?

`.claude/.backups/{agent}/{timestamp}.md`

### How do I restore a backup?

```bash
cp .claude/.backups/rails-expert/2025-10-11T14-30-22-123Z.md .claude/agents/rails-expert.md
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
2. It's in your PATH
3. You have the latest version

### Why am I getting keychain errors?

See [Authentication Troubleshooting](authentication.md#troubleshooting).

---

## Best Practices

### Should I use coaching mode for junior developers?

Yes! Coaching mode prevents agents from writing code directly, instead guiding juniors through the process. This helps them learn.

### How often should I create experiments?

When you notice patterns:
- Same modifications across sessions
- Team members tweaking agents similarly
- Wanting to test new approaches

### Should I create agents for every tech stack?

Start with broad agents (rails-expert, frontend-expert). Add specialized agents as needed.

### How do I share configs with my team?

1. Configure agents in the web UI at org or project level
2. Team members run `crewkit code` to sync
3. Agents are automatically synced for everyone

---

## Security

### Is it safe to store tokens in the keychain?

Yes. OS keychains are designed for secure credential storage and require authentication to access.

### What if I lose my laptop?

1. Log in to https://crewkit.io
2. Go to Settings → Security
3. Revoke access for lost devices

### Can I use crewkit in untrusted environments?

No. Only use crewkit on trusted machines where you control the environment.

### Does crewkit collect telemetry?

We use Sentry for crash reports (opt-in). No usage telemetry is collected without consent.

---

## Billing & Plans

### What's included in the free tier?

Check https://crewkit.io/pricing for current plans.

### Can I change plans?

Yes, via the web UI at https://crewkit.io/settings/billing.

### What happens if I exceed limits?

Depends on your plan. See https://crewkit.io/pricing.

---

## Support

### How do I report a bug?

[Open an issue](https://github.com/karibew/crewkit-cli/issues/new?template=bug_report.yml) with:
- CLI version
- OS and Node version
- Steps to reproduce
- Error messages

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
- [Command Reference](commands.md)
- [Ask in Discussions](https://github.com/karibew/crewkit-cli/discussions)
