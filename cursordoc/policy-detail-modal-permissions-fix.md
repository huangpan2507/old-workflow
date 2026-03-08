# Policy Detail Modal Permissions Fix

## Problem Description

The Policy Detail Modal component was showing "Reject" and "Approve" buttons in both "My Policies" and "Approval" workspaces, which was incorrect. According to the requirements:

- **My Policies workspace**: Should only allow users to view policy status (read-only)
- **Approval workspace**: Should provide approval functionality (Reject/Approve buttons)

## Solution

### 1. Added `readonly` Property to PolicyDetailModal

Modified the `PolicyDetailModal` component to accept a `readonly` boolean property:

```typescript
interface PolicyDetailModalProps {
  isOpen: boolean
  onClose: () => void
  policy: Policy
  onUpdate?: () => void
  readonly?: boolean // If true, hide approval/reject buttons
}
```

### 2. Updated Modal Footer Logic

Modified the modal footer to conditionally show approval buttons based on the `readonly` property:

```typescript
{!readonly && policy.approval_status === "pending" ? (
  <>
    <Button leftIcon={<CloseIcon />} colorScheme="red" onClick={() => handleReview("reject")}>
      Reject
    </Button>
    <Button leftIcon={<CheckIcon />} colorScheme="green" onClick={() => handleReview("approve")}>
      Approve
    </Button>
  </>
) : (
  <Button variant="ghost" onClick={onClose}>
    Close
  </Button>
)}
```

### 3. Updated My Policies Workspace

Modified the PolicyUserInterface component to pass `readonly={true}` when opening the policy detail modal:

```typescript
<PolicyDetailModal
  isOpen={isDetailOpen}
  onClose={onDetailClose}
  policy={selectedPolicy}
  onUpdate={loadPolicies}
  readonly={true} // My Policies workspace should only allow viewing
/>
```

### 4. Verified Approval Workspace

Confirmed that the PolicyApprovals component continues to use the default `readonly={false}` value, maintaining approval functionality.

## Files Modified

1. `/frontend/src/components/PolicyDetailModal/PolicyDetailModal.tsx`
   - Added `readonly` property to interface
   - Updated component function signature
   - Modified modal footer logic

2. `/frontend/src/components/PolicyUserInterface/PolicyUserInterface.tsx`
   - Added `readonly={true}` to PolicyDetailModal usage

## Result

- **My Policies workspace**: Policy detail modal now shows only "Close" button (read-only mode)
- **Approval workspace**: Policy detail modal continues to show "Reject" and "Approve" buttons for pending policies
- **Policy Request workspace**: Maintains approval functionality (uses default `readonly={false}`)

This fix ensures that approval functionality is properly restricted to the appropriate workspace while maintaining a clean, consistent user experience.
