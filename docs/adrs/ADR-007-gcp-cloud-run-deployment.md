# ADR-007: GCP Cloud Run Deployment - Project Intelligence Platform

```yaml
Document: ADR-007-project-intelligence-gcp-cloud-run
Version: 1.0.0
Purpose: Select Google Cloud Platform (GCP) for deployment with Cloud Run
Audience: Engineering teams, DevOps, infrastructure engineers
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-005 (FastAPI), ADR-006 (Next.js)
```

---

## Executive Summary

**Decision**: Deploy on **Google Cloud Platform (GCP)** using **Cloud Run** for serverless containers.

**Why GCP Cloud Run**:
- ✅ **Serverless**: Auto-scaling from 0 to 1000+ instances
- ✅ **Pay-per-Use**: Billed per request (no idle costs)
- ✅ **Zero-Config**: Deploy with `gcloud run deploy` (30 seconds)
- ✅ **Cloud SQL Integration**: Managed PostgreSQL with automatic backups
- ✅ **Team Familiarity**: Team has GCP experience

**Architecture**:
```
Cloud Load Balancer (HTTPS)
├─ Cloud Run (FastAPI Backend) → Cloud SQL (PostgreSQL)
├─ Cloud Run (Next.js Frontend)
└─ Cloud Run (ChromaDB)
```

**Alternatives Rejected**: AWS Lambda (cold starts), Azure (team unfamiliarity), Self-hosted Kubernetes (operational overhead)

---

## Context and Problem Statement

### Requirements

1. **Auto-Scaling**: Handle 10 → 10,000 users without manual intervention
2. **Managed Database**: PostgreSQL with automated backups, HA
3. **Cost-Effective**: Pay-per-use (no idle costs)
4. **Fast Deployment**: Deploy in <5 minutes
5. **High Availability**: 99.95% uptime SLA

---

## Decision Drivers

1. **Serverless** - Auto-scaling, pay-per-use
2. **Managed Services** - Database, load balancer, TLS
3. **Team Familiarity** - GCP experience
4. **Cost-Effective** - <$500/month for 1000 users
5. **Fast Deployment** - Deploy in minutes

---

## Considered Options

### Option 1: **GCP Cloud Run** (SELECTED ✅)

**Architecture**:
```
Cloud Load Balancer (34.8.51.57)
├─ Cloud Run: coditect-api (FastAPI)
│   ├─ Min instances: 1
│   ├─ Max instances: 100
│   ├─ Memory: 2 GB
│   └─ CPU: 2 vCPU
├─ Cloud Run: coditect-frontend (Next.js)
│   ├─ Min instances: 1
│   ├─ Max instances: 50
│   ├─ Memory: 1 GB
│   └─ CPU: 1 vCPU
└─ Cloud Run: chromadb (Vector DB)
    ├─ Min instances: 1
    ├─ Max instances: 10
    ├─ Memory: 4 GB
    └─ CPU: 2 vCPU

Cloud SQL (PostgreSQL)
├─ Instance: db-custom-4-16384
├─ vCPU: 4
├─ Memory: 16 GB
├─ Storage: 100 GB SSD
└─ Backup: Daily automated
```

**Deployment**:
```bash
# Deploy FastAPI backend
gcloud run deploy coditect-api \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 100 \
  --memory 2Gi \
  --cpu 2

# Deploy Next.js frontend
gcloud run deploy coditect-frontend \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 50 \
  --memory 1Gi

# Deploy ChromaDB
gcloud run deploy chromadb \
  --image chromadb/chroma:latest \
  --region us-central1 \
  --min-instances 1 \
  --max-instances 10 \
  --memory 4Gi \
  --cpu 2
```

**Pros**:
- ✅ **Serverless**: Auto-scale from 0 to 1000+ instances
- ✅ **Pay-per-Use**: Billed per request (no idle costs)
- ✅ **Zero-Config**: Deploy in 30 seconds
- ✅ **Managed Database**: Cloud SQL (PostgreSQL) with auto backups
- ✅ **HTTPS Built-In**: Automatic TLS certificates
- ✅ **High Availability**: 99.95% uptime SLA

**Cons**:
- ⚠️ **Cold Starts**: First request takes 1-2 seconds (mitigated by min instances)
- ⚠️ **Request Timeout**: 60-minute max (sufficient for webhooks)

**Cost** (1000 users, 100K req/month):
- Cloud Run (API): ~$50/month
- Cloud Run (Frontend): ~$30/month
- Cloud Run (ChromaDB): ~$40/month
- Cloud SQL (PostgreSQL): ~$200/month
- **Total**: ~$320/month

---

### Option 2: AWS Lambda + RDS

**Pros**:
- ✅ **Serverless**: Auto-scaling
- ✅ **Large Ecosystem**: AWS services

**Cons**:
- ❌ **Cold Starts**: Slower than Cloud Run (3-5 seconds)
- ❌ **Complex Networking**: VPC, subnets, security groups
- ❌ **Team Unfamiliarity**: Learning curve

**Why Rejected**: Cold starts, team prefers GCP

---

### Option 3: Azure App Service

**Pros**:
- ✅ **Auto-Scaling**: Built-in
- ✅ **Managed Services**: Azure SQL

**Cons**:
- ❌ **Team Unfamiliarity**: No Azure experience
- ❌ **Higher Costs**: ~2x GCP for similar resources

**Why Rejected**: Team unfamiliarity, higher costs

---

### Option 4: Self-Hosted Kubernetes (GKE)

**Pros**:
- ✅ **Full Control**: Custom configurations
- ✅ **No Vendor Lock-In**: Portable to any Kubernetes

**Cons**:
- ❌ **Operational Overhead**: Cluster management, monitoring
- ❌ **Higher Costs**: Always-on nodes ($500+/month)
- ❌ **Complexity**: Manual scaling, load balancing

**Why Rejected**: Operational overhead, higher costs

---

## Decision Outcome

**Chosen Option**: **GCP Cloud Run** (Option 1)

### Rationale

1. **Serverless**: Auto-scaling eliminates manual capacity planning
2. **Pay-per-Use**: No idle costs when traffic is low
3. **Team Familiarity**: Team has GCP experience
4. **Fast Deployment**: Deploy in 30 seconds
5. **Managed Services**: Cloud SQL handles backups, HA

---

## Consequences

### Positive ✅

1. **Auto-Scaling**: Handle traffic spikes automatically
2. **Cost-Effective**: ~$320/month for 1000 users
3. **Fast Deployment**: Deploy new versions in <5 minutes
4. **High Availability**: 99.95% uptime SLA

### Negative ⚠️

1. **Cold Starts**: First request takes 1-2 seconds
   - **Mitigation**: Set min instances to 1

2. **Vendor Lock-In**: GCP-specific features
   - **Mitigation**: Containerized apps are portable

---

## Related Decisions

- **ADR-005**: [FastAPI Backend](ADR-005-fastapi-over-flask-django.md)
- **ADR-006**: [Next.js Frontend](ADR-006-react-nextjs-frontend.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership, DevOps Team

---

**Made with ❤️ by CODITECT Engineering**
