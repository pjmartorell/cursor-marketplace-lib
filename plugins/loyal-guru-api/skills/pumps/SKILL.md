---
name: pumps
description: Work with the Pumps system that uploads PostgreSQL data to BigQuery. Use when creating, modifying, or debugging pumps, BigQuery jobs, pump JSON configs, or when the user mentions pumps, BigQuery, or data uploads to BQ.
---

# Pumps — PostgreSQL to BigQuery

For full architecture details, see [/docs/PUMPS.md](/docs/PUMPS.md).

## Key files

| Area | Path |
|------|------|
| JSON configs (one per table) | `config/pumps/` |
| Core execution | `app/models/bigquery/pump.rb` |
| JSON reader / schema builder | `app/models/bigquery/pump_definition.rb` |
| CSV generator (PG COPY) | `app/models/bigquery/pump_data_file_generator.rb` |
| Append type logic | `app/models/bigquery/append_definition.rb` |
| Append type auto-detection | `app/models/bigquery/pump_validator.rb` |
| Allowed tables list | `app/models/company.rb` → `ALLOWED_PUMPS` |
| Job scheduling | `app/workers/bigquery_job_scheduler_worker.rb` |
| API endpoint | `app/api/v1/bigquery_jobs.rb` |

## Workflow: Create a new pump

1. **Create the JSON config** in `config/pumps/<table_name>.json`:

```json
{
  "sql": "SELECT id, col1, col2, created_at, updated_at FROM <table_name>",
  "schema": {
    "fields": [
      { "name": "id", "type": "INTEGER" },
      { "name": "col1", "type": "STRING" },
      { "name": "col2", "type": "INTEGER" },
      { "name": "created_at", "type": "TIMESTAMP" },
      { "name": "updated_at", "type": "TIMESTAMP" }
    ]
  }
}
```

2. **Include `id`** — always required.
3. **Include `updated_at`** if you want incremental (append) pumps. `PumpValidator` auto-selects append type based on available columns:
   - Has `id` + `updated_at` → `daily` append
   - Has only `id` → `by_id` append
   - Neither → full pump only
4. **Add the table to `Company::ALLOWED_PUMPS`** in `app/models/company.rb`.
5. **Register it in a job** — add the table to `generate_main_pumps_job` (full) or `generate_main_pumps_append_job` (append) in `Company`.
6. **BigQuery types** supported in schema: `INTEGER`, `STRING`, `FLOAT`, `BOOLEAN`, `TIMESTAMP`, `DATE`, `DATETIME`, `RECORD`.

## Workflow: Modify an existing pump

1. Find the JSON in `config/pumps/<table_name>.json`.
2. Update `sql` and/or `schema.fields` — both must stay in sync.
3. If adding/removing columns, the BigQuery table schema must be compatible (adding nullable columns is safe; removing/renaming requires a full reload).
4. Run a full pump after schema changes: `rake pump:full[<table_name>]`.

## Workflow: Debug a failing pump

1. Check if the table is forbidden: `pump_forbidden_tables` feature flag or `Exceptions::ForbiddenPump`.
2. Check the job status via API: `GET /bigquery_jobs/:id`.
3. Verify the JSON config is valid: `PumpDefinition.new("<table_name>")`.
4. Check the execution chain: `BigqueryJobSchedulerWorker` → `BigqueryJobWorker` → `PumpWorker` → `Bigquery::Pump#execute`.
5. `PumpWorker` uses a semaphore — concurrent pump execution is blocked. Check for stuck semaphores if pumps appear stalled.

## Gotchas

- **Never delete columns** from a pump JSON without a full reload — BigQuery will reject rows with missing fields.
- The `pump_service` feature flag is **deprecated** — ignore `ExternalClients::Pump` code paths.
- Append pumps do DELETE + INSERT in a transaction (merge). If the temporary BQ table load fails, no data is lost.
- `PumpDataFileGenerator` uses PostgreSQL `COPY ... TO STDOUT` for CSV — SQL must be valid PostgreSQL, not BigQuery SQL.
- Rake task `pump:full[table]` runs for **all companies** — use with caution.
