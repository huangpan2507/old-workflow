# Account Approval System Migration Guide

## Overview
This document provides step-by-step instructions for migrating to the new unified account approval system.

## Prerequisites
- Backup your database before migration
- Ensure backend services are stopped
- Review the migration script `s20260110_refactor_account_approval.py`

## Migration Steps

### Step 1: Backup Database
```bash
# Connect to backend container
docker-compose exec -T backend bash

# Backup PostgreSQL database
pg_dump -h db -U postgres -d foundation > /tmp/foundation_backup_$(date +%Y%m%d_%H%M%S).sql
```

### Step 2: Apply Alembic Migration
```bash
# From project root directory
cd /home/song/workspace/xproject/foundation/backend

# Run migration
alembic upgrade head
```

### Step 3: Verify Migration
```bash
# Check if new table was created
docker-compose exec -T backend alembic current

# Verify data migration
docker-compose exec -T backend python3 -c "
from app.database import get_session
from app.models import AccountApprovalRequest, User
from sqlmodel import select

session = next(get_session())

# Check account_approval_requests table
approval_requests = session.exec(select(AccountApprovalRequest)).all()
print(f'Total approval requests: {len(approval_requests)}')

# Check users table
users = session.exec(select(User)).all()
active_users = [u for u in users if u.account_status == 'active']
print(f'Active users: {len(active_users)}')
"
```

### Step 4: Restart Services
```bash
# Restart backend to apply changes
docker-compose restart backend
```

### Step 5: Manual Testing Checklist
- [ ] User registration creates account with status 'email_verified'
- [ ] Email verification works correctly
- [ ] Profile submission creates activation request
- [ ] User status changes to 'profile_submitted'
- [ ] Admin can approve activation requests
- [ ] User status changes to 'active' after approval
- [ ] Active users can submit profile update requests
- [ ] Profile update requests show as 'profile_update' type
- [ ] Admin can approve/reject profile update requests
- [ ] Historical profile update requests migrated correctly
- [ ] Old API endpoints still work (with deprecation warnings)

### Step 6: Frontend Verification
- [ ] Frontend can call new account approval endpoints
- [ ] Approval status updates correctly in UI
- [ ] Email notifications sent for new requests
- [ ] Email notifications sent for approvals/rejections
- [ ] i18n keys display correctly (EN/ZH)

## Rollback Plan

If migration fails:
```bash
# Rollback database migration
cd /home/song/workspace/xproject/foundation/backend
alembic downgrade -1

# Restore from backup (if needed)
docker-compose exec -T db psql -U postgres -d foundation < /tmp/foundation_backup.sql

# Restart services
docker-compose restart
```

## Post-Migration Tasks

### Phase 7: Cleanup (After 1 week of stable operation)

Once the new system is confirmed stable for at least one week:

1. **Remove Legacy Models**
   - Remove `UserProfileUpdateRequest` model from `backend/app/models.py`
   - Remove `approval_status`, `profile_completed`, `password_set`, `profile_submitted_at`, `approved_by`, `approved_at` from User model

2. **Remove Legacy CRUD Functions**
   - Remove `get_user_profile_update_requests()` from `backend/app/crud.py`
   - Remove `get_pending_update_requests()` from `backend/app/crud.py`
   - Remove `approve_user_profile_update_request()` from `backend/app/crud.py`
   - Remove `reject_user_profile_update_request()` from `backend/app/crud.py`
   - Remove `revoke_user_profile_update_request()` from `backend/app/crud.py`

3. **Remove Legacy API Endpoints**
   - Remove `POST /users/{user_id}/approve` from `backend/app/api/v1/users.py`
   - Remove `POST /users/{user_id}/reject` from `backend/app/api/v1/users.py`
   - Remove `GET /users/profile-update-requests` from `backend/app/api/v1/users.py`
   - Remove `POST /users/profile-update-requests/{request_id}/approve` from `backend/app/api/v1/users.py`
   - Remove `POST /users/profile-update-requests/{request_id}/reject` from `backend/app/api/v1/users.py`
   - Remove `POST /users/profile-update-requests/{request_id}/revoke` from `backend/app/api/v1/users.py`

4. **Create Final Cleanup Migration**
   - Drop `user_profile_update_requests` table
   - Drop legacy columns from `users` table:
     - `approval_status`
     - `profile_completed`
     - `password_set`
     - `profile_submitted_at`
     - `approved_by`
     - `approved_at`

5. **Frontend Cleanup**
   - Remove calls to old API endpoints
   - Remove unused i18n keys (if any)

## Monitoring

### Key Metrics to Track
- Number of pending approval requests
- Approval request processing time
- Account activation rate
- Profile update request frequency
- Error rates on approval endpoints

### Alert Thresholds
- Alert if pending requests > 50
- Alert if approval processing time > 24 hours
- Alert if error rate > 5%

## Support Contacts

If you encounter issues during migration:
- Check logs: `docker-compose logs backend`
- Review migration script: `backend/app/alembic/versions/s20260110_refactor_account_approval.py`
- Contact development team for assistance

