# Design: deterministic `test_quality_score` (Phase D2, v1)

> Status: **gate-2 APPROVED w/ revisions (folded in)** · Author: planning session 2026-07-09 ·
> SDLC gate 1 ✅ (LLM-judge variant threat-modelled; that variant DEFERRED — see below) ·
> gate 2 ✅ (M-1..M-3 + L-1..L-3 folded in below) · build not started.
> Components: API (Rails) + small dashboard surface.

## Problem & the decision that shapes it

PLAN.md Phase D2 asks for a "test-quality evaluator." Gate 1 established that the literal
**LLM-as-judge over test code** would be crewkit's first egress of agent-written *code* to Anthropic
— under Loi 25/PIPEDA + Karibew's client-confidentiality rule it needs Félix/Rémi legal/DPA sign-off
(escalations E2/E3) and heavy gating. **Owner decision: skip the LLM path for now.**

This ships the **deterministic** alternative the research doc (`docs/testing-strategy-research.md`)
actually champions — "deterministic gates before LLM judges" — with **zero code egress and no LLM
call**: a `test_quality_score` on `llm_sessions` computed from the session's already-captured tool
events.

**Honest naming:** v1 measures test **discipline** (were tests written, and roughly in proportion to
the source changed?), NOT test content quality. We keep the roadmap column name `test_quality_score`
(so the deferred LLM judge can later refine the same field) but document the v1 semantics everywhere.

## Threat surface (collapsed vs the LLM variant)

No LLM, no egress, no new external surface. It reads the session's OWN already-stored `session_events`
and writes one clamped number.

**[M-1, corrected] Capture-mode reality (the design must NOT misstate this):** the scorer reads
`tool_input.file_path`. In `minimal` mode `tool_input` is stripped entirely
(`event_ingestion_service.rb` `SENSITIVE_DATA_KEYS` includes `"tool_input"`), so `file_path` is
absent ⇒ `source_files` empty ⇒ `test_quality_score = nil` for every minimal-mode session, silently
and **by design** (fail-safe: nil = "not applicable", never a false score). In `structured` (default)
and `full`, `tool_input` is retained — and it ALSO contains `tool_input.content` (the actual code).
So egress-safety comes from the **scorer's discipline** (it reads ONLY `tool_name` + `file_path` and
stores ONLY a number), NOT from content being absent. An implementer MUST NOT "fix" the minimal-mode
nil by reaching into content or loosening the sanitizer — that would reintroduce the egress this
feature was de-scoped to avoid.

Carried-forward gate-1 constraints that still apply:
- **Tenant scoping**: read only the target session's own events; write only its own score (trivially
  org-safe — it's the session's own data; no cross-session/cross-org joins).
- **Score clamp**: `test_quality_score` clamped to [0.0, 1.0] on write; DB range check (mirror
  `quality_score`).
- **No-clobber / idempotent**: deterministic over the same events → recompute yields the same value;
  safe to re-run. No fail-open path — insufficient data ⇒ leave `nil`.
- **Capture-mode robust**: needs only `tool_name`+`file_path` (present even in `minimal`); no reliance
  on stripped content, so no privacy gate needed (we never read code content).

Explicitly **NOT** in v1 (deferred, documented): the LLM content-quality judge (needs E1–E4 legal
sign-off); per-session reversion penalty (reversion is a lagging **org-level** metric — a revert-rate
analytics endpoint already exists; keep it there, don't fold `pr_reverted_at` into the per-session
score).

## Score definition (deterministic; v1)

Input: the session's `PreToolUse` `Write`/`Edit`/`MultiEdit` events. Classify each edit's `file_path`:
- **[L-1] Mutually exclusive, test-precedence** (a `*_spec.rb` under `app/` is a test, not source):
  - `test_files` = distinct files where `test_path?(p)`.
  - `source_files` = distinct files where `source_path?(p) && !test_path?(p)`.

Rules:
- **Applicability gate**: if `source_files` is empty (no source changed — read-only/planning/docs
  session, or no edit events) → `test_quality_score = nil` ("not applicable", not zero). Only sessions
  that changed source are test-scored.
- **Score** when applicable:
  - `test_files` empty → **0.0** (changed source, wrote no tests — worst discipline).
  - else → `min(1.0, test_files.count.to_f / source_files.count)` — tests roughly in proportion to
    source changed. Rounded to 2 decimals, clamped [0,1].
- This is a transparent discipline proxy, documented as such (not content quality). The applicability
  gate + 0.0-when-no-tests + proportion shape are the contract.
- **[L-1] Advisory only** — like `quality_score`/`efficiency_score`, this is a coaching signal, NEVER
  an enforcement gate. The ratio is gameable (touch an empty test file) and a pure test-writing
  session scores nil; acceptable for an advisory metric, unacceptable as a gate. Do not wire it to any
  block/deny path.

## Reuse / structure

**[M-2] Extract the FULL "inspect an edit event" surface** — not just the path predicates — into a
shared module `TriggerClassifiers::EventEditInspection`, `include`d by both `Base` (behavior
unchanged) and the scorer, so there is **one audited code path that reads only `tool_name` +
`file_path`** and the scorer isn't a second place touching raw event data. Extract:
`SOURCE_PATH_PATTERNS`, `SOURCE_EXTENSIONS`, `TEST_PATH_PATTERNS`, `EDIT_TOOLS`, `source_path?`,
`test_path?`, `data_for`, `tool_input`, `tool_name`, `file_path`, `edit_tool?`, `pre_tool_use?`.
**Do NOT extract or call `edit_contents`** (it reads `content`/`new_string`) — leave it in `Base` for
the secret-scanning classifiers only; the scorer must never call it.
Constant-resolution verified safe: all classifiers reference these via subclass *instance* methods
(resolve through the ancestor chain after `include`); no class/singleton-level constant refs — so
`include`-ing into `Base` preserves behavior exactly.

## Stages (test-first; gate-3 trio at the end)

### Stage 1 — API: migration + scorer + compute job
Files: migration, `app/services/test_quality_scorer.rb`, `app/services/trigger_classifiers/file_path_heuristics.rb` (extracted), `app/services/trigger_classifiers/base.rb` (include the module), `app/jobs/compute_test_quality_score_job.rb`, `app/controllers/api/v1/sessions_controller.rb` (enqueue at complete).
- Migration: `add_column :llm_sessions, :test_quality_score, :decimal, precision: 3, scale: 2` (nullable) + range check constraint (mirror `check_llm_quality_score_range`). Update `db/schema.rb` (CI loads schema).
- `TestQualityScorer.call(llm_session)` → returns a Float in [0,1] or `nil`. Pure; reuses
  `EventEditInspection`. **[L-3] Scope the read** to `llm_session.session_events.where(event_type: "PreToolUse")` (select the `data` column) to bound memory — don't materialize the whole association. Distinct-file counting.
- `ComputeTestQualityScoreJob.perform(llm_session_id)` → loads session, computes, `session.update_column(:test_quality_score, score)` (skip validations/callbacks; clamp in the scorer; DB check constraint is the backstop). Idempotent; `RecordNotFound` rescued/logged like `ClassifySessionEventsJob`.
- Enqueue `ComputeTestQualityScoreJob.perform_later(@session.id)` at session-complete, beside `ClassifySessionEventsJob`. **[L-3] Single compute at complete is sufficient** (events immutable post-completion; deterministic → recompute is identical) — do NOT wire a redundant re-analysis re-trigger. Note the pre-existing late-event race shared with the sibling (acceptable: undercounts if events land after complete).
- Tests FIRST: scorer — source+no-tests→0.0; source+proportional-tests→scaled/1.0; no-source→nil; no-edit-events→nil; distinct-file counting (same test file edited 3× counts once); **[L-1]** `app/foo_spec.rb`-only session → test (nil source) not 1.0; **[M-1] minimal-mode event stream (no `tool_input`) → nil, and the scorer never reads content**; clamp. Job — writes the score; nil leaves column nil; not-found safe.

### Stage 2 — API: surface (serializer + insights + analytics)
Files: `app/serializers/llm_session_serializer.rb`, `app/controllers/api/v1/session_insights_controller.rb`, `app/services/session_analytics_service.rb`, `swagger/v1/openapi.yaml`.
- Serializer: add `test_quality_score` (Float or null) beside `quality_score`.
- Insights: return `test_quality_score&.to_f`.
- **[M-3] Analytics: `AVG(llm_sessions.test_quality_score)`** — plain AVG (SQL ignores NULLs, keeping legitimate `0.0` rows, excluding "n/a"). **MUST NOT** copy the sibling's `CASE WHEN analysis_status = 'completed'` gate — this score is written at complete, independent of `analysis_status`; gating on it would wrongly drop valid scores. Add it where `avg_quality_score` lives: **by-user (`:256`) and team-overview (`:417`)** — NOT the `summary` method (which has no quality avg). Test: a session with `analysis_status != 'completed'` but a non-null score IS included; a nil row is excluded (not 0).
- **[L-2] OpenAPI `description`** for `test_quality_score` AND `avg_test_quality_score`: "v1: deterministic test-writing discipline (distinct test files vs distinct source files changed); NOT content quality." `openapi:validate STRICT=1`.

### Stage 3 — dashboard: display
Files: `dashboard/src/lib/api/sessions.ts` (type), the insight panel (`insight-panel.tsx`), `dashboard/src/types/api-generated.d.ts` (regenerate). 
- Show `test_quality_score` as a stat beside quality/efficiency. **[L-2] The visible label MUST lead
  with "Test discipline"** (reserve "quality" for the JSON key only), with a tooltip ("deterministic:
  tests written vs source changed; content-quality judging is a future enhancement"). Null → "—"/"n/a"
  (not 0).
- Regenerate types (with `--export-type`); `validate:api-contract`. **Watch react-doctor** (no
  `form.watch`; this is display-only so likely N/A) and biome format before commit.
- Tests: panel renders the score; renders "n/a" when null.

### Stage 4 — gate-3 trio + docs
implementation-validator → adversarial-code-reviewer → security-expert. Update handoff §3.3 + PLAN.md
Phase D2 note (v1 deterministic shipped; LLM judge deferred pending legal). Record the LLM-judge
follow-up with its gate-1 escalations (E1–E4) so it isn't lost.

## Verification note
API (Stages 1–2) verified in **CI** (no local Postgres here — will re-confirm the Ruby-3 kwargs /
PaperTrail / filter_parameters style traps that bit the last API feature). Dashboard (Stage 3)
verified locally (test/lint/typecheck/format/react-doctor/contract).

## Gate-3 accepted follow-ups (non-blocking; all 3 reviewers APPROVED)
- **user_detail analytics**: the per-user drilldown (`SessionAnalyticsService#user_detail`) surfaces
  `avg_quality_score`/`avg_efficiency_score` but not `avg_test_quality_score` (gate-2 M-3 enumerated
  only by-user + team-overview, which are done). One-line follow-up to mirror it there for symmetry;
  no current dashboard consumer, so deferred.
- **Serializer consistency (pre-existing)**: `test_quality_score` emits a JSON number (`&.to_f`), but
  sibling `quality_score`/`efficiency_score` still emit BigDecimal-as-string through the serializer
  (pre-existing, unconsumed via that path — insights `&.to_f`s all three). Optional separate cleanup;
  out of scope here.
- **Diagnosability**: the job logs only the final score. A follow-up could log
  `test_files.size`/`source_files.size` (counts, never content) so a surprising 0.0/nil is
  self-explaining from logs. Deferred (would expand the scorer's return contract; metric is
  deterministic/reproducible in console meanwhile).

## Deferred follow-up (tracked, not built)
LLM-as-judge content-quality scoring over test code — needs gate-1 escalations **E1** (structured-mode
policy), **E2** (Loi 25 consent/notice for code egress), **E3** (per-client confidentiality sign-off),
**E4** (cost cap) resolved by Félix/Rémi first, then full-capture-mode-only + sensitive-excluded +
secret-scrubbed + structured-output-clamped + default-OFF per-org flag. Would refine the same
`test_quality_score` column.
