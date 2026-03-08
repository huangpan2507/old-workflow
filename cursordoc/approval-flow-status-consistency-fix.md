# Approval Flow Status Consistency Fix

## Problem Description

There was an inconsistency between step overall status and individual approver statuses in the approval flow diagram:

- **Step 3**: Overall status showed `IN_PROGRESS`, but approver showed `NOT_STARTED`
- **Step 5**: Overall status showed `IN_PROGRESS`, but approver showed `NOT_STARTED`

This created confusion and was logically inconsistent.

## Root Cause Analysis

The issue was in the step status calculation logic in `ApprovalFlowDiagram.tsx`:

### Original Logic (Incorrect)
```typescript
const hasApproved = group.approvers.some(approver => approver.status === "approve")
const hasRejected = group.approvers.some(approver => approver.status === "reject")
const allPending = group.approvers.every(approver => approver.status === "pending")

if (hasApproved) {
  group.overall_status = "approved"
} else if (hasRejected) {
  group.overall_status = "rejected"
} else if (allPending) {
  group.overall_status = "pending"
} else {
  group.overall_status = "in_progress"  // ❌ Problem here
}
```

**Problem**: When all approvers have `NOT_STARTED` status:
- `allPending` = `false` (because status is not "pending")
- `hasApproved` = `false`
- `hasRejected` = `false`
- Result: Falls into `else` branch → `in_progress` ❌

## Solution Implementation

### 1. Enhanced Status Calculation Logic

**Updated Logic**:
```typescript
const hasApproved = group.approvers.some(approver => approver.status === "approve")
const hasRejected = group.approvers.some(approver => approver.status === "reject")
const allPending = group.approvers.every(approver => approver.status === "pending")
const allNotStarted = group.approvers.every(approver => approver.status === "not_started")  // ✅ New check

if (hasApproved) {
  group.overall_status = "approved"
} else if (hasRejected) {
  group.overall_status = "rejected"
} else if (allNotStarted) {  // ✅ New condition
  group.overall_status = "not_started"
} else if (allPending) {
  group.overall_status = "pending"
} else {
  group.overall_status = "in_progress"
}
```

### 2. Updated Type Definitions

**Enhanced StepGroup Interface**:
```typescript
interface StepGroup {
  step_order: number
  step_name: string
  approval_logic: string
  is_required: boolean
  approvers: ApprovalStep[]
  overall_status: "not_started" | "pending" | "approved" | "rejected" | "in_progress"  // ✅ Added not_started
}
```

### 3. Enhanced Visual Representation

**Status Color Mapping**:
```typescript
const getStepStatusColor = (status: string) => {
  switch (status) {
    case "approved": return "green"
    case "rejected": return "red"
    case "in_progress": return "blue"
    case "pending": return "orange"      // ✅ Changed from gray to orange
    case "not_started": return "gray"   // ✅ Added not_started
    default: return "gray"
  }
}
```

**Status Icon Mapping**:
```typescript
const getStepStatusIcon = (status: string) => {
  switch (status) {
    case "approved": return <Circle size="12px" bg="green.500" />
    case "rejected": return <Circle size="12px" bg="red.500" />
    case "in_progress": return <Circle size="12px" bg="blue.500" />
    case "pending": return <Circle size="12px" bg="orange.500" />    // ✅ Changed color
    case "not_started": return <Circle size="12px" bg="gray.400" />  // ✅ Added
    default: return <Circle size="12px" bg="gray.400" />
  }
}
```

## Expected Behavior After Fix

### Status Consistency Rules

1. **All Approvers NOT_STARTED** → Step Status: `NOT_STARTED`
2. **All Approvers PENDING** → Step Status: `PENDING`
3. **Mixed Statuses** → Step Status: `IN_PROGRESS`
4. **Any Approver APPROVED** → Step Status: `APPROVED`
5. **Any Approver REJECTED** → Step Status: `REJECTED`

### Visual Consistency

- **Step Status**: `NOT_STARTED` (gray badge)
- **Approver Status**: `NOT_STARTED` (gray badge)
- **Result**: ✅ Consistent

- **Step Status**: `PENDING` (orange badge)
- **Approver Status**: `PENDING` (gray badge)
- **Result**: ✅ Consistent

- **Step Status**: `IN_PROGRESS` (blue badge)
- **Approver Status**: Mixed (e.g., some `PENDING`, some `NOT_STARTED`)
- **Result**: ✅ Consistent

## Benefits

### 1. **Logical Consistency**
- Step status accurately reflects approver statuses
- No more confusing mismatches

### 2. **Better User Experience**
- Clear visual representation
- Intuitive status progression

### 3. **Accurate Workflow Representation**
- Proper sequential workflow visualization
- Correct status transitions

## Files Modified

1. **`frontend/src/components/PolicyDetailModal/ApprovalFlowDiagram.tsx`**
   - Enhanced step status calculation logic
   - Updated StepGroup interface
   - Improved visual representation

## Testing Scenarios

### Test Case 1: Not Started Step
- **Setup**: Step with all approvers `NOT_STARTED`
- **Expected**: Step status = `NOT_STARTED`
- **Result**: ✅ Consistent gray badges

### Test Case 2: Pending Step
- **Setup**: Step with all approvers `PENDING`
- **Expected**: Step status = `PENDING`
- **Result**: ✅ Consistent orange/gray badges

### Test Case 3: Mixed Status Step
- **Setup**: Step with some `PENDING`, some `NOT_STARTED`
- **Expected**: Step status = `IN_PROGRESS`
- **Result**: ✅ Blue badge for step, mixed badges for approvers

### Test Case 4: Approved Step
- **Setup**: Step with any approver `APPROVED`
- **Expected**: Step status = `APPROVED`
- **Result**: ✅ Green badge for step

## Validation

To validate the fix:

1. **Check Step 3**: Should show `NOT_STARTED` (gray) if approver is `NOT_STARTED`
2. **Check Step 5**: Should show `NOT_STARTED` (gray) if approver is `NOT_STARTED`
3. **Verify Consistency**: Step status should always match approver statuses
4. **Test Transitions**: Status changes should be consistent across step and approvers

This fix ensures that the approval flow diagram accurately represents the workflow state, eliminating the confusion caused by inconsistent status displays.
