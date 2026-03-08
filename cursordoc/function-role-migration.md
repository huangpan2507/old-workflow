# Function Role Code Migration

## Overview

This document describes the function role code migration functionality added to `function_roles.py`. The migration updates existing function role codes from the old format (uppercase/underscore) to the new format that matches the Library column values from the mapping table.

## Migration Functions

### 1. `migrate_function_role_codes(session: Session)`

**Purpose**: Migrates existing function role codes from old format to new format.

**Code Mappings**:
```python
code_migrations = {
    "ADMISSIONS": "Admissions",
    "MARKETING": "Marketing", 
    "COMMUNICATIONS": "Communications",
    "COUNSELLING": "Counselling",
    "FACILITIES": "Facilities",
    "FINANCE": "Finance",
    "JUNIOR_SCHOOL": "Junior School",
    "LIBRARY": "Library",
    "NURSE": "Nurse",
    "OPERATIONS": "Operations",
    "PASTORAL": "Pastoral",
    "SENIOR_SCHOOL": "Senior School",
    "TEACHING_SUPPORT": "Teaching Support",
    "LEGAL": "Legal",
    "MUSIC": "Music"
    # Note: CLT, DUCKS, HR, IS, IT, BC&D, BD, PE remain unchanged
}
```

**Features**:
- Checks for existing roles with old codes
- Prevents conflicts by checking if new code already exists
- Updates codes safely with transaction commits
- Provides detailed logging of migration progress

### 2. `run_function_role_migration_only(session: Session)`

**Purpose**: Runs only the migration without creating new roles.

**Use Case**: When you only want to migrate existing data without initializing new roles.

### 3. `init_function_roles(session: Session)` (Updated)

**Purpose**: Initializes function roles with migration support.

**Process**:
1. First runs `migrate_function_role_codes()` to update existing codes
2. Then creates any missing roles with new code format

## Usage Examples

### Running Migration Only
```python
from sqlmodel import Session
from app.core.database.default_data.function_roles import run_function_role_migration_only

# Get your database session
session = Session(engine)

# Run migration only
run_function_role_migration_only(session)
```

### Running Full Initialization (with Migration)
```python
from sqlmodel import Session
from app.core.database.default_data.function_roles import init_function_roles

# Get your database session
session = Session(engine)

# Run full initialization (migration + creation)
init_function_roles(session)
```

### Manual Migration
```python
from sqlmodel import Session
from app.core.database.default_data.function_roles import migrate_function_role_codes

# Get your database session
session = Session(engine)

# Run migration manually
migrate_function_role_codes(session)
```

## Migration Safety Features

### Conflict Prevention
- Checks if target code already exists before migration
- Skips migration if conflict is detected
- Logs warnings for skipped migrations

### Transaction Safety
- Each migration is committed individually
- Failed migrations don't affect other migrations
- Detailed logging for troubleshooting

### Rollback Considerations
- Migration is one-way (old codes -> new codes)
- No automatic rollback functionality
- Manual rollback would require reverse mapping

## Expected Output

When running migration, you should see output like:
```
Starting function role code migration...
Migrated function role code: ADMISSIONS -> Admissions
Migrated function role code: MARKETING -> Marketing
Migrated function role code: JUNIOR_SCHOOL -> Junior School
Migrated function role code: TEACHING_SUPPORT -> Teaching Support
No existing role found with code: FACILITIES
Function role code migration completed.
```

## Integration Points

### Database Initialization
The migration is automatically run when `init_function_roles()` is called, ensuring that:
- Existing data is migrated to new format
- New installations get the correct format from the start
- No manual intervention required for most cases

### Alembic Migrations
This migration should be run as part of a database migration script or during application startup to ensure all existing data is updated.

## Testing Recommendations

1. **Backup Database**: Always backup before running migration
2. **Test Environment**: Test migration in development environment first
3. **Verify Results**: Check that all codes match expected Library column values
4. **Check Dependencies**: Ensure no foreign key constraints are broken

## Troubleshooting

### Common Issues
- **Conflicts**: If target code already exists, migration will be skipped
- **Missing Roles**: Roles not found with old codes will be logged but not cause errors
- **Transaction Issues**: Each migration is committed individually to prevent rollback issues

### Debugging
- Check console output for detailed migration logs
- Verify database state before and after migration
- Check for any foreign key constraint violations
