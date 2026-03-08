# SMTP OAuth Test Guide

This guide explains how to test SMTP OAuth authentication with Microsoft 365/Outlook.com.

## Overview

Microsoft is deprecating Basic Authentication for SMTP. This test script helps verify that OAuth authentication works correctly before migrating from Basic Auth to OAuth.

## Prerequisites

1. **Azure AD Application Registration**
   - Register an application in Azure AD (Microsoft Entra ID)
   - Note down:
     - Application (Client) ID
     - Client Secret
     - Tenant ID

2. **Application Permissions** (for Client Credentials Flow)
   - Add `SMTP.SendAsApp` permission
   - Get tenant admin consent
   - Register service principal in Exchange (see Microsoft docs)

3. **API Permissions** (for Authorization Code Flow)
   - Add `SMTP.Send` delegated permission
   - User consent or admin consent required

4. **Redirect URI Configuration**
   - For Authorization Code Flow: Add `http://localhost:8080` as redirect URI in Azure AD app registration

5. **Python Dependencies**
   ```bash
   pip install requests
   ```

## OAuth Flows

### 1. Authorization Code Flow (User-based)

**Use when:** Testing with a specific user account, interactive testing

**Requirements:**
- User must authenticate in browser
- Delegated permission: `https://outlook.office.com/SMTP.Send`
- Redirect URI configured in Azure AD

**Usage:**
```bash
cd /home/song/workspace/xproject/foundation/backend
python test_smtp_oauth.py \
  --flow auth_code \
  --client-id <YOUR_CLIENT_ID> \
  --client-secret <YOUR_CLIENT_SECRET> \
  --tenant-id <YOUR_TENANT_ID> \
  --user-email <SENDER_EMAIL> \
  --to-email <RECIPIENT_EMAIL>
```

**Process:**
1. Script opens browser for user authentication
2. User logs in and grants permissions
3. Script receives authorization code via local HTTP server
4. Script exchanges code for access token
5. Script sends test email using OAuth

### 2. Client Credentials Flow (App-based)

**Use when:** Service-to-service authentication, no user interaction

**Requirements:**
- Application permission: `SMTP.SendAsApp`
- Tenant admin consent granted
- Service principal registered in Exchange
- Mailbox permissions configured

**Usage:**
```bash
cd /home/song/workspace/xproject/foundation/backend
python test_smtp_oauth.py \
  --flow client_credentials \
  --client-id <YOUR_CLIENT_ID> \
  --client-secret <YOUR_CLIENT_SECRET> \
  --tenant-id <YOUR_TENANT_ID> \
  --user-email <MAILBOX_EMAIL> \
  --to-email <RECIPIENT_EMAIL>
```

**Note:** For Client Credentials Flow, the `--user-email` should be the mailbox email that the service principal has access to.

## Environment Variables

You can also use environment variables instead of command-line arguments:

```bash
export SMTP_OAUTH_CLIENT_ID="<YOUR_CLIENT_ID>"
export SMTP_OAUTH_CLIENT_SECRET="<YOUR_CLIENT_SECRET>"
export SMTP_OAUTH_TENANT_ID="<YOUR_TENANT_ID>"
export SMTP_OAUTH_USER_EMAIL="<SENDER_EMAIL>"
export SMTP_OAUTH_TO_EMAIL="<RECIPIENT_EMAIL>"
export SMTP_OAUTH_FLOW="auth_code"  # or "client_credentials"

python test_smtp_oauth.py
```

## Azure AD Configuration Steps

### For Authorization Code Flow:

1. Go to Azure Portal → Azure Active Directory → App registrations
2. Create new registration or select existing app
3. Add Redirect URI: `http://localhost:8080` (Platform: Web)
4. Go to API permissions → Add permission → Microsoft Graph
5. Add delegated permission: `SMTP.Send` (under Mail permissions)
6. Grant admin consent if required

### For Client Credentials Flow:

1. Go to Azure Portal → Azure Active Directory → App registrations
2. Create new registration or select existing app
3. Go to API permissions → Add permission → Office 365 Exchange Online
4. Add application permission: `SMTP.SendAsApp`
5. Grant admin consent (required)
6. Register service principal in Exchange Online PowerShell:
   ```powershell
   Install-Module -Name ExchangeOnlineManagement
   Import-module ExchangeOnlineManagement
   Connect-ExchangeOnline -Organization <tenantId>
   
   # Get service principal details
   $AADServicePrincipalDetails = Get-AzureADServicePrincipal -SearchString YourAppName
   
   # Register in Exchange
   New-ServicePrincipal -AppId $AADServicePrincipalDetails.AppId -ObjectId $AADServicePrincipalDetails.ObjectId -DisplayName "EXO Serviceprincipal for EntraAD App"
   
   # Grant mailbox access
   $EXOServicePrincipal = Get-ServicePrincipal -Identity "EXO Serviceprincipal for EntraAD App"
   Add-MailboxPermission -Identity "mailbox@domain.com" -User $EXOServicePrincipal.Identity -AccessRights FullAccess
   ```

## Testing

1. Run the test script with appropriate parameters
2. For Authorization Code Flow: Complete browser authentication
3. Check if test email is received
4. Review console output for any errors

## Troubleshooting

### Common Issues:

1. **"Authentication failed"**
   - Check client ID, secret, and tenant ID
   - Verify redirect URI matches Azure AD configuration
   - Ensure permissions are granted

2. **"SMTP authentication error"**
   - Verify access token is valid
   - Check user email matches token
   - For Client Credentials: Verify service principal is registered and has mailbox access

3. **"No authorization code received"**
   - Check if browser opened correctly
   - Verify redirect URI is accessible
   - Check firewall/network settings

4. **"Token exchange failed"**
   - Verify client secret is correct
   - Check token endpoint URL
   - Ensure scopes are correct

## Next Steps

After successful testing:

1. Integrate OAuth into `EmailService` class
2. Add OAuth configuration to database/system config
3. Implement token refresh mechanism
4. Update SMTP connection logic in `email_service.py`
5. Test with production mailboxes

## References

- [Microsoft OAuth Documentation](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth)
- [OAuth 2.0 Protocol Overview](https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow)



