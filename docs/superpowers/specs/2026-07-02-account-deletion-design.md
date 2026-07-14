# Account Deletion — GDPR/Loi 25 (#228) + Org-Removal (#254)

Design doc. Gate-1 threat model recorded on #228; decisions settled with the product owner 2026-07-02. **Gate-2 security review pending before any code.**

## 1. Scope & the two flows

One shared primitive — `anonymize_to_former_member(scope)` = null `user_id` on the in-scope sessions (rendered "Former member"). Two callers:

| Flow | Issue | Session scope nulled | User account | Other orgs | Auth |
|---|---|---|---|---|---|
| **Self-service account deletion** | #228 | all orgs | soft-deleted → purged after grace | membership removed everywhere | owner + **step-up** |
| **Org "remove & anonymize former member"** | #254 (separate) | one org (`org_id = X`) | untouched | untouched | org admin (Pundit) |

This doc covers **#228**; the org-scoped caller (#254) reuses the primitive and ships separately. Plain "remove member" is unchanged.

**Keep-optional (self-service):** the user chooses at request time:
- `anonymize` (default) — null `user_id` on their sessions everywhere, scrub residual PII; org keeps metrics. Irreversible anonymization (Loi 25 standard).
- `purge` — additionally delete the sessions + retract their KB docs. Stronger erasure. **Full-purge of an org's business records at the individual's sole discretion needs Félix-Antoine/Rémi sign-off** (Loi 25 controller nuance) — gated behind a config flag until then.

## 2. API contracts (API-first)

Replaces the current `DELETE /auth/me` immediate hard-delete (`auth_controller.rb:697`), which is retired/redirected.

```
POST   /api/v1/account/deletion            # request deletion → sends step-up challenge; 202
  body: { mode: "anonymize" | "purge" }    # default "anonymize"; "purge" rejected 403 unless flag on
  → 202 { status: "step_up_sent", channel: "email" }

POST   /api/v1/account/deletion/confirm    # redeem step-up token → schedule deletion; 202
  body: { step_up_token: "<one-time>" }
  → 202 { status: "pending", grace_ends_at, cancel_until }

DELETE /api/v1/account/deletion            # cancel during grace (auth as the deleting user)
  → 200 { status: "cancelled" }

GET    /api/v1/account/deletion            # current request status (pending/none)
  → 200 { status, mode, requested_at, grace_ends_at } | 404
```

- Account-scoped, **never** accepts a user/id param (no IDOR), mirroring `account_exports_controller`.
- Rate-limited (Rack::Attack): `POST .../deletion` 5/user/day; `.../confirm` 10/user/hour.
- Every transition emits a `SecurityEvent` (`account.deletion.{requested,confirmed,cancelled,completed,failed}`) + an `EventPublisher` domain event.
- OpenAPI updated; `openapi:validate STRICT=1` must pass.

## 3. Data model

**Migration 1 — soft-delete + grace on `users`** (User has no `acts_as_paranoid` today):
- `deletion_requested_at` (datetime, null), `deletion_mode` (string, null: "anonymize"|"purge"), `deletion_scheduled_job_id` (string, null, for cancellation-race safety), `deleted_at` (datetime, null — soft-disable marker).
- A user with `deletion_requested_at` present is "pending deletion": login allowed **only** to cancel; new session creation rejected; refresh blocked except the cancel path.

**Migration 2 — nullable `user_id` (B1)**:
- `llm_sessions.user_id` → `null: true`; FK `on_delete: :nullify`.
- `LlmSession#belongs_to :user` → `optional: true` (`llm_session.rb:27`).
- `User has_many :llm_sessions` → `dependent: :nullify` (was `:destroy`, `user.rb:25`) so account destroy anonymizes rather than deletes conversations.

**Migration 3 — `account_deletions` audit/state table** (mirrors `data_exports`): `user_id` (nullable, survives user row), `mode`, `status` (enum pending/processing/completed/failed/cancelled), `requested_at`, `grace_ends_at`, `completed_at`, `external_id`, `steps_completed` (jsonb — per-step checkpoint for resumable job, gate-2 H3). Retained as the compliance receipt after the user row is gone.
> **Gate-2 M7 — the receipt must not re-identify.** Do NOT persist `email`/`name`/any PII here (incl. in error fields). Store a **non-reversible user hash** (e.g. HMAC of external_id) + timestamps only; `error_message` must be a scrubbed code/class, never raw text that could carry an email. Added to the §7 erasure checklist.

**Migration 4 — orphan-attribution FKs → `ON DELETE nullify`** for the no-cascade tables (blueprints.created_by_id, invitations.inviter_id/invitee_id, session_shares.created_by_id, share_annotations.author_id, blueprint_chat_messages.user_id, cloud_jobs.user_id, resources.created_by_id, resource_version_comparisons.created_by_id). Preserves records, drops attribution.

## 4. The primitive + services/jobs

- **`AccountAnonymizationService.anonymize_to_former_member(user:, org:)`** — the shared primitive; `org` is a **required keyword (gate-2 M6)**: the org-scoped caller (#254) must pass a concrete org and can never reach the all-orgs path, which is invoked only by the authenticated-self flow via an explicit separate entrypoint. `scope = LlmSession.where(user:)`, narrowed to `organization: org` for the org caller; → `update_all(user_id: nil)` + scrub denormalized user fields on that scope's child rows + git-author/free-text PII. The `mode: purge` gate flag is checked **server-side in the service, not only the controller** (defense in depth). Sentinel: an org-admin call can never null `user_id` outside its org.
- **`AccountDeletionService`** — orchestrates request/confirm/cancel; writes `account_deletions` + `SecurityEvent`; schedules `PermanentlyDeleteAccountJob` at `grace_ends_at`; records the job id for cancellation.
- **`PermanentlyDeleteAccountJob(account_deletion_id)`** — runs after grace. **Idempotent + resumable** via `steps_completed` checkpoints (each step records completion; re-run skips done steps, never redoes an irreversible one). **Cancellation-race-safe (gate-2 H4):** the job's FIRST action, inside a row lock, is `account_deletion.lock! (SELECT … FOR UPDATE)`, abort unless `status == pending`, then flip to `processing` atomically; the cancel endpoint also takes `FOR UPDATE` and refuses once `processing`. Ordered:
  1. `UserTokenRevocationService` — revoke at provider (GitHub/GitLab/Bitbucket/Google) **before** touching connection rows. **Fail-CLOSED (gate-2 H5):** on revoke failure keep the connection row, set the deletion `status: failed`, emit a per-provider `account.deletion.failed`, and retry via the sweep — do NOT delete the row (and do not proceed to destroy the user) until every provider revoke is confirmed. A live-token-with-no-local-record is the harm we refuse.
  2. If `mode == purge`: delete the user's `llm_sessions` (→ cascades `artifacts` → `after_destroy` retracts pgvector + DO KB). **KB retraction is BLOCKING for purge (gate-2 H3):** bounded retry; on failure → `status: failed` + alert, and do NOT destroy the user until DO KB deletion is confirmed (orphaned vectors are the Loi 25 breach; a lingering user row is recoverable). If `mode == anonymize`: `anonymize_to_former_member` (null user_id, keep sessions).
  3. `AuditLogCleanupService` — null `versions.whodunnit`, scrub `object/object_changes/request_params` of the user's email; delete/nullify orphaned `security_events`/`admin_audit_logs`/`billing_audit_logs`.
  4. Nullify orphan-attribution FKs (covered by Migration 4 on user destroy).
  5. Destroy the user row (personal org via `really_destroy!` as today); emit `account.deletion.completed`. Return the actual post-lock outcome — never an optimistic success.
- **`PurgeStaleAccountDeletionsJob`** (recurring, `recurring.yml`) — sweeps failed/orphaned rows, like `PurgeExpiredDataExportsJob`.
- **KB retraction hardening** — the artifact `after_destroy` KB delete currently swallows errors; add retry + a failure metric so a purge can't silently leave vectors in DO KB (re-identification risk).

## 5. Step-up auth (B4)

Reuse `MagicLinkToken` with a new `purpose` column (10-min TTL, one-time, digest-only). `POST /account/deletion` mints a `step_up` link to the user's email; `POST /account/deletion/confirm` redeems it and only then schedules deletion. Passkey step-up (`WebauthnChallenge` `challenge_type: "step_up"`) is a parallel path. Refresh-token revocation reuses the existing `refresh_tokens.each(&:revoke!)`.

> **Gate-2 C1 — the step-up token MUST be bound, server-side.** `MagicLinkToken.find_valid` today matches on **digest only** (no user/purpose). Before `use!`, `confirm` MUST assert all three atomically: `token.purpose == "step_up"` AND `token.user_id == current_user.id` AND unused/unexpired. Add a `purpose` column + scoped finder. Without this, a normal login link is redeemable at `/confirm`, and a `step_up` token is redeemable at the login endpoint to mint a full session. This binding is a **sentinel test**.

## 5a. Pending-deletion enforcement — the single load-bearing control (gate-2 C2)

"Login only to cancel; reject new sessions; refresh blocked" MUST be **centralized at the auth chokepoint**, not per-endpoint. Today `authenticate_api_request!` (`base_controller.rb:41`) is a bare `User.find_by!(external_id:)` with no state check, so a still-valid 4-hr JWT keeps working everywhere and `refresh` (`auth_controller.rb:563`) re-mints a fresh 4-hr JWT **plus** a new 30-day refresh token — indefinitely extending access during grace.

**Fix:** in `authenticate_api_request!`, if `current_user.deletion_requested_at?`, reject **403** for every route except an explicit cancel-flow allowlist (controller+action). Additionally block `refresh`, `token`, passkey-login, and device-verify for pending users (the cancel path uses step-up, not a full session). This is the one control the whole grace guarantee rests on → **sentinel test** covering: valid pre-existing JWT is refused on a normal endpoint while pending; refresh is refused; only the cancel endpoint works.

## 6. Nullable-user_id reconciliation (must ship WITH Migration 2)

Null `user_id` = "Former member". Sites to fix (from investigation):
- `llm_session.rb:27` `belongs_to :user, optional: true`; `llm_session_serializer.rb:117` nil-guard → `{ name: "Former member" }`.
- `session_analytics_service.rb` — `joins(:user)`→`left_joins(:user)` at ~196/229/264/346; `.compact` on plucked user_id at ~414/491; `by_user` groups a "Former member" bucket for null.
- `sessions_controller.rb:114` `.compact.uniq`; `project_active_work_controller.rb:104` `.compact`; `convention_override_analytics_service.rb:45` decide null handling.
- These are covered by targeted tests asserting analytics don't drop null-user rows.

## 7. Erasure surface (anonymize vs purge)

Full checklist in the investigation notes; highlights:
- **Cascade-clean already:** auth credentials (webauthn/magic/device/refresh), UOR, subagent_sessions/turns/plans/skill_invocations, artifacts→pgvector+DO KB (callback).
- **New gaps to build:** provider token revocation (§4.1), PaperTrail sanitization (§4.3), orphan-attribution FKs (Migration 4), `convention_overrides.resolved_by` string scrub.

## 8. Threat-model responses (gate-1 → design)

- Irreversible anonymization (null, no back-reference) — not reversible de-id. ✓
- Tokens revoked at provider before row delete. ✓ (§4.1)
- Grace: soft-disable, revoke refresh, login-only-to-cancel, reject new sessions, cancel aborts job. ✓ (§3, §5)
- Job idempotent/ordered/cancellation-race-safe/audited. ✓ (§4)
- Sole-owner-of-shared-org: reuse existing 409 guard (`auth_controller.rb:707`), extended to the new endpoint. ✓
- Loi 25: confirmation email + `account_deletions` receipt + logged request time. ✓
- **Pre-existing, out of scope, flag to Félix/Rémi:** `user_git_connections` tokens are plaintext (oauth_connections are encrypted). Should be encrypted regardless.

## 9. Staging (ordered PRs, each gated)

1. **Schema + nullable reconciliation** — Migrations 1–2, association + analytics/serializer fixes, tests. (Ships safely alone; no behavior change yet.)
2. **Anonymize primitive + org caller (#254)** — `AccountAnonymizationService` + org endpoint/confirmation/audit. (Delivers #254.)
3. **Self-service request/confirm/cancel + step-up** — endpoints, `AccountDeletionService`, `account_deletions` table, grace enforcement, retire `DELETE /auth/me`.
4. **PermanentlyDeleteAccountJob + erasure machinery** — token revocation, audit sanitization, orphan FKs (Migration 4), KB hardening, recurring sweep.
5. **Dashboard UI** — "Delete my account" in the Danger Zone (mode choice, step-up, pending/cancel states), mirroring the export section.

Each stage: affected-tier tests + sentinels + `implementation-validator → adversarial-code-reviewer → security-expert`.

## 10. Open items needing a human decision

- **Full-purge policy sign-off** (Félix/Rémi) — before enabling `mode: "purge"` of org business records.
- **Plaintext git tokens** — separate hardening ticket.
