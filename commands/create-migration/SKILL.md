---
name: create-migration
description: "Generates a Flyway migration file with timestamp naming. Usage: /create-migration <name>"
---

# Create Migration

Generate a Flyway SQL migration file with proper timestamp-based naming in the correct directory.

## Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `name` | Yes | Migration description in snake_case | `Add_customer_table` |

If the parameter is missing, prompt the user before generating.

## Derived Values

- **timestamp**: Current date-time formatted as `yyyyMMddHHmmss` (e.g., `20260225143000`)
- **filename**: `V{timestamp}__{name}.sql` (two underscores between version and name)
- **Detect the migration directory**: Scan the project for existing `db/migration/` directories. For multi-module core-tier projects, this is in `{service}-models/src/main/resources/db/migration/`. For single-module projects, it is `src/main/resources/db/migration/`.

## File Placement

| Project Type | Location |
|--------------|----------|
| Core tier (multi-module) | `{service}-models/src/main/resources/db/migration/V{timestamp}__{name}.sql` |
| Single-module | `src/main/resources/db/migration/V{timestamp}__{name}.sql` |

If the `db/migration/` directory does not exist, create it.

## Migration Template

```sql
-- Migration: {name}
-- Created: {ISO 8601 timestamp}
-- Description: TODO - describe the purpose of this migration

-- ============================================================
-- UP Migration
-- ============================================================

-- Example: Create table
-- CREATE TABLE table_name (
--     id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
--     name        VARCHAR(255) NOT NULL,
--     description TEXT,
--     status      VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
--     created_at  TIMESTAMP NOT NULL DEFAULT now(),
--     updated_at  TIMESTAMP NOT NULL DEFAULT now()
-- );

-- Example: Add column
-- ALTER TABLE table_name ADD COLUMN new_column VARCHAR(255);

-- Example: Create index
-- CREATE INDEX idx_table_name_column ON table_name (column_name);

-- Example: Add foreign key
-- ALTER TABLE child_table
--     ADD CONSTRAINT fk_child_parent
--     FOREIGN KEY (parent_id) REFERENCES parent_table (id);
```

## Context-Aware Generation

When the user provides context about what the migration should do, generate actual SQL instead of comments. Common patterns:

### Create Table

```sql
CREATE TABLE {table_name} (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- domain columns here
    created_at  TIMESTAMP NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP NOT NULL DEFAULT now()
);
```

### Add Column

```sql
ALTER TABLE {table_name} ADD COLUMN {column_name} {type} {constraints};
```

### Create Index

```sql
CREATE INDEX idx_{table}_{column} ON {table_name} ({column_name});
```

### Add Enum Type

```sql
CREATE TYPE {type_name} AS ENUM ('VALUE_1', 'VALUE_2', 'VALUE_3');
```

### Add Audit Trigger

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_{table}_updated_at
    BEFORE UPDATE ON {table_name}
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

## Naming Conventions

- Table names: `snake_case`, plural (e.g., `customers`, `account_transactions`)
- Column names: `snake_case` (e.g., `created_at`, `account_number`)
- Index names: `idx_{table}_{column}` (e.g., `idx_customers_email`)
- Foreign key names: `fk_{child}_{parent}` (e.g., `fk_accounts_customer`)
- Enum type names: `snake_case` (e.g., `account_status`, `transaction_type`)
- Every table should include `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`, `created_at TIMESTAMP NOT NULL DEFAULT now()`, and `updated_at TIMESTAMP NOT NULL DEFAULT now()`

## After Generation

1. Review the generated SQL and replace placeholder comments with actual schema.
2. Verify the filename ordering: run `ls` on the migration directory to confirm no version conflicts.
3. Remind the user: Flyway migrations are immutable once applied. Never edit a migration that has already been run against a database.
4. Suggest running `mvn flyway:validate` or starting the application to test the migration.
