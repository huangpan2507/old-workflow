# Sequential Approval Workflow Implementation

## Problem Description

The approval workflow was missing sequential processing logic. All approval steps were visible to their respective approvers simultaneously, rather than following a proper sequential flow where each step becomes active only after the previous step is completed.

## User's Issue

> "新建的政策需要经过4层审批节点，其中第一层和最后一层是必须的，正常审批过程是，第一层先经过审批，然后 2 3 4层才能看到自身的审批，现在是第一层还没有审批，最后一层就可以看到审批请求了"

**Translation**: "New policies need to go through 4 approval levels, where the first and last levels are required. The normal approval process should be: the first level gets approved first, then levels 2, 3, 4 can see their own approvals. But now the first level hasn't been approved yet, and the last level can already see approval requests."

## Root Cause Analysis

### Current (Incorrect) Behavior
- All approval steps were created with status "pending"
- All approvers could see their approval requests immediately
- No sequential activation logic

### Expected (Correct) Behavior
- Only Step 1 should be "pending" initially
- Steps 2, 3, 4 should be "not_started" initially
- Each step becomes "pending" only after the previous step is completed

## Solution Implementation

### 1. Enhanced Step Status Model

**Updated Step Status Definition**:
```python
# Step Status
status: str = Field(
    default="not_started", 
    description="Step status: not_started, pending, in_progress, approved, rejected, skipped"
)
```

**New Status Flow**:
- `not_started` → `pending` → `approved/rejected/skipped`
- Only `pending` steps are visible to approvers

### 2. Sequential Step Creation

**Before**:
```python
status="pending" if node_def.step_order == 1 else "pending"  # All steps pending
```

**After**:
```python
# Only the first step should be "pending", others should be "not_started"
step_status = "pending" if node_def.step_order == 1 else "not_started"
step_started_at = current_time if node_def.step_order == 1 else None
```

### 3. Sequential Step Activation

**Enhanced `_get_next_step` Method**:
```python
def _get_next_step(self, instance: PolicyApprovalInstance) -> Optional[PolicyApprovalStep]:
    """Get the next not_started step to activate"""
    return self.session.exec(
        select(PolicyApprovalStep).where(
            and_(
                PolicyApprovalStep.instance_id == instance.id,
                PolicyApprovalStep.status == "not_started",  # Look for inactive steps
                PolicyApprovalStep.step_order > instance.current_step
            )
        ).order_by(PolicyApprovalStep.step_order)
    ).first()
```

**Step Activation Logic**:
```python
# Move to next step
next_step = self._get_next_step(instance)
if next_step:
    # Activate the next step
    next_step.status = "pending"
    next_step.started_at = current_time
    instance.current_step = next_step.step_order
    instance.current_approver_id = self._get_next_approver(next_step)
```

### 4. Approver Visibility Control

**Updated `get_my_pending_approvals` Query**:
```python
# Only show flows for steps that are currently active (status = "pending")
query = select(
    PolicyApprovalFlow,
    PolicyApprovalStep,
    Policy
).join(
    PolicyApprovalStep, PolicyApprovalFlow.step_id == PolicyApprovalStep.id
).join(
    PolicyApprovalInstance, PolicyApprovalStep.instance_id == PolicyApprovalInstance.id
).join(
    Policy, PolicyApprovalInstance.policy_id == Policy.id
).where(
    and_(
        *conditions,
        PolicyApprovalStep.status == "pending"  # Only show active steps
    )
)
```

## Workflow Behavior

### Initial State
- **Step 1**: `pending` (visible to approvers)
- **Step 2**: `not_started` (hidden from approvers)
- **Step 3**: `not_started` (hidden from approvers)
- **Step 4**: `not_started` (hidden from approvers)

### After Step 1 Completion
- **Step 1**: `approved/rejected`
- **Step 2**: `pending` (now visible to approvers)
- **Step 3**: `not_started` (still hidden)
- **Step 4**: `not_started` (still hidden)

### After Step 2 Completion
- **Step 1**: `approved/rejected`
- **Step 2**: `approved/rejected`
- **Step 3**: `pending` (now visible to approvers)
- **Step 4**: `not_started` (still hidden)

### And So On...

## Benefits

### 1. **Proper Sequential Flow**
- Steps are activated in the correct order
- Approvers only see their tasks when it's their turn

### 2. **Reduced Confusion**
- No more premature approval requests
- Clear workflow progression

### 3. **Better User Experience**
- Approvers see only relevant tasks
- Workflow feels natural and logical

### 4. **Audit Trail**
- Clear step activation timestamps
- Proper workflow progression tracking

## Files Modified

1. **`backend/app/models.py`**
   - Updated `PolicyApprovalStep.status` field description
   - Added `not_started` status

2. **`backend/app/services/policy_approval_service.py`**
   - Modified step creation logic
   - Enhanced `_get_next_step` method
   - Added step activation logic

3. **`backend/app/api/v1/policies.py`**
   - Updated `get_my_pending_approvals` query
   - Added step status filtering

## Testing Scenarios

### Test Case 1: Initial State
- **Setup**: Create new policy with 4-step workflow
- **Expected**: Only Step 1 approvers see approval requests
- **Result**: Steps 2, 3, 4 approvers see no requests

### Test Case 2: Sequential Activation
- **Setup**: Step 1 gets approved
- **Expected**: Step 2 becomes visible to its approvers
- **Result**: Step 2 approvers now see approval requests

### Test Case 3: Mixed Required/Optional Steps
- **Setup**: Step 1 (required) → Step 2 (optional) → Step 3 (required)
- **Expected**: Sequential activation regardless of required status
- **Result**: All steps follow sequential activation

### Test Case 4: Step Rejection
- **Setup**: Step 1 gets rejected
- **Expected**: Workflow stops, no further steps activated
- **Result**: Policy marked as rejected, no more steps activated

## Validation

To validate the fix:

1. **Create a new policy** with multi-step workflow
2. **Check initial state**: Only first step approvers see requests
3. **Complete first step**: Verify second step becomes visible
4. **Continue workflow**: Verify sequential activation
5. **Check final state**: Verify proper completion logic

This implementation ensures that approval workflows follow proper sequential processing, eliminating the confusion where later steps were visible before earlier steps were completed.
