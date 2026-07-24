# Save States — EOD Team Digest Job Implementation Plan (Plan 2 of 4)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.

**Goal:** A recurring job that, at each organization's configured end-of-day hour, summarizes that project's day of sessions into a rolling `team_daily` digest (weekdays) or a separate rolling `team_weekend` digest (weekends), built from existing session summaries.

**Architecture:** Add a per-org `daily_summary_settings` jsonb (end-of-day hour, timezone, weekend days) following the existing `telemetry_settings` pattern. A `GenerateTeamDigestsJob` runs **hourly**; for each org whose local time just crossed its EOD hour, it fans out per project. A `TeamDigestService` computes the digest window (with weekend accumulation), pulls each contributor's existing `deep_summary`/`summary`, asks Haiku for a brief per-person roll-up, sanitizes, and upserts the project's `team_daily`/`team_weekend` `save_states` row.

**Tech Stack:** Rails 8.1, solid_queue recurring jobs, `Time.zone`/`in_time_zone`, AnthropicService (Haiku), Minitest.

**Depends on:** Plan 1 (the `save_states` table + `SaveState` model already exist and are merged).

> Prefix all `bin/rails` with `RBENV_VERSION=3.4.9`; tests with `PARALLEL_WORKERS=1`. Never run `db:test:prepare`/`db:environment:set`/`db:reset` (wipes dev DB). Lowercase commit subjects. Work on `Arthur`.

---

## EOD semantics (settled)

- The digest is generated when an org's **local** hour equals `end_of_day_hour`. It covers the **window ending at that boundary**: `window_end = now_local.change(min:0, sec:0)`, `window_start = window_end - 1.day`. The **digest_day** is `window_start.to_date` (the day whose work is summarized). Default `end_of_day_hour = 0` (midnight ET) → the window is the full previous calendar day.
- **Weekday vs weekend** is classified by `digest_day.wday` against `weekend_days` (default `[0, 6]` = Sun/Sat, Ruby `Date#wday`).
- **Weekend accumulation:** for a weekend digest, `window_start` walks back over consecutive weekend days so Sunday's run re-summarizes the whole weekend (`period_start` = first weekend day). Friday's weekday digest is never overwritten by a weekend run (different `kind`).
- **Sessions** are selected by `started_at` within `[window_start, window_end)` (UTC), for the project, with completed analysis, excluding `sensitive?` / non-public-repo sessions.

## File structure

- Create: `api/db/migrate/20260616000002_add_daily_summary_settings_to_organizations.rb`
- Modify: `api/app/models/organization.rb` — settings accessors.
- Create: `api/app/services/team_digest_service.rb` — window math + content build + upsert.
- Create: `api/app/jobs/generate_team_digests_job.rb` — hourly fan-out.
- Modify: `api/config/recurring.yml` — schedule the job hourly.
- Test: `api/test/models/organization_test.rb` (add cases), `api/test/services/team_digest_service_test.rb`, `api/test/jobs/generate_team_digests_job_test.rb`.

---

### Task 1: Per-org daily-summary settings

**Files:**
- Create: `api/db/migrate/20260616000002_add_daily_summary_settings_to_organizations.rb`
- Modify: `api/app/models/organization.rb`
- Test: `api/test/models/organization_test.rb`

- [ ] **Step 1: Write the migration**

```ruby
# api/db/migrate/20260616000002_add_daily_summary_settings_to_organizations.rb
class AddDailySummarySettingsToOrganizations < ActiveRecord::Migration[8.1]
  def change
    add_column :organizations, :daily_summary_settings, :jsonb, null: false, default: {}
  end
end
```

- [ ] **Step 2: Run the migration**

Run: `RBENV_VERSION=3.4.9 bin/rails db:migrate`
Expected: `daily_summary_settings` appears on `organizations` in `db/schema.rb`.

- [ ] **Step 3: Write the failing model test**

Add to `api/test/models/organization_test.rb`:

```ruby
test "effective_daily_summary_settings merges defaults" do
  org = organizations(:acme)
  s = org.effective_daily_summary_settings
  assert_equal 0, s["end_of_day_hour"]
  assert_equal "America/New_York", s["timezone"]
  assert_equal [ 0, 6 ], s["weekend_days"]
  assert_equal true, s["enabled"]
  assert_equal false, s["require_review"]
end

test "daily_summary overrides win over defaults" do
  org = organizations(:acme)
  org.update!(daily_summary_settings: { "end_of_day_hour" => 18, "enabled" => false })
  s = org.effective_daily_summary_settings
  assert_equal 18, s["end_of_day_hour"]
  assert_equal false, s["enabled"]
  assert_equal "America/New_York", s["timezone"] # default preserved
end

test "weekend_day? respects the configured weekend days" do
  org = organizations(:acme)
  assert org.weekend_day?(Date.new(2026, 6, 14)) # Sunday
  assert org.weekend_day?(Date.new(2026, 6, 13)) # Saturday
  assert_not org.weekend_day?(Date.new(2026, 6, 12)) # Friday
end
```

- [ ] **Step 4: Run it to verify it fails**

Run: `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/models/organization_test.rb`
Expected: FAIL — `NoMethodError: effective_daily_summary_settings`.

- [ ] **Step 5: Add the accessors to Organization** (mirror the `telemetry_settings` pattern at `organization.rb:79-106`)

```ruby
  DEFAULT_DAILY_SUMMARY_SETTINGS = {
    "enabled" => true,
    "end_of_day_hour" => 0,            # local hour (0-23) the digest is cut
    "timezone" => "America/New_York",  # IANA tz the hour is evaluated in
    "weekend_days" => [ 0, 6 ],        # Ruby Date#wday: 0=Sun … 6=Sat
    "require_review" => false          # false = auto-publish; true = hold for manager review
  }.freeze

  def effective_daily_summary_settings
    DEFAULT_DAILY_SUMMARY_SETTINGS.merge(daily_summary_settings || {})
  end

  def daily_summary_timezone
    effective_daily_summary_settings["timezone"]
  end

  def weekend_day?(date)
    effective_daily_summary_settings["weekend_days"].include?(date.wday)
  end
```

- [ ] **Step 6: Run it to verify it passes**

Run: `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/models/organization_test.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add api/db/migrate/20260616000002_add_daily_summary_settings_to_organizations.rb \
        api/db/schema.rb api/app/models/organization.rb api/test/models/organization_test.rb
git commit -m "feat(sessions): per-org daily-summary settings (eod hour, tz, weekend days)"
```

---

### Task 1b: Publish gate on team digests (amends Plan 1 model + controller)

Adds a `published_at` column so `require_review` can hold a digest as an unpublished draft. **Recall must inject only published team digests.** Personal rows are unaffected (always available to their owner).

**Files:**
- Create: `api/db/migrate/20260616000003_add_published_at_to_save_states.rb`
- Modify: `api/app/models/save_state.rb` — `published?`/scope + filter in `recall_bundle`.
- Modify: `api/app/controllers/api/v1/save_states_controller.rb` — `update_team` publishes on review.
- Modify: `api/app/serializers/save_state_serializer.rb` — expose `published_at`.
- Test: `api/test/models/save_state_test.rb`, `api/test/controllers/api/v1/save_states_controller_test.rb`.

- [ ] **Step 1: Migration**

```ruby
# api/db/migrate/20260616000003_add_published_at_to_save_states.rb
class AddPublishedAtToSaveStates < ActiveRecord::Migration[8.1]
  def change
    add_column :save_states, :published_at, :datetime
  end
end
```

Run: `RBENV_VERSION=3.4.9 bin/rails db:migrate` (schema updated).

- [ ] **Step 2: Failing model tests** (add to `save_state_test.rb`)

```ruby
test "recall_bundle excludes an unpublished team_daily" do
  @org.save_states.find_by(kind: "team_daily")&.update!(published_at: nil)
  bundle = SaveState.recall_bundle(project: @project, user: @alice)
  assert_nil bundle[:team_daily], "draft (unpublished) digest must not be recalled"
end

test "recall_bundle includes a published team_daily" do
  @org.save_states.find_by(kind: "team_daily")&.update!(published_at: Time.current)
  bundle = SaveState.recall_bundle(project: @project, user: @alice)
  assert_not_nil bundle[:team_daily]
end

test "published? is true for personal regardless of published_at" do
  assert save_states(:alice_personal_portal).published?
end
```

> The `acme_team_daily` fixture (from Plan 1) has no `published_at`; update the fixture to set `published_at: <%= Time.current %>` so existing Plan-1 recall tests (which expect `team_daily` in the bundle) keep passing. Make that fixture edit in this step.

- [ ] **Step 3: Run → fail.** `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/models/save_state_test.rb`

- [ ] **Step 4: Implement** — in `app/models/save_state.rb` add:

```ruby
  def published?
    personal? || published_at.present?
  end
```

and filter team layers in `recall_bundle` (only the `daily`/`weekend` lookups change — personal stays as-is):

```ruby
    daily = find_by(project: project, kind: TEAM_DAILY).then { |s| s&.published? ? s : nil }
    weekend = find_by(project: project, kind: TEAM_WEEKEND).then { |s| s&.published? ? s : nil }
```

In `app/controllers/api/v1/save_states_controller.rb#update_team`, publish on review (set alongside the existing `reviewed_by`/`reviewed_at`):

```ruby
        state.reviewed_by = current_user
        state.reviewed_at = Time.current
        state.published_at ||= Time.current
```

In `app/serializers/save_state_serializer.rb`, add `:published_at` to the `attributes` list.

- [ ] **Step 5: Run → pass.** `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/models/save_state_test.rb test/controllers/api/v1/save_states_controller_test.rb`
Expected: PASS (including the Plan-1 recall tests, now that the fixture is published).

- [ ] **Step 6: Commit**

```bash
git add api/db/migrate/20260616000003_add_published_at_to_save_states.rb api/db/schema.rb \
        api/app/models/save_state.rb api/app/controllers/api/v1/save_states_controller.rb \
        api/app/serializers/save_state_serializer.rb api/test/fixtures/save_states.yml \
        api/test/models/save_state_test.rb api/test/controllers/api/v1/save_states_controller_test.rb
git commit -m "feat(sessions): publish gate so unpublished team digests are not recalled"
```

---

### Task 2: `TeamDigestService` — window math, content, upsert

**Files:**
- Create: `api/app/services/team_digest_service.rb`
- Test: `api/test/services/team_digest_service_test.rb`

**Interface:** `TeamDigestService.generate(project:, digest_day:)` → upserts (or skips) the project's `team_daily`/`team_weekend` SaveState for that `digest_day`; returns the SaveState or `nil` (no eligible sessions).

- [ ] **Step 1: Write the failing test** (stub AnthropicService; mirror `route_inbound_message_job_test.rb` stubbing + `stub_api_key`)

```ruby
# api/test/services/team_digest_service_test.rb
require "test_helper"

class TeamDigestServiceTest < ActiveSupport::TestCase
  setup do
    @org = organizations(:acme)
    @project = projects(:customer_portal)
    @alice = users(:alice)
    stub_api_key("test-key")
  end

  def make_session(day:, summary: "did some work", sensitive: false, status: "completed")
    LlmSession.create!(
      organization: @org, project: @project, user: @alice,
      external_id: SecureRandom.uuid, codename: "td-#{SecureRandom.hex(3)}",
      status: status, analysis_status: "completed",
      deep_summary: summary, sensitive: sensitive,
      started_at: day.in_time_zone("America/New_York").change(hour: 10)
    )
  end

  test "weekday digest_day upserts a team_daily row from session summaries" do
    make_session(day: Date.new(2026, 6, 12)) # Friday
    AnthropicService.stub(:complete, { "content" => [ { "type" => "text", "text" => "Friday roll-up." } ] }) do
      state = TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 12))
      assert_equal "team_daily", state.kind
      assert_equal Date.new(2026, 6, 12), state.digest_date
      assert_includes state.handoff, "Friday roll-up."
    end
  end

  test "weekend digest_day writes team_weekend and leaves team_daily untouched" do
    daily = @org.save_states.create!(project: @project, kind: "team_daily",
      handoff: "Friday digest", digest_date: Date.new(2026, 6, 12))
    make_session(day: Date.new(2026, 6, 14)) # Sunday
    AnthropicService.stub(:complete, { "content" => [ { "type" => "text", "text" => "Weekend roll-up." } ] }) do
      state = TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 14))
      assert_equal "team_weekend", state.kind
      assert_equal Date.new(2026, 6, 13), state.period_start # Saturday (weekend start)
    end
    assert_equal "Friday digest", daily.reload.handoff, "weekday digest must be untouched"
  end

  test "excludes sensitive sessions; no eligible sessions returns nil and writes nothing" do
    make_session(day: Date.new(2026, 6, 12), sensitive: true)
    assert_nil TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 12))
    assert_nil @org.save_states.find_by(project: @project, kind: "team_daily")
  end

  test "auto-publishes by default" do
    make_session(day: Date.new(2026, 6, 12))
    AnthropicService.stub(:complete, { "content" => [ { "type" => "text", "text" => "x" } ] }) do
      state = TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 12))
      assert_not_nil state.published_at
    end
  end

  test "holds as a draft when require_review is on" do
    @org.update!(daily_summary_settings: { "require_review" => true })
    make_session(day: Date.new(2026, 6, 12))
    AnthropicService.stub(:complete, { "content" => [ { "type" => "text", "text" => "x" } ] }) do
      state = TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 12))
      assert_nil state.published_at, "require_review should produce an unpublished draft"
    end
  end

  test "re-generating the same day overwrites the rolling row" do
    make_session(day: Date.new(2026, 6, 12))
    AnthropicService.stub(:complete, { "content" => [ { "type" => "text", "text" => "v2." } ] }) do
      TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 12))
      TeamDigestService.generate(project: @project, digest_day: Date.new(2026, 6, 12))
    end
    assert_equal 1, @org.save_states.where(project: @project, kind: "team_daily").count
  end
end
```

- [ ] **Step 2: Run it to verify it fails**

Run: `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/services/team_digest_service_test.rb`
Expected: FAIL — `uninitialized constant TeamDigestService`.

- [ ] **Step 3: Implement the service**

```ruby
# api/app/services/team_digest_service.rb
class TeamDigestService
  DIGEST_MODEL = "claude-haiku-4-5-20251001"
  DIGEST_MAX_TOKENS = 1024
  HANDOFF_BYTE_CAP = SaveState::MAX_HANDOFF_BYTES

  def self.generate(project:, digest_day:)
    new(project, digest_day).generate
  end

  def initialize(project, digest_day)
    @project = project
    @org = project.organization
    @digest_day = digest_day.to_date
  end

  def generate
    sessions = eligible_sessions
    return nil if sessions.empty?

    handoff = ContentSanitizer.clean(build_handoff(sessions))
    return nil if handoff.blank?

    state = @org.save_states.find_or_initialize_by(project: @project, kind: kind)
    state.assign_attributes(
      handoff: handoff.byteslice(0, HANDOFF_BYTE_CAP),
      digest_date: @digest_day,
      period_start: weekend? ? window_start_date : nil,
      published_at: auto_publish? ? Time.current : nil,
      metadata: { "session_ids" => sessions.map(&:external_id),
                  "contributors" => contributor_names(sessions) }
    )
    state.save!
    state
  end

  # Auto-publish unless the org opted into manager review. On regeneration of a
  # row that was already published, keep it published (don't silently unpublish).
  def auto_publish?
    return true unless @org.effective_daily_summary_settings["require_review"]

    SaveState.where(project: @project, kind: kind).where.not(published_at: nil).exists?
  end

  private

  def weekend?
    @org.weekend_day?(@digest_day)
  end

  def kind
    weekend? ? SaveState::TEAM_WEEKEND : SaveState::TEAM_DAILY
  end

  # For a weekend, walk back over consecutive weekend days so Sunday's run
  # covers the whole weekend; for a weekday it's just that day.
  def window_start_date
    d = @digest_day
    d -= 1 while weekend? && @org.weekend_day?(d - 1)
    d
  end

  def tz
    @org.daily_summary_timezone
  end

  def window
    start_at = window_start_date.in_time_zone(tz).beginning_of_day
    end_at = (@digest_day + 1).in_time_zone(tz).beginning_of_day
    [ start_at, end_at ]
  end

  def eligible_sessions
    start_at, end_at = window
    @project.llm_sessions
            .where(analysis_status: "completed")
            .where(started_at: start_at...end_at)
            .where(sensitive: [ false, nil ])
            .select { |s| s.repository.nil? || s.repository.public_visibility? }
  end

  def contributor_names(sessions)
    sessions.filter_map { |s| s.user&.name || s.user&.email }.uniq
  end

  def build_handoff(sessions)
    per_session = sessions.map do |s|
      who = s.user&.name || s.user&.email || "unknown"
      "- #{who}: #{s.deep_summary.presence || s.summary.presence || s.initial_prompt.to_s.truncate(200)}"
    end.join("\n")

    label = weekend? ? "weekend (#{window_start_date}–#{@digest_day})" : @digest_day.to_s
    system = "You write a brief team digest. Summarize, per person, what each " \
             "teammate did. <=200 words, markdown bullets, no preamble."
    prompt = "Project: #{@project.name}\nPeriod: #{label}\nRaw per-session notes:\n#{per_session}"

    resp = AnthropicService.complete(
      model: DIGEST_MODEL, max_tokens: DIGEST_MAX_TOKENS,
      system: system, messages: [ { role: "user", content: prompt } ]
    )
    text = resp&.dig("content", 0, "text")
    # Fall back to the raw per-person notes if the LLM is unavailable.
    "## Team digest — #{label}\n\n#{text.presence || per_session}"
  end
end
```

> Verify `LlmSession` has `repository` association and `public_visibility?` on the repo (the plan-1 `session_recallable?` used the same gate — reuse that knowledge). If `repository` is absent, drop that `select` filter and rely on `sensitive`.

- [ ] **Step 4: Run it to verify it passes**

Run: `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/services/team_digest_service_test.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add api/app/services/team_digest_service.rb api/test/services/team_digest_service_test.rb
git commit -m "feat(sessions): team digest service (window math, weekend accumulation, upsert)"
```

---

### Task 3: `GenerateTeamDigestsJob` — hourly fan-out

**Files:**
- Create: `api/app/jobs/generate_team_digests_job.rb`
- Modify: `api/config/recurring.yml`
- Test: `api/test/jobs/generate_team_digests_job_test.rb`

- [ ] **Step 1: Write the failing job test**

```ruby
# api/test/jobs/generate_team_digests_job_test.rb
require "test_helper"

class GenerateTeamDigestsJobTest < ActiveJob::TestCase
  setup do
    @org = organizations(:acme)
    @project = projects(:customer_portal)
    @org.update!(daily_summary_settings: { "end_of_day_hour" => 0, "timezone" => "America/New_York" })
  end

  test "generates digests only for orgs at their eod hour" do
    # 00:30 ET on 2026-06-16 → eod hour 0 matches; digest_day = 2026-06-15
    travel_to Time.find_zone("America/New_York").local(2026, 6, 16, 0, 30) do
      called = []
      TeamDigestService.stub(:generate, ->(project:, digest_day:) { called << [ project.id, digest_day ]; nil }) do
        GenerateTeamDigestsJob.perform_now
      end
      assert_includes called, [ @project.id, Date.new(2026, 6, 15) ]
    end
  end

  test "does nothing for orgs whose local hour is not the eod hour" do
    travel_to Time.find_zone("America/New_York").local(2026, 6, 16, 14, 0) do
      called = false
      TeamDigestService.stub(:generate, ->(**) { called = true; nil }) do
        GenerateTeamDigestsJob.perform_now
      end
      assert_not called
    end
  end

  test "skips orgs with disabled daily summaries" do
    @org.update!(daily_summary_settings: { "enabled" => false, "end_of_day_hour" => 0 })
    travel_to Time.find_zone("America/New_York").local(2026, 6, 16, 0, 30) do
      called = false
      TeamDigestService.stub(:generate, ->(**) { called = true; nil }) do
        GenerateTeamDigestsJob.perform_now
      end
      assert_not called
    end
  end
end
```

- [ ] **Step 2: Run it to verify it fails**

Run: `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/jobs/generate_team_digests_job_test.rb`
Expected: FAIL — `uninitialized constant GenerateTeamDigestsJob`.

- [ ] **Step 3: Implement the job** (mirror the `Organization.find_each` fan-out at `aggregate_resource_stats_from_sessions_job.rb`)

```ruby
# api/app/jobs/generate_team_digests_job.rb
class GenerateTeamDigestsJob < ApplicationJob
  queue_as :default

  def perform
    Organization.find_each do |org|
      settings = org.effective_daily_summary_settings
      next unless settings["enabled"]

      now_local = Time.current.in_time_zone(settings["timezone"])
      next unless now_local.hour == settings["end_of_day_hour"].to_i

      digest_day = (now_local.to_date - 1)
      org.projects.find_each do |project|
        TeamDigestService.generate(project: project, digest_day: digest_day)
      rescue StandardError => e
        Rails.logger.error("[GenerateTeamDigestsJob] project #{project.external_id}: #{e.message}")
        Sentry.capture_exception(e, extra: { project_id: project.id }) if defined?(Sentry)
      end
    end
  end
end
```

> `digest_day = now_local.to_date - 1` is correct for the midnight default (window = previous calendar day). For a non-midnight `end_of_day_hour`, `TeamDigestService`'s window math already keys off `digest_day`; if you later support midday EOD, revisit whether `digest_day` should be `now_local.to_date` — out of scope here (default is midnight).

- [ ] **Step 4: Run it to verify it passes**

Run: `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/jobs/generate_team_digests_job_test.rb`
Expected: PASS.

- [ ] **Step 5: Schedule it hourly** in `api/config/recurring.yml` (add to the `development`, `staging`, and `production` blocks, mirroring existing entries):

```yaml
  generate_team_digests:
    class: GenerateTeamDigestsJob
    schedule: "0 * * * *"  # top of every hour; the job self-filters by each org's EOD hour
    queue: default
```

- [ ] **Step 6: Commit**

```bash
git add api/app/jobs/generate_team_digests_job.rb api/config/recurring.yml \
        api/test/jobs/generate_team_digests_job_test.rb
git commit -m "feat(sessions): hourly job that generates per-project team digests at eod"
```

---

### Task 4: Gate

- [ ] **Step 1:** `RBENV_VERSION=3.4.9 PARALLEL_WORKERS=1 bin/rails test test/services/team_digest_service_test.rb test/jobs/generate_team_digests_job_test.rb test/models/organization_test.rb test/smoke/` → all green.
- [ ] **Step 2:** `RBENV_VERSION=3.4.9 bin/rubocop` on the new files → clean.
- [ ] **Step 3:** Run the `security-expert` agent over the diff — focus: the digest must never include sensitive/private-repo sessions, and `metadata.session_ids` must not leak across orgs.

## Self-review

- **Spec coverage:** EOD hour + tz + weekend-days + `require_review` setting (Task 1), publish gate so drafts aren't recalled + publish-on-review (Task 1b), summarize-from-existing-summaries (Task 2 `build_handoff`), weekday→`team_daily` / weekend→`team_weekend` with Friday preserved + Sunday accumulation (Task 2 + tests), auto-publish vs hold-for-review (Task 2 `auto_publish?` + tests), privacy exclusion (Task 2 `eligible_sessions`), no-op empty day (Task 2 returns nil), rolling overwrite (Task 2 `find_or_initialize_by`), recurring schedule (Task 3 + recurring.yml), per-tenant EOD via hourly self-filtering (Task 3). The recall **consumption** of these rows is Plan 3 (CLI); the manager publish/review UI is Plan 4 (dashboard).
- **Naming consistency:** `effective_daily_summary_settings`, `weekend_day?`, `TeamDigestService.generate(project:, digest_day:)`, `window_start_date`, `kind`/`digest_date`/`period_start` match the SaveState model from Plan 1.
- **Assumptions to verify in execution:** `LlmSession.repository#public_visibility?` exists (Plan 1 used it); `AnthropicService.complete` response shape is `{"content" => [{"text" => ...}]}` (confirmed in extraction); `travel_to` is available (ActiveSupport::Testing::TimeHelpers — include if not already).
