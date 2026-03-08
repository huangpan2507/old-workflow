# Policy Library API Examples

## Authentication

All API requests require authentication. Include the access token in the Authorization header:

```bash
Authorization: Bearer <your_access_token>
```

## Policy Management APIs

### 1. Create a New Policy

**Endpoint:** `POST /api/v1/policies/`

**Request (multipart/form-data):**
```bash
curl -X POST "http://localhost:8001/api/v1/policies/" \
  -H "Authorization: Bearer <token>" \
  -F "policy_name=IT Security Policy 2024" \
  -F "policy_summary=Updated security guidelines for all employees" \
  -F "revision_type=revised" \
  -F "responsible_department_id=5" \
  -F "applicable_business_units=[1,2,3]" \
  -F "applicable_departments=[5,6]" \
  -F "effective_date=2024-01-01" \
  -F "valid_until=2025-12-31" \
  -F "person_in_charge_name=John Doe" \
  -F "file=@/path/to/policy.pdf"
```

**Response:**
```json
{
  "id": 123,
  "policy_name": "IT Security Policy 2024",
  "policy_summary": "Updated security guidelines for all employees",
  "revision_type": "revised",
  "responsible_department_id": 5,
  "applicable_business_units": "[1,2,3]",
  "applicable_departments": "[5,6]",
  "effective_date": "2024-01-01",
  "valid_until": "2025-12-31",
  "person_in_charge_name": "John Doe",
  "file_object_name": "policies/policy_123.pdf",
  "approval_status": "pending",
  "current_approver_id": "uuid-of-approver",
  "created_by": "uuid-of-creator",
  "create_time": "2024-10-14T12:00:00Z",
  "update_time": "2024-10-14T12:00:00Z"
}
```

### 2. List Policies

**Endpoint:** `GET /api/v1/policies/`

**Query Parameters:**
- `skip`: Offset for pagination (default: 0)
- `limit`: Number of items (default: 100)
- `status`: Filter by approval status (draft, pending, approved, rejected)
- `department_id`: Filter by responsible department
- `search`: Search in policy name and summary

**Request:**
```bash
curl -X GET "http://localhost:8001/api/v1/policies/?status=approved&search=security&limit=10" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
[
  {
    "id": 123,
    "policy_name": "IT Security Policy 2024",
    "policy_summary": "Updated security guidelines",
    "approval_status": "approved",
    ...
  },
  ...
]
```

### 3. Get Policy Details

**Endpoint:** `GET /api/v1/policies/{policy_id}`

**Request:**
```bash
curl -X GET "http://localhost:8001/api/v1/policies/123" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "id": 123,
  "policy_name": "IT Security Policy 2024",
  "policy_summary": "Updated security guidelines for all employees",
  "revision_type": "revised",
  "responsible_department_id": 5,
  "applicable_business_units": "[1,2,3]",
  "applicable_departments": "[5,6]",
  "effective_date": "2024-01-01",
  "valid_until": "2025-12-31",
  "person_in_charge_id": "uuid-of-pic",
  "person_in_charge_name": "John Doe",
  "file_object_name": "policies/policy_123.pdf",
  "file_size": 1024000,
  "file_type": "application/pdf",
  "original_filename": "security_policy.pdf",
  "approval_status": "approved",
  "current_approver_id": null,
  "created_by": "uuid-of-creator",
  "create_time": "2024-10-14T12:00:00Z",
  "update_time": "2024-10-14T13:30:00Z"
}
```

### 4. Update Policy

**Endpoint:** `PUT /api/v1/policies/{policy_id}`

**Request:**
```bash
curl -X PUT "http://localhost:8001/api/v1/policies/123" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "policy_summary": "Updated security guidelines for all employees - Version 2",
    "valid_until": "2026-12-31"
  }'
```

**Response:**
```json
{
  "id": 123,
  "policy_summary": "Updated security guidelines for all employees - Version 2",
  "valid_until": "2026-12-31",
  ...
}
```

### 5. Delete Policy

**Endpoint:** `DELETE /api/v1/policies/{policy_id}`

**Request:**
```bash
curl -X DELETE "http://localhost:8001/api/v1/policies/123" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "success": true,
  "message": "Policy deleted successfully"
}
```

## Approval Workflow APIs

### 6. Submit Policy for Approval

**Endpoint:** `POST /api/v1/policies/{policy_id}/submit`

**Request:**
```bash
curl -X POST "http://localhost:8001/api/v1/policies/123/submit" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "success": true,
  "message": "Policy submitted for approval"
}
```

### 7. Approve or Reject Policy

**Endpoint:** `POST /api/v1/policies/{policy_id}/approve`

**Request (Approve):**
```bash
curl -X POST "http://localhost:8001/api/v1/policies/123/approve" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "decision": "approve",
    "comments": "Policy looks good, approved for implementation"
  }'
```

**Request (Reject):**
```bash
curl -X POST "http://localhost:8001/api/v1/policies/123/approve" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "decision": "reject",
    "comments": "Please revise section 3.2 for clarity"
  }'
```

**Response:**
```json
{
  "success": true,
  "message": "Policy approved successfully"
}
```

### 8. Get Approval History

**Endpoint:** `GET /api/v1/policies/{policy_id}/approval-history`

**Request:**
```bash
curl -X GET "http://localhost:8001/api/v1/policies/123/approval-history" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "policy_id": 123,
  "approval_status": "approved",
  "steps": [
    {
      "step_order": 1,
      "approver_id": "uuid-of-approver",
      "status": "approved",
      "decision": "approve",
      "comments": "Policy looks good, approved for implementation",
      "assigned_at": "2024-10-14T12:00:00Z",
      "reviewed_at": "2024-10-14T13:30:00Z"
    }
  ]
}
```

### 9. Get My Pending Approvals

**Endpoint:** `GET /api/v1/policies/my-approvals/pending`

**Request:**
```bash
curl -X GET "http://localhost:8001/api/v1/policies/my-approvals/pending?limit=50" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "total": 3,
  "policies": [
    {
      "id": 124,
      "policy_name": "HR Leave Policy 2024",
      "approval_status": "pending",
      ...
    },
    {
      "id": 125,
      "policy_name": "Finance Expense Policy",
      "approval_status": "pending",
      ...
    }
  ]
}
```

## File Storage APIs

### 10. Upload Policy File

**Endpoint:** `POST /api/v1/policy-library/upload`

**Request:**
```bash
curl -X POST "http://localhost:8001/api/v1/policy-library/upload" \
  -H "Authorization: Bearer <token>" \
  -F "file=@/path/to/policy.pdf" \
  -F "folder=policies" \
  -F "custom_filename=it_security_policy_2024.pdf"
```

**Response:**
```json
{
  "success": true,
  "message": "Policy library file uploaded successfully",
  "data": {
    "filename": "it_security_policy_2024.pdf",
    "original_filename": "policy.pdf",
    "object_name": "policies/it_security_policy_2024.pdf",
    "bucket": "policy-library",
    "size": 1024000,
    "content_type": "application/pdf",
    "folder": "policies",
    "upload_time": "2024-10-14T12:00:00Z",
    "presigned_url": "https://minio.../presigned-url"
  }
}
```

### 11. Get File Download URL

**Endpoint:** `GET /api/v1/policy-library/presigned-url/{object_name}`

**Request:**
```bash
curl -X GET "http://localhost:8001/api/v1/policy-library/presigned-url/policies%2Fit_security_policy_2024.pdf?expires_hours=24" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "success": true,
  "presigned_url": "https://minio-server.com/policy-library/policies/it_security_policy_2024.pdf?X-Amz-Algorithm=...",
  "expires_in_hours": 24
}
```

### 12. List Policy Files

**Endpoint:** `GET /api/v1/policy-library/list`

**Request:**
```bash
curl -X GET "http://localhost:8001/api/v1/policy-library/list?folder=policies&recursive=false&max_keys=100" \
  -H "Authorization: Bearer <token>"
```

**Response:**
```json
{
  "success": true,
  "message": "Found 5 policy library files",
  "data": [
    {
      "object_name": "policies/it_security_policy_2024.pdf",
      "size": 1024000,
      "last_modified": "2024-10-14T12:00:00Z",
      "etag": "abc123...",
      "is_dir": false
    },
    ...
  ],
  "total": 5
}
```

## Error Responses

### 400 Bad Request
```json
{
  "detail": "Policy name is required"
}
```

### 403 Forbidden
```json
{
  "detail": "No permission to access this policy"
}
```

### 404 Not Found
```json
{
  "detail": "Policy not found"
}
```

### 500 Internal Server Error
```json
{
  "detail": "Failed to create policy: Database error"
}
```

## Common Use Cases

### Use Case 1: Create and Submit Policy for Approval

```bash
# Step 1: Upload policy file
UPLOAD_RESPONSE=$(curl -X POST "http://localhost:8001/api/v1/policy-library/upload" \
  -H "Authorization: Bearer <token>" \
  -F "file=@policy.pdf" \
  -F "folder=policies")

OBJECT_NAME=$(echo $UPLOAD_RESPONSE | jq -r '.data.object_name')

# Step 2: Create policy with uploaded file
POLICY_RESPONSE=$(curl -X POST "http://localhost:8001/api/v1/policies/" \
  -H "Authorization: Bearer <token>" \
  -F "policy_name=New Security Policy" \
  -F "revision_type=new" \
  -F "responsible_department_id=5" \
  -F "applicable_business_units=[1,2]" \
  -F "effective_date=2024-01-01" \
  -F "file_object_name=$OBJECT_NAME")

POLICY_ID=$(echo $POLICY_RESPONSE | jq -r '.id')

# Step 3: Submit for approval (automatic on creation, but can also be done manually)
curl -X POST "http://localhost:8001/api/v1/policies/$POLICY_ID/submit" \
  -H "Authorization: Bearer <token>"
```

### Use Case 2: Approve Pending Policy

```bash
# Step 1: Get pending approvals
curl -X GET "http://localhost:8001/api/v1/policies/my-approvals/pending" \
  -H "Authorization: Bearer <token>"

# Step 2: Review and approve specific policy
curl -X POST "http://localhost:8001/api/v1/policies/123/approve" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "decision": "approve",
    "comments": "Approved after review"
  }'
```

### Use Case 3: Search and Download Policy

```bash
# Step 1: Search for policy
POLICIES=$(curl -X GET "http://localhost:8001/api/v1/policies/?search=security&status=approved" \
  -H "Authorization: Bearer <token>")

# Step 2: Get download URL
POLICY_ID=$(echo $POLICIES | jq -r '.[0].id')
OBJECT_NAME=$(echo $POLICIES | jq -r '.[0].file_object_name')

URL_RESPONSE=$(curl -X GET "http://localhost:8001/api/v1/policy-library/presigned-url/$OBJECT_NAME" \
  -H "Authorization: Bearer <token>")

# Step 3: Download file
DOWNLOAD_URL=$(echo $URL_RESPONSE | jq -r '.presigned_url')
wget "$DOWNLOAD_URL" -O policy.pdf
```

## Integration with Frontend

The frontend components automatically use these APIs:

- **PolicyUploadForm**: Calls upload and create policy APIs
- **PolicyList**: Uses list policies API with filters
- **PolicyDetailModal**: Gets policy details and approval history
- **PolicyApprovals**: Fetches pending approvals and handles review

All API calls are abstracted in `frontend/src/utils/api.ts` for easy reuse.

