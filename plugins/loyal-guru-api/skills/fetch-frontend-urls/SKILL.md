---
name: fetch-frontend-urls
description: >-
  Find all frontend URLs and routes in the loyal-guru-app-owner-v2 Angular app
  that are affected by a backend API change. Searches routing modules, modal
  routes, module routes, and service files. Use when the user asks for affected
  frontend URLs, wants to know which pages use a given API endpoint, or needs
  frontend URLs to test after a backend permission or API change.
---

# Fetch Frontend URLs

Given a backend resource name (e.g. `segment_categories`, `rewards`, `messages`), find all frontend routes and pages in `loyal-guru-app-owner-v2` that call or reference that resource.

## Workflow

### Step 1: Determine the search term

From the user's request, extract:
- The **API endpoint name** (snake_case, e.g. `segment_categories`, `score_exchanges`)
- **Variations**: the singular form, camelCase, kebab-case, and any known aliases (e.g. `segment_categories` → `segment-categories`, `segmentCategories`, `segment_category`)

### Step 2: Search routing files (run all in parallel)

The owner app lives at `/Users/pmartorell/code/loyal-guru-app-owner-v2`.

Search for all route references across these file types:

1. **Top-level routing** — `src/app/app.routing.ts`
2. **Resource routing modules** — `src/app/resources/**/*-routing.module.ts`
3. **Module route files** — `src/app/resources/modules/**/*.routes.ts`
4. **Modal routing** — `src/app/shared/modals/modals-routing.module.ts`

Use Grep with the search term variations across these files. For each match, extract:
- The `path:` value
- The parent route context (read surrounding lines to get the full nested path)

### Step 3: Search service files

Search for API endpoint references in service files:

```
Grep pattern: <endpoint_name>
Path: /Users/pmartorell/code/loyal-guru-app-owner-v2/src/app
Glob: *.service.ts
```

This reveals which Angular services call the API endpoint, confirming which features are affected.

### Step 4: Search component files

Search component files for direct references to the resource:

```
Grep pattern: <endpoint_name>
Path: /Users/pmartorell/code/loyal-guru-app-owner-v2/src/app
Glob: *.component.ts
```

Look for `apiEndPoint:` values in data table configurations and `routerLink` or `router.navigate` calls — these indicate direct URL usage.

### Step 5: Reconstruct full URLs

Angular routes are nested. Build the full URL by tracing the route hierarchy:

1. Find the **top-level path** in `app.routing.ts` (e.g. `path: 'segments'` → `loadChildren: SegmentsModule`)
2. Follow into the loaded module's routing file to get child paths
3. Also check `modules/**/*.routes.ts` — the same component may be mounted under a different module path (e.g. `campaign-personalization/audiences/categories` reuses `TabSegmentsCategoriesComponent`)

**Modal routes** use Angular named outlets. Represent them as:
- `(modal:new/segment_categories)` — from `modals-routing.module.ts`

**Full page routes** are joined with `/`:
- Parent `campaign-personalization` + child `audiences` + child `categories` → `/campaign-personalization/audiences/categories`

### Step 6: Format output

Present results grouped by type:

```markdown
**Frontend pages**
- `/campaign-personalization/audiences/categories` — Segment categories list
- `/segments/segment_categories/:id` — Segment category detail page
- `(modal:new/segment_categories)` — Create new segment category
- `(modal:show/segment_categories/:id)` — Show segment category
- `(modal:update/segment_categories/:id)` — Edit segment category

**API service files**
- `src/app/resources/segments/segment-categories.service.ts` — main service
```

## Key locations reference

| What | Path |
|------|------|
| Owner app root | `/Users/pmartorell/code/loyal-guru-app-owner-v2` |
| Top-level routing | `src/app/app.routing.ts` |
| Resource routing | `src/app/resources/**/*-routing.module.ts` |
| Module routes | `src/app/resources/modules/**/*.routes.ts` |
| Modal routing | `src/app/shared/modals/modals-routing.module.ts` |
| Services | `src/app/**/*.service.ts` |
| Components | `src/app/**/*.component.ts` |

## Common pitfalls

- **Module routes are easy to miss**: The same component (e.g. `TabSegmentsCategoriesComponent`) can be mounted under multiple module paths. Always search `modules/**/*.routes.ts` in addition to the resource's own routing module.
- **Modal routes use named outlets**: They are in `modals-routing.module.ts`, not in the resource routing files.
- **Services reveal API endpoint names**: The `apiEndPoint` property in data table configs and HTTP calls in `.service.ts` files confirm the exact API path used.
