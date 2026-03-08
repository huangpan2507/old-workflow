# My Approval Page Optimization - Single Request with Multiple Statuses

## Problem
Previously, the My Approval page made three separate API requests for each status (pending, approved, rejected), which was inefficient.

## Solution
Implemented a single API request that can filter by multiple statuses at once, supporting user's selection of one or more status filters.

## Changes Made

### 1. Backend API Changes (`backend/app/api/v1/policies.py`)

**API Endpoint Renamed:**
- Old: `GET /policies/my-approvals/pending`
- New: `GET /policies/my-approvals`

**Parameter Changes:**
```python
# Old
status: Optional[str] = Query(None, ...)

# New  
status: Optional[List[str]] = Query(None, description="Filter by flow status (pending, approved, rejected). Can specify multiple statuses.")
```

**Query Logic:**
```python
# Support multiple statuses using IN clause
if status and len(status) > 0:
    conditions.append(PolicyApprovalFlow.status.in_(status))
else:
    # Default to pending if no status specified
    conditions.append(PolicyApprovalFlow.status == "pending")
```

### 2. Frontend API Changes (`frontend/src/utils/api.ts`)

**Function Renamed and Enhanced:**
- Old: `getMyPendingApprovals()`
- New: `getMyApprovals()`

**Parameter Type:**
```typescript
// Old
status?: string

// New
status?: string[]
```

**URL Construction:**
```typescript
if (params?.status && params.status.length > 0) {
    params.status.forEach(status => searchParams.append("status", status))
}
```

This creates URL parameters like: `?status=pending&status=approved&status=rejected`

### 3. Frontend Data Loading (`frontend/src/components/PolicyUserInterface/shared/hooks/useOnDemandData.ts`)

**Single Request Instead of Three:**
```typescript
// Old: Made three separate requests
const pendingResponse = await getMyPendingApprovals({ status: "pending" })
const approvedResponse = await getMyPendingApprovals({ status: "approved" })
const rejectedResponse = await getMyPendingApprovals({ status: "rejected" })

// New: Single request with all statuses
const response = await getMyApprovals({ 
    limit: 100,
    status: ["pending", "approved", "rejected"]
})
```

## Benefits

1. **Reduced Network Requests**: One request instead of three
2. **Better Performance**: Less overhead, faster loading
3. **Cleaner API**: URL path doesn't contain specific status (`pending`)
4. **More Flexible**: Easy to add new statuses or modify filtering logic
5. **Supports User Selection**: Users can select multiple status filters in the UI

## Usage Example

### Backend API Call
```bash
GET /api/v1/policies/my-approvals?status=pending&status=approved&status=rejected
```

### Frontend Code
```typescript
const response = await getMyApprovals({ 
    limit: 100,
    status: ["pending", "approved", "rejected"]
})
```

## Testing
1. Open My Approval page
2. Check Network tab - should see only one request with multiple status parameters
3. Click different status filters - UI filters work correctly
4. Verify all three statuses (pending, approved, rejected) are loaded correctly

