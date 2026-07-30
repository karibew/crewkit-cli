# Troubleshooting Guide

Common issues and solutions for crewkit CLI.

---

## Installation Issues

### Command Not Found After Install

**Problem**: `crewkit: command not found`

**Solution**:
```bash
# Check npm global bin path
npm bin -g

# Add to PATH (add to ~/.bashrc, ~/.zshrc, or ~/.profile)
export PATH="$(npm bin -g):$PATH"

# Reload shell
source ~/.bashrc  # or ~/.zshrc
```

### Permission Errors (EACCES)

**Problem**: `EACCES: permission denied`

**Solution**: Use a Node version manager:
```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Install Node
nvm install 18
nvm use 18

# Install crewkit
npm install -g @crewkit/cli
```

---

## Authentication Issues

### Keychain Access Denied (macOS)

**Problem**: "User denied access to keychain"

**Solution**:
1. Open **Keychain Access.app**
2. Search for "crewkit"
3. Right-click → Get Info → Access Control
4. Add `crewkit` to "Always allow access"

### No Secret Service (Linux)

**Problem**: "No secret service available"

**Solution**:
```bash
# Install keyring
sudo apt-get install gnome-keyring libsecret-1-0

# Start daemon
eval $(gnome-keyring-daemon --start)
```

### Token Expired

**Problem**: "Authentication failed: token expired"

**Solution**:
```bash
crewkit auth logout
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

### Project Not Found in crewkit

**Problem**: "Project not found: org/repo"

**Solution**:
1. Log in to https://crewkit.io
2. Create organization and project
3. Run `crewkit init org project`

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

4. Force sync:
   ```bash
   rm -rf .claude/agents
   crewkit code
   ```

### File Backup Errors

**Problem**: "Could not create backup"

**Solution**:
```bash
# Check permissions
ls -la .claude

# Recreate directory
rm -rf .claude
mkdir -p .claude/agents
mkdir -p .claude/.backups
```

---

## Claude Code Launch Issues

### Claude Code Not Found

**Problem**: "Claude Code executable not found"

**Solution**:
1. Install Claude Code from https://claude.ai/download

2. Ensure it's in PATH:
   ```bash
   # macOS
   which claude

   # Add to PATH if needed
   export PATH="/Applications/Claude.app/Contents/MacOS:$PATH"
   ```

3. Try absolute path:
   ```bash
   CLAUDE_PATH="/Applications/Claude.app/Contents/MacOS/claude" crewkit code
   ```

### Claude Code Crashes

**Problem**: Claude Code crashes on launch

**Solution**:
1. Check Claude Code version (must be latest)
2. Clear Claude Code cache
3. Check `.claude/agents/` files for corruption:
   ```bash
   ls -lh .claude/agents/
   file .claude/agents/*
   ```

---

## Network Issues

### API Connection Failed

**Problem**: "Could not connect to crewkit.io API"

**Solution**:
1. Check internet connection
2. Check if API is up: https://status.crewkit.io
3. Try with verbose logging:
   ```bash
   DEBUG=* crewkit code
   ```

4. Check firewall/proxy settings

### Slow Syncing

**Problem**: Agent syncing takes a long time

**Solution**:
1. Check network speed
2. Reduce number of agents in project settings
3. Use `--no-watch` flag to skip file watching:
   ```bash
   crewkit code --no-watch
   ```

---

## Configuration Issues

### Invalid Config File

**Problem**: "Invalid .agent/config.yml"

**Solution**:
```bash
# Validate YAML syntax
cat .agent/config.yml

# Regenerate config
crewkit init --force
```

Expected format:
```yaml
org: your-org
project: your-project
```

### Merge Conflict in Config

**Problem**: Git merge conflict in `.agent/config.yml`

**Solution**:
```bash
# Accept theirs
git checkout --theirs .agent/config.yml

# Or accept ours
git checkout --ours .agent/config.yml

# Mark as resolved
git add .agent/config.yml
```

---

## Performance Issues

### High CPU Usage

**Problem**: `crewkit` process using high CPU

**Solution**:
- Disable file watching: `crewkit code --no-watch`
- Check for runaway processes: `ps aux | grep crewkit`
- Update to latest version: `npm update -g @crewkit/cli`

### High Memory Usage

**Problem**: `crewkit` using too much memory

**Solution**:
- Restart the CLI
- Clear cache: `rm -rf ~/.cache/crewkit`
- Report issue with memory profile

---

## Debugging

### Enable Debug Logging

```bash
# All debug output
DEBUG=* crewkit code

# Specific modules
DEBUG=crewkit:* crewkit code
DEBUG=crewkit:api crewkit code
```

### Check Logs

```bash
# macOS
tail -f ~/Library/Logs/crewkit/debug.log

# Linux
tail -f ~/.cache/crewkit/logs/debug.log

# Windows
type %APPDATA%\crewkit\logs\debug.log
```

### Report an Issue

When reporting issues, include:

1. **Version**: `crewkit --version`
2. **OS**: `uname -a` (macOS/Linux) or `systeminfo` (Windows)
3. **Node**: `node --version`
4. **Error message**: Full error output
5. **Steps to reproduce**: Exact commands run
6. **Logs**: Debug logs if available

Use the [bug report template](https://github.com/karibew/crewkit-cli/issues/new?template=bug_report.yml).

---

## Common Error Messages

### `ENOENT: no such file or directory`

**Cause**: File or directory doesn't exist

**Fix**: Check file paths, ensure `.claude/` directory exists

### `EACCES: permission denied`

**Cause**: Insufficient permissions

**Fix**: Check file permissions, use `chmod` if needed

### `ETIMEDOUT: network timeout`

**Cause**: Network connection issue

**Fix**: Check internet, retry, check firewall

### `401 Unauthorized`

**Cause**: Invalid or expired token

**Fix**: `crewkit auth logout && crewkit auth login`

### `404 Not Found`

**Cause**: Project doesn't exist

**Fix**: Verify org/project in https://crewkit.io, run `crewkit init`

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
- [Command Reference](commands.md)
- [FAQ](faq.md)
