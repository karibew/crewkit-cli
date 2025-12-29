# CrewKit Security Model

> Comprehensive reference for multi-tenant security, permissions, and audit logging

## Overview

CrewKit uses a multi-tenant architecture with two distinct role systems:

1. **Platform Roles** - System-wide permissions (crewkit staff)
2. **Organization Permissions** - Per-organization access control
3. **Agent Roles** - Agent behavior configuration (coaching/collaborative/autonomous)

## Role Hierarchy

### Platform Roles (User.platform_role)

```
crewkit_owner (0)     → Full system access, can impersonate
├─ crewkit_staff (1)  → View all orgs, support tasks (read-only by default)
└─ org_user (4)       → Regular user (default), access via org memberships
```

**Security Rules:**
- Platform roles are immutable by org users
- Only crewkit_owner can grant/revoke platform roles
- crewkit staff cannot belong to organizations (enforced by validation)

### Organization Permissions (UserOrganizationRole.permission_level)

```
owner (0)             → Full org control, billing, delete org
├─ admin (1)          → Manage members, projects, experiments
├─ manager (2)        → Deploy experiments, edit agent configs
├─ developer (3)      → Create experiments, view sessions
└─ viewer (4)         → Read-only access (default)
```

**Permission Matrix:**

| Action | Viewer | Developer | Manager | Admin | Owner |
|--------|--------|-----------|---------|-------|-------|
| View org data | ✅ | ✅ | ✅ | ✅ | ✅ |
| Create experiments | ❌ | ✅ | ✅ | ✅ | ✅ |
| Deploy experiments | ❌ | ❌ | ✅ | ✅ | ✅ |
| Edit agent configs | ❌ | ❌ | ✅ | ✅ | ✅ |
| Create projects | ❌ | ❌ | ✅ | ✅ | ✅ |
| Invite members | ❌ | ❌ | ❌ | ✅ | ✅ |
| Change member roles | ❌ | ❌ | ❌ | ✅ | ✅ |
| Manage billing | ❌ | ❌ | ❌ | ❌ | ✅ |
| Delete org | ❌ | ❌ | ❌ | ❌ | ✅ |

### Agent Roles (OrganizationRole)

Agent behavior configuration, separate from permissions:
- **coaching** - Guide users, don't write code (for junior devs)
- **collaborative** - Work together with user
- **autonomous** - Take initiative, full autonomy

**Note:** Agent roles do NOT grant permissions. They only affect agent behavior.

## Security Principles

### 1. Privilege Escalation Prevention

**Rule:** Users cannot grant permissions higher than their own.

```ruby
# ✅ Admin can grant developer/viewer
admin.can_grant?(:developer) # => true

# ❌ Admin cannot grant owner/admin
admin.can_grant?(:owner) # => false
admin.can_grant?(:admin) # => false

# ✅ Owner can grant any permission
owner.can_grant?(:owner) # => true
```

**Implementation:**
```ruby
class UserOrganizationRole < ApplicationRecord
  PERMISSION_HIERARCHY = {
    owner: 0,
    admin: 1,
    manager: 2,
    developer: 3,
    viewer: 4
  }.freeze

  def can_grant?(target_permission)
    PERMISSION_HIERARCHY[permission_level.to_sym] <= PERMISSION_HIERARCHY[target_permission.to_sym]
  end
end
```

### 2. Tenant Isolation

**Rule:** All queries must be scoped to current organization.

```ruby
# ✅ Scoped query
current_org.projects.where(...)

# ❌ Global query (NEVER do this)
Project.where(...)
```

**Implementation:**
```ruby
class ApplicationController < ActionController::Base
  before_action :set_current_organization

  private

  def set_current_organization
    @current_organization = current_user.organizations.find_by(id: params[:organization_id])
    raise Pundit::NotAuthorizedError unless @current_organization
  end
end
```

### 3. Owner Protection

**Rule:** Organizations must always have at least one owner.

```ruby
# ✅ Can remove owner if another exists
org.owners.count > 1 # => allow removal

# ❌ Cannot remove last owner
org.owners.count == 1 # => block removal

# ✅ Can downgrade owner if promoting another first
promote_new_owner && downgrade_old_owner # => atomic transaction
```

### 4. Self-Privilege Restrictions

**Rule:** Users cannot change their own permissions (prevents escalation).

```ruby
# ❌ Cannot grant yourself higher permissions
current_user.grant_permission(:owner) # => blocked

# ✅ Another admin must grant you permissions
other_admin.grant_permission_to(current_user, :owner) # => allowed
```

**Exception:** Owners can downgrade themselves if not the last owner.

### 5. Crewkit Staff Boundaries

**Rule:** Staff cannot modify production data without explicit authorization.

```ruby
# ✅ Staff can view (read-only)
policy.show? # => true

# ❌ Staff cannot edit by default
policy.update? # => false (unless impersonating with audit trail)
```

### 6. Invitation Security

**Rule:** Invitations must be scoped and time-limited.

```ruby
class OrganizationInvitation
  validates :expires_at, presence: true
  validates :invited_by, presence: true

  # Cannot invite with higher permission than inviter
  validate :inviter_can_grant_permission

  # Expires in 7 days
  before_create -> { self.expires_at = 7.days.from_now }

  def valid_for_acceptance?
    !accepted? && !expired? && organization.active?
  end
end
```

## Audit Trail

All sensitive actions must be logged:

```ruby
# Required for Paper Trail
class UserOrganizationRole < ApplicationRecord
  has_paper_trail on: [:create, :update, :destroy], meta: {
    performed_by: :whodunnit,
    organization_id: :organization_id
  }
end
```

**Tracked events:**
- Role grants/revocations
- Organization membership changes
- Experiment deployments
- Agent configuration changes
- Permission escalation attempts (even failed ones)

## Implementation Checklist

- [x] Platform role enum on User
- [ ] Permission level enum on UserOrganizationRole
- [ ] Pundit policies for all resources
- [ ] Current organization context service
- [ ] Owner protection validations
- [ ] Self-privilege restriction checks
- [ ] Invitation model with expiration
- [ ] Paper Trail for audit logging
- [ ] Security tests (privilege escalation, tenant isolation)
- [ ] Rate limiting for permission changes

## Testing Strategy

### Required Security Tests

```ruby
# test/models/user_organization_role_test.rb
test "admin cannot grant owner permission" do
  assert_not @admin_role.can_grant?(:owner)
end

test "cannot change own permissions" do
  assert_raises(SecurityError) do
    @user.update_own_permission(:owner)
  end
end

test "cannot remove last owner" do
  assert_raises(ActiveRecord::RecordInvalid) do
    @org.last_owner.destroy
  end
end

# test/policies/organization_policy_test.rb
test "users cannot access other orgs" do
  other_org = organizations(:other)
  assert_not OrganizationPolicy.new(@user, other_org).show?
end

test "crewkit staff have read-only access" do
  policy = OrganizationPolicy.new(@staff_user, @org)
  assert policy.show?
  assert_not policy.update?
end
```

## Emergency Procedures

### Compromised Owner Account

1. Crewkit owner can force-revoke access
2. Audit trail preserved
3. Notify remaining owners
4. Force password/passkey reset

### Data Breach Response

1. Identify affected organizations
2. Force logout all sessions
3. Rotate JWT secrets
4. Notify affected users
5. Review audit logs

## Future Enhancements

- [ ] Time-limited permission grants
- [ ] Approval workflows for sensitive actions
- [ ] Two-factor authentication for permission changes
- [ ] IP allowlisting per organization
- [ ] Session recording for crewkit staff impersonation
