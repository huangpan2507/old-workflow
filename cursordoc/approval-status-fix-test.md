# Policy Approval Status Fix Test

## Problem Description

The policy approval workflow had a bug where policies were marked as "REJECTED" even when there were still PENDING steps in the approval flow. This happened because the `_check_workflow_completion` method only checked required steps, ignoring non-required steps that were still pending.

## Root Cause Analysis

From the approval history shown in the images:

1. **STEP 1**: Mixed results (one reject, one approve)
2. **STEP 2**: REJECTED 
3. **STEP 3**: REJECTED
4. **STEP 5**: PENDING (this step was ignored in the old logic)

The old `_check_workflow_completion` method only checked `is_required == True` steps, so if STEP 5 was marked as non-required (`is_required == False`), the workflow would be considered completed even though STEP 5 was still pending.

## Fix Implementation

### 1. Updated `_check_workflow_completion` Method

**Before:**
```python
def _check_workflow_completion(self, instance: PolicyApprovalInstance) -> bool:
    """Check if workflow is completed by checking only required steps"""
    # Get all required steps
    required_steps = self.session.exec(
        select(PolicyApprovalStep).where(
            and_(
                PolicyApprovalStep.instance_id == instance.id,
                PolicyApprovalStep.is_required == True
            )
        )
    ).all()
    
    # Check if all required steps are completed
    for step in required_steps:
        if step.status not in ["approved", "rejected"]:
            return False
    
    return True
```

**After:**
```python
def _check_workflow_completion(self, instance: PolicyApprovalInstance) -> bool:
    """Check if workflow is completed by checking all steps"""
    # Get all steps for this instance
    all_steps = self.session.exec(
        select(PolicyApprovalStep).where(
            PolicyApprovalStep.instance_id == instance.id
        )
    ).all()
    
    # Check if all steps are completed (approved, rejected, or skipped)
    for step in all_steps:
        if step.status not in ["approved", "rejected", "skipped"]:
            return False
    
    return True
```

### 2. Added `_check_required_steps_rejected` Method

```python
def _check_required_steps_rejected(self, instance: PolicyApprovalInstance) -> bool:
    """Check if any required steps were rejected"""
    # Get all required steps
    required_steps = self.session.exec(
        select(PolicyApprovalStep).where(
            and_(
                PolicyApprovalStep.instance_id == instance.id,
                PolicyApprovalStep.is_required == True
            )
        )
    ).all()
    
    # Check if any required step was rejected
    for step in required_steps:
        if step.status == "rejected":
            return True
    
    return False
```

### 3. Updated Policy Status Logic

**Before:**
```python
if required_steps_completed:
    if instance.rejected_steps > 0:
        # Set to rejected
    else:
        # Set to approved
else:
    # Move to next step
```

**After:**
```python
if workflow_completed:
    if required_steps_rejected:
        # Set to rejected
    else:
        # Set to approved
else:
    # Keep as pending and move to next step
```

## Expected Behavior After Fix

For the policy shown in the images:

1. **STEP 1**: Mixed (one reject, one approve) - Step completed
2. **STEP 2**: REJECTED - Step completed  
3. **STEP 3**: REJECTED - Step completed
4. **STEP 5**: PENDING - Step not completed

**Result**: Since STEP 5 is still PENDING, `_check_workflow_completion` will return `False`, and the policy status will be set to "PENDING" instead of "REJECTED".

## Test Cases

### Test Case 1: All Steps Completed, No Required Steps Rejected
- **Expected**: Policy status = "APPROVED"

### Test Case 2: All Steps Completed, Some Required Steps Rejected  
- **Expected**: Policy status = "REJECTED"

### Test Case 3: Some Steps Still Pending (regardless of required status)
- **Expected**: Policy status = "PENDING"

### Test Case 4: Mixed Scenario (like the one in images)
- STEP 1: Mixed results
- STEP 2: REJECTED
- STEP 3: REJECTED  
- STEP 5: PENDING
- **Expected**: Policy status = "PENDING" (because STEP 5 is still pending)

## Files Modified

- `backend/app/services/policy_approval_service.py`
  - Updated `_check_workflow_completion` method
  - Added `_check_required_steps_rejected` method
  - Updated policy status update logic in `process_approval_decision` method

## Validation

To validate the fix:

1. Check the policy in the UI - it should now show "PENDING" instead of "REJECTED"
2. The approval history should still show STEP 5 as PENDING
3. The policy should be accessible to the appropriate approvers for STEP 5
4. Once STEP 5 is completed (approved/rejected), the policy status should be updated accordingly
