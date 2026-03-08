# Email Templates Structure

## Overview

Email templates are now organized as separate files for better maintainability and organization.

## File Structure

```
backend/app/services/email_templates/
├── __init__.py                    # Module initialization and exports
├── approval_pending.py            # Template for new approval requests
├── approval_forwarded.py          # Template for forwarded approvals
├── approval_approved.py          # Template for approved notifications
└── approval_rejected.py          # Template for rejected notifications
```

## Template Files

### 1. approval_pending.py

**Purpose**: Notify approver of new approval request

**Function**: `render_approval_pending_template()`

**Parameters**:
- `resource_type`: "Policy Revision" or "Policy Deviation"
- `resource_title`: Title of the resource
- `resource_id`: ID of the resource
- `approver_name`: Name of the approver
- `creator_name`: Name of the creator
- `frontend_url`: Frontend URL for approval link

**Usage**:
```python
from app.services.email_templates import render_approval_pending_template

html = render_approval_pending_template(
    resource_type="Policy Revision",
    resource_title="Update Policy XYZ",
    resource_id=123,
    approver_name="John Doe",
    creator_name="Jane Smith",
    frontend_url="http://localhost:5173"
)
```

### 2. approval_forwarded.py

**Purpose**: Notify next approver when approval is forwarded

**Function**: `render_approval_forwarded_template()`

**Parameters**:
- `resource_type`: "Policy Revision" or "Policy Deviation"
- `resource_title`: Title of the resource
- `resource_id`: ID of the resource
- `approver_name`: Name of the next approver
- `previous_approver_name`: Name of the previous approver
- `frontend_url`: Frontend URL for approval link

**Usage**:
```python
from app.services.email_templates import render_approval_forwarded_template

html = render_approval_forwarded_template(
    resource_type="Policy Revision",
    resource_title="Update Policy XYZ",
    resource_id=123,
    approver_name="John Doe",
    previous_approver_name="Jane Smith",
    frontend_url="http://localhost:5173"
)
```

### 3. approval_approved.py

**Purpose**: Notify creator when approval is approved

**Function**: `render_approval_approved_template()`

**Parameters**:
- `resource_type`: "Policy Revision" or "Policy Deviation"
- `resource_title`: Title of the resource
- `resource_id`: ID of the resource
- `creator_name`: Name of the creator
- `approver_name`: Name of the approver who approved
- `frontend_url`: Frontend URL for viewing the resource

**Usage**:
```python
from app.services.email_templates import render_approval_approved_template

html = render_approval_approved_template(
    resource_type="Policy Revision",
    resource_title="Update Policy XYZ",
    resource_id=123,
    creator_name="Jane Smith",
    approver_name="John Doe",
    frontend_url="http://localhost:5173"
)
```

### 4. approval_rejected.py

**Purpose**: Notify creator when approval is rejected

**Function**: `render_approval_rejected_template()`

**Parameters**:
- `resource_type`: "Policy Revision" or "Policy Deviation"
- `resource_title`: Title of the resource
- `resource_id`: ID of the resource
- `creator_name`: Name of the creator
- `approver_name`: Name of the approver who rejected
- `frontend_url`: Frontend URL for viewing the resource

**Usage**:
```python
from app.services.email_templates import render_approval_rejected_template

html = render_approval_rejected_template(
    resource_type="Policy Revision",
    resource_title="Update Policy XYZ",
    resource_id=123,
    creator_name="Jane Smith",
    approver_name="John Doe",
    frontend_url="http://localhost:5173"
)
```

## Module Initialization

The `__init__.py` file exports all template functions for convenient importing:

```python
from app.services.email_templates import (
    render_approval_pending_template,
    render_approval_forwarded_template,
    render_approval_approved_template,
    render_approval_rejected_template
)
```

## Benefits of This Structure

1. **Better Organization**: Each template is in its own file, making it easier to find and edit
2. **Maintainability**: Changes to one template don't affect others
3. **Readability**: Smaller files are easier to read and understand
4. **Version Control**: Git diffs are cleaner when templates are in separate files
5. **Reusability**: Templates can be easily reused or extended

## Customizing Templates

To customize a template:

1. Open the corresponding template file (e.g., `approval_pending.py`)
2. Modify the HTML content and styling
3. Update the function parameters if needed
4. Test the changes

## Adding New Templates

To add a new email template:

1. Create a new file in `email_templates/` directory (e.g., `custom_notification.py`)
2. Define the template function
3. Export it in `__init__.py`
4. Use it in `notification_service.py` or other services

Example:

```python
# email_templates/custom_notification.py
def render_custom_notification_template(
    recipient_name: str,
    message: str,
    frontend_url: str
) -> str:
    html = f"""
    <!DOCTYPE html>
    <html>
    ...
    </html>
    """
    return html
```

Then add to `__init__.py`:
```python
from .custom_notification import render_custom_notification_template

__all__ = [
    ...,
    "render_custom_notification_template",
]
```

## Template Design Guidelines

1. **Responsive Design**: Templates should work on both desktop and mobile
2. **Accessibility**: Use semantic HTML and proper contrast ratios
3. **Email Client Compatibility**: Test in major email clients (Gmail, Outlook, etc.)
4. **Branding**: Use consistent colors and styling
5. **Clear Call-to-Action**: Make action buttons prominent and clear

## Testing Templates

To test a template:

```python
from app.services.email_templates import render_approval_pending_template

html = render_approval_pending_template(
    resource_type="Policy Revision",
    resource_title="Test Policy",
    resource_id=1,
    approver_name="Test User",
    creator_name="Test Creator",
    frontend_url="http://localhost:5173"
)

# Save to file for preview
with open("test_email.html", "w") as f:
    f.write(html)
```

Then open `test_email.html` in a browser to preview.

