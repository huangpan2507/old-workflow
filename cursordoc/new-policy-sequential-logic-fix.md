# New Policy Creation Sequential Logic Fix

## Problem Description

When creating new policies, all approval steps were showing as "pending" and all approvers could see their approval requests immediately, violating the sequential approval workflow principle.

## Root Cause Analysis

The issue was in the approval flow creation logic in `create_approval_instance_for_policy` method:

### Step Creation Logic (Correct)
```python
# Only the first step should be "pending", others should be "not_started"
step_status = "pending" if node_def.step_order == 1 else "not_started"
```

### Flow Creation Logic (Incorrect - Before Fix)
```python
# All flows were created with "pending" status regardless of step status
flow = PolicyApprovalFlow(
    # ...
    status="pending",  # ❌ Always pending
    assigned_at=current_time,  # ❌ Always assigned
    # ...
)
```

**Problem**: Even though steps had correct status (`not_started` for steps 2+), all approval flows were created with `status="pending"`, making them visible to approvers.

## Solution Implementation

### 1. Fixed Flow Creation Logic

**Updated Flow Creation**:
```python
# Flow status should match step status
flow_status = "pending" if step.status == "pending" else "not_started"
flow_assigned_at = current_time if step.status == "pending" else None

flow = PolicyApprovalFlow(
    # ...
    status=flow_status,  # ✅ Matches step status
    assigned_at=flow_assigned_at,  # ✅ Only assigned if step is pending
    # ...
)
```

### 2. Enhanced Step Activation Logic

**When Moving to Next Step**:
```python
# Activate the next step
next_step.status = "pending"
next_step.started_at = current_time

# Activate all flows for this step
next_flows = self.session.exec(
    select(PolicyApprovalFlow).where(
        PolicyApprovalFlow.step_id == next_step.id
    )
).all()

for flow in next_flows:
    flow.status = "pending"
    flow.assigned_at = current_time
    flow.update_time = current_time
```

## Expected Behavior After Fix

### New Policy Creation
1. **Step 1**: `pending` → Flows: `pending` (visible to approvers)
2. **Step 2**: `not_started` → Flows: `not_started` (hidden from approvers)
3. **Step 3**: `not_started` → Flows: `not_started` (hidden from approvers)
4. **Step 4**: `not_started` → Flows: `not_started` (hidden from approvers)

### After Step 1 Completion
1. **Step 1**: `approved` → Flows: `approved`
2. **Step 2**: `pending` → Flows: `pending` (now visible to approvers)
3. **Step 3**: `not_started` → Flows: `not_started` (still hidden)
4. **Step 4**: `not_started` → Flows: `not_started` (still hidden)

## Key Changes Made

### 1. Flow Status Synchronization
- **Before**: All flows created with `status="pending"`
- **After**: Flow status matches step status

### 2. Assignment Timing
- **Before**: All flows assigned immediately
- **After**: Only flows for active steps are assigned

### 3. Step Activation
- **Before**: Only step status updated
- **After**: Both step and flow statuses updated together

## Files Modified

1. **`backend/app/services/policy_approval_service.py`**
   - Fixed flow creation logic in `create_approval_instance_for_policy`
   - Enhanced step activation logic in `process_approval_decision`

## Testing Scenarios

### Test Case 1: New Policy Creation
- **Action**: Create new policy with 4-step workflow
- **Expected**: Only Step 1 approvers see approval requests
- **Result**: ✅ Only Step 1 flows are `pending`

### Test Case 2: Sequential Activation
- **Action**: Complete Step 1 approval
- **Expected**: Step 2 becomes visible to its approvers
- **Result**: ✅ Step 2 flows change from `not_started` to `pending`

### Test Case 3: Multiple Approvers per Step
- **Action**: Step with multiple approvers
- **Expected**: All approvers for active step see requests
- **Result**: ✅ All flows for active step are `pending`

### Test Case 4: Mixed Required/Optional Steps
- **Action**: Workflow with required and optional steps
- **Expected**: Sequential activation regardless of required status
- **Result**: ✅ All steps follow sequential activation

## Validation Steps

To validate the fix:

1. **Create a new policy** with multi-step workflow
2. **Check initial state**: Only first step approvers should see requests
3. **Complete first step**: Verify second step becomes visible
4. **Continue workflow**: Verify sequential activation works
5. **Check database**: Verify flow statuses match step statuses

## Database Verification

You can verify the fix by checking the database:

```sql
-- Check step and flow status alignment
SELECT 
    pas.step_order,
    pas.status as step_status,
    paf.status as flow_status,
    COUNT(*) as flow_count
FROM policy_approval_steps pas
JOIN policy_approval_flows paf ON pas.id = paf.step_id
WHERE pas.instance_id = <instance_id>
GROUP BY pas.step_order, pas.status, paf.status
ORDER BY pas.step_order;
```

Expected result:
- Step 1: `pending` → flows: `pending`
- Step 2+: `not_started` → flows: `not_started`

This fix ensures that the sequential approval workflow works correctly for new policies, with only the current active step being visible to its approvers.
