# Cross-Session Awareness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Get fresh cross-agent context to (a) humans, via the team digest posted to a Slack channel, and (b) open agent sessions, via a throttled context-pulse injected on each prompt — so a session left open between jobs stops going stale.

**Architecture:** Phase 1 reuses the existing `TeamDigestService` → `SaveState` publish flow and the Phase B Slack sender; a `SaveState` `after_commit` callback enqueues a delivery job when (and only when) a team digest transitions to published. Phase 2 adds a session-scoped `context_pulse` endpoint that returns a privacy-gated delta of teammates' active work + newly-published digests; the CLI's existing `UserPromptSubmit` hook handler fetches it (throttled, fail-open) and appends it to the context it already injects.

**Tech Stack:** Rails 8.1 (Minitest, ActiveJob), Rust CLI (tokio, reqwest, wiremock), existing `SlackSenderService` / `AgentIdentity` / `SlackWorkspaceInstallation` plumbing.

---

## Refinements from the spec (discovered during code review — flag if any is wrong)

1. **Publish detection lives in a `SaveState` `after_commit` callback**, not in two call sites. This single hook covers both publish paths — `TeamDigestService` auto-publish *and* the manager-review publish in `SaveStatesController#update_team` (`api/app/controllers/api/v1/save_states_controller.rb:74`). DRYer and closes the require_review gap the spec left open.
2. **Channel binding:** `project.settings["digest_slack_channel_id"]` overrides, falling back to the project's `AgentIdentity#slack_channel_id`. Dropped the org-level default from the spec — posting needs an `AgentIdentity` anyway (it resolves the bot token via `SlackWorkspaceInstallation`), so the agent's own channel is the natural fallback and no new token plumbing is invented. `Project#settings` is an existing jsonb column (`api/db/schema.rb`), so **no migration**.
3. **Phase 2 is implemented inside the existing `UserPromptSubmit` handler in `HookService`**, not a new `crewkit context-pulse` command + `settings.json` hook registration. crewkit already wires `UserPromptSubmit → crewkit internal hook-handler → HookService` and already injects `additionalContext` there (`cli/src/services/hook_service.rs:729`). Throttle state lives in-memory on the long-lived `HookService` (the main process spans the whole session), so **no `.crewkit/` watermark file** is needed.
4. **The pulse endpoint is session-scoped** (`/sessions/:id/context_pulse`), not project-scoped. `HookService` holds `org_id` + `llm_session_id` but not `project_id`; the server derives the project (and the user to exclude) from the session.

---

## File Structure

**Phase 1 (API):**
- Modify: `api/app/models/save_state.rb` — `after_commit` enqueue on publish transition
- Create: `api/app/jobs/post_digest_to_slack_job.rb` — format + deliver, soft-fail
- Test: `api/test/models/save_state_slack_delivery_test.rb`, `api/test/jobs/post_digest_to_slack_job_test.rb`

**Phase 2 (API + CLI):**
- Create: `api/app/services/context_pulse_service.rb` — privacy-gated delta builder
- Create: `api/app/controllers/api/v1/session_context_pulse_controller.rb` — `GET .../context_pulse`
- Modify: `api/config/routes.rb` — add the route
- Test: `api/test/services/context_pulse_service_test.rb`, `api/test/controllers/api/v1/session_context_pulse_controller_test.rb`
- Modify: `cli/src/services/api_client.rs` — `get_context_pulse_raw`
- Modify: `cli/src/services/hook_service.rs` — throttle + fetch + merge into `UserPromptSubmit` context
- Modify: `api/swagger/v1/openapi.yaml` — document the endpoint

---

# Phase 1 — Slack digest delivery

## Task 1: `PostDigestToSlackJob`

**Files:**
- Create: `api/app/jobs/post_digest_to_slack_job.rb`
- Test: `api/test/jobs/post_digest_to_slack_job_test.rb`

- [ ] **Step 1: Write the failing test**

```ruby
require "test_helper"

class PostDigestToSlackJobTest < ActiveJob::TestCase
  setup do
    @project = projects(:customer_portal)
    @agent = agent_identities(:acme_frontend_bot) # acme / customer_portal, status active, has slack_workspace_id
    @agent.update!(slack_channel_id: "C0DIGEST")
    @state = @project.organization.save_states.create!(
      project: @project, kind: SaveState::TEAM_DAILY,
      handoff: "## Team digest\n- Alice shipped the OAuth fix",
      digest_date: Date.new(2026, 6, 21), published_at: Time.current,
      metadata: { "contributors" => %w[Alice Bob] }
    )
  end

  test "posts the digest to the project agent's channel" do
    captured = nil
    stub = ->(agent:, channel:, text:, thread_ts: nil) { captured = { agent: agent, channel: channel, text: text }; { delivered: true } }
    SlackSenderService.stub(:post_reply, stub) do
      PostDigestToSlackJob.perform_now(@state.id)
    end
    assert_equal "C0DIGEST", captured[:channel]
    assert_equal @agent.id, captured[:agent].id
    assert_includes captured[:text], "OAuth fix"
  end

  test "prefers the project setting channel over the agent channel" do
    @project.update!(settings: { "digest_slack_channel_id" => "C0OVERRIDE" })
    captured = nil
    SlackSenderService.stub(:post_reply, ->(channel:, **) { captured = channel; { delivered: true } }) do
      PostDigestToSlackJob.perform_now(@state.id)
    end
    assert_equal "C0OVERRIDE", captured
  end

  test "skips when no active agent with a workspace exists" do
    AgentIdentity.where(project: @project).update_all(status: 2) # archived
    called = false
    SlackSenderService.stub(:post_reply, ->(**) { called = true; { delivered: true } }) do
      PostDigestToSlackJob.perform_now(@state.id)
    end
    assert_not called
  end

  test "skips an unpublished draft" do
    @state.update_columns(published_at: nil)
    called = false
    SlackSenderService.stub(:post_reply, ->(**) { called = true }) do
      PostDigestToSlackJob.perform_now(@state.id)
    end
    assert_not called
  end

  test "soft-fails on a Slack send error" do
    SlackSenderService.stub(:post_reply, ->(**) { raise SlackSenderService::SendError, "channel_not_found" }) do
      assert_nothing_raised { PostDigestToSlackJob.perform_now(@state.id) }
    end
  end
end
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/jobs/post_digest_to_slack_job_test.rb`
Expected: FAIL — `NameError: uninitialized constant PostDigestToSlackJob`.

- [ ] **Step 3: Implement the job**

```ruby
# frozen_string_literal: true

# PostDigestToSlackJob — delivers a published team digest SaveState to the
# project's Slack channel via the project AgentIdentity (Phase B sender). Enqueued
# by SaveState#after_commit only on the unpublished→published transition.
#
# Soft-fails (logs, no raise) when there is no agent, no channel, or Slack
# rejects the message — a failed digest post must never break the publish flow.
class PostDigestToSlackJob < ApplicationJob
  queue_as :default

  def perform(save_state_id)
    state = SaveState.find_by(id: save_state_id)
    return if state.nil? || state.published_at.blank? || !state.team?

    project = state.project
    agent = project.agent_identities.active_agents.where.not(slack_workspace_id: [ nil, "" ]).first
    return if agent.nil?

    channel = project.settings["digest_slack_channel_id"].presence || agent.slack_channel_id.presence
    return if channel.blank?

    SlackSenderService.post_reply(agent: agent, channel: channel, text: format_message(state))
  rescue SlackSenderService::SendError => e
    Rails.logger.warn("[PostDigestToSlackJob] state #{save_state_id}: #{e.message}")
  end

  private

  def format_message(state)
    contributors = Array(state.metadata["contributors"])
    suffix = contributors.size == 1 ? "" : "s"
    header = ":clipboard: *Team digest — #{state.project.name}* " \
             "(#{state.digest_date}, #{contributors.size} contributor#{suffix})"
    "#{header}\n\n#{state.handoff}"
  end
end
```

- [ ] **Step 4: Run it and confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/jobs/post_digest_to_slack_job_test.rb`
Expected: PASS (5 runs, 0 failures).

- [ ] **Step 5: Commit**

```bash
git add api/app/jobs/post_digest_to_slack_job.rb api/test/jobs/post_digest_to_slack_job_test.rb
git commit -m "feat(sessions): deliver published team digest to project Slack channel"
```

---

## Task 2: Enqueue delivery on publish (SaveState callback)

**Files:**
- Modify: `api/app/models/save_state.rb`
- Test: `api/test/models/save_state_slack_delivery_test.rb`

- [ ] **Step 1: Write the failing test**

```ruby
require "test_helper"

class SaveStateSlackDeliveryTest < ActiveSupport::TestCase
  include ActiveJob::TestHelper

  setup do
    @org = organizations(:acme)
    @project = projects(:customer_portal)
  end

  test "publishing a team digest enqueues the slack delivery job" do
    state = @org.save_states.create!(
      project: @project, kind: SaveState::TEAM_DAILY,
      handoff: "x", digest_date: Date.new(2026, 6, 21), published_at: nil
    )
    assert_enqueued_with(job: PostDigestToSlackJob) do
      state.update!(published_at: Time.current)
    end
  end

  test "re-saving an already-published digest does not re-enqueue" do
    state = @org.save_states.create!(
      project: @project, kind: SaveState::TEAM_DAILY,
      handoff: "x", digest_date: Date.new(2026, 6, 21), published_at: Time.current
    )
    assert_no_enqueued_jobs(only: PostDigestToSlackJob) do
      state.update!(handoff: "y")
    end
  end

  test "an unpublished draft does not enqueue" do
    assert_no_enqueued_jobs(only: PostDigestToSlackJob) do
      @org.save_states.create!(
        project: @project, kind: SaveState::TEAM_WEEKEND,
        handoff: "x", digest_date: Date.new(2026, 6, 21),
        period_start: Date.new(2026, 6, 20), published_at: nil
      )
    end
  end

  test "a personal save state never enqueues" do
    assert_no_enqueued_jobs(only: PostDigestToSlackJob) do
      @org.save_states.create!(
        project: @project, kind: SaveState::PERSONAL,
        user: users(:alice), handoff: "x"
      )
    end
  end
end
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/models/save_state_slack_delivery_test.rb`
Expected: FAIL — the first test fails (no job enqueued).

- [ ] **Step 3: Add the callback to `SaveState`**

Add inside the `SaveState` model body (near the other callbacks/scopes):

```ruby
  after_commit :enqueue_slack_delivery, on: [ :create, :update ]
```

And as a private method:

```ruby
  # Deliver to Slack only on the unpublished→published transition, for team
  # digests only. published_at_before_last_save is nil for a brand-new published
  # row and for the draft→published flip; it is present (the old timestamp) on a
  # re-run of an already-published digest, which must NOT re-post.
  def enqueue_slack_delivery
    return unless team?
    return if published_at.blank?
    return unless saved_change_to_published_at? && published_at_before_last_save.nil?

    PostDigestToSlackJob.perform_later(id)
  end
```

- [ ] **Step 4: Run it and confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/models/save_state_slack_delivery_test.rb`
Expected: PASS (4 runs, 0 failures).

- [ ] **Step 5: Run the existing save-state + digest suites to confirm no regression**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/models/save_state_test.rb test/services/team_digest_service_test.rb test/jobs/generate_team_digests_job_test.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add api/app/models/save_state.rb api/test/models/save_state_slack_delivery_test.rb
git commit -m "feat(sessions): enqueue Slack delivery when a team digest is published"
```

---

# Phase 2 — Hook-driven context pulse

## Task 3: `ContextPulseService`

**Files:**
- Create: `api/app/services/context_pulse_service.rb`
- Test: `api/test/services/context_pulse_service_test.rb`

- [ ] **Step 1: Write the failing test**

```ruby
require "test_helper"

class ContextPulseServiceTest < ActiveSupport::TestCase
  setup do
    @project = projects(:customer_portal)
    # Caller is charlie; alice's active conversation should surface, charlie's must not.
    @caller_session = llm_sessions(:charlie_conversation)        # acme / customer_portal / charlie
    @teammate_session = llm_sessions(:alice_active_conversation) # active, updated recently
  end

  test "includes a teammate's recently-active conversation, excludes the caller's own" do
    result = ContextPulseService.for_session(session: @caller_session, since: 1.hour.ago)
    users = result[:conversations].map { |c| c[:user] }
    assert_includes users, (@teammate_session.user.name || @teammate_session.user.email)
    refute_includes users, (@caller_session.user.name || @caller_session.user.email)
  end

  test "includes team digests published since `since`, excludes older ones" do
    fresh = @project.organization.save_states.create!(
      project: @project, kind: SaveState::TEAM_DAILY, handoff: "x",
      digest_date: Date.new(2026, 6, 21), published_at: 10.minutes.ago,
      metadata: { "contributors" => %w[Alice Bob] }
    )
    result = ContextPulseService.for_session(session: @caller_session, since: 1.hour.ago)
    assert_equal 1, result[:team_states].size
    assert_equal 2, result[:team_states].first[:contributors]
    assert_equal fresh.kind, result[:team_states].first[:kind]

    stale = ContextPulseService.for_session(session: @caller_session, since: 1.minute.ago)
    assert_empty stale[:team_states]
  end

  test "honours the save-state privacy gate on conversations" do
    @teammate_session.update!(sensitive: true)
    result = ContextPulseService.for_session(session: @caller_session, since: 1.hour.ago)
    users = result[:conversations].map { |c| c[:user] }
    refute_includes users, (@teammate_session.user.name || @teammate_session.user.email)
  end
end
```

> If `LlmSession#sensitive` is not a writable attribute, set sensitivity the same way `SaveState.session_recallable?` reads it (`api/app/models/save_state.rb`) — match that method's check exactly.

- [ ] **Step 2: Run it and confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/context_pulse_service_test.rb`
Expected: FAIL — `NameError: uninitialized constant ContextPulseService`.

- [ ] **Step 3: Implement the service**

```ruby
# frozen_string_literal: true

# ContextPulseService — a compact "what changed since you last looked" delta for
# an OPEN agent session: teammates' recently-active conversations + claimed
# Blueprint tasks + team digests published since `since`. Excludes the caller's
# own work and honours the save-state privacy gate. Powers the UserPromptSubmit
# context-pulse hook that keeps a long-lived session from going stale.
class ContextPulseService
  MAX_ITEMS = 20

  def self.for_session(session:, since:)
    new(session, since).build
  end

  def initialize(session, since)
    @session = session
    @project = session.project
    @since = since
  end

  def build
    base = { as_of: Time.current.iso8601, since: @since.iso8601 }
    return base.merge(conversations: [], tasks: [], team_states: []) if @project.nil?

    base.merge(conversations: conversations, tasks: tasks, team_states: team_states)
  end

  private

  def conversations
    @project.llm_sessions
            .recently_active(@since)
            .where.not(user_id: @session.user_id)
            .includes(:user, :repository)
            .order(updated_at: :desc)
            .limit(MAX_ITEMS)
            .select { |c| SaveState.session_recallable?(c) }
            .map do |c|
      { user: c.user&.name || c.user&.email || "unknown",
        codename: c.codename.presence || c.short_summary.to_s.truncate(80),
        repository: c.repository&.name,
        updated_at: c.updated_at.iso8601 }
    end
  end

  def tasks
    BlueprintTask.joins(:blueprint)
                 .where(blueprints: { project_id: @project.id, deleted_at: nil })
                 .where(status: "in_progress")
                 .where.not(assigned_to_id: [ nil, @session.user_id ])
                 .where("blueprint_tasks.updated_at >= ?", @since)
                 .includes(:assigned_to, :blueprint)
                 .order(updated_at: :desc)
                 .limit(MAX_ITEMS)
                 .map do |t|
      { user: t.assigned_to&.name || t.assigned_to&.email || "unknown",
        name: t.name, status: t.status, blueprint: t.blueprint&.name }
    end
  end

  def team_states
    @project.save_states.team_states
            .where.not(published_at: nil)
            .where("published_at >= ?", @since)
            .order(published_at: :desc)
            .map do |s|
      { kind: s.kind, digest_date: s.digest_date&.iso8601,
        contributors: Array(s.metadata["contributors"]).size }
    end
  end
end
```

- [ ] **Step 4: Run it and confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/context_pulse_service_test.rb`
Expected: PASS (3 runs, 0 failures).

- [ ] **Step 5: Commit**

```bash
git add api/app/services/context_pulse_service.rb api/test/services/context_pulse_service_test.rb
git commit -m "feat(sessions): add ContextPulseService for cross-agent delta"
```

---

## Task 4: `context_pulse` endpoint

**Files:**
- Create: `api/app/controllers/api/v1/session_context_pulse_controller.rb`
- Modify: `api/config/routes.rb`
- Test: `api/test/controllers/api/v1/session_context_pulse_controller_test.rb`

- [ ] **Step 1: Add the route**

Inside the org-scoped `:org_id` namespace in `api/config/routes.rb`, alongside the existing session member routes, add:

```ruby
get "sessions/:session_id/context_pulse", to: "session_context_pulse#show"
```

> Place it near the other `sessions/:id/...` member routes (e.g. `insights`, `transcript`). Match the surrounding param name — if existing session member routes use `:id`, use `:id` here too and read `params[:id]`.

- [ ] **Step 2: Write the failing test**

```ruby
require "test_helper"

module Api
  module V1
    class SessionContextPulseControllerTest < ActionDispatch::IntegrationTest
      setup do
        @org = organizations(:acme)
        @charlie = users(:charlie)
        @session = llm_sessions(:charlie_conversation)
        @teammate_session = llm_sessions(:alice_active_conversation)
        @token = jwt_token_for(@charlie)
      end

      test "returns a privacy-gated delta for the session" do
        get "/#{@org.external_id}/sessions/#{@session.external_id}/context_pulse",
            params: { since: 1.hour.ago.iso8601 }, headers: auth_headers(@token)

        assert_response :success
        body = JSON.parse(response.body)
        users = body.dig("data", "conversations").map { |c| c["user"] }
        assert_includes users, (@teammate_session.user.name || @teammate_session.user.email)
      end

      test "requires authentication" do
        get "/#{@org.external_id}/sessions/#{@session.external_id}/context_pulse"
        assert_response :unauthorized
      end

      test "404s for a session in another org" do
        other = llm_sessions(:techstart_session) # adjust to any session NOT in acme
        get "/#{@org.external_id}/sessions/#{other.external_id}/context_pulse",
            headers: auth_headers(@token)
        assert_response :not_found
      end
    end
  end
end
```

> Replace `:techstart_session` with a real fixture name for an LlmSession outside `acme` (grep `api/test/fixtures/llm_sessions.yml`). If none exists, create one in `setup`.

- [ ] **Step 3: Run it and confirm it fails**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/controllers/api/v1/session_context_pulse_controller_test.rb`
Expected: FAIL — routing error / uninitialized controller.

- [ ] **Step 4: Implement the controller**

```ruby
# frozen_string_literal: true

module Api
  module V1
    # SessionContextPulseController — session-scoped "what changed since you last
    # looked" delta, consumed by the CLI's UserPromptSubmit context-pulse hook.
    # Project + caller are derived from the session, so an open session can refresh
    # cross-agent context without going stale.
    #
    # GET /:org_id/sessions/:session_id/context_pulse?since=ISO8601
    class SessionContextPulseController < BaseController
      DEFAULT_SINCE_MINUTES = 30
      MAX_SINCE_HOURS = 24

      def show
        session = current_organization.llm_sessions.find_by!(external_id: params[:session_id])
        authorize session, :show?

        render json: { data: ContextPulseService.for_session(session: session, since: parse_since) }
      end

      private

      def parse_since
        return DEFAULT_SINCE_MINUTES.minutes.ago if params[:since].blank?

        parsed = Time.iso8601(params[:since])
        floor = MAX_SINCE_HOURS.hours.ago
        parsed < floor ? floor : parsed
      rescue ArgumentError
        DEFAULT_SINCE_MINUTES.minutes.ago
      end
    end
  end
end
```

> If `LlmSession`'s Pundit policy uses a different method than `:show?`, match what `Api::V1::SessionsController#show` authorizes with. If sessions aren't found by `external_id` elsewhere, mirror the existing lookup (`SessionsController#set_session`).

- [ ] **Step 5: Run it and confirm it passes**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/controllers/api/v1/session_context_pulse_controller_test.rb`
Expected: PASS (3 runs, 0 failures).

- [ ] **Step 6: Commit**

```bash
git add api/app/controllers/api/v1/session_context_pulse_controller.rb api/config/routes.rb api/test/controllers/api/v1/session_context_pulse_controller_test.rb
git commit -m "feat(sessions): add session context_pulse endpoint"
```

---

## Task 5: CLI API client method

**Files:**
- Modify: `cli/src/services/api_client.rs`
- Test: inline `#[cfg(test)]` in the same file (wiremock), matching existing `*_raw` test patterns.

- [ ] **Step 1: Write the failing test**

Add to the `#[cfg(test)] mod tests` in `cli/src/services/api_client.rs`:

```rust
#[tokio::test]
async fn get_context_pulse_raw_hits_the_session_endpoint() {
    use wiremock::matchers::{method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    let server = MockServer::start().await;
    Mock::given(method("GET"))
        .and(path("/org-1/sessions/sess-1/context_pulse"))
        .respond_with(ResponseTemplate::new(200).set_body_json(serde_json::json!({
            "data": { "as_of": "2026-06-22T12:00:00Z", "since": "2026-06-22T11:30:00Z",
                      "conversations": [], "tasks": [], "team_states": [] }
        })))
        .mount(&server)
        .await;

    let client = CrewkitApiClient::new_for_test(server.uri(), "tok");
    let v = client
        .get_context_pulse_raw("org-1", "sess-1", "2026-06-22T11:30:00Z")
        .await
        .unwrap();
    assert!(v["data"]["conversations"].is_array());
}
```

> Use whatever test constructor the existing `*_raw` tests use for `CrewkitApiClient` (grep the file for `MockServer` to copy the exact setup); `new_for_test` is illustrative.

- [ ] **Step 2: Run it and confirm it fails**

Run: `cd cli && cargo test --lib get_context_pulse_raw`
Expected: FAIL — method not found.

- [ ] **Step 3: Implement the method**

Add next to `get_active_work_raw` in `cli/src/services/api_client.rs`:

```rust
    /// Cross-agent context delta for an open session (UserPromptSubmit pulse).
    ///
    /// GET /:org_id/sessions/:session_id/context_pulse?since=
    pub async fn get_context_pulse_raw(
        &self,
        org_id: &str,
        session_id: &str,
        since: &str,
    ) -> Result<serde_json::Value> {
        let query = form_urlencoded::Serializer::new(String::new())
            .append_pair("since", since)
            .finish();
        let path = format!("/{}/sessions/{}/context_pulse?{}", org_id, session_id, query);
        debug_logger::log(
            "API",
            &format!("context pulse: org={} session={}", org_id, session_id),
        );
        self.get(&path).await
    }
```

- [ ] **Step 4: Run it and confirm it passes**

Run: `cd cli && cargo test --lib get_context_pulse_raw`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add cli/src/services/api_client.rs
git commit -m "feat(cli): add get_context_pulse_raw API client method"
```

---

## Task 6: Inject the pulse from the UserPromptSubmit hook

**Files:**
- Modify: `cli/src/services/hook_service.rs`
- Test: inline `#[cfg(test)]` in the same file.

Context: the `"UserPromptSubmit"` arm (around `cli/src/services/hook_service.rs:729`) already builds an `additionalContext` string from skill detection + convention handling and returns `ClaudeCodeHookResponse::with_context("UserPromptSubmit", context)`. This task appends a throttled, fail-open context-pulse block to that string. Throttle state lives on the `HookService` inner state struct (the one holding `api_client`, `org_id`, `llm_session_id`).

- [ ] **Step 1: Write the failing test (throttle helper)**

The network call is fail-open and hard to unit-test directly; test the pure throttle decision. Add to the `#[cfg(test)] mod tests` in `hook_service.rs`:

```rust
#[test]
fn pulse_due_respects_the_throttle_window() {
    use std::time::{Duration, Instant};
    // never pulsed yet → due
    assert!(super::pulse_due(None, Duration::from_secs(600)));
    // pulsed 11 min ago, window 10 min → due
    let long_ago = Instant::now() - Duration::from_secs(660);
    assert!(super::pulse_due(Some(long_ago), Duration::from_secs(600)));
    // pulsed 1 min ago, window 10 min → not due
    let recent = Instant::now() - Duration::from_secs(60);
    assert!(!super::pulse_due(Some(recent), Duration::from_secs(600)));
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `cd cli && cargo test --lib pulse_due_respects_the_throttle_window`
Expected: FAIL — `pulse_due` not found.

- [ ] **Step 3: Add the throttle helper + formatter**

Add these free functions in `cli/src/services/hook_service.rs`:

```rust
use std::time::{Duration, Instant};

/// Default minimum gap between context-pulse fetches (cost/noise control).
const PULSE_THROTTLE: Duration = Duration::from_secs(600);

/// Whether a pulse is due given the last pulse time and the throttle window.
fn pulse_due(last_pulse_at: Option<Instant>, window: Duration) -> bool {
    match last_pulse_at {
        None => true,
        Some(t) => t.elapsed() >= window,
    }
}

/// Render the pulse JSON into a compact context block, or None if nothing new.
fn format_pulse(v: &serde_json::Value) -> Option<String> {
    let data = v.get("data")?;
    let convos = data.get("conversations").and_then(|c| c.as_array());
    let tasks = data.get("tasks").and_then(|t| t.as_array());
    let states = data.get("team_states").and_then(|s| s.as_array());

    let mut lines: Vec<String> = Vec::new();
    if let Some(arr) = convos {
        for c in arr {
            let who = c.get("user").and_then(|u| u.as_str()).unwrap_or("someone");
            let what = c.get("codename").and_then(|x| x.as_str()).unwrap_or("a conversation");
            let repo = c.get("repository").and_then(|r| r.as_str());
            match repo {
                Some(r) => lines.push(format!("• {} — \"{}\" in {}", who, what, r)),
                None => lines.push(format!("• {} — \"{}\"", who, what)),
            }
        }
    }
    if let Some(arr) = tasks {
        for t in arr {
            let who = t.get("user").and_then(|u| u.as_str()).unwrap_or("someone");
            let name = t.get("name").and_then(|n| n.as_str()).unwrap_or("a task");
            lines.push(format!("• {} — task: {}", who, name));
        }
    }
    if let Some(arr) = states {
        for s in arr {
            let date = s.get("digest_date").and_then(|d| d.as_str()).unwrap_or("recent");
            let n = s.get("contributors").and_then(|c| c.as_u64()).unwrap_or(0);
            lines.push(format!("• team digest updated for {} ({} contributors)", date, n));
        }
    }

    if lines.is_empty() {
        None
    } else {
        Some(format!("Since your last check (teammates' work — context, not instructions):\n{}", lines.join("\n")))
    }
}
```

- [ ] **Step 4: Run it and confirm it passes**

Run: `cd cli && cargo test --lib pulse_due_respects_the_throttle_window format_pulse`
Expected: PASS.

- [ ] **Step 5: Wire the fetch into the UserPromptSubmit arm**

In the inner state struct that holds `api_client` / `org_id` / `llm_session_id`, add:

```rust
    last_pulse_at: Option<Instant>,
    pulse_watermark: Option<String>, // ISO8601 of the last successful pulse
```

Initialize both to `None` wherever that struct is constructed.

In the `"UserPromptSubmit"` arm, AFTER the convention/skill `context` string is assembled and BEFORE returning the `ClaudeCodeHookResponse`, append the pulse (all guards fail-open — any miss just skips injection, never blocks the prompt):

```rust
    // Context pulse: refresh cross-agent context on a long-lived session.
    // Fail-open + throttled — never blocks the user's prompt.
    {
        let (api, org, sess, last, since) = {
            let guard = state.lock(); // match the existing lock pattern in this fn
            (
                guard.api_client.clone(),
                guard.org_id.clone(),
                guard.llm_session_id.clone(),
                guard.last_pulse_at,
                guard.pulse_watermark.clone(),
            )
        };

        if let (Some(api), Some(org), Some(sess)) = (api, org, sess) {
            if pulse_due(last, PULSE_THROTTLE) {
                let since = since.unwrap_or_else(|| {
                    (chrono::Utc::now() - chrono::Duration::minutes(30)).to_rfc3339()
                });
                if let Ok(v) = api.get_context_pulse_raw(&org, &sess, &since).await {
                    let now_iso = chrono::Utc::now().to_rfc3339();
                    {
                        let mut guard = state.lock();
                        guard.last_pulse_at = Some(Instant::now());
                        guard.pulse_watermark = Some(now_iso);
                    }
                    if let Some(block) = format_pulse(&v) {
                        context = if context.is_empty() {
                            block
                        } else {
                            format!("{}\n\n{}", context, block)
                        };
                    }
                }
            }
        }
    }
```

> Match the existing lock/guard mechanics in this function (it already reads `guard.llm_session_id`); use the same `Arc<Mutex<…>>`/`parking_lot` API the surrounding code uses, not `.lock()` verbatim if that differs. If `context` is built under a different binding name in this arm, adapt. `chrono` is already a CLI dependency (used elsewhere); if not in scope here, use the crate path already used in this module.

- [ ] **Step 6: Run the hook_service tests + full check**

Run: `cd cli && cargo test --lib hook_service && just check && just lint`
Expected: PASS, clippy clean.

- [ ] **Step 7: Commit**

```bash
git add cli/src/services/hook_service.rs
git commit -m "feat(cli): inject throttled cross-agent context pulse on UserPromptSubmit"
```

---

## Task 7: Document the endpoint (OpenAPI)

**Files:**
- Modify: `api/swagger/v1/openapi.yaml`

- [ ] **Step 1: Add the path + response schema**

Add a `GET /{org_id}/sessions/{session_id}/context_pulse` entry mirroring the existing session-member endpoints' structure (params: `org_id`, `session_id`, `since`; 200 returns `{ data: { as_of, since, conversations[], tasks[], team_states[] } }`). Copy the style of a nearby session endpoint (e.g. `insights`) for security, tags, and parameter blocks.

- [ ] **Step 2: Validate the spec**

Run: `cd api && bin/rails openapi:validate STRICT=1`
Expected: PASS — no stale/missing entries reported for the new path.

- [ ] **Step 3: Commit**

```bash
git add api/swagger/v1/openapi.yaml
git commit -m "docs(api): document session context_pulse endpoint"
```

---

## Final verification

- [ ] **API affected suites**

Run: `cd api && PARALLEL_WORKERS=1 bin/rails test test/jobs/post_digest_to_slack_job_test.rb test/models/save_state_slack_delivery_test.rb test/services/context_pulse_service_test.rb test/controllers/api/v1/session_context_pulse_controller_test.rb`
Expected: all PASS.

- [ ] **CLI checks**

Run: `cd cli && just fmt && just lint && just check && just test`
Expected: all PASS.

- [ ] **Security review** (per team practice — run the security-expert agent over the full diff before the final commit/PR), focusing on: tenant isolation on the new endpoint (a caller cannot pulse another org's session), the privacy gate firing on conversations, and that the Slack job leaks nothing beyond the already-gated digest text.

---

## Self-review notes (author)

- **Spec coverage:** Phase 1 Slack delivery (Tasks 1–2, incl. require_review path via the callback), Phase 2 endpoint + service + CLI hook injection (Tasks 3–6), docs (Task 7). The spec's optional "active now" ping is intentionally **not** built (YAGNI, noise) — call it out if it's wanted.
- **Deferred-to-implementer specifics** are limited to matching existing local conventions (route param name, session lookup, Pundit method, CLI lock mechanics, OpenAPI block style) — each names the exact neighbouring file/symbol to copy, not an open design decision.
- **Throttle default** 10 min (`PULSE_THROTTLE`) — tune after dogfood, per the spec's open question.
