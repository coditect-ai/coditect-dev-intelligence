# ADR-005: FastAPI over Flask/Django - Project Intelligence Platform

```yaml
Document: ADR-005-project-intelligence-fastapi-backend
Version: 1.0.0
Purpose: Select FastAPI for async-first backend with high performance and type safety
Audience: Engineering teams, backend developers, architects
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-002 (PostgreSQL), ADR-007 (GCP Deployment)
```

---

## Executive Summary

**Decision**: Use **FastAPI** as the backend framework for the CODITECT Project Intelligence Platform.

**Why FastAPI**:
- ✅ **Async/Await Native**: Non-blocking I/O for real-time sync from git
- ✅ **Type Safety**: Pydantic models catch bugs at development time
- ✅ **Auto-Generated OpenAPI**: Interactive API docs out-of-the-box
- ✅ **High Performance**: Comparable to Node.js, Go (ASGI-based)
- ✅ **Modern Python**: Python 3.11+ with latest async features

**Use Case**: Real-time git webhook processing, semantic search queries, multi-tenant API

**Alternatives Rejected**: Flask (no async), Django (too heavy, sync-first)

---

## Context and Problem Statement

### Requirements

The Project Intelligence Platform backend must:

1. **Handle Real-Time Webhooks**: GitHub push events trigger background sync jobs
2. **Async Database Operations**: PostgreSQL + ChromaDB queries without blocking
3. **Type Safety**: Catch bugs before production (Pydantic models)
4. **API Documentation**: Auto-generated OpenAPI spec for frontend team
5. **High Concurrency**: Support 1000+ concurrent users

---

## Decision Drivers

1. **Async/Await Support** - Non-blocking I/O (critical for webhooks)
2. **Type Safety** - Pydantic models for request/response validation
3. **Performance** - Handle 1000+ req/sec
4. **Auto Documentation** - OpenAPI/Swagger UI
5. **Modern Python** - Python 3.11+ support

---

## Considered Options

### Option 1: **FastAPI** (SELECTED ✅)

**Key Features**:
```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel

app = FastAPI()

class MessageCreate(BaseModel):
    checkpoint_id: UUID
    content: str

@app.post("/api/messages")
async def create_message(
    data: MessageCreate,
    background_tasks: BackgroundTasks
):
    # Insert into PostgreSQL (async)
    message = await db.execute(
        "INSERT INTO messages (checkpoint_id, content) VALUES ($1, $2)",
        data.checkpoint_id, data.content
    )

    # Queue embedding generation (non-blocking)
    background_tasks.add_task(embed_message, message.id)

    return {"id": message.id, "status": "created"}
```

**Pros**:
- ✅ **Async/Await Native**: Non-blocking I/O for webhooks, database queries
- ✅ **Type Safety**: Pydantic models validate requests/responses automatically
- ✅ **Auto OpenAPI**: `/docs` endpoint with Swagger UI
- ✅ **High Performance**: ~20,000 req/sec (comparable to Node.js)
- ✅ **Modern**: Python 3.11+ with latest async features
- ✅ **Dependency Injection**: Clean architecture patterns
- ✅ **WebSockets**: Native support for real-time features

**Cons**:
- ⚠️ **Young Framework**: Only 5 years old (less mature than Flask/Django)
- ⚠️ **Learning Curve**: Async patterns require understanding

**Performance**: 20,000 req/sec (Uvicorn + ASGI)

---

### Option 2: Flask

**Pros**:
- ✅ **Mature**: 13+ years, large ecosystem
- ✅ **Simple**: Easy to learn
- ✅ **Flexible**: Minimal framework

**Cons**:
- ❌ **No Native Async**: WSGI-based (synchronous)
- ❌ **No Type Safety**: No built-in request validation
- ❌ **Manual Docs**: OpenAPI requires manual setup

**Why Rejected**: No async support = blocking I/O for database/webhook operations

---

### Option 3: Django REST Framework

**Pros**:
- ✅ **Batteries Included**: Admin, ORM, auth built-in
- ✅ **Mature**: 17+ years
- ✅ **Django Ecosystem**: Large community

**Cons**:
- ❌ **Sync-First**: Async added late (Django 3.1+), still incomplete
- ❌ **Heavy**: Large framework for simple API
- ❌ **ORM Lock-In**: Django ORM required (we use SQLAlchemy)

**Why Rejected**: Too heavy for API-only backend, sync-first architecture

---

## Decision Outcome

**Chosen Option**: **FastAPI** (Option 1)

### Rationale

1. **Async/Await Native**: Critical for real-time webhook processing
2. **Type Safety**: Pydantic catches bugs at development time
3. **Auto Documentation**: OpenAPI spec auto-generated
4. **High Performance**: 20,000 req/sec = handles scale
5. **Modern Python**: Best-in-class developer experience

### Implementation

```python
# main.py
from fastapi import FastAPI, Depends, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import text

app = FastAPI(title="CODITECT Project Intelligence API", version="1.0.0")

# CORS for React frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://coditect.ai"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)

# RLS context middleware
@app.middleware("http")
async def set_rls_context(request: Request, call_next):
    user = get_current_user(request)
    async with db.begin():
        await db.execute(text(f"SET app.current_user_id = '{user.id}'"))
    return await call_next(request)

# GitHub webhook endpoint (async)
@app.post("/api/webhooks/github")
async def github_webhook(payload: dict, background_tasks: BackgroundTasks):
    # Verify signature
    if not verify_github_signature(payload):
        raise HTTPException(403, "Invalid signature")

    # Queue sync job (non-blocking)
    background_tasks.add_task(sync_from_git, payload["repository"]["html_url"])

    return {"status": "queued"}
```

---

## Consequences

### Positive ✅

1. **High Concurrency**: Async/await handles 1000+ concurrent requests
2. **Type Safety**: Pydantic catches bugs before production
3. **Developer Experience**: Auto-complete, type hints, auto docs
4. **Performance**: 20,000 req/sec = scale to millions of users

### Negative ⚠️

1. **Learning Curve**: Async patterns require understanding
   - **Mitigation**: Team training, code reviews

2. **Young Framework**: 5 years old (less mature than Flask)
   - **Mitigation**: Active development, 70K+ GitHub stars

---

## Related Decisions

- **ADR-002**: [PostgreSQL as Primary Database](ADR-002-postgresql-as-primary-database.md)
- **ADR-007**: [GCP Cloud Run Deployment](ADR-007-gcp-cloud-run-deployment.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership

---

**Made with ❤️ by CODITECT Engineering**
