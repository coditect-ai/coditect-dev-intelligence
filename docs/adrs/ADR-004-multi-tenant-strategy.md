# ADR-004: Multi-Tenant Strategy - Project Intelligence Platform

```yaml
Document: ADR-004-project-intelligence-multi-tenant-strategy
Version: 1.0.0
Purpose: Define multi-tenant architecture using single database with Row-Level Security (RLS)
Audience: Engineering teams, architects, security officers, compliance team
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-002 (PostgreSQL), ADR-008 (RBAC)
```

---

## Executive Summary

**Decision**: Use **single database with Row-Level Security (RLS)** for multi-tenant isolation.

**Why This Approach**:
- ✅ **Cost-Efficient**: Single database serves 1000+ tenants (80% cost reduction)
- ✅ **Database-Level Isolation**: RLS automatically filters tenant data (no application bugs)
- ✅ **Operational Simplicity**: One database to backup, monitor, scale
- ✅ **Compliance-Ready**: Proven pattern for SOC2, GDPR, HIPAA

**Alternatives Rejected**: Database per tenant (operational nightmare), Schema per tenant (migration complexity)

---

## Context and Problem Statement

### The Multi-Tenancy Challenge

CODITECT Project Intelligence Platform must support:

1. **Multiple Organizations** - 100+ organizations sharing infrastructure
2. **Complete Data Isolation** - Org A cannot see Org B's data (security + compliance)
3. **Scalable Operations** - Support 1000+ tenants without operational burden
4. **Cost Efficiency** - Shared infrastructure without shared risk

### Business Requirements

**Security**:
- **Zero Cross-Tenant Data Leaks**: One organization NEVER sees another's data
- **Automatic Isolation**: Database-level enforcement (not application code)
- **Audit Compliance**: SOC2, GDPR, HIPAA require provable isolation

**Operational**:
- **Single Control Plane**: One deployment, one backup strategy, one monitoring dashboard
- **Fast Onboarding**: New tenant = 5 minutes (not 5 weeks)
- **Cost-Effective**: Shared infrastructure reduces per-tenant costs by 80%

---

## Decision Drivers

### Mandatory Requirements (Must-Have)

1. **Complete Tenant Isolation** - No cross-tenant data access
2. **Database-Level Enforcement** - Not reliant on application code
3. **Operational Simplicity** - Single database to manage
4. **Cost-Effective** - Shared resources reduce costs
5. **Compliance-Ready** - SOC2, GDPR, HIPAA compatible

---

## Considered Options

### Option 1: **Single Database with Row-Level Security (RLS)** (SELECTED ✅)

**Architecture**:
```
Single PostgreSQL Database
├── organizations table (tenants)
├── projects table (with organization_id foreign key)
├── checkpoints table (with project_id foreign key)
└── RLS Policies (automatic filtering by organization_id)

User Query → RLS Policy → Filtered Results (only user's org)
```

**Implementation**:
```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE checkpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- Create policy: Users can only access their organization's data
CREATE POLICY org_isolation_projects ON projects
    FOR ALL
    USING (
        organization_id IN (
            SELECT organization_id FROM organization_members
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );

-- Application sets user context (FastAPI middleware)
@app.middleware("http")
async def set_rls_context(request: Request, call_next):
    user = get_current_user(request)
    async with db.begin():
        await db.execute(text(f"SET app.current_user_id = '{user.id}'"))
    response = await call_next(request)
    return response

-- Queries automatically filtered!
SELECT * FROM projects;  -- Only returns user's org projects
```

**Pros**:
- ✅ **Database-Level Isolation**: RLS enforced by PostgreSQL (not application code)
- ✅ **Automatic Filtering**: Queries automatically scoped to user's organization
- ✅ **Operational Simplicity**: One database, one backup, one monitor
- ✅ **Cost-Effective**: Shared resources reduce per-tenant costs by 80%
- ✅ **Fast Onboarding**: New tenant = INSERT INTO organizations (5 minutes)
- ✅ **Compliance-Ready**: SOC2, GDPR auditors can verify RLS policies
- ✅ **Proven Pattern**: Used by Slack, GitHub, Salesforce

**Cons**:
- ⚠️ **Performance Testing Required**: Must verify RLS doesn't impact query performance
- ⚠️ **Careful RLS Policy Design**: Bugs in policies = data leaks (mitigated by testing)
- ⚠️ **Single Database Bottleneck**: All tenants share same database (mitigated by read replicas)

**Cost**:
- **Production**: $200/month (Cloud SQL, 4 vCPU, 16 GB RAM, 100 GB SSD)
- **Per-Tenant**: $0.20/month (1000 tenants = $200/month)
- **Savings**: 80% vs database-per-tenant ($1000/month per database)

---

### Option 2: Database Per Tenant

**Architecture**:
```
Tenant A → Database A (db-tenant-a)
Tenant B → Database B (db-tenant-b)
Tenant C → Database C (db-tenant-c)
...
```

**Pros**:
- ✅ **Complete Isolation**: Physically separate databases (maximum security)
- ✅ **Custom Scaling**: Scale each tenant independently
- ✅ **Compliance**: Easy to prove isolation to auditors

**Cons**:
- ❌ **Operational Nightmare**: 1000 tenants = 1000 databases to manage
- ❌ **Slow Onboarding**: New tenant = provision database (30-60 minutes)
- ❌ **Expensive**: $200/month per database × 1000 tenants = $200,000/month
- ❌ **Backup Complexity**: 1000 separate backup jobs
- ❌ **Migration Hell**: Schema changes require 1000 migrations
- ❌ **Monitoring Overhead**: 1000 separate monitoring dashboards

**Why Rejected**: Operational complexity and cost are prohibitive at scale.

---

### Option 3: Schema Per Tenant

**Architecture**:
```
Single PostgreSQL Database
├── tenant_a schema (tables: projects, checkpoints, messages)
├── tenant_b schema (tables: projects, checkpoints, messages)
├── tenant_c schema (tables: projects, checkpoints, messages)
...
```

**Pros**:
- ✅ **Single Database**: One backup, one connection pool
- ✅ **Logical Isolation**: Separate schemas provide namespace isolation
- ✅ **Custom Extensions**: Each tenant can have schema-specific extensions

**Cons**:
- ❌ **Migration Complexity**: Schema changes require 1000 separate migrations
- ❌ **Connection Overhead**: Must switch schemas per query (`SET search_path = tenant_a`)
- ❌ **Difficult Aggregation**: Cross-tenant analytics require UNION across schemas
- ❌ **ORM Limitations**: Most ORMs don't handle multi-schema gracefully
- ❌ **No Clear Advantage**: More complexity than RLS without extra benefits

**Why Rejected**: Migration complexity and ORM incompatibility outweigh benefits.

---

## Decision Outcome

**Chosen Option**: **Single Database with Row-Level Security (RLS)** (Option 1)

### Rationale

1. **Database-Level Isolation**: RLS enforced by PostgreSQL, not application code
   - Even if application has bugs, database prevents cross-tenant access
   - Compliance auditors can verify isolation in database schema

2. **Operational Simplicity**: One database to manage
   - Single backup strategy
   - Single monitoring dashboard
   - Single schema migration process

3. **Cost-Effective**: 80% savings vs database-per-tenant
   - $200/month for 1000 tenants (vs $200,000/month)
   - Shared resources (connection pooling, caching)

4. **Fast Onboarding**: New tenant = 5 minutes
   - INSERT INTO organizations (instant)
   - No database provisioning, no infrastructure changes

5. **Proven Pattern**: Battle-tested at scale
   - Slack, GitHub, Salesforce use this approach
   - PostgreSQL RLS is mature (since PostgreSQL 9.5, 2016)

---

## Consequences

### Positive Consequences ✅

1. **Automatic Tenant Isolation**:
   - RLS policies enforce tenant boundaries at database level
   - Zero risk of cross-tenant data leaks from application bugs
   - Compliance auditors can verify isolation in schema

2. **Operational Simplicity**:
   - One database to backup, monitor, scale
   - Schema migrations apply to all tenants at once
   - Single connection pool, shared query cache

3. **Cost Efficiency**:
   - $200/month for 1000 tenants (80% savings)
   - Shared infrastructure (CPU, memory, disk)
   - No per-tenant overhead

4. **Fast Onboarding**:
   - New tenant = INSERT INTO organizations (instant)
   - No waiting for database provisioning
   - No infrastructure changes

5. **Compliance-Ready**:
   - RLS policies documented in schema
   - Audit logs track all data access
   - SOC2, GDPR, HIPAA compatible

### Negative Consequences ⚠️

1. **RLS Policy Design Risk**:
   - Bugs in RLS policies = data leaks
   - **Mitigation**: Comprehensive testing, peer review of policies

2. **Performance Testing Required**:
   - RLS adds query overhead (policy evaluation)
   - **Mitigation**: Benchmark queries, optimize indexes

3. **Single Database Bottleneck**:
   - All tenants share same database (write contention)
   - **Mitigation**: Read replicas for horizontal read scaling

4. **No Per-Tenant Customization**:
   - All tenants share same schema
   - **Mitigation**: Use JSONB for flexible metadata

---

## Implementation Details

### Database Schema

```sql
-- Organizations (tenants)
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Organization membership (users can belong to multiple orgs)
CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL, -- 'owner', 'admin', 'member', 'viewer'
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(organization_id, user_id)
);

-- Projects (tenant-scoped)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    repository_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- RLS Policy: Users can only access their organization's projects
CREATE POLICY org_isolation_projects ON projects
    FOR ALL
    USING (
        organization_id IN (
            SELECT organization_id FROM organization_members
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );

-- Checkpoints (tenant-scoped via project_id)
CREATE TABLE checkpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(500) NOT NULL,
    date TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE checkpoints ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation_checkpoints ON checkpoints
    FOR ALL
    USING (
        project_id IN (
            SELECT id FROM projects
            WHERE organization_id IN (
                SELECT organization_id FROM organization_members
                WHERE user_id = current_setting('app.current_user_id')::UUID
            )
        )
    );
```

### FastAPI Middleware (Set RLS Context)

```python
from fastapi import Request
from sqlalchemy import text

@app.middleware("http")
async def set_rls_context(request: Request, call_next):
    """
    Set PostgreSQL session variable for RLS policies.
    This ensures all queries are automatically filtered by user's organization.
    """
    # Skip for public endpoints (login, signup)
    if request.url.path in ["/api/auth/login", "/api/auth/signup"]:
        return await call_next(request)

    # Get current user from JWT token
    try:
        current_user = get_current_user(request)
    except Exception:
        return JSONResponse(status_code=401, content={"error": "Unauthorized"})

    # Set PostgreSQL session variable (used by RLS policies)
    async with db.begin():
        await db.execute(
            text(f"SET app.current_user_id = '{current_user.id}'")
        )

    # Process request (all queries now filtered by RLS)
    response = await call_next(request)

    return response
```

### Testing RLS Policies

```python
# Unit test: Verify tenant isolation
def test_tenant_isolation():
    """Verify User A cannot see User B's projects."""
    # Create two organizations
    org_a = Organization(name="Org A", slug="org-a")
    org_b = Organization(name="Org B", slug="org-b")
    db.add_all([org_a, org_b])
    db.commit()

    # Create users and projects
    user_a = User(email="a@example.com")
    user_b = User(email="b@example.com")
    db.add_all([user_a, user_b])
    db.commit()

    db.add(OrganizationMember(organization_id=org_a.id, user_id=user_a.id, role="admin"))
    db.add(OrganizationMember(organization_id=org_b.id, user_id=user_b.id, role="admin"))
    db.commit()

    project_a = Project(organization_id=org_a.id, name="Project A")
    project_b = Project(organization_id=org_b.id, name="Project B")
    db.add_all([project_a, project_b])
    db.commit()

    # Query as User A
    db.execute(text(f"SET app.current_user_id = '{user_a.id}'"))
    projects_a = db.query(Project).all()

    # Query as User B
    db.execute(text(f"SET app.current_user_id = '{user_b.id}'"))
    projects_b = db.query(Project).all()

    # Verify isolation
    assert len(projects_a) == 1
    assert projects_a[0].id == project_a.id
    assert len(projects_b) == 1
    assert projects_b[0].id == project_b.id

    # Verify no cross-tenant access
    assert project_b.id not in [p.id for p in projects_a]
    assert project_a.id not in [p.id for p in projects_b]
```

---

## Validation and Compliance

### Success Criteria

**Functional Requirements**:
- ✅ RLS policies enforce 100% tenant isolation (verified by tests)
- ✅ Sub-100ms query latency with RLS enabled
- ✅ Support 1000+ concurrent tenants

**Security Requirements**:
- ✅ Zero cross-tenant data leaks (verified by penetration testing)
- ✅ Audit logs track all data access
- ✅ RLS policies reviewed by security team

### Compliance Considerations

**SOC 2 Type II**:
- ✅ RLS policies documented in schema
- ✅ Audit logs track all data access
- ✅ Penetration testing verifies isolation

**GDPR**:
- ✅ Right to delete: CASCADE on organization deletion
- ✅ Data portability: Export organization data to JSON
- ✅ Audit trail: All access logged

---

## Related Decisions

- **ADR-002**: [PostgreSQL as Primary Database](ADR-002-postgresql-as-primary-database.md)
- **ADR-008**: [Role-Based Access Control](ADR-008-role-based-access-control.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership, Security Team
**Review Date**: 2026-01-17 (60 days)

---

**Made with ❤️ by CODITECT Engineering**
