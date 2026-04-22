# PRD: Notification System

**Author**: Engineering Team
**Status**: Ready for Development
**Priority**: High
**Target**: Q1 2026

## Problem Statement

crewkit users have no way to learn about important events without actively checking the dashboard. Managers don't know when cloud jobs complete or fail, developers miss task assignments from blueprints, and team leads can't track when experiments reach statistical significance. This creates a reactive workflow where users discover problems hours or days after they occur.

## Goals

1. Deliver timely, actionable notifications for key crewkit events
2. Support multiple delivery channels (in-app, email, Slack webhook)
3. Allow per-user preference control over what they receive and how
4. Provide organization-level defaults that admins can configure
5. Keep notification volume manageable — batch low-priority items, never spam

## Non-Goals

- Push notifications (mobile app doesn't exist yet)
- SMS delivery (cost prohibitive at current scale)
- Real-time WebSocket delivery (polling + SSE sufficient for v1)

## User Stories

### Manager
- As a manager, I want to be notified when a cloud job fails so I can investigate or reassign the task
- As a manager, I want a daily digest of blueprint progress so I can track team velocity without checking each blueprint
- As a manager, I want to know when an experiment reaches significance so I can make deployment decisions promptly

### Developer
- As a developer, I want to be notified when I'm assigned a blueprint task so I can plan my work
- As a developer, I want to know when a blocking task is completed so I can start my dependent work
- As a developer, I want to see a notification when a resource I maintain gets a new version published

### Admin
- As an admin, I want to configure organization-wide notification defaults
- As an admin, I want to be alerted on security events (failed auth attempts, new member joins)
- As an admin, I want to control which notification channels are enabled for the organization

## Event Catalog

### High Priority (real-time delivery)
| Event | Default Channel | Recipients |
|-------|----------------|------------|
| `cloud_job.failed` | in-app + email | Job owner + blueprint task assignee |
| `cloud_job.completed` | in-app | Job owner + blueprint task assignee |
| `security.suspicious_login` | in-app + email | All admins |
| `blueprint_task.assigned` | in-app + email | Assigned user |
| `blueprint_task.blocked` | in-app | Task assignee + blueprint owner |
| `experiment.significant` | in-app + email | Experiment creator + org admins |

### Medium Priority (batched, max 1 per hour)
| Event | Default Channel | Recipients |
|-------|----------------|------------|
| `blueprint_task.completed` | in-app | Blueprint owner + epic assignees |
| `blueprint_task.dependency_resolved` | in-app | Dependent task assignees |
| `resource.version_published` | in-app | Resource maintainers |
| `member.joined` | in-app | All admins |
| `member.role_changed` | in-app | Affected user + all admins |

### Low Priority (daily digest only)
| Event | Default Channel | Recipients |
|-------|----------------|------------|
| `blueprint.progress_update` | email digest | Blueprint owner |
| `session.analysis_complete` | in-app | Session owner |
| `resource.stats_available` | in-app | Resource maintainers |
| `subscription.usage_warning` | email | All admins |

## Data Model

### Notification
Core record — one per notification per recipient.

| Field | Type | Notes |
|-------|------|-------|
| id | bigint | PK |
| external_id | uuid | Public identifier |
| organization_id | bigint | FK, tenant isolation |
| user_id | bigint | FK, the recipient |
| event_type | string | e.g. "cloud_job.failed" |
| title | string | Human-readable title |
| body | text | Markdown content |
| priority | string | high / medium / low |
| channel | string | in_app / email / slack |
| status | string | pending / delivered / read / dismissed |
| source_type | string | Polymorphic: "CloudJob", "BlueprintTask", etc. |
| source_id | bigint | FK to the source record |
| metadata | jsonb | Extra context (links, action URLs) |
| read_at | datetime | When user marked as read |
| delivered_at | datetime | When sent via channel |
| created_at | datetime | |

### NotificationPreference
Per-user override of organization defaults.

| Field | Type | Notes |
|-------|------|-------|
| id | bigint | PK |
| organization_id | bigint | FK |
| user_id | bigint | FK |
| event_type | string | e.g. "cloud_job.failed" or "*" for all |
| channel | string | in_app / email / slack |
| enabled | boolean | Override: true = force on, false = force off |
| created_at | datetime | |

Unique index on (organization_id, user_id, event_type, channel).

### OrganizationNotificationSetting
Org-level defaults and channel configuration.

| Field | Type | Notes |
|-------|------|-------|
| id | bigint | PK |
| organization_id | bigint | FK, unique |
| email_enabled | boolean | default true |
| slack_enabled | boolean | default false |
| slack_webhook_url | string | Encrypted, validated URL format |
| digest_enabled | boolean | default true |
| digest_frequency | string | daily / weekly |
| digest_time | string | "09:00" UTC |
| quiet_hours_start | string | "22:00" UTC (optional) |
| quiet_hours_end | string | "08:00" UTC (optional) |
| metadata | jsonb | Future expansion |

## API Endpoints

### Notifications (user-facing)
```
GET    /:org_id/notifications                    # List (paginated, filterable by status/type/priority)
GET    /:org_id/notifications/unread_count        # Badge count for UI
PATCH  /:org_id/notifications/:id/read            # Mark as read
POST   /:org_id/notifications/read_all            # Mark all as read
DELETE /:org_id/notifications/:id                 # Dismiss
```

### Notification Preferences (user-facing)
```
GET    /:org_id/notification_preferences          # List user's preferences
PUT    /:org_id/notification_preferences          # Bulk update preferences
```

### Organization Notification Settings (admin-only)
```
GET    /:org_id/notification_settings             # Show org settings
PATCH  /:org_id/notification_settings             # Update org settings
POST   /:org_id/notification_settings/test_slack  # Send test Slack message
```

## Services

### NotificationService
Orchestrates notification creation and delivery. Entry point for all event handlers.

```ruby
NotificationService.notify(
  organization: org,
  event_type: "cloud_job.failed",
  source: cloud_job,
  recipients: [user1, user2],
  title: "Cloud job failed: #{cloud_job.external_id}",
  body: "Job failed with error: #{cloud_job.error_message}",
  priority: "high",
  metadata: { cloud_job_id: cloud_job.external_id, blueprint_task_id: task&.external_id }
)
```

Responsibilities:
- Resolve effective channels per recipient (org defaults + user overrides)
- Respect quiet hours (queue for later delivery)
- Create Notification records
- Enqueue delivery jobs per channel

### NotificationDeliveryService
Handles actual delivery per channel.

- **In-app**: Just creates the record (dashboard polls or uses SSE)
- **Email**: Enqueues `NotificationEmailJob` via ActionMailer
- **Slack**: Posts to webhook URL via `SlackWebhookService`

### NotificationDigestService
Aggregates low-priority notifications into digest emails.

- Runs on schedule (daily at org's configured time)
- Groups notifications by type and source
- Generates digest email with summary + links
- Marks individual notifications as "delivered via digest"

### SlackWebhookService
Formats and posts messages to Slack.

- Validates webhook URL format (must be hooks.slack.com)
- Formats as Slack Block Kit message
- Handles rate limiting (1 req/sec per webhook)
- Retries with exponential backoff (3 attempts)
- Records delivery status

## Dashboard UI

### Notification Bell (header)
- Bell icon in top nav with unread count badge
- Click opens dropdown panel showing recent notifications
- Each notification: icon (by type), title, relative time, read/unread indicator
- "Mark all as read" and "View all" links
- Poll for updates every 30 seconds (or SSE if available)

### Notification Center Page (/kit/notifications)
- Full list view with filters: All / Unread / By Type / By Priority
- Bulk actions: Mark as read, Dismiss
- Click notification navigates to source (cloud job, blueprint task, etc.)
- Empty state: "You're all caught up"

### Notification Preferences Page (/kit/settings/notifications)
- Table of event types with toggle switches per channel
- Group by category (Cloud Jobs, Blueprints, Resources, Security, Billing)
- "Reset to organization defaults" button
- Quiet hours configuration (personal override)

### Admin: Org Notification Settings (/kit/settings/organization/notifications)
- Channel toggles (email, Slack)
- Slack webhook URL configuration with "Test" button
- Digest frequency and time
- Quiet hours (org-wide default)
- Per-event-type default channel matrix

## Background Jobs

| Job | Schedule | Purpose |
|-----|----------|---------|
| `NotificationEmailJob` | On-demand | Send individual email notification |
| `NotificationSlackJob` | On-demand | Post to Slack webhook |
| `NotificationDigestJob` | Daily (configurable) | Compile and send digest emails |
| `NotificationCleanupJob` | Weekly | Delete dismissed notifications older than 90 days |
| `NotificationBatchJob` | Every 15 minutes | Deliver batched medium-priority notifications |

## Integration Points

### Event Publishers (where notifications get triggered)

These existing services need to publish notification events:

1. **CloudJobService** / **Orchestrator Controller** — on job completion/failure
2. **BlueprintService** — on task assignment, status transitions, dependency resolution
3. **BlueprintExecutionService** — on execution start (confirm to owner)
4. **MemberManagementService** — on member join, role change, removal
5. **SecurityEventService** — on suspicious login, failed auth patterns
6. **VersionBenchmarkService** — on experiment reaching significance
7. **ResourceConfigurationService** — on version publish

Each integration point calls `NotificationService.notify(...)` — thin integration, all logic in the notification service.

## Security Requirements

- Notifications scoped to organization (multi-tenant isolation)
- Users can only see their own notifications
- Admins can configure org settings but cannot read other users' notifications
- Slack webhook URLs encrypted at rest (Rails credentials or attr_encrypted)
- Email delivery respects user's verified email only
- No sensitive data in notification body (link to source instead)
- Rate limit notification creation: max 100 per user per hour (prevent runaway loops)

## Performance Requirements

- Notification creation: < 50ms (async delivery)
- Unread count query: < 10ms (indexed)
- List query: < 100ms (paginated, indexed by user + status)
- Digest compilation: < 30s per organization
- Slack delivery: < 2s including retry
- Storage: ~100 bytes per notification, auto-cleanup after 90 days

## Success Metrics

- 80% of high-priority notifications read within 1 hour
- Daily active notification users > 50% of org members
- < 5% notification opt-out rate (indicates good signal-to-noise)
- Zero missed security notifications
- Slack delivery success rate > 99%

## Rollout Plan

1. **Phase A**: Data model + NotificationService + in-app delivery only
2. **Phase B**: Dashboard UI (bell, center, preferences)
3. **Phase C**: Email delivery + digest system
4. **Phase D**: Slack integration
5. **Phase E**: Event publisher integrations (start with cloud jobs + blueprints)
6. **Phase F**: Remaining integrations + admin settings UI

## Open Questions

1. Should we support @mentions in blueprint task descriptions that trigger notifications?
2. Do we need notification grouping (e.g., "3 tasks completed in Blueprint X")?
3. Should the CLI show notification count in the TUI sidebar?
4. What's the right batch window for medium-priority? 15 min vs 1 hour?
