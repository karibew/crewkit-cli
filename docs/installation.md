# Installation Guide

## Prerequisites

- **Operating System**: macOS, Linux, or Windows
- **Node.js** (optional): Version 18+ if using npm install method

---

## Install via curl (Recommended)

The fastest way to install crewkit:

```bash
curl -fsSL https://crewkit.io/install.sh | sh
```

This will:
- Detect your platform automatically
- Download the appropriate binary
- Verify the checksum
- Install to `~/.local/bin`

### Install Options

```bash
# Install specific version
curl -fsSL https://crewkit.io/install.sh | sh -s -- --version 0.2.0

# Install to custom directory
CREWKIT_INSTALL_DIR=/usr/local/bin curl -fsSL https://crewkit.io/install.sh | sh

# Show help
curl -fsSL https://crewkit.io/install.sh | sh -s -- --help
```

### Add to PATH

After installation, add crewkit to your PATH:

```bash
# Add to ~/.bashrc, ~/.zshrc, or ~/.profile
export PATH="$HOME/.crewkit/bin:$PATH"
```

Restart your terminal or run `source ~/.bashrc` (or `~/.zshrc`).

---

## Install via npm

If you have Node.js 18+, you can also install via npm:

```bash
npm install -g @crewkit/cli
```

Verify installation:
```bash
crewkit --version
```

### Local Installation

Install in a specific project:

```bash
npm install @crewkit/cli
```

Run via npx:
```bash
npx crewkit --version
```

---

## Update crewkit

### Auto-update (built-in)

crewkit automatically updates itself in the background. On every startup, it checks for updates (hourly cache) and downloads new versions automatically. The update takes effect on the next CLI invocation.

```bash
# Manual check/update (if needed)
crewkit update --check
crewkit update
```

### Manual update

**If installed via curl:**
```bash
curl -fsSL https://crewkit.io/install.sh | sh
```

**If installed via npm:**
```bash
npm update -g @crewkit/cli
```

Check what version you're running:
```bash
crewkit --version
```

---

## Uninstall

```bash
npm uninstall -g @crewkit/cli
```

This will remove:
- The `crewkit` command
- All installed plugins

**Note**: Your authentication tokens and configs will remain in:
- `~/.config/crewkit/` (configs)
- OS Keychain (tokens)

To fully clean up:
```bash
rm -rf ~/.config/crewkit
rm -rf ~/.cache/crewkit
```

---

## Platform-Specific Notes

### macOS

No additional setup required. Tokens are stored in macOS Keychain.

### Linux

Tokens are stored using Secret Service (GNOME Keyring, KWallet, etc.)

If you get keychain errors, install:
```bash
# Ubuntu/Debian
sudo apt-get install libsecret-1-dev

# Fedora
sudo dnf install libsecret-devel

# Arch
sudo pacman -S libsecret
```

### Windows

No additional setup required. Tokens are stored in Windows Credential Manager.

---

## Troubleshooting

### Permission Errors

If you get `EACCES` errors on macOS/Linux:

```bash
# Option 1: Use a Node version manager (recommended)
# Install nvm: https://github.com/nvm-sh/nvm
nvm install 18
nvm use 18
npm install -g @crewkit/cli

# Option 2: Change npm's default directory
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @crewkit/cli
```

### Command Not Found

If `crewkit` command is not found after installation:

1. Check npm global bin path:
   ```bash
   npm bin -g
   ```

2. Add it to your PATH:
   ```bash
   # Add to ~/.bashrc, ~/.zshrc, or ~/.profile
   export PATH="$(npm bin -g):$PATH"
   ```

3. Restart your terminal

### Installation Fails

If installation fails:

1. Clear npm cache:
   ```bash
   npm cache clean --force
   ```

2. Try again:
   ```bash
   npm install -g @crewkit/cli
   ```

3. Check npm logs:
   ```bash
   npm install -g @crewkit/cli --verbose
   ```

---

## Next Steps

After installation:

1. **[Authenticate](authentication.md)**: Log in to crewkit.io
2. **Initialize Project**: Run `crewkit init` to set up your first project
3. **Start Coding**: Run `crewkit code` to launch Claude Code with synced agents

---

## Getting Help

- [Troubleshooting Guide](troubleshooting.md)
- [FAQ](faq.md)
- [Report an Issue](https://github.com/karibew/crewkit-cli/issues)
