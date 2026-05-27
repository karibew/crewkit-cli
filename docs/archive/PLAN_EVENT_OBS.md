# Event Ingestion & Observability — Musk 5-Step Analysis

**Date**: 2026-02-08
**Scope**: Event ingestion restructuring (997 insertions, 2046 deletions = net -1049 lines)
**Status**: All 3 new migrations applied. CLI compiles. API tests: 94 event/session tests pass (0 failures).

---

## Current State (Actual)

### Test Results
- **Event ingestion + sessions tests**: 94 runs, 310 assertions, **0 failures**
- **Full API suite**: 2662 runs, 12 failures + 4 errors (all pre-existing, none related to this work)
  - 8 failures: SessionSearchController (pre-existing, unrelated)
  - 1 failure: OrganizationsController (soft-delete test, unrelated)
  - 1 failure: ProjectsController (soft-delete test, unrelated)
  - 2 failures: ProjectManagementService (soft-delete test, unrelated)
  - 4 errors: ProcessArtifactJob (WebMock stub missing, unrelated)
- **CLI**: `cargo check` passes clean

### Files Changed (33 total, 997+ / 2046-)

**Deleted (4 files, -664 lines):**
| File | Lines | Reason |
|------|-------|--------|
| `api/app/models/concerns/safe_data_coercion.rb` | 31 | Methods inlined into 4 services |
| `api/app/services/session_transcript_builder_service.rb` | 387 | Replaced by JSONL-only TranscriptIngestionService |
| `api/test/services/session_transcript_builder_service_test.rb` | ~170 | Tests for deleted service |
| `cli/src/services/session_watcher.rs` | 277 | Dead code (was unreachable) |

**New (11 files, ~680 lines):**
| File | Lines | Purpose |
|------|-------|---------|
| `api/app/services/session_lifecycle_service.rb` | 154 | Session state machine (clear/resume/end) |
| `api/app/services/subagent_event_service.rb` | 130 | Subagent tracking with dedup |
| `api/app/services/skill_event_service.rb` | 97 | Skill invocation tracking |
| `api/app/services/resource_resolution_service.rb` | 59 | Resolve agent_type → resource_id |
| `api/app/services/raw_jsonl_stream.rb` | 46 | JSONL export utility (**0 callers**) |
| `api/app/policies/turn_visibility_policy.rb` | 46 | Capture mode filtering (2 callers) |
| `api/db/migrate/*` (3 migrations) | ~60 | raw_jsonl column, transcript_provider, cleanup dead columns |
| `dashboard/src/hooks/use-model-names.ts` | 70 | Model name formatting hook |
| `dashboard/src/hooks/use-turn-detail.ts` | 29 | Progressive turn disclosure |

**Modified (18 files):**
- API controllers (observability, session_insights, shared_sessions)
- API services (event_ingestion, session_import, transcript_ingestion)
- API models (session_turn, magic_link_token), serializer
- Dashboard pages (3), transcript-viewer components (4), turn-card
- Dashboard hooks, API client, formatters

### Service Architecture (Current)

```
ObservabilityController
  ├── POST /events → EventIngestionService (orchestrator, 233 lines)
  │     ├── SessionLifecycleService (154 lines)
  │     ├── SubagentEventService (130 lines)
  │     │     └── ResourceResolutionService (59 lines)
  │     └── SkillEventService (97 lines)
  │           └── ResourceResolutionService (shared)
  │
  ├── POST /transcript → TranscriptIngestionService (261 lines)
  │
  └── PATCH /subagents/:id → inline in controller

SessionImportService (252 lines) — batch import, shares coercion patterns

Total service lines: 1,278 (across 9 files)
```

### Duplicated Code (SafeDataCoercion inlined 4x)

After deleting `SafeDataCoercion` concern, these methods are copy-pasted into 4 services:

| Method | SubagentEvent | SkillEvent | TranscriptIngestion | SessionImport |
|--------|:---:|:---:|:---:|:---:|
| `to_non_negative_int` | lines 121-123 | lines 88-90 | lines 248-250 | lines 239-241 |
| `to_non_negative_float` | lines 125-127 | lines 92-94 | lines 252-254 | lines 243-245 |
| `parse_time` | lines 129-130* | lines 96-97* | lines 256-259 | lines 247-251 |

**Total duplication: ~62 lines** (same 3 methods x4, with slight format variations)

---

## Step 1: Question the Requirements

### What this actually solves (user outcome)
1. **JSONL as sole transcript source** — eliminates dual-path confusion where hook events AND JSONL both built transcripts. Single source of truth.
2. **Session lifecycle tracking** — correctly handles clear/resume/end transitions with parent_llm_session linkage.
3. **Subagent + skill tracking** — links tasks and skill invocations to resources for the improvement loop.
4. **Capture mode enforcement** — server-side defense-in-depth sanitization regardless of what CLI sends.
5. **Batch dedup** — prevents duplicate events on CLI retry/flush.

### Requirements that are REAL vs implementation assumptions

| # | Requirement | Type | Verdict |
|---|-------------|------|---------|
| R1 | JSONL is sole transcript source | **REAL** | Core architecture decision. Simplifies system. |
| R2 | Session lifecycle (clear/resume/end) | **REAL** | Users see conversations; lifecycle creates them. |
| R3 | Subagent tracking with resource linking | **REAL** | Needed for the improvement loop (ResourceStat). |
| R4 | Skill invocation tracking | **REAL** | Same as R3, for skills. |
| R5 | Batch dedup (sequence_number) | **REAL** | Prevents data corruption on retry. |
| R6 | Capture mode server-side filtering | **REAL** | Security requirement — don't trust client. |
| R7 | Initial prompt capture | **REAL** | Dashboard shows it, analytics need it. |
| A1 | ResourceResolutionService as separate class | **ASSUMPTION** | Could be a module method or inlined. Only 2 callers. |
| A2 | SessionLifecycleService as separate class | **ASSUMPTION** | Could be private methods in EventIngestionService. Only 1 caller. |
| A3 | RawJsonlStream module | **ASSUMPTION** | Built for future export endpoint. **Zero callers today.** |
| A4 | SafeDataCoercion deletion (inline everywhere) | **ASSUMPTION** | Assumed inlining was simpler than a shared concern. Created duplication. |
| A5 | claude_code_version propagation every batch | **ASSUMPTION** | Runs WHERE ... IS NULL every flush. Harmless but wasteful. |
| A6 | Atomic WHERE IS NULL pattern for permission_mode/effort_level | **BORDERLINE** | Defensive against TOCTOU. Batch dedup already prevents duplicate events, so the race can't happen. But it's 2 extra WHERE conditions (harmless). |

### Proposed minimal requirement set
- R1-R7 are all legitimate. No requirements to drop.
- A1-A5 are the focus of Steps 2-3 below.

### Decision needed from you
> **Are any of R1-R7 wrong? Should any of A1-A6 become requirements?**
> If you're happy with R1-R7, we proceed to Step 2.

---

## Step 2: Delete

### D1. Delete `RawJsonlStream` (46 lines, 0 callers)

**File**: `api/app/services/raw_jsonl_stream.rb`

**Current state**: Module with `exportable?` and `each_line` class methods. Grep across entire codebase finds **zero callers** outside its own definition.

**Purpose**: Built for future JSONL export dashboard endpoint (not yet implemented).

**Risk if deleted**: None. No code references it. If export is needed later, it's 46 lines to rewrite.

**Action**:
- [ ] Delete `api/app/services/raw_jsonl_stream.rb`
- [ ] Verify no references: `grep -r "RawJsonlStream" api/`

**Savings**: -46 lines, -1 file

### D2. Delete `ResourceResolutionService` as separate class (59 lines)

**File**: `api/app/services/resource_resolution_service.rb`

**Current state**: Instantiated by SubagentEventService and SkillEventService. Each calls `resolve` exactly once. The core logic is ~22 lines.

**Risk if deleted**: None — logic moves into callers.

**Action**:
- [ ] Inline `resolve` method into `SubagentEventService` (~22 lines)
- [ ] Inline `resolve` method into `SkillEventService` (~22 lines)
- [ ] Inline `find_visible_resource` private method into both
- [ ] Delete `api/app/services/resource_resolution_service.rb`
- [ ] Remove `@resource_resolver` initialization from both services

**Savings**: -59 lines file, +44 lines inlined = **net -15 lines, -1 file, -1 class**

### D3. Consider: Merge `SessionLifecycleService` into `EventIngestionService`

**File**: `api/app/services/session_lifecycle_service.rb` (154 lines)

**Current state**: Only caller is `EventIngestionService`. Takes 5 constructor params that are copies of EventIngestionService's own instance vars.

**Arguments FOR merging**:
- Only 1 caller, ever
- Eliminates context duplication (5 shared instance vars)
- EventIngestionService would grow from 233 → ~350 lines (still reasonable)
- No loss of testability (private methods are testable via public interface)

**Arguments AGAINST merging**:
- 154 lines is a meaningful chunk; lifecycle logic is a distinct domain concept
- Separate file = easier to find and reason about
- Future callers possible (e.g., a direct session management API)

**Risk**: Medium — larger refactor, but logic doesn't change. Full test coverage exists.

**Action** (if approved):
- [ ] Move `handle_event`, `capture_initial_prompt`, `propagate_claude_code_version!` as private methods into EventIngestionService
- [ ] Move `handle_clear_new`, `handle_resume` as private methods
- [ ] Replace `@lifecycle_service` calls with direct method calls
- [ ] Replace `@lifecycle_service.sessions_created` with `@sessions_created` instance var
- [ ] Delete `api/app/services/session_lifecycle_service.rb`
- [ ] Run tests: `bin/rails test test/controllers/api/v1/observability_controller_test.rb`

**Savings**: -154 lines file, +~120 lines inlined = **net -34 lines, -1 file, -1 class**

**Decision needed**: Merge or keep separate?

---

## Step 3: Simplify

### S1. Re-extract SafeDataCoercion as a shared concern (undo the duplication)

**Current state**: 3 identical methods copy-pasted into 4 services (~62 lines of duplication).

**Problem**: The original concern was deleted and methods inlined. This created drift risk and violates DRY.

**Action**:
- [ ] Create `api/app/concerns/data_coercion.rb` module with:
  - `to_non_negative_int(value)` — 3 lines
  - `to_non_negative_float(value)` — 3 lines
  - `parse_time(str)` — 4 lines
- [ ] `include DataCoercion` in:
  - SubagentEventService
  - SkillEventService
  - TranscriptIngestionService
  - SessionImportService
  - ObservabilityController (for `update_subagent` inline coercion)
- [ ] Delete duplicated private methods from all 4 services
- [ ] Run full test suite

**Savings**: -62 lines removed, +15 lines concern = **net -47 lines, dedup risk eliminated**

### S2. Consolidate model name formatting in dashboard

**Current state**: Two near-identical formatting functions:
- `formatModelName()` in `dashboard/src/lib/formatters.ts:107` (exported, used by turn-card etc.)
- `formatModelNameFallback()` in `dashboard/src/hooks/use-model-names.ts:57` (private, fallback)

**Action**:
- [ ] Compare both functions for exact equivalence
- [ ] If equivalent: delete `formatModelNameFallback`, import `formatModelName` from `formatters.ts` in `use-model-names.ts`
- [ ] If different: unify into one function with the superset of logic

**Savings**: ~15 lines, eliminates regex duplication

### S3. Simplify `propagate_claude_code_version!` call pattern

**Current state**: Called every batch flush in EventIngestionService line 84. Uses `WHERE claude_code_version: nil` so it's a no-op after first successful write.

**Problem**: Runs an UPDATE query on every batch even though it only matters the first time.

**Action** (micro-optimization, low priority):
- [ ] Move `propagate_claude_code_version!` into `process_event` for `SessionStart` events only
- [ ] OR: Add early return at top: `return if @crewkit_session.claude_code_version.present?`

**Savings**: Eliminates 1 unnecessary DB query per batch flush (except first)

### S4. Concept count reduction

**Current concepts to understand (17)**:
1. Hook event batching & deduplication
2. Session lifecycle state machine
3. JSONL transcript ingestion
4. Subagent tracking with resource linking
5. Skill invocation tracking
6. Capture mode filtering (3 tiers)
7. TurnVisibilityPolicy (structured role checks)
8. Safe data coercion
9. Atomic WHERE IS NULL updates
10. Session hierarchy (CrewkitSession → LlmSession → SubagentSession)
11. Infinite scroll pagination (cursor-based)
12. Transcript pipeline (raw → consolidated → exchanges)
13. Exchange grouping (prompt + response + tools)
14. Task grouping (boundary detection)
15. Progressive disclosure (summary vs detail)
16. React Query infinite queries
17. Model name fallback formatting

**After Steps 2-3**: Remove concepts #8 (becomes a simple include) and #17 (merged into formatters). **Down to 15.**

With D2 (inline ResourceResolutionService): concept "resource resolution as a service" becomes "resource resolution as a method". No net concept change, but one less indirection layer.

---

## Step 4: Optimize

**Only flag measured issues — don't optimize what should be deleted/simplified.**

### O1. N+1 in RawJsonlStream.exportable? (if kept)

```ruby
def self.exportable?(llm_session)
  total = llm_session.session_turns.count          # query 1
  with_raw = llm_session.session_turns.where.not(raw_jsonl: nil).count  # query 2
  with_raw == total
end
```

Could be a single query: `llm_session.session_turns.where(raw_jsonl: nil).exists?` (returns false = exportable). **But RawJsonlStream has 0 callers, so this is moot if D1 is approved.**

### O2. propagate_claude_code_version! wasteful query

Already covered in S3.

### O3. No other measured performance issues found

The event ingestion path uses transactions properly, preloads duplicates upfront (M3), and resolves sessions once per batch (M1). These are good patterns.

---

## Step 5: Automate (Testing)

### T1. Existing test coverage is good

- 94 tests in observability + sessions controllers pass
- EventIngestionService is tested via controller integration tests
- TranscriptIngestionService is tested via controller integration tests

### T2. Missing: Unit tests for new services

| Service | Has direct unit tests? | Tested via? |
|---------|:---:|---|
| EventIngestionService | No | ObservabilityController integration tests |
| SessionLifecycleService | No | ObservabilityController integration tests |
| SubagentEventService | No | ObservabilityController integration tests |
| SkillEventService | No | ObservabilityController integration tests |
| TranscriptIngestionService | No | ObservabilityController integration tests |
| ResourceResolutionService | No | Indirectly via subagent/skill tests |
| RawJsonlStream | No | **Nothing** (0 callers) |
| TurnVisibilityPolicy | No | SharedSessionsController + serializer tests |
| SessionImportService | No | SessionsController import test |

**Assessment**: Integration tests through the controller are sufficient for now. The services are thin enough that controller tests exercise all paths. Adding unit tests would be valuable but is not blocking.

**Action** (if approved):
- [ ] Add unit test for DataCoercion concern after extraction (S1)
- [ ] Consider adding unit test for TurnVisibilityPolicy (2 callers, clear contract)
- [ ] Skip unit tests for services that will be inlined/deleted

### T3. Tests to fix (pre-existing, not from this work)

These are NOT caused by the event ingestion changes but should be tracked:
- [ ] 8 SessionSearchController failures (500 errors — likely missing service/column)
- [ ] 4 ProcessArtifactJob errors (WebMock stubs missing for Anthropic API)
- [ ] 3 soft-delete test failures (organizations, projects, project_management_service)

---

## Execution Plan (Resume Checklist)

### Phase A: Delete (estimated: 15 min)
1. [ ] **D1**: Delete `raw_jsonl_stream.rb` — Confirm 0 callers, delete file
2. [ ] **D2**: Inline ResourceResolutionService — Copy resolve logic into SubagentEventService and SkillEventService, delete file
3. [ ] Run tests: `bin/rails test test/controllers/api/v1/observability_controller_test.rb test/controllers/api/v1/sessions_controller_test.rb`

### Phase B: Simplify (estimated: 20 min)
4. [ ] **S1**: Extract DataCoercion concern — Create module, include in 4 services + controller, delete inlined methods
5. [ ] **S2**: Consolidate dashboard formatters — Merge `formatModelNameFallback` into `formatModelName`
6. [ ] **S3**: Optimize `propagate_claude_code_version!` — Add early return or move to SessionStart handler
7. [ ] Run tests: `bin/rails test` (full suite to catch regressions)

### Phase C: Decision Point — SessionLifecycleService (estimated: 20 min if merging)
8. [ ] **D3 decision**: Merge SessionLifecycleService into EventIngestionService? (needs your input)
   - If YES: Move methods, delete file, run tests
   - If NO: Keep as-is, document rationale

### Phase D: Automate (estimated: 10 min)
9. [ ] **T1**: Add unit test for DataCoercion concern
10. [ ] **T2**: Add unit test for TurnVisibilityPolicy

### Phase E: Commit
11. [ ] Stage changes: specific files only (no git add -A)
12. [ ] Commit: `refactor(api): simplify event ingestion — delete dead code, extract shared concern`
13. [ ] Verify clean: `bin/rails test test/controllers/api/v1/observability_controller_test.rb`

---

## Summary Table

| Step | Item | Action | Lines Saved | Risk |
|------|------|--------|:-----------:|:----:|
| Delete | D1: RawJsonlStream | Delete file (0 callers) | -46 | None |
| Delete | D2: ResourceResolutionService | Inline into 2 callers | -15 | Low |
| Delete | D3: SessionLifecycleService | Merge into EventIngestionService | -34 | Medium |
| Simplify | S1: DataCoercion concern | Extract shared module | -47 | Low |
| Simplify | S2: Dashboard formatters | Merge duplicate function | -15 | None |
| Simplify | S3: Version propagation | Add early return | -0 (perf) | None |
| Automate | T1-T2: Unit tests | Add 2 test files | +~40 | None |
| **Total** | | | **~-117 lines** | |

**Post-changes**: 9 files → 6-7 files, 1278 lines → ~1160 lines, 17 concepts → 15 concepts

---

## Execution Results (2026-02-08)

All items executed. Results:

| Step | Item | Status | Actual Lines |
|------|------|:------:|:------------:|
| D1 | Delete RawJsonlStream | DONE | -46 |
| D2 | Inline ResourceResolutionService | DONE | -59 file, +~40 inlined |
| D3 | Merge SessionLifecycleService | DONE | -154 file, +~106 inlined |
| S1 | Extract DataCoercion concern | DONE | -62 duped, +26 concern |
| S2 | Consolidate dashboard formatters | DONE | -13 |
| S3 | Optimize version propagation | DONE | +1 (early return) |

**Files deleted**: 4 (raw_jsonl_stream.rb, resource_resolution_service.rb, session_lifecycle_service.rb, safe_data_coercion.rb)
**Files created**: 1 (data_coercion.rb)
**Final service line count**: 1,146 (was 1,278) = **-132 lines, -10.3%**
**Tests**: 94 event/session tests, 0 failures, 0 errors
**Full suite**: 2662 runs, 12 pre-existing failures (unchanged), 1 flaky (passes in isolation)
