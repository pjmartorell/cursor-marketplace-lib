---
name: cloudwatch-streaming
description: Query AWS CloudWatch logs and metrics for the Streaming API Gateway using the awslabs.cloudwatch-mcp-server MCP tool. Use when investigating API Gateway 429 rate limits, streaming API errors, execution logs, latency issues, or when the user mentions CloudWatch, API Gateway logs, 429 errors, rate limiting, or streaming API monitoring.
---

# CloudWatch Streaming API Gateway

## MCP Server

- **Server**: `user-awslabs.cloudwatch-mcp-server`
- **Region**: `eu-west-1` (always pass explicitly)
- **Profile**: `default` (pass explicitly to avoid config resolution errors)

## Streaming API Gateway

| Name | ID | Log Group |
|------|-----|-----------|
| `streaming` | `u9zut7wlwj` | `API-Gateway-Execution-Logs_u9zut7wlwj/production` |

ARN: `arn:aws:logs:eu-west-1:770705722922:log-group:API-Gateway-Execution-Logs_u9zut7wlwj/production`

### Other API Gateways (same account)

| Name | ID | Purpose |
|------|-----|---------|
| `api-streaming-v1` | `brscrjt2h1` | Streaming v1 (legacy) |
| `stamps_proxy` | `2wjdqn8cwh` | Bonpreu stamps |
| `pyrenees_proxy` | `fvawqyqo0e` | Pyrenees proxy |
| `cuevas_proxy` | `9jgd8di5j3` | Cuevas proxy |
| `sendgrid` | `kkhpyvc7w0` | SendGrid webhook |
| `Maintentance` | `dn0wv906ic` | Maintenance mode |

## Available Tools

### Logs

| Tool | Purpose |
|------|---------|
| `describe_log_groups` | Discover log groups by prefix |
| `execute_log_insights_query` | Run CloudWatch Logs Insights queries (waits for results) |
| `get_logs_insight_query_results` | Retrieve results of a timed-out query |
| `cancel_logs_insight_query` | Cancel a running query |
| `analyze_log_group` | Detect anomalies, top patterns, error patterns |

### Metrics

| Tool | Purpose |
|------|---------|
| `get_metric_data` | Retrieve metric time-series data |
| `analyze_metric` | Analyze seasonality, trend, statistics |
| `get_metric_metadata` | Get metric description, unit, recommended stats |

### Alarms

| Tool | Purpose |
|------|---------|
| `get_active_alarms` | List all alarms currently in ALARM state |
| `get_alarm_history` | Get state transition history for an alarm |
| `get_recommended_metric_alarms` | Get alarm recommendations for a metric |

## Common Queries

### Query 429 rate-limit errors (hourly distribution)

```json
{
  "server": "user-awslabs.cloudwatch-mcp-server",
  "toolName": "execute_log_insights_query",
  "arguments": {
    "log_group_names": ["API-Gateway-Execution-Logs_u9zut7wlwj/production"],
    "start_time": "2026-03-25T00:00:00+00:00",
    "end_time": "2026-03-26T00:00:00+00:00",
    "query_string": "filter @message like /Method completed with status: 429/ | stats count(*) as cnt by bin(1h) as hour | sort hour",
    "limit": 50,
    "region": "eu-west-1",
    "profile_name": "default"
  }
}
```

### Query specific HTTP status codes

Replace `429` with any status code (500, 502, 503, etc.):

```
filter @message like /Method completed with status: <CODE>/
| stats count(*) as cnt by bin(1h) as hour
| sort hour
```

### Find requests to a specific path

```
filter @message like /Resource Path:/ and @message like /activities\/realtime/
| fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

### Find requests by API Key suffix

```
filter @message like /API Key:/ and @message like /b7162d/
| fields @timestamp, @message
| sort @timestamp desc
| limit 50
```

### Analyze error patterns in the log group

```json
{
  "server": "user-awslabs.cloudwatch-mcp-server",
  "toolName": "analyze_log_group",
  "arguments": {
    "log_group_arn": "arn:aws:logs:eu-west-1:770705722922:log-group:API-Gateway-Execution-Logs_u9zut7wlwj/production",
    "start_time": "2026-03-25T00:00:00+00:00",
    "end_time": "2026-03-26T00:00:00+00:00",
    "region": "eu-west-1",
    "profile_name": "default"
  }
}
```

### Get API Gateway metrics (Count, 4xx, 5xx, Latency)

```json
{
  "server": "user-awslabs.cloudwatch-mcp-server",
  "toolName": "get_metric_data",
  "arguments": {
    "namespace": "AWS/ApiGateway",
    "metric_name": "Count",
    "start_time": "2026-03-25T00:00:00+00:00",
    "end_time": "2026-03-26T00:00:00+00:00",
    "dimensions": [{"name": "ApiName", "value": "streaming"}],
    "statistic": "SUM",
    "region": "eu-west-1",
    "profile_name": "default"
  }
}
```

Available metric names: `Count`, `4XXError`, `5XXError`, `Latency`, `IntegrationLatency`.

## Known Issues

### AWS session expiry

If you see `Your session has expired`, the user must reauthenticate: `aws login`

### MCP config requirements

The MCP server (`~/.cursor/mcp.json`) must have:
- `"command": "uvx"` with `"args": ["--with", "botocore[crt]", "awslabs.cloudwatch-mcp-server@latest"]` for CRT dependency
- `"AWS_PROFILE": "default"` (not the account ID)

## Log Message Format

API Gateway execution logs use this structure per request (each is a separate log entry with a request ID prefix):

```
(<request-id>) Starting execution for request: <request-id>
(<request-id>) HTTP Method: POST, Resource Path: /activities/realtime
(<request-id>) API Key: ****...suffix
(<request-id>) API Key ID: <key-id>
(<request-id>) Verifying Usage Plan for request: ...
(<request-id>) Usage Plan check succeeded for API Key ...
(<request-id>) Method completed with status: 200
(<request-id>) Successfully completed execution
```

When rate-limited (429), the request is rejected at the Usage Plan check and the log shows `Method completed with status: 429` without the integration/backend execution entries.
