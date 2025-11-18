# Database Architecture for Cloud-Based Project Intelligence Platform

**Status**: Design Proposal
**Purpose**: Multi-tenant project intelligence with role-based access control
**Target Users**: Executive Leadership, Senior Leadership, Auditors, Team Members
**Created**: 2025-11-17

---

## Executive Summary

**Ideal Architecture**: **Multi-Database Hybrid System**

```
PostgreSQL (Structured Data) + ChromaDB (Semantic Search) + Redis (Sessions) + S3 (Files)
```

**Why This Stack:**
- ✅ **PostgreSQL**: Industry standard for multi-tenant, ACID-compliant structured data
- ✅ **ChromaDB**: Purpose-built for AI-powered semantic search on conversations
- ✅ **Redis**: Sub-millisecond session management and caching
- ✅ **S3/GCS**: Infinite scalability for export files and attachments

**Key Features:**
- 🔐 Row-level security (RLS) for multi-tenant isolation
- 👥 Role-based access control (RBAC) with granular permissions
- 🔍 Semantic search across 1,601+ messages
- 📊 Real-time analytics dashboard
- 🌐 Cloud-native (GCP/AWS/Azure)
- ♾️ Horizontally scalable

---

## Database Comparison

| Database | Use Case | Pros | Cons | Verdict |
|----------|----------|------|------|---------|
| **PostgreSQL** | Primary structured data | ACID, RLS, JSON support, mature | Vertical scaling limits | ✅ **PRIMARY** |
| **ChromaDB** | Semantic search | AI-native, embeddings, vector search | No RBAC built-in | ✅ **SECONDARY** |
| **Redis** | Session + cache | Sub-ms latency, simple | Volatile (need persistence) | ✅ **CACHE LAYER** |
| **MongoDB** | Document store | Flexible schema, horizontal scale | No RLS, eventual consistency | ⚠️ Consider for logs |
| **ClickHouse** | Analytics | Column-store, petabyte scale | No transactions | ⚠️ Consider for metrics |
| **FoundationDB** | Distributed KV | Multi-tenant, ACID, scalable | Complex operations | ⚠️ Already in use for sessions |

---

## Recommended Architecture

### 1. **PostgreSQL** (Primary Database)

**Purpose**: Source of truth for all structured data

**Schema Design:**

```sql
-- Multi-tenancy with Row-Level Security
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255),
    password_hash TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL, -- 'owner', 'admin', 'member', 'viewer', 'auditor'
    permissions JSONB DEFAULT '{}', -- Granular permissions
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(organization_id, user_id)
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    repository_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE checkpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    name VARCHAR(500) NOT NULL,
    checkpoint_type VARCHAR(50), -- 'checkpoint', 'export', 'milestone'
    date TIMESTAMPTZ NOT NULL,
    source_file TEXT,
    phase VARCHAR(100),
    focus_area VARCHAR(100),
    message_count INTEGER DEFAULT 0,
    task_count INTEGER DEFAULT 0,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checkpoint_id UUID REFERENCES checkpoints(id) ON DELETE CASCADE,
    content_hash VARCHAR(64) UNIQUE NOT NULL, -- SHA-256
    role VARCHAR(50), -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    source_type VARCHAR(50), -- 'export_standard', 'export_compact_circle', etc.
    timestamp TIMESTAMPTZ,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checkpoint_id UUID REFERENCES checkpoints(id) ON DELETE CASCADE,
    text TEXT NOT NULL,
    status VARCHAR(50) DEFAULT 'completed', -- 'completed', 'pending', 'in_progress'
    task_type VARCHAR(50), -- 'checkbox', 'section', 'milestone'
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL, -- 'view_checkpoint', 'export_data', 'invite_user'
    resource_type VARCHAR(50), -- 'checkpoint', 'message', 'project'
    resource_id UUID,
    ip_address INET,
    user_agent TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Row-Level Security Policies
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE checkpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only access their organization's data
CREATE POLICY org_isolation_projects ON projects
    FOR ALL
    USING (
        organization_id IN (
            SELECT organization_id FROM organization_members
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );

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

-- Indexes for performance
CREATE INDEX idx_checkpoints_project_date ON checkpoints(project_id, date DESC);
CREATE INDEX idx_messages_checkpoint ON messages(checkpoint_id);
CREATE INDEX idx_messages_hash ON messages(content_hash);
CREATE INDEX idx_tasks_checkpoint ON tasks(checkpoint_id);
CREATE INDEX idx_audit_log_org_created ON audit_log(organization_id, created_at DESC);
```

**Benefits:**
- ✅ **Row-Level Security**: Automatic tenant isolation
- ✅ **ACID Transactions**: Data consistency guaranteed
- ✅ **JSON Support**: Flexible metadata storage
- ✅ **Mature Ecosystem**: Extensive tooling and ORMs
- ✅ **Audit Trail**: Complete compliance with `audit_log` table

---

### 2. **ChromaDB** (Semantic Search Engine)

**Purpose**: AI-powered semantic search across messages and tasks

**Architecture:**

```python
# Collection structure
collections = {
    "messages": {
        "documents": ["message content"],
        "embeddings": [vector_embeddings],  # Generated via OpenAI/Anthropic
        "metadatas": [{
            "checkpoint_id": "uuid",
            "organization_id": "uuid",
            "project_id": "uuid",
            "date": "2025-11-17",
            "focus_area": "Backend",
            "phase": "Week 1"
        }],
        "ids": ["message_uuid"]
    },
    "tasks": {
        "documents": ["task text"],
        "embeddings": [vector_embeddings],
        "metadatas": [{
            "checkpoint_id": "uuid",
            "organization_id": "uuid",
            "completed_at": "2025-11-17"
        }],
        "ids": ["task_uuid"]
    }
}
```

**Queries:**

```python
# Semantic search example
results = collection.query(
    query_texts=["backend API authentication issues"],
    n_results=10,
    where={"organization_id": "org-uuid"},  # Tenant filtering
    where_document={"$contains": "authentication"}
)

# Multi-vector search
results = collection.query(
    query_texts=["database schema design"],
    n_results=5,
    where={
        "organization_id": "org-uuid",
        "focus_area": {"$in": ["Database", "Backend"]}
    }
)
```

**Integration with PostgreSQL:**

```python
# Sync pipeline (on message insert)
def sync_message_to_chromadb(message_id: UUID):
    # 1. Get message from PostgreSQL
    message = db.query(Message).filter_by(id=message_id).first()

    # 2. Generate embedding
    embedding = openai.Embedding.create(
        input=message.content,
        model="text-embedding-ada-002"
    )

    # 3. Add to ChromaDB
    collection.add(
        documents=[message.content],
        embeddings=[embedding["data"][0]["embedding"]],
        metadatas=[{
            "checkpoint_id": str(message.checkpoint_id),
            "organization_id": str(message.checkpoint.project.organization_id),
            "date": message.timestamp.isoformat()
        }],
        ids=[str(message.id)]
    )
```

**Benefits:**
- ✅ **Semantic Search**: Find relevant messages without exact keywords
- ✅ **Vector Similarity**: Discover related conversations automatically
- ✅ **Fast**: Optimized for high-dimensional vector operations
- ✅ **AI-Native**: Built for LLM applications

---

### 3. **Redis** (Session Store + Cache)

**Purpose**: High-performance session management and caching

**Use Cases:**

```python
# Session management
redis.setex(
    f"session:{session_id}",
    3600,  # 1 hour TTL
    json.dumps({
        "user_id": "uuid",
        "organization_id": "uuid",
        "role": "admin",
        "permissions": ["read", "write", "delete"]
    })
)

# Query result caching
cache_key = f"timeline:{project_id}:{year}:{month}"
cached = redis.get(cache_key)
if cached:
    return json.loads(cached)
else:
    timeline = db.query(...).all()
    redis.setex(cache_key, 300, json.dumps(timeline))  # 5 min cache
    return timeline

# Real-time counters
redis.incr(f"checkpoint:{checkpoint_id}:views")
redis.zadd("trending_checkpoints", {checkpoint_id: timestamp})
```

**Benefits:**
- ✅ **Sub-millisecond latency**
- ✅ **Automatic expiration** (TTL)
- ✅ **Atomic operations**
- ✅ **Pub/Sub** for real-time updates

---

### 4. **S3/GCS** (Object Storage)

**Purpose**: Export files, attachments, and large blobs

**Structure:**

```
s3://coditect-project-intelligence/
├── {organization_id}/
│   ├── {project_id}/
│   │   ├── exports/
│   │   │   ├── 2025-11-17-EXPORT-SESSION-1.txt
│   │   │   ├── 2025-11-17-EXPORT-SESSION-2.txt
│   │   ├── checkpoints/
│   │   │   ├── 2025-11-17T10-21-00Z-Week-1-Phase-1-Complete.md
│   │   ├── attachments/
│   │   │   ├── diagram-1.png
│   │   │   ├── architecture-v2.pdf
```

**PostgreSQL Integration:**

```sql
CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    filename VARCHAR(500) NOT NULL,
    s3_key TEXT NOT NULL,  -- Full S3 path
    s3_bucket VARCHAR(100) NOT NULL,
    mime_type VARCHAR(100),
    size_bytes BIGINT,
    uploaded_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Benefits:**
- ✅ **Infinite scalability**
- ✅ **99.999999999% durability** (11 nines)
- ✅ **Versioning** built-in
- ✅ **Lifecycle policies** (auto-archive old files)

---

## Role-Based Access Control (RBAC)

### Role Definitions

| Role | Permissions | Use Case |
|------|-------------|----------|
| **Owner** | Full access, billing, delete org | Founders, CTOs |
| **Admin** | All except billing/delete org | Engineering Managers |
| **Member** | Read/write projects, checkpoints | Software Engineers |
| **Viewer** | Read-only access | Stakeholders, PMs |
| **Auditor** | Read-only + audit logs | Compliance, Security |
| **Executive** | Read-only + analytics dashboard | CEO, CFO, Board |

### Permission Matrix

```python
PERMISSIONS = {
    "owner": ["*"],  # All permissions
    "admin": [
        "projects:create", "projects:read", "projects:update", "projects:delete",
        "checkpoints:create", "checkpoints:read", "checkpoints:update", "checkpoints:delete",
        "members:invite", "members:remove",
        "audit_logs:read"
    ],
    "member": [
        "projects:read", "projects:update",
        "checkpoints:create", "checkpoints:read", "checkpoints:update",
        "messages:read", "tasks:read"
    ],
    "viewer": [
        "projects:read",
        "checkpoints:read",
        "messages:read",
        "tasks:read"
    ],
    "auditor": [
        "projects:read",
        "checkpoints:read",
        "audit_logs:read",
        "audit_logs:export"
    ],
    "executive": [
        "projects:read",
        "checkpoints:read",
        "analytics:view",
        "reports:generate"
    ]
}
```

### Implementation (FastAPI Example)

```python
from functools import wraps
from fastapi import HTTPException, Depends

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            # Check permission
            member = db.query(OrganizationMember).filter_by(
                user_id=current_user.id,
                organization_id=kwargs.get('organization_id')
            ).first()

            if not member:
                raise HTTPException(403, "Not a member of this organization")

            if permission not in PERMISSIONS[member.role] and "*" not in PERMISSIONS[member.role]:
                raise HTTPException(403, f"Missing permission: {permission}")

            # Log audit event
            db.add(AuditLog(
                organization_id=member.organization_id,
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
@app.get("/api/projects/{project_id}/checkpoints")
@require_permission("checkpoints:read")
async def get_checkpoints(project_id: UUID, current_user: User):
    # Automatically filtered by RLS
    checkpoints = db.query(Checkpoint).filter_by(project_id=project_id).all()
    return checkpoints
```

---

## API Architecture

### REST API (FastAPI)

```python
# Example endpoints
@app.post("/api/organizations")
async def create_organization(...)

@app.get("/api/organizations/{org_id}/projects")
async def list_projects(...)

@app.get("/api/projects/{project_id}/timeline")
async def get_timeline(...)

@app.get("/api/checkpoints/{checkpoint_id}")
async def get_checkpoint(...)

@app.post("/api/search/semantic")
async def semantic_search(query: str, filters: dict)

@app.get("/api/analytics/dashboard")
async def get_dashboard_metrics(...)
```

### GraphQL API (Optional)

```graphql
type Query {
  organization(id: ID!): Organization
  project(id: ID!): Project
  timeline(projectId: ID!, filters: TimelineFilters): Timeline
  semanticSearch(query: String!, limit: Int = 10): [SearchResult]
}

type Organization {
  id: ID!
  name: String!
  projects: [Project!]!
  members: [Member!]!
}

type Project {
  id: ID!
  name: String!
  description: String
  checkpoints: [Checkpoint!]!
  timeline: Timeline
}

type Checkpoint {
  id: ID!
  name: String!
  date: DateTime!
  focusArea: String
  messageCount: Int
  taskCount: Int
  messages: [Message!]!
  tasks: [Task!]!
}
```

---

## Git as Source of Truth

**Critical Principle**: Git repositories are the canonical source - database is a derived view for performance/search.

### Architecture Pattern

```
Git Repository (Source of Truth)
  ├── MEMORY-CONTEXT/dedup_state/unique_messages.jsonl
  ├── MEMORY-CONTEXT/dedup_state/checkpoint_index.json
  ├── CHECKPOINTS/*.md
  └── docs/PROJECT-TIMELINE-DATA.json
         ↓
    GitHub Webhook (on push)
         ↓
    Sync Service (FastAPI background job)
         ↓
    PostgreSQL + ChromaDB (Derived view)
         ↓
    Frontend (reads from database, links back to git)
```

### Sync Mechanisms

#### 1. **GitHub Webhook Integration**

```python
# Webhook endpoint
@app.post("/api/webhooks/github")
async def github_webhook(payload: dict, background_tasks: BackgroundTasks):
    """
    Triggered on git push to sync database with latest git state.
    """
    # Verify webhook signature
    if not verify_github_signature(payload):
        raise HTTPException(403, "Invalid signature")

    # Extract commit info
    repo_url = payload["repository"]["html_url"]
    ref = payload["ref"]  # refs/heads/main
    commits = payload["commits"]

    # Queue sync job
    background_tasks.add_task(sync_from_git, repo_url, ref)

    return {"status": "queued"}


async def sync_from_git(repo_url: str, ref: str):
    """
    Clone/pull repository and sync database with git state.
    """
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
    checkpoints_md = list((Path(repo_dir) / "CHECKPOINTS").glob("*.md"))

    # 3. Sync to database (upsert)
    project = get_or_create_project(repo_url)

    for checkpoint_name, checkpoint_data in checkpoint_index.items():
        # Upsert checkpoint
        db_checkpoint = db.query(Checkpoint).filter_by(
            project_id=project.id,
            name=checkpoint_name
        ).first()

        if not db_checkpoint:
            db_checkpoint = Checkpoint(
                project_id=project.id,
                name=checkpoint_name,
                source_file=checkpoint_data.get("source_file"),
                date=checkpoint_data.get("file_timestamp"),
                message_count=len(checkpoint_data.get("message_hashes", [])),
                git_commit_sha=get_git_commit_sha(repo_dir),  # Track git commit
                metadata=checkpoint_data
            )
            db.add(db_checkpoint)
        else:
            # Update if git commit changed
            current_sha = get_git_commit_sha(repo_dir)
            if db_checkpoint.git_commit_sha != current_sha:
                db_checkpoint.message_count = len(checkpoint_data.get("message_hashes", []))
                db_checkpoint.git_commit_sha = current_sha
                db_checkpoint.updated_at = datetime.now()

    db.commit()

    # 4. Sync messages (only new ones based on content_hash)
    for line in unique_messages:
        msg_data = json.loads(line)
        content_hash = msg_data["hash"]

        # Check if already exists
        exists = db.query(Message).filter_by(content_hash=content_hash).first()
        if not exists:
            # Add new message
            checkpoint_name = msg_data["checkpoint"]
            db_checkpoint = db.query(Checkpoint).filter_by(
                project_id=project.id,
                name=checkpoint_name
            ).first()

            if db_checkpoint:
                message = Message(
                    checkpoint_id=db_checkpoint.id,
                    content_hash=content_hash,
                    content=msg_data["message"]["content"],
                    role=msg_data["message"]["role"],
                    source_type=msg_data["message"]["source_type"]
                )
                db.add(message)

                # Sync to ChromaDB for semantic search
                sync_message_to_chromadb(message)

    db.commit()

    # 5. Record sync event
    db.add(SyncEvent(
        project_id=project.id,
        git_commit_sha=get_git_commit_sha(repo_dir),
        sync_type="webhook",
        messages_synced=len(unique_messages),
        checkpoints_synced=len(checkpoint_index)
    ))
    db.commit()
```

#### 2. **Manual Sync Command**

```bash
# CLI tool for manual sync
coditect-sync --repo https://github.com/coditect-ai/coditect-rollout-master

# Sync specific project
coditect-sync --project-id uuid --force
```

#### 3. **Scheduled Sync (Cron)**

```python
# Run every 15 minutes
@scheduler.scheduled_job('cron', minute='*/15')
async def scheduled_sync():
    """
    Sync all projects with their git repositories.
    """
    projects = db.query(Project).filter_by(active=True).all()

    for project in projects:
        try:
            await sync_from_git(project.repository_url, "main")
        except Exception as e:
            logger.error(f"Sync failed for {project.id}: {e}")
```

### Git Verification

**Every database query returns git metadata:**

```python
@app.get("/api/checkpoints/{checkpoint_id}")
async def get_checkpoint(checkpoint_id: UUID):
    checkpoint = db.query(Checkpoint).filter_by(id=checkpoint_id).first()

    return {
        "id": checkpoint.id,
        "name": checkpoint.name,
        "data": checkpoint.to_dict(),
        "git_metadata": {
            "commit_sha": checkpoint.git_commit_sha,
            "commit_url": f"{checkpoint.project.repository_url}/commit/{checkpoint.git_commit_sha}",
            "source_file": checkpoint.source_file,
            "source_url": f"{checkpoint.project.repository_url}/blob/{checkpoint.git_commit_sha}/{checkpoint.source_file}",
            "last_synced": checkpoint.updated_at.isoformat(),
            "verified": verify_git_consistency(checkpoint)  # Hash check
        }
    }
```

### Git Consistency Verification

```python
def verify_git_consistency(checkpoint: Checkpoint) -> bool:
    """
    Verify database checkpoint matches git source.
    """
    # 1. Fetch file from git at specific commit
    git_url = f"{checkpoint.project.repository_url}/raw/{checkpoint.git_commit_sha}/{checkpoint.source_file}"
    response = requests.get(git_url)

    if response.status_code != 200:
        return False

    # 2. Load git checkpoint index
    git_checkpoint_index = json.loads(response.text)
    git_checkpoint_data = git_checkpoint_index.get(checkpoint.name)

    if not git_checkpoint_data:
        return False

    # 3. Compare hashes
    db_hashes = set(db.query(Message.content_hash).filter_by(
        checkpoint_id=checkpoint.id
    ).all())

    git_hashes = set(git_checkpoint_data.get("message_hashes", []))

    return db_hashes == git_hashes
```

### Frontend Git Links

**Every UI element links back to git:**

```typescript
// React component
function CheckpointCard({ checkpoint }) {
  return (
    <Card>
      <CardHeader>
        <h3>{checkpoint.name}</h3>
        <Badge>
          <GitIcon />
          <a href={checkpoint.git_metadata.commit_url} target="_blank">
            {checkpoint.git_metadata.commit_sha.slice(0, 7)}
          </a>
        </Badge>
      </CardHeader>

      <CardContent>
        <p>{checkpoint.message_count} messages</p>
        <p>{checkpoint.task_count} tasks</p>

        {!checkpoint.git_metadata.verified && (
          <Alert variant="warning">
            ⚠️ Database out of sync with git.
            <Button onClick={() => syncCheckpoint(checkpoint.id)}>
              Sync Now
            </Button>
          </Alert>
        )}
      </CardContent>

      <CardFooter>
        <a href={checkpoint.git_metadata.source_url} target="_blank">
          View source in GitHub →
        </a>
      </CardFooter>
    </Card>
  );
}
```

### Database Schema Addition

```sql
-- Add git tracking columns
ALTER TABLE checkpoints ADD COLUMN git_commit_sha VARCHAR(40);
ALTER TABLE checkpoints ADD COLUMN git_branch VARCHAR(255) DEFAULT 'main';
ALTER TABLE checkpoints ADD COLUMN git_verified BOOLEAN DEFAULT TRUE;
ALTER TABLE checkpoints ADD COLUMN last_synced_at TIMESTAMPTZ;

CREATE TABLE sync_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    git_commit_sha VARCHAR(40) NOT NULL,
    sync_type VARCHAR(50), -- 'webhook', 'manual', 'scheduled'
    messages_synced INTEGER DEFAULT 0,
    checkpoints_synced INTEGER DEFAULT 0,
    duration_ms INTEGER,
    status VARCHAR(50), -- 'success', 'failed', 'partial'
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for quick verification
CREATE INDEX idx_checkpoints_git_sha ON checkpoints(git_commit_sha);
CREATE INDEX idx_sync_events_project ON sync_events(project_id, created_at DESC);
```

### Benefits of Git-First Architecture

✅ **Single Source of Truth**: Git is canonical, database is cache
✅ **Auditability**: Every database record traces to git commit
✅ **Disaster Recovery**: Recreate entire database from git
✅ **Offline Capability**: Clone git repo, work offline
✅ **Trust**: Users can verify database matches git
✅ **Version History**: Full git history preserved
✅ **Collaboration**: Git workflows (branches, PRs) still work

---

## Deployment Architecture

### GCP (Recommended)

```
Frontend (React + Next.js)
  ↓
Cloud Load Balancer (HTTPS)
  ↓
Cloud Run (FastAPI Backend)
  ├→ Cloud SQL (PostgreSQL) - Primary database
  ├→ Cloud Memorystore (Redis) - Session + cache
  ├→ Cloud Storage (GCS) - Export files
  └→ ChromaDB (Cloud Run) - Semantic search

Monitoring:
  - Cloud Monitoring (metrics, logs, traces)
  - Cloud Logging (audit logs)
  - Cloud IAM (authentication)
```

**Cost Estimate (1000 users, 100 orgs):**
- Cloud SQL (PostgreSQL): ~$200/month
- Cloud Memorystore (Redis): ~$50/month
- Cloud Storage (GCS): ~$20/month
- Cloud Run (Backend): ~$100/month
- ChromaDB (self-hosted): ~$50/month
- **Total: ~$420/month** (scales linearly)

---

## Migration Plan

### Phase 1: Infrastructure Setup (Week 1)

- [ ] Provision Cloud SQL PostgreSQL instance
- [ ] Setup Cloud Memorystore Redis
- [ ] Create GCS buckets
- [ ] Deploy ChromaDB to Cloud Run

### Phase 2: Schema & Migration (Week 2)

- [ ] Create PostgreSQL schema
- [ ] Write migration script from JSONL to PostgreSQL
- [ ] Migrate 1,601 messages from `dedup_state/unique_messages.jsonl`
- [ ] Migrate 49 checkpoints from `dedup_state/checkpoint_index.json`
- [ ] Extract 242 tasks from checkpoint markdown files

### Phase 3: Backend API (Weeks 3-4)

- [ ] Implement FastAPI backend (already in `coditect-cloud-backend` submodule)
- [ ] Add authentication (JWT + OAuth2)
- [ ] Implement RBAC middleware
- [ ] Create REST API endpoints
- [ ] Add semantic search integration with ChromaDB

### Phase 4: Frontend (Weeks 5-6)

- [ ] Enhance interactive timeline (already created)
- [ ] Add login/authentication UI
- [ ] Build organization management
- [ ] Create project dashboard
- [ ] Implement role-based UI (hide/show based on permissions)

### Phase 5: Testing & Launch (Week 7-8)

- [ ] Load testing (100 concurrent users)
- [ ] Security audit
- [ ] Compliance review (SOC 2, GDPR)
- [ ] Beta launch with 10 organizations
- [ ] Production launch

---

## Security Considerations

### Data Encryption

```python
# At rest
- PostgreSQL: AES-256 encryption (Cloud SQL default)
- Redis: Encryption enabled
- S3: Server-side encryption (SSE-S3)

# In transit
- TLS 1.3 for all connections
- Certificate pinning for mobile apps
```

### Authentication

```python
# Multi-factor authentication (MFA)
- Email + password (Argon2 hashing)
- Google OAuth2
- GitHub OAuth2
- SAML SSO (enterprise)
- Hardware keys (YubiKey)
```

### Compliance

- ✅ **GDPR**: Right to delete, data portability
- ✅ **SOC 2**: Audit logs, access controls
- ✅ **HIPAA**: Encrypted at rest/transit (if needed)

---

## Monitoring & Observability

### Metrics (Prometheus + Grafana)

```python
# Custom metrics
- coditect_api_requests_total{endpoint, method, status}
- coditect_db_query_duration_seconds{query_type}
- coditect_cache_hit_ratio
- coditect_semantic_search_latency_ms
- coditect_active_sessions
```

### Logging (Structured JSON)

```json
{
  "timestamp": "2025-11-17T21:30:00Z",
  "level": "INFO",
  "user_id": "uuid",
  "organization_id": "uuid",
  "action": "view_checkpoint",
  "resource_id": "checkpoint-uuid",
  "duration_ms": 45,
  "status": "success"
}
```

### Alerts

- 🚨 **P0**: Database down, >5% error rate
- ⚠️  **P1**: High latency (>1s p95), cache miss >50%
- 📊 **P2**: Disk usage >80%, memory >90%

---

## Next Steps

1. ✅ **Review this architecture** with team
2. ⏸️ **Decide on cloud provider** (GCP recommended)
3. ⏸️ **Allocate budget** (~$500/month initial)
4. ⏸️ **Assign team**:
   - Backend engineer (FastAPI + PostgreSQL)
   - Frontend engineer (React + Next.js)
   - DevOps engineer (GCP + Terraform)
5. ⏸️ **Start Phase 1** (Infrastructure setup)

---

## References

- [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [ChromaDB Documentation](https://docs.trychroma.com/)
- [FastAPI Best Practices](https://fastapi.tiangolo.com/tutorial/)
- [GCP Architecture Patterns](https://cloud.google.com/architecture)

---

**Status**: ✅ Design Complete - Ready for Implementation
**Owner**: TBD
**Last Updated**: 2025-11-17
