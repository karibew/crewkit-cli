---
doc_tier: 1
doc_type: report
doc_status: implemented
created: 2025-11-01
last_reviewed: 2026-01-06
owner: platform-team
tags: [telemetry, storage, digitalocean, decision]
---

# Telemetry Storage - Final Recommendation (Based on Actual DO Offerings)

**Context**: crewkit needs telemetry storage for 10M events/day
**Constraint**: DigitalOcean does NOT offer managed ClickHouse
**Available on DO Managed Databases**: PostgreSQL, MySQL, MongoDB, Kafka, Valkey, OpenSearch

---

## DigitalOcean Managed Database Options (Verified)

Based on actual DO pricing: https://www.digitalocean.com/pricing/managed-databases

### PostgreSQL (Managed)

| Tier | CPU | RAM | Storage | Price/Month |
|------|-----|-----|---------|-------------|
| Basic | 1 vCPU (shared) | 1 GB | 10 GB | $15 |
| Small | 1 vCPU | 2 GB | 25 GB | $30 |
| Medium | 2 vCPU | 4 GB | 38 GB | $60 |
| **Large** | **4 vCPU** | **8 GB** | **115 GB** | **$120** |
| X-Large | 8 vCPU | 16 GB | 270 GB | $240 |
| 2X-Large | 16 vCPU | 32 GB | 580 GB | $480 |

**Additional storage**: $2/10GB increments
**Max storage**: 30 TB (PostgreSQL only)

**High Availability** (recommended for production):
- Primary + Standby nodes for automatic failover
- Cost: 2x (e.g., 4 vCPU HA = $120 × 2 = $240/month)

---

## Telemetry Storage Options - REAL Comparison

### Option 1: DigitalOcean Managed PostgreSQL ⭐ BEST FOR GETTING STARTED

**Cost**:
- 4 vCPU, 8 GB RAM, 115 GB storage: **$120/month**
- High Availability (2 nodes): **$240/month**
- Additional 70 GB storage: $14/month (7 × $2)

**Total**: $120-240/month (depending on HA requirement)

**Performance**:
- Query latency: 1-5 seconds (acceptable for dashboards)
- Write throughput: 5,000-10,000 rows/sec
- Compression: 2-3x
- Max scale: ~10M events/day (current load ✅)

**Operational overhead**:
- Setup: 2 hours (just create tables)
- Ongoing: **2-3 hours/month** (monitor slow queries, add indexes)

**Pros**:
- ✅ Fully managed by DigitalOcean
- ✅ Familiar PostgreSQL (team already knows it)
- ✅ Can reuse existing app DB (or separate cluster)
- ✅ Automatic backups, monitoring, failover
- ✅ Good enough for 10M events/day

**Cons**:
- ❌ Slower than columnar databases (1-5s vs 50ms)
- ❌ Poor compression (2x vs 10x)
- ❌ Storage costs grow faster
- ❌ Won't scale beyond 50M events/day

**When to upgrade**: Queries >5s, storage >500 GB, or >50M events/day

---

### Option 2: Timescale Cloud (Managed TimescaleDB) ⭐ BEST LONG-TERM

**Cost** (for 10M events/day, 70GB storage after compression):
- Compute: 4 CPU, 8 GB RAM = ~$100/month
- Storage: 70 GB × $0.001212/GB-hour × 730 hrs = ~$62/month
- **Total: ~$162/month**

**Performance**:
- Query latency: 200-500ms (10x faster than PostgreSQL)
- Write throughput: 10,000-50,000 rows/sec
- Compression: 3-5x (better than PostgreSQL)
- Max scale: 100M events/day

**Operational overhead**:
- Setup: 4 hours (migration from PostgreSQL)
- Ongoing: **1-2 hours/month** (cost monitoring only)

**Pros**:
- ✅ Fully managed (zero-ops)
- ✅ 10x faster than PostgreSQL for time-series
- ✅ PostgreSQL-compatible (easy migration)
- ✅ Time-series optimizations
- ✅ 99.95% uptime SLA
- ✅ Scales to 100M events/day

**Cons**:
- ❌ Not on DigitalOcean (separate vendor)
- ❌ More expensive than DO PostgreSQL ($162 vs $120)
- ❌ Still slower than ClickHouse (500ms vs 50ms)

**When to use**: After 6 months when PostgreSQL queries become slow

---

### Option 3: ClickHouse Cloud (Managed) - If Budget Allows

**Cost** (for 10M events/day, 70GB storage):
- Development tier (single node): ~$50-100/month
- **Production tier** (2 replicas, SLA): **$300-500/month**
- Enterprise tier (dedicated): $2,669+/month

**Performance**:
- Query latency: 50ms (best in class)
- Write throughput: 1M+ rows/sec
- Compression: 6-10x (best)
- Max scale: 1B+ events/day

**Operational overhead**:
- Setup: 4 hours (schema design, migration)
- Ongoing: **1-2 hours/month** (cost monitoring only)

**Pros**:
- ✅ Fully managed (zero-ops)
- ✅ Fastest queries (50ms)
- ✅ Best compression (10x)
- ✅ Scales infinitely
- ✅ Purpose-built for analytics

**Cons**:
- ❌ Expensive ($300-500/month for production)
- ❌ Not on DigitalOcean (separate vendor)
- ❌ Overkill for 10M events/day
- ❌ Recent 30% price increase (Jan 2025)

**When to use**: Budget >$500/month, need <100ms queries, scale >100M events/day

---

### Option 4: Self-Managed ClickHouse on DO Droplets ❌ NOT RECOMMENDED

**Cost**:
- Droplet: 4 vCPU, 8 GB RAM = $48/month
- Backups: $10/month
- Monitoring: $10/month
- **Engineering time**: 12 hrs/month @ $100/hr = **$1,200/month**
- **Total**: $1,268/month

**Performance**: Same as ClickHouse Cloud (50ms queries)

**Operational overhead**: **12 hours/month**
- Manual backups and restore testing
- Security patches
- Performance tuning
- Scaling
- Monitoring and alerting
- On-call for outages

**Pros**:
- ✅ Cheapest infrastructure ($68/month)
- ✅ Full control

**Cons**:
- ❌ **Massive operational burden** (12 hrs/month)
- ❌ **True cost higher than managed alternatives** ($1,268 vs $300)
- ❌ No HA (single point of failure)
- ❌ Manual everything (backups, scaling, monitoring)
- ❌ You're on-call 24/7

**Verdict**: **Never do this** - costs more than managed ClickHouse Cloud!

---

## Updated Cost Comparison (Realistic)

| Solution | Monthly Cost | Ops Hours | Query Speed | Scale Limit | Recommendation |
|----------|-------------|-----------|-------------|-------------|----------------|
| **DO PostgreSQL (managed)** | **$120-240** | 2-3 hrs | 1-5s | 50M/day | **✅ START HERE** |
| **Timescale Cloud** | **$162** | 1-2 hrs | 200-500ms | 100M/day | **✅ UPGRADE TO THIS** |
| ClickHouse Cloud | $300-500 | 1-2 hrs | 50ms | 1B+/day | ⚠️ If budget allows |
| Self-managed ClickHouse | $1,268* | 12 hrs | 50ms | 1B+/day | ❌ NOT WORTH IT |
| Snowflake | $1,872 | 1 hr | 200ms | 1B+/day | ❌ TOO EXPENSIVE |

*Includes $1,200/month in engineer time

---

## My Final Recommendation (Based on Real DO Options)

### Phase 1 (Now - 6 Months): DigitalOcean Managed PostgreSQL ⭐

**Why this is the right choice**:
1. **Already on DigitalOcean** - Single vendor, simpler ops
2. **Fully managed** - Automatic backups, failover, monitoring
3. **Good enough** - 1-5s queries acceptable for dashboards
4. **Affordable** - $120/month (or $240 with HA)
5. **Zero new systems** - Team already knows PostgreSQL
6. **Easy to upgrade later** - Can migrate to Timescale/ClickHouse when needed

**Setup**:
```sql
-- Create tables in DO managed PostgreSQL
CREATE TABLE telemetry_metrics (
    timestamp TIMESTAMPTZ NOT NULL,
    organization_id UUID NOT NULL,
    session_id UUID NOT NULL,
    metric_name TEXT NOT NULL,
    metric_value NUMERIC(20, 4) NOT NULL,
    metric_unit TEXT NOT NULL,
    attributes JSONB,
    date DATE GENERATED ALWAYS AS (DATE(timestamp)) STORED
);

-- Partition by month for performance
CREATE TABLE telemetry_metrics_2025_10 PARTITION OF telemetry_metrics
FOR VALUES FROM ('2025-10-01') TO ('2025-11-01');

-- Indexes for common queries
CREATE INDEX idx_metrics_org_time ON telemetry_metrics(organization_id, timestamp DESC);
CREATE INDEX idx_metrics_session ON telemetry_metrics(session_id);
CREATE INDEX idx_metrics_date ON telemetry_metrics(date, organization_id);

-- Materialized view for daily rollups (faster queries)
CREATE MATERIALIZED VIEW telemetry_daily_summary AS
SELECT
    DATE(timestamp) as date,
    organization_id,
    metric_name,
    COUNT(*) as event_count,
    AVG(metric_value) as avg_value,
    SUM(metric_value) as total_value
FROM telemetry_metrics
GROUP BY DATE(timestamp), organization_id, metric_name;

-- Refresh daily
CREATE INDEX ON telemetry_daily_summary(organization_id, date);
```

**Monitoring**:
- Query performance (alert if queries >5s)
- Storage growth (alert if >400 GB)
- Write throughput (alert if writes slow)

**When to migrate**: Queries consistently >5s, storage >500 GB, or scale >50M events/day

**Cost**: $120-240/month (depending on HA)

---

### Phase 2 (6-12 Months): Timescale Cloud ⭐

**Trigger**: When PostgreSQL queries >5s or storage >500 GB

**Why upgrade to Timescale**:
1. **10x faster** - 200-500ms queries vs 1-5s
2. **Better compression** - 3-5x vs 2x (lower storage costs)
3. **PostgreSQL-compatible** - Easy migration (just `pg_dump` → `pg_restore`)
4. **Time-series optimized** - Built for this exact use case
5. **Fully managed** - Zero-ops, 99.95% SLA
6. **Scales to 100M/day** - Room to grow

**Migration** (2 weeks):
1. Week 1: Set up Timescale Cloud cluster, test schema
2. Week 2: Backfill historical data with `pg_dump` → `timescaledb_parallel_copy`
3. Week 3: Parallel write (PostgreSQL + Timescale for validation)
4. Week 4: Cutover queries to Timescale, keep PostgreSQL as backup

**Cost**: $162/month

---

### Phase 3 (Optional, 12+ Months): ClickHouse Cloud

**Trigger**: Budget >$500/month AND need <100ms queries AND scale >100M events/day

**Only if**:
- You've raised Series B+ ($10M+)
- You have >5 data analysts needing ultra-fast queries
- Dashboard latency is a competitive advantage
- You're processing >100M events/day

**Cost**: $300-500/month

---

## Architecture Decision

### For crewkit (Series A, 1,000 users, small team)

**Now → 6 months**:
```
CLI → API → Solid Queue → DigitalOcean Managed PostgreSQL
                           ↓
                      Queries: 1-5s
                      Cost: $120-240/month
```

**6-12 months** (if queries slow down):
```
CLI → API → Solid Queue → Timescale Cloud
                           ↓
                      Queries: 200-500ms
                      Cost: $162/month
```

**12+ months** (if needed):
```
CLI → API → Solid Queue → ClickHouse Cloud
                           ↓
                      Queries: 50ms
                      Cost: $300-500/month
```

---

## Implementation Plan (Updated)

### Week 1: Set Up PostgreSQL Schema

**Task**: Create telemetry tables in DO managed PostgreSQL

**Checklist**:
- [ ] Create `telemetry_metrics` table with partitioning
- [ ] Create `telemetry_events` table
- [ ] Create `telemetry_sessions_summary` materialized view
- [ ] Add indexes for common query patterns
- [ ] Set up pg_cron for daily materialized view refresh
- [ ] Test write throughput (insert 10K rows)
- [ ] Test query performance (aggregation queries)

**Time**: 4 hours
**Cost**: $0 (use existing PostgreSQL or provision new $120/month cluster)

---

### Week 2-4: Implement API + Jobs

**Task**: Build Solid Queue jobs to write to PostgreSQL

**Checklist**:
- [ ] Create `TelemetryMetricsWriteJob` (batch insert)
- [ ] Create `TelemetryEventsWriteJob` (batch insert)
- [ ] Implement batching (1000 events per job)
- [ ] Add retry logic (exponential backoff)
- [ ] Test with mock data (1M events)
- [ ] Monitor query performance
- [ ] Set up alerts (slow queries >5s)

**Time**: 2-3 days
**Cost**: $0 (part of API development)

---

### Month 1-6: Monitor & Optimize

**Tasks**:
- Monitor query performance weekly
- Optimize indexes based on slow query log
- Add materialized views for common aggregations
- Partition old tables monthly
- Test query performance at scale (10M, 50M, 100M events)

**When to migrate**: If queries consistently >5s or storage >500 GB

---

## Key Takeaways

### What Changed from Original Plan

❌ **Original assumption**: ClickHouse managed on DO at $48/month
✅ **Reality**: No managed ClickHouse on DO

❌ **Original plan**: Use ClickHouse immediately
✅ **Correct plan**: Start with DO PostgreSQL, upgrade later if needed

❌ **Ignored cost**: Engineering time for self-managed
✅ **Factored in**: $1,200/month ops burden makes self-managed expensive

### Why This Plan Works

1. **Start simple**: DO managed PostgreSQL ($120/month)
2. **Good enough**: 1-5s queries acceptable for dashboards
3. **Fully managed**: Zero-ops, automatic backups, monitoring
4. **Single vendor**: Everything on DigitalOcean (simpler)
5. **Upgrade path**: Migrate to Timescale when needed
6. **Cost effective**: Save $21,748/year vs Snowflake

### When to Reconsider

Upgrade to **Timescale Cloud** if:
- [ ] Queries consistently >5s
- [ ] Storage >500 GB
- [ ] Scale >50M events/day
- [ ] Need <500ms query latency

Upgrade to **ClickHouse Cloud** if:
- [ ] Budget >$500/month
- [ ] Need <100ms query latency
- [ ] Scale >100M events/day
- [ ] Have >5 data analysts

**Never self-manage ClickHouse** unless you have a dedicated platform engineering team.

---

## Summary

**Recommended Architecture**:
1. **Now**: DigitalOcean Managed PostgreSQL - $120-240/month ✅
2. **6 months**: Timescale Cloud (if needed) - $162/month
3. **12 months**: ClickHouse Cloud (if needed) - $300-500/month

**Cost savings** vs original (incorrect) plan:
- Year 1: $120/month × 12 = $1,440
- vs Snowflake: $1,872/month × 12 = $22,464
- **Savings: $21,024 in Year 1**

**Key decision**: Start with what DigitalOcean actually offers (managed PostgreSQL), upgrade only when performance demands it.

---

**Document Version**: 1.2 (Final, Corrected)
**Last Updated**: 2025-10-17
**Based on**: Actual DigitalOcean managed database offerings
**Recommendation**: Start with DO PostgreSQL, upgrade to Timescale Cloud when queries slow down
