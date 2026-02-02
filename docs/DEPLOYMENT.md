---
doc_tier: 2
doc_type: guide
doc_status: active
created: 2025-10-01
last_reviewed: 2026-01-06
owner: platform-team
tags: [deployment, digitalocean, operations]
---

# Deployment Guide

## Environment Variables for DigitalOcean

### Staging Environment

Set these environment variables in your DigitalOcean App Platform staging environment:

```bash
# Required - Rails secret key base
SECRET_KEY_BASE=3c60a5b824fa545aceba7253d09fa69fef5d456df03c22d96ca9d251a17059cf6cc252b1e73cbc80e09c5fb23f5a3cad8feb8a44ee19ce5a7f002de8e313242c

# Required - Devise JWT secret key
DEVISE_JWT_SECRET_KEY=8868aacec68190a6f129effae102227e762f9dad720a1d7731d0e3111ac06b6fee040e2696f8c3dd2ce401f258b45643496df7fb0c336911dc83593d4539600f

# Required - Rails environment
RAILS_ENV=production

# Required - Database URL (automatically set by DigitalOcean if using their database)
DATABASE_URL=postgresql://user:password@host:port/database

# Optional - Sentry DSN (for error tracking)
# SENTRY_DSN=your_sentry_dsn_here

# Optional - SMTP settings (if using email)
# SMTP_ADDRESS=smtp.example.com
# SMTP_PORT=587
# SMTP_DOMAIN=example.com
# SMTP_USER_NAME=your_username
# SMTP_PASSWORD=your_password
```

### Production Environment

Set these environment variables in your DigitalOcean App Platform production environment:

```bash
# Required - Rails secret key base
SECRET_KEY_BASE=3690bff334023a8e235a0b88d64a15cd88b3ad34c3a3829b16ba0417dc6e3c6b8a9b2950f1f3846ea363a2dd10b605ea0022789c2eac4000cb4a6ba1a1a9f64d

# Required - Devise JWT secret key
DEVISE_JWT_SECRET_KEY=29d9a9d5db90079bda71f5af5779f5a4eb12196cec6efb8c942e7ff5fe51fe98f6f36b0236c44dcb6f35a4ae4e629808c90a2f25bc89cba56c4dba0ce7059628

# Required - Rails environment
RAILS_ENV=production

# Required - Database URL (automatically set by DigitalOcean if using their database)
DATABASE_URL=postgresql://user:password@host:port/database

# Optional - Sentry DSN (for error tracking)
# SENTRY_DSN=your_sentry_dsn_here

# Optional - SMTP settings (if using email)
# SMTP_ADDRESS=smtp.example.com
# SMTP_PORT=587
# SMTP_DOMAIN=example.com
# SMTP_USER_NAME=your_username
# SMTP_PASSWORD=your_password
```

## How to Set Environment Variables in DigitalOcean

### Via Web Console

1. Go to your App in the DigitalOcean dashboard
2. Click on "Settings" tab
3. Scroll down to "App-Level Environment Variables" or component-specific environment variables
4. Click "Edit" and add each variable with its value
5. Click "Save" and redeploy

### Via doctl CLI

```bash
# List current environment variables
doctl apps list
doctl apps spec get YOUR_APP_ID

# Update environment variables via app spec
# Edit your app spec file and update with doctl apps update
```

## Important Notes

1. **Different secrets per environment**: Staging and production use different `SECRET_KEY_BASE` and `DEVISE_JWT_SECRET_KEY` values for security isolation
2. **Never commit secrets**: These values should NEVER be committed to git
3. **DATABASE_URL**: DigitalOcean automatically sets this if you use their managed database service
4. **RAILS_ENV**: Must be set to `production` for both staging and production environments (Rails doesn't have a staging environment by default)

## Security Best Practices

- Rotate secrets periodically
- Use different secrets for each environment
- Enable 2FA on your DigitalOcean account
- Restrict access to production environment variables
- Monitor Sentry for security-related errors

## Troubleshooting

### Missing secret_key_base error
```
ArgumentError: Missing `secret_key_base` for 'production' environment
```
**Solution**: Ensure `SECRET_KEY_BASE` environment variable is set in DigitalOcean

### JWT authentication not working
**Solution**: Verify `DEVISE_JWT_SECRET_KEY` is set correctly

### Database connection errors
**Solution**: Check `DATABASE_URL` is correctly formatted and the database is accessible

## Regenerating Secrets

If you need to regenerate secrets (e.g., after a security incident):

```bash
cd ~/code/karibew/crewkit/api

# Generate new secret_key_base
bin/rails secret

# Generate new JWT secret
bin/rails secret
```

Then update the environment variables in DigitalOcean and redeploy.

**Warning**: Changing `SECRET_KEY_BASE` will invalidate all existing sessions. Changing `DEVISE_JWT_SECRET_KEY` will invalidate all JWT tokens.

## Build Performance Optimizations

The Dockerfile has been optimized for faster builds on DigitalOcean:

### Docker BuildKit Cache Mounts

The Dockerfile uses BuildKit cache mounts to speed up builds:

1. **Bundle install cache** (`--mount=type=cache,target=/usr/local/bundle/cache`)
   - Caches downloaded gems between builds
   - Only re-downloads when Gemfile/Gemfile.lock changes
   - Saves 2-5 minutes on typical builds

2. **Assets precompilation cache** (`--mount=type=cache,target=/rails/tmp/cache/assets`)
   - Caches compiled assets between builds
   - Speeds up asset precompilation significantly
   - Only recompiles changed assets

### Layer Caching Strategy

The Dockerfile is structured to maximize layer caching:

```
1. Base image (ruby:3.3.0-slim) - Rarely changes
2. System packages - Rarely changes
3. Gemfile + bundle install - Only invalidated when dependencies change
4. Application code - Changes frequently (but doesn't invalidate above layers)
5. Bootsnap precompile - Fast, based on app code
6. Asset precompilation - Cached via BuildKit mount
```

### Build Context Optimization

The `.dockerignore` file excludes unnecessary files from the build context:

- Test files (`/test/`, `/spec/`)
- Documentation files (`*.md`)
- Development files (`.devcontainer`, editor configs)
- Local databases (`*.sqlite3`)
- Git history (`/.git/`)

This reduces the size of the build context sent to DigitalOcean, speeding up the initial upload.

### Expected Build Times

With these optimizations:

- **Cold build** (no cache): 8-12 minutes
- **Warm build** (with cache, no Gemfile changes): 3-5 minutes
- **Hot build** (only code changes): 2-3 minutes

### Monitoring Build Performance

Check build logs in DigitalOcean:

```bash
# Get latest deployment
doctl apps list-deployments bf2708bb-5c0d-4500-9522-901e19db7af6

# View build logs for a specific deployment
doctl apps logs bf2708bb-5c0d-4500-9522-901e19db7af6 --deployment <deployment-id> --type build
```

### Further Optimizations

If builds are still slow, consider:

1. **Reduce gem dependencies** - Audit Gemfile for unused gems
2. **Pre-built base image** - Create a custom base image with common gems pre-installed
3. **Parallel builds** - DigitalOcean App Platform automatically uses buildkit for parallel layer builds
4. **Asset optimization** - Minimize JavaScript/CSS before precompilation
