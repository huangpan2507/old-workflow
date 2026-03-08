# Policy Approval Rejection Immediate Fail Fix

## Problem Description

When a required step in the policy approval workflow was rejected, the system would still continue to the next step instead of immediately terminating the workflow. This caused the approval flow to show subsequent steps as "PENDING" even though the workflow should have been terminated at the first rejection.

## Root Cause Analysis

In the `process_approval_decision` method in `approval_workflow_service.py`, the logic was:

1. When a step was completed and rejected, it would set `step.status = "rejected"`
2. Then check if the entire workflow was completed using `_check_workflow_completion`
3. If the workflow was not completed (because subsequent steps hadn't started yet), it would unconditionally call `_move_to_next_step`, which would activate the next step

This meant that even if Step 1 (a required step) was rejected, the system would still move to Step 2 and show it as "PENDING", which is incorrect behavior.

## Correct Logic

According to approval workflow design principles:

1. **Required Step Rejected**: If any required step is rejected, the workflow should immediately terminate and be marked as "failed"
2. **Approved Step**: Only when a step is approved should the workflow continue to the next step
3. **Optional Step Rejected**: If an optional step is rejected, the workflow can continue but won't activate the next step

## Fix Implementation

### Updated `process_approval_decision` Method

**Before:**
```python
step.completed_at = current_time
instance.completed_steps += 1

# Check if workflow is completed
workflow_completed = self._check_workflow_completion(instance)
if workflow_completed:
    required_steps_rejected = self._check_required_steps_rejected(instance)
    if required_steps_rejected:
        instance.status = "failed"
        instance.completed_at = current_time
        result = {"status": "workflow_failed", "reason": "One or more required steps were rejected"}
    else:
        instance.status = "completed"
        instance.completed_at = current_time
        result = {"status": "workflow_completed", "instance_id": instance.id}
else:
    # Move to next step
    self._move_to_next_step(instance)
    # ...
```

**After:**
```python
step.completed_at = current_time
instance.completed_steps += 1

# If step is rejected and it's required, immediately fail the workflow
if step.status == "rejected" and step.is_required:
    instance.status = "failed"
    instance.completed_at = current_time
    result = {"status": "workflow_failed", "reason": "Required step was rejected"}
else:
    # Check if workflow is completed
    workflow_completed = self._check_workflow_completion(instance)
    if workflow_completed:
        required_steps_rejected = self._check_required_steps_rejected(instance)
        if required_steps_rejected:
            instance.status = "failed"
            instance.completed_at = current_time
            result = {"status": "workflow_failed", "reason": "One or more required steps were rejected"}
        else:
            instance.status = "completed"
            instance.completed_at = current_time
            result = {"status": "workflow_completed", "instance_id": instance.id}
    else:
        # Only move to next step if current step was approved
        if step.status == "approved":
            # Move to next step
            self._move_to_next_step(instance)
            # ...
        else:
            # Step rejected but not required, workflow continues but no next step
            result = {
                "status": "step_completed", 
                "instance_id": instance.id,
                "next_approver_id": None,
                "next_step_order": None
            }
```

## Expected Behavior After Fix

### Scenario 1: Required Step Rejected
- **Step 1**: REQUIRED, REJECTED ❌
- **Step 2**: NOT_STARTED (should not be activated)
- **Workflow Status**: FAILED ✅
- **Policy Status**: REJECTED ✅

### Scenario 2: Required Step Approved
- **Step 1**: REQUIRED, APPROVED ✅
- **Step 2**: PENDING (activated) ✅
- **Workflow Status**: ACTIVE ✅
- **Policy Status**: PENDING ✅

### Scenario 3: Optional Step Rejected
- **Step 1**: OPTIONAL, REJECTED ❌
- **Step 2**: NOT_STARTED (not activated, but workflow continues)
- **Workflow Status**: ACTIVE ✅
- **Policy Status**: PENDING ✅

## Files Modified

- `backend/app/services/approval_workflow_service.py`
  - Updated `process_approval_decision` method to immediately fail workflow when a required step is rejected
  - Added check to only move to next step when current step is approved

## Integration with Policy Status

The `PolicyApprovalAdapter.process_approval_decision_for_policy` method already handles the `workflow_failed` status correctly:

```python
elif result.get("status") == "workflow_failed":
    policy.approval_status = "rejected"
    policy.current_approver_id = None
```

This ensures that when a required step is rejected and the workflow fails, the policy status is automatically updated to "rejected".

## Validation

To validate the fix:

1. **Test Case 1**: Submit a policy with 2 required steps, reject Step 1
   - Expected: Workflow should fail immediately
   - Expected: Step 2 should remain NOT_STARTED
   - Expected: Policy status should be REJECTED

2. **Test Case 2**: Submit a policy with 2 required steps, approve Step 1
   - Expected: Step 2 should be activated and show as PENDING
   - Expected: Policy status should remain PENDING

3. **Test Case 3**: Submit a policy with 1 required step and 1 optional step, reject the optional step
   - Expected: Workflow should continue
   - Expected: Required step should still be processable

