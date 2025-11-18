# CODITECT Project Intelligence Platform - Claude Code Configuration

## Project Overview

**CODITECT Project Intelligence Platform** is a multi-tenant SaaS platform for project visibility, reporting, searching, integrity, and auditability. This repository contains the implementation of the complete system documented during Phase 1 (Architecture & Design).

### Purpose
Transform git repositories (checkpoints, conversations, tasks) into an interactive, searchable, AI-powered intelligence platform with enterprise-grade security and multi-tenant isolation.

### Essential Reading
Before working on this project, read these documents IN ORDER:

1. **[EXECUTIVE-SUMMARY-PROJECT-INTELLIGENCE.md](docs/EXECUTIVE-SUMMARY-PROJECT-INTELLIGENCE.md)** - Business case, ROI, target users (15 KB)
2. **[SOFTWARE-DESIGN-DOCUMENT-PROJECT-INTELLIGENCE.md](docs/SOFTWARE-DESIGN-DOCUMENT-PROJECT-INTELLIGENCE.md)** - Complete technical specification (63 KB, 67,000 words)
3. **[C4-DIAGRAMS-PROJECT-INTELLIGENCE.md](docs/C4-DIAGRAMS-PROJECT-INTELLIGENCE.md)** - System architecture diagrams (43 KB, 9 diagrams)
4. **[DATABASE-ARCHITECTURE-PROJECT-INTELLIGENCE.md](docs/DATABASE-ARCHITECTURE-PROJECT-INTELLIGENCE.md)** - Multi-database design (29 KB)
5. **[docs/adrs/README.md](docs/adrs/README.md)** - Architecture Decision Records index (8 ADRs, 40/40 quality)

---

## Current Status

**Phase 1: Architecture & Design** - ✅ **COMPLETE** (Weeks 1-2)

**Deliverables Created**:
- ✅ Software Design Document (67,000 words)
- ✅ C4 Architecture Diagrams (9 diagrams in GitHub-compatible Mermaid)
- ✅ Architecture Decision Records (8 ADRs with 40/40 quality scores)
- ✅ Database schema (PostgreSQL + ChromaDB + Redis)
- ✅ Executive summary with complete business case
- ✅ Foundation proof-of-concept (1,601 messages, interactive timeline)

**Phase 2: Backend Implementation** - ⏸️ **READY TO START** (Weeks 3-8)

**What to build next**:
- FastAPI backend with async processing
- Git ingestion pipeline (processes 1,601 messages in <60s)
- Semantic search engine (ChromaDB vector embeddings)
- Timeline & reporting APIs
- Multi-tenant security (RLS + RBAC + audit logging)
- Test suite (80% coverage, 100 concurrent users)

---

## Architecture Overview

### System Design (C4 Level 1: Context)

```
┌─────────────────────────────────────────────────────────────┐
│                     Users (1000+)                           │
│  - Executive Leadership (CEO, CTO, CFO)                     │
│  - Senior Leadership (Engineering Managers, PMs)            │
│  - Auditors (Internal/External Compliance)                  │
│  - Team Members (Software Engineers)                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│       CODITECT Project Intelligence Platform (SaaS)         │
│                                                             │
│  Core Features:                                             │
│  • Visibility: Executive dashboards, real-time metrics      │
│  • Reporting: Automated reports, audit trail               │
│  • Searching: Semantic search (AI-powered)                 │
│  • Integrity: Git-first architecture, hash verification    │
│  • Auditability: Complete audit log, RBAC (6 roles)        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
       ┌──────────────────────────────┐
       │    Git Repositories          │
       │  (Source of Truth)           │
       │  - Checkpoints/*.md          │
       │  - unique_messages.jsonl     │
       │  - checkpoint_index.json     │
       └──────────────────────────────┘
```

### Technology Stack (C4 Level 2: Containers)

**Backend:**
- **FastAPI** (Python 3.11+) - Async web framework
- **PostgreSQL 15** - Multi-tenant database with row-level security (RLS)
- **ChromaDB** - Vector database for semantic search
- **Redis** - Caching layer (sub-millisecond)
- **Celery** - Background job processing

**Frontend:**
- **Next.js 14** - React framework with SSR
- **TypeScript** - Type-safe JavaScript
- **Chakra UI** - Accessible component library
- **React Query** - Data fetching and caching

**Infrastructure:**
- **GCP Cloud Run** - Serverless containers (auto-scaling)
- **Cloud SQL** - Managed PostgreSQL (HA configuration)
- **Cloud Storage** - S3-compatible object storage
- **Cloud Memorystore** - Managed Redis
- **Cloud Load Balancer** - SSL termination

---

## Key Architecture Decisions (ADRs)

### ADR-001: Git as Source of Truth
**Decision**: Database is a derived view of git repositories. Git = canonical source, database = performance cache.

**Why**:
- Complete trust and verification
- Disaster recovery (recreate DB from git)
- Audit trail (every record traces to git commit SHA)

**Implementation**: Every database record includes `git_commit_sha` for verification.

### ADR-002: PostgreSQL as Primary Database
**Decision**: PostgreSQL 15+ with Row-Level Security (RLS) for multi-tenant isolation.

**Why**:
- Database-level tenant isolation (not application code)
- 80% cost savings vs database-per-tenant
- Zero cross-tenant data leaks

**Implementation**: RLS policies on all tenant-scoped tables.

### ADR-003: ChromaDB for Semantic Search
**Decision**: ChromaDB vector embeddings for AI-powered semantic search.

**Why**:
- Find conversations without exact keywords
- 90% faster context discovery
- OpenAI text-embedding-3-small (1536 dimensions)

**Implementation**: Hybrid search (ChromaDB + PostgreSQL full-text) with Reciprocal Rank Fusion.

### ADR-004: Multi-Tenant Strategy
**Decision**: Single database with RLS, not database-per-tenant.

**Why**:
- 80% cost reduction
- Simpler operations (one connection pool)
- Database enforces isolation (no application bugs)

**Implementation**: `organization_id` in all tables, RLS policies enforce scoping.

### ADR-005: FastAPI Backend
**Decision**: FastAPI over Flask/Django for async-first architecture.

**Why**:
- Non-blocking I/O (1000+ concurrent users)
- Automatic OpenAPI docs
- Type safety with Pydantic

**Implementation**: Python 3.11+ with async/await throughout.

### ADR-006: React + Next.js Frontend
**Decision**: Next.js 14 with server-side rendering.

**Why**:
- SEO optimization (server-side rendering)
- API routes (backend-for-frontend pattern)
- TypeScript type safety

**Implementation**: App Router with React Server Components.

### ADR-007: GCP Cloud Run Deployment
**Decision**: Deploy on Google Cloud Platform using Cloud Run.

**Why**:
- Serverless auto-scaling (0 to 1000+ instances)
- Pay-per-use (no idle costs)
- Fast deployment (<5 minutes)

**Implementation**: Containerized apps deployed via `gcloud run deploy`.

### ADR-008: Role-Based Access Control (RBAC)
**Decision**: 6 roles with granular permissions.

**Roles**: Owner, Admin, Member, Viewer, Auditor, Executive

**Why**:
- Enterprise compliance (SOC2, GDPR, HIPAA)
- Fine-grained access control
- Audit-friendly (track who did what)

**Implementation**: Permission decorator `@require_permission("projects:delete")`.

---

## Database Schema (PostgreSQL)

### Core Tables

```sql
-- Multi-tenant isolation
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    user_id UUID REFERENCES users(id),
    role VARCHAR(50) NOT NULL, -- owner, admin, member, viewer, auditor, executive
    UNIQUE(organization_id, user_id)
);

-- Project data (tenant-scoped with RLS)
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    git_url TEXT,
    git_commit_sha VARCHAR(64), -- Verification
    created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation_projects ON projects
    FOR ALL
    USING (
        organization_id IN (
            SELECT organization_id FROM organization_members
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );

-- Messages from git (deduplicated)
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    project_id UUID REFERENCES projects(id),
    checkpoint_id UUID REFERENCES checkpoints(id),
    content TEXT NOT NULL,
    content_hash VARCHAR(64) UNIQUE NOT NULL, -- SHA-256
    role VARCHAR(50), -- user, assistant, system
    source_type VARCHAR(100), -- export_compact_bullet, checkpoint_section, etc.
    git_commit_sha VARCHAR(64), -- Trace to git
    created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- Audit log (append-only)
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    user_id UUID REFERENCES users(id),
    action VARCHAR(100) NOT NULL, -- 'projects:delete', 'members:invite'
    resource_type VARCHAR(50),
    resource_id UUID,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_log_org_created ON audit_log(organization_id, created_at DESC);
```

---

## API Endpoints (FastAPI)

### Authentication
- `POST /api/auth/login` - OAuth2 login
- `POST /api/auth/logout` - Logout
- `GET /api/auth/me` - Current user info

### Projects
- `GET /api/projects` - List projects (RLS enforced)
- `POST /api/projects` - Create project
- `GET /api/projects/{id}` - Get project details
- `PUT /api/projects/{id}` - Update project
- `DELETE /api/projects/{id}` - Delete project (requires owner/admin)

### Checkpoints
- `GET /api/projects/{id}/checkpoints` - List checkpoints
- `GET /api/checkpoints/{id}` - Get checkpoint details
- `GET /api/checkpoints/{id}/messages` - Get checkpoint messages

### Search
- `POST /api/search/full-text` - PostgreSQL full-text search
- `POST /api/search/semantic` - ChromaDB semantic search
- `POST /api/search/hybrid` - Hybrid search (full-text + semantic)

### Timeline
- `GET /api/timeline` - Get timeline data (calendar view)
- `GET /api/timeline/tasks` - Get completed tasks

### Analytics (Executive Dashboard)
- `GET /api/analytics/metrics` - Real-time metrics
- `GET /api/analytics/activity` - Activity heatmap
- `GET /api/analytics/focus-areas` - Focus area breakdown

### Audit
- `GET /api/audit-logs` - Get audit logs (requires auditor/executive)
- `POST /api/audit-logs/export` - Export audit logs (CSV/JSON)

---

## Development Workflow

### Phase 2: Backend Implementation (Weeks 3-8)

**Week 3-4: Core Backend**
1. FastAPI project setup (PostgreSQL + SQLAlchemy)
2. Authentication (OAuth2 + JWT)
3. Multi-tenant RLS implementation
4. CRUD APIs for projects, checkpoints, messages

**Week 5-6: Search & Ingestion**
1. Git ingestion pipeline (parse unique_messages.jsonl, checkpoint_index.json)
2. Full-text search (PostgreSQL tsvector)
3. Semantic search (ChromaDB integration)
4. Hybrid search (Reciprocal Rank Fusion)

**Week 7-8: Security & Testing**
1. RBAC implementation (6 roles)
2. Audit logging
3. Unit tests (80% coverage)
4. Integration tests (GitHub webhook sync)
5. Load testing (100 concurrent users)

### Phase 3: Frontend & Deployment (Weeks 9-12)

**Week 9-10: Frontend**
1. Next.js project setup (TypeScript + Chakra UI)
2. Timeline visualization (calendar view)
3. Search interface (full-text, semantic, hybrid)
4. Executive dashboard (metrics, activity heatmap)

**Week 11-12: Deployment**
1. Terraform for GCP infrastructure
2. Cloud Run deployment (backend + frontend)
3. CI/CD pipeline (GitHub Actions)
4. E2E testing (Playwright)
5. Production launch

---

## Code Patterns

### FastAPI Permission Decorator

```python
from functools import wraps
from fastapi import HTTPException, Depends

PERMISSIONS = {
    "owner": ["*"],  # All permissions
    "admin": ["projects:*", "checkpoints:*", "members:*", "audit_logs:read"],
    "member": ["projects:read", "projects:update", "checkpoints:*"],
    "viewer": ["projects:read", "checkpoints:read", "messages:read"],
    "auditor": ["projects:read", "audit_logs:*"],
    "executive": ["projects:read", "analytics:*", "reports:*"]
}

def require_permission(permission: str):
    """Decorator to enforce permission-based access control."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            org_id = kwargs.get('organization_id')
            member = db.query(OrganizationMember).filter_by(
                user_id=current_user.id,
                organization_id=org_id
            ).first()

            if not member:
                raise HTTPException(403, "Not a member of this organization")

            # Check permission
            user_permissions = PERMISSIONS[member.role]
            if permission not in user_permissions and "*" not in user_permissions:
                raise HTTPException(403, f"Missing permission: {permission}")

            # Log audit event
            db.add(AuditLog(
                organization_id=org_id,
                user_id=current_user.id,
                action=permission,
                resource_type=kwargs.get('resource_type'),
                resource_id=kwargs.get('resource_id')
            ))
            db.commit()

            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator

# Usage
@app.delete("/api/projects/{project_id}")
@require_permission("projects:delete")
async def delete_project(project_id: UUID, current_user: User):
    project = db.query(Project).filter_by(id=project_id).first()
    db.delete(project)
    db.commit()
    return {"status": "deleted"}
```

### Git Sync with Hash Verification

```python
import hashlib
import subprocess
from pathlib import Path

async def sync_from_git(repo_url: str, ref: str):
    """Clone/pull repository and sync database with git state."""
    # 1. Clone or pull repository
    repo_dir = f"/tmp/repos/{hashlib.md5(repo_url.encode()).hexdigest()}"
    if not os.path.exists(repo_dir):
        subprocess.run(["git", "clone", repo_url, repo_dir])
    else:
        subprocess.run(["git", "-C", repo_dir, "pull", "origin", ref])

    # 2. Load data from git
    dedup_state_dir = Path(repo_dir) / "MEMORY-CONTEXT" / "dedup_state"
    unique_messages = load_jsonl(dedup_state_dir / "unique_messages.jsonl")
    checkpoint_index = load_json(dedup_state_dir / "checkpoint_index.json")

    # 3. Get git commit SHA
    git_commit_sha = subprocess.check_output(
        ["git", "-C", repo_dir, "rev-parse", "HEAD"]
    ).decode().strip()

    # 4. Sync to database (idempotent upsert)
    for msg_data in unique_messages:
        content_hash = msg_data["hash"]

        # Check if already exists
        existing = db.query(Message).filter_by(content_hash=content_hash).first()
        if not existing:
            message = Message(
                organization_id=org_id,
                project_id=project_id,
                content=msg_data["message"]["content"],
                content_hash=content_hash,
                role=msg_data["message"]["role"],
                source_type=msg_data["message"]["source_type"],
                git_commit_sha=git_commit_sha  # Track git commit
            )
            db.add(message)

    db.commit()

    # 5. Generate embeddings for new messages (background job)
    celery_app.send_task("generate_embeddings", args=[project_id])
```

### Semantic Search (ChromaDB)

```python
from chromadb import Client
from openai import OpenAI

openai_client = OpenAI()
chroma_client = Client()

async def semantic_search(query: str, organization_id: UUID, limit: int = 10):
    """AI-powered semantic search across conversations."""
    # 1. Generate query embedding
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    query_embedding = response.data[0].embedding

    # 2. Search ChromaDB with organization filtering
    collection = chroma_client.get_collection("messages")
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=limit,
        where={"organization_id": str(organization_id)}  # Tenant isolation
    )

    # 3. Return results with metadata
    return [
        {
            "message_id": results["ids"][0][i],
            "content": results["documents"][0][i],
            "similarity_score": 1 - results["distances"][0][i],  # Convert distance to similarity
            "metadata": results["metadatas"][0][i]
        }
        for i in range(len(results["ids"][0]))
    ]
```

---

## Testing Strategy

### Unit Tests (80% coverage target)

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def db_session():
    """Create test database session."""
    engine = create_engine("postgresql://localhost/test_project_intelligence")
    Session = sessionmaker(bind=engine)
    session = Session()

    # Create tables
    Base.metadata.create_all(engine)

    yield session

    # Cleanup
    session.close()
    Base.metadata.drop_all(engine)

def test_rls_tenant_isolation(db_session):
    """Test that RLS prevents cross-tenant data access."""
    # Create two organizations
    org1 = Organization(name="Org 1")
    org2 = Organization(name="Org 2")
    db_session.add_all([org1, org2])
    db_session.commit()

    # Create projects for each org
    project1 = Project(organization_id=org1.id, name="Project 1")
    project2 = Project(organization_id=org2.id, name="Project 2")
    db_session.add_all([project1, project2])
    db_session.commit()

    # Set current user to org1 member
    db_session.execute("SET app.current_user_id = :user_id", {"user_id": user1.id})

    # Should only see org1 projects
    projects = db_session.query(Project).all()
    assert len(projects) == 1
    assert projects[0].organization_id == org1.id
```

### Integration Tests

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_github_webhook_sync():
    """Test that GitHub webhook triggers git sync."""
    # 1. Send webhook payload
    response = client.post("/api/webhooks/github", json={
        "repository": {"clone_url": "https://github.com/test/repo.git"},
        "ref": "refs/heads/main"
    })
    assert response.status_code == 200

    # 2. Wait for background job
    time.sleep(5)

    # 3. Verify messages synced to database
    messages = client.get("/api/messages").json()
    assert len(messages) > 0
    assert messages[0]["git_commit_sha"] is not None
```

---

## Performance Targets

### API Latency
- **GET /api/checkpoints**: p50=20ms, p95=50ms, p99=100ms
- **POST /api/search/semantic**: p50=100ms, p95=300ms, p99=500ms
- **POST /api/webhooks/github**: p50=50ms, p95=100ms, p99=200ms

### Database Queries
- **Timeline (50 checkpoints)**: <50ms (with RLS enabled)
- **Semantic search (10 results)**: <300ms (ChromaDB + metadata filtering)
- **Audit log export (10,000 records)**: <5s

### Sync Performance
- **GitHub webhook → Sync start**: <5s (background job queued)
- **Sync 1,601 messages**: <60s (upsert to PostgreSQL + ChromaDB)
- **Embedding generation**: <5s/message (OpenAI API batching)

---

## Security Considerations

### Multi-Tenant Isolation
- **Database-level enforcement**: PostgreSQL RLS policies (not application code)
- **Automatic tenant scoping**: `current_setting('app.current_user_id')`
- **Zero cross-tenant leaks**: 100% verified by unit tests

### Authentication & Authorization
- **OAuth2 + JWT**: Secure token-based authentication
- **RBAC (6 roles)**: Owner, Admin, Member, Viewer, Auditor, Executive
- **Permission enforcement**: Decorator `@require_permission("projects:delete")`

### Audit Trail
- **Complete logging**: Every data access logged (user, timestamp, action)
- **Immutable logs**: Append-only table (cannot be modified)
- **Compliance exports**: CSV/JSON for SOC2, GDPR audits

### Data Integrity
- **Git verification**: Every record traces to git commit SHA
- **Hash verification**: SHA-256 content hashing
- **Disaster recovery**: Recreate entire database from git

---

## Deployment Checklist

### Pre-Deployment
- [ ] Review all ADRs with engineering team
- [ ] Security review (penetration testing)
- [ ] Performance benchmarking (load testing)
- [ ] Database migration scripts tested
- [ ] Backup strategy validated
- [ ] Monitoring dashboards configured (Prometheus, Grafana)
- [ ] Incident response plan documented

### Deployment (GCP Cloud Run)
- [ ] Provision GCP infrastructure (Cloud Run, Cloud SQL)
- [ ] Deploy PostgreSQL schema (with RLS policies)
- [ ] Deploy ChromaDB container (Cloud Run)
- [ ] Deploy FastAPI backend (Cloud Run)
- [ ] Deploy Next.js frontend (Cloud Run)
- [ ] Configure GitHub webhooks
- [ ] Run initial sync (git → database)
- [ ] Verify RLS policies (tenant isolation)
- [ ] Load test (1000 concurrent users)

### Post-Deployment
- [ ] Monitor error rates (target: <1%)
- [ ] Monitor latency (p95 <100ms)
- [ ] Verify git sync (webhook processing)
- [ ] Review audit logs (security monitoring)
- [ ] Customer feedback (beta testing)

---

## AI Agent Guidelines

### When Working in This Repository

1. **Always Read Documentation First**
   - Start with `docs/EXECUTIVE-SUMMARY-PROJECT-INTELLIGENCE.md`
   - Then read `docs/SOFTWARE-DESIGN-DOCUMENT-PROJECT-INTELLIGENCE.md`
   - Review relevant ADRs in `docs/adrs/`

2. **Follow Architecture Decisions**
   - Git-first architecture (database is derived view)
   - Multi-tenant with RLS (not database-per-tenant)
   - Async-first (FastAPI async/await)
   - Type safety (Pydantic models, TypeScript)

3. **Implement Security by Default**
   - Always use RLS policies on tenant-scoped tables
   - Always use `@require_permission()` decorator for protected endpoints
   - Always log to audit_log table
   - Never skip authentication checks

4. **Write Tests First (TDD)**
   - Unit tests for business logic (80% coverage target)
   - Integration tests for API endpoints
   - Security tests for RLS tenant isolation
   - Performance tests for latency targets

5. **Maintain Documentation**
   - Update ADRs if architecture changes
   - Update API.md when adding endpoints
   - Update CLAUDE.md with new patterns
   - Keep README.md current with project status

---

## Common Tasks

### Add New API Endpoint

```python
# 1. Define Pydantic model
class ProjectCreate(BaseModel):
    name: str
    git_url: str

# 2. Add endpoint with RBAC
@app.post("/api/projects")
@require_permission("projects:create")
async def create_project(
    project: ProjectCreate,
    current_user: User = Depends(get_current_user)
):
    new_project = Project(
        organization_id=current_user.organization_id,
        name=project.name,
        git_url=project.git_url
    )
    db.add(new_project)
    db.commit()
    return new_project

# 3. Write test
def test_create_project():
    response = client.post("/api/projects", json={
        "name": "Test Project",
        "git_url": "https://github.com/test/repo.git"
    })
    assert response.status_code == 200
    assert response.json()["name"] == "Test Project"
```

### Add Database Migration

```bash
# 1. Generate migration
alembic revision --autogenerate -m "Add new_column to projects"

# 2. Edit migration file (add RLS policy if needed)
# migrations/versions/xxx_add_new_column.py

# 3. Apply migration
alembic upgrade head

# 4. Test rollback
alembic downgrade -1
alembic upgrade head
```

### Deploy to GCP

```bash
# 1. Build and deploy backend
cd backend
gcloud run deploy coditect-api \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 100

# 2. Build and deploy frontend
cd frontend
gcloud run deploy coditect-frontend \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 50
```

---

## Related Repositories

- **[Master Repo](https://github.com/coditect-ai/coditect-rollout-master)** - Orchestration repository
- **[CODITECT Framework](https://github.com/coditect-ai/coditect-project-dot-claude)** - Core framework (.coditect/)
- **[Context API](https://github.com/coditect-ai/coditect-context-api)** - Local developer productivity tool

---

## Support & Contact

- **Documentation**: See `docs/` directory
- **Issues**: [GitHub Issues](https://github.com/coditect-ai/coditect-project-intelligence/issues)
- **Project Lead**: TBD
- **Technical Lead**: TBD

---

**Status**: ✅ **ARCHITECTURE COMPLETE - READY FOR PHASE 2**
**Current Phase**: 1 of 3 Complete
**Next Milestone**: Backend Implementation (Weeks 3-8)
**Recommendation**: **STRONG GO** (95% confidence)
**Last Updated**: November 18, 2025
**Repository**: https://github.com/coditect-ai/coditect-project-intelligence
**Owner**: AZ1.AI INC
