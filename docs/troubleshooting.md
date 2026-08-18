# Troubleshooting Guide

Common issues and solutions for crewkit CLI.

---

## First Stop: Debug Logs

Almost every issue below is easier to diagnose with the debug log. crewkit
writes a per-session log to `.crewkit/` in your project root:

```bash
crewkit code --debug          # --debug works on any command
tail -f .crewkit/debug-latest.log
```

For code-intelligence (LSP) issues, run `crewkit lsp doctor` — it checks the
whole setup and prints a fix-it hint per failing line.

---

## Installation Issues

### Command Not Found After Install

**Problem**: `crewkit: command not found`

**Solution** depends on the install method:

```bash
# curl install — ensure ~/.local/bin is on PATH
# (add to ~/.bashrc, ~/.zshrc, or ~/.profile)
export PATH="$HOME/.local/bin:$PATH"

# npm install — ensure npm's global bin dir is on PATH
export PATH="$(npm prefix -g)/bin:$PATH"
```

Then restart your terminal or `source` your shell profile.

### Permission Errors with npm (EACCES)

**Problem**: `EACCES: permission denied` during `npm install -g`

**Solution**: use a Node version manager (nvm, fnm, mise) instead of a
system Node, or change npm's global prefix to a user-writable directory:

```bash
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
export PATH=~/.npm-global/bin:$PATH
npm install -g @crewkit/cli
```

Alternatively, skip Node entirely — the curl install has no dependencies:

```bash
curl -fsSL https://crewkit.io/install.sh | sh
```

---

## Authentication Issues

### Token Expired

**Problem**: "Authentication failed: token expired"

**Solution**:
```bash
crewkit auth logout
crewkit auth login
```

### Persistent Auth Failures

**Problem**: 401 errors even after logging in again

**Solution**: check for a stale `CREWKIT_TOKEN` in your environment — it
takes precedence over the stored login:

```bash
env | grep CREWKIT_TOKEN && unset CREWKIT_TOKEN
```

If the problem persists, reset the credential vault:

```bash
crewkit auth logout
rm -f ~/.config/crewkit/auth.vault ~/.config/crewkit/.auth.key
crewkit auth login
```

---

## Project Detection Issues

### Git Remote Not Found

**Problem**: "Could not detect project from git remote"

**Solution**:
1. Ensure you're in a git repository:
   ```bash
   git status
   ```

2. Check git remote:
   ```bash
   git remote -v
   ```

3. Add remote if missing:
   ```bash
   git remote add origin git@github.com:org/repo.git
   ```

If detection stays ambiguous (e.g., multiple remotes), pin the project:

```bash
crewkit init
```

This writes `.agent/config.yml` with your org and project slugs.

### Project Not Found in crewkit

**Problem**: "Project not found: org/repo"

**Solution**:
1. Log in to https://crewkit.io and check the project exists
2. Run `crewkit init` to pin the org/project mapping locally

---

## File Sync Issues

### Agents Not Syncing

**Problem**: `.claude/agents/` directory is empty

**Solution**:
1. Check authentication:
   ```bash
   crewkit auth status
   ```

2. Check project config:
   ```bash
   cat .agent/config.yml
   ```

3. Verify project exists:
   - Log in to https://crewkit.io
   - Check organization/project settings

4. Force a clean re-sync:
   ```bash
   rm -rf .claude/agents .crewkit/cache
   crewkit code
   ```

---

## Claude Code Launch Issues

### Claude Code Not Found

**Problem**: "Claude Code executable not found"

crewkit launches the `claude` CLI from your PATH.

**Solution**:
1. Install Claude Code — see https://docs.claude.com/en/docs/claude-code

2. Verify it's reachable:
   ```bash
   which claude
   claude --version
   ```

3. If it's installed but not found, your shell's PATH differs from the
   terminal where you installed it — add Claude Code's install directory to
   the same shell profile crewkit runs under.

Note: Claude Code (the `claude` CLI) is not the Claude Desktop app —
installing the desktop app does not provide the `claude` binary.

---

## Network Issues

### API Connection Failed

**Problem**: "Could not connect to crewkit.io API"

**Solution**:
1. Check internet connection
2. Retry with the debug log:
   ```bash
   crewkit code --debug
   tail .crewkit/debug-latest.log
   ```
3. Check firewall/proxy settings — the CLI needs HTTPS to `api.crewkit.io`
   and `github.com` (for updates)

---

## Update Issues

### Auto-Update Keeps Failing

The updater verifies release signatures and pauses itself after a signature
mismatch rather than retrying forever — it will tell you on a later run.

**Solution**:
```bash
crewkit update --check     # See what the updater thinks
crewkit update             # Force an update attempt
```

If you installed via Homebrew, npm, or Chocolatey, self-update intentionally
defers to that package manager and prints the right command instead.

To disable background auto-update entirely: `CREWKIT_NO_AUTO_UPDATE=1`.

---

## Common Error Messages

### `401 Unauthorized`

**Cause**: Invalid or expired token (or a stale `CREWKIT_TOKEN` env var)

**Fix**: `crewkit auth logout && crewkit auth login`

### `404 Not Found`

**Cause**: Project doesn't exist

**Fix**: Verify org/project in https://crewkit.io, run `crewkit init`

### `ETIMEDOUT` / network timeout

**Cause**: Network connection issue

**Fix**: Check internet, retry, check firewall

---

## Reporting an Issue

When reporting issues, include:

1. **Version**: `crewkit --version`, and how you installed (curl/brew/npm/choco)
2. **OS**: `uname -a` (macOS/Linux) or `systeminfo` (Windows)
3. **Error message**: full error output
4. **Steps to reproduce**: exact commands run
5. **Logs**: the relevant part of `.crewkit/debug-latest.log`
   (sanitize anything sensitive), and `crewkit lsp doctor` output for
   code-intelligence issues

Use the [bug report template](https://github.com/karibew/crewkit-cli/issues/new?template=bug_report.yml).

---

## Still Having Issues?

1. **Search existing issues**: https://github.com/karibew/crewkit-cli/issues
2. **Check FAQ**: [faq.md](faq.md)
3. **Ask in Discussions**: https://github.com/karibew/crewkit-cli/discussions
4. **Report a bug**: https://github.com/karibew/crewkit-cli/issues/new

---

## Getting Help

- [Installation Guide](installation.md)
- [Authentication Guide](authentication.md)
- [Command Reference](../README.md#commands)
- [FAQ](faq.md)
