# Sentry Error Monitoring

## Setup

**Rails API** (`api/config/initializers/sentry.rb`):
```bash
SENTRY_DSN=<rails-dsn>
SENTRY_TRACES_SAMPLE_RATE=0.1
SENTRY_PROFILES_SAMPLE_RATE=0.1
```

**CLI** (`cli/src/lib/sentry.ts`):
```bash
SENTRY_DSN=<cli-dsn>
SENTRY_TRACES_SAMPLE_RATE=0.1
```

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
