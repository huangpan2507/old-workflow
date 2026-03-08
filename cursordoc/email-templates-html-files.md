# Email Templates - HTML Files Structure

## Overview

Email templates are now stored as independent HTML files using Jinja2 template engine. This allows for better separation of concerns and easier editing by non-developers.

## File Structure

```
backend/app/services/email_templates/
├── __init__.py                    # Module exports (unchanged API)
├── template_loader.py             # Jinja2 template loader and renderer
├── approval_pending.py            # Python wrapper for approval_pending.html
├── approval_forwarded.py          # Python wrapper for approval_forwarded.html
├── approval_approved.py           # Python wrapper for approval_approved.html
├── approval_rejected.py           # Python wrapper for approval_rejected.html
└── templates/                     # HTML template files directory
    ├── approval_pending.html
    ├── approval_forwarded.html
    ├── approval_approved.html
    └── approval_rejected.html
```

## HTML Template Files

All HTML templates are stored in the `templates/` directory:

### 1. approval_pending.html

**Purpose**: Notify approver of new approval request

**Variables**:
- `approver_name` - Name of the approver
- `creator_name` - Name of the creator
- `resource_type` - "Policy Revision" or "Policy Deviation"
- `resource_title` - Title of the resource
- `resource_id` - ID of the resource
- `frontend_url` - Frontend base URL
- `approval_url` - Full URL to approval page

### 2. approval_forwarded.html

**Purpose**: Notify next approver when approval is forwarded

**Variables**:
- `approver_name` - Name of the next approver
- `previous_approver_name` - Name of the previous approver
- `resource_type` - "Policy Revision" or "Policy Deviation"
- `resource_title` - Title of the resource
- `resource_id` - ID of the resource
- `frontend_url` - Frontend base URL
- `approval_url` - Full URL to approval page

### 3. approval_approved.html

**Purpose**: Notify creator when approval is approved

**Variables**:
- `creator_name` - Name of the creator
- `approver_name` - Name of the approver who approved
- `resource_type` - "Policy Revision" or "Policy Deviation"
- `resource_title` - Title of the resource
- `resource_id` - ID of the resource
- `frontend_url` - Frontend base URL
- `view_url` - Full URL to view the resource

### 4. approval_rejected.html

**Purpose**: Notify creator when approval is rejected

**Variables**:
- `creator_name` - Name of the creator
- `approver_name` - Name of the approver who rejected
- `resource_type` - "Policy Revision" or "Policy Deviation"
- `resource_title` - Title of the resource
- `resource_id` - ID of the resource
- `frontend_url` - Frontend base URL
- `view_url` - Full URL to view the resource

## Template Syntax

Templates use Jinja2 syntax for variable substitution:

```html
<!-- Simple variable -->
<p>Dear {{ approver_name }},</p>

<!-- Variable in HTML attribute -->
<a href="{{ approval_url }}">Click here</a>

<!-- Conditional (if needed) -->
{% if resource_type == "Policy Revision" %}
    <p>This is a policy revision</p>
{% endif %}

<!-- Loops (if needed) -->
{% for item in items %}
    <li>{{ item }}</li>
{% endfor %}
```

## Template Loader

The `template_loader.py` module provides:

- **Jinja2 Environment**: Configured with auto-escaping for security
- **Template Loading**: Loads templates from the `templates/` directory
- **Rendering**: Renders templates with provided variables

### Usage

```python
from app.services.email_templates.template_loader import render_template

html = render_template(
    "approval_pending.html",
    approver_name="John Doe",
    creator_name="Jane Smith",
    resource_type="Policy Revision",
    resource_title="Update Policy XYZ",
    resource_id=123,
    frontend_url="http://localhost:5173",
    approval_url="http://localhost:5173/approvals"
)
```

## Python Wrapper Functions

Each template has a corresponding Python wrapper function that:

1. Prepares the data (e.g., constructs URLs)
2. Calls the template loader
3. Returns the rendered HTML

**API remains unchanged** - existing code doesn't need to be modified:

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

## Benefits

### 1. **Separation of Concerns**
- HTML templates are separate from Python code
- Designers can edit HTML without touching Python
- Easier to maintain and version control

### 2. **Better Editing Experience**
- HTML files can be opened in any text editor or IDE
- Syntax highlighting for HTML/CSS
- Can be previewed directly in browsers

### 3. **Template Reusability**
- Templates can be shared with frontend developers
- Can be used for other purposes (e.g., PDF generation)
- Easier to create variations

### 4. **Version Control**
- Git diffs are cleaner for HTML changes
- Easier to track template changes over time
- Can use Git blame to see who changed what

### 5. **Testing**
- Can test templates independently
- Can validate HTML structure
- Can preview templates with sample data

## Editing Templates

### Direct HTML Editing

1. Open the HTML file in your editor:
   ```bash
   vim backend/app/services/email_templates/templates/approval_pending.html
   ```

2. Make your changes to HTML/CSS

3. Test the changes:
   ```python
   from app.services.email_templates import render_approval_pending_template
   
   html = render_approval_pending_template(...)
   # Save to file and open in browser to preview
   ```

### Using Jinja2 Features

You can use Jinja2 features in templates:

```html
<!-- Conditional rendering -->
{% if resource_type == "Policy Revision" %}
    <p>Policy Revision</p>
{% else %}
    <p>Policy Deviation</p>
{% endif %}

<!-- Filters -->
<p>{{ resource_title|upper }}</p>
<p>{{ approver_name|title }}</p>

<!-- Includes (if needed) -->
{% include "common/header.html" %}
```

## Adding New Templates

1. **Create HTML file** in `templates/` directory:
   ```bash
   touch backend/app/services/email_templates/templates/my_template.html
   ```

2. **Create Python wrapper**:
   ```python
   # backend/app/services/email_templates/my_template.py
   from .template_loader import render_template
   
   def render_my_template(param1: str, param2: str) -> str:
       return render_template(
           "my_template.html",
           param1=param1,
           param2=param2
       )
   ```

3. **Export in `__init__.py`**:
   ```python
   from .my_template import render_my_template
   
   __all__ = [
       ...,
       "render_my_template",
   ]
   ```

## Testing Templates

### Test Template Rendering

```python
from app.services.email_templates import render_approval_pending_template

# Test with sample data
html = render_approval_pending_template(
    resource_type="Policy Revision",
    resource_title="Test Policy",
    resource_id=1,
    approver_name="Test Approver",
    creator_name="Test Creator",
    frontend_url="http://localhost:5173"
)

# Save to file for preview
with open("test_email.html", "w") as f:
    f.write(html)

# Open test_email.html in browser
```

### Validate HTML

```python
from html.parser import HTMLParser

class HTMLValidator(HTMLParser):
    def __init__(self):
        super().__init__()
        self.errors = []
    
    def error(self, message):
        self.errors.append(message)

# Validate rendered HTML
validator = HTMLValidator()
validator.feed(html)
if validator.errors:
    print("HTML errors:", validator.errors)
```

## Best Practices

1. **Keep Templates Simple**: Avoid complex logic in templates
2. **Use CSS Inline**: Email clients have limited CSS support
3. **Test in Email Clients**: Test rendered emails in Gmail, Outlook, etc.
4. **Escape Variables**: Jinja2 auto-escaping is enabled for security
5. **Document Variables**: Document all template variables in comments
6. **Version Control**: Commit template changes separately from code changes

## Troubleshooting

### Template Not Found

**Error**: `TemplateNotFound: approval_pending.html`

**Solution**: Ensure the template file exists in `templates/` directory

### Variable Not Defined

**Error**: `UndefinedError: 'variable_name' is undefined`

**Solution**: Ensure all variables are passed to `render_template()`

### HTML Not Rendering

**Issue**: Variables show as `{{ variable_name }}` in output

**Solution**: Check that Jinja2 syntax is correct and variables are passed

## Migration Notes

- **No code changes required**: Existing code continues to work
- **Backward compatible**: Python API remains the same
- **Gradual migration**: Can migrate templates one at a time
- **Easy rollback**: Can revert to Python strings if needed

