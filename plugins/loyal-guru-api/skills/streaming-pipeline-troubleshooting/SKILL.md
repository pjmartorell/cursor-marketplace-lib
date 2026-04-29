---
name: streaming-pipeline-troubleshooting
description: Troubleshoot the streaming data pipeline — Pub/Sub dead letter queues, BigQuery push failures, pb-resource-consumer errors, and message reprocessing. Use when investigating DLQ messages, Pub/Sub delivery failures, streaming API errors, BigQuery ingestion issues, or when the user mentions dead letter, DLQ, activities-dl, pb-consumer, streaming-push-bigquery, or message reprocessing.
---

# Streaming Pipeline Troubleshooting

## Architecture Overview

```
External data → Connector publishes to per-entity Pub/Sub topics
                    ↓
            pb-resource-consumer (Cloud Run, Python/Flask)
            subscribes via per-company subscriptions
                    ↓
            Calls Streaming API via AWS API Gateway (rate-limited)
                    ↓
            AWS API Gateway → App Engine (loyal-guru-api-streaming-v2)
            POST /activities/realtime, /customers, etc.
                    ↓
            Streaming API saves to Google Datastore (tickets)
            and publishes to "streaming-push-bigquery" topic
                    ↓
            streaming-push-bigquery (Go, App Engine)
            consumes → CSV to GCS → loads into BigQuery
```

### Key GCP project: `streaming-west`
### AWS API Gateway: `streaming` (ID `u9zut7wlwj`, Edge-optimized, eu-west-1)
### CloudWatch log group: `API-Gateway-Execution-Logs_u9zut7wlwj/production`

Other APIs in the same account: `api-streaming-v1` (`brscrjt2h1`), `stamps_proxy` (`2wjdqn8cwh`), `pyrenees_proxy` (`fvawqyqo0e`), `cuevas_proxy` (`9jgd8di5j3`), `sendgrid` (`kkhpyvc7w0`), `Maintentance` (`dn0wv906ic`).

### Repos involved

| Repo | Role | Language |
|------|------|----------|
| `loyalguru/pb-resource-consumer` | Pub/Sub consumer, calls Streaming API | Python/Flask (Cloud Run) |
| `loyal-guru-api-streaming-v2` | HTTP API, saves to Datastore, publishes to BQ topic | Go (App Engine) |
| `streaming-push-bigquery` | Consumes BQ topic, uploads CSV to BigQuery | Go (App Engine) |

## Investigation Workflow

### Step 1: Identify the DLQ message

From the GCP console or CLI, inspect the DLQ message. Key attributes:

| Attribute | What it tells you |
|-----------|-------------------|
| `CloudPubSubDeadLetterSourceSubscription` | Which subscription failed (identifies the pipeline stage) |
| `CloudPubSubDeadLetterSourceDeliveryCount` | How many times delivery was attempted |
| `company_slug` | Which company the message belongs to |
| `entity` | Entity type (activity, customer, score, etc.) |
| `action` | Operation (create, update, delete) |
| `synchro_id` | Which connector sync batch produced this message |
| `source` | Origin (connector, api, etc.) |

### Step 2: Check if the data was processed

Query `entities_queues_logs` in BigQuery to see if the message was ever successfully processed:

```sql
SELECT * FROM `streaming-west.raw_data.entities_queues_logs`
WHERE company_slug = '<COMPANY>'
  AND message LIKE '%<ENTITY_ID>%'
LIMIT 10
```

If **not found**, the consumer crashed before logging — the data never reached the Streaming API successfully.

If **found with status 200**, the Streaming API processed it but a downstream step (BQ push) failed.

### Step 3: Check consumer logs

The consumer is a Cloud Run service. Use the `user-cloud-logging` MCP tool:

```
projectId: streaming-west
filter: resource.type="cloud_run_revision" AND resource.labels.service_name="pb-consumer-<entity>" AND severity>=ERROR
```

Service naming convention: `pb-consumer-activity`, `pb-consumer-customer`, etc.

### Step 4: Check AWS API Gateway logs (CloudWatch)

If you suspect a 429 rate limit, check AWS CloudWatch using the `awslabs.cloudwatch-mcp-server` MCP or the AWS CLI. API Gateway access logs are in `eu-west-1`.

```bash
# Query for 429 errors (hourly distribution)
aws logs start-query \
  --log-group-name 'API-Gateway-Execution-Logs_u9zut7wlwj/production' \
  --start-time <EPOCH_START> \
  --end-time <EPOCH_END> \
  --query-string 'filter @message like /Method completed with status: 429/ | stats count(*) as cnt by bin(1h) as hour | sort hour' \
  --region eu-west-1

# Get results (wait ~15s for the query to complete)
aws logs get-query-results --query-id "<QUERY_ID>" --region eu-west-1
```

**Key indicator for 429**: Message absent from both `entities_queues_logs` AND `streaming_api_raw`, with `JSONDecodeError` in consumer Cloud Run logs. The 429 is returned by the gateway before reaching App Engine.

### Step 5: Check the Streaming API logs

If the consumer successfully called the Streaming API but got an unexpected response:

```
projectId: streaming-west
filter: resource.type="gae_app" AND SEARCH("<entity_id>")
```

### Step 6: Find the original topic

```bash
gcloud pubsub subscriptions describe <SOURCE_SUBSCRIPTION> \
  --project=streaming-west \
  --format="value(topic)"
```

## Common Failure Modes

### 1. `JSONDecodeError` in `pb-resource-consumer`

**Bug location**: `entities/entity.py` → `generate_log()` method.

`response.json()` is called without checking if the response body is valid JSON. If the response has an empty or non-JSON body, the consumer crashes before acking.

**Effect**: Message retries until max delivery count → DLQ. Data never saved. The consumer has correct retry logic for 429/409/500/502+ in `main.py`, but `generate_log()` crashes before it's reached.

### 2. AWS API Gateway rate limiting (429)

The Streaming API is proxied by AWS API Gateway, which is rate-limited. When rate-limited, the gateway returns a **429 with an empty body**. This request **never reaches App Engine**, so there is no record in `streaming_api_raw`. The consumer crashes on `response.json()` (see #1 above).

**Key indicator**: Message absent from both `entities_queues_logs` AND `streaming_api_raw`, with `JSONDecodeError` in consumer logs.

### 3. `stop.queue.consuming` LaunchDarkly flag

If enabled for a company in `streaming-push-bigquery`, messages are Nacked and redelivered indefinitely. Check the flag value.

### 4. Company not found

If the company slug doesn't exist in the DB, `streaming-push-bigquery` Acks (drops) the message. These won't appear in the DLQ.

### 5. BigQuery load failure

`streaming-push-bigquery` moves failed CSV files to `unprocessed/` prefix in GCS. Not a Pub/Sub DLQ issue.

## Reprocessing Messages

### Prerequisites

1. **Confirm the root cause is resolved** — otherwise the message will fail again
2. **Check if data already exists** — tickets in Datastore (NOT PostgreSQL), other entities in their respective stores
3. **Note the Redis dedup window** — `streaming-push-bigquery` deduplicates by `activity_id + company` with 48h TTL

### Republish to original topic

```bash
gcloud pubsub topics publish <ORIGINAL_TOPIC> \
  --project=streaming-west \
  --message='<MESSAGE_BODY>' \
  --attribute='action=create,company_slug=<COMPANY>,entity=<ENTITY>,source=connector,...'
```

Preserve ALL original attributes from the DLQ message (except the `CloudPubSubDeadLetter*` ones).

### Purge DLQ after reprocessing

```bash
gcloud pubsub subscriptions pull <DLQ_SUBSCRIPTION> \
  --project=streaming-west \
  --limit=100 \
  --auto-ack
```

### Alternative: re-trigger connector sync

If many messages are affected, re-triggering the connector sync for the company (`synchro_id`) may be simpler than republishing individual messages.

## Key BigQuery Tables

| Table | Purpose |
|-------|---------|
| `streaming-west.raw_data.entities_queues_logs` | Logs of all messages processed by pb-resource-consumer |
| `streaming-west.raw_data.streaming_api_raw` | Raw request/response logs from the Streaming API itself |

### Check if pb-resource-consumer processed the message

```sql
SELECT entity, status_code, COUNT(*) as count
FROM `streaming-west.raw_data.entities_queues_logs`
WHERE TIMESTAMP_TRUNC(created_at, DAY) = TIMESTAMP('<DATE>')
  AND company_slug = '<COMPANY>'
GROUP BY entity, status_code
ORDER BY count DESC
```

### Check if the Streaming API received the request

Even if `entities_queues_logs` has no record (consumer crashed before logging), the Streaming API may have received and processed the request. Check `streaming_api_raw`:

```sql
SELECT created_at, method, path, response_status_code, response_body
FROM `streaming-west.raw_data.streaming_api_raw`
WHERE company = '<COMPANY>'
  AND request_body LIKE '%<ENTITY_ID>%'
ORDER BY created_at ASC
```

If this shows `200`, the data was saved to Datastore even though the consumer crashed.

## Subscription Naming Conventions

| Pattern | Example |
|---------|---------|
| Main subscription | `activities-<company>-subscription` |
| Dead letter subscription | `activities-dl-sub` |
| BQ push subscription | `sub-streaming-push-bigquery` |
