# Source Field Migration: Policy to User Action

## Migration Overview

This migration updates the `source` field in the `file_ingestion_records` table to align with new source field values.

## Changes

### 1. Source Field Value Updates
- **Old value**: `policy` 
- **New value**: `user_action` (represents user's key actions in the system)

### 2. New Supported Values
The `source` field now supports the following values:
- `disk_scan` - Files scanned from disk
- `user_action` - User's key actions in the system (previously "policy")
- `sharepoint` - Files from SharePoint (new)
- `onedrive` - Files from OneDrive (new)

## Migration File

**Revision:** `x20250207_update_source`

**File:** `backend/app/alembic/versions/x20250207_update_source_field_policy_to_user_action.py`

### Migration Steps

1. **Data Migration**: Updates all existing records with `source='policy'` to `source='user_action'`
2. **No Schema Changes**: The field is already a string type, so no table structure changes are needed

### Rollback Support

The migration includes a `downgrade()` function that reverts `user_action` back to `policy` if needed.

## Code Updates

### Files Modified

1. **Migration File**: `backend/app/alembic/versions/x20250207_update_source_field_policy_to_user_action.py`
   - Created new migration to update existing data

2. **API File**: `backend/app/api/v1/policy_revisions.py`
   - Updated `source="policy"` to `source="user_action"` (line 1452)

3. **Service File**: `backend/app/services/dimension_analysis_service.py`
   - Updated documentation comments to reflect new source values

4. **Model File**: `backend/app/models.py`
   - Comment already updated to reflect new values: `disk_scan, user_action, sharepoint, onedrive`

## Execution

To apply this migration:

```bash
cd /home/song/workspace/xproject/foundation/backend
docker compose exec backend alembic upgrade head
```

## Current Status

- ✅ Migration file created
- ✅ Code updated to use `user_action` instead of `policy`
- ✅ Documentation comments updated
- ⏳ Migration ready to be applied to database

