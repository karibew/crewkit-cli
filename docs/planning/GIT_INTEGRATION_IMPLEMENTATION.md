---
doc_tier: 1
doc_type: plan
doc_status: implemented
created: 2025-11-01
last_reviewed: 2026-01-06
owner: platform-team
tags: [git, integration, api]
---

# Git Integration Implementation Plan

> **Phase**: 4 (from PLAN.md)
> **Status**: Implemented

## Summary

Implement Vercel-style GitHub integration for crewkit enabling:
- Repository browsing and project import from GitHub
- Webhook-driven event notifications
- User identity mapping for commit attribution

## Architecture

### Two-Tier Model

```
┌─────────────────────────────────────────────────────────────────────┐
│  Organization Level: GitHub App Installation                        │
│  - Admin installs crewkit GitHub App                                │
│  - Grants access to selected repositories                           │
│  - Receives webhooks (push, PR, installation events)                │
│  - One installation per org → GitHub org mapping                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  User Level: GitHub OAuth                                           │
│  - Users connect their GitHub account                               │
│  - Identity verification (who authored a commit?)                   │
│  - Required for user-context actions                                │
│  - Enables future "Login with GitHub" flow                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Database Schema

```sql
-- Organization-level GitHub App installations
CREATE TABLE git_integrations (
  id BIGSERIAL PRIMARY KEY,
  organization_id BIGINT NOT NULL REFERENCES organizations(id),
  provider VARCHAR NOT NULL,  -- 'github', 'gitlab', 'bitbucket'
  provider_id VARCHAR NOT NULL,  -- GitHub installation ID
  provider_name VARCHAR,  -- GitHub org name
  provider_url VARCHAR,
  access_token VARCHAR,  -- Encrypted
  token_expires_at TIMESTAMP,
  permissions JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  webhook_secret VARCHAR,  -- Encrypted
  external_id UUID NOT NULL UNIQUE,
  status INTEGER DEFAULT 0,  -- active/suspended/revoked
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  UNIQUE(organization_id, provider)
);

-- User-level OAuth for identity
CREATE TABLE user_git_connections (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  provider VARCHAR NOT NULL,
  provider_user_id VARCHAR NOT NULL,
  provider_username VARCHAR,
  provider_email VARCHAR,
  provider_avatar_url VARCHAR,
  access_token VARCHAR,  -- Encrypted
  refresh_token VARCHAR,  -- Encrypted
  token_expires_at TIMESTAMP,
  scopes JSONB DEFAULT '[]',
  external_id UUID NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  UNIQUE(user_id, provider),
  UNIQUE(provider, provider_user_id)
);

-- Project to repository links
CREATE TABLE project_repositories (
  id BIGSERIAL PRIMARY KEY,
  project_id BIGINT NOT NULL REFERENCES projects(id),
  git_integration_id BIGINT NOT NULL REFERENCES git_integrations(id),
  provider VARCHAR NOT NULL,
  repo_id VARCHAR NOT NULL,
  repo_full_name VARCHAR NOT NULL,
  repo_url VARCHAR,
  default_branch VARCHAR DEFAULT 'main',
  webhooks_enabled BOOLEAN DEFAULT true,
  settings JSONB DEFAULT '{}',
  external_id UUID NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  UNIQUE(project_id, provider)
);

-- Webhook event log
CREATE TABLE git_webhook_events (
  id BIGSERIAL PRIMARY KEY,
  git_integration_id BIGINT NOT NULL REFERENCES git_integrations(id),
  project_repository_id BIGINT REFERENCES project_repositories(id),
  provider VARCHAR NOT NULL,
  event_type VARCHAR NOT NULL,
  delivery_id VARCHAR UNIQUE,
  payload JSONB NOT NULL,
  status VARCHAR DEFAULT 'pending',  -- pending/processed/failed
  error_message TEXT,
  processed_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

### API Endpoints

```
# GitHub App Flow
GET  /api/v1/git/github/install                    # Redirect to GitHub App
GET  /api/v1/git/github/callback                   # Handle installation
POST /api/v1/git/github/webhooks                   # Receive webhooks

# Organization Git Integrations
GET  /api/v1/:org_id/git-integrations              # List integrations
GET  /api/v1/:org_id/git-integrations/:provider    # Show integration
DELETE /api/v1/:org_id/git-integrations/:provider  # Disconnect

# Repository Browsing
GET  /api/v1/:org_id/git-integrations/:provider/repos           # List repos
GET  /api/v1/:org_id/git-integrations/:provider/repos/:id       # Show repo
GET  /api/v1/:org_id/git-integrations/:provider/repos/:id/branches

# Project Repository Linking
GET    /api/v1/:org_id/projects/:id/repository     # Get linked repo
POST   /api/v1/:org_id/projects/:id/repository     # Link repo
DELETE /api/v1/:org_id/projects/:id/repository     # Unlink repo
POST   /api/v1/:org_id/projects/import             # Create from repo

# User Git Connections
GET    /api/v1/user/git-connections                # List connections
GET    /api/v1/user/git-connections/:provider/authorize
GET    /api/v1/user/git-connections/:provider/callback
DELETE /api/v1/user/git-connections/:provider      # Disconnect
```

## Implementation Schedule

### Week 1: Database Foundation

**Objective**: Create schema and models

| Day | Task | Files |
|-----|------|-------|
| 1-2 | Create 4 migrations | `api/db/migrate/` |
| 3-4 | Create 4 models with encryption, Paper Trail | `api/app/models/` |
| 5 | Tests, fixtures, add octokit gem | `api/test/`, `api/Gemfile` |

**Deliverables**:
- [ ] Migrations run successfully
- [ ] Models with validations, associations
- [ ] Token encryption working
- [ ] Paper Trail integration
- [ ] Model test coverage

### Week 2: GitHub Provider & Installation

**Objective**: Build GitHub API integration and installation flow

| Day | Task | Files |
|-----|------|-------|
| 1-2 | GitProviders::BaseProvider, GitHubProvider | `api/app/services/git_providers/` |
| 3-4 | GitHubController (install, callback, webhooks) | `api/app/controllers/api/v1/git/` |
| 5 | Background jobs for webhooks and token refresh | `api/app/jobs/` |

**Key Implementation Details**:

```ruby
# GitHub App JWT generation (10min expiry)
def generate_jwt
  private_key = OpenSSL::PKey::RSA.new(credentials.app_private_key)
  payload = {
    iat: Time.now.to_i,
    exp: Time.now.to_i + 600,
    iss: credentials.app_id
  }
  JWT.encode(payload, private_key, 'RS256')
end

# Webhook signature verification
def verify_webhook_signature(payload, signature)
  expected = "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', webhook_secret, payload)}"
  Rack::Utils.secure_compare(expected, signature)
end
```

**Deliverables**:
- [ ] GitHub provider with Octokit
- [ ] Installation flow working
- [ ] Webhooks received and verified
- [ ] Token auto-refresh
- [ ] Pundit policy for GitIntegration

### Week 3: Repository Browsing & Linking

**Objective**: Enable repo listing and project import

| Day | Task | Files |
|-----|------|-------|
| 1-2 | GitRepositoryService | `api/app/services/git_repository_service.rb` |
| 3 | RepositoriesController, RepositoryController | `api/app/controllers/` |
| 4 | UserGitConnectionService and controller | `api/app/services/`, `api/app/controllers/` |
| 5 | Serializers, policies, comprehensive tests | `api/app/serializers/`, `api/app/policies/` |

**Deliverables**:
- [ ] Repo listing with search
- [ ] Branch listing
- [ ] Project import from GitHub
- [ ] Project-repo linking
- [ ] User OAuth flow

### Week 4: Dashboard Integration

**Objective**: Build frontend UI

| Day | Task | Files |
|-----|------|-------|
| 1-2 | API client, hooks, types | `dashboard/src/lib/api/`, `dashboard/src/hooks/` |
| 3 | Integrations settings page | `dashboard/src/app/kit/settings/integrations/` |
| 4 | Project import flow | `dashboard/src/app/kit/projects/import/` |
| 5 | Polish, E2E tests | Various |

**Deliverables**:
- [ ] Integrations settings page
- [ ] GitProviderCard component
- [ ] RepoSelector component
- [ ] Import project flow
- [ ] Updated sidebar navigation
- [ ] E2E test coverage

## Security Considerations

### Token Encryption

All tokens encrypted at rest using Rails 7+ built-in encryption:

```ruby
class GitIntegration < ApplicationRecord
  encrypts :access_token
  encrypts :webhook_secret
end
```

### Webhook Verification

Every incoming webhook verified using HMAC-SHA256:

```ruby
def verify_signature(payload, signature)
  expected = "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', secret, payload)}"
  Rack::Utils.secure_compare(expected, signature)
end
```

### Scope Minimization

| Integration Type | Scopes |
|-----------------|--------|
| GitHub App | `contents:read`, `metadata:read` |
| User OAuth | `read:user`, `user:email` |

### Access Control

- Only org admins/owners can install GitHub App
- Pundit policies enforce permission checks
- All changes tracked via Paper Trail

## Error Handling

### GitHub API Unavailable

```ruby
rescue Octokit::Error, Faraday::TimeoutError => e
  Rails.logger.error("GitHub API unavailable: #{e.message}")
  Sentry.capture_exception(e)
  ServiceResult.failure(["GitHub API is currently unavailable"])
end
```

### Token Refresh Failure

- Retry with exponential backoff (3 attempts)
- After max retries, mark integration as `suspended`
- Alert via Sentry

### Webhook Processing Failure

- Event remains in `failed` status
- Error message stored for debugging
- Manual retry possible via admin interface (future)

## Prerequisites

### GitHub App Registration

Before Week 2, manually register GitHub App at https://github.com/settings/apps:

1. **App name**: `crewkit` (prod) or `crewkit-dev` (dev)
2. **Homepage URL**: `https://crewkit.io`
3. **Callback URL**: `https://api.crewkit.io/api/v1/git/github/callback`
4. **Webhook URL**: `https://api.crewkit.io/api/v1/git/github/webhooks`
5. **Webhook secret**: Generate with `SecureRandom.hex(32)`
6. **Permissions**:
   - Repository: Contents (Read), Metadata (Read)
7. **Subscribe to events**: Push, Pull Request, Installation
8. Generate private key and download

### Rails Credentials

Add to `api/config/credentials.yml.enc`:

```yaml
github:
  app_id: "123456"
  app_slug: "crewkit"
  app_private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
  oauth_client_id: "Iv1.xxxxxxxx"
  oauth_client_secret: "xxxxx"
  webhook_secret: "xxxxx"
```

### Local Development

For webhook testing, use [smee.io](https://smee.io):

```bash
npx smee -u https://smee.io/YOUR_CHANNEL -t http://localhost:3050/api/v1/git/github/webhooks
```

## Dependencies

### API (Gemfile)

```ruby
gem 'octokit', '~> 9.0'  # GitHub API client
# jwt gem already present for device auth
```

### Dashboard (package.json)

No new dependencies required - uses existing TanStack Query, shadcn/ui patterns.

## Related Documents

- **PRD**: `/Users/felixp/.claude/plans/lazy-hopping-meerkat.md`
- **PLAN.md**: Phase 4 section
- **Security Model**: `api/docs/SECURITY_MODEL.md`

## Future Enhancements (Not in Scope)

- GitLab provider implementation
- Bitbucket provider implementation
- Agent sync from `.claude/` in repo on push
- PR agent review integration
- Branch-specific agent configurations