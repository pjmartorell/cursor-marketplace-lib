---
name: safe-migrations
description: Write safe, zero-downtime Rails database migrations for the Apartment multi-tenant setup. Use when creating, reviewing, or modifying database migrations, adding columns, indexes, tables, or constraints, or when the user mentions migrations, schema changes, or database safety.
---

# Safe Migrations

## Architecture Context

- **Multi-tenant via Apartment gem** (`config/initializers/apartament.rb`)
- Each `Company` maps to a PostgreSQL **schema** (`Company.pluck(:slug)`)
- **~87 tenant schemas** — every migration runs once per tenant sequentially
- **Excluded models** (public schema only): `Company`, `User`, `Admin`, `UserLocation`, `UserCountry`, `Jobtitude`, `SystemStatus`, `SystemStatusLog`, `PasswordHistory`, `CompanyConfiguration`, `TriggerType`, `TriggerAction`, `ScheduledExecution`
- Migrations deploy via **Cloud Run job** (`migrate-database-job`) with a finite timeout
- PostgreSQL 11+ (supports metadata-only `ADD COLUMN ... DEFAULT`)

## Critical Rule: Time Budget

Every migration runs **~87 times** (once per tenant). A 5-second operation becomes ~7 minutes total. A 30-second operation becomes ~43 minutes and **will timeout the deployment**.

Before writing any migration, estimate: `per_tenant_time × 87 = total_time`. The Cloud Run job timeout must accommodate the total.

## Rules

### 1. Always add indexes concurrently

```ruby
# WRONG — blocks writes on the table for every tenant
class AddIndex < ActiveRecord::Migration[6.1]
  def change
    add_index :orders, :customer_id
  end
end

# CORRECT
class AddIndex < ActiveRecord::Migration[6.1]
  disable_ddl_transaction!

  def change
    add_index :orders, :customer_id, algorithm: :concurrently
  end
end
```

### 2. Never combine `add_column` and `add_index concurrently` in one migration

`disable_ddl_transaction!` disables the DDL transaction for the **entire** migration. If the index creation fails, the column already exists and re-running fails with "column already exists."

```ruby
# WRONG — not atomic, not safely re-runnable
class AddPermissions < ActiveRecord::Migration[6.1]
  disable_ddl_transaction!

  def change
    add_column :accounts, :permissions, :string, array: true, default: []
    add_index :accounts, :permissions, using: :gin, algorithm: :concurrently
  end
end

# CORRECT — split into two migrations
class AddPermissionsColumn < ActiveRecord::Migration[6.1]
  def change
    add_column :accounts, :permissions, :string, array: true, default: []
  end
end

class AddPermissionsIndex < ActiveRecord::Migration[6.1]
  disable_ddl_transaction!

  def change
    add_index :accounts, :permissions, using: :gin, algorithm: :concurrently
  end
end
```

### 3. Excluded-model tables: Apartment runs migrations per tenant

Tables for excluded models (`accounts`, `companies`, etc.) live **only in the public schema**. However, Apartment still runs the migration against every tenant's `search_path`. Most DDL statements (e.g. `add_column`, `rename_column`) are idempotent or target the public table regardless — no special guard is needed.

For index creation with `algorithm: :concurrently`, use an `index_name_exists?` guard (not `index_exists?`) to make it safely re-runnable. `index_exists?` is schema-path-sensitive and returns false when Apartment sets a tenant search path, even if the index already exists in the public schema — causing a retry loop.

```ruby
class AddIndexToAccounts < ActiveRecord::Migration[6.1]
  disable_ddl_transaction!

  def change
    unless index_name_exists?(:accounts, 'index_accounts_on_permissions')
      add_index :accounts, :permissions, using: :gin, algorithm: :concurrently
    end
  end
end
```

### 4. Adding columns with defaults is safe (PG 11+)

PostgreSQL 11+ handles `ADD COLUMN ... DEFAULT <non-volatile>` as a **metadata-only** operation. No table rewrite. This is fast regardless of table size.

```ruby
# SAFE — metadata-only, completes in milliseconds
add_column :orders, :status, :string, default: "pending"
add_column :users, :permissions, :string, array: true, default: []
```

**Volatile defaults still rewrite the table.** Split into three steps: add column, backfill existing rows, then set the default for new rows.

```ruby
# DANGEROUS — rewrites entire table
add_column :users, :uuid, :uuid, default: "gen_random_uuid()"

# SAFE — Step 1: add column + set default (separate migration)
class AddUuidToUsers < ActiveRecord::Migration[6.1]
  def up
    add_column :users, :uuid, :uuid
    change_column_default :users, :uuid, from: nil, to: "gen_random_uuid()"
  end

  def down
    remove_column :users, :uuid
  end
end

# SAFE — Step 2: backfill existing rows (separate migration)
class BackfillUuidOnUsers < ActiveRecord::Migration[6.1]
  disable_ddl_transaction!

  def up
    User.unscoped.in_batches(of: 10_000) do |relation|
      relation.where(uuid: nil).update_all("uuid = gen_random_uuid()")
      sleep(0.01)
    end
  end
end
```

### 5. Never remove a column without ignoring it first

Active Record caches column names. Dropping a column causes exceptions until the app restarts.

```ruby
# Step 1: Deploy code that ignores the column
class User < ApplicationRecord
  self.ignored_columns += ["legacy_field"]
end

# Step 2: After deploy, create migration
class RemoveLegacyField < ActiveRecord::Migration[6.1]
  def change
    remove_column :users, :legacy_field
  end
end
```

### 6. Never rename columns or tables in-place

Renaming causes immediate application errors. Instead: create new → write to both → backfill → migrate reads → drop old.

### 7. Setting NOT NULL safely

```ruby
# WRONG — scans and locks every row
change_column_null :users, :email, false

# CORRECT — two-step with check constraint
class AddNotNullConstraint < ActiveRecord::Migration[6.1]
  def change
    add_check_constraint :users, "email IS NOT NULL",
      name: "users_email_null", validate: false
  end
end

class ValidateNotNullConstraint < ActiveRecord::Migration[6.1]
  def change
    validate_check_constraint :users, name: "users_email_null"
    change_column_null :users, :email, false
    remove_check_constraint :users, name: "users_email_null"
  end
end
```

### 8. Adding foreign keys safely

```ruby
# WRONG — locks both tables
add_foreign_key :orders, :users

# CORRECT — add unvalidated, then validate separately
class AddForeignKey < ActiveRecord::Migration[6.1]
  def change
    add_foreign_key :orders, :users, validate: false
  end
end

class ValidateForeignKey < ActiveRecord::Migration[6.1]
  def change
    validate_foreign_key :orders, :users
  end
end
```

### 9. Backfilling data safely

Never backfill in the same transaction as a schema change.

```ruby
class BackfillStatus < ActiveRecord::Migration[6.1]
  disable_ddl_transaction!

  def up
    Order.unscoped.in_batches(of: 10_000) do |relation|
      relation.where(status: nil).update_all(status: "pending")
      sleep(0.01)
    end
  end
end
```

## Migration Checklist

Before merging any migration, verify:

- [ ] Indexes use `algorithm: :concurrently` with `disable_ddl_transaction!`
- [ ] `add_column` and `add_index concurrently` are in **separate** migrations
- [ ] Concurrent index additions use `index_name_exists?` guard (not `index_exists?`) for idempotency
- [ ] No volatile defaults on `add_column`
- [ ] No `remove_column` without `self.ignored_columns` deployed first
- [ ] No `rename_column` or `rename_table`
- [ ] No `change_column_null` without check constraint pattern
- [ ] Foreign keys added with `validate: false`, validated separately
- [ ] Estimated total time: `per_tenant_seconds × 87` fits within Cloud Run timeout
- [ ] Backfills use `disable_ddl_transaction!` with batching and throttling

## Reference

- [PostgreSQL at Scale: Schema Changes Without Downtime](https://medium.com/braintree-product-technology/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680)
