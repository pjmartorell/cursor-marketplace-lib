---
name: add-i18n-keys
description: Add or update translation keys in the Loyal Guru owner app i18n JSON files. Use when adding new UI strings, labels, or messages that need translation, or when the user mentions i18n, translations, or locale files.
---

# Add i18n Keys

## Scope

- **Files**: `src/assets/i18n/*.json`
- **Locales**: `en`, `es`, `it`, `fr`, `hr`, `ro`

## Workflow

1. **Add key to `en.json`** (source of truth). Use nested keys (e.g. `"section": { "subsection": { "key": "Value" } }`).
2. **Add same key to `es.json`, `it.json`, `fr.json`** with translated values for those locales.
3. **Add same key to `ro.json` and `hr.json`** with the **same English value** as in `en.json`. Do not translate into Romanian or Croatian; these locales exist for date/number formatting only.

## Rules

- Keep JSON valid (trailing commas, quoted keys).
- Preserve existing key order / structure when adding to an existing section.
- For new keys, follow existing nesting patterns in the file.

## Reference

- [docs/project-overview.md](../../docs/project-overview.md) — i18n and ro/hr convention.
