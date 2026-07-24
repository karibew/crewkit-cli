# Design: Team-activity redesign — active sessions + current-work summary

**Status:** design (brainstorm approved; gate-1/2 security review cleared — 9 must-fix constraints folded into "Security constraints" below) · **Date:** 2026-07-24
**Scope:** cross-component (api `ActiveWorkQuery` + serializer, cli `format_active_work`, dashboard `active-work-strip` + `org-active-work`)

## Problem

The team-activity / active-work view is meant to show *who is working on what right now* so teammates and agents avoid collisions. Today it (a) can include idle-but-open sessions, cluttering the view, and (b) shows only a title (often "(untitled)" mid-session, since summaries are generated at session end) — it does not convey **what the session is currently doing**, which is the most useful signal.

## Goals

- Show only sessions with **real activity in the last 24h** (not merely open).
- Surface a **brief summary of current work** per session — the primary field.
- Row shape: **Name · Session · current work · last update time**.
- Two audiences, one payload: **agent** view (dense, conflict-first, scannable) and **dashboard** view (human-readable).

## Non-goals

- No new LLM summarization / no transcript egress (Loi 25). The summary source is existing agent-posted `work_updates` only.
- No schema change (query-time).
- No change to how sessions are created or to end-of-session analysis.

## Design

### Summary source (decided)
The **latest `work_update.body`** for the session (`work_updates` already exists: `body`, `llm_session_id`, `created_at`, indexed on `llm_session_id`). Agents post these via the `share_update` MCP tool (#225). No new write path.

- **No update posted →** the current-work field shows the literal placeholder **`(no update posted)`** (nudges agents to broadcast). No heuristic fallback.

### Active filter (decided)
A session appears only when its **`updated_at` is within the last 24h** (real activity) — **not** the current `recently_active` semantics (`status='active' OR updated_at >= since`), whose `status='active'` branch keeps open-but-idle sessions forever and is the reason idle sessions clutter today. Implementation MUST gate on `updated_at >= since` (dropping the `OR status='active'` escape) and **verify `updated_at` is actually bumped on events/turns** (if not, use the latest event/turn timestamp instead). `since` window default becomes **1440 min (24h)**. Ended sessions are excluded.

### Row shape
`Name · Session · current work · last update`
- **Session:** codename if set, else short session id + repo name.
- **current work:** latest `work_update.body` (trimmed to a bound), else `(no update posted)`.
- **last update:** the work-update `created_at` (relative) if present, else last-activity time.

### Two renderings, one payload
The API payload carries per-session `{ name, session (codename/id + repo), current_work (body|null), current_work_at, last_activity_at }`. Renderers diverge:
- **Agent — `format_active_work` (MCP tool result):** one dense line per session, sorted by recency, conflict-first, e.g.
  `testbot · portal-auth (customer-portal) · "reworking JWT refresh races" · 4m ago`.
  Remains inside the `wrap_untrusted` nonce fence — a `work_update.body` is untrusted free text (see untrusted-injection-fencing).
- **Dashboard — `active-work-strip` / `org-active-work`:** human-readable; the current-work sentence becomes the prominent line (not hover-only), with name/avatar, session link, and relative time.

### Visibility (security — decided: hide entirely)
Sensitive **or** private sessions must **not appear at all** for non-owners (row dropped, not redacted). `ActiveWorkQuery` already applies `.visible_to(user, permission_level)` (repo `session_visibility`: public/restricted/private) and `.exclude_sensitive_unless_owner(user)` — both row-dropping WHERE clauses. **Invariant:** the latest-work-update lookup rides on the **already-filtered** session set, so a hidden session's body can never be joined/leaked. This is a new read path over `LlmSession` content → the recurring session-visibility-leak class applies; sentinel-tested.

### Security constraints (gate-1/2 — MUST hold; block if violated)
1. **Join key = surviving `llm_session_id` set only.** Fetch the latest update via `WorkUpdate.where(llm_session_id: <ids that passed `apply_visibility` in `conversations_relation`>)`. **Forbidden:** `policy_scope(WorkUpdate)` and `@project.work_updates` (their scope is `WorkUpdatePolicy::Scope` = **org-only**, no visibility/sensitivity gate → leaks bodies from sensitive/private sessions), and any key on `user_id` / `project_id` / `blueprint_task_id` (bypasses the row filters). The join lives **inside** `ActiveWorkQuery`, downstream of `apply_visibility`, with a doc-comment marking the chokepoint. `recallable?` is NOT a substitute (it's viewer-independent and misses restricted repos for non-managers).
2. **`current_work` attaches to conversation rows only** — never to `claimed_blueprint_tasks` rows, because a `work_update.blueprint_task_id` can reference an update whose *source session* is hidden. Task-row current-work is out of scope; if ever added it must re-derive session visibility, not key on `blueprint_task_id`.
3. **Sensitive-after-the-fact → row-drop suppresses the body everywhere.** Because the read is query-time on `exclude_sensitive_unless_owner`, flagging `sensitive` post-hoc drops the whole row on both renderings (no stale copy to retract). Sentinel-tested (below).
4. **NULL `llm_session_id` never falls back.** FK is `on_delete: :nullify`; a deleted session NULLs the update's `llm_session_id`. `WHERE llm_session_id IN (non-null ids)` excludes these — no `user_id`/`project_id` fallback.
5. **One bounded query, no N+1.** Newest body per surviving session via `DISTINCT ON (llm_session_id) … ORDER BY llm_session_id, created_at DESC` (bodies, not just `MAX(created_at)`). Inherits the ≤`MAX_TOTAL_CONVERSATIONS` (500) surviving-id bound; `limit_per_user` (≤25) + `since` (≤24h) clamps stay server-side. Belt-and-suspenders: add `.where(organization_id: current_organization.id)`.
6. **Agent path: `current_work` renders INSIDE the `wrap_untrusted` fence** (built into `body`, like the conversation title), never in the trusted count header. The `(no update posted)` placeholder is crewkit-authored/trusted. Sentinel asserts `<crewkit_untrusted:` around a body carrying breakout/`ignore previous instructions` text.
7. **Dashboard: `current_work` is PLAIN TEXT** — React default escaping only; **no** `dangerouslySetInnerHTML`, **no** markdown renderer on this field (would reintroduce `[x](javascript:)`/image XSS+SSRF). Because the dashboard has no invisible-strip, the **serializer re-runs `ContentSanitizer.clean(body)` on read** (idempotent, cheap) to neutralize any legacy/unsanitized row before it reaches a strip-less client.
8. **Explicit char-based payload trim** on the body, independent of the model's write-time `MAX_BODY_CHARS` (280), so a future cap bump can't silently blow the strip/dashboard budget.
9. **Cross-poster is intended:** an update's `user_id` may differ from the session owner (any project member can post to a session). Visibility follows the **session**, not the poster — do not "correct" this into a `user_id` filter.

### Plumbing (decided)
**Query-time**: one `DISTINCT ON (llm_session_id) … ORDER BY llm_session_id, created_at DESC` over the surviving filtered session ids (returns the newest *body* per session, not just a timestamp), bounded by the ≤500 surviving-id set (per constraint 5). No denormalized column, no migration.

## Edge cases

- Multiple sessions per user → existing `limit_per_user` clamp applies; sort by recency.
- Long `work_update.body` → trim to a fixed char bound in the payload.
- Session with updates but now idle (>24h) → excluded by the activity filter even if it has updates.
- `include_self` behavior unchanged (dashboard strip passes `false`).
- Null-user / `IS DISTINCT FROM` correctness preserved (per prior #275 fix).

## Testing

- `ActiveWorkQuery` unit: latest-update-per-session (DISTINCT ON), 24h `updated_at` window excludes idle/stale, placeholder when none, NULL `llm_session_id` excluded, single-query (no N+1).
- **Security sentinel** (in `api/test/sentinels/`, so a regression short-circuits CI): sessions each carrying a `work_update` — (a) `sensitive` (flagged after the update was posted), (b) private-repo, (c) restricted-repo viewed by a `developer` (the case `recallable?` misses) — do **NOT** appear and their body never leaks, on **both** the agent payload and dashboard payload; all appear for the owner. Include the cross-poster case (poster ≠ session owner) to prove visibility follows the session.
- `format_active_work` (cli): dense per-session line; `(no update posted)` placeholder; `current_work` stays inside the fence (`<crewkit_untrusted:`) even with breakout text in the body.
- Dashboard components: current-work prominent, rendered as escaped plain text (no markdown/HTML); serializer scrubs invisibles; empty/self-excluded states unchanged.

## Files

- api: `app/services/active_work_query.rb` (the join chokepoint), `app/serializers/active_work_conversation_serializer.rb` (current_work + read-side `ContentSanitizer.clean` + char-trim), `OrgActiveWorkController`/project active-work endpoint (window default), `test/sentinels/` leak test. Note: do **not** use `WorkUpdatePolicy::Scope` as the source (org-only).
- cli: `src/commands/mcp.rs` `format_active_work`.
- dashboard: `components/features/projects/active-work-strip.tsx`, `components/features/dashboard/org-active-work.tsx`, `lib/api/active-work` types.

## Next (repo SDLC)

Design → **gate-1 security threat model + gate-2 adversarial design review** (visibility/leak focus) → writing-plans → staged build with gate-3 trio.
