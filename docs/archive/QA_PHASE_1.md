# Phase 1 QA Review — Session-Resource Bridge

> Reviewed: 2026-02-02 | Commits: `93f5f92`, `732505c` | Scope: Phase 1A, 1B, 1C
> Agents: Rails Expert, API Designer, Security Expert, Schema Architect, CLI Expert, Data Analyst

---

## Summary

| Severity | Count | Status |
|----------|-------|--------|
| HIGH     | 10    | Must fix before deployment |
| MEDIUM   | 19    | Fix before GA |
| LOW      | 12    | Cleanup / improvement |

---

## HIGH Priority

### H1. Cross-tenant resource resolution (SECURITY)

**Files:** `api/app/controllers/api/v1/observability_controller.rb:230-251`, `api/app/controllers/api/v1/sessions_controller.rb:343,382`

`Resource.find_by(external_id:)` is unscoped to the current organization. A crafted `resource_external_id` from the CLI could link a SubagentSession or SkillInvocation to a Resource owned by a different organization. The snapshot-based fallback has the same issue.

**Fix:** Scope all `Resource.find_by(external_id:)` calls to `current_organization.resources` or validate the resolved resource's org matches the session's org.

---

### H2. Non-idempotent aggregation job

**Files:** `api/app/jobs/aggregate_resource_stats_from_sessions_job.rb:22-28,222-228`

High-water mark stored in `Rails.cache` (Solid Cache). If cache is flushed (deployment, crash, TTL expiry at 1 week), `last_aggregated_at` returns nil, job falls back to `INITIAL_LOOKBACK = 2.hours`, and all sessions completed in the last 2 hours get re-processed. Since `ResourceStatsService.record_session` increments counters, this double-counts everything.

**Fix:** Store the high-water mark in the database (e.g., `system_settings` table or dedicated metadata row).

---

### H3. `to_unsafe_h` on untrusted CLI input

**Files:** `api/app/controllers/api/v1/observability_controller.rb:42`, `api/app/controllers/api/v1/sessions_controller.rb:411`

Both bypass Strong Parameters entirely. The full hash is stored into JSONB columns. Arbitrary keys and deeply nested structures are accepted with no schema validation or size limiting.

**Fix:** Define allowed event schemas. For `model_metrics`, validate that values conform to the expected `{input_tokens:, output_tokens:, ...}` shape and cap the number of model entries. For events, add a max payload size check.

---

### H4. No batch size limit on event ingestion

**Files:** `api/app/controllers/api/v1/observability_controller.rb:33,40-166`

Accepts an unbounded array of events. Entire batch processed inside one `ActiveRecord::Base.transaction`, holding row locks for the full duration. A misbehaving CLI could send thousands of events, causing long-running transactions.

**Fix:** Enforce a max batch size (e.g., 500). Return 422 if exceeded. Consider per-event savepoints.

---

### H5. Fat ObservabilityController (~380 lines)

**Files:** `api/app/controllers/api/v1/observability_controller.rb` (entire file)

Contains subagent creation, skill invocation tracking, session lifecycle management (clear_new, resume), transcript building orchestration, and resource resolution. Violates the project convention that controllers stay thin.

**Fix:** Extract to `EventIngestionService` that receives parsed params and returns a result object. Controller should parse params, call the service, and render.

---

### H6. `resource_snapshot` accepted without schema validation

**Files:** `api/app/controllers/api/v1/sessions_controller.rb:42`

Stored as-is. Later queried by `resolve_resource_id` which expects specific keys (`name`, `resource_type`, `chain` array). Malformed snapshot silently breaks resource resolution — returns nil with no error.

**Fix:** Add a validation method or form object that checks the snapshot structure on ingest. At minimum validate top-level array structure and required keys.

---

### H7. Aggregation job queries globally without org scoping

**Files:** `api/app/jobs/aggregate_resource_stats_from_sessions_job.rb:48-203`

All four aggregation methods query tables without organization filter. Single job run processes all tenants in one pass. Combined with H1, stats could be attributed to wrong org. Creates a scaling bottleneck.

**Fix:** Either scope queries per-org (enqueue one job per org) or add org-aware grouping that validates resource ownership.

---

### H8. SubagentTracker dispatch is still a stub

**Files:** `cli/src/services/hook_service.rs:212-216`

PLAN.md Phase 1A.4 says this should be wired. Events just log and return OK. Local JSONL stat extraction, summary generation, and API calls for subagent sessions never happen from the CLI side. Events do flow to API via EventBuffer, but local processing is missing.

**Fix:** Wire SubagentTracker dispatch. At minimum, route SubagentStart/Stop events to the tracker service for local stat extraction and API reporting.

---

### H9. Invalid statistical test in version comparison

**Files:** `api/app/jobs/evaluate_resource_comparisons_job.rb:245-273`

Missing validation of sample sizes, normality assumptions for the t-test. Small samples or skewed distributions will produce unreliable p-values and recommendations.

**Fix:** Add minimum sample size guard (e.g., n >= 10 per version). Consider Mann-Whitney U for non-normal distributions. Add confidence intervals.

---

### H10. Session start failure permanently loses resource snapshot

**Files:** `cli/src/commands/code.rs:696-710`

Snapshot moved into `session_params`. On `start_session` failure (transient network error), it's gone — no local cache, no retry. The core Phase 1 deliverable is permanently lost for that session.

**Fix:** Cache snapshot locally (e.g., `~/.cache/crewkit/last-snapshot.json`) so it can be retried or attached to the next session.

---

## MEDIUM Priority

### M1. N+1 query for active llm_session inside event loop

**Files:** `api/app/controllers/api/v1/observability_controller.rb:61,78`

`crewkit_session.llm_sessions.where(status: "active").order(:started_at).last` fires for every event in the batch. A batch of 50 events = 50+ identical SELECT queries.

**Fix:** Resolve active LlmSession once before the loop, refresh only when clear_new/resume events mutate it.

---

### M2. N+1 on turn_number calculation

**Files:** `api/app/services/session_transcript_builder_service.rb:57`

`llm_session.session_turns.maximum(:turn_number)` fires a SELECT MAX per turn created. 20 transcript-eligible events = 20 queries.

**Fix:** Maintain counter in memory or query once at loop start.

---

### M3. Per-event duplicate detection instead of batch-level

**Files:** `api/app/controllers/api/v1/observability_controller.rb:51-57`

`SessionEvent.find_by(batch_id:, sequence_number:)` fires individually per event. On retry of 50-event batch, 50 individual SELECTs.

**Fix:** Query `SessionEvent.where(batch_id:).pluck(:sequence_number).to_set` once at top of method.

---

### M4. `session_created: false` hardcoded in ingestion response

**Files:** `api/app/controllers/api/v1/observability_controller.rb:173`

Always returns `false`, but `handle_clear_new` and `handle_resume` both create new LlmSessions. CLI has no way to know a new conversation was created.

**Fix:** Track new session creation in the loop and include IDs in response.

---

### M5. Inconsistent response envelope for sessions#create

**Files:** `api/app/controllers/api/v1/sessions_controller.rb:67-72`

Uses flat `{session:, crewkit_session_id:, telemetry_settings:}` instead of standard `{data:, meta:, _links:}` envelope.

**Fix:** Normalize to `{data: LlmSessionSerializer, meta: {crewkit_session_id:, telemetry_settings:}}`.

---

### M6. TelemetrySettings show requires update permission

**Files:** `api/app/controllers/api/v1/telemetry_settings_controller.rb:17`

GET endpoint uses `authorize current_organization, :update?`. Read-only users can't see settings.

**Fix:** Use `:show?` or create a `TelemetrySettingsPolicy` with separate read permissions.

---

### M7. No validation on telemetry setting values

**Files:** `api/app/controllers/api/v1/telemetry_settings_controller.rb:29-31`, `api/app/models/organization.rb`

`capture_mode` could be any string. `retention_days` could be negative. Booleans not type-checked.

**Fix:** Add model-level validation for the JSONB structure with allowed values.

---

### M8. MODEL_PRICING hardcoded in controller

**Files:** `api/app/controllers/api/v1/sessions_controller.rb:393-408`

Three model families hardcoded with string matching (`key.include?("opus")`). Unknown models silently get Sonnet pricing.

**Fix:** Extract to configuration file or pricing service.

---

### M9. SkillInvocation lacks organization_id denormalization

**Files:** `api/app/models/skill_invocation.rb`, `api/app/jobs/aggregate_resource_stats_from_sessions_job.rb:83-86`

Unlike LlmSession which has org_id for direct queries, SkillInvocation requires join through parent chain. Aggregation job queries globally.

**Fix:** Add `organization_id` column (denormalized) for direct tenant-scoped queries.

---

### M10. Missing uniqueness constraint on [llm_session_id, turn_number]

**Files:** `api/db/migrate/20260202200001_create_session_turns.rb:21`

Non-unique index. Auto-increment logic not protected against concurrent requests. Two concurrent ingestion requests could create duplicate turn numbers.

**Fix:** Add unique index on `[llm_session_id, turn_number]`.

---

### M11. N+1 on Resource.find_by inside snapshot aggregation loop

**Files:** `api/app/jobs/aggregate_resource_stats_from_sessions_job.rb:183`

`Resource.find_by(external_id:)` per snapshot entry per session. 100 sessions × 10 resources = 1000 queries.

**Fix:** Batch-resolve all external_ids upfront with a single WHERE IN query.

---

### M12. SkillInvocationsController index has no pagination

**Files:** `api/app/controllers/api/v1/skill_invocations_controller.rb:20-29`

Returns all invocations for a session with no limit.

**Fix:** Add pagination with `.page(params[:page]).per(per_page)`.

---

### M13. thread_data.length materializes full query for count

**Files:** `api/app/controllers/api/v1/sessions_controller.rb:227`

`.length` loads all result rows to count them, then re-executes with offset/limit.

**Fix:** Use `SELECT COUNT(DISTINCT conversation_thread_id)` subquery.

---

### M14. Missing GIN index on resource_snapshot JSONB

**Files:** schema (crewkit_sessions table)

Aggregation job queries resource_snapshot for resource lookups. No GIN index means sequential scans.

**Fix:** Add `add_index :crewkit_sessions, :resource_snapshot, using: :gin`.

---

### M15. Non-standard route param for skill_invocations

**Files:** `api/config/routes.rb:374`, `api/app/controllers/api/v1/skill_invocations_controller.rb:48`

Uses `param: :external_id` in routes. Every other resource uses default `param: :id` and maps internally.

**Fix:** Use default `param: :id` pattern for consistency.

---

### M16. Client-side-only sanitization — server trusts CLI

**Files:** `cli/src/services/event_sanitizer.rs`, `api/app/controllers/api/v1/observability_controller.rb:42`

Entire sanitization runs in CLI. Server stores raw payload via `to_unsafe_h`. Modified CLI or direct API call bypasses all sanitization. Raw SessionEvent.data has unsanitized content even when SessionTranscriptBuilderService respects capture_mode for turns.

**Fix:** Add server-side sanitization layer before storing SessionEvent.data, or document that raw events are intentionally unfiltered and restrict access.

---

### M17. ResourceType stored as String in ResourceSnapshotEntry

**Files:** `cli/src/types/resources.rs:207`, `cli/src/services/resource_sync.rs:173`

`ResourceSnapshotEntry.resource_type` is `String` instead of the `ResourceType` enum. Loses compile-time type safety. `Display` impl changes break API contract silently.

**Fix:** Change to `resource_type: ResourceType`. The enum already has correct serde serialization.

---

### M18. Cross-source double-counting in aggregation

**Files:** `api/app/jobs/aggregate_resource_stats_from_sessions_job.rb:25-39`

Same sessions may be counted by multiple aggregation methods (subagent, skill, primary resource, snapshot). No deduplication across methods.

**Fix:** Add deduplication logic or clearly separate what each method counts, ensuring no overlap.

---

### M19. No developer notification when full capture mode enabled

**Files:** `cli/src/services/event_sanitizer.rs` (privacy concern)

Org admin can silently switch to `capture_mode: "full"`, enabling capture of all developer code, commands, and outputs. No consent mechanism for individual developers.

**Fix:** CLI should display a notice when full capture mode is active. Consider requiring developer acknowledgment.

---

## LOW Priority

### L1. CrewkitSessionSerializer self link will 404

**Files:** `api/app/serializers/crewkit_session_serializer.rb:19-21`

Link points to `/sessions/:crewkit_session_external_id` but controller queries LlmSession.

### L2. Skill invocation completion matches by name only

**Files:** `api/app/controllers/api/v1/observability_controller.rb:337-354`

Fragile for concurrent same-skill invocations. Should match by correlation ID.

### L3. Telemetry defaults duplicated between migration and model

**Files:** `api/db/migrate/20260202200000_add_telemetry_settings_to_organizations.rb:5-12`, `api/app/models/organization.rb:41-48`

Migration column default and Ruby `DEFAULT_TELEMETRY_SETTINGS` will drift independently.

### L4. Recurring jobs only defined for production

**Files:** `api/config/recurring.yml:12`

Dev/staging environments can't test the pipeline without manual `perform_now`.

### L5. Missing HATEOAS links on SkillInvocation/SubagentSession serializers

**Files:** `api/app/serializers/skill_invocation_serializer.rb`, `api/app/serializers/subagent_session_serializer.rb`

No `self` or parent session links.

### L6. No OpenAPI spec updates for new Phase 1 endpoints

**Files:** `api/swagger/v1/openapi.yaml` (missing updates)

New endpoints need OpenAPI documentation per project conventions.

### L7. conversation_id attribute in LlmSessionSerializer may be undefined

**Files:** `api/app/serializers/llm_session_serializer.rb:11`

No custom method and possibly no column. May raise NoMethodError at runtime.

### L8. PATCH permits summary/short_summary fields

**Files:** `api/app/controllers/api/v1/sessions_controller.rb:461-468`

These are end-of-session fields being accepted on mid-session PATCH.

### L9. Redundant clones and format! in CLI

**Files:** `cli/src/services/resource_sync.rs:154,173`, `cli/src/commands/code.rs:352`

`format!("{}", x)` instead of `.to_string()`, unnecessary `.clone()` calls.

### L10. Minimal test coverage for HookService dispatch paths

**Files:** `cli/src/services/hook_service.rs:299-309`

Single test (port binding). Missing tests for capture mode, skill detection, session lifecycle.

### L11. primary_agent_name detection only works with exactly 1 agent

**Files:** `cli/src/commands/code.rs:626-636`

Multi-agent orgs always send `None`. Future improvement needed.

### L12. Missing composite index for by_agent analytics queries

**Files:** schema

`SessionAnalyticsService` likely queries by agent across sessions. No composite index supports this.

---

## Recommended Fix Order

### Phase A — Security & Data Integrity (do first)
1. **H1** — Scope resource resolution to org
2. **H2** — Move aggregation high-water mark to database
3. **H3 + H4 + H6** — Input validation, batch limits, schema validation
4. **H7** — Org-scoped aggregation queries
5. **M10** — Unique constraint on turn_number

### Phase B — Architecture & Performance
6. **H5** — Extract EventIngestionService
7. **M1 + M2 + M3 + M11** — Fix N+1 queries
8. **M5** — Normalize response envelope
9. **M14** — Add GIN index on resource_snapshot

### Phase C — Functional Gaps
10. **H8** — Wire SubagentTracker dispatch
11. **H10** — Cache resource snapshot on session failure
12. **M4** — Return session_created status correctly
13. **M9** — Denormalize org_id on SkillInvocation

### Phase D — Validation & Polish
14. **M6 + M7** — Telemetry settings permissions and validation
15. **M8** — Extract pricing to config
16. **M15 + M17** — Convention consistency
17. **L1-L12** — Low priority cleanup
