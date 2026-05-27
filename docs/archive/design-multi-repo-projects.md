# Multi-Repository Project Support

**Status:** Implementation In Progress
**Created:** 2026-01-20
**Reviewed by:** rails-expert, cli-expert, frontend-expert, security-expert

## Problem

Teams often split their codebase across multiple repositories:
- **Multi-repo**: Frontend in one repo, backend in another
- **Monorepo**: Single repo with multiple components (api/, dashboard/, cli/)
- **Hybrid**: Mix of both approaches

Currently crewkit assumes 1:1 between Project and repository. This creates friction for teams with multi-repo setups who want unified session views, cross-repo analytics, and consistent agent configuration.

## Solution

Introduce a **Repository** entity that links git repos to Projects:

```
Organization
  └── Project (business unit - "Customer Portal")
        └── Repository (git repo - where CLI runs)
              └── Session (work done in that repo)
```

**Project** = What teams care about (a product, feature area)
**Repository** = Technical detail for session attribution and context

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Config inheritance | Project-level with component-aware context | Avoid 4-tier complexity |
| First CLI run | Auto-prompt to link/create project inline | Frictionless onboarding |
| Component storage | On session, not repository | Allows flexibility per-session |
| Single-repo UX | Hide Repository concept, show "add repo" affordance | Feels familiar, discoverable |
| Repo identification | Git remote URL (origin), normalized | Universal, auto-detected |
| Haiku analysis | **Opt-in with explicit consent** | Privacy protection |

## Data Model

### New: Repository Model

```ruby
class Repository < ApplicationRecord
  belongs_to :project
  belongs_to :organization  # derived from project, validated
  has_many :crewkit_sessions

  # Identification
  validates :remote_url, presence: true, uniqueness: { scope: :organization_id }
  validates :name, presence: true  # derived from remote, editable
  validate :remote_url_is_safe
  validate :organization_matches_project

  # Session visibility for cross-repo features
  # :public - visible to all project members
  # :restricted - visible only to repo contributors + managers
  # :private - visible only to session owner + admins
  enum :session_visibility, { public: 0, restricted: 1, private: 2 }, default: :public

  before_validation :normalize_remote_url
  before_validation :set_organization_from_project, on: :create

  private

  def set_organization_from_project
    self.organization_id = project&.organization_id
  end

  def organization_matches_project
    return unless project && organization
    errors.add(:organization, "must match project's organization") unless project.organization_id == organization_id
  end

  def normalize_remote_url
    return unless remote_url
    # Normalize: strip .git suffix, lowercase host
    self.remote_url = remote_url.strip.chomp('.git')
  end

  def remote_url_is_safe
    return if remote_url.blank?

    uri = Addressable::URI.parse(remote_url) rescue nil
    unless uri
      errors.add(:remote_url, "is not a valid URL")
      return
    end

    # Block private IPs, localhost, cloud metadata endpoints
    host = uri.host&.downcase
    if private_or_internal_host?(host)
      errors.add(:remote_url, "cannot point to private or internal addresses")
    end
  end
end
```

### Updated: CrewkitSession

```ruby
class CrewkitSession < ApplicationRecord
  belongs_to :repository, optional: true  # nullable for historical sessions
  belongs_to :project       # denormalized, synced from repository
  belongs_to :organization

  # Component stored on session for flexibility (user may work on different components)
  attribute :component, :string  # "api", "dashboard", "cli", etc.

  # Users can mark sessions as sensitive (excluded from prior work discovery)
  attribute :sensitive, :boolean, default: false

  validate :project_matches_repository, if: :repository
  before_validation :sync_project_from_repository, if: :repository

  private

  def sync_project_from_repository
    self.project_id = repository&.project_id
  end

  def project_matches_repository
    return unless repository && project
    errors.add(:project, "must match repository's project") unless repository.project_id == project_id
  end
end
```

### Migration Strategy

**Phase 1: Add Repository (backward compatible)**
```ruby
create_table :repositories, id: :uuid do |t|
  t.references :project, null: false, foreign_key: true, type: :uuid
  t.references :organization, null: false, foreign_key: true, type: :uuid
  t.string :remote_url, null: false
  t.string :name, null: false
  t.integer :session_visibility, default: 0, null: false
  t.timestamps

  t.index [:organization_id, :remote_url], unique: true
  t.index [:project_id]
end

# Add nullable FK and component to sessions
add_reference :crewkit_sessions, :repository, foreign_key: true, type: :uuid
add_column :crewkit_sessions, :component, :string
add_column :crewkit_sessions, :sensitive, :boolean, default: false
add_index :crewkit_sessions, [:repository_id, :started_at]
```

**Phase 2: Backfill existing data**
- Create default Repository for each Project with existing sessions (where remote_url can be inferred)
- Populate repository_id on sessions where possible
- **Historical sessions without repo context remain with NULL repository_id**

**Phase 3: Require repository for NEW sessions only**
- Application-level validation, not database constraint
- Allows historical data to remain intact

## CLI Behavior

### First Run in Unlinked Repository

When CLI starts in a repo not linked to any project:

```bash
$ crewkit code
Detected: git@github.com:acme/frontend.git (not linked)

? What would you like to do?
  > Quick link to existing project
  > Analyze repository and get recommendations (~3s)
  > Create new project
  > Skip linking (offline mode)
```

**If user chooses "Analyze repository":**

```bash
Analyzing repository structure requires sending file metadata to crewkit.

Files to be analyzed:
  - package.json (dependencies only, no scripts)
  - Directory structure (names only)

? Allow analysis? [y/N]: y

Analyzing...
  ├── Next.js 16 frontend detected
  ├── Likely related to "Customer Portal" (similar naming)
  └── Recommended agents: frontend-expert, api-designer

? Link to existing project?
  > Customer Portal (1 repo linked)
  > Create new project

Linked to "Customer Portal" ✓
Syncing agents...
```

### Haiku Analysis - Privacy Controls

**Explicit opt-in required.** Before any analysis:

1. Show exactly what files will be sent
2. Explain data handling (sent to Anthropic Claude Haiku)
3. Allow permanent opt-out in user config

**What Haiku receives (sanitized):**
- Package manifest dependencies (NOT scripts, NOT private registry URLs)
- Directory names (NOT file contents)
- File type counts

**What Haiku does NOT receive:**
- Source code
- Environment variables
- Private registry credentials
- Full file paths

**Permanent opt-out:**
```toml
# ~/.config/crewkit/config.toml
[privacy]
allow_haiku_analysis = false
```

### Offline Fallback

When network unavailable or user skips analysis:

```rust
fn detect_stack_locally(repo_root: &Path) -> TechStack {
    let mut stack = TechStack::default();

    if repo_root.join("package.json").exists() {
        stack.languages.push("javascript");
        // Check for framework indicators
        if repo_root.join("next.config.js").exists() || repo_root.join("next.config.ts").exists() {
            stack.frameworks.push("nextjs");
        }
    }
    if repo_root.join("Cargo.toml").exists() {
        stack.languages.push("rust");
    }
    if repo_root.join("Gemfile").exists() {
        stack.languages.push("ruby");
        if repo_root.join("config/routes.rb").exists() {
            stack.frameworks.push("rails");
        }
    }

    stack
}
```

### Component Detection (Improved)

Check **all** path segments, not just first:

```rust
fn detect_component(working_dir: &Path, repo_root: &Path) -> Option<String> {
    let relative = working_dir.strip_prefix(repo_root).ok()?;

    // Check all path segments for component indicators
    for component in relative.components() {
        let segment = component.as_os_str().to_str()?;
        match segment.to_lowercase().as_str() {
            "api" | "backend" | "server" => return Some("api".into()),
            "dashboard" | "frontend" | "web" | "client" => return Some("dashboard".into()),
            "cli" | "cmd" | "tools" => return Some("cli".into()),
            // Skip common wrapper directories
            "packages" | "apps" | "services" | "src" | "lib" => continue,
            _ => continue,
        }
    }
    None
}
```

### Local Cache

```toml
# .crewkit/config.toml (gitignored)
version = 1  # Schema version for future migrations

[repository]
id = "repo_abc123"
remote_url = "git@github.com:acme/frontend"  # For validation
linked_at = "2026-01-20T10:30:00Z"

[component]
detected = "dashboard"
override = null  # User can set to override detection

[analysis]
consented = true
timestamp = "2026-01-20T10:30:00Z"
stack = ["nextjs", "typescript"]
recommended_agents = ["frontend-expert"]

[sync]
last_sync = "2026-01-20T10:35:00Z"
agents = ["frontend-expert", "api-designer"]
```

**CLI validates cache against current git remote** - if mismatch, cache is invalidated.

### CI/CD Support

```bash
# Non-interactive mode for CI
crewkit code --yes --repo-id repo_abc123

# Or via environment
CREWKIT_REPOSITORY_ID=repo_abc123 crewkit code --yes
```

Auto-detect non-TTY and provide helpful error:
```
Error: Interactive prompts not available in non-TTY environment.

To use crewkit in CI/CD:
  1. Link repository locally first: crewkit init
  2. Use --yes flag: crewkit code --yes
  3. Or set CREWKIT_REPOSITORY_ID environment variable
```

## API Changes

### New Endpoints

```ruby
# Repository management (nested under project for creation context)
POST   /api/v1/:org_id/projects/:project_id/repositories
GET    /api/v1/:org_id/projects/:project_id/repositories

# Repository operations (flat, no project context needed)
GET    /api/v1/:org_id/repositories/:id
DELETE /api/v1/:org_id/repositories/:id

# Repository lookup (for CLI)
GET    /api/v1/:org_id/repositories/lookup?remote_url=...

# Haiku analysis (opt-in, requires consent)
POST   /api/v1/:org_id/repositories/analyze
# Body: { files: [...], consent_given: true }
# Returns: { tech_stack, project_type, related_projects, recommended_agents }
```

### Repository Lookup - Security

**Prevent organization enumeration:**

```ruby
def lookup
  repository = current_organization.repositories.find_by(
    remote_url: normalize_url(params[:remote_url])
  )

  if repository
    render json: { data: RepositorySerializer.new(repository) }
  else
    # IMPORTANT: Same response structure whether repo exists in other org or not
    render json: {
      data: nil,
      meta: { message: "Repository not linked to any project" }
    }
  end
end
```

### Session Creation - Error Handling

**Use 422, not 404 for unlinked repos:**

```ruby
POST /api/v1/:org_id/sessions
{
  "remote_url": "git@github.com:acme/backend",
  "component": "api",
  "working_directory": "/Users/dev/code/acme/api"
}
```

**If remote_url not linked:**
```json
HTTP 422 Unprocessable Entity
{
  "type": "repository_not_linked",
  "title": "Repository Not Linked",
  "status": 422,
  "detail": "No repository linked for this remote URL",
  "suggested_action": "link_repository",
  "links": {
    "lookup": "/api/v1/:org_id/repositories/lookup",
    "projects": "/api/v1/:org_id/projects"
  }
}
```

**Server validates repository_id against git remote:**
```ruby
def create
  if params[:repository_id].present?
    repository = current_organization.repositories.find_by(external_id: params[:repository_id])

    # Validate remote_url matches (prevents cache manipulation attacks)
    unless repository&.remote_url == normalize_url(params[:remote_url])
      return render_error(422, "repository_mismatch", "Cached repository does not match git remote")
    end
  end
  # ... create session
end
```

### Pundit Policies

```ruby
class RepositoryPolicy < ApplicationPolicy
  def index?
    same_organization?
  end

  def show?
    same_organization?
  end

  def create?
    # Manager+ can link repos to projects
    same_organization? && (user.manager? || user.admin?)
  end

  def destroy?
    # Manager+ can unlink repos (affects existing sessions)
    same_organization? && (user.manager? || user.admin?)
  end

  def analyze?
    # Any org member can analyze (read-only, opt-in)
    same_organization?
  end

  private

  def same_organization?
    user.organization_ids.include?(record.organization_id)
  end
end
```

## Dashboard UX

### Single-Repo Projects

Repository concept is **hidden**, but discoverable:
- No "Repositories" section in main navigation
- Sessions list shows directly under project
- **"Add another repository" link** in project settings or overview
- Feels like current behavior

### Multi-Repo Projects

Repository becomes visible when 2+ repos linked:

```
Project: Customer Portal
├── Repositories (3)
│   ├── acme/backend (api) - 142 sessions
│   ├── acme/frontend (dashboard) - 89 sessions
│   └── acme/infrastructure - 23 sessions
├── Sessions [Filter: All repos ▼]
│   ├── Session 1 (frontend) - 2h ago
│   ├── Session 2 (backend) - 3h ago
│   └── Session 3 (frontend) - 5h ago
├── Analytics (aggregated across repos)
└── Agent Config (applies to all repos)
```

**Transition UX:** When adding second repo, show one-time toast explaining the new Repositories section.

### URL Structure

Use **slug** in query params for readability:

```
/kit/projects/[id]                           # Project overview
/kit/projects/[id]/repositories              # List repos (only if multi-repo)
/kit/projects/[id]/conversations             # All conversations
/kit/projects/[id]/conversations?repo=backend  # Filtered by repo slug
```

### State Management

URL-driven filter state via React hook:

```typescript
// hooks/use-repo-filter.ts
export function useRepoFilter() {
  const searchParams = useSearchParams()
  const router = useRouter()
  const pathname = usePathname()

  const repoSlug = searchParams.get('repo') // null means "all"

  const setRepoFilter = (slug: string | null) => {
    const params = new URLSearchParams(searchParams)
    if (slug) {
      params.set('repo', slug)
    } else {
      params.delete('repo')
    }
    router.push(`${pathname}?${params.toString()}`)
  }

  return { repoSlug, setRepoFilter }
}
```

### API Integration

- `GET /projects` - does NOT include repositories (list view efficiency)
- `GET /projects/:id` - DOES include repositories (detail view)
- Filter endpoints accept `?repo=slug` parameter

### New Components

- `RepositoryCard` - displays repo name, git remote, session count
- `RepositoryList` - grid/list of repos for multi-repo projects
- `RepoFilterDropdown` - filter selector (only visible when repos > 1)
- `RepoBadge` - small tag showing repo name on session rows

## Cross-Repo Features

### Prior Work Discovery - Privacy Controls

Surface related work from other repos, **respecting visibility settings**:

```
Starting session in frontend-repo...

Related prior work:
├── [frontend] Auth component refactor (Sarah, 2d ago)
├── [backend] API rate limiting (Mike, 3d ago)  ← Cross-repo
└── [frontend] Login page redesign (Alice, 1w ago)

Continue? [Y/n/details]
```

**What's shown:**
- Session timestamp
- Repository name
- User name (if visibility allows)
- High-level topic (AI-generated summary, sanitized)

**What's NOT shown:**
- Actual prompts
- Code snippets
- Error messages
- Detailed file paths

**Exclusions:**
- Sessions marked as `sensitive: true`
- Sessions in repos with `session_visibility: :private`
- Sessions older than configured threshold

**Audit logging:**
```ruby
def show_prior_work
  related_sessions = find_cross_repo_sessions

  related_sessions.each do |session|
    if session.repository_id != current_repository_id
      AuditLog.create!(
        user: current_user,
        action: :viewed_cross_repo_session,
        target: session,
        context: { from_repository: current_repository_id }
      )
    end
  end
end
```

### Analytics Aggregation

Project-level analytics aggregate across all repos:
- Total sessions, costs, tokens by project
- Breakdown by repository available as drill-down
- Cross-repo patterns visible in team dashboards

## Implementation Phases

### Phase 1: Core Model (MVP)
- [x] Add Repository model with validation and normalization
- [x] Add repository_id (nullable) and component to CrewkitSession
- [ ] CLI: Detect git remote, prompt for project linking
- [x] CLI: Offline fallback with local stack detection
- [x] API: Repository CRUD + lookup endpoint
- [ ] API: Session creation with 422 error for unlinked repos
- [x] Backfill existing projects with default repository

### Phase 2: Intelligent Onboarding
- [ ] Haiku analysis endpoint with explicit consent flow
- [ ] Project matching based on naming/org patterns
- [ ] Agent recommendations based on detected stack
- [ ] Improved component detection (all path segments)
- [ ] CI/CD support (--yes flag, env vars)

### Phase 3: Dashboard Integration
- [ ] Show/hide repositories based on count
- [ ] "Add repository" affordance for single-repo projects
- [ ] Repository filter dropdown on sessions list
- [ ] RepoBadge component on session rows
- [ ] Per-repo analytics drill-down

### Phase 4: Cross-Repo Features
- [ ] Session visibility settings per repository
- [ ] Sensitive session flag
- [ ] Prior work discovery with privacy controls
- [ ] Cross-repo audit logging
- [ ] Component-aware agent context variables

## Security Checklist

Before shipping:
- [ ] Repository lookup returns identical responses for "not found" vs "exists in other org"
- [ ] Remote URL validation blocks SSRF to internal/metadata endpoints
- [ ] Remote URL normalization applied consistently
- [ ] Haiku analysis has explicit opt-in consent with preview
- [ ] Server validates repository_id against git remote on session creation
- [ ] Cross-repo prior work excludes prompts and respects visibility settings
- [ ] Rate limiting on lookup (10/min) and analyze (5/min) endpoints
- [ ] Audit logging for cross-repo access patterns
- [ ] Privacy policy updated for Haiku analysis data handling

## Open Questions

1. **Renamed remotes / repo migrations** - How to handle when a repo URL changes?
   - Proposed: Admin endpoint to merge repositories

2. **Forked repos** - Should forks link to same project or separate?
   - Proposed: Separate by default, manual linking option

3. **Max repos per project** - Design limit?
   - Proposed: Soft limit of 20, dropdown becomes searchable above 5

## Success Metrics

- **Activation**: % of orgs with 2+ repos linked to one project
- **Engagement**: Cross-repo session views (users finding value in unified view)
- **Retention**: Multi-repo projects have higher team engagement than single-repo
- **Privacy**: < 1% of users opt-out of Haiku analysis after seeing consent flow
