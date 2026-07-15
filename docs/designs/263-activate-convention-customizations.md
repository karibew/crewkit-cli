# Design: activate PlaybookConventionCustomization (#263 follow-on)

**Status:** design (gate-1 + gate-2 cleared — PROCEED-WITH-CHANGES folded in below; awaiting build)

## Gate-2 folded-in (must/should)

1. **for_stacks emits into the existing `value` field only** (never a new/parallel
   serializer field). The CLI's `generate_playbook_markdown`
   (`cli/src/services/resource_sync.rs:890-908`) already runs its own
   `sanitize_for_markdown` on the `value` field, so a customized value inherits the
   same neutralization as an author-authored `conv.value` — no new injection class.
   Sentinel: a customized value with `\n## Injected` / `<!-- -->` / leading `#` is
   neutralized on the markdown path.
2. **Lookups key on internal `conv.id`** (matches `index_by(&:playbook_convention_id)`),
   NOT `external_id`. A mismatch fails safe (baseline) but silently no-ops the
   feature — test that a customization actually reflects in `for_stacks` output.
3. **Log when customizations collapse a non-empty baseline to an empty set**
   (matters once the write route ships; harmless now).
4. **Expiring-`disable` asymmetry:** an expiring `disable` keeps a convention
   suppressed for ≤15 min after it should re-enforce (bounded under-enforcement);
   a scheduled disable becoming effective errs toward MORE enforcement (safe).
   Documented; revisit the TTL / add a time-based bust when the write route lands.
5. **Carryover:** the non-disableable security-convention lock + manager-vs-admin
   authz bar MUST ship in the SAME change as the eventual CRUD write route, not after.

**Component:** api
**Scope:** render-only activation of the dormant per-project convention
customization layer, applied to **all** convention surfaces.

## Decisions (locked with the owner)

- **Q1 — surface scope: ALL surfaces (owner's explicit call).** Apply
  customizations in `generate_agent_content` (CLI `for_sync` content,
  resource-config injection, blueprint generation) **and** the cached `for_stacks`
  serialized path, extending the `for_stacks` cache token to bust on customization
  edits. Rationale: preview == enforcement everywhere; no surface disagrees.
  > **If a future dev wants a different scope** (e.g. leave `for_stacks` raw), the
  > apply is centralised in two clearly-marked seams: `PlaybookConfigurationService`
  > (markdown paths) and `PlaybookSerializer#serialize_conventions` +
  > `PlaybooksController#for_stacks` cache token (serialized path). Remove the
  > serializer hook + revert the cache token to drop `for_stacks` from scope.
- **Q2 — render-only (owner's call).** Apply EXISTING customizations
  (settable only via console/seed today — no write route). The **CRUD write
  route + security-convention lock (non-disableable conventions) + manager-vs-admin
  authz bar are HELD for Félix/Rémi** (product/security-policy). Because there is
  no product write path, the "manager silently disables a security convention"
  abuse vector gate-1 flagged is not reachable by activating rendering now.

## Model recap (already built, unchanged)

`PlaybookConventionCustomization` belongs to a `project_playbook_assignment` +
`playbook_convention` + `organization`. Canonical primitives on the model:
- `#effective_value` — `override`→`custom_value`; `augment`→`"value, custom_value"`;
  `disable`→`nil`.
- `#disables_convention?`, `#currently_effective?`, scope `currently_effective`
  (active + within effective_from/until).

These are the single source of truth; every application site below calls them.

## Part A — one sanctioned tenant-scoped loader

New `PlaybookConventionCustomization.effective_map_for(project)`:

```ruby
# {playbook_convention_id => customization}, currently-effective, tenant-scoped.
# The ONLY sanctioned load for rendering/enforcement: scoped through the
# project's OWN assignments + organization_id, so no cross-project/cross-org
# customization can bleed in. NEVER a global where(playbook_convention_id: ...)
# — platform convention rows are shared across orgs, so a global-by-id load would
# apply another org's customization here (the cross-tenant leak). Fail-safe: {}.
def self.effective_map_for(project)
  return {} if project.nil?
  where(project_playbook_assignment_id: project.project_playbook_assignments.select(:id),
        organization_id: project.organization_id)
    .currently_effective
    .includes(:playbook_convention)
    .index_by(&:playbook_convention_id)
rescue StandardError => e
  Rails.logger.warn("[PlaybookConventionCustomization] effective_map_for failed: #{e.class}")
  {}
end
```

Match customizations to conventions **by `playbook_convention_id` == `convention.id`**,
never by `"category/key"` (two playbooks can share a key; a customization against
the platform row must NOT apply to an org override — id-match makes the no-match
case a safe no-op).

## Part B — PlaybookConfigurationService (markdown paths)

`generate_agent_content(playbooks:, project:, format:)` currently ignores
`project`. Apply customizations to the merged conventions before rendering:

```ruby
def generate_agent_content(playbooks:, project: nil, format: DEFAULT_FORMAT)
  merged = merge_playbook_conventions(playbooks: playbooks)
  conventions = apply_customizations(merged.values, project)
  return "" if conventions.empty?
  build_markdown(conventions, format)   # builders now take an Array, not the Hash
end

# Fail-safe at BOTH levels: a bad map load or a bad single customization drops to
# the stricter platform/org BASELINE, never an empty/partial enforcement file.
def apply_customizations(conventions, project)
  return conventions if project.nil?
  custom_map = PlaybookConventionCustomization.effective_map_for(project)
  return conventions if custom_map.empty?

  conventions.filter_map do |conv|
    customization = custom_map[conv.id]
    next conv unless customization
    apply_one(conv, customization)   # nil => dropped (disable)
  end
rescue StandardError => e
  Rails.logger.warn("[PlaybookConfigurationService] customization apply failed, baseline: #{e.class}")
  conventions
end

def apply_one(conv, customization)
  return nil if customization.disables_convention?
  customized = conv.dup            # in-memory ONLY — never saved (real row has
  customized.value = customization.effective_value  # after_commit cache-bust + PaperTrail)
  customized
rescue StandardError
  conv                             # per-convention fail-safe: baseline value
end
```

- **Accepted residual (gate-3, finding 1): value-only customization.** Only the
  convention `value` is customized; `anti_patterns` / `challenge_message` /
  `rationale` were authored for the original value and render as-is, so an
  `override` can read as contradictory (value `clerk` alongside a "we recommend
  Devise" challenge_message). The customization model is value-only by design.
  The coherent rendered form of a customized convention's challenge guidance
  (suppress it / derive it from the customization `reason` / add per-field
  customization) is a **product decision deferred to the HELD CRUD slice**
  (Félix/Rémi) — not built here. Not reachable via product today (customizations
  are console/seed-only). Applies identically on both the markdown and for_stacks
  surfaces, so no surface disagrees.
- **Sanitization:** the dup flows through the unchanged `to_agent_markdown` /
  `to_concise_markdown`, so `sanitize_for_markdown` runs on the customized value
  (override) and the whole `"value, custom_value"` (augment) — parity with
  platform/org-authored values. The dup is never persisted.
- `build_markdown`/`build_verbose`/`build_concise` refactored to take the ordered
  `Array<PlaybookConvention>` (they only did `merged.values.group_by`); `project`
  arg drops from them (already applied upstream).

## Part C — for_stacks (cached serialized path)

`for_stacks` serializes playbooks (`PlaybookSerializer`), emitting raw `conv.value`.
Apply the same customizations there:

- Build `custom_map = project ? PlaybookConventionCustomization.effective_map_for(project) : {}`
  inside the cache block; pass as `customizations:` in the serializer context.
- `PlaybookSerializer#serialize_conventions` consults `context[:customizations]`:
  `disable` → skip the convention; `override`/`augment` → emit
  `customization.effective_value` (per-convention rescue → `conv.value`). When the
  map is empty (org-wide, or no customizations) behaviour is byte-identical to today.
- **Cache token** extended to include customizations so an add/remove/edit busts
  the cache (today it keys on assignments only → stale on in-place edits):
  ```ruby
  rel   = project.project_playbook_assignments
  custs = PlaybookConventionCustomization.where(project_playbook_assignment_id: rel.select(:id))
  "#{rel.count},#{rel.maximum(:updated_at)&.to_f || 0}," \
    "#{custs.count},#{custs.maximum(:updated_at)&.to_f || 0}"
  ```
  **Accepted residual:** effective_from/until crossings aren't row changes, so a
  scheduled/expiring customization is bounded by the existing 15-min TTL, not
  instant. Documented; acceptable for a 15-min enforcement cache.
- **Value trust note (for gate-2):** `for_stacks` emits values as JSON data (NOT
  markdown); it does not run `sanitize_for_markdown` today for `conv.value` either.
  Customized values keep that exact parity. Markdown-injection sanitization is the
  markdown-render path's job (Part B) / the CLI's rendering job — unchanged here.

## Part D — dashboard reconciliation (`ProjectPlaybookAssignment#effective_conventions`)

Today it diverges from reality: `augment` returns just `custom_value` (should be
`"value, custom_value"`) and it uses `.active` (ignores effective dates). Fix so
the dashboard **preview == what agents receive**:

```ruby
def effective_conventions
  base = conventions.ordered.to_a
  custom_map = customizations.currently_effective.index_by(&:playbook_convention_id)
  base.filter_map do |convention|
    customization = custom_map[convention.id]
    next if customization&.disables_convention?
    { convention: convention, customization: customization,
      effective_value: customization ? customization.effective_value : convention.value,
      is_customized: customization.present? }
  end
end
```

Per-assignment (its own `customizations` — already tenant-safe). **Output change:**
augment previews now show the concatenated value; scheduled/expired customizations
no longer appear. Low blast radius; noted for owners.

## Part E — for_sync convention_count consistency

`for_sync` recomputes `convention_count` via `merge_playbook_conventions` WITHOUT
customizations → after activation a `disable` makes count > rendered conventions.
Fix: count the post-customization set. Add
`PlaybookConfigurationService.effective_conventions_for(playbooks:, project:)`
(the merged+customized `Array`) used by both content generation and the count so
they can't disagree. (`version_hash` derives from `content` — already correct.)

## Security constraints (gate-1, all MUST)

- Tenant isolation via `effective_map_for` only (project's own assignments +
  org_id); id-match; never global-by-convention-id.
- Fail-safe toward the stricter baseline at map-load and per-convention.
- `currently_effective` everywhere (never `.active`) on enforcement/injection.
- Dup never persisted.
- Disable-abuse of security conventions: not reachable now (no write route);
  the non-disableable lock + authz bar are HELD for Félix/Rémi.

## Tests

Service (`playbook_configuration_service_test`): override replaces value; disable
drops convention; augment shows `"orig, custom"`; no-project unchanged;
scheduled + expired NOT applied; a raising customization load → baseline
(fail-safe); cross-key customization (targets platform row overridden by org) does
NOT apply.
Serializer/for_stacks (`playbooks_controller` / serializer test): customization
reflected in serialized conventions; org-wide unaffected; cache token changes when
a customization is added/edited.
Dashboard (`project_playbook_assignment_test`): augment now concatenated; scheduled
excluded; disable dropped.
for_sync count: disable reduces `convention_count`.
**Sentinel (`test/sentinels/`):** org B's `for_sync` AND `for_stacks` output is
unchanged by org A's customization on a shared platform convention id (cross-tenant).

## Gates

`PARALLEL_WORKERS=1 bin/rails test` on the affected files + `test/sentinels/` on
`crewkit_test_wt1`; `rubocop`; `bin/rails openapi:validate STRICT=1` (for_stacks
response shape unchanged — values are same fields — so no spec change expected).
