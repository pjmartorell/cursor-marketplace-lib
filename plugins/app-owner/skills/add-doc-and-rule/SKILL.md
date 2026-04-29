---
name: add-doc-and-rule
description: Add a new documentation topic and matching Cursor rule to the Loyal Guru owner app. Use when introducing a new convention, pattern, or topic that should be documented and applied by file-pattern, or when the user asks to add docs or a Cursor rule for a topic.
---

# Add Doc and Cursor Rule

## Purpose

Keep [AGENTS.md](../../AGENTS.md) as the single index: every doc has a corresponding rule and a row in the Documentation table.

## Workflow

1. **Create `docs/<topic>.md`**
   - One topic per file; keep under 80 lines.
   - Inline key rules; prefer file-path references over long snippets.

2. **Create `.cursor/rules/<topic>.mdc`**
   - Frontmatter: `description`, and `globs` (file pattern) or `alwaysApply: true`.
   - Body: inline the most important rules; point to full doc: `Full reference: docs/<topic>.md`.
   - Keep rule under 25 lines.

3. **Update AGENTS.md**
   - Add a row to the Documentation table: Topic | [docs/<topic>.md](docs/<topic>.md) | glob or "always".

## Conventions

- Topic names: lowercase-with-hyphens (e.g. `forms-multiselect`, `code-style`).
- Rule scope: use globs that match where the convention applies (e.g. `src/**/*.component.ts`, `**/*.spec.ts`).
