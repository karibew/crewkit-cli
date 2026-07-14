# Verification Framework — Slice 1 (issue #25)

**Scope (approved):** governance config + prompt injection. **Deferred:** custom script
execution, hard session-blocking (Stop-hook), verification-result analytics, A/B testing.

Gate-1 threat model complete (security-expert). This doc = plan + gate-1 ship conditions.

## Data model

- `verification_templates` — 3-tier reusable requirement. `organization_id` **nullable**
  (NULL = platform tier, crewkit_staff-writable, world-readable; non-null = org tier).
  `name`, `description`, `category` (server enum: `unit_tests | e2e | api_contract | browser | manual | custom`),
  `body` (multi-line markdown checklist, `MAX_BODY_LENGTH` bounded), `external_id`, PaperTrail.
- `project_verification_configs` — join. Denormalized `organization_id` (from project),
  `project_id`, `verification_template_id`, `advisory` boolean (see honesty note), `position`,
  `external_id`. Unique (project_id, verification_template_id). Per-project count cap. PaperTrail.

## Gate-1 ship conditions (MUST)

1. Pundit: `VerificationTemplatePolicy::Scope` = platform (org_id NULL) ∪ `organization_scoped`;
   `ProjectVerificationConfigPolicy::Scope` = `organization_scoped` (denormalized org_id).
2. **Cross-org template-reference IDOR guard**: `verification_template_id` validated to
   `organization_id IN (NULL, current_org.id)` at **write** (model validation) AND re-validated
   at **render** (don't trust stored FK). Cross-org ref → 404/422, never rendered.
3. Authz: view = viewer+; org template create/edit = manager+, destroy = admin; platform template
   write = crewkit_staff only; project config create/edit = manager+, unassign = admin. Every write
   also checks record-org membership (not role alone).
4. **Body-appropriate sanitizer** (NOT the convention `sanitize_for_markdown`): preserve newlines +
   list markers; neutralize HTML comments + fence-break constructs (`-->`, wrapper markers, leading
   `#`/`SYSTEM:` section forgery). `category` = server enum allowlist.
5. `MAX_BODY_LENGTH` + per-project template cap at model validation.
6. Script execution stays out. Hard-blocking stays out — **`required`/`advisory` labeled honestly**:
   rendered markdown + API docs + UI say "advisory — not automatically enforced".
7. No new egress: rendered server-side → CLI writes `.claude/verification.md` + `CLAUDE.md` reference
   → read locally by `claude`. No cache in slice 1.
8. Sentinels: tenant isolation (auto via BaseController inheritance + pundit_scope), cross-org IDOR,
   platform immutability, render scoping (only platform ∪ this-project's-assigned, in order),
   fence-breakout resistance, bounds. PaperTrail with org meta on both models.

## Rendering + injection

- `VerificationRequirementsService#render(project:)` → fenced markdown (`<!-- CREWKIT VERIFICATION -->`
  … `<!-- END CREWKIT VERIFICATION -->`), body sanitized, honest advisory labeling. Mirrors
  `PlaybookConfigurationService.build_markdown` shape.
- Endpoint `GET /:org/projects/:id/verification_requirements` → `{ content, version_hash }`.
  Project scoping copies `playbooks_controller#for_sync:165-181` (find_by! on current_org.projects,
  intersect with `project_verification_configs`).
- CLI sync step writes `.claude/verification.md` + references it in `CLAUDE.md`
  (mirror `resource_sync.rs sync_playbooks_from_conventions:581` + `ensure_playbook_reference:633`).

## Stages (each: build → affected gates → implementation-validator → adversarial-code-reviewer → security-expert)

- **Stage 1 (API foundation):** 2 migrations + platform seed, 2 models, 2 policies, 2 serializers,
  2 controllers, render service, endpoint, routes, OpenAPI, sentinels. (Security-critical core.)
- **Stage 2 (CLI injection):** fetch + write `.claude/verification.md` + CLAUDE.md reference; tests.
- **Stage 3 (Dashboard):** project verification-config panel (assign platform/org templates,
  toggle advisory `required`, unassign) on the project page; tests.
  **Deferred within Stage 3:** the org-template *builder/creation* UI. The create/delete template
  endpoints exist server-side (Stage 1) but slice 1's panel only assigns platform + already-existing
  templates. The builder is a follow-up (add the client methods + a form then).

Patterns to copy: `GitIntegration` (full CRUD stack), `ProjectPlaybookAssignment` (project join +
policy + project scoping), `resource.rb` (3-tier), `20260113100006_seed_platform_playbooks.rb` (seed).
