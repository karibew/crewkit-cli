# crewkit - Feature Specification

> AI Agent Management Platform for Engineering Teams
> CLI-first platform with role-based configs, session tracking, and continuous improvement.

**Stack**: Rails 8.0 API + Next.js Dashboard + Rust CLI + PostgreSQL

---

## Table of Contents

1. [Authentication](#1-authentication)
2. [Organizations & Projects](#2-organizations--projects)
3. [Resources & Agents](#3-resources--agents)
4. [Session Analytics](#4-session-analytics)
5. [Playbooks & Conventions](#5-playbooks--conventions)
6. [Experiments (A/B Testing)](#6-experiments-ab-testing)
7. [Briefs (LLM Prompts)](#7-briefs-llm-prompts)
8. [Git Integration](#8-git-integration)
9. [Admin Features](#9-admin-features)
10. [CLI Features](#10-cli-features)
11. [Dashboard Features](#11-dashboard-features)

---

## 1. Authentication

Multi-method passwordless authentication with enterprise security.

### 1.1 Device Flow (OAuth 2.1)

CLI authentication without passwords. User initiates from terminal, completes in browser.

**Endpoints:**
- `POST /api/v1/auth/device` - Initiate device code flow
- `POST /api/v1/auth/token` - Poll for token using device code
- `POST /api/v1/auth/device/verify` - JWT-authenticated verification

**Flow:**
1. CLI calls `/auth/device` → receives `device_code` + `user_code`
2. User visits dashboard with `user_code`
3. CLI polls `/auth/token` until verified
4. Returns JWT access token + refresh token

**Security:**
- Device codes expire in 10 minutes
- Atomic claim mechanism prevents replay attacks
- 5-second polling interval

### 1.2 Passkeys (WebAuthn/FIDO2)

Phishing-resistant biometric authentication.

**Endpoints:**
- `POST /api/v1/auth/passkey/login/challenge` - Get login challenge
- `POST /api/v1/auth/passkey/login` - Complete passkey login
- `POST /api/v1/auth/passkey/register/challenge` - Get registration challenge
- `POST /api/v1/auth/passkey/register` - Complete passkey registration
- `GET /api/v1/auth/passkeys` - List user's passkeys
- `DELETE /api/v1/auth/passkeys/:id` - Remove passkey

### 1.3 Magic Links

Email-based passwordless login for dashboard.

**Endpoints:**
- `POST /api/v1/auth/magic_link` - Request magic link email
- `POST /api/v1/auth/magic_link/verify` - Verify magic link token

### 1.4 Token Management

JWT-based access with refresh rotation.

**Endpoints:**
- `POST /api/v1/auth/refresh` - Refresh access token
- `POST /api/v1/auth/revoke` - Revoke token
- `DELETE /api/v1/auth/logout` - Alias for revoke

**Token Specs:**
- Access tokens: 4-hour expiration
- Refresh tokens: Rotation strategy (one-time use)
- JWT denylist for revocation

### 1.5 User Profile

**Endpoints:**
- `GET /api/v1/auth/me` - Get current user
- `PATCH /api/v1/auth/me` - Update profile
- `DELETE /api/v1/auth/me` - Delete account
- `POST /api/v1/auth/complete_profile` - Complete onboarding
- `GET /api/v1/auth/organizations` - User's organizations
- `GET /api/v1/auth/activity` - Login activity history

### 1.6 Platform Roles

Three-tier role system:
- **crewkit_owner** - Full system access
- **crewkit_staff** - Support/admin operations
- **org_user** - Regular user (default)

---

## 2. Organizations & Projects

Multi-tenant hierarchy with row-level security.

### 2.1 Organizations

**Endpoints:**
- `POST /api/v1/organizations` - Create organization
- `GET /api/v1/organizations` - List organizations
- `GET /api/v1/organizations/:id` - Get organization
- `PATCH /api/v1/organizations/:id` - Update organization
- `DELETE /api/v1/organizations/:id` - Soft delete
- `GET /api/v1/organizations/slug-available` - Check slug availability

**Features:**
- Slug-based URL params (external_id)
- Soft deletes with Paper Trail audit
- Multi-region support (NA, EU endpoints)
- Playbook subscriptions

### 2.2 Projects

**Endpoints:**
- `POST /api/v1/:organization_id/projects` - Create project
- `GET /api/v1/:organization_id/projects` - List projects
- `GET /api/v1/:organization_id/projects/:id` - Get project
- `PATCH /api/v1/:organization_id/projects/:id` - Update project
- `DELETE /api/v1/:organization_id/projects/:id` - Delete project
- `GET /api/v1/:organization_id/projects/slug-available` - Check slug
- `POST /api/v1/projects/resolve` - Resolve project from git repo

**Features:**
- Auto-detected framework stacks (Rails, Next.js, etc.)
- Manual stack override
- Playbook assignments
- Agent assignments

### 2.3 Members & Invitations

**Member Management:**
- `GET /api/v1/:organization_id/members` - List members
- `PATCH /api/v1/:organization_id/members/:user_id` - Update role
- `DELETE /api/v1/:organization_id/members/:user_id` - Remove member
- `POST /api/v1/:organization_id/members/:user_id/transfer_ownership` - Transfer ownership

**Invitations:**
- `POST /api/v1/:organization_id/members/invitations` - Invite user
- `GET /api/v1/:organization_id/members/invitations` - List pending
- `POST /api/v1/:organization_id/members/invitations/:token/resend` - Resend
- `DELETE /api/v1/:organization_id/members/invitations/:token` - Cancel
- `GET /api/v1/invitations/:token` - View invitation (public)
- `POST /api/v1/invitations/:token/accept` - Accept
- `POST /api/v1/invitations/:token/decline` - Decline

### 2.4 Organization Roles

Flexible per-organization role system.

**Endpoints:**
- `GET /api/v1/:organization_id/roles` - List roles
- `POST /api/v1/:organization_id/roles` - Create role
- `PATCH /api/v1/:organization_id/roles/:id` - Update role
- `DELETE /api/v1/:organization_id/roles/:id` - Delete role

**Role Attributes:**
- `key` - Unique identifier (junior, senior, admin)
- `level` - Numeric hierarchy (1=junior, 2=senior, 3=admin)
- `custom_name` - Display name
- `coaching_mode_enabled` - Enables guided learning mode

**Built-in Roles:**
- **Junior** (level 1) - Coaching mode enabled, guided suggestions
- **Senior** (level 2) - Collaborative mode, balanced autonomy
- **Admin** (level 3) - Autonomous mode, full control

---

## 3. Resources & Agents

Unified resource model with 3-tier inheritance.

### 3.1 Resource Types

| Type | Description |
|------|-------------|
| `agent` | AI agent configurations |
| `skill` | Reusable capabilities |
| `command` | CLI slash commands |
| `tool` | Integration tools |
| `workflow` | Automation workflows |
| `playbook` | Team conventions |

### 3.2 Ownership Tiers

Three-level configuration inheritance:

```
Platform (base)
    ↓ inherits
Organization (overrides)
    ↓ inherits
Project (overrides)
```

**Ownership Modes:**
- `inherit` - Inherits from parent, stores only overrides
- `standalone` - Fully owned, no parent
- `forked` - Snapshot of another resource, independent after fork

### 3.3 Resource CRUD

**Platform Resources:**
- `GET /api/v1/resources` - List platform resources
- `GET /api/v1/resources/:id` - Get resource
- `POST /api/v1/resources` - Create (staff only)
- `PATCH /api/v1/resources/:id` - Update

**Organization Resources:**
- `GET /api/v1/:organization_id/resources` - List org resources
- `POST /api/v1/:organization_id/resources` - Create org resource
- `POST /api/v1/:organization_id/resources/import` - Import local files

### 3.4 Effective Configuration

Resolution of merged configs across tiers.

**Endpoints:**
- `GET /api/v1/resources/effective` - Bulk effective configs (CLI sync)
- `GET /api/v1/resources/:id/effective` - Single resource effective
- `GET /api/v1/resources/:id/tiered` - View inheritance chain

**Resolution Order:**
1. Platform base resource
2. Organization overrides (if exists)
3. Project overrides (if exists)
4. Role modifiers (coaching mode, etc.)

### 3.5 Resource Operations

**Fork & Publish:**
- `POST /api/v1/resources/:id/fork` - Fork to standalone
- `POST /api/v1/resources/:id/publish` - Publish to marketplace

### 3.6 Version Management

**Endpoints:**
- `GET /api/v1/resources/:id/versions` - List versions
- `GET /api/v1/resources/:id/versions/:hash` - Get version
- `GET /api/v1/resources/:id/diff/:from/:to` - Compare versions
- `POST /api/v1/resources/:id/versions/rollback` - Rollback
- `GET /api/v1/resources/:id/versions/upgrade_available` - Check upgrade
- `POST /api/v1/resources/:id/versions/upgrade` - Upgrade to parent

**Version Tracking:**
- SHA-256 content hash
- Full content snapshots
- Change summaries
- Created by user

### 3.7 Resource Statistics

**Endpoints:**
- `GET /api/v1/resources/:id/stats` - Resource usage stats
- `GET /api/v1/resources/:id/stats/compare` - Compare versions
- `GET /api/v1/stats/resources` - Platform-wide (staff)
- `GET /api/v1/:organization_id/stats/resources` - Org stats

### 3.8 A/B Testing (Benchmarks)

**Endpoints:**
- `GET /api/v1/resources/:id/benchmarks` - List benchmarks
- `POST /api/v1/resources/:id/benchmarks` - Create benchmark
- `POST /api/v1/resources/:id/benchmarks/:id/complete` - Mark complete
- `GET /api/v1/resources/:id/benchmarks/:id/version` - Get benchmarked version

### 3.9 Project Agents

Assign agents to specific projects.

**Endpoints:**
- `GET /api/v1/:organization_id/projects/:id/agents` - List assigned
- `POST /api/v1/:organization_id/projects/:id/agents` - Assign agent
- `DELETE /api/v1/:organization_id/projects/:id/agents/:agent_id` - Remove

---

## 4. Session Analytics

Comprehensive analytics with KPIs and time-series.

### 5.1 Analytics Endpoints

```
GET /api/v1/:org_id/sessions/analytics/summary       # KPIs
GET /api/v1/:org_id/sessions/analytics/timeseries    # Time-series data
GET /api/v1/:org_id/sessions/analytics/by-agent      # Stats by agent
GET /api/v1/:org_id/sessions/analytics/by-user       # Stats by user
GET /api/v1/:org_id/sessions/analytics/by-project    # Stats by project
GET /api/v1/:org_id/sessions/analytics/cost-breakdown # Cost analysis
GET /api/v1/:org_id/sessions/analytics/success-trend  # Success rate trend
GET /api/v1/:org_id/sessions/analytics/coaching       # Coaching effectiveness
```

### 5.2 KPI Summary

Returns key performance indicators:
- Total sessions, active sessions
- Success rate, failure rate
- Total cost, average cost per session
- Total tokens (input/output/cache)
- Average duration
- Lines of code changed
- Commits, pull requests

### 5.3 Time-series Data

Aggregated by period (hourly, daily, weekly, monthly):
- Session count over time
- Token usage trends
- Cost trends
- Success rate trends

### 5.4 Coaching Effectiveness

Metrics for coaching mode:
- Sessions with coaching vs. without
- Success rate comparison
- Time to completion
- User progression

### 5.5 Session Insights

AI-generated insights per session:
- Performance summary
- Coaching tips (personalized guidance)
- Actionable recommendations
- Efficiency metrics

---

## 6. Playbooks & Conventions

Framework-aware development standards and team conventions.

### 6.1 Framework Stacks

Reference data for framework detection.

**Endpoints:**
- `GET /api/v1/framework_stacks` - List stacks
- `GET /api/v1/framework_stacks/:slug` - Get stack

**Supported Stacks:**
- Rails + Hotwire
- Next.js
- TypeScript/JavaScript
- Rust
- Python/Django
- And more...

### 6.2 Playbooks

Team conventions and best practices.

**Endpoints:**
- `GET /api/v1/:organization_id/playbooks/for_stacks` - Get playbooks for detected stacks

**Features:**
- 2-tier inheritance (Platform → Organization)
- Cached by framework stack
- Contains multiple conventions

### 6.3 Playbook Subscriptions

Marketplace for subscribing to playbooks.

**Endpoints:**
- `GET /api/v1/:org_id/playbook_subscriptions` - List subscriptions
- `GET /api/v1/:org_id/playbook_subscriptions/available` - Available playbooks
- `POST /api/v1/:org_id/playbook_subscriptions` - Subscribe
- `DELETE /api/v1/:org_id/playbook_subscriptions/:id` - Unsubscribe
- `POST /api/v1/:org_id/playbook_subscriptions/:id/enable` - Enable
- `POST /api/v1/:org_id/playbook_subscriptions/:id/disable` - Disable
- `GET /api/v1/:org_id/playbook_subscriptions/:id/stats` - Usage stats

### 6.4 Conventions

Individual rules within playbooks.

**Endpoints:**
- `GET /api/v1/playbooks/:playbook_id/conventions` - List conventions
- `GET /api/v1/playbooks/:playbook_id/conventions/effective` - Merged conventions
- `POST /api/v1/playbooks/:playbook_id/conventions` - Create convention
- `PATCH /api/v1/playbooks/:playbook_id/conventions/:id` - Update
- `DELETE /api/v1/playbooks/:playbook_id/conventions/:id` - Delete
- `POST /api/v1/playbooks/:playbook_id/conventions/challenge` - Challenge deviation
- `POST /api/v1/playbooks/:playbook_id/conventions/import_from_markdown` - Import

**Convention Structure:**
```yaml
category: "testing"
key: "test_framework"
value: "minitest"
rationale: "Rails default, simpler than RSpec"
anti_patterns:
  - "rspec"
  - "rspec-rails"
challenge_severity: "warning"  # info, warning, error
alert_channels:
  - "slack"
  - "email"
```

### 6.5 Convention Extraction

AI-powered convention extraction.

**Endpoint:**
- `POST /api/v1/playbooks/extract_convention` - Extract from code

### 6.6 Convention Overrides

Track when developers deviate from conventions.

**Endpoints:**
- `GET /api/v1/:org_id/observability/convention_overrides` - List overrides
- `POST /api/v1/:org_id/observability/convention_overrides` - Record override
- `GET /api/v1/:org_id/observability/convention_overrides/analytics` - Analytics

### 6.7 Convention Alerts

Notifications when conventions are overridden.

**Endpoints:**
- `GET /api/v1/:org_id/observability/convention_alert_configs` - List configs
- `POST /api/v1/:org_id/observability/convention_alert_configs` - Create
- `PATCH /api/v1/:org_id/observability/convention_alert_configs/:id` - Update
- `DELETE /api/v1/:org_id/observability/convention_alert_configs/:id` - Delete
- `POST /api/v1/:org_id/observability/convention_alert_configs/:id/test` - Test

**Alert Channels:**
- Webhook
- Email
- Slack

---

## 7. Experiments (A/B Testing)

Statistical comparison of agent configurations.

### 7.1 Experiment Management

**Endpoints:**
- `GET /api/v1/:organization_id/experiments` - List experiments
- `POST /api/v1/:organization_id/experiments` - Create experiment
- `GET /api/v1/:organization_id/experiments/:id` - Get experiment
- `PATCH /api/v1/:organization_id/experiments/:id` - Update
- `DELETE /api/v1/:organization_id/experiments/:id` - Delete

### 7.2 Experiment Lifecycle

**Endpoints:**
- `POST /api/v1/:org_id/experiments/:id/start` - Start experiment
- `POST /api/v1/:org_id/experiments/:id/stop` - Stop experiment
- `POST /api/v1/:org_id/experiments/:id/deploy` - Deploy winner
- `POST /api/v1/:org_id/experiments/:id/archive` - Archive

**Statuses:**
- `draft` - Created, not started
- `running` - Actively collecting data
- `completed` - Data collection complete
- `deployed` - Winner deployed
- `archived` - Archived

### 7.3 Experiment Configuration

**Attributes:**
- `agent_name` - Agent being tested
- `slug` - 3-word identifier (swift-amber-falcon)
- `control_config` - Control variant config
- `variant_config` - Test variant config

### 7.4 Experiment Assignment

Consistent per-user assignment using MD5 hash:
```
variant = MD5(experiment_id + user_id) % 2
```
- 0 = control
- 1 = variant

### 7.5 Experiment Metrics

**Endpoint:**
- `GET /api/v1/:organization_id/experiments/:id/metrics`

**Metrics:**
- Session count per variant
- Success rate comparison
- Cost efficiency
- Token usage
- Statistical significance

**Minimum Data:** 10 sessions per variant

---

## 8. Briefs (LLM Prompts)

Platform-managed prompts for AI-generated summaries and insights.

### 8.1 Brief Types

| Type | Purpose |
|------|---------|
| `subagent_summary` | Summarize subagent tasks |
| `session_summary` | Generate session summaries |
| `quality_assessment` | Evaluate session quality |
| `session_analysis` | Analyze session patterns |
| `error_detection` | Identify errors |
| `improvement_suggestion` | Suggest improvements |

### 8.2 Brief Endpoints

**Public (read-only):**
- `GET /api/v1/briefs` - List briefs
- `GET /api/v1/briefs/:slug` - Get brief
- `GET /api/v1/briefs/current?type=<type>` - Get active brief by type

### 8.3 Brief Versions

**Endpoints:**
- `GET /api/v1/briefs/:slug/versions` - List versions
- `GET /api/v1/briefs/:slug/versions/:number` - Get version

**Versioning:**
- Full content snapshots
- Content hash for change detection
- Change reason documentation
- Parameters (model settings)

### 8.4 Admin Operations

**Endpoints:**
- `POST /api/v1/admin/briefs` - Create brief
- `PATCH /api/v1/admin/briefs/:slug` - Update
- `DELETE /api/v1/admin/briefs/:slug` - Delete
- `POST /api/v1/admin/briefs/:slug/activate` - Activate
- `POST /api/v1/admin/briefs/:slug/deprecate` - Deprecate
- `POST /api/v1/admin/briefs/:slug/versions` - Create version
- `POST /api/v1/admin/briefs/:slug/versions/:number/make_current` - Set active

**Lifecycle:**
- `draft` - Not yet active
- `active` - Currently used (one per type)
- `deprecated` - No longer used

---

## 9. Git Integration

Connect organizations and projects to git repositories.

### 9.1 Git Integrations (Organization-level)

**Endpoints:**
- `GET /api/v1/:organization_id/git_integrations` - List integrations
- `GET /api/v1/:organization_id/git_integrations/:id` - Get integration
- `DELETE /api/v1/:organization_id/git_integrations/:id` - Delete
- `POST /api/v1/:organization_id/git_integrations/:id/refresh` - Refresh token

**OAuth Flow:**
- `POST /api/v1/git_integrations/:provider/initiate` - Start OAuth
- `POST /api/v1/git_integrations/:provider/callback` - OAuth callback

**Providers:**
- GitHub
- GitLab
- Bitbucket
- Manual (for git tokens)

### 9.2 Project Repositories

Link projects to specific repositories.

**Endpoints:**
- `GET /api/v1/:org_id/projects/:id/repositories` - List repos
- `GET /api/v1/:org_id/projects/:id/repositories/available` - Available repos
- `POST /api/v1/:org_id/projects/:id/repositories` - Link repo
- `DELETE /api/v1/:org_id/projects/:id/repositories/:id` - Unlink
- `POST /api/v1/:org_id/projects/:id/repositories/:id/sync` - Sync
- `POST /api/v1/:org_id/projects/:id/repositories/:id/webhook` - Register webhook

### 9.3 Webhook Events

Track git webhook events.

**Endpoints:**
- `GET /api/v1/:org_id/git_integrations/:id/webhook_events` - List events
- `GET /api/v1/:org_id/git_integrations/:id/webhook_events/:id` - Get event
- `POST /api/v1/:org_id/git_integrations/:id/webhook_events/:id/retry` - Retry

### 9.4 User Git Connections

Personal git tokens (under auth namespace).

**Endpoints:**
- `GET /api/v1/auth/git_connections` - List connections
- `GET /api/v1/auth/git_connections/:id` - Get connection
- `DELETE /api/v1/auth/git_connections/:id` - Delete
- `POST /api/v1/auth/git_connections/:provider/initiate` - Start OAuth
- `POST /api/v1/auth/git_connections/:provider/callback` - Callback

---

## 10. Admin Features

Platform administration (crewkit_staff only).

### 10.1 User Management

**Endpoints:**
- `GET /api/v1/admin/users` - List all users
- `GET /api/v1/admin/users/:id` - Get user
- `PATCH /api/v1/admin/users/:id` - Update user
- `POST /api/v1/admin/users/:id/disable` - Disable user
- `POST /api/v1/admin/users/:id/enable` - Enable user

### 10.2 Organization Management

**Endpoints:**
- `GET /api/v1/admin/organizations` - List all orgs
- `GET /api/v1/admin/organizations/:id` - Get org
- `DELETE /api/v1/admin/organizations/:id` - Hard delete
- `POST /api/v1/admin/organizations/:id/restore` - Restore soft-deleted

### 10.3 Audit Logs

**Endpoints:**
- `GET /api/v1/admin/audit_logs` - List logs
- `GET /api/v1/admin/audit_logs/:id` - Get log
- `GET /api/v1/admin/audit_logs/actions` - Available actions
- `GET /api/v1/admin/audit_logs/resource_types` - Tracked types

### 10.4 Platform Statistics

**Endpoints:**
- `GET /api/v1/admin/stats` - Platform stats
- `GET /api/v1/admin/stats/users` - User statistics
- `GET /api/v1/admin/stats/organizations` - Org statistics
- `GET /api/v1/admin/stats/activity` - Activity metrics

### 10.5 Events

**Endpoints:**
- `GET /api/v1/admin/events` - List events
- `GET /api/v1/admin/events/:id` - Get event
- `GET /api/v1/admin/events/event_types` - Event types
- `GET /api/v1/admin/events/aggregate_types` - Aggregation types

---

## 11. CLI Features

Rust CLI with TUI and headless modes.

### 11.1 Commands

| Command | Description |
|---------|-------------|
| `crewkit code` | Start Claude Code session with hooks |
| `crewkit auth login` | Device flow authentication |
| `crewkit auth status` | Check authentication |
| `crewkit auth logout` | Logout |
| `crewkit org info` | Organization information |
| `crewkit resources list` | List resources |
| `crewkit resources import` | Import local resources |
| `crewkit experiments list` | List experiments |
| `crewkit experiments metrics` | Get experiment metrics |
| `crewkit lsp install` | Install LSP plugin |

### 11.2 TUI Mode

Interactive terminal UI with ratatui:
- Real-time session status
- Token usage display
- Cost tracking
- Session resume capability

### 11.3 Headless Mode

Scripting support:
```bash
crewkit code -p "prompt" --headless
```

### 11.4 Hook System

Real-time event capture:
- SubagentStart, SubagentStop
- SessionStart, SessionEnd
- PreCompact (context compaction)
- ProcessMetrics (CPU, memory)

### 11.5 LSP Integration

Language Server Protocol support:
- **Languages:** Ruby, TypeScript, Rust, JavaScript
- **Capabilities:** documentSymbol, workspaceSymbol, goToDefinition, findReferences, callHierarchy
- **Storage:** redb database with file/symbol indexes
- Auto-detected on `crewkit code` startup

---

## 12. Dashboard Features

Next.js dashboard with shadcn/ui components.

### 12.1 Page Structure

```
/kit                                    # Dashboard home
/kit/agents                             # Agent management
/kit/projects                           # Project listing
  /kit/projects/[id]                    # Project overview (KPIs)
  /kit/projects/[id]/settings           # Project settings
  /kit/projects/[id]/conversations      # Conversation list
  /kit/projects/[id]/sessions           # Session history
  /kit/projects/[id]/threads            # Conversation threads
/kit/resources                          # Resource management
/kit/playbooks                          # Playbook marketplace
/kit/analytics                          # Organization analytics
  /kit/analytics/convention-overrides   # Convention tracking
/kit/settings                           # Organization settings
/kit/profile                            # User profile
```

### 12.2 Key Components

| Component | Purpose |
|-----------|---------|
| `kpi-grid` | Analytics KPI cards |
| `session-table` | Session listing |
| `session-token-breakdown` | Token usage visualization |
| `agent-performance-table` | Performance metrics |
| `agent-task-list` | Task listing |
| `playbook-assignment-card` | Playbook management |
| `tiered-editor` | Resource inheritance editor |
| `ai-insight-card` | AI-generated insights |

### 12.3 Data Hooks

| Hook | Purpose |
|------|---------|
| `use-sessions` | Fetch sessions |
| `use-llm-sessions` | Fetch LlmSessions |
| `use-session-hierarchy` | Session tree navigation |
| `use-agents` | Agent data |
| `use-playbooks` | Playbook data |
| `use-convention-overrides` | Convention tracking |
| `use-experiments` | Experiment data |
| `use-resources` | Resource listing |

---

## Key Services

### Session Services
- `SessionLiveStatsService` - Real-time stats calculation
- `SessionInsightsService` - AI-generated coaching tips
- `SessionConsolidationService` - Consolidate streamed items
- `SessionStreamingService` - Real-time JSONL streaming
- `HooksEventIngestionService` - Ingest CLI hook events

### Resource Services
- `ResourceConfigurationService` - Tiered inheritance resolution
- `ResourceVersionService` - Version management
- `ResourceImportService` - Import local resources

### Analytics Services
- `SessionAnalyticsService` - KPI aggregation
- `DashboardAnalyticsService` - Dashboard queries

### AI Services
- `AnthropicService` - Anthropic API client
- `AnthropicBatchService` - Batch API (50% cheaper)
- `ConventionExtractionService` - AI convention extraction

---

## Background Jobs

### Session Jobs
- `SessionConsolidationJob` - Consolidate items to S3
- `CleanupStaleSessionsJob` - Cleanup timed-out sessions
- `GenerateSessionSummaryJob` - AI summaries

### Analytics Jobs
- `RefreshAnalyticsMaterializedViewsJob` - Refresh KPI views
- `ExpireAnalyticsCachesJob` - Clear analytics cache

### Batch Processing
- `EnqueueBatchSummariesJob` - Queue for Batch API
- `ProcessBatchResultsJob` - Process completed batches

---

## Security

### Authentication Security
- Atomic device code claims
- Passkey support (WebAuthn/FIDO2)
- Magic link tokens with expiration
- JWT denylist for revocation

### Data Security
- Row-level security (organization_id scoping)
- Pundit policies for authorization
- Soft deletes with Paper Trail audit
- Encrypted sensitive fields

### API Security
- JWT access tokens (4-hour expiration)
- Refresh token rotation
- CSRF protection
- Webhook signature verification

---

## Data Flow Examples

### Session Tracking
```
CLI hooks → POST /observability/sessions/:id/events
         → HooksEventIngestionService
         → LlmSessionItem + SubagentSessionItem
         → Dashboard polls /llm_sessions/:id/items
         → Real-time stats via /sessions/:id/live
         → Consolidated to S3 after session
         → AI summary via GenerateSessionSummaryJob
```

### Resource Resolution
```
CLI sync → GET /resources/effective
        → ResourceConfigurationService
        → Platform + Org + Project + Role modifiers
        → Merged YAML returned
```

### Playbook Matching
```
CLI detects stacks → POST /projects/resolve
                   → PlaybookMatcherService
                   → Match subscribed playbooks
                   → Download conventions
                   → Inject into agent prompt
```

---

## Summary

| Category | Count |
|----------|-------|
| API Controllers | 50+ |
| Data Models | 50+ |
| Services | 45+ |
| Background Jobs | 18+ |
| CLI Commands | 12+ |
| Dashboard Pages | 15+ |
| Analytics Endpoints | 8 |
| Authentication Methods | 4 |
| Resource Types | 6 |
| Session Levels | 3 |

crewkit provides comprehensive AI agent management with:
- **Observability** - Full session tracking and analytics
- **Governance** - Role-based access and conventions
- **Continuous Improvement** - A/B testing and insights
- **Team Collaboration** - Shared resources and playbooks
