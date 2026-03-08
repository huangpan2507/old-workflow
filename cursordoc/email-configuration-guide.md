# Email Service Configuration Guide

## Overview

This guide explains the email service configuration parameters that need to be set in environment files for the email notification system.

## Configuration Parameters

### SMTP Server Configuration

#### SMTP_HOST
- **Type**: String
- **Required**: Yes (if email notifications are enabled)
- **Description**: SMTP server hostname or IP address
- **Example**: `smtp.gmail.com`, `smtp.office365.com`, `smtp.example.com`
- **Default**: None (must be set to enable email)

#### SMTP_PORT
- **Type**: Integer
- **Required**: No
- **Description**: SMTP server port number
- **Common Values**:
  - `587` - TLS/STARTTLS (recommended)
  - `465` - SSL
  - `25` - Standard SMTP (often blocked)
- **Default**: `587`

#### SMTP_TLS
- **Type**: Boolean
- **Required**: No
- **Description**: Enable TLS/STARTTLS encryption
- **Values**: `True` or `False`
- **Default**: `True`
- **Note**: Use `True` for port 587, `False` for port 465 (SSL)

#### SMTP_SSL
- **Type**: Boolean
- **Required**: No
- **Description**: Use SSL connection (for port 465)
- **Values**: `True` or `False`
- **Default**: `False`
- **Note**: Use `True` for port 465, `False` for port 587 (TLS)

#### SMTP_USER
- **Type**: String
- **Required**: Yes (if SMTP server requires authentication)
- **Description**: SMTP authentication username
- **Example**: `noreply@example.com`, `your-email@gmail.com`
- **Default**: None

#### SMTP_PASSWORD
- **Type**: String
- **Required**: Yes (if SMTP server requires authentication)
- **Description**: SMTP authentication password or app password
- **Security**: Keep this secure, never commit to version control
- **Default**: None

### Email Sender Configuration

#### EMAILS_FROM_EMAIL
- **Type**: Email String
- **Required**: Yes (if email notifications are enabled)
- **Description**: Email address used as sender
- **Example**: `noreply@example.com`, `notifications@example.com`
- **Default**: None
- **Note**: Must match SMTP authentication if required

#### EMAILS_FROM_NAME
- **Type**: String
- **Required**: No
- **Description**: Display name for email sender
- **Example**: `Policy Management System`, `EIM Foundation`
- **Default**: Uses `PROJECT_NAME` if not set

### Email Queue Configuration

#### EMAIL_QUEUE_BATCH_SIZE
- **Type**: Integer
- **Required**: No
- **Description**: Number of emails to process per batch
- **Range**: 1-100 (recommended: 5-20)
- **Default**: `10`
- **Impact**: 
  - Higher values = faster processing but more memory usage
  - Lower values = slower processing but less memory usage

#### EMAIL_QUEUE_PROCESS_INTERVAL
- **Type**: Float
- **Required**: No
- **Description**: Seconds between email queue processing cycles
- **Range**: 1.0-60.0 (recommended: 5.0-10.0)
- **Default**: `5.0`
- **Impact**:
  - Lower values = more frequent processing, faster delivery
  - Higher values = less frequent processing, less server load

#### EMAIL_MAX_RETRY_COUNT
- **Type**: Integer
- **Required**: No
- **Description**: Maximum number of retry attempts for failed emails
- **Range**: 1-10 (recommended: 5-6)
- **Default**: `6`
- **Retry Delays**: Exponential backoff (1min → 5min → 15min → 1h → 6h → 24h)

## Configuration Examples

### Gmail Configuration

```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
EMAILS_FROM_EMAIL=your-email@gmail.com
EMAILS_FROM_NAME=Policy Management System
```

**Note**: Gmail requires an [App Password](https://support.google.com/accounts/answer/185833) instead of your regular password.

### Office 365 / Outlook Configuration

```bash
SMTP_HOST=smtp.office365.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
SMTP_USER=your-email@outlook.com
SMTP_PASSWORD=your-password
EMAILS_FROM_EMAIL=your-email@outlook.com
EMAILS_FROM_NAME=Policy Management System
```

### Custom SMTP Server (SSL)

```bash
SMTP_HOST=smtp.example.com
SMTP_PORT=465
SMTP_TLS=False
SMTP_SSL=True
SMTP_USER=noreply@example.com
SMTP_PASSWORD=your-password
EMAILS_FROM_EMAIL=noreply@example.com
EMAILS_FROM_NAME=Policy Management System
```

### Custom SMTP Server (TLS)

```bash
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
SMTP_USER=noreply@example.com
SMTP_PASSWORD=your-password
EMAILS_FROM_EMAIL=noreply@example.com
EMAILS_FROM_NAME=Policy Management System
```

## Environment-Specific Recommendations

### Development (env.dev)

```bash
# Use local SMTP server or test service
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_TLS=False
SMTP_SSL=False
SMTP_USER=
SMTP_PASSWORD=
EMAILS_FROM_EMAIL=test@localhost
EMAILS_FROM_NAME=Dev Environment

# Faster processing for testing
EMAIL_QUEUE_BATCH_SIZE=5
EMAIL_QUEUE_PROCESS_INTERVAL=2.0
EMAIL_MAX_RETRY_COUNT=3
```

### Staging (env.staging)

```bash
# Use production-like SMTP but with test credentials
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
SMTP_USER=staging@example.com
SMTP_PASSWORD=staging-password
EMAILS_FROM_EMAIL=staging@example.com
EMAILS_FROM_NAME=Staging Environment

# Standard settings
EMAIL_QUEUE_BATCH_SIZE=10
EMAIL_QUEUE_PROCESS_INTERVAL=5.0
EMAIL_MAX_RETRY_COUNT=6
```

### Production (env.prod)

```bash
# Production SMTP server
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
SMTP_USER=noreply@example.com
SMTP_PASSWORD=secure-production-password
EMAILS_FROM_EMAIL=noreply@example.com
EMAILS_FROM_NAME=Policy Management System

# Optimized for production load
EMAIL_QUEUE_BATCH_SIZE=20
EMAIL_QUEUE_PROCESS_INTERVAL=5.0
EMAIL_MAX_RETRY_COUNT=6
```

## Testing Email Configuration

### Check if Email is Enabled

The system checks if email is enabled by verifying:
```python
emails_enabled = bool(SMTP_HOST and EMAILS_FROM_EMAIL)
```

If either is missing, email notifications will be disabled.

### Test Email Sending

1. **Check Redis Connection**: Email queue requires Redis
2. **Check SMTP Connection**: Verify SMTP credentials
3. **Create Test Approval**: Create a policy revision or deviation
4. **Check Logs**: Monitor application logs for email sending status
5. **Check Queue**: Use Redis CLI to check email queue status

## Troubleshooting

### Emails Not Sending

1. **Check SMTP Configuration**
   - Verify `SMTP_HOST` and `SMTP_PORT`
   - Check `SMTP_USER` and `SMTP_PASSWORD`
   - Ensure `SMTP_TLS`/`SMTP_SSL` matches port

2. **Check Redis Connection**
   - Email queue requires Redis
   - Verify `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`

3. **Check Logs**
   - Look for email service errors
   - Check queue processor logs
   - Verify background tasks are running

### Common SMTP Errors

#### Authentication Failed
- **Cause**: Wrong username/password
- **Solution**: Verify credentials, use app password for Gmail

#### Connection Timeout
- **Cause**: Wrong host/port or firewall blocking
- **Solution**: Verify SMTP server address and port

#### TLS/SSL Error
- **Cause**: Wrong TLS/SSL setting for port
- **Solution**: Port 587 → `SMTP_TLS=True`, Port 465 → `SMTP_SSL=True`

## Security Best Practices

1. **Never Commit Passwords**: Use environment variables, never hardcode
2. **Use App Passwords**: For Gmail/Office365, use app-specific passwords
3. **Restrict SMTP Access**: Use dedicated email account for notifications
4. **Monitor Email Queue**: Regularly check failed queue for issues
5. **Rotate Credentials**: Periodically update SMTP passwords

## Performance Tuning

### High Volume Scenarios

```bash
# Increase batch size and reduce interval
EMAIL_QUEUE_BATCH_SIZE=50
EMAIL_QUEUE_PROCESS_INTERVAL=2.0
EMAIL_MAX_RETRY_COUNT=6
```

### Low Volume Scenarios

```bash
# Smaller batches, less frequent processing
EMAIL_QUEUE_BATCH_SIZE=5
EMAIL_QUEUE_PROCESS_INTERVAL=10.0
EMAIL_MAX_RETRY_COUNT=6
```

### Balanced (Recommended)

```bash
# Default settings work well for most scenarios
EMAIL_QUEUE_BATCH_SIZE=10
EMAIL_QUEUE_PROCESS_INTERVAL=5.0
EMAIL_MAX_RETRY_COUNT=6
```

## Related Documentation

- [Email Notification Architecture](./email-notification-architecture.md)
- [Email Notification Flow Diagrams](./email-notification-flow-diagrams.md)
- [AsyncIOScheduler Introduction](./AsyncIOScheduler-introduction.md)

