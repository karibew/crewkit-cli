# Cross-Session Awareness — Design Spec

**Date:** 2026-06-22
**Branch context:** builds on `feat/session-save-states`
**Status:** Approved design, pre-implementation

---

## Problem

Save states and active-work telemetry already exist, but they are **pull-only and
resolved once per session** — effectively at session start (`load_state` /
`get_active_work` fire when the dev asks, usually early). A session a developer
leaves open between jobs never refreshes: it is blind to the save states and
teammate activity produced *after* it started.

Two distinct gaps follow:

1. **Humans** have no out-of-band feed of "what the team did / is doing" — they
   must open the dashboard or re-ask in a session.
2. **Open agent sessions** go stale and reason with outdated cross-agent context.

## Goals

- Deliver the existing **team digest** to a designated Slack channel when it is
  published (humans).
- Give a long-lived open session a way to pick up **fresh cross-agent context**
  automatically, without the developer manually re-asking (agents).
- Reuse existing primitives: `TeamDigestService`, the `active_work` endpoint,
  Phase B Slack plumbing, the hook system, the save-state privacy gate.

## Non-goals

- **Claude Code Artifacts are not built.** Verified against official docs
  (`code.claude.com/docs/en/artifacts.md`, 2026-06): they have no MCP/hook/
  programmatic-creation path, are off by default in Agent SDK/MCP/CI contexts,
  enforce a strict CSP (no live API calls at view time), require an explicit URL
  to update, and are org-policy-gated (disabled under ZDR/HIPAA/CMEK). They are a
  static snapshot surface, not a mechanism for session-to-session context. Noted
  as optional future polish only.
- No new LLM summarization — Phase 1 ships the digest text already generated.
- No push-into-an-open-session mechanism (Claude Code has none); Phase 2 is
  hook-triggered pull.

---

## Architecture

```
┌─ Phase 1 (humans) ───────────────────────────────────────────────┐
GenerateTeamDigestsJob → TeamDigestService → SaveState(published)
                                  └→ PostDigestToSlackJob
                                          → SlackSenderService → #channel
└──────────────────────────────────────────────────────────────────┘

┌─ Phase 2 (sessions) ─────────────────────────────────────────────┐
open session → UserPromptSubmit hook → `crewkit context-pulse`
   → GET /:org_id/projects/:project_id/context_pulse?since=<watermark>
   → { active-work delta, team states published since watermark }   (privacy-gated)
   → inject compact block into context; advance watermark
└──────────────────────────────────────────────────────────────────┘
```

---

## Phase 1 — Slack digest delivery

Ships first; lowest risk; serves humans.

### Trigger
When `TeamDigestService.generate` upserts a team SaveState **and the state is
published** (`published_at` present — drafts held under `require_review` must
**not** be posted), enqueue `PostDigestToSlackJob`.

- Publish path lives in `TeamDigestService` (`api/app/services/team_digest_service.rb`).
- Enqueue only on the transition to published, not on every re-run — guard on the
  state being newly published this run (avoid duplicate posts when the hourly job
  re-evaluates an already-published digest).

### Channel binding
- New **project-level** setting `digest_slack_channel_id`.
- Falls back to an **org-level default** added to `daily_summary_settings`
  (alongside `enabled`, `timezone`, `end_of_day_hour`, `require_review`,
  `weekend_days`).
- **Opt-in:** nothing posts until a channel is configured. No global default channel.

### Delivery
- `PostDigestToSlackJob` (`api/app/jobs/`) loads the published SaveState, formats
  via `SlackNotifier`, posts via `SlackSenderService`.
- Content is the existing `handoff` text + a header line (project, digest date,
  contributor count from `metadata.contributors`). No new generation.
- Resolve target channel: project setting → org default; skip (log) if neither set.

### Privacy
- The digest is already privacy-gated per session inside `TeamDigestService`
  (`SaveState.session_recallable?`: `sensitive?` + private-repo). The job sends
  the already-gated text — no new gating logic.
- **Egress note:** wiring a channel means the gated summary leaves for that Slack
  workspace. Opt-in per project keeps this an explicit admin choice — relevant for
  client isolation / Loi 25. The chosen workspace/channel is the admin's
  responsibility to scope to the right client.

### Optional (default OFF, not in v1)
A periodic "active now" ping to the channel. Omitted from v1 to avoid channel
noise; revisit if requested.

### Testing
- Job test: posts on publish; **does not** post for a `require_review` draft;
  no duplicate post on re-run of an already-published digest.
- Channel resolution: project → org fallback → skip when unset.
- Formatting: handoff + header render; contributor count from metadata.

---

## Phase 2 — Hook-driven context pulse

Fixes the stale open session.

### New endpoint
`GET /:org_id/projects/:project_id/context_pulse?since=<ISO8601>`

- Returns, in one round-trip:
  - **active-work delta** since `since` (reuse the active-work query;
    `include_self=false`), and
  - **team save states published since `since`** (daily/weekend).
- Honors the same privacy gate as active-work and save-state recall.
- Default `since` if absent: 30 min ago (matches active-work default).
- Response is compact and intended for direct rendering by the CLI command.

### New CLI command — `crewkit context-pulse`
- Reads a **per-session watermark** file under `.crewkit/` keyed by
  `CREWKIT_LLM_SESSION_ID` (last-injection timestamp).
- Calls `context_pulse?since=<watermark>` via the existing `api_client`
  (`cli/src/services/api_client.rs`).
- If the response is empty → prints nothing.
- If non-empty → prints a compact injection block, e.g.:
  ```
  📋 Since your last check:
  • Alice — started "OAuth fix" in api (4m ago)
  • Team digest updated for 2026-06-21 (3 contributors)
  ```
  and **advances the watermark**.
- Logic lives in Rust (unit + integration tested), not a shell script.

### Hook wiring
- A **`UserPromptSubmit`** hook, synced via the resource system, that invokes
  `crewkit context-pulse` and passes its stdout into context.
- **Fail-open:** any error (API down, auth, timeout) → emit nothing, exit 0,
  never block the developer's prompt.
- **Throttle:** a minimum interval (default ~10 min) enforced via the watermark so
  rapid successive prompts don't re-inject or run up cost. Injection only fires
  when (a) the throttle window has elapsed **and** (b) there is new content.

### Privacy
- `include_self=false`; same gate as Phase 1. A session never sees gated content.

### Testing
- API: `context_pulse` returns delta correctly for a `since`; honors privacy gate;
  excludes self; empty when nothing new.
- CLI: watermark read/write; empty response prints nothing; non-empty prints block
  and advances watermark; throttle suppresses within window; **fail-open** on API
  error (wiremock 500 / unreachable → exit 0, no output).

---

## Key decisions

| Decision | Choice | Rationale |
|---|---|---|
| Slack channel scope | Per-project, org fallback | Projects map to clients; avoids one global channel leaking across engagements |
| Draft digests to Slack | No | Respect `require_review`; never leak unreviewed content |
| CC Artifacts | Not built | No programmatic/MCP/hook path; static; policy-gated. Optional future polish only |
| Phase 2 transport | Hook-triggered pull (not webhook/push) | Claude Code cannot receive a push into an open session |
| Pulse round-trips | One new endpoint, not two calls | Keep the per-prompt hook cheap |
| Throttle default | ~10 min | Balance freshness vs cost/noise; tune after dogfood |

## Rollout
1. Phase 1 (API job + settings + Slack post) — independently shippable.
2. Phase 2 (endpoint + CLI command + hook) — depends on nothing in Phase 1.
3. Dogfood the throttle interval; revisit the optional "active now" ping only if asked.

## Open questions
- Throttle interval: 10 min is a starting default — confirm after dogfooding.
- Whether the org default channel is worth shipping in v1 or per-project only.
