# skill-builder-format-multifile - 2026-02-13

## Current Task
Align Skill Builder with official Claude Code SKILL.md format and add multi-file support

## Progress
- [ ] **Phase A: Format Alignment**
  - [ ] A1. Update system prompt to teach correct SKILL.md frontmatter (`name`, `description`, `user-invocable`, `allowed-tools`, `model`, `context`, `agent`, `hooks`)
  - [ ] A2. Update 5 skill templates to use correct format (remove `tags`/`category` from frontmatter)
  - [ ] A3. Update DEFAULT_CONTENT in builder page
  - [ ] A4. Verify SkillValidationService compatibility (currently requires `name` + `description` — both valid)
  - [ ] A5. Verify finalize handles missing `tags`/`category` gracefully (defaults already exist)
- [ ] **Phase B: Multi-File Support**
  - [ ] B1. Migration: add `draft_files` JSONB to `skill_builder_sessions`
  - [ ] B2. Migration: add `supporting_files` JSONB to `resources`
  - [ ] B3. API: permit/serialize `draft_files`, copy to `supporting_files` on finalize
  - [ ] B4. Dashboard: tabbed editor (SKILL.md always first, add/remove supporting files)
  - [ ] B5. AI assistant: include all files in context, file-targeted diffs
  - [ ] B6. Types/API client/hooks: add `draft_files` to TypeScript types
  - [ ] B7. Validation: file path safety, total size limit
  - [ ] B8. Tests for multi-file create/update/finalize

## Key Design Decisions
- `content` field = SKILL.md (entrypoint), backward compatible
- `supporting_files` JSONB = `{ "examples/basic.md": "...", "patterns/auth.md": "..." }` (additional files)
- SKILL.md frontmatter: only `name` and `description` required by validator; other fields (`user-invocable`, `allowed-tools`, etc.) are optional
- CLI sync unchanged for now (single-file) — multi-file CLI sync is a separate work item

## Key Files
- `api/app/services/skill_builder_assistant_service.rb` — System prompt (SYSTEM_PROMPT constant)
- `api/app/services/skill_validation_service.rb` — Validation logic
- `api/app/controllers/api/v1/skill_builder_sessions_controller.rb` — Create/update/finalize
- `dashboard/src/lib/skill-templates.ts` — 5 templates with wrong frontmatter
- `dashboard/src/app/kit/skills/builder/page.tsx` — Builder page, DEFAULT_CONTENT, editor state
- `dashboard/src/components/features/skill-builder/chat-panel.tsx` — Chat with diff support
- `dashboard/src/types/skill-builder.ts` — TypeScript types
- `dashboard/src/lib/api/skill-builder.ts` — API client
- `dashboard/src/hooks/use-skill-builder.ts` — TanStack Query hooks

## Reference
- Official SKILL.md docs: `.claude/skills/claude-code-guide/docs/skills.md`
- Full plan: `.claude/plans/splendid-dreaming-dijkstra.md`

## Components
- API (rails-expert)
- Dashboard (frontend-expert)
