---
name: integration-testing-fixtures
description: test.PrepareDatabase folders in order from testdata/fixtures/. testfixtures PostgreSQL. Double-load common then specific. TearDown or t.Cleanup. Examples api/, service/discount/, postgres/.
---

# Integration test fixtures

## Load

- `test.PrepareDatabase(folder1, folder2, ...)` — folders loaded **in order** from `testdata/fixtures/<folder>` relative to the package under test.
- [testfixtures](https://github.com/go-testfixtures/testfixtures) (PostgreSQL). Later folders add/overwrite rows.
- `test.TearDown(db)` or `t.Cleanup`.

## Fixture roots (examples)

| Package | Root |
|---------|------|
| `api` | `api/testdata/fixtures/` |
| `api/integration` | `api/integration/testdata/fixtures/` |
| `service/discount` | `service/discount/testdata/fixtures/` |
| `integration_tests/ticket_tests` | `integration_tests/ticket_tests/testdata/fixtures/` |
| `postgres` | `postgres/testdata/fixtures/` |

Elsewhere: `testdata/fixtures/` next to the test package.

## Common vs specific (double load)

- **Common**: shared base (companies, accounts, countries, currencies, locations, profiles, configs).
- **Specific**: scenario-only (rules, products, vouchers, score exchanges, …).

Always **common first**, then **specific**:

```go
db := test.PrepareDatabase("new_discount_engine_start_configuration", "redeem")
```

### Common files often

`accounts.yml`, `companies.yml`, `countries.yml`, `currencies.yml`, `locations.yml`, `profiles.yml`, `configurations.yml`, `currency_conversions.yml` (if needed).

### Specific often

`loyalty_rules.yml`, `loyalty_rule_histories.yml`, `products.yml`, `feature_products.yml`, `vouchers.yml`, `campaign_history_coupons.yml`, `score_exchanges.yml`, `rewards.yml`, …

## New / changed tests

- Prefer double load + reuse existing common for the area (e.g. `new_discount_engine_start_configuration` for discount engine).
- Split monolithic folders into common + specific when a base already exists.
- Keep order: common → specific.

## Code references

- `service/discount/redeem_test.go`: `PrepareDatabase("new_discount_engine_start_configuration", "redeem")`
- `api/integration/discount_test.go`: `PrepareDatabase("discounts/new_discount_engine_start_configuration", "discounts/redeem")`
- `api/tickets_test.go`: `configure_test` + base folder + scenario folder
