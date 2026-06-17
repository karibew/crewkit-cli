# D1 Item 1 — Convention Override-Resolution Loop (design)

> Status: **proposed** (design-first). Phase D1, item 1 in [PLAN.md](../../PLAN.md).
> Companion shipped work: abandoned-subagent sweep (#192), override analytics read path (#193).

## Problem

The CLI logs a `ConventionOverride` with `outcome: "overridden"` at *detection*
time — before the user/agent has even responded to the challenge. So:

1. **Outcomes are fiction.** Every detected deviation is recorded as "overridden"
   regardless of whether the user ultimately complied with the convention. The
   `accepted` enum value is never written.
2. **Resolved conventions are re-challenged every session.** There is no persisted
   "this was already decided" state; `ChallengeTracker` is in-memory only and
   nothing reads prior resolutions back.

"Closing the loop" = (A) detect the real outcome, and (B) stop re-challenging
conventions the user already explicitly overrode.

## Key facts (from code scout)

- **Write path already honors a CLI `outcome`** — `ObservabilityController#convention_overrides`
  does `outcome: sp[:outcome] || "overridden"`. Sending `"pending"` needs no
  controller change.
- **The transcript is persisted and sufficient for judgment.** `SessionTurn`
  stores `role`, `content_type`, `content`, and `raw_jsonl` (full Claude format),
  ordered by `turn_number` — capture-mode dependent (`full`/`structured` keep
  content; `minimal` does not). An LLM can read these to judge whether a flagged
  deviation was applied (overridden) or abandoned for the convention (accepted).
- **The analysis pipeline does NOT currently read raw turns.** `SessionAnalysisService`
  builds an *aggregated* context (summary, metrics, tool tallies) for the Batch
  Haiku pass — not the raw transcript. So resolution needs its own transcript read.
- **`ChallengeTracker`** (`cli/src/services/challenge_tracker.rs`) is in-memory
  only; no `resolve_challenge`, no persistence, no dedup. It already carries
  `convention_id`, `proposed_value`, `convention_value`, `session_id` per challenge.
- **No CLI path fetches prior resolutions.** Conventions are fetched via
  `playbooks/for_stacks` into `PlaybookCache`; there is no resolved-overrides fetch.
- **Read path now exists** (#193): `ConventionOverridesController` (`index`,
  `analytics`). The dedup read can extend it. Analytics already scopes to
  `.overridden`, so `pending` rows won't pollute it.

## Recommended architecture

**Detect outcome server-side at session-end via a dedicated transcript-reading
job** (chosen over mid-session tool-watching: more robust, no fragile tool-output
correlation, reuses persisted data, keeps the CLI thin).

### Flow

```
1. CLI detects deviation → logs ConventionOverride with outcome="pending"
   (was "overridden"). Challenge still injected/blocked as today.
2. Session ends → analysis pipeline runs (existing).
3. NEW: ResolveConventionOverridesJob, enqueued after a session's analysis
   completes, finds that session's pending overrides, reads its SessionTurns,
   and asks Haiku per override (or batched): "Given the convention and the
   transcript, was this ultimately accepted or overridden?" → updates
   outcome + resolved_at + resolved_by="llm_analysis".
   - Capture-mode = minimal (no content) → mark outcome="expired"
     (unresolvable), resolved_by="unresolvable".
4. NEW (dedup): At session start the CLI fetches the project's resolved
   *overridden* decisions, caches them, and skips re-challenging a
   convention+deviation already overridden (logs LogOnly instead).
```

### Why "pending"

`pending` is honest (outcome unknown until judged) and inert: the analytics from
#193 count `.overridden` only, so pending rows are invisible there until resolved.
After resolution the numbers become real.

## Proposed work (sequenced PRs)

### PR A — schema + resolution job (API)
- Migration: add `pending` to `ConventionOverride::OUTCOMES`; add `resolved_at`
  (datetime) and `resolved_by` (string) columns; index `(organization_id,
  convention_id, outcome)`.
- `ConventionResolutionService` — given a session, load pending overrides + that
  session's turns (respecting `sensitive`/privacy gating, mirroring
  `SessionVectorizerJob`), build a focused prompt, call `AnthropicService` (Haiku),
  parse `{outcome}` per override, update records. Batch all of a session's
  pending overrides into one prompt to bound cost.
- `ResolveConventionOverridesJob` — enqueued from `SessionAnalysisService#apply_analysis_to_session`
  (alongside `SessionVectorizerJob`) so it runs once analysis is done and the
  transcript is settled. Idempotent (only acts on `outcome="pending"`).
- Model scopes: `pending`, `resolved`.
- Tests: service (accepted/overridden/expired branches, minimal-capture fallback,
  privacy gating), job (idempotency, enqueue), model.

### PR B — dedup read endpoint (API)
- Extend `ConventionOverridesController` with `resolved` (or a `?resolved=true&convention_id[]=`
  filter on `index`): returns overridden decisions for a project/convention set,
  shaped for the CLI to consume cheaply.
- Policy already covers it (`index?`/Scope). Tests + (optional) OpenAPI.

### PR C — CLI: log pending + dedup (CLI)
- Change the detection log call to send `outcome: "pending"` (one-line change in
  the `ConventionOverrideRequest` builder).
- At session start, fetch resolved-overridden decisions for the project, store in
  `PlaybookCache` (or a sibling cache).
- In `handle_user_prompt_submit`, before the Haiku check, skip/downgrade to
  LogOnly when the convention+deviation is already overridden.
- Add `ChallengeTracker::resolve_challenge` + clear-on-session-end wiring.
- Tests: dedup skip, pending payload, tracker resolution.

## Open decisions (need your call before building)

1. **Resolution trigger** — dedicated `ResolveConventionOverridesJob` after analysis
   (recommended, decoupled, own transcript read + own Haiku call) **vs.** folding
   resolution into the existing Batch analysis prompt (one fewer LLM call, but
   couples two concerns and the batch prompt lacks raw turns today). *Recommend:
   dedicated job.*
2. **Dedup scope/key** — skip re-challenging keyed on `(project, convention_id,
   normalized proposed_value)` **vs.** `(project, convention_id)` regardless of the
   specific deviation **vs.** per-user instead of per-project. Narrower (with
   proposed_value) is safer but re-challenges novel deviations of the same
   convention. *Recommend: `(project, convention_id, normalized proposed_value)`.*
3. **Dedup horizon** — does an override decision persist forever, or expire (e.g.
   re-challenge after 30/90 days or after the convention version changes)? A
   convention edit should arguably re-open challenges. *Recommend: invalidate on
   convention version change; otherwise persist.*
4. **Minimal-capture sessions** — mark unresolved overrides `expired`, or leave
   `pending` for a later retry, or fall back to the old "assume overridden"?
   *Recommend: `expired` (honest: we can't know).*

## Risks

- **LLM judgment accuracy.** Mitigate: low temperature, structured output, and the
  conservative default (if the transcript doesn't clearly show compliance, treat as
  overridden — matches today's behavior, so no regression).
- **Cost.** One extra Haiku call per session-with-pending-overrides; batch all of a
  session's pending overrides into a single prompt.
- **Privacy.** Resolution reads transcript content → must honor the same
  `sensitive`/private gating as `SessionVectorizerJob`. Sessions that can't be read
  resolve to `expired`.
- **Analytics interaction.** #193 counts `.overridden`; pending/expired are excluded
  by construction, so totals only firm up as resolution lands. `needs_review` will
  become accurate (real overrides only) rather than inflated by detection-time logs.
```
