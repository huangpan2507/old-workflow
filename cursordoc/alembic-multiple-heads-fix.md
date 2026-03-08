# Alembic Multiple Heads Issue - Fix Documentation

## Problem

When running `alembic upgrade head`, the following error occurred:

```
ERROR [alembic.util.messaging] Multiple head revisions are present for given argument 'head'; 
please specify a specific target revision, '<branchname>@head' to narrow to a specific head, 
or 'heads' for all heads
```

This happens when Alembic migration history has multiple "head" revisions (migrations that are not referenced by any other migration).

## Root Cause

The migration history had two main head revisions:
1. `s20260101_add_profile_req` - User profile update requests migration
2. `db436762d9a1` - Merge migration for employee level removal and function names normalization

These two heads needed to be merged into a single head.

## Solution

Created a merge migration file: `m20260104_merge_profile_req_and_employee_level_heads.py`

This merge migration:
- Merges `s20260101_add_profile_req` and `db436762d9a1` into a single head
- Contains no schema changes (standard for merge migrations)
- Creates revision `m20260104_merge_heads`

## File Location

`backend/app/alembic/versions/m20260104_merge_profile_req_and_employee_level_heads.py`

## Next Steps

After creating the merge migration, you can now run:

```bash
docker compose exec backend alembic upgrade head
```

Or it will run automatically during the prestart phase.

## Note

There may be other head revisions in the migration history that are not actively used. These are typically old migration branches that were never properly merged. The merge migration created here addresses the immediate issue with the two main active heads.

If you encounter similar issues in the future, you can:

1. Check all heads: `alembic heads`
2. Check migration history: `alembic history`
3. Create a merge migration: `alembic merge -m "merge description" head1 head2`

