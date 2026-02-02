# crewkit Development Roadmap

> Last synced with CLAUDE.md: 2026-01-08

## ✅ Completed

### Phase 1: Security & Permissions
- [x] Multi-organization support (remove single org_id from users)
- [x] Permission hierarchy (owner/admin/manager/developer/viewer)
- [x] Privilege escalation prevention
- [x] Owner protection (can't remove last owner)
- [x] Pundit authorization policies
- [x] Paper Trail audit logging
- [x] Tenant isolation
- [x] Security documentation (SECURITY_MODEL.md)
- [x] Test fixtures updated
- [x] Security tests passing

### API Authentication (Phase 1.5)
- [x] JWT authentication with refresh tokens
- [x] Email/password login endpoint
- [x] User registration endpoint
- [x] Password reset flow
- [x] Passkey/WebAuthn support
- [x] OAuth device flow for CLI

### Dashboard Foundation
- [x] Next.js 16+ with App Router
- [x] shadcn/ui component library
- [x] Auth context with token management
- [x] API client wired to Rails backend
- [x] Login/register/forgot-password forms
- [x] TanStack Query integration

## 🚧 Next Steps

### Phase 2: API Resource Management (In Progress)
> PRD: docs/planning/api-resource-management-prd.md

#### 2.1: Organization Management API ✅
- [x] GET /api/v1/organizations/:id (show with HATEOAS links)
- [x] PATCH /api/v1/organizations/:id (update name, slug, settings)
- [x] DELETE /api/v1/organizations/:id (soft-delete with confirmation)
- [x] GET /api/v1/organizations/slug-available (check slug availability)
- [x] OrganizationManagementService (update, delete, check_slug)
- [x] HATEOAS serializer for organization responses
- [x] CLI: `crewkit org info/update/delete` commands
- [x] Dashboard: Organization settings page

#### 2.2: Project Management API ✅
- [x] GET /api/v1/:organization_id/projects (list with pagination)
- [x] POST /api/v1/:organization_id/projects (create)
- [x] GET /api/v1/:organization_id/projects/:id (show)
- [x] PATCH /api/v1/:organization_id/projects/:id (update)
- [x] DELETE /api/v1/:organization_id/projects/:id (soft-delete with confirmation)
- [x] GET /api/v1/:organization_id/projects/slug-available (check slug availability)
- [x] ProjectManagementService (CRUD operations)
- [x] ProjectsController with full CRUD
- [x] ProjectSerializer with HATEOAS links
- [x] CLI: `crewkit project list/create/info/update/delete` commands
- [x] Dashboard: Project management pages (/kit/projects/*)

#### 2.3: Member Management API ✅
- [x] GET /api/v1/organizations/:org_id/members (list members)
- [x] POST /api/v1/organizations/:org_id/members/invitations (invite)
- [x] PATCH /api/v1/organizations/:org_id/members/:user_id (update role)
- [x] DELETE /api/v1/organizations/:org_id/members/:user_id (remove)
- [x] MemberManagementService (invite, update, remove, last_owner check)
- [x] Invitations table and model
- [x] InvitationMailerJob (Solid Queue)
- [x] CLI: `crewkit members list/invite/update/remove` commands
- [x] Dashboard: Team management page

#### 2.4: Platform Admin Namespace ✅
- [x] GET /api/v1/admin/organizations (list all orgs, crewkit staff only)
- [x] GET /api/v1/admin/organizations/:id (show any org)
- [x] GET /api/v1/admin/users (list all users)
- [x] GET /api/v1/admin/users/:id (show any user)
- [x] GET /api/v1/admin/audit_logs (view Paper Trail + custom events)
- [x] GET /api/v1/admin/stats (system statistics)
- [x] Admin::BaseController with platform_role enforcement
- [x] Enhanced audit logging for admin actions
- [x] CLI: No admin commands (security - admin actions via dashboard only)
- [x] Dashboard: Admin panel (crewkit staff only)

#### 2.5: API Standards & ESB ✅
- [x] HATEOAS links in all API responses (_links with self, related)
- [x] Events table for ESB pattern
- [x] EventPublisher service (publish state changes)
- [x] Event types: organization.*, project.*, member.*, user.*
- [x] Rate limiting (Rack::Attack) per IP and per user
- [x] Standard error response format with request_id
- [x] OpenAPI spec updates for new endpoints

### Phase 3: Dashboard Integration ✅
- [x] E2E auth flow testing (dashboard ↔ API) - comprehensive test suite exists
- [x] Protected route guards on /kit/* pages - middleware.ts with cookie-based auth
- [x] Organization settings page (from 2.1)
- [x] Project management pages (from 2.2)
- [x] Team management page (from 2.3)
- [x] Admin panel for crewkit staff (from 2.4)
- [x] User profile page - /kit/profile with name update, password change, account deletion, org memberships
- [x] TanStack Query hooks for all new endpoints

### Session Infrastructure (Backend) ✅
> 3-level session hierarchy: CrewkitSession → LlmSession → SubagentSession

- [x] Session models (CrewkitSession, LlmSession, SubagentSession)
- [x] Session items (LlmSessionItem, SubagentSessionItem) for streamed JSONL data
- [x] Real-time JSONL streaming with batch idempotency
- [x] Stream status state machine (inactive → streaming → consolidating → completed)
- [x] Automatic consolidation to compressed files
- [x] Session analytics backend (8 API endpoints)
- [x] Materialized views for fast dashboard queries
- [x] Background jobs for analytics refresh
- [x] Cursor pagination for session items

### Hook System (Backend) ✅
- [x] Hook registry and settings generation
- [x] SubagentStart/SubagentStop hooks for Task tool tracking
- [x] SubagentTracker HTTP server (localhost)
- [x] JSONL stats extraction from agent transcripts
- [x] Internal hook-handler CLI command
- [x] Event buffer for batch processing

### Brief System ✅
- [x] Brief model (name, prompt_template, context_type)
- [x] BriefService for fetching applicable briefs
- [x] HaikuService for LLM calls (subagent summaries)
- [x] Session and subagent summary generation

### LSP Module (Phase 1-2) ✅
> PRD: docs/prd-lsp-module.md

- [x] LSP server scaffold (tower-lsp, stdio/TCP transport)
- [x] tree-sitter parsing (Rust, TypeScript, JavaScript, Ruby)
- [x] redb storage with SWMR for concurrent access
- [x] Symbol index (functions, classes, methods, modules, traits, interfaces)
- [x] LSP methods: documentSymbol, workspaceSymbol, goToDefinition, findReferences, hover
- [x] Plugin-based Claude Code integration
  - [x] `crewkit lsp install --project` - install plugin
  - [x] `crewkit lsp uninstall --project` - remove plugin
  - [x] `crewkit lsp status --project` - check status
- [x] Automatic plugin check on `crewkit code` startup
- [x] Conflicting plugin detection (rust-analyzer-lsp, typescript-lsp)
- [x] Inline disable prompt for conflicting plugins

## 🚧 Next Steps

### Phase 4: Session Analytics Dashboard (Frontend)
> Show value from existing backend infrastructure

- [ ] Session list page with filtering and search
- [ ] Session detail page with hierarchy visualization
- [ ] Chat log viewer with cursor pagination
- [ ] Live stats and token breakdown displays
- [ ] KPI dashboard (costs, tokens, success rates)
- [ ] Time-series charts (hourly/daily/weekly)
- [ ] Cost breakdown visualizations

### Phase 5: Role-Based Configuration & Coaching Mode
> Key differentiator for teams - combine with agent configuration

- [ ] Base agent roles seeding (coaching/collaborative/autonomous)
- [ ] GET /api/v1/organizations/:org_id/effective-role
- [ ] RoleConfigurationService (resolve coaching mode, tool restrictions)
- [ ] Add coaching_enforcement column to organizations (soft/hard)
- [ ] Agent configuration API endpoints
- [ ] Agent configuration UI (organization-level)
- [ ] Project-level agent overrides
- [ ] Dashboard: Coaching mode settings in org settings page

### Phase 6: Claude Code v2.1.0 Integration
> Leverage new Claude Code features for role-based agent control

#### 6a: Tool Restrictions by Role (Core Coaching)
- [ ] Use `--tools` flag for role-based tool access
- [ ] Use `Task(AgentName)` syntax to disable specific subagents per role
- [ ] Junior role: limited tools (Read, Grep, Glob only)
- [ ] Senior role: full tool access
- [ ] CLI: --append-system-prompt injection for coaching mode
- [ ] CLI: --allowedTools restriction for hard enforcement

#### 6b: Agent Frontmatter Hooks
- [ ] Embed telemetry hooks directly in agent markdown frontmatter
- [ ] Add PreToolUse hooks for input logging/sanitization
- [ ] Add PostToolUse hooks for tool usage tracking
- [ ] Add Stop hooks for session cleanup/summary

#### 6c: Advanced Features
- [ ] Skill hot-reload (sync to `.claude/skills/`)
- [ ] Skill frontmatter: `context: fork` for isolated execution
- [ ] PreToolUse `updatedInput` middleware pattern
- [ ] `once: true` for initialization hooks
- [ ] MCP `list_changed` for dynamic tool registration

### Phase 7: Experiments (A/B Testing)
- [ ] Experiment creation API
- [ ] 3-word slug generator (adjective-color-noun)
- [ ] Variant assignment service (consistent hashing)
- [ ] Experiment metrics collection (depends on session analytics)
- [ ] Statistical significance calculator
- [ ] Experiment deployment workflow
- [ ] Web UI for experiments
- [ ] Experiment mode: controlled tool sets for A/B testing

### Phase 8: Basic Billing (Stripe)
> Move billing earlier to enable revenue - start simple

#### 8.1: Core Billing
- [ ] Add billing columns to organizations (stripe_customer_id, billing_email)
- [ ] Create plans table with limits and Stripe price IDs
- [ ] Create subscriptions table (org → plan link)
- [ ] Billing::StripeCustomerService - create/update customers
- [ ] Billing::SubscriptionService - subscribe, upgrade, downgrade, cancel
- [ ] Webhook endpoint (POST /api/v1/webhooks/stripe)
- [ ] /kit/settings/billing - basic billing page

#### 8.2: Usage Tracking & Limits
- [ ] Create usage_records table (period-based tracking)
- [ ] Billing::UsageTrackingService - increment counters
- [ ] Billing::UsageEnforcementService - check limits
- [ ] Soft warnings at 80%, hard blocks at 100%

### Phase 9: Critical Emails
> Split from full email phase - just essentials for MVP

- [ ] SendGrid gem integration
- [ ] Password reset (existing Devise flow enhancement)
- [ ] Organization invitation
- [ ] Security alerts (login from new device)
- [ ] Payment failed notification

### Phase 10: Accuracy & Evaluation
- [ ] LLM-as-judge evaluator
- [ ] Rule-based checks
- [ ] Pattern detection service
- [ ] Auto-suggest experiments
- [ ] Accuracy dashboard

### Phase 11: MCP Server
- [ ] MCP server implementation (stdio transport, @modelcontextprotocol/sdk)
- [ ] Tools: crewkit_session_info, crewkit_list_experiments, crewkit_create_experiment, crewkit_get_agent_config, crewkit_submit_feedback
- [ ] Resources: crewkit://project/config, crewkit://session/metrics, crewkit://experiments/active
- [ ] CLI: `crewkit mcp install/uninstall/status/serve` commands
- [ ] Auto-generate .mcp.json in `crewkit init`

### Phase 12: Git Integration (Vercel-style)
> Nice-to-have for adoption, moved later in roadmap

Two-tier connection model (like Vercel):
1. **Org-level GitHub App** - Admin installs app, grants access to repos
2. **User-level OAuth** - Users connect GitHub for identity verification

- [ ] Create git_integrations, user_git_connections, project_repositories tables
- [ ] Register crewkit GitHub App (github.com/settings/apps)
- [ ] GitProviders::GitHubProvider - JWT auth, installation tokens
- [ ] Api::V1::Git::GitHubController - install, callback, webhooks
- [ ] /kit/settings/integrations - connect GitHub App
- [ ] /kit/projects/import - repo browser with search
- [ ] ProcessGitWebhookJob - handle push, PR events

### Phase 13: Extended Email & Billing
> Full email suite and advanced billing features

#### Extended Email
- [ ] Welcome email, email verification
- [ ] Team emails (invitation accepted, role changed, removed)
- [ ] Billing emails (invoice, payment failed, trial ending)
- [ ] Product emails (experiment results, usage alerts, digests)

#### Advanced Billing
- [ ] Metered billing for AI overage
- [ ] Invoice history with PDF download
- [ ] Payment method management
- [ ] Feature gates (experiments, API access, SSO)

### Phase 14: Data Management & Production
- [x] Soft delete implementation (paranoia gem)
- [ ] Restore functionality (admin-only endpoints)
- [ ] Automatic cleanup job (90+ days)
- [ ] GDPR compliance (data export/deletion)
- [ ] Performance monitoring
- [ ] Database backups
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Deployment setup (DigitalOcean/Kamal)

## 🔮 Future Features

### Git Worktree per Session
Each crewkit session runs in its own git worktree for complete isolation:
- **Parallel work**: Multiple sessions on same project without conflicts
- **Clean review**: Session changes isolated for easy PR review
- **Safe experimentation**: Failed sessions don't affect main working directory
- **Automatic cleanup**: Worktrees removed when session ends (configurable)

**CLI:** `crewkit code --worktree` (opt-in initially)
**Model changes:** `CrewkitSession.worktree_path`, `CrewkitSession.worktree_branch`
**Workflow:**
1. `crewkit code --worktree` creates `.crewkit-sessions/{session-id}/` worktree
2. Session runs in isolated worktree with its own branch
3. On session end: prompt to merge/PR/discard changes
4. Worktree cleanup (immediate or after retention period)

### PM Tool Integration
- [ ] MCP server configuration for JIRA/Linear/GitHub
- [ ] Auto-configure PM credentials
- [ ] Agent context from project management tools

### Git Workflow Enhancements
- [ ] Branch-specific agent configurations
- [ ] PR agent review integration (auto-review via webhook)
- [ ] `.claude/` auto-sync from repo on push events
- [ ] Store org/project git workflows

### Dev Environment Setup
- [ ] Docker Compose templates
- [ ] Auto-detect project type
- [ ] Environment metrics tracking

## 📋 Current Status

**Latest:** LSP Module (Phase 1-2) complete with plugin-based Claude Code integration
**Next:** Phase 4 (Session Analytics Dashboard - show value from existing backend)

### What's Complete
- Security & Permissions (Pundit, Paper Trail, tenant isolation)
- API Authentication (JWT, passkeys, device flow)
- Dashboard Foundation (Next.js 16+, shadcn, TanStack Query)
- API Resource Management (Orgs, Projects, Members, Admin)
- Dashboard Integration (all CRUD pages, protected routes)
- Session Infrastructure (3-level hierarchy, streaming, analytics backend)
- Hook System (SubagentStart/Stop, tracker server, event buffer)
- Brief System (prompts, Haiku service, summaries)
- LSP Module (Phase 1-2: scaffold, storage, symbol index, plugin integration)

### What's Next (MVP Path)
| Phase | Focus | Why |
|-------|-------|-----|
| 4 | Session Analytics Dashboard | Show value from existing backend |
| 5 | Role-Based Config & Coaching | Key differentiator for teams |
| 6 | Claude Code v2.1.0 Integration | Leverage new CC features |
| 7 | A/B Experiments | Core product feature |
| 8 | Basic Billing | Enable revenue |

### What Exists vs What's Missing

| Resource | API | CLI | Dashboard |
|----------|-----|-----|-----------|
| Organizations | ✅ | ✅ | ✅ |
| Projects | ✅ | ✅ | ✅ |
| Members | ✅ | ✅ | ✅ |
| Sessions (backend) | ✅ | ✅ | ⏳ Phase 4 |
| Hook System | ✅ | ✅ | N/A |
| LSP Module | N/A | ✅ | N/A |
| Role-Based Config | ⏳ Phase 5 | ⏳ Phase 5 | ⏳ Phase 5 |
| Experiments | ⏳ Phase 7 | ⏳ Phase 7 | ⏳ Phase 7 |
| Billing | ⏳ Phase 8 | N/A | ⏳ Phase 8 |
| Git Integration | ⏳ Phase 12 | ⏳ Phase 12 | ⏳ Phase 12 |
