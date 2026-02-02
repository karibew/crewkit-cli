---
doc_tier: 1
doc_type: plan
doc_status: active
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
related_docs:
  - docs/prds/API_STANDARDIZATION_PLAN.md
tags: [api, migration, checklist]
---

# API Migration Checklist - Controller Audit

**Status**: Active
**Created**: 2026-01-03

This checklist tracks the migration of all API controllers to use the standardized response format with serializers.

---

## Migration Status Summary

| Status | Count | Percentage |
|--------|-------|------------|
| ✅ Compliant (Using Serializers) | 2 | 15% |
| ⚠️ Partially Compliant | 1 | 8% |
| ❌ Needs Migration | 11 | 85% |

**Total Controllers Audited**: 13 (excluding admin controllers)

---

## ✅ Compliant Controllers (Using Serializers)

These controllers already use the BaseSerializer pattern with HATEOAS links.

### 1. ResourcesController
- **File**: `api/app/controllers/api/v1/resources_controller.rb`
- **Serializer**: `ResourceSerializer`
- **Format**: `{ data:, _links: }` ✅
- **Status**: **No action needed**

**Actions**:
- `index` - Collection with pagination
- `show` - Single resource
- `create` - Single resource (201)
- `update` - Single resource
- `destroy` - 204 No Content
- `effective` - Bulk effective configs
- `effective_single` - Merged config
- `tiered` - Editor breakdown
- `publish` - Action endpoint
- `fork` - Action endpoint

**Notes**: Exemplary implementation. Uses `HateoasResponses` concern.

---

### 2. ProjectsController
- **File**: `api/app/controllers/api/v1/projects_controller.rb`
- **Serializer**: `ProjectSerializer`
- **Format**: `{ data:, _links: }` ✅
- **Status**: **No action needed**

**Actions**:
- `index` - Collection with pagination
- `show` - Single resource
- `create` - Single resource (201)
- `update` - Single resource
- `destroy` - Custom response with message
- `resolve` - Custom response (project resolution)
- `slug_available` - Custom response (availability check)

**Notes**: Properly uses serializers. `resolve` and `slug_available` use custom format (acceptable for utility endpoints).

---

## ⚠️ Partially Compliant Controllers

These controllers use the correct structure but have minor issues.

### 3. SessionAnalyticsController
- **File**: `api/app/controllers/api/v1/session_analytics_controller.rb`
- **Serializer**: None (manual JSON)
- **Format**: `{ data:, meta:, links: }` ⚠️
- **Issue**: Uses `links` instead of `_links`
- **Priority**: **HIGH** (easy fix)

**Actions**:
- `summary` - KPIs
- `timeseries` - Time-series data
- `by_agent` - Grouped stats
- `by_user` - Grouped stats
- `by_project` - Grouped stats
- `cost_breakdown` - Cost analysis
- `success_trend` - Trend data
- `coaching_effectiveness` - Coaching metrics

**Migration Steps**:
1. Global find-replace: `links:` → `_links:`
2. Update OpenAPI docs (Rswag)
3. Update CLI parser (if consuming analytics endpoints)
4. **Estimated Effort**: 1-2 hours

---

## ❌ Controllers Needing Migration

These controllers use manual JSON builders and need to be migrated to use serializers.

### 4. SessionsController ⭐ (High Priority)
- **File**: `api/app/controllers/api/v1/sessions_controller.rb`
- **Current Format**: `{ sessions: [...] }`, `{ session: {...} }`
- **Needs**: `SessionSerializer`
- **Priority**: **HIGH** (frequently used by CLI)

**Actions**:
- `index` - Returns `{ sessions:, meta: }`
- `show` - Returns `{ session: }`
- `create` - Returns `{ session: }`
- `update` - Returns `{ session: }`
- `summary` - Returns `{ session:, modifications:, duration:, cost_per_turn: }`
- `end_session` - Returns `{ session:, message: }`
- `statistics` - Returns `{ statistics: }`

**Manual JSON Helpers**:
- `session_json(session)`
- `modification_json(modification)`
- `calculate_statistics(scope)`

**Migration Steps**:
1. Create `SessionSerializer` with HATEOAS links
2. Replace `session_json` calls with `render_resource`
3. Replace `index` with `render_collection`
4. Handle `summary` (nested data with modifications)
5. Update tests to expect new format
6. Update CLI session response parsing

**Estimated Effort**: 6-8 hours

---

### 5. AgentsController ⭐ (High Priority)
- **File**: `api/app/controllers/api/v1/agents_controller.rb`
- **Current Format**: `{ agents: [...] }`, `{ agent_name:, config:, ... }`
- **Needs**: `AgentConfigurationSerializer`, `AgentModificationSerializer`
- **Priority**: **HIGH** (core agent management)

**Actions**:
- `index` - Returns `{ agents: [...] }` (base + org + project agents)
- `effective_config` - Returns `{ agent_name:, config:, effective_role:, sources: }`
- `create_modification` - Returns `{ modification: {...} }`
- `show_modification` - Returns `{ modification: {...} }`
- `list_modifications` - Returns `{ modifications:, meta: }`
- `list_configurations` - Returns `{ configurations: [...] }` (base + org + project)
- `show_configuration` - Returns `{ configuration: {...} }`
- `create_configuration` - Returns `{ configuration: {...} }`
- `update_configuration` - Returns `{ configuration: {...} }`
- `destroy_configuration` - 204 No Content ✅

**Manual JSON Helpers**:
- `modification_json(modification)`
- `modification_detail_json(modification)`
- `configuration_json(config, level)`
- `configuration_detail_json(config)`

**Migration Steps**:
1. Create `AgentConfigurationSerializer`
2. Create `AgentModificationSerializer`
3. Migrate all actions to use serializers
4. Handle complex endpoints (`effective_config`, `list_configurations`)
5. Update tests
6. Update CLI agent response parsing

**Estimated Effort**: 8-10 hours

---

### 6. OrganizationsController (Unknown Status)
- **File**: `api/app/controllers/api/v1/organizations_controller.rb`
- **Serializer**: `OrganizationSerializer` (exists)
- **Priority**: **MEDIUM**
- **Action**: **Audit controller to verify serializer usage**

**Estimated Effort**: 1 hour audit + 2-4 hours migration (if needed)

---

### 7. ExperimentsController (Unknown Status)
- **File**: `api/app/controllers/api/v1/experiments_controller.rb` (if exists)
- **Priority**: **MEDIUM** (A/B testing feature)
- **Action**: **Audit controller**

**Estimated Effort**: TBD (controller may not exist yet)

---

### 8. MembersController (Unknown Status)
- **File**: `api/app/controllers/api/v1/members_controller.rb` (if exists)
- **Serializer**: `MemberSerializer` (exists)
- **Priority**: **MEDIUM**
- **Action**: **Audit controller**

**Estimated Effort**: 1 hour audit + 2-4 hours migration (if needed)

---

### 9. GitIntegrationsController (Unknown Status)
- **File**: `api/app/controllers/api/v1/git_integrations_controller.rb` (if exists)
- **Serializer**: `GitIntegrationSerializer` (exists)
- **Priority**: **LOW** (optional feature)
- **Action**: **Audit controller**

**Estimated Effort**: 1 hour audit + 2-3 hours migration (if needed)

---

### 10. ProjectRepositoriesController (Unknown Status)
- **File**: `api/app/controllers/api/v1/project_repositories_controller.rb`
- **Serializer**: `ProjectRepositorySerializer` (exists)
- **Priority**: **LOW**
- **Action**: **Audit controller**

**Estimated Effort**: 1 hour audit + 2-3 hours migration (if needed)

---

### 11. UserGitConnectionsController (Unknown Status)
- **File**: `api/app/controllers/api/v1/user_git_connections_controller.rb`
- **Serializer**: `UserGitConnectionSerializer` (exists)
- **Priority**: **LOW**
- **Action**: **Audit controller**

**Estimated Effort**: 1 hour audit + 2-3 hours migration (if needed)

---

### 12. AuthController (Special Case)
- **File**: `api/app/controllers/api/v1/auth_controller.rb`
- **Current Format**: Custom (JWT tokens, device codes)
- **Priority**: **MEDIUM**
- **Action**: **Review for consistency** (may not need full serialization)

**Notes**: Auth endpoints have unique response formats (tokens, device codes). May warrant custom format but should be documented.

**Estimated Effort**: 2 hours review + documentation

---

### 13. Observability Controllers (Special Case)
- **Files**:
  - `api/app/controllers/api/v1/observability/ingest_controller.rb`
  - `api/app/controllers/api/v1/observability/sessions_controller.rb`
  - `api/app/controllers/api/v1/observability/llm_gateway_controller.rb`
- **Priority**: **LOW** (internal observability)
- **Action**: **Review for consistency**

**Estimated Effort**: 2-3 hours review + migration (if needed)

---

## New Endpoints to Add

### API Root Discovery
- **Endpoint**: `GET /api/v1`
- **Controller**: New `Api::V1::RootController`
- **Purpose**: API discovery, rate limit info
- **Priority**: **MEDIUM**

**Implementation**:
```ruby
# app/controllers/api/v1/root_controller.rb
module Api
  module V1
    class RootController < BaseController
      skip_before_action :set_current_organization

      def index
        render json: {
          api: "crewkit",
          version: "v1",
          _links: {
            self: "/api/v1",
            organizations: { href: "/api/v1/organizations", method: "GET" },
            resources: { href: "/api/v1/resources", method: "GET" },
            documentation: { href: "/api-docs", method: "GET" }
          },
          meta: {
            authenticated: current_user.present?,
            user_id: current_user&.external_id
          }
        }
      end
    end
  end
end
```

**Estimated Effort**: 2-3 hours

---

## Implementation Timeline

### Phase 1: Quick Wins (Week 1)
- [ ] Fix SessionAnalyticsController (`links` → `_links`) - **2 hours**
- [ ] Audit unknown controllers - **4 hours**
- [ ] Create SessionSerializer - **3 hours**
- [ ] Migrate SessionsController - **6 hours**

**Total Phase 1**: ~15 hours

---

### Phase 2: Core Features (Week 2)
- [ ] Create AgentConfigurationSerializer - **2 hours**
- [ ] Create AgentModificationSerializer - **2 hours**
- [ ] Migrate AgentsController - **8 hours**
- [ ] Migrate OrganizationsController (if needed) - **4 hours**

**Total Phase 2**: ~16 hours

---

### Phase 3: Remaining Controllers (Week 3)
- [ ] Migrate MembersController (if needed) - **4 hours**
- [ ] Migrate ExperimentsController (if needed) - **4 hours**
- [ ] Migrate GitIntegrationsController (if needed) - **3 hours**
- [ ] Review AuthController - **2 hours**
- [ ] Add API root discovery endpoint - **3 hours**

**Total Phase 3**: ~16 hours

---

### Phase 4: Documentation & Testing (Week 4)
- [ ] Update all Rswag specs - **8 hours**
- [ ] Update CLI Rust types - **4 hours**
- [ ] Update dashboard API client - **4 hours**
- [ ] Manual curl testing - **2 hours**
- [ ] Integration testing - **4 hours**

**Total Phase 4**: ~22 hours

---

## Total Estimated Effort

| Phase | Effort |
|-------|--------|
| Phase 1 | 15 hours |
| Phase 2 | 16 hours |
| Phase 3 | 16 hours |
| Phase 4 | 22 hours |
| **Total** | **69 hours** |

**Sprint Estimate**: 2-3 weeks (2 developers working part-time)

---

## Success Metrics

- [ ] All controllers use serializers (no manual JSON builders)
- [ ] All responses have `{ data:, _links:, meta: }` format
- [ ] All tests passing (MiniTest)
- [ ] OpenAPI docs updated (Rswag)
- [ ] CLI parsing new format (no Rust compilation errors)
- [ ] Dashboard consuming new format (no TypeScript errors)
- [ ] Zero Sentry errors related to response format (7-day monitoring)

---

## Rollout Plan

1. **Development**: Feature branch `feature/api-standardization`
2. **Testing**: MiniTest + manual curl + CLI integration
3. **Staging**: Deploy with both formats supported (if needed)
4. **Production**: Deploy after 7-day staging soak
5. **Monitoring**: Watch Sentry for 14 days post-deploy

---

## References

- **Full Plan**: `API_STANDARDIZATION_PLAN.md`
- **Quick Guide**: `API_RESPONSE_FORMAT_GUIDE.md`
- **BaseSerializer**: `app/serializers/base_serializer.rb`
- **Example**: `app/serializers/resource_serializer.rb`

---

**Last Updated**: 2026-01-03
**Next Review**: After Phase 1 completion