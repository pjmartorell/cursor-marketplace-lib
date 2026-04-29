---
name: configurations
description: >-
  Tenant-scoped Management API configurations (`entity` + `slug`): `Configuration` model, `/v1/configurations`, Angular `ConfigurationService`.
  Detail: `docs/CONFIGURATIONS.md`. Code-derived key list: `docs/CONFIGURATION_INVENTORY.md`.
  Use when creating, reading, patching, documenting, or debugging those surfaces.
---

# Configurations (`entity` + `slug`)

Use [docs/CONFIGURATIONS.md](/docs/CONFIGURATIONS.md) for JSONB shape, Rails helpers, `fallback_to_public` / public merge, HTTP API, and Angular payload notes. Use [docs/CONFIGURATION_INVENTORY.md](/docs/CONFIGURATION_INVENTORY.md) for the code-derived `(entity, slug)` list and dynamic key families.

## Rules of thumb

- Prefer `Configuration.get_configuration_value(...)` for simple reads; use `get_configuration(...)` when you need the full row.
- Only use `fallback_to_public: true` when you explicitly intend a platform-wide default with tenant override semantics.
- If a setting is boolean-like, coerce with `ActiveModel::Type::Boolean` before branching (JSON can store strings).
