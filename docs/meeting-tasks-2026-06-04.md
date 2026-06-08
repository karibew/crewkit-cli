# Task List — Dev Sync 2026-06-04

> Generated from the meeting transcript of 2026-06-04 (Arthur Genest, Félix-Antoine Paradis, Rémi Gagnon).
> This is the **dogfooding proof-of-concept**: crewkit consuming its own meeting to produce its own backlog.
> Workstreams map to the [PLAN.md](../PLAN.md) phases where applicable.

## Legend

- `[ ]` todo · `[~]` in progress · `[x]` done
- **Owner** in **bold** · maps phase in _italics_

---

## Action items (committed in the meeting)

- [ ] **Arthur** — Install **GitHub CLI** (`gh`) locally + `gh auth login`, so crewkit's PR creation can shell out to it instead of taking screenshots. _(blocks A1)_
- [x] **Arthur** — ~~Create Rust ticket: session keeps the initial branch instead of tracking the newly selected branch.~~ **Fixed this session** — commit `f79b7fa` re-collects git stats every 2s in the TUI loop. Open the ticket only to record the fix, or skip. _(A3)_
- [~] **Arthur** — Build this task list from the meeting transcript in crewkit + run a functional test of the flow. _(this document — dogfooding)_
- [ ] **Félix-Antoine** — Configure **SendGrid** receiving domains for inbound email; add `staging` and `dev` domains for testing. _(D4)_
- [ ] **Arthur** — Verify domain: check the Crewkit Operations docs ("Crooked Operations" in transcript) for valid domain parameters. _(D5, blocks D4)_

---

## Epic A — Dev-flow polish _(Pillar: Observe)_

- [ ] **A1** — Wire crewkit PR creation to `gh` CLI (auto-generated PR summary). _(depends on gh install)_
- [x] **A2** — Adopt `Plan.md` (ex-`Project.md`) as the convention for structuring task lists in Claude. _Validated in meeting; this file follows it._
- [x] **A3** — Fix TUI branch-switch tracking (`f79b7fa`). Sidebar now reflects `git checkout` mid-session.
- [ ] **A4** — Confirm crewkit summaries remain faithful after the branch-refresh change (no regression). _Quick manual check._

## Epic B — Context ingestion _(Pillar: Ingest — extends Artifact + DO KB)_

- [ ] **B1** — Define the ingestion funnel: transcript / email / Slack thread → `Artifact` → chunk → vectorize → project-searchable.
- [ ] **B2** — Build the **project-routing** step: auto-guess target project from participants + content similarity to existing artifacts.
- [ ] **B3** — Dashboard **"Unassigned / Inbox" tray**: human confirms or re-assigns the auto-guessed project for vague-titled meetings.
- [ ] **B4** — Handle low-confidence routing gracefully (default to tray, never silently misfile).

## Epic C — Meeting-bot agent _(Pillar: Ingest + Collaborate)_

- [ ] **C1** — Integrate with an existing transcription source (Gemini / Fireflies / Otter) via API/webhook — do **not** build transcription in-house.
- [ ] **C2** — On finished transcript: create an `Artifact` and route it through the Epic B funnel.
- [ ] **C3** — Spike: can the agent auto-join scheduled calls, or is post-meeting webhook ingestion sufficient for v1? _(prefer the simpler webhook path first)_

## Epic D — Agent identities (email + Slack) _(Pillar: Collaborate — Phase B, in-flight)_ ⭐

> Stakeholder-favorite feature — protect across refactors.

- [ ] **D1** — SendGrid **Inbound Parse** → webhook → crewkit, routing inbound email to the right agent.
- [ ] **D2** — Per-agent catch-all addressing (e.g. `agent-name@agents.crewkit.io`) — no real mailbox provisioned per user.
- [ ] **D3** — **VCF contact-card** endpoint on agent creation → import to Outlook / Google Contacts → invite agent to meetings.
- [ ] **D4** — SendGrid receiving-domain setup: `dev`, `staging`, `production`. _(Félix)_
- [ ] **D5** — Verify valid domain parameters in Crewkit Operations docs. _(Arthur, blocks D4)_

## Epic E — Local-first processing _(cross-cutting cost lever)_

- [ ] **E1** — Design opt-in "local mode": delegate background analysis/synthesis to the user's authenticated local CLI (their compute + their tokens) instead of cloud.
- [ ] **E2** — Keep a cloud fallback for headless/cron runs with no local CLI attached.
- [ ] **E3** — Identify which jobs are latency-tolerant enough to run locally (summaries, vectorization prep, contradiction scans).

## Epic F — Institutional memory & contradiction detection _(Pillar: Govern — Phase C/D)_

> **F3 is now the headline next step** — F1/F2 are implemented (`feat/session-vectorizer`), so the searchable substrate exists. Contradiction detection is the layer the meeting actually asked for.

- [ ] **F3** — Contradiction pass: on each new decision/artifact, semantic-search prior decisions; flag conflicts with established business rules. _(Phase D, sits on top of F1 — next PR)_ ⭐
- [x] **F1** — Vectorize analyzed conversations for project search. _Implemented on `feat/session-vectorizer`: `SessionVectorizationService` + `SessionVectorizerJob` + recurring scan, auto-triggered on analysis completion. Pending merge._
- [x] **F2** — Single queryable knowledge base with `scope: "sessions"` search + "Find prior work" dashboard card. _Sessions flow in today; meetings/email/Slack arrive via Epics C/D (search layer already supports all sources)._
- [ ] **F4** — Spike: index/share Claude's local user-directory memories within project context.

### Landing `feat/session-vectorizer` (in progress 2026-06-04)

- [x] Merge current `main` into the branch (clean, no conflicts — brings in Slack OAuth UI + CLI branch fix)
- [ ] Run migration `20260603000001_add_llm_session_to_artifacts` + verify `schema.rb`
- [ ] Run API tests (`PARALLEL_WORKERS=1`)
- [ ] Run dashboard tests (`npm test`)
- [ ] Open PR + confirm CI green

---

## Sequencing notes

1. **Unblock D first** (D5 → D4) — SendGrid domains gate the most-wanted feature.
2. **F1 done → F3 next** — vectorization substrate is built and landing; contradiction detection (F3) rides on it and is the highest-aligned follow-on.
3. **B before C** — the ingestion funnel + routing tray must exist before the meeting bot has somewhere to file transcripts.
4. **A1** is a quick local unblock; **A3/A4** essentially done.
