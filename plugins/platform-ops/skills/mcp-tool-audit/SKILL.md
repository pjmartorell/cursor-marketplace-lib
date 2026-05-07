---
name: mcp-tool-audit
description: Audit an MCP server by mining Cursor agent transcripts for tool call failures, repeated retries, validation errors, and capability gaps. Produces a ranked list of issues with root causes and concrete description/schema fixes. Use when the user asks to improve an MCP, reduce failure rate, analyze which MCP tools fail, or review tool descriptions.
---

# MCP Tool Audit

Mines agent transcripts to find where a specific MCP server causes friction, then produces concrete improvements to tool descriptions and schemas.

## Workflow

### 1. Discover transcripts

```bash
find /Users/<user>/.cursor/projects -name "*.jsonl" -path "*/agent-transcripts/*" \
  | xargs grep -l "<mcp-server-name>" 2>/dev/null
```

### 2. Read tool descriptors

Read every `tools/*.json` file in `~/.cursor/projects/<workspace>/mcps/<server>/tools/` to understand current schemas.

### 3. Extract tool calls and error signals

The transcript format is one JSON object per line: `{"role": "...", "message": {"content": [...]}}`.  
Tool calls are in `assistant` lines as `{"type": "tool_use", "name": "CallMcpTool", "input": {"server": "...", "toolName": "...", "arguments": {...}}}`.  
Results are **not** stored as separate lines — read the following `text` blocks from the assistant for error signals.

Write a Python script to:
1. Find all `CallMcpTool` blocks where `input.server` matches the target MCP
2. For each call, collect the next 5–8 assistant `text` blocks
3. Detect issue patterns in that text (see patterns below)
4. Aggregate by tool name

```python
import json, glob
from collections import defaultdict

files = glob.glob("/Users/<user>/.cursor/projects/*/agent-transcripts/**/*.jsonl", recursive=True)
# filter to files containing the server name, then parse as above
```

### 4. Issue patterns to detect

| Pattern | Keywords to match in following text |
|---------|-------------------------------------|
| Retry / filter adjustment | `let me try`, `different approach`, `adjust the filter`, `broaden`, `retry`, `instead try` |
| No results (wrong project) | `no results`, `no logs`, `empty`, `0 logs`, `no entries`, returning empty |
| Permission / auth error | `permission denied`, `unauthorized`, `403`, `401`, `access denied` |
| Validation / schema error | `validation error`, `invalid argument`, `bad request`, `400`, `schema`, `required field` |
| Capability gap | `not supported`, `can't query`, `only supports`, `doesn't have`, `no way to` |
| Truncation | `truncated`, `cut off`, `full message`, `getlogdetail` to get full |

### 5. Identify root causes

For each issue cluster, look at the tool descriptor (`tools/<toolName>.json`) and ask:
- Is the parameter description missing context the agent needed?
- Is there a known-values list (project IDs, resource types, enum values) that would have prevented the wrong guess?
- Does the description explain what the tool *cannot* do (scope limits, unsupported params)?
- Are there interaction effects between parameters that aren't documented?

### 6. Produce improvements

For each issue, write a concrete fix:
- **Description patch**: exact new text for the field or tool description
- **Schema addition**: new parameter or enum to add
- **Limitation note**: caveat to add so the agent doesn't attempt unsupported use cases

### 7. Output

Produce a markdown report (save as `IMPROVEMENTS.md` in the MCP repo if it exists) with:
- Summary stats: total calls, retry rate per tool
- Active issues ranked by occurrence count, each with: description, root cause, concrete fix
- Fixed issues table (to avoid reopening them)
- Appendix: useful parameter examples and gotchas

Also create a canvas for visual presentation.

## Key lessons from past audits

### cloud-logging MCP (May 2026)

Common failure patterns found:

- **Wrong project ID (62% retry rate)**: No mapping of service → project in the `projectId` description. Fix: inline a known-projects table.
- **Org-level logs undiscoverable**: `resourceNames: ["organizations/<id>"]` works but is undocumented. Agents never find it.
- **`summaryFields` silently fails on protobuf paths**: Works for `textPayload`/`jsonPayload.*` but not `protoPayload.*`. Needs a note.
- **`aggregateLogs` missing `resourceNames`**: Agents copy the pattern from `queryLogs` and get silently wrong scope.
- **`getLogDetail` fails for org-scoped log IDs**: Tool description doesn't warn that it only works for project-level IDs.
- **`SEARCH()` + field filter without `AND`**: Returns empty results; needs an explicit warning in examples.

These patterns generalize: the most common MCP failure is the agent not knowing *which project/resource/scope to use*, not a code bug.
