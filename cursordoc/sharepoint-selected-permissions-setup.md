# SharePoint Selected Permissions Setup Guide

## Overview

This guide explains how to configure **Sites.Selected** and **Files.SelectedOperations.Selected** permissions for the Azure AD application to access specific SharePoint folders.

## Current Status

The Azure AD application (`ca9147f1-7b3c-4639-95dd-76298c795469`) has the following permissions configured:

- ✅ `Sites.Selected` (Application)
- ✅ `Files.SelectedOperations.Selected` (Application)
- ✅ `ListItems.SelectedOperations.Selected` (Application)
- ✅ `Lists.SelectedOperations.Selected` (Application)

However, these are **permission capabilities**, not actual access grants. **A SharePoint Administrator must grant the application access to the specific site.**

## Required Action by SharePoint Administrator

### Method 1: Using Microsoft Graph API (Recommended)

The SharePoint administrator needs to grant the application read permission to the site using the Graph API.

#### Step 1: Get the Site ID

First, the admin needs to get the Site ID for `https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal`

Using Graph Explorer (https://developer.microsoft.com/graph/graph-explorer) or PowerShell:

```http
GET https://graph.microsoft.com/v1.0/sites/dcigroupadmin.sharepoint.com:/teams/EiMStaffPortal
```

Response will include:
```json
{
    "id": "dcigroupadmin.sharepoint.com,xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx,xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "displayName": "EiMStaffPortal",
    ...
}
```

#### Step 2: Grant Application Permission to the Site

Using the Site ID from Step 1, grant read permission:

```http
POST https://graph.microsoft.com/v1.0/sites/{site-id}/permissions
Content-Type: application/json

{
    "roles": ["read"],
    "grantedToIdentities": [{
        "application": {
            "id": "ca9147f1-7b3c-4639-95dd-76298c795469",
            "displayName": "EIM Foundation App"
        }
    }]
}
```

### Method 2: Using PnP PowerShell

```powershell
# Install PnP PowerShell if not already installed
Install-Module -Name PnP.PowerShell -Force

# Connect to SharePoint Admin Center
Connect-PnPOnline -Url "https://dcigroupadmin-admin.sharepoint.com" -Interactive

# Grant the application read permission to the specific site
Grant-PnPAzureADAppSitePermission -AppId "ca9147f1-7b3c-4639-95dd-76298c795469" `
    -DisplayName "EIM Foundation App" `
    -Site "https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal" `
    -Permissions Read
```

### Method 3: Using SharePoint REST API

The admin can also use SharePoint REST API to grant permissions:

```http
POST https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/site/permissions
Authorization: Bearer {admin_token}
Content-Type: application/json

{
    "roles": ["read"],
    "grantedToIdentities": [{
        "application": {
            "id": "ca9147f1-7b3c-4639-95dd-76298c795469"
        }
    }]
}
```

## Verification

After the administrator grants permissions, they can verify by listing site permissions:

```http
GET https://graph.microsoft.com/v1.0/sites/{site-id}/permissions
```

The response should include an entry for the application:

```json
{
    "value": [
        {
            "id": "xxxxx",
            "roles": ["read"],
            "grantedToIdentitiesV2": [
                {
                    "application": {
                        "id": "ca9147f1-7b3c-4639-95dd-76298c795469",
                        "displayName": "EIM Foundation App"
                    }
                }
            ]
        }
    ]
}
```

## Why Site-Level Permission is Required

### Question: Why do we need site read permission when we only need to access a specific folder?

**Answer:** This is a limitation of Microsoft Graph API's current architecture. Here's why:

1. **Microsoft Graph API Architecture Limitation**
   - Microsoft Graph API currently **does not support** granting application permissions at the folder or drive level
   - The **most granular level** of permission that can be granted to an application is at the **site collection level**
   - Even when using `Files.SelectedOperations.Selected` permission model, the application must first have access to the site before it can access files/folders within that site

2. **What "Selected" Actually Means**
   - `Sites.Selected` means the application can only access **selected sites** (sites explicitly granted permission)
   - `Files.SelectedOperations.Selected` means the application can perform operations on files/folders **within the selected sites**
   - The "Selected" refers to **selected sites**, not selected folders

3. **Permission Scope Comparison**

   | Permission Type | Scope | Access Level |
   |----------------|-------|--------------|
   | `Sites.Read.All` | All sites in tenant | ❌ Too broad - can access everything |
   | `Sites.Selected` + Site Permission | Only explicitly granted sites | ✅ Limited to specific sites |
   | Folder-level permission | Specific folders | ❌ Not supported by Graph API |

4. **Actual Security Boundary**
   - While the permission is granted at the site level, the application **can only access sites that are explicitly granted**
   - The application **cannot** access other sites in the tenant
   - This is still more secure than `Sites.Read.All` which would grant access to all sites

### How to Explain to Customer IT Security Team

**Key Points to Communicate:**

1. **We are NOT requesting `Sites.Read.All`** - This would grant access to all sites in the tenant, which is too broad.

2. **We are requesting `Sites.Selected`** - This only allows access to sites that are explicitly granted permission by the administrator.

3. **The permission is site-scoped, but access is limited** - While the permission model requires site-level access, the application can only access the specific site(s) that the administrator explicitly grants.

4. **This is the most granular permission currently available** - Microsoft Graph API does not support folder-level permissions for applications. Site-level is the finest granularity available.

5. **Access can be revoked at any time** - The administrator can remove the site permission at any time, immediately revoking access.

**Suggested Response to Customer:**

> "We understand your security concerns. The permission is granted at the site level due to Microsoft Graph API's architecture limitations - folder-level permissions are not currently supported. However, this is still a **restricted permission**:
> 
> - The application uses `Sites.Selected` (not `Sites.Read.All`), meaning it can only access sites explicitly granted by your administrator
> - The application cannot access other sites in your tenant
> - Access can be revoked immediately by removing the site permission
> - This is the most granular permission level currently available from Microsoft
> 
> While we only need access to a specific folder, Microsoft's API requires site-level access as a prerequisite. The actual access is still limited to the specific site you grant, not your entire SharePoint environment."

## What We Need from the Customer IT

Please ask the customer IT to:

1. **Grant site-level permission** using one of the methods above
2. **Provide the following IDs** (if possible):
   - Site ID: `dcigroupadmin.sharepoint.com,{site-guid},{web-guid}`
   - Drive ID: The ID of the `docshare` document library
   - Folder Item ID: The ID of the `Public Document` folder

These IDs will help us verify the configuration and access the resources correctly.

## Alternative Approaches (If Site Permission is Not Acceptable)

If the customer IT security team cannot approve site-level permission, here are alternative approaches:

### Option 1: Use SharePoint REST API with App-Only Authentication
- Requires configuring an App Principal in SharePoint
- Can potentially have more granular control at the list/library level
- More complex setup, requires SharePoint admin configuration

### Option 2: Use Service Account with Delegated Permissions
- Create a service account user
- Grant folder-level permissions to that user account
- Application uses delegated permissions (user context)
- Requires managing a service account password

### Option 3: Wait for Microsoft to Support Folder-Level Permissions
- Microsoft may introduce folder-level application permissions in future Graph API updates
- Monitor Microsoft Graph API roadmap for updates

**Note:** All these alternatives have trade-offs. The `Sites.Selected` + site-level permission approach is currently the recommended method by Microsoft for application-level access.

## References

- [Microsoft Graph: Sites.Selected permission](https://learn.microsoft.com/en-us/graph/permissions-reference#sitesselected)
- [Grant application access to a specific site](https://learn.microsoft.com/en-us/graph/api/site-post-permissions)
- [PnP PowerShell: Grant-PnPAzureADAppSitePermission](https://pnp.github.io/powershell/cmdlets/Grant-PnPAzureADAppSitePermission.html)
- [Microsoft Graph: Selected permissions model](https://learn.microsoft.com/en-us/graph/cloud-connectivity-concept-overview#selected-permissions)
- [Limitations of folder-level permissions](https://learn.microsoft.com/en-us/answers/questions/2115242/how-to-grant-graph-api-access-only-to-a-specific-s)


