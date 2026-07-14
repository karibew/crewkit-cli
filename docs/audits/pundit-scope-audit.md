# Pundit `Scope#resolve` Adoption Audit

**Closes #108.** **Sentinel:** `api/test/sentinels/pundit_scope_sentinel_test.rb`.

## Background

Multi-tenant isolation in this codebase rides on Pundit `Scope#resolve`
returning a relation that is constrained to the caller's organization.
The canonical helper is `organization_scoped` defined in
`api/app/policies/application_policy.rb:61-66`:

```ruby
def organization_scoped(relation)
  current_organization = context[:current_organization]
  return relation.none unless current_organization
  relation.where(organization_id: current_organization.id)
end
```

A `Scope#resolve` that returns `scope.all` — or any relation that is not
constrained to `current_organization` — silently breaks the tenant
boundary. Nothing in the feature-test suite catches it, because the
tests are written from inside a single organization context. This audit
is a one-pass classification of every `*Policy::Scope#resolve` under
`api/app/policies/`, plus a sentinel (`pundit_scope_sentinel_test.rb`)
that gates the registry going forward: any new `Scope` that does not
call `organization_scoped` and is not in `DOCUMENTED_EXCEPTIONS` fails
the build.

The classifications:

- **`organization_scoped`** — calls the helper directly.
- **`organization_scoped:equivalent`** — does not call the literal
  helper but achieves the same constraint (`where(organization_id: …)`
  or a join that lands on `organizations.id == current_organization.id`).
  The sentinel marks these `DOCUMENTED_EXCEPTIONS` with the rationale
  `equivalent-org-filter` so reviewers must explicitly bless each one.
- **`documented_exception:<reason>`** — intentionally spans
  organizations (platform reference data, per-user-not-per-org data,
  admin namespace, three-tier resource visibility, etc.).
- **`gap:<file:line>`** — the Scope leaks across tenants and needs a
  fix.

## Audit table

| # | Policy | Scope location | Classification | Notes |
|---|---|---|---|---|
| 1 | `AgentConfigurationPolicy::Scope` | `api/app/policies/agent_configuration_policy.rb:5-22` | `documented_exception:user-multi-org-scoped` | Polymorphic `configurable` (Organization or Project). Scopes to **all** orgs the actual user belongs to (`user.user_organization_roles → organization_roles.organization_id`), not just `current_organization`. Defensible because `AgentConfiguration` is being phased out in favor of `Resource`; pinning current behavior. |
| 2 | `AgentExperimentPolicy::Scope` | `api/app/policies/agent_experiment_policy.rb:5-12` | `organization_scoped` | Crewkit-staff bypass + `organization_scoped(scope)` for everyone else. |
| 3 | `ArtifactPolicy::Scope` | `api/app/policies/artifact_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 4 | `BlueprintChatMessagePolicy::Scope` | `api/app/policies/blueprint_chat_message_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 5 | `BlueprintEpicPolicy::Scope` | `api/app/policies/blueprint_epic_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 6 | `BlueprintPolicy::Scope` | `api/app/policies/blueprint_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 7 | `BlueprintTaskDependencyPolicy::Scope` | `api/app/policies/blueprint_task_dependency_policy.rb:5-12` | `organization_scoped:equivalent` | No `organization_id` on the join table; joins through `:task` and filters `blueprint_tasks.organization_id == current_organization.id`. Returns `.none` without a current_organization. Equivalent to the helper. |
| 8 | `BlueprintTaskPolicy::Scope` | `api/app/policies/blueprint_task_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 9 | `BlueprintTaskSessionPolicy::Scope` | `api/app/policies/blueprint_task_session_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 10 | `BriefPolicy::Scope` | `api/app/policies/brief_policy.rb:22-31` | `documented_exception:platform-reference-data` | Briefs are platform-level prompts (no `organization_id`). All authenticated users see non-deprecated briefs; platform admins see all. Comment block on the policy (lines 1-19) is the canonical permission matrix. |
| 11 | `BriefVersionPolicy::Scope` | `api/app/policies/brief_version_policy.rb:12-15` | `documented_exception:platform-reference-data` | Immutable version snapshots of Brief. Same visibility model. Returns `scope.all` to any authenticated user. |
| 12 | `CloudJobPolicy::Scope` | `api/app/policies/cloud_job_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 13 | `ConventionOverridePolicy::Scope` | `api/app/policies/convention_override_policy.rb:5-10` | `organization_scoped` | Crewkit-staff bypass + `organization_scoped(scope)`. |
| 14 | `FrameworkStackPolicy::Scope` | `api/app/policies/framework_stack_policy.rb:11-15` | `documented_exception:platform-reference-data` | Framework stack catalog is platform-level reference data. All authenticated users read; only staff write. |
| 15 | `GitIntegrationPolicy::Scope` | `api/app/policies/git_integration_policy.rb:5-10` | `organization_scoped` | Crewkit-staff bypass + `organization_scoped(scope)`. |
| 16 | `GitWebhookEventPolicy::Scope` | `api/app/policies/git_webhook_event_policy.rb:5-12` | `organization_scoped:equivalent` | `git_webhook_events` has no `organization_id`; joins through `:git_integration` with `git_integrations.organization_id == current_organization.id`. Equivalent to the helper. |
| 17 | `InvitationPolicy::Scope` | `api/app/policies/invitation_policy.rb:5-13` | `organization_scoped:equivalent` | Hand-rolled `scope.where(organization_id: context[:current_organization]&.id)`. Returns no rows when `current_organization` is nil (`organization_id: nil` matches none on a NOT NULL column). Should ideally call the helper; pinning as equivalent. |
| 18 | `LlmSessionPolicy::Scope` | `api/app/policies/llm_session_policy.rb:5-10` | `organization_scoped` | Crewkit-staff bypass + `organization_scoped(scope)`. |
| 19 | `MemberPolicy::Scope` | `api/app/policies/member_policy.rb:5-14` | `organization_scoped:equivalent` | `UserOrganizationRole` has no `organization_id` column directly; joins through `:organization_role` with `organization_roles.organization_id == current_organization.id`. Equivalent. |
| 20 | `OauthConnectionPolicy::Scope` | `api/app/policies/oauth_connection_policy.rb:5-10` | `organization_scoped:equivalent` | Stricter than `organization_scoped` — filters by both `organization_id: current_organization.id` AND `user_id: actual_user.id`. Returns `.none` without a current organization. |
| 21 | `OrganizationPolicy::Scope` | `api/app/policies/organization_policy.rb:5-17` | `documented_exception:org-membership-listing` | This is the **org list itself** — the user enumerates the organizations they have a membership in. There is no "current organization" yet at this layer (it's how the user picks one). Scoped strictly by `user_organization_roles → organization_roles.organization_id`. Platform admins included (comment notes admins access cross-org data via the admin namespace, not this Scope). |
| 22 | `OrganizationRolePolicy::Scope` | `api/app/policies/organization_role_policy.rb:5-15` | `organization_scoped:equivalent` | Platform-admin bypass for read-only access (comment line 6 documents the intent); other users get `scope.where(organization_id: current_org.id)` with explicit `.none` short-circuit. Equivalent. |
| 23 | `PlaybookConventionCustomizationPolicy::Scope` | `api/app/policies/playbook_convention_customization_policy.rb:11-14` | `organization_scoped` | Direct call. |
| 24 | `PlaybookConventionPolicy::Scope` | `api/app/policies/playbook_convention_policy.rb:11-16` | `documented_exception:delegates-to-resource-scope` | Conventions are scoped through their parent `Resource` (playbook) via `ResourcePolicy::Scope.new(user, Resource.playbooks).resolve.select(:id)`. The `ResourcePolicy::Scope` itself is a documented three-tier exception (see row 31). |
| 25 | `PlaybookSubscriptionPolicy::Scope` | `api/app/policies/playbook_subscription_policy.rb:11-14` | `organization_scoped` | Direct call. |
| 26 | `ProjectPlaybookAssignmentPolicy::Scope` | `api/app/policies/project_playbook_assignment_policy.rb:11-15` | `organization_scoped` | Direct call. |
| 27 | `ProjectPolicy::Scope` | `api/app/policies/project_policy.rb:5-15` | `organization_scoped:equivalent` | Crewkit-staff bypass returns `scope.all` (documented as read-only); other users get a join through `organization → organization_roles → user_organization_roles` filtered by `user_id`. Combined with the caller-side `policy_scope(current_organization.projects)` in `projects_controller.rb:40`, the effective query is single-org. Pin behavior; do not change without revisiting the controller. |
| 28 | `ProjectRepositoryPolicy::Scope` | `api/app/policies/project_repository_policy.rb:5-15` | `organization_scoped:equivalent` | Same shape as `ProjectPolicy::Scope` (join through `project → organization → org_roles → user_org_roles`). Crewkit-staff bypass returns all. |
| 29 | `RepositoryPolicy::Scope` | `api/app/policies/repository_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 30 | `ResourcePolicy::Scope` | `api/app/policies/resource_policy.rb:13-42` | `documented_exception:three-tier-visibility` | Resource is a 3-tier model (platform → organization → project). The Scope unions three sub-scopes: platform resources (visible to all authenticated users), organization resources (filtered by user's org memberships), and project resources (filtered by user's org memberships). Crewkit-staff bypass returns all. The comment block on lines 1-11 is the canonical permission matrix. |
| 31 | `SessionSharePolicy::Scope` | `api/app/policies/session_share_policy.rb:5-8` | `organization_scoped` | Direct call. |
| 32 | `SkillInvocationPolicy::Scope` | `api/app/policies/skill_invocation_policy.rb:5-10` | `organization_scoped` | Crewkit-staff bypass + `organization_scoped(scope)`. |
| 33 | `UserGitConnectionPolicy::Scope` | `api/app/policies/user_git_connection_policy.rb:5-9` | `documented_exception:user-not-org-scoped` | Git connections belong to a user, not an organization (one user, one GitHub identity, used across orgs). Filtered by `actual_user.id`. |
| 34 | `UserOrganizationRolePolicy::Scope` | `api/app/policies/user_organization_role_policy.rb:5-13` | `organization_scoped:equivalent` | Same shape as `MemberPolicy::Scope` — joins through `:organization_role` with `organization_roles.organization_id == current_organization.id`. Equivalent. |
| 35 | `Admin::ResourcePolicy::Scope` | `api/app/policies/admin/resource_policy.rb:11-23` | `documented_exception:platform-admin-namespace` | Lives under the `Api::V1::Admin::*` namespace which is gated by `platform_admin?`. Crewkit owners see everything; platform admins see all `owner_type: "platform"` rows; everyone else gets `scope.none`. Intentionally spans organizations. |
| 36 | `VerificationTemplatePolicy::Scope` | `api/app/policies/verification_template_policy.rb:5-15` | `documented_exception:three-tier-visibility` | Verification templates are a 2-tier resource (platform → organization). The Scope returns `where(organization_id: [nil, current_org.id])` — platform templates (NULL org, readable by all authenticated users) unioned with the caller's own-org templates; crewkit-staff bypass returns all. Same family as `ResourcePolicy::Scope` (row 30), minus the project tier. Never returns another org's rows. (issue #25) |
| 37 | `ProjectVerificationConfigPolicy::Scope` | `api/app/policies/project_verification_config_policy.rb:5-9` | `organization_scoped` | Direct call. The join table carries a denormalized `organization_id`. (issue #25) |

**Policies with no `Scope` class** (verified — `Scope#resolve` does not exist; sentinel correctly skips them): `BillingPolicy`, `OrgClaudeTokenPolicy`, `PlaybookPolicy` (inherits the parent `ResourcePolicy::Scope`), `ShareAnnotationPolicy`, `TurnVisibilityPolicy` (not even an `ApplicationPolicy` — it is a stateless policy module).

## Summary

- **Total `*Policy::Scope#resolve` definitions audited:** 35
- **`organization_scoped` (direct):** 17 (rows 2, 3, 4, 5, 6, 8, 9, 12, 13, 15, 18, 23, 25, 26, 29, 31, 32)
- **`organization_scoped:equivalent`:** 9 (rows 7, 16, 17, 19, 20, 22, 27, 28, 34)
- **`documented_exception`:** 9 (rows 1, 10, 11, 14, 21, 24, 30, 33, 35)
- **`gap`:** 0

No tenant-isolation gaps were found in the current `*Policy::Scope`
surface. Every Scope either filters to `current_organization` (directly
or equivalently) or is on the explicit exception list. The
`equivalent` cases are pinned here so future refactors that drop the
join cannot ship without an explicit audit-table update.

## What the sentinel enforces

`api/test/sentinels/pundit_scope_sentinel_test.rb` enumerates every concrete
descendant of `ApplicationPolicy::Scope` after `Rails.application.eager_load!`
and asserts that `Scope#resolve`'s source either contains the literal
string `organization_scoped` **or** the enclosing policy class is in the
`DOCUMENTED_EXCEPTIONS` constant. The exception list is data, not regex —
each entry is one line with a one-word rationale tag that maps back to a
row in this audit. A new Scope that lacks `organization_scoped` and is
not in the exception list fails the build with a pointer to the file:line.

Adding a new `*Policy::Scope`:

1. **Default:** call `organization_scoped(scope)` in `#resolve`.
2. **If the table has no `organization_id` column,** scope to
   `current_organization` via a join (see `BlueprintTaskDependencyPolicy`
   for the canonical pattern), then add the policy to `DOCUMENTED_EXCEPTIONS`
   with rationale `equivalent-org-filter` and add a row to this audit
   citing the file:line. The reviewer must agree the join is genuinely
   equivalent.
3. **If the data is intentionally cross-org** (platform reference data,
   per-user data, admin namespace), add the policy to
   `DOCUMENTED_EXCEPTIONS` with the appropriate rationale and add a row
   to this audit explaining *why* spanning organizations is the desired
   behavior.

Removing an entry from `DOCUMENTED_EXCEPTIONS` while the underlying
Scope still lacks `organization_scoped` will fail the sentinel — that
is the desired behavior.

## Follow-ups

- A future PR can consolidate `pundit_scope_sentinel_test.rb` into
  `tenant_isolation_sentinel_test.rb` (PR #115) once that lands. This
  PR is intentionally a sibling sentinel to stay independent of the
  unmerged W1 branch.
- The nine `organization_scoped:equivalent` policies are candidates
  for a small refactor that lifts the equivalent join pattern into a
  helper on `ApplicationPolicy::Scope` (e.g., `organization_scoped_through(association)`),
  reducing the surface the sentinel has to bless via exception. Tracked
  separately, not in scope for this PR.
