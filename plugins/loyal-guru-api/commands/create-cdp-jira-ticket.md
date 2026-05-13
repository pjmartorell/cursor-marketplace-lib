---
name: create-cdp-jira-ticket
description: Create a Jira ticket in the CDP project using the Atlassian MCP tool. Use when the user asks to open a bug, task, story, or subtask in the CDP Jira board.
---

Create a Jira ticket in the CDP project using the Atlassian MCP tool (`createJiraIssue`).

## Pre-configured values (do NOT fetch these from Jira)

- **Cloud ID**: `loyal-guru.atlassian.net`
- **Project Key**: `CDP`
- **Team field** (`customfield_10001`): `"56ed7d1d-84db-461e-a323-9d5520602360-7"` (CDP team)
- **Sprint field**: `customfield_10010` (NOT `customfield_10020`)
- **Board ID**: `106` (CDP board)

## Issue type

Ask the user which type to create if not obvious from context.
Use the **Spanish names** for `issueTypeName` (the Jira instance is in Spanish):

| English | `issueTypeName` value | Notes |
|---------|----------------------|-------|
| Bug     | `Error`              | Requires extra fields (see below) |
| Task    | `Tarea`              | Default for general work |
| Story   | `Historia`           | For user stories |
| Subtask | `Subtarea`           | Must include `parent` field |

## Required fields for Bug type

Bugs have mandatory custom fields. Use ADF (Atlassian Document Format) for textarea fields:

| Field | Key | Type | Notes |
|-------|-----|------|-------|
| Steps to reproduce | `customfield_10200` | ADF document | Ordered list of steps |
| Expected Behavior | `customfield_10201` | ADF document | What should happen |
| Severity | `customfield_10628` | Option object | Default: Medium `{"id": "10302"}` |
| Finder | `customfield_10644` | Option object | See allowed values below |

### Severity options

| Value | ID |
|-------|----|
| Critical | `10300` |
| High | `10301` |
| Medium | `10302` |
| Low | `10303` |

### Finder options

| Value | ID |
|-------|----|
| Client | `10342` |
| Customer Success | `10343` |
| Development | `10344` |
| Integrations | `10515` |
| Opsgenie | `10519` |
| Others | `10345` |
| Product Owner | `10518` |
| QA Automation | `10346` |
| QA Manual | `10347` |

## ADF format for textarea fields

Textarea custom fields (Steps to reproduce, Expected Behavior) must use Atlassian Document Format:

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "paragraph",
      "content": [
        {"type": "text", "text": "Your text here"}
      ]
    }
  ]
}
```

For ordered lists:

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "orderedList",
      "attrs": {"order": 1},
      "content": [
        {
          "type": "listItem",
          "content": [
            {
              "type": "paragraph",
              "content": [{"type": "text", "text": "Step 1"}]
            }
          ]
        }
      ]
    }
  ]
}
```

## Assignee

- Default: leave unassigned unless the user specifies one.
- Pere Joan Martorell: `712020:40debf5f-3891-4b09-9831-4a02f176d26b`
- For other users, use `lookupJiraAccountId` with their name or email.

## Components (`components` field)

| Name | ID |
|---|---|
| Management API | `10407` |
| Bigquery Push | `10408` |
| Customers | `10560` |
| Activity | `10406` |
| Segments | `10573` |
| Users & Roles | `10561` |
| Scores | `10581` |
| Loyalty Score Program | `10580` |

Set via `additional_fields`: `"components": [{"id": "10407"}]`

> **Important:** The parameter name is `additional_fields` (snake_case), NOT `additionalFields`. Using camelCase will silently drop all custom fields.

## Sprint assignment

### Known sprints (do NOT fetch these)

| Sprint name | ID |
|---|---|
| CDP - Backlog Fast Tech | `3631` |

Set directly: `"customfield_10010": 3631` (plain number, not an object).

### Finding the active sprint

When the user asks to assign to the active sprint:

1. Find an issue in the active sprint (avoid OpsGenie/Datadog-generated tickets — they often have no sprint):
   ```
   searchJiraIssuesUsingJql: "project = CDP AND sprint in openSprints() AND summary !~ 'Datadog' ORDER BY updated DESC"
   fields: ["customfield_10010"]
   ```
2. Extract the sprint ID from `customfield_10010[0].id`.
3. Set the sprint on the new issue via `editJiraIssue`:
   ```
   fields: { "customfield_10010": <sprint_id_as_number> }
   ```
   Note: the sprint ID must be a plain number (e.g., `3135`), NOT an object.

## Defaults

- **Content format**: use `"contentFormat": "markdown"` for the description field
- **Labels**: add relevant labels based on context (e.g., `sentry`, `airflow`, `bigquery`)

## Instructions

1. Gather the issue details from the user or from context (e.g., a Sentry issue investigation).
2. Determine the issue type (Bug, Task, or Story).
3. Create the ticket using `createJiraIssue` with the pre-configured values above. Do NOT call `getAccessibleAtlassianResources`, `getJiraProjectIssueTypesMetadata`, or `getJiraIssueTypeMetaWithFields` — all the needed IDs and field definitions are already listed above.
4. If it's a Bug, include all required custom fields with proper ADF formatting.
5. Always set the team to CDP via `customfield_10001`.
6. Return the ticket key and link: `https://loyal-guru.atlassian.net/browse/<TICKET_KEY>`
