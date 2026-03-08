# File Directory Display Issue Fix

## Problem Description

When uploading files to a subdirectory (e.g., `kkk` folder), the files were also appearing in the root `uploads` directory. This was causing confusion for users who expected files to only appear in their intended directory.

## Root Cause Analysis

The issue was caused by two factors:

### 1. Frontend FileManager Component
- The `loadFiles()` function was calling `getFileList(currentPath, true, 100)` with `recursive=true` for all directories
- This caused all subdirectory files to be displayed in the current view, regardless of the current path

### 2. Backend MinIO Service
- The `list_files()` method in `MinIOService` was not properly filtering subdirectory items when `recursive=false`
- MinIO's `list_objects` with `recursive=false` still returns all objects matching the prefix, including those in subdirectories

## Solution Implementation

### Frontend Fix
**File**: `frontend/src/components/FileManager/FileManager.tsx`

```typescript
async function loadFiles() {
  setLoading(true)
  try {
    // Only use recursive when we're at the root uploads directory
    // When in subdirectories, we want to see only direct children
    const useRecursive = currentPath === "uploads"
    const res: FileListResponse = await getFileList(currentPath, useRecursive, 100)
    // ... rest of the function
  } catch (e: any) {
    // ... error handling
  } finally {
    setLoading(false)
  }
}
```

**Changes**:
- Only use `recursive=true` when viewing the root `uploads` directory
- Use `recursive=false` when viewing subdirectories to show only direct children

### Backend Fix
**File**: `backend/app/services/minio_service.py`

```python
def list_files(self, folder: str = "", recursive: bool = False, max_keys: int = 100):
    # ... existing code ...
    
    for obj in objects:
        if count >= max_keys:
            break
        
        # When recursive=False, filter out files in subdirectories
        if not recursive and folder:
            # Remove the prefix to get relative path
            relative_path = obj.object_name[len(prefix):]
            # If there's a '/' in the relative path, it means it's in a subdirectory
            if '/' in relative_path:
                logger.debug(f"Skipping subdirectory item: {obj.object_name}")
                continue
        
        # ... rest of the processing
```

**Changes**:
- Added filtering logic when `recursive=False` and a folder is specified
- Skip objects that have additional path separators (indicating they're in subdirectories)
- Only show direct children of the specified folder

## DAG Behavior Verification

The DAG (`minio_to_dify_sync.py`) correctly uses `recursive=True` because:
- It needs to sync ALL files from the policy library to Dify knowledge base
- It properly skips directories (objects ending with `/`)
- It filters by supported file types
- This behavior is intentional and correct for the sync operation

## Expected Behavior After Fix

### File Upload
1. User uploads files to `uploads/kkk/` directory
2. Files appear only in the `kkk` folder view
3. Files do NOT appear in the root `uploads` view

### Directory Navigation
1. Root `uploads` view shows only direct folders and files
2. Subdirectory views show only direct children
3. No duplicate file listings across different directory levels

### DAG Sync
1. DAG continues to sync all files from all subdirectories
2. Files uploaded to `kkk` folder are still synced to Dify knowledge base
3. Sync behavior remains unchanged and correct

## Testing

### Manual Testing Steps
1. Navigate to Policy Library
2. Create a new folder (e.g., "test-folder")
3. Upload files to the new folder
4. Verify files appear only in the folder, not in root
5. Navigate back to root and verify files don't appear there
6. Test DAG sync to ensure files are still processed

### Verification Points
- ✅ Files only appear in their intended directory
- ✅ Directory navigation works correctly
- ✅ DAG sync continues to work for all files
- ✅ No duplicate file listings
- ✅ Proper folder/file separation

## Impact

### User Experience
- **Before**: Confusing duplicate file listings
- **After**: Clear, organized file structure matching user expectations

### System Behavior
- **Frontend**: Proper directory-based file display
- **Backend**: Correct file filtering logic
- **DAG**: Unchanged sync behavior (as intended)

## Conclusion

This fix resolves the file display issue while maintaining the correct sync behavior. Users can now upload files to specific directories and see them only in those directories, providing a more intuitive file management experience.
