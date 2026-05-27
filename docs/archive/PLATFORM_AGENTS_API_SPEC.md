---
doc_tier: 1
doc_type: spec
doc_status: active
created: 2025-11-01
last_reviewed: 2026-01-06
owner: platform-team
tags: [api, agents, specification]
---

# Platform Base Agents API Specification

**Version**: v1
**Namespace**: `/api/v1/platform/base_agents`
**Authentication**: JWT (devise-jwt)
**Authorization**: Platform roles only (`crewkit_owner`, `crewkit_staff`, `platform_admin`)

---

## Design Decisions

### 1. Namespace: `/api/v1/platform/`

**Rationale**: Clearly separates platform admin functionality from organization-level endpoints. Signals privileged access.

- `/api/v1/agents` → Organization-level (all users)
- `/api/v1/platform/base_agents` → Platform admin only

### 2. Endpoint Structure: Hybrid RESTful + Custom Actions

**Standard REST**: CRUD operations use conventional HTTP methods
**Custom Actions**: State transitions and complex operations use POST to named sub-resources

```ruby
# REST
GET    /api/v1/platform/base_agents
POST   /api/v1/platform/base_agents
PATCH  /api/v1/platform/base_agents/:id

# Custom actions (POST for state changes)
POST   /api/v1/platform/base_agents/:id/publish
POST   /api/v1/platform/base_agents/:id/deprecate
POST   /api/v1/platform/base_agents/:id/rollback
```

**Rationale**: POST for state changes is more explicit than `PATCH {status: "published"}` and allows validation logic specific to transitions.

### 3. Preview Endpoint: Query Params (Read-Only)

```
GET /api/v1/platform/base_agents/:id/preview?org_id=123&project_id=456&role=junior
```

**Rationale**: Preview is read-only, query params are sufficient. No need for POST body complexity.

### 4. Versioning: Nested Resource

```
GET /api/v1/platform/base_agents/:id/versions
GET /api/v1/platform/base_agents/:id/versions/:version_number
```

**Rationale**: Versions are conceptually children of base agents. Nested route reflects this relationship.

### 5. Filtering: Rails Standard Query Params

```
?category=development&status=published&tags[]=rails&tags[]=backend
```

**Rationale**: Simpler than JSON:API filter syntax, familiar to Rails developers. Easy to cache by URL.

### 6. Caching: ETags + Cache-Control

**Strategy**: Combine ETags (strong validation) with Cache-Control (expiration)

```http
Cache-Control: public, max-age=300
ETag: "a1b2c3d4e5f6"
```

**Rationale**: ETags enable conditional requests (304 Not Modified), Cache-Control enables browser/CDN caching.

### 7. Rate Limiting: Conservative Limits

**Platform admin endpoints**: 1000 requests/hour per user
**Public read endpoints**: 5000 requests/hour per user

**Rationale**: Platform admins need higher limits for bulk operations, but still prevent abuse.

---

## Complete Endpoint List

### Core CRUD

| Method | Path | Description | Auth | Cache |
|--------|------|-------------|------|-------|
| GET | `/api/v1/platform/base_agents` | List all base agents (with filtering) | Platform admin | 5 min |
| GET | `/api/v1/platform/base_agents/:id` | Get single base agent | Platform admin | 5 min |
| POST | `/api/v1/platform/base_agents` | Create draft agent | Platform admin | — |
| PATCH | `/api/v1/platform/base_agents/:id` | Update agent (auto-save) | Platform admin | Invalidate |
| DELETE | `/api/v1/platform/base_agents/:id` | Delete draft agent (published cannot be deleted) | Platform admin | Invalidate |

### State Transitions

| Method | Path | Description | Auth | Cache |
|--------|------|-------------|------|-------|
| POST | `/api/v1/platform/base_agents/:id/publish` | Publish draft agent | Platform admin | Invalidate |
| POST | `/api/v1/platform/base_agents/:id/deprecate` | Deprecate published agent | Platform admin | Invalidate |
| POST | `/api/v1/platform/base_agents/:id/rollback` | Rollback to previous version | Platform admin | Invalidate |

### Versioning

| Method | Path | Description | Auth | Cache |
|--------|------|-------------|------|-------|
| GET | `/api/v1/platform/base_agents/:id/versions` | List all versions of agent | Platform admin | 5 min |
| GET | `/api/v1/platform/base_agents/:id/versions/:version_number` | Get specific version | Platform admin | 5 min |

### Testing & Analytics

| Method | Path | Description | Auth | Cache |
|--------|------|-------------|------|-------|
| GET | `/api/v1/platform/base_agents/:id/preview` | Preview effective config | Platform admin | 1 min |
| GET | `/api/v1/platform/base_agents/:id/analytics` | Get usage analytics | Platform admin | 5 min |

### Bulk Operations

| Method | Path | Description | Auth | Cache |
|--------|------|-------------|------|-------|
| POST | `/api/v1/platform/base_agents/bulk_update` | Update multiple agents | Platform admin | Invalidate |
| GET | `/api/v1/platform/base_agents/categories` | List all categories (for filters) | Platform admin | 1 hour |
| GET | `/api/v1/platform/base_agents/tags` | List all tags (for autocomplete) | Platform admin | 1 hour |

---

## Request/Response Formats

### 1. List All Base Agents

**Request:**
```http
GET /api/v1/platform/base_agents?category=development&status=published&page=1&per_page=25&sort=name&order=asc
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
```ruby
{
  category: "development" | "frontend" | "backend" | "devops" | "data" | "design",
  status: "draft" | "published" | "deprecated",
  tags: ["rails", "backend"], # Array, matches ANY tag
  search: "rails expert", # Searches name + description
  page: 1,
  per_page: 25, # Max 100
  sort: "name" | "created_at" | "published_at" | "organizations_count" | "sessions_count",
  order: "asc" | "desc"
}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "rails-expert",
      "category": "development",
      "tags": ["rails", "backend", "ruby"],
      "description": "Rails 8.0 specialist with Hotwire expertise",
      "status": "published",
      "version_number": 3,
      "created_by": {
        "id": 5,
        "email": "admin@crewkit.io",
        "name": "Platform Admin"
      },
      "published_by": {
        "id": 5,
        "email": "admin@crewkit.io",
        "name": "Platform Admin"
      },
      "published_at": "2025-10-10T14:30:00Z",
      "organizations_count": 47,
      "sessions_count": 1823,
      "created_at": "2025-09-15T10:00:00Z",
      "updated_at": "2025-10-10T14:30:00Z"
    }
  ],
  "meta": {
    "total": 12,
    "page": 1,
    "per_page": 25,
    "total_pages": 1
  }
}
```

**Response Headers:**
```http
Cache-Control: public, max-age=300
ETag: "a1b2c3d4e5f6"
X-Total-Count: 12
X-Page: 1
X-Per-Page: 25
```

**Performance Target**: < 100ms (p95)

---

### 2. Get Single Base Agent

**Request:**
```http
GET /api/v1/platform/base_agents/1
Authorization: Bearer <jwt_token>
```

**Response (200 OK):**
```json
{
  "data": {
    "id": 1,
    "name": "rails-expert",
    "category": "development",
    "tags": ["rails", "backend", "ruby"],
    "description": "Rails 8.0 specialist with Hotwire expertise",
    "content": "# Rails Expert Agent\n\nYou are a Rails 8.0 expert...",
    "frontmatter": {
      "name": "rails-expert",
      "version": "1.0.0",
      "model": "claude-sonnet-4-5",
      "temperature": 0.7
    },
    "status": "published",
    "version_number": 3,
    "parent_version_id": 2,
    "created_by": {
      "id": 5,
      "email": "admin@crewkit.io",
      "name": "Platform Admin"
    },
    "published_by": {
      "id": 5,
      "email": "admin@crewkit.io",
      "name": "Platform Admin"
    },
    "published_at": "2025-10-10T14:30:00Z",
    "deprecated_at": null,
    "organizations_count": 47,
    "sessions_count": 1823,
    "created_at": "2025-09-15T10:00:00Z",
    "updated_at": "2025-10-10T14:30:00Z"
  }
}
```

**Response Headers:**
```http
Cache-Control: public, max-age=300
ETag: "xyz123"
```

**Performance Target**: < 50ms (p95)

---

### 3. Create Draft Agent

**Request:**
```http
POST /api/v1/platform/base_agents
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "name": "rails-expert",
  "category": "development",
  "tags": ["rails", "backend", "ruby"],
  "description": "Rails 8.0 specialist",
  "content": "# Rails Expert Agent\n\nYou are...",
  "frontmatter": {
    "name": "rails-expert",
    "version": "1.0.0",
    "model": "claude-sonnet-4-5"
  }
}
```

**Validation Rules:**
- `name`: Required, unique, lowercase, hyphenated (regex: `/^[a-z][a-z0-9-]*$/`)
- `category`: Required, must be in allowed list
- `tags`: Optional, array of strings
- `description`: Required, 10-500 characters
- `content`: Required, valid markdown
- `frontmatter`: Optional, valid YAML

**Response (201 Created):**
```json
{
  "data": {
    "id": 42,
    "name": "rails-expert",
    "category": "development",
    "tags": ["rails", "backend", "ruby"],
    "description": "Rails 8.0 specialist",
    "content": "# Rails Expert Agent\n\nYou are...",
    "frontmatter": {
      "name": "rails-expert",
      "version": "1.0.0",
      "model": "claude-sonnet-4-5"
    },
    "status": "draft",
    "version_number": 1,
    "created_by": {
      "id": 5,
      "email": "admin@crewkit.io",
      "name": "Platform Admin"
    },
    "created_at": "2025-10-15T16:45:00Z",
    "updated_at": "2025-10-15T16:45:00Z"
  }
}
```

**Performance Target**: < 200ms (p95)

---

### 4. Update Agent (Auto-Save)

**Request:**
```http
PATCH /api/v1/platform/base_agents/42
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "content": "# Updated Rails Expert Agent\n\nYou are...",
  "description": "Updated description",
  "tags": ["rails", "backend", "ruby", "hotwire"]
}
```

**Response (200 OK):**
```json
{
  "data": {
    "id": 42,
    "name": "rails-expert",
    "content": "# Updated Rails Expert Agent...",
    "description": "Updated description",
    "tags": ["rails", "backend", "ruby", "hotwire"],
    "updated_at": "2025-10-15T16:46:30Z"
  }
}
```

**Notes:**
- Only draft agents can be updated (published agents require new version)
- Auto-save should be debounced client-side (30s)
- Use optimistic locking to prevent conflicts (`If-Match: <etag>`)

**Performance Target**: < 100ms (p95)

---

### 5. Publish Agent

**Request:**
```http
POST /api/v1/platform/base_agents/42/publish
Authorization: Bearer <jwt_token>
```

**Response (200 OK):**
```json
{
  "data": {
    "id": 42,
    "name": "rails-expert",
    "status": "published",
    "version_number": 1,
    "published_by": {
      "id": 5,
      "email": "admin@crewkit.io",
      "name": "Platform Admin"
    },
    "published_at": "2025-10-15T17:00:00Z"
  }
}
```

**Business Rules:**
- Only draft agents can be published
- Publishing creates immutable version
- Invalidates all caches for this agent
- Triggers sync to Git (during migration phase)

**Performance Target**: < 500ms (p95, includes cache invalidation)

---

### 6. Deprecate Agent

**Request:**
```http
POST /api/v1/platform/base_agents/42/deprecate
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "reason": "Replaced by rails-expert-v2"
}
```

**Response (200 OK):**
```json
{
  "data": {
    "id": 42,
    "name": "rails-expert",
    "status": "deprecated",
    "deprecated_at": "2025-10-15T17:05:00Z"
  }
}
```

**Business Rules:**
- Only published agents can be deprecated
- Deprecated agents remain visible but marked
- Organizations can still use deprecated agents (no breaking changes)

---

### 7. Rollback to Previous Version

**Request:**
```http
POST /api/v1/platform/base_agents/42/rollback
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "version_number": 2
}
```

**Response (201 Created):**
```json
{
  "data": {
    "id": 99,
    "name": "rails-expert",
    "status": "draft",
    "version_number": 4,
    "parent_version_id": 2,
    "content": "# Content from version 2...",
    "created_at": "2025-10-15T17:10:00Z"
  },
  "message": "Created draft from version 2. Review and publish to deploy."
}
```

**Business Rules:**
- Creates new draft from previous version (never modifies published versions)
- Admin must review and publish manually
- Preserves audit trail

---

### 8. Preview Effective Config

**Request:**
```http
GET /api/v1/platform/base_agents/42/preview?org_id=123&project_id=456&role=junior
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
```ruby
{
  org_id: 123, # Optional, use default if not provided
  project_id: 456, # Optional
  role: "junior" | "intermediate" | "senior" # Optional, default: "intermediate"
}
```

**Response (200 OK):**
```json
{
  "data": {
    "base_agent": {
      "name": "rails-expert",
      "content": "# Rails Expert Agent..."
    },
    "org_override": {
      "prompt_suffix": "Always use our coding standards..."
    },
    "project_override": {
      "context": "This project uses Rails 8.0 with Hotwire..."
    },
    "role_modifier": {
      "coaching_mode": true,
      "prompt_prefix": "CRITICAL: You are in COACHING MODE..."
    },
    "effective_config": {
      "name": "rails-expert",
      "content": "# MERGED CONTENT WITH ALL OVERRIDES..."
    }
  }
}
```

**Performance Target**: < 100ms (p95, cache org/project configs)

---

### 9. List Versions

**Request:**
```http
GET /api/v1/platform/base_agents/42/versions
Authorization: Bearer <jwt_token>
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": 42,
      "version_number": 3,
      "status": "published",
      "published_by": {
        "id": 5,
        "email": "admin@crewkit.io"
      },
      "published_at": "2025-10-15T14:30:00Z",
      "change_summary": "Updated Hotwire patterns"
    },
    {
      "id": 38,
      "version_number": 2,
      "status": "deprecated",
      "published_at": "2025-10-01T10:00:00Z"
    },
    {
      "id": 24,
      "version_number": 1,
      "status": "deprecated",
      "published_at": "2025-09-15T09:00:00Z"
    }
  ]
}
```

**Performance Target**: < 50ms (p95)

---

### 10. Get Usage Analytics

**Request:**
```http
GET /api/v1/platform/base_agents/42/analytics?period=30d
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
```ruby
{
  period: "7d" | "30d" | "90d" | "all" # Default: 30d
}
```

**Response (200 OK):**
```json
{
  "data": {
    "total_organizations": 47,
    "total_sessions": 1823,
    "active_sessions_last_7d": 234,
    "organizations": [
      {
        "id": 123,
        "name": "Acme Corp",
        "sessions_count": 342,
        "last_used_at": "2025-10-15T12:00:00Z"
      }
    ],
    "sessions_by_day": [
      {"date": "2025-10-15", "count": 45},
      {"date": "2025-10-14", "count": 38}
    ],
    "average_session_duration_minutes": 24.5,
    "success_rate": 0.94
  }
}
```

**Performance Target**: < 200ms (p95, use cached aggregates)

---

### 11. Bulk Update

**Request:**
```http
POST /api/v1/platform/base_agents/bulk_update
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "ids": [42, 43, 44],
  "updates": {
    "category": "backend",
    "tags": ["rails", "ruby"]
  }
}
```

**Response (200 OK):**
```json
{
  "data": {
    "updated_count": 3,
    "failed": []
  }
}
```

**Business Rules:**
- Only draft agents can be bulk updated
- All updates succeed or all fail (transaction)

---

### 12. List Categories

**Request:**
```http
GET /api/v1/platform/base_agents/categories
Authorization: Bearer <jwt_token>
```

**Response (200 OK):**
```json
{
  "data": [
    {"slug": "development", "name": "Development", "count": 5},
    {"slug": "frontend", "name": "Frontend", "count": 3},
    {"slug": "backend", "name": "Backend", "count": 2},
    {"slug": "devops", "name": "DevOps", "count": 1},
    {"slug": "data", "name": "Data & ML", "count": 1}
  ]
}
```

**Performance Target**: < 20ms (1-hour cache)

---

### 13. List Tags (Autocomplete)

**Request:**
```http
GET /api/v1/platform/base_agents/tags?q=rai
Authorization: Bearer <jwt_token>
```

**Response (200 OK):**
```json
{
  "data": [
    {"tag": "rails", "count": 12},
    {"tag": "railway", "count": 2}
  ]
}
```

**Performance Target**: < 50ms (1-hour cache)

---

## Standardized Error Response Format

All errors follow this format:

```json
{
  "error": "Human-readable error message",
  "code": "ERROR_CODE",
  "details": {
    "field": "specific_field",
    "reason": "validation failed"
  },
  "message": "Actionable guidance for fixing the error",
  "documentation_url": "https://docs.crewkit.io/api/errors#ERROR_CODE",
  "request_id": "req_abc123xyz"
}
```

### Error Codes

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `INVALID_REQUEST` | Malformed JSON or missing required fields |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT token |
| 403 | `FORBIDDEN` | Insufficient permissions (not platform admin) |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Name already exists, version conflict |
| 422 | `VALIDATION_ERROR` | Business rule validation failed |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |

### Example Error Responses

**401 Unauthorized:**
```json
{
  "error": "Authentication required",
  "code": "UNAUTHORIZED",
  "message": "Please log in with 'crewkit auth login' and try again",
  "documentation_url": "https://docs.crewkit.io/api/authentication",
  "request_id": "req_abc123"
}
```

**403 Forbidden:**
```json
{
  "error": "Insufficient permissions",
  "code": "FORBIDDEN",
  "details": {
    "required_role": "platform_admin",
    "current_role": "org_user"
  },
  "message": "Only platform administrators can manage base agents",
  "request_id": "req_def456"
}
```

**422 Validation Error:**
```json
{
  "error": "Validation failed",
  "code": "VALIDATION_ERROR",
  "details": {
    "name": ["has already been taken"],
    "content": ["cannot be blank"],
    "category": ["is not included in the list"]
  },
  "message": "Please fix the validation errors and try again",
  "request_id": "req_ghi789"
}
```

**409 Conflict (Publishing):**
```json
{
  "error": "Cannot publish agent in current state",
  "code": "INVALID_STATE_TRANSITION",
  "details": {
    "current_status": "published",
    "attempted_action": "publish"
  },
  "message": "Only draft agents can be published. This agent is already published.",
  "suggested_action": "Create a new version if you want to make changes",
  "request_id": "req_jkl012"
}
```

---

## Caching Strategy

### Cache Layers

1. **Application Cache (Solid Cache)**: Database-backed, shared across servers
2. **HTTP Cache (ETags + Cache-Control)**: Browser/CDN caching
3. **Query Result Cache**: Rails query caching within request

### Caching Rules

| Endpoint | Cache Type | TTL | Invalidation |
|----------|-----------|-----|--------------|
| `GET /base_agents` | HTTP + App | 5 min | On create/update/publish |
| `GET /base_agents/:id` | HTTP + App | 5 min | On update |
| `GET /base_agents/:id/preview` | App only | 1 min | On config change |
| `GET /base_agents/:id/analytics` | App only | 5 min | On session end |
| `GET /base_agents/categories` | HTTP + App | 1 hour | On agent create |
| `GET /base_agents/tags` | HTTP + App | 1 hour | On agent update |

### Cache Keys

```ruby
# Application cache
"base_agents/list/#{page}/#{per_page}/#{filters_hash}"
"base_agents/#{id}/v#{updated_at.to_i}"
"base_agents/#{id}/preview/#{org_id}/#{project_id}/#{role}"
"base_agents/categories/#{BaseAgent.maximum(:updated_at).to_i}"

# ETags (based on updated_at)
"#{id}-#{updated_at.to_i}"
```

### Cache Invalidation

**On create:**
- Invalidate list cache
- Invalidate categories cache

**On update:**
- Invalidate single record cache
- Invalidate list cache
- Invalidate preview cache

**On publish/deprecate:**
- Invalidate all caches for this agent
- Invalidate effective config cache for all orgs using this agent

---

## Authorization Rules

### Platform Admin Endpoints

All `/api/v1/platform/*` endpoints require one of:
- `crewkit_owner`
- `crewkit_staff`
- `platform_admin`

```ruby
# In controller
before_action :require_platform_admin

def require_platform_admin
  unless current_user.platform_admin? || current_user.crewkit_owner? || current_user.crewkit_staff?
    render json: {
      error: "Insufficient permissions",
      code: "FORBIDDEN"
    }, status: :forbidden
  end
end
```

### Organization-Level Endpoints (Future)

Organization members can view published agents (read-only):

```ruby
# GET /api/v1/agents (existing endpoint)
# Returns published agents available to org
```

---

## Rswag Spec Outline

### File Structure

```
spec/
└── integration/
    └── api/
        └── v1/
            └── platform/
                ├── base_agents_spec.rb
                ├── base_agents_publish_spec.rb
                ├── base_agents_versions_spec.rb
                └── base_agents_analytics_spec.rb
```

### Example Spec

```ruby
# spec/integration/api/v1/platform/base_agents_spec.rb
require 'swagger_helper'

RSpec.describe 'API V1 Platform Base Agents', type: :request do
  path '/api/v1/platform/base_agents' do
    get 'List all base agents' do
      tags 'Platform Admin - Base Agents'
      produces 'application/json'
      security [bearer_auth: []]

      parameter name: :category, in: :query, type: :string, required: false,
                description: 'Filter by category'
      parameter name: :status, in: :query, type: :string, required: false,
                description: 'Filter by status (draft, published, deprecated)'
      parameter name: :page, in: :query, type: :integer, required: false,
                description: 'Page number (default: 1)'
      parameter name: :per_page, in: :query, type: :integer, required: false,
                description: 'Items per page (max 100, default: 25)'

      response '200', 'Success' do
        schema type: :object,
               properties: {
                 data: {
                   type: :array,
                   items: {
                     type: :object,
                     properties: {
                       id: { type: :integer },
                       name: { type: :string },
                       category: { type: :string },
                       status: { type: :string }
                     }
                   }
                 },
                 meta: {
                   type: :object,
                   properties: {
                     total: { type: :integer },
                     page: { type: :integer },
                     per_page: { type: :integer }
                   }
                 }
               }

        let(:Authorization) { "Bearer #{jwt_token}" }
        run_test! do |response|
          expect(response.headers['Cache-Control']).to eq('public, max-age=300')
          expect(JSON.parse(response.body)['data']).to be_an(Array)
        end
      end

      response '401', 'Unauthorized' do
        let(:Authorization) { 'Bearer invalid' }
        run_test!
      end

      response '403', 'Forbidden (not platform admin)' do
        let(:Authorization) { "Bearer #{org_user_jwt_token}" }
        run_test!
      end
    end

    post 'Create draft base agent' do
      tags 'Platform Admin - Base Agents'
      consumes 'application/json'
      produces 'application/json'
      security [bearer_auth: []]

      parameter name: :agent, in: :body, schema: {
        type: :object,
        properties: {
          name: { type: :string },
          category: { type: :string },
          description: { type: :string },
          content: { type: :string },
          tags: { type: :array, items: { type: :string } }
        },
        required: ['name', 'category', 'description', 'content']
      }

      response '201', 'Created' do
        let(:Authorization) { "Bearer #{jwt_token}" }
        let(:agent) do
          {
            name: 'test-agent',
            category: 'development',
            description: 'Test agent',
            content: '# Test Agent',
            tags: ['test']
          }
        end

        run_test! do |response|
          expect(JSON.parse(response.body)['data']['status']).to eq('draft')
        end
      end

      response '422', 'Validation Error' do
        let(:Authorization) { "Bearer #{jwt_token}" }
        let(:agent) { { name: '' } }
        run_test!
      end
    end
  end
end
```

### Swagger Configuration

```ruby
# spec/swagger_helper.rb
RSpec.configure do |config|
  config.swagger_root = Rails.root.join('swagger').to_s

  config.swagger_docs = {
    'v1/swagger.yaml' => {
      openapi: '3.0.1',
      info: {
        title: 'crewkit Platform API',
        version: 'v1',
        description: 'API for managing AI agent configurations'
      },
      servers: [
        {
          url: 'http://localhost:3050',
          description: 'Development'
        },
        {
          url: 'https://api.crewkit.io',
          description: 'Production'
        }
      ],
      components: {
        securitySchemes: {
          bearer_auth: {
            type: :http,
            scheme: :bearer,
            bearerFormat: 'JWT'
          }
        }
      }
    }
  }
end
```

---

## Performance Optimization Checklist

### Database Indexes

```ruby
# db/migrate/..._add_indexes_to_base_agents.rb
add_index :base_agents, :name, unique: true
add_index :base_agents, :status
add_index :base_agents, :category
add_index :base_agents, [:status, :published_at]
add_index :base_agents, :parent_version_id
add_index :base_agents, :created_by_id
add_index :base_agents, :tags, using: :gin # For array searches
```

### N+1 Query Prevention

```ruby
# In controller
def index
  @base_agents = BaseAgent
    .includes(:created_by, :published_by) # Eager load users
    .where(filters)
    .page(params[:page])
    .per(params[:per_page])
end

def show
  @base_agent = BaseAgent
    .includes(:created_by, :published_by)
    .find(params[:id])
end

def versions
  @versions = @base_agent
    .versions # self-referential association
    .includes(:published_by)
    .order(version_number: :desc)
end
```

### Query Optimization

```ruby
# Use counter caches
class BaseAgent < ApplicationRecord
  has_many :agent_sessions
  has_many :organizations, through: :agent_configurations

  # Update counts asynchronously
  after_commit :update_counters, on: [:create, :update]

  def update_counters
    UpdateBaseAgentCountersJob.perform_later(id)
  end
end

# Efficient filtering with scopes
scope :published, -> { where(status: :published) }
scope :by_category, ->(category) { where(category: category) if category.present? }
scope :search, ->(query) {
  where("name ILIKE ? OR description ILIKE ?", "%#{query}%", "%#{query}%") if query.present?
}
```

### Caching Examples

```ruby
# In controller
def index
  cache_key = "base_agents/list/#{cache_key_params}"

  @base_agents = Rails.cache.fetch(cache_key, expires_in: 5.minutes) do
    BaseAgent
      .includes(:created_by, :published_by)
      .where(filters)
      .page(params[:page])
      .to_a # Force query execution
  end
end

def show
  @base_agent = Rails.cache.fetch("base_agents/#{params[:id]}/#{@base_agent.updated_at.to_i}", expires_in: 5.minutes) do
    BaseAgent.includes(:created_by, :published_by).find(params[:id])
  end

  # ETag for conditional requests
  if stale?(@base_agent)
    render json: BaseAgentSerializer.new(@base_agent).serializable_hash
  end
end

private

def cache_key_params
  params.slice(:category, :status, :page, :per_page).to_json
end
```

---

## API Versioning Strategy

### Current: v1

All endpoints under `/api/v1/*`

### Future: v2 (Breaking Changes)

When breaking changes are needed:

1. **Create new namespace**: `/api/v2/platform/base_agents`
2. **Duplicate controllers**: `Api::V2::Platform::BaseAgentsController`
3. **Version-specific serializers**: `V2::BaseAgentSerializer`
4. **Maintain v1**: Keep v1 endpoints for 12 months minimum
5. **Deprecation headers**: Add `Deprecation: true` header to v1 responses

### Versioning Rules

**Non-breaking changes (can be added to v1):**
- New optional query parameters
- New response fields
- New endpoints
- Performance improvements

**Breaking changes (require v2):**
- Removing response fields
- Changing response structure
- Renaming endpoints
- Changing authentication method
- Removing endpoints

### Version Sunset Policy

1. **Announce deprecation**: 6 months before sunset
2. **Add deprecation headers**: `Sunset: Sat, 1 Jan 2026 00:00:00 GMT`
3. **Contact users**: Email notifications to CLI users
4. **Sunset date**: 12 months after v2 release
5. **Hard cutoff**: Return 410 Gone after sunset

---

## Rate Limiting

### Implementation

```ruby
# In controller
class Api::V1::Platform::BaseAgentsController < Api::V1::BaseController
  before_action :rate_limit_check

  def rate_limit_check
    limiter = Rack::Attack::RateLimit.new(request)

    unless limiter.allowed?
      render json: {
        error: "Rate limit exceeded",
        code: "RATE_LIMIT_EXCEEDED",
        details: {
          limit: limiter.limit,
          remaining: 0,
          reset_at: limiter.reset_time
        },
        message: "You have exceeded the rate limit. Please wait #{limiter.retry_after} seconds.",
        documentation_url: "https://docs.crewkit.io/api/rate-limits"
      }, status: :too_many_requests

      response.headers['X-RateLimit-Limit'] = limiter.limit.to_s
      response.headers['X-RateLimit-Remaining'] = '0'
      response.headers['X-RateLimit-Reset'] = limiter.reset_time.to_s
      response.headers['Retry-After'] = limiter.retry_after.to_s
    end
  end
end
```

### Rate Limit Headers (All Responses)

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1697462400
```

---

## Manual Testing (curl Commands)

After implementing the API, use these curl commands for validation:

```bash
# Set variables
API_URL="http://localhost:3050"
TOKEN="your_jwt_token_here"

# 1. List all base agents
curl -X GET "$API_URL/api/v1/platform/base_agents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json"

# 2. List with filtering
curl -X GET "$API_URL/api/v1/platform/base_agents?category=development&status=published&page=1&per_page=10" \
  -H "Authorization: Bearer $TOKEN"

# 3. Get single agent
curl -X GET "$API_URL/api/v1/platform/base_agents/1" \
  -H "Authorization: Bearer $TOKEN"

# 4. Create draft agent
curl -X POST "$API_URL/api/v1/platform/base_agents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-expert",
    "category": "development",
    "description": "Test agent for validation",
    "tags": ["test", "demo"],
    "content": "# Test Expert Agent\n\nYou are a test expert...",
    "frontmatter": {
      "name": "test-expert",
      "version": "1.0.0"
    }
  }'

# 5. Update draft agent
curl -X PATCH "$API_URL/api/v1/platform/base_agents/42" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated description",
    "content": "# Updated Test Expert Agent\n\nUpdated content..."
  }'

# 6. Publish agent
curl -X POST "$API_URL/api/v1/platform/base_agents/42/publish" \
  -H "Authorization: Bearer $TOKEN"

# 7. Preview effective config
curl -X GET "$API_URL/api/v1/platform/base_agents/42/preview?org_id=123&project_id=456&role=junior" \
  -H "Authorization: Bearer $TOKEN"

# 8. Get version history
curl -X GET "$API_URL/api/v1/platform/base_agents/42/versions" \
  -H "Authorization: Bearer $TOKEN"

# 9. Rollback to previous version
curl -X POST "$API_URL/api/v1/platform/base_agents/42/rollback" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"version_number": 2}'

# 10. Deprecate agent
curl -X POST "$API_URL/api/v1/platform/base_agents/42/deprecate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"reason": "Replaced by v2"}'

# 11. Get usage analytics
curl -X GET "$API_URL/api/v1/platform/base_agents/42/analytics?period=30d" \
  -H "Authorization: Bearer $TOKEN"

# 12. List categories
curl -X GET "$API_URL/api/v1/platform/base_agents/categories" \
  -H "Authorization: Bearer $TOKEN"

# 13. List tags (autocomplete)
curl -X GET "$API_URL/api/v1/platform/base_agents/tags?q=rai" \
  -H "Authorization: Bearer $TOKEN"

# 14. Bulk update
curl -X POST "$API_URL/api/v1/platform/base_agents/bulk_update" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "ids": [42, 43, 44],
    "updates": {
      "category": "backend",
      "tags": ["rails", "ruby"]
    }
  }'

# Test error handling
# 15. Unauthorized (missing token)
curl -X GET "$API_URL/api/v1/platform/base_agents"

# 16. Forbidden (org user token, not platform admin)
curl -X GET "$API_URL/api/v1/platform/base_agents" \
  -H "Authorization: Bearer $ORG_USER_TOKEN"

# 17. Not found
curl -X GET "$API_URL/api/v1/platform/base_agents/99999" \
  -H "Authorization: Bearer $TOKEN"

# 18. Validation error (invalid name)
curl -X POST "$API_URL/api/v1/platform/base_agents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Invalid Name With Spaces",
    "category": "development"
  }'

# 19. Conflict (duplicate name)
curl -X POST "$API_URL/api/v1/platform/base_agents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rails-expert",
    "category": "development",
    "description": "Duplicate",
    "content": "# Duplicate"
  }'

# Test caching
# 20. Check ETag behavior
curl -X GET "$API_URL/api/v1/platform/base_agents/1" \
  -H "Authorization: Bearer $TOKEN" \
  -D - # Show response headers

# 21. Conditional GET (should return 304 if not modified)
ETAG="a1b2c3d4e5f6" # Replace with actual ETag from previous response
curl -X GET "$API_URL/api/v1/platform/base_agents/1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "If-None-Match: \"$ETAG\"" \
  -D -

# Performance testing
# 22. Measure response time
curl -X GET "$API_URL/api/v1/platform/base_agents" \
  -H "Authorization: Bearer $TOKEN" \
  -w "\nTime: %{time_total}s\n" \
  -o /dev/null -s
```

### Expected Response Times

```
GET  /base_agents           < 100ms
GET  /base_agents/:id       < 50ms
POST /base_agents           < 200ms
PATCH /base_agents/:id      < 100ms
POST /base_agents/:id/publish < 500ms
GET  /base_agents/:id/preview < 100ms
GET  /base_agents/:id/versions < 50ms
GET  /base_agents/:id/analytics < 200ms
GET  /base_agents/categories < 20ms
GET  /base_agents/tags      < 50ms
```

---

## Implementation Checklist

Before marking the API complete, verify:

- [ ] All endpoints respond with correct HTTP status codes
- [ ] Response times meet performance targets (< 100ms for GETs)
- [ ] No N+1 queries (verified with bullet gem)
- [ ] Database indexes on all queried columns
- [ ] Caching implemented with proper TTL
- [ ] Pagination on list endpoints (default: 25/page, max: 100)
- [ ] Eager loading for associations (`.includes()`)
- [ ] Error responses follow standardized format
- [ ] curl commands provided for manual testing (above)
- [ ] Rswag integration tests added
- [ ] Performance tracked in logs (duration, SQL queries)
- [ ] Authorization checks on all endpoints
- [ ] Rate limiting configured
- [ ] ETag support for cacheable endpoints
- [ ] API documentation auto-generated via Rswag

---

## Next Steps for Implementation

1. **rails-expert agent**: Implement controllers, services, models
2. **code-reviewer agent**: Write RSpec tests + validate with curl
3. **frontend-expert agent**: Build UI (after API is tested and working)

**This API specification is ready for implementation.**
