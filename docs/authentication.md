# Authentication Guide

crewkit uses OAuth 2.1 device flow for secure authentication. No passwords are stored on your machine.

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

You'll see output like:
```
Please visit: https://crewkit.io/device
Enter code: ABCD-EFGH

Waiting for authentication...
```

### Step 2: Authenticate in Browser

1. Open the URL shown: `https://crewkit.io/device`
2. Enter the code: `ABCD-EFGH`
3. Sign in with your crewkit.io account
4. Authorize the CLI

### Step 3: Success

Once authorized, the CLI will automatically:
- Receive access token
- Store it securely in your OS keychain
- Display success message

```
✓ Authentication successful!
Logged in as: you@example.com
Organization: acme-corp
```

---

## Token Storage

Tokens are stored securely:
- **macOS**: Keychain
- **Linux**: Secret Service (GNOME Keyring, KWallet)
- **Windows**: Credential Manager

**Tokens are NEVER stored in plain text files.**

---

## Check Authentication Status

```bash
crewkit auth status
```

Output when authenticated:
```
✓ Authenticated

User: you@example.com
Organization: acme-corp
Role: senior
Token expires: in 45 minutes
```

Output when not authenticated:
```
✗ Not authenticated

Run `crewkit auth login` to get started.
```

---

## Logout

```bash
crewkit auth logout
```

This will:
- Remove tokens from OS keychain
- Revoke the access token on the server
- Clear any cached data

---

## Token Expiry

- **Access tokens**: Expire after 1 hour
- **Refresh tokens**: Automatically renew access tokens
- **Auto-refresh**: Happens transparently

If your access token expires, the CLI will automatically:
1. Use refresh token to get new access token
2. Update keychain
3. Continue operation

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

### Keychain Access Denied (macOS)

If you get keychain permission errors:

1. Open **Keychain Access.app**
2. Search for "crewkit"
3. Right-click → Get Info
4. Access Control tab → Add `crewkit` to "Always allow access"

### Secret Service Not Available (Linux)

If you get "No secret service available":

```bash
# Ubuntu/Debian
sudo apt-get install gnome-keyring

# Fedora
sudo dnf install gnome-keyring

# Arch
sudo pacman -S gnome-keyring
```

Start the keyring daemon:
```bash
eval $(gnome-keyring-daemon --start)
export $(gnome-keyring-daemon --start)
```

### Token Refresh Fails

If token refresh fails:

1. Log out: `crewkit auth logout`
2. Clear cache: `rm -rf ~/.cache/crewkit`
3. Log in again: `crewkit auth login`

### Browser Doesn't Open

The CLI doesn't automatically open your browser. Manually:

1. Copy the URL from terminal
2. Open it in your browser
3. Enter the code shown
4. Return to terminal

---

## Security Best Practices

### Do Not Share Tokens

- Never share your device code with anyone
- Never commit tokens to git
- Never post tokens in issues/forums

### Revoke Lost Tokens

If you lose access to a machine:

1. Log in to https://crewkit.io
2. Go to Settings → Security
3. Revoke access for lost devices

### Use Separate Accounts

For work vs personal projects:

- Use different crewkit.io accounts
- Different organizations
- Log out/in when switching

---

## API Access

All API requests include:
- `Authorization: Bearer <access_token>` header
- Automatic token refresh
- Request retries on auth failures

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
