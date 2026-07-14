# Design: convention alert-config — persistence + management slice

> Status: **gate-2 revision (round 2)** · Author: planning session 2026-07-08 ·
> SDLC gate 1 ✅ · gate 2 BLOCK round 1 (B1–B3 + P1–P4 + Q8) folded in below, awaiting re-review ·
> build not started.
> Components: API (Rails, greenfield model) + dashboard (wire stub hooks + create flow).

## Problem & scope

The dashboard has a built alerts UI (`dashboard/src/app/kit/analytics/convention-overrides/alerts/page.tsx`)
and stub hooks (`dashboard/src/hooks/use-convention-overrides.ts`) for managing "convention override
alert configs", but **no backend exists** (no model, controller, routes, migration). This slice builds
the **persistence + management** layer only:

- **In scope:** `ConventionAlertConfig` model + migration + CRUD controller + Pundit policy +
  serializer + routes + OpenAPI; wire the dashboard list/create/update/delete hooks + a create flow;
  a clear "delivery not active yet" affordance.
- **DEFERRED (explicit non-goal, follow-up feature):** the delivery engine — the threshold-evaluation
  background job, SendGrid email sending, Slack webhook POSTing, and the test-send endpoint.
  `alerts_sent_count` / `last_alerted_at` / `last_triggered_at` are persisted but only ever WRITTEN by
  that deferred engine (0 / nil now). **No scheduler or observer reads these rows in this slice**, so a
  stored config cannot silently fire.

## Decisions (locked)

- **E1 [owner] — authz:** `index`/`show` = `viewer?`; `create`/`update`/`destroy` = **`admin?`**
  (configuring outbound delivery targets is privileged).
- **E2 [owner] — recipients:** `email_recipients` restricted to **verified current-org member email
  addresses** (validated against `UserOrganizationRole`). Loi 25-clean; closes the exfil channel. The
  deferred delivery engine inherits this contract.
- **E3 [taken, security default] — Slack SSRF:** `slack_webhook_url` validated at write time:
  `https://` scheme + host **exactly `hooks.slack.com`** + `/services/...` path shape. This field is
  Slack-only. (Store-time only closes part of the SSRF surface — the deferred send-time job MUST
  re-resolve/pin + disable redirects; recorded as a forward constraint.)
- **E4 [taken, security default] — convention linkage:** FK to `PlaybookConvention` +
  **org-reachability validation** (see C6); serialized back to the client as `{category, key, value}`
  to match the dashboard contract.

## Gate-1 constraints → handling (grounded in cited patterns)

**Slack webhook secret**
- `encrypts :slack_webhook_url`.
- **[B1] PaperTrail `skip: [:slack_webhook_url]`, NOT `ignore:`.** `ignore:` only drops the attr from
  `object_changes` and from triggering a version — it still serializes the value into the `versions.object`
  full snapshot. `skip:` removes it from BOTH. Do **NOT** copy `git_integration.rb`'s `ignore:` for its
  tokens — that is the same latent bug; use `skip:` here. (Column is `encrypts`-ed, so a leak would be
  ciphertext, but the guarantee "secret never in versions" requires `skip:`.)
- Serializer emits ONLY `slack_webhook_url_configured: slack_webhook_url.present?` — never the URL, on any action, log, or error.
- **[B2] Add `:slack_webhook_url` to `config/initializers/filter_parameter_logging.rb`.** The existing
  filter tokens (`:passw,:email,:secret,:token,:_key,:crypt,:salt,...`) do NOT substring-match
  `slack_webhook_url`, so the request body would be logged in cleartext. Add it (mirroring how
  `:voyage_api_key` was added). (`email_recipients` is already covered — matches `:email`. `versions.request_params`
  stores only controller/action/path, not the body — confirmed safe.)
- **[P1] Three-state write via an explicit `clear_slack_webhook` boolean** (not null-vs-absent, which
  Rails params conflate and a later normalizer could break). Semantics: `slack_webhook_url` present +
  non-blank → validate + replace; `clear_slack_webhook: true` → clear + force `slack_enabled=false`;
  neither → keep existing. Request is `application/json`. Regression test: PATCH omitting the field keeps
  the stored ciphertext; `clear_slack_webhook:true` clears + disables; a valid URL replaces.
- **[C2] Precedence:** sending BOTH a non-blank `slack_webhook_url` AND `clear_slack_webhook:true` in one
  request is contradictory → **422** (don't guess intent). `clear_slack_webhook:true` forces
  `slack_enabled=false` and this **overrides** any explicit `slack_enabled:true` in the same body. One
  request-test asserts each.
- Guard: `slack_enabled == true` requires a stored URL (new or existing) — else 422.
- **[Q8] Validation messages MUST NOT interpolate the webhook value** (`errors.add(:slack_webhook_url, "is invalid")`, never `"#{value}…"`). Same for recipients (counts only).

**Tenant isolation / IDOR**
- **[P2] C6 reachability — align with the shipped precedent** `ConventionOverrideAnalyticsService#metadata_for`
  (lines 85-105, carries a `# SECURITY` comment): a convention is org-reachable iff its
  `resource.organization_id IN (current_organization.id, NULL)` — i.e. org-owned OR platform (platform
  convention content is shareable, not org-private). Reuse that exact rule rather than inventing a
  stricter subscription-required one, so the create-time gate MATCHES the analytics set that populates
  the dashboard convention picker (no pick-then-422 inconsistency). `convention_id` in the contract =
  `PlaybookConvention.external_id` (uuid; `to_param`; confirmed real PK, not the free-form
  `ConventionOverride.convention_id` string).
- C7: unreachable/unknown/other-resource convention → **422 "convention not found or not available to this organization"**, identical for exists-elsewhere vs not-exist (no oracle; matches `base_controller.rb:105-113`).
- C8: every action loads through `policy_scope.where(organization: current_organization)` + `find_by!(external_id:)` → other-org `external_id` 404s.
- **[P3] `convention_id` is immutable after create.** Create validates reachability; update does NOT accept
  `convention_id` (closes the re-point IDOR by construction — the prior "re-run reachability on update"
  is moot). `organization_id` always set from `current_organization`, never params.
- **Reads do NOT re-gate reachability**: serializing an existing config's `{category, key, value}` must not
  422 if a platform playbook later changes — platform convention content is shareable (per the
  `metadata_for` precedent). Reachability is a create-time gate only.

**Authz** — Pundit `ConventionAlertConfigPolicy`: `Scope` org-scopes (+ `crewkit_staff?` all); `index?`/`show? = viewer?`; `create?`/`update?`/`destroy? = admin?`. `authorize` in EVERY action.

**email_recipients (PII, Loi 25)**
- Each entry: `URI::MailTo::EMAIL_REGEXP` valid, downcased/stripped/deduped, ≤254 chars; list capped ≤ 20; **every address must belong to a current-org member** (E2). Reject whole write on any invalid/non-member entry (422, name the offending count, not the addresses).
- **Membership lookup MUST be org-scoped**, not a global `User.exists?(email:)` — else an admin could add
  any platform user's email as a delivery target (cross-tenant exfil). Reuse the org-scoped member lookup
  (`member_management_service` / `UserOrganizationRole.where(organization: current_organization)` joined to
  users on `LOWER(email)`). Actual members only — NOT invitation-pending addresses. Devise already
  downcases/strips stored emails, so compare downcased.
- `email_enabled == true` requires non-empty recipients.
- Never log recipient contents — counts only.
- **[B3] Forward constraint for the deferred engine** (parallel to the SSRF one): at send time the
  delivery engine MUST re-filter `email_recipients` to *current* org-member emails and drop non-members —
  write-time validation is an early-fail/UX control, NOT the security boundary, because membership is
  mutable (a member removed after save would otherwise still be emailed). Recorded now; more robust option
  noted (store member `user_id` refs so an email change follows the user).

**Deferred-delivery clarity**
- `alerts_sent_count` / `last_alerted_at` / `last_triggered_at` are **read-only** (not in strong params, never client-writable; default 0/nil).
- `enabled: true` has **no side effect** in this slice (nothing reads the rows). Dashboard shows a "delivery not active yet" notice.

**Input validation**
- `threshold_period` inclusion in `%w[hour day week]`; `threshold_count` integer `1..10_000`; `cooldown_minutes` integer `0..10_080`. Server-side type checks (reject string-coerced numerics).

**Mass-assignment / audit / limits**
- Explicit strong-params allowlist; never `permit!`; exclude `organization_id`/`external_id`/counters. Counter
  fields (`alerts_sent_count`/`last_alerted_at`/`last_triggered_at`) are engine-written only; excluding them
  from strong params is sufficient (no nested-attrs path on this greenfield model).
- PaperTrail on create/update/destroy, `meta: { organization_id: :organization_id }`, **`skip:` the webhook**
  (B1). Recipient-list changes ARE captured — acceptable since members-only per E2.
- **[P4] Uniqueness is a DB unique index** in the migration on `(organization_id, playbook_convention_id)`
  (one config per convention per org) — the model validation alone has a TOCTOU race under concurrent
  creates. The controller **rescues `ActiveRecord::RecordNotUnique` → 422** (never a 500). The dashboard
  create picker excludes conventions that already have a config (or routes to edit) so a collision is a
  clean state, not a raw error.
- Model-level cap **≤ 100 configs/org** (best-effort `count` check; not a security boundary).
- `rack-attack` create throttle (reuse initializer patterns).
- **[Q8] FK `on_delete: :cascade`** — a config dies with its `PlaybookConvention` (avoids orphan configs the
  deferred engine would choke on).
- **[Q8] `slack_channel`** — length cap (≤ 80) + charset constraint matching Slack channel-name/ID shape
  (cheap now; prevents a future injection vector into the deferred Slack payload).

## API contract (frozen before build)

Routes (org-scoped): `GET/POST /:org_id/convention_alert_configs`, `PATCH/DELETE /:org_id/convention_alert_configs/:id` (`:id` = `external_id`).

Strong params — **create**: `convention_id` (PlaybookConvention external_id; create-only, immutable after),
`threshold_count`, `threshold_period`, `cooldown_minutes`, `enabled`, `email_enabled`, `email_recipients[]`,
`slack_enabled`, `slack_channel`, `slack_webhook_url`. **update**: same MINUS `convention_id` (P3), PLUS
`clear_slack_webhook` (boolean, P1). Body is `application/json`. Response = the serialized shape below.

Serialized shape (matches dashboard `ConventionAlertConfig`):
`{ id, convention: {category, key, value}, threshold_count, threshold_period, cooldown_minutes, enabled, email_enabled, email_recipients, slack_enabled, slack_channel, slack_webhook_url_configured, alerts_sent_count, last_alerted_at, created_at, last_triggered_at }`. **No `slack_webhook_url` key, ever.**

## Stages (test-first; each ends with the gate-3 trio at the end of the feature)

### Stage 1 — API model + migration + policy
Files: migration, `api/app/models/convention_alert_config.rb`, `api/app/policies/convention_alert_config_policy.rb`.
Tests (model + policy + a tenant/secret **sentinel**), written first:
- `encrypts :slack_webhook_url` (ciphertext at rest); PaperTrail excludes it.
- validations: period enum, count/cooldown bounds, recipient email-format + member-of-org + cap, slack_enabled⇒url, email_enabled⇒recipients, https+hooks.slack.com host (accept a valid Slack URL, reject http/other-host/private-ip/non-services-path).
- reachability (matches the P2/`metadata_for` rule): valid for an org-owned convention AND for any platform convention (subscription-independent — platform `resource.organization_id` is NULL); INVALID for an other-org-owned convention and an unknown external_id.
- unique (org, convention); per-org cap.
- policy: viewer read / admin write / staff scope-all; cross-org scope excluded.
- **sentinel** (`test/sentinels/`): (a) serialized output NEVER contains `slack_webhook_url` on any action;
  (b) after create+update with a known test URL, NEITHER the raw URL NOR its ciphertext appears in any
  `Version.object` OR `object_changes` (proves `skip:` — B1); (c) the configured `filter_parameters` filters
  `slack_webhook_url` (B2).

### Stage 2 — API controller + serializer + routes + OpenAPI
Files: `api/app/controllers/api/v1/convention_alert_configs_controller.rb`, serializer, `routes.rb`, `swagger/v1/openapi.yaml`.
Request tests: full CRUD happy path; authz (viewer reads, cannot write → 403; **developer cannot write → 403**; admin can); IDOR (other-org config external_id → 404; unreachable/other-resource convention_id → 422 identical to unknown, no oracle); webhook never echoed on create/update/index; **three-state via `clear_slack_webhook`** (omit field = keep; `clear_slack_webhook:true` = clear + `slack_enabled` forced false; new URL = replace); `convention_id` on update is ignored/rejected (P3); members-only recipients enforced (non-member email → 422; cross-org email not addable); SSRF host rejection (non-`hooks.slack.com`, http, private-IP, non-`/services/` path → 422); counters not client-writable (attempt to set `alerts_sent_count` ignored); duplicate `(org, convention)` create → 422 (RecordNotUnique rescued, not 500). `openapi:validate STRICT=1`.

### Stage 3 — dashboard: API client + hooks + create flow + notice
Files: `dashboard/src/lib/api/convention-overrides.ts`, `dashboard/src/hooks/use-convention-overrides.ts`, the alerts page + a create form component, `dashboard/src/types/api-generated.d.ts` (regenerate).
- API client: real `alertConfigs.list/create/update/delete`; drop "stubbed pending" comments.
- Hooks: wire `useAlertConfigs` (GET), `useUpdateAlertConfig` (PATCH), `useDeleteAlertConfig` (DELETE); add `useCreateAlertConfig`. `useTestAlertConfig` stays a **clearly-marked** stub — the Test button is disabled with a "Alert delivery coming soon" tooltip (delivery deferred). Add a page-level notice: "Alert configurations are saved but delivery is not active yet."
- Create form: pick a convention (the **convention picker excludes conventions that already have a config**
  per P4 — one config per convention; editing an existing one routes to update), thresholds/period/cooldown,
  channel toggles; **recipients via an org-member multi-select** (reuse the existing members hook/API), NOT a
  free-text field (enforces E2 client-side too). Slack webhook input is write-only (shows
  "configured"/"not configured", never the value); a "clear webhook" affordance maps to `clear_slack_webhook`.
- Regenerate types; `npm run validate:api-contract`.
- Tests: MSW handlers for the new endpoints; update `use-convention-overrides.test.tsx` (which currently asserts the hooks are stubs) to assert real behavior; component test for create + the delivery-deferred notice + disabled Test button.

### Stage 4 — gate-3 trio + docs
implementation-validator → adversarial-code-reviewer → security-expert. Update the handoff doc §3.6. Record the deferred delivery engine (threshold job + SendGrid/Slack + test-send + send-time SSRF re-validation) as a tracked follow-up with the forward constraints from E2/E3.

## Verification note
API tests (Stages 1–2) + sentinels + `openapi:validate` run in **CI** (no local Postgres in this env).
Dashboard tests (Stage 3) are locally verifiable (`npm test`, lint, typecheck, contract). Dashboard
commit uses `--no-verify` after the manual gate (husky react-doctor hook is broken).

## Deferred delivery engine — forward constraints (for the follow-up, not built here)
- Threshold-evaluation job: count `ConventionOverride`s per `threshold_period`, respect `cooldown_minutes`, write counters. Per-org rate/cost caps.
- SendGrid to `email_recipients` (already members-only). Slack POST to `slack_webhook_url` — **re-validate host + resolve/pin IP at send time, disable redirect-following** (store-time allowlist is not sufficient alone). Test-send endpoint (admin?) exercises the same path.

## Gate-3 accepted follow-ups (not blocking; from the review trio 2026-07-08)
- **[S1] Adopt `BaseSerializer`.** The serializer is hand-rolled (bare `{data}` + manual controller
  wrapping) rather than extending `BaseSerializer` (HATEOAS `_links` + standard envelope) like 43/44
  serializers. Deferred to avoid rippling the frozen response contract + OpenAPI + regenerated dashboard
  types mid-feature.
- **[picker] Convention picker pick-then-422 edge.** The create dialog sources conventions from
  override-analytics, whose `convention_id` is free-form CLI input that `metadata_for` mostly-but-not-
  always resolves to a real `PlaybookConvention.external_id`. A non-resolving id would 422 on create
  (rare — the CLI logs real ids; no security issue). Proper fix = a dedicated conventions-for-alerts
  endpoint.
- **[M2] Create-path reachability query.** The create validation reads `convention.resource` after a
  `joins`-based resolution → one extra query per create (negligible). Could `.includes(:resource)`.
