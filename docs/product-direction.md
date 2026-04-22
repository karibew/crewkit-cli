# Product Direction

**Last updated**: 2026-04-21 (post-strategy meeting)

## The pivot in one paragraph

Crewkit started as a CLI wrapper around Claude Code — observing sessions, capturing telemetry, injecting agents and playbooks. That foundation works. But the CLI wrapper is no longer the differentiator: each new Claude release subsumes more of what the CLI does today. What Claude can't do — and will continue to struggle with — is the **human/context gap**: the decisions made in meetings, the threads on Slack, the emails between stakeholders, the legacy patterns that only one person remembers. Crewkit's job is to close that gap. The product is an institutional memory layer, a governance layer, and a collaboration layer — for the humans *and* the agents on a project.

## Four pillars

### 1. Observe (existing, simplify)

Track every Claude session that touches a project, plus the artifacts of work (commits, PRs, deploys, Sentry errors). The domain simplifies to **Conversation → Task**: a conversation is an LLM session keyed by `claude_session_id`, a task is a subagent invocation. The old "Run" container is retired — Claude Code already models its own sessions, and wrapping them added a tier without adding value.

### 2. Govern (existing, extend)

- **Playbooks & conventions** per project (including legacy code patterns — conventions differ between a greenfield and a 10-year React codebase).
- **Role-based autonomy**: coaching / collaborative / autonomous tiers.
- **Resource inheritance** (platform → organization → project).
- **Safeguards** running as background jobs: test-quality evaluation, convention drift detection, contradiction detection across meetings and decisions.

### 3. Ingest context (new)

Project context comes from more than the repo. We ingest and vectorize:

- **Meeting transcripts** — a Firefly-style bot participates, transcripts route per-project.
- **Slack threads** — channel messages, thread replies, DMs (with consent).
- **Email threads** — project-scoped via dedicated addresses (e.g., `proj+{token}@crewkit-inbox.io`), basecamp-style.
- **Git log** — commits, PRs, reviewer comments, merge decisions.
- **Past sessions** — Claude session transcripts, analysis summaries, turn-level replay.
- **Google Drive docs** — specs, PRDs, retros.
- **Sentry / observability** — deploy regressions route to the last committer.

All of this rides on the existing `Artifact` + `ArtifactSearchService` + DigitalOcean Knowledge Base stack. New adapters, same retrieval primitives.

### 4. Collaborate (new)

Agents stop being invoked — they start participating.

- **Agent identity**: each agent has an email address and a Slack handle, scoped to a project or org. Inbound messages route to the right agent; outbound agent messages appear in the same channels humans use.
- **Cross-user active-work visibility**: a team member opens a project and sees what other teammates are actively working on (conversations in progress, Blueprint tasks claimed). Prevents stepping on each other's changes.
- **Bi-directional**: agents can ask clarifying questions via Slack/email instead of silently guessing.
- **Living project plan**: Blueprint + epics + tasks becomes the shared understanding of what's decided, what's in flight, what's next — updated continuously from sessions, meetings, and commits.

## What we are *not* building

- A general-purpose project management tool — we don't replace Jira; we integrate.
- An AI product for solo hobbyists — the product's value compounds with team size.
- A CLI rewrite from scratch — the existing CLI survives and simplifies.
- A new meeting transcriber — we consume existing tools (Firefly, Meet, Zoom exports).

## Why now

- Solo-founder / 2-person projects are the norm at Karibou today. Any team member going on vacation stalls the project. The institutional-memory layer unblocks this.
- Claude 5.8+ will close the technical-convention gap. That's not where the moat lives.
- Vacation season (summer 2026) is the forcing function — projects should continue, not pause.

## Execution

See [PLAN.md](../PLAN.md) for the concrete phase breakdown (A through E). Phase A (flatten CrewkitSession) is the first block — everything else plugs into a cleaner data model.

## Decisions frozen on 2026-04-20

| # | Decision |
|---|---|
| 1 | Drop CrewkitSession; flatten to LlmSession + SubagentSession. |
| 2 | CLI V1 is refactored in place — no parallel V2 rewrite. |
| 3 | Orchestrator and cloud_jobs stay; rewires to LlmSession. |
| 4 | Agent-as-employee (email, Slack) is near-term, not 2027. |
| 5 | Blueprint is enhanced and stabilized as the living project plan. |
| 6 | Ticket-based UX stays alongside Blueprint, not replaced. |
| 7 | Session import survives and may become an LLM gateway. |
| 8 | Session sharing stays; session vectorization + cross-user active-work are explicit scope. |

Anything outside this list that surfaces as "built-in" today was an implementation assumption, not a requirement.
