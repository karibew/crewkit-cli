# Security Policy

## Supported Versions

We support the latest version of the crewkit CLI with security updates.

| Version | Supported          |
| ------- | ------------------ |
| 0.1.x   | :white_check_mark: |
| < 0.1   | :x:                |

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

- **Acknowledgment**: Within 48 hours
- **Initial assessment**: Within 1 week
- **Status updates**: Every 1-2 weeks until resolved
- **Fix timeline**: Depends on severity
  - Critical: 1-7 days
  - High: 1-2 weeks
  - Medium: 2-4 weeks
  - Low: Best effort

### Responsible Disclosure

We ask that you:

- **Do not publicly disclose** the vulnerability until we've released a fix
- **Do not exploit** the vulnerability beyond what's necessary to demonstrate it
- **Allow reasonable time** for us to address the issue

In return, we commit to:

- **Credit you** in the release notes (if desired)
- **Keep you informed** throughout the process
- **Act in good faith** to resolve the issue quickly

---

## Security Best Practices

When using crewkit CLI:

### 1. Keep Updated

```bash
# Check your version
crewkit --version

# Update to latest
npm update -g @crewkit/cli
```

### 2. Protect Your Credentials

- Never commit `.agent/config.yml` with sensitive data
- Use environment variables for secrets
- Revoke tokens if compromised: `crewkit auth logout`

### 3. Review Agent Configurations

- Audit agent configs before deploying
- Limit agent permissions appropriately
- Monitor agent modifications via session logs

### 4. Secure Your API Keys

- Rotate API keys regularly
- Use separate keys for dev/prod
- Store keys in OS keychain (macOS/Linux)

---

## Known Security Considerations

### Authentication Tokens

- **Storage**: Tokens are stored in OS keychain
- **Expiry**: Access tokens expire after 1 hour
- **Refresh**: Refresh tokens are automatically rotated

### File System Access

- **Agent files**: Written to `.claude/agents/`
- **Backups**: Stored in `.claude/.backups/`
- **Permissions**: Uses default user permissions

### Network Communication

- **API calls**: HTTPS only to `api.crewkit.io`
- **No telemetry**: We don't collect usage data without consent
- **Error tracking**: Sentry for crash reports (opt-in)

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

---

Thank you for helping keep crewkit secure! 🔒
