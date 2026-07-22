# crewkit Completeness & Gap Assessment — 2026-07-20

> **⚠️ PARTIALLY SUPERSEDED (2026-07-22).** Several gaps below closed after this snapshot and their
> rows are now stale — verified against `main`:
> - **§4 #3 / §2 / §5.4 — legacy `AgentExperiment` `/kit/experiments` dead-end: already removed** (no
>   route, no `/kit/experiments` dashboard dir, no `agent_experiment.rb` model). Recommendation §8 #2 done.
> - **§2 / §3 — convention-drift detector: shipped** (`services/trigger_classifiers/convention_drift.rb` +
>   `anti_pattern_matcher.rb`), no longer "planned-only".
> - **§5 / §8 #6 — the three missing sentinels: shipped** (`single_session_read_visibility`,
>   `webhook_signature`, `orchestrator_service_auth`, plus `convention_alert_config_secret`).
> - **§4 #11 — convention alert *delivery*: shipped** (#265: `ConventionAlertThresholdJob` +
>   `ConventionAlertDeliveryService`).
> - **§2 (Observe) — org-level session-sharing dashboard view: shipped** (`/kit/shares`).
>
> The remaining open items (Loi 25 merge, pgvector cutover, DNS/MX + Slack, org-wide active-work PRs,
> flag flips, CLI vault key #276, `crewkit run` CLI, GitLab/Bitbucket, role-tiered autonomy enforcement)
> are still accurate. See `reviewable_merge_commits.md` for the live state.

**Method:** Read-only. Nine parallel verification agents (4 pillars, 3 component lanes,
1 flags/doc-drift/dead-code miner, 1 security/Loi 25), each verifying claims against
code (`file:line`), diffed against the prior baseline (`docs/audits/deep-assessment-2026-06-11.md`,
`docs/audits/pundit-scope-audit.md`). Where a doc and the code disagreed, the code won and
the drift is recorded as a finding. State assessed on `main`.

**Classification tags:** `live` (shipped + reachable) · `flag-gated-off` (built, behind an
off flag/env by default) · `held` (built, not merged/enabled pending human sign-off) ·
`partial` (some layers missing) · `planned-only` (design/plan, no code) · `dead-end` (code
exists, nothing on the live path uses it).

---

## 1. Executive summary

**Overall completeness: ~72% to "product-complete," medium-high confidence.**

crewkit is a genuinely broad, mature system — ~95 API controllers / ~90 models / ~130
services / ~40 jobs, a substantial Next.js dashboard, a large well-tested Rust CLI, and a
functional orchestrator. **Observe** and **Govern** are the most complete pillars and are
largely live; **Ingest** is architecturally complete but running on the *old* retrieval
backend with the strategic migration dormant; **Collaborate** is code-complete but not
end-to-end exercisable by a real customer. The codebase is in better shape than its own docs
claim in several places (override-resolution loop, session-sharing UI, convention-override
analytics, active-work leak fix are all shipped despite docs/memory saying otherwise) — but
worse in a few strategically important ones (the pgvector migration is dark, two experiment
systems ship side-by-side, Loi 25 erasure/export are unmerged).

**The 5 things that most limit "done":**

1. **The pgvector/Voyage retrieval migration (Phase F2) is built but dormant** behind three
   default-off env flags (`RETRIEVAL_BACKEND=do_kb`, `EMBEDDING_PROVIDER=null`,
   `RETRIEVAL_DUAL_WRITE=off`). DO Knowledge Base is the live backend; nothing populates
   pgvector in prod. This blocks self-host and is the headline vision-drift item.
2. **Loi 25 / PIPEDA right-to-erasure and right-to-access are not on main.** Account
   deletion (#256/#257/#259) and data export (#229) are all open, unmerged PRs. This is a
   compliance obligation, not a feature nicety.
3. **Two parallel experiment systems ship at once.** The legacy `AgentExperiment` `/kit/experiments`
   surface is still routed, nav-linked, and command-palette-indexed — with permanently-empty
   metrics — while the *actual* A/B loop lives, unlabeled, under resource benchmarks. Users
   clicking "Experiments" land on the dead one.
4. **The Collaborate pillar can't be run end-to-end.** Agent inbound email is code-complete
   but `agent.crewkit.ai` DNS/MX is unprovisioned; Slack needs a real app install; org-wide
   active-work awareness sits in four unmerged PRs (#270/#224/#225/#191). Project-scoped
   active-work is the only live piece.
5. **Several finished surfaces are flag-gated dark:** marketplace GA (`MARKETPLACE_GA_ENABLED`
   off), the `/kit/runs` cloud-jobs dashboard (`NEXT_PUBLIC_FEATURE_RUNS` off), LLM-gateway
   server capture (`llm_gateway_enabled` off per-org). Built ≠ reachable.

---

## 2. Per-pillar scorecard

### Observe — **~90% live**
| Capability | Tag | Evidence |
|---|---|---|
| LlmSession lifecycle, SubagentSession, 11 hook event types, SessionTurn, SkillInvocation | live | `sessions_controller.rb:27,191`; `cli/src/hooks/registry.rs:20-31`; `event_ingestion_service.rb:194,202` |
| Session end → Haiku summary + Batch deep analysis → coaching/patterns/scores | live (no-ops without Anthropic key) | scheduled `recurring.yml:32,37`; `submit_session_analysis_batch_job.rb:26`; `session_analysis_service.rb:333-336` |
| Analytics (summary/timeseries/by-agent/cost/by-user/team-overview) | live | `session_analytics_controller.rb:28-65`; dashboard `use-analytics.ts` |
| Revert-rate | partial | `session_analytics_service.rb:111-149` real, but data only populates via `crewkit run` cloud-jobs + git webhook (`pull_request_event_service.rb:39`) |
| Insights/transcript/export, JSONL import, sharing + annotations | live | `session_insights_controller.rb`; `session_import_service.rb`; dashboard share dialog + annotation panel exist |
| Cost/token calc (prefers OTEL `cost_usd`, falls back to pricing table) | live | `sessions_controller.rb:520,545`; `model_pricing_service.rb:23` |
| Stranded-subagent "abandoned" sweep | live | `reap_abandoned_subagent_sessions_job.rb`; scheduled `recurring.yml:85` |

**Gaps:** revert-rate data supply is narrow; deep-analysis silently no-ops with no Anthropic
key (no degraded path); `ClassifySessionEventsJob` is a possible redundant/unenqueued path
(low-confidence dead-end); SkillInvocation has an endpoint with no confirmed dashboard consumer.

### Govern — **~85% live**
| Capability | Tag | Evidence |
|---|---|---|
| Resource 3-tier inheritance, immutable versions, rollback, effective_version_hash | live | `resource_configuration_service.rb:21,176`; `resource_version_service.rb:124` |
| Playbooks: conventions CRUD, subscriptions, assignments, AI extraction | live | `conventions_controller.rb`; `ConventionExtractionService` |
| PlaybookConventionCustomization applied to generated content | partial | render-applied `playbook_configuration_service.rb:156`, **no write route** |
| Role-tiered autonomy (coaching/collaborative/autonomous) | **partial (advisory only)** | `apply_role_modifiers` injects prompt *text* into agent resources only; **no capability/tool/action gating** |
| Severity-aware enforcement + deterministic-first detection (#264) | live | `cli/.../user_prompt_submit.rs:106,561`; deterministic never hard-Blocks |
| Override-resolution loop + override analytics read path | **live (docs say "open" — STALE)** | `ConventionResolutionService`, `ResolveConventionOverridesJob` scheduled; `convention_overrides_controller#index/#analytics`; dashboard page exists |
| Contradiction detector | live | `ContradictionDetectionService`; recurring + event-driven jobs |
| Deterministic test_quality_score (#266) | live | `test_quality_scorer.rb` (no LLM/egress); `ComputeTestQualityScoreJob` enqueued |
| Convention drift detector | planned-only | no code (skill-drift is unrelated marketplace concern) |
| A/B experiments (resource-version, D3 P0–P4) | live | `VersionBenchmarkService`; serve split `resource_configuration_service.rb:251`; manager+ gates `resource_policy.rb:158,167,169`; org opt-out default-ON `schema.rb:1201` |
| Legacy AgentExperiment/AgentConfiguration | **dead-end** | fully routed (`routes.rb:437`) but `ExperimentAssignmentService` called only by its own test; metrics permanently empty (`agent_experiment.rb:44`) |

**Gaps:** autonomy tiers are advisory not enforced (overstated pillar claim); two experiment
surfaces (one dead, still linked); convention-customization has no write path; convention
*drift* detector unbuilt.

### Ingest — **~75%; architecturally complete, strategic backend dormant**
| Capability | Tag | Evidence |
|---|---|---|
| Artifact model, upload, project-scoping, types (PRD/transcript/contract/email/slack/session) | live | `artifact.rb:47-53`; `routes.rb:483-507` |
| Retrieval seam (swappable factory) | live | `retrieval.rb:20-34` |
| **DO Knowledge Base backend** | **live (default)** | `RETRIEVAL_BACKEND` unset ⇒ `do_kb` |
| **pgvector + Voyage backend** | **flag-gated-off** | fully built + tested (`retrieval/pgvector_backend.rb`) but 3 default-off flags; nothing populates it in prod |
| ProcessArtifactJob indexing/chunking, `/artifacts/search` scope filter | live | `process_artifact_job.rb`; `artifact_search_service.rb:42-44` |
| Session vectorization + privacy gating + retract-on-delete | live | `session_vectorization_service.rb:73-77` |
| Google Drive import | live (needs creds) | `GoogleDriveService`; `import_from_drive`; `ImportGoogleDriveFileJob` |
| Agent email/Slack → Artifact **and indexed** | live (docs say "vectorization pending" — STALE) | `sendgrid_controller.rb:49`, `slack_controller.rb:87` enqueue ProcessArtifactJob; land `injectable:false` |
| Git-log ingestion | planned-only | no code (matches docs) |
| crewkit MCP context tools (5) | live | `cli/src/commands/mcp.rs:389,426,455,494,540`; auto-registered via `--mcp-config` |

**Gaps:** pgvector migration dormant (self-host blocked); unconfigured deployments silently
fall back to SQL ILIKE keyword search masquerading as "semantic"; git-log ingestion unbuilt.

### Collaborate — **~60%; code-complete, not customer-exercisable**
| Capability | Tag | Evidence |
|---|---|---|
| AgentIdentity model/controller/policy/serializer, rotate creds | live | `agent_identity.rb:14`; `agent_identities_controller.rb:91` |
| Inbound email (SendGrid, basic-auth + routing) | partial (DNS/MX-blocked) | `sendgrid_controller.rb:150`; `agent.crewkit.ai` unprovisioned |
| Inbound Slack (HMAC + replay window + routing) | live code (needs app install) | `slack_controller.rb:148` |
| Outbound AgentResponseJob (Haiku, injection-fenced, usage-metered) | live | `agent_response_job.rb:234`; `agent_usage_service.rb:8-15` |
| All inbound/outbound as Artifact | live | 4 artifact types, `injectable:false` until confirmed |
| Active-work "who's on what" (project-scoped) | live | `project_active_work_controller.rb:93-97` (leak fixed on main); `active-work-strip.tsx`, 30s poll |
| Org-wide active-work board | held | **PR #270 open, not on main** |
| Blueprint (models, CRUD, co-pilot chat, cloud-job spawn, session auto-match, TUI sidebar) | live (Phase E deprioritized, lightly proven) | `BlueprintExecutionService`; `BlueprintMatchingJob` at `sessions_controller.rb:255` |
| Cross-session awareness / work-updates / save-states | held | PRs #224/#225/#191 open |

**Gaps:** whole pillar not runnable E2E (DNS/MX + Slack install); org-level awareness in
unmerged PRs; Blueprint collaboration surfaces least battle-tested.

---

## 3. Phase status table (PLAN.md claimed vs verified)

| Phase / item | Claimed | Verified | Note |
|---|---|---|---|
| A — Flatten CrewkitSession→LlmSession | ✅ | ✅ | `crewkit_sessions` gone; `llm_sessions` root. Cosmetic "Run" back-compat stubs remain in CLI/dashboard types. |
| B — AgentIdentity + webhooks | ✅ | ✅ (DNS/MX external, as claimed) | all models/controllers/jobs present |
| C — Vectorization + active-work | ✅ | ✅ | jobs + endpoints present; project active-work leak fixed on main |
| D1 — Override-resolution loop | "remaining/open" | **SHIPPED — STALE claim** | `ResolveConventionOverridesJob` + `ConventionResolutionService` scheduled + event-driven |
| D1 — Override analytics read path | "remaining/open" | **SHIPPED — STALE claim** | `convention_overrides_controller#index/#analytics` + dashboard page |
| D1 — deterministic-first, stranded sweep, severity-aware | ✅ | ✅ | verified |
| D2 — Contradiction detector | ✅ | ✅ | real LLM service, not stub |
| D2 — test_quality_score (deterministic) | ✅ | ✅ | LLM-judge variant deferred (Loi 25) |
| D2 — Convention drift detector | pending | ✅ pending (no code) | accurate |
| D3 — Resource-version A/B (P0–P4) | ✅ | ✅ | on benchmarks stack; legacy AgentExperiment is the dead-end |
| F1 — crewkit MCP server | ✅ | ✅ | `crewkit mcp serve`, 5 tools, auto-registered |
| F2 — pgvector migration | (no ✅) | built-but-not-cutover | seam live, backend dormant (3 flags off) |
| F10 — Sidecar `--no-tui` | ✅ | ✅ observer; leader seam inert (as claimed) | shared registry confirmed not duplicated |
| F3–F9 (gateway promotion, analytics loops, IA overhaul/Bridge, TUI batch, qmd, LSP hover, remote attach) | (no ✅) | mostly planned/partial | Bridge rename not started; see §5 |

**Net stale PLAN.md claims:** the D-status snapshot line "only the override-resolution loop
+ override analytics read path remain" is stale on **both** counts — both shipped.

---

## 4. Prioritized gap list

| # | Gap | Impact | Effort | Tag |
|---|---|---|---|---|
| 1 | Loi 25 erasure (#256/#257/#259) + export (#229) not on main | **Compliance obligation unmet** (Quebec Loi 25 / PIPEDA) | M (built, needs merge + sign-off) | held |
| 2 | pgvector/Voyage migration dormant; DO KB is live | Blocks self-host; strategic vision item; empty ANN table until backfill | M–L (backfill runbook + cutover window) | flag-gated-off |
| 3 | Legacy AgentExperiment `/kit/experiments` dead-end still shipped + linked | Users land on a surface with permanently-empty metrics | S (remove/redirect route + nav + palette) | dead-end |
| 4 | Collaborate not E2E: DNS/MX for `agent.crewkit.ai` + Slack app install | "Agents participate" pillar can't be demoed to a customer | S–M (ops/provisioning) | partial |
| 5 | Org-wide active-work + cross-session awareness in 4 unmerged PRs | No org-level "who's on what" today | M (review + merge #270/#224/#225/#191) | held |
| 6 | Role-tiered autonomy is advisory (prompt text), not enforced | Overstated governance claim; no real capability gating | M–L | partial |
| 7 | Marketplace GA + `/kit/runs` dashboard + LLM-gateway capture flag-gated off | Finished features are dark to users | S (flag flips + sign-off) | flag-gated-off |
| 8 | Convention-customization write route absent; convention-drift detector unbuilt | Govern loop incomplete | S / M | held / planned-only |
| 9 | CLI token encryption key stored beside vault (not OS keychain) | Local-attacker token theft | M | live-but-weak |
| 10 | `crewkit run` has no CLI command surface (despite "Phase 1-3" claim) | Cloud jobs not CLI-submittable | M | planned-only (CLI) |
| 11 | GitLab/Bitbucket git integration, convention alert *delivery* | Partial integrations | M | partial |
| 12 | Revert-rate data supply narrow; deep-analysis no-ops without key | Analytics gaps for non-cloud-job teams | S | partial |

---

## 5. Vision-drift (vs `docs/product-direction.md`)

- **Retrieval (Ingest pillar core):** product-direction rides everything on "Artifact +
  ArtifactSearchService + DO Knowledge Base." PLAN F2 promises a *full production migration*
  to pgvector + Voyage. Reality: seam exists, migration is dormant. Direction and PLAN
  disagree with each other, and code matches neither's "done" framing.
- **LLM gateway (decision #7):** product-direction.md:75 states the server-side ingestion
  endpoint "was removed in 2026-01 (commit 18c6931)" and the gateway "proxies and file-logs
  locally only." **Code contradicts this** — `observability_controller.rb:99 ingest_gateway_requests`
  (route `routes.rb:625`) and the CLI `GatewayUploader` (`llm_gateway.rs:234`) both exist and
  are wired. Server capture is live (opt-in, per-org `llm_gateway_enabled` default-off). The
  doc is stale, not the code.
- **Dashboard→"Bridge" rename (Phase F5):** not started. Route root still `/kit`, sidebar
  still "Dashboard"; only stray unrelated "Bridge" strings in `ai-editor.tsx`.
- **`crewkit run` (decision #3, cloud jobs):** backend + orchestrator + dashboard exist, but
  the dashboard is flag-off and there is **no CLI `run` command** — so the user-facing
  "crewkit run" story is not reachable from the CLI.

---

## 6. Risks (most-severe-first)

1. **[Loi 25 — HIGH] Right-to-erasure and right-to-access not shipped.** Deletion (#256/#257/#259)
   and export (#229) are open PRs. Model-level anonymization plumbing is on main
   (`llm_session.rb:98-101`, `IS DISTINCT FROM` guard), but no user-facing request/execution
   flow. Blocked on Félix/Rémi + merge.
2. **[Security — MEDIUM] CLI token encryption key stored beside the vault.** `secure_auth.rs`
   deliberately "replaces the OS keychain"; `auth.vault` + its decryption key `.auth.key`
   both live `0o600` on the same disk. Local malware/attacker as the user recovers tokens →
   encryption is obfuscation, not protection. Contradicts the "OS keychain" posture in CLAUDE.md.
3. **[Loi 25 — LOW] Prefix-based secret redaction is best-effort.** `secret_redactor.rs`
   misses unprefixed secrets/PEM/base64; raw public-repo prompts+code egress to Anthropic + DO KB.
4. **[Tech-debt — MEDIUM] Two parallel experiment systems shipped.** Confusing dead-end
   (`/kit/experiments`) with empty metrics still user-reachable.
5. **[Doc-drift]** Multiple stale claims (see §7). Notably: the brief's own premise that the
   active-work leak is still on main is **false** — it's fixed and sentinel-pinned; #270 is a
   new feature, not that fix.

**Sentinel-coverage gaps:** (a) single-session content reads (`show`/`insights`/`transcript`/
`export`) rely on a per-controller `authorize_session_visibility!` `before_action` that no
sentinel enumerates — a dropped `before_action` (the exact recurring leak shape) wouldn't
trip CI. (b) Only Stripe webhook signatures are at sentinel tier; Slack/SendGrid/GitHub/
Bitbucket signature verification has ordinary controller tests only. (c) No sentinel over the
CLI vault key/permission invariant.

---

## 7. Doc-drift list (doc says X, code shows Y)

| Doc claim | Reality | Evidence |
|---|---|---|
| PLAN.md: override-resolution loop + analytics "remain" | Both shipped | `ConventionResolutionService`, `convention_overrides_controller#analytics` |
| CLAUDE.md: agent email/Slack "vectorization pending" | Already indexed (injection-gated, not vectorization-gated) | `sendgrid_controller.rb:49` |
| product-direction.md:75: gateway server ingestion "removed… local-only" | Live server endpoint + CLI uploader exist | `observability_controller.rb:99`; `llm_gateway.rs:234` |
| CLAUDE.md: "crewkit run Phase 1–3 shipped" | Dashboard flag-off; no CLI `run` command | `feature-flags.ts:10`; CLI `Commands` enum has no Run |
| CLAUDE.md/memory: "no dashboard share UI / annotation client" | Both exist | `share-dialog.tsx`, `share-annotations-panel.tsx` |
| CLAUDE.md: convention-override "no read/analytics path yet" | Analytics page + endpoints exist | `kit/analytics/convention-overrides/page.tsx` |
| Brief/memory: active-work leak "presumably still on main" | Fixed + sentinel-pinned on main | `project_active_work_controller.rb:87-96`; `active_work_visibility_sentinel_test.rb` |
| CLAUDE.md: convention parent-chain merge applied | Not yet merged | `conventions_controller.rb:82`; `playbook_configuration_service.rb:22` |
| CLAUDE.md CLI: "tokens in OS keychain" | Encrypted vault file, key on disk | `secure_auth.rs:3-4,39-40` |

---

## 8. Recommended next moves (shortest path to closing the biggest gaps)

1. **Merge the Loi 25 stack (#256/#257/#259 + #229)** after Félix/Rémi sign-off — this is a
   compliance gap, not a feature. Highest priority, lowest technical risk (code is built).
2. **Retire or redirect the legacy `/kit/experiments` dead-end** (remove route `routes.rb:437`,
   nav link `projects/[id]/page.tsx:311`, palette entry `search-index.ts:41`) and label the
   resource-benchmarks A/B surface as "Experiments." Cheap, removes user confusion + a whole
   dead subsystem (2 models, 2 tables, 2 policies, 2 services).
3. **Provision `agent.crewkit.ai` DNS/MX + Slack app install** to make the Collaborate pillar
   demoable; then review/merge #270 for org-level active-work.
4. **Sequence the pgvector cutover** (backfill runbook, dual-write on → shadow-read validate →
   flip `RETRIEVAL_BACKEND`) — the code is done; this is an ops migration, and it unblocks
   self-host.
5. **Refresh the stale docs** (PLAN.md D1 status, CLAUDE.md agent-vectorization/share-UI/
   convention-analytics/CLI-keychain lines, product-direction.md gateway note) — cheap, and
   the drift itself is eroding trust in the roadmap.
6. **Add three sentinels** (single-session-read `before_action` presence; non-Stripe webhook
   signatures; CLI vault key/perms) to pin the invariants that currently rely on convention.

---

*All claims above are cited to `file:line` on `main` as of 2026-07-20 and were adversarially
verified (reachable? tested? flag-gated? dead-ended?) rather than taken from docs. Items that
could not be verified are flagged inline (e.g. `ClassifySessionEventsJob` as a low-confidence
dead-end).*
