# crewkit Deep Assessment — 2026-06-11

> Multi-agent audit of the entire platform: 12 subsystem deep-reads, 3 competitor research sweeps,
> 101 adversarial verification agents, and a 227-item fix sweep. Every finding below was
> independently verified against the code by a skeptical agent instructed to refute it; the
> confirmation rate was 96% (251 of 258 raw findings stood, 7 refuted or unfixable as claimed).
>
> Companion artifacts: verified findings JSON (session transcript dir), fix sweep on branch
> `chore/deep-assessment-sweep`, roadmap below.

## Executive summary

crewkit's foundations are stronger than its surface: the TUI's PTY/vt100 rendering core, the
hook pipeline, resource inheritance, and the analytics data model are well-built and mostly
well-tested. The product's stated differentiators, however, are systematically **half-wired**:
the share dialog renders but doesn't share, the LSP server runs but Claude Code never consumes
it, the LLM gateway captures but uploads into a deleted endpoint, conventions challenge but
never record decisions, the deep-analysis pipeline silently no-ops on fresh deploys, and the
"suggest skills/agents" loop has telemetry but no recommender. The pattern is consistent:
**observation layers shipped; the loops that close them didn't.**

The strategic position is excellent. Competitor research shows the team-context market has
converged on MCP-pull (Glean, Unblocked, Sourcegraph, Rovo all ship MCP servers), while
**nobody occupies push-context-mid-session** — exactly what crewkit's hook pipeline + local
gateway make possible. The moat is the combination: observed sessions → institutional memory →
injected back into the next session, scoped to the team.

## 1. Verified findings overview

| Category | Raw | Confirmed | Partial | Refuted |
|---|---|---|---|---|
| Bugs | 103 (100 after dedup) | 87 | 12 | 1 |
| Stray code | 71 | 63 | 8 | 0 |
| Doc drift | 84 | 77 | 5 | 2 |

Severity of confirmed bugs: 3 critical, ~30 high. The three criticals:

1. **14 management CLI commands are no-op stubs that print fabricated success** (org/members/
   projects management). A team admin "removing" a member sees success; nothing happens.
   Trust-destroying. (Stubs removed in the sweep; full wiring is roadmap.)
2. **Claude Code never consumes the crewkit LSP server** — the per-session TCP server +
   `.claude/.lsp.json` + leader election is dead weight; only the stdio plugin path is real.
3. **Production + staging Rails secrets remain in git history** (DEPLOYMENT.md, rotation
   unconfirmed — carried over from the 2026-06-11 doc audit). **Risk accepted 2026-06-11**
   (private, access-controlled repo); revisit if repo visibility or team access changes.

Notable confirmed high-severity items (all with fix recipes in the verified findings):

- `/clear` **never resets TUI telemetry** — `current_session_id` is write-once; tokens/cost/
  summary silently accumulate across conversations (the 12 unit tests all simulate a state
  production can't produce).
- **Force-quit doesn't force**: Ctrl+Q and the 15s shutdown timeout funnel into an unbounded
  `join()` on a thread whose Haiku step has no timeout — frozen raw-mode screen.
- **EventBuffer mutex held across 30s network flushes** stalls every hook (the lock-free
  `EventBufferHandle` actor was built and never wired in).
- **Hooks cannot block anything** — the `PreToolUse` deny path is dead code; playbook
  "enforcement" is advisory-only despite dashboard settings implying otherwise.
- **Gateway uploads 404** against an endpoint deleted 2026-01-18; failures are silent.
- **Deep-analysis pipeline no-ops on fresh deploys** — the required `session_analysis` Brief
  is never seeded; batch failures strand sessions in `analyzing` forever.
- **ResourceStat counts are systematically double-counted** (re-incremented on batch-analysis
  updates), corrupting the data the improvement loop depends on.
- **Restricted-repo sessions leak into the project-wide KB** readable by any org viewer.
- **Convention challenges are logged as "overridden" before the user decides**, and the
  override-resolution loop was never implemented — users get re-challenged forever.
- **Session sharing is broken end-to-end**: TUI dialog is a stub, `share_url` points at a
  dashboard route that doesn't exist, and no share viewer page exists — while the marketing
  page advertises the feature.
- **Headless mode (`crewkit code -p`) records nothing** — a regression from the Feb 2026
  phase refactor; CI/cloud usage is invisible.

## 2. What the sweep fixed

Branch `chore/deep-assessment-sweep` applies ~200 verified trivial/small fixes across
220+ files (cli, api, dashboard, docs, agent briefs). All three test gates passed after
minor repairs:

- **cli**: `cargo fmt` clean, clippy identical to the pre-existing baseline, 921 unit tests
  plus all integration suites and doctests green.
- **api**: full suite 4,532 runs / 12,856 assertions, 0 failures; RuboCop clean on every
  changed file.
- **dashboard**: lint 0 errors, typecheck clean, 1,529 vitest tests green.

Two items were deliberately not applied (need decisions): wiring `record_benchmark_session`
requires a schema migration, and deleting the `/kit/shares` stubs depends on the sharing
roadmap (§6). All medium/large items appear in the roadmap (§9) with their verified fix
recipes.

## 3. Hooks vs. built-in LLM gateway — recommendation: hybrid, gateway-forward

**What exists today.** Hooks capture 10+ lifecycle event types (sanitized, batched); the
`--llm-gateway` flag spawns a local axum proxy and points Claude Code at it via
`ANTHROPIC_BASE_URL`. The gateway already captures full request/response pairs — but uploads
to a deleted endpoint, ignores org privacy settings (`capture_mode`, `retain_responses`,
`sensitive`), hardcodes the upstream, kills streams at 300s, can't decompress gzip, rejects
>10MB bodies, and has no recovery if its task dies. Product-direction decision #7 already
anticipated this evolution ("Session import … may become an LLM gateway").

**The decision is not either/or — the layers see different things:**

| Capability | Hooks | Gateway |
|---|---|---|
| Lifecycle events (session start/end, tool use, subagent) | ✅ native | ❌ must infer |
| Full prompts/completions, system prompt, token-level data | ❌ | ✅ |
| Block a tool call (PreToolUse deny) | ✅ (unwired today) | ❌ (request already past) |
| **Inject context mid-conversation** | ⚠️ only at UserPromptSubmit | ✅ every request |
| Model allow-lists, token budgets, outbound redaction | ❌ | ✅ |
| Survives Claude Code update churn | ✅ stable contract | ⚠️ API-shape coupling |
| Failure blast radius | low (events lost) | high (session bricked if proxy dies) |

**Recommendation.**
1. Keep hooks as the *governance and lifecycle* plane: wire the real `PreToolUse`
   `permissionDecision: deny/ask` for `severity: error` conventions; keep event telemetry.
2. Promote the gateway to the *capture and context* plane, in stages: (a) fix correctness +
   privacy gating (sweep fixed the trivial parts; endpoint rebuild is roadmap), (b) enable for
   headless/cloud-job runs first (controlled environment, fixes the headless observability hole
   at the same time), (c) then default-on for `crewkit code` with automatic bypass-on-death
   (env unset + restart guidance) so a dead proxy can't brick a session.
3. The unique payoff is **dynamic, cache-aware context injection** (§5) and request-level
   governance — the two things hooks can never do. This is also the competitive whitespace.

## 4. LSP server — kill the dead path, ship knowledge providers

**Today**: tree-sitter indexing into redb works; the stdio plugin path is the only one Claude
Code actually consumes. The TCP server / `.claude/.lsp.json` / leader-election path is dead
weight (confirmed critical); redb's "read-only fallback" retries the same exclusive open so a
second concurrent session simply gets no index; the initial scan holds the write lock for its
entire duration; deleted/renamed symbols leave phantom goto-definition entries. Silent failure
everywhere — users believe code intelligence is on when it isn't.

**Recommendation.**
1. Consolidate on the stdio plugin; delete the TCP/registry path (~roadmap, recipe verified).
2. Fix the index lifecycle (batched transactions, no lock-held scan, stale-symbol pruning).
3. Add `crewkit lsp doctor` + LSP status in the TUI sidebar (silent failure is the #1 DX gap).
4. **Build the crewkit-knowledge providers — the only durable differentiator.** The LSP is the
   one surface Claude Code queries *natively, mid-task, for free*. Inject business-relationship
   logic as: hover ("owned by payments team; convention: use MoneyGem, not Float; 2 open
   Blueprint tasks touch this"), code-lens ("Alice has an active conversation editing this file
   — view"), diagnostics for convention violations (deterministic rules first, LLM later), and
   "who knows this code" from session history. Each provider is a thin API call against data
   crewkit already has (`active_work`, conventions, session vectors, `ResourceStat`).

## 5. Dynamic KB / context injection — the core product promise

**Today**: ingestion → vectorization → DO KB search works (artifacts, sessions). But injection
happens once at session start with a hardcoded query ("general project context"), inbound
email/Slack artifacts were never vectorized (fixed in sweep), headless bypasses everything,
and there is no mid-session path. KB search silently degrades to ILIKE in unconfigured
environments with no signal.

**Recommended architecture (three stages, additive):**
1. **Session-start, fixed** (sweep): query built from git context + recent commits + active
   Blueprint task instead of the hardcoded string.
2. **Mid-session pull — crewkit MCP server** (roadmap, high priority): expose
   `search_project_context`, `find_prior_work`, `get_active_work`, `get_conventions`,
   `get_blueprint_state`, `who_knows` as MCP tools. This is table stakes in the market (Glean/
   Unblocked/Sourcegraph all ship it) and the cheapest distribution: any MCP-capable agent —
   not just the crewkit TUI — can consume crewkit context. It also makes the KB *agent-queryable
   for validation* ("check the KB before assuming").
3. **Mid-session push** (the whitespace): UserPromptSubmit hook → KB + active-work query →
   `additionalContext` injection when relevance crosses a threshold ("Bob's meeting decision
   yesterday contradicts this approach; see artifact #142"). Later, gateway-level injection
   with prompt-cache-aware placement (append-only blocks to avoid cache busts).

## 6. Session sharing & reporting — the DX promise to fix first

The journey "I just finished a great session, send it to my teammate" is **impossible today**
(stub dialog → no URL → no viewer page). The backend (SessionSharingService, tokens, expiry,
annotations) is complete and tested. Fix order (roadmap §9, "Now"):
1. Dashboard `/shared/[token]` viewer page (medium — blocks everything else).
2. Wire the TUI ShareDialog submit → mint link → OSC52 copy to clipboard → print OSC8
   hyperlink (small; the dialog and API both exist).
3. `crewkit sessions list / show / share / report` — the `sessions` namespace currently has
   exactly one subcommand (`import`). `report` = weekly digest per project/team (cost, outcomes,
   hotspots) reusing SessionAnalyticsService; later email/Slack delivery.
4. Close the share-policy gap (sensitive/restricted-repo sessions, fixed in sweep) and add the
   share group to the OpenAPI spec so the dashboard client is typed.

## 7. Analytics — close the loops the product already promises

Verified state vs. the four promised loops:

| Promise | State | Path |
|---|---|---|
| Code hotspots | ❌ missing | PreToolUse file_path events are already captured; aggregate per project/file/week |
| Correction rates | ❌ missing | Derivable from existing turn stream (user interrupts, re-prompts after edits, reverts) |
| Subagent/skill performance | ⚠️ captured, double-counted, no UI | Fix ResourceStat idempotency (sweep), ship team-facing skill analytics |
| Suggest skills/agents | ❌ missing | Recommender over ResourceStat + session vectors ("teams like yours use X for Y") |

Also: seed the `session_analysis` Brief in a migration (silent no-op fixed in sweep), recover
stranded `analyzing` sessions, replace placeholder p-values with the Welch's t-test already
implemented in ExperimentMetricsService, and meter analysis cost against billing.

## 8. TUI & web app — what "really cool" means concretely

**TUI (ranked by impact/effort, from competitor research + verified gaps):**
1. Which-key overlay on F12 leader + contextual keybinding footer (discoverability — zellij/
   helix pattern; the chord system exists but is invisible).
2. One-keystroke share (§6).
3. Fix the feel killers (sweep): 25ms arrow-key sleep, paste freeze, scrollback phantom lines,
   force-quit hang, `/clear` telemetry reset.
4. Teammate presence in the sidebar ("felix · auth refactor · 12m") — `active_work` endpoint
   already exists; this is the Collaborate pillar made visible every minute of the day.
5. In-TUI session picker (leader+r) with server-side fuzzy search over vectorized sessions —
   atuin-style, backed by institutional memory nobody else has.
6. Focus-aware notifications ("Claude is waiting for input") — crewkit owns the PTY; nobody
   else wraps the agent and can do this.
7. Asciinema `.cast` recording teed from the PTY, uploaded on session end, embedded in the
   share viewer (turns share pages from JSON dumps into replayable sessions).
8. Mouse support in the chrome; adaptive light-terminal theme; context-gauge + cost burn
   sparkline in the sidebar.

**Web app:**
1. Conversations as first-class org-level navigation (today the center-of-gravity entity is
   buried inside project pages).
2. "Team activity" view answering *what is my team doing right now* across projects — and make
   home-page Active Sessions actually live (it currently never refreshes but looks live).
3. Conversation detail as a story: timeline, corrections, subagent tree, cost, share button —
   not a data dump.
4. Recommendation surfaces (skills/agents to install) closing the improvement loop.
5. Ship the `/shared/[token]` viewer (§6).

## 9. Prioritized roadmap

**Now (1–2 weeks, unblock trust + the DX promise):**
- Share viewer page + TUI share wiring + `sessions list/show/report` (§6).
- Wire `EventBufferHandle` actor (kills the 30s hook stalls) + disk spool for offline.
- Re-wire headless mode onto the session pipeline (observability for CI/cloud).
- Seed `session_analysis` Brief migration everywhere; recover stranded sessions.
- PreToolUse deny for `severity: error` conventions (real enforcement, minimal scope).
- `crewkit lsp doctor` + sidebar LSP status.

**Next (2–6 weeks, the strategic unlocks):**
- crewkit MCP server (context pull — table stakes + distribution).
- Gateway rehabilitation: ingestion endpoint on LlmSession, privacy gating, headless-first
  rollout (§3).
- Convention override-resolution loop + override analytics (the governance buyer's view).
- Correction rates + code hotspots from existing telemetry (§7).
- LSP: delete TCP path, fix index lifecycle, first knowledge provider (hover: ownership +
  conventions + active work).
- TUI: which-key overlay, teammate presence, session picker.
- Dashboard: org-level conversations nav + team activity view.

**Later (the moat):**
- Mid-session push injection (hook-driven, then gateway cache-aware) (§5).
- Skill/agent recommender; session-insight clustering over vectors.
- PR-outcome correlation (sessions → merged lines → ResourceStat enrichment).
- Replay-against-version-N for A/B benchmarking (raw_jsonl already stored); real statistics.
- Asciinema replay in share pages; eval suites gating resource-version promotion.
- Expertise graph ("who knows X") from session history.

## 10. Agent briefs & ways of working

- Two briefs (task-orchestrator, crewkit-guide) were corrupted by resource sync truncation —
  restored in the sweep; the sync bug that ate them needs a regression test (roadmap).
- Six briefs delegate to a `code-reviewer` agent that doesn't exist; dispatch tables updated.
- Recommended merges: streaming-ingestion-expert + batch-ingestion-expert → one
  telemetry-ingestion-expert; retire or rewrite design-system-expert for shadcn/Tailwind v4.
- Coverage gaps: no owning brief for the LSP, the gateway, or the orchestrator.
- Implementer briefs lacked the testing discipline (PARALLEL_WORKERS=1 etc.) — added in sweep.
- AGENT_ROSTER.md regenerated; keep it generated, not hand-edited.

## 11. Documentation architecture

The sweep fixed the verified drift (wrong verbs/paths in CLAUDE.md, deleted endpoints in
public docs, stale release flow, phase-status contradictions). The structural fix (this audit
establishes the pattern): **one canonical entry point** (README → CLAUDE.md for agents,
docs/README.md as the human index), per-component canon in `.claude/rules/`, planning docs
archived when contradicted, and doc-drift CI guards (roadmap). The orchestrator remains
undocumented — it needs a component doc before Phase E builds on it.

## Appendix: methodology

Three chained multi-agent workflows (2026-06-11): (1) 15 mappers/researchers, (2) 101
adversarial verifiers (every finding re-derived from code by an agent prompted to refute it),
(3) 30+ fix agents in file-disjoint lanes + 3 test gates. ~250 agents, ~10M tokens. Findings
JSON, verdicts, and per-stage fix notes are preserved in the session transcript directory;
workflows saved to `.claude/workflows/` for re-runs.
