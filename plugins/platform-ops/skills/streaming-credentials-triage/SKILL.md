---
name: streaming-credentials-triage
description: Triage Cloud Function/Cloud Run errors where a company falls through to generic GCP credentials instead of its own service account. Covers Sentry error analysis, Cloud Logging to identify the affected company, reading source code to determine the correct LaunchDarkly project, adding the company to the right LD flag, and creating/updating a Jira ticket. Use when investigating 403 IAM errors in streaming Cloud Functions, "Use generic credentials" log messages, bigquery.external.link flag targeting issues, or resourcemanager.projects.get permission denials.
---

# Streaming Credentials Triage

When a Cloud Function/Cloud Run service logs `Use generic credentials in <company>` and hits a 403, the root cause is almost always that the company is missing from the correct LaunchDarkly `bigquery.external.link` flag — not a GCP IAM issue.

## Key Context

The `getCredentials` pattern used across streaming Cloud Functions:

```go
ff, _ := featureFlag.Check("bigquery.external.link", ldclient.NewUser(company))
if !ff || credentialsStr == nil {
    // Use generic $CREDENTIALS env var
} else {
    // Use company's own GCP credentials from DB
}
```

**There are multiple `bigquery.external.link` flags, one per LaunchDarkly project.** Each service uses a different project's SDK key. Adding a company to the wrong project does nothing.

| LD Project | SDK key env var | Used by |
|---|---|---|
| `default` | — | Management API |
| `push-bigquery` | — | BigQuery Push |
| `workflows` | `LAUNCHDARKLY_SDKKEY` | `tickets-delete-datastore`, `tickets-delete-duplicates`, `tickets-delete-duplicates-by-code` |
| `data-flow` | — | Dataflow Launcher (other services) |

## Workflow

### Step 1: Fetch the Sentry issue
Use `get_sentry_resource` with the Sentry URL. Note the error message, culprit (service name), and timestamp.

### Step 2: Find the affected company via Cloud Logging
Query `streaming-west` (or relevant GCP project) for logs from the service:
```
resource.labels.service_name="<service-name>" AND SEARCH("TicketsDeleteDatastoreRunner")
```
Look for `Use generic credentials in <company>` log entries near the Sentry error timestamp. The company logged ~100-300ms before the Sentry error is the affected one.

### Step 3: Confirm the company has DB credentials
Query the `companies` table in postgres-production:
```sql
SELECT slug, google_project_credentials IS NOT NULL as has_credentials
FROM companies WHERE slug = '<company>';
```
If `has_credentials = false`, the DB is missing credentials (different fix). If `true`, it's a LD flag issue.

### Step 4: Find the correct LD project
Read the service's source code on GitHub (`cloudbuild.yaml` + `launchdarkly.go`) to find which env var stores the SDK key. Then cross-reference:
- Look at which companies ARE working (showing "Use company credentials")
- Check `bigquery.external.link` across LD projects to find which one contains those companies
- The correct project is the one whose production flag matches the working companies

**Shortcut:** For `tickets-delete-datastore` (and `tickets-delete-duplicates*`), the correct project is **`workflows`**.

### Step 5: Add the company to the correct flag
Use `update-individual-targets` on `plugin-launchdarkly-LaunchDarkly_Feature_Management`:
- Project: (found in step 4)
- Flag: `bigquery.external.link`
- Environment: `production`
- Instruction: `addTargets`, variationIndex `0` (true)

Verify with `get-feature-flag` that the company appears in the targets and the flag version bumped.

### Step 6: Test
Trigger the Cloud Function manually:
```bash
curl -X POST "https://europe-west1-streaming-west.cloudfunctions.net/<service-name>" \
  -H "Authorization: bearer $(gcloud auth print-identity-token)"
```
The HTTP body is ignored by these functions. Confirm logs show `Use company credentials in <company>`.

### Step 7: Create/update Jira ticket
- If it's a new issue → create in **CDP** board (not PLAT — this is not a GCP IAM issue)
- Keep the description short: what company, which flag was missing, fix applied
- Link to the Sentry issue

## Common Mistakes

- **Wrong LD project**: Adding to `data-flow` when the service uses `workflows` (or vice versa). Always verify by cross-referencing working companies.
- **Routing to Platform**: This looks like a GCP IAM permission error but is actually a LaunchDarkly targeting gap. No infra changes needed.
- **Not checking DB credentials**: If `google_project_credentials IS NULL`, adding to the LD flag won't help — the DB column needs to be populated first.
