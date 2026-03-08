# Role Display Fix

## Problem
In the Approval Node Configuration modal, when editing existing approval nodes, the role names were not being displayed. Instead, only "Primary 0" was shown, indicating that the role information was missing.

## Root Cause
The issue was in the backend API where role assignments were loaded without their corresponding role details:

1. **Model Issue**: `PolicyApprovalNodeRoleRead` model was missing a `role` field to store role information
2. **API Issue**: Backend APIs (`list_template_nodes` and `get_node_definition`) were not loading role details when fetching role assignments

## Solution

### 1. Updated Model (`backend/app/models.py`)
```python
class PolicyApprovalNodeRoleRead(PolicyApprovalNodeRoleBase):
    id: int
    assigned_at: datetime
    assigned_by: uuid.UUID | None

    role: Optional["RolePublic"] = Field(default=None, description="Role information")
```

### 2. Updated Backend APIs

#### `backend/app/api/v1/policy_approval_templates.py`
- Modified `list_template_nodes` function to load role details for each role assignment
- Added logic to create `RolePublic` objects and include them in `PolicyApprovalNodeRoleRead` objects

#### `backend/app/api/v1/approval_nodes.py`
- Modified `get_node_definition` function to load role details for each role assignment
- Added necessary imports for `Role` and `RolePublic` models

### 3. Frontend Code
The frontend code was already correct and expected the `role.role?.name` and `role.role?.code` fields to be available.

## Files Modified
1. `backend/app/models.py` - Added `role` field to `PolicyApprovalNodeRoleRead`
2. `backend/app/api/v1/policy_approval_templates.py` - Updated role loading logic
3. `backend/app/api/v1/approval_nodes.py` - Updated role loading logic

## Testing
To test the fix:
1. Start the backend server
2. Open the Approval Node Configuration modal
3. Edit an existing approval node that has role assignments
4. Verify that role names and codes are now displayed correctly in the "Selected Roles" section

## Expected Result
Role assignments should now display:
- Role name (e.g., "Manager", "Admin")
- Role code (e.g., "MGR", "ADM")
- Primary toggle
- Priority number
- Delete button

Instead of just showing "Primary 0" without the role name.
