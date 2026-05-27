# QA Session: Week 1 + Month 1 testing-strategy rollout
**Started**: 2026-05-15
**Mode**: Team
**Scope**: 14 open PRs (#112–#126, excluding the merged-elsewhere #124 issue), across api/, cli/, dashboard/, ci/, docs/
**Review Waves**: 3 planned

## DOD Checklist
- [x] Wave 0: PR scope verification (8 PRs carried 5 unintended Phase B commits each; force-rebased onto origin/main; PR #118 auto-closed → reopened)
- [x] Wave 1: Security + API contract review (15 findings logged)
- [x] Wave 2: Apply fixes for 15 findings (security-paranoid + rails-expert in parallel, branches updated, force-pushed)
- [x] Wave 3: Re-verified — sentinels green (85 runs / 165 assertions / 0 failures on #122 branch), openapi:validate STRICT=1 passes on #113 branch, 333 undocumented-routes (down from 341)
- [x] All findings documented in the table below
- [x] All findings fixed (zero deferred)
- [x] Tests pass (sentinel suite, vitest, cargo test --lib, openapi:validate STRICT=1)
- [x] Build clean
- [x] Progress file updated with final status

## Wave 0 result (2026-05-15)

| PR | before (files) | after (files) | status |
|---|---|---|---|
| #112 | 30 | **2** | ✅ rebased |
| #113 | 30 | **2** | ✅ rebased |
| #114 | 30 | **2** | ✅ rebased |
| #115 | 5  | 5  | already clean |
| #116 | 31 | **3** | ✅ rebased |
| #117 | 31 | **3** | ✅ rebased |
| #118 | 40 | **13** | ✅ rebased + reopened (auto-closed when force-push left it empty mid-recovery) |
| #119 | 30 | **2** | ✅ rebased |
| #120 | 4  | 4  | already clean |
| #121 | 2  | 2  | already clean |
| #122 | 12 | 12 | already clean |
| #123 | 2  | 2  | already clean |
| #125 | 13 | 13 | already clean |
| #126 | 29 | **1** | ✅ rebased |

5 Phase B AgentIdentity commits (04143c58, 35b1f08f, 2ea2f161, 5b35270f, 80ceb189) are still on local `main` and need separate handling — not part of this QA session's scope. Flagging in final summary.

Schema.rb conflict resolved during #118 rebase: kept version `2026_05_15_000003` (matches the renamed-time-but-otherwise-valid migration on top of origin/main's `2026_04_22_000003`).

## PR diff sizes (note suspicious counts)

| PR | files | lines | branch | expected scope |
|---|---|---|---|---|
| #112 cargo-nextest | 30 | 2682+9 | chore/cli-pin-cargo-nextest | 2–3 |
| #113 spec cleanup | 30 | 2662+625 | chore/api-openapi-remove-stale-entries | 2 |
| #114 CI gate flip | 30 | 2664+10 | feat/ci-block-on-openapi-drift | 2 |
| #115 W1 sentinels | 5 | 489+0 | feat/api-sentinels-first-three | 4–5 ✅ |
| #116 AGENTS.md | 31 | 2776+7 | docs/agents-md | 3 |
| #117 pg docs | 31 | 2695+10 | docs/macos-pg-parallel-test-workaround | 3 |
| #118 max_ci_rounds | 40 | 3394+16 | feat/cloud-job-max-ci-rounds | ~7 |
| #119 test-runner subagent | 30 | 2766+7 | feat/test-runner-subagent | 2 |
| #120 test-affected | 4 | 377+0 | feat/api-test-affected-and-hook | 4 ✅ |
| #121 CLI sanitizer | 2 | 484+0 | feat/cli-sanitizer-sentinel-proptest | 2 ✅ |
| #122 M1 sentinels | 12 | 1307+9 | feat/api-sentinels-complete-catalog | 6–8 |
| #123 Pundit audit | 2 | 407+0 | audit/pundit-scope-audit-and-sentinel | 2 ✅ |
| #125 revert-rate widget | 13 | 694+42 | feat/dashboard-reversion-rate-widget | ~12 ✅ |
| #126 research doc | 29 | 3372+7 | docs/testing-strategy-research | 1 |

**Wave 0 hypothesis:** the PRs marked 30+ files have inherited 5 unpushed Phase B commits from local main (AgentIdentity work). Verify and decide treatment.

## Findings Summary
| Severity | Found | Fixed |
|----------|-------|-------|
| Critical | 0     | 0     |
| High     | 1     | 1     |
| Medium   | 9     | 9     |
| Low      | 4     | 4     |

## Findings Detail
| # | Severity | Description | File | Reviewer | Fix Status |
|---|----------|-------------|------|----------|------------|
| 1 | HIGH | PR #118: `BlueprintExecutionService.handle_job_completion` case statement does not handle `halted` status. Orchestrator controller now calls it for ALL terminal statuses via `job.terminal?` (incl. halted), but case falls through to `else` branch, logs "Unexpected terminal status" warning, and returns without transitioning the linked blueprint task. Any blueprint task linked to a halted cloud job is left orphaned in its prior status. | api/app/services/blueprint_execution_service.rb:48-56 | api-contract | fixed |
| 2 | MEDIUM | PR #113: deleted `/{organization_id}/telemetry_settings` from both specs (correct — that path is not a route), but no replacement entry was added for the real `/{organization_id}/telemetry/settings` (GET + PATCH at routes.rb:315-318). STRICT=1 won't catch this (the validator only fails on stale spec, not undocumented routes), so the gap ships silently. Public/internal callers lose the typed contract for telemetry settings. | api/swagger/v1/openapi.yaml, api/swagger/v1/openapi-internal.yaml, api/config/routes.rb:315 | api-contract | fixed |
| 3 | MEDIUM | PR #118: orchestrator's `CloudJob` struct in Rust (orchestrator/src/jobs/types.rs:33-74) does NOT declare `max_ci_rounds` or `ci_rounds_used` fields. The API now sends both in the `claim_next` payload, but serde silently discards them. The "orchestrator already received it via claim_next" claim in the issue body is technically true at the wire level but the orchestrator can't actually act on the cap from the payload. Either add the fields to the struct (preferred — even if unused now, locks in the contract) or document that enforcement is API-only. | orchestrator/src/jobs/types.rs:33-74 | api-contract | fixed |
| 4 | MEDIUM | PR #120: `FALLTHROUGH_PATTERNS` in api/bin/test-affected does not include `api/swagger/v1/openapi.yaml` or `api/swagger/v1/openapi-internal.yaml`. The post-edit hook in .claude/scripts/post-edit-check.sh only inspects `api/app/*.rb`, so edits to the OpenAPI spec trigger zero local verification. Contract drift can ship from session edits unless CI catches it later. Add the spec files to FALLTHROUGH_PATTERNS (so sentinels+smoke run) and to post-edit-check.sh's dispatcher. | api/bin/test-affected:48-52, .claude/scripts/post-edit-check.sh:36-46 | api-contract | fixed |
| 5 | MEDIUM | PR #125: revert_rate service returns sparkline of fixed length 30 regardless of `window` param. For `window=7d`, frontend receives 30 sparkline points; for `window=30d`, also 30. Forward-compat concern from QA brief: "sparkline array length is the window in days". When real data lands, this either (a) gets retrofitted into a contract change downstream or (b) ships forever with mismatched semantics. The OpenAPI spec also doesn't pin the array length, so the contract is ambiguous. Either pin sparkline length to window-in-days in the service + document min/max items in spec, or document that sparkline is always 30 days irrespective of window. | api/app/services/session_analytics_service.rb:111-118, api/swagger/v1/openapi.yaml (revert-rate spec) | api-contract | fixed |
| 6 | LOW | PR #125: OpenAPI schema for revert-rate marks all response fields as optional (no `required:` arrays anywhere in `data`, `agent`, `human`, or sparkline items). The generated `api-generated.d.ts` ends up with `window?`, `agent?`, `rate?`, `n?`, all nullable. Forward-compat principle from QA brief: contract must be specific so frontend doesn't have to defensively handle missing fields when real data lands. Add `required: [window, agent, human, sparkline]` on `data`, `required: [rate, n]` on each bucket, `required: [date, agent_rate, human_rate]` on sparkline items. Spec also documents only 401; should also document 403 since `enforce_advanced_analytics` will reject non-paying tiers. | api/swagger/v1/openapi.yaml (revert-rate operation schema) | api-contract | fixed |
| 7 | MEDIUM | PR #122: `TokenService.refill!` fix has a narrow race window. The new pre-existence check + the existing `rescue ActiveRecord::RecordNotUnique` cover (a) sequential duplicate webhook deliveries and (b) simultaneous deliveries where validators run before either commit. The uncovered case: Thread A commits BETWEEN Thread B's pre-check (validator sees no row → false) and Thread B's validator run (now sees A's row → `RecordInvalid`). `credit!` wraps `RecordInvalid` as `TokenError`, which is NOT caught by `rescue ActiveRecord::RecordNotUnique`. `TokenError` propagates out of `refill!`, the webhook errors, Stripe retries. Fix: catch `TokenError` (or specifically the wrapped `RecordInvalid`) in `refill!`, or wrap the pre-check + `credit!` in a transaction with `SELECT ... FOR UPDATE` semantics. | api/app/services/token_service.rb:117-148 | security-reviewer | fixed |
| 8 | MEDIUM | PR #122: `token_math_sentinel_test.rb` does not exercise the `rescue ActiveRecord::RecordNotUnique` path. The idempotency test (`refill! is idempotent for duplicate Stripe invoice ids`) goes through the pre-existence check on the second call and never reaches `credit!`. If a future refactor drops the rescue (or the unique DB index), the sentinel still passes. The concurrent-debit test exists for debits but no equivalent concurrent-refill test was added. Add a `Concurrent::Future` test that disables the pre-check (or hits both threads tightly enough to bypass it) and proves the rescue catches the DB-level race. | api/test/sentinels/token_math_sentinel_test.rb:62-104 | security-reviewer | fixed |
| 9 | MEDIUM | PR #122: `token_math_sentinel_test.rb` header comment (lines 9-14, 17-22) claims `TokenService.refill!` relies on "the unique index on `token_transactions.stripe_payment_intent_id` plus a `rescue ActiveRecord::RecordNotUnique`" as the idempotency mechanism, but the actual implementation now relies primarily on the new pre-existence check. The sentinel docs are out of sync with the code being pinned. Update the docstring to describe the layered defense: (1) pre-check short-circuit (single-threaded duplicates), (2) RecordNotUnique rescue (DB-level race for true concurrent inserts). | api/test/sentinels/token_math_sentinel_test.rb:9-22 | security-reviewer | fixed |
| 10 | MEDIUM | PR #115: `StripeSignatureSentinelTest#with_webhook_secret` builds a `Minitest::Mock` that only `expect`s a single `dig(:stripe, :webhook_secret)` call. If anything else in Rails (Devise initializer, Sentry, middleware) touches `Rails.application.credentials` during the request processing inside the stub block, the mock raises `MockExpectationError` and the sentinel fails for the wrong reason. Tests pass today because Rails 8 happens not to touch credentials on this path, but this is brittle. Replace with a permissive object responding to `dig` that returns the secret for `[:stripe, :webhook_secret]` and delegates everything else back to real credentials. | api/test/sentinels/stripe_signature_sentinel_test.rb:103-116 | security-reviewer | fixed |
| 11 | MEDIUM | PR #115: `TenantIsolationSentinelTest::ALLOWLIST` has 11 entries; Issue #98 Q5 triage specifies "exactly these 13 classes" including `Webhooks::SendgridController` and `Webhooks::SlackController`. The sentinel's inline comment defends the gap as "planned for Phase B; add them here when the controllers land", but the Phase B controllers exist in 5 unpushed-to-origin commits on local main (per Wave 0 notes — `80ceb189` et al). When those commits land, the sentinel will fail with two new "offenders". Either coordinate with Phase B PR plan to add the entries simultaneously, or update the sentinel comment to reference the specific blocking commits so the next person merging Phase B doesn't get a confusing red CI. | api/test/sentinels/tenant_isolation_sentinel_test.rb:32-44 | security-reviewer | fixed |
| 12 | MEDIUM | PR #123: `DOCUMENTED_EXCEPTIONS` maps each Scope class to a rationale tag (e.g., `:equivalent_org_filter`), but the sentinel never cross-references those tags against `docs/audits/pundit-scope-audit.md`. The class set is enforced (entries must `safe_constantize`; non-stale check that resolve doesn't already call `organization_scoped`); the tag is only documentation. A future PR could change a tag from `:equivalent_org_filter` to `:platform_reference_data` without the sentinel catching that the audit doc no longer matches. Either parse the audit table and assert rationales line up, or document explicitly in the sentinel header that audit↔constant cross-check is manual. | api/test/sentinels/pundit_scope_sentinel_test.rb:67-87 | security-reviewer | fixed |
| 13 | LOW | PR #123: audit doc summary states "organization_scoped:equivalent: 10" and "documented_exception: 8" (lines 86-88), but counting the table rows yields 9 equivalent (rows 7, 16, 17, 19, 20, 22, 27, 28, 34) and 9 documented_exception (rows 1, 10, 11, 14, 21, 24, 30, 33, 35). Total still 35 = 17 direct + 9 + 9. Correct the per-category counts in the summary so the audit is internally consistent. | docs/audits/pundit-scope-audit.md:86-88 | security-reviewer | fixed |
| 14 | LOW | PR #122: `auth_sentinel_test.rb` "rejects valid JWT for a user with disabled_at set" has an `ensure @user&.update_column(:disabled_at, nil) end` block. Rails' default `use_transactional_tests = true` already rolls back `@user.update!(disabled_at: Time.current)`. The ensure is harmless but suggests cross-test pollution exists when it doesn't, and `update_column` skips paper_trail/callbacks (different semantics from the rollback). Remove the ensure or replace with a transactional fixture reset. | api/test/sentinels/auth_sentinel_test.rb:73-82 | security-reviewer | fixed |
| 15 | LOW | PR #123: `read_method_body` contains dead code at line ~233: `next if i == start_line - 1`. The loop `(start_line...lines.length).each do |i|` never yields `i == start_line - 1` (range is exclusive on both ends, starts at `start_line`). Inherited from earlier iteration; remove for clarity. No behavior change. | api/test/sentinels/pundit_scope_sentinel_test.rb:233 | security-reviewer | fixed |

## Closure (2026-05-20)

**12 of 14 PRs merged.** Two blocked on OAuth `workflow` scope (PRs #115 and #122 modify `.github/workflows/api-ci.yml`); user needs to either run `gh auth refresh -h github.com -s workflow` or merge via the GitHub UI.

| Status | Count | PRs |
|---|---|---|
| Merged | 12 | #112, #113, #114, #116, #117, #118, #119, #120, #121, #123, #125, #126, #127 |
| Blocked on workflow scope | 2 | #115, #122 |

The merged set includes the Phase B sidecar (#127 — AgentIdentity + inbound webhooks + race-safe usage metering), which was rebased onto post-#113-merge main; three schema.rb version-line conflicts resolved by keeping the latest migration timestamp (`2026_05_15_000003`).

Two non-test-strategy commits authored during the close-out drive:
- `11b24dd3` on #113: regenerated `dashboard/src/types/api-generated.d.ts` after spec cleanup (unblocked Contract Validation across 7 downstream PRs).
- `a068fb77` on #118: regenerated dashboard types for new CloudJob `max_ci_rounds`/`ci_rounds_used`/`halted` enum entry.

Pre-existing infra flakes (Playwright `postgres role "root"` and integration `chaos-resilience.test.ts > Empty fixture` failures) **were failing on main itself** for weeks; not blocking. Recommended separate workstream.

Local repository state: single branch (`main`), single worktree, no stale local branches. All agent worktrees pruned.

## Progress Log
- 2026-05-15 QA session started. Scope: 14 PRs across 5 components.
- 2026-05-15 Initial PR diff-size scan reveals 8 PRs likely carrying unintended Phase B commits (30+ files where 2–3 expected). Wave 0 to verify.
- 2026-05-15 Wave 1B (api-contract): reviewed PRs #113, #114, #118, #120, #125. 1 HIGH (halted blueprint orphan), 4 MEDIUM, 1 LOW logged. PR #114 clean. PR #113's 14 stale-spec deletions are all genuinely stale at the routes level — but one (telemetry_settings) was deleted because the path was wrong rather than because the endpoint went away; the real path /telemetry/settings is now undocumented (see finding #2).
- 2026-05-15 Wave 2A (security-paranoid fixes): findings #7, #8, #9, #10, #11, #12, #13, #14, #15 fixed across PRs #115, #122, #123. Three commits force-pushed; sentinel suite green (14 + 53 + 3 = 70 runs, 0 failures) and rubocop clean on touched files. Remaining 6 findings (#1–#6) are api-contract scope, awaiting Wave 2B.
- 2026-05-15 Wave 2B (api-contract fixes): findings #1–#6 fixed across PRs #118, #113, #120, #125. Four commits pushed. PR #118 also extended the orchestrator Rust types (CloudJob.max_ci_rounds/ci_rounds_used + JobStatus::Halted variant) alongside the BlueprintExecutionService halted-arm fix; targeted tests pass (blueprint_execution_service_test 29 runs, cloud_job_test 44 runs, session_analytics_controller_test 25 runs, test_affected_test 17 runs, dashboard vitest 1520 tests). OpenAPI STRICT validation passes on PR #113 branch (post-fix). All 15 findings now `fixed`.
