# Policy Library Management System - Implementation Guide

## Overview

The Policy Library Management System is a comprehensive solution for managing company policies with features including:
- Policy upload and metadata management
- Automated approval workflows
- Permission-based access control (by Business Unit and Department)
- Policy search and retrieval
- Audit logging
- AI-powered policy Q&A (integration ready)

## Architecture

### Backend Components

#### 1. Data Models (`backend/app/models.py`)

**Policy Model**
- Stores policy metadata including name, summary, revision type
- Links to responsible department and person in charge
- Tracks applicable business units and departments (JSON arrays)
- Includes file storage information (MinIO object reference)
- Manages approval status and current approver

**PolicyApprovalFlow Model**
- Tracks multi-step approval workflows
- Records approver decisions and comments
- Maintains approval history with timestamps

**PolicyAccessLog Model**
- Audit trail for policy access
- Records user actions (view, download, search)
- Captures IP address and user agent

#### 2. API Endpoints (`backend/app/api/v1/policies.py`)

**CRUD Operations:**
- `POST /api/v1/policies/` - Create new policy
- `GET /api/v1/policies/` - List policies (with permission filtering)
- `GET /api/v1/policies/{id}` - Get policy details
- `PUT /api/v1/policies/{id}` - Update policy
- `DELETE /api/v1/policies/{id}` - Soft delete policy

**Approval Workflow:**
- `POST /api/v1/policies/{id}/submit` - Submit policy for approval
- `POST /api/v1/policies/{id}/approve` - Approve/reject policy
- `GET /api/v1/policies/{id}/approval-history` - View approval history
- `GET /api/v1/policies/my-approvals/pending` - Get pending approvals

#### 3. Permission Control

**Access Rules:**
- Users can only view policies applicable to their BU and Department
- Superusers have access to all policies
- Policy creators can always access their policies
- Current approvers can access policies pending their review
- Only approved policies are visible to general users

**Approval Flow:**
- System automatically determines approver based on responsible department
- Approver is the department head (function_head level user)
- Department head reviews and approves/rejects policies

### Frontend Components

#### 1. Policy Upload Form (`frontend/src/components/PolicyUploadForm/PolicyUploadForm.tsx`)

**Form Fields:**
- Revision Type (New/Revised)
- Policy Name* (required)
- Policy Summary
- Responsible Department* (required)
- Applicable Business Units* (required, multi-select)
- Applicable Departments (optional, multi-select)
- Effective Date* (required)
- Valid Until (optional)
- Person-In-Charge
- File Attachment (optional)

**Features:**
- Dynamic loading of departments and business units
- File upload integration with MinIO
- Form validation
- Auto-submit to approval workflow

#### 2. Policy List (`frontend/src/components/PolicyList/PolicyList.tsx`)

**Features:**
- Search by policy name or summary
- Filter by approval status
- View policy details
- Download policy documents
- Submit for approval (draft policies)
- Delete policies

**Display:**
- Table view with policy metadata
- Status badges (Draft, Pending, Approved, Rejected)
- Action buttons for each policy

#### 3. Policy Detail Modal (`frontend/src/components/PolicyDetailModal/PolicyDetailModal.tsx`)

**Tabs:**
- **Information Tab**: Shows all policy metadata
- **Approval History Tab**: Displays approval workflow steps

**Actions (for approvers):**
- Approve with comments
- Reject with comments
- Download policy document

#### 4. Policy Approvals (`frontend/src/components/PolicyApprovals/PolicyApprovals.tsx`)

**Features:**
- Lists policies pending user's approval
- Quick review and approve/reject
- View policy details before decision

#### 5. Policy Library Page (`frontend/src/routes/_layout/policy-library.tsx`)

**Tabs:**
- **Policies Tab**: Main policy list and management
- **File Manager Tab**: MinIO file browser for policy documents

### Database Migration

**Migration File:** `backend/app/alembic/versions/ab12cd34ef56_add_policy_library_tables.py`

**Tables Created:**
- `policies` - Main policy information
- `policy_approval_flows` - Approval workflow tracking
- `policy_access_logs` - Access audit logs

**Indexes:**
- Policy status, department, effective date, created_by
- Approval flow: policy_id, approver_id, status
- Access log: policy_id, user_id, access_time

## Workflow

### 1. Policy Creation and Submission

```
User uploads policy → System creates policy record → 
System determines approver based on department → 
Policy status = "pending" → Approval workflow created
```

### 2. Approval Process

```
Approver receives notification → Reviews policy → 
Makes decision (approve/reject) → 
System updates policy status → 
Decision recorded in approval history
```

### 3. Policy Access

```
User searches/browses policies → 
System filters by user's BU/Department → 
User views approved policies → 
Access logged for audit
```

## Permission Model

### Policy Visibility Rules

1. **Creators**: Can always see their own policies (any status)
2. **Approvers**: Can see policies pending their approval
3. **General Users**: Can only see approved policies applicable to their BU/Department
4. **Superusers**: Can see all policies

### Approval Authority

- Department heads (function_head level) approve policies for their department
- System automatically assigns the appropriate approver based on responsible department
- Only current approver can approve/reject policy

## API Integration Points

### Organization Data
- Departments: `GET /api/v1/organization/departments`
- Business Units: `GET /api/v1/organization/business-units`

### File Storage
- Upload: `POST /api/v1/policy-library/upload`
- Download: `GET /api/v1/policy-library/presigned-url/{object_name}`

### Policy Management
- All policy endpoints: `/api/v1/policies/`

## Future Enhancements

### AI Q&A Integration
- Connect to knowledge base sync: `POST /api/v1/policy-library/sync`
- Enable natural language queries on policy content
- Provide AI-powered policy recommendations

### Notifications
- Email notifications for pending approvals
- Alerts for policy expiration
- Updates on policy changes

### Advanced Search
- Full-text search across policy content
- Filter by multiple criteria
- Saved searches

### Versioning
- Track policy revisions
- Compare versions
- Rollback capabilities

### Analytics
- Policy access statistics
- Approval time metrics
- Compliance reports

## Usage Instructions

### For Policy Creators

1. Navigate to Policy Library → Policies tab
2. Click "New Policy" button
3. Fill in all required fields:
   - Select revision type
   - Enter policy name and summary
   - Choose responsible department
   - Select applicable BUs and departments
   - Set effective date
   - Attach policy document
4. Click "Submit" to create policy
5. System automatically submits to approval workflow

### For Approvers

1. Navigate to Policy Library → Your notifications will show pending approvals
2. Click "Review" on pending policy
3. Review policy details and download document
4. Enter comments (optional)
5. Click "Approve" or "Reject"

### For Policy Users

1. Navigate to Policy Library → Policies tab
2. Use search bar to find policies by keyword
3. Filter by status or department
4. Click "View Details" to see policy information
5. Download policy documents as needed

## Security Considerations

- All API endpoints require authentication
- Permission checks at both API and UI levels
- Soft delete for data retention
- Audit logging for compliance
- File storage secured in MinIO with presigned URLs

## Database Schema

### policies table
- id (PK)
- policy_name
- policy_summary
- revision_type
- responsible_department_id (FK → functions.id)
- applicable_business_units (JSON)
- applicable_departments (JSON)
- effective_date
- valid_until
- person_in_charge_id (FK → users.id)
- file_object_name
- approval_status
- current_approver_id (FK → users.id)
- created_by (FK → users.id)
- timestamps

### policy_approval_flows table
- id (PK)
- policy_id (FK → policies.id)
- step_order
- approver_id (FK → users.id)
- approver_department_id (FK → functions.id)
- status
- decision
- comments
- assigned_at
- reviewed_at
- timestamps

### policy_access_logs table
- id (PK)
- policy_id (FK → policies.id)
- user_id (FK → users.id)
- action
- ip_address
- user_agent
- access_time

## Deployment Notes

1. Run database migration:
   ```bash
   alembic upgrade head
   ```

2. Ensure MinIO bucket exists:
   - Bucket name configured in settings: `POLICY_LIBRARY_BUCKET_NAME`

3. Verify organization structure:
   - Departments (functions) must exist
   - Business units must exist
   - Department heads must be assigned

4. Configure environment variables:
   - MinIO credentials
   - Database connection
   - API base URL

## Testing Checklist

- [ ] Policy creation with all field combinations
- [ ] File upload and download
- [ ] Approval workflow (approve/reject)
- [ ] Permission filtering (BU/Department)
- [ ] Search and filter functionality
- [ ] Audit log creation
- [ ] Error handling and validation
- [ ] Multi-user approval scenarios
- [ ] Edge cases (no approver, expired policies, etc.)

## Support and Maintenance

For issues or questions, refer to:
- API Documentation: http://localhost:8001/docs
- Backend logs: Check application logs for errors
- Frontend console: Browser developer tools for UI issues

