# Security Policy

## Supported Versions

We support the latest released version of the crewkit CLI with security
updates. The CLI auto-updates by default, so staying current requires no
action on most installs.

---

## Reporting a Vulnerability

**Please do not open public issues for security vulnerabilities.**

If you discover a security vulnerability, please email us at:

**security@crewkit.io**

### What to Include

Please provide:

1. **Description** of the vulnerability
2. **Steps to reproduce** the issue
3. **Potential impact** (what could an attacker do?)
4. **Suggested fix** (if you have one)
5. **Your contact information** (for follow-up questions)

### What to Expect

We aim to acknowledge reports within 2 business days and to send an initial
assessment within a week. We'll keep you informed while a fix is in
progress; timelines depend on severity, with critical issues prioritized
for out-of-band releases.

### Responsible Disclosure

We ask that you:

- **Do not publicly disclose** the vulnerability until we've released a fix
- **Do not exploit** the vulnerability beyond what's necessary to demonstrate it
- **Allow reasonable time** for us to address the issue

In return, we aim to:

- **Credit you** in the release notes (if desired)
- **Keep you informed** throughout the process
- **Act in good faith** to resolve the issue quickly

---

## How crewkit Handles Your Data

### Authentication

- **Flow**: OAuth 2.1 device flow — you authenticate in the browser; no
  passwords ever touch the CLI.
- **Storage**: tokens are stored encrypted at rest in a local vault under
  `~/.config/crewkit/`. For CI, the `CREWKIT_TOKEN` environment variable
  takes precedence over the stored login — keep it in your CI secret store.
- **Expiry**: access tokens expire after 4 hours; refresh tokens are rotated
  on every use.
- **Logout**: `crewkit auth logout` removes the credentials from your
  machine. If you believe a token was compromised, contact
  security@crewkit.io.

### Session Telemetry

Observing sessions is what crewkit does — telemetry (events, timing, token
usage, cost, outcomes) is sent over HTTPS to your organization's crewkit
account, and to nowhere else:

- Hook events are sanitized before upload.
- Request/response bodies are captured only when your organization's capture
  mode is `full`, and are always secret-redacted first; otherwise only
  metadata is recorded.
- `crewkit code --sensitive` excludes a session's content from capture,
  search indexing, and gateway telemetry.
- Local gateway logs (`~/.crewkit/logs/`) follow the same body rules, are
  created owner-only (0600), and are pruned after 7 days.

### Crash Reporting

The CLI reports crashes and errors to Sentry by default so we can fix them.
Disable with `CREWKIT_NO_TELEMETRY=1` or the standard `DO_NOT_TRACK=1`.

### Releases and Updates

- Release binaries are signed and checksummed (`SHA256SUMS` ships with every
  release).
- The auto-updater verifies release signatures against keys compiled into
  the CLI and refuses to install on any mismatch.
- The install script verifies SHA256 checksums before installing.

### File System

- Agent files are written to `.claude/agents/` in your project; backups of
  overwritten files go to `.claude/.backups/`.
- Per-project state (debug logs, caches) lives in `.crewkit/` — add it to
  your `.gitignore` (workspace init does this for you).

---

## Security Best Practices

1. **Stay current** — auto-update is on by default; `crewkit update` forces
   a check, and `crewkit --version` shows what you're running.
2. **Protect CI tokens** — store `CREWKIT_TOKEN` in your CI provider's
   secret store, never in the repository.
3. **`.agent/config.yml` is safe to commit** — it contains only your
   organization and project slugs, no secrets.
4. **Use `--sensitive`** for sessions touching data that shouldn't be
   captured or indexed.

---

## Security Updates

Security updates are released as:

- **Patch versions** for low/medium severity issues
- **Out-of-band releases** for critical vulnerabilities
- **Advisories** published on GitHub Security Advisories

Subscribe to releases on GitHub to get notified.

---

## Contact

For security concerns: **security@crewkit.io**

For general questions: [Open a discussion](https://github.com/karibew/crewkit-cli/discussions)
