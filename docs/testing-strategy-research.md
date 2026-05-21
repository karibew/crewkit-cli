# Testing Strategy Research — AI-Agent-Driven Development on crewkit

**Status:** Implemented through Month 1 as of 2026-05-20. See [Rollout status](#0-rollout-status) below.
**Date drafted:** 2026-05-06
**Scope:** Whole monorepo (api/, cli/, dashboard/, orchestrator/, integration-tests/).

---

## 0. Rollout status (2026-05-20)

**Week 1 + Month 1 shipped.** Tracked by issue [#111](https://github.com/karibew/crewkit/issues/111). All 14 PRs merged.

| Section | Deliverable | PR |
|---|---|---|
| §3.2 / §9 W1.1 | Block PRs on OpenAPI contract drift (STRICT=1 + remove continue-on-error) | [#114 ✅](https://github.com/karibew/crewkit/pull/114) merged |
| §3.2 sidecar | Remove 14 stale OpenAPI spec entries (prerequisite for STRICT=1) | [#113 ✅](https://github.com/karibew/crewkit/pull/113) merged |
| §4.1, §4.2, §4.4 / §9 W1.2 | `api/test/sentinels/` + first 3 sentinels (tenant, JWT scope, Stripe sig) | [#115 ✅](https://github.com/karibew/crewkit/pull/115) merged |
| §7.1 / §9 W1.3 | `AGENTS.md` at repo root | [#116 ✅](https://github.com/karibew/crewkit/pull/116) merged |
| §3.6 / §9 W1.4 | Document macOS pg parallel-test segfault | [#117 ✅](https://github.com/karibew/crewkit/pull/117) merged |
| §5.3 / §9 W1.5 | Pin `cargo-nextest` | [#112 ✅](https://github.com/karibew/crewkit/pull/112) merged |
| §11.4 / §9 W1.6 | `max_ci_rounds=2` on CloudJob (Stripe Minions pattern) | [#118 ✅](https://github.com/karibew/crewkit/pull/118) merged |
| §4.3, §4.5, §4.6, §4.9, §4.10 / §9 M1.1 | Complete sentinel catalog (incl. TokenService.refill! idempotency fix) | [#122 ✅](https://github.com/karibew/crewkit/pull/122) merged |
| §6.3, §7.2 / §9 M1.2 | `bin/test-affected` + PostToolUse hook | [#120 ✅](https://github.com/karibew/crewkit/pull/120) merged |
| §4.7 / §9 M1.3 | CLI `sanitizer_sentinel.rs` proptest | [#121 ✅](https://github.com/karibew/crewkit/pull/121) merged |
| §3.3 / §9 M1.4 | Pundit Scope audit (35 scopes, 0 gaps) | [#123 ✅](https://github.com/karibew/crewkit/pull/123) merged |
| §7.3 / §9 M1.5 | `test-runner` subagent | [#119 ✅](https://github.com/karibew/crewkit/pull/119) merged |
| §11.4 / §9 M1.6 | Reversion-rate widget (Shopify pattern) | [#125 ✅](https://github.com/karibew/crewkit/pull/125) merged |
| §0 (this doc) | Research foundation committed | [#126 ✅](https://github.com/karibew/crewkit/pull/126) merged |
| Sidecar | Phase B AgentIdentity (5 commits surfaced during Wave 0) | [#127 ✅](https://github.com/karibew/crewkit/pull/127) merged |

**Real bugs found and fixed during the rollout (not in the original gap analysis):**

1. **`TokenService.refill!` idempotency bug** (caught by #122's token-math sentinel). Pre-existing `rescue ActiveRecord::RecordNotUnique` was dead code — the AR validator fired first, raising `RecordInvalid` rewrapped as `TokenError`. Every duplicate Stripe webhook delivery would fail. Fixed via pre-existence check + scoped rescue.
2. **5 paranoid models with un-scoped uniqueness validators** (caught by #122's paranoia sentinel): `SessionShare#share_token`, `Project#slug`, `Organization#slug`, `AgentExperiment#slug`, `AgentConfiguration#agent_name`. Pinned in `KNOWN_OFFENDERS`; new offenders blocked; existing ones tracked separately.
3. **`BlueprintExecutionService` orphaned tasks** (caught by #118's QA review): the new `halted` cloud-job status wasn't handled, leaving linked blueprint tasks stuck in their prior state. Fixed.
4. **Orchestrator `CloudJob` Rust struct silently dropped `max_ci_rounds`/`ci_rounds_used`** (caught by #118 QA): serde discarded the fields because they weren't declared. Added with `#[serde(default)]`.

**Pre-existing flakes flagged but deferred** (failing on main since before this work):
- Playwright E2E: postgres `role "root"` infra bug
- integration tests: `chaos-resilience.test.ts > Empty fixture` failures

**Quarter 1 items still deferred** per the original phased rollout — trigger classifiers over `SessionEvent`, shadow mode for ResourceVersion changes, `api/test/system/` deletion decision, inner-loop adherence metric.

**Open Questions from §10** — 5 of 10 triaged in [#98](https://github.com/karibew/crewkit/issues/98) on 2026-05-15; Q3, Q7, Q8, Q10 still deferred.

---

## 1. Executive summary

1. **The current suite is large but slow-to-feedback for agents.** ~6,990 unit/integration tests across four languages. Dashboard runs 1,514 tests in **11.91s** wall-clock (great). CLI runs 1,047 tests in **~7s test execution + 60s build** (great once compiled). Rails API has **3,924 tests** that ran **85.3s** serially with `PARALLEL_WORKERS=1` locally, with **2 skips, 0 failures**; the parallel runner **segfaulted** at 8 workers (`pg-1.6.2-arm64-darwin/lib/pg/connection.rb:944`). CI is also pinned to `PARALLEL_WORKERS=1`. 85s is fine for end-of-PR but too slow for the inner loop — agents iterating in `api/` need sub-30s feedback on the change they just made, not an 85s full-suite run.
2. **CI is component-scoped, not module-scoped.** Workflows trigger on `api/**`, `cli/**`, `dashboard/**` path filters (`api-ci.yml:5`, `cli-ci.yml:5`, `e2e-tests.yml:5`). That's coarse. A one-line change to `app/services/token_service.rb` runs the same ~10-minute job as a model rewrite. There is no `nx affected`-equivalent inside a component.
3. **Sentinel coverage exists but is informal and incomplete.** Smoke tests live at `api/test/smoke/` (15 tests, runs in ~1s). They cover health, auth/me, device flow, and resource-effective. Nothing in there yet covers tenant-isolation regressions, money/token math, Stripe webhook signature verification, or cloud-job JWT scoping — all of which are the things most likely to silently break under heavy agent edits.
4. **Money math is concentrated and well-locked.** `TokenBalance#debit!` / `credit!` (`api/app/models/token_balance.rb:39-85`) use pessimistic locking, an immutable transaction log (`TokenTransaction` rejects mutation by design), and a unique index on `stripe_payment_intent_id` for idempotency. This is a model worth defending with a sentinel ring; an agent could still subtly break it by skipping `with_lock` or by changing the validation around `balance >= 0`.
5. **Multi-tenant isolation depends on a single base controller and Pundit.** `api/app/controllers/api/v1/base_controller.rb:104-126` wires `set_current_organization` and the 404-not-401 response that prevents enumeration. 43 of 75 models declare `belongs_to :organization`. **Nothing prevents an agent from writing a controller that bypasses `BaseController` (e.g., via `ActionController::API` directly, like `webhooks/stripe_controller.rb`), or a Pundit `Scope#resolve` that forgets to call `organization_scoped`.** No automated check for "this controller inherits from BaseController" or "this Scope filters by org" exists.
6. **The biggest gap for AI agents is *contract* and *invariant* testing, not coverage.** API openapi:validate already runs in CI (`api-ci.yml:168`); dashboard has `validate:api-contract` (e2e-tests.yml:171). Both are good but partial: openapi:validate is informational by default, and the dashboard contract job is `continue-on-error: true` (`e2e-tests.yml:174`) so a contract drift never blocks a merge. Strengthening these gates is cheaper than writing more unit tests.
7. **CLI test wall-clock is excellent but cargo build dominates total time.** `cargo test` ran 1,047 tests in ~7s of test execution but ~70s total wall-clock. With incremental builds an agent's inner loop will be 5–15s. Adding `cargo-nextest` (cli/justfile already references it; not installed locally) would compress further and is the single biggest CLI feedback-loop win.
8. **The Rails parallel-test segfault on macOS is real and recurring.** Reproduced cleanly on this checkout: 8-worker `bin/rails test` from a clean DB segfaults inside `pg-1.6.2-arm64-darwin/lib/pg/connection.rb:944`. Documented in CLAUDE.md as a "slug collision" workaround using `PARALLEL_WORKERS=1`, but the real pain is local: agents cannot get a fast green-light from running the full suite, so they don't, so they push and wait for CI.
9. **Pushback: the framing in the brief should change.** The brief implies "pick a tier strategy and add a hooks.md." The real bottleneck is **observability of what agents broke**, not test-tier proliferation. Crewkit already has rich coverage (~7K tests) — what's missing is (a) a sub-30s sentinel ring an agent can run *before* declaring done, (b) machine-readable test selection within `api/`, and (c) hard gates on contract drift. Adding a fifth tier ("AI-judge eval") is a Phase-D problem; the inner-loop problem is solvable now without ML.
10. **Recommended near-term targets.** Build a `bin/sentinels` (~30 sentinel-class assertions, sub-30s) covering tenant isolation, JWT scopes, money math idempotency, and webhook signature verification. Wire `cargo-nextest` into the CLI pre-push hook. Flip `validate:api-contract` and `openapi:validate STRICT=1` to **block** merges. Add a single `PostToolUse` hook in CLAUDE.md that runs the language-appropriate fast-tier on edited files. Don't invest in affected-test selection until you measure the actual cost — most components are already <15s.
11. **Industry research convergence (added 2026-05-06; full detail in Section 11).** Parallel research on Shopify, Stripe, SpaceX, Tesla surfaced five cross-company patterns: (a) **production-sampled eval sets > synthetic** (Shopify GTX, Stripe real-API grader, Tesla data engine — synthetic eval sets demonstrably collapse in production); (b) **deterministic gates before LLM judges** (Shopify N-Stage Gated Rewards, Stripe Minions blueprint nodes); (c) **reversion rate as north-star metric**, not test count or coverage (Shopify); (d) **hard caps on agent thrash** — Stripe Minions cap CI retries at 2 before human handoff; (e) **shadow mode** for resource-version changes (Tesla → directly applicable since `LlmSession` already captures the substrate). Two concrete adds to the Week 1 / Month 1 rollout in Section 11.4: `max_ci_rounds = 2` on `CloudJob`, and a reversion-rate telemetry widget tagging `agent_authored` PRs.

---

## 2. Current state inventory

### 2.1 Languages and toolchains

| Component | Language | Test framework | Test count | Wall-clock (local) |
|---|---|---|---|---|
| `api/` | Ruby 3.3.8 / Rails 8.0.3 | Minitest + webmock + paranoia + paper_trail | **3,924** runs (11,125 assertions) across 224 files | **85.3s** serial (`PARALLEL_WORKERS=1`); **segfaults** at parallel=8 on macOS |
| `cli/` | Rust 2024 edition (stable) | inline `#[test]` (cargo) + `assert_cmd` integration tests | 926 inline + 121 integration = **1,047** | ~7s test exec, ~70s with build |
| `dashboard/` | TypeScript 5 / Next.js 16.1.6 | Vitest 4 + RTL + MSW + Playwright | **1,514 unit/component/hook** + 13 E2E spec files (~221 cases) | **11.91s** vitest |
| `orchestrator/` | Rust 2021 (stable) | none | **0** | n/a |
| `integration-tests/` | TypeScript 5 / Node 22 | Vitest + axios | **223** | requires API+CLI; not run locally |

LOC reference (excluding generated files, `node_modules`, `target/`):

```
api/app           49,644 LOC of Ruby
api/test          57,812 LOC of test code
cli/src           62,817 LOC of Rust
cli/tests          1,979 LOC of integration tests
dashboard/src    134,532 LOC of TS/TSX
dashboard/src/**__tests__**  28,320 LOC of test code
dashboard/e2e      5,736 LOC of Playwright spec
integration-tests  3,560 LOC of Vitest
orchestrator       2,384 LOC of Rust (no tests)
```

### 2.2 Test directory layout

**API (`api/test/`)** — 224 test files:
```
controllers/api/v1/        74 files
models/                    57 files
services/                  55 files
jobs/                      12 files
policies/                   9 files
serializers/                5 files
integration/                4 files (auth/soft-delete, rack-attack, headers, session-resource bridge)
smoke/                      4 files (auth, health, resources, sessions; ~15 tests, <1s)
mailers/                    2 files
system/                     2 files (Rails default scaffolds, mostly idle since UI moved out)
```

**Dashboard (`dashboard/src/**/__tests__/`)** — 101 test files:
```
hooks/__tests__/                39 files
components/features/**/__tests__/  56 files
lib/api/__tests__/               5 files (incl. contract-validation.test.ts: 44 cases)
lib/adapters/__tests__/          1 file
test/mocks/__tests__/            1 file
```

Plus `dashboard/e2e/` with 13 Playwright spec files (auth, agents, admin, navigation, project-subroutes, etc.). `playwright.config.ts:31` runs `fullyParallel: false` with `workers: 1`.

**CLI (`cli/`)** — 8 integration test binaries plus inline tests:
```
tests/cli_help.rs        23 cases (CLI help, version, subcommand help)
tests/cli_auth.rs        19 cases (auth commands without server)
tests/cli_code.rs        13 cases
tests/cli_errors.rs      13 cases
tests/cli_status.rs      10 cases
tests/cli_feedback.rs     8 cases
tests/cli_whoami.rs       6 cases
tests/cli_update.rs       6 cases
src/                    926 inline `#[test]` / `#[tokio::test]` across 70 modules
```

Largest test files (top 5 by lines):
- `api/test/controllers/api/v1/auth_controller_test.rb` 1,249
- `api/test/controllers/api/v1/resources_controller_test.rb` 1,174
- `api/test/controllers/api/v1/sessions_controller_test.rb` 1,049
- `api/test/controllers/api/v1/observability_controller_test.rb` 890
- `api/test/services/stripe_webhook_service_test.rb` 781

### 2.3 CI configuration

`.github/workflows/`:

| Workflow | Trigger | Jobs | Notes |
|---|---|---|---|
| `api-ci.yml` | `api/**` PR + main | `lint` (rubocop+brakeman), `test` (full Rails suite, **PARALLEL_WORKERS=1**), `test-smoke`, `openapi-validation`, `coverage-comment` (65% gate) | `test` ~5–10 min in CI; coverage gate enforces ≥65% on PRs |
| `cli-ci.yml` | `cli/**` PR + main | `check` (fmt+clippy+check), `test` (3-OS matrix: ubuntu/macos-xlarge/windows, with cargo-nextest, doc tests, ci-profile binary build, llvm-cov 50% gate on Linux) | Slowest of the three CLIs because of cross-platform build |
| `e2e-tests.yml` | `dashboard/**` or `api/**` | `dashboard-unit` (vitest+coverage), `e2e-tests` (Playwright against built dashboard+API), `contract-validation` (**continue-on-error: true**), `e2e-staging` (main only) | Contract validation is informational, not blocking |
| `integration-tests.yml` | cli, api, integration-tests | Full stack: builds CLI, starts API, runs `npm test` in `integration-tests/` | 223 tests; 20-min timeout |
| `nightly-regression.yml` | cron 06:00 UTC + manual | `api-full`, `dashboard-unit`, plus other regression jobs | Only path that runs the entire Rails suite reliably |
| `release-cli.yml`, `release-please.yml`, `deploy-agent.yml`, `sync-public-docs.yml` | misc | release/deploy paths | not in scope here |

CI selection mechanism today is **GitHub Actions `paths:` filters** at the workflow level. There is no within-component selection — touching `api/app/controllers/api/v1/sessions_controller.rb` runs the same ~3,930 tests as touching `api/app/services/anthropic_service.rb`.

### 2.4 Existing agent-instruction files

- `CLAUDE.md` (root, 14KB) — domain model, four pillars, API endpoints, commit rules. Mentions `cli/` pre-push checks but no testing-specific guidance for agents.
- `.claude/CLAUDE.md` — pointer to playbooks.
- `.claude/rules/general.md` — naming, conventional commits, security baseline.
- `.claude/rules/api.md` — Rails patterns; testing section says "Run tests: `bin/rails test`" and that's all.
- `.claude/rules/cli.md` — pre-push checklist (`just fmt && just lint && just check && just test`).
- `.claude/rules/dashboard.md` — points to Playwright MCP for browser tests (note: not the same as `npm run test:e2e`).
- `.claude/rules/testing.md` — the most useful existing doc. Lists the `bin/rails test`, `npm test`, `cargo test` commands and the test-tier table. **No mention of sentinels, fast/slow split, or what an agent should run after editing a specific file.**

There is **no `AGENTS.md`** anywhere in the repo, no `.cursorrules`. The project uses the Claude rules system (`.claude/rules/*.md`) keyed on path globs.

### 2.5 Static analysis

- Ruby: rubocop-rails-omakase, brakeman; both gate `api-ci.yml:lint`.
- TypeScript: ESLint + `tsc --noEmit` via `npm run lint`. `tsconfig.json` strict mode is on (CLAUDE.md confirms).
- Rust: rustfmt + clippy with `-D warnings` (`cli-ci.yml:50`); pre-commit hook also enforces.
- No Sorbet/RBS in the Ruby codebase.
- API has **OpenAPI specs** at `api/swagger/v1/openapi.yaml` (public) and `openapi-internal.yaml` (admin); dashboard generates types from the public spec into `dashboard/src/types/api-generated.d.ts` via `npm run generate:api-types`.

---

## 3. Gap analysis (where this fails under heavy agent use)

### 3.1 Inner loop is too slow in `api/`

After an agent edits `app/services/token_service.rb`, the natural feedback signal is `bin/rails test test/services/token_service_test.rb`. That actually works (sub-second once Rails boots). But when an agent edits a **controller** or a **policy**, the natural reach is `bin/rails test test/controllers/api/v1/...` which takes much longer; and an agent that doesn't know to scope down will run `bin/rails test`, which is **85s** serial locally (measured this run) or **segfaults** at the macOS-default parallel count.

**Concrete example:** an agent modifying `api/app/controllers/api/v1/sessions_controller.rb` (the most-edited file in `api/app/`, 14 commits in 90 days, 1,049-line spec file). A change there should ideally run only `sessions_controller_test.rb`. Without guidance it risks running the full 85s suite, or hitting the segfault and either (a) reporting "tests broke" when nothing is wrong, or (b) ignoring tests entirely.

### 3.2 Contract drift can land silently

`e2e-tests.yml:174` — `contract-validation` job has `continue-on-error: true`. That means if an agent renames a Rails endpoint and forgets to update `swagger/v1/openapi.yaml`, the dashboard's generated types diverge from reality, vitest still passes (because MSW handlers are hand-written), and the bug reaches Playwright E2E or production. Same for the openapi:validate step in `api-ci.yml:194` — by default it's informational; only `STRICT=1` fails on drift. CI does not pass `STRICT=1`.

### 3.3 Tenant isolation has no automated invariant check

`BaseController#set_current_organization` (`api/app/controllers/api/v1/base_controller.rb:104`) is the trust boundary. The convention is: every API controller inherits from `BaseController`. **Nothing enforces this.** Counter-examples already exist:

- `api/app/controllers/webhooks/stripe_controller.rb:18` — `class StripeController < ActionController::API`. Correct (webhooks must not auth via JWT) but illustrative: an agent could trivially write a "convenience" endpoint outside `BaseController` and lose tenant scoping.
- `api/app/policies/application_policy.rb:65` — `Scope#organization_scoped` is the reusable helper. Subclasses can simply *not call it* and resolve the unscoped relation. No test asserts every Pundit Scope invokes `organization_scoped` (or some equivalent filter).

### 3.4 Stripe webhook idempotency depends on a unique index

`TokenTransaction.stripe_payment_intent_id` has `validates :stripe_payment_intent_id, uniqueness: true, allow_nil: true` (`api/app/models/token_transaction.rb:28`). Combined with `TokenService.refill!` rescuing `ActiveRecord::RecordNotUnique` (`token_service.rb:130`), this gives idempotent webhooks. **An agent removing the `rescue` block or the unique constraint would not be caught by current tests** — `stripe_webhook_service_test.rb` tests happy-path paid invoices but I did not find a sentinel test that asserts "double-processed `invoice.paid` does not double-credit." (Reading 781 lines of spec was outside this pass — flag in Open Questions.)

### 3.5 The CLI's `event_sanitizer` is the user-facing privacy layer and could regress invisibly

`cli/src/services/event_sanitizer.rs` (1,048 lines) sanitizes hook events before sending to the API. A regression here means **leaked file content into telemetry**. Inline tests exist (the 926 inline `#[test]` cases include sanitizer tests) but there's no policy test of the form: "for every known event type, content fields are stripped in `Structured` mode." An agent adding a new event type or tool name could miss the sanitizer path.

### 3.6 Rails parallel-test instability on macOS

Reproduced this run: clean schema, 8-worker `bin/rails test`, segfault inside `pg-1.6.2-arm64-darwin/lib/pg/connection.rb:944` shortly after "Running 3924 tests in parallel using 8 processes". The `PARALLEL_WORKERS=1` workaround (used in CI per `api-ci.yml:74`) sidesteps it but turns the suite into a 3–10 minute serial run on a developer machine. **An agent will not naturally hit the workaround** unless we tell it to.

### 3.7 No top-level test orchestrator

`package.json:test:all` is `npm run test && npm run test:integration` — but `test` only invokes `test:cli`. The dashboard suite and the API suite are not in `test:all`. An agent reading the root `package.json` to discover "how do I run tests" gets a misleading subset.

### 3.8 Skips and ignored tests are minimal — not a gap

Only **2 real `skip` calls** in `api/test/`:
- `api/test/integration/rack_attack_test.rb:120` — localhost is safelisted; the throttle response is implicitly tested in production.
- `api/test/policies/user_organization_role_policy_test.rb:45` — an owner-permission check that needs controller context.

No `.skip` / `.todo` markers in dashboard tests. No `#[ignore]` in the CLI. **This is a positive signal**: the team has resisted the temptation to skip-then-forget.

---

## 4. Sentinel test catalog

These are the invariants that, if silently broken by an agent, are most likely to cause user harm. Each entry includes a proposed assertion and a recommended location. **Recommendations only — no tests are being added in this pass.**

Ranked by criticality (1 = highest):

### 4.1 Tenant isolation — every authenticated controller scopes to `current_organization`

- **What protects it now:** `BaseController#set_current_organization` and `BaseController#user_has_access_to_org?` (`api/app/controllers/api/v1/base_controller.rb:104-126`). Pundit Scopes use `organization_scoped` from `application_policy.rb:65`.
- **Sentinel assertion:** "for each controller class under `Api::V1`, it inherits (transitively) from `BaseController`; for each `*Policy::Scope#resolve`, it filters by `organization_id` (statically check via `inspect.method_source` or a registry pattern)."
- **Where:** new file `api/test/sentinels/tenant_isolation_sentinel_test.rb`.
- **Cost:** ~1 day. Some tolerance needed for legitimate exceptions (`webhooks/stripe_controller.rb`, `health_controller.rb`).

### 4.2 JWT scope enforcement (`agent_job` cannot access non-allowlisted endpoints)

- **What protects it now:** `BaseController#enforce_agent_job_scope!` with `AGENT_JOB_ALLOWLIST` (`api/app/controllers/api/v1/base_controller.rb:137-159`).
- **Sentinel assertion:** "an `agent_job`-scoped JWT cannot reach `/api/v1/projects`, `/api/v1/organizations`, `/api/v1/admin/*`, etc.; it CAN reach exactly the 6 endpoints in the allowlist."
- **Where:** `api/test/sentinels/jwt_scope_sentinel_test.rb`.
- **Cost:** ~half a day. One request-spec covering 8–10 endpoints is enough.

### 4.3 Token math: debit cannot underflow, credit cannot exceed `MAX_BALANCE`, refill is idempotent

- **What protects it now:** `TokenBalance#debit!` raises on negative balance (`api/app/models/token_balance.rb:42-55`); `credit!` raises on `> MAX_BALANCE` (`token_balance.rb:71-77`); `TokenService.refill!` rescues `RecordNotUnique` keyed on `stripe_payment_intent_id`.
- **Sentinel assertion:** "concurrent debits do not double-spend (use `Concurrent::Future` x2 with a balance of `n` and assert exactly one succeeds when both want `n`); double-processing the same `invoice.paid` event credits the org exactly once."
- **Where:** `api/test/sentinels/token_math_sentinel_test.rb`.
- **Cost:** ~1 day. The concurrent test is fiddly but valuable.

### 4.4 Stripe webhook signature verification — only signed events processed

- **What protects it now:** `Webhooks::StripeController#create` calls `StripeService.verify_webhook` (`api/app/controllers/webhooks/stripe_controller.rb:24-32`), returns 400 on bad signature, then delegates.
- **Sentinel assertion:** "POST /webhooks/stripe with a payload but no/wrong `Stripe-Signature` header returns 400 *and* `StripeWebhookService.process` is not called."
- **Where:** `api/test/sentinels/stripe_signature_sentinel_test.rb` (or augment existing `stripe_webhook_service_test.rb`).
- **Cost:** ~2 hours.

### 4.5 SSRF protection on `Repository.remote_url`

- **What protects it now:** `Repository#remote_url_is_safe` + `private_or_internal_host?` (`api/app/models/repository.rb:79-125`) — blocks loopback, private, link-local, `localhost`, `.local`, `metadata.google.internal`.
- **Sentinel assertion:** "a curated list of malicious URLs (`http://localhost`, `http://169.254.169.254/latest/meta-data/`, `http://10.0.0.1`, `git@host.local:org/repo`, `http://metadata.google.internal/`, IPv6 `::1`) ALL fail validation."
- **Where:** `api/test/sentinels/ssrf_sentinel_test.rb` (existing `repository_test.rb` may already cover some — verify before duplicating).
- **Cost:** ~2 hours.

### 4.6 Authentication negative paths — disabled accounts, expired tokens, nonexistent users

- **What protects it now:** `BaseController#authenticate_api_request!` (`base_controller.rb:22-64`).
- **Sentinel assertion:** "401 returned for: missing token, malformed token, expired token (`JWT::ExpiredSignature`), unknown sub, account with non-null `disabled_at`."
- **Where:** `api/test/sentinels/auth_sentinel_test.rb`.
- **Cost:** ~3 hours. Some of this is already in `auth_controller_test.rb` (1,249 lines) — sentinel version is a tight subset that runs in <100ms.

### 4.7 CLI event-sanitizer never emits secret material in `Structured` mode

- **What protects it now:** `cli/src/services/event_sanitizer.rs` strips `content`, `old_string`, `new_string` from `tool_input` per the table at `event_sanitizer.rs:18-23`.
- **Sentinel assertion:** "for a synthetic `PreToolUse` event for each tool name (`Read, Write, Edit, MultiEdit, Bash, Task, Skill, Glob, Grep, WebFetch, ...`), the sanitized output contains no field with the literal `'AKIA...'`/`'sk-ant-api03-...'` injected at known positions in the raw input." Property test with `proptest` (already a dev-dependency, `cli/Cargo.toml:129`).
- **Where:** `cli/tests/sanitizer_sentinel.rs`.
- **Cost:** ~1 day. proptest setup is minimal but enumerating tools needs care.

### 4.8 OpenAPI spec ↔ routes ↔ generated types

- **What protects it now:** `api/lib/tasks/openapi.rake` (presumed) + `dashboard/scripts/validate-api-contract`.
- **Sentinel assertion:** **flip both gates to blocking**: (a) `api-ci.yml` runs `bin/rails openapi:validate STRICT=1`; (b) `e2e-tests.yml` removes `continue-on-error: true` from `contract-validation`.
- **Where:** workflow YAML, no new tests required — this is a CI flip.
- **Cost:** ~1 hour after one merge cycle to unblock real drift.

### 4.9 Soft-delete-aware uniqueness — `acts_as_paranoid` records don't shadow

- **What protects it now:** Repository validates `uniqueness: { scope: :organization_id, conditions: -> { where(deleted_at: nil) } }` (`repository.rb:27`).
- **Sentinel assertion:** "for each model with `acts_as_paranoid` AND a uniqueness validator, the validator is scoped to `deleted_at IS NULL` so a soft-deleted record can be re-created."
- **Where:** `api/test/sentinels/paranoia_uniqueness_sentinel_test.rb`. This is a meta-test that scans all models.
- **Cost:** ~3 hours.

### 4.10 Rate-limit gates are configured (rack-attack throttles still wired)

- **What protects it now:** `api/config/initializers/rack_attack.rb` (auth=5/20s, general=300/5m, authenticated=1000/5m).
- **Sentinel assertion:** "auth endpoint returns 429 after `auth_limit + 1` requests within `auth_period`. (Already partially asserted in `rack_attack_test.rb` but the localhost-safelist is in the way locally; sentinel should run in production-like mode.)"
- **Where:** `api/test/sentinels/rate_limit_sentinel_test.rb` with the safelist explicitly disabled.
- **Cost:** ~3 hours.

**Out-of-scope for sentinels but worth noting:**
- PII filtering in Sentry (`config/initializers/sentry.rb:37`) is a one-time invariant — easy to verify by reading; less valuable as a test.
- `filter_parameter_logging.rb` covers `:passw, :email, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn, :cvv, :cvc` — adequate as-is.
- Tax math: not present in the codebase. Crewkit doesn't compute tax — Stripe does. Skip.

---

## 5. Recommended test tiering

**Goal:** every agent edit gets a feedback loop matched to its scope. Don't run the whole suite when a 5-second sentinel pass would catch the realistic break.

### 5.1 Tiers (per language)

| Tier | Purpose | Wall-clock budget | Run when |
|---|---|---|---|
| **Sentinel** | Catch high-stakes invariant breaks (Section 4) | ≤30s | Every agent inner loop; pre-push hook; first job on every CI run |
| **Fast (unit)** | Pure unit, no DB, no network | ≤60s per component | After every meaningful edit |
| **Integration** | DB, fixtures, multi-service | ≤5 min per component | Before merging; on PR push |
| **E2E** | Browser + API + CLI working together | ≤10 min | On PR ready, on main |
| **Nightly / heavy** | Slow regressions, perf, full Playwright matrix | unlimited | cron |

### 5.2 API (Ruby/Rails/Minitest)

```bash
# Sentinel (sub-30s) — NEW
bin/rails test test/sentinels/

# Fast — model + service + policy unit tests (no controllers)
bin/rails test test/models/ test/services/ test/policies/ test/serializers/

# Integration — controllers + integration tests
bin/rails test test/controllers/ test/integration/ test/jobs/ test/mailers/

# Smoke (already exists, ~1s) — health/auth/sessions/resources
bin/rails test test/smoke/
```

**Naming convention:** sentinel files end in `_sentinel_test.rb` and live under `test/sentinels/`. Both directory and suffix exist purely so they can be globbed.

**Wall-clock target for sentinel + smoke + fast:** under 60s on `PARALLEL_WORKERS=1`. The full suite measured at **85.3s** serial locally — that's the upper bound, run only when the change touches integration paths or `BaseController`/concerns/migrations.

**Action item:** put `parallelize(workers: ENV.fetch("PARALLEL_WORKERS") { 1 }.to_i)` in `test_helper.rb` (currently it caps at 8) until the macOS pg crash is resolved upstream, OR document that local agents must `export PARALLEL_WORKERS=1`. Prefer the latter; CI already pins it.

### 5.3 CLI (Rust)

```bash
# Sentinel (NEW)
cargo test --test sanitizer_sentinel

# Fast — inline unit tests (current 926 cases)
cargo test --lib

# Integration — assert_cmd binaries (current 121 cases)
cargo test --tests

# Pre-push (already exists in cli/justfile)
just fmt && just lint && just check && just test
```

**Add `cargo-nextest`** to dev tooling — CLAUDE.md/.claude/rules/testing.md reference it but `cargo nextest run` errors with `no such command` locally. Either add to mise/`.tool-versions` or install via `cargo install cargo-nextest --locked`. Wall-clock improvement for CLI integration tests is significant (parallel binary execution).

### 5.4 Dashboard (TypeScript)

```bash
# Sentinel — NEW: just the contract-validation suite, no MSW boot, <2s
npx vitest run src/lib/api/__tests__/contract-validation.test.ts

# Fast — full vitest (currently 11.91s)
npm test

# Integration — Playwright critical path (already exists)
npm run test:e2e:critical

# Full E2E (CI only)
npm run test:e2e
```

**Vitest projects** — `dashboard/vitest.config.ts:1` doesn't use the `projects` feature today. If we ever need a "fast subset" beyond contract-validation, the right move is `vitest --project=fast` (a Vitest 4 feature) rather than path globs.

### 5.5 Integration-tests

Current: `integration-tests/tests/*.test.ts` 4 files, 223 cases. Already a single tier. Don't subdivide — it's a low-frequency suite (full-stack), wall-clock is dominated by CLI build + API boot. **Mark as "integration" tier; don't run in agent inner loop unless touching cross-component plumbing.**

### 5.6 Tags / file conventions summary

| Component | Sentinel | Fast | Integration | Slow / E2E |
|---|---|---|---|---|
| api | `test/sentinels/*_sentinel_test.rb` | `test/{models,services,policies,serializers,smoke}/` | `test/{controllers,integration,jobs,mailers}/` | nightly |
| cli | `tests/*_sentinel.rs` (proposed) | `cargo test --lib` | `cargo test --tests` | (none yet) |
| dashboard | `src/lib/api/__tests__/contract-validation.test.ts` | `vitest run` | `playwright test --grep @critical` | `playwright test` (full) |

---

## 6. Selective-testing recommendation

**TL;DR — don't build a Bazel/Nx affected-test pipeline yet.** Component-level path filters in GitHub Actions are doing 80% of the job already. The remaining 20% (within-component selection inside `api/`) is worth solving, but cheaply.

### 6.1 What exists

- **GitHub Actions `paths:` filters** at workflow level. Touching `cli/**` runs only `cli-ci.yml` and `integration-tests.yml`. Same for `api/**` and `dashboard/**`.
- **No build graph** otherwise: `cli/` and `orchestrator/` are independent crates, not a Cargo workspace. `dashboard/` and `integration-tests/` are independent npm projects, not workspaces.

### 6.2 What's missing

Within `api/`, every PR that touches any Ruby file runs the full ~3,924-test suite. Measured locally at 85.3s serial; CI runs in ~5–10 min including bundle install + DB setup. For an agent making focused service edits, that's overkill.

### 6.3 Recommendation: minitest's `--name` plus a tiny dispatch script

Don't add a tool. Use minitest's existing `-n` / `--name` filter and write a `bin/test-affected` script that:

1. Reads `git diff --name-only origin/main...HEAD` for `api/app/...`.
2. Maps each path with a hand-written rule table:
   - `app/services/<x>_service.rb` → `test/services/<x>_service_test.rb`
   - `app/models/<x>.rb` → `test/models/<x>_test.rb`
   - `app/controllers/api/v1/<x>_controller.rb` → `test/controllers/api/v1/<x>_controller_test.rb`
   - `app/policies/<x>_policy.rb` → `test/policies/<x>_policy_test.rb`
   - Anything else → fallthrough to "run sentinels + smoke."

This is ~30 lines of Ruby. It catches the 70% case (1:1 mapping). For dependency-graph-aware selection (touching `Organization` model should run anything that uses `belongs_to :organization`), don't bother: the Rails community has tried this with `m`, `tested_files`, etc., and the brittleness usually loses.

### 6.4 If selection ever genuinely needs to scale

Migration path that doesn't require a big-bang rewrite:

| Step | Effort | What you get |
|---|---|---|
| 1. `bin/test-affected` script in `api/bin/` | half day | 70% selection, no new dep |
| 2. Wire `bin/test-affected` to a `PostToolUse` hook | hours | agents auto-run the right slice |
| 3. CI: `bin/rails test test/sentinels test/smoke` first, fail fast | hours | sub-30s gate before the big job |
| 4. (Only if 1–3 prove insufficient) `pants` or a custom AST-walk to derive transitive deps | weeks | true affected-test selection |

Do **not** introduce nx, turborepo, or pants until you have measured pain. The current dashboard wall-clock (12s) and CLI wall-clock (7s) do not justify selection tooling; only `api/` does, and that's solvable with the script above.

---

## 7. Agent-loop integration

### 7.1 Proposed `AGENTS.md` testing section

Place at repo root as `AGENTS.md`. (None exists today.) Content below is the actual proposed text — the project doesn't have one and OpenCode + other agents look for it specifically.

```markdown
# Testing for AI agents

Every edit you make should end with a green test signal proportional to the
change. The default is: **run the sentinel suite plus the affected fast tier**.
Do NOT run the full suite for a one-file change.

## After editing api/app/...

```bash
cd api
PARALLEL_WORKERS=1 bin/rails test test/sentinels/      # ~30s, must pass
PARALLEL_WORKERS=1 bin/rails test <one or more files>  # 1:1 map below
```

Mapping (1:1 rule of thumb):
- `app/services/<x>_service.rb` → `test/services/<x>_service_test.rb`
- `app/models/<x>.rb` → `test/models/<x>_test.rb`
- `app/controllers/api/v1/<x>_controller.rb` → `test/controllers/api/v1/<x>_controller_test.rb`
- `app/policies/<x>_policy.rb` → `test/policies/<x>_policy_test.rb`

If the change is to a base class, a concern, or `BaseController` → run the
controller tier: `PARALLEL_WORKERS=1 bin/rails test test/controllers test/integration`.

**macOS local quirk:** the parallel runner segfaults on `pg`. Always export
`PARALLEL_WORKERS=1` locally. CI is already pinned.

## After editing cli/src/...

```bash
cd cli
just lint && just check
cargo test --lib                    # ~7s
# If the change touches services/event_sanitizer.rs or hooks/:
cargo test --test sanitizer_sentinel   # (once added)
```

Pre-push: `just fmt && just lint && just check && just test`.

## After editing dashboard/src/...

```bash
cd dashboard
npm test                                                  # ~12s, must pass
npm run validate:api-contract                             # if you touched src/lib/api or src/types
# If the change touches a route under app/kit/:
npm run test:e2e:critical                                 # ~5min, before pushing
```

## Cross-cutting changes

If your change spans api + dashboard (e.g., a new endpoint), run:

```bash
cd integration-tests && npm test    # 223 cases, requires API + CLI running
```

## Don't

- Don't `bin/rails test` (full suite) without `PARALLEL_WORKERS=1` on macOS.
- Don't skip tests with `skip "TODO"`. There are only 2 skips in the entire
  repo today (rack-attack-localhost, owner-permission). Keep it that way.
- Don't add fixtures with real PII. All fixtures in `api/test/fixtures/` are
  synthetic.
- Don't disable contract validation just to get a green CI. The fix is the
  spec or the type, not the gate.
```

### 7.2 Proposed Claude Code hooks (`.claude/settings.json`)

The current `.claude/settings.json` only enables a plugin. Add a `PostToolUse` hook that runs the cheapest possible verification on the edited file. Proposed snippet:

```json
{
  "enabledPlugins": { "crewkit-lsp@crewkit-local": true },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/scripts/post-edit-check.sh \"$CLAUDE_FILE_PATHS\""
          }
        ]
      }
    ]
  }
}
```

Where `.claude/scripts/post-edit-check.sh` is a 30-line dispatcher:

```bash
#!/usr/bin/env bash
# Runs the cheapest verification for the edited file. Silent on success,
# prints diagnostics on failure. Capped at 30s wall-clock.
set -euo pipefail
for f in "$@"; do
  case "$f" in
    api/app/services/*.rb|api/app/models/*.rb|api/app/policies/*.rb|api/app/serializers/*.rb)
      base=$(basename "$f" .rb)
      dir=$(echo "$f" | sed 's|api/app|api/test|; s|\.rb$|_test.rb|')
      [ -f "$dir" ] && (cd api && PARALLEL_WORKERS=1 timeout 30 bin/rails test "$dir" 2>&1 | tail -5)
      ;;
    cli/src/**)
      (cd cli && timeout 30 cargo check 2>&1 | tail -5)
      ;;
    dashboard/src/**)
      (cd dashboard && timeout 30 npx tsc --noEmit 2>&1 | tail -10)
      ;;
  esac
done
```

(Don't run the test on every keystroke. The hook runs after `Edit/Write/MultiEdit` — meaning once per logical change.)

### 7.3 Subagent definitions worth creating

The repo already has 18+ subagents in `.claude/agents/`. The gap for testing is **a sentinel-runner / fast-tier dispatcher** that knows the file-to-test mapping above.

Proposed: `.claude/agents/test-runner.md` — a small subagent invoked when an implementation agent wants a verification sweep. It reads `git diff`, runs the appropriate tier (sentinel + fast affected), reports pass/fail with the failing assertion. This is purely a thin layer over `bin/test-affected` (Section 6.3) and the post-edit hook.

**Don't create new code-writing subagents for tests.** rails-expert, cli-expert, frontend-expert already cover that. The missing piece is dispatch logic, not authorial expertise.

### 7.4 Reference: Stripe Minions / Roast / SpaceX patterns considered

The brief invoked these as evaluation references. Findings:

- **Stripe Minions** (validation-first, types-as-spec, blueprint orchestration): Good fit for the **OpenAPI ↔ generated types ↔ MSW handlers** flow already in place. The actionable lift is making contract validation **blocking** (Section 4.8). Adopting Minions wholesale (a separate runtime, blueprints) is overkill for crewkit's current size.
- **Shopify Roast** (deterministic + LLM-judged validation, production-sampled eval sets): Relevant for crewkit's *own* product (resource performance, agent quality) — not for *crewkit's* test strategy. Don't conflate. Phase D in `PLAN.md` already plans LLM-as-judge for **production data**, which is correct.
- **SpaceX HITL** (high-fidelity simulators in CI): The crewkit equivalent is the `integration-tests/` directory (real CLI + real Rails + real Postgres). Already in place; the gap is sub-second feedback for the inner loop, not higher fidelity.
- **Bazel/Pants/Nx**: Section 6 covers this — premature for crewkit's size. Revisit when full-suite local wall-clock crosses 5 minutes per component.

---

## 8. Risk register

Top 5 things most likely to go wrong as agent usage scales, ranked by probability × impact:

| # | Risk | Likely trigger | Mitigation |
|---|---|---|---|
| 1 | **Tenant isolation regression** | Agent writes a controller bypassing BaseController, or a Pundit Scope without `organization_scoped`. | Sentinel 4.1; review process flags any new controller class that doesn't transitively inherit `BaseController`. |
| 2 | **Contract drift API ↔ dashboard** | Agent renames a Rails endpoint without updating `swagger/v1/openapi.yaml`. Vitest passes (MSW uses hand-written handlers); Playwright catches it 5–10 minutes later in CI; production sees it if Playwright doesn't cover that page. | Flip `validate:api-contract` and `openapi:validate STRICT=1` to **blocking**; remove `continue-on-error: true` from `e2e-tests.yml:174`. |
| 3 | **Webhook idempotency or signature regression** | Agent removes the `rescue ActiveRecord::RecordNotUnique` in `TokenService.refill!` or weakens `StripeService.verify_webhook`. Double-credit user funds; or worse, accept unsigned webhooks. | Sentinel 4.3 (idempotency under double-deliver) and 4.4 (signature). |
| 4 | **Privacy regression in CLI sanitizer** | Agent adds a new tool to `cli/src/services/event_sanitizer.rs` and forgets the strip rules; secrets leak into telemetry. | Sentinel 4.7 (proptest over the tool catalog). |
| 5 | **Slow inner loop → agents stop running tests at all** | Rails segfaults or 5-minute serial runs train agents to skip the verification step. | (a) Sentinel suite + post-edit hook (Section 7.2); (b) document `PARALLEL_WORKERS=1` in `AGENTS.md`; (c) measure: track CI runs where tests didn't run locally first. |

---

## 9. Phased rollout

Each item: concrete, has a definition of done, no more than 5 per phase.

### Week 1 (high-leverage CI flips + sentinel skeleton)

1. **Flip contract validation to blocking.**
   - Edit `e2e-tests.yml`: remove `continue-on-error: true` from `contract-validation` job.
   - Edit `api-ci.yml`: pass `STRICT=1` to the openapi:validate step.
   - **DoD:** a PR that drifts the spec or types fails CI before merge. Verified by intentionally desyncing one endpoint name in a draft PR.

2. **Create `api/test/sentinels/` directory and the first 3 sentinels.**
   - 4.1 (tenant isolation), 4.2 (JWT scope), 4.4 (Stripe signature).
   - **DoD:** `bin/rails test test/sentinels/` passes, runs in ≤30s with `PARALLEL_WORKERS=1`.

3. **Write `AGENTS.md` at repo root** with the content from Section 7.1.
   - **DoD:** new agents starting a session find it; OpenCode + Claude Code both read it.

4. **Document the macOS pg parallel-test crash in `.claude/rules/testing.md`** with the `PARALLEL_WORKERS=1` workaround.
   - **DoD:** an agent reading the rule does not run unscoped `bin/rails test` on macOS.

5. **Install `cargo-nextest` toolchain pin** (add to `cli/rust-toolchain.toml` or `cli/Cargo.toml` dev-dependency, or document `cargo install cargo-nextest --locked` in `cli/justfile`'s `setup` target).
   - **DoD:** `just test-nextest` works on a fresh checkout.

### Month 1 (sentinel completion + inner loop)

1. **Complete the sentinel catalog (4.3, 4.5, 4.6, 4.7, 4.9, 4.10).**
   - **DoD:** `test/sentinels/` has 8 files covering Section 4.3–4.10; total wall-clock under 60s; one CI job runs sentinels first and short-circuits if any fail.

2. **Build `api/bin/test-affected`** (Section 6.3) and wire it to the `PostToolUse` hook.
   - **DoD:** an Edit to `app/services/token_service.rb` runs `test/services/token_service_test.rb` automatically and reports back to the agent.

3. **Add `cli/tests/sanitizer_sentinel.rs`** with proptest coverage of every tool in `SAFE_RESPONSE_TOOLS` and the strip table.
   - **DoD:** removing a strip rule for any tool fails the sentinel test in <5s.

4. **Audit Pundit Scopes for `organization_scoped` adoption.** Output: a short report (~1 page) noting any Scope subclass that resolves a relation without org filtering, and whether it's intentional (admin scope, public route).
   - **DoD:** one PR fixes the unintentional gaps; sentinel 4.1 includes them in its registry.

5. **Add `.claude/agents/test-runner.md`** (Section 7.3).
   - **DoD:** rails-expert, cli-expert, frontend-expert can SendMessage to it for verification.

### Quarter 1 (measure, then maybe scale)

1. **Track inner-loop adherence.** Add a CI step that flags PRs whose first commit's CI failure could have been caught by sentinel + fast-tier locally. Surface in PR comment.
   - **DoD:** monthly metric: % of CI failures that local sentinel run would have caught.

2. **Decide on within-`api/` affected-test selection.** Re-evaluate the `bin/test-affected` script. If 70% selection is enough, stop. If not, evaluate `m` gem or a custom AST-walk; do not default to a heavyweight tool.
   - **DoD:** a written decision in `docs/testing-strategy-research.md` (this file) updating Section 6.

3. **Test-quality evaluator (Phase D from PLAN.md) — production-data LLM judge.** Out of scope for inner-loop, in scope for product. Track separately.
   - **DoD:** Phase D ships in PLAN.md; not addressed here.

4. **Move `system/` tests out** if still scaffold-only. Two files in `api/test/system/` haven't been touched since Feb 2026; UI moved to dashboard. Delete or note.
   - **DoD:** directory removed from default `bin/rails test` invocation, or test files removed.

5. **Quarterly sentinel review.** Each new high-stakes feature (cloud agent identity in Phase B; vectorization in Phase C) should add or extend a sentinel as part of its definition of done.
   - **DoD:** Phase B PRs include a sentinel diff alongside the feature diff.

---

## 10. Open questions

Things I could not determine from the code alone in one pass; flag for a human:

1. **Is the macOS pg parallel segfault a known upstream issue, or specific to this checkout?** I reproduced it cleanly today (8 workers, fresh DB). The CI workaround (`PARALLEL_WORKERS=1`) is documented but I didn't find an open upstream issue link in `Gemfile`, `CHANGELOG`, or the test_helper comment. Worth filing if not already.

2. **Does `stripe_webhook_service_test.rb` (781 lines) already cover the double-deliver idempotency case?** I did not read all 781 lines. Section 4.3 assumes it does *not*; verify before adding a duplicate sentinel.

3. **What's the actual CI wall-clock budget today?** I have anchored sentinel/fast targets to local measurements. The brief's "sub-30s inner loop where possible" is ambitious for the API serial run. Confirm what CI actually runs in (look at last 10 PRs).

4. **Is the `api/system/` directory actively used?** Two files, last touched Feb 2026, before the UI moved out. Likely scaffolding. Confirm before deleting.

5. **Does `BaseController` have any subclasses that should NOT enforce tenant scoping (legitimate exceptions)?** I found `health_controller.rb` and `webhooks/stripe_controller.rb` as obvious exceptions (no auth needed). Are there others I should not flag in sentinel 4.1?

6. **Should the sentinel directory be `api/test/sentinels/` or `api/test/integration/sentinels/`?** I propose top-level `sentinels/` for visibility. Confirm with team taste.

7. **Where does the test-quality LLM judge (PLAN.md Phase D) read its eval set from?** Not relevant for the inner-loop strategy here, but the answer affects whether sentinel coverage and judge coverage should overlap.

8. **The `cli_auth.rs` integration target showed `1 target failed` on the first `cargo test` invocation, then passed cleanly on the second.** Possible flake under cold cache. Worth tracking as a known risk; not blocking.

9. **Can we reach Stripe webhook signature secret in test env?** Sentinel 4.4 needs to construct an unsigned payload — feasible — and a wrongly signed one — needs `STRIPE_WEBHOOK_SECRET` set in test. Confirm test env variable is wired.

10. **Are there agent-edit metrics already collected in production telemetry that would tell us where the actual breakage hot-spots are?** Crewkit literally observes Claude Code sessions. Mining its own data for "files agents edit that subsequently fail CI" would let the next iteration of this strategy be data-driven rather than heuristic.

---

*End of original report — Sections 11+ added 2026-05-06 after parallel research on Shopify, Stripe, SpaceX, Tesla.*

---

## 11. Industry research addendum (Shopify / Stripe / SpaceX / Tesla)

Three teammates ran in parallel against primary sources. Full reports preserved at `/tmp/research-shopify.md`, `/tmp/research-stripe.md`, `/tmp/research-spacex-tesla.md`. This section is the synthesis: where the four companies converge, what's new for crewkit, what doesn't transfer.

### 11.1 Strong cross-company convergences

When four very different shops independently arrive at the same pattern, that's the signal.

1. **Production-sampled eval sets > synthetic benchmarks.** Three of four explicitly:
   - Shopify Sidekick uses "Ground Truth Sets (GTX)" sampled from real merchant conversations; the Flow team built 300 hand-crafted offline cases that reached parity in eval and **collapsed in production**. ([shopify.engineering, Aug 2025](https://shopify.engineering/building-production-ready-agentic-systems))
   - Stripe's published agent benchmark drives a real terminal + browser + Stripe API, then grades by inspecting Checkout Sessions in the API — not by `assert response.status == 200`. ([Stripe blog, Mar 2026](https://stripe.com/blog/can-ai-agents-build-real-stripe-integrations))
   - Tesla's eval set is ~6,000 clips across ~70 scenario categories, mined from fleet data; Karpathy's "data engine" reframes the whole methodology as iterated production sampling. ([CVPR 2021 transcript](https://pharath.github.io/self%20driving/Karpathy-CVPR-2021/))
   **Implication for crewkit:** crewkit is uniquely well-positioned because LlmSession captures real sessions in production. The eval set should be sampled from there, not hand-curated. This is the single strongest signal across all four reports.

2. **Cheap deterministic gates run *before* expensive LLM/judge gates.** Shopify Sidekick's "N-Stage Gated Rewards" pushed syntax pass-rate 93% → 99% with procedural checks before semantic eval fires. Stripe Minions alternates "deterministic nodes (parse, lint, test, schema-validate)" with "agentic nodes (plan, generate, interpret)." SpaceX won't even build if static analysis fails. **Implication:** crewkit's sentinel ring (Section 4) is exactly this pattern — formalize that sentinels run first, fail fast, before any LLM-judge ever fires.

3. **Hard caps on agent-thrash loops.** Stripe's Minions cap CI retries at **two** before handing off to a human — explicit guardrail against "the agent burns budget chasing CI green." ([bytebytego recap](https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs)) **Implication:** crewkit's `BlueprintTask` / `CloudJob` should encode a `max_ci_rounds` policy. The original report missed this entirely; it's a one-line schema change with high leverage.

4. **Reversion rate / behavioral diff as the north-star metric — not test count or coverage.** Shopify: Farhan Thawar publicly states the success criterion is "PR volume up, reversion rate flat." Tesla: ship gate is "net-positive on safety metrics over N million shadow miles," not "no test fails." **Implication:** crewkit should add a `reverted_at` / `revert_of_pr` signal to its session telemetry and dashboard the agent-authored revert rate — independent of whether sentinels passed.

5. **Static analysis as a *hard*, *unoverrideable* gate.** SpaceX: "the code will not build unless it passes the static-analysis step" ([LWN, 2013](https://lwn.net/Articles/540368/)). Stripe and Shopify both say the same in less dramatic terms. crewkit already does this for rubocop/brakeman/clippy; the gap is the **OpenAPI contract** gates that today are `continue-on-error: true` (Section 3.2). The brief's existing recommendation to flip those gates is the exact pattern SpaceX would recognize.

### 11.2 New patterns for crewkit (not in the original report)

The original report covered tiering, sentinels, hooks, and selective testing. Three patterns from the industry research are genuinely additive:

**A. Shadow mode for resource-version changes (Tesla → crewkit, HIGH transferability).**
When an org publishes a new version of an agent / playbook / skill, run the new version against the last N completed sessions for that resource and diff outputs. Don't merge — observe. Surface in the dashboard's `ResourceVersionComparison` flow. crewkit *literally already has the data substrate*: `LlmSession` captures real prompts + responses + tool calls + outcomes. This is Tesla shadow-mode, exactly translated. The infrastructure cost is mostly UI; the model-replay machinery already exists for the AnthropicService batch path.

**B. Trigger classifiers over `SessionEvent` (Tesla data-engine + Shopify failure-mode catalog, HIGH).**
Shopify documented three concrete reward-hacking failure modes (opt-out, tag-hacking, schema-violations); Tesla runs ~221 hand-defined trigger classifiers over fleet telemetry. crewkit equivalents — cheap regex / AST / structural checks over `SessionEvent`:

- Edits-without-tests (PreToolUse `Write|Edit` to `app/` with no later `Write|Edit` to `test/`)
- Cross-tenant query without `organization_id` (Bash `psql` or `Rails.console` runs containing "WHERE" without "organization_id")
- Secret-shaped strings reaching the API (`AKIA...`, `sk-ant-api03-...`, JWT shape)
- `acts_as_paranoid` bypass (`unscoped`, `with_deleted`, `delete_all` on a paranoid model)
- Billing-table writes from non-billing services (writes to `token_balances`, `token_transactions`, `invoices` from a controller/service that isn't in the billing namespace)

These run as background classifiers on already-captured `SessionEvent` data, surface as warnings on the conversation detail page, and double as the seed for a future eval set. Cheap to start, compounding value.

**C. Net-positive ship gate, with a fixed safety-critical core (Tesla + Waymo, HIGH).**
For changes that affect *prompts* (not source code) — an agent template, a playbook, a system message — accept that some eval cases will flip both ways. Define a small **safety-critical core** that must always pass (the sentinels in Section 4) and a **broader eval suite** where net-positive is the ship gate. Mirrors the Tesla pattern of "zero-regression on AEB precision/recall + improvement on the broader suite." This is the answer to the open question "what about prompt-equivalent changes" — those don't fit a binary pass/fail tier.

### 11.3 Patterns that don't transfer (honest non-fits)

- **SpaceX hardware-in-the-loop / "table rocket".** No SaaS analog beyond a staging environment. Don't force-fit; staging already exists.
- **Stripe-scale 3M-test selective CI / "Leverage" platform team.** crewkit's full suite is ~7K tests in ~85s; selection ROI doesn't exist yet. This was already in Section 6; the Stripe research confirms it.
- **Shopify's GTX labeling pipeline (3+ product experts per conversation).** Heavyweight. Reversion rate is cheap and immediate; ship that signal first.
- **Triple-redundant runtime voting (SpaceX) and "1,300 PRs/week" throughput (Stripe).** Both depend on scale crewkit doesn't have. Optimize review throughput before agent throughput.
- **Tesla's 15M-mile pre-ship validation timescale.** SaaS PRs ship in hours; the timescale gap kills the pattern.
- **Custom in-house agent framework (Stripe Minions, Shopify Roast).** Both companies built their own because their stack is unusual. crewkit should use Claude Code + hooks + skills, *not* build a framework. Patterns transfer; the gem/runtime doesn't.

### 11.4 Updates to the phased rollout (Section 9)

Two adds, one re-prioritization. No removals.

**Add to Week 1:**
- **`max_ci_rounds = 2` policy on `CloudJob` / `BlueprintTask`.** Schema change + cloud-agent runner respect. DoD: a cloud job that fails CI twice halts and notifies the human owner instead of a third retry. Source: Stripe Minions.

**Add to Month 1:**
- **Reversion-rate telemetry.** Tag PRs `agent_authored` (CLI knows this); add `reverted_at` / `revert_of_pr` to existing PR/session linkage; dashboard widget showing 7/30-day revert rate vs. human-authored baseline. DoD: one number on the analytics page that answers "are agents getting better or worse this month?" Source: Shopify Thawar interview.

**Re-prioritize for Quarter 1:**
- **Trigger classifiers over `SessionEvent`** is now a Q1 priority, not nice-to-have. The five classifiers in §11.2.B cover the same risk surface as the current sentinel catalog (Section 4) but operate on *already-captured* production data instead of synthetic test cases. They double as the seed for a production-sampled eval set.

**Hold for later (do not add to roadmap yet):**
- Shadow mode UI / `ResourceVersionComparison` replay — high transferability but requires the trigger classifiers and the eval-set substrate first. Phase D-adjacent.
- LLM-judge calibration vs human Cohen's Kappa (Shopify Sidekick) — only worth the cost once production-sampled eval sets exist. Don't build judges before evals.

### 11.5 One-line takeaway

The four companies converge on: **production-sampled eval sets, deterministic gates before LLM gates, reversion rate as the north star, hard caps on agent thrash, static analysis with no overrides.** crewkit is uniquely positioned to execute on the first two because it already captures real sessions. Build the trigger classifiers and the reversion-rate widget before the LLM judges; the order matters.

*End of report.*
