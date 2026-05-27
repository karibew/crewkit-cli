# changelog-docs-licensing - 2026-02-07 19:50

## Current Task
Changelog automation, documentation planning, and license cleanup for crewkit

## Progress
- [x] Investigated --session-id behavior for transcript correlation
- [x] Decided on provider_session_id field (separate from external_id) — DEFERRED
- [x] Generated changelog automation script (cli/scripts/generate-changelog.sh)
- [x] Integrated changelog job into release-cli.yml (option C: Haiku rewrites GitHub Release body only)
- [x] Fixed release-please conflict (release-please owns cli/CHANGELOG.md, Haiku owns release body)
- [x] Fixed first-release edge case (fallback to all commits from repo root)
- [x] Updated all 7 public releases on karibew/crewkit-cli with Haiku-generated notes
- [x] Completed open-source audit (16 findings across 13 files)
- [x] Fixed all 16 MIT/open-source references — now proprietary, ToS-linked
- [x] Created documentation plan (.claude/designs/documentation-plan.md)
- [x] Created changelog automation design doc (.claude/designs/changelog-automation.md)
- [x] Implement documentation (fumadocs setup, content authoring — Phases 1-6)
- [x] Fix skip-github-release in release-please to prevent release body flicker
- [x] ~~Add provider_session_id to LlmSession model~~ — closed as unnecessary (claude_session_id already covers this)

## Key Files
- cli/scripts/generate-changelog.sh — Haiku-powered release notes generator
- .github/workflows/release-cli.yml — CI with changelog job
- .github/workflows/release-please.yml — Release orchestration
- release-please-config.json — Release-please config (changelog-path: cli/CHANGELOG.md)
- .claude/designs/documentation-plan.md — Full docs architecture (fumadocs, flat, opencode-inspired)
- .claude/designs/changelog-automation.md — Changelog automation design doc
- dashboard/src/components/landing/hero.tsx — Changed "Open Source" to "Free CLI"
- dashboard/src/app/(public)/terms/page.tsx — Updated legal text
- cli/Cargo.toml — license = "LicenseRef-Proprietary"
- npm/crewkit/package.json — license = "SEE LICENSE IN LICENSE"
- choco/crewkit.nuspec — licenseUrl points to crewkit.io/terms

## Decisions
- License: Proprietary with ToS (option A — simplest, no BSL/SSPL complexity)
- Changelog: Option C — release-please owns CHANGELOG.md, Haiku rewrites GitHub Release body
- Session IDs: Keep independent (provider_session_id field, not --session-id control)
- Docs framework: fumadocs (Next.js-native equivalent of Astro Starlight)
- Docs structure: Flat files, sidebar-grouped, one page per concept (opencode-inspired)
- External API docs: Only user-facing routes (admin routes in internal docs only)
- Haiku model: claude-haiku-4-5-20251001 (pinned for CI stability)

## Blockers
- None currently

## Next Steps
1. ~~Set up fumadocs infrastructure in dashboard (Phase 1)~~ ✓
2. ~~Write external docs content (Phases 2-4, ~26 .mdx files)~~ ✓ (28 files)
3. ~~Write internal docs content (Phase 5, ~33 .mdx files)~~ ✓ (33 files)
4. ~~Fix skip-github-release in release-please config~~ ✓

---
Session ID: changelog-docs-licensing
Created: 2026-02-07T19:50:00Z
Last Updated: 2026-02-07T19:50:00Z
