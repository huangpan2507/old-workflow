# SharePoint API Test Script Guide

## Overview

The `test_graph_api_sharepoint.py` script tests reading files from SharePoint using Microsoft Graph API with OAuth2 Client Credentials Flow. This script is designed for application-level accounts that don't require user interaction.

## Prerequisites

1. Azure AD Application registered with the following permissions:
   - `Sites.Read.All` (Application permission)
   - `Files.Read.All` (Application permission)
   - Admin consent granted for these permissions

2. Required credentials:
   - Azure AD Tenant ID
   - Azure AD Application (Client) ID
   - Azure AD Application Client Secret
   - SharePoint Site URL

## Usage

### Method 1: Using Command-Line Arguments

#### List Root Folder Contents
```bash
python test_graph_api_sharepoint.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --site-url <SITE_URL>
```

#### List Specific Folder
```bash
python test_graph_api_sharepoint.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --site-url <SITE_URL> \
  --list-folder "Documents/SubFolder"
```

#### Read Specific File
```bash
python test_graph_api_sharepoint.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --site-url <SITE_URL> \
  --file-path "Documents/Test.docx"
```

#### Read and Save File
```bash
python test_graph_api_sharepoint.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --site-url <SITE_URL> \
  --file-path "Documents/Test.docx" \
  --output-file "/tmp/downloaded_file.docx"
```

### Method 2: Using Environment Variables

```bash
export SMTP_OAUTH_CLIENT_ID="<CLIENT_ID>"
export SMTP_OAUTH_CLIENT_SECRET="<CLIENT_SECRET>"
export SMTP_OAUTH_TENANT_ID="<TENANT_ID>"
export SHAREPOINT_SITE_URL="<SITE_URL>"
export SHAREPOINT_FILE_PATH="Documents/Test.docx"  # Optional
export SHAREPOINT_FOLDER_PATH="Documents"  # Optional

python test_graph_api_sharepoint.py
```

## Running in Docker Container

### Example: Running in Backend Container

```bash
# List root folder
docker compose exec backend python /app/test_graph_api_sharepoint.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --site-url <SITE_URL>

# Read specific file
docker compose exec backend python /app/test_graph_api_sharepoint.py \
  --client-id <CLIENT_ID> \
  --client-secret <CLIENT_SECRET> \
  --tenant-id <TENANT_ID> \
  --site-url <SITE_URL> \
  --file-path "Documents/Test.docx" \
  --output-file "/tmp/test.docx"
```

### Using Environment Variables from Container

If the environment variables are already set in the container (e.g., via docker-compose.yml):

```bash
docker compose exec backend python /app/test_graph_api_sharepoint.py \
  --site-url <SITE_URL> \
  --file-path "Documents/Test.docx"
```

## Site URL Format

The SharePoint site URL should be in one of these formats:
- `https://contoso.sharepoint.com/sites/MySite`
- `https://contoso.sharepoint.com/sites/MySite/SubSite`
- `https://contoso.sharepoint.com` (root site)

## File Path Format

File paths are relative to the SharePoint drive root:
- `Documents/Test.docx` - File in Documents folder
- `Shared Documents/Report.pdf` - File in Shared Documents folder
- `Test.docx` - File in root folder

## Troubleshooting

### Common Error Codes

#### 401 Unauthorized
- Check if the access token is valid and not expired
- Verify the token has correct scopes (Sites.Read.All, Files.Read.All application permissions)

#### 403 Forbidden
- Verify the service principal has 'Sites.Read.All' and 'Files.Read.All' application permissions
- Ensure admin consent is granted for the application
- Check if the service principal has access to the SharePoint site

#### 404 Not Found
- Verify the site URL is correct
- Ensure the site exists and is accessible
- Check if the file path is correct
- Verify the path format (e.g., 'Documents/Test.docx')

### Required Permissions

The Azure AD application must have the following Microsoft Graph API permissions:
- **Sites.Read.All** (Application) - Required for accessing SharePoint sites
- **Files.Read.All** (Application) - Required for reading files

Both permissions require admin consent.

## Examples

### Example 1: List Root Folder
```bash
python test_graph_api_sharepoint.py \
  --client-id "12345678-1234-1234-1234-123456789012" \
  --client-secret "your-secret" \
  --tenant-id "87654321-4321-4321-4321-210987654321" \
  --site-url "https://contoso.sharepoint.com/sites/MySite"
```

### Example 2: Read a Document
```bash
python test_graph_api_sharepoint.py \
  --client-id "12345678-1234-1234-1234-123456789012" \
  --client-secret "your-secret" \
  --tenant-id "87654321-4321-4321-4321-210987654321" \
  --site-url "https://contoso.sharepoint.com/sites/MySite" \
  --file-path "Documents/Policy.pdf"
```

### Example 3: Download File to Local
```bash
python test_graph_api_sharepoint.py \
  --client-id "12345678-1234-1234-1234-123456789012" \
  --client-secret "your-secret" \
  --tenant-id "87654321-4321-4321-4321-210987654321" \
  --site-url "https://contoso.sharepoint.com/sites/MySite" \
  --file-path "Documents/Report.xlsx" \
  --output-file "./downloaded_report.xlsx"
```

## Related Documentation

- [Microsoft Graph API - Get DriveItem](https://learn.microsoft.com/en-us/graph/api/driveitem-get)
- [Microsoft Graph API - List DriveItem Children](https://learn.microsoft.com/en-us/graph/api/driveitem-list-children)
- [Microsoft Graph API - Get Site](https://learn.microsoft.com/en-us/graph/api/site-get)

