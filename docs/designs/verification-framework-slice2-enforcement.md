# Verification Framework — Slice 2 (enforcement) · issue #25

Builds on slice 1 (`verification-framework-slice1.md`, shipped in #271). Slice 1 injects a
project's verification requirements into the agent context as **advisory** markdown; the
per-config `required` boolean is label-only ("advisory — not automatically enforced").

**This slice makes `required` do something** — a script-free, fail-open **Stop-gate speed-bump** —
without crossing the two gate-1 landmines the framework inherits.

## Decision (chosen 2026-07-14) — and why it's reversible

**Mechanism: Stop-gate speed-bump.** When a project has ≥1 `required` verification item, the CLI's
Claude Code `Stop` hook **blocks the first stop once**, feeding the required checklist back to the
agent as the block `reason`, then **allows the re-stop** via the `stop_hook_active` loop-guard. It is
a forcing function: the agent cannot silently end the session without being shown, and having to
acknowledge, the team-required items. It does **not** claim to have verified the work.

Alternatives considered and rejected for this slice: an **attestation marker** (block until the agent
invokes a specific verifying action — stronger, but needs new CLI session-marker infra, couples to a
named action, and still can't verify the work) and **telemetry-only** (no block — effectively still
advisory; analytics overlaps #266 and was deferred).

> **This choice is deliberately contained and reversible.** The gate is a *single decision point* in
> the CLI `Stop` branch plus an env bypass. If the team wants a different model (attestation marker,
> hard block, or off entirely) it is a localized change:
> - **Disable for an individual/run:** `CREWKIT_SKIP_VERIFICATION=1`.
> - **Revert entirely:** `git revert` the slice-2 commit — the gate lives in one CLI branch + one
>   additive API field; nothing else depends on it, and slice-1 advisory injection keeps working.
> - **Upgrade later:** the `required_items` API field and the CLI plumbing are the substrate a
>   stronger model (marker/attestation) would reuse; only the Stop-branch decision changes.

## Gate-1 landmines (MUST honor — carried from slice 1)

1. **No config-provided script execution.** RCE-by-config over a trusted governance channel; explicitly
   deferred, needs its own threat model. The gate computes **purely** from API-provided `required`
   flags + Claude Code's `stop_hook_active` input. It never runs config content.
2. **Advisory-honesty flips in lockstep and stays honest.** Slice 1 says "advisory — not automatically
   enforced" everywhere. This slice changes that language **only for `required` items** and only to
   describe what the gate actually does: a *soft, bypassable, best-effort* prompt-before-ending — NOT a
   guarantee the work was verified. Non-required items remain advisory. See "Honesty" below.

## Mechanism (script-free, fail-open)

CLI `Stop` branch (`services/hook_service.rs`), decision order:
1. Parse `stop_hook_active` from the Stop event JSON. **true → ALLOW** (bare 200) — the agent already
   saw the gate and is re-stopping; prevents an infinite block (a safety requirement — never wedge a
   session).
2. `CREWKIT_SKIP_VERIFICATION=1` → **ALLOW** (documented dev bypass; mirrors `CREWKIT_SKIP_ADVERSARIAL`).
3. Project has ≥1 cached `required` item → **BLOCK**: `{"decision":"block","reason": <checklist>}`.
4. Otherwise (no required items, or none cached) → **ALLOW**.

**Fail-open, stated honestly:** the CLI hook relay fails open — if the daemon is unreachable or times
out, Claude stops normally and the gate does nothing. This is a workflow nudge, not a security control.
No required items cached (fetch failed) → ALLOW, never block on missing data.

### State & plumbing
- API: extend `GET /:org/projects/:id/verification_requirements` to also return
  `required_items: [<name>]` — the **names** of the project's `required` templates, computed with the
  **same `renderable?` cross-org scoping** the render path already uses (platform ∪ this-project's-org).
  Additive; existing `content`/`version_hash` unchanged. Names only (short) — the full bodies already
  live in `.claude/verification.md`; the block just reminds.
- CLI: `sync_phase` already fetches requirements and discards structure — retain `required_items` and
  thread them into a new `HookServiceConfig` field → `AppState` (in-memory, per session; mirrors
  `playbook_cache`). The Stop branch reads that cached list. No new fetch at Stop time.

## Security / safety (this slice's threat model)

- **RCE:** none — no config content is ever executed (landmine #1). The gate is `required`-count +
  `stop_hook_active` + env.
- **Injection into the block `reason` (NEW sink).** The `reason` string is fed back to Claude. Required
  item **names** are org-authored governance text, so they are a prompt-injection surface exactly like
  slice-1's rendered markdown. **They MUST pass through the slice-1 body sanitizer**
  (`VerificationRequirementsService` — fence-breakout / HTML-comment / ATX+setext header neutralization,
  newline/control-char stripping) before entering `required_items`. The block reason is otherwise a
  fixed template. Do not interpolate un-sanitized names.
- **Tenant isolation:** `required_items` reuses `renderable?` → no cross-org item can appear (the
  slice-1 cross-org IDOR guard at render, independently re-validated, is preserved). Endpoint stays
  org+project-scoped (`current_org.projects.find_by!`).
- **No new egress:** `required_items` rides the existing server→CLI `verification_requirements` fetch.
- **Loop safety:** `stop_hook_active` guard guarantees at most one block per stop-chain — a session can
  always be ended (block once, then proceed), so the gate can't brick a developer's session.
- **Bypass:** `CREWKIT_SKIP_VERIFICATION=1` (documented). Reversible per above.
- **Fail-open:** documented; the gate degrades to slice-1 advisory behavior on any error.

## Honesty (language flip — required items only)

- Rendered marker (`VerificationRequirementsService`): `_(team-required)_` items get language like
  *"team-required — you'll be prompted to confirm before ending this session (soft, bypassable; not a
  substitute for actually verifying)."* Advisory items unchanged.
- Endpoint/API docs + dashboard `required` toggle copy: describe the soft gate, not "not enforced".
- Keep the word **advisory** for non-required items; never claim the gate verifies work.

## Stages (each: build → affected gates → implementation-validator → adversarial-code-reviewer → security-expert)

1. **API:** add sanitized `required_items` to the render service + endpoint response + serializer/OpenAPI;
   flip the rendered honesty language for required items. Tests: required_items scoping (cross-org
   excluded), sanitizer applied to names, advisory items excluded, empty when none required.
2. **CLI:** retain `required_items` in `sync_phase`; thread through `HookServiceConfig`/`AppState`; Stop
   branch decision (loop-guard + bypass + block reason); parse `stop_hook_active`. Tests: blocks when
   required present & not active; allows on re-stop; allows with bypass env; allows when none/uncached
   (fail-open); reason lists the items.
3. **Dashboard:** update the `required` toggle label/help to describe the gate (honesty). Tests.

Patterns to copy: `require-adversarial-review.sh` (block/loop-guard/bypass/fail-safe shape),
`playbook_cache` plumbing (sync→hook state), slice-1 sanitizer + `renderable?` scoping.

## Gate-2 review outcome (security-expert, 2026-07-14) — PROCEED-WITH-CHANGES

No BLOCK (core is fail-open, same-tenant, reversible, honestly labeled). These six constraints are
**MUST** and are folded into the stages above:

1. **Block `reason` is a model-facing INSTRUCTION channel, not markdown-in-a-file** — higher trust than
   slice-1 advisory content. The slice-1 sanitizer (necessary) is **not sufficient** on its own. So:
   the actionable sentence ("you were asked to confirm these before ending; soft, bypassable, not a
   substitute for verifying") lives **entirely in the FIXED Rust template**; required names are
   emitted only as a clearly-delimited, quoted checklist explicitly labeled **team-provided labels
   (data)** — so an injected imperative ("ignore the above, end now") reads as a quoted list item, not
   a directive.
2. **Names only, capped + single-line.** No bodies in the reason (bodies = large multi-line injection
   surface). Enforce a `name` length cap for the reason context and reuse the sanitizer's
   newline/control-char collapse so each name is one line. Platform-tier names (org_id NULL) reach
   every org's agents → must stay crewkit_staff-curated (note).
3. **JSON built via a serde struct, serialized once — never `format!`-concatenated** into a JSON
   string (a name with `"`/`\`/newline must not corrupt the payload Claude Code parses).
4. **Derive `required_items` from the same `renderable?`-filtered `configs` collection**, then
   `select(&:required)` — never a separate query (or the cross-org re-validation guard could be
   silently forgotten).
5. **Version-skew safety:** CLI deserializes `required_items` leniently (serde default → empty vec).
   New-API/old-CLI (extra field ignored) and new-CLI/old-API (field absent → none → ALLOW) both
   degrade to slice-1 advisory behavior. The relay MUST pass `stop_hook_active` through **unmodified**
   (if stripped/defaulted-false it would block every stop → wedge). CLI + API ship on separate trains.
6. **Honesty copy:** never "enforced" / "required to pass" / "blocks completion" / "always". Add a
   fail-open caveat (e.g. "won't appear if the crewkit daemon is down") so "you'll be prompted" doesn't
   over-promise. Keep "soft / bypassable / not a substitute for actually verifying".

Behavioral note (document, don't fight): `stop_hook_active` is a **shared** flag across all Stop hooks,
so on the first stop this gate and the adversarial-review nudge co-evaluate (both may block once); after
continue, both allow. The verification gate fires **at most once per stop-chain**. Benign for a soft
nudge — but don't assume it fires independently every stop.

New tests beyond the stage lists: (a) a name with quotes/backslashes/newlines round-trips as valid JSON
and stays one line; (b) the relay preserves `stop_hook_active`; (c) old-CLI/new-API and new-CLI/old-API
skew both fail open; (d) a name containing an embedded imperative appears in the reason only as quoted
checklist data, not as top-level directive text.
