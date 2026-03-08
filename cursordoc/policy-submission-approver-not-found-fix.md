# Policy Submission Approval Error Fix

## Problem Description

When submitting a policy for approval, the system throws an error:
```
Failed to submit policy for approval: No approvers found for step Department Head Approval
```

## Root Cause

The error occurs because the system cannot find a Group-level function head for the policy's responsible function role. The approval workflow requires:

1. **Policy must have a responsible function role** (`responsible_function_role_id`)
2. **Group must exist** (EIM Global group with code "eim_global")
3. **OrganizationFunctionHead record must exist** with:
   - `group_id` matching the EIM Global group
   - `function_role_id` matching the policy's responsible function role
   - `is_active == True`
   - `effective_date` is null or <= today
   - `expiry_date` is null or >= today

If any of these conditions are not met, the system cannot find an approver for the "Department Head Approval" step.

## Solution

### 1. Improved Error Messages

Enhanced the error message in `approval_workflow_service.py` to provide better diagnostics:

- **Before**: Generic message "No Group-level X function head found"
- **After**: Detailed message with diagnostics:
  - Shows policy name and function role name
  - If records exist but are inactive/expired/future: Shows count of each type
  - Provides clear path to fix: "Admin > Organization Settings > Function Head Management"

### 2. How to Fix the Issue

**Option A: Add Missing OrganizationFunctionHead Record**

1. Navigate to **Admin > Organization Settings > Function Head Management**
2. Click "Add Function Head" or "Edit" if a record exists
3. Set:
   - **Organization Level**: Group
   - **Group**: EIM Global
   - **Function Role**: Select the function role that matches the policy's responsible function role
   - **User**: Select an active user to be the function head
   - **Effective Date**: Leave blank or set to today or earlier
   - **Expiry Date**: Leave blank (no expiry) or set to a future date
   - **Status**: Active (checked)

4. Save the record

**Option B: Check Existing Records**

If a record exists but is inactive or expired:
1. Go to **Admin > Organization Settings > Function Head Management**
2. Find the record for the function role
3. Edit it:
   - Set **Status** to Active
   - Update **Effective Date** if needed (must be <= today)
   - Update **Expiry Date** if needed (must be >= today or null)

### 3. Code Changes

**File**: `backend/app/services/approval_workflow_service.py`

**Method**: `_find_policy_group_function_role_head()`

**Changes**:
- Added diagnostic checks when no active record is found
- Checks for inactive, expired, or future-dated records
- Provides detailed error messages with counts and guidance

## Testing

After fixing the OrganizationFunctionHead record:

1. Create or edit a policy
2. Set the responsible department (function role)
3. Try to submit the policy for approval
4. The submission should succeed without the "No approvers found" error

## Prevention

To prevent this issue in the future:

1. **Set up OrganizationFunctionHead records** for all function roles used in policies
2. **Monitor expiry dates** and update them before they expire
3. **Use bulk import** if multiple function heads need to be set up
4. **Regular audit** of OrganizationFunctionHead records to ensure they are active and valid

## Related Files

- `backend/app/services/approval_workflow_service.py` - Core approval workflow logic
- `backend/app/services/policy_approval_adapter.py` - Policy-specific adapter
- `backend/app/api/v1/policies.py` - Policy API endpoints
- `backend/app/core/database/default_data/approval_workflow_templates.py` - Approval workflow templates
- `backend/app/api/v1/organization.py` - OrganizationFunctionHead API endpoints

