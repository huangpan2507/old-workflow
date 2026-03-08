# Approval Workflow Refactoring Progress

## Completed Work

### Phase 1: Database Models ✓
- Created new generic approval workflow models in `backend/app/models.py`
- Added `ApprovalWorkflowTemplate` with `resource_type` field
- Added `ApprovalWorkflowNodeDefinition`, `ApprovalWorkflowNodeUser`, `ApprovalWorkflowNodeRole`
- Added `ApprovalWorkflowInstance` with generic `resource_type` + `resource_id` association
- Added `ApprovalWorkflowStep` and `ApprovalWorkflowFlow`
- Added all Pydantic schemas for the new models
- Updated `Policy` model to include `approval_instance_id` field

### Phase 2: Service Layer ✓
- Created `backend/app/services/approval_workflow_service.py`
  - Generic service supporting multiple resource types
  - Methods: `get_default_template()`, `create_approval_instance()`, `get_approval_instance()`, `process_approval_decision()`
- Created `backend/app/services/policy_approval_adapter.py`
  - Maintains backward compatibility
  - Wraps `ApprovalWorkflowService` for Policy-specific operations

### Phase 3: Database Migration ✓
- Created migration script `backend/app/alembic/versions/j1234567890e_refactor_approval_workflow_to_generic.py`
- Migrates data from old `policy_approval_*` tables to new `approval_workflow_*` tables
- Includes data migration for existing Policy approval instances

## Remaining Work

### Phase 4: API Layer (In Progress)

#### 4.1 Update Policy API
**File**: `backend/app/api/v1/policies.py`

Key changes needed:
- Update `/policies/{policy_id}/submit` endpoint to use `PolicyApprovalAdapter` instead of `PolicyApprovalService`
- Update `/policies/{policy_id}/approve` endpoint to use new service
- Update imports to include `PolicyApprovalAdapter`
- Maintain backward compatibility - keep same endpoint signatures

```python
# Change imports
from app.services.policy_approval_adapter import PolicyApprovalAdapter

# In submit endpoint, replace:
approval_service = PolicyApprovalService(session)
# With:
approval_service = PolicyApprovalAdapter(session)

# Rest of the logic remains the same
```

#### 4.2 Rename and Update Template Management API
**File**: `backend/app/api/v1/policy_approval_templates.py` → Rename to `approval_workflow_templates.py`

Key changes:
- Rename file to `approval_workflow_templates.py`
- Add `resource_type` parameter to all endpoints
- Update template listing to support filtering by resource type
- Update imports to use new `ApprovalWorkflowTemplate` models
- Keep backward compatibility by filtering for `resource_type='policy'` by default

```python
# Change imports
from app.models import (
    ApprovalWorkflowTemplate,
    ApprovalWorkflowTemplateCreate,
    ApprovalWorkflowTemplateUpdate,
    ApprovalWorkflowTemplateRead,
    # ... etc
)

# Update list endpoint
@router.get("/", response_model=List[ApprovalWorkflowTemplateRead])
async def list_approval_templates(
    *,
    session: SessionDep,
    current_user: CurrentUser,
    resource_type: str = Query("policy", description="Filter by resource type"),
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    active_only: bool = Query(True, description="Show only active templates")
):
    """List approval templates"""
    query = select(ApprovalWorkflowTemplate).where(
        and_(
            ApprovalWorkflowTemplate.resource_type == resource_type,
            ApprovalWorkflowTemplate.yn == True
        )
    )
    # ... rest of logic
```

#### 4.3 Update API Router
**File**: `backend/app/api/v1/__init__.py`

Key changes:
- Import the renamed router
- Update the router registration with new name

```python
from .approval_workflow_templates import router as approval_workflow_templates_router

# Update registration
api_router.include_router(
    approval_workflow_templates_router, 
    prefix="/approval-workflow-templates", 
    tags=["approval-workflow-templates"]
)
```

### Phase 5: Frontend Refactoring

#### 5.1 Update API Client
**File**: `frontend/src/utils/api.ts`

Key changes needed:
- Update base URLs for approval workflow endpoints
- Change `/policy-approval-templates` to `/approval-workflow-templates`
- Add `resource_type` parameter support
- Add new functions for generic approval workflow operations

```typescript
// Example change
export async function getApprovalTemplates(resourceType: string = 'policy'): Promise<ApprovalTemplate[]> {
  return apiRequest<ApprovalTemplate[]>(`/approval-workflow-templates/?resource_type=${resourceType}`)
}
```

#### 5.2 Update Components
Following components need updates to use new API endpoints:
- `frontend/src/components/PolicyUserInterface/workflow-config/ApprovalWorkflowConfig.tsx`
- `frontend/src/components/PolicyUserInterface/workflow-config/ApprovalNodeConfig.tsx`
- `frontend/src/components/PolicyUserInterface/my-approval/ApprovalDrawer.tsx`
- `frontend/src/components/PolicyUserInterface/policy-detail/PolicyDetailModal.tsx`
- `frontend/src/components/PolicyUserInterface/policy-detail/ApprovalFlowDiagram.tsx`

### Phase 6: Testing

**Backend Testing:**
```bash
cd /home/song/workspace/xproject/foundation/backend
# Run the migration
docker compose exec backend alembic upgrade head
# Test the new endpoints
pytest tests/ -v -k approval
```

**Frontend Testing:**
- Test Policy submission flow
- Test approval/rejection workflow
- Test template management
- Verify no UI/UX regressions

### Phase 7: Documentation

Create documentation file:
- Architecture overview
- Resource type integration guide
- API usage examples
- Migration notes

## Key Design Decisions

1. **Backward Compatibility**: 
   - Old `PolicyApprovalService` still exists but is deprecated
   - New `PolicyApprovalAdapter` wraps `ApprovalWorkflowService`
   - API endpoints maintain same signatures

2. **Database Migration**:
   - Old tables remain for now (soft delete only)
   - New tables created alongside old ones
   - Migration script copies data to new structure
   - Old tables can be dropped in a future migration

3. **Resource Type Implementation**:
   - Each resource type has its own default template
   - Queries filtered by `resource_type`
   - Generic service handles common logic
   - Resource-specific logic in adapters

## Next Steps

1. Complete API layer refactoring (Phase 4)
2. Update frontend API client and components (Phase 5)
3. Run comprehensive tests (Phase 6)
4. Create final documentation (Phase 7)
5. Delete old database tables in a follow-up migration

## Notes

- The migration preserves all existing Policy approval data
- The `approval_instance_id` field on Policy table provides quick lookup
- The `template_snapshot` field stores template state for audit trail
- All new code follows existing patterns and conventions
