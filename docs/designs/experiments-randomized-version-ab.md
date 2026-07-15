# Design: Experiments — randomized version A/B (Slice B), phased & validated

**Status:** PLAN — validated by security-expert (PROCEED-WITH-CHANGES), rails-expert
(FEASIBLE-WITH-CHANGES), adversarial-code-reviewer (CHANGES-NEEDED). All folds
incorporated below. Awaiting owner green light. No code yet.
**Component:** api (+ CLI snapshot fidelity, + dashboard Deploy button).
**Supersedes:** the observational slice (BLOCKED).

## Goal
Randomly split traffic between two **versions** of a Resource (agent), serve each
arm its version, attribute each session's outcome to the version it **actually ran**,
compare **concurrently** with a valid test, and **promote the winner** (human-gated).
On the Resource runtime system only — never `AgentExperiment`/`AgentConfiguration`.

## THE metric reframe (validation's biggest correction)
The winner is judged on **feedback quality scores**, and — critically — **primary
agents never accrue `session_count`**: `aggregate_resource_stats_from_sessions_job.rb:210`
does `next if has_primary && resource_type == "agent"`. So for the main use case
`ResourceStat.session_count == 0` and the only real signal is `feedback_count` +
`overall_score` (written by `feed_resource_score → update_feedback_scores`).
**Therefore the sample unit, the gate, and the weighting all key on `feedback_count`/
`total_feedback`, NOT `session_count`.** (Weighting by session_count → 0/0 → nil →
never significant. This is the crux, not the z-test math.)

## Design principles (safety-first)
1. **Fail-safe serving is inviolable, and PER-RESOURCE.** The bulk `/resources/effective`
   `map` has no rescue (`resources_controller.rb:194`); an error on one experiment'd
   resource must NOT 500 the whole sync. Wrap override resolution per resource → on any
   raise, serve that resource's **current** version; others unaffected.
2. **Deterministic sticky-per-user assignment.** No `session_id` exists at sync time
   (sync precedes `POST /sessions`), so assignment keys on **user (+project)**:
   `MD5("#{comparison.id}-#{user.id}") % 2`. A user never flips arms → no attribution
   contamination. Reuse/extend `VersionBenchmarkService.get_benchmark_version` to accept
   a user key rather than adding a parallel method.
3. **Human-gated deploy.** Completion marks a recommendation; promoting requires an
   explicit manager action; reversible via `rollback_to_version!`.
4. **Backward-compatible attribution.** New `version_hash:` params default to
   `resource.version_hash` — existing callers unchanged.
5. **Tenant-scoped + platform-forbidden.** Experiments only on org/project-owned
   resources of the caller's org; **platform-owned resources are rejected** (a platform
   experiment would split every tenant at once — the highest-harm hole). Assignment keys
   on internal `user.id` only, never email/name.

---

## Phase 0 — Attribution foundation (ship first; the crux)
Stamp `ResourceStat` by the version each session **actually ran**. Add optional
`version_hash:` (default `resource.version_hash`) to `record_session`, `record_feedback`,
`find_or_create_stat`. Thread the **per-source** served hash — there are FOUR sources,
not just primary:
- **Feedback path (the one the winner is judged on):** `SessionAnalysisService#feed_resource_score`
  (`session_analysis_service.rb:435`) — primary uses `session.primary_resource_version_hash`;
  the subagent loop (`:424`) passes `sa.resource_version_hash`.
- **Count/duration path** (`AggregateResourceStatsFromSessionsJob`): source 1 subagent
  (`sa.resource_version_hash`, ~:84), source 2 skill (`si.resource_version_hash`, ~:117),
  source 3 primary (`ls.primary_resource_version_hash`, ~:153 — note primaries are skipped
  for agents), source 4 snapshot leaf (~:221).
- **MUST test** both paths agree on the same hash for a session (no split across two buckets).
- **Blast radius (regression-test, don't assume):** `ResourceStat.aggregate` feeds `get_stats`,
  `performance_report#by_version`/`avg_quality_score`, `resource_stats_controller` endpoints,
  `compare_versions`, `UpgradeNotificationService`, and the `AggregateResourceStatsJob` rollup
  (already groups by `version_hash` — safe). Schema-safe (composite unique key includes
  `version_hash`, `schema.rb:1453`).
- **Standalone value (honest framing):** "correct historical per-version breakdown." It does
  NOT remove the temporal confound — that needs P1's concurrent split.

## Phase 1 — Serve-time assignment + version pin + snapshot fidelity (do together)
- **Batch-load** running comparisons once before the serve loop:
  `ResourceVersionComparison.running.where(resource_id: ids, project_id: [project&.id, nil])`
  → an override map. (Never a per-resource query inside the map — N+1.)
- Pass a **scalar** `version_override:` into each `compute_all` call (resolved from the map
  in the controller), NOT a `{id=>hash}` hash into the service (avoids the
  `resolve_to_project_override` swap keying bug, `:223`).
- In `compute_all`: swap `resource.content` reads for `get_versioned_content(resource, override)`
  (fallback-to-current already covers a missing/deleted candidate) **and** stamp
  `build_chain`'s emitted `version_hash` (`:183`) with the served hash, so the effective
  response → CLI snapshot → `LlmSession.primary_resource_version_hash` → attribution all agree.
- Apply in **BOTH** serve paths: `build_effective_resource_data` (bulk, `:910`) **and**
  `effective_single` (`:241`) — via one shared helper, or the two diverge.
- **Per-resource fail-safe** (principle 1) wraps the whole thing; null-user → current.
- Result (with P0): users split, each served + snapshotted + attributed to their arm.

## Phase 2 — Concurrent measurement + valid stats + completion
- Fold a **feedback_count-weighted** mean into `ResourceStat.aggregate`:
  `SUM(overall_score * feedback_count) / NULLIF(SUM(feedback_count), 0)` over scored rows
  (matches the existing rollup weight, `aggregate_resource_stats_job.rb:108`). Use it for
  both `score_delta` and the test.
- Replace `calculate_significance` with a two-proportion z-test on that rate; `n =
  total_feedback` per arm; pooled SE; `p_value = Math.erfc(z.abs/Math.sqrt(2))`;
  `MIN_SAMPLE_SIZE = 30` unified (bump the `calculate_results` gate to 30 **and** delete
  the in-method check; also reconcile `VersionBenchmarkService::MIN_SESSIONS_PER_VERSION`
  / `has_sufficient_data?`). Guards → not significant: nil rate, `se==0`, below-min, clamp
  sqrt arg ≥ 0. Never raises (endpoint can't 500).
- Fix `has_sufficient_data?`/`auto_complete_if_ready` to read the now-populated per-arm
  ResourceStat (feedback-based). **Retire the dead counter path** —
  `record_benchmark_session`, `baseline/candidate_session_count`, `calculate_progress`,
  `/benchmarks/:id/version` — rather than ship two counting mechanisms. Auto-complete marks
  recommendation; **never deploys.**
- **Honesty copy (must ship with the math):** (a) randomized field experiment on real
  workloads, not a lab RCT; (b) metric is a mean quality score, conservative test; (c)
  **clustering caveat** — randomization is per-user but analysis is per-feedback-observation,
  so a few heavy users in one arm can overstate confidence (per-user analysis needs per-user
  stats we don't aggregate; documented, not silently "valid"). Recommendation stays advisory.

## Phase 3 — Deploy = human-gated promotion
- Deploy endpoint takes only `comparison_id` (NEVER a version param). Server reads
  `comparison.candidate_version_hash`, verifies the resource belongs to the caller's org,
  Pundit manager+, and gates on `completed? && recommendation == "upgrade" && is_significant`.
  → `resource.rollback_to_version!(candidate_hash)` (existing transactional, snapshot-safe
  promotion; post-promote hash == candidate hash, content-addressable). PaperTrail; reversible.
- Reject platform/inherited/cross-org/incomplete/insignificant/unauthorized.

## Phase 4 — Dashboard (mostly wiring + one real piece)
Existing: `VersionComparisonCard` is already rendered on the resource detail page
(`resources/[id]/page.tsx:641`); benchmark API client/hooks/query keys exist. **New work
(name it):** a create-experiment control (pick candidate version), live per-arm
feedback counts + significance display, and a **Deploy-winner button + mutation + MSW
handler** for the P3 endpoint. Small but not zero.

---

## Cross-cutting: creation authz + platform ban (gate P1/P3)
`start_benchmark` (`version_benchmark_service.rb:27`) has no authz — the create endpoint
MUST Pundit-gate (manager+ on the resource's org) and reject platform-owned / cross-org /
inherited resources. This is the same gate that closes the all-tenant-split hole.

## Loi 25 / transparency (gate P4 exposure — Félix/Rémi)
No new PII processing (assignment = hash of internal user.id; no personal attribute keyed
or exposed; no new egress). BUT silently serving different agent instructions to different
named developers and scoring per-arm is a **transparency/consent posture** call. Needs a
one-line product/governance sign-off + a team-docs note that experiments run, **before the
P4 dashboard exposes it.** Non-blocking for P0–P3 build.

## Dead-machinery disposition (avoid shipping two systems)
- Retire the dead benchmark-count path (P2, above).
- `AgentExperiment`/`AgentConfiguration` config-A/B remains the disjoint dead-end —
  deprecate/relabel its "Experiments" dashboard page separately (product decision, flag
  to Félix/Rémi). NOT touched by this work.

## Tests (per phase)
- **P0:** per-source served-version stamping (subagent/skill/primary/snapshot) → correct
  buckets; feedback path stamps the served hash; default-param backward-compat; the two
  paths agree (no double-count); blast-radius regression (by_version, compare_versions,
  performance_report, UpgradeNotification).
- **P1:** deterministic sticky assignment (same user→same arm, ~50/50 across users); served
  content == arm version; snapshot/`primary_resource_version_hash` == served arm; **fail-safe
  sentinel** (broken/deleted candidate, null user, raising lookup → current version, other
  resources still served, sync succeeds); both serve paths consistent; **platform-resource
  experiment rejected**; cross-org isolation; no N+1 (one comparisons query per sync).
- **P2:** z-test golden values on feedback-rate; significant when truly different / inconclusive
  when not; guards → not significant; auto-complete marks-not-deploys; retired counters gone.
- **P3:** deploy promotes candidate from the record, gated (completed+upgrade+significant+
  manager+own-org); param-supplied version ignored; incomplete/insignificant/cross-org/
  platform/inherited → rejected; reversible.
- **Sentinels:** platform-experiment ban; per-resource serve fail-safe; cross-org serving
  isolation; deploy authorization + record-derived candidate.

## Explicit non-goals
Per-session assignment; auto-deploy; multi-arm (>2); inherited/pinned-parent resources;
touching AgentExperiment/AgentConfiguration.

## Rollout / reversibility
No `running` experiments → path is inert (kill-switch by construction; test byte-identical
sync vs pre-feature). Each phase reversible: P0 default-compatible; P1 fail-safe + inert-when-
none; P2 pure computation; P3 human-triggered + `rollback_to_version!`-reversible.
