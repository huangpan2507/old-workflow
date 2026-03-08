# Migration Issue and Solution

## Issue
The data migration in the Alembic migration script has complex dependencies that are failing. The subquery for finding matching node IDs returns NULL values.

## Solution
Skip the automated data migration in the first migration and handle it manually or in a follow-up migration after the schema is created.

**For now:**
1. The schema migration will create all new tables
2. The Policy approval templates API will continue to work (uses old tables)
3. Data migration can be done manually or via a separate migration script later

## Alternative: Simplified Migration

The migration file can be simplified to only:
1. Create new tables
2. Add the approval_instance_id column to policies
3. Skip the data migration portion for now

This allows the schema to be created without errors, and the application can be tested with the new structure while old data remains in the old tables.

## Manual Steps After Schema Migration

Once the schema is created:
1. Manually copy templates from `policy_approval_templates` to `approval_workflow_templates` with `resource_type='policy'`
2. Copy node definitions
3. Copy node users and roles
4. Create a separate, simpler migration for data migration if needed

