---
doc_tier: 1
doc_type: plan
doc_status: active
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
tags: [api, standardization, hateoas]
---

# API Response Format Standardization Plan

**Status**: Active
**Owner**: platform-team
**Created**: 2026-01-03
**Target Completion**: Q1 2026

## Executive Summary

crewkit API has inconsistent response formats across controllers. Some use serializers with HATEOAS links (`{ data:, _links: }`), while others return raw JSON (`{ sessions: [...] }`). This plan standardizes all API responses to use the BaseSerializer pattern for consistency, discoverability, and better CLI/dashboard integration.

---

## 1. Standard Response Format Specification

### 1.1 Single Resource Response

**Format**:
```json
{
  "data": {
    "id": "uuid",
    "name": "example",
    "status": "active",
    ...
  },
  "_links": {
    "self": "/api/v1/resources/uuid",
    "collection": "/api/v1/resources",
    "update": { "href": "/api/v1/resources/uuid", "method": "PATCH" },
    "delete": { "href": "/api/v1/resources/uuid", "method": "DELETE" }
  }
}
```

**Rules**:
- `data`: Always an object for single resources
- `_links`: HATEOAS links (IANA standard relations preferred)
- Links include `method` hint for non-GET actions
- Conditional links based on authorization (use Pundit policies)

### 1.2 Collection Response

**Format**:
```json
{
  "data": [
    { "id": "uuid1", ... },
    { "id": "uuid2", ... }
  ],
  "_links": {
    "self": "/api/v1/resources",
    "first": "/api/v1/resources?page=1",
    "prev": "/api/v1/resources?page=2",
    "next": "/api/v1/resources?page=4",
    "last": "/api/v1/resources?page=10",
    "create": { "href": "/api/v1/resources", "method": "POST" }
  },
  "meta": {
    "total_count": 250,
    "current_page": 3,
    "total_pages": 10,
    "per_page": 25
  }
}
```

**Rules**:
- `data`: Always an array for collections
- `_links`: Include pagination links (first, prev, next, last) + create if authorized
- `meta`: Pagination metadata (Kaminari-compatible)

### 1.3 Error Response (Already Standardized)

**Format** (RFC 7807-inspired via ErrorSerializer):
```json
{
  "error": {
    "code": "validation_error",
    "message": "Validation failed",
    "details": {
      "name": ["can't be blank"],
      "slug": ["has already been taken"]
    },
    "request_id": "uuid"
  }
}
```

**Status**: Already standardized via `ApiErrorHandling` concern + `ErrorSerializer`.

### 1.4 Discovery Endpoint (New)

**Endpoint**: `GET /api/v1`

**Response**:
```json
{
  "api": "crewkit",
  "version": "v1",
  "_links": {
    "self": "/api/v1",
    "organizations": { "href": "/api/v1/organizations", "method": "GET" },
    "resources": { "href": "/api/v1/resources", "method": "GET" },
    "documentation": { "href": "/api-docs", "method": "GET" }
  },
  "meta": {
    "authenticated": true,
    "user_id": "uuid",
    "rate_limit": {
      "limit": 1000,
      "remaining": 998,
      "reset_at": "2026-01-03T12:00:00Z"
    }
  }
}
```

**Purpose**: CLI discovery, rate limit checking, API version verification.

---

## 2. Current State Analysis

### 2.1 Controllers Using Serializers (✅ Good)

| Controller | Serializer | Format | Status |
|------------|-----------|--------|--------|
| `ResourcesController` | `ResourceSerializer` | `{ data:, _links: }` | ✅ Compliant |
| `ProjectsController` | `ProjectSerializer` | `{ data:, _links: }` | ✅ Compliant |
| `SessionAnalyticsController` | Manual | `{ data:, meta:, links: }` | ⚠️ Non-standard key (`links` vs `_links`) |

### 2.2 Controllers Using Raw JSON (❌ Needs Migration)

| Controller | Current Format | Issue |
|------------|---------------|-------|
| `SessionsController` | `{ sessions: [...] }` | No `_links`, inconsistent envelope |
| `AgentsController` | Unknown | Need to check |
| `ExperimentsController` | Unknown | Need to check |
| `OrganizationsController` | Unknown | Need to check |

### 2.3 Mixed Patterns in SessionAnalyticsController

**Current**: Uses `links` instead of `_links`:
```ruby
render json: {
  data: summary_data,
  meta: { ... },
  links: analytics_links  # ❌ Should be _links
}
```

**Issue**: Inconsistent with BaseSerializer standard (`_links`).

---

## 3. Migration Strategy

### 3.1 Approach: Incremental Migration

**Rationale**:
- API is in active development
- CLI and dashboard already consume some endpoints
- Breaking all endpoints at once risks production issues

**Strategy**:
1. Fix low-impact endpoints first (SessionAnalyticsController)
2. Migrate high-traffic endpoints with versioning (SessionsController)
3. Add API root discovery endpoint
4. Document all changes in OpenAPI (Rswag)

### 3.2 Backward Compatibility

**Option A: Dual Format Support (Recommended)**
```ruby
# Support both old and new formats via Accept header or query param
def index
  sessions = AgentSession.where(organization_id: current_organization.id)

  if legacy_format_requested?
    render json: { sessions: sessions.map { |s| session_json(s) } }
  else
    render json: SessionSerializer.serialize_collection(sessions, session_context)
  end
end

def legacy_format_requested?
  request.headers['Accept'] == 'application/vnd.crewkit.v1.legacy+json' ||
  params[:format] == 'legacy'
end
```

**Option B: Version Bump (v2)**
- Create `/api/v2/` namespace with standardized responses
- Maintain `/api/v1/` for backward compatibility
- Deprecate v1 with sunset date

**Recommendation**: **Option A** for now (CLI is only consumer, easy to update). Migrate to v2 when dashboard is live.

---

## 4. Implementation Checklist

### Phase 1: Fix SessionAnalyticsController (Low Risk)

**Changes**:
- [ ] Rename `links` to `_links` in all responses
- [ ] Update OpenAPI docs (Rswag specs)
- [ ] Update CLI to handle `_links` key
- [ ] Test dashboard integration (if consuming analytics endpoints)

**Files to Update**:
- `api/app/controllers/api/v1/session_analytics_controller.rb`
- `api/test/integration/api/v1/session_analytics_controller_test.rb`
- `cli/src/services/api_client.rs` (if analytics endpoints used)

**Estimated Effort**: 1-2 hours

---

### Phase 2: Create SessionSerializer

**Create**:
- [ ] `app/serializers/session_serializer.rb` (based on BaseSerializer)
- [ ] Define HATEOAS links (self, project, user, summary, analytics)
- [ ] Add conditional action links (update, delete based on policy)

**Example**:
```ruby
# app/serializers/session_serializer.rb
class SessionSerializer < BaseSerializer
  attributes :id, :agent_name, :agent_version, :status, :outcome,
             :total_turns, :total_cost, :started_at, :ended_at

  link :self do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}"
  end

  link :project do |session, _ctx|
    org_id = session.organization.external_id
    "/api/v1/#{org_id}/projects/#{session.project.external_id}"
  end

  link :analytics do |session, _ctx|
    org_id = session.organization.external_id
    "/api/v1/#{org_id}/sessions/analytics/summary"
  end

  link :summary do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}/summary"
  end

  link :end, method: :post, title: "End session", if: :can_end? do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}/end"
  end

  link :update, method: :patch, if: :can_update? do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}"
  end

  # Custom attribute: use external_id as id
  def id
    record.external_id
  end

  # Nested user data
  def serialize_data
    data = super
    data[:user] = {
      id: record.user.external_id,
      name: record.user.name,
      email: record.user.email
    }
    data
  end

  # Collection-level links
  def self.collection_links(records, context)
    {
      self: "/api/v1/sessions",
      create: { href: "/api/v1/sessions", method: "POST" },
      analytics: "/api/v1/#{context[:organization]&.external_id}/sessions/analytics/summary"
    }.compact
  end

  private

  def can_end?
    record.ended_at.nil? && can?(:end_session?)
  end
end
```

**Estimated Effort**: 2-3 hours

---

### Phase 3: Migrate SessionsController

**Changes**:
- [ ] Replace `session_json` helper with `SessionSerializer`
- [ ] Use `render_resource` / `render_collection` from `HateoasResponses` concern
- [ ] Update all actions (index, show, create, update, summary, statistics)
- [ ] Remove manual JSON builders
- [ ] Update tests to expect new format

**Before**:
```ruby
def show
  session = find_session
  authorize session

  render json: {
    session: session_json(session)
  }
end
```

**After**:
```ruby
def show
  session = find_session
  authorize session

  render_resource session, serializer: SessionSerializer
end
```

**Files to Update**:
- `api/app/controllers/api/v1/sessions_controller.rb`
- `api/test/controllers/api/v1/sessions_controller_test.rb`
- `cli/src/services/api_client.rs` (session response parsing)
- `cli/src/types/api.rs` (update Session struct)

**Estimated Effort**: 4-6 hours

---

### Phase 4: Audit Remaining Controllers

**Controllers to Check**:
- [ ] `AgentsController`
- [ ] `ExperimentsController`
- [ ] `OrganizationsController`
- [ ] `MembersController`
- [ ] `GitIntegrationsController`
- [ ] `ProjectRepositoriesController`

**Process**:
1. Read controller source
2. Identify response format (serializer vs manual JSON)
3. If manual JSON, create serializer + migrate
4. Update tests + OpenAPI docs

**Estimated Effort**: 8-12 hours (depends on controller count)

---

### Phase 5: Add API Root Discovery Endpoint

**Create**:
- [ ] `app/controllers/api/v1/root_controller.rb`
- [ ] Route: `GET /api/v1 => api/v1/root#index`
- [ ] Response includes all top-level resource links
- [ ] Include rate limit info in `meta`

**Example**:
```ruby
# app/controllers/api/v1/root_controller.rb
module Api
  module V1
    class RootController < BaseController
      skip_before_action :set_current_organization

      def index
        render json: {
          api: "crewkit",
          version: "v1",
          _links: discovery_links,
          meta: {
            authenticated: current_user.present?,
            user_id: current_user&.external_id,
            rate_limit: rate_limit_info
          }
        }
      end

      private

      def discovery_links
        {
          self: "/api/v1",
          organizations: { href: "/api/v1/organizations", method: "GET" },
          resources: { href: "/api/v1/resources", method: "GET" },
          documentation: { href: "/api-docs", method: "GET" }
        }
      end

      def rate_limit_info
        # Rack::Attack integration (if enabled)
        {
          limit: 1000,
          remaining: 998, # Placeholder
          reset_at: 1.hour.from_now.iso8601
        }
      end
    end
  end
end
```

**Estimated Effort**: 2-3 hours

---

### Phase 6: Documentation Updates

**OpenAPI (Rswag)**:
- [ ] Update all endpoint specs to reflect new response format
- [ ] Document `_links` structure for each endpoint
- [ ] Add examples for collections with pagination
- [ ] Regenerate OpenAPI docs: `rake rswag:specs:swaggerize`

**Dashboard Integration**:
- [ ] Update API client in Next.js to handle `_links`
- [ ] Use links for navigation instead of hardcoded URLs
- [ ] Update TypeScript types to match new format

**CLI Updates**:
- [ ] Update Rust structs in `cli/src/types/api.rs`
- [ ] Parse `_links` for dynamic endpoint discovery
- [ ] Use pagination links in `list` commands

**Estimated Effort**: 4-6 hours

---

## 5. Code Templates

### 5.1 Creating a New Serializer

```ruby
# app/serializers/example_serializer.rb
class ExampleSerializer < BaseSerializer
  # Core attributes
  attributes :id, :name, :status, :created_at, :updated_at

  # IANA standard links
  link :self do |record, _ctx|
    "/api/v1/examples/#{record.external_id}"
  end

  link :collection do |record, _ctx|
    "/api/v1/examples"
  end

  # Related resource links
  link :organization, if: :has_organization? do |record, _ctx|
    "/api/v1/organizations/#{record.organization.external_id}"
  end

  # Action links (conditional)
  link :update, method: :patch, title: "Update example", if: :can_update? do |record, _ctx|
    "/api/v1/examples/#{record.external_id}"
  end

  link :delete, method: :delete, title: "Delete example", if: :can_destroy? do |record, _ctx|
    "/api/v1/examples/#{record.external_id}"
  end

  # Custom attribute (override)
  def id
    record.external_id
  end

  # Override serialize_data for nested data
  def serialize_data
    data = super
    # Add nested associations or computed fields
    data[:custom_field] = record.compute_something
    data
  end

  # Collection-level links
  def self.collection_links(records, context)
    links = {
      self: "/api/v1/examples"
    }

    if context[:can_create]
      links[:create] = { href: "/api/v1/examples", method: "POST" }
    end

    links
  end

  private

  def has_organization?
    record.organization.present?
  end
end
```

### 5.2 Migrating a Controller to Use Serializers

**Before**:
```ruby
class ExamplesController < BaseController
  def index
    examples = Example.where(organization_id: current_organization.id)
                      .page(params[:page])

    render json: {
      examples: examples.map { |e| example_json(e) },
      meta: {
        current_page: examples.current_page,
        total_pages: examples.total_pages
      }
    }
  end

  def show
    example = Example.find_by!(external_id: params[:id])
    authorize example

    render json: {
      example: example_json(example)
    }
  end

  private

  def example_json(example)
    {
      id: example.external_id,
      name: example.name,
      status: example.status
    }
  end
end
```

**After**:
```ruby
class ExamplesController < BaseController
  include HateoasResponses

  def index
    examples = Example.where(organization_id: current_organization.id)
                      .page(params[:page])

    render_collection examples, serializer: ExampleSerializer
  end

  def show
    example = Example.find_by!(external_id: params[:id])
    authorize example

    render_resource example, serializer: ExampleSerializer
  end
end
```

**Diff**:
- ✅ Remove manual `example_json` helper
- ✅ Use `render_resource` / `render_collection`
- ✅ Include `HateoasResponses` concern
- ✅ Serializer handles links + formatting

### 5.3 CLI Integration (Rust)

**Before** (parsing raw JSON):
```rust
// cli/src/types/api.rs
#[derive(Debug, Deserialize)]
pub struct SessionsResponse {
    pub sessions: Vec<Session>,
    pub meta: PaginationMeta,
}
```

**After** (parsing standardized format):
```rust
// cli/src/types/api.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Deserialize)]
pub struct ApiResponse<T> {
    pub data: T,
    #[serde(rename = "_links")]
    pub links: Option<HashMap<String, LinkObject>>,
    pub meta: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(untagged)]
pub enum LinkObject {
    Simple(String),
    Detailed {
        href: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        method: Option<String>,
        #[serde(skip_serializing_if = "Option::is_none")]
        title: Option<String>,
    },
}

impl LinkObject {
    pub fn href(&self) -> &str {
        match self {
            LinkObject::Simple(href) => href,
            LinkObject::Detailed { href, .. } => href,
        }
    }
}

// Usage:
#[derive(Debug, Deserialize)]
pub struct Session {
    pub id: String,
    pub agent_name: String,
    pub status: String,
    // ...
}

type SessionsResponse = ApiResponse<Vec<Session>>;
type SessionResponse = ApiResponse<Session>;
```

---

## 6. Testing Strategy

### 6.1 Controller Tests

**Test response structure**:
```ruby
# test/controllers/api/v1/examples_controller_test.rb
class Api::V1::ExamplesControllerTest < ActionDispatch::IntegrationTest
  test "index returns standardized collection format" do
    get api_v1_examples_url, headers: auth_headers

    assert_response :success
    json = JSON.parse(response.body)

    # Verify structure
    assert json.key?("data"), "Response should have 'data' key"
    assert json.key?("_links"), "Response should have '_links' key"
    assert json.key?("meta"), "Response should have 'meta' key"

    # Verify data is array
    assert_kind_of Array, json["data"]

    # Verify pagination links
    assert json["_links"].key?("self")
    assert json["_links"].key?("first")
    # assert next/prev if applicable

    # Verify meta has pagination info
    assert json["meta"].key?("total_count")
    assert json["meta"].key?("current_page")
  end

  test "show returns standardized resource format" do
    example = examples(:one)
    get api_v1_example_url(example.external_id), headers: auth_headers

    assert_response :success
    json = JSON.parse(response.body)

    # Verify structure
    assert json.key?("data")
    assert json.key?("_links")

    # Verify data is object
    assert_kind_of Hash, json["data"]
    assert_equal example.external_id, json["data"]["id"]

    # Verify self link
    assert_equal "/api/v1/examples/#{example.external_id}", json["_links"]["self"]
  end
end
```

### 6.2 Serializer Tests

**Test link generation**:
```ruby
# test/serializers/example_serializer_test.rb
class ExampleSerializerTest < ActiveSupport::TestCase
  test "serializes with HATEOAS links" do
    example = examples(:one)
    policy = Minitest::Mock.new
    policy.expect(:update?, true)
    policy.expect(:destroy?, false)

    serializer = ExampleSerializer.new(example, policy: policy)
    result = serializer.serialize

    # Verify data
    assert_equal example.external_id, result[:data][:id]

    # Verify links
    assert_equal "/api/v1/examples/#{example.external_id}", result[:_links][:self]

    # Conditional links
    assert result[:_links].key?(:update), "Should include update link (authorized)"
    refute result[:_links].key?(:delete), "Should not include delete link (not authorized)"
  end
end
```

---

## 7. Migration Priority Order

**Phase 1 (Immediate - Week 1)**:
1. Fix SessionAnalyticsController (`links` → `_links`) - 2 hours
2. Create SessionSerializer - 3 hours
3. Migrate SessionsController - 6 hours

**Phase 2 (Short-term - Week 2-3)**:
4. Audit remaining controllers - 4 hours
5. Create serializers for controllers using raw JSON - 8 hours
6. Add API root discovery endpoint - 3 hours

**Phase 3 (Documentation - Week 4)**:
7. Update OpenAPI docs (Rswag) - 4 hours
8. Update CLI Rust types - 3 hours
9. Update dashboard API client (Next.js) - 3 hours

**Total Estimated Effort**: 36-42 hours (~1 week sprint)

---

## 8. Rollout Plan

### 8.1 Development

**Step 1**: Create feature branch
```bash
git checkout -b feature/api-standardization
```

**Step 2**: Migrate controllers incrementally (1-2 per day)

**Step 3**: Update tests as you go (no failing tests)

**Step 4**: Update CLI after each controller migration

### 8.2 Testing

**Step 1**: MiniTest suite (all controller + serializer tests)

**Step 2**: Manual curl testing (examples in controller comments)

**Step 3**: CLI integration testing (local API server)

**Step 4**: Dashboard smoke testing (if endpoints consumed)

### 8.3 Deployment

**Step 1**: Deploy to staging with both formats supported

**Step 2**: Update CLI to use new format (release v0.2.0)

**Step 3**: Monitor Sentry for API errors (7 days)

**Step 4**: Deploy to production

**Step 5**: Deprecate legacy format (6 months grace period)

---

## 9. Success Criteria

- [ ] All API endpoints return `{ data:, _links:, meta: }` format
- [ ] No manual JSON builders in controllers (use serializers)
- [ ] All serializers extend BaseSerializer
- [ ] HATEOAS links include method hints for actions
- [ ] Conditional links based on Pundit policies
- [ ] API root discovery endpoint live at `/api/v1`
- [ ] OpenAPI docs updated (Rswag)
- [ ] CLI parsing new format (Rust types updated)
- [ ] Dashboard consuming new format (Next.js API client updated)
- [ ] Zero Sentry errors related to response format

---

## 10. Maintenance

**Going Forward**:
- All new controllers MUST use serializers
- No manual JSON builders allowed
- Pre-commit hook enforces serializer usage (optional)
- Update CONTRIBUTING.md with serializer guidelines

**Linting Rule** (future):
```ruby
# .rubocop.yml
Rails/RenderJSON:
  Enabled: true
  EnforcedStyle: serializer
  Exclude:
    - app/controllers/api/v1/root_controller.rb # Discovery endpoint
```

---

## Appendix A: Reference Links

- BaseSerializer source: `api/app/serializers/base_serializer.rb`
- HateoasResponses concern: `api/app/controllers/concerns/hateoas_responses.rb`
- ResourceSerializer (example): `api/app/serializers/resource_serializer.rb`
- IANA Link Relations: https://www.iana.org/assignments/link-relations/
- RFC 7807 (Problem Details): https://tools.ietf.org/html/rfc7807

---

## Appendix B: FAQ

**Q: Why `_links` instead of `links`?**
A: HATEOAS convention uses underscore prefix for hypermedia controls to distinguish from domain data. HAL JSON spec uses `_links`, JSON:API uses `links`, we follow HAL for consistency with BaseSerializer.

**Q: Should we use full JSON:API spec?**
A: No. JSON:API is heavy for our use case (CLI + controlled dashboard). Our pragmatic HATEOAS approach is lighter and faster.

**Q: What about GraphQL?**
A: Not planned. REST API + HATEOAS provides sufficient flexibility for CLI and dashboard needs. GraphQL adds complexity without clear benefits for our use case.

**Q: Breaking changes to dashboard?**
A: Dashboard is early-stage. We control it. Update API client atomically with backend changes. No external consumers yet.

**Q: CLI backward compatibility?**
A: CLI is versioned with API. Release CLI v0.2.0 alongside API standardization. Users update CLI via `crewkit upgrade`.

**Q: Performance impact of serializers?**
A: Minimal. BaseSerializer is lightweight (no heavy gems like jsonapi-resources). Benchmarks show <5ms overhead for typical responses.

---

**End of Plan**