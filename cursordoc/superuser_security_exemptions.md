# Superuser Security Exemptions

## Overview

Superuser accounts (accounts with `is_superuser=True`) are exempt from certain security restrictions to prevent lockout of system administrators. This is especially important for default admin accounts (e.g., `admin@example.com`) that may not have real email addresses and cannot use email-based password recovery.

## Exemptions

### 1. Account Lockout Exemption

**Function**: `check_account_locked(user: User) -> bool`

**Behavior**:
- Superuser accounts are never considered locked, even if `locked_until` is set
- Returns `False` immediately for superuser accounts
- Prevents superuser lockout in login endpoint and `get_current_user` dependency

**Rationale**: 
- Prevents accidental lockout of system administrators
- Allows superusers to access the system even after multiple failed login attempts
- Critical for system recovery scenarios

### 2. Failed Login Attempt Tracking Exemption

**Function**: `increment_failed_login_attempts(session: Session, user: User) -> None`

**Behavior**:
- Superuser accounts do not have failed login attempts incremented
- Superuser accounts are never locked due to failed login attempts
- Function returns early without any database updates for superusers

**Rationale**:
- Prevents tracking of failed attempts for superusers
- Ensures superuser accounts remain accessible
- Reduces risk of denial-of-service attacks targeting admin accounts

### 3. Password Expiration Exemption

**Function**: `check_password_expired(user: User, expiry_days: int | None = None) -> bool`

**Behavior**:
- Superuser accounts are never considered to have expired passwords
- Returns `False` immediately for superuser accounts
- Prevents password expiration blocking for superusers in login endpoint

**Rationale**:
- Default admin accounts (e.g., `admin@example.com`) may not have real email addresses
- Prevents lockout of administrators who cannot use email-based password recovery
- Allows superusers to maintain system access for critical operations

## Implementation Details

### Modified Functions

1. **`backend/app/core/security.py`**:
   - `check_account_locked()`: Added superuser check at the beginning
   - `increment_failed_login_attempts()`: Returns early for superusers

2. **`backend/app/core/password_policy.py`**:
   - `check_password_expired()`: Added superuser check at the beginning

### Affected Endpoints

- `POST /api/v1/login/access-token`: Superusers bypass account lockout and password expiration checks
- `GET /api/v1/users/me` (via `get_current_user`): Superusers bypass account lockout check

## Security Considerations

### Benefits

1. **System Recovery**: Ensures administrators can always access the system
2. **Default Account Protection**: Protects default admin accounts without real email addresses
3. **Operational Continuity**: Prevents accidental lockout of critical accounts

### Trade-offs

1. **Reduced Security for Superusers**: Superuser accounts have weaker security restrictions
2. **No Failed Attempt Tracking**: Cannot detect brute-force attacks on superuser accounts
3. **No Password Expiration**: Superuser passwords can remain unchanged indefinitely

### Recommendations

1. **Use Strong Passwords**: Superuser accounts should use very strong, unique passwords
2. **Monitor Access**: Implement additional monitoring for superuser account access
3. **Regular Manual Review**: Periodically review and update superuser passwords manually
4. **Limit Superuser Count**: Keep the number of superuser accounts to a minimum
5. **Use Real Email for Production**: In production, configure superuser accounts with real email addresses if possible

## Testing

### Verify Superuser Exemptions

```python
# Test account lockout exemption
user = get_user_by_email("admin@example.com")
user.is_superuser = True
user.locked_until = datetime.now(timezone.utc) + timedelta(minutes=30)
assert check_account_locked(user) == False  # Should return False

# Test failed login attempt exemption
increment_failed_login_attempts(session, user)
# Should not increment failed_login_attempts for superuser

# Test password expiration exemption
user.password_changed_at = datetime.now(timezone.utc) - timedelta(days=100)
assert check_password_expired(user) == False  # Should return False
```

### Verify Regular User Restrictions

```python
# Regular users should still be subject to restrictions
user = get_user_by_email("regular@example.com")
user.is_superuser = False
user.locked_until = datetime.now(timezone.utc) + timedelta(minutes=30)
assert check_account_locked(user) == True  # Should return True
```

## Configuration

No additional configuration is required. The exemptions are automatically applied based on the `is_superuser` flag in the user model.

## Related Documentation

- [Password Expiration Blocking](./password_expiration_blocking_test.md)
- [Security Policy Testing Guide](./security_policy_testing_guide.md)

