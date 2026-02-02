# Plan: LlmSession-Centric Display

## Problem Statement

Currently, crewkit displays sessions based on **CrewkitSession** (CLI process lifetime). However, users think in terms of **conversations** (LlmSessions), which:

1. Can span multiple CLI invocations (via `--continue` resume)
2. Can have multiple sessions within one CLI run (via `/clear new`)
3. Can be compacted multiple times (autocompact)

The relationship is **many-to-many**, not hierarchical.

## Key Events to Track

| Event | Current Tracking | Needed |
|-------|------------------|--------|
| Session start | ✅ `SessionStart` hook | Add `source` field (startup/resume/clear) |
| Session end | ⚠️ Partial | Add `close_reason` (clear_new/exit/timeout/error) |
| Session resume | ⚠️ `is_resumed` flag only | Track `conversation_thread_id` |
| Session compact | ❌ Not tracked | Add `compaction_count`, optional history |

## Data Model Changes

### LlmSession Table Additions

```sql
ALTER TABLE llm_sessions ADD COLUMN conversation_thread_id uuid;
ALTER TABLE llm_sessions ADD COLUMN close_reason varchar;
ALTER TABLE llm_sessions ADD COLUMN closed_at timestamp;
ALTER TABLE llm_sessions ADD COLUMN compaction_count integer DEFAULT 0;
ALTER TABLE llm_sessions ADD COLUMN last_compacted_at timestamp;

-- Denormalized for direct queries (avoid joins to crewkit_session)
ALTER TABLE llm_sessions ADD COLUMN project_id bigint REFERENCES projects(id);
ALTER TABLE llm_sessions ADD COLUMN user_id bigint REFERENCES users(id);
ALTER TABLE llm_sessions ADD COLUMN agent_name varchar;

-- Indexes
CREATE INDEX idx_llm_sessions_thread ON llm_sessions(organization_id, conversation_thread_id);
CREATE INDEX idx_llm_sessions_started ON llm_sessions(organization_id, started_at DESC);
CREATE INDEX idx_llm_sessions_project ON llm_sessions(organization_id, project_id, started_at DESC);
```

### Optional: Compaction History Table

If we need detailed compaction tracking (when, what was summarized):

```sql
CREATE TABLE llm_session_compactions (
  id bigserial PRIMARY KEY,
  llm_session_id bigint NOT NULL REFERENCES llm_sessions(id) ON DELETE CASCADE,
  compacted_at timestamp NOT NULL DEFAULT NOW(),
  tokens_before integer,
  tokens_after integer,
  summary_length integer,
  created_at timestamp NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_compactions_session ON llm_session_compactions(llm_session_id);
```

### Close Reasons Enum

```ruby
enum :close_reason, {
  clear_new: 'clear_new',    # /clear new command
  exit: 'exit',              # User exit or CLI close
  timeout: 'timeout',        # Inactivity timeout
  error: 'error',            # Session errored
  logout: 'logout',          # Explicit logout
  compact: 'compact'         # Session ended due to compaction (if applicable)
}
```

## Conversation Thread Logic

### How `conversation_thread_id` Works

```
Scenario 1: Normal session
  SessionStart (source=startup)
  → Generate new conversation_thread_id (UUID)

Scenario 2: Resume via --continue
  SessionStart (source=resume, resumed_from=<previous_session_id>)
  → Copy conversation_thread_id from resumed_from session

Scenario 3: /clear new within same CLI
  SessionEnd (reason=clear_new) on current session
  → Close current LlmSession
  SessionStart (source=clear)
  → Generate NEW conversation_thread_id (new conversation)
  → Same CrewkitSession, different thread

Scenario 4: Compaction
  CompactionEvent (session_id, tokens_before, tokens_after)
  → Increment compaction_count
  → Update last_compacted_at
  → Optionally create llm_session_compactions record
```

### Thread Queries

```ruby
# Get all sessions in a conversation thread
LlmSession.where(conversation_thread_id: thread_id).order(:started_at)

# Get conversation threads for a project
LlmSession.where(project_id: project.id)
          .distinct
          .pluck(:conversation_thread_id)

# Get thread with most compactions (potential quality issues)
LlmSession.where(organization_id: org.id)
          .where('compaction_count > 0')
          .order(compaction_count: :desc)
```

## API Changes

### New Endpoints

```
GET  /api/v1/:org_id/conversations
     List LlmSessions grouped by conversation_thread_id

GET  /api/v1/:org_id/conversations/:thread_id
     All sessions in a conversation thread

GET  /api/v1/:org_id/llm_sessions
     Flat list of LlmSessions (primary list view)

GET  /api/v1/:org_id/llm_sessions/:id
     Single LlmSession detail

GET  /api/v1/:org_id/llm_sessions/:id/compactions
     Compaction history for a session
```

### Response Shape

```json
{
  "data": {
    "id": "abc123",
    "otel_session_id": "17dbaead-76c0-4540-a990-ea7307317561",
    "conversation_thread_id": "550e8400-e29b-41d4-a716-446655440000",
    "agent_name": "rails-expert",
    "is_resumed": false,
    "close_reason": "exit",
    "compaction_count": 2,
    "last_compacted_at": "2026-01-16T10:30:00Z",
    "started_at": "2026-01-16T09:00:00Z",
    "closed_at": "2026-01-16T11:00:00Z",
    "metrics": {
      "input_tokens": 45000,
      "output_tokens": 12000,
      "estimated_cost_usd": 0.45,
      "message_count": 24,
      "tool_call_count": 87
    },
    "thread_info": {
      "position": 1,
      "total_in_thread": 3,
      "total_compactions_in_thread": 5
    },
    "project": { "id": "proj_123", "name": "crewkit" },
    "user": { "id": "user_456", "name": "Felix" }
  }
}
```

## CLI Changes

### Hook Event Updates

**SessionStart** - add fields:
```json
{
  "event_type": "SessionStart",
  "session_id": "17dbaead...",
  "source": "startup|resume|clear",
  "resumed_from_session_id": "previous-id-if-resume",
  "conversation_thread_id": "uuid-from-cli-state"
}
```

**SessionEnd** - ensure we capture:
```json
{
  "event_type": "SessionEnd",
  "session_id": "17dbaead...",
  "reason": "clear|logout|prompt_input_exit|other"
}
```

**Compaction** - new or enhanced event:
```json
{
  "event_type": "Compaction",
  "session_id": "17dbaead...",
  "tokens_before": 128000,
  "tokens_after": 45000
}
```

### CLI State Management

```rust
struct SessionState {
    crewkit_session_id: String,
    current_llm_session_id: Option<String>,
    conversation_thread_id: Option<String>,
}

// On SessionStart:
// - If source=startup: generate new conversation_thread_id
// - If source=resume: preserve conversation_thread_id from resumed session
// - If source=clear: generate NEW conversation_thread_id (new conversation)

// On Compaction:
// - Send compaction event with token counts
```

## Dashboard Changes

### Sessions List Page

```
Tabs: [Conversations] [CLI Sessions] [By Project] [By User]

Conversations tab (default):
┌─────────────────────────────────────────────────────────────────┐
│ Conversation          Agent         User    Cost    Status      │
├─────────────────────────────────────────────────────────────────┤
│ 📝 HttpOnly Migration  rails-expert  Felix   $0.45   Completed  │
│   └─ 3 sessions, 2 compactions                                  │
│ 📝 Command Palette     frontend      Felix   $1.23   Active     │
│   └─ 5 sessions, 4 compactions                                  │
│ 📝 Auth Bug Fix        rails-expert  Sarah   $0.12   Completed  │
│   └─ 1 session, 0 compactions                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Session Detail

```
┌─ Conversation: HttpOnly Migration ──────────────────────────────┐
│                                                                 │
│ Thread: 3 sessions | 2 compactions | $0.45 total               │
│                                                                 │
│ Timeline:                                                       │
│ ┌───────────────────────────────────────────────────────────┐  │
│ │ ● Session 1 (9:00 AM)                                      │  │
│ │   └─ 45 messages, 87 tool calls                           │  │
│ │   └─ ⚡ Compacted at 9:45 AM (128k → 45k tokens)          │  │
│ │                                                            │  │
│ │ ○ /clear new (10:00 AM) ← Thread break                    │  │
│ │                                                            │  │
│ │ ● Session 2 (10:01 AM) - NEW THREAD                       │  │
│ │   └─ Continued in different conversation                   │  │
│ └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│ [Overview] [Timeline] [Insights] [Subagents] [Usage]           │
└─────────────────────────────────────────────────────────────────┘
```

## Implementation Phases

### Phase 1: Data Model (Week 1)
- [ ] Migration: Add columns to llm_sessions
- [ ] Migration: Create llm_session_compactions table
- [ ] Backfill: Generate conversation_thread_id for existing sessions
- [ ] Backfill: Denormalize project_id, user_id, agent_name
- [ ] Model: Add scopes, enums, helpers to LlmSession

### Phase 2: CLI Tracking (Week 1-2)
- [ ] Update SessionStart handler to track source
- [ ] Add SessionEnd handling with close_reason
- [ ] Track Compaction events
- [ ] Manage conversation_thread_id in CLI state
- [ ] Preserve thread across resume, reset on /clear new

### Phase 3: API Endpoints (Week 2)
- [ ] GET /conversations (grouped view)
- [ ] GET /llm_sessions (flat list)
- [ ] GET /llm_sessions/:id (detail)
- [ ] GET /llm_sessions/:id/compactions (history)
- [ ] Update analytics for LlmSession aggregation

### Phase 4: Dashboard (Week 2-3)
- [ ] New LlmSessionTable component
- [ ] Conversation thread grouping UI
- [ ] Compaction indicators
- [ ] Session detail with thread timeline
- [ ] Feature flag for rollout

### Phase 5: Rollout (Week 3-4)
- [ ] Internal testing
- [ ] Feature flag enabled for beta users
- [ ] Monitor performance and accuracy
- [ ] Full rollout with old view toggle
- [ ] Deprecate CrewkitSession-first view

## Open Questions

1. **Compaction detail level**: Do we need full compaction history, or is count + last_compacted_at enough?
   - Recommendation: Start with count + timestamp, add history table if needed

2. **Thread breaks**: When /clear new is used, should we show it as:
   - Same "conversation" with a break indicator?
   - Completely separate conversations?
   - Recommendation: Separate conversations (new thread_id)

3. **Analytics granularity**: Should analytics aggregate by:
   - LlmSession (each conversation segment)?
   - Conversation thread (full logical conversation)?
   - Recommendation: Support both via query param

4. **URL structure**:
   - `/kit/sessions/[id]` → LlmSession detail?
   - `/kit/conversations/[thread_id]` → Thread view?
   - Recommendation: Keep /sessions for LlmSession, add /conversations for threads

## Files to Modify

### API
- `api/db/migrate/` - New migrations
- `api/app/models/llm_session.rb` - New fields, scopes, enums
- `api/app/models/llm_session_compaction.rb` - New model (optional)
- `api/app/controllers/api/v1/llm_sessions_controller.rb` - New controller
- `api/app/controllers/api/v1/conversations_controller.rb` - New controller
- `api/app/services/hooks_event_ingestion_service.rb` - Lifecycle handling
- `api/config/routes.rb` - New routes

### CLI
- `cli/src/services/subagent_tracker.rs` - Event handling
- `cli/src/services/event_buffer.rs` - Thread state
- `cli/src/hooks/types.rs` - Event types

### Dashboard
- `dashboard/src/app/kit/sessions/page.tsx` - List view
- `dashboard/src/components/features/sessions/llm-session-table.tsx` - New component
- `dashboard/src/components/features/sessions/conversation-thread.tsx` - New component
- `dashboard/src/hooks/use-llm-sessions.ts` - New hooks
- `dashboard/src/lib/api/llm-sessions.ts` - New API client
- `dashboard/src/types/sessions.ts` - New types
