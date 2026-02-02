---
doc_tier: 1
doc_type: report
doc_status: archived
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
related_docs:
  - docs/prds/API_STANDARDIZATION_PLAN.md
tags: [api, migration, example, sessions]
---

# SessionsController Migration Example

**Purpose**: Concrete before/after example showing how to migrate SessionsController from manual JSON to serializers.

---

## Step 1: Create SessionSerializer

```ruby
# app/serializers/session_serializer.rb
class SessionSerializer < BaseSerializer
  # Core attributes
  attributes :id, :agent_name, :agent_version, :status, :outcome,
             :total_turns, :total_cost, :total_input_tokens, :total_output_tokens,
             :started_at, :ended_at, :created_at, :updated_at

  # IANA standard links
  link :self do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}"
  end

  link :collection do |session, _ctx|
    "/api/v1/sessions"
  end

  # Related resources
  link :project do |session, _ctx|
    org_id = session.organization.external_id
    "/api/v1/#{org_id}/projects/#{session.project.external_id}"
  end

  link :organization do |session, _ctx|
    "/api/v1/organizations/#{session.organization.external_id}"
  end

  link :analytics do |session, _ctx|
    org_id = session.organization.external_id
    "/api/v1/#{org_id}/sessions/analytics/summary"
  end

  # Additional data endpoints
  link :summary do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}/summary"
  end

  link :modifications do |session, _ctx|
    "/api/v1/agents/modifications?session_id=#{session.external_id}"
  end

  # Action links (conditional based on state and authorization)
  link :end, method: :post, title: "End session", if: :can_end? do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}/end"
  end

  link :update, method: :patch, title: "Update session", if: :can_update? do |session, _ctx|
    "/api/v1/sessions/#{session.external_id}"
  end

  # Custom attributes
  def id
    record.external_id
  end

  # Computed fields
  def status
    if record.ended_at.present?
      "completed"
    elsif record.started_at > 1.hour.ago
      "active"
    else
      "stale"
    end
  end

  # Override serialize_data to include nested user data
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
    links = {
      self: "/api/v1/sessions"
    }

    # Add create link if user can create sessions
    if context[:can_create]
      links[:create] = { href: "/api/v1/sessions", method: "POST" }
    end

    # Add analytics link if organization context available
    if context[:organization]
      org_id = context[:organization].external_id
      links[:analytics] = "/api/v1/#{org_id}/sessions/analytics/summary"
    end

    links
  end

  private

  # Authorization checks (uses Pundit policy from context)
  def can_end?
    record.ended_at.nil? && can?(:end_session?)
  end

  def can_update?
    can?(:update?)
  end
end
```

---

## Step 2: Update SessionsController

### Before (Manual JSON)

```ruby
module Api
  module V1
    class SessionsController < BaseController
      # GET /api/v1/sessions
      def index
        authorize :agent_session, :index?

        project = find_project if params[:project_external_id]

        sessions = AgentSession
          .where(organization_id: current_organization.id)
          .order(started_at: :desc)

        sessions = sessions.where(project_id: project.id) if project
        sessions = sessions.where(agent_name: params[:agent_name]) if params[:agent_name]
        sessions = sessions.where(user_id: current_user.id) if params[:current_user_only]

        sessions = sessions.page(params[:page]).per(params[:per_page] || 25)

        render json: {
          sessions: sessions.map { |s| session_json(s) },
          meta: {
            current_page: sessions.current_page,
            total_pages: sessions.total_pages,
            total_count: sessions.total_count
          }
        }
      end

      # GET /api/v1/sessions/:id
      def show
        session = find_session
        authorize session

        render json: {
          session: session_json(session)
        }
      end

      # POST /api/v1/sessions
      def create
        agent_name = params.require(:agent_name)
        project_external_id = params.require(:project_external_id)
        project = find_project

        session = AgentSession.new(
          organization_id: current_organization.id,
          project_id: project.id,
          user_id: current_user.id,
          agent_name: agent_name,
          agent_version: agent_version_string(agent_name, project),
          started_at: Time.current,
          total_turns: 0,
          total_cost: 0.0
        )

        authorize session
        session.save!

        render json: {
          session: session_json(session)
        }, status: :created
      end

      private

      def session_json(session)
        {
          id: session.external_id,
          agent_name: session.agent_name,
          agent_version: session.agent_version,
          total_turns: session.total_turns,
          total_cost: session.total_cost&.to_f,
          outcome: session.outcome,
          started_at: session.started_at,
          ended_at: session.ended_at,
          user: {
            id: session.user.external_id,
            name: session.user.name,
            email: session.user.email
          }
        }
      end
    end
  end
end
```

### After (Serializers)

```ruby
module Api
  module V1
    class SessionsController < BaseController
      include HateoasResponses  # ✅ Add this

      # GET /api/v1/sessions
      def index
        authorize :agent_session, :index?

        project = find_project if params[:project_external_id]

        sessions = AgentSession
          .where(organization_id: current_organization.id)
          .order(started_at: :desc)

        sessions = sessions.where(project_id: project.id) if project
        sessions = sessions.where(agent_name: params[:agent_name]) if params[:agent_name]
        sessions = sessions.where(user_id: current_user.id) if params[:current_user_only]

        sessions = sessions.page(params[:page]).per(params[:per_page] || 25)

        # ✅ Use render_collection instead of manual JSON
        render_collection sessions, serializer: SessionSerializer,
                         organization: current_organization,
                         can_create: policy(:agent_session).create?
      end

      # GET /api/v1/sessions/:id
      def show
        session = find_session
        authorize session

        # ✅ Use render_resource instead of manual JSON
        render_resource session, serializer: SessionSerializer
      end

      # POST /api/v1/sessions
      def create
        agent_name = params.require(:agent_name)
        project_external_id = params.require(:project_external_id)
        project = find_project

        session = AgentSession.new(
          organization_id: current_organization.id,
          project_id: project.id,
          user_id: current_user.id,
          agent_name: agent_name,
          agent_version: agent_version_string(agent_name, project),
          started_at: Time.current,
          total_turns: 0,
          total_cost: 0.0
        )

        authorize session

        if session.save
          # ✅ Use render_resource with 201 status
          render_resource session, serializer: SessionSerializer, status: :created
        else
          render_validation_error(session)
        end
      end

      # ❌ REMOVE session_json helper (no longer needed)
    end
  end
end
```

---

## Step 3: Update Tests

### Before

```ruby
# test/controllers/api/v1/sessions_controller_test.rb
class Api::V1::SessionsControllerTest < ActionDispatch::IntegrationTest
  test "index returns sessions" do
    get api_v1_sessions_url, headers: auth_headers

    assert_response :success
    json = JSON.parse(response.body)
    assert json.key?("sessions")
    assert json.key?("meta")
  end

  test "show returns session" do
    session = agent_sessions(:one)
    get api_v1_session_url(session.external_id), headers: auth_headers

    assert_response :success
    json = JSON.parse(response.body)
    assert json.key?("session")
    assert_equal session.external_id, json["session"]["id"]
  end
end
```

### After

```ruby
# test/controllers/api/v1/sessions_controller_test.rb
class Api::V1::SessionsControllerTest < ActionDispatch::IntegrationTest
  test "index returns standardized collection format" do
    get api_v1_sessions_url, headers: auth_headers

    assert_response :success
    json = JSON.parse(response.body)

    # ✅ Verify standardized structure
    assert json.key?("data"), "Response should have 'data' key"
    assert json.key?("_links"), "Response should have '_links' key"
    assert json.key?("meta"), "Response should have 'meta' key"

    # Verify data is array
    assert_kind_of Array, json["data"]

    # Verify HATEOAS links
    assert json["_links"].key?("self")
    assert json["_links"].key?("create")

    # Verify pagination
    assert json["meta"].key?("total_count")
    assert json["meta"].key?("current_page")
  end

  test "show returns standardized resource format" do
    session = agent_sessions(:one)
    get api_v1_session_url(session.external_id), headers: auth_headers

    assert_response :success
    json = JSON.parse(response.body)

    # ✅ Verify standardized structure
    assert json.key?("data")
    assert json.key?("_links")

    # Verify data is object
    assert_kind_of Hash, json["data"]
    assert_equal session.external_id, json["data"]["id"]

    # Verify HATEOAS links
    assert_equal "/api/v1/sessions/#{session.external_id}", json["_links"]["self"]
    assert json["_links"].key?("project")
    assert json["_links"].key?("analytics")

    # Verify conditional action links
    if session.ended_at.nil?
      assert json["_links"].key?("end"), "Should include 'end' link for active session"
    end
  end

  test "create returns 201 with standardized format" do
    project = projects(:one)

    post api_v1_sessions_url, headers: auth_headers, params: {
      agent_name: "rails-expert",
      project_external_id: project.external_id
    }

    assert_response :created
    json = JSON.parse(response.body)

    # ✅ Verify standardized structure
    assert json.key?("data")
    assert json.key?("_links")

    # Verify created resource
    assert_equal "rails-expert", json["data"]["agent_name"]
    assert json["_links"].key?("self")
  end
end
```

---

## Step 4: Update CLI (Rust)

### Before

```rust
// cli/src/types/api.rs
#[derive(Debug, Deserialize)]
pub struct SessionsResponse {
    pub sessions: Vec<Session>,
    pub meta: PaginationMeta,
}

#[derive(Debug, Deserialize)]
pub struct SessionResponse {
    pub session: Session,
}

#[derive(Debug, Deserialize)]
pub struct Session {
    pub id: String,
    pub agent_name: String,
    pub agent_version: String,
    pub total_turns: i32,
    pub total_cost: Option<f64>,
    pub outcome: Option<String>,
    pub started_at: String,
    pub ended_at: Option<String>,
    pub user: User,
}
```

### After

```rust
// cli/src/types/api.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

// ✅ Generic API response wrapper
#[derive(Debug, Deserialize)]
pub struct ApiResponse<T> {
    pub data: T,
    #[serde(rename = "_links")]
    pub links: Option<HashMap<String, LinkObject>>,
    pub meta: Option<serde_json::Value>,
}

// ✅ Link object (simple string or detailed object)
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

// ✅ Session struct (unchanged)
#[derive(Debug, Deserialize)]
pub struct Session {
    pub id: String,
    pub agent_name: String,
    pub agent_version: String,
    pub total_turns: i32,
    pub total_cost: Option<f64>,
    pub outcome: Option<String>,
    pub started_at: String,
    pub ended_at: Option<String>,
    pub user: User,
}

// ✅ Type aliases for responses
pub type SessionsResponse = ApiResponse<Vec<Session>>;
pub type SessionResponse = ApiResponse<Session>;
```

### Usage Example

```rust
// cli/src/services/api_client.rs
impl ApiClient {
    pub async fn list_sessions(&self, org_id: &str) -> Result<Vec<Session>> {
        let response: SessionsResponse = self
            .get(&format!("/api/v1/sessions"))
            .await?;

        // ✅ Access data field
        Ok(response.data)
    }

    pub async fn get_session(&self, session_id: &str) -> Result<Session> {
        let response: SessionResponse = self
            .get(&format!("/api/v1/sessions/{}", session_id))
            .await?;

        // ✅ Access data field
        Ok(response.data)
    }

    // ✅ Use HATEOAS links for navigation
    pub async fn follow_link(&self, link: &LinkObject) -> Result<String> {
        let href = link.href();
        self.get(href).await
    }
}
```

---

## Step 5: Update OpenAPI Docs (Rswag)

```ruby
# test/integration/api/v1/sessions_spec.rb (Rswag)
require 'swagger_helper'

RSpec.describe 'api/v1/sessions', type: :request do
  path '/api/v1/sessions' do
    get('List sessions') do
      tags 'Sessions'
      security [bearer_auth: []]
      produces 'application/json'

      parameter name: :page, in: :query, type: :integer, required: false
      parameter name: :per_page, in: :query, type: :integer, required: false
      parameter name: :agent_name, in: :query, type: :string, required: false

      response(200, 'successful') do
        schema type: :object,
               properties: {
                 data: {
                   type: :array,
                   items: { '$ref' => '#/components/schemas/Session' }
                 },
                 _links: {
                   type: :object,
                   properties: {
                     self: { type: :string },
                     first: { type: :string },
                     prev: { type: :string },
                     next: { type: :string },
                     last: { type: :string },
                     create: { '$ref' => '#/components/schemas/Link' }
                   }
                 },
                 meta: {
                   type: :object,
                   properties: {
                     total_count: { type: :integer },
                     current_page: { type: :integer },
                     total_pages: { type: :integer },
                     per_page: { type: :integer }
                   }
                 }
               },
               required: ['data', '_links', 'meta']

        run_test!
      end
    end
  end
end
```

---

## Verification Checklist

After migration, verify:

- [ ] All tests passing: `rails test`
- [ ] OpenAPI docs generated: `rake rswag:specs:swaggerize`
- [ ] Manual curl test (index):
  ```bash
  curl http://localhost:3050/api/v1/sessions \
    -H "Authorization: Bearer YOUR_TOKEN"
  # Expected: { data: [...], _links: {...}, meta: {...} }
  ```
- [ ] Manual curl test (show):
  ```bash
  curl http://localhost:3050/api/v1/sessions/SESSION_ID \
    -H "Authorization: Bearer YOUR_TOKEN"
  # Expected: { data: {...}, _links: {...} }
  ```
- [ ] CLI compiles: `cd cli && just build`
- [ ] CLI can list sessions: `just dev sessions list`
- [ ] No Sentry errors

---

## Rollback Plan

If migration causes issues:

1. Revert controller changes
2. Keep serializer (for future use)
3. Deploy rollback to staging/production
4. Investigate issues in dev environment
5. Fix and retry migration

---

**Estimated Time**: 6-8 hours (includes testing + CLI updates)