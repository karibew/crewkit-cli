# Installation Guide

crewkit ships as a native binary — no runtime required. Node.js 18+ is only
needed if you install through npm.

Supported platforms: macOS (Apple Silicon), Linux (x64), and Windows (x64) via
WSL. Native Windows builds are paused — see below.

---

## Install via Homebrew (macOS/Linux)

```bash
brew install karibew/tap/crewkit
```

Updates come through Homebrew: `brew upgrade crewkit`.

---

## Install via curl (macOS/Linux)

```bash
curl -fsSL https://crewkit.io/install.sh | sh
```

This will:
- Detect your platform automatically
- Download the appropriate binary from GitHub Releases
- Verify the SHA256 checksum
- Install to `~/.local/bin`

### Install Options

```bash
# Install a specific version
curl -fsSL https://crewkit.io/install.sh | sh -s -- --version X.Y.Z

# Install to a custom directory
CREWKIT_INSTALL_DIR=/usr/local/bin curl -fsSL https://crewkit.io/install.sh | sh

# Show help
curl -fsSL https://crewkit.io/install.sh | sh -s -- --help
```

### Add to PATH

If `~/.local/bin` isn't already on your PATH:

```bash
# Add to ~/.bashrc, ~/.zshrc, or ~/.profile
export PATH="$HOME/.local/bin:$PATH"
```

Restart your terminal or run `source ~/.bashrc` (or `~/.zshrc`).

---

## Install via npm

Requires Node.js 18+:

```bash
npm install -g @crewkit/cli
```

The npm package bundles prebuilt binaries per platform — nothing is compiled
or downloaded at install time.

Verify installation:
```bash
crewkit --version
```

---

## Windows — use WSL

Native Windows binaries are paused, so releases ship macOS arm64 and Linux x64
only. Install inside WSL with any Linux method above:

```bash
wsl npm install -g @crewkit/cli
```

The Chocolatey package (`choco install crewkit`) is frozen at 0.4.1 and no
longer follows releases. WSL is the better target regardless: background
observation is Unix-only, so it works there and not on native Windows.

---

## Shell completions

crewkit can generate tab completions for bash, zsh, fish, PowerShell, and
elvish via `crewkit completions <shell>`.

**Homebrew installs set this up automatically** — no action needed (start a
new shell after installing).

For every other install method, wire it up once:

### zsh

```zsh
mkdir -p ~/.zfunc
crewkit completions zsh > ~/.zfunc/_crewkit
```

Then make sure `~/.zshrc` adds the directory to `fpath` **before** `compinit`
runs:

```zsh
fpath+=~/.zfunc
autoload -Uz compinit && compinit
```

Alternatively, `source <(crewkit completions zsh)` anywhere **after**
`compinit` stays current automatically at the cost of a few milliseconds of
shell startup.

### bash

```bash
echo 'source <(crewkit completions bash)' >> ~/.bashrc
```

### fish

```fish
mkdir -p ~/.config/fish/completions
crewkit completions fish > ~/.config/fish/completions/crewkit.fish
```

The zsh and fish setups write a static file: regenerate it after crewkit
updates (which happen automatically in the background) to pick up new
commands and flags. The bash and PowerShell setups regenerate on every
shell start and never go stale.

### PowerShell

Add to your `$PROFILE`:

```powershell
crewkit completions powershell | Out-String | Invoke-Expression
```

---

## Update crewkit

### Auto-update (curl installs)

For binaries installed via curl/install script, crewkit keeps itself current:
on startup it checks for a new release (at most hourly), verifies the release
signature against keys built into the CLI, and installs it in the background.
The update takes effect on the next invocation.

```bash
crewkit update --check   # See if an update is available
crewkit update           # Update now
```

To disable background auto-update, set `CREWKIT_NO_AUTO_UPDATE=1`.

### Package-manager installs

If crewkit was installed through Homebrew, npm, or Chocolatey, self-update
defers to the package manager (and tells you the right command):

```bash
brew upgrade crewkit          # Homebrew
npm update -g @crewkit/cli    # npm
choco upgrade crewkit         # Chocolatey (frozen at 0.4.1 — see above)
```

---

## Uninstall

Remove the binary using the method you installed with:

```bash
brew uninstall crewkit           # Homebrew
npm uninstall -g @crewkit/cli    # npm
choco uninstall crewkit          # Chocolatey
rm ~/.local/bin/crewkit          # curl install
```

If you use the code-intelligence plugin, run `crewkit lsp uninstall` first
to deregister it from Claude Code.

**Note**: your login and configs remain in:
- `~/.config/crewkit/` (settings and the encrypted auth vault)
- `~/.cache/crewkit/` (update cache and markers)

To fully clean up:
```bash
rm -rf ~/.config/crewkit
rm -rf ~/.cache/crewkit
```

Projects where you ran crewkit also have a local `.crewkit/` directory
(debug logs, caches) you can delete at any time.

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
