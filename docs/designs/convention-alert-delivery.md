# Convention Alert Delivery (#265 deferred half)

Slice A (#265) shipped `ConventionAlertConfig` persistence + management UI + a **store-time** SSRF
allowlist, but **no delivery**. This builds the delivery engine: a threshold-check job that, when a
convention is overridden too often, notifies the team via email and/or Slack — honoring the send-time
security constraints #265 recorded as forward requirements.

## What a config already provides (engine consumes; writes 3 counters)

`threshold_count` (1..10k), `threshold_period` (hour|day|week), `cooldown_minutes` (0..10080),
`enabled`, `email_enabled` + `email_recipients` (jsonb), `slack_enabled` + `slack_webhook_url`
(**encrypted**) + `slack_channel`, unique `(organization_id, playbook_convention_id)`. Engine WRITES
only: `alerts_sent_count` (++), `last_alerted_at` (drives cooldown), `last_triggered_at`.

## Trigger + threshold

Countable signal = `ConventionOverride` rows with `outcome: "overridden"` (resolved server-side).
Query (indexed, org-scoped): count `org.convention_overrides.overridden.in_period(period_start, now)`
where `convention_id == config.convention.external_id` (NOTE: `ConventionOverride.convention_id` is the
convention's **string external_id**, not the FK — map via `config.convention.external_id`;
convention lookup stays org-scoped). `period_start` = now − {hour|day|week}. Fire when
`count >= threshold_count`. Reuses the `ConventionOverrideAnalyticsService` scope pattern.

## Engine — `ConventionAlertThresholdJob` (recurring)

`perform(config_id = nil)` — targeted (one config) vs sweep (all `enabled` configs); mirrors
`ResolveConventionOverridesJob`. Registered in all three `config/recurring.yml` blocks (dev ~10m,
staging ~30m, prod hourly — match existing cadence). Per config, **inside `config.with_lock`** (row
`SELECT … FOR UPDATE` — the concurrency idiom here; no advisory-lock gem):
1. Skip unless `enabled` and at least one channel enabled.
2. **Cooldown:** skip if `last_alerted_at` present and `> cooldown_minutes.ago`.
3. Count overrides in the period; skip if `< threshold_count`.
4. Set `last_triggered_at = now`; deliver to enabled channels; set `last_alerted_at = now` and
   `alerts_sent_count += 1`.
`with_lock` + the cooldown re-read make overlapping cron ticks double-send-safe. Best-effort:
`last_alerted_at` is set on **attempt** (threshold crossed + past cooldown), so a transient channel
failure doesn't hammer the team every tick; per-channel failures are logged (never fatal to the job).
`retry_on Net::OpenTimeout/ReadTimeout` for transient egress errors; the sweep is bounded by config count.

## Delivery — minimal, hardened

`ConventionAlertDeliveryService` builds ONE minimal payload and fans out to enabled channels.

- **Email:** new `ConventionAlertMailer` (ActionMailer; SMTP-via-SendGrid already configured for
  prod/staging, letter_opener dev, `:test` in tests). **Re-filter `email_recipients` to CURRENT
  org-member emails at send** (reuse the model's org-member lookup — actual members, never
  `User.exists?`); drop non-members; skip email if none remain.
- **Slack:** send-time-hardened POST to the decrypted `slack_webhook_url`. Model the HTTP on
  `SlackNotifier` BUT add the controls below (SlackNotifier is for a trusted ENV webhook; this URL is
  user-supplied).

## Security / threat model (gate-1) — these are MUST

1. **SSRF (send-time re-pin).** The store-time allowlist is a DNS-independent string check. At send:
   after decrypt, **re-parse** the URL and re-verify `scheme https + host == "hooks.slack.com" +
   path starts /services/`; **disable redirects** (`max_redirects 0`); short timeouts. Refuse+log
   (no URL) if it no longer matches. (Prevents a stored-then-mutated or redirect/rebind egress.)
2. **Recipient re-validation (send-time).** Membership is mutable — re-filter `email_recipients` to
   current org members at send; a removed member MUST NOT receive. Write-time validation is UX only.
3. **Secret non-logging.** The decrypted `slack_webhook_url` must NEVER reach `Rails.logger`, Sentry
   `extra:`, or any error/exception message. Log only response code / a boolean. The secret sentinel
   (`convention_alert_config_secret_sentinel_test`) must stay green — no new leak surface.
4. **Loi 25 / PII minimization.** The alert body carries **convention category/key + "overridden N
   times in {period}" + a dashboard deep-link ONLY**. It MUST NOT include `proposed_value`,
   `convention_value`, `override_reason`, or any override free-text/snippet — those are CLI-captured,
   can hold code/PII/secrets, and egress to third parties (Slack/email). Mirrors the model's own
   "counts only, never the raw content" discipline.
5. **Tenant isolation.** Config, override count, and convention lookup all org-scoped; `convention_id`
   is untrusted string input so the count query is bound to `org.convention_overrides`.
6. **Dedup / DoS.** `cooldown_minutes` + `with_lock` re-read prevent alert spam and double-send; the
   sweep only touches `enabled` configs.
7. **Test-send** (below) reuses the SAME delivery path + security; it does NOT touch counters/cooldown.

## Test-send endpoint

`POST /:org_id/convention_alert_configs/:id/test` — manager+ (matches the config write authz). Sends a
clearly-labeled **sample** alert immediately to the config's enabled channels (verifies webhook +
recipients work), bypassing threshold/cooldown, **without** mutating counters. Same send-time SSRF
re-pin + recipient re-filter + no-PII body + secret non-logging. Rate-limit / manager-gate to avoid
abuse.

## Stages (each: build → affected gates → implementation-validator → adversarial-code-reviewer → security-expert)

1. **Delivery core:** `ConventionAlertDeliveryService` + `ConventionAlertMailer` + the hardened Slack
   POST (send-time re-pin, no redirects, no secret logging) + the minimal no-PII payload. Tests:
   Slack SSRF re-pin (reject non-hooks.slack.com after decrypt; no redirects), recipient re-filter
   (removed member dropped), body excludes override free-text (WebMock body assertion + mailer
   deliveries), secret never logged.
2. **Threshold job:** `ConventionAlertThresholdJob` (targeted+sweep, `with_lock`, cooldown, counters) +
   `recurring.yml` (3 blocks) + `retry_on`. Tests: fires at threshold, respects cooldown, no
   double-send under re-run, targeted vs sweep, org-scoped count, counters written.
3. **Test-send endpoint:** route + controller action + policy (manager+) + OpenAPI; no counter
   mutation. Tests: authz, sends via delivery service, no-PII, cross-org 404.

Patterns to copy: `ResolveConventionOverridesJob` (targeted/sweep), `ReapAbandonedSubagentSessionsJob`
(recurring + in-loop re-check), `NotifyCloudJobHaltedJob` (webhook-posting job + retry_on), the
existing mailers under `app/mailers/`, `SlackNotifier` (HTTP shape — plus the send-time pinning),
`ConventionOverrideAnalyticsService` (org-scoped override count), `SendgridSenderService` egress test
(WebMock body capture).

## Gate-2 review outcome (security-expert, 2026-07-14) — PROCEED-WITH-CHANGES

No unbounded-SSRF or cross-tenant hole. Fold these in during the build.

**BLOCK-level (must):**
- **B1 — test-send authz = `admin?`, not manager+.** Config write is `admin?` (policy E1); test-send
  triggers on-demand third-party egress → must be ≥ that. `manager?` is strictly weaker; use `admin?`.
- **B2 — Time-precise count.** Do NOT reuse `ConventionOverride.in_period` (it floors args to
  `beginning_of_day`/`end_of_day` → an `hour`/`week` window over-counts → premature/duplicate egress).
  Use `where(created_at: period_start..now)` on the org-scoped `overridden` relation.
- **B3 — Display fields from the AUTHORED convention, not override rows.** `ConventionOverride` has
  denormalized `convention_key`/`convention_category`/`convention_value` = CLI free-form (untrusted,
  Loi-25/injection risk). The alert's key/category MUST come from `config.convention`
  (`PlaybookConvention`, authored). Exclusion list explicitly adds the override's
  `convention_key`/`convention_category`/`convention_value` (on top of `proposed_value`/`override_reason`).
- **B4 — Resolve retry vs best-effort.** Rescue egress inline (never raise, no secret); commit
  `last_alerted_at`/`last_triggered_at`/`alerts_sent_count++` **before** egress (that ordering IS the
  dedup — biases to drop-over-double, correct here); scope `retry_on` to DB/transient-infra only,
  NOT delivery.
- **B5 — Don't hold `with_lock` across egress.** Claim under lock (re-check cooldown+threshold, set
  counters, commit → release), then deliver best-effort OUTSIDE the lock (avoids lock-pileup across
  the sweep + realizes B4 ordering). Guard the `config.convention` nil case.

**Defense-in-depth (do):**
- **S1 — SSRF:** re-pin (scheme https + host literal `hooks.slack.com` + path `/services/`) + no
  redirects + timeout is adequate (attacker doesn't control the host, so classic rebinding N/A). Add
  cheaply: reject if the resolved address is loopback/private/link-local (poisoned-resolver guard);
  force **port 443**; treat ANY 3xx/non-2xx as a hard failure that is never followed (never read
  `Location`). Full connect-by-IP+SNI is NOT required.
- **S2 — Secret seams:** no `uri`/`url`/`path` in any rescue message; do NOT log `response.body`
  (status code / boolean only); normalize delivery + test-send error surfaces (never the URL or raw
  upstream body); confirm `:slack_webhook_url` in `filter_parameters`; rescue
  `ActiveRecord::Encryption` decrypt errors → skip + log without the value.
- **S3 — Email:** NO per-user attribution (who overrode = Loi-25 monitoring PII) — count only; if the
  convention `value` is included, source it from `config.convention`; send per-recipient or `bcc:` so
  addresses aren't disclosed in `To:`; reuse the model's exact org-scoped downcased member query. The
  20-recipient model cap bounds fan-out (no extra cap needed).
- **S4 — test-send abuse:** add a concrete `rack-attack` throttle keyed on org+config; response
  normalized (inert as SSRF oracle); assert it never touches `last_alerted_at`/counters.
- **S5 — Observability:** emit a log/metric (no secret) on delivery failure so a suppressed-then-
  dropped alert is visible to ops.
