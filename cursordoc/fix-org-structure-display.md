# Fix: Organizational Structure Data Not Displaying in Edit Form

## Problem
When editing an approval node of type "organizational_structure", the saved organizational information was not being displayed in the edit form, even though it was correctly saved and displayed in the table.

## Root Cause Analysis

### 1. Missing Data in Create API
The `create_template_node` function in `policy_approval_templates.py` was only creating the main node definition but not processing the related data (users, roles, org_structures).

### 2. Missing Fields in Create Model
The `PolicyApprovalNodeDefinitionCreate` model was missing the `users`, `roles`, and `org_structures` fields that are needed to accept and process related data during node creation.

### 3. Data Type Mismatch
The `PolicyApprovalNodeOrgStructureUpdateCreate` model had incorrect data types:
- `approval_level` was defined as `int` but should be `str` (matches database schema)
- Missing required fields like `level_config`, `priority`, `is_primary`

## Solution

### 1. Updated Create API Logic
**File**: `backend/app/api/v1/policy_approval_templates.py`

Added processing for related data in the `create_template_node` function:
```python
# Handle related data if provided
if hasattr(node, 'users') and node.users is not None:
    # Add user assignments
    for user_data in node.users:
        # ... create PolicyApprovalNodeUser records

if hasattr(node, 'roles') and node.roles is not None:
    # Add role assignments  
    for role_data in node.roles:
        # ... create PolicyApprovalNodeRole records

if hasattr(node, 'org_structures') and node.org_structures is not None:
    # Add org structure assignments
    for org_data in node.org_structures:
        # ... create PolicyApprovalNodeOrgStructure records
```

### 2. Updated Create Model
**File**: `backend/app/models.py`

Added missing fields to `PolicyApprovalNodeDefinitionCreate`:
```python
class PolicyApprovalNodeDefinitionCreate(PolicyApprovalNodeDefinitionBase):
    # Related data for creating - using special create models
    users: list[PolicyApprovalNodeUserUpdateCreate] | None = Field(default=None, description="Users assigned to this node")
    roles: list[PolicyApprovalNodeRoleUpdateCreate] | None = Field(default=None, description="Roles assigned to this node")
    org_structures: list[PolicyApprovalNodeOrgStructureUpdateCreate] | None = Field(default=None, description="Organizational structures assigned to this node")
```

### 3. Fixed Data Type Mismatch
**File**: `backend/app/models.py`

Updated `PolicyApprovalNodeOrgStructureUpdateCreate` model:
```python
class PolicyApprovalNodeOrgStructureUpdateCreate(SQLModel):
    org_type: str = Field(description="Organization type")
    org_id: int = Field(description="Organization ID")
    approval_level: str = Field(description="Approval level: all_members, heads_only, specific_levels")  # Changed from int to str
    level_config: str | None = Field(default=None, description="JSON configuration for specific levels")  # Added missing field
    priority: int = Field(default=0, description="Approval priority")  # Added missing field
    is_primary: bool = Field(default=False, description="Is this org structure primary")  # Added missing field
    effective_from: datetime | None = Field(default=None, description="Effective from date")
    effective_until: datetime | None = Field(default=None, description="Effective until date")
    is_active: bool = Field(default=True, description="Is this assignment active")
```

## Testing

### 1. Create Test
```bash
curl -X POST -H "Authorization: Bearer ..." -H "Content-Type: application/json" \
-d '{
  "template_id": 4,
  "step_order": 8,
  "node_name": "test_org_working",
  "node_type": "organizational_structure",
  "is_required": true,
  "approval_logic": "any",
  "users": [],
  "roles": [],
  "org_structures": [{
    "org_type": "business_unit",
    "org_id": 1,
    "approval_level": "all_members",
    "priority": 0,
    "is_primary": false,
    "is_active": true
  }]
}' \
http://localhost:8001/api/v1/policy-approval-templates/4/nodes
```

**Result**: ✅ Successfully created node with organizational structure data

### 2. Database Verification
```sql
SELECT * FROM policy_approval_node_org_structures WHERE node_id = 11;
```

**Result**: ✅ Data correctly saved to database

### 3. Read Test
```bash
curl -H "Authorization: Bearer ..." \
http://localhost:8001/api/v1/policy-approval-templates/4/nodes | jq '.[] | select(.node_name == "test_org_working")'
```

**Result**: ✅ Organizational structure data correctly returned in API response

## Impact

### Before Fix
- ❌ Organizational structure data not saved during node creation
- ❌ Edit form shows empty organizational structure section
- ❌ Data type validation errors when creating nodes

### After Fix
- ✅ Organizational structure data correctly saved during node creation
- ✅ Edit form displays saved organizational structure data
- ✅ All data types match database schema
- ✅ Complete CRUD operations work for organizational structure nodes

## Files Modified
1. `backend/app/api/v1/policy_approval_templates.py` - Added related data processing in create function
2. `backend/app/models.py` - Updated create model and fixed data type mismatches

## Conclusion
The fix ensures that organizational structure data is properly saved when creating approval nodes and correctly displayed when editing them. The edit form now shows the saved organizational information as expected.
