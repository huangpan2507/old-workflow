# Deviation Resubmission Approval Instance Fix

## Problem Description

When a deviation request was rejected twice (Round 1 and Round 2), and then resubmitted for Round 3, the deviation status was updated to "pending" but no new approval tracking data (Round 3 instance) was created. This meant the deviation appeared to be pending approval but had no active approval workflow.

## Root Cause Analysis

The issue occurred in the `create_approval_instance_for_deviation` method in `deviation_approval_adapter.py`. When resubmitting a deviation:

1. The method would call `create_approval_instance` to create a new approval workflow instance
2. If the instance creation succeeded but flows couldn't be created (e.g., no approvers found), the instance might be created without flows
3. The deviation's `approval_instance_id` would be updated, but there would be no actual approval flows to track

Additionally, there was no validation to ensure:
- The instance was actually created successfully
- The instance had flows (indicated by `current_approver_id` being set)
- No active instance already existed for the deviation

## Solution

### 1. Added Active Instance Check

Before creating a new instance, check if an active instance already exists:

```python
# Check if there's an existing active instance (shouldn't happen, but handle it)
existing_instance = self.workflow_service.get_approval_instance("policy_deviation", deviation_id)
if existing_instance and existing_instance.status == "active":
    raise ValueError(f"Active approval instance already exists for deviation {deviation_id}")
```

### 2. Added Instance Validation

After creating the instance, validate that:
- The instance was created successfully
- The instance has a current approver (which indicates flows were created)

```python
# Verify instance was created successfully with flows
if not instance:
    raise ValueError(f"Approval instance creation returned None for deviation {deviation_id}")

# Verify instance has a current approver (which means flows were created)
if not instance.current_approver_id:
    raise ValueError(f"Approval instance created but no current approver found for deviation {deviation_id}")
```

### 3. Improved Error Handling

Added better error context when instance creation fails:

```python
try:
    instance = self.workflow_service.create_approval_instance(
        resource_type="policy_deviation",
        resource_id=deviation_id,
        template_id=template_id
    )
except ValueError as e:
    # Re-raise with more context
    raise ValueError(f"Failed to create approval instance for deviation {deviation_id}: {str(e)}") from e
```

## Expected Behavior After Fix

### Scenario: Resubmission After Rejection

1. **Round 1**: Deviation submitted → Instance created → Rejected → Instance status = "failed"
2. **Round 2**: Deviation resubmitted → New instance created → Rejected → Instance status = "failed"
3. **Round 3**: Deviation resubmitted → New instance created → Instance status = "active" → Flows created → Deviation status = "pending" ✅

### Validation Points

- ✅ No active instance exists before creating new one
- ✅ Instance is created successfully
- ✅ Instance has flows (current_approver_id is set)
- ✅ Deviation's approval_instance_id is updated
- ✅ Deviation's current_approver_id is updated
- ✅ Deviation status is "pending"

### Error Cases

If any validation fails:
- Exception is raised with clear error message
- Transaction is rolled back (in `submit_deviation_for_approval`)
- Deviation status remains "rejected" or "draft"
- No partial/incomplete approval instance is created

## Files Modified

- `backend/app/services/deviation_approval_adapter.py`
  - Added active instance check before creating new instance
  - Added instance validation after creation
  - Improved error handling with better context

## Related Issues

This fix ensures that when a deviation is resubmitted after rejection:
1. A new approval instance (Round 3) is properly created
2. Approval flows are created for the new instance
3. The deviation is correctly linked to the new instance
4. The approval history shows all rounds (Round 1, Round 2, Round 3)

## Testing

To test the fix:

1. Create a deviation request
2. Submit it for approval (Round 1)
3. Reject it
4. Resubmit it (Round 2)
5. Reject it again
6. Resubmit it (Round 3)
7. Verify:
   - Deviation status is "pending"
   - Approval instance exists with status "active"
   - Approval flows exist for Round 3
   - Approval history shows all three rounds

