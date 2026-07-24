# Team-Activity Current-Work Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: use subagent-driven-development (recommended) or executing-plans to implement task-by-task. Steps use `- [ ]` checkboxes. **Every task ends with the gate-3 trio** (implementation-validator → adversarial-code-reviewer → security-expert) per `.claude/rules/planning-and-review.md`, plus the affected component gates.

**Goal:** Show only sessions active in the last 24h, each with a brief "current work" summary sourced from its latest agent-posted `work_update`, rendered dense for agents (MCP, fenced) and readable for people (dashboard, plain text).

**Architecture:** Query-time only. `ActiveWorkQuery` (a) switches the conversation window from `recently_active` (which keeps idle open sessions) to a strict `updated_at >= since`, and (b) after visibility filtering, fetches the newest `work_update.body` per **surviving** `llm_session_id` in one `DISTINCT ON` query and threads it through the serializers as `current_work`/`current_work_at`. CLI + dashboard render the new field. No schema change.

**Tech Stack:** Rails 8.1 (Minitest, Pundit, Postgres `DISTINCT ON`), Rust CLI (`format_active_work` + `wrap_untrusted`), Next.js dashboard (types generated from OpenAPI).

**Spec:** `docs/designs/team-activity-current-work.md` — the 9 security constraints there are binding.

---

## File Structure

**API (`api/`)**
- `app/models/llm_session.rb` — add `active_work_window(since)` scope (strict `updated_at`, excludes terminal statuses).
- `app/services/active_work_query.rb` — use the new scope; add `latest_updates_by_session(session_ids)`; thread the map into grouping. **The join chokepoint** — doc-comment it.
- `app/serializers/active_work_conversation_serializer.rb` — add `current_work` (read-side `ContentSanitizer.clean` + char-trim) + `current_work_at`.
- `app/serializers/active_work_user_serializer.rb` — thread `updates_by_session` context into the conversation serializer.
- `swagger/v1/openapi.yaml` — add `current_work` (nullable string) + `current_work_at` (nullable date-time) to `ActiveWorkConversation`.
- `test/sentinels/active_work_visibility_sentinel_test.rb` — extend with work_update leak cases.
- `test/services/active_work_query_test.rb`, `test/controllers/api/v1/project_active_work_controller_test.rb` — window-change + current_work assertions.

**CLI (`cli/`)**
- `src/commands/mcp.rs` — `format_active_work` row shape + `current_work` inside the fence.

**Dashboard (`dashboard/`)**
- `src/types/api-generated.d.ts` — regenerated (not hand-edited).
- `src/components/features/projects/active-work-strip.tsx` + `src/components/features/dashboard/org-active-work.tsx` — `ConversationRow` renders `current_work` as prominent plain text.

---

## Phase A — API

### Task A1: Strict 24h active window (drop the `status='active' OR` branch)

**Files:**
- Modify: `api/app/models/llm_session.rb` (near the `recently_active` scope, ~line 82)
- Modify: `api/app/services/active_work_query.rb:89`
- Test: `api/test/services/active_work_query_test.rb`, and fix `api/test/controllers/api/v1/project_active_work_controller_test.rb:125` + `api/test/sentinels/active_work_visibility_sentinel_test.rb:173` which rely on the old OR-branch.

- [ ] **Step 1 — Confirm the terminal status values.** Run:
  `cd api && grep -nE "status.*(active|completed|ended)|enum :status|STATUSES" app/models/llm_session.rb`
  Note the exact set (expected: `active`, `completed`, possibly `ended`). Use the real terminal values in Step 3 (do not guess).

- [ ] **Step 2 — Failing test** in `active_work_query_test.rb`:
```ruby
test "idle open session (status active, stale updated_at) is excluded by the 24h window" do
  stale = llm_sessions(:alice_active) # status active
  stale.update_column(:updated_at, 3.days.ago)
  result = ActiveWorkQuery.new(
    sessions_scope: LlmSession.all, tasks_scope: BlueprintTask.all,
    current_user: users(:bob), permission_level: :manager, project_id: nil,
    since: nil, include_self: true
  ).result
  ids = result[:data][:users].flat_map { |u| u[:active_conversations].map { |c| c[:data][:external_id] } }
  assert_not_includes ids, stale.external_id
end
```

- [ ] **Step 3 — Run it; expect FAIL** (currently included via the OR-branch):
  `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/active_work_query_test.rb -n /idle_open/`

- [ ] **Step 4 — Add the scope** in `llm_session.rb` (use the confirmed terminal statuses from Step 1; example assumes `completed`):
```ruby
# Active-work window: real recent activity only. Unlike `recently_active`, this
# does NOT keep open-but-idle sessions (no `status='active' OR`) — an abandoned
# open session must age out. Excludes terminal (ended/completed) sessions.
scope :active_work_window, ->(since) {
  where("llm_sessions.updated_at >= ?", since).where.not(status: %w[completed])
}
```

- [ ] **Step 5 — Use it** in `active_work_query.rb` `conversations_relation` (line 89): replace `.recently_active(@since)` with `.active_work_window(@since)`.

- [ ] **Step 6 — Update the two tests that asserted the OR-branch.** In `project_active_work_controller_test.rb:125` and `active_work_visibility_sentinel_test.rb:173`, the sessions relying on "status=active makes it always recent" must be given a fresh `updated_at` (within window) to keep showing, or moved outside to assert exclusion. Read each and adjust the fixture/`update_column(:updated_at, …)` so intent is preserved.

- [ ] **Step 7 — Run the affected suites; expect PASS:**
  `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/active_work_query_test.rb test/controllers/api/v1/project_active_work_controller_test.rb test/sentinels/active_work_visibility_sentinel_test.rb`

- [ ] **Step 8 — Commit:** `git add -A && git commit -m "feat(sessions): active-work window uses strict 24h updated_at, drops idle-open sessions"`

- [ ] **Step 9 — Gate-3 trio + `test-runner`.**

### Task A2: Fetch latest work_update per surviving session (the join chokepoint)

**Files:** Modify `api/app/services/active_work_query.rb`; Test `api/test/services/active_work_query_test.rb`.

- [ ] **Step 1 — Failing test:**
```ruby
test "latest_updates_by_session returns newest body per session, scoped to given ids" do
  s = llm_sessions(:alice_active); s.update_column(:updated_at, 1.minute.ago)
  WorkUpdate.create!(organization: s.organization, project: s.project, user: s.user,
                     llm_session_id: s.id, body: "older", created_at: 2.hours.ago)
  WorkUpdate.create!(organization: s.organization, project: s.project, user: s.user,
                     llm_session_id: s.id, body: "newest", created_at: 1.minute.ago)
  q = ActiveWorkQuery.new(sessions_scope: LlmSession.all, tasks_scope: BlueprintTask.all,
        current_user: s.user, permission_level: :manager, include_self: true)
  map = q.send(:latest_updates_by_session, [s.id])
  assert_equal "newest", map[s.id].body
  assert_empty q.send(:latest_updates_by_session, []) # empty ids → no query, empty map
end
```

- [ ] **Step 2 — Run; expect FAIL** (`NoMethodError`): `cd api && PARALLEL_WORKERS=1 bin/rails test test/services/active_work_query_test.rb -n /latest_updates_by_session/`

- [ ] **Step 3 — Implement** in `active_work_query.rb` (private). The surviving `session_ids` already came from the org-scoped, visibility-filtered relation, so keying on them is the guarantee (constraint 1); no extra org filter is needed in the core impl:
```ruby
# The ONLY source of current-work bodies. Keyed STRICTLY on the surviving
# session-id set (already visibility+sensitivity filtered upstream in
# `apply_visibility`). NEVER use policy_scope(WorkUpdate) / @project.work_updates
# (their scope is org-only → leaks sensitive/private bodies) or key on
# user_id/project_id/blueprint_task_id (bypasses row-visibility). One DISTINCT ON
# query; a NULL llm_session_id (deleted session, on_delete: :nullify) is excluded
# by the IN. This is the join chokepoint the security sentinel guards.
def latest_updates_by_session(session_ids)
  return {} if session_ids.blank?

  WorkUpdate
    .where(llm_session_id: session_ids)
    .select("DISTINCT ON (llm_session_id) work_updates.*")
    .order("llm_session_id, created_at DESC")
    .index_by(&:llm_session_id)
end
```

- [ ] **Step 4 — (Optional hardening) belt-and-suspenders org filter.** Constraint 5 allows an extra `organization_id` filter as defense-in-depth. Only add it if it's clean — e.g. thread the caller's `organization_id` into the initializer and add `.where(organization_id: @organization_id)`. If threading it complicates the call sites, SKIP it: the id-set join (Step 3) is the real guarantee, and adding a `@sessions_scope.pluck` would fire an extra query. Document the decision in a one-line comment.

- [ ] **Step 5 — Run; expect PASS.**

- [ ] **Step 6 — Commit:** `git commit -am "feat(sessions): latest work_update per visible session (DISTINCT ON, id-set-scoped)"`

- [ ] **Step 7 — Gate-3 trio** (security-expert MUST confirm the chokepoint comment + that no other fetch path is used).

### Task A3: Serialize `current_work` + `current_work_at` (sanitized, trimmed, conversation rows only)

**Files:** Modify `active_work_conversation_serializer.rb`, `active_work_user_serializer.rb`, and `active_work_query.rb` `group_by_user`; Test `api/test/serializers/active_work_conversation_serializer_test.rb` (create).

- [ ] **Step 1 — Failing serializer test** (create the file):
```ruby
require "test_helper"
class ActiveWorkConversationSerializerTest < ActiveSupport::TestCase
  test "current_work is sanitized + trimmed; placeholder when no update" do
    s = llm_sessions(:alice_active)
    upd = WorkUpdate.new(body: "reworking JWT\u{200B} races", created_at: Time.current)
    out = ActiveWorkConversationSerializer.new(s, current_work: upd).serialize[:data]
    assert_equal "reworking JWT races", out[:current_work]         # invisible stripped
    assert_equal upd.created_at.iso8601, out[:current_work_at]
    none = ActiveWorkConversationSerializer.new(s, current_work: nil).serialize[:data]
    assert_nil none[:current_work]                                 # renderers show placeholder
    assert_nil none[:current_work_at]
  end
end
```

- [ ] **Step 2 — Run; expect FAIL:** `cd api && PARALLEL_WORKERS=1 bin/rails test test/serializers/active_work_conversation_serializer_test.rb`

- [ ] **Step 3 — Implement** in `active_work_conversation_serializer.rb`: add `:current_work, :current_work_at` to `attributes`, and:
```ruby
CURRENT_WORK_MAX_CHARS = 280 # payload trim, independent of WorkUpdate write cap

def current_work
  upd = context[:current_work]
  return nil unless upd&.body.present?
  ContentSanitizer.clean(upd.body).to_s.strip.truncate(CURRENT_WORK_MAX_CHARS)
end

def current_work_at
  context[:current_work]&.created_at&.iso8601
end
```

- [ ] **Step 4 — Thread the context** in `active_work_user_serializer.rb`:
```ruby
def active_conversations
  updates = context[:updates_by_session] || {}
  conversations.map { |c| ActiveWorkConversationSerializer.new(c, current_work: updates[c.id]).serialize }
end
```
and in `active_work_query.rb` `group_by_user`, fetch once and pass it:
```ruby
def group_by_user(conversations, tasks, users)
  updates_by_session = latest_updates_by_session(conversations.map(&:id))
  by_conv = conversations.group_by(&:user_id)
  by_task = tasks.group_by(&:assigned_to_id)
  (by_conv.keys + by_task.keys).uniq.map do |uid|
    user = users[uid]; next nil unless user
    ActiveWorkUserSerializer.new(user,
      conversations: (by_conv[uid] || []).first(@limit),
      tasks: (by_task[uid] || []).first(@limit),
      updates_by_session: updates_by_session
    ).serialize[:data]
  end.compact.sort_by { |u| u[:last_activity_at] || Time.at(0) }.reverse
end
```
(`current_work` attaches to conversation rows only — tasks never receive it, satisfying constraint 2.)

- [ ] **Step 5 — Run serializer + query tests; expect PASS.**

- [ ] **Step 6 — Commit:** `git commit -am "feat(sessions): active-work conversations carry current_work summary"`

- [ ] **Step 7 — Gate-3 trio.**

### Task A4: Window default → 24h

- [ ] **Step 1 — Test:** assert a session updated 45 min ago (previously excluded by the 30-min default) now appears with no `since` param.
- [ ] **Step 2 — Run; FAIL.**
- [ ] **Step 3 — Change** `DEFAULT_SINCE_MINUTES = 30` → `1440` in `active_work_query.rb`.
- [ ] **Step 4 — Run; PASS.** Fix any fixture-dependent tests keyed on the 30-min default.
- [ ] **Step 5 — Commit:** `git commit -am "feat(sessions): active-work default window 24h"`
- [ ] **Step 6 — Gate-3 trio.**

### Task A5: OpenAPI contract

**Files:** Modify `api/swagger/v1/openapi.yaml` (`ActiveWorkConversation` schema).

- [ ] **Step 1 — Locate the schema:** `cd api && grep -n "ActiveWorkConversation:" swagger/v1/openapi.yaml`
- [ ] **Step 2 — Add** under its `properties`:
```yaml
current_work:
  type: string
  nullable: true
  description: Latest agent-posted work-update body (sanitized, trimmed); null when none posted.
current_work_at:
  type: string
  format: date-time
  nullable: true
```
- [ ] **Step 3 — Validate:** `cd api && bin/rails openapi:validate STRICT=1` → expect PASS.
- [ ] **Step 4 — Commit:** `git commit -am "docs(api): active-work current_work fields in OpenAPI spec"`

### Task A6: Security sentinel — no body leak (blocking gate item)

**Files:** Extend `api/test/sentinels/active_work_visibility_sentinel_test.rb`.

- [ ] **Step 1 — Add cases**, each: create a session + a `work_update` on it, then assert the body is absent for a non-owner and present for the owner, in the serialized `result` (agent + dashboard share this payload):
```ruby
test "work_update body from a sensitive session never leaks to a non-owner" do
  s = llm_sessions(:alice_active); s.update_column(:updated_at, 1.minute.ago)
  WorkUpdate.create!(organization: s.organization, project: s.project, user: s.user,
                     llm_session_id: s.id, body: "SECRET popped prod", created_at: 1.minute.ago)
  s.update!(sensitive: true) # flagged AFTER the update

  bob = users(:bob) # non-owner, manager
  data = ActiveWorkQuery.new(sessions_scope: LlmSession.all, tasks_scope: BlueprintTask.all,
           current_user: bob, permission_level: :manager, include_self: true).result
  bodies = data[:data][:users].flat_map { |u| u[:active_conversations].map { |c| c[:data][:current_work] } }
  assert_not_includes bodies, "SECRET popped prod"
  refute data.to_json.include?("SECRET popped prod"), "body must not appear anywhere in payload"
end
```
  Repeat for: (b) private-repo session, (c) restricted-repo session viewed by a `developer`, (d) cross-poster (update.user_id ≠ session.user_id) — assert it DOES show when the session is visible (visibility follows the session). Add an owner-sees-it positive case.

- [ ] **Step 2 — Run; expect PASS:** `cd api && PARALLEL_WORKERS=1 bin/rails test test/sentinels/active_work_visibility_sentinel_test.rb`
- [ ] **Step 3 — Commit:** `git commit -am "test(sessions): sentinel — active-work current_work respects session visibility"`
- [ ] **Step 4 — Gate-3 trio (security-expert is the owner of this task's sign-off).**

---

## Phase B — CLI

### Task B1: `format_active_work` new row shape, current_work inside the fence

**Files:** Modify `cli/src/commands/mcp.rs` `format_active_work` (~line 916) + its tests.

- [ ] **Step 1 — Failing test** in `mcp.rs` tests:
```rust
#[test]
fn format_active_work_shows_current_work_inside_fence() {
    let resp = serde_json::json!({
        "data": { "users": [{
            "name": "testbot",
            "active_conversations": [{ "data": {
                "codename": "portal-auth", "external_id": "sess-1",
                "current_work": "reworking JWT refresh races",
                "current_work_at": "2026-07-24T10:00:00Z",
                "repository": { "name": "customer-portal" }
            }}]
        }]},
        "meta": {}
    });
    let out = format_active_work(&resp, 30);
    assert!(out.contains("<crewkit_untrusted:"), "current_work must be fenced");
    let close = out.rfind("</crewkit_untrusted:").unwrap();
    assert!(!out[close..].contains("reworking JWT"), "body must be inside the fence");
    assert!(out.contains("testbot"));
    assert!(out.contains("reworking JWT refresh races"));
    assert!(out.contains("(no update posted)")==false); // has an update
}
```

- [ ] **Step 2 — Run; expect FAIL:** `cd cli && cargo test --bin crewkit format_active_work_shows_current_work`

- [ ] **Step 3 — Rework `format_active_work`** to build per-conversation body lines of the shape
  `{name} · {session} · "{current_work|(no update posted)}" · {last update rel-time}`
  where `session = codename ?? external_id` + `(repo)`, reading `current_work`/`current_work_at` from each conversation's `data`. Keep the trusted count header OUTSIDE and pass the assembled lines through `wrap_untrusted` (as today). The `(no update posted)` literal is trusted text but lives in the fenced body for layout — that is fine (it is crewkit-authored, not attacker input). Relative time from `current_work_at` (fallback: last activity) via the existing `relative_age` helper.

- [ ] **Step 4 — Run; expect PASS.** Update the existing `format_active_work_empty` + any fixture tests to the new shape.

- [ ] **Step 5 — CLI gates:** `cd cli && just fmt && just lint && just check && just test` and `cargo clippy -- -D warnings`.

- [ ] **Step 6 — Commit:** `git commit -m "feat(cli): active-work shows current-work summary per session (fenced)"`

- [ ] **Step 7 — Gate-3 trio.**

---

## Phase C — Dashboard

### Task C1: Render current_work as prominent plain text (both components)

**Files:** Regenerate `src/types/api-generated.d.ts`; modify `active-work-strip.tsx` + `org-active-work.tsx` `ConversationRow`; Test the two `__tests__`.

- [ ] **Step 1 — Regenerate types** (after Task A5 lands): `cd dashboard && npm run generate:api-types && npm run validate:api-contract` → expect PASS; confirm `ActiveWorkConversation` now has `current_work`/`current_work_at`.

- [ ] **Step 2 — Failing component test** in `active-work-strip.test.tsx`: mock a conversation with `current_work: "reworking JWT races"` and assert it renders as text; mock a `current_work` containing `<img src=x onerror=alert(1)>` and assert it appears **escaped** (no real element / `dangerouslySetInnerHTML`).

- [ ] **Step 3 — Update `ConversationRow`** in BOTH components to make `current_work` the primary line, with the session id + relative `current_work_at` secondary, falling back to `(no update posted)` when `current_work` is null:
```tsx
function ConversationRow({ conv }: { conv: ActiveWorkConversation }) {
  const d = conv.data;
  const work = d?.current_work ?? "(no update posted)";
  const label = d?.codename ?? d?.external_id ?? "Conversation";
  const href = conv._links?.dashboard;
  const line = (
    <span className="block truncate text-xs">
      <span className="text-foreground">{work}</span>       {/* PLAIN TEXT — React escapes; never dangerouslySetInnerHTML / markdown */}
      <span className="ml-1 text-muted-foreground">· {label}</span>
    </span>
  );
  return href ? <Link href={href} className="block hover:underline">{line}</Link> : line;
}
```
  (No markdown renderer on `current_work` — constraint 7.)

- [ ] **Step 4 — Run:** `cd dashboard && npm test -- active-work` → expect PASS.

- [ ] **Step 5 — Dashboard gates:** `cd dashboard && npm test && npm run lint && npm run typecheck && npm run validate:api-contract`.

- [ ] **Step 6 — Commit** (`--no-verify` per the broken husky hook, after manual gates): `git commit -m "feat(dashboard): active-work shows current-work summary (plain text)"`

- [ ] **Step 7 — Gate-3 trio (+ design-review-agent on the live widget if desired).**

---

## Cross-cutting QA (after all phases)

- [ ] Re-run each component's affected gates fresh.
- [ ] Confirm the three gate-3 verification points the security-expert named: chokepoint comment present in `active_work_query.rb`; sentinel exercises the payload both renderers consume; no markdown renderer on `current_work`.
- [ ] `cd integration-tests && npm test` (spans components).
- [ ] Update `docs/designs/team-activity-current-work.md` status → shipped; log commits in `reviewable_merge_commits.md`.

## Notes / risks

- **Window change blast radius:** `active_work_window` replaces `recently_active` only at the ActiveWorkQuery call site; `recently_active` stays for any other consumer (none today). Existing tests asserting the OR-branch are updated in A1/A6 — do not skip Step A1.6.
- **`updated_at` must track activity.** A1 assumes `LlmSession#updated_at` advances on events/turns. If Step A1.1 shows it does not, use the latest event/turn timestamp column instead and adjust the scope + tests.
- **No new egress / no LLM** — current_work is purely the stored work_update body. Loi 25-clean.
