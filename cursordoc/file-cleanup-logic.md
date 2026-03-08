# File Cleanup Logic Documentation

## Overview
This document explains how the file cleanup functionality works when disk space exceeds the warning threshold.

## Cleanup Flow

### 1. Calculate Space to Free (`disk_space_service.get_space_to_free()`)

**Location**: `backend/app/services/disk_space_service.py:127-147`

**Logic**:
```python
target_used = total * (recovery_target / 100)
space_to_free = max(0, used - target_used)
```

**Example**:
- Current usage: 34.6% (348.66 GB used)
- Recovery target: 20% (set in `DISK_SPACE_RECOVERY_TARGET`)
- Total space: 1006.85 GB
- Target used: 1006.85 GB × 20% = 201.37 GB
- Space to free: 348.66 GB - 201.37 GB = **147.29 GB**

### 2. Find Files to Delete (`file_retention_service.get_oldest_retention_files()`)

**Location**: `backend/app/services/file_retention_service.py:262-306`

**Query Conditions**:
```python
FileIngestionRecord.is_retained == True
FileIngestionRecord.file_storage_path.isnot(None)
```

**Sorting Order**:
1. `retention_expires_at` ASC (NULL values first)
2. `uploaded_at` ASC (oldest first)

**Selection Logic**:
- Iterate through retained files in order
- Accumulate file sizes until `total_space >= target_space_bytes`
- Return selected files list

### 3. Execute Cleanup (`file_retention_service.cleanup_old_files()`)

**Location**: `backend/app/services/file_retention_service.py:308-376`

**Steps**:
1. Get files to delete from `get_oldest_retention_files()`
2. For each file:
   - Delete from MinIO using `delete_retained_file()`
   - Update database record:
     - `is_retained = False`
     - `file_storage_path = None`
     - `retention_expires_at = None`
   - Accumulate `freed_space` from `file_size`
3. Commit database transaction
4. Return statistics

## Why Cleanup Might Return 0 Bytes

### Possible Reasons:

1. **No Retained Files**
   - No files with `is_retained == True` in database
   - Check: Query `SELECT COUNT(*) FROM file_ingestion_records WHERE is_retained = true`

2. **No Storage Path**
   - Files marked as retained but `file_storage_path` is NULL
   - This happens if file upload failed or file wasn't saved to MinIO
   - Check: Query `SELECT COUNT(*) FROM file_ingestion_records WHERE is_retained = true AND file_storage_path IS NOT NULL`

3. **All Files Already Cleaned**
   - Previous cleanup already removed all retained files
   - Check retention status via `/file-ingestion/retention-status` API

4. **File Size is 0 or NULL**
   - Files exist but `file_size` is 0 or NULL
   - These files won't contribute to `freed_space` calculation
   - Check: Query `SELECT COUNT(*) FROM file_ingestion_records WHERE is_retained = true AND file_size > 0`

## File Retention Conditions

A file is eligible for cleanup if:
- ✅ `is_retained == True`
- ✅ `file_storage_path IS NOT NULL`
- ✅ File exists in MinIO bucket (`file-retention` by default)

## Cleanup Priority

Files are deleted in this order:
1. **Expired files first**: Files with `retention_expires_at <= now()`
2. **Oldest files**: Files with `retention_expires_at IS NULL` (sorted by `uploaded_at`)
3. **Newer files**: Files not yet expired (sorted by `retention_expires_at`)

## API Endpoints

### Check Retention Status
```bash
GET /api/v1/file-ingestion/retention-status
```

Returns:
- `retained_count`: Total number of retained files
- `retained_total_size`: Total size of retained files
- `soon_expire_count`: Files expiring in 7 days
- `expired_count`: Files already expired

### Execute Cleanup
```bash
POST /api/v1/file-ingestion/cleanup-old-files
```

Returns:
- `deleted_count`: Number of files deleted
- `freed_space`: Bytes freed
- `freed_space_formatted`: Human-readable size
- `target_space`: Target space to free
- `errors`: List of errors (if any)

## Debugging

### Check Database
```sql
-- Count retained files
SELECT COUNT(*) FROM file_ingestion_records WHERE is_retained = true;

-- Count retained files with storage path
SELECT COUNT(*) FROM file_ingestion_records 
WHERE is_retained = true AND file_storage_path IS NOT NULL;

-- Count retained files with size > 0
SELECT COUNT(*), SUM(file_size) FROM file_ingestion_records 
WHERE is_retained = true AND file_storage_path IS NOT NULL AND file_size > 0;

-- List oldest retained files
SELECT id, file_name, file_size, retention_expires_at, uploaded_at, file_storage_path
FROM file_ingestion_records 
WHERE is_retained = true AND file_storage_path IS NOT NULL
ORDER BY retention_expires_at ASC NULLS FIRST, uploaded_at ASC
LIMIT 10;
```

### Check MinIO
- Verify files exist in MinIO bucket (`file-retention` by default)
- Check if `file_storage_path` matches actual object names in MinIO

## Common Issues

### Issue: Cleanup returns 0 bytes but files exist

**Possible causes**:
1. Files are marked as retained but not saved to MinIO
2. `file_storage_path` is NULL
3. Files were deleted from MinIO but database wasn't updated
4. `file_size` is NULL or 0

**Solution**: Check retention status API and database queries above.

### Issue: Cleanup doesn't free enough space

**Possible causes**:
1. Not enough retained files to reach target
2. Files are too small
3. Some files failed to delete (check `errors` in response)

**Solution**: Check `errors` array in cleanup response, verify MinIO connectivity.

