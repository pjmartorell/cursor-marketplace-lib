---
name: sidekiq-monitoring
description: Query and analyze Sidekiq job lifecycle events from BigQuery for monitoring and troubleshooting. Use when investigating job failures, slow queues, dead jobs, error rates, retry storms, or when the user mentions sidekiq monitoring, job lifecycle, or BQ job events.
---

# Sidekiq Job Lifecycle Monitoring

## Architecture

Sidekiq middleware (`app/middleware/sidekiq/`) publishes job lifecycle events to Google Pub/Sub, which streams into BigQuery.

- **Client middleware** (`client/job_lifecycle.rb`): fires on enqueue (before job runs)
- **Server middleware** (`server/job_lifecycle.rb`): fires on start, success, and error outcomes

## BigQuery Table

```
streaming-west.monitoring.sidekiq_job_lifecycle_events
```

### Schema

| Column | Type | Description |
|---|---|---|
| `event` | STRING | Lifecycle event (see below) |
| `team` | STRING | Owning team: `cdp`, `oe`, `le`, or `null` |
| `job_class` | STRING | Worker class name, e.g. `PumpWorker` |
| `job_id` | STRING | Sidekiq JID |
| `args` | STRING | JSON-serialized `perform` arguments |
| `queue` | STRING | Sidekiq queue name |
| `retry_count` | INTEGER | Retry attempt number (`null` on first run) |
| `error_class` | STRING | Exception class on failure, else `null` |
| `error_message` | STRING | Exception message on failure, else `null` |
| `timestamp` | TIMESTAMP | UTC time the event was recorded |

### Event Types

| Event | Side | Meaning |
|---|---|---|
| `enqueue_success` | Client | Job was pushed to Redis successfully |
| `enqueue_failure` | Client | Job failed to enqueue (with error details) |
| `start` | Server | Job execution began |
| `success` | Server | Job completed without error |
| `retry` | Server | Job failed but will retry (retry_count < max) |
| `dead` | Server | Job failed and exhausted all retries |
| `failure` | Server | Job failed and retries are disabled |

## Running Queries

Use the `bq` CLI. Always filter by day partition for cost efficiency:

```bash
bq query --use_legacy_sql=false --format=json 'YOUR_QUERY'
```

## Common Queries

### Errors today

```sql
SELECT job_class, error_class, error_message, queue, team, timestamp
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP(CURRENT_DATE())
  AND event IN ('failure', 'dead', 'retry')
ORDER BY timestamp DESC
LIMIT 200
```

### Error counts by worker (today)

```sql
SELECT job_class, event, error_class, COUNT(*) AS cnt
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP(CURRENT_DATE())
  AND event IN ('failure', 'dead', 'retry')
GROUP BY job_class, event, error_class
ORDER BY cnt DESC
```

### Dead jobs (exhausted retries)

```sql
SELECT job_class, error_class, error_message, args, queue, team, timestamp
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP(CURRENT_DATE())
  AND event = 'dead'
ORDER BY timestamp DESC
```

### Job throughput by queue (today)

```sql
SELECT queue, event, COUNT(*) AS cnt
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP(CURRENT_DATE())
  AND event IN ('start', 'success')
GROUP BY queue, event
ORDER BY cnt DESC
```

### Jobs by team

```sql
SELECT team, job_class, event, COUNT(*) AS cnt
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP(CURRENT_DATE())
  AND team IS NOT NULL
GROUP BY team, job_class, event
ORDER BY team, cnt DESC
```

### Track a specific job by JID

```sql
SELECT *
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE job_id = 'THE_JID_HERE'
ORDER BY timestamp
```

### Retry storms (workers retrying excessively)

```sql
SELECT job_class, COUNT(*) AS retry_count, MIN(timestamp) AS first_retry, MAX(timestamp) AS last_retry
FROM `streaming-west.monitoring.sidekiq_job_lifecycle_events`
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP(CURRENT_DATE())
  AND event = 'retry'
GROUP BY job_class
HAVING retry_count > 50
ORDER BY retry_count DESC
```

### Specific date range

Replace `CURRENT_DATE()` with a date literal and optionally add time bounds:

```sql
WHERE TIMESTAMP_TRUNC(timestamp, DAY) = TIMESTAMP("2026-03-25")
  AND timestamp BETWEEN "2026-03-25 10:00:00" AND "2026-03-25 12:00:00"
```

## Tips

- Always include the `TIMESTAMP_TRUNC(timestamp, DAY)` partition filter to avoid full-table scans.
- Use `--format=json` for machine-readable output, `--format=pretty` for human-readable tables.
- The `team` field is `null` for workers that don't declare `self.team` in their class. Known teams: `cdp`, `oe`, `le`.
- The `args` column is a JSON string; use `JSON_EXTRACT` or `JSON_EXTRACT_SCALAR` to filter on specific argument values.
