---
doc_tier: 2
doc_type: spec
doc_status: active
created: 2025-10-01
last_reviewed: 2026-01-06
owner: platform-team
related_docs:
  - CLAUDE.md
tags: [api, documentation, endpoints]
---

# Crewkit API Documentation

Base URL: `https://api.crewkit.io/api/v1` (production) or `http://localhost:3000/api/v1` (development)

## Resource IDs

All API responses use **UUID v4** for resource IDs (e.g., `432e16a3-0bf4-4117-8ee0-9a3b21ebe07b`). This prevents exposing sequential information about your data and provides better security than integer IDs.

Example:
```json
{
  "session": {
    "id": "432e16a3-0bf4-4117-8ee0-9a3b21ebe07b",
    ...
  }
}
```

## Authentication

All API requests (except auth endpoints) require a Bearer token in the Authorization header:

```
Authorization: Bearer <access_token>
```

### Device Flow (OAuth 2.1)

#### 1. Initiate Device Flow

```http
POST /api/v1/auth/device
```

**Response:**
```json
{
  "device_code": "abc123...",
  "user_code": "ABCD-EFGH",
  "verification_uri": "https://crewkit.io/auth/verify",
  "expires_in": 600,
  "interval": 5
}
```

#### 2. Exchange Device Code for Token

```http
POST /api/v1/auth/token
Content-Type: application/json

{
  "device_code": "abc123..."
}
```

**Response:**
```json
{
  "access_token": "eyJhbGci...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "refresh123..."
}
```

---

## Agents

### List Available Agents

```http
GET /api/v1/agents?project_slug=customer-portal
```

**Query Parameters:**
- `project_slug` (optional): Filter agents for specific project

**Response:**
```json
{
  "agents": [
    {
      "name": "rails-expert",
      "source": "base"
    },
    {
      "name": "frontend-expert",
      "source": "organization"
    },
    {
      "name": "custom-agent",
      "source": "project"
    }
  ]
}
```

### Get Effective Agent Configuration

Returns merged configuration (base + org + project + role modifiers).

```http
GET /api/v1/agents/rails-expert/effective_config?project_slug=customer-portal
```

**Query Parameters:**
- `project_slug` (optional): Include project-level overrides

**Response:**
```json
{
  "agent_name": "rails-expert",
  "config": {
    "version": "1.0",
    "instructions": "You are an expert Rails developer...",
    "tools": [],
    "prompt_prefix": "CRITICAL: You are in COACHING MODE...",
    "coaching_mode": true,
    "collaborative_mode": false,
    "autonomous_mode": false
  },
  "sources": {
    "base": true,
    "organization": true,
    "project": false
  }
}
```

### Log Agent Modification

```http
POST /api/v1/agents/rails-expert/modifications
Content-Type: application/json

{
  "session_id": 123,
  "file_path": ".claude/agents/rails-expert.md",
  "change_type": "edit",
  "content_before": "...",
  "content_after": "..."
}
```

**Response:**
```json
{
  "modification": {
    "id": 456,
    "agent_name": "rails-expert",
    "file_path": ".claude/agents/rails-expert.md",
    "change_type": "edit",
    "created_at": "2025-10-11T14:30:00Z"
  }
}
```

---

## Experiments

### Create Experiment

```http
POST /api/v1/experiments
Content-Type: application/json

{
  "agent_name": "rails-expert",
  "project_slug": "customer-portal",
  "variant_config": {
    "instructions": "Modified instructions for testing...",
    "tools": ["new-tool"]
  }
}
```

**Response:**
```json
{
  "experiment": {
    "id": 1,
    "slug": "swift-amber-falcon",
    "agent_name": "rails-expert",
    "status": "active",
    "control_config": { ... },
    "variant_config": { ... },
    "created_at": "2025-10-11T14:00:00Z",
    "updated_at": "2025-10-11T14:00:00Z"
  }
}
```

### Get Experiment Details

```http
GET /api/v1/experiments/swift-amber-falcon
```

**Response:**
```json
{
  "experiment": {
    "id": 1,
    "slug": "swift-amber-falcon",
    "agent_name": "rails-expert",
    "status": "active",
    "control_config": { ... },
    "variant_config": { ... },
    "created_at": "2025-10-11T14:00:00Z",
    "updated_at": "2025-10-11T14:00:00Z"
  }
}
```

### Get Experiment Metrics

```http
GET /api/v1/experiments/swift-amber-falcon/metrics
```

**Response:**
```json
{
  "experiment": { ... },
  "metrics": {
    "control": {
      "total_sessions": 100,
      "success_rate": 85.5,
      "avg_cost": 0.0234,
      "avg_turns": 12.5,
      "p95_cost": 0.0567
    },
    "variant": {
      "total_sessions": 98,
      "success_rate": 89.2,
      "avg_cost": 0.0198,
      "avg_turns": 10.8,
      "p95_cost": 0.0489
    },
    "significance": {
      "p_value": 0.05,
      "significant": false,
      "confidence": 0.95
    }
  }
}
```

### Deploy Experiment

Promotes variant configuration to project level.

```http
POST /api/v1/experiments/swift-amber-falcon/deploy
```

**Response:**
```json
{
  "experiment": {
    "id": 1,
    "slug": "swift-amber-falcon",
    "status": "deployed",
    ...
  },
  "message": "Experiment deployed successfully"
}
```

---

## Sessions

### Create Session

```http
POST /api/v1/sessions
Content-Type: application/json

{
  "agent_name": "rails-expert",
  "project_slug": "customer-portal"
}
```

**Response:**
```json
{
  "session": {
    "id": 123,
    "agent_name": "rails-expert",
    "agent_version": "base-1.0_org-1234567890_proj-1234567899",
    "total_turns": 0,
    "total_cost": 0.0,
    "outcome": null,
    "started_at": "2025-10-11T14:00:00Z",
    "ended_at": null,
    "user": {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

### Update Session

```http
PATCH /api/v1/sessions/123
Content-Type: application/json

{
  "total_turns": 15,
  "total_cost": 0.0456,
  "outcome": "success",
  "ended_at": "2025-10-11T14:30:00Z"
}
```

**Response:**
```json
{
  "session": {
    "id": 123,
    "agent_name": "rails-expert",
    "agent_version": "base-1.0_org-1234567890_proj-1234567899",
    "total_turns": 15,
    "total_cost": 0.0456,
    "outcome": "success",
    "started_at": "2025-10-11T14:00:00Z",
    "ended_at": "2025-10-11T14:30:00Z",
    "user": { ... }
  }
}
```

### Get Session Summary

```http
GET /api/v1/sessions/123/summary
```

**Response:**
```json
{
  "session": { ... },
  "modifications": [
    {
      "id": 456,
      "agent_name": "rails-expert",
      "file_path": ".claude/agents/rails-expert.md",
      "change_type": "edit",
      "created_at": "2025-10-11T14:15:00Z"
    }
  ],
  "duration": 1800,
  "cost_per_turn": 0.00304
}
```

### List Sessions

```http
GET /api/v1/sessions?project_slug=customer-portal&agent_name=rails-expert&page=1&per_page=25
```

**Query Parameters:**
- `project_slug` (optional): Filter by project
- `agent_name` (optional): Filter by agent
- `current_user_only` (optional): Only show current user's sessions
- `page` (optional): Page number (default: 1)
- `per_page` (optional): Items per page (default: 25)

**Response:**
```json
{
  "sessions": [
    {
      "id": 123,
      "agent_name": "rails-expert",
      "agent_version": "base-1.0_org-1234567890_proj-1234567899",
      "total_turns": 15,
      "total_cost": 0.0456,
      "outcome": "success",
      "started_at": "2025-10-11T14:00:00Z",
      "ended_at": "2025-10-11T14:30:00Z",
      "user": { ... }
    }
  ],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_count": 123
  }
}
```

---

## Error Responses

All error responses follow this format:

```json
{
  "error": "Error message"
}
```

### HTTP Status Codes

- `200 OK` - Request succeeded
- `201 Created` - Resource created successfully
- `400 Bad Request` - Missing or invalid parameters
- `401 Unauthorized` - Missing or invalid authentication token
- `404 Not Found` - Resource not found
- `422 Unprocessable Entity` - Validation error
- `500 Internal Server Error` - Server error

### Example Validation Error

```json
{
  "error": "Validation failed: Name can't be blank",
  "details": {
    "name": ["can't be blank"]
  }
}
```

---

## Rate Limiting

- 1000 requests per hour per organization
- 100 requests per minute per IP address

Rate limit headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1697040000
```

---

## Webhooks (Future)

Coming soon: Webhooks for experiment completion, session events, and pattern detection.
