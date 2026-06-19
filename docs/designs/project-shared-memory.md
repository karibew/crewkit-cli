# Design: Project Shared Memory (Claude Code memory governance)

Status: **CLI service landed** (`cli/src/services/claude_memory/`). The API-pull
trigger and server side are **designed, not built** — gated behind the
security/product requirements below (security-sandwich SDLC).

## Problem

Give a project a *shared brain* in every teammate's Claude Code sessions: surface
shared decisions and "what teammates are working on" into the agent's context,
without each developer hand-syncing `CLAUDE.md`. Built on the leader-election seam
(`with_leader_lease`) that until now had no API consumer.

## Architecture

```
crewkit API (shared ProjectMemory, derived "teammates")  ──pull──►  project LEADER (sidecar)
            ▲                                                          │ projects into
            └──────────────── push (local learnings) ◄──── reads ──── ▼
                                              ~/.claude/projects/<slug>/memory/crewkit-<key>.md  (per-user, 0600)
                                              + one managed link block in MEMORY.md
                                                          │ loaded on demand
                                                          ▼
                                              Claude Code sessions (this developer)
```

### Why crewkit owns its own files (not committed CLAUDE.md)

crewkit **never edits a committed `CLAUDE.md`**. It owns `crewkit-<key>.md` topic
files in the per-user auto-memory dir and maintains one delimited managed link
block in the human-curated `MEMORY.md`. This closes the two worst vectors raised
at gate-1: prompt-injection into the team's committed instruction surface, and
git churn / leaking presence data into history. The committed-file managed-block
machinery (`markers.rs`) stays available for a future *durable* shared-config case
but is not in the write API.

## The landed CLI service (`cli/src/services/claude_memory/`)

- `markers.rs` — keyed, full-line, namespaced (`crewkit:managed:<key>`) managed
  blocks; exactly-one-pair-or-refuse; byte-offset splice preserving surroundings;
  body rejects `<!--`/`-->`/marker-namespace/`@import` lines, 64 KiB cap.
- `io.rs` — confinement-checked file IO over the shared `services::safe_io`
  primitives (`O_NOFOLLOW`, owner/regular verify, atomic 0600 temp+rename):
  lexical + canonical-parent confinement, symlink-ancestor refusal on read AND
  write, confined dir creation (no `create_dir_all` symlink-follow), 5 MiB read
  cap.
- `discovery.rs` — auto-memory slug derivation; enumeration of human memory
  (CLAUDE.md/.claude/CLAUDE.md/CLAUDE.local.md/rules/nested) + auto-memory dir;
  per-entry mtime (time-aware navigation).
- `leader.rs` — `LeaderGate` runs every read/write inside `with_leader_lease`
  (leader-only + single-writer; lock held across read-modify-write). `Unchecked`
  is `#[cfg(test)]`-only — production cannot construct an ungated access.
- `mod.rs` — `ClaudeMemoryService`: `navigate`, `read_memories` (incl. non-crewkit
  human files), `put_crewkit_file` (idempotent on logical content; `scope` +
  `updated_at` time-space metadata), `read_crewkit_file` / `list_crewkit_memories`
  (structured recovery for harmonization), `remove_crewkit_file`. Slug-collision
  guard via a `.crewkit-project` identity sentinel (fail closed on mismatch).

Consumer-agnostic and filesystem-only; the sidecar/API live above it.

## API-pull trigger — design (NOT built)

### CLI integration (sidecar leader)
- Add a periodic `memory_tick` arm to `sidecar::supervisor::daemon_main`'s
  `select!`; a cheap `is_leader` probe gates it, authoritative re-check happens
  under the lock per write.
- New `sidecar/memory_refresh.rs` orchestrates: (a) async PULL shared memories,
  (b) `spawn_blocking` the sync leader-gated `put_crewkit_file` calls (the flock
  is blocking — must not run on the reactor), (c) async PUSH local learnings.
  Do NOT hold one lease across pull→write→push; re-establish leadership per write.
- Repo root: `git_detector::main_repo_root(cwd)` (landed) resolves the worktree→
  main mapping (`--git-common-dir`) so all worktrees share one auto-memory dir.
  Cache it at launch in `DaemonHandles`; org/project/auth come from the already-
  authenticated `Arc<CrewkitApiClient>` the daemon holds.
- Degrade (never dead-end): missing project_id / api_client / repo root / offline
  → skip the tick, keep observing.

### Server contract (api-designer)
- New `ProjectMemory` (canonical, server-owned, versioned, `org_id`+Pundit,
  PaperTrail) + `MemoryContribution` (raw per-user push pre-consolidation). Do
  NOT reuse `Artifact` (wrong lifecycle). The "teammates working on" memory is
  **derived, not stored** — computed from the existing `active_work` building
  blocks (recently-active sessions + in-progress blueprint tasks) with a TTL.
- Endpoints under `scope ":organization_id" / projects/:project_id`:
  `GET memories` (incremental via `version`/`since`, `include_body`),
  `GET memories/teammates` (derived, `expires_at` TTL), `POST memories` (→
  contribution, `202`, sanitized).
- Consolidation/harmonization = a debounced background job (Haiku/Batch, like
  session analysis). MVP: single-contribution last-writer-wins, no LLM. Later:
  multi-contribution Haiku merge + contradiction flagging (reuse
  `ContradictionDetectionService`).

## Security gates BEFORE building the trigger (security-expert, gate-1 for the trigger)

The trigger inverts trust: content flows from the API / other teammates into
files Claude loads as context. Marker/`@import` hardening is necessary but NOT
sufficient. MUST-haves before implementation:

- **T1 Prompt-injection (CRITICAL):** a hostile branch name / task title / shared
  body becomes agent instructions. Mandatory structural inert-framing (fence +
  per-field "untrusted status data" preamble), escape free-text as data, hard
  line/length caps on status blocks well below 64 KiB.
- **T2 Per-source authz:** every projected memory must match the resolved
  `org_id`+`project_id`; fail closed on mismatch (ties to the slug-collision
  guard). Respect session privacy/`capture_mode` gating — never project a
  teammate's private/sensitive session content.
- **T3 Freshness/TTL:** status memories carry `expires_at`; the leader must not
  project (and must remove) expired ones; sanity-bound `updated_at` against a
  trusted clock.
- **T4 Transport/authenticity:** enforce TLS cert validation; treat every response
  as untrusted even on 200; consider per-memory provenance signing.
- **T5 Anti-rollback:** record last-applied `version` per key; refuse regressions
  (replay defense).
- **T6 Push-back trust:** server must re-validate + attribute + authz-scope pushed
  content before it can ever be re-projected to teammates; re-validate recovered
  local content client-side before upload; rate-limit pushes.

## Product sequencing (product-strategist) — ship THIN first

- **MVP (strong wedge):** leader pulls one team-curated `crewkit-context.md` +
  a short factual active-work line → appears in every teammate's sessions via the
  existing `--append-system-prompt` path. Human-curated (no auto-generation),
  clearly delimited/attributed, reciprocal visibility, last-updated shown.
- **Defer (behind MVP metrics + pgvector seam):** auto-consolidation,
  harmonization, compression, breaking-news cadence, local push-back — this is
  where trust/privacy risk concentrates.
- **Resolve first:** the PLAN.md active-work privacy question (recommend per-user
  opt-in/opt-out, reciprocal, never a manager surveillance view).
- **Frame as collaboration, not surveillance:** show activity not idleness/output;
  team-authored content; visible because you chose to share.

## Differentiation / moat

Not the file injection (anyone can append a system prompt) — the **live
active-work + observed-outcome substrate** (only possible because crewkit observes
every session) and the **leader-election coordination** that makes multi-session,
multi-developer projects behave coherently.
