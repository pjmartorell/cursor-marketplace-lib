---
name: customer-attribute-sqid-audit
description: Audit BigQuery→PostgreSQL scheduled_query_id mismatches in the Customer Attributes system. Use when investigating Sentry errors about "CustomerAttributeDefinition not found", BigQuery transfer config ID mismatches, or when the user mentions scheduled_query_id, BQ transfer configs, or customer attribute execution notification failures.
---

# Customer Attribute scheduled_query_id Audit

## Background

When BigQuery finishes a scheduled query, it sends a Pub/Sub notification to `/api/v1/customer_attribute_executions`. The handler calls `Services::CustomerAttributes::Definition::ProcessExecutionNotification`, which searches for a `CustomerAttributeDefinition` by `scheduled_query_id`. If the UUID stored in PostgreSQL doesn't match the BQ transfer config ID in the notification, the lookup fails with a Sentry exception.

The `scheduled_query_id` column stores the **full resource path**:
```
projects/<bq_project_id>/locations/europe/transferConfigs/<uuid>
```

The parsing logic in `execution_event.rb` extracts the name using `.split(/\runs/).first`.

**Common causes of mismatch:**
- BQ transfer configs were recreated (new UUID assigned) but PostgreSQL was not updated
- `scheduled_query_id` was never set (empty string or NULL)
- Trailing newlines in the stored value

---

## Audit Workflow

### Step 1 — Find affected tenants in Sentry

Use the `user-Sentry` MCP tool. The relevant Sentry issue title is:
`Services::CustomerAttributes::Definition::ProcessExecutionNotification.search_and_change_company`

```
search_issues: query="Services::CustomerAttributes::Definition::ProcessExecutionNotification"
get_issue_tag_values: tag="bigquery_dataset_id" or "schedule_query_id"
search_issue_events: date ranges to analyze recency
```

Extract from event context:
- `bigquery_dataset_id` → identifies the tenant
- `schedule_query_id` → the UUID BQ is sending (this is what PG should store)

### Step 2 — Map datasets to tenant schemas

Using the `user-postgres-production` MCP tool:

```sql
SELECT slug, bigquery_dataset_id, bigquery_project_id
FROM public.companies
WHERE bigquery_dataset_id IN ('dataset1', 'dataset2', ...)
ORDER BY slug;
```

Tenant schemas are named after `companies.slug`.

### Step 3 — Inspect customer_attribute_definitions per tenant

Check for two failure modes:

**A) Empty table or no scheduled_query_id set:**
```sql
SELECT COUNT(*), COUNT(scheduled_query_id) FILTER (WHERE scheduled_query_id != '')
FROM <tenant_schema>.customer_attribute_definitions;
```

**B) Stale UUIDs (records exist but IDs are wrong):**
```sql
SELECT id, name, scheduled_query_id, updated_at
FROM <tenant_schema>.customer_attribute_definitions
ORDER BY id;
```

For many tenants at once use a UNION ALL query across schemas.

**Watch for:**
- Empty strings (`scheduled_query_id = ''`)
- Trailing newlines (`scheduled_query_id LIKE '%\n%'` or `char_length(scheduled_query_id) != length(trim(scheduled_query_id))`)
- UUIDs that don't match what Sentry shows BQ is sending

### Step 4 — Classify tenants

| Category | Condition | Fix |
|---|---|---|
| Empty table | No CA definition rows | Recreate definitions in PG or delete BQ transfer configs |
| No sqid set | Rows exist, `scheduled_query_id` is empty/null | Set the correct full BQ resource path |
| Stale UUID | Rows exist, UUID doesn't match current BQ config | Update to new UUID from BQ Data Transfer API |
| Data quality | Trailing newlines in sqid | `UPDATE … SET scheduled_query_id = trim(scheduled_query_id)` |

### Step 5 — Analyze recency

Compare Sentry events across two recent time windows (e.g. one week ago vs. today) to determine if errors are ongoing, stopped, or escalated. Sentry has a max lookback of ~90 days.

### Step 6 — Present and report

1. Create a canvas using the canvas skill for a rich summary table
2. Post findings to the relevant Jira issue with `addCommentToJiraIssue` (markdown format)

---

## Fix SQL Templates

**Update a stale UUID:**
```sql
UPDATE <tenant_schema>.customer_attribute_definitions
SET scheduled_query_id = 'projects/<bq_project>/locations/europe/transferConfigs/<new_uuid>'
WHERE id = <id>;
```

**Strip trailing newlines (bulk):**
```sql
UPDATE <tenant_schema>.customer_attribute_definitions
SET scheduled_query_id = trim(scheduled_query_id)
WHERE scheduled_query_id != trim(scheduled_query_id);
```

---

## Key Files

- `app/api/v1/customer_attribute_executions.rb` — endpoint receiving BQ notifications
- `app/services/customer_attributes/definition/process_execution_notification.rb` — `search_and_change_company` is where the lookup fails
- `app/services/customer_attributes/definition/execution_event.rb` — parses `scheduled_query_id` from the notification payload
