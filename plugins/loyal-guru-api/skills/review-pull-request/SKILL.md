---
name: review-pull-request
description: >-
  Review a GitHub pull request against its linked Jira issue requirements.
  Fetches PR diff, reads the Jira ticket, analyzes all changed files, identifies
  gaps/bugs/improvements, and creates a pending GitHub review with categorized
  inline comments. Use when the user asks to review a PR, analyze a pull request,
  or check if a PR meets Jira requirements.
---

# Review Pull Request

## Workflow

### Step 1: Gather PR and Jira Context

Run these in parallel:

1. **PR metadata** — Use `pull_request_read` with `method: "get"` to get title, body, base/head SHA, and linked issue key.
2. **PR diff** — Use `pull_request_read` with `method: "get_diff"` to get the full diff (needed for inline comment line numbers).
3. **Jira issue** — Extract the issue key (e.g. `CDP-XXXXX`) from the PR title/body/branch. Fetch it with `getJiraIssue` (Atlassian MCP, `cloudId: "e622d9b6-bde4-49e2-bdc4-a938d2298b2b"`, `responseContentFormat: "markdown"`).
4. **Changed files** — Run `git diff origin/master...HEAD --name-only` to list files, then read every file changed in the PR.

### Step 2: Understand Requirements

From the Jira issue, extract:
- Required endpoints / features
- Expected request/response contracts
- Validations and edge cases
- Any "TBD" items that might be intentionally deferred

### Step 3: Analyze the Code

Check each area below. Only flag items that are genuinely problematic — avoid nitpicks.

#### A. Requirements Coverage
- Every endpoint/feature listed in the Jira ticket is implemented (or explicitly deferred with a follow-up ticket).
- Response contracts match what the ticket specifies (field names, types).
- All stated validations are present and tested.

#### B. Correctness
- Query logic is sound (JOINs, WHERE, GROUP BY, HAVING).
- Pagination: count query and data query use the same joins/filters.
- LEFT JOIN vs INNER JOIN: WHERE clauses on joined tables don't accidentally convert LEFT JOINs to INNER JOINs.
- Edge cases: nil handling, empty results, invalid input.

#### C. Performance
- N+1 queries (especially in representers/serializers that call AR finders per row).
- Unnecessary queries that could be avoided by passing already-loaded objects.

#### D. Security
- SQL injection: prefer parameterized queries over string interpolation.
- Authorization: Pundit policy present and correct.
- No secrets or credentials exposed.

#### E. Code Quality
- Follows existing codebase patterns (see workspace rules for Grape API and RSpec standards).
- Responses wrapped in a Representer.
- Tests use FactoryBot and the `response_json` helper.
- No RuboCop offenses.

### Step 4: Present Findings to User

Before creating the review, present a summary table to the user:

```
| Requirement | Status | Notes |
|---|---|---|
| Feature X | Implemented | ... |
| Feature Y | Missing | ... |
| Validation Z | Implemented + tested | ... |
```

And list any bugs, performance issues, or improvements found. Wait for the user to confirm or adjust before creating the GitHub review.

### Step 5: Create Pending Review

**Never submit the review directly.** Always create it as pending so the user can review and edit before submitting.

1. **Create the pending review** (no `event` parameter = pending):

```
Tool: pull_request_review_write
  method: "create"
  owner: "loyalguru"
  repo: "loyal-guru-api"
  pullNumber: <number>
  commitID: <head SHA>
  body: <review summary — see format below>
```

2. **Add inline comments** on specific lines:

```
Tool: add_comment_to_pending_review
  owner: "loyalguru"
  repo: "loyal-guru-api"
  pullNumber: <number>
  path: <file path relative to repo root>
  body: <comment text>
  subjectType: "LINE"
  line: <line number in the NEW file>
  side: "RIGHT"
  # For multi-line comments, also set:
  startLine: <first line>
  startSide: "RIGHT"
```

**Important:** The `line` parameter is the line number in the **new version** of the file (right side of the diff), not the diff hunk line number.

### Review Body Format

```markdown
## Review — <JIRA-KEY>

<1-2 sentence overall impression>

### Must fix
1. **Issue title** — Brief explanation.
2. ...

### Should fix
3. **Issue title** — Brief explanation.
4. ...

### Questions
5. **Topic** — Question for the author.
```

### Inline Comment Format

Each inline comment should:
- Start with a **bold title** summarizing the issue.
- Explain the **problem** concisely.
- Provide a **concrete fix** with code when possible (use fenced code blocks).

Example:
```markdown
**Count query / data query mismatch — pagination will be inaccurate**

The `count_query` only joins X ↔ Y, but `build_query` also joins Z.
This means `total_count` can be higher than actual rows → ghost pages.

**Proposed fix:**
\`\`\`ruby
def count_query
  # ... aligned query ...
end
\`\`\`
```

### Step 6: Inform the User

After creating the pending review, tell the user:
- How many inline comments were added.
- The review is **pending** and can be edited on the PR page.
- Provide the PR URL.

## Updating an Existing Pending Review

If the user asks to add, update, or remove comments from a pending review:

- **Add:** Use `add_comment_to_pending_review` directly.
- **Update:** Delete the pending review with `pull_request_review_write` (`method: "delete_pending"`), then recreate it with all comments (including the updated ones).
- **Remove:** Same delete-and-recreate approach, omitting the removed comment.

## Key Reminders

- Always read the **full diff** to get correct line numbers for inline comments.
- Cross-reference **every** requirement in the Jira ticket, not just the obvious ones.
- Flag fields/endpoints that are in the ticket but missing from the implementation.
- When suggesting code fixes, follow the patterns already in the codebase.
- The Atlassian cloud ID for this workspace is `e622d9b6-bde4-49e2-bdc4-a938d2298b2b`.
