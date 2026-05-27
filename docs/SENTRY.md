---
doc_tier: 2
doc_type: guide
doc_status: active
created: 2025-10-01
last_reviewed: 2026-01-06
owner: platform-team
tags: [sentry, monitoring, operations]
---

# Sentry Error Monitoring

## Setup

**Rails API** (`api/config/initializers/sentry.rb`):
```bash
SENTRY_DSN=<rails-dsn>
SENTRY_TRACES_SAMPLE_RATE=0.1
SENTRY_PROFILES_SAMPLE_RATE=0.1
```

**CLI** (`cli/src/main.rs` — `init_sentry()`, via the `sentry` Rust crate):
The CLI's DSN is compiled in; release tracking uses `sentry::release_name!()`.
No env vars are required for the CLI.

## GitHub Actions Release Tracking

1. Create Sentry auth token (Settings → Auth Tokens)
   - Scopes: `project:releases`, `org:read`
2. Add to GitHub Secrets: `SENTRY_AUTH_TOKEN`
3. Releases auto-created on `git push --tags`

## Testing

**Rails**:
```ruby
Sentry.capture_message("Test from Rails API")
```

**CLI**:
```bash
crewkit some-invalid-command  # Errors auto-captured
```
