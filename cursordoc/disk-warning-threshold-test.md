# Disk Warning Threshold Test Guide

## Overview
This document describes how to test the disk warning threshold functionality when disk usage exceeds 80%.

## Current Status
- Current disk usage: ~37% (as of test time)
- Warning threshold: 80% (default)
- Recovery target: 50%

## Testing Methods

### Method 1: Temporarily Lower the Threshold (Recommended for Testing)

To test the warning behavior without actually filling the disk, you can temporarily lower the warning threshold:

1. **Modify environment configuration file**

   Edit `env.dev` (or your active environment file) and change:
   ```bash
   DISK_SPACE_WARNING_THRESHOLD=30.0
   ```

   This will make the current 37% usage trigger the warning.

2. **Restart the backend service**
   ```bash
   docker compose restart backend
   ```

3. **Test the UI behavior**
   - Navigate to `/data-ingestor/disk-scan`
   - Click on "Cloud Disk Management" (or use Quick Links)
   - You should see:
     - Red border around the disk status card
     - "Critical" badge
     - Red progress bar
     - Warning message: "Warning: Disk space is above 30%"
     - "Clean Up" button (for admin users)

4. **Test file upload blocking**
   - Try to upload a file
   - Should receive error: "Disk space insufficient" (HTTP 507)
   - Admin users should see a "Clean Up" button in the error toast

5. **Restore original threshold**
   - Edit `env.dev` and change back to: `DISK_SPACE_WARNING_THRESHOLD=80.0`
   - Restart backend: `docker compose restart backend`

### Method 2: Check Current Implementation

The warning threshold behavior is implemented as follows:

**Backend (`backend/app/api/v1/file_ingestion.py`):**
- Line 119-124: Checks `disk_space_service.is_disk_space_critical()` before file upload
- Returns HTTP 507 (Insufficient Storage) if threshold exceeded
- Line 794-829: `/disk-status` endpoint returns `is_critical` flag

**Frontend (`frontend/src/routes/_layout/data-ingestor.disk-scan.tsx`):**
- Line 1009-1011: Red border when `diskStatus.is_critical` is true
- Line 1024-1026: Shows "Critical" badge
- Line 1030: Red/yellow progress bar based on critical status
- Line 1038-1045: Warning message display
- Line 1047-1056: "Clean Up" button for admins

**Service (`backend/app/services/disk_space_service.py`):**
- Line 101-112: `is_disk_space_critical()` checks if usage >= threshold
- Default threshold: 80% (from config)

## Expected Behavior When Threshold Exceeded

### UI Changes:
1. **Disk Status Card:**
   - Red border (2px) instead of gray (1px)
   - Red progress bar instead of green/yellow
   - "Critical" badge displayed
   - Warning text: "Warning: Disk space is above {threshold}%"
   - Admin users see "Clean Up" button

2. **File Upload:**
   - Upload is blocked with HTTP 507 error
   - Error message: "Disk space insufficient. Please clean up old retained files before uploading."
   - Admin users see "Clean Up" button in error toast
   - Non-admin users see message to contact administrator

3. **Cleanup Modal (Admin Only):**
   - Shows current disk usage
   - Shows space to free
   - Shows recovery target (50%)
   - Allows cleanup of old retained files

## Testing Checklist

- [ ] Disk status card shows red border when threshold exceeded
- [ ] "Critical" badge is displayed
- [ ] Progress bar turns red
- [ ] Warning message is shown
- [ ] "Clean Up" button appears for admin users
- [ ] File upload is blocked with appropriate error
- [ ] Error toast shows cleanup option for admins
- [ ] Cleanup modal displays correct information
- [ ] Cleanup functionality works correctly
- [ ] After cleanup, disk status returns to normal

## Notes

- The threshold check happens in real-time when:
  - Loading disk status (every 60 seconds)
  - Attempting to upload a file
  - Opening the disk management page

- The warning threshold is configurable via environment variable in `env.dev`, `env.staging`, `env.prod`:
  ```bash
  DISK_SPACE_WARNING_THRESHOLD=80.0  # Default: 80%
  ```

- Recovery target (after cleanup) is also configurable:
  ```bash
  DISK_SPACE_RECOVERY_TARGET=50.0  # Default: 50%
  ```

- All disk space management configurations are now in environment files:
  - `MINIO_DATA_VOLUME_PATH=/data/minio` - Path to monitor disk usage
  - `DISK_SPACE_WARNING_THRESHOLD=80.0` - Warning threshold percentage
  - `DISK_SPACE_RECOVERY_TARGET=50.0` - Target usage after cleanup
  - `FILE_RETENTION_DEFAULT_DAYS=30` - Default retention period for files
  - `FILE_RETENTION_BUCKET_NAME=file-retention` - MinIO bucket for retained files

