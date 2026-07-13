# Design: project-tier playbook enforcement (for_stacks project scoping)

> Status: **gate-2 revision (round 2)** · Author: planning session 2026-07-07 ·
> SDLC gate 1 ✅ · gate 2 BLOCK (round 1) — 4 findings folded in below, awaiting re-review ·
> build not started.
> Feature chosen scope: **full enforcement fix** (not the cosmetic for_sync-only change).

## Problem

`crewkit code` fetches playbooks at session start via `GET /:org_id/playbooks/for_stacks`
(`api/app/controllers/api/v1/playbooks_controller.rb:72`), which returns
`current_organization.applicable_playbooks` filtered by framework stack — **org-level, no
project scoping**. Those conventions populate the CLI `PlaybookCache`, which drives convention
**enforcement + injection** during the session, and are also written to `.claude/playbooks.md`
(`sync_playbooks_from_conventions`). So `ProjectPlaybookAssignment` — the whole project-tier
governance surface — is ignored by the path that actually enforces.

`for_sync` (`playbooks_controller.rb:144`) already has an (unused-in-practice) `project_id`
branch. `for_sync` is consumed **only** by the MCP `get_conventions` tool
(`cli/src/commands/mcp.rs:524`); the `.claude/playbooks.md` file sync goes through `for_stacks`,
not `for_sync`.

**Goal:** when a project is resolvable, scope the enforcement/sync/display paths to that
project's assigned playbooks. Consistency invariant (corrected per gate-2 finding #4):
**enforced == synced-to-file ⊆ MCP-shown** — `for_stacks` (enforce + file sync) excludes
`legacy_mode` assignments; `for_sync` (MCP display) includes them as advisory-only extras. You
never *enforce* a convention the agent wasn't *shown*; the asymmetry only adds shown-but-advisory
entries. Do **not** use the false equality "enforced == shown" as a test oracle.

## Decisions (locked with owner this session)

- **A — zero assignments → fall back to org-applicable.** A project with **no**
  `ProjectPlaybookAssignment` rows keeps today's org-applicable behavior (backward-compatible;
  no silent enforcement loss for the many projects that never configured assignments).
  Assignments are an **opt-in narrowing**, not a kill-switch. Fall-back trigger is
  `assignments.none?` (no rows at all) — **not** "no rows after filtering."
- **B — exclude `legacy_mode` from hard enforcement.** `for_stacks` uses `.active` (=
  `legacy_mode: false`) assignments. `for_sync`/MCP display keep showing legacy assignments
  (no `.active`), honoring the model's "challenge but don't block" semantics.
- **C — per-project convention customizations are DEFERRED.** Verified: neither `for_sync` nor
  `for_stacks` applies `PlaybookConventionCustomization` today (`generate_agent_content` uses
  `merge_playbook_conventions`, which ignores it; `effective_conventions` is called only by the
  dashboard read endpoint `project_playbook_assignments_controller.rb:196`). Both paths stay on
  **raw base conventions** — same base values on both paths, customization layer off (so no new
  divergence introduced). Activating the customization layer
  (override/disable/augment + `currently_effective` windows) is a separate feature with its own
  gate 1/2.

## Gate-1 security constraints (all folded into the design below)

1. Resolve `project_id` via `current_organization.projects.find_by!(external_id:)` — cross-org
   and nonexistent IDs both 404 (no existence oracle). No separate 403 branch.
2. `find_by!` runs **before** `Rails.cache.fetch`; a bad/cross-org `project_id` 404s and is
   never cached.
3. Cache key gains a non-colliding project dimension; the "no project" sentinel is a literal
   that cannot equal a UUID `external_id`.
4. Because `for_stacks` is an enforcement path, assignment changes must invalidate its cache —
   achieved by folding an assignment-set token into the key (self-invalidating; no mutation-site
   hooks needed).
5. CLI `None`-project path omits the param entirely (not `""`) → server takes the org-wide
   branch (`params[:project_id].present?` treats blank as absent).
6. Session start never dead-ends: a `project_id` 404 / network / auth failure on the fetch stays
   non-fatal (existing `playbook_failed` path), degrading to "no playbooks this session."
7. Enforcement (`for_stacks`), file-sync (`for_stacks`), and MCP display (`for_sync`) scope
   identically. MCP guards the param behind `if let Some(pid) = &scope.project_id`.
8. No unbounded cache-key axis: project dimension is bounded by `find_by!` (invalid IDs never
   reach the cache). `MAX_STACKS` unchanged.

---

## API contract (frozen before implementation)

`GET /api/v1/:org_id/playbooks/for_stacks`
- **New optional query param** `project_id` (project `external_id`, UUID).
- Absent / blank → **unchanged** behavior (org-applicable, existing cache key).
- Present + resolves to an org-owned project:
  - If the project has **≥1** assignment → return only playbooks whose `Resource#id` ∈
    `project.project_playbook_assignments.active.by_priority.pluck(:playbook_resource_id)`,
    intersected with org-`applicable_playbooks`, then stack-filtered. (Assignment priority order
    preserved for the pluck; final stack filter unchanged.)
  - If the project has **0** assignments → org-applicable (decision A).
- Present + unknown/cross-org → **404** (`RecordNotFound` → existing rescue), uncached.
- Response body shape **unchanged** (`PlaybookSerializer.serialize_collection`). Only the set of
  playbooks changes.

`GET /api/v1/:org_id/playbooks/for_sync` (already accepts `project_id`)
- Change its zero-assignment behavior to **match decision A** (fall back to org-applicable
  instead of returning empty), and route through the same shared helper as `for_stacks` but with
  `active_only: false` (decision B). Safe: no current consumer passes `project_id` yet.

OpenAPI: add the `project_id` query param to `for_stacks` in `swagger/v1/openapi.yaml`; run
`bin/rails openapi:validate STRICT=1`.

---

## Implementation stages (test-first; each ends with the gate-3 trio)

### Stage 1 — API: shared scoping helper + `for_stacks` project param + cache key

Files: `api/app/controllers/api/v1/playbooks_controller.rb`, `swagger/v1/openapi.yaml`.
Tests: `api/test/controllers/api/v1/playbooks_controller_test.rb` (+ a sentinel assertion for
cross-org 404, see Stage 3).

1. **Failing tests first** in `playbooks_controller_test.rb` for `for_stacks`:
   - project with assignments → returns only assigned∩applicable, stack-filtered.
   - project with only `legacy_mode` assignments → returns **empty** (active filter; not fallback).
   - project with **zero** assignments → returns org-applicable (decision A).
   - unknown `project_id` → 404; cross-org `project_id` (project owned by another org) → **404,
     identical body** (no oracle).
   - no `project_id` → unchanged org-applicable result.
   - two projects with different assignments → different results (cache-key isolation).
   - assignment **added** → response changes within the same test (cache-token invalidation).
   - assignment **edited** without row-count change (e.g. `legacy_mode` toggle, priority change)
     → response changes (gate-2 finding #1b: token must track `max(updated_at)`, not just count).
2. Add a **private helper** shared by both actions:
   ```ruby
   # Returns the applicable playbooks scoped to a project's assignments, or the
   # full org-applicable set when the project has no assignments at all (decision A).
   # active_only: exclude legacy_mode assignments (true for enforcement/for_stacks).
   def project_scoped_playbooks(all_playbooks, project, active_only:)
     return all_playbooks if project.nil?
     assignments = project.project_playbook_assignments
     return all_playbooks if assignments.none?            # decision A: opt-in narrowing
     scope = active_only ? assignments.active : assignments
     assigned_ids = scope.by_priority.pluck(:playbook_resource_id)
     by_id = all_playbooks.index_by(&:id)
     assigned_ids.filter_map { |id| by_id[id] }
   end
   ```
3. In `for_stacks`: resolve project **before** the cache block:
   ```ruby
   project = params[:project_id].present? ? current_organization.projects.find_by!(external_id: params[:project_id]) : nil
   ```
   (bad/cross-org → `RecordNotFound` → 404, before any cache write.)
4. Cache key gains project + assignment-set token. **(Corrected per gate-2 finding #1 — the
   earlier `pick(Arel.sql("count(*), coalesce(max(updated_at),'0')"))` was broken twice: the
   `'0'`→timestamp coercion raises `PG::DatatypeMismatch` on every call, and a single comma-
   containing `Arel.sql` arg silently drops `max(updated_at)`. Use two plain AR aggregates — no
   `Arel.sql`, no `coalesce`, dead-obvious and correct):**
   ```ruby
   project_seg = project ? "project:#{project.external_id}" : "project:org-wide"
   token =
     if project
       rel = project.project_playbook_assignments
       "#{rel.count},#{rel.maximum(:updated_at)&.to_f || 0}"   # "N,<epoch.frac>" — nil→0 for empty set
       # .to_f (not .to_i): fractional epoch closes the same-wall-clock-second double-edit window (gate-2 nit #1)
     else
       "org-wide"
     end
   cache_key = "playbooks/for_stacks/org:#{org.external_id}/#{project_seg}/asg:#{token}/stacks:#{stacks.sort.join(',')}"
   ```
   Two aggregate queries on cache-miss only; both hit the `(project_id)`-indexed table.
   `count` catches add/delete; `max(updated_at)` catches in-place edits (priority / `legacy_mode`
   toggle) that leave the count unchanged. Compute the token — and the `find_by!` — **before**
   `Rails.cache.fetch` (constraint 2). Inside the `fetch` block, apply
   `project_scoped_playbooks(all_playbooks, project, active_only: true)` before the stack filter.
5. `for_sync`: replace its inline zero-empty branch with
   `project_scoped_playbooks(all_playbooks, project, active_only: false)`.
6. OpenAPI + `bin/rails openapi:validate STRICT=1`.

**Gate**: `cd api && PARALLEL_WORKERS=1 bin/rails test test/controllers/api/v1/playbooks_controller_test.rb test/sentinels/` + `rubocop` + `openapi:validate STRICT=1`.

### Stage 2 — CLI: thread `project_id` into `for_stacks` fetch (enforcement + file sync)

Files: `cli/src/services/api_client.rs`, `cli/src/commands/code.rs`.
Tests: `cli/src/services/api_client.rs` inline `#[cfg(test)]` (wiremock: assert `project_id`
query present when Some, absent when None).

1. Failing wiremock tests for `get_playbooks_for_stacks`: param appended iff `Some`; **plus a
   test that a 404 on the project-scoped call triggers exactly one org-wide retry** (finding #2).
2. Change signature: `get_playbooks_for_stacks(&self, org_id, stacks, project_id: Option<&str>)`;
   append `("project_id", pid)` to `query_params` only when `Some`. (reqwest `.query` builder.)
3. Caller `code.rs:1193`: pass `resolve.project_id.as_deref()`. `None` → param omitted → server
   org-wide branch (constraint 5).
4. **404-retry-to-org-wide (gate-2 finding #2).** A stale/deleted `project_id` (disk-cached
   resolution) would 404 and, left as-is, degrade to *zero* playbooks — a regression below
   today's org-wide baseline. So: when the fetch returns `Err(CrewkitError::Api { status: 404, .. })`
   **and** `project_id` was `Some`, retry once with `None` (org-wide) before falling through to
   the existing non-fatal `playbook_failed` path. Net effect: project-scoped when possible,
   org-wide when the project can't be resolved server-side, never silently empty. (Server still
   returns 404 with no oracle — constraint 1 intact; this is purely CLI degradation.) While here,
   confirm `resolve.project_id` is only `Some` when server-confirmed; the retry is the safety net
   regardless.
5. **Second call site (gate-2 finding #3): `cli/src/commands/init.rs:350`** also calls
   `get_playbooks_for_stacks` — pass `None` (`crewkit init` is org-wide + pre-project, uses
   `selected_org.id`, no resolved project). No retry needed there (already `None`).
6. Update any test constructing these calls.

**Gate**: `cd cli && just fmt && just lint && just check && cargo test --bin crewkit`.

### Stage 3 — CLI: thread `project_id` into `for_sync` (MCP `get_conventions`)

Files: `cli/src/services/api_client.rs`, `cli/src/commands/mcp.rs`.

1. Failing wiremock test for `get_playbooks_for_sync_raw`: `project_id` present iff `Some`.
2. Signature: `get_playbooks_for_sync_raw(&self, org_id, stacks, format, project_id: Option<&str>)`;
   append only when `Some`.
3. `mcp.rs:524` (`get_conventions`): pass `scope.project_id.as_deref()` (guard is implicit —
   `Option`). Blueprint tool already models the `if let Some` pattern.

**Gate**: same CLI gate as Stage 2.

### Stage 4 — Sentinel + integration + docs

1. Add/extend a **tenant-isolation sentinel** (`api/test/sentinels/`) asserting a cross-org
   `project_id` on `for_stacks` returns 404 with a body identical to the unknown-id 404 (no
   existence oracle). This is the highest-stakes invariant (gate-1 constraint 1).
2. `integration-tests/` (only if time permits / spanning components): a session-start fetch with
   a project that has assignments returns the scoped set end-to-end.
3. Docs: update CLAUDE.md domain-model note ("sync & enforcement paths currently stop at the org
   tier — only Blueprint generation reads assignments") — now enforcement reads assignments too.
   Update `docs/planning/situation-handoff-2026-07-07.md` §3.1 to "shipped."
4. **Out of scope, note explicitly**: `crewkit playbooks list` (uses the *subscriptions*
   endpoint, not `for_sync` — different change) and convention customizations (decision C) are
   fast-follows; `list.rs`'s "--project not supported" warning stays honest.

---

## Risks / edge cases

- **Assignment-token query cost**: two AR aggregates (`count` + `maximum(:updated_at)`) per
  uncached session start when a project is present — indexed, negligible; only on cache miss.
- **`max(updated_at)` granularity**: `.to_f` gives fractional-second precision and `count` is a
  co-token, so add/delete/edit are all caught with no realistic staleness window.
- **`applicable_playbooks` staleness** (org subscribe/unsubscribe) is pre-existing 15-min behavior
  on the org-wide key; this change does not widen it (project path recomputes intersection inside
  the fetch block).
- **Priority-order vs stack-filter**: `assigned_ids` preserves assignment priority; the existing
  in-memory stack filter runs after and preserves order. No behavior change for the org-wide path.
- **Backward compat**: unchanged for every project with zero assignments and every `None`-project
  CLI invocation — the dominant case today.
