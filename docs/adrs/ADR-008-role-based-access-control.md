# ADR-008: Role-Based Access Control (RBAC) - Project Intelligence Platform

```yaml
Document: ADR-008-project-intelligence-rbac
Version: 1.0.0
Purpose: Define 6 roles with granular permissions for multi-tenant access control
Audience: Engineering teams, security officers, compliance team, product managers
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-004 (Multi-Tenant Strategy), ADR-002 (PostgreSQL)
```

---

## Executive Summary

**Decision**: Implement **6 roles** with granular permissions using RBAC pattern.

**Roles**:
1. **Owner** - Full access (billing, delete org)
2. **Admin** - All except billing/delete org
3. **Member** - Read/write projects, checkpoints
4. **Viewer** - Read-only access
5. **Auditor** - Read-only + audit logs
6. **Executive** - Read-only + analytics dashboard

**Why RBAC**:
- ✅ **Enterprise-Ready**: Meets SOC2, GDPR, HIPAA requirements
- ✅ **Granular Permissions**: Fine-grained access control
- ✅ **Flexible**: Add new permissions without schema changes
- ✅ **Audit-Friendly**: Track who did what

**Alternatives Rejected**: Simple user/admin (too coarse), Fine-grained permissions (too complex)

---

## Context and Problem Statement

### Requirements

CODITECT Project Intelligence Platform must support:

1. **Multiple Roles**: Different access levels (Owner, Admin, Member, Viewer, Auditor, Executive)
2. **Granular Permissions**: Control access to specific actions (create, read, update, delete)
3. **Multi-Tenant**: Users can have different roles in different organizations
4. **Audit Trail**: Track all permission-based actions
5. **Compliance**: SOC2, GDPR, HIPAA require access controls

---

## Decision Drivers

1. **Enterprise Requirements** - Multiple roles for different responsibilities
2. **Granular Permissions** - Fine-grained access control
3. **Audit Trail** - Track all actions for compliance
4. **Flexibility** - Add new permissions without schema changes
5. **Compliance** - SOC2, GDPR, HIPAA compatible

---

## Considered Options

### Option 1: **6 Roles with Granular Permissions** (SELECTED ✅)

**Roles**:

| Role | Description | Use Case |
|------|-------------|----------|
| **Owner** | Full access, billing, delete org | Founders, CTOs |
| **Admin** | All except billing/delete org | Engineering Managers |
| **Member** | Read/write projects, checkpoints | Software Engineers |
| **Viewer** | Read-only access | Stakeholders, PMs |
| **Auditor** | Read-only + audit logs | Compliance, Security |
| **Executive** | Read-only + analytics dashboard | CEO, CFO, Board |

**Permission Matrix**:
```python
PERMISSIONS = {
    "owner": ["*"],  # All permissions

    "admin": [
        "projects:create", "projects:read", "projects:update", "projects:delete",
        "checkpoints:create", "checkpoints:read", "checkpoints:update", "checkpoints:delete",
        "members:invite", "members:remove", "members:update_role",
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

**Implementation**:
```python
from functools import wraps
from fastapi import HTTPException, Depends

def require_permission(permission: str):
    """Decorator to enforce permission-based access control."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            # Get user's role in organization
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

**Pros**:
- ✅ **Enterprise-Ready**: Meets compliance requirements
- ✅ **Granular**: Fine-grained access control
- ✅ **Flexible**: Add new permissions without schema changes
- ✅ **Audit Trail**: All actions logged
- ✅ **Multi-Tenant**: Users can have different roles in different orgs

**Cons**:
- ⚠️ **Permission Matrix Complexity**: Must maintain permission mappings
- ⚠️ **Testing Overhead**: Test all role combinations

---

### Option 2: Simple User/Admin

**Roles**:
- **User**: Basic access
- **Admin**: Full access

**Pros**:
- ✅ **Simple**: Easy to implement

**Cons**:
- ❌ **Too Coarse**: Can't distinguish between viewer, member, auditor
- ❌ **Not Enterprise-Ready**: Doesn't meet compliance requirements

**Why Rejected**: Too coarse for enterprise use cases

---

### Option 3: Fine-Grained Permissions (AWS IAM-style)

**Example**:
```json
{
  "user_id": "123",
  "permissions": [
    "project:abc:read",
    "project:abc:update",
    "checkpoint:xyz:delete"
  ]
}
```

**Pros**:
- ✅ **Maximum Flexibility**: Per-resource permissions

**Cons**:
- ❌ **Too Complex**: Hard to manage at scale
- ❌ **Performance**: Checking hundreds of permissions per request
- ❌ **User Confusion**: Users don't understand granular permissions

**Why Rejected**: Too complex, overkill for our use case

---

## Decision Outcome

**Chosen Option**: **6 Roles with Granular Permissions** (Option 1)

### Rationale

1. **Enterprise Requirements**: Meets SOC2, GDPR, HIPAA
2. **Right Balance**: Not too simple (user/admin), not too complex (AWS IAM)
3. **Flexible**: Add new permissions without schema changes
4. **Audit-Friendly**: All actions logged with role context

---

## Consequences

### Positive ✅

1. **Compliance-Ready**: SOC2, GDPR, HIPAA compatible
2. **Flexible**: Add new roles/permissions without database changes
3. **Audit Trail**: Complete log of who did what
4. **Multi-Tenant**: Users can have different roles in different orgs

### Negative ⚠️

1. **Permission Matrix Complexity**: Must maintain mappings
   - **Mitigation**: Code generation from YAML schema

2. **Testing Overhead**: Test all role combinations
   - **Mitigation**: Automated permission matrix tests

---

## Implementation Details

### Database Schema

```sql
CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL CHECK (role IN ('owner', 'admin', 'member', 'viewer', 'auditor', 'executive')),
    permissions JSONB DEFAULT '{}', -- Custom permissions (future)
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(organization_id, user_id)
);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL, -- 'projects:delete', 'members:invite'
    resource_type VARCHAR(50), -- 'project', 'checkpoint'
    resource_id UUID,
    ip_address INET,
    user_agent TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_log_org_created ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at DESC);
```

### Frontend Role-Based UI

```typescript
// components/ProjectCard.tsx
function ProjectCard({ project }: { project: Project }) {
  const { user } = useAuth();
  const userRole = user.organizations.find(o => o.id === project.organization_id)?.role;

  return (
    <Card>
      <CardHeader>
        <Heading>{project.name}</Heading>
      </CardHeader>

      <CardBody>
        <Text>{project.description}</Text>
      </CardBody>

      <CardFooter>
        {/* Show delete button only for owner/admin */}
        {['owner', 'admin'].includes(userRole) && (
          <Button colorScheme="red" onClick={() => deleteProject(project.id)}>
            Delete
          </Button>
        )}

        {/* Show edit button for owner/admin/member */}
        {['owner', 'admin', 'member'].includes(userRole) && (
          <Button onClick={() => editProject(project.id)}>
            Edit
          </Button>
        )}

        {/* View button available to all roles */}
        <Button onClick={() => viewProject(project.id)}>
          View
        </Button>
      </CardFooter>
    </Card>
  );
}
```

---

## Related Decisions

- **ADR-004**: [Multi-Tenant Strategy](ADR-004-multi-tenant-strategy.md)
- **ADR-002**: [PostgreSQL as Primary Database](ADR-002-postgresql-as-primary-database.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership, Security Team, Compliance Team

---

**Made with ❤️ by CODITECT Engineering**
