# Work Updates — Design Spec

**Date:** 2026-06-22
**Builds on:** the cross-session-awareness feature (branch `feat/cross-session-awareness` / PR #224) — specifically `ContextPulseService`, the `context_pulse` endpoint, and the CLI `format_pulse` hook. This feature adds the **send** side and a **dashboard recap**.
**Status:** Approved design, pre-implementation

---

## Problem

Cross-session awareness currently only **receives**: an open session pulls a delta of teammates' activity. But the only thing a live session *sends* is passive telemetry — `updated_at` bumps and a static `codename`. `short_summary` is written only at session end; task progress can't be set by an agent at all. There is no way for an agent to deliberately tell the project *"I switched to the OAuth refactor"* or *"I finished the migration."*

This feature adds an explicit **send** surface (an MCP tool the agent calls), feeds those messages into the existing receive path (other sessions' pulse), and surfaces them as a **dashboard recap** of what the agents are telling each other. (Slack delivery is intentionally deferred.)

## Goals

- An agent can post a short, project-visible work update, optionally linked to a Blueprint task.
- Those updates flow into the existing `ContextPulseService` so other open sessions receive them.
- A project-page dashboard panel shows the recent feed of updates.
- Reuse the existing privacy gate (`SaveState.session_recallable?`) on both write and read.

## Non-goals

- No Slack/webhook delivery (deferred — UI first).
- No automatic/heuristic posting — updates are **explicit agent tool calls** only.
- No per-call user approval (kept frictionless; mitigated by the privacy gate + sanitization + a tool description that instructs high-level, non-confidential notes).
- No `kind`/category taxonomy in v1 (free-text note + optional task link only).
- No editing/threading of updates — append-only feed.

---

## Architecture

```
Agent (live session)
  └─ MCP tool share_update{note, task_id?}
        → POST /:org/projects/:id/work_updates   (sanitize, write-gate on session, create)
                                   │
                          WorkUpdate (append-only, project-scoped)
                          ┌────────┴─────────┐
   ContextPulseService.updates             WorkUpdatesController#index
   (since, gated, exclude self)            (recent, gated, all users)
            │                                       │
   CLI format_pulse "Updates"             Dashboard WorkUpdatesPanel
   (injected into other open sessions)    (project page recap, ~30s poll)
```

---

## Components

### 1. Data model — `WorkUpdate`

New table `work_updates` (append-only; sibling to `save_states`):

| Column | Type | Notes |
|---|---|---|
| `external_id` | uuid | PK-style public id, unique index |
| `organization_id` | bigint FK | tenant scope |
| `project_id` | bigint FK | feed scope |
| `user_id` | bigint FK | author |
| `llm_session_id` | bigint FK, nullable | source session; `on_delete: :nullify`; drives the privacy gate |
| `blueprint_task_id` | bigint FK, nullable | optional linked task; `on_delete: :nullify` |
| `body` | string/text | the note; validated present, length ≤ `MAX_BODY_CHARS` (280) |
| `created_at` | datetime | feed ordering |
| `updated_at` | datetime | standard |

- Associations: `belongs_to :organization, :project, :user`; `belongs_to :llm_session, optional: true`; `belongs_to :blueprint_task, optional: true`.
- Index: `[project_id, created_at]` (feed queries) and unique `external_id`.
- Recallability helper: a `WorkUpdate` is recallable iff its `llm_session` is nil or `SaveState.session_recallable?(llm_session)`. Add `WorkUpdate#recallable?` delegating to that single source of truth.
- No soft-delete in v1 (append-only, no client delete path). Body sanitized at write.

### 2. Send — MCP tool + API

**MCP tool `share_update`** (in `cli/src/commands/mcp.rs`, alongside `save_state`):
- Input: `{ note: String, task_id: Option<String> }`.
- Description instructs the agent: call on a focus change or milestone; one short, high-level, non-confidential line; optionally pass the Blueprint `task_id` it's progressing.
- Resolves scope + `project_id`; sends `source_session_id = ctx.session_id` (the active `CREWKIT_LLM_SESSION_ID`), exactly as `save_state` does.
- Calls `api_client.create_work_update(org_id, project_id, params)`.

**API `POST /:org_id/projects/:project_id/work_updates`** (`WorkUpdatesController#create`):
- Pundit: project member, developer+ (mirror `SaveStatePolicy` personal-write tier).
- `ContentSanitizer.clean` the note; reject if blank after clean; truncate to `MAX_BODY_CHARS`.
- Resolve `source_session_id` (optional) → if present and **not** `SaveState.session_recallable?`, return **422** (don't broadcast from a sensitive/private session). Absent session → allowed (treated as internal).
- Optional `task_id` → resolve a `BlueprintTask` whose blueprint is in this project; ignore silently if not found (don't fail the post over a bad link) — or 422; **decision: ignore + omit the link** to keep posting robust.
- Create `WorkUpdate`; return the serialized record.

### 3. Receive — extend `ContextPulseService`

Add a 4th key `updates` to `ContextPulseService#build`:
- `@project.work_updates.where("created_at >= ?", @since).where.not(user_id: @session.user_id).includes(:user, :blueprint_task, :llm_session).order(created_at: :desc).limit(MAX_ITEMS)`.
- Filter `.select(&:recallable?)` (gate by source session).
- Map → `{ user: name.presence || "a teammate", note:, task: blueprint_task&.name, created_at: iso8601 }`.
- The `note` is intentionally included verbatim (the agent deliberately authored a shareable line — unlike the auto-generated `short_summary` we deliberately dropped earlier).

**CLI `format_pulse`** (`cli/src/services/hook_service.rs`): add an "updates" section — `• {user}: {note}` (append `(task: {task})` when present). Unit-test the new branch.

### 4. Dashboard recap

- **API `GET /:org_id/projects/:project_id/work_updates?limit=`** (`WorkUpdatesController#index`): recent updates, gated by `recallable?`, **all users** (it's a team recap), ordered `created_at desc`, `limit` default 50 / max 100. Serialize `{ external_id, user: {name}, note, task: {external_id, name} | null, created_at }`.
- **Client + hook:** `dashboard/src/lib/api/work-updates.ts` + `dashboard/src/hooks/use-work-updates.ts` (poll ~30s, mirror `use-active-work.ts`).
- **Component:** `dashboard/src/components/features/projects/work-updates-panel.tsx` — `WorkUpdatesPanel`: list of name · note · linked-task chip · relative time; empty state. Slot into `dashboard/src/app/kit/projects/[id]/page.tsx` beside `ActiveWorkStrip` / `SaveStatesPanel`.
- Types regenerated from OpenAPI.

### 5. Privacy / tenancy

- `SaveState.session_recallable?` gate on **write** (refuse broadcasting from sensitive/private sessions) **and read** (pulse + dashboard filter out updates whose source session is non-recallable).
- Org/project scoped everywhere (`current_organization.projects.find_by!`); `@project.work_updates`.
- Pulse **excludes the caller's own** updates; the dashboard recap **includes all** (it's a team view).
- Body is `ContentSanitizer.clean`ed at write (strips invisible/control chars) and length-capped.

### 6. Token posture (constraint, tuned later)

Explicit short notes are cheap; the pulse caps `updates` at `MAX_ITEMS` and is already `since`-windowed + throttled (~10 min). The dashboard read is server-side (free to the agent). Deeper token-minimization is a follow-up, not part of this spec.

---

## Data flow

1. Agent calls `share_update("switched to the OAuth refactor", task_id: "task-abc")`.
2. CLI POSTs to `work_updates` with the active session id; server sanitizes, gates on the session, creates the `WorkUpdate`.
3. A teammate's open session fires its throttled pulse → `ContextPulseService` returns the new update in `updates` → `format_pulse` injects `• Alice: switched to the OAuth refactor (task: …)`.
4. On the dashboard project page, `WorkUpdatesPanel` polls `work_updates` and shows the same line in a timeline.

## Error handling

- Bad/missing note → 422 (validation).
- Sensitive/private source session → 422 (write-gate), surfaced as a tool error to the agent.
- Unresolvable `task_id` → posted without the link (robust), not a failure.
- Dashboard/pulse reads degrade to empty on error (pulse is already fail-open).

## Testing

- **Model:** validations (body presence/length), associations, `recallable?` (nil session, recallable, sensitive).
- **Controller:** `create` (happy, sanitize, write-gate 422, bad task ignored), `index` (recent feed, gated, tenant isolation 404, auth), Pundit tier.
- **Service:** `ContextPulseService` `updates` source — included since `since`, excludes caller's own, gated by sensitive session, capped.
- **CLI:** `create_work_update` client (wiremock, `/api/v1` prefix), `share_update` tool path, `format_pulse` updates section.
- **Dashboard:** `WorkUpdatesPanel` render + empty state (RTL), MSW handler, hook.
- **OpenAPI:** document `POST` + `GET` `work_updates`; `openapi:validate STRICT=1`.

## Build order (for the plan)

1. `WorkUpdate` model + migration (+ `recallable?`).
2. `WorkUpdatesController` create + index + route + policy.
3. `ContextPulseService` `updates` source.
4. CLI `create_work_update` + `share_update` MCP tool.
5. CLI `format_pulse` updates section.
6. Dashboard client + hook + `WorkUpdatesPanel` + page wiring.
7. OpenAPI docs.

## Open questions / decisions locked

- Name: **`WorkUpdate`** (locked).
- **No** per-broadcast approval (locked).
- Free-text note + optional task link; no `kind` taxonomy (locked).
- Unresolvable `task_id` → post without link (decided, robust over strict).
- `MAX_BODY_CHARS` = 280 (starting value; tune in review).
