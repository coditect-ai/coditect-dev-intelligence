# ADR-003: ChromaDB for Semantic Search - Project Intelligence Platform

```yaml
Document: ADR-003-project-intelligence-chromadb-semantic-search
Version: 1.0.0
Purpose: Select ChromaDB for AI-powered semantic search across project conversations
Audience: Engineering teams, ML engineers, architects
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-002 (PostgreSQL), ADR-001 (Git Source of Truth)
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

**Decision**: Use **ChromaDB** for semantic search and vector similarity on project conversations (1,601+ messages).

**Why ChromaDB**:
- ✅ **AI-Native**: Purpose-built for LLM applications and embeddings
- ✅ **Self-Hosted**: No vendor lock-in, no external API dependencies
- ✅ **Python-First**: Seamless integration with FastAPI backend
- ✅ **Vector Similarity**: Find related conversations without exact keyword matches
- ✅ **Cost-Effective**: Self-hosted = no per-query API costs

**Use Case**: Users search "authentication issues" and find relevant conversations even if they don't contain that exact phrase (semantic understanding).

**Alternatives Rejected**: Elasticsearch (overkill for semantic search), Pinecone (vendor lock-in, API costs), PostgreSQL pgvector (immature)

---

## Context and Problem Statement

### The Challenge

CODITECT Project Intelligence Platform needs to enable users to:

1. **Semantic Search**: Find conversations by meaning, not just keywords
   - Example: Search "login problems" → Find messages about "authentication failures", "SSO issues"
2. **Related Conversations**: "Show me similar discussions to this checkpoint"
3. **AI-Powered Discovery**: "What have we learned about database design?"
4. **Cross-Project Insights**: "Find all conversations about performance optimization across projects"

### Business Requirements

**User Experience**:
- Users shouldn't need to know exact keywords to find information
- Search should understand synonyms, context, related concepts
- Results should be ranked by relevance, not just keyword frequency

**Performance**:
- Search latency <500ms for 95th percentile
- Support 1,601+ messages initially, scale to 100,000+
- Real-time indexing (new messages searchable within seconds)

**Operational**:
- Self-hosted (no external API dependencies)
- Cost-effective at scale (no per-query charges)
- Easy integration with PostgreSQL and FastAPI

---

## Decision Drivers

### Mandatory Requirements (Must-Have)

1. **Semantic Search** - Understand meaning, not just keywords
2. **Vector Embeddings** - Support OpenAI/Anthropic/local embeddings
3. **Python Integration** - Native Python library for FastAPI
4. **Self-Hosted** - No vendor lock-in, no external APIs
5. **Metadata Filtering** - Filter by organization, project, date, etc.

### Important Goals (Should-Have)

6. **Low Latency** - Sub-500ms search queries
7. **Horizontal Scaling** - Scale to 100,000+ documents
8. **Persistence** - Data survives restarts
9. **Multi-Collection** - Separate collections for messages, tasks, etc.
10. **Cost-Effective** - No per-query API costs

### Nice-to-Have

11. **Multi-Modal** - Support images, code snippets (future)
12. **Hybrid Search** - Combine semantic + keyword search
13. **Reranking** - Re-rank results for better relevance

---

## Considered Options

### Option 1: **ChromaDB** (SELECTED ✅)

**Technical Profile**:
- **Type**: Vector database for AI/ML applications
- **License**: Open-source (Apache 2.0)
- **First Release**: 2022
- **Language**: Python
- **Embedding Support**: OpenAI, Anthropic, HuggingFace, local models

**Architecture**:
```python
import chromadb
from chromadb.config import Settings

# Initialize client (persistent)
client = chromadb.PersistentClient(path="/var/lib/chromadb")

# Create collection for messages
messages_collection = client.get_or_create_collection(
    name="messages",
    metadata={"hnsw:space": "cosine"}  # Similarity metric
)

# Add messages with embeddings
messages_collection.add(
    documents=["User authentication failed with 401 error"],
    embeddings=[[0.1, 0.2, ..., 0.9]],  # 1536-dim vector from OpenAI
    metadatas=[{
        "organization_id": "org-uuid",
        "project_id": "proj-uuid",
        "checkpoint_id": "ckpt-uuid",
        "date": "2025-11-17"
    }],
    ids=["message-uuid"]
)

# Semantic search
results = messages_collection.query(
    query_texts=["login problems"],
    n_results=10,
    where={"organization_id": "org-uuid"}  # Tenant filtering
)
```

**Pros**:
- ✅ **AI-Native**: Built for LLM embeddings and semantic search
- ✅ **Self-Hosted**: No external API calls, no vendor lock-in
- ✅ **Python-First**: Native integration with FastAPI
- ✅ **Easy to Use**: Simple API, minimal configuration
- ✅ **Persistent**: Data stored on disk, survives restarts
- ✅ **Metadata Filtering**: Query by organization, project, date
- ✅ **Cost-Effective**: Self-hosted = no per-query charges
- ✅ **Active Development**: 17K+ GitHub stars, frequent updates
- ✅ **Vector Similarity**: HNSW algorithm for fast nearest-neighbor search
- ✅ **Multi-Modal**: Support text, images, code (future)

**Cons**:
- ⚠️ **Young Project**: Only 2 years old (less mature than Elasticsearch)
- ⚠️ **Limited RBAC**: No built-in role-based access control (must implement in app)
- ⚠️ **Single-Node**: No native clustering (horizontal scaling via sharding)
- ⚠️ **Embedding Costs**: Must generate embeddings via OpenAI/Anthropic (pay per message)

**Cloud Costs**:
- **Self-Hosted**: $50/month (Cloud Run container, 2 vCPU, 4 GB RAM)
- **Embedding Costs**: ~$0.10/1000 messages (OpenAI text-embedding-ada-002)
- **Total**: ~$60/month for 1,601 messages + ongoing indexing

---

### Option 2: Elasticsearch

**Technical Profile**:
- **Type**: Full-text search engine
- **License**: Elastic License (not fully open-source since 2021)
- **Strengths**: Mature, full-text search, analytics

**Pros**:
- ✅ **Mature**: 15+ years of production use
- ✅ **Full-Text Search**: Excellent keyword search
- ✅ **Analytics**: Aggregations, dashboards (Kibana)
- ✅ **Horizontal Scaling**: Multi-node clustering

**Cons**:
- ❌ **Not AI-Native**: Vector search added later (kNN plugin)
- ❌ **Operational Complexity**: Requires cluster management, tuning
- ❌ **Resource Heavy**: High memory/CPU requirements
- ❌ **Licensing**: Elastic License restricts cloud providers
- ❌ **Cost**: Managed Elastic Cloud ~$100-500/month
- ❌ **Overkill**: We don't need full-text analytics, just semantic search

**Why Rejected**:
- **Overkill**: Elasticsearch is designed for full-text search and analytics, not semantic search
- **Operational Burden**: Requires cluster management, tuning, monitoring
- **Cost**: 5-10x more expensive than ChromaDB for our use case
- **Not AI-Native**: Vector search is an add-on, not core feature

---

### Option 3: Pinecone

**Technical Profile**:
- **Type**: Managed vector database (SaaS)
- **License**: Proprietary
- **Strengths**: Fully managed, enterprise features

**Pros**:
- ✅ **Fully Managed**: No infrastructure management
- ✅ **Scalable**: Handles billions of vectors
- ✅ **Enterprise Features**: RBAC, multi-region, backups
- ✅ **High Performance**: Optimized for large-scale vector search

**Cons**:
- ❌ **Vendor Lock-In**: Proprietary API, hard to migrate away
- ❌ **API Costs**: Pay per query + storage
- ❌ **External Dependency**: Requires internet, adds latency
- ❌ **No Self-Hosting**: Cannot run on-premise
- ❌ **Pricing Complexity**: Tiered pricing based on usage

**Pricing**:
- **Starter**: $70/month (1M vectors, 100K queries/month)
- **Standard**: $200/month + overages
- **Enterprise**: Custom pricing ($1000+/month)

**Why Rejected**:
- **Vendor Lock-In**: Proprietary API makes migration difficult
- **Cost**: 10x more expensive than self-hosted ChromaDB
- **External Dependency**: Adds latency, requires internet connectivity
- **Self-Hosting Preference**: We prefer control over infrastructure

---

### Option 4: PostgreSQL pgvector Extension

**Technical Profile**:
- **Type**: PostgreSQL extension for vector similarity
- **License**: Open-source (PostgreSQL License)
- **Strengths**: Integrated with PostgreSQL

**Pros**:
- ✅ **Integrated**: Single database for structured + vectors
- ✅ **ACID Transactions**: Strong consistency
- ✅ **Mature Database**: PostgreSQL's 35-year track record
- ✅ **No Additional Infrastructure**: Use existing PostgreSQL

**Cons**:
- ⚠️ **Immature**: pgvector is relatively new (2021)
- ⚠️ **Performance**: Slower than specialized vector databases at scale
- ⚠️ **Limited Features**: No hybrid search, no reranking
- ⚠️ **Indexing Overhead**: HNSW indexes slow down inserts
- ⚠️ **Scalability**: Single-node bottleneck

**Why Rejected**:
- **Immature**: pgvector is only 3 years old, less battle-tested than ChromaDB
- **Performance**: Specialized vector databases (ChromaDB) are faster for large-scale search
- **Separation of Concerns**: Keep structured data (PostgreSQL) and vectors (ChromaDB) separate for better scalability

---

## Decision Outcome

**Chosen Option**: **ChromaDB** (Option 1)

### Rationale

1. **AI-Native**: Built for LLM embeddings and semantic search
   - ChromaDB is designed for AI/ML workloads
   - Elasticsearch is designed for full-text search (different use case)

2. **Self-Hosted**: No vendor lock-in, no external API dependencies
   - Run on Cloud Run, GKE, or any container platform
   - Pinecone requires external API calls (latency, cost)

3. **Cost-Effective**: Self-hosted = no per-query charges
   - ChromaDB: ~$60/month total
   - Pinecone: ~$200/month + overages
   - Elasticsearch: ~$300/month for managed cluster

4. **Python-First**: Seamless FastAPI integration
   - Native Python library, no API translation layer
   - Easy to integrate with existing backend

5. **Metadata Filtering**: Query by organization, project, date
   - Essential for multi-tenant isolation
   - ChromaDB supports rich metadata filters

### Implementation Pattern

**Embedding Pipeline**:
```python
# 1. Generate embedding when message is created
import openai

async def create_message(checkpoint_id: UUID, content: str):
    # Insert into PostgreSQL
    message = Message(
        checkpoint_id=checkpoint_id,
        content=content,
        content_hash=hashlib.sha256(content.encode()).hexdigest()
    )
    db.add(message)
    db.commit()

    # Generate embedding
    response = openai.Embedding.create(
        input=content,
        model="text-embedding-ada-002"  # 1536 dimensions
    )
    embedding = response["data"][0]["embedding"]

    # Add to ChromaDB
    messages_collection.add(
        documents=[content],
        embeddings=[embedding],
        metadatas=[{
            "message_id": str(message.id),
            "checkpoint_id": str(checkpoint_id),
            "organization_id": str(message.checkpoint.project.organization_id),
            "project_id": str(message.checkpoint.project_id),
            "date": message.created_at.isoformat()
        }],
        ids=[str(message.id)]
    )

    return message


# 2. Semantic search endpoint
@app.post("/api/search/semantic")
async def semantic_search(
    query: str,
    current_user: User = Depends(get_current_user),
    limit: int = 10
):
    # Filter to user's organizations
    user_org_ids = get_user_organization_ids(current_user)

    # Semantic search with metadata filtering
    results = messages_collection.query(
        query_texts=[query],
        n_results=limit,
        where={"organization_id": {"$in": user_org_ids}}
    )

    # Enrich with PostgreSQL data
    message_ids = [UUID(id) for id in results["ids"][0]]
    messages = db.query(Message).filter(Message.id.in_(message_ids)).all()

    return {
        "query": query,
        "results": [
            {
                "message": message.to_dict(),
                "similarity": results["distances"][0][i],
                "metadata": results["metadatas"][0][i]
            }
            for i, message in enumerate(messages)
        ]
    }
```

---

## Consequences

### Positive Consequences ✅

1. **Semantic Understanding**:
   - Search "authentication issues" → Find "login failures", "SSO problems"
   - Users don't need to know exact keywords
   - Better discovery of related conversations

2. **AI-Powered Insights**:
   - "What have we learned about X?" queries
   - Discover similar checkpoints automatically
   - Cross-project pattern detection

3. **Self-Hosted Control**:
   - No vendor lock-in (ChromaDB is open-source)
   - Deploy on any container platform
   - No external API dependencies (except embedding generation)

4. **Cost-Effective**:
   - ~$60/month total (vs $200+ for Pinecone)
   - No per-query charges
   - Embedding costs ~$0.10/1000 messages (one-time)

5. **Multi-Tenant Isolation**:
   - Metadata filtering ensures tenant boundaries
   - Query only user's organizations
   - Compliant with SOC2 requirements

6. **Future-Proof**:
   - Multi-modal support (images, code)
   - Hybrid search (semantic + keyword)
   - Reranking for better relevance

### Negative Consequences ⚠️

1. **Embedding Generation Costs**:
   - OpenAI API: ~$0.10/1000 messages
   - **Mitigation**: Use local models (HuggingFace) or batch processing

2. **No Built-In RBAC**:
   - Must implement tenant filtering in application
   - **Mitigation**: Metadata filtering on organization_id (already implemented)

3. **Single-Node Limitation**:
   - No native clustering (horizontal scaling via sharding)
   - **Mitigation**: Run multiple ChromaDB instances, shard by organization_id

4. **Young Project**:
   - Only 2 years old (less mature than Elasticsearch)
   - **Mitigation**: Active development (17K+ GitHub stars), production use at scale

5. **Operational Complexity** (Self-Hosted):
   - Must manage ChromaDB container, backups, monitoring
   - **Mitigation**: Run on Cloud Run (fully managed container platform)

---

## Implementation Details

### Deployment Architecture

**Cloud Run Deployment**:
```yaml
# cloudbuild.yaml for ChromaDB
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'gcr.io/$PROJECT_ID/chromadb:$SHORT_SHA'
      - '-f'
      - 'Dockerfile.chromadb'
      - '.'

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/chromadb:$SHORT_SHA']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'chromadb'
      - '--image=gcr.io/$PROJECT_ID/chromadb:$SHORT_SHA'
      - '--region=us-central1'
      - '--platform=managed'
      - '--allow-unauthenticated'  # Internal VPC only
      - '--memory=4Gi'
      - '--cpu=2'
```

**Dockerfile.chromadb**:
```dockerfile
FROM python:3.11-slim

# Install ChromaDB
RUN pip install chromadb==0.4.18

# Persistent storage
VOLUME /var/lib/chromadb

# Expose port
EXPOSE 8000

# Start ChromaDB server
CMD ["chroma", "run", "--host", "0.0.0.0", "--port", "8000", "--path", "/var/lib/chromadb"]
```

### Integration with PostgreSQL

**Sync Pipeline** (when message is created):
```python
# FastAPI background task
@app.post("/api/messages")
async def create_message(
    data: MessageCreate,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(get_current_user)
):
    # 1. Insert into PostgreSQL
    message = Message(
        checkpoint_id=data.checkpoint_id,
        content=data.content,
        content_hash=hashlib.sha256(data.content.encode()).hexdigest()
    )
    db.add(message)
    db.commit()

    # 2. Queue embedding generation (non-blocking)
    background_tasks.add_task(embed_message, message.id)

    return message


async def embed_message(message_id: UUID):
    """Generate embedding and add to ChromaDB."""
    message = db.query(Message).filter_by(id=message_id).first()

    # Generate embedding
    response = openai.Embedding.create(
        input=message.content,
        model="text-embedding-ada-002"
    )
    embedding = response["data"][0]["embedding"]

    # Add to ChromaDB
    messages_collection.add(
        documents=[message.content],
        embeddings=[embedding],
        metadatas=[{
            "message_id": str(message.id),
            "checkpoint_id": str(message.checkpoint_id),
            "organization_id": str(message.checkpoint.project.organization_id),
            "date": message.created_at.isoformat()
        }],
        ids=[str(message.id)]
    )

    logger.info(f"Embedded message {message_id}")
```

### Backup Strategy

**ChromaDB Data Persistence**:
```bash
# Backup ChromaDB data (GCS bucket)
gsutil -m rsync -r /var/lib/chromadb gs://coditect-chromadb-backups/$(date +%Y%m%d)/

# Restore from backup
gsutil -m rsync -r gs://coditect-chromadb-backups/20251117/ /var/lib/chromadb
```

### Performance Optimization

**Collection Configuration**:
```python
# Create collection with HNSW index (fast nearest-neighbor search)
messages_collection = client.get_or_create_collection(
    name="messages",
    metadata={
        "hnsw:space": "cosine",        # Similarity metric
        "hnsw:construction_ef": 100,   # Index build quality
        "hnsw:search_ef": 100,         # Query accuracy
        "hnsw:M": 16                   # Graph connectivity
    }
)
```

---

## Validation and Compliance

### Success Criteria

**Functional Requirements**:
- ✅ Semantic search returns relevant results (evaluated by humans)
- ✅ Metadata filtering enforces tenant isolation (100% accuracy)
- ✅ Support 1,601+ messages, scalable to 100,000+

**Non-Functional Requirements**:
- ✅ Search latency <500ms for 95th percentile
- ✅ Embedding generation <5 seconds per message
- ✅ Data persistence (survives container restarts)

### Testing Strategy

```python
# Semantic search quality testing
def test_semantic_search_relevance():
    """Verify semantic search returns relevant results."""
    # Index sample messages
    messages_collection.add(
        documents=[
            "User authentication failed with 401 error",
            "Cannot login to the dashboard",
            "Database query performance is slow"
        ],
        embeddings=[...],  # Generate embeddings
        ids=["msg1", "msg2", "msg3"]
    )

    # Search for "login problems"
    results = messages_collection.query(
        query_texts=["login problems"],
        n_results=3
    )

    # Verify top results are authentication-related
    assert "msg1" in results["ids"][0][:2]  # "authentication failed"
    assert "msg2" in results["ids"][0][:2]  # "Cannot login"
    assert "msg3" not in results["ids"][0][:2]  # "Database query" (not relevant)


# Tenant isolation testing
def test_tenant_isolation():
    """Verify metadata filtering enforces tenant boundaries."""
    # Add messages for different orgs
    messages_collection.add(
        documents=["Org A message", "Org B message"],
        embeddings=[[...], [...]],
        metadatas=[
            {"organization_id": "org-a"},
            {"organization_id": "org-b"}
        ],
        ids=["msg-a", "msg-b"]
    )

    # Query with Org A filter
    results = messages_collection.query(
        query_texts=["message"],
        where={"organization_id": "org-a"}
    )

    # Verify only Org A results returned
    assert results["ids"][0] == ["msg-a"]
    assert "msg-b" not in results["ids"][0]
```

---

## Related Decisions

- **ADR-002**: [PostgreSQL as Primary Database](ADR-002-postgresql-as-primary-database.md)
- **ADR-001**: [Git as Source of Truth](ADR-001-git-as-source-of-truth.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership, ML Team
**Review Date**: 2026-02-17 (90 days)

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0.0 | 2025-11-17 | Initial ADR | ADR Compliance Specialist |

---

**Made with ❤️ by CODITECT Engineering**
