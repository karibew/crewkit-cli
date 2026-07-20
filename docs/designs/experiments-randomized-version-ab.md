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

## Gate-1 fold — P1 (security-expert, 2026-07-16): PROCEED-WITH-CHANGES
Re-run before P1 build (P1 changes what real users are served). Verdict PROCEED-WITH-CHANGES;
4 blocking findings folded in below. Stale-fact correction: creation **is** already Pundit-gated
(`benchmarks_controller.rb:57` → `authorize @resource, :experiment?`) — tighten the policy, don't add a gate.

- **B1 — platform experiments not actually banned (all-tenant split).** `ResourcePolicy#experiment?`
  short-circuits `return true if crewkit_owner?`, so staff could start a platform-resource experiment,
  which the batch serve query matches on *every* tenant. **Fold:** hard-reject non-`project_owned?` and
  `inherits?` resources in `VersionBenchmarkService.start_benchmark` **unconditionally** (service-level,
  before any policy bypass) → raises `ArgumentError`. AND the serve path is inert for any non-project /
  inherited resource **before** consulting the override map (neutralizes laundered/legacy `running` rows).
- **B2 — attribution is client-controlled (org-insider can rig the human-gated promote).** Buckets on the
  client-supplied `primary_resource_version_hash`; the server never persists which arm it served, so a
  patched CLI (or a manager who also creates the experiment) can report the arm they want to win. **Fold
  (design decision):** the fix (server **re-derives** the attributed arm from `MD5(comparison.id-user.id)`,
  or at minimum validates the reported hash ∈ {baseline, candidate}) lands in **P2**, where the stats are
  actually consumed for significance/promotion. Not exploitable pre-P2 (nothing reads per-arm stats yet).
  P1 must not foreclose it — assignment stays a pure deterministic function of `(comparison.id, user.id)`.
- **B3 — ambiguous arm when both org-level (`project_id: nil`) and project-level comparisons match one
  resource** → user could flip arms per request (attribution contamination). **Fold:** map builder picks a
  **deterministic** single comparison per resource — project-scoped wins over org-level, then lowest id.
- **B4 — reconcile to project-owned-ONLY for P1.** The policy only permits `owner_type == "project"`, and
  `resolve_to_project_override`'s resource-swap (`:223`) is a no-op only for `project_owned?` resources (an
  org-level experiment would silently no-op for users with a project override, skewing the sample).
  **Fold:** P1 is **project-owned + standalone only**; org-owned added to non-goals. Serve gate mirrors this
  (`return nil unless resource.project_owned? && resource.standalone?`).

Non-blocking, folded: **N1** raise create bar `developer?`→`manager?` (aligns with P3 deploy gate);
**N2** do NOT surface arm/comparison fields in the effective response (byte-compatible schema except the
already-content-addressed hash shift); **N3** scope the per-resource rescue to `StandardError` around the
experiment path only + `Sentry.capture_exception` (never mask an unrelated serving bug); **N4** full
`Digest::MD5.hexdigest(...).to_i(16)` parity, extend the existing service (no parallel method). **N5/N6**:
per-arm read surfaces (P2/P4) must not become a new session-visibility leak; P1–P3 add no PII/egress (Loi 25
clean), P4 exposure remains held for Félix/Rémi.

**Implementation approach (chosen):** the override is applied as an **in-memory, readonly `dup`** of the
resource with `content` + `frontmatter` + `version_hash` swapped to the assigned arm's `ResourceVersion`
(never persisted) — so every downstream read (merged content, frontmatter, `build_chain`'s emitted
`version_hash`, tier_breakdown) is automatically consistent, and the served hash == the arm hash the CLI
snapshots → attribution buckets under the arm that ran. Swapping both content+frontmatter matches what
P3's `rollback_to_version!` promotes. Missing/deleted candidate version → falls back to current (fail-safe).

## Gate-1 fold — P2 (security-expert, 2026-07-16): PROCEED-WITH-CHANGES
P2 is split into two reviewable commits: **P2a** (attribution integrity — the deferred B2 fix) and
**P2b** (valid stats + completion). Gate-1 corrected the B2 primitive and raised 4 blocking findings.

**The primitive change (B2-BLOCK-1/2/3): FREEZE the arm at session-create, do NOT re-derive at stats-write
time.** Re-derivation is (1) applied to the wrong layer — the poisoned quantity is the shared attribution
key `(resource_id, version_hash)` that FOUR write sources populate (subagent, skill, primary, snapshot),
so guarding only the primary path leaves a crafted subagent/skill/snapshot row claiming the experiment'd
`resource_id` + candidate hash + high score as an open poison; (2) unstable under account anonymization
(`user_id → NULL` breaks `MD5(comparison.id-user.id)`); (3) a lossy reconstruction of what serve did (serve
used the sync-request project + `running` status; write-time uses the client-supplied `session.project` + a
fragile time-window). 

**P2a design (folded):**
- At `POST /sessions`, when the resolved primary resource is `project_owned? && standalone?` and has a
  governing `running` comparison for the session's project scope (project-scoped > org-wide > lowest id,
  same precedence as P1) and the user is known, **persist frozen columns on `LlmSession`:
  `experiment_comparison_id` + `experiment_arm_version_hash`** (arm = `arm_version_hash(comparison, user)`,
  the same fn P1 served with), and **override `primary_resource_version_hash` with the arm** (server-
  authoritative — a lying client can't set it).
- **Source-agnostic guard** at every ResourceStat write (all 4 job sources + `feed_resource_score`'s primary
  AND subagent loop — the feedback/score path is the winner-judged one): for a resource under a **running**
  experiment (batch-loaded `running_experiment_resource_ids` set), force the owning session's frozen
  `experiment_arm_version_hash` iff it's that session's frozen primary; otherwise **EXCLUDE the write**
  (never fall back to the client hash — B2-BLOCK-1). Non-governed resources: unchanged.
- **Fail-closed, never client hash** when the arm can't be determined (null user, primary didn't resolve,
  cross-scope) → the governed write is excluded, biasing toward the un-experimented baseline, never poisoned
  (B2-BLOCK-2). Frozen columns are immutable ⇒ job re-runs + re-analysis are idempotent (same bucket forever).
- Scope identity: attribution binds to the session's own `primary_resource_id` + frozen comparison, only for
  `project_owned? && standalone?` — cross-tenant/laundered attribution structurally impossible (mirrors serve).

**P2b design (folded):** **B2-BLOCK-4** — the promotion input must fail **closed**: a single `MIN_SAMPLE=30`
floor **on `feedback_count`** enforced INSIDE significance so `is_significant` is structurally false below it
for auto-complete, manual `POST .../complete`, and the P3 read path alike (delete the duplicate in-method
`min_sample=30` and the `total_sessions` gate). `has_sufficient_data?`/`auto_complete_if_ready`/manual
complete read **live per-arm ResourceStat feedback**, never the retired counter columns. Weighted mean
`SUM(overall_score*feedback_count)/NULLIF(SUM(feedback_count),0)`; two-proportion z-test on that rate,
`p_value=Math.erfc(z.abs/√2)`, pooled SE, guards (nil/se==0/below-min/clamp√≥0) → not-significant, never
raises. Retire `record_benchmark_session` (no callers); repurpose/relabel the stored counts to
**feedback** counts written at completion (N2 — rename to `*_feedback_count` in the serializer contract to
avoid a metric lie on the P4 deploy screen). **N1 (non-blocking, fold into P2b):** auto-complete/manual
complete "first time significant" is optional-stopping → inflates false upgrades; add a min-observation
window / peek cap and put "sequential peeking, not fixed-horizon" in the honesty copy (alongside the
clustering caveat). **N3:** late deep-analysis feedback after `completed_at` is dropped — evaluate on a
settle delay, document. **N4:** keep the `NULLIF` divide-guard.

**Loi 25:** attribution stays internal-id-only (`user.id`, `comparison.id`), no new egress, no arm field in
any user/manager-facing response (P1 N2 holds); P4 exposure still held for Félix/Rémi.

**P2a accepted residuals (gate-3, documented — all fail-closed, non-blocking):** (a) **mid-run race** —
serve is at resource-sync, freeze at session-create; if a comparison *starts* in that window the session is
frozen/attributed to a freshly-derived arm though the user ran the pre-experiment content. Deterministic +
narrow; fail-toward-arm beats trusting the client hash. (b) **serve/create project divergence** — a client
that syncs under project A then creates the session under project B can diverge served-vs-frozen; confined to
the user's own deterministic arm, auditable via the frozen `experiment_comparison_id`. (c) **governed-stat
suppression blast radius** — a governed resource's legit-but-unfrozen sessions (pre-experiment start, null
user, unresolved primary) are `:exclude`d from ResourceStat for the whole experiment, so the resource's
*general* rollup undercounts during a run (never poisons). Worth a P2b observability metric on excluded-write
volume. (d) **stale-comparison arm credit** — attribution keys on the frozen comparison, not the currently-
running one; only bites if a version_hash is reused as an arm across two experiments on one resource. (e)
**`record_benchmark_session`** stays dead/unreachable until P2b retires it — wiring it up would bypass the
guard (re-open B2-BLOCK-1).

## P2b follow-ups (gate-3, non-blocking — tracked, not built)
- **Gate-3 adversarial caught + fixed IN P2b:** the feedback-weighted mean divided by `total_feedback`
  while its SQL numerator skipped NULL-score rows → unscored feedback dragged an arm's rate down and padded
  its n (spurious "upgrade"). Fixed: `ResourceStat.aggregate` adds `scored_feedback`
  (`SUM(feedback_count) FILTER (WHERE overall_score IS NOT NULL)`), and the weighted mean, the z-test n, and
  the MIN_SAMPLE floor all key on `scored_feedback`. Regression tests pin it.
- **Divergent second significance path (security follow-up 1):** `ResourceStatsService#calculate_statistical_analysis`
  (behind `GET .../stats/compare`) still uses the retired placeholder math + a session-count floor. Read-only
  analytics (not a promotion path) and fails closed for primary agents (session_count 0), but it can render a
  "statistically better" metric lie in the compare view. Route it through the two-proportion feedback z-test
  (or label its output a raw non-inferential delta) — separate slice.
- **Optional-stopping / peeking (N1):** auto/manual-complete fire the first time the floor+significance
  coincide, inflating false upgrades; documented in the honesty copy only. Add a peek-cap / min-observation
  window before **P3** makes the recommendation actionable.
- **Excluded-write observability (P2a residual c):** add a metric on governed-write exclusions so a long
  experiment silently under-counting a resource's general stats is observable.
- **Count-column rename → P4:** `baseline/candidate_session_count` (+ the `sessions:` status key) now carry
  FEEDBACK counts; rename to `*_feedback_count` when the P4 deploy UI + its contract land.

## Gate-1 fold — P3 (security-expert, 2026-07-16): PROCEED-WITH-CHANGES
Human-gated deploy = `POST /:org_id/resources/:resource_id/benchmarks/:id/deploy`, input is ONLY the
comparison id, server reads `comparison.candidate_version_hash`, promotes via `rollback_to_version!`.
Verified safe by construction: candidate is record-derived + resource-scoped (no cross-resource/tenant
promotion), and the `completed? && recommendation=="upgrade" && is_significant` AND-gate has no fail-open
branch. **3 blocking findings folded:**
- **B1 — post-promote hash can silently diverge.** `rollback_to_version!` restores content+frontmatter but
  NOT `license/attribution/license_url`, and `Resource#frontmatter_for_hash` folds provenance into
  `version_hash` → if the live resource's provenance differs from the candidate `ResourceVersion`'s, the
  recomputed hash ≠ candidate hash (mints a THIRD version, breaks content-addressability, mis-keys
  attribution). **Fix:** inside the deploy transaction, after promotion, ASSERT
  `resource.reload.version_hash == comparison.candidate_version_hash`, else raise → rollback → 422
  (fail-closed). Standalone authored agents (blank provenance) match and succeed; a provenance-divergent
  candidate fails closed rather than minting a wrong hash (restoring provenance too = follow-up).
- **B2 — no `crewkit_owner?` short-circuit on deploy.** `deploy_experiment?` must require genuine
  `belongs_to_project_organization? && manager?` (project-owned only, reject inherited) — do NOT copy
  `experiment?`'s `return true if crewkit_owner?`, or a platform owner + URL-supplied org could silently
  rewrite a client tenant's agent instructions (Loi 25 / client-isolation prohibited shape for a mutation).
- **B3 — single-lock atomicity.** `deployed_at.nil?` idempotency check, gate re-check, candidate-existence
  re-check, `rollback_to_version!`, and the `deployed_at`/`deployed_by_id` set ALL inside one lock;
  `deployed_at` set only after a successful rollback. Also wrap in `resource.with_lock` (resource-first,
  matching `start_benchmark`) to serialize sibling deploys (N1).

**Design notes folded:** deploy grants no capability a manager lacks (they already hold `rollback?`), so the
gate is a **safety rail, not a security boundary** — keep manager+, but ADD the audit (`deployed_by_id` +
`deployed_at` + PaperTrail) the plain rollback path lacks. New nullable
`resource_version_comparisons.deployed_at` + `deployed_by_id`; second deploy → 422 (idempotent). Service
re-rejects non-project/inherited (mirror `start_benchmark`). Candidate-version-deleted → `ArgumentError` →
422 (no partial mutation). Peek-cap fold: `auto_complete_if_ready` gets a `MIN_OBSERVATION_WINDOW` guard
(manual `complete` stays the manager's explicit call, documented alongside the CAVEAT).
**Non-blocking follow-ups (documented):** N2 sibling-experiment coherence (a still-running comparison on the
same resource now compares against a stale baseline — recommend auto-cancel-siblings-on-deploy or block;
product call, deferred); N3 legacy pre-P2b completed rows trusted (bounded — manager could rollback anyway);
N4 `set_resource` global lookup = 403-vs-404 enumeration oracle (minor, consistent with existing actions);
N6 manual complete has no min-window (human judgment + MIN_SAMPLE floor bound it).

## Rollout / reversibility
No `running` experiments → path is inert (kill-switch by construction; test byte-identical
sync vs pre-feature). Each phase reversible: P0 default-compatible; P1 fail-safe + inert-when-
none; P2 pure computation; P3 human-triggered + `rollback_to_version!`-reversible.
