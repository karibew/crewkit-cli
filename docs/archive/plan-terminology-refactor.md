# Terminology & Navigation Refactor - COMPLETED

## Summary

Refactored the dashboard to use consistent terminology and project-centric navigation. All sessions, conversations, and threads now live under their respective projects.

## Terminology

| Internal Model | User-Facing Term | Description |
|---------------|------------------|-------------|
| `CrewkitSession` | **Run** | CLI process lifetime |
| `LlmSession` | **Conversation** | Single Claude Code chat |
| `SubagentSession` | **Task** | Delegated agent work |
| `conversation_thread_id` | **Thread** | Related conversations (resume chain) |

## URL Structure

All session-related data is now project-scoped:

```
/kit/projects                                     # List all projects
/kit/projects/[id]                                # Project overview
/kit/projects/[id]/conversations                  # Project's conversations
/kit/projects/[id]/conversations/[conversationId] # Single conversation detail
/kit/projects/[id]/threads                        # Project's threads
/kit/projects/[id]/threads/[threadId]             # Thread detail
/kit/projects/[id]/settings                       # Project settings
```

## Sidebar Navigation

Cleaned up sidebar - removed confusing duplicate entries:

```
Dashboard
Projects        ← Primary entry point for conversations/threads
Agents
Playbooks
Experiments
Analytics       ← Cross-project reporting
API Docs
Team
Integrations
Settings
```

**Removed:**
- ~~Sessions~~ (moved to project-level)
- ~~Conversations~~ (moved to project-level)
- ~~Resources~~ (merged with Agents)

## Project Detail Quick Actions

From the project detail page, users can access:

1. **Conversations** → `/kit/projects/[id]/conversations`
2. **Threads** → `/kit/projects/[id]/threads`
3. **Agents** → `/kit/agents?project=[id]`
4. **Playbooks** → `/kit/projects/[id]/settings`
5. **Experiments** → `/kit/experiments?project=[id]`

## Files Created

1. `dashboard/src/lib/display-names.ts` - Terminology constants
2. `dashboard/src/app/kit/projects/[id]/conversations/page.tsx` - Conversations list
3. `dashboard/src/app/kit/projects/[id]/conversations/[conversationId]/page.tsx` - Conversation detail
4. `dashboard/src/app/kit/projects/[id]/threads/page.tsx` - Threads list
5. `dashboard/src/app/kit/projects/[id]/threads/[threadId]/page.tsx` - Thread detail

## Files Removed

1. `dashboard/src/app/kit/sessions/` - Entire directory
2. `dashboard/src/app/kit/conversations/` - Entire directory (except for existing thread pages that were moved)

## Files Modified

### Navigation
- `dashboard/src/components/layouts/sidebar.tsx` - Removed Sessions, Conversations, Resources

### Components (project-scoped URLs)
- `dashboard/src/components/features/sessions/llm-sessions-table.tsx` - Uses `sessionLinkPrefix` prop
- `dashboard/src/components/features/sessions/conversation-thread-table.tsx` - Uses `getThreadHref` prop
- `dashboard/src/components/features/sessions/timeline/thread-timeline.tsx` - Project-scoped links
- `dashboard/src/components/features/sessions/session-table.tsx` - Project-scoped links
- `dashboard/src/components/features/sessions/session-error-state.tsx` - Links to /kit/projects
- `dashboard/src/components/features/dashboard/recent-sessions.tsx` - Project-scoped links
- `dashboard/src/components/features/analytics/live-activity-feed.tsx` - Project-scoped links
- `dashboard/src/components/features/projects/project-recent-sessions.tsx` - Project-scoped links

### Command Palette
- `dashboard/src/lib/command-palette/actions.ts` - Removed session-related quick actions
- `dashboard/src/lib/command-palette/search-index.ts` - Session search uses project-scoped URLs

### Terminology Updates
- `dashboard/src/components/features/sessions/hierarchy/agent-task-card.tsx` - "Task" instead of "Agent Task"
- `dashboard/src/components/features/sessions/hierarchy/agent-task-list.tsx` - "Tasks" instead of "Agent Tasks"
- `dashboard/src/components/features/sessions/timeline/items/subagent-item.tsx` - "Task" instead of "Subagent"

### API (session includes project info)
- `api/app/controllers/api/v1/sessions_controller.rb` - Added `project` to response
- `dashboard/src/types/api.ts` - Added `project` to Session interface

## Key Principle

**A conversation cannot exist without a project.** This is enforced at both:
- Database level: `CrewkitSession.project_id` is NOT NULL
- UI level: All conversation/thread URLs require a project ID

## Migration Notes

- Old URLs (`/kit/sessions/*`, `/kit/conversations/*`) no longer work
- All links now include project context
- Sessions without `project.id` in API responses cannot be linked to (skipped in search)
