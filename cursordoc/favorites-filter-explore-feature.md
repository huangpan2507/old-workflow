# Favorites Filter in Explore Page Feature

## Overview

This document describes the implementation of the favorites filtering feature in the Policy Library Explore page. Users can now filter policies to show only their favorited policies directly from the Explore page, without needing to navigate to a separate "My Favorites" page.

## Features Implemented

### 1. Backend API Enhancement

#### Updated `/policies/` Endpoint
- Added `favorites_only` query parameter to the main policies listing endpoint
- When `favorites_only=true`, the API returns only policies that the current user has favorited
- Maintains all existing filtering capabilities (status, department, search) while adding favorites filtering

#### Implementation Details
- Added `favorites_only: Optional[bool] = Query(None, description="Filter to show only favorited policies")` parameter
- Added JOIN with `PolicyFavorite` table when `favorites_only=True`
- Filter is applied before other filters to ensure proper query structure

### 2. Frontend Implementation

#### Updated API Function
- Enhanced `getPolicies()` function in `utils/api.ts` to support `favorites_only` parameter
- Parameter is properly encoded in URL search params

#### Updated usePolicies Hook
- Modified `usePolicies` hook to accept `showFavoritesOnly` parameter
- Hook automatically reloads policies when `showFavoritesOnly` state changes
- Maintains backward compatibility with existing functionality

#### Updated PolicyUserInterface Component
- Added `showFavoritesOnly` state variable
- Added "Favorites Only" toggle button in the Explore page filter section
- Button shows star icon and changes appearance when active
- Button text changes between "Favorites Only" and "Show All" based on state
- Integrated with existing function role filters

### 4. UI Cleanup

#### Removed Redundant Quick Link
- Removed "My Favorites" button from the Quick Links panel
- Updated QuickLinksPanel component to remove favorites-related props and handlers
- Cleaned up unused imports and state variables
- Simplified navigation by consolidating favorites functionality into Explore page

## Technical Implementation

### Backend Changes
```python
# In app/api/v1/policies.py
@router.get("/", response_model=List[PolicyRead])
async def list_policies(
    *,
    session: SessionDep,
    current_user: CurrentUser,
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    status: Optional[str] = Query(None, description="Filter by approval status"),
    department_id: Optional[int] = Query(None, description="Filter by responsible department"),
    search: Optional[str] = Query(None, description="Search in policy name and summary"),
    favorites_only: Optional[bool] = Query(None, description="Filter to show only favorited policies")
):
    # Apply favorites filter first if needed
    if favorites_only:
        query = query.join(PolicyFavorite).where(
            PolicyFavorite.user_id == current_user.id
        )
```

### Frontend Changes
```typescript
// In utils/api.ts
export async function getPolicies(params?: {
  skip?: number
  limit?: number
  status?: string
  department_id?: number
  search?: string
  favorites_only?: boolean
}): Promise<Policy[]> {
  // ... existing code ...
  if (params?.favorites_only !== undefined) 
    searchParams.append("favorites_only", params.favorites_only.toString())
  // ... rest of function
}
```

## Usage

1. Navigate to the Policy Library Explore page
2. In the "Filter by Function Roles" section, click the "Favorites Only" button
3. The page will reload and show only policies that the user has favorited
4. Click "Show All" to return to showing all policies
5. The favorites filter can be combined with other filters (department, search)

## Benefits

1. **Improved User Experience**: Users can quickly filter to their favorite policies without navigation
2. **Consistent Interface**: Favorites filtering is integrated into the main Explore page
3. **Flexible Filtering**: Can be combined with existing filters for more precise results
4. **Simplified Navigation**: Removed redundant "My Favorites" quick link since functionality is now integrated
5. **Performance**: Server-side filtering reduces data transfer and improves performance

## Future Enhancements

1. **Filter Persistence**: Remember filter state across page refreshes
2. **Multiple Filter Types**: Add more filter options (date range, policy type, etc.)
3. **Filter Combinations**: Visual indicators for active filter combinations
4. **Quick Filter Presets**: Pre-defined filter combinations for common use cases
