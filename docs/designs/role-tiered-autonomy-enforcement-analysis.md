# Role-Tiered Autonomy Enforcement — SDLC Analysis & Recommendation

**Status:** Analysis complete (gate-1 threat model done). **No implementation.** Awaiting product sign-off.
**Date:** 2026-07-22
**Author:** engineering (Arthur), with security-expert (gate 1), product-strategist, and a full-surface code map.
**Question asked:** is role-tiered autonomy *enforcement* necessary, wanted, and secure/safe for the rest of the project?

**One-line verdict:** **Don't build enforcement on the local `crewkit code` path — reframe the claim instead. Hold a single narrow, fail-closed slice for the cloud (`crewkit run`) path as a fast-follow to that path's GA. Do one small security hardening now regardless.**

---

## 1. What "role-tiered autonomy" is today (verified in code)

Two *independent* role axes are frequently conflated. Keeping them separate is the crux of the whole analysis.

| Axis | Field | Values | Enforced? |
|---|---|---|---|
| **Autonomy tier** | `OrganizationRole.agent_behavior` | coaching / collaborative / autonomous (`organization_role.rb:7-11`) | **No teeth — inert** |
| **Authz tier (RBAC)** | `UserOrganizationRole.permission_level` | owner / admin / manager / developer / viewer (`user_organization_role.rb:13-19`) | **Yes — every Pundit policy** (`application_policy.rb:85-119`) |

The autonomy tier is **inert**: every read of `agent_behavior` is storage, display, or serialization — none gates capability. The only role-derived "teeth" that exist are on the *other* axis:
- `permission_level` → Pundit (real, server-enforced API authz).
- One `OrganizationRole.level >= 90` check gating Resource creation (`resources_controller.rb:848-855`).
- `coaching_mode_enabled` → prepends advisory prompt **text** ("COACHING MODE") to agent content (`resource_configuration_service.rb:365-379`). Client-delivered; nothing enforces the model obeys it.
- `permissions` jsonb on `OrganizationRole` is **write-and-display-only** — permitted at `roles_controller.rb:179`, echoed back at `:196`, read by no policy or service. Free-form, `default {}`, no schema.

There is **no dashboard UI** to set the autonomy tier — org roles are created via seeds/console/org-bootstrap only; the member dialog edits `permission_level` only (`update-member-dialog.tsx:48,61`).

So the assessment's "advisory only" tag (§4 #6) is exactly right, and the gap is narrower than it sounds: it's not "enforcement is half-built," it's "the tier is a stored label wired to nothing."

## 2. The trust boundary (the fact that decides everything)

**Local `crewkit code`: the tool-execution decision is 100% client-side.** Claude Code runs on the user's machine; crewkit observes via `127.0.0.1` hooks and **never returns a tool allow/deny** — the `PreToolUse` handler returns `OK` (`hook_service.rs:732`). The user can pass `--dangerously-skip-permissions` (`tui/pty.rs:71-73`), edit the synced `.claude/` files, drop the settings file, or just run raw `claude`. **The API cannot block a tool call in a local session.** Any autonomy "enforcement" here is inherently advisory.

crewkit *does* emit blocking hook responses today — but only on two non-tool workflow events: the `Stop` verification gate (#25) and `UserPromptSubmit` for `severity=error` conventions (`hook_service.rs:961-966`). Both are honest, cooperative-user nudges. That's the ceiling of local enforcement.

**Cloud `crewkit run` → orchestrator → LXC: execution is server-provisioned, so a real control surface exists** — but it's at the *infrastructure* layer, not per-tool. The server controls repo snapshot, injected credentials, the `JobManifest` (budget/turns/timeout/output-branch), the LXC boundary, network/fs isolation, and a scoped agent JWT. It runs Claude Code with `--dangerously-skip-permissions` *inside* the container (`orchestrator/scripts/run-agent.sh:64,84`) and reads `agent_behavior` nowhere. So the server can gate *what enters the container and the blast radius* — which is exactly where a coaching tier could be denied deploy/push credentials + a human-approval gate. This is a genuine, fail-closable control that has no local equivalent.

## 3. Necessary? — No (it's a wording problem, not a capability gap)

The Govern pillar stands on four legs, and three are real and shipped: conventions/playbooks (**enforced** via the severity deny path), resource 3-tier inheritance (live), background safeguards (contradiction / test-quality / convention-drift detectors, live). The pillar's credibility does not rest on the fourth leg — and one of the other legs *already hard-blocks*. What's dishonest is the *word* "autonomy," which promises a capability ceiling the code doesn't have. Fixing the word closes the "overstated claim" finding at near-zero cost. **The convention hard-block is currently under-sold; surfacing it is a net credibility gain.**

## 4. Wanted? — No, not on the developer path

- Buyer personas want **visibility, nudges, and standards** (all exist), not hard capability-gating on their developers' AI agents — that's friction they'd disable.
- The tier names (coaching → collaborative → autonomous) frame a **trust/progression ladder, not a lockdown ladder.** Re-purposing them as a permission-enforcement scale fights the product's own metaphor.
- **Claude Code already owns the tool-permission layer** (allow/deny lists, permission modes, deny hooks). crewkit rebuilding a per-role capability gate re-implements the harness's job at a worse altitude.
- Industry pattern is *guide during, gate at the boundary* (linters/AI advise mid-stream; branch protection / required CI / PR review enforce at merge). Per-keystroke per-role handcuffs are the thing developers reflexively turn off. The pivot thesis itself says CLI-wrapper mechanics are **not** the moat (product-direction.md:7).

## 5. Secure & safe? — Local enforcement is theater; only the cloud path is enforceable

From the gate-1 threat model. Two mutually exclusive adversary models — a design must pick one:

- **Adversary A — the over-eager agent harming an honest user (the "seatbelt").** Client-side advisory-plus is *acceptable and useful* here; the user isn't the attacker.
- **Adversary B — the determined insider exceeding their tier (the "containment cell").** The attacker owns the execution host → **no client-side control survives.** Achievable *only* on the cloud path.

**The only defensible product goal for the local path is Adversary A. Adversary B is unachievable locally and must not be claimed.**

**Blocking constraints any future design MUST satisfy** (carry these into gate 2 if a slice is ever built):

1. **No false containment claim.** Docs/UI/API must not state or imply the local path *prevents* exceeding a tier. (Newspaper test: "crewkit told orgs it blocked junior deploys; a junior deployed anyway in 10 seconds.")
2. **Local tiering can't be a security control because it must fail *open*.** The CLI's core guarantee is "every path ends with Claude launching" — a role-fetch failure must fail-open or the product breaks. A control that fails open on a network blip isn't a control. **Cloud-path gating MUST fail *closed*** (can't resolve tier/policy → deny the privileged action).
3. **Close self-escalation.** `OrganizationRolePolicy#update?` currently lets any admin edit the `agent_behavior`/`permissions` of the role they themselves hold → coaching→autonomous self-promotion in one PATCH (`roles_controller.rb:179`, policy `:26-38`). Inert *today* (tier gates nothing) but must be closed **before** the tier is load-bearing: a user cannot raise the tier of a role assigned to themselves; tier changes PaperTrail-audited; only owner / a separated duty raises a tier above the editor's own. (`UserOrganizationRolePolicy:33-34` already blocks self-*reassignment* — the role-*edit* backdoor is the gap.)
4. **`permissions` jsonb is a confused-deputy / injection vector.** If enforcement ever keys off it: strict server-validated allowlist schema (enum values, no free strings), unknown keys fail *safe*, and any value rendered into hook config / frontmatter / shell-adjacent context is treated as untrusted and escaped.
5. **Strict org-scoping.** Role/permission config flows through `resources/effective` merge and is written to `.claude/` on disk — prove every read is Pundit-org-scoped (this is the recurring session-visibility leak class; grep the new read surface).
6. **Server-side enforcement = new synchronous pre-execution egress → Loi 25 gate.** To decide deny/allow server-side, the API must inspect tool input (shell command, target path, maybe file content) *before* execution — exactly the data the sanitizer currently **strips** (`event_sanitizer.rs`). Real server enforcement means egressing protected data synchronously → explicit Loi 25 / per-client legal sign-off (same bar as the deferred LLM-judge test-quality work).
7. **Keep the two axes categorically separate.** `agent_behavior` must never become an implicit authz path for API mutations — Pundit `permission_level` remains the only server-enforced authz.

## 6. Recommendation

### 6a. Now — reframe the claim (product decision; docs-only, no code)

Split the one overstated pillar line into the two mechanisms that actually exist, named accurately. Draft wording:

- **Convention enforcement (the enforced lever, currently under-sold):** "Conventions carry severity; `severity: error` conventions hard-block the agent at the deny hook. Lower severities challenge and log."
- **Role-tiered agent behavior (the advisory lever):** "Member roles (coaching / collaborative / autonomous) shape agent behavior via prompt-level modifiers — e.g. coaching mode makes agents explain and mentor. These tune how the agent works *with* a person; they are behavior-shaping, not capability gates."

Targets: `docs/product-direction.md` pillar 2, `CLAUDE.md` Four Pillars → pillar 2. **Held for Félix/Rémi** — this re-characterizes a marketed pillar, so it's a positioning call, not a unilateral doc-drift fix.

### 6b. Now — one security hardening, regardless of the above (in my lane; needs its own SDLC pass)

Close the **self-escalation gap** (constraint #3). It's a latent authz bug that becomes exploitable the moment the tier is ever wired to anything. Small, self-contained, and worth pinning with a sentinel. Recommend implementing + full gate-3 review as a separate task. *(Not done in this planning pass.)*

### 6c. Later — the one narrow slice worth building, and only when unblocked

**Cloud-path, fail-closed tier enforcement.** A coaching/collaborative tier on `crewkit run` = the orchestrator withholds deploy/push credentials and gates privileged actions behind human approval (composes with the existing `CloudJobPrService` review boundary). This gates *a robot with no human at the keyboard*, so it's genuine, friction-free, and the honest home for Adversary B.

**Blocked twice over:** `crewkit run` has no CLI command and its dashboard is flag-off (assessment §5 / gap #10) — the path it would protect isn't reachable yet. So this is a **fast-follow to cloud-exec GA**, tracked behind Phase F, gated by constraints #2/#4/#5/#6 above. Not work for now.

### What NOT to do

Do **not** build role→capability gating on the interactive developer path, in any form labeled "enforcement." It is advisory by construction in a local-CLI product, and shipping it as a control is a liability, not a feature.

## 7. Sequencing

Ranks **last** among open items. Above it: Loi 25 erasure/export merge (legal, HIGH, code built), pgvector cutover (unblocks self-host), Collaborate-pillar E2E (DNS/MX + Slack — the pillar can't be demoed), org-wide active-work PRs. The only piece actionable now is the 6a reframe (cheap, product sign-off) and the 6b hardening (small, my lane).

---

*Inputs: gate-1 security threat model (security-expert), product-necessity analysis (product-strategist), and a full end-to-end code map. All claims cite `file:line` on `main` as of 2026-07-22. This satisfies SDLC gate 1; a design for slice 6c (if ever pursued) must pass gate 2 against the §5 constraints before any build.*
