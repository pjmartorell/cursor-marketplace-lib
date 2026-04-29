---
name: skill-usage-trace
description: Find and explain every conversation where a specific Cursor skill was triggered. Shows which transcripts read the skill vs merely mentioned it, the original user query, and the agent's reasoning at each read. Use when the user asks in which context a skill was used, which conversations triggered a skill, why a skill was activated, or to understand skill trigger patterns.
---

# Skill Usage Trace

Searches all agent transcripts across `~/.cursor/projects/` for a specific skill, distinguishing actual reads (tool_use) from text mentions, and reporting the agent's reasoning at each read point.

## Workflow

### 1. Run the trace script

```bash
python3 ~/.cursor/skills/skill-usage-trace/trace.py <skill-name-or-path-fragment>
```

The fragment is matched case-insensitively against the full file path. Use the skill directory name (e.g. `safe-migrations`, `fetch-frontend-urls`, `canvas`).

Output per matching transcript:
- `[READ]` — the skill's `SKILL.md` was actually opened via the `Read` tool
- `[MENTION ONLY]` — the skill name appeared in assistant text but was never read
- User query that started the conversation
- Per read: message index, file path, and agent reasoning text immediately before the read

### 2. Report findings in chat

After running, summarize:
- How many transcripts total vs how many had actual reads
- For each READ transcript: user query + why the agent reached for the skill (quote the reasoning)
- For MENTION ONLY transcripts: note they don't count as activations
- Patterns: was the skill proactively reached for, or triggered by a specific keyword?

### Notes

- Multiple reads in one transcript = the skill was re-read across subagent steps in a long session (412-message sessions are common).
- The script captures the assistant's text in the **same message** as the Read tool call. This is the agent's stated reasoning for reading it.
- Reads at message 0–30 = proactive (agent checked the skill early). Reads at high message indices = the agent revisited the skill mid-session (e.g. to update it, or because a new subagent started).
- Transcripts from all workspaces are included — not just the active one.
