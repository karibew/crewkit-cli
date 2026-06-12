# Task List — Dev Sync 2026-06-04

> Generated from the meeting transcript of 2026-06-04 (Arthur Genest, Félix-Antoine Paradis, Rémi Gagnon).
> This is the **dogfooding proof-of-concept**: crewkit consuming its own meeting to produce its own backlog.
> Workstreams map to the [PLAN.md](../PLAN.md) phases where applicable.

## Legend

- `[ ]` todo · `[~]` in progress · `[x]` done
- **Owner** in **bold** · maps phase in _italics_

---

## Action items (committed in the meeting)

- [x] **Arthur** — ~~Install **GitHub CLI** (`gh`) locally + `gh auth login`~~ **Done** — `gh 2.93.0` installed + authed. A1 (wiring PR creation to it) now unblocked. _(blocks A1)_
- [x] **Arthur** — ~~Create Rust ticket: session keeps the initial branch instead of tracking the newly selected branch.~~ **Fixed this session** — commit `f79b7fa` re-collects git stats every 2s in the TUI loop. Open the ticket only to record the fix, or skip. _(A3)_
- [~] **Arthur** — Build this task list from the meeting transcript in crewkit + run a functional test of the flow. _(this document — dogfooding)_
- [x] **Félix-Antoine** — ~~Configure **SendGrid** receiving domains for inbound email; add `staging` and `dev` domains for testing.~~ **Done** — three Inbound Parse domains configured + wired via env-aware `AgentIdentity.email_domain` (`f47b761`). _(D4)_
- [x] **Arthur** — ~~Verify domain: check the Crewkit Operations docs ("Crooked Operations" in transcript) for valid domain parameters.~~ **Done** — verified domains provided by Félix; example addresses confirmed against local-part rules. _(D5)_

---

## Epic A — Dev-flow polish _(Pillar: Observe)_

- [x] **A1** — ~~Wire crewkit PR creation to `gh` CLI~~ **Done / satisfied** — the meeting's motivation was the *local* dev flow (PRs "instead of screenshots"); with `gh` now installed + authed locally, Claude Code creates PRs via `gh pr create` directly. The **cloud-job** PR path intentionally stays on `CloudJobPrService` (Octokit + the user's OAuth token) — server-side Octokit is the right tool there; shelling `gh` into the LXC container would add a dependency + token-handling + a fallback path for no real gain. _(decision: 2026-06-09)_
- [x] **A2** — Adopt `Plan.md` (ex-`Project.md`) as the convention for structuring task lists in Claude. _Validated in meeting; this file follows it._
- [x] **A3** — Fix TUI branch-switch tracking (`f79b7fa`). Sidebar now reflects `git checkout` mid-session.
- [ ] **A4** — Confirm crewkit summaries remain faithful after the branch-refresh change (no regression). _Quick manual check._

## Epic B — Context ingestion _(Pillar: Ingest — extends Artifact + DO KB)_

- [~] **B1** — Define the ingestion funnel: transcript / email / Slack thread → `Artifact` → chunk → vectorize → project-searchable. _Routing + staging legs landed (B2–B4); transcript/email/Slack source adapters still arrive via Epics C/D._
- [x] **B2** — ~~Build the **project-routing** step~~ **Done** — `ProjectRoutingService` auto-guesses target project; agent webhook inbound routed through it (`81c50d4`, `50d7205`).
- [x] **B3** — ~~Dashboard **"Unassigned / Inbox" tray**~~ **Done** — `inbound_messages` staging + Inbox tray (`49d56f8`, `50d7205`).
- [x] **B4** — ~~Handle low-confidence routing gracefully~~ **Done** — low-confidence inbound defaults to the Inbox tray for human triage; never silently misfiled.

## Epic C — Meeting-bot agent _(Pillar: Ingest + Collaborate)_

- [~] **C1** — Integrate with an existing transcription source (Gemini / Fireflies / Otter) via API/webhook — do **not** build transcription in-house. _Seam + Fireflies fetch-by-id client scaffolded (`TranscriptProviders::Fireflies::Client`) but gated on `FIREFLIES_API_KEY` and unverified against the live API — provision the key, then confirm the GraphQL field mapping._
- [x] **C2** — ~~On finished transcript: create an `Artifact` and route it through the Epic B funnel.~~ **Done** — `POST /webhooks/transcripts/:provider` → `TranscriptProviders` adapter (HMAC-verified) → `MeetingTranscriptIngestionService` resolves the org from participants, dedups on the provider meeting id, routes via `ProjectRoutingService` (auto-file / Inbox triage), and enqueues `ProcessArtifactJob` for vectorization. Fireflies inline-payload path fully tested; live fetch is C1.
- [x] **C3** — ~~Spike: auto-join vs post-meeting webhook~~ **Spike done** — see [spike-c3-meeting-bot-ingestion.md](./spike-c3-meeting-bot-ingestion.md). **Decision: post-meeting webhook for v1** (Option A), built behind a `TranscriptProvider` seam so an auto-join agent (Option B) or a 3rd-party like Fireflies plugs in later without rework. Reuses the shipped Epic B funnel (route → Inbox → vectorize).

## Epic D — Agent identities (email + Slack) _(Pillar: Collaborate — Phase B, in-flight)_ ⭐

> Stakeholder-favorite feature — protect across refactors.

- [x] **D1** — ~~SendGrid **Inbound Parse** → webhook → crewkit~~ **Done** — `POST /webhooks/sendgrid/inbound` → `AgentInboundRoutingService` routes inbound email to the matching agent.
- [x] **D2** — ~~Per-agent catch-all addressing~~ **Done** — `AgentIdentity#email_address` → `{local}@agent.crewkit.ai`, unique active index, no per-user mailbox.
- [x] **D3** — ~~**vCard contact-card** endpoint~~ **Done (this session)** — `GET /:org/agent_identities/:id/vcard` returns a vCard 3.0 `.vcf` attachment (`AgentIdentity#to_vcard`) for import to Outlook / Google Contacts. HATEOAS `vcard` link added.
- [x] **D4** — ~~SendGrid receiving-domain setup~~ **Done** — Félix configured the three SendGrid Inbound Parse domains: `agent.crewkit.ai` (prod), `staging.agent.crewkit.ai`, `dev.agent.crewkit.ai`. Wired into code via env-aware `AgentIdentity.email_domain` (+`AGENT_EMAIL_DOMAIN` override); the webhook matcher now accepts each env's domain. _(Félix infra + code: this session)_
- [x] **D5** — ~~Verify valid domain parameters~~ **Done** — Félix provided the verified domains (above); example addresses `roger123@…` / `testing123-crewkit-a898gf2d@…` confirmed valid against the model's local-part rules.

## Epic E — Local-first processing _(cross-cutting cost lever)_

> **Spike done (2026-06-09):** see [spike-e-local-first-processing.md](./spike-e-local-first-processing.md). Reuse the cloud-job `claim_next`/heartbeat/stale pattern for a local-worker queue; cloud fallback by default; start with session deep-analysis + vectorization. Defer the build until analysis cost is a felt line item; never make a user-facing/governance path depend on a local worker.

- [x] **E1** — ~~Design opt-in "local mode"~~ **Designed** — local CLI registers as a worker, claims `local_eligible` jobs, runs them with local compute, posts results back. (org token kept for billing consistency; only compute is local.)
- [x] **E2** — ~~Cloud fallback~~ **Designed** — claim deadline → cloud reclaims unclaimed jobs; headless/cron always cloud. Best-effort accelerator, never a dependency.
- [x] **E3** — ~~Identify latency-tolerant jobs~~ **Done** — eligible: session deep-analysis, vectorization, contradiction scans. Excluded: webhook/artifact processing and anything user-blocking.

## Epic F — Institutional memory & contradiction detection _(Pillar: Govern — Phase C/D)_

> **F3 is now the headline next step** — F1/F2 are implemented (`feat/session-vectorizer`), so the searchable substrate exists. Contradiction detection is the layer the meeting actually asked for.

- [x] **F3** — ~~Contradiction pass~~ **Done** — `ContradictionDetectionService` semantic-searches prior decisions on each new artifact and flags conflicts (`a08b776`); cascades on org/project hard-delete (`9ffa528`). ⭐
- [x] **F1** — Vectorize analyzed conversations for project search. _Implemented: `SessionVectorizationService` + `SessionVectorizerJob` + recurring scan, auto-triggered on analysis completion. Merged (PR #168)._
- [x] **F2** — Single queryable knowledge base with `scope: "sessions"` search + "Find prior work" dashboard card. _Sessions flow in today; meetings/email/Slack arrive via Epics C/D (search layer already supports all sources)._
- [x] **F4** — ~~Spike: index/share Claude's local user-directory memories within project context.~~ **Spike done** — see [spike-f4-local-memory-ingestion.md](./spike-f4-local-memory-ingestion.md). Verdict: mechanics are trivial (thin adapter on the session-vectorizer funnel + a `crewkit memory sync` CLI command), but the real work is the **privacy/consent model**, which overlaps `proposal-curated-context-ingestion.md`. **Defer the build until that consent decision lands**; ingest only `project`/`reference` memories behind explicit per-file opt-in — never an always-on/hook path.

### Landing `feat/session-vectorizer` (in progress 2026-06-04)

- [x] Merge current `main` into the branch (clean, no conflicts — brings in Slack OAuth UI + CLI branch fix)
- [x] Run migration `20260603000001_add_llm_session_to_artifacts` + verify `schema.rb`
- [x] Run API tests (`PARALLEL_WORKERS=1`)
- [x] Run dashboard tests (`npm test`)
- [x] Open PR + confirm CI green — **merged (PR #168)**

---

## Sequencing notes

1. ~~D code path is done (D1–D3); D4/D5 remain the only blockers~~ **Epic D complete** — D4/D5 (SendGrid receiving domains + verification) landed via env-aware `AgentIdentity.email_domain` (`f47b761`); end-to-end inbound email is unblocked.
2. ~~F1 done → F3 next~~ **Epic F complete** — F1/F2 (vectorization substrate), F3 (contradiction detection) shipped; F4 spike closed (defer build until the curated-ingestion consent model lands).
3. ~~B before C~~ **B2–B4 done; C2 shipped** — the ingestion funnel now has a transcript intake edge (`POST /webhooks/transcripts/:provider` behind the `TranscriptProviders` seam). **C1** is the only Epic C remainder: provision a transcription-provider key (Fireflies first) and verify the live fetch.
4. **A1** resolved (local `gh` flow satisfies it; cloud-job PRs stay on Octokit). **A3** done; **A4** is a quick manual regression check.

> **Next codeable, no external blocker:** C1 live-fetch verification (needs a Fireflies key first), then **A4** (manual summary-faithfulness regression). All other workstreams (A/B/C2/D/E-spike/F) are closed.
