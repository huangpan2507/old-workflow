# Policy Approval ANY Logic Fix Test

## Problem Description

The policy approval workflow had a bug where steps with "ANY" approval logic were not being marked as completed even when one approver approved. This caused policies to remain in PENDING status even when all required steps were actually completed.

## Root Cause Analysis

From the test case shown in the image:

1. **STEP 1** has two approvers:
   - `admin@example.com` - APPROVE ✅
   - `qsfan@qq.com` - PENDING ⏳

2. **STEP 1** has LOGIC = "ANY", meaning only one approval is needed

3. **STEP 5** has one approver:
   - `DCB Head Head` - APPROVE ✅

4. Both STEP 1 and STEP 5 are marked as REQUIRED = YES

5. **Expected Result**: Policy should be APPROVED (both required steps completed)
6. **Actual Result**: Policy remains PENDING

## Root Cause

The issue was in the step completion logic. The `_check_step_completion` method correctly identified that STEP 1 was completed (because `actual_approved > 0` for ANY logic), but the step status update logic had a flaw:

**Problem**: The step status was being updated every time a new approval decision was processed, even if the step was already completed. This could lead to:
- Duplicate counting of completed steps
- Incorrect step status updates
- Workflow completion not being properly detected

## Fix Implementation

### Updated Step Status Update Logic

**Before:**
```python
if step_completed:
    # Always update step status and counters
    if actual_approved > 0:
        step.status = "approved"
        instance.approved_steps += 1
    else:
        step.status = "rejected"
        instance.rejected_steps += 1
    
    step.completed_at = current_time
    instance.completed_steps += 1
```

**After:**
```python
if step_completed:
    # Only update step status if it's not already completed
    if step.status not in ["approved", "rejected"]:
        # Determine step status based on actual flows
        flows = self.session.exec(
            select(PolicyApprovalFlow).where(PolicyApprovalFlow.step_id == step.id)
        ).all()
        
        actual_approved = len([f for f in flows if f.status == "approve"])
        actual_rejected = len([f for f in flows if f.status == "reject"])
        
        if actual_approved > 0:
            step.status = "approved"
            instance.approved_steps += 1
        else:
            step.status = "rejected"
            instance.rejected_steps += 1
        
        step.completed_at = current_time
        instance.completed_steps += 1
```

## Key Changes

1. **Added Status Check**: Only update step status if it's not already completed
2. **Prevented Duplicate Counting**: Avoid incrementing counters multiple times for the same step
3. **Maintained Logic Integrity**: The `_check_step_completion` method remains unchanged as it was working correctly

## Expected Behavior After Fix

For the test case in the image:

1. **STEP 1**: 
   - `admin@example.com` approves → Step completion checked → Step marked as "approved"
   - `qsfan@qq.com` still pending → No further step status updates (already completed)

2. **STEP 5**:
   - `DCB Head Head` approves → Step completion checked → Step marked as "approved"

3. **Workflow Completion**:
   - All steps completed → `_check_workflow_completion` returns `True`
   - No required steps rejected → Policy status set to "APPROVED"

## Test Cases

### Test Case 1: ANY Logic with One Approval
- **Setup**: Step with 2 approvers, LOGIC = "ANY"
- **Action**: First approver approves
- **Expected**: Step status = "approved", workflow continues

### Test Case 2: ANY Logic with Multiple Approvals
- **Setup**: Step with 2 approvers, LOGIC = "ANY"  
- **Action**: First approver approves, second approver approves later
- **Expected**: Step status remains "approved", no duplicate counting

### Test Case 3: ALL Logic
- **Setup**: Step with 2 approvers, LOGIC = "ALL"
- **Action**: First approver approves
- **Expected**: Step status remains "pending" until all approvers approve

### Test Case 4: Mixed Required/Non-Required Steps
- **Setup**: Required step completed, non-required step pending
- **Expected**: Policy status = "PENDING" (from previous fix)

## Files Modified

- `backend/app/services/policy_approval_service.py`
  - Updated step status update logic in `process_approval_decision` method
  - Added check to prevent duplicate step status updates

## Validation

To validate the fix:

1. **Test the specific case from the image**:
   - STEP 1: `admin@example.com` approves
   - STEP 5: `DCB Head Head` approves
   - Expected: Policy status should change to "APPROVED"

2. **Test edge cases**:
   - Multiple approvers for ANY logic
   - Mixed approval decisions
   - Non-required steps still pending

3. **Verify UI updates**:
   - Policy status should update immediately
   - Approval history should show correct step statuses
   - Pending approvers should still see their tasks
