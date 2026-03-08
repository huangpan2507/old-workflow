# SMTP Configuration Migration Guide

## Overview

SMTP configuration has been migrated from environment variables to database-backed configuration with admin UI.

## Database Migration

Run the Alembic migration to create the `system_configs` table:

```bash
# In backend container
docker exec foundation-backend-container alembic upgrade head
```

Or manually create migration:

```bash
docker exec foundation-backend-container alembic revision --autogenerate -m "add_system_configs_table"
docker exec foundation-backend-container alembic upgrade head
```

## Default Data Initialization

SMTP configuration defaults from `env.dev` are automatically initialized when the database is initialized:

- SMTP_HOST: smtp.163.com
- SMTP_PORT: 465
- SMTP_TLS: False
- SMTP_SSL: True
- SMTP_USER: fqs-1983@163.com
- SMTP_PASSWORD: GSiKH38ATYha5xCk
- EMAILS_FROM_EMAIL: fqs-1983@163.com
- EMAILS_FROM_NAME: (empty)

## Configuration Access

### Admin UI

1. Log in as superuser
2. Navigate to Admin → SMTP Configuration
3. Configure SMTP settings
4. Test email sending

### API Endpoints

- `GET /api/v1/system-config/smtp` - Get SMTP configuration (superuser only)
- `PUT /api/v1/system-config/smtp` - Update SMTP configuration (superuser only)
- `POST /api/v1/system-config/test-email` - Test email sending (superuser only)

## Fallback Behavior

If database is unavailable, the system falls back to environment variables from `settings`.

## Security

- Only superusers can access SMTP configuration
- Password fields are masked in API responses
- Sensitive values show only last 4 characters

