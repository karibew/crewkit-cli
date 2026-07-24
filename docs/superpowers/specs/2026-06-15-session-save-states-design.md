# Session Save States — Design Spec

**Date:** 2026-06-15 · **Revised:** 2026-06-16 (v2)
**Status:** Revised design (v2) — pending implementation plan
**Scope:** api · cli · dashboard
**Pillar:** Collaborate (session-to-session continuity)

> **v2 change (2026-06-16):** Replaces the named-checkpoint model (up to 3 keyed
> "save states" per project) with **two automatic-recall layers**: one **manual,
> rolling personal** state per user×project, and one **automated, rolling team
> daily** digest per project. Recall is no longer by name — both layers are
> injected automatically at session start. See history at the bottom.

## Problem

A new session starts cold. To continue prior work, the agent re-reads commits or
relies on fuzzy semantic recall. Two distinct gaps:

1. **My own thread** — no deterministic "this is where *I* left off" that survives
   into my next session without me hunting for it.
2. **The team's movement** — no compact, reliable picture of **what everyone else
   changed** in the project recently, so concurrent work collides or surprises.

## Novelty vs. existing substrate

crewkit already vectorizes analyzed sessions into searchable `Artifact`s
(`source: "session"`), exposes a `scope: "sessions"` hybrid search, and an MCP
tool `find_prior_work()`. That recall is **automatic, semantic, fuzzy, per-session**.

This feature adds the missing layer: **two curated, deterministic, auto-injected
context files** — one personal, one team — that a new session opens with.

| Capability | Status |
|---|---|
| Session → searchable artifact, privacy-gated | already built |
| `scope: "sessions"` semantic search + `find_prior_work()` | already built |
| Parent/thread linking across `/clear --continue` | already built |
| Per-session end summary + deep analysis (`/sessions/:id/end`) | already built |
| **Rolling personal save state (1 per user×project), manual** | NEW |
| **Rolling team daily digest (1 per project), automated at EOD** | NEW |
| **Separate rolling weekend digest (keeps Friday intact)** | NEW |
| **Automatic two-layer injection at session start** | NEW |

## Goals

- **Personal state** — a user asks the agent to save "where I left off." It's a
  **compact curated handoff** (not a transcript): what we were doing, key
  decisions, open threads, files in play. **One slot per user×project**; saving
  again **overwrites** it. It **persists across idle days** — taking a day off
  never leaves you with a blank state.
- **Team daily digest** — once per day at a configurable **end-of-day** hour
  (default **00:00 America/New_York**), crewkit compiles a **brief per-user
  summary of what each person/agent did** in the project that day, into **one
  rolling digest** per project (today's overwrites yesterday's).
- **Weekend digest** — weekend EOD runs write a **separate** rolling
  `team_weekend` digest instead of overwriting the weekday one, so **Friday's
  summary stays intact** for Monday morning while still capturing any work done
  Saturday/Sunday. Monday injects both; the weekend layer auto-suppresses once a
  newer weekday digest exists.
- **Automatic recall** — a new session in a project opens with **both** layers
  injected: *my* personal state + the project's latest team digest. Deep context
  on my own work, plus awareness of changes landed alongside it.
- Personal saves are **manual + human-confirmed** (CLI). The team digest is
  **automated**, with **optional review/edit by a manager in the dashboard**.
- **Publish gate (decided 2026-06-16):** digests **auto-publish by default** so
  mornings are useful even when no manager is online. An org setting
  `require_review` (default off) flips this to **hold-until-reviewed**: the digest
  is generated as an unpublished draft and only becomes injectable once a manager
  edits/publishes it. Recall injects **published** team digests only.

## Non-goals (v1)

- Named/keyed checkpoints, multiple personal slots, or recall-by-name (dropped
  from v1 — superseded by the rolling model).
- Semantic discovery of save states (`find_prior_work` stays the fuzzy path).
- File attachments (and therefore no AV/ClamAV — see Security). Revisit if added.
- Per-user team digests — the digest is one shared, project-scoped row.

## Data model

New table `save_states` with a `kind` discriminator:

| Column | Type | Notes |
|---|---|---|
| `external_id` | uuid | API addressing (crewkit convention) |
| `organization_id` | FK | denormalized tenant scoping |
| `project_id` | FK | save states are project-scoped |
| `kind` | string (enum) | `personal` \| `team_daily` \| `team_weekend` |
| `user_id` | FK → users, nullable | set for `personal` (owner); null for `team_*` |
| `created_by_id` | FK → users, nullable | last human to save/edit (attribution); null when system-generated |
| `reviewed_by_id` | FK → users, nullable | `team_*` only — lead/admin who reviewed |
| `reviewed_at` | datetime, nullable | `team_*` only |
| `published_at` | datetime, nullable | `team_*` only — when the digest became injectable. Auto-set on generation unless the org's `require_review` is on, in which case a manager edit/publish sets it. `personal` rows ignore this (always available to their owner). |
| `source_llm_session_id` | FK → llm_sessions, nullable, `on_delete: nullify` | `personal` provenance |
| `digest_date` | date, nullable | `team_*` only — day summarized (`team_weekend`: the weekend's end date) |
| `period_start` | date, nullable | `team_weekend` only — Friday-EOD boundary the weekend accrues from |
| `handoff` | text | curated markdown, stored **inline** |
| `metadata` | jsonb, default `{}` | `team_*`: source session ids / per-user contributors |
| `created_at` / `updated_at` | datetime | `updated_at` reflects last refresh |

**Constraints / rules:**

- **Unique index `(project_id, kind, user_id)`** with `user_id` null for
  `team_*` — gives exactly **one `personal` row per user×project**, **one
  `team_daily` row per project**, and **one `team_weekend` row per project**. All
  saves are **upserts** (overwrite-in-place).
- **`handoff`** validated present and size-capped (~16 KB) to keep injection light.
- **No DO KB / embedding** — recall is `find_by(project:, kind:, user:)` and the
  content is injected directly. Deterministic, cheap, low-token.
- **Privacy gate (`personal`):** save blocked when `source_llm_session.sensitive?`
  (or private-repo), mirroring `SessionVectorizationService#vectorizable?`.
- **Privacy gate (`team_*`):** the digest only summarizes sessions that are
  themselves vectorizable/non-sensitive — sensitive sessions are excluded from
  the roll-up, never quoted.

Pundit scopes everything to `current_organization` + project membership. Any
member may read/recall; a member may write/delete **their own** `personal` row;
**`team_*` review/edit/delete** requires admin or project-lead.

## Personal save flow (manual + confirm)

```
In session, user: "save where I left off"
1. Agent authors the curated handoff from its own context.
2. Agent calls MCP tool  save_state(handoff, title?).
3. CLI renders the draft in a TUI confirm overlay  ← confirm-as-context gate
   [Confirm] [Edit] [Cancel] — nothing stored until a human OKs it.
4. On confirm, CLI PUTs → PUT /:org/projects/:id/save_states/personal
      { handoff, source_session_id }.
5. API validates (privacy gate · size), sanitizes, upserts the user's row.
6. Tool returns success → agent: "Saved your personal state for this project."
```

- **The agent authors; the CLI gates; the server enforces.** The human-confirm
  step lives at the CLI boundary (reuse the existing TUI `pending_confirm` /
  `ConfirmAction` overlay). The API re-validates independently — the CLI is
  convenience, not the trust boundary.
- Overwrite is the norm: re-saving replaces the single slot; the prior state is
  gone (no history in v1).

## Team daily digest flow (automated + optional review)

```
EOD job (per project, at configured hour; default 00:00 ET):
1. Recurring solid_queue job enumerates the project's sessions for the period.
2. Excludes sensitive/private sessions (privacy gate).
3. Summarizes from EXISTING per-session summaries/deep analysis (not raw
   transcripts) → a brief "what each person/agent did" digest.
4. Routes by the day just ended (in the configured timezone):
     • WEEKDAY (Mon–Fri) → upsert the team_daily row (overwrites yesterday).
     • WEEKEND (Sat/Sun) → upsert the team_weekend row; team_daily is UNTOUCHED,
       so Friday's summary survives. Sunday's run re-summarizes the whole
       weekend (period_start = last Friday EOD), overwriting Saturday's.
5. Sanitizes, records digest_date (+ period_start for weekend) + contributors.
6. Publish: if org.require_review is OFF (default) → published_at = now (injectable
   immediately). If ON → published_at = nil (draft; not injected until reviewed).

Optional review (dashboard, manager):
- Each digest surfaces in a Save States panel; a manager can edit, publish, or delete it.
- Editing/publishing sets reviewed_by/reviewed_at and published_at, attributing the change.
```

**Weekend rationale:** most people are off Sat/Sun. If the weekend EOD overwrote
`team_daily`, Monday morning would inject a near-empty weekend summary and lose
Friday's context — the opposite of useful. Writing weekends to a separate
`team_weekend` row keeps Friday's `team_daily` intact while still capturing
anyone who *did* work over the weekend. "Weekend" defaults to **Sat/Sun** but is
**configurable** alongside the EOD setting (e.g. an org that works Sun–Thu).

> **DECISION (2026-06-16):** digests **auto-publish by default** (useful mornings
> even with no manager online). The org setting **`require_review`** (default off)
> flips to **hold-until-reviewed** — the digest is generated as an unpublished
> draft and a manager edit/publish makes it injectable. Recall injects published
> team digests only; `personal` rows are unaffected.

- **Cost control:** the digest is built by summarizing summaries we already
  generate on `/sessions/:id/end`, once per day per project — bounded and cheap.
- **No sessions that day** → skip; leave the prior digest in place (no blank).

## Recall flow (automatic injection)

```
Startup:    crewkit code            (in a project)
              → CLI GETs the recall bundle for (project, current user)
              → injects the applicable layers into the opening context:
                  • personal    (mine, this project)
                  • team_daily   (this project, latest weekday digest)
                  • team_weekend (ONLY if newer than team_daily — see rule)
In-session: agent calls MCP load_state("personal" | "team")
              → returns that layer to re-read mid-conversation.
Bundle:     GET /:org/projects/:id/save_states/recall   (resolved layers)
Opt-out:    crewkit code --no-context  skips injection for this run.
```

**Weekend injection rule:** include `team_weekend` **only when
`team_weekend.digest_date > team_daily.digest_date`** — i.e. the weekend is more
recent than the last weekday summarized. So **Monday** injects Friday's
`team_daily` **plus** the weekend digest; by **Tuesday** the new Monday
`team_daily` is newer, so the (now-stale) weekend layer is suppressed
automatically. The server resolves this in the recall bundle; the CLI just
injects what it's handed. **Unpublished team digests are never injected** — when
`require_review` is on, a draft digest stays out of the recall bundle until a
manager publishes it.

**Injection format:** each layer is injected as a clearly-labeled, nonce-fenced
**reference** block, framed as untrusted context, never as instructions —
e.g. `── personal save state (Arthur, 2026-06-16) ──`,
`── team daily digest (crewkit, 2026-06-13 Fri) ──`,
`── team weekend digest (crewkit, 2026-06-14–15) ──`. Absent layers are omitted.

## Security

The threat model is **prompt injection**, not malware. A handoff/digest is
injected into a future session and may summarize a session that ingested
untrusted content (Epic B inbound, web, files) — so it can carry a smuggled
payload forward. The **team digest amplifies blast radius**: it folds in every
user's work and injects into **every member's** next session, so one poisoned
session could otherwise reach the whole project.

Controls (reusing the Epic B confirm-as-context model):

- **`ContentSanitizer.clean` at the write boundary** (server-side, on every save
  and on digest generation) — strips invisible/bidi/control chars (ASCII
  smuggling) before persisting.
- **Sanitize again at the read boundary** (on recall/inject) — defense-in-depth
  for rows predating a sanitizer update, same pattern as `artifacts_controller`.
- **Fenced injection** — labeled, nonce-fenced reference block so a smuggled
  "ignore previous instructions" cannot pose as a directive.
- **Human-in-the-loop, per layer:**
  - `personal` — **confirm-at-save** in the CLI; a person reads it before storage.
  - `team_daily` / `team_weekend` — **optional lead review/edit** in the
    dashboard, *and* both are summarized from already-sanitized session
    summaries, never raw transcript.
- **Privacy gate on both** — sensitive/private sessions never feed either layer.

**AV scanning deferred:** ClamAV (Epic B) scans uploaded *files*; a save state is
agent-authored inline *text* with no file, so a virus scanner is security theater
here. If file attachments are ever added, route them through the existing ClamAV
quarantine. Run the `security-expert` agent over the implementation diff (as for
Epic B); add AV only if it surfaces a concrete risk.

## Management surfaces & API

`personal` create/update is agent-only (a handoff needs a session). `team_daily`
is job-generated. Everything else is manageable from CLI and dashboard.

- **CLI:** `crewkit save-state show [personal|team]`, `crewkit save-state rm`
  (deletes your personal row). The recall bundle is auto-injected by
  `crewkit code`.
- **Dashboard:** a **Save States** panel on the project page — view my personal
  state, view the team daily + weekend digests, and (admin/lead) **edit or
  delete** either digest.

| Verb | Path | Purpose |
|---|---|---|
| PUT | `/:org/projects/:id/save_states/personal` | upsert caller's personal state |
| GET | `/:org/projects/:id/save_states/recall` | resolved injection bundle (personal + weekday/weekend per rule) |
| GET | `/:org/projects/:id/save_states` | list visible states (mine + team_daily + team_weekend) |
| PATCH | `/:org/projects/:id/save_states/:kind` | lead edit of a team digest (`team_daily` \| `team_weekend`) |
| DELETE | `/:org/projects/:id/save_states/:external_id` | delete (own personal, or a team digest if lead) |

The team digests are produced by a **recurring `solid_queue` job** keyed off an
org/project **end-of-day setting** (default `00:00 America/New_York`) plus a
**weekend-days setting** (default Sat/Sun), not an API verb. OpenAPI spec +
regenerated dashboard types updated alongside (Contract Validation CI).

## Edge cases

- Re-saving personal → overwrite-in-place (refresh `handoff`/`updated_at`/`created_by`).
- Sensitive source session → `422` (privacy gate) on personal save.
- Source session later deleted → personal state survives (inline content; FK nullify).
- Project deleted → all its save states cascade.
- `handoff` empty or oversize (>~16 KB) → `422`.
- EOD job, no eligible sessions → no-op; prior digest retained (no blanking).
- EOD job, only sensitive sessions → no-op (nothing eligible to summarize).
- User with no personal state in a project → recall returns team layers only.
- DST / timezone change of the EOD setting → next run honors the new local hour.
- **Weekend EOD never overwrites `team_daily`** → Friday's digest survives to Monday.
- **Empty weekend** (no work Sat/Sun) → no `team_weekend` row written; Monday
  injects Friday's `team_daily` alone.
- **Stale weekend digest** (it's now Tuesday+) → suppressed by the recall rule
  (`team_daily` is newer); never deleted, just not injected.
- **Weekend-days reconfigured** (e.g. Sun–Thu workweek) → the job classifies the
  ended day against the current setting; the "weekend" rows track the off-days.

## Testing

- **Model:** `(project, kind, user_id)` uniqueness incl. null-user for team_daily;
  personal overwrite/upsert; size cap; privacy gate; write-boundary sanitization;
  `kind`-conditional column rules (user_id required for personal, null for team).
- **Request:** personal upsert; recall bundle (both layers, missing layers);
  team_daily lead-only edit/delete; **tenant isolation** (Pundit scope); privacy
  refusal; plus a **sanitizer sentinel** (invisible chars stripped at write *and*
  read), mirroring Epic B.
- **Job:** EOD digest builds from session summaries; excludes sensitive sessions;
  overwrites the rolling row; no-op on empty day; honors the configured hour/tz;
  **weekday run writes `team_daily`, weekend run writes `team_weekend` and leaves
  `team_daily` untouched**; Sunday run re-summarizes the full weekend.
- **Recall resolution:** weekend layer included only when newer than the weekday
  layer (Monday → both; Tuesday → weekday only); personal-only and team-only and
  empty-bundle cases.
- **CLI:** `save-state show/rm`; auto-injection of the recall bundle on
  `crewkit code`; `--no-context` opt-out; `load_state` tool.
- **Dashboard:** Save States panel — view personal, view/edit/delete team digest,
  permission gating (MSW handlers + component test).

## Future (out of scope for v1)

- Optional named/manual checkpoints alongside the rolling layers (revive v1 keys).
- History/versions for the personal slot (currently overwrite-only).
- Semantic discoverability of save states (let `find_prior_work` surface them).
- File attachments (+ ClamAV quarantine for those files).
- Per-user team digests; "hold until reviewed" digest gating as a setting.

## History

- **v1 (2026-06-15, approved):** up to 3 user-named, deterministic checkpoints per
  project, recalled by exact key (`--resume <key>`), shared + attributed, manual +
  human-confirmed. Superseded by v2.
- **v2 (2026-06-16):** rolling personal (manual, 1/user×project) + rolling team
  daily digest (automated EOD, 1/project, optional lead review) + separate rolling
  weekend digest (keeps Friday intact, auto-suppressed once stale), all
  auto-injected at session start. Keys/cap/`--resume` dropped.
