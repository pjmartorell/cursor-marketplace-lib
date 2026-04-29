---
name: kallax-removal
description: Remove Kallax ORM from a model and replace all generated store/query/transaction usage with sqlbuilder + sqlx. Use when removing Kallax from a postgres/ service, stripping kallax.Model from structs, cleaning up kallax.go, or running QA tests after a Kallax removal PR.
---

# Kallax Removal

Reference implementations: `postgres/location_service.go`, `postgres/location_taxonomy_service.go`, `postgres/score_store.go`.

## 1. Domain struct

Remove `kallax.Model` embedding and the `kallax` import. Add explicit `db:` tags for sqlx.

```go
// Before
type LocationTaxonomy struct {
    kallax.Model
    Name   string
    Slug   string
    Origin string
}

// After
type LocationTaxonomy struct {
    ID     int64  `db:"id"     json:"id"`
    Name   string `db:"name"   json:"name"`
    Slug   string `db:"slug"   json:"slug"`
    Origin string `db:"origin" json:"origin"`
}
```

Rules:
- All DB columns → `db:"column_name"` (snake_case matching the table).
- Computed / non-column fields (e.g. `Features`, `FullName`) → `db:"-"`.
- Timestamps: `db:"created_at"`, `db:"updated_at"` — **do not alias** in SELECT queries.

## 2. Service struct

Remove `storeCache` and `getStore()`. Inject `*Pool` only.

```go
type LocationTaxonomyService struct {
    pool *Pool
}
func NewLocationTaxonomyService(pool *Pool) *LocationTaxonomyService {
    return &LocationTaxonomyService{pool: pool}
}
```

## 3. CRUD replacements

### FindOne → sqlbuilder SELECT + sqlx.Get

```go
sb := sqlbuilder.PostgreSQL.NewSelectBuilder()
sb.Select("id", "name", "slug", "origin")
sb.From("location_taxonomies")
sb.Where(sb.E("slug", slug))
query, args := sb.Build()

var lt streaming.LocationTaxonomy
if err := ls.pool.Get(company).Get(&lt, query, args...); err != nil {
    if errors.Is(err, sql.ErrNoRows) {
        return nil, nil   // or streaming.ErrXxxNotFound
    }
    return nil, err
}
```

### FindAll → sqlbuilder SELECT + sqlx.Select

```go
var items []*streaming.LocationFeature
if err := ls.pool.Get(company).Select(&items, query, args...); err != nil {
    return nil, err
}
```

### Insert → raw INSERT ... RETURNING id

```go
err := ls.pool.Get(company).QueryRowContext(ctx,
    `INSERT INTO location_taxonomies (name, slug, origin) VALUES ($1, $2, $3) RETURNING id`,
    lt.Name, lt.Slug, lt.Origin,
).Scan(&lt.ID)
```

### Update → sqlbuilder UpdateBuilder

```go
ub := sqlbuilder.PostgreSQL.NewUpdateBuilder()
ub.Update("location_taxonomies")
ub.Set(ub.Assign("name", lt.Name))
ub.Where(ub.E("id", lt.ID))
query, args := ub.Build()

if _, err := ls.pool.Get(company).Exec(query, args...); err != nil {
    return err
}
```

### Transaction (cascaded delete / multi-step write) → BeginTxx

```go
db := ls.pool.Get(company)
tx, err := db.BeginTxx(ctx, nil)
if err != nil {
    return err
}
defer tx.Rollback()

if _, err := tx.ExecContext(ctx,
    "DELETE FROM location_taxonomy_terms WHERE taxonomy_slug=$1", slug,
); err != nil {
    return err
}
if _, err := tx.ExecContext(ctx,
    "DELETE FROM location_taxonomies WHERE slug=$1", slug,
); err != nil {
    return err
}
return tx.Commit()
```

### Unique-violation guard (on Insert)

```go
if pqErr, ok := err.(*pq.Error); ok {
    if pqErr.Code.Name() == "unique_violation" {
        return streaming.ErrLocationAlreadyExists
    }
}
```

## 4. kallax.go cleanup

`make kallax` panics on Go ≥ 1.20 (generator incompatibility). **Edit `kallax.go` manually.**

For each removed model, delete all generated blocks:
- `type <Model>Store struct` + all its methods
- `type <Model>Query struct` + all its methods
- `type <Model>ResultSet struct` + all its methods
- `schema<Model>` type and its `newSchema<Model>()` factory
- The `Schema.<Model>` field initialisation inside `var Schema = struct { ... }{ ... }`
- All model methods that reference the removed types (`GetID`, `ColumnAddress`, `Value`, `NewRelationshipRecord`, `SetRelationship`, …)

After editing: `make validate` must pass before committing.

## 5. Commit strategy (multi-model PR)

When removing Kallax from several related models in one PR, use one commit per service layer model (keeping `kallax.Model` temporarily in structs for compilation), then a final cleanup commit:

| Commit | Content |
|--------|---------|
| `[CDP-XXXX] Replace Kallax in FooService` | Service file only; struct still has `kallax.Model` |
| … one per model … | |
| `Remove kallax.Model from structs, add db: tags, clean kallax.go` | All structs + `kallax.go` |

This keeps each commit reviewable and the branch always green.

## 6. QA tests

### Environment setup

```bash
BASE="https://<env>.appspot.com"   # or API Gateway URL
H1="X-Api-Key: <key>"
H2="X-Api-Secret: <secret>"
H3="X-Api-Version: 1"
H4="Content-Type: application/json"
H5="Accept: application/json"
H6="Accept-Language: en"
```

### Minimum test matrix for Location-family models

| # | Operation | What to verify |
|---|-----------|---------------|
| GET /:id | 200 + correct fields | id, code, features array populated |
| GET /:id (missing) | 404 + correct error code | `location_not_found` |
| POST (no features) | 200 + created record | id assigned, code stored |
| POST (with features) | 200 + GET confirms features | features persisted in `location_terms` |
| POST (duplicate) | 4xx + `already_exists` code | pq unique_violation mapped correctly |
| PATCH (fields) | 200 + GET confirms | updated fields persisted |
| PATCH (replace external features) | 200 + GET confirms | only `origin='external'` terms replaced |
| PATCH (empty features) | 200 + GET confirms | external terms removed, internal kept |
| DELETE | 200 + DB confirms | row gone, cascaded rows gone |
| Taxonomy POST / PATCH / DELETE | 200 each | cascade on delete verified in DB |
| Term POST / PATCH / DELETE | 200 each | term id populated on create |

### DB reconciliation

The SQL queries below work the same way regardless of how you connect. Pick whichever option is available to you.

#### Option A — postgres-staging2 Cursor MCP (recommended if available)

Add this block to your `~/.cursor/mcp.json` (connection string from 1Password → *Streaming API staging DB*):

```json
"postgres-staging2": {
  "command": "npx",
  "args": [
    "-y",
    "@modelcontextprotocol/server-postgres",
    "postgresql://<user>:<password>@<host>/staging2?sslmode=require"
  ],
  "env": {
    "NODE_TLS_REJECT_UNAUTHORIZED": "0"
  }
}
```

Once configured, the Cursor agent can run read-only SQL directly in chat — no terminal needed.

#### Option B — psql (always available, read-only enforced)

Run individual queries with `-c` (no interactive session, no accidental writes):

```bash
# Get the connection string from 1Password → "Streaming API staging DB"
psql "postgresql://<user>:<password>@<host>/staging2?sslmode=require" \
  -c "SELECT id FROM demo1.locations WHERE code = 'my_code';"
```

If you need an interactive session, lock it to read-only immediately on connect:

```bash
psql "postgresql://<user>:<password>@<host>/staging2?sslmode=require" \
  --single-transaction \
  -c "SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY;" \
  -c "\i /dev/stdin"
```

Or inside the psql prompt:

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY;
-- any subsequent write will now raise ERROR:  cannot execute ... in a read-only transaction
```

#### Reconciliation queries

```sql
-- Verify features for a location
SELECT lt.location_taxonomy_term_id, lt.location_taxonomy_slug
FROM <tenant>.location_terms lt WHERE lt.location_id = <id>;

-- Verify record deleted
SELECT id FROM <tenant>.locations WHERE code = '<code>';

-- Verify taxonomy deleted
SELECT id FROM <tenant>.location_taxonomies WHERE slug = '<slug>';

-- Verify taxonomy term deleted
SELECT id FROM <tenant>.location_taxonomy_terms
WHERE taxonomy_slug = '<slug>' AND external_id = <external_id>;
```

Replace `<tenant>` with the company slug used in the request (e.g. `demo1`).

### Regression check (compare with previous branch)

Run the same PATCH → GET sequence on **both** the branch env and the master env to confirm:
- PATCH response asymmetry (only shows features from request) → pre-existing, not a regression.
- GET returns complete feature set including internal taxonomy features → expected.
- Kallax known bug: on master, GET after PATCH may return **duplicate** feature entries. The sqlbuilder implementation returns each feature once (this is an improvement).

### Key known behaviors

- **PATCH/GET asymmetry**: `Update()` returns `found` built by `getById()` which doesn't load features. PATCH response only echoes features sent in the body. GET always returns the full set. Pre-existing in Kallax; preserved by design.
- **External vs internal features**: `deleteLocationExternalFeatures` only deletes `location_terms` rows whose `location_taxonomy_slug` has `origin='external'` in `location_taxonomies`. Internal taxonomy features survive PATCH.
- **`make kallax` broken on Go ≥ 1.20**: Edit `kallax.go` manually; always run `make validate` to confirm consistency before committing.
