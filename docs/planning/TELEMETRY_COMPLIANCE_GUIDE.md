# Telemetry Data Compliance Guide

**Frameworks**: SOC2, GDPR, HIPAA
**Purpose**: Design telemetry storage for future compliance certification
**Status**: Preventive architecture (implement before you need it)

---

## Table of Contents

1. [Compliance Overview](#compliance-overview)
2. [Data Classification](#data-classification)
3. [Architecture Changes for Compliance](#architecture-changes-for-compliance)
4. [SOC2 Requirements](#soc2-requirements)
5. [GDPR Requirements](#gdpr-requirements)
6. [HIPAA Requirements](#hipaa-requirements)
7. [Implementation Checklist](#implementation-checklist)
8. [Cost Impact](#cost-impact)

---

## 1. Compliance Overview

### What Each Framework Cares About

| Framework | Focus | Key Requirement for Telemetry |
|-----------|-------|------------------------------|
| **SOC2** | Security, availability, confidentiality | Audit logs, access controls, encryption |
| **GDPR** | Data privacy (EU residents) | Right to erasure, data minimization, consent |
| **HIPAA** | Healthcare data protection (US) | PHI encryption, audit trails, BAA contracts |

### Critical Question: Does Telemetry Contain PII/PHI?

**Our telemetry data**:
- ✅ Session IDs (pseudonymous identifiers)
- ✅ Metrics (tokens, cost, response time)
- ✅ Model names, agent names
- ❌ User prompts (redacted by default in Claude Code)
- ❌ Code content (not captured)
- ⚠️ File paths (could reveal project structure - consider PII)

**Verdict**: Mostly operational data, minimal PII. However, **treat as sensitive** to be safe.

---

## 2. Data Classification

### Sensitivity Levels

**Level 1: Public** (no compliance risk)
- Aggregate statistics (e.g., "Average cost across all users: $0.50")
- Anonymous metrics (e.g., "Total API calls: 1M")

**Level 2: Internal** (low compliance risk)
- Session IDs (UUIDs, not directly identifiable)
- Metrics per organization (org IDs are pseudonymous)
- Agent performance stats

**Level 3: Confidential** (compliance-sensitive)
- User IDs linked to sessions
- Email addresses (if stored)
- Project names, file paths (could reveal business info)

**Level 4: Restricted** (high compliance risk)
- User prompts (if opt-in enabled)
- Code snippets (if captured for debugging)
- Credentials, API keys (should NEVER be in telemetry)

### Recommended Classification for crewkit Telemetry

Store in ClickHouse:
- ✅ Level 1: Aggregate stats (safe)
- ✅ Level 2: Pseudonymous metrics (org_id, session_id)
- ⚠️ Level 3: User_id linkage (add compliance controls)
- ❌ Level 4: No user prompts or code (keep redacted)

---

## 3. Architecture Changes for Compliance

### 3.1 Data Encryption

**At Rest** (stored data):
```sql
-- ClickHouse encryption
CREATE TABLE telemetry_metrics (
    -- ... columns ...
) ENGINE = MergeTree()
SETTINGS
    storage_policy = 'encrypted_storage';  -- Enable disk encryption
```

**DigitalOcean Managed ClickHouse**:
- ✅ Automatic disk encryption (AES-256)
- ✅ Encrypted backups
- ✅ No additional config needed

**At Transit** (data moving):
```bash
# Force TLS for ClickHouse connections
CLICKHOUSE_HOST=your-cluster.db.ondigitalocean.com
CLICKHOUSE_PORT=8443  # TLS port (not 8123)
CLICKHOUSE_TLS=true
```

```ruby
# api/lib/clickhouse_client.rb
def connection
  uri = URI("https://#{host}:#{port}/")  # HTTPS, not HTTP
  http = Net::HTTP.new(uri.hostname, uri.port)
  http.use_ssl = true
  http.verify_mode = OpenSSL::SSL::VERIFY_PEER
  http
end
```

### 3.2 Access Controls (RBAC)

**ClickHouse Users** (principle of least privilege):
```sql
-- Read-only user for dashboards
CREATE USER dashboard_readonly
IDENTIFIED BY 'strong_password'
SETTINGS readonly = 1;

GRANT SELECT ON crewkit_telemetry.* TO dashboard_readonly;

-- Write-only user for API
CREATE USER api_writer
IDENTIFIED BY 'strong_password';

GRANT INSERT ON crewkit_telemetry.telemetry_metrics TO api_writer;
GRANT INSERT ON crewkit_telemetry.telemetry_events TO api_writer;

-- Admin user (for migrations only)
CREATE USER admin
IDENTIFIED BY 'strong_password'
SETTINGS readonly = 0;

GRANT ALL ON crewkit_telemetry.* TO admin;
```

**Rails API** (row-level security):
```ruby
# ALWAYS scope queries by organization
class TelemetryQueryService
  def self.session_summary(session)
    query = <<~SQL
      SELECT * FROM telemetry_metrics
      WHERE organization_id = {org_id:String}  -- Mandatory filter
        AND session_id = {session_id:String}
    SQL

    ClickhouseClient.execute(query, {
      org_id: session.organization_id.to_s,
      session_id: session.id.to_s
    })
  end
end
```

### 3.3 Audit Logging

**What to log**:
- ✅ Who accessed telemetry data (user_id, IP, timestamp)
- ✅ What they accessed (organization_id, session_id)
- ✅ What they did (read, export, delete)
- ✅ When (timestamp)

**Implementation**:

```ruby
# api/app/models/audit_log.rb
class AuditLog < ApplicationRecord
  belongs_to :user
  belongs_to :organization

  # Store in PostgreSQL (not ClickHouse) for tamper-evidence
  def self.log_telemetry_access(user:, organization:, action:, resource:)
    create!(
      user: user,
      organization: organization,
      action: action,  # 'read', 'export', 'delete'
      resource_type: 'telemetry',
      resource_id: resource[:session_id],
      metadata: {
        ip_address: user.current_sign_in_ip,
        user_agent: user.current_sign_in_user_agent,
        query: resource[:query]
      },
      created_at: Time.current
    )
  end
end

# api/app/controllers/api/v1/telemetry_controller.rb
def session_summary
  session = current_organization.agent_sessions.find_by!(external_id: params[:session_id])

  summary = TelemetryQueryService.session_summary(session)

  # Log the access
  AuditLog.log_telemetry_access(
    user: current_user,
    organization: current_organization,
    action: 'read',
    resource: { session_id: session.id }
  )

  render json: summary
end
```

**Retention**: Keep audit logs for **7 years** (SOC2 requirement)

### 3.4 Data Retention Policies

**GDPR**: Must delete user data on request (right to erasure)
**SOC2**: Must retain audit logs for 7 years
**Recommendation**: Tiered retention

```sql
-- Telemetry data retention
CREATE TABLE telemetry_metrics (
    -- ... columns ...
    date Date MATERIALIZED toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY (organization_id, toYYYYMM(date))
TTL
    date + INTERVAL 90 DAY DELETE,           -- Delete raw data after 90 days
    date + INTERVAL 30 DAY TO VOLUME 'cold'  -- Move to cheaper storage after 30 days
```

**Multi-tier storage**:
- 0-30 days: Hot SSD (fast queries, expensive)
- 30-90 days: Cold storage (slower, cheaper)
- >90 days: Deleted (unless legal hold)

**Exception**: Aggregated/anonymized data can be kept longer
```sql
-- Aggregated metrics (no PII) - keep for 3 years
CREATE TABLE telemetry_metrics_monthly_anon (
    -- No user_id, no session_id (fully anonymized)
    month Date,
    organization_id String,
    metric_name String,
    avg_value Float64,
    total_count UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(month)
TTL month + INTERVAL 3 YEAR DELETE;
```

### 3.5 Data Minimization (GDPR)

**Principle**: Only collect what you need

**What to redact/exclude**:
```typescript
// cli/src/services/telemetry-server.ts
function sanitizeAttributes(attrs: Record<string, any>): Record<string, any> {
  return {
    model: attrs.model,                    // ✅ Keep
    tokens: attrs.tokens,                  // ✅ Keep
    // Redact potentially sensitive fields
    file_path: attrs.file_path ? '[REDACTED]' : undefined,  // ❌ Remove
    prompt: undefined,                     // ❌ Never send (even if opt-in)
    code_snippet: undefined,               // ❌ Never send
    error_message: attrs.error_message?.slice(0, 100),  // ⚠️ Truncate
  }
}
```

**Prompt opt-in** (if you add it later):
```typescript
// Only send if user explicitly opts in AND accepts data usage terms
if (user.telemetry_prompts_enabled && user.accepted_terms_version >= 2) {
  metrics.prompt_length = prompt.length  // Send length, not content
  // Do NOT send actual prompt text
}
```

### 3.6 Anonymization & Pseudonymization

**Pseudonymization**: Replace identifiers with pseudonyms (reversible)
- Use session UUIDs (not sequential IDs)
- Use organization slugs (not company names)

**Anonymization**: Remove all identifiers (irreversible)
- Aggregate data only (no session/user linkage)
- Use k-anonymity (minimum group size)

**Example**:
```sql
-- Pseudonymized (can link back to user)
SELECT session_id, user_id, cost FROM telemetry_metrics;

-- Anonymized (cannot link back)
SELECT
    toStartOfWeek(date) as week,
    count(DISTINCT session_id) as sessions,  -- Count only
    avg(cost) as avg_cost
FROM telemetry_metrics
GROUP BY week
HAVING sessions >= 5;  -- k-anonymity: min 5 sessions per group
```

---

## 4. SOC2 Requirements

### 4.1 Security Controls (Trust Service Criteria)

**CC6.1: Logical Access Controls**
- ✅ Multi-factor authentication (Devise MFA)
- ✅ Role-based access (Pundit policies)
- ✅ Password complexity requirements
- ✅ Session timeouts (JWT expiry)

**CC6.6: Data Protection**
- ✅ Encryption at rest (ClickHouse disk encryption)
- ✅ Encryption in transit (TLS 1.3)
- ✅ Secure key management (Rails credentials)

**CC6.7: System Monitoring**
- ✅ Intrusion detection (Sentry alerts)
- ✅ Log monitoring (audit logs)
- ✅ Anomaly detection (rate limit breaches)

**CC7.1: Change Management**
- ✅ Version control (Git)
- ✅ Code review (PR approvals)
- ✅ Automated testing (CI/CD)

### 4.2 Audit Evidence (What Auditors Will Ask For)

**You need to provide**:
1. **Access logs**: Who accessed telemetry data (AuditLog model)
2. **Change logs**: Schema migrations, config changes (Git history)
3. **Incident logs**: Security events, breaches (Sentry)
4. **Backup logs**: When backups occurred, restoration tests (ClickHouse backups)
5. **Encryption proof**: TLS certs, disk encryption enabled (DO screenshots)

**Implementation**:
```ruby
# api/app/models/audit_log.rb
class AuditLog < ApplicationRecord
  # SOC2 requirement: tamper-evident logs
  before_create :calculate_checksum

  def calculate_checksum
    # Hash of previous log + current log data
    previous = AuditLog.order(:created_at).last
    data = "#{previous&.checksum}|#{user_id}|#{action}|#{created_at}"
    self.checksum = Digest::SHA256.hexdigest(data)
  end

  # Verify chain integrity
  def self.verify_chain
    logs = order(:created_at).to_a
    logs.each_cons(2).all? do |prev, curr|
      expected = Digest::SHA256.hexdigest("#{prev.checksum}|#{curr.user_id}|#{curr.action}|#{curr.created_at}")
      expected == curr.checksum
    end
  end
end
```

**Migration**:
```ruby
# api/db/migrate/YYYYMMDDHHMMSS_create_audit_logs.rb
class CreateAuditLogs < ActiveRecord::Migration[8.0]
  def change
    create_table :audit_logs do |t|
      t.references :user, null: false, foreign_key: true
      t.references :organization, null: false, foreign_key: true
      t.string :action, null: false  # 'read', 'write', 'delete', 'export'
      t.string :resource_type, null: false  # 'telemetry', 'session', 'organization'
      t.string :resource_id
      t.json :metadata  # IP, user agent, query details
      t.string :checksum, null: false  # Tamper detection
      t.datetime :created_at, null: false

      # Performance + compliance
      t.index [:organization_id, :created_at]
      t.index [:user_id, :created_at]
      t.index [:resource_type, :resource_id]
      t.index :checksum, unique: true
    end

    # Prevent updates/deletes (append-only for tamper-evidence)
    reversible do |dir|
      dir.up do
        execute <<~SQL
          CREATE OR REPLACE FUNCTION prevent_audit_log_changes()
          RETURNS TRIGGER AS $$
          BEGIN
            RAISE EXCEPTION 'Audit logs are immutable';
          END;
          $$ LANGUAGE plpgsql;

          CREATE TRIGGER audit_log_immutable
            BEFORE UPDATE OR DELETE ON audit_logs
            FOR EACH ROW EXECUTE FUNCTION prevent_audit_log_changes();
        SQL
      end
    end
  end
end
```

### 4.3 SOC2 Compliance Checklist

- [ ] **Encryption**: TLS for API, disk encryption for ClickHouse
- [ ] **Access control**: RBAC in Rails, ClickHouse user permissions
- [ ] **Audit logs**: PostgreSQL AuditLog model (tamper-evident)
- [ ] **Data backups**: Automated daily (ClickHouse managed backups)
- [ ] **Incident response**: Documented runbook, Sentry alerts
- [ ] **Vendor management**: BAA with DigitalOcean (if needed)
- [ ] **Change management**: Git history, PR approvals, CI/CD
- [ ] **Monitoring**: Sentry, queue depth alerts, error rates

---

## 5. GDPR Requirements

### 5.1 Right to Erasure ("Right to be Forgotten")

**Challenge**: ClickHouse is append-only (no traditional DELETE)

**Solution 1**: Partition-level deletion (fast)
```sql
-- Drop entire organization partition (when org deleted)
ALTER TABLE telemetry_metrics
DROP PARTITION ('{org_id}', '202510');  -- Drop Oct 2025 data

-- Repeat for all partitions
```

**Solution 2**: Mark as deleted (lazy deletion)
```sql
-- Add deleted_at column
ALTER TABLE telemetry_metrics
ADD COLUMN deleted_at Nullable(DateTime64(3)) AFTER attributes;

-- Update (mutation, slow but works)
ALTER TABLE telemetry_metrics
UPDATE deleted_at = now()
WHERE organization_id = '{org_id}';

-- Filter out deleted rows in queries
SELECT * FROM telemetry_metrics
WHERE organization_id = '{org_id}'
  AND deleted_at IS NULL;  -- Exclude deleted

-- Physically delete later (background job)
ALTER TABLE telemetry_metrics
DELETE WHERE deleted_at < now() - INTERVAL 30 DAY;
```

**Solution 3**: Hybrid approach (recommended)
```ruby
# api/app/services/gdpr_erasure_service.rb
class GdprErasureService
  def self.erase_organization(organization)
    # 1. Delete from PostgreSQL (fast)
    organization.agent_sessions.delete_all
    organization.agent_configurations.delete_all

    # 2. Queue ClickHouse deletion (slow, async)
    GdprClickhouseErasureJob.perform_later(
      organization_id: organization.id
    )

    # 3. Mark organization as deleted
    organization.update!(
      deleted_at: Time.current,
      name: '[DELETED]',
      email: nil
    )
  end
end

# api/app/jobs/gdpr_clickhouse_erasure_job.rb
class GdprClickhouseErasureJob < ApplicationJob
  queue_as :low

  def perform(organization_id:)
    # Drop partitions (fast)
    partitions = ClickhouseClient.execute(<<~SQL)
      SELECT DISTINCT partition FROM system.parts
      WHERE database = 'crewkit_telemetry'
        AND table = 'telemetry_metrics'
        AND partition LIKE '#{organization_id}%'
    SQL

    partitions.each do |partition|
      ClickhouseClient.execute(<<~SQL)
        ALTER TABLE telemetry_metrics DROP PARTITION #{partition}
      SQL
    end

    Rails.logger.info("[GDPR] Deleted #{partitions.size} partitions for org #{organization_id}")
  end
end
```

**Timeline**: GDPR requires deletion within **30 days** of request

### 5.2 Data Portability

**Requirement**: User can export their data in machine-readable format

```ruby
# api/app/controllers/api/v1/telemetry_controller.rb
def export
  authorize current_organization, :export_telemetry?

  # Generate CSV/JSON export
  export = TelemetryExportService.generate(
    organization: current_organization,
    format: params[:format] || 'csv',
    start_date: params[:start_date],
    end_date: params[:end_date]
  )

  # Log the export (audit trail)
  AuditLog.log_telemetry_access(
    user: current_user,
    organization: current_organization,
    action: 'export',
    resource: { format: params[:format], date_range: "#{params[:start_date]} to #{params[:end_date]}" }
  )

  send_data export.to_csv, filename: "telemetry-#{Date.current}.csv"
end

# api/app/services/telemetry_export_service.rb
class TelemetryExportService
  def self.generate(organization:, format:, start_date:, end_date:)
    query = <<~SQL
      SELECT
        timestamp,
        session_id,
        metric_name,
        metric_value,
        metric_unit
      FROM telemetry_metrics
      WHERE organization_id = {org_id:String}
        AND date >= {start_date:Date}
        AND date <= {end_date:Date}
      FORMAT CSV
    SQL

    ClickhouseClient.execute(query, {
      org_id: organization.id.to_s,
      start_date: start_date,
      end_date: end_date
    })
  end
end
```

### 5.3 Consent Management

**Requirement**: User must consent to data collection

```ruby
# api/app/models/user.rb
class User < ApplicationRecord
  # GDPR consent tracking
  def telemetry_consent_given?
    accepted_telemetry_terms_at.present? &&
    accepted_telemetry_terms_version >= CURRENT_TERMS_VERSION
  end

  def give_telemetry_consent!
    update!(
      accepted_telemetry_terms_at: Time.current,
      accepted_telemetry_terms_version: CURRENT_TERMS_VERSION
    )
  end

  def revoke_telemetry_consent!
    update!(
      accepted_telemetry_terms_at: nil,
      telemetry_enabled: false
    )
  end
end

# api/db/migrate/YYYYMMDDHHMMSS_add_telemetry_consent_to_users.rb
class AddTelemetryConsentToUsers < ActiveRecord::Migration[8.0]
  def change
    add_column :users, :telemetry_enabled, :boolean, default: true
    add_column :users, :accepted_telemetry_terms_at, :datetime
    add_column :users, :accepted_telemetry_terms_version, :integer

    add_index :users, :telemetry_enabled
  end
end
```

**CLI Check**:
```typescript
// cli/src/commands/code.ts
async run() {
  // Check if user has consented to telemetry
  const user = await this.apiClient.get('/api/v1/users/me')

  if (!user.telemetry_consent_given) {
    const consent = await this.promptTelemetryConsent()
    if (!consent) {
      // Disable telemetry for this session
      return this.runWithoutTelemetry()
    }
  }

  // Continue with telemetry enabled
  await this.startTelemetryServer()
}
```

### 5.4 GDPR Compliance Checklist

- [ ] **Data minimization**: Only collect necessary fields
- [ ] **Consent**: Explicit opt-in, granular (telemetry vs marketing)
- [ ] **Right to access**: API endpoint to view own data
- [ ] **Right to erasure**: Delete within 30 days (partition drop)
- [ ] **Data portability**: CSV/JSON export
- [ ] **Breach notification**: 72-hour disclosure plan
- [ ] **Data retention**: Auto-delete after retention period (TTL)
- [ ] **Processor agreements**: BAA with DigitalOcean (GDPR-compliant)

---

## 6. HIPAA Requirements

### 6.1 Does crewkit Handle PHI?

**Protected Health Information (PHI)** includes:
- Names, addresses, SSNs
- Medical record numbers
- Health conditions, treatments
- Payment info (if health-related)

**crewkit telemetry**: Likely **does NOT contain PHI** unless:
- ❌ Users are healthcare orgs coding medical apps
- ❌ Code snippets contain patient data
- ❌ Prompts mention patient names/conditions

**Recommendation**:
- If **no healthcare customers**: HIPAA not required
- If **healthcare customers exist**: Sign BAA, implement HIPAA controls

### 6.2 HIPAA Technical Safeguards (if applicable)

**Access Control** (§164.312(a)):
- ✅ Unique user IDs (Devise)
- ✅ Emergency access procedures (admin override)
- ✅ Automatic logoff (JWT expiry)
- ✅ Encryption (TLS + disk)

**Audit Controls** (§164.312(b)):
- ✅ Record all access to PHI (AuditLog model)
- ✅ Tamper-evident logs (checksums)

**Integrity Controls** (§164.312(c)):
- ✅ Prevent unauthorized modification (immutable audit logs)
- ✅ Detect tampering (checksum verification)

**Transmission Security** (§164.312(e)):
- ✅ TLS 1.3 for all API calls
- ✅ No telemetry via email/unencrypted channels

### 6.3 Business Associate Agreement (BAA)

**Required with**:
- DigitalOcean (infrastructure provider)
- Any third-party tool that touches PHI (Sentry?)

**DigitalOcean BAA**: Available on request for managed services

**Template BAA clause**:
> "Business Associate shall implement administrative, physical, and technical safeguards that reasonably and appropriately protect the confidentiality, integrity, and availability of PHI..."

### 6.4 HIPAA Compliance Checklist

- [ ] **BAA signed**: With DigitalOcean, Sentry (if PHI exposed)
- [ ] **Encryption**: AES-256 at rest, TLS 1.3 in transit
- [ ] **Access logs**: Audit every PHI access (who, what, when)
- [ ] **Minimum necessary**: Only expose PHI to authorized users
- [ ] **Secure disposal**: Shred data after retention period
- [ ] **Incident response**: 60-day breach notification plan
- [ ] **Risk assessment**: Annual HIPAA security review

---

## 7. Implementation Checklist

### 7.1 Immediate (Do Now)

- [ ] **Enable TLS**: Force HTTPS for ClickHouse (port 8443)
- [ ] **Add audit logs**: Create AuditLog model (PostgreSQL)
- [ ] **Data minimization**: Redact file paths, never store prompts
- [ ] **Consent tracking**: Add telemetry consent fields to User model
- [ ] **Document retention**: Set ClickHouse TTL (90 days)

### 7.2 Before Launch (Phase 1)

- [ ] **Encryption at rest**: Verify ClickHouse disk encryption
- [ ] **RBAC**: Create ClickHouse users (readonly, writer, admin)
- [ ] **Row-level security**: Always filter by organization_id
- [ ] **Backup testing**: Verify ClickHouse backups can restore
- [ ] **Privacy policy**: Update to mention telemetry collection

### 7.3 Before SOC2 Audit (Phase 2)

- [ ] **Vendor BAA**: Sign with DigitalOcean, Sentry
- [ ] **Incident response**: Document runbook (breach, outage)
- [ ] **Access reviews**: Quarterly review of who has ClickHouse access
- [ ] **Penetration testing**: Hire third party to test security
- [ ] **Compliance docs**: Gather evidence (logs, configs, screenshots)

### 7.4 Before GDPR Launch (EU Customers)

- [ ] **Data residency**: Deploy EU ClickHouse cluster (Frankfurt region)
- [ ] **Deletion API**: Implement partition-drop erasure
- [ ] **Export API**: CSV/JSON export endpoint
- [ ] **Cookie consent**: Add telemetry consent to onboarding
- [ ] **Privacy policy**: GDPR-compliant wording, data controller contact

### 7.5 Before HIPAA Launch (Healthcare Customers)

- [ ] **BAA signed**: With all vendors (DO, Sentry, etc)
- [ ] **Risk assessment**: HIPAA security risk analysis
- [ ] **PHI detection**: Scan telemetry for accidental PHI exposure
- [ ] **Breach plan**: 60-day notification procedures
- [ ] **Training**: HIPAA training for engineers with access

---

## 8. Cost Impact

### 8.1 Infrastructure Costs

**SOC2** (minimal cost):
- Audit logs: +10 GB PostgreSQL storage = +$2/month
- TLS certificates: Free (Let's Encrypt)
- **Total: +$2/month**

**GDPR** (moderate cost):
- EU data residency: ClickHouse cluster in Frankfurt = +$48/month
- Backup retention (7 years): +50 GB = +$5/month
- **Total: +$53/month**

**HIPAA** (high cost):
- Dedicated cluster (no shared hosting) = +$100/month
- Penetration testing (annual) = $5,000/year ≈ $420/month
- HIPAA consultant (audit prep) = $10,000 one-time
- **Total: +$520/month**

**Recommendation**: Start with SOC2 (cheapest, most customers need it)

### 8.2 Engineering Time

| Task | Effort | When |
|------|--------|------|
| Add audit logs | 2 days | Phase 1 |
| GDPR deletion API | 3 days | Phase 2 |
| Consent management | 2 days | Phase 1 |
| SOC2 audit prep | 2 weeks | Before audit |
| GDPR compliance (full) | 4 weeks | Before EU launch |
| HIPAA compliance (full) | 8 weeks | Before healthcare launch |

---

## 9. Summary & Recommendations

### 9.1 Compliance Priority

**Phase 1: SOC2** (all SaaS companies need this)
- Easiest to implement
- Required by enterprise customers
- Low cost (~$2/month)
- **Timeline**: 3 months to audit-ready

**Phase 2: GDPR** (if EU customers)
- Medium complexity
- Legal requirement for EU residents
- Moderate cost (~$50/month)
- **Timeline**: 2 months after SOC2

**Phase 3: HIPAA** (only if healthcare)
- Hardest to implement
- Only if handling PHI
- High cost (~$500/month)
- **Timeline**: 6 months after SOC2

### 9.2 Architecture Changes (By Priority)

**Must-Have** (do now):
1. ✅ Enable TLS for ClickHouse connections
2. ✅ Add AuditLog model (PostgreSQL)
3. ✅ Redact sensitive fields (file paths, prompts)
4. ✅ Set ClickHouse TTL (90 days)
5. ✅ Add consent tracking to User model

**Should-Have** (before launch):
6. ✅ RBAC in ClickHouse (readonly/writer users)
7. ✅ Row-level security (organization_id filtering)
8. ✅ Tamper-evident audit logs (checksums)
9. ✅ GDPR deletion API (partition drop)
10. ✅ Export API (CSV/JSON)

**Nice-to-Have** (before audit):
11. ✅ EU data residency (Frankfurt cluster)
12. ✅ Backup encryption verification
13. ✅ Incident response runbook
14. ✅ Access review automation

### 9.3 Key Takeaways

**ClickHouse vs Kafka**:
- ClickHouse = long-term storage + fast queries (what you need)
- Kafka = event streaming to multiple consumers (overkill for now)
- Recommendation: Use ClickHouse only, add Kafka if you need >100M events/day

**Compliance**:
- Start with SOC2 (easiest, most valuable)
- Add GDPR if EU customers
- Only do HIPAA if handling healthcare data
- Architecture changes are minimal (mostly additive)

**Cost**:
- SOC2: +$2/month (audit logs)
- GDPR: +$50/month (EU cluster)
- HIPAA: +$500/month (dedicated, pen testing)

**Timeline**:
- SOC2 audit-ready: 3 months
- GDPR compliant: 2 months after SOC2
- HIPAA certified: 6 months after SOC2

---

## 10. Next Steps

1. **Review with legal**: Confirm compliance requirements for your market
2. **Phase 1 implementation**: Add audit logs + TLS + consent tracking (1 week)
3. **Privacy policy**: Update to mention telemetry collection (1 day)
4. **SOC2 audit**: Hire auditor, prepare evidence (3 months)
5. **GDPR/HIPAA**: Implement only if required by customer contracts

**Questions to answer**:
- Do you have/plan to have EU customers? → GDPR
- Do you have/plan to have healthcare customers? → HIPAA
- Do enterprise customers require SOC2? → SOC2 (most likely yes)

---

**Document Version**: 1.0
**Last Updated**: 2025-10-17
**Owner**: crewkit engineering + legal teams
