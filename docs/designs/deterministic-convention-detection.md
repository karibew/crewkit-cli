# Design: deterministic-first convention-violation detection

> Status: **gate-2 revision (round 2)** · Author: planning session 2026-07-08 ·
> SDLC gate 1 ✅ (12 constraints) · gate 2 BLOCK round 1 (2 control-flow bugs + widen-gate decision)
> — folded in below, awaiting re-review · build not started.
> Component: CLI only (`cli/src/hooks/convention_handlers/`). No API/tenancy/egress surface added
> — this change *reduces* Haiku egress.

## Problem

Today every `contains_tech_keywords`-positive `UserPromptSubmit` triggers an LLM call
(`HaikuService::check_conventions`, 5s timeout, fail-open) to detect convention violations
(`cli/src/hooks/convention_handlers/user_prompt_submit.rs`). Each `PlaybookConvention` already
carries `anti_patterns: Vec<String>` (e.g. `["twilio","sms"]` for a convention whose `value` is
`"email_sendgrid"`), but that field is only consumed inside Haiku's prompt. When an anti-pattern
appears **literally** in the user prompt, we can detect the (candidate) violation locally and skip
the Haiku round-trip — cheaper, faster, and it egresses less prompt text to Anthropic.

**Goal:** add a local deterministic pre-pass that runs *before* Haiku. Haiku remains the fallback
for everything the deterministic pass can't confidently decide. Net egress: the `Act` path skips
Haiku entirely (less prompt text sent to Anthropic); the widened-gate `Defer` path can *add*
Haiku calls for anti-pattern prompts that previously fell through the keyword gate (owner-approved
trade for closing the coverage hole). Most anti-pattern prompts hit `Act`, so aggregate egress
still trends down.

## The central risk (why this needs care)

Deterministic substring/word matching is **context-blind**: `"don't use twilio, use sendgrid"`,
`"why is sms banned?"`, `"remove the twilio dependency"` all literally contain an anti-pattern but
are **not** violations. Acting on those fires a wrong Challenge (erodes trust) or — worst case — a
wrong hard **Block** (denies legitimate work). The newspaper test the design must beat:
*"crewkit blocked a developer from asking how to remove Twilio because the word 'twilio' appeared."*

## Decisions (locked)

- **C2 [owner] — a deterministic-only match MUST NOT hard-Block.** Deterministic matches act
  directly only for the **Challenge** and **LogOnly** tiers. If a matched convention is **Block**
  tier (`challenge_severity == "error"`), the deterministic pass **defers entirely to Haiku** to
  confirm intent before any hard denial. Moreover, if *any* matched convention in the prompt is
  Block-tier, the whole prompt defers to Haiku (so Haiku adjudicates the most-severe case rather
  than a lower-tier match masking it).
- **C8 [default a] — dedup key.** Deterministic `proposed_value` = the matched anti_pattern token
  (org-authored). Accept that this keys separately from Haiku's richer values (e.g. `"twilio_sms"`)
  in `already_overridden`, so a prior Haiku-override may not dedup a later deterministic match →
  at most one extra challenge. Documented UX wrinkle, not a correctness hole.
- **C10 [default + follow-up] — anti-pattern quality floor.** The pass skips anti_patterns that are
  empty/whitespace, `< 3` chars (single-token), or in a small English stop-word denylist — a
  defensive floor against an org footgun (anti_pattern `"the"` → matches everything). Proper
  server-side validation at authoring time is a separate **API backlog follow-up** (out of scope).
- **C3 [default] — negation guard is best-effort English.** Documented limitation: non-English
  prompts get no negation suppression (they fall through to Haiku on match, which is safe). The
  convention/keyword lists are already English-only, so no regression. The guard scans a small
  **preceding** token window only (dropped the coarse global trailing-`?` cue per gate-2 finding
  6 — it over-suppressed). Residual: a question-framed prompt whose anti-pattern has no preceding
  cue (e.g. `"why is twilio banned?"`) may fire a soft **Challenge** nudge — bounded to Challenge,
  never Block (C2), and the user can decline. Accepted.
- **[owner, gate-2 finding 3] — widen the check gate to be anti-pattern-aware.** Today both Haiku
  and the deterministic pass are gated behind `contains_tech_keywords`, so an anti-pattern that
  isn't a generic tech keyword (e.g. `"firebase"`) in a keyword-free prompt is silently never
  enforced. Decision: **a prompt containing any valid literal anti-pattern also trips the "worth
  checking" gate** (implemented via the `Defer` arm below). Fixes a real coverage hole and the
  same pre-existing Haiku blind spot; net Haiku egress still drops because most anti-pattern
  prompts are acted on locally.

## Gate-1 constraints → where handled

| # | Constraint | Handling |
|---|---|---|
| C1 | Deterministic-FIRST, never -ONLY: no confident match → Haiku | `deterministic_check` returns `DetResult`; `Defer` and keyword-positive `NoMatch` → existing Haiku path. Test: keyword-but-no-anti-pattern prompt (`NoMatch`) still reaches Haiku. |
| C2 | Deterministic never Blocks | `deterministic_check` returns `Defer` (→ Haiku) on ANY Block-tier literal match, checked BEFORE the negation heuristic (gate-2 finding 1). |
| C3 | Negation/intent guard | `has_negation_cue` over a small preceding-token window; negated match is skipped (→ falls through). Best-effort. |
| C4 | Word-boundary, no regex | Tokenize prompt once (`split(|c| !c.is_alphanumeric())` + case-fold, same discipline as `TECH_KEYWORDS_EXACT`). Multi-word patterns = contiguous token-sequence match. Literal comparison only — anti_patterns NEVER compiled as regex. |
| C5 | Fail-open parity | Deterministic pass is local/infallible; no actionable match → Haiku (unchanged fail-open). No path becomes more restrictive on failure. |
| C6 | No prompt content in logs / `proposed_value` | Logs only convention id/category/key + a `det-match` bool (never the prompt slice/window). `proposed_value` = the **convention's** anti_pattern string, never a prompt slice. |
| C7 | Same dedup/tier/logging path | Both passes produce a `ConventionAnalysis` and converge on the **existing** post-analysis code (current lines ~165–249): `already_overridden` → `enforcement_for` → `log_convention_violation_async` → `response_for_violation`. No parallel enforcement branch. |
| C9 | Honest confidence | Deterministic `confidence = 0.8`: clears the `>= 0.7` action gate but stays `< 0.85` so injected language is "may conflict with" (not "conflicts with"). Never invent a value that overstates intent certainty. |
| C10 | Anti-pattern quality floor | `valid_anti_pattern?` filter (len/stop-word). |
| C11 | Injection surface unchanged | Reuses existing `build_violation_context`/`build_block_reason` (which `sanitize_for_prompt` all content). `proposed_value` is org-authored (C6), so no new prompt-injection vector. |
| C12 | Bounded work | Tokenize prompt once into a `Vec<&str>` + `HashSet<&str>` for O(1) single-token lookup; multi-token patterns slide over the token vec. O(conventions × anti_patterns) set lookups; no per-token rescan of the raw string. |

## Design

Restructure the top of `handle_user_prompt_submit`: fetch conventions, tokenize the prompt once,
run the deterministic pre-pass, and branch on its three-way result. Both passes converge on the
**existing** post-analysis code (confidence gate → find convention → D1 dedup → enforce) — refactor,
don't duplicate.

```rust
// conventions fetched; if empty → Allow (unchanged)
let tokens = tokenize(prompt);                       // lowercased, empties filtered — C4/C5/C12
let token_set: HashSet<&str> = tokens.iter().copied().collect();

let analysis = match deterministic_check(&tokens, &token_set, &conventions) {
    // Actionable Challenge/LogOnly literal match → act locally, SKIP Haiku.
    DetResult::Act(det) => det,
    // Matched an anti-pattern but must confirm (any Block-tier match, or all matches negated).
    // This is ALSO the widened gate: an anti-pattern present with no generic keyword lands here.
    DetResult::Defer => match haiku_with_timeout(prompt, &conventions).await {
        Some(a) => a,
        None => return HookResponse::Allow,          // fail-open (unchanged) — C5
    },
    // No anti-pattern present at all → fall back to today's generic keyword gate.
    DetResult::NoMatch => {
        if contains_tech_keywords(prompt) {
            match haiku_with_timeout(prompt, &conventions).await {
                Some(a) => a,
                None => return HookResponse::Allow,   // fail-open
            }
        } else {
            return HookResponse::Allow;               // fast path: no anti-pattern, no keyword
        }
    }
};
// UNCHANGED from here: `if !violated || confidence < 0.7 { Allow }`, find convention,
// already_overridden dedup, enforcement_for, response_for_violation.
```

`haiku_with_timeout` = the existing `tokio::time::timeout(CHECK_TIMEOUT, HaikuService::check_conventions)`
block, extracted so both `Defer` and keyword-`NoMatch` reuse it (fail-open on Err/timeout).

The three-way result and the loop that fixes gate-2 findings 1, 2, 4:

```rust
enum DetResult { Act(ConventionAnalysis), Defer, NoMatch }

enum ConventionMatch { Actionable(String), NegatedOnly, NoMatch }  // per-convention

fn deterministic_check(tokens: &[&str], set: &HashSet<&str>, conventions: &[PlaybookConvention]) -> DetResult {
    let mut challenge: Option<ConventionAnalysis> = None;
    let mut log_only:  Option<ConventionAnalysis> = None;
    let mut any_matched = false;

    for c in conventions {
        let m = convention_match(tokens, set, c);
        if matches!(m, ConventionMatch::NoMatch) { continue; }
        any_matched = true;

        // Finding 1: Block check runs on ANY literal match (negated or not), BEFORE the negation
        // heuristic can consume it. Any Block-tier literal match → defer to Haiku (negation-aware).
        if enforcement_for(c) == ConventionEnforcement::Block {
            return DetResult::Defer;                              // C2
        }
        // Non-Block: act only on an un-negated (Actionable) match; NegatedOnly contributes to
        // `any_matched` (→ Defer) but is not acted on locally.
        if let ConventionMatch::Actionable(anti) = m {
            match enforcement_for(c) {
                ConventionEnforcement::Challenge => if challenge.is_none() { challenge = Some(det_analysis(c, anti)); },
                ConventionEnforcement::LogOnly   => if log_only.is_none()  { log_only  = Some(det_analysis(c, anti)); },
                ConventionEnforcement::Block => unreachable!(),
            }
        }
    }

    // Finding 2: prefer the higher-actionable tier (Challenge) over LogOnly, regardless of cache order.
    if let Some(a) = challenge.or(log_only) {
        DetResult::Act(a)
    } else if any_matched {
        DetResult::Defer                                          // matched but all negated → Haiku
    } else {
        DetResult::NoMatch
    }
}

fn det_analysis(c: &PlaybookConvention, anti: String) -> ConventionAnalysis {
    ConventionAnalysis {
        violated: true,
        convention_id: Some(c.id.clone()),
        proposed_value: Some(anti),                // C6: org anti_pattern, NOT a prompt slice
        confidence: 0.8,                            // C9: clears 0.7, < 0.85 → "may conflict with"
        reasoning: Some("Literal anti-pattern match (deterministic pre-pass)".into()),
        suggested_alternative: Some(c.value.clone()),
    }
}
```

Helpers (all pure, unit-testable, no network):
- `tokenize(s)`: lowercase, `split(|c| !c.is_alphanumeric())`, **filter empty tokens** (finding 5:
  consecutive delimiters / `_` yield empties). Returns `Vec<&str>`.
- `convention_match(tokens, set, c) -> ConventionMatch` (finding 4 — per-anti-pattern negation):
  for each `valid_anti_pattern` of `c`, tokenize the pattern with the **same** `tokenize` fn
  (finding 5 — so `"google-maps"`/`"google maps"`/`"google_maps"` all normalize to `[google,maps]`);
  single-token → `set` lookup; multi-token → contiguous window over `tokens`. On a match at index
  `at`, classify via `has_negation_cue`. Return `Actionable(pattern)` for the first **un-negated**
  match; if matches exist but all are negated → `NegatedOnly`; none → `NoMatch`.
- `has_negation_cue(tokens, at)`: scan `tokens[at.saturating_sub(NEG_WINDOW)..at]` (preceding
  window only) for a small cue set (`dont, not, no, stop, avoid, remove, drop, deprecate, ban,
  banned, without, instead`). `NEG_WINDOW = 4`. Best-effort English (C3). Note `"don't"` tokenizes
  to `["don","t"]` — include both `don`+`t` handling or match the `dont`/`don` token; covered by tests.
- `valid_anti_pattern(p)`: non-empty after trim; single-token patterns `>= 3` chars; not in a small
  `STOP_WORDS` denylist (C10). Short tech names (`s3`, `go`, `k8s`) below the floor are simply not
  caught by the *deterministic* pass — they still reach Haiku via the keyword gate or `Defer`, so
  no enforcement is lost, only the local fast-path. Documented coverage note.

## Stages (test-first; ends with gate-3 trio)

### Stage 1 — pure helpers + unit tests
Files: `cli/src/hooks/convention_handlers/user_prompt_submit.rs`.
Add `tokenize`, `valid_anti_pattern`, `convention_match`, `has_negation_cue`, `det_analysis`,
`deterministic_check` (+ the `DetResult`/`ConventionMatch` enums) with `#[cfg(test)]` FIRST:
- literal single-token match, Challenge tier → `Act`, `proposed_value == anti_pattern`, `confidence == 0.8`.
- word-boundary: `"sms"` does NOT match `"smsc"`/`"smshub"`; `"aws"` not in `"flaws"`.
- multi-word anti_pattern matches contiguous tokens (`"google maps"`, `"google-maps"`, `"google_maps"` all match `"use google maps"`); scattered tokens do NOT match.
- negation (Challenge tier): `"don't use twilio"`, `"stop using sms"`, `"remove the twilio dependency"` → `Defer` (not `Act`).
- **finding 1**: a Block-tier convention whose anti-pattern is present → `Defer`, EVEN when a Challenge-tier convention also matches un-negated (Block must not be masked). Include the negated-Block variant (`"don't use mysql"` where mysql is Block) → still `Defer`.
- **finding 2**: a LogOnly convention ordered BEFORE a Challenge convention, both matched un-negated → `Act` returns the **Challenge** analysis, not the LogOnly one.
- **finding 4**: convention with `["twilio","sms"]`, prompt `"don't use twilio but add sms"` → `Act` on the un-negated `"sms"`.
- all-negated multi-pattern → `NegatedOnly` → contributes to `Defer`.
- stop-word / short single-token anti_pattern (`"the"`, `"a"`, 2-char) → ignored (not matched).
- no anti-pattern present at all → `NoMatch`.
- confidence == 0.8; `proposed_value` is the convention's anti_pattern, never a prompt slice (C6).

### Stage 2 — wire into `handle_user_prompt_submit`
Restructure per the Design: fetch conventions → tokenize → `deterministic_check` → branch on
`Act`/`Defer`/`NoMatch`, converging on the existing post-analysis block (no duplication; C7).
Extract `haiku_with_timeout`.
**CRITICAL (gate-2 build checkpoint 1): DELETE the existing early fast-path return** at
`user_prompt_submit.rs:101–108` (`if !contains_tech_keywords(prompt) { return Allow }`) and fold
that logic into the `NoMatch` arm. If left in place, `deterministic_check` never runs for
keyword-less anti-pattern prompts and the widened gate is dead code. The empty-conventions early
return (`:114–120`) correctly STAYS before the pre-pass. Add handler-level tests:
- **C1 / widened gate**: `NoMatch` + tech keyword → Haiku path invoked; `NoMatch` + no keyword → `Allow` (fast path); `Defer` (anti-pattern present, no keyword) → Haiku invoked (the widened gate). Use a Haiku test seam/fake; if none exists, assert `deterministic_check`'s result for each input and that the branch selects Haiku vs Allow accordingly.
- deterministic `Act` (Challenge) → `HookResponse::Modify` with the existing context, and Haiku is NOT called.
- deterministic `Act` on a log-only convention → `Allow` (C7: same opt-out via `enforcement_for`/`response_for_violation`).
- deterministic `Act` still routes through `already_overridden` — a pre-overridden det match → `Allow` (D1 dedup, C7).
Update the module doc comment to describe the three-way pre-pass + widened gate.

### Stage 3 — verify + docs
- `cd cli && just fmt && just lint && just check && cargo test --bin crewkit` (all local, verifiable here).
- Note the C10 API-authoring-validation follow-up in the handoff doc; record the C3/C8 limitations
  in the module doc.

## Non-goals / follow-ups
- Server-side anti_pattern authoring validation (min length, stop-word rejection) — API backlog (C10).
- Non-English negation handling (C3).
- Unifying det/Haiku dedup keys (C8) — accept separate keying for now.

## Known residual (gate-3 security, accepted by owner 2026-07-08)
Because an actionable deterministic `Act` **skips Haiku entirely** (the locked C2/§Design decision),
a prompt that literally matches a Challenge/LogOnly convention can **mask** a *co-occurring*
Block-tier (severity:error) convention that Haiku would have caught only **semantically** — i.e.
one whose literal `anti_patterns` do NOT appear in the prompt (so `convention_match` returns
`NoMatch` and the C2 literal-Block defer never trips). On such a prompt the Block-tier violation is
not caught in that turn. This is **narrow** (requires the co-occurrence of a literal lower-tier
match and a semantic-only Block violation in one prompt) and **fails in the safe direction**
(under-enforce, never a wrong hard-block — the newspaper-test risk the whole design optimizes
against). **Accepted as-is.** If it ever matters, the fix is a guard: when an actionable det match
co-occurs with a Block-tier convention that did not literally match, `Defer` to Haiku instead of
`Act` — at the cost of the fast-path (and its egress/latency savings) for any org that uses
severity:error conventions.
