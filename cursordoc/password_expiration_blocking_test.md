# Password Expiration Blocking - Testing Guide

## Implementation Summary

### Backend Changes
- **File**: `backend/app/api/v1/login.py`
- **Change**: Added password expiration check after successful authentication
- **Behavior**: If password has expired, login is blocked with HTTP 403 error
- **Error Message**: "Your password has expired. Please reset your password via email recovery to continue."

### Frontend Changes
- **File**: `frontend/src/routes/login.tsx`
- **Change**: Added error handling for password expiration
- **Behavior**: 
  - Detects "expired" keyword in error message
  - Displays red warning box with password expiration message
  - Provides link to password recovery page

## Testing Steps

### 1. Create a Test User with Expired Password

```sql
-- Connect to database
docker compose exec backend python3 << 'EOF'
import sys
sys.path.insert(0, '/app')
from app.core.database import engine
from app.models import User
from app.core.security import get_password_hash
from sqlmodel import Session
from datetime import datetime, timedelta, timezone

with Session(engine) as session:
    # Find or create test user
    user = session.query(User).filter(User.email == "test-expired@example.com").first()
    if not user:
        user = User(
            email="test-expired@example.com",
            hashed_password=get_password_hash("Test1234!"),
            is_active=True,
            full_name="Test Expired User"
        )
        session.add(user)
        session.commit()
    
    # Set password_changed_at to 91 days ago (expired, since default is 90 days)
    expired_date = datetime.now(timezone.utc) - timedelta(days=91)
    user.password_changed_at = expired_date
    session.add(user)
    session.commit()
    
    print(f"User {user.email} password set to expired (changed {expired_date})")
EOF
```

### 2. Test Login with Expired Password

**Expected Behavior:**
- Login attempt should fail
- Error message: "Your password has expired. Please reset your password via email recovery to continue."
- Red warning box appears on login page
- Link to password recovery page is provided

**Test Command:**
```bash
curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=test-expired@example.com&password=Test1234!" \
  -v
```

**Expected Response:**
```json
{
  "detail": "Your password has expired. Please reset your password via email recovery to continue."
}
```
Status Code: 403

### 3. Test Password Recovery Flow

1. Navigate to login page
2. Attempt to login with expired password
3. Click "Click here to reset your password" link
4. Enter email address
5. Check email for password reset link
6. Reset password
7. Attempt login again - should succeed

### 4. Test Near-Expiry Warning (Not Blocked)

```sql
-- Set password to expire in 5 days (should still allow login)
docker compose exec backend python3 << 'EOF'
import sys
sys.path.insert(0, '/app')
from app.core.database import engine
from app.models import User
from sqlmodel import Session
from datetime import datetime, timedelta, timezone

with Session(engine) as session:
    user = session.query(User).filter(User.email == "test-expired@example.com").first()
    if user:
        # Set to 85 days ago (5 days remaining)
        near_expiry_date = datetime.now(timezone.utc) - timedelta(days=85)
        user.password_changed_at = near_expiry_date
        session.add(user)
        session.commit()
        print(f"User password set to near expiry (5 days remaining)")
EOF
```

**Expected Behavior:**
- Login should succeed
- `password_expires_in_days` in response should be 5
- Frontend can show warning (if implemented) but login is allowed

### 5. Test Non-Expired Password

```sql
-- Set password to recent (not expired)
docker compose exec backend python3 << 'EOF'
import sys
sys.path.insert(0, '/app')
from app.core.database import engine
from app.models import User
from sqlmodel import Session
from datetime import datetime, timezone

with Session(engine) as session:
    user = session.query(User).filter(User.email == "test-expired@example.com").first()
    if user:
        # Set to current time (not expired)
        user.password_changed_at = datetime.now(timezone.utc)
        session.add(user)
        session.commit()
        print(f"User password set to current time (not expired)")
EOF
```

**Expected Behavior:**
- Login should succeed normally
- No expiration warnings

## Verification Checklist

- [ ] Expired password blocks login (HTTP 403)
- [ ] Error message is clear and actionable
- [ ] Frontend displays red warning box
- [ ] Password recovery link is functional
- [ ] Near-expiry passwords still allow login
- [ ] Non-expired passwords work normally
- [ ] Password reset successfully updates `password_changed_at`
- [ ] After password reset, login succeeds

## Edge Cases

1. **User without `password_changed_at`**: Should not be blocked (backward compatibility)
2. **Password exactly at expiry date**: Should be blocked
3. **Password 1 day past expiry**: Should be blocked
4. **Password reset after expiry**: Should allow login after reset

