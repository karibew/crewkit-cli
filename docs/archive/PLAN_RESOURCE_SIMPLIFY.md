# Resource System Simplification Plan (Musk 5-Step)

## Status: Steps 2-4 complete, QA review fixes applied

## Confirmed Requirements (KEEP)
1. 3-tier inheritance (platform → org → project) — core value
2. 2 ownership modes: standalone + inherit
3. Versioning: immutable ResourceVersion snapshots, rollback
4. Effective content: merge inheritance chain, output final content
5. CLI sync: download effective content to `.claude/` directories
6. Import: upload local files to API
7. Stats: per-version metrics (ResourceStat) — feeds A/B testing
8. A/B testing (ResourceVersionComparison) — **keep, rebuild properly later** for turn counts/corrections
9. Resource snapshots on sessions — keep storing for future A/B
10. Playbooks: conventions at platform/org level, injected into agent content

## Step 2: Delete List (confirmed)

| Target | Lines | Risk | Status |
|---|---|---|---|
| ResourceOverride model + references | ~125 | LOW | TODO |
| base_override field usage | ~25 | LOW | TODO |
| forked ownership mode + callbacks | ~60 | LOW | TODO |
| Section-based content replacement (replace_section) | ~15 | ZERO | TODO |
| Circular inheritance detection (MAX_DEPTH) | ~30 | ZERO | TODO |
| Orphaned child conversion callbacks | ~30 | ZERO | TODO |
| ResourceVersioningService (entire file) | ~116 | MEDIUM | TODO |
| ResourceEditorService duplication | ~130 | MEDIUM | TODO |
| Unused ResourceStat scopes | ~8 | ZERO | TODO |
| Unused frontmatter fields (external_id, synced_at) | ~40 | LOW | TODO |
| 500KB content validation | ~10 | ZERO | TODO |
| **Total** | **~590** | | |

## Step 3: Simplify (after deletes)
1. ~~Consolidate version services~~ → DONE in Step 2 (ResourceVersioningService deleted, inlined)
2. ~~Slim ConfigurationService (692L → ~400L)~~ → DONE: 399 lines
   - Unified `merge_inheritance_chain` + `apply_tier_guidelines` → single `build_merged_content`
   - Merged `find_org_resource_by_name` + `find_project_resource_by_name` → `find_resource_by_name(owner_type:)`
   - Merged `build_tier_data` + `build_tier_data_with_overrides` → single `build_tier_data(is_source:)`
   - Removed redundant `else` branch in effective_content
3. ~~De-duplicate EditorService~~ → DONE: removed dead `can_fork?` permission + test
4. Playbook tier restriction → single model validation (TODO)
5. ~~Fix CLI checksum sync (broken — always backs up + overwrites)~~ → DONE
   - Root cause: `strip_crewkit_metadata` round-tripped YAML through serde_yaml, producing different output than Ruby's to_yaml, so integrity hash always failed
   - Fix: If `content_hash` matches API hash, skip immediately. No integrity check needed.
   - Removed dead code: `strip_crewkit_metadata`, `extract_version_hash`, `hash_content_without_metadata`
6. ~~Simplify CLI frontmatter (4 fields → 2)~~ → DONE
   - Kept: `version_hash` (informational) + `content_hash` (for sync comparison)
   - Removed: `external_id` (unused), `synced_at` (unused)

## Step 4: Optimize (after simplify)
1. ~~N+1 inheritance chain queries → eager-load~~ → DONE
   - Bulk effective endpoint now eager-loads `inherits_from` chain (3 levels) + `resource_versions`
   - Eliminates 1 query per chain level per resource
2. ~~4x redundant computation per resource~~ → DONE
   - Added `compute_all` method: computes effective_content, version_hash, chain, tier_breakdown in one pass
   - Both `effective` (bulk) and `effective_single` endpoints use it
   - Was: 4 separate ConfigurationService instantiations per resource, each re-walking the chain
   - Now: 1 instantiation, chain walked once

## QA Review Fixes (post Step 4)
1. ~~CRITICAL: `rollback_to_version!` blanket rescue swallowed exceptions~~ → DONE: removed rescue, added ensure
2. ~~CRITICAL: DB constraint still permitted 'forked'~~ → DONE: migration updated
3. ~~CRITICAL: `post :fork` still in org-scoped routes~~ → DONE: removed
4. ~~MEDIUM: Unbounded chain walk (DoS)~~ → DONE: re-added MAX_CHAIN_DEPTH + cycle detection
5. ~~MEDIUM: Content size validation removed~~ → DONE: re-added 500KB limit
6. ~~MEDIUM: Hard delete orphans children~~ → DONE: added `dependent: :restrict_with_exception`
7. ~~WARN: Dead `fork?` policy method~~ → DONE: deleted
8. ~~WARN: `compute_all` chain not reused by `build_merged_content`~~ → DONE: uses `@cached_chain`
9. ~~WARN: CLAUDE.md references deleted ResourceVersioningService~~ → DONE: removed

## Step 5: Automate
- Delete placeholder A/B test tests
- Delete ResourceOverride tests
- Delete fork-related tests
- Add integration test: sync twice with no changes → assert no file writes
- Add test for inheritance chain resolved in 1 query
