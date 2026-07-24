# Work Updates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let an agent broadcast short, project-visible work updates (optionally task-linked) that flow into other sessions' context pulse and a new dashboard recap panel.

**Architecture:** New append-only `WorkUpdate` model written via a `share_update` MCP tool → `POST .../work_updates` (sanitized, privacy-gated on the source session). Reads: the existing `ContextPulseService` gains an `updates` source (CLI hook injects it), and a new `WorkUpdatesController#index` feeds a dashboard `WorkUpdatesPanel`.

**Tech Stack:** Rails 8.1 (Minitest), Rust CLI (rmcp tools, reqwest, wiremock), Next.js 16 + React Query + Vitest/RTL/MSW.

**Branch:** `feat/work-updates`, stacked on `feat/cross-session-awareness` (PR #224). All commands assume those base commits are present (notably `ContextPulseService` and `format_pulse`).

---

## File Structure

**API**
- Create: `api/db/migrate/20260622000001_create_work_updates.rb`, `api/app/models/work_update.rb`, `api/app/controllers/api/v1/work_updates_controller.rb`, `api/app/policies/work_update_policy.rb`
- Modify: `api/app/models/project.rb` (`has_many :work_updates`), `api/config/routes.rb`, `api/app/services/context_pulse_service.rb`, `api/swagger/v1/openapi.yaml`
- Test: `api/test/models/work_update_test.rb`, `api/test/controllers/api/v1/work_updates_controller_test.rb`, `api/test/services/context_pulse_service_test.rb` (extend), `api/test/fixtures/work_updates.yml`

**CLI**
- Modify: `cli/src/types/api.rs` (`CreateWorkUpdateParams`), `cli/src/services/api_client.rs` (`create_work_update`), `cli/src/commands/mcp.rs` (`ShareUpdateInput` + `share_update` tool), `cli/src/services/hook_service.rs` (`format_pulse` updates section)

**Dashboard**
- Create: `dashboard/src/lib/api/work-updates.ts`, `dashboard/src/hooks/use-work-updates.ts`, `dashboard/src/components/features/projects/work-updates-panel.tsx`, `dashboard/src/components/features/projects/__tests__/work-updates-panel.test.tsx`
- Modify: `dashboard/src/app/kit/projects/[id]/page.tsx`, `dashboard/src/test/mocks/handlers.ts`

---

## Task 1: `WorkUpdate` model + migration

**Files:**
- Create: `api/db/migrate/20260622000001_create_work_updates.rb`, `api/app/models/work_update.rb`, `api/test/models/work_update_test.rb`
- Modify: `api/app/models/project.rb`

- [ ] **Step 1: Write the migration**

```ruby
class CreateWorkUpdates < ActiveRecord::Migration[8.1]
  def change
    create_table :work_updates do |t|
      t.uuid :external_id, default: -> { "gen_random_uuid()" }, null: false
      t.references :organization, null: false, foreign_key: true
      t.references :project, null: false, foreign_key: true
      t.references :user, null: false, foreign_key: true
      # source session drives the privacy gate; nullify so a deleted session leaves the update intact
      t.references :llm_session, null: true, foreign_key: { on_delete: :nullify }
      t.references :blueprint_task, null: true, foreign_key: { on_delete: :nullify }
      t.string :body, null: false

      t.timestamps

      t.index :external_id, unique: true
      t.index [ :project_id, :created_at ]
    end
  end
end
```

> If `bin/rails db:migrate` reports a version collision, bump the timestamp prefix to one later than the newest file in `api/db/migrate/`. Keep the class name `CreateWorkUpdates`.

- [ ] **Step 2: Run the migration**

Run: `cd api && bin/rails db:migrate`
Expected: `work_updates` table created; `db/schema.rb` updated.

- [ ] **Step 3: Write the failing model test**

```ruby
require "test_helper"

class WorkUpdateTest < ActiveSupport::TestCase
  setup do
    @org = organizations(:acme)
    @project = projects(:customer_portal)
    @user = users(:alice)
  end

  def build_update(attrs = {})
    @project.work_updates.new({ organization: @org, user: @user, body: "switched to OAuth refactor" }.merge(attrs))
  end

  test "valid with a body" do
    assert build_update.valid?
  end

  test "requires a body" do
    u = build_update(body: "")
    assert_not u.valid?
    assert u.errors[:body].any?
  end

  test "rejects a body over the limit" do
    u = build_update(body: "x" * (WorkUpdate::MAX_BODY_CHARS + 1))
    assert_not u.valid?
  end

  test "recallable? is true with no session" do
    assert build_update(llm_session: nil).recallable?
  end

  test "recallable? is false when tied to a sensitive session" do
    s = llm_sessions(:alice_active_conversation)
    s.update!(sensitive: true)
    assert_not build_update(llm_session: s).recallable?
  end
end
```

- [ ] **Step 4: Run it, confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/models/work_update_test.rb`
Expected: FAIL — uninitialized constant `WorkUpdate`.

- [ ] **Step 5: Write the model**

```ruby
class WorkUpdate < ApplicationRecord
  belongs_to :organization
  belongs_to :project
  belongs_to :user
  belongs_to :llm_session, optional: true
  belongs_to :blueprint_task, optional: true

  MAX_BODY_CHARS = 280

  validates :body, presence: true, length: { maximum: MAX_BODY_CHARS }

  scope :recent, -> { order(created_at: :desc) }

  def to_param
    external_id
  end

  # Hidden only when tied to a non-recallable (sensitive / private-repo) session.
  # No session ⇒ nothing to gate ⇒ visible. Reuses the single source of truth.
  def recallable?
    llm_session.nil? || SaveState.session_recallable?(llm_session)
  end
end
```

- [ ] **Step 6: Add the project association**

In `api/app/models/project.rb`, alongside the other `has_many` declarations:

```ruby
  has_many :work_updates, dependent: :destroy
```

- [ ] **Step 7: Run it, confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/models/work_update_test.rb`
Expected: PASS (5 runs, 0 failures).

- [ ] **Step 8: Commit**

```bash
git add api/db/migrate/20260622000001_create_work_updates.rb api/db/schema.rb api/app/models/work_update.rb api/app/models/project.rb api/test/models/work_update_test.rb
git commit -m "feat(sessions): add WorkUpdate model"
```

---

## Task 2: `WorkUpdatesController` (create + index) + route + policy

**Files:**
- Create: `api/app/controllers/api/v1/work_updates_controller.rb`, `api/app/policies/work_update_policy.rb`, `api/test/controllers/api/v1/work_updates_controller_test.rb`
- Modify: `api/config/routes.rb`

- [ ] **Step 1: Add the route**

In `api/config/routes.rb`, right after the `active_work` project scope (near line 540), add:

```ruby
        # Work updates — agent broadcasts ("what I'm working on"), project feed
        scope "projects/:project_id" do
          resources :work_updates, only: %i[index create],
                    controller: "work_updates", param: :external_id
        end
```

- [ ] **Step 2: Write the policy**

```ruby
# frozen_string_literal: true

class WorkUpdatePolicy < ApplicationPolicy
  class Scope < ApplicationPolicy::Scope
    def resolve
      organization_scoped(scope)
    end
  end

  # Any member can read the feed.
  def index?
    current_organization.present? && member_viewer?
  end

  # Posting an update is a write — developer and up.
  def create?
    current_organization.present? && member_developer?
  end

  private

  def member_viewer?
    belongs_to_organization? && viewer?
  end

  def member_developer?
    belongs_to_organization? && developer?
  end
end
```

> Match the actual helper names in `SaveStatePolicy`/`ApplicationPolicy` (it uses `class_or_member? { viewer? }` / `{ developer? }` and `belongs_to_organization?`). If those helpers differ, mirror `SaveStatePolicy#index?`/`#personal?` exactly rather than the sketch above.

- [ ] **Step 3: Write the failing controller test**

```ruby
require "test_helper"

module Api
  module V1
    class WorkUpdatesControllerTest < ActionDispatch::IntegrationTest
      setup do
        @org = organizations(:acme)
        @project = projects(:customer_portal)
        @alice = users(:alice)
        @session = llm_sessions(:alice_active_conversation)
        @token = jwt_token_for(@alice)
      end

      def path
        "/#{@org.external_id}/projects/#{@project.external_id}/work_updates"
      end

      test "creates an update" do
        assert_difference -> { WorkUpdate.count }, 1 do
          post path, params: { note: "switched to the OAuth refactor", source_session_id: @session.external_id },
               headers: auth_headers(@token)
        end
        assert_response :created
        assert_equal "switched to the OAuth refactor", JSON.parse(response.body).dig("data", "note")
      end

      test "rejects a blank note" do
        post path, params: { note: "  " }, headers: auth_headers(@token)
        assert_response :unprocessable_entity
      end

      test "refuses to broadcast from a sensitive session" do
        @session.update!(sensitive: true)
        post path, params: { note: "secret work", source_session_id: @session.external_id },
             headers: auth_headers(@token)
        assert_response :unprocessable_entity
      end

      test "posts without a link when task_id is unresolvable" do
        post path, params: { note: "progress", task_id: "task-does-not-exist" }, headers: auth_headers(@token)
        assert_response :created
        assert_nil JSON.parse(response.body).dig("data", "task")
      end

      test "index returns the recent feed" do
        @project.work_updates.create!(organization: @org, user: @alice, body: "earlier note")
        get path, headers: auth_headers(@token)
        assert_response :success
        assert JSON.parse(response.body)["data"].any? { |u| u["note"] == "earlier note" }
      end

      test "index hides updates from a sensitive session" do
        @project.work_updates.create!(organization: @org, user: @alice, llm_session: @session, body: "gated note")
        @session.update!(sensitive: true)
        get path, headers: auth_headers(@token)
        notes = JSON.parse(response.body)["data"].map { |u| u["note"] }
        assert_not_includes notes, "gated note"
      end

      test "requires authentication" do
        get path
        assert_response :unauthorized
      end

      test "404 for a project in another org" do
        other = projects(:admin_dashboard) # techstart project
        get "/#{@org.external_id}/projects/#{other.external_id}/work_updates", headers: auth_headers(@token)
        assert_response :not_found
      end
    end
  end
end
```

> Confirm `projects(:admin_dashboard)` belongs to a different org (techstart); if not, pick any non-acme project fixture. Confirm `BlueprintTask` has an `external_id` column (used by the controller's task lookup); if it uses `id`, adjust `lookup_task`.

- [ ] **Step 4: Run it, confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/controllers/api/v1/work_updates_controller_test.rb`
Expected: FAIL — routing/controller missing.

- [ ] **Step 5: Write the controller**

```ruby
# frozen_string_literal: true

module Api
  module V1
    # WorkUpdatesController — agent work-update broadcasts for a project.
    # create: an agent posts a short note (privacy-gated on its source session).
    # index: the recent gated feed for the dashboard recap.
    class WorkUpdatesController < BaseController
      before_action :set_project

      DEFAULT_FEED = 50
      MAX_FEED = 100

      # GET /:org_id/projects/:project_id/work_updates?limit=
      def index
        authorize WorkUpdate

        limit = parse_limit
        updates = @project.work_updates.recent
                          .includes(:user, :blueprint_task, :llm_session)
                          .limit(limit * 2) # over-fetch so the gate can't starve the feed
                          .select(&:recallable?)
                          .first(limit)

        render json: { data: updates.map { |u| serialize(u) } }
      end

      # POST /:org_id/projects/:project_id/work_updates  { note, task_id?, source_session_id? }
      def create
        authorize WorkUpdate, :create?

        note = ContentSanitizer.clean(params[:note].to_s)
        return render_error(:unprocessable_entity, "note is required") if note.blank?

        session = lookup_session(params[:source_session_id])
        unless WorkUpdate.new(llm_session: session).recallable?
          return render_error(:unprocessable_entity,
            "source session is sensitive or private and cannot broadcast")
        end

        update = @project.work_updates.new(
          organization: current_organization,
          user: current_user,
          llm_session: session,
          blueprint_task: lookup_task(params[:task_id]),
          body: note[0, WorkUpdate::MAX_BODY_CHARS]
        )

        return render_validation_error(update) unless update.save

        render json: { data: serialize(update) }, status: :created
      end

      private

      def set_project
        @project = current_organization.projects.find_by!(external_id: params[:project_id])
      end

      def parse_limit
        n = params[:limit].to_i
        return DEFAULT_FEED if n <= 0

        [ n, MAX_FEED ].min
      end

      def lookup_session(external_id)
        return nil if external_id.blank?

        @project.llm_sessions.find_by(external_id: external_id)
      end

      def lookup_task(external_id)
        return nil if external_id.blank?

        BlueprintTask.joins(:blueprint)
                     .where(blueprints: { project_id: @project.id })
                     .find_by(external_id: external_id)
      end

      def serialize(update)
        {
          external_id: update.external_id,
          user: { name: update.user&.name },
          note: update.body,
          task: update.blueprint_task && {
            external_id: update.blueprint_task.external_id,
            name: update.blueprint_task.name
          },
          created_at: update.created_at.iso8601
        }
      end
    end
  end
end
```

> `render_error` and `render_validation_error` are the same helpers `SaveStatesController` uses — confirm their signatures there and match.

- [ ] **Step 6: Run it, confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/controllers/api/v1/work_updates_controller_test.rb`
Expected: PASS (8 runs, 0 failures).

- [ ] **Step 7: Commit**

```bash
git add api/app/controllers/api/v1/work_updates_controller.rb api/app/policies/work_update_policy.rb api/config/routes.rb api/test/controllers/api/v1/work_updates_controller_test.rb
git commit -m "feat(sessions): add work_updates create + index endpoints"
```

---

## Task 3: `ContextPulseService` `updates` source

**Files:**
- Modify: `api/app/services/context_pulse_service.rb`
- Test: `api/test/services/context_pulse_service_test.rb`

- [ ] **Step 1: Add the failing test**

Append to `ContextPulseServiceTest`:

```ruby
  test "includes teammates' work updates since `since`, excludes the caller's own" do
    @project = projects(:customer_portal)
    @project.work_updates.create!(organization: @project.organization, user: users(:alice),
                                  body: "alice switched to OAuth")
    @project.work_updates.create!(organization: @project.organization, user: users(:charlie),
                                  body: "charlie's own note")
    result = ContextPulseService.for_session(session: @caller_session, since: 1.hour.ago)
    notes = result[:updates].map { |u| u[:note] }
    assert_includes notes, "alice switched to OAuth"
    refute_includes notes, "charlie's own note"   # caller is charlie
  end

  test "excludes work updates from a sensitive session" do
    @project = projects(:customer_portal)
    sess = llm_sessions(:alice_active_conversation)
    @project.work_updates.create!(organization: @project.organization, user: users(:alice),
                                  llm_session: sess, body: "gated update")
    sess.update!(sensitive: true)
    result = ContextPulseService.for_session(session: @caller_session, since: 1.hour.ago)
    refute_includes result[:updates].map { |u| u[:note] }, "gated update"
  end
```

> `@caller_session` is `llm_sessions(:charlie_conversation)` per the existing setup in this test file.

- [ ] **Step 2: Run it, confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/context_pulse_service_test.rb`
Expected: FAIL — `result[:updates]` is nil.

- [ ] **Step 3: Add the `updates` source**

In `api/app/services/context_pulse_service.rb`, change `build` to include `updates` (both the populated and the no-project branch):

```ruby
  def build
    base = { as_of: Time.current.iso8601, since: @since.iso8601 }
    return base.merge(conversations: [], tasks: [], team_states: [], updates: []) if @project.nil?

    base.merge(conversations: conversations, tasks: tasks, team_states: team_states, updates: updates)
  end
```

And add the private method (next to `team_states`):

```ruby
  def updates
    # Agent-authored short broadcasts. The note is intentionally included verbatim
    # (deliberately shareable text), unlike the auto-generated short_summary we drop.
    @project.work_updates
            .where("work_updates.created_at >= ?", @since)
            .where.not(user_id: @session.user_id)
            .includes(:user, :blueprint_task, :llm_session)
            .order(created_at: :desc)
            .limit(MAX_ITEMS)
            .select(&:recallable?)
            .map do |u|
      { user: u.user&.name.presence || "a teammate",
        note: u.body,
        task: u.blueprint_task&.name,
        created_at: u.created_at.iso8601 }
    end
  end
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/context_pulse_service_test.rb`
Expected: PASS (all, including the 2 new).

- [ ] **Step 5: Commit**

```bash
git add api/app/services/context_pulse_service.rb api/test/services/context_pulse_service_test.rb
git commit -m "feat(sessions): surface work updates in the context pulse"
```

---

## Task 4: CLI `create_work_update` + `share_update` MCP tool

**Files:**
- Modify: `cli/src/types/api.rs`, `cli/src/services/api_client.rs`, `cli/src/commands/mcp.rs`

- [ ] **Step 1: Add the params type**

In `cli/src/types/api.rs`, next to `SavePersonalStateParams`:

```rust
#[derive(Debug, serde::Serialize)]
pub struct CreateWorkUpdateParams {
    pub note: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub task_id: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub source_session_id: Option<String>,
}
```

- [ ] **Step 2: Add the failing api_client test**

In the `#[cfg(test)] mod tests` of `cli/src/services/api_client.rs` (mirror the `get_context_pulse_raw` test idiom — note the `/api/v1` prefix in the mock path):

```rust
#[tokio::test]
async fn create_work_update_posts_to_project_endpoint() {
    use wiremock::matchers::{method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    let server = MockServer::start().await;
    let client = CrewkitApiClient::with_base_url(server.uri());
    Mock::given(method("POST"))
        .and(path("/api/v1/org-1/projects/proj-1/work_updates"))
        .respond_with(ResponseTemplate::new(201).set_body_json(serde_json::json!({
            "data": { "external_id": "wu-1", "note": "did a thing" }
        })))
        .mount(&server)
        .await;

    let params = crate::types::CreateWorkUpdateParams {
        note: "did a thing".into(), task_id: None, source_session_id: Some("sess-1".into()),
    };
    let v = client.create_work_update("org-1", "proj-1", &params).await.unwrap();
    assert_eq!(v["data"]["external_id"], "wu-1");
}
```

- [ ] **Step 3: Run it, confirm it fails**

Run: `cd cli && cargo test --bin crewkit create_work_update`
Expected: FAIL — method missing.

- [ ] **Step 4: Add the client method**

In `cli/src/services/api_client.rs`, near `save_personal_state`:

```rust
    /// Post an agent work-update broadcast for a project.
    ///
    /// POST /:org_id/projects/:project_id/work_updates
    pub async fn create_work_update(
        &self,
        org_id: &str,
        project_id: &str,
        params: &crate::types::CreateWorkUpdateParams,
    ) -> Result<serde_json::Value> {
        let path = format!("/{}/projects/{}/work_updates", org_id, project_id);
        self.post(&path, params).await
    }
```

> Use whatever generic POST helper the client already has. If `self.post` doesn't return `Result<serde_json::Value>`, mirror the exact idiom of an existing POST (e.g. how `save_personal_state` calls `self.put`) — return the parsed body.

- [ ] **Step 5: Run it, confirm it passes**

Run: `cd cli && cargo test --bin crewkit create_work_update`
Expected: PASS.

- [ ] **Step 6: Add the `share_update` MCP tool**

In `cli/src/commands/mcp.rs`, add the input struct near `SaveStateInput`:

```rust
#[derive(Debug, serde::Deserialize, schemars::JsonSchema)]
pub struct ShareUpdateInput {
    /// One short, high-level, non-confidential line about what you just changed or
    /// finished — e.g. "switched to the OAuth refactor" or "finished the token migration".
    pub note: String,
    /// Optional external_id of the Blueprint task this update relates to.
    #[serde(default)]
    pub task_id: Option<String>,
}
```

And add the tool method inside the `#[tool_router]` impl block (next to `save_state`):

```rust
    #[tool(
        description = "Broadcast a SHORT work update to the project so teammates' open sessions and the dashboard see what you're doing. Call it when you CHANGE focus or hit a MILESTONE (started or finished something). One short, high-level, non-confidential line — no user approval needed. Optionally pass the Blueprint task_id it relates to."
    )]
    async fn share_update(
        &self,
        Parameters(input): Parameters<ShareUpdateInput>,
    ) -> std::result::Result<CallToolResult, ErrorData> {
        let scope = match self.ctx.ready().await {
            Ok(s) => s,
            Err(msg) => return Ok(tool_error(msg)),
        };
        let Some(project_id) = scope.project_id.as_deref() else {
            return Ok(tool_error(NO_PROJECT_MSG.to_string()));
        };

        let params = crate::types::CreateWorkUpdateParams {
            note: input.note,
            task_id: input.task_id,
            source_session_id: self.ctx.session_id.clone(),
        };

        match self
            .ctx
            .api
            .create_work_update(&scope.org_id, project_id, &params)
            .await
        {
            Ok(_) => Ok(tool_text("Shared your update with the project.".to_string())),
            Err(e) => Ok(tool_error(api_error_message(&e))),
        }
    }
```

> Match the surrounding tool methods exactly (the `self.ctx.ready()`, `tool_error`/`tool_text`/`api_error_message`, `NO_PROJECT_MSG` helpers all exist and are used by `save_state`/`load_state`).

- [ ] **Step 7: Verify build + checks**

Run: `cd cli && cargo test --bin crewkit create_work_update && just fmt && just lint && just check`
Expected: PASS; no NEW clippy warnings (a pre-existing `tui/mod.rs` `collapsible_match` warning is unrelated).

- [ ] **Step 8: Commit**

```bash
git add cli/src/types/api.rs cli/src/services/api_client.rs cli/src/commands/mcp.rs
git commit -m "feat(cli): add share_update MCP tool + work_update API client"
```

---

## Task 5: Render work updates in `format_pulse`

**Files:**
- Modify: `cli/src/services/hook_service.rs`

- [ ] **Step 1: Add the failing test**

In the `hook_service.rs` tests, extend the populated-pulse test (or add one) asserting updates render:

```rust
#[test]
fn format_pulse_renders_work_updates() {
    let v = serde_json::json!({"data":{
        "conversations":[], "tasks":[], "team_states":[],
        "updates":[{"user":"Alice","note":"switched to OAuth refactor","task":"Q3 Auth"}]
    }});
    let out = super::format_pulse(&v).unwrap();
    assert!(out.contains("Alice"));
    assert!(out.contains("switched to OAuth refactor"));
    assert!(out.contains("Q3 Auth"));
}
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `cd cli && cargo test --bin crewkit format_pulse`
Expected: FAIL — updates not rendered.

- [ ] **Step 3: Add the updates branch to `format_pulse`**

In `cli/src/services/hook_service.rs`, inside `format_pulse`, after the `team_states` block and before the `lines.is_empty()` check:

```rust
    if let Some(arr) = data.get("updates").and_then(|u| u.as_array()) {
        for u in arr {
            let who = u.get("user").and_then(|x| x.as_str()).unwrap_or("a teammate");
            let note = u.get("note").and_then(|x| x.as_str()).unwrap_or("");
            if note.is_empty() {
                continue;
            }
            match u.get("task").and_then(|t| t.as_str()) {
                Some(task) => lines.push(format!("• {} — {} (task: {})", who, note, task)),
                None => lines.push(format!("• {} — {}", who, note)),
            }
        }
    }
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `cd cli && cargo test --bin crewkit format_pulse && just fmt && just lint && just check`
Expected: PASS; clippy clean (no new warnings).

- [ ] **Step 5: Commit**

```bash
git add cli/src/services/hook_service.rs
git commit -m "feat(cli): render work updates in the context pulse"
```

---

## Task 6: Dashboard recap panel

**Files:**
- Create: `dashboard/src/lib/api/work-updates.ts`, `dashboard/src/hooks/use-work-updates.ts`, `dashboard/src/components/features/projects/work-updates-panel.tsx`, `dashboard/src/components/features/projects/__tests__/work-updates-panel.test.tsx`
- Modify: `dashboard/src/app/kit/projects/[id]/page.tsx`, `dashboard/src/test/mocks/handlers.ts`

- [ ] **Step 1: API client module**

Create `dashboard/src/lib/api/work-updates.ts`:

```ts
import { apiClient } from "./client";

export interface WorkUpdate {
  external_id: string;
  user: { name: string | null };
  note: string;
  task: { external_id: string; name: string } | null;
  created_at: string;
}

export interface WorkUpdateListResponse {
  data: WorkUpdate[];
}

export const workUpdatesApi = {
  list: (orgId: string, projectId: string, limit = 50): Promise<WorkUpdateListResponse> =>
    apiClient<WorkUpdateListResponse>(
      `/api/v1/${orgId}/projects/${encodeURIComponent(projectId)}/work_updates?limit=${limit}`,
      { method: "GET" },
    ),
};
```

- [ ] **Step 2: Query hook**

Create `dashboard/src/hooks/use-work-updates.ts` (mirror `use-active-work.ts`, 30s poll):

```ts
"use client";

import { useQuery } from "@tanstack/react-query";
import { workUpdatesApi, type WorkUpdateListResponse } from "@/lib/api/work-updates";
import { useOrganization } from "@/lib/auth/organization-context";

export const workUpdateKeys = {
  all: ["work-updates"] as const,
  list: (orgId: string, projectId: string) =>
    [...workUpdateKeys.all, orgId, projectId] as const,
};

const POLL_INTERVAL_MS = 30_000;

export function useWorkUpdates(projectId: string | null | undefined) {
  const { currentOrganization } = useOrganization();
  const orgId = currentOrganization?.id || "";
  return useQuery<WorkUpdateListResponse>({
    queryKey: workUpdateKeys.list(orgId, projectId || ""),
    queryFn: () => workUpdatesApi.list(orgId, projectId as string),
    enabled: !!orgId && !!projectId,
    refetchInterval: POLL_INTERVAL_MS,
    refetchIntervalInBackground: false,
    staleTime: 10_000,
  });
}
```

- [ ] **Step 3: Write the failing component test**

Create `dashboard/src/components/features/projects/__tests__/work-updates-panel.test.tsx`:

```tsx
import { render, screen, waitFor } from "@/test/test-utils";
import { WorkUpdatesPanel } from "../work-updates-panel";

describe("WorkUpdatesPanel", () => {
  it("renders updates from the feed", async () => {
    render(<WorkUpdatesPanel projectId="proj-1" />);
    await waitFor(() =>
      expect(screen.getByText(/switched to the OAuth refactor/i)).toBeInTheDocument(),
    );
  });
});
```

- [ ] **Step 4: Add the MSW handler + run to confirm it fails**

In `dashboard/src/test/mocks/handlers.ts`, add to the appropriate group:

```ts
  http.get("*/api/v1/:orgId/projects/:projectId/work_updates", () =>
    HttpResponse.json({
      data: [
        {
          external_id: "wu-1",
          user: { name: "Alice" },
          note: "switched to the OAuth refactor",
          task: { external_id: "task-1", name: "Q3 Auth" },
          created_at: "2026-06-22T12:00:00Z",
        },
      ],
    }),
  ),
```

Run: `cd dashboard && npx vitest run src/components/features/projects/__tests__/work-updates-panel.test.tsx`
Expected: FAIL — `WorkUpdatesPanel` not found.

- [ ] **Step 5: Write the component**

Create `dashboard/src/components/features/projects/work-updates-panel.tsx`:

```tsx
"use client";

import { useWorkUpdates } from "@/hooks/use-work-updates";

function relativeTime(iso: string): string {
  const diffMs = Date.now() - new Date(iso).getTime();
  const mins = Math.round(diffMs / 60000);
  if (mins < 1) return "just now";
  if (mins < 60) return `${mins}m ago`;
  const hrs = Math.round(mins / 60);
  if (hrs < 24) return `${hrs}h ago`;
  return `${Math.round(hrs / 24)}d ago`;
}

export function WorkUpdatesPanel({ projectId }: { projectId: string }) {
  const { data, isLoading } = useWorkUpdates(projectId);
  const updates = data?.data ?? [];

  return (
    <section className="rounded-lg border p-4">
      <h2 className="mb-3 text-sm font-medium text-muted-foreground">Team activity</h2>
      {isLoading ? (
        <p className="text-sm text-muted-foreground">Loading…</p>
      ) : updates.length === 0 ? (
        <p className="text-sm text-muted-foreground">No recent updates from the team.</p>
      ) : (
        <ul className="space-y-2">
          {updates.map((u) => (
            <li key={u.external_id} className="text-sm">
              <span className="font-medium">{u.user.name ?? "A teammate"}</span>{" "}
              <span>{u.note}</span>
              {u.task && (
                <span className="ml-2 rounded bg-muted px-1.5 py-0.5 text-xs">{u.task.name}</span>
              )}
              <span className="ml-2 text-xs text-muted-foreground">{relativeTime(u.created_at)}</span>
            </li>
          ))}
        </ul>
      )}
    </section>
  );
}
```

- [ ] **Step 6: Run it, confirm it passes**

Run: `cd dashboard && npx vitest run src/components/features/projects/__tests__/work-updates-panel.test.tsx`
Expected: PASS.

- [ ] **Step 7: Wire it into the project page**

In `dashboard/src/app/kit/projects/[id]/page.tsx`, import and render `WorkUpdatesPanel` beside the existing `SaveStatesPanel` (pass the same `projectId` prop those panels receive):

```tsx
import { WorkUpdatesPanel } from "@/components/features/projects/work-updates-panel";
// ...near the SaveStatesPanel usage:
<WorkUpdatesPanel projectId={project.id} />
```

> Use the same `projectId` value the sibling panels use (the project's external id from the page data).

- [ ] **Step 8: Run lint/typecheck + the touched tests**

Run: `cd dashboard && npm run lint && npm run typecheck && npx vitest run src/components/features/projects/__tests__/work-updates-panel.test.tsx`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add dashboard/src/lib/api/work-updates.ts dashboard/src/hooks/use-work-updates.ts dashboard/src/components/features/projects/work-updates-panel.tsx dashboard/src/components/features/projects/__tests__/work-updates-panel.test.tsx dashboard/src/app/kit/projects/[id]/page.tsx dashboard/src/test/mocks/handlers.ts
git commit -m "feat(dashboard): add team activity (work updates) panel"
```

---

## Task 7: OpenAPI docs

**Files:**
- Modify: `api/swagger/v1/openapi.yaml`

- [ ] **Step 1: Document both endpoints**

Add a `/{organization_id}/projects/{project_id}/work_updates` path with:
- `GET` (index): `limit` query param; 200 → `{ data: [ { external_id, user: {name}, note, task: {external_id,name}|null, created_at } ] }`.
- `POST` (create): body `{ note (required), task_id?, source_session_id? }`; 201 → `{ data: <one update> }`; 401/403/422.

Mirror the structure/param refs of the nearby `active_work` / session member paths (copy their `OrganizationId` param ref, `project_id` path param, `security`, and shared error `$ref`s).

- [ ] **Step 2: Validate**

Run: `cd api && bin/rails openapi:validate STRICT=1`
Expected: PASS — no stale entries; `work_updates` no longer in the undocumented list.

- [ ] **Step 3: Regenerate dashboard types (contract)**

Run: `cd dashboard && npm run generate:api-types && npm run validate:api-contract`
Expected: types regenerate and validate (commit the regenerated `src/types/api-generated.d.ts` if it changed).

- [ ] **Step 4: Commit**

```bash
git add api/swagger/v1/openapi.yaml dashboard/src/types/api-generated.d.ts
git commit -m "docs(api): document work_updates endpoints"
```

---

## Final verification

- [ ] **API affected suites**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/models/work_update_test.rb test/controllers/api/v1/work_updates_controller_test.rb test/services/context_pulse_service_test.rb`
Expected: all PASS.

- [ ] **CLI checks**

Run: `cd cli && cargo test --bin crewkit work_update && cargo test --bin crewkit format_pulse && just fmt && just lint && just check`
Expected: PASS; no new clippy warnings.

- [ ] **Dashboard checks**

Run: `cd dashboard && npm run lint && npm run typecheck && npx vitest run src/components/features/projects/__tests__/work-updates-panel.test.tsx`
Expected: PASS.

- [ ] **Security review** (team practice): run `security-expert` over the full diff — focus on the write privacy-gate (no broadcast from sensitive/private sessions), the read gate on index + pulse, tenant isolation on the project-scoped routes, and that the `note` is sanitized and length-capped.

---

## Self-review notes (author)

- **Spec coverage:** model (T1), send tool+API (T2,T4), receive/pulse (T3,T5), dashboard recap (T6), docs (T7), privacy gate on write+read (T2,T3), tenancy (T2). All spec sections map to a task.
- **Locked decisions honored:** `WorkUpdate` name; no per-broadcast approval (tool posts directly); free-text note + optional task link, no `kind`; unresolvable `task_id` → posted without link (T2 test); `MAX_BODY_CHARS=280`.
- **Deferred-to-implementer specifics** are limited to matching local conventions (policy helper names vs `SaveStatePolicy`, `render_error` signatures, the generic POST helper, `BlueprintTask#external_id`, the project page's `projectId` prop, OpenAPI block style) — each names the exact sibling to copy.
- **Token follow-up** (minimizing the pulse's per-prompt cost) is intentionally out of scope, per the spec.
