---
name: segment-engine
description: Work with the Segment Engine — customer segmentation, condition trees, BigQuery query construction, and tagging. Use when creating, modifying, or debugging segments, segment conditions, segment queries, tagging workers, or when the user mentions segments, segmentation, or customer audiences.
---

# Segment Engine

For full architecture details, see [/docs/SEGMENTS.md](/docs/SEGMENTS.md).

## Key files

| Area | Path |
|------|------|
| Segment model | `app/models/segment.rb` |
| Condition model | `app/models/condition.rb` |
| Query builder (entry point) | `app/services/segment_engine/segmentate.rb` |
| Root AND/OR combinator | `app/services/segment_engine/conditions/group.rb` |
| All condition types | `app/services/segment_engine/conditions/` |
| Operator expressions | `app/services/segment_engine/conditions/operator.rb` |
| Sales base class | `app/services/segment_engine/conditions/base_sales_by.rb` |
| Nested segment condition | `app/services/segment_engine/conditions/nested.rb` |
| Raw SQL condition | `app/services/segment_engine/conditions/sql.rb` |
| Customer list pagination | `app/services/segment_engine/customer_list.rb` |
| Tagging worker | `app/workers/segment_manager_tag_worker.rb` |

## How queries are built

`Segment#query_for(mode, options)` → `Segmentate#build_query`:

```sql
SELECT {headers}
FROM {dataset}.accounts JOIN {dataset}.profiles
JOIN ({conditions_query | static_query}) AS segment ON segment.customer_id = accounts.id
WHERE {quick_filters}
```

All queries run against **BigQuery** (`company.bigquery_dataset_id`), never directly against PostgreSQL.

## Adding a new condition type

1. Create `app/services/segment_engine/conditions/<name>.rb` implementing `#query` that returns a BigQuery SQL string yielding `customer_id` rows.
2. The class is resolved at runtime via `key_field.camelize.constantize` — the `key_field` stored on the `Condition` record must exactly match the class name (underscored).
3. Inherit from `BaseSalesBy` for purchase-based conditions, `BaseFilter` for simple field comparisons.
4. Add specs in `spec/services/segment_engine/conditions/`.

## Condition configuration

Each `Condition` has a `configuration` JSON blob. Contents are condition-specific — inspect the relevant class to understand expected keys. Common patterns:

- `operator`: `'and'` / `'or'` (groups), `'in'` / `'not_in'` (membership), or an `Operator` alias (`eq`, `gt`, `bt`, etc.)
- `date_from` / `date_to`: relative date strings parsed by `DateParser`
- `location_ids` / `channel_slugs`: array filters on activities
- `segment_id` + `apply`: for `nested` conditions

## Raw SQL condition (`Conditions::Sql`)

- `configuration['sql_query']`: the raw BigQuery SQL
- `[company]` is substituted with the company's dataset id
- `%today%` is substituted with the evaluation date
- Must return a `customer_id` column

## Static vs dynamic

- **Dynamic**: conditions re-evaluated every run.
- **Static** (`tag.applies > 0`): reads last tagging result from `dataset.tag_customers`. No conditions run.

## Gotchas

- `key_field` is resolved at runtime — a typo raises `NameError` only when the query is built, not on save.
- `evaluation_date` defaults to `Date.current` but can be overridden (used for backfill and scheduled tagging).
- Groups support AND (`INTERSECT DISTINCT`) and OR (`UNION DISTINCT`). The API enforces max 1 level of nesting.
- Feature flag `segment-use-customers-table` switches the core table from `accounts JOIN profiles` to `customers` (per-company, off by default).
