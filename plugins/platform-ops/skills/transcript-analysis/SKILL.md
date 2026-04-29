---
name: transcript-analysis
description: Analyze Cursor agent transcripts to rank MCP tool calls and skill reads. Produces a canvas with charts and a full ranking table. Use when the user asks which MCP tools, skills, or capabilities have been used the most, wants a usage report, or mentions agent transcripts, tool usage, or skill usage.
---

# Transcript Analysis

Ranks MCP tool calls and skill reads across all agent transcripts in `~/.cursor/projects/`.

## Workflow

### 1. Run the analysis script

```bash
python3 ~/.cursor/skills/transcript-analysis/analyze.py
```

The script scans every `.jsonl` transcript file under all `~/.cursor/projects/*/agent-transcripts/` directories and outputs three sections:
- `TRANSCRIPTS` — total transcript count
- `SKILLS` — skill reads (count + path), sorted descending
- `MCP_TOOLS` — MCP tool calls (`server / toolName`), sorted descending

### 2. Post-process the output

Parse the tab-separated output into two ranked lists:
- **Skills**: normalize paths to short names (e.g. extract the skill directory name from the path). Group by category: Git plugin, Cursor system, project skills (by repo name), personal (`~/.cursor/skills/`), public plugins.
- **MCP tools**: group by server (normalize `user-postgres-production`, `user-postgres`, `user-postgres-staging1`, `user-postgres-staging2` → "postgres (all envs)").

### 3. Produce a canvas

Read the canvas skill and create a canvas at the standard path. Include:
- **3 summary stats**: total transcript count, distinct skills, total MCP calls
- **BarChart** — top 10 skills (horizontal)
- **BarChart** — MCP tools by server (horizontal, grouped)
- **Table** — full skill ranking (rank, reads, name, category, note)
- **Table** — full MCP tool ranking (calls, server, tool)
- **Insights section** with `<Text weight="medium">` highlights

Canvas path: `/Users/<user>/.cursor/projects/<active-workspace>/canvases/transcript-analysis.canvas.tsx`

## Notes

- Skill reads are detected by `Read` tool calls whose `path` contains `SKILL.md` or ends in `.md` inside a `/skills/` directory.
- MCP calls are detected by `CallMcpTool` tool uses; `server` and `toolName` are both captured.
- Transcripts from all workspaces are included — not just the active one. This often reveals heavy usage in secondary workspaces (e.g. a 155-transcript workspace that dwarfs the current one).
- The `apm-gif-search` skill also reads a helper script (`fetch_gifs.py`) — count those separately as "script reads" in the note column.
