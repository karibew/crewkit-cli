# Spike F4 — Index Claude's local user-directory memories in project context

> **Status:** spike / recommendation (not a commitment). Owner: Arthur. Date: 2026-06-09.
> Source task: [meeting-tasks-2026-06-04.md](./meeting-tasks-2026-06-04.md) Epic F (F4).
> Related: [proposal-curated-context-ingestion.md](./proposal-curated-context-ingestion.md) ("company brain" opt-in ingestion).

## Question

Can — and should — we pull Claude Code's **local user-directory memories** into a crewkit
project's searchable context, so one teammate's hard-won local notes become discoverable by
the rest of the team (and by agents) through the existing Knowledge Base?

## What "local memories" actually are

Claude Code keeps per-user, per-project auto-memory on the local filesystem:

```
~/.claude/projects/<project-slug>/memory/
  MEMORY.md                         # index (one line per memory)
  <slug>.md                         # one fact per file, with frontmatter
```

Each memory file carries frontmatter that is decisive for this spike:

```yaml
---
name: dev-db-reseed-gotcha
description: <one-line summary used for recall>
metadata:
  type: user | feedback | project | reference
---
```

This repo's own memory dir (today) holds 4 facts + an index — e.g.
`team-agents-demoed-interest`, `curated-context-ingestion-proposal`,
`dev-db-reseed-gotcha`, `context-retrieval-slice-verified`. These are **not in git**;
they live only on the author's machine. That is exactly why they're invisible to the team
and why F4 is interesting — and also why the privacy question is sharp.

> Scope note: `CLAUDE.md` / `.claude/rules/*.md` are *also* "memory" in a loose sense, but
> they're checked into the repo and already shared. F4 is specifically about the
> **un-shared, local, per-user** auto-memory files above.

## Technical path (low risk — the funnel already exists)

F4 needs **no new infrastructure**. It mirrors the session-vectorizer slice that already
shipped (F1/F2):

```
local memory/*.md  ──(CLI reads)──▶  POST artifacts (source: "memory")
                                          │
                                          ▼
                                   ProcessArtifactJob  ──▶  DO Knowledge Base (chunk + embed)
                                          │
                                          ▼
                              ArtifactSearchService  ◀──  search scope: "memories"
```

Concretely, reuse what `SessionVectorizationService` established:

- **Artifact source**: add `"memory"` to `Artifact::SOURCES` (today:
  `upload google_drive url content email slack session`). One-line change + migration-free
  (it's a validated string, not an enum column).
- **Search scope**: add `SCOPE_MEMORIES = "memories"` alongside the existing
  `SCOPE_SESSIONS` in `ArtifactSearchService` — same source-filtering mechanism.
- **Ingestion entry point**: the **CLI**, not the API. The Rails API has no access to the
  user's `~/.claude` filesystem; the CLI does. There is currently **no `crewkit artifact`
  or ingest command**, so this is net-new CLI surface (a `crewkit memory sync` command),
  posting each file to the existing artifact-create endpoint.

So the build is small: ~1 API constant + 1 search-scope branch + 1 CLI command. The
indexing/search layers are untouched.

## The crux: privacy & consent (this is the whole decision)

The session vectorizer already encodes the rule we must respect — vectorized artifacts
**surface to every project member**, so it refuses to index anything sensitive:

```ruby
# SessionVectorizationService#vectorizable?
return false if @session.sensitive?
return false if @session.repository&.private_visibility?
```

Local memories are **more** sensitive than sessions, and the frontmatter `type` tells us
how:

| `type`     | Contents | Default disposition |
|------------|----------|---------------------|
| `project`  | ongoing work, goals, constraints | **shareable** — team-relevant by definition |
| `reference`| pointers to dashboards/tickets/docs | **shareable** |
| `feedback` | coaching the *user* gave the agent ("how I want you to work") | **exclude by default** — personal working style |
| `user`     | who the person is (role, prefs, expertise) | **exclude by default** — personal |

**Recommendation:** ingest `project` + `reference` only, behind an **explicit opt-in**, and
never silently. This matches the spirit of `proposal-curated-context-ingestion.md` (opt-in
"company brain") and the session privacy gate. `user`/`feedback` memories stay local unless
the author explicitly elects to share a specific file.

Additional guardrails:
- **Per-file opt-in over bulk** — a `crewkit memory sync` dry-run that lists what *would* be
  shared (with `type` + description) and requires confirmation. Never auto-run on a hook.
- **Attribution** — tag the artifact with `uploaded_by` so the team sees whose memory it is.
- **Revocability** — deleting the artifact retracts it from the KB (the retract path already
  exists for sessions).
- **No secrets** — memory bodies can contain tokens/paths; run the same sanitization the
  hook/event pipeline already applies before upload.

## Proposed v1 (if we proceed)

1. **API**: add `"memory"` to `Artifact::SOURCES`; add `SCOPE_MEMORIES` to
   `ArtifactSearchService`. (~30 min, fully covered by existing artifact tests + 2 new.)
2. **CLI**: `crewkit memory sync [--project] [--include feedback,user] [--dry-run]`
   - reads `~/.claude/projects/<slug>/memory/*.md`
   - filters to `type ∈ {project, reference}` unless `--include` widens it
   - dry-run prints the share list; real run POSTs each as `source: "memory"`,
     `artifact_type: "memory"`, tagged `memory` + the frontmatter `type`
   - idempotent by memory `name` (skip/refresh existing, same as the session vectorizer)
3. **Dashboard**: memories appear in the existing artifact list + "Find prior work" search;
   add a `memory` source filter chip. (No new view.)

## Recommendation

**Defer, but it's cheap and aligned when we pick it up.** Ranking against the rest of Epic F
and the backlog:

- The **mechanics are trivial** (the funnel is built; F4 is a thin adapter + CLI command).
- The **value is real but niche today** — with a 4-file memory dir, the payoff is small until
  more of the team dogfoods auto-memory. It compounds later.
- The **privacy surface is the actual work** and overlaps the unresolved
  `proposal-curated-context-ingestion.md` consent model. **F4 should ride on that decision,
  not front-run it** — building per-file opt-in here first would pre-empt a product call that
  deserves to be made once, centrally.

**So:** keep F4 as a spike (this doc closes it), and schedule the build *after* the curated-
ingestion consent model is settled. When we do, it's a half-day slice. Do **not** ship an
always-on / hook-driven version — that would index personal `user`/`feedback` memory to the
whole team without consent.

## Open questions (for the curated-ingestion decision, not F4 alone)

1. Is "share my local memory" a per-file opt-in, a per-type default, or an org policy?
2. Do shared memories stay attributed to the author, or fold into a neutral "project memory"?
3. Should agents be able to *write back* team memories locally (round-trip), or is this
   one-way (local → KB) only? (v1: one-way.)
