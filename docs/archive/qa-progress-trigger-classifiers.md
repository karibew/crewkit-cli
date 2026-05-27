# QA Session: §11.2.B trigger classifiers (SessionEvent → SessionWarning → dashboard)
**Started**: 2026-05-21
**Mode**: Team
**Scope**: ~25 files across api + dashboard (feature only — Ruby 3.4.9/Rails 8.1 toolchain churn EXCLUDED, it's a separate landed workstream)
**Review Waves**: 3

**Status: COMPLETED — 9 findings, 9 fixed, 0 deferred.**

## DOD Checklist
- [x] All review waves completed (W1 security+contract; W2 rails-quality + dashboard; W3 alignment collapsed — contract already verified twice)
- [x] All findings documented
- [x] All findings fixed (zero deferred)
- [x] Tests pass (API 142 runs/403 assertions/0 fail incl. controller+model+smoke; dashboard typecheck + vitest 9/9)
- [x] Build clean (rubocop 0 offenses; openapi:validate pass; tsc clean; eslint pass on touched files)
- [x] Progress file updated with final status

## Coordination note
Reviewers report findings via **TaskCreate** (concurrency-safe) + a summary message to the
lead. The **lead owns this progress file** — reviewers do NOT edit it directly (avoids
parallel-write conflicts; per concurrent-sessions-shared-checkout hazard).

## Scope — files under review
**API (new):** app/services/trigger_classifiers/{base,edits_without_tests,cross_tenant_without_org_id,secret_shaped_strings,acts_as_paranoid_bypass,billing_table_writes}.rb,
app/services/session_event_classifier_service.rb, app/jobs/classify_session_events_job.rb,
app/models/session_warning.rb, app/serializers/session_warning_serializer.rb,
db/migrate/20260521073500_create_session_warnings.rb, + tests
**API (modified):** app/controllers/api/v1/sessions_controller.rb, app/models/llm_session.rb,
swagger/v1/openapi.yaml, test/controllers/api/v1/sessions_controller_test.rb
**Dashboard:** src/components/features/sessions/warnings-section.tsx (+ __tests__, index.ts),
src/app/kit/projects/[id]/conversations/[conversationId]/page.tsx, src/types/api.ts

## Findings Summary
| Severity | Found | Fixed |
|----------|-------|-------|
| Critical | 0     | 0     |
| High     | 1     | 1     |
| Medium   | 2     | 2     |
| Low      | 6     | 6     |
**Total: 9 found, 9 fixed (0 deferred).**

## Findings Detail
| # | Severity | Description | File | Reviewer | Fix Status |
|---|----------|-------------|------|----------|------------|
| A1 | MEDIUM | `evidence` is NOT NULL (jsonb default {}) but missing from OpenAPI SessionWarning `required` list — 3-way drift; generated client would treat it optional | swagger/v1/openapi.yaml | api-reviewer | FIXED |
| A2 | LOW | `show#warnings` has no `.limit` (adjacent `plans` uses `.limit(20)`); unbounded on a pathological session | sessions_controller.rb | api-reviewer | FIXED |
| A3 | LOW | Doc debt made visible: subagent_sessions/plans not documented in LlmSession OpenAPI schema (warnings is the better pattern) — note, not regression | swagger/v1/openapi.yaml | api-reviewer | FIXED |
| S4 | HIGH | Warning `evidence` (raw SQL/code snippets, file paths, where-secret-appeared) exposed to ANY viewer; bypasses the owner/manager gate that hides `initial_prompt`. Gate warnings behind same predicate as LlmSessionSerializer#initial_prompt | sessions_controller.rb#show | sec-reviewer | FIXED (shared LlmSession#viewable_with_sensitive_context?; ungated → warnings:[]) |
| S5 | LOW | `redact` reveals ~53% of a 15-char token; the `return s if s.length <= 12` branch returns secret VERBATIM (unreachable today — all 6 patterns min>12 — but latent footgun) | trigger_classifiers/secret_shaped_strings.rb | sec-reviewer | FIXED (≤5 chars revealed; <8 → [redacted]) |
| S6 | LOW | Job uses `insert_all!` (bang) — a fingerprint collision RAISES inside the retry-re-raising transaction, nuking the batch. Use `insert_all(unique_by: %i[llm_session_id fingerprint])` to degrade gracefully | classify_session_events_job.rb | sec-reviewer | FIXED (insert_all + unique_by ON CONFLICT DO NOTHING) |
| D1 | MEDIUM | Fragile icon-color hack: parsed Tailwind class string at runtime to color icon | warnings-section.tsx | dash-fixer | FIXED (explicit iconClass field) |
| D2 | LOW | File-chip React key collision on duplicate paths | warnings-section.tsx | dash-fixer | FIXED (key=`${i}-${file}`) |
| D3 | LOW | A11y: decorative severity icon not aria-hidden | warnings-section.tsx | dash-fixer | FIXED (aria-hidden) |

## Cleared by reviewers (verified safe, no action)
ReDoS (all regexes benchmarked linear on 64KB adversarial input), multi-tenant isolation (org_id/project_id sourced from org-scoped session; show reads via current_organization.llm_sessions), injection (read-only pipeline, no eval/SQL-interp), stored-XSS (dashboard renders escaped JSX, no dangerouslySetInnerHTML), raw secret never persists/logs (fingerprint hashes redacted match), severity enum bypass (classifiers hardcode allowlisted values), skipping paper_trail/paranoid (defensible for derived records), dependent: :delete_all (no orphans).

## Progress Log
- 2026-05-21 QA session started. Team "qa-review" created. Scope: ~25 files, api + dashboard. 3 waves planned.
- 2026-05-21 Wave 1 api-reviewer done: 3 findings (1 MED, 2 LOW). Contract clean (serializer↔OpenAPI↔TS match; redaction verified).
- 2026-05-21 Wave 1 sec-reviewer done: 3 findings (1 HIGH, 2 LOW). Big risks cleared (ReDoS/tenant-iso/injection/XSS).
- 2026-05-21 Wave 1 total: 1 HIGH, 1 MED, 4 LOW. Dispatching rails fixer (all 6 are api/) + frontend quality review (parallel, disjoint dirs).
