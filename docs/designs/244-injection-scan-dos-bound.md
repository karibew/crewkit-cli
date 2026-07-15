# Design: injection-scanner DoS bound (#244, closes #158 hardening)

**Status:** design (gate-1 cleared; awaiting gate-2)
**Component:** api
**Scope:** two contained DoS bounds on the static security scanner. No regex rewrites.

## Problem

`StaticSecurityAnalyzer.analyze` (`static_security_analyzer.rb:71-103`) is the
regex detector behind marketplace prompt-injection scanning
(`Marketplace::InjectionScan`, #158 slice 6) and authored-resource scanning.

### P1 — O(n²) substring-copy match loop (the stated #244 bound)

```ruby
lines.each_with_index do |line, idx|
  offset = 0
  while (m = line[offset..].match(pattern))   # allocates a fresh tail copy per match
    key = [idx+1, category, description]
    unless seen[key] ; seen[key]=true ; findings << {...} ; end
    offset += m.begin(0) + [m[0].length, 1].max
  end
  return {...} if max_findings && findings.size >= max_findings
end
```

On a single long line (content with no newlines → one line up to
`InjectionScan::MAX_FIELD_BYTES` = 100KB) with many matches, `line[offset..]`
re-copies the tail on every match → O(matches × line_length) copying.

**The loop is also pointless:** findings are deduped by
`[line, category, description]`, and each PATTERN row has a fixed
`(category, description)`, so only the *first* match per `(pattern, line)` is
ever recorded — every later iteration re-scans/re-copies to add nothing. Proven
by the existing test (`static_security_analyzer_test.rb:292`):
`'password = "abc" password = "def"'` → exactly **1** finding.

### P2 — regex backtracking survives the P1 fix (adjacent, real)

The loop fix removes substring O(n²) but not regex-engine backtracking. The
backtick pattern `/`[^`]*(?:…)[^`]*`/` has two greedy negated-class stars around
an alternation → super-linear backtracking on a long run with no closing
backtick. On a 100KB line that is a genuine multi-second-to-minutes DoS that the
byte cap does **not** bound. Shipping P1 alone would be a false sense of safety.

## Design

### Part 1 — collapse the match loop to a single leftmost match

Behavior-identical to today's deduped output; O(patterns × content_length), no
per-iteration allocation.

```ruby
def analyze(content, max_findings: nil)
  return { findings: [], scanned_at: Time.current } if content.blank?

  findings = []
  seen = {}
  lines = content.lines   # hoisted once (line numbers + no-cross-\n depend on it)

  PATTERNS.each do |pattern, severity, category, description|
    lines.each_with_index do |line, idx|
      m = pattern.match(line)          # leftmost match only — what fires today
      next unless m

      key = [ idx + 1, category, description ]
      next if seen[key]                # kept: collapses the theoretical cross-pattern same-key case

      seen[key] = true
      findings << {
        severity: severity, category: category, description: description,
        line: idx + 1, match: sanitize_match(m[0].to_s, category)
      }
      return { findings: findings, scanned_at: Time.current } if max_findings && findings.size >= max_findings
    end
  end

  { findings: findings, scanned_at: Time.current }
end
```

Invariants gate-1 requires (all preserved):
- MUST keep per-line iteration (`content.lines`) — line numbers + implicit
  "`.` never crosses `\n`" depend on it. No matching against the whole blob.
- MUST keep the `seen` map keyed exactly `[idx+1, category, description]`.
- MUST record the FIRST (leftmost) match only. `clean = findings.empty?` is
  invariant under first-vs-all-match, so **no input that flags today stops
  flagging** — fail-closed detection unchanged.
- MUST keep `sanitize_match(m[0].to_s, category)` (secret redaction + truncate).
- No PATTERN uses `^ $ \A \z \G` → dropping `offset` is strictly *more* correct
  (removes an artificial-substring-start `\b` boundary hazard + the zero-width
  infinite-loop guard's reason to exist). Latent-bug removal, not a regression.

Callers verified to not depend on >1 finding per (pattern, line):
`injection_scan` (`empty?` + `uniq` + `first(50)`), `analyze_resource_security_job`
(dedups, strips `match`, feeds `SecurityScoreCalculator` — counts distinct),
`skill_validation_service` (selects `critical`), `llm_security_analyzer`
(independent Haiku path). Scores don't move (already deduped).

### Part 2 — DROPPED after implementation discovery: Rails already sets the backstop

**Finding (during build, refutes both gate-1's and gate-2's premise that no
`Regexp.timeout` is set):** Rails 8.1 sets a **process-global `Regexp.timeout =
1.0`** by default via `config.load_defaults`
(`railties .../application/configuration.rb:342` → `Regexp.timeout ||= 1`).
Confirmed at runtime (`bin/rails runner "puts Regexp.timeout"` → `1.0`).

Consequences:
- The catastrophic-backtracking DoS (P2) is **already bounded to 1.0s app-wide**,
  and the scanner already fails closed on the resulting `Regexp::TimeoutError`
  (`injection_scan.rb` `rescue StandardError` → `{ clean: false }`).
- A per-object timeout of `3.0` would **override the stricter 1.0 global with a
  more lenient value** (per-object takes precedence) — a *weakening*, not a fix.

So Part 2 is dropped. No `REGEXP_TIMEOUT`, no per-object timeout, no global set.
The analyzer/injection_scan comments are updated to document that the framework
global is the ReDoS backstop and why we deliberately don't set our own. A test
asserts the global backstop is present and that no pattern overrides it.

<details><summary>Original Part 2 (superseded — kept for the record)</summary>

Bake a per-object timeout into each PATTERN regex …

</details>

### ~~Part 2 — per-`Regexp` timeout backstop (P2) — **per-object, NOT global** (gate-2)~~ (superseded)

Bake a **per-object** timeout into each PATTERN regex. This confines the ReDoS
net to the scanner's own regexes — it does **not** touch the process-global
`Regexp.timeout` (which would change regex semantics app-wide: routing,
ActiveRecord, gems, crewkit's own MB-scale `gsub`/`scan` over transcripts/
imports — none of which fail-closed) and does **not** leak across Puma threads
(the timeout is a field on the object, nullifying `injection_scan.rb:18-21`'s
whole reason for avoiding per-scan set/restore).

```ruby
REGEXP_TIMEOUT = 3.0 # seconds, per match operation, per pattern

# built from the raw table:
PATTERNS = RAW_PATTERNS.map do |regex, severity, category, description|
  [ Regexp.new(regex.source, regex.options, timeout: REGEXP_TIMEOUT), severity, category, description ]
end.freeze
```

- Applied to **all** patterns (cleanest; any could backtrack) — verified
  `Regexp.new(src, opts, timeout:)` preserves source+options identically.
- Process-global `Regexp.timeout` stays `nil` — a test asserts this so nobody
  re-introduces the global.
- Fail-closed already wired: `Regexp::TimeoutError < RegexpError < StandardError`,
  so `injection_scan.rb:113` `rescue StandardError` → `nil` → `{ clean: false }`.
  No code change there; a test proves it.
- **Bound is per match-operation, not per-scan.** For multi-line pathological
  input the total scan time is `REGEXP_TIMEOUT × pathological_lines` — this does
  NOT bound a whole request/job (see Out of scope). It bounds the single-line
  100KB case (the InjectionScan surface, which is byte-capped to one field's
  worth) and prevents any one match from hanging unbounded.
- Update the `injection_scan.rb:18-21` comment: the per-object timeout is now the
  sanctioned net and does not leak across threads.

## Out of scope (documented)

- **Rewriting/anchoring any regex** — behavior-changing surgery on a fail-closed
  detector; regression risk. Separate issue if we ever want linear-time patterns.
- **Per-caller byte caps + a per-SCAN (whole-job) time bound** for the nil-path
  callers (`analyze_resource_security_job`, `skill_validation_service`) — the
  per-object timeout bounds each match op but not total scan time over many
  pathological lines. Accepted residual: those callers are fed authored content
  by authenticated org members (not cross-tenant marketplace attackers), so the
  threat is lower; a whole-scan bound is a separate follow-up.
- **Puma thread-pool starvation** (N concurrent malicious scans each pinning a
  worker) — neither timeout mechanism addresses it. Out of scope.

## Tests (api)

`static_security_analyzer_test.rb`:
1. **Perf/dedup sentinel:** a single ~100KB line of thousands of
   `password = "x"` matches → exactly **1** finding, and completes well under a
   generous wall-clock bound (regression guard against re-introducing O(n²)).
2. **Multi-pattern same line:** `password="x"` + `eval("y")` on ONE line → both a
   `secrets` and an `injection` finding (proves `seen` wasn't over-broadened).
3. Existing 30+ tests stay green (dedup, line numbers, all categories).

`marketplace/injection_scan` fail-closed timeout sentinel:
4. With `Regexp.timeout` temporarily lowered (ensure-restored), a pathological
   backtracking input → `InjectionScan.scan(...)[:clean] == false` (fail-closed
   on `Regexp::TimeoutError`, not a hang, never `clean: true`). Kept fast by
   toggling the timeout locally rather than waiting the real 3s.

## Gates

- `PARALLEL_WORKERS=1 bin/rails test test/services/static_security_analyzer_test.rb
  test/services/marketplace/injection_scan_test.rb` + `test/sentinels/` on the
  isolated `crewkit_test_wt1` DB; `rubocop`. No route/response change → no OpenAPI.
