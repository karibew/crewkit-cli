# Superpowers Integration Plan

> Status: **planning** (multi-agent design pass, 2026-05-23). Execution tracked in the
> integration EPIC (see "Issue map" below) and EPIC #135.
> This document consolidates the design output of a 10-teammate review/planning wave.
> No production code decisions here are final until the linked issues are scoped.

## TL;DR

Two tracks, derived from evaluating [obra/superpowers](https://github.com/obra/superpowers) (14 skills, MIT):

- **Track A — Reference marketplace.** Surface superpowers' skills *as-is* from the crewkit
  marketplace: license-respecting, version-pinned, drift-aware (fed by the new `skill-watch`
  skill). Orgs adopt a reference; it syncs to `.claude/skills/` via the CLI.
- **Track B — Built-in disciplines.** Ship crewkit's *own* cooked-in capabilities — a lightweight
  skill-discovery hook + native **plan / debug / investigate** — derived from superpowers'
  *patterns* (brainstorming, writing-plans, systematic-debugging, verification-before-completion)
  but owned, governed (3-tier resources, role tiers), and measured (`ResourceStat`) by crewkit.

The strategic split: **Track A makes crewkit the place teams find and trust skills; Track B makes
crewkit the place teams can't easily leave.** Installing superpowers directly gives a frozen clone
with no drift signal, no team-wide pinning, no audit trail, no telemetry — crewkit adds exactly
those.

## What already shipped this session

- **4 watcher skills** with scheduled GitHub Actions (least-privilege, hardened):
  - `skill-watch` — tracks upstream skill repos (obra/superpowers, anthropics/skills,
    anthropics/claude-code CHANGELOG, hesreallyhim/awesome-claude-code). Weekly cron.
  - `claude-code-prompt-watch` — snapshots Claude Code's built-in prompts from the published
    npm artifact (manifest = `{sha256, len, label}` only; **no verbatim proprietary text** committed;
    raw extract gitignored). Daily cron.
  - `codex-prompt-watch` — tracks `openai/codex` base/per-model prompts (Apache-2.0, committed verbatim). Weekly.
  - `opencode-prompt-watch` — tracks **`anomalyco/opencode`** (the canonical home, 164k★, MIT — the
    project moved from `sst/opencode`) prompts: 42 system/agent/tool/command prompt files. Weekly.
- **EPIC #135 + #136–#144** — discipline-skill enhancements derived from superpowers (TDD Iron Law,
  per-task spec-compliance review, verification-before-completion, brainstorming gate,
  systematic-debugging, receiving-code-review, CSO description discipline, rails-api reference skill,
  seed default platform skills/commands).

## Key findings that shape the plan (verified against the codebase)

1. **The reference plumbing already exists.** `ExternalSkillSource` → `GitHubSkillParserService` →
   `ExternalSkillSyncService` already import `skills/*/SKILL.md` from GitHub repos into platform/org
   `Resource` records with upstream provenance in `import_metadata` (sha, download_url, repo_url).
   Stripe/Cloudflare are already seeded this way (`api/db/seeds.rb`).
2. **BLOCKER: the parser only handles a flat directory.** `GitHubSkillParserService` does a single
   Contents-API call and keeps only top-level `.md` files. superpowers is laid out as
   `skills/<name>/SKILL.md` (one dir per skill), so a naive seed imports **zero** skills. Nested-layout
   support is a hard prerequisite (Track A, issue A1).
3. **`forked` ownership mode is dead in the DB.** `Resource::OWNERSHIP_MODES = %w[inherit standalone]`
   and the check constraint dropped `forked`. A marketplace "fork" must be a `standalone` snapshot, or
   `forked` must be revived in a separate migration. Reference adoption = `inherit` + pinned version.
4. **No license/attribution columns** exist anywhere — required for redistributing MIT content (A2).
5. **The SessionStart hook is wired but inert.** `cli/src/hooks/registry.rs` already emits a
   `SessionStart` hook; `hook_service.rs::handle_session_start` only logs and returns `OK`. The
   `ClaudeCodeHookResponse::with_context` / `additionalContext` mechanism is already used by
   `UserPromptSubmit`. So the built-in hook is a ~30-line addition, not a new subsystem (B1).
6. **Claude Code prompts are not in a bundled `cli.js` anymore.** As of v2.1.150 the npm package is a
   thin wrapper; the prompt-bearing code is a native binary in the per-platform optionalDependency.
   `claude-code-prompt-watch` scans that binary. (Documented in the skill.)
7. **Issue-tree correction:** epic #135's `#141` is **receiving-code-review**, not writing-plans — so
   the "plan-shape" discipline (`writing-plans`) is untracked and needs a new issue (B2).
8. **`.claude/skills/skill/SKILL.md` is misnamed:** `name: skill` but the body is a Slack GIF creator
   (leftover upstream sample). Must be fixed before seeding default skills (D1).

---

## Track A — Reference Marketplace

### Data model (3 new tables; reuse `resources`/`resource_versions`)

```
external_skill_sources          [EXISTS — reuse]   repo-grained upstream source
  └─ marketplace_skill_listings  [NEW]             per-skill catalog entry
       └─ skill_adoptions        [NEW]             org/project ⇄ listing (mode reference|fork, pinned_ref)
            ├─ resource_id ─▶ resources             [EXISTS] fork materializes a Resource (+ResourceVersion)
            └─ skill_drift_statuses [NEW] 1:1       adopted vs current upstream; fed by skill-watch
```

- All new tables: bigint PK + uuid `external_id`, `acts_as_paranoid`, `has_paper_trail` (per repo
  conventions). Additive-only migrations (strong_migrations-safe; concurrent index creation).
- **Reference adoption** = `Resource(ownership_mode: "inherit", import_source_id, pinned_parent_version_hash)`.
  No content copied; resolved through the existing 3-tier merge. Mirrors the existing `install` verb.
- **Fork adoption** = `Resource(ownership_mode: "standalone")` snapshot with `import_source_id` kept as
  provenance; immutable `ResourceVersion` created by existing hooks. Drift is informational only.
- **Backfill:** existing `resources` rows with `import_source_id` set become `fork` adoptions
  (they're already snapshots), pinned to `import_metadata['sha']`.

### API (`/:org/marketplace/skills`)

| Method | Path | Purpose |
|---|---|---|
| GET | `/marketplace/skills` | list listings (platform-curated + org sources); `?source_id`, `?stack`, `?adopted`, `?q` |
| GET | `/marketplace/skills/:source_id/:skill_slug` | listing detail + upstream version + drift + adoption state |
| POST | `/marketplace/skills/:source_id/:skill_slug/adopt` | adopt (`mode=reference\|fork`, `pin_to`, `project_id`) |
| DELETE | `/marketplace/adoptions/:id` | unadopt (soft-delete) |
| GET | `/marketplace/adoptions` | list adopted + per-item drift; `?drift=behind` |
| GET | `/marketplace/adoptions/:id/check_updates` | live drift recompute |
| POST | `/marketplace/adoptions/:id/pin` | pin to a specific upstream version |
| POST | `/marketplace/adoptions/:id/upgrade` | move pin to latest (reference mode only) |
| POST | `/marketplace/adoptions/:id/assign` | assign to a project (reuses `project_skills`) |

JSON:API + HATEOAS `_links` (the `upgrade`/`pin` links emitted conditionally only when
`drift == "behind"`, mirroring the existing `:upgrade` conditional link). Pundit: browse = any member;
adopt/pin/upgrade = admin/manager (reuse `ResourcePolicy#install?`), behind a plan-feature check.
OpenAPI spec update + `STRICT=1` validation required.

Once adopted, a skill is a normal `Resource` — it appears in `/resources/effective`,
`/resources/:id/versions`, `/resources/:id/stats` for free. The marketplace is a discovery +
upstream-relationship facade, **not** parallel storage.

### Drift (skill-watch → marketplace)

Detection stays **file-side** (skill-watch already owns snapshots, `.skill-watch-meta`, diff logic,
the weekly Action). A thin bridge mirrors the verdict into `skill_drift_statuses`: either the Action
POSTs `{owner, repo, path, head_sha, version, checksum}` to a sync endpoint, or a recurring
`MarketplaceDriftSyncJob` reads the committed snapshots. The API only *reads* drift — never computes
it in the request path. **Watch pointer ≠ adopt pointer**: drift never silently re-pins an adoption.

### Dashboard UX

- **IA:** `Skills` sidebar → `Marketplace` (browse) / `Adopted` (govern, at `/kit/skills`) / `Builder`.
  Adopted page has tabs: **Built-in** | **Adopted** | **Drift (n)**.
- **Source-badge taxonomy** (icon + color + label — the legibility primitive): `Built-in` (Sparkles,
  blue), `Platform` (Shield, indigo), `External` (GitBranch, amber), `Org` (Building2, purple).
- **Adopt dialog:** reference (default, recommended — keeps drift tracking) vs fork (snapshot), with
  pin-version select and org-vs-projects assignment. Reuse `resource-fork-dialog.tsx`.
- **Drift / Update-Review sheet:** "what changed" summary + reuse `version-diff.tsx`; "Pin & update"
  advances the pin (optimistic, syncs next CLI session). Drift count is the one amber sidebar badge.
- **Reuse heavily:** `version-diff`, `version-history`, `upgrade-banner`, `resource-inheritance-viz`,
  `security-score-badge`. New components: `source-badge`, `drift-badge`, `upstream-attribution`,
  `adopt-dialog`, `update-review-sheet`, `sync-status-cell`, `builtin-capability-card`.
- **Sync is eventual, not live** — every "Synced ✓" reflects the last CLI session; never imply a socket.

### Seeding superpowers

Seed **one** `ExternalSkillSource` (repo `obra/superpowers`, branch `main`, `nested_layout: true`,
`license: MIT`, attribution) + a post-deploy sync — **do not** hand-copy 14 SKILL.md bodies into a
migration. Depends on the nested-parser fix (A1) and license columns (A2). Attribution is written into
the synced file's frontmatter so on-disk copies stay license-clean.

---

## Track B — Built-in Disciplines

### B1 — Lightweight SessionStart skill-discovery hook (CLI)

crewkit's cooked-in equivalent of superpowers' bootstrap, **without the token tax**. Measured contrast:

| | superpowers | crewkit |
|---|---|---|
| injected per session | full `using-superpowers` SKILL.md (~1,300 tok) | ~200–250 tok nudge + skill index |
| re-injected on `/compact`? | yes (matcher includes `compact`) | no (`startup\|resume` only) |
| skill catalog included? | no (forces a Skill-tool round-trip) | yes (name + trigger) |
| register | `<EXTREMELY_IMPORTANT>` / "not negotiable" | one scoped imperative |

- **Mechanism:** `handle_session_start` returns `hookSpecificOutput.additionalContext` (the channel
  `UserPromptSubmit` already uses) with: (a) a one-paragraph "check adopted skills before non-trivial
  work" habit; (b) a name+trigger index of the org's adopted skills (descriptions only, bodies load
  lazily via the Skill tool); (c) crewkit entry points (`/plan-feature`, `/document-issue`,
  `/logs-watchdog`, `/verify`); (d) a subagent guard + CLAUDE.md-precedence line + dismissal pointer.
- **Source-gated** to `startup|resume`, hard byte-cap (unit-tested so it can't regress into full-text
  injection), reads `.crewkit/skills.json` written during the existing resource sync (no hot-path
  network call). Overridable via an `enable_session_bootstrap` flag and `crewkit config skill-discovery off`.
- **Lowest-priority context channel** — strictly weaker than CLAUDE.md / `.claude/rules`.

### Plan / Debug / Investigate (skills, not Rust subcommands)

These are methodologies the model follows mid-conversation — author them as skills (so they ship to
orgs as platform-tier resources via sync), with thin command entry points where a typed verb helps.

| Capability | Surface | Status |
|---|---|---|
| **plan** | Enhance `/plan-feature` (brainstorming hard-gate, #139) + **new `writing-plans` skill (B2)** | #139 exists; writing-plans untracked |
| **debug** | **`systematic-debugging` skill (#140)** + thin `/debug` command (B4) wiring `/work`'s failure path | #140 exists; /debug new |
| **investigate** | **new `investigate` skill (B3)** (read-only root-cause/triage) + gate completion claims via verification-before-completion (#138, B5) | investigate new; #138 exists |

- All three authored to the CSO description discipline (#142) and seeded at the platform tier (#144) so
  customer orgs inherit them. Every gate stays **overridable** (explicit user waiver) — consistent with
  the "pending tests first, intelligent coverage later" philosophy.
- `investigate` instruments crewkit's real boundaries (CLI↔API, API↔dashboard, `.crewkit/debug-latest.log`)
  then hands off to `systematic-debugging` for the fix; `debug` enforces root-cause → single hypothesis
  → fix-with-test + "3 failed fixes ⇒ question the architecture".

---

## Track C — Security & Ops (gates GA)

**Premise:** an adopted skill is natural-language instructions an AI agent reads and *acts on* with the
developer's local privileges — there is no sandbox between "skill text" and "agent behavior." Treat
every adopted skill as untrusted code.

Required guardrails before any third-party skill auto-syncs into `.claude/skills/`:

1. **Pin to a reviewed `commit_sha` + per-file checksum, never a branch.** The CLI verifies each file's
   checksum against the pinned manifest before writing, and refuses on mismatch or downgrade (rollback).
   Branch tracking is for *detection* only.
2. **Enforced human review gate** before adopt (audited via Paper Trail); reviewer sees the exact pinned bytes.
3. **Drift requires explicit re-approval** — skill-watch never silently re-pins. Watch pointer ≠ adopt pointer.
4. **Skill-bundled scripts never auto-run** — explicit per-invocation consent, sandbox (no network,
   scoped FS, no ambient credentials), default-deny.
5. **Injection-marker content scan** at adopt-time and on every drift (override phrases, exfil patterns,
   security-control-weakening language, embedded creds/URLs) — a tripwire, not a guarantee.
6. **Trust tiers** — `platform-curated` vs `community`, enforced in Pundit + CLI.
7. **Provenance record** per adoption (repo, commit_sha, license, approved_by, checksum) for breach answerability.

**Watcher-workflow hardening** (script/YAML `${{ }}`-injection, `gh issue create` fence-breakout, manifest
proprietary-text leak, idempotency, regex escaping — already fixed this session; the remaining items are
repo-settings/design): open a **PR instead of pushing snapshots to `main`**, add `.github/CODEOWNERS` over
`.claude/skills/**` + `.github/workflows/**`, **SHA-pin** `actions/checkout` + `setup-node`, harden
`tar -xz` (`--no-same-owner` + entry validation) in the claude-code capture.

---

## Phasing

**P0 — Foundation (ship now; mostly tracked).** #144 (seed pipeline — the keystone), #136, #137, #138
(keystone discipline), #143; the 4 watcher skills (already landing).

**P1 — The two tracks land.**
- Track A: A1 (nested parser — blocker) → A2 (license) → A3 (data model) / A4 (API) / A5 (UX) → A6
  (seed superpowers) → A7 (drift wiring). C1 (security guardrails) gates GA.
- Track B: B1 (hook) → B2/B3/B4/B5 (plan/debug/investigate skills + wiring); #139/#140/#141/#142.

**P2 — Compounding loops.** Marketplace + built-in usage analytics (`ResourceStat`), coaching-signal
feedback, drift-to-adoption SLA dashboarding, fork-rate as a curation signal.

## Success metrics (90-day targets)

| Metric | Target |
|---|---|
| Orgs adopting ≥1 referenced skill | 40% |
| Avg superpowers skills adopted per active org | ≥4 |
| Sessions/week invoking built-in plan/debug/investigate | 10+ per active team |
| Median drift-to-adoption latency | < 7 days |
| Correctness/quality/verification score lift (built-ins vs not) | +15% relative |
| Fork rate (inverse signal — high = curation gap) | < 20% |
| Discovery-hook tokens added per session (anti token-tax) | flat or lower vs eager-load |

## Issue map

- **EPIC #135** — Adopt superpowers patterns into default skills (foundational discipline skills).
  Children #136–#144.
- **New integration EPIC** — crewkit skill marketplace + cooked-in disciplines. Children:
  - Track A: nested parser (blocker), license/attribution, marketplace data model, marketplace API,
    marketplace dashboard UX, seed obra/superpowers, skill-watch→marketplace drift wiring.
  - Track B: built-in SessionStart hook, `writing-plans` skill, `investigate` skill, `/debug` command,
    gate `/document-issue` & `/logs-watchdog` on verification.
  - Track C: marketplace supply-chain guardrails (gates GA), harden watcher workflows.
  - Cleanup: fix the misnamed `.claude/skills/skill/SKILL.md`.

## Risks & tradeoffs

- **Reference vs fork** — default to version-pinned reference + explicit "adopt new version"; treat high
  fork rates as a curation signal, not a failure.
- **Token-tax anti-pattern** — the discovery hook MUST be on-demand (reference index), never eager-load.
- **Built-in vs reference confusion** — built-ins are the opinionated default (pit of success);
  references are community alternatives. The hook prefers built-ins.
- **Supply chain** — Track C guardrails gate GA; internal/alpha can precede.
- **License** — "as-is reference" preserves obra/superpowers attribution + license; non-negotiable for GA.
