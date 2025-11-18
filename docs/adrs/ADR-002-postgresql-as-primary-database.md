# ADR-002: PostgreSQL as Primary Database - Project Intelligence Platform

```yaml
Document: ADR-002-project-intelligence-postgresql-primary-database
Version: 1.0.0
Purpose: Select PostgreSQL for structured data storage with multi-tenant row-level security
Audience: Engineering teams, architects, database administrators, security officers
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-001 (Git Source of Truth), ADR-004 (Multi-Tenant Strategy)
```

---

## Table of Contents
- [Executive Summary](#executive-summary)
- [Context and Problem Statement](#context-and-problem-statement)
- [Decision Drivers](#decision-drivers)
- [Considered Options](#considered-options)
- [Decision Outcome](#decision-outcome)
- [Consequences](#consequences)
- [Implementation Details](#implementation-details)

---

## Executive Summary

**Decision**: Use **PostgreSQL** as the primary structured database for the CODITECT Project Intelligence Platform.

**Why PostgreSQL**:
- ✅ **Row-Level Security (RLS)** - Native multi-tenant isolation at database level
- ✅ **ACID Transactions** - Data consistency guaranteed
- ✅ **JSONB Support** - Flexible metadata storage without schema migrations
- ✅ **Mature Ecosystem** - 35+ years of production battle-testing
- ✅ **Cloud Native** - Managed services on GCP (Cloud SQL), AWS (RDS), Azure

**Alternatives Rejected**: MongoDB (no RLS), FoundationDB (operational complexity), MySQL (weaker JSON support)

---

## Context and Problem Statement

### The Challenge

CODITECT Project Intelligence Platform requires a database that can:

1. **Store Structured Data**: Organizations, users, projects, checkpoints, messages, tasks
2. **Multi-Tenant Isolation**: Complete data separation between organizations
3. **Complex Queries**: Timeline aggregations, analytics, audit trails
4. **ACID Guarantees**: No data loss or corruption during concurrent operations
5. **Flexible Metadata**: Store arbitrary JSON without schema changes
6. **Cloud Deployment**: Managed service availability on major cloud providers

### Business Requirements

**Multi-Tenancy**:
- 1000+ organizations sharing single database
- Complete data isolation (Org A cannot see Org B's data)
- Row-level security enforced at database layer
- Compliance with SOC2, GDPR, HIPAA

**Performance**:
- Sub-100ms query latency for 95th percentile
- Support 10,000+ concurrent users
- Handle 1M+ messages across all tenants
- Efficient indexes for timeline queries

**Operational**:
- Managed service (no manual database ops)
- Automated backups and point-in-time recovery
- High availability (99.95% uptime)
- Horizontal read scaling (read replicas)

---

## Decision Drivers

### Mandatory Requirements (Must-Have)

1. **Row-Level Security (RLS)** - Automatic tenant isolation at database layer
2. **ACID Transactions** - Consistency guarantees
3. **Mature & Proven** - Production-tested at scale
4. **Cloud Managed Service** - Available on GCP/AWS/Azure
5. **Strong Community** - Active development and support

### Important Goals (Should-Have)

6. **JSON Support** - Store metadata without schema migrations
7. **Full-Text Search** - Native search capabilities (bonus)
8. **Geospatial Support** - Future-proof for location features
9. **Time-Series Optimization** - Efficient date-range queries
10. **Cost-Effective** - Reasonable pricing at scale

### Nice-to-Have

11. **PL/pgSQL Support** - Stored procedures for complex logic
12. **Foreign Data Wrappers** - Query external data sources
13. **Logical Replication** - Real-time data streaming

---

## Considered Options

### Option 1: **PostgreSQL** (SELECTED ✅)

**Technical Profile**:
- **Type**: Relational database (RDBMS)
- **License**: Open-source (PostgreSQL License)
- **First Release**: 1996 (35+ years of development)
- **Version**: 15+ (recommend 15 or 16 for production)

**Key Features**:
```sql
-- Row-Level Security (RLS) - automatic tenant isolation
ALTER TABLE checkpoints ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation ON checkpoints
    FOR ALL
    USING (
        project_id IN (
            SELECT id FROM projects
            WHERE organization_id = current_setting('app.current_user_id')::UUID
        )
    );

-- JSONB support - flexible metadata without schema changes
CREATE TABLE checkpoints (
    id UUID PRIMARY KEY,
    metadata JSONB DEFAULT '{}',
    -- Query JSON fields efficiently
    -- Example: WHERE metadata->>'phase' = 'Week 1'
);

-- Full-text search (built-in)
CREATE INDEX idx_messages_fts ON messages
    USING gin(to_tsvector('english', content));

-- Query: SELECT * FROM messages WHERE to_tsvector('english', content) @@ to_tsquery('authentication');
```

**Pros**:
- ✅ **Row-Level Security (RLS)**: Automatic tenant isolation, zero application code changes
- ✅ **ACID Transactions**: Strong consistency, no data loss
- ✅ **JSONB**: Query JSON fields with indexes (GIN indexes)
- ✅ **Mature**: 35+ years of production use
- ✅ **Ecosystem**: ORMs (SQLAlchemy, TypeORM), migration tools (Alembic, Flyway)
- ✅ **Cloud Managed**: Cloud SQL (GCP), RDS (AWS), Flexible Server (Azure)
- ✅ **Full-Text Search**: Built-in (no external search engine needed for basic use)
- ✅ **Extensions**: PostGIS (geospatial), TimescaleDB (time-series), pgvector (embeddings)
- ✅ **Cost**: ~$200/month for 100 GB Cloud SQL instance

**Cons**:
- ⚠️ **Vertical Scaling Limits**: Single-node write bottleneck (mitigated with read replicas)
- ⚠️ **Schema Rigidity**: Schema changes require migrations (mitigated with JSONB)
- ⚠️ **Complex Sharding**: Horizontal write scaling requires manual sharding (Citus extension helps)

**Cloud Costs** (GCP Cloud SQL):
- **Development**: db-f1-micro (1 vCPU, 1.7 GB RAM, 10 GB SSD) - ~$25/month
- **Production**: db-custom-4-16384 (4 vCPU, 16 GB RAM, 100 GB SSD) - ~$200/month
- **Enterprise**: db-custom-16-65536 (16 vCPU, 64 GB RAM, 500 GB SSD) - ~$800/month

---

### Option 2: MongoDB

**Technical Profile**:
- **Type**: Document database (NoSQL)
- **License**: SSPL (not fully open-source)
- **Strengths**: Flexible schema, horizontal scaling

**Pros**:
- ✅ **Flexible Schema**: No migrations, evolve schema dynamically
- ✅ **Horizontal Scaling**: Native sharding built-in
- ✅ **Document Model**: Natural fit for JSON-heavy workloads
- ✅ **Aggregation Pipeline**: Powerful data processing

**Cons**:
- ❌ **No Row-Level Security**: Must implement tenant isolation in application code (error-prone!)
- ❌ **Eventual Consistency**: Default consistency is eventual, not strong
- ❌ **ACID Limitations**: Multi-document transactions added only in v4.0 (2018)
- ❌ **Query Complexity**: Complex joins difficult (no native JOINs)
- ❌ **Licensing**: SSPL license restricts cloud providers

**Why Rejected**:
- **No RLS**: Tenant isolation must be implemented in application code
  - Every query must manually add `{organization_id: current_user.org_id}`
  - One forgotten filter = data leak across tenants
  - High security risk for multi-tenant SaaS
- **Eventual Consistency**: Risk of reading stale data during concurrent updates

---

### Option 3: FoundationDB

**Technical Profile**:
- **Type**: Distributed key-value store
- **License**: Open-source (Apache 2.0)
- **Strengths**: ACID at scale, multi-tenant key prefixing

**Note**: FoundationDB is already used in CODITECT v5 for session storage and real-time collaboration.

**Pros**:
- ✅ **ACID at Scale**: Distributed transactions with strong consistency
- ✅ **Multi-Tenant Native**: Key prefixing (`tenant-a/users/123`)
- ✅ **Horizontal Scaling**: Add nodes dynamically
- ✅ **Proven**: Used by Apple, Snowflake

**Cons**:
- ❌ **Operational Complexity**: Requires cluster management, monitoring
- ❌ **No Managed Service**: Self-host only (no Cloud SQL equivalent)
- ❌ **Limited Querying**: No SQL, no indexes (must build on top)
- ❌ **Learning Curve**: Unique architecture requires expertise
- ❌ **No Native JSON**: Must serialize/deserialize manually

**Why Rejected**:
- **Operational Burden**: No managed service, must run our own cluster
- **Limited Querying**: No SQL, complex queries require custom indexing
- **Already Using**: CODITECT v5 uses FoundationDB for sessions - don't duplicate for structured data
- **Best Fit**: Key-value workloads (sessions, cache), not structured relational data

---

### Option 4: MySQL

**Technical Profile**:
- **Type**: Relational database (RDBMS)
- **License**: Open-source (GPL) + Commercial
- **Strengths**: Widespread adoption, read performance

**Pros**:
- ✅ **Mature**: 28+ years of production use
- ✅ **Widespread**: Large community, many hosting options
- ✅ **Read Performance**: Optimized for read-heavy workloads

**Cons**:
- ⚠️ **Weaker JSON Support**: JSON type added later, not as optimized as PostgreSQL JSONB
- ⚠️ **No Row-Level Security**: Must implement tenant isolation in application
- ⚠️ **Full-Text Search**: Less powerful than PostgreSQL
- ⚠️ **Extensions**: Smaller extension ecosystem

**Why Rejected**:
- **No RLS**: Same tenant isolation problem as MongoDB
- **Weaker JSON**: JSONB in PostgreSQL is faster and more feature-rich
- **No Clear Advantage**: PostgreSQL offers more features with same operational complexity

---

## Decision Outcome

**Chosen Option**: **PostgreSQL** (Option 1)

### Rationale

1. **Row-Level Security is Critical**: Multi-tenant SaaS requires database-level isolation
   - PostgreSQL RLS = automatic tenant isolation
   - MongoDB/MySQL = manual application-level isolation (error-prone)

2. **ACID Guarantees**: Strong consistency required for audit trails, compliance
   - PostgreSQL = full ACID since day 1
   - MongoDB = eventual consistency by default

3. **Mature & Proven**: 35+ years of production use
   - Banks, governments, enterprises trust PostgreSQL
   - Battle-tested at massive scale (Uber, Instagram, Reddit)

4. **Cloud Managed Services**: Available everywhere
   - GCP Cloud SQL, AWS RDS, Azure Flexible Server
   - Automated backups, high availability, read replicas

5. **JSON Flexibility**: JSONB support without sacrificing relational model
   - Store flexible metadata in JSONB columns
   - Query JSON fields with indexes (GIN indexes)
   - Best of both worlds: structured + flexible

### Implementation Pattern

**Multi-Tenant Isolation via RLS**:
```sql
-- 1. Enable RLS on all tenant-scoped tables
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE checkpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- 2. Create RLS policies (automatic tenant filtering)
CREATE POLICY org_isolation_projects ON projects
    FOR ALL
    USING (
        organization_id IN (
            SELECT organization_id FROM organization_members
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );

-- 3. Application sets user context (FastAPI middleware)
@app.middleware("http")
async def set_user_context(request: Request, call_next):
    current_user = get_current_user(request)

    # Set PostgreSQL session variable
    async with db.begin():
        await db.execute(
            text(f"SET app.current_user_id = '{current_user.id}'")
        )

    response = await call_next(request)
    return response

-- 4. Queries automatically filtered by RLS
# Application code (no manual tenant filtering needed!)
checkpoints = db.query(Checkpoint).all()  # Automatically filtered to user's org!
```

---

## Consequences

### Positive Consequences ✅

1. **Automatic Tenant Isolation**:
   - RLS policies enforce tenant boundaries at database level
   - Zero risk of cross-tenant data leaks from application bugs
   - Compliance auditors can verify isolation in database schema

2. **Strong Consistency**:
   - ACID transactions guarantee data integrity
   - No eventual consistency edge cases
   - Audit trails are always accurate

3. **Mature Ecosystem**:
   - ORMs: SQLAlchemy (Python), TypeORM (TypeScript), Diesel (Rust)
   - Migration tools: Alembic, Flyway, Liquibase
   - Monitoring: pgAdmin, DataGrip, Metabase

4. **Cloud Native**:
   - Managed services handle backups, HA, scaling
   - No database operations expertise required
   - 99.95% uptime SLA

5. **Cost Efficiency**:
   - Single database serves 1000+ tenants
   - $200/month production instance handles 10,000+ users
   - 80% cheaper than database-per-tenant

6. **JSON Flexibility**:
   - Store metadata without schema migrations
   - Query JSON fields with GIN indexes
   - Gradual schema evolution

### Negative Consequences ⚠️

1. **Vertical Scaling Limits**:
   - Single-node write bottleneck (max ~10,000 writes/sec)
   - **Mitigation**: Read replicas for read scaling, Citus extension for sharding

2. **Schema Migration Overhead**:
   - Schema changes require migrations (downtime risk)
   - **Mitigation**: Use JSONB for flexible fields, blue-green deployments

3. **Operational Complexity** (Self-Hosted):
   - Database backups, monitoring, tuning required
   - **Mitigation**: Use managed service (Cloud SQL, RDS)

4. **Eventual Sharding Complexity**:
   - If single database exceeds limits (1M+ writes/sec)
   - **Mitigation**: Citus extension, or shard by organization_id

---

## Implementation Details

### Database Schema

```sql
-- Complete schema in ADR-001, key tables summarized here:

-- Organizations (tenants)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users (global, can belong to multiple orgs)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Organization membership (with roles)
CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL, -- 'owner', 'admin', 'member', 'viewer', 'auditor'
    UNIQUE(organization_id, user_id)
);

-- Projects (tenant-scoped)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    repository_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Row-Level Security
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation_projects ON projects
    FOR ALL
    USING (
        organization_id IN (
            SELECT organization_id FROM organization_members
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );
```

### Performance Optimization

```sql
-- Indexes for common queries
CREATE INDEX idx_checkpoints_project_date ON checkpoints(project_id, date DESC);
CREATE INDEX idx_messages_checkpoint ON messages(checkpoint_id);
CREATE INDEX idx_messages_hash ON messages(content_hash); -- Deduplication
CREATE INDEX idx_audit_log_org_created ON audit_log(organization_id, created_at DESC);

-- JSONB indexes (GIN)
CREATE INDEX idx_checkpoints_metadata ON checkpoints USING gin(metadata);

-- Full-text search (if not using ChromaDB)
CREATE INDEX idx_messages_fts ON messages USING gin(to_tsvector('english', content));

-- Partial indexes (more efficient)
CREATE INDEX idx_active_projects ON projects(organization_id)
    WHERE deleted_at IS NULL;
```

### Connection Pooling (FastAPI)

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Connection pool settings
engine = create_engine(
    DATABASE_URL,
    pool_size=20,          # Max connections per pod
    max_overflow=10,       # Extra connections during spikes
    pool_pre_ping=True,    # Verify connection before use
    pool_recycle=3600      # Recycle connections every hour
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

### Backup Strategy

**Managed Service (Cloud SQL)**:
- Automated daily backups (retained 7 days)
- Point-in-time recovery (PITR) - restore to any second in last 7 days
- Cross-region replication for disaster recovery

**Self-Hosted**:
```bash
# Daily backup cron job
0 2 * * * pg_dump -h localhost -U postgres -d coditect_db | gzip > /backups/coditect_$(date +\%Y\%m\%d).sql.gz

# Retention: Keep 30 daily, 12 monthly
find /backups -name "coditect_*.sql.gz" -mtime +30 -delete
```

---

## Validation and Compliance

### Success Criteria

**Functional Requirements**:
- ✅ RLS policies enforce tenant isolation (verified with test users)
- ✅ Sub-100ms query latency for 95th percentile
- ✅ Support 1000+ concurrent connections
- ✅ Zero data loss during transactions

**Non-Functional Requirements**:
- ✅ 99.95% uptime (Cloud SQL SLA)
- ✅ Automated backups every 24 hours
- ✅ Point-in-time recovery (PITR) available
- ✅ Read replicas for horizontal read scaling

### Testing Strategy

```python
# RLS testing
def test_tenant_isolation():
    """Verify user A cannot see user B's data."""
    # User A (Org 1)
    db.execute(text("SET app.current_user_id = '{user_a_id}'"))
    projects_a = db.query(Project).all()

    # User B (Org 2)
    db.execute(text("SET app.current_user_id = '{user_b_id}'"))
    projects_b = db.query(Project).all()

    # Verify isolation
    assert len(projects_a) > 0
    assert len(projects_b) > 0
    assert set(projects_a) & set(projects_b) == set()  # No overlap!


# Performance testing
def test_query_latency():
    """Verify timeline query completes in <100ms."""
    start = time.time()

    checkpoints = (
        db.query(Checkpoint)
        .filter(Checkpoint.project_id == project_id)
        .order_by(Checkpoint.date.desc())
        .limit(50)
        .all()
    )

    duration_ms = (time.time() - start) * 1000
    assert duration_ms < 100  # Sub-100ms
```

### Compliance Considerations

**SOC 2 Type II**:
- ✅ RLS policies documented in schema
- ✅ Audit logs track all data access
- ✅ Encrypted at rest (AES-256)
- ✅ Encrypted in transit (TLS 1.3)

**GDPR**:
- ✅ Right to delete: `DELETE FROM users WHERE id = '{user_id}'` (cascades)
- ✅ Data portability: `pg_dump` exports in standard format
- ✅ Audit trail: `audit_log` table tracks all access

---

## Related Decisions

- **ADR-001**: [Git as Source of Truth](ADR-001-git-as-source-of-truth.md)
- **ADR-003**: [ChromaDB for Semantic Search](ADR-003-chromadb-for-semantic-search.md)
- **ADR-004**: [Multi-Tenant Strategy](ADR-004-multi-tenant-strategy.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership, Security Team
**Review Date**: 2026-01-17 (60 days)

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2025-11-17 | Initial ADR | ADR Compliance Specialist |

---

**Made with ❤️ by CODITECT Engineering**
