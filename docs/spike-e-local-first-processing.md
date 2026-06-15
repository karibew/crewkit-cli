# Spike E — Local-first processing (cost lever)

> **Status:** spike / recommendation (not a commitment). Owner: Arthur. Date: 2026-06-09.
> Source: [meeting-tasks-2026-06-04.md](./meeting-tasks-2026-06-04.md) Epic E (E1/E2/E3).

## Question

Can we cut crewkit's compute/token cost by running **latency-tolerant background
analysis on the user's already-authenticated local CLI** (their machine, their tokens)
instead of in the cloud — with a cloud fallback when no local worker is attached?

## What runs in the background today

These jobs are the candidates (in `api/app/jobs/`):

| Job | Latency-tolerant? | Local-eligible? |
|---|---|---|
| `submit_session_analysis_batch_job` (deep analysis) | yes (already delayed via Batch API) | ✅ best candidate |
| `session_vectorizer_job` | yes | ✅ |
| `detect_project_contradictions_job` / `contradiction_detection_job` | yes | ✅ |
| `blueprint_analysis_job` | mostly | ⚠️ maybe |
| `process_webhook_event_job`, `process_artifact_job` | no — must be fast/always-on | ❌ stay cloud |
| `expire_analytics_caches_job` | n/a (no LLM) | ❌ |

Note: deep analysis already bills the **org's Claude token** (`org_claude_token`) via the
Anthropic Batch API. So "local mode" doesn't change *whose* tokens pay — it shifts the
**orchestration + compute** off crewkit's cloud onto the user's machine, and swaps the Batch
API call for the user's local CLI invocation.

## Mechanism — reuse the cloud-job worker pattern (E1)

No new infrastructure. The `crewkit run` cloud path already has the exact primitive:

```
CloudJob.claim_next(worker_id:, capacity:)  +  heartbeat  +  stale scope (5 min)
```

Mirror it for a local-job queue:

```
local CLI (authed)  ──long-poll──▶  POST /local_jobs/claim_next   (claim latency-tolerant jobs)
       │                                     │
       │ runs job with local Claude          ▼
       └──────── POST /local_jobs/:id/result ── result validated, persisted
                          ▲
                   heartbeat every ~30s; no heartbeat in 5 min ⇒ reclaimed
```

- The CLI registers as a worker on session start (it's already authenticated) and opts in via
  config/flag (`crewkit worker --local` or a setting).
- The API marks eligible jobs `local_eligible` and offers them to local workers first.

## Cloud fallback (E2)

- Each local-eligible job gets a **claim deadline** (e.g. 60s). If no local worker claims it,
  the cloud runner takes it — identical to the stale-reclaim the cloud path already does.
- **Headless / cron / orchestrator runs always go cloud** (no human CLI attached).
- Net effect: local mode is a *best-effort accelerator/cost-saver*, never a correctness or
  availability dependency.

## Which jobs qualify (E3)

Ship the **smallest safe slice first**: `submit_session_analysis_batch_job` (session deep
analysis) and `session_vectorizer_job`. Both are already delayed, owner-scoped, and produce
*advisory* output (coaching tips, search index) — not authoritative state. Contradiction
scans are a fast follow. Webhook/artifact processing and anything user-blocking stay cloud.

## Security & trust (the real design work)

- **Data exposure: low.** A local worker only ever processes *its own user's* project data on
  *their own* machine — nothing crosses a trust boundary that wasn't already theirs.
- **Result integrity: the concern.** A compromised/buggy local worker could post poisoned
  analysis. Mitigations: (1) a worker may only claim jobs it's authorized for (own
  org/project, enforced by the existing JWT scoping); (2) local-produced results are
  **advisory** (summaries/tips/index), never governance-authoritative; (3) validate/limit
  result payloads server-side as we do for orchestrator reports.
- **Don't leak cloud secrets to the worker** — the job carries only the user's own context;
  no org-wide credentials are handed down.

## Recommendation

**Real lever, but sequence it after Phase B/C product work.** Reasons:

- The **plumbing is cheap** (claim/heartbeat exists; this is a parallel queue + a CLI worker
  loop + an eligibility flag).
- The **savings are proportional to analysis volume** — worth it once usage grows, marginal
  today.
- The **risk is contained** by keeping local results advisory + cloud fallback default.

**So:** keep E as a spike (this doc closes E1/E2/E3 as design). Build v1 when analysis cost
becomes a felt line item — start with session deep-analysis + vectorization, opt-in,
cloud-fallback-by-default. Do **not** make any user-facing or governance path *depend* on a
local worker being present.

## Open questions (for build time)
1. Worker lifecycle: tie to an interactive CLI session, or a long-lived `crewkit worker`
   daemon the user starts explicitly?
2. Does local mode use the **user's** Claude auth or still the **org** token? (Leaning org
   token for billing consistency; user compute only.)
3. How is "local saved us $X" surfaced, so the cost win is visible?
