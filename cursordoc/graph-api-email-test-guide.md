# Microsoft Graph API Email Test Guide

This guide explains how to test email sending using Microsoft Graph API with OAuth2 Client Credentials Flow.

## Overview

This test script uses Microsoft Graph API to send emails, which is an alternative to SMTP OAuth. Graph API is often simpler to use and doesn't require SMTP server configuration.

## Prerequisites

1. **Azure AD Application Registration**
   - Register an application in Azure AD (Microsoft Entra ID)
   - Note down:
     - Application (Client) ID
     - Client Secret
     - Tenant ID

2. **Application Permissions**
   - Add `Mail.Send` application permission (under Microsoft Graph)
   - Get tenant admin consent (required)
   - The service principal must have access to the mailbox you want to send from

3. **Python Dependencies**
   ```bash
   pip install requests
   ```

## Usage

### Using Command-Line Arguments

```bash
cd /home/song/workspace/xproject/foundation/backend
python test_graph_api_email.py \
  --client-id <YOUR_CLIENT_ID> \
  --client-secret <YOUR_CLIENT_SECRET> \
  --tenant-id <YOUR_TENANT_ID> \
  --from-email <SENDER_EMAIL> \
  --to-email <RECIPIENT_EMAIL>
```

### Using Environment Variables

```bash
export SMTP_OAUTH_CLIENT_ID="<YOUR_CLIENT_ID>"
export SMTP_OAUTH_CLIENT_SECRET="<YOUR_CLIENT_SECRET>"
export SMTP_OAUTH_TENANT_ID="<YOUR_TENANT_ID>"
export SMTP_OAUTH_USER_EMAIL="<SENDER_EMAIL>"
export SMTP_OAUTH_TO_EMAIL="<RECIPIENT_EMAIL>"

python test_graph_api_email.py
```

### Running in Docker Container

```bash
cd /home/song/workspace/xproject/foundation
docker compose exec backend python /app/test_graph_api_email.py \
  --client-id <YOUR_CLIENT_ID> \
  --client-secret <YOUR_CLIENT_SECRET> \
  --tenant-id <YOUR_TENANT_ID> \
  --from-email <SENDER_EMAIL> \
  --to-email <RECIPIENT_EMAIL>
```

### Additional Options

```bash
# Custom subject and body
python test_graph_api_email.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --from-email <FROM_EMAIL> \
  --to-email <TO_EMAIL> \
  --subject "Custom Subject" \
  --body "Custom email body" \
  --body-type HTML  # or Text
```

## Azure AD Configuration

### Step 1: Register Application

1. Go to Azure Portal → Azure Active Directory → App registrations
2. Create new registration or select existing app
3. Note down:
   - Application (Client) ID
   - Directory (Tenant) ID

### Step 2: Create Client Secret

1. Go to "Certificates & secrets"
2. Click "New client secret"
3. Note down the secret value (it's only shown once)

### Step 3: Add Application Permission

1. Go to "API permissions"
2. Click "Add a permission"
3. Select "Microsoft Graph"
4. Select "Application permissions"
5. Search for and add `Mail.Send`
6. Click "Add permissions"

### Step 4: Grant Admin Consent

1. Click "Grant admin consent for [Your Organization]"
2. Confirm the consent

### Step 5: Grant Mailbox Access (if needed)

If you need to send emails on behalf of a specific mailbox:

1. The mailbox must exist in your tenant
2. The service principal needs appropriate permissions
3. You may need to configure mailbox permissions via Exchange Online PowerShell

## Comparison: Graph API vs SMTP OAuth

| Feature | Graph API | SMTP OAuth |
|---------|----------|------------|
| **Complexity** | Simpler | More complex |
| **Setup** | API permissions only | SMTP + Exchange service principal |
| **Protocol** | REST API | SMTP protocol |
| **Ports** | HTTPS (443) | SMTP (587/465) |
| **Firewall** | Usually open | May need SMTP ports |
| **Error Handling** | JSON responses | SMTP error codes |
| **Features** | Full Graph API features | Standard SMTP features |

## Troubleshooting

### Common Issues

1. **"ErrorInvalidUser" (404)**
   - Verify the `from_email` address is correct
   - Ensure the mailbox exists in your tenant
   - Check if the service principal has access to the mailbox

2. **"Insufficient privileges" (403)**
   - Verify `Mail.Send` application permission is added
   - Ensure admin consent is granted
   - Check if the service principal has mailbox access

3. **"InvalidAuthenticationToken" (401)**
   - Check if the access token is valid and not expired
   - Verify the token has correct scopes
   - Ensure client ID and secret are correct

4. **"Token acquisition failed"**
   - Verify client ID, secret, and tenant ID are correct
   - Check if the application is enabled
   - Ensure the secret hasn't expired

## API Response Codes

- **200 OK**: Email sent successfully
- **202 Accepted**: Email accepted for delivery
- **400 Bad Request**: Invalid request format
- **401 Unauthorized**: Invalid or expired token
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: User/mailbox not found
- **500 Internal Server Error**: Server error

## Next Steps

After successful testing:

1. Integrate Graph API into `EmailService` class
2. Add Graph API configuration to database/system config
3. Implement token caching and refresh mechanism
4. Add error handling and retry logic
5. Test with production mailboxes

## References

- [Microsoft Graph API Send Mail](https://learn.microsoft.com/en-us/graph/api/user-sendmail)
- [Microsoft Graph API Authentication](https://learn.microsoft.com/en-us/graph/auth-v2-service)
- [Mail.Send Permission](https://learn.microsoft.com/en-us/graph/permissions-reference#mail-permissions)



