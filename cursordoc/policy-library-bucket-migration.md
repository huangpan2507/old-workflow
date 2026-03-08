# Policy Library Bucket Migration

## Overview

This document describes the migration of Policy Library file storage from the `uploads` directory within the `app-files` bucket to a dedicated `policy-library-files` bucket. This change provides better separation of concerns and allows for independent management of policy library files.

## Changes Made

### Backend Changes

#### 1. Configuration Updates (`backend/app/core/config.py`)
- Added `POLICY_LIBRARY_BUCKET_NAME` configuration variable
- Default value: `policy-library-files`
- Environment variable: `POLICY_LIBRARY_BUCKET_NAME`

#### 2. New Policy Library MinIO Service (`backend/app/services/policy_library_minio_service.py`)
- Created dedicated `PolicyLibraryMinIOService` class
- Uses `policy-library-files` bucket instead of `app-files`
- Provides all necessary file operations:
  - Upload files
  - Download files
  - List files/folders
  - Delete files/folders
  - Create folders
  - Get presigned URLs
  - Get file information

#### 3. New Policy Library API (`backend/app/api/v1/policy_library.py`)
- Created dedicated API endpoints for policy library operations
- All endpoints prefixed with `/policy-library`
- Maintains same interface as general file API but uses policy library service
- Endpoints:
  - `POST /policy-library/upload` - Upload files
  - `GET /policy-library/list` - List files/folders
  - `GET /policy-library/download/{object_name}` - Download files
  - `GET /policy-library/info/{object_name}` - Get file info
  - `DELETE /policy-library/{object_name}` - Delete files
  - `POST /policy-library/folder` - Create folders
  - `DELETE /policy-library/folder/{folder_path}` - Delete folders
  - `GET /policy-library/presigned-url/{object_name}` - Get presigned URLs

#### 4. API Router Registration (`backend/app/api/v1/__init__.py`)
- Registered policy library router with prefix `/policy-library`
- Tagged as `policy-library` for API documentation

### Frontend Changes

#### 1. API Functions (`frontend/src/utils/api.ts`)
- Added policy library specific API functions:
  - `uploadFileToPolicyLibrary()`
  - `getPolicyLibraryFileList()`
  - `deletePolicyLibraryFile()`
  - `getPolicyLibraryFileDownloadUrl()`
  - `createPolicyLibraryFolder()`
  - `deletePolicyLibraryFolder()`
  - `renamePolicyLibraryFile()`

#### 2. FileManager Component Updates (`frontend/src/components/FileManager/FileManager.tsx`)
- Updated all file operations to use policy library APIs
- Changed initial path from `uploads` to empty string (root of policy library bucket)
- Updated breadcrumb navigation to show "Policy Library" as root
- Updated all user-facing messages to indicate "Policy Library" operations
- Removed unused imports from general file API

## Architecture Changes

### Before
```
app-files bucket
├── uploads/           # Policy library files
│   ├── file1.docx
│   ├── file2.pdf
│   └── kkk/
│       ├── file3.docx
│       └── file4.pdf
└── other-app-files/   # Other application files
```

### After
```
app-files bucket                    policy-library-files bucket
├── other-app-files/               ├── file1.docx
│   └── ...                       ├── file2.pdf
└── ...                           └── kkk/
                                      ├── file3.docx
                                      └── file4.pdf
```

## Benefits

1. **Separation of Concerns**: Policy library files are now isolated from other application files
2. **Independent Management**: Policy library can be managed independently without affecting other applications
3. **Better Security**: Can apply different access policies to policy library bucket
4. **Scalability**: Policy library can scale independently
5. **Backup/Restore**: Can backup/restore policy library separately
6. **Monitoring**: Can monitor policy library usage independently

## Migration Notes

### For Users
- Policy Library interface remains the same
- All existing functionality preserved
- Files are now stored in dedicated bucket
- No user action required

### For Administrators
- New bucket `policy-library-files` will be created automatically
- Existing files in `uploads` directory need to be migrated manually if needed
- Environment variable `POLICY_LIBRARY_BUCKET_NAME` can be customized
- Monitor new bucket usage and performance

### For Developers
- Policy Library now uses dedicated API endpoints
- All file operations go through policy library service
- General file API (`/files/*`) remains unchanged for other applications
- Airflow DAG already updated to use new bucket (as mentioned by user)

## Configuration

### Environment Variables
```bash
# Policy Library Bucket Configuration
POLICY_LIBRARY_BUCKET_NAME=policy-library-files

# General MinIO Configuration (unchanged)
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123
MINIO_ENDPOINT=localhost:9000
MINIO_SECURE=false
MINIO_BUCKET_NAME=app-files  # Still used by other applications
```

### API Endpoints
- Policy Library: `/api/v1/policy-library/*`
- General Files: `/api/v1/files/*` (unchanged)

## Testing

### Manual Testing Steps
1. Access Policy Library page
2. Upload files to root directory
3. Create folders and upload files to folders
4. Navigate between folders
5. Download files
6. Delete files and folders
7. Verify files are stored in `policy-library-files` bucket
8. Verify other applications still use `app-files` bucket

### Verification Points
- ✅ Policy Library uses `policy-library-files` bucket
- ✅ Other applications still use `app-files` bucket
- ✅ All file operations work correctly
- ✅ Folder navigation works properly
- ✅ File upload/download/delete operations work
- ✅ Knowledge base sync still works (DAG updated separately)

## Rollback Plan

If rollback is needed:
1. Revert frontend changes to use general file API
2. Revert backend API changes
3. Update Airflow DAG to use `app-files/uploads` again
4. Migrate files back to `app-files/uploads` directory

## Future Enhancements

1. **File Migration Tool**: Create tool to migrate existing files from `uploads` to new bucket
2. **Bucket Policies**: Implement specific access policies for policy library bucket
3. **Monitoring**: Add monitoring for policy library bucket usage
4. **Backup Strategy**: Implement backup strategy for policy library bucket
5. **Performance Optimization**: Optimize policy library operations for large file counts
