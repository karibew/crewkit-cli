# Design: three tenant-visibility fixes (active-work leak + threads subqueries + inherits_from IDOR)

**Status:** design (gate-1 cleared; awaiting gate-2)
**Component:** api
**Scope:** three read-path security fixes found in the 2026-07-15 health scan. No product/schema change.

## Fix 1 (HIGH) — active-work session-visibility leak

`ProjectActiveWorkController#active_conversations` (`project_active_work_controller.rb:82-89`)
queries `policy_scope(LlmSession)` (org-only) with no visibility scopes, and
`ActiveWorkConversationSerializer` exposes `short_summary` + repo → a project viewer
sees every member's active-session summary/codename/component/repo, including
`sensitive` and restricted/private-repo sessions. The recurring leak class
(#267/#269/#275).

**Fix:**
- `include SessionVisibilityGuard` (for `permission_level_symbol` only — do NOT wire
  `authorize_session_visibility!` as a before_action; there is no `@session` here).
- Chain on the `LlmSession` query:
  `.visible_to(current_user, permission_level_symbol).exclude_sensitive_unless_owner(current_user)`.
  `.visible_to` (role-aware) is correct — the endpoint is cross-user by design.
- `claimed_tasks` (BlueprintTask) is left as-is: org-visible by design, and
  `ActiveWorkTaskSerializer` exposes no session content (gate-1 confirmed).
- No second unscoped session path (serializer reads the already-filtered array).

## Fix 2 (MEDIUM) — threads subquery leak

`sessions_controller#threads`: the grouped main query is scoped, but `latest_sessions`
(`:293`) and `user_ids_by_thread` (`:302`) re-query `LlmSession` UNSCOPED → a hidden
latest session's `latest_summary` + owner leak.

**Fix:** build one visible base and reuse it for both subqueries:
```ruby
visible = policy_scope(LlmSession)
  .where(organization: current_organization)
  .visible_to(current_user, permission_level_symbol)
  .exclude_sensitive_unless_owner(current_user)
  .nontrivial   # match the main query so "latest" can't be a trivial/excluded row
```
- `latest_sessions = visible.where(conversation_thread_id: thread_ids).includes(:user)
  .order(:conversation_thread_id, started_at: :desc)
  .select("DISTINCT ON (conversation_thread_id) llm_sessions.*")`
  — **DISTINCT ON requires `conversation_thread_id` as the leftmost ORDER BY**; set it
  explicitly (don't rely on the prior `started_at`-only order).
- `user_ids_by_thread = visible.where(conversation_thread_id: thread_ids).group(...).pluck(...)`.
- Semantics (fail-closed, correct): when the true latest is hidden, show the latest
  *visible* session's summary; `users[]` contains only owners of visible sessions —
  now consistent with `conversation_count`/`has_active` (which already aggregate visible rows).

## Fix 3 (MEDIUM) — cross-tenant `inherits_from_id` IDOR

`resources_controller#create_params:526` resolves the parent via un-org-scoped
`Resource.find_by(external_id:)`; `Resource#valid_inheritance_mode` only checks presence;
`ResourceConfigurationService#build_chain` walks `inherits_from` with no org check → an
`inherit` resource in org A can point at org B's resource and merge its private `content`
via `GET /resources/:id/effective|tiered`.

**Fix (write + read, both required):**
1. **Model invariant** (`valid_inheritance_mode`, runs on every save — covers create and any
   future update path): when `inherits_from` is present, it MUST be
   `platform?` OR `organization_id == self.organization_id`, AND `inherits_from_id != id`
   (self-reference reject). Else add an error. Legit chains (platform→org→project) all use
   platform-or-same-org parents, so nothing valid breaks. `organization_id` is populated
   before validation (build_resource / install path). Do NOT add `inherits_from_id` to
   `update_params` (keep it non-repointable; the validate is belt-and-suspenders).
2. **Controller lookup scoping** (`create_params`): resolve the parent only among
   platform-or-current-org resources, so a foreign `external_id` → nil (no existence
   oracle; with `ownership_mode: inherit` it then hits the existing "must be present" 422).
3. **Read-path fail-closed guard** (`build_chain`) — REQUIRED to neutralize pre-existing /
   laundered cross-org rows the model validation can't retroactively catch: capture the
   starting resource's `organization_id`; when the next `inherits_from` is neither
   `platform?` nor same-org as the start, STOP the walk (don't append the parent). The
   resource resolves to its own content only — fail-closed, no cross-org content merged.
   No data-repair migration in this pass (the guard neutralizes at read; cleanup is a
   separate ticket → flag to Félix/Rémi).

## Gate-3 note (folded in)

- The threads subqueries now inherit `.nontrivial` from the shared `visible` base
  (previously unscoped). This is a small, intentional behavior change beyond the
  security scope: `latest_summary`/`users[]` are drawn only from *nontrivial*
  visible sessions — consistent with the grouped `conversation_count` (which was
  already `.nontrivial`). A thread whose newest visible session is trivial now
  shows the newest *nontrivial* visible one. More correct, not a regression.

## Gate-2 notes (folded in)

- `ResourceEditorService.assign_to_project` is a 4th write path (`create!` with
  `inherits_from_id`); it's protected by `find_source_resource` scoping + `authorize`,
  and now additionally by the model invariant (3.1) — no independent hole.
- **Accepted side-effect:** with the model invariant, saving a resource that ALREADY
  holds a laundered cross-org `inherits_from` (even for an unrelated content edit) now
  422s. Fail-closed and rare; the read guard already neutralizes such rows. A one-off
  data-cleanup migration to repair pre-existing rows is a **separate ticket → Félix/Rémi**
  (not in this pass).
- build_chain guard predicate is same-org-**as-the-start** (not immediate-child): legit
  3-tier chains never contain two distinct non-platform orgs, so this never wrongly
  truncates; a laundered hop stops before the foreign parent is appended.

## Bonus fix surfaced while building (pre-existing 500)

Writing the threads sentinel exposed that `sessions#threads` **500'd on every
multi-session thread** — its SELECT aliased `SUM(...) as total_tokens`, but (a)
the raw SQL referenced a non-existent `total_tokens` COLUMN, and (b) the alias
`total_tokens` collides with `LlmSession#total_tokens` (a method that reads
`input_tokens`, absent on a grouped record). The endpoint had no test, so it went
unnoticed. Fixed in the same method: `SUM(COALESCE(input_tokens,0) +
COALESCE(output_tokens,0)) as total_tokens_sum`, read as `t.total_tokens_sum`.
Now covered by the threads sentinel (which asserts a 200 + correct counts).

## Fail-safe review (gate-1)

- Fix 1/2: `visible_to(:owner/:admin)` → all; `exclude_sensitive_unless_owner` keeps the
  owner's own sensitive rows; `IS DISTINCT FROM` handles null-owner rows. No over-restriction.
- Fix 3: parent-not-loadable → `inherits_from` nil → chain stops → own content only. A
  supplied-but-foreign `inherits_from_id` with `inherit` mode → "must be present" 422.

## Sentinels (`test/sentinels/`)

- **active_work**: two same-org repos (one private/restricted), B's sessions. Owner+admin see
  B's session in the payload; viewer A does NOT (and `short_summary` absent); B's `sensitive`
  session excluded for A.
- **threads**: thread of 2 sessions where the latest is hidden/sensitive. Owner sees latest
  real summary + both users; non-owner sees the older *visible* summary, hidden owner absent
  from `users[]`, `conversation_count` == visible count.
- **inherits_from**: (write) cross-org parent create → 422, foreign external_id → nil/no oracle;
  (read, key) seed a cross-org `inherits_from_id` directly (bypass validation) and assert
  `GET /resources/:id/effective` does NOT include the other org's content; self-reference reject.

## Gates

`PARALLEL_WORKERS=1 bin/rails test` on the affected controller/model/service tests + the new
sentinels + `test/sentinels/`; `rubocop`; `openapi:validate STRICT=1` (no response-shape change
expected). Isolated `crewkit_test_wt1` DB.
