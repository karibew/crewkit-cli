# Authentication Guide

crewkit uses OAuth 2.1 device flow for secure authentication. No passwords
are ever entered into or stored by the CLI.

---

## Quick Start

```bash
# Log in to crewkit.io
crewkit auth login

# Check authentication status
crewkit auth status

# Log out
crewkit auth logout
```

---

## Login Flow

### Step 1: Start Login

```bash
crewkit auth login
```

The CLI shows a verification URL and one-time code, and opens your browser
automatically when it can:

```
Please visit: https://crewkit.io/device-verify?code=ABCD-EFGH
Enter code: ABCD-EFGH

Waiting for authentication...
```

### Step 2: Authenticate in Browser

1. If the browser didn't open, visit the URL from the terminal
2. Confirm the code matches what the CLI displayed
3. Sign in with your crewkit.io account
4. Authorize the CLI

### Step 3: Success

Once authorized, the CLI receives its tokens, stores them encrypted, and
confirms:

```
✓ Authentication successful!
Logged in as: you@example.com
Organization: acme-corp
```

---

## Token Storage

Tokens are stored encrypted at rest in a local vault:

- **Location**: `~/.config/crewkit/` (`auth.vault` + key file)
- **No plaintext**: tokens are never written unencrypted to disk
- **No OS keychain required**: works the same on macOS, Linux (including
  headless servers), and Windows

For CI and scripts, set the `CREWKIT_TOKEN` environment variable — it takes
precedence over the stored login. Keep it in your CI provider's secret store.

---

## Check Authentication Status

```bash
crewkit auth status
```

Shows whether you're authenticated, as whom, and your organization. When not
authenticated, run `crewkit auth login` to get started.

---

## Logout

```bash
crewkit auth logout
```

This removes the stored credentials from your machine.

---

## Token Expiry

- **Access tokens**: expire after 4 hours
- **Refresh tokens**: automatically renew access tokens, and are rotated on
  every use

When an access token expires, the CLI transparently refreshes it and
continues — you only re-authenticate if the refresh token itself has
expired or been revoked.

---

## Multiple Accounts

Currently, crewkit supports one authenticated account at a time.

To switch accounts:
```bash
crewkit auth logout
crewkit auth login
```

---

## Troubleshooting

### Login Hangs or Times Out

1. Check that you can reach https://crewkit.io in a browser
2. Complete the browser step before the device code expires; if it expired,
   run `crewkit auth login` again for a fresh code

### Corrupted Credentials

If auth calls fail persistently after a successful login:

```bash
crewkit auth logout
rm -f ~/.config/crewkit/auth.vault ~/.config/crewkit/.auth.key
crewkit auth login
```

### CI Sessions Use the Wrong Account

`CREWKIT_TOKEN` takes precedence over your stored login. If a local shell
has it exported, unset it: `unset CREWKIT_TOKEN`.

### Browser Doesn't Open

In non-interactive environments the CLI prints the URL instead of opening a
browser:

1. Copy the URL from the terminal
2. Open it in any browser (another machine works too)
3. Enter the code shown
4. Return to the terminal

---

## Security Best Practices

### Do Not Share Tokens or Codes

- Never share your device code with anyone
- Never commit tokens to git
- Never post tokens in issues/forums

### If a Machine Is Lost or Compromised

- Your access tokens expire within 4 hours on their own
- Review recent account activity on your profile at https://crewkit.io
- Contact **security@crewkit.io** to revoke credentials for the lost device

---

## Next Steps

After authentication:
1. **Initialize Project**: Run `crewkit init` to set up your project
2. **Start Coding**: Run `crewkit code` to launch Claude Code with synced agents
3. **Get Help**: Run `crewkit --help` to see all available commands

---

## Getting Help

- [Troubleshooting Guide](troubleshooting.md)
- [FAQ](faq.md)
- [Report an Issue](https://github.com/karibew/crewkit-cli/issues)
